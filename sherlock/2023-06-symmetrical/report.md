| Severity | Title | 
|:--:|:---|
| [H-01](#h-01-liquidators-can-front-run-the-system-or-other-liquidations-to-receive-100-of-the-profit-with-small-amounts-of-gas-paid) | Liquidators can front-run the system or other liquidations to receive 100% of the profit with small amounts of gas paid |
| [M-01](#m-01-suspended-partyb-users-can-transfer-their-funds-to-not-suspended-partya-to-withdraw-them) | Suspended partyB users can transfer their funds to not suspended partyA to withdraw them |
| [M-02](#m-02-inefficient-allocation-in-depositandallocateforpartyb-low-allocation-amounts-for-partyb) | Inefficient allocation in `depositAndAllocateForPartyB()`: Low allocation amounts for PartyB |





# [H-01] Liquidators can front-run the system or other liquidations to receive 100% of the profit with small amounts of gas paid

## Summary
Liquidators can front-run other liquidators or the system (when liquidating users with too small debts to be profitable), for small amounts of gas gaining 100% of the profits from liquidation and paying small amounts of gas, thus leaving 0% of the profits for the front-runned liquidator who pays the large gas cost. 
## Vulnerability Detail
Since the system works by using 4 liquidate function that should be called after each other a liquidator seeking bigger profits can actually pay small amounts of gas calling the first 2, and leave the heavy gas consuming latter 2 for the other liquidator, and because the rewards are calculated within the first 2 function the fist liquidator gains 100% of the rewards, while the second one does not make any money in return.

**Example:**

 - **LiquidatorA:** tries to call the 4 functions in a row (to liquidate partyA)  [`liquidatePartyA()`](https://github.com/sherlock-audit/2023-06-symmetrical/blob/main/symmio-core/contracts/facets/liquidation/LiquidationFacetImpl.sol#L20-L32) => [`setSymbolsPrice()`](https://github.com/sherlock-audit/2023-06-symmetrical/blob/main/symmio-core/contracts/facets/liquidation/LiquidationFacetImpl.sol#L34-L97) => [`liquidatePendingPositionsPartyA()`](https://github.com/sherlock-audit/2023-06-symmetrical/blob/main/symmio-core/contracts/facets/liquidation/LiquidationFacetImpl.sol#L99-L124) => [`liquidatePositionsPartyA()`](https://github.com/sherlock-audit/2023-06-symmetrical/blob/main/symmio-core/contracts/facets/liquidation/LiquidationFacetImpl.sol#L126-L238)

 - **LiquidatorB:** front-runs only the first 2 functions `liquidatePartyA()` and `setSymbolsPrice()` and since the liquidator's profit is calculated only in these functions he will receive 100% of money from liquidation.

In [`liquidatePartyA()`](https://github.com/sherlock-audit/2023-06-symmetrical/blob/main/symmio-core/contracts/facets/liquidation/LiquidationFacetImpl.sol#L20-L32)
```jsx
AccountStorage.layout().liquidators[partyA].push(msg.sender);
```
Also in [`setSymbolsPrice()`](https://github.com/sherlock-audit/2023-06-symmetrical/blob/main/symmio-core/contracts/facets/liquidation/LiquidationFacetImpl.sol#L34-L97)
```jsx
AccountStorage.layout().liquidators[partyA].push(msg.sender);
```
And finally the [`liquidatePositionsPartyA()`](https://github.com/sherlock-audit/2023-06-symmetrical/blob/main/symmio-core/contracts/facets/liquidation/LiquidationFacetImpl.sol#L126-L238) sends the fees, but since bolt `[accountLayout.liquidators[partyA][0]` and `[accountLayout.liquidators[partyA][1]` is **liquidatorB** he gets the profits.
```jsx
            uint256 lf = accountLayout.liquidationDetails[partyA].liquidationFee;
            if (lf > 0) {
                accountLayout.allocatedBalances[accountLayout.liquidators[partyA][0]] += lf / 2;
                accountLayout.allocatedBalances[accountLayout.liquidators[partyA][1]] += lf / 2;
            }
```
`liquidatePartyA()` and `setSymbolsPrice()` quite cheap on gas, they are just setters of variable and don't do many calculations. On the other hand `liquidatePendingPositionsPartyA()` `liquidatePositionsPartyA()` are quite heavy in gas since they iterate (with a `for()` loop) over pending or locked quotes of users.

Another profitable scenario is when the system liquidates players with small positions, because it is not profitable for normal liquidators to liquidate them. Instead smart liquidators can front-run the system with `liquidatePartyA()` and `setSymbolsPrice()` and leave the system to pay the high gas cost,
## Impact
Liquidators gaming the system or other liquidators.
## Code Snippet
[`liquidatePartyA()`](https://github.com/sherlock-audit/2023-06-symmetrical/blob/main/symmio-core/contracts/facets/liquidation/LiquidationFacetImpl.sol#L20-L32)
[`setSymbolsPrice()`](https://github.com/sherlock-audit/2023-06-symmetrical/blob/main/symmio-core/contracts/facets/liquidation/LiquidationFacetImpl.sol#L34-L97)
[`liquidatePendingPositionsPartyA()`](https://github.com/sherlock-audit/2023-06-symmetrical/blob/main/symmio-core/contracts/facets/liquidation/LiquidationFacetImpl.sol#L99-L124)
[`liquidatePositionsPartyA()`](https://github.com/sherlock-audit/2023-06-symmetrical/blob/main/symmio-core/contracts/facets/liquidation/LiquidationFacetImpl.sol#L126-L238)
## Tool used

Manual Review

## Recommendation
I find it quite difficult to recommend a simple solution to this issue. Best this I can suggest is to make a single function that combines the 4 and use that, or make the calculation of  rewards in the latter 2, because they re more gas heavy.


# [M-01] Suspended partyB users can transfer their funds to not suspended partyA to withdraw them


## Summary
Suspended partyB users can transfer their funds to not suspended partyA (can be account under their control) to withdraw them 
## Vulnerability Detail
Bolth methods for withdraw and even the one for opening a quote (`sendQuote()`) have a modifier [`notSuspended(msg.sender)`](https://github.com/sherlock-audit/2023-06-symmetrical/blob/main/symmio-core/contracts/utils/Accessibility.sol#L73-L79), this modifier is preventing suspended  users from withdrawing any balances from the system. However there is no check on partyB side if this user is suspended, thus allowing suspended partyB users to create partyA account and trade for loss on B side, to transfer the funds to A side.

Example(it's quite long):

B is suspended and wants to withdraw his money from the system, so he creates A account to help him with the task.

**A:** opens up a  LIMIT SHORT position on BTC for 30k (current price is 25k), this guaranties immediate wins for A, he also sets  `partyBsWhiteList` to his B address and finnaly sets lf, mm, cva to approparete parameters for such a trade, all other parameters are irelevant

```jsx
    function sendQuote(
        address[] memory partyBsWhiteList,// his B address
        uint256 symbolId, //BTC/USD
        PositionType positionType, //SHORT
        OrderType orderType, //LIMIT
        uint256 price, // 30 000e18
        uint256 quantity, //1 BTC 
    ) 
```
**B:** [`lockQuote()`](https://github.com/sherlock-audit/2023-06-symmetrical/blob/main/symmio-core/contracts/facets/PartyB/PartyBFacetImpl.sol#L22-L38) and afterwards calls [`openPosition()`](https://github.com/sherlock-audit/2023-06-symmetrical/blob/main/symmio-core/contracts/facets/PartyB/PartyBFacetImpl.sol#L112-L254) with 
```jsx
    function openPosition(
        uint256 filledAmount,// 100% => 1 BTC => 1e18
        uint256 openedPrice,// A requested price 30 000e18
    )
```
This position passes the important [require](https://github.com/sherlock-audit/2023-06-symmetrical/blob/main/symmio-core/contracts/facets/PartyB/PartyBFacetImpl.sol#L137-L140) and [solvency](https://github.com/sherlock-audit/2023-06-symmetrical/blob/main/symmio-core/contracts/libraries/LibSolvency.sol#L15-L97) checks and afterwards the quote becomes **OPENED**:
```jsx
else {
  require(openedPrice >= quote.requestedOpenPrice,"PartyBFacet: Opened price isn't valid");
}
LibSolvency.isSolventAfterOpenPosition(quoteId, filledAmount, upnlSig);
```
**A:**  calls `requestToClosePosition()` again with the same parameters (I have included the important ones only, the rest are irrelevant):
```jsx
    function requestToClosePosition(
        uint256 closePrice,// same price from beginning  30 000e18
        uint256 quantityToClose,// 1 BTC=> 1e18
        OrderType orderType,// LIMIT
    )
```
**B:** Finally calls [`fillCloseRequest()`](https://github.com/sherlock-audit/2023-06-symmetrical/blob/main/symmio-core/contracts/facets/PartyB/PartyBFacetImpl.sol#L256-L293)
```jsx
    function fillCloseRequest(
        uint256 filledAmount, // 100%, 1 BTC => 1e18
        uint256 closedPrice, // 30 000e18
    )
```
The parameters are checked if they match **A**'s request and the quote is [closed](https://github.com/sherlock-audit/2023-06-symmetrical/blob/main/symmio-core/contracts/libraries/LibQuote.sol#L149-L208). On the close request, profits are calculated and funds are [distributed around](https://github.com/sherlock-audit/2023-06-symmetrical/blob/main/symmio-core/contracts/libraries/LibQuote.sol#L169-L176):
```jsx
        (bool hasMadeProfit, uint256 pnl) = LibQuote.getValueOfQuoteForPartyA(
            closedPrice,
            filledAmount,
            quote
        );
        if (hasMadeProfit) {
            accountLayout.allocatedBalances[quote.partyA] += pnl;
            accountLayout.partyBAllocatedBalances[quote.partyB][quote.partyA] -= pnl;
        }
```
**NOTE**: Because I was not able to get any values for `upnlSig` the amounts that I used are arbitrary and may be different in real world example, **but the concept remains the same**. Suspended B is not locked from accepting trades from A so it is possible to "transfer" his funds away with the use of trades.

## Impact
Suspended partyB users are able to withdraw their funds thru trades.
## Code Snippet
[`fillCloseRequest()`](https://github.com/sherlock-audit/2023-06-symmetrical/blob/main/symmio-core/contracts/facets/PartyB/PartyBFacet.sol#L192)
## Tool used

Manual Review

## Recommendation
For the fix I suggest to include `notSuspended()` modifier in [`fillCloseRequest()`](https://github.com/sherlock-audit/2023-06-symmetrical/blob/main/symmio-core/contracts/facets/PartyB/PartyBFacet.sol#L192), to prevent B from filling out quote requests.



# [M-02] Inefficient allocation in `depositAndAllocateForPartyB()`: Low allocation amounts for PartyB


## Summary
The function `depositAndAllocateForPartyB()` is intended to deposit and allocate funds for PartyB but performs the allocation incorrectly. Due to missing conversion to 18 decimals before allocation, the resulting allocation amounts are significantly lower than expected.
## Vulnerability Detail
We can see the correct implementation, when depositing funds for partyA that the function `deposit` in 6 DEC (internally it converts it to 18) and converts to 18 DEC afterwards so it can `allocate()` in 18DEC
```jsx
    function depositAndAllocate(
        uint256 amount
    ) external whenNotAccountingPaused notLiquidatedPartyA(msg.sender) {
        AccountFacetImpl.deposit(msg.sender, amount);
        uint256 amountWith18Decimals = (amount * 1e18) /
        (10 ** IERC20Metadata(GlobalAppStorage.layout().collateral).decimals());
        AccountFacetImpl.allocate(amountWith18Decimals);
        emit Deposit(msg.sender, msg.sender, amount);
        emit AllocatePartyA(msg.sender, amountWith18Decimals);
    }
```
But the function `depositAndAllocateForPartyB()`, does not convert to 18 DEC before allocation, leading to low allocation amounts:
```jsx
    function depositAndAllocateForPartyB(
        uint256 amount,
        address partyA
    ) external whenNotPartyBActionsPaused onlyPartyB {
        AccountFacetImpl.depositForPartyB(amount);
        AccountFacetImpl.allocateForPartyB(amount, partyA, true);
        emit DepositForPartyB(msg.sender, amount);
        emit AllocateForPartyB(msg.sender, partyA, amount);
    }
```
Example:
 - partyB calls `depositAndAllocateForPartyB()` with 1000USDC
 - 1000USDC gets deposited and converted to 18DEC
 - 1000USDC gets allocated in 6 DEC

 The balance for this partyB is **1000e18**, but the allocated amount is only **1000e6**, which is inefficient to do anything
## Impact
Wrong function implementation, leading to low allocations for certain party.
## Code Snippet
[Account/AccountFacet.sol#L74-L82](https://github.com/sherlock-audit/2023-06-symmetrical/blob/main/symmio-core/contracts/facets/Account/AccountFacet.sol#L74-L82)
## Tool used

Manual Review

## Recommendation
Add the conversion to 18 DEC
```jsx
    function depositAndAllocateForPartyB(
        uint256 amount,
        address partyA
    ) external whenNotPartyBActionsPaused onlyPartyB {
        AccountFacetImpl.depositForPartyB(amount);
 -      AccountFacetImpl.allocateForPartyB(amount, partyA, true);
 +      uint256 amountWith18Decimals = (amount * 1e18) / 
 +      (10 ** IERC20Metadata(GlobalAppStorage.layout().collateral).decimals());
 +      AccountFacetImpl.allocateForPartyB(amountWith18Decimals, partyA, true);
        emit DepositForPartyB(msg.sender, amount);
        emit AllocateForPartyB(msg.sender, partyA, amount);
    }
```