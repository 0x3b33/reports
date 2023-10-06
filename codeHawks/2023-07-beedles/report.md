| Severity | Title | 
|:--:|:---|
| [H-01](#h-01-refinance-reduces-the-pools-balance-by-the-debt-twice) | `refinance` reduces the pool's balance by the debt twice |
| [H-02](#h-02-front-running-the-first-deposit-on-stake-yields-the-whole-weth-amount) | Front running the first deposit on stake yields the whole WETH amount |
| [H-03](#h-03-any-lender-can-drain-the-contract) | Any lender can drain the contract | 
| [H-04](#h-04-any-borrower-can-avoid-liquidation-by-doing-a-refinance-with-small-amounts-of-tokens) | Any borrower can avoid liquidation by doing a `refinance` with small amounts of tokens |
| [H-05](#h-05-due-to-sellprofits-being-public-people-can-reduce-the-profits-for-the-system) | Due to `sellProfits` being public, people can reduce the profits for the system |
| [H-06](#h-06-any-user-can-drain-the-contract-with-the-help-of-setpool) | Any User can drain the contract with the help of `setPool` |
| [H-07](#h-07-buyloan-can-be-called-on-another-users-pool-thus-messing-up-their-plans) | `buyLoan` can be called on another user's pool, thus messing up their plans |
| [M-01](#m-01-no-slippage-check-in-sellprofits-it-could-be-exploited-by-a-mev-bot) | No slippage check in `sellProfits`, it could be exploited by a MEV bot |
| [M-02](#m-02-user-can-drain-fees-due-to-them-using-the-uni-swap-router) | User can drain Fees due to them using the UNI swap router |
| [M-03](#m-03-lack-of-specific-time-input-can-result-in-mevs-exploiting-sellprofits) | Lack of specific time input can result in MEVs exploiting `sellProfits` |


## [H-01] `refinance` reduces the pool's balance by the debt twice

## Summary
[refinance](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Lender.sol#L591-L710) accidentally reduces the new pool balance **2** times instead of one, charging it 2x the debt balance.

## Vulnerability Details
`refinance` as the name suggest refinances a loan, this means it can change a few of it's parameters, like reducing or increasing debt or changing the pool it uses and so on. 

The current issue  with this function is that it accidentally reduces the balances of the new pool twice once [here](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Lender.sol#L636) and twice [here](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Lender.sol#L698). This means that it charges the pool of the lender twice instead of once thus lowering his potential profits, messing up the accounting and lowering the amount he can withdraw from the system.

This can be exploited accidentally when a normal user tries to refinance his loan or maliciously when someone want to cause harm on a given user.

Example: 
This is Alice's pool
| *Preconditions*       | *values*   |
|-----------------------|------------|
| Loan : Debt tokens    | USDC : DAI |
| Max borrow percentage | 60%        |
| Pool balance          | 1000 USDC  |

1.  Bob sees this pool, but he does not like Alice (for some reason), so he makes his own pool with the same parameter.
2.  Then Bob takes a loan of 300 USDC and provides 500 DAI.
3.  Bob calls [refinance](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Lender.sol#L591-L710) on his loan to move it from his pool to Alice's one.
4.  Now because of how refinance works Bob's pool is 100% sloven so he can withdraw everything. 


Lets see how the refinance TX went:
Firstly everything is [extracted](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Lender.sol#L593-L606) and then [checked](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Lender.sol#L608-L619), as expected the checks pass (because Bob created the exact same pool as Alice's one). Afterwards Bob [pays](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Lender.sol#L622-L626) small fees for his action and his pool [becomes solvent](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Lender.sol#L629-L633) again. Now Alice's pool gets [charged](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Lender.sol#L636-L637) the new debt

```jsx
            _updatePoolBalance(poolId, pools[poolId].poolBalance - debt);
            pools[poolId].outstandingLoans += debt;
```
After some [transfers](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Lender.sol#L639-L674) to make sure everything is up and right the loan is moved to the new pool where the new pool balances [are reduced](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Lender.sol#L698) again.
```jsx
pools[poolId].poolBalance -= debt;
```

5. Now because of the double debt removal, Alice's pool recorded balance (`pools[poolId].poolBalance`) is 400 USDC (600 less), and her collateral is only 500 DAI

## Tools Used
Manual review

## Recommendations
Deduct the debt from the pool balance only once, and remove the [second](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Lender.sol#L698C13-L698C38) reduction.
```jsx
            loans[loanId].collateral = collateral;
            loans[loanId].interestRate = pool.interestRate;
            loans[loanId].startTimestamp = block.timestamp;
            loans[loanId].auctionStartTimestamp = type(uint256).max;
            loans[loanId].auctionLength = pool.auctionLength;
            loans[loanId].lender = pool.lender;
-           pools[poolId].poolBalance -= debt; //@audit this is not needed
```


## [H-02] Front running the first deposit on stake yields the whole WETH amount


## Summary
By inserting a transaction between the deployment of [staking](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Staking.sol#L88) and depositing of the first amount of  WETH you can steal this whole amount.

## Vulnerability Details
Lets say Beedle has been active for some time and the owners have collected some fees. They decide to deploy staking and but 1 WETH inside of it as the initial deposit. After sending their TX for deployment and before the TX for sending WETH, an attacker can would be able to insert his own TX
in the middle of them to. He would just need to deposit any amount of TNK (it could be as little as 1 wei). adn when the WETH is send he can call [claim](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Staking.sol#L53-L58) and claim 100% of it.

Example
1. Owners deploy the contract and afterwards send 1 WETH to it, as staking incentives
2. Attacker sees that so he inserts his [deposit](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Staking.sol#L38-L42) TX of 1 wei between these 2 and boost the 3 TX with Flashbots (for he to be sure they will execute in this order) 

Now after the deposit [update](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Staking.sol#L61-L76) and [updateFor](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Staking.sol#L80-L94). Where on update the index will be 0, since the [2 balances](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Staking.sol#L65) will be equal.
```jsx
            uint256 _balance = WETH.balanceOf(address(this));//@audit this is 0
            if (_balance > balance) {// if (0 > ?) => false
``` 
Because index is 0 [updateFor](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Staking.sol#L92) will set the attacker index as 0 also.

3. After 1 WETH is send the attacker just calls [claim](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Staking.sol#L53-L58) where [update](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Staking.sol#L66-L73) will trigger and the global index will become 1e36.

```jsx
            uint256 _balance = WETH.balanceOf(address(this));
            if (_balance > balance) {
                uint256 _diff = _balance - balance;
                if (_diff > 0) {
                    //@audit 1e18 * 1e18 / 1 => 1e36
                    uint256 _ratio = _diff * 1e18 / totalSupply; 
                    if (_ratio > 0) {
                      index = index + _ratio;
                      balance = _balance;
                    }
                }
```
From there on his share [would increase](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Staking.sol#L88-L89) to 1e18, and claim will [send it](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Staking.sol#L55).

## Impact
Attacker would be able to steal 100% of the initial deposit with amount of TKN as low as 1 wei.

## Tools Used
Manual review and Echidna 2.0

## Recommendations
You can inprove on the math fro calculating and updating the rewards, or after deployment make sure there are a lot of staked users and then send the WETH. 


## [H-03] Any lender can drain the contract


## Summary
Any lender can drain the contract, or at least mess up the accounting with the the help of [seizeLoan](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Lender.sol#L548-L586) 

## Vulnerability Details
Due to [seizeLoan](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Lender.sol#L548-L586) doing 2 [transfers](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Lender.sol#L563-L568), before doing any updates to the loan, it is possible for a malicious lender to reenter and cause some harm to the protocol.

Pools can be made with any token (even malicious one) meaning sooner or later a ERC777 token (or any token that has before/after transfer hook) will be used. Any lender who has a pool with such token as collateral will be able to reenter `seizeLoan` dues to it's code not following "checks-effects-interactions" pattern. 

```jsx
            IERC20(loan.collateralToken).transfer(feeReceiver, govFee);
            // transfer the collateral tokens from the contract to the lender
            IERC20(loan.collateralToken).transfer(
                loan.lender,
                loan.collateral - govFee
            );
            ...
            pools[poolId].outstandingLoans -= loan.debt;
            ...
            delete loans[loanId];
        }
```

From the code snippet above we can see that a transfer is made before any changes on the loan is done. So a malicious lender needs to call this function again with the same parameters and score a double payout, or even call it many times in a row for increased profits.

Example scenario:



| *Preconditions*                 | *values*       |
|---------------------------------|----------------|
| Loan tokenA : tokenB Collateral | ERC20 : ERC777 |
| Pool size                       | 10 000         |
| max LTV                         | 50%            |
| Borrower count                  | 8              |
| tokenA left                     | 2 000          |
| tokenB in                       | 16 000         |
| Borrow amount to be seized      | 1 000          |


Now one of the borrowers is insolvent so the pool owner (must be a contract) [starts an auction](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Lender.sol#L437-L459) and afterwards calls [seizeLoan](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Lender.sol#L548-L586). Now [this](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Lender.sol#L563-L568) transfer will trigger his [tokensReceived](https://github.com/0xjac/ERC777/blob/devel/contracts/ERC777TokensRecipient.sol) function in his smart contract, from where `tokensReceived` will call multiple times `seizeLoan`, gaining 1000 collateral tokens with every call, and because the loan is [modified](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Lender.sol#L575-L584) after the transfer he would be able to successfully extract as much collateral tokens from the contract as he wants.

## Impact
Lender would be able to drain the contract

## Tools Used
Manual review

## Recommendations
Save the amounts, then delete/update the loan and then transfer.


## [H-04] Any borrower can avoid liquidation by doing a `refinance` with small amounts of tokens


## Summary
Any borrower that has a loan due to be seized can call refinance with small amount and move his loan to a different pool, stopping the liquidation/seizing.

## Vulnerability Details
When an auction is started on any loan it's [start time](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Lender.sol#L448) is set and from this moment on-wards the pool owner who started the auction can call [seizeLoan](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Lender.sol#L548-L586) to seize the loan and claim it's collateral. However any borrower can set up a fron-runner bot and when he sees that his loan is gonna be seized he can call [refinance](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Lender.sol#L591-L710) with small amount of collateral/loan tokens and move his loan to another pool, and because with `refinance` the loan auction start time [is reset](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Lender.sol#L692) it's pending call to [seizeLoan](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Lender.sol#L548-L586) will also fail.
```jsx
            loans[loanId].auctionStartTimestamp = type(uint256).max;
```
This is all possible mainly because under `refinance` the borrower is not expected to pay any part for his loan, as long as he [matches](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Lender.sol#L618) the next pool **loan to collateral ratio** he is free to move.
```jsx
uint256 loanRatio = (debt * 10 ** 18) / collateral;
```

## Impact
Borrower can DoS [seizeLoan](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Lender.sol#L548-L586) making his function not sizable.

## Tools Used
Manual review

## Recommendations
Place a if statement that prohibits a borrower to call `refinance`.
```jsx
if(loans[loanId].auctionStartTimestamp != type(uint256).max) revert BorrowerUnderLiquidation();
``` 
Or make him pay some part of the loan in order to be eligible to refinance.


## [H-05] Due to `sellProfits` being public people can reduce the profits for the system 


## Summary
Due to [sellProfits](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Fees.sol#L26-L44) being public people can wait and call it in bad times to reduce the profits for the system.

## Vulnerability Details
Example scenario is when [Lender](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Lender.sol) contract generates 5 WETH in profits and current price of WETH 2000$ two thing can occur:
- Attacker waits until the price raises (for example 2100$) and then calls [sellProfits](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Fees.sol#L26-L44).
- Because it is public an attacker, may sandwich this TX with a flash loan trade.

---

To sandwich it with your smart contract (or a bot) you will need to execute a single transaction where:

**1** Take a Flash Loan.

**2** Buy WETH from the uni pool that [Fees](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Fees.sol) use.

**3** Call [sellProfits](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Fees.sol#L26-L44)

**4** Sell all of the WETH that you have bought.

**5** Repay the Flash Loan.

**6** Profit

---

## Impact
Reduced profits for the system.

## Tools Used
Manual review

## Recommendations
Put an access modifier on the function, so only the owner can call it.


## [H-06] Any User can drain the contract with the help of `setPool` 


## Summary
Because [setPool](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Lender.sol#L130-L176) does a transfer before configuring the pool it is possible for anyone to re-enter and drain the contract.

## Vulnerability Details
From the beetle we can see that Lenders make pool that borrowers borrow from, but all tokens are stored in one shared contract - [Lender](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Lender.sol). Now if any of the lenders makes a pool with an ERC777 token (or any other token with a hook) other Users will be able to drain this pool and steal all of the tokens. This is all possible because  [setPool](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Lender.sol#L130-L176) does [this transfer](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Lender.sol#L157-L163) before [setting](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Lender.sol#L175) the storage of a given pool.
```jsx
       } else if (p.poolBalance < currentBalance) {
            // if new balance < current balance then transfer the difference back to the lender
            IERC20(p.loanToken).transfer(
                p.lender,
                currentBalance - p.poolBalance
            );
        }
        ... //some events are emitted here
        pools[poolId] = p;
```

How to re-enter:
There are 2 pools **A : B** and **C : D** where one of the two tokens is re-enterable.

| *Preconditions*                                 | *values*       |
|-------------------------------------------------|----------------|
| Loan tokenA : tokenB Collateral                 | ERC777 : ERC20 |
| Pool size                                       | 10 000         |
| TokenA token left in the pool(ERC777 remaining) | 5000           |
| Loan tokenC : tokenD Collateral                 | ERC20 : ERC777 |
| Pool size                                       | 4000           |
| TokenD token left in the pool(ERC777 remaining) | 3000           |
| **Total ERC777 in contract**                    | **8000**       |

**Steal from A : B**

1. Create a pool A : B with 1000 of tokenA
2. Call [setPool](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Lender.sol#L130-L176) on that pool, only changing the loan amount from 1000 to 500
3. [This](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Lender.sol#L157-L163) transfer will trigger and transfer 500 tokens to your contract from where you will re-enter 5000 / 500 = 10 times
4. Withdraw your initial 1000 loan tokens 

**Steal from C : D**

1. Make a pool with D as loan token (the re-enter "hack" only steals loan tokens) with again 1000 tokenD
2. Call [setPool](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Lender.sol#L130-L176) on that pool, only changing the loan amount from 1000 to 500
3. [This](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Lender.sol#L157-L163) transfer will trigger and transfer 500 tokens to your contract from where you will re-enter 3000 / 500 = 6 times
4. Withdraw your initial 1000 loan tokens 

## Impact
Any user can drain every ERC777 token from the contract.
 
## Tools Used
Manual review.

## Recommendations
Either prohibit ERC777 tokens, by having whitelist on allowed tokens, or use [ReentrancyGuard](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/security/ReentrancyGuard.sol) by OZ.


## [H-07] `buyLoan` can be called on another user's pool, thus messing up their plans 


## Summary
[buyLoan](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Lender.sol#L465-L534) can be called by any user, even ones **not owning a pool** to move a random loan to a different pool.

## Vulnerability Details
Although  [comments](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Lender.sol#L462C1-L462C1) suggests that this function should only interact with your pool, there is no reinforcement on it. And because of that anyone can DoS any pool owner by simply moving any of the loans to his pool.

Example:

- Pool owner is planing to leave the system, having 4k in loan tokens and last borrower of his spool has just payed 1k (5k in total) 
- Malicious user calls [buyLoan](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Lender.sol#L465-L534) on any random loan (that matches collateral and loan tokens) to move it to his pool and eat up his loan tokens.
- Now pool owner is not being able to withdraw his loan tokens and needs to wait until this borrower pays his debt

## Impact
Anyone can mess up the poo owner intentions and Dos them of their loan/collateral tokens.

## Tools Used
Manual review 

## Recommendations
Have an `if()` to prohibit using this function with any pool that is not yours.
```jsx
    function buyLoan(uint256 loanId, bytes32 poolId) public {

        Loan memory loan = loans[loanId];
+       if (msg.sender != loan.lender) revert Unauthorized();
        ...
```


## [M-01] No slippage check in `sellProfits`, it could be exploited by a MEV bot


## Summary
The code sets `amountOutMinimum` to 0 in `ExactInputSingleParams`, potentially enabling exploitation by MEVs. 

## Vulnerability Details
From code below, the variable [amountOutMinimum](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Fees.sol#L38) is assigned a value of 0. In the context of [ExactInputSingleParams](https://docs.uniswap.org/contracts/v3/guides/swaps/single-swaps#call-the-function), it is understood that [amountOutMinimum](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Fees.sol#L38) represents the minimum expected output. If the actual output falls below this specified minimum, the UNI contract will revert the transaction. However, setting [amountOutMinimum](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Fees.sol#L38) to 0 can potentially lead to exploitation, as it allows for significant slippage in the token's value before executing the transaction. Consequently, transactions like these become attractive targets for MEVs.

```jsx
        ISwapRouter.ExactInputSingleParams memory params = ISwapRouter
            .ExactInputSingleParams({
                tokenIn: _profits,
                tokenOut: WETH,
                fee: 3000,
                recipient: address(this),
                deadline: block.timestamp,
                amountIn: amount,
                amountOutMinimum: 0,
                sqrtPriceLimitX96: 0
            });
```
## Impact
TX could be gamed for max slippage extraction.

## Tools Used

Manual Review 

## Recommendations
```jsx
-   function sellProfits(address _profits) public {
+   function sellProfits(address _profits,uint minOut) public {
         ...
        ISwapRouter.ExactInputSingleParams memory params = ISwapRouter
            .ExactInputSingleParams({
                tokenIn: _profits,
                tokenOut: WETH,
                fee: 3000,
                recipient: address(this),
                deadline: block.timestamp,
                amountIn: amount,
-               amountOutMinimum: 0,
+               amountOutMinimum: minOut,
                sqrtPriceLimitX96: 0
            });
            ...

```


## [M-02] User can drain [Fees](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Fees.sol) due to them using the UNI swap router


## Summary
This is not about slippage or MEV, but more on tokens and their pools. Because unknown number of tokens are gonna pass-through [Fees](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Fees.sol), some of them, small or big might not have the correct pool created. Thus users will be able to make the uni pools on their own terms and trade on them.

## Vulnerability Details
Because there is no restriction on which tokens can be used with [Lender](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Lender.sol) there is no restriction which tokens pass through [Fees](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Fees.sol), as some tokens might not have the 0.3% fee poll that Fees [is requesting](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Fees.sol#L34). For the tokens that don't have 0.3% fee pools, users will be able to in one transaction, make the pool and trade on it. Unfortunately owner of the pool, in it's creation, decides the ratios of token0 to token1, the creator of the pool can put tokenA (10 000 USD) to weth (2000 USD) for 1 : 1 ratio, thus trading 1 weth for 1 of tokenA,  although tokenA is 5x more expensive. 

**Example:**

| *Preconditions*                                                                                                                                                                    |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| [StakeWise](https://defillama.com/protocol/stakewise) **SWISE** [does not](https://info.uniswap.org/#/tokens/0x48c3399719b582dd63eb5aadf12a40b4c3f52fa2) have a pool with 0.3% fee |
| [Fees](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Fees.sol) has 500 000 **SWISE**                                                                                      |
| **SWISE** price - 0.09 USD                                                                                                                                                         |
| **WETH**  price - 1875 USD                                                                                                                                                         |
| **WETH : SWISE** is 1 : 20833                                                                                                                                                      |
 

 Attacker makes a UNIv3 pool with SWICE : WETH with ratio 208 000 SWISE and 1 WETH (10x the CEX price). Then he makes the trade to this pool and withdraws all of it's liquidity. This way the attacker bough SWISE for 10x cheaper. And since everything could be packed in one TX he would be able to flash loan all of it.
 
## Impact
User would be able to steal small cap or new tokens out of fees.

## Tools Used
Manual review

## Recommendations
To give advice for such a thing is complicated, because I am not sure how the founders want everything to work. My suggestion is to use a whitelist for the allowed tokens.


## [M-03] Lack of specific time input can result in MEVs exploiting `sellProfits`


## Summary
Lack of time specific input can result in MEVs exploiting the [`sellProfits`](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Fees.sol#L26-L44) TX. 

## Vulnerability Details
From the code snippet bellow we can see that for the UNI call the [deadline](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Fees.sol#L36) parameter is set to `block.timestamp`. Deadline is where is the last time that the TX can be executed, and any time after it it revert. The issue here is that `block.timestamp` can be easily manipulated by the MEVs (they can include it in this block,top or bottom, or even pay the block proposers to put it in the next block).  This all means that the deadline is useless, since `block.timestamp` is when the TX is executed and no matter when it's executed it's gonna pass (now, a day later, or even a year).
```jsx
        ISwapRouter.ExactInputSingleParams memory params = ISwapRouter
            .ExactInputSingleParams({
                tokenIn: _profits,
                tokenOut: WETH,
                fee: 3000,
                recipient: address(this),
                deadline: block.timestamp,// <== this is useless
                amountIn: amount,
                amountOutMinimum: 0,
                sqrtPriceLimitX96: 0
            });
```
## Impact
MEVs, could game the [`sellProfits`](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Fees.sol#L26-L44) TX. 
## Tools Used

## Recommendations
Set a deadline settle in the function.
```jsx
function sellProfits(address _profits,uint _deadline) public {
        ...
        ISwapRouter.ExactInputSingleParams memory params = ISwapRouter
            .ExactInputSingleParams({
                tokenIn: _profits,
                tokenOut: WETH,
                fee: 3000,
                recipient: address(this),
                deadline: _deadline,
                amountIn: amount,
                amountOutMinimum: 0,
                sqrtPriceLimitX96: 0
            });
            ...
```