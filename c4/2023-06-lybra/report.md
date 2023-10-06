
| Severity | Title | 
|:--:|:---|
| [H-01](#h-01-all-the-functions-in-lybraconfigurator-can-be-called-by-malicious-accounts) | All the functions in LybraConfigurator.sol can be called by malicious accounts |
| [M-01](#m-01-the-governance-cant-function-correctly) | The governance can't function correctly |
| [M-02](#m-02-rewards-could-be-stolen-on-users-who-have-withdrawn-most-of-their-lp-but-not-replayed-their-borrow) | Rewards could be stolen on users who have withdrawn most of their LP but not replayed their borrow |
| [M-03](#m-03-if-statement-makes-rewards-malfunction) | `if()` statement makes rewards malfunction  |
| [M-04](#m-04-volatile-prices-can-cause-issue-when-users-try-to-do-rigidredemption) | Volatile prices can cause issue when users try to do `rigidRedemption` |
| [M-05](#m-05-the-function-getexchangeratio-does-not-exist)  | The function `getExchangeRatio` does not exist  |
| [M-06](#m-06-wrong-logic-in-_repay-causes-users-to-be-charged-lower-fees) | Wrong logic in `_repay` causes users to be charged lower fees |



## [H-01] All the functions in LybraConfigurator can be called by malicious accounts

## Impact
All the functions in LybraConfigurator.sol can be called by malicious accounts. This breaks the whole protocol, as anyone can change the tokens, the fees and the ratios, the borrow apy or mint unlimited amount of tokens, etc..

## Proof of concept
A malicious user calls a **LybraConfigurator** function that is supposed to be executable only by the **DAO** or the **TIMELOCK** . The following modifiers should check if the caller has the needed rights. Both `GovernanceTimelock.checkOnlyRole()`  and `GovernanceTimelock.checkRole() return booleans indicating if a user has the needed role. However, the returned value is not checked, allowing anyone to execute the certain function.

```jsx
 modifier onlyRole(bytes32 role) {
        GovernanceTimelock.checkOnlyRole(role, msg.sender);
        _;
}

    modifier checkRole(bytes32 role) {
        GovernanceTimelock.checkRole(role, msg.sender);
        _;
    }
```
## Tools Used
Manual review

## Recommended Mitigation Steps
In both the modifiers, add a require statement that checks the return value of the GovernanceTimelock function. 
```jsx
modifier onlyRole(bytes32 role) {
        require(GovernanceTimelock.checkOnlyRole(role, msg.sender), "NO_ROLE");
        _;
}


modifier checkRole(bytes32 role) {
    require(GovernanceTimelock.checkRole(role, msg.sender), "NO_ROLE");
    _;
}
```
## [M-01] The governance can't function correctly 

## Impact
Due to inappropriate short `votingPeriod`  and `votingDelay`, it is near impossible for the governance to function correctly.

## Proof of Concept
When making proposals with the [`Governor`](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/governance/Governor.sol#L299-L308) contract OZ uses `votingPeriod`
```jsx
        uint256 snapshot = currentTimepoint + votingDelay();
        uint256 duration = votingPeriod();

        _proposals[proposalId] = ProposalCore({
            proposer: proposer,
            voteStart: SafeCast.toUint48(snapshot),//@audit votingDelay() for when the voting starts
            voteDuration: SafeCast.toUint32(duration),//@audit votingPeriod() for the duration
            executed: false,
            canceled: false
        });
``` 
But currently Lybra has implemented wrong amounts for bolt [`votingPeriod`](https://github.com/code-423n4/2023-06-lybra/blob/main/contracts/lybra/governance/LybraGovernance.sol#L143-L145) and [`votingDelay`](https://github.com/code-423n4/2023-06-lybra/blob/main/contracts/lybra/governance/LybraGovernance.sol#L147-L149), which means proposals from the governance will be near impossible to be voted on.
```jsx
    function votingPeriod() public pure override returns (uint256){
         return 3;//@audit this should be time in seconds 
    }

     function votingDelay() public pure override returns (uint256){
         return 1;//@audit this should be time in seconds 
    }
```
## Tools Used
Manual Review

## Recommended Mitigation Steps
You can implement it as OZ suggest's i their [examples](https://docs.openzeppelin.com/contracts/4.x/governance)
```jsx
    function votingDelay() public pure override returns (uint256) {
        return 7200; // 1 day
    }

    function votingPeriod() public pure override returns (uint256) {
        return 50400; // 1 week
    }
```

## [M-02] Rewards could be stolen on users who have withdrawn most of their LP but not replayed their borrow


## Impact
Rewards could be stolen on users who have withdrawn most of their LP from the UNIv2 ETH/LBR pool, but not replayed their borrow.

## Proof of Concept
Because [`EUSDMiningIncentives`](https://github.com/code-423n4/2023-06-lybra/blob/main/contracts/lybra/miner/EUSDMiningIncentives.sol) has a function to purchase rewards of users with small amount of LP it is possible to do the same for users with large amounts too, if they are not caucus with their funds. The exploit trigers when users with many rewards first unstake the LP and then repay the borrow amount. Between the 2 transaction MEV could be squished to extract his rewards. This exploit is possible dues to how [`isOtherEarningsClaimable`](https://github.com/code-423n4/2023-06-lybra/blob/main/contracts/lybra/miner/EUSDMiningIncentives.sol#L188-L190) works
```jsx
//if all of the LP is withdrawn the calculation comes to "0 / borrow_amount < 500" => true
    function isOtherEarningsClaimable(address user) public view returns (bool) {
// (user_LP_in_pools * 10 000) / borrow_amount_from_vaults < 500
        return (stakedLBRLpValue(user) * 10000) / stakedOf(user) < 500;
    }
```
[`stakedLBRLpValue`](https://github.com/code-423n4/2023-06-lybra/blob/main/contracts/lybra/miner/EUSDMiningIncentives.sol#L149-L157) is the value staked in the UNIv2 ETH/LBR, represented in `ethlbrLpToken`
[`stakedOf`](https://github.com/code-423n4/2023-06-lybra/blob/main/contracts/lybra/miner/EUSDMiningIncentives.sol#L136-L147) is the borrow amount from any of the vaults ([RETH](https://github.com/code-423n4/2023-06-lybra/blob/main/contracts/lybra/pools/LybraRETHVault.sol) /  [stETH](https://github.com/code-423n4/2023-06-lybra/blob/main/contracts/lybra/pools/LybraWstETHVault.sol) ...)
Example:
- UserA has 1000e18 `ethlbrLpToken`, combined dept of 200e18 and some unclaimed rewards of 100e18 EUSD
- UserA unstakes his LP from the UNI ETH/LBR pool and sends another transaction to claim his rewards and repay
- UserB sees the second transaction and quickly front-runs userA with `purchaseOtherEarnings`, where `isOtherEarningsClaimable` is triggered

```jsx
    function isOtherEarningsClaimable(address user) public view returns (bool) {
        // (0 * 10000) / 200e18 = 0 ===> 0 < 500 -true
        return (stakedLBRLpValue(user) * 10000) / stakedOf(user) < 500;
    }
```
- UserB pays only 1/3 of the fees generated by UserA due to this [calculation]() and claims all of the rewards
## Tools Used
Manual Review

## Recommended Mitigation Steps
You can remove or refactor this function, but dues to it's complexity I am not sure how it should be refactored.


## [M-03] `if()` statement makes rewards malfunction 


## Impact
If `ProtocolRewardsPool` is insufficient in EUSD, but has enough PeUSD to give rewards it still reverts, due to wrong `if()` statement, thus it is  unable to send the rewards to users.

## Proof of Concept
Users have just emptied [`ProtocolRewardsPool`](https://github.com/code-423n4/2023-06-lybra/blob/main/contracts/lybra/miner/ProtocolRewardsPool.sol) out of EUSD, by claiming rewards with  `getReward()`. Now the protocol has a new distribution of PeUSD tokens, with `LybraConfigurator.distributeRewards()`, but when users try to claim their rewards [`getReward()`](https://github.com/code-423n4/2023-06-lybra/blob/main/contracts/lybra/miner/ProtocolRewardsPool.sol#L190-L218) reverts because of this:
```jsx
   function getReward() external updateReward(msg.sender) {
        uint reward = rewards[msg.sender];
        if (reward > 0) {
            rewards[msg.sender] = 0;
            IEUSD EUSD = IEUSD(configurator.getEUSDAddress());//get the address
            uint256 balance = EUSD.sharesOf(address(this));//get the balance == 
//@aduit here eUSDShare = balance >= reward-false => reward - balance => rewards - 0 | eUSDShare = reward
            uint256 eUSDShare = balance >= reward ? reward : reward - balance;
//here it tries to send the rewards amount, but it reverts since it has not tokens 
            EUSD.transferShares(msg.sender, eUSDShare);

```
Because of the constant revert users are not able to claim their rewards and need to wait for EUSD distribution. The other bad thing is that the PeUSD is uncalimable to most extent.Again because of the line bellow, if:

 - Protocol has 40e18 EUSD and 100e18 PeUSD.
 - UserA tries to claim his rewards, that are 100e18 in rewards tokens.
 ```jsx
//eUSDShare = balance >= reward-false => reward - balance => 100e18 - 40e18 => eUSDShare = 60e18 
uint256 eUSDShare = balance >= reward ? reward : reward - balance;
//again reverts, because contract has 40, whily trying to send 60
EUSD.transferShares(msg.sender, eUSDShare);
```
Now PeUSD is un-claimable and remains in the contract.
### Foundry PoC
```jsx
    function test_no_EUSD() public {
        //make 2 random users
        deal(address(lbr), user1, 1000e18);
        deal(address(lbr), user2, 4000e18);

        //stake for bolt of them
        vm.prank(user1);
        rewardsPool.stake(1000e18); 

        vm.prank(user2);
        rewardsPool.stake(4000e18);   

        //get some PeUSD in the config and call distributeRewards() to send it to the pool
        //@notice here we don't send any EUSD => rewardsPool has 0 EUSD
        deal(address(PeUSD),address(configurator),1e21);
        configurator.distributeRewards();

        //to make sure the balance is sent
        PeUSD.balanceOf(address(rewardsPool));

        //user rewards is actually 2e17 per 1e18 => 2e20 total for user1
        vm.prank(user1);
        //but here reverts, because it is unable to send any EUSD
        rewardsPool.getReward();
        console.log(rewardsPool.earned(user1));
        console.log("pEUSD user1: ", PeUSD.balanceOf(user1));
        console.log("pEUSD pool : ", PeUSD.balanceOf(address(rewardsPool)));
        console.log();
    }
```
## Tools Used
Manual Review

## Recommended Mitigation Steps
update the [`if()`](https://github.com/code-423n4/2023-06-lybra/blob/main/contracts/lybra/miner/ProtocolRewardsPool.sol) as:
```jsx
-  uint256 eUSDShare = balance >= reward ? reward : reward - balance;
+  uint256 eUSDShare = balance >= reward ? reward : balance;
```

## [M-04] Volatile prices can cause issue when users try to do `rigidRedemption`


## Impact
Volatile prices can cause issue when users try to do [`rigidRedemption`](https://github.com/code-423n4/2023-06-lybra/blob/main/contracts/lybra/pools/base/LybraPeUSDVaultBase.sol#L157-L168)

## Proof of Concept
Volatile prices can cause slippage loss, when users use `rigidRedemption()`. This function takes PeUSD (stable coin) amount and converts it to WstETH/stETH (variable price). Unfortunately `rigidRedemption()` does not include `timestamp` or `minAmount` received , meaning this trade can be executed later in time and at a different price than user previously expected.

**Example:**
- provider has 100 **wstETH** and **wstETH** price is 2000$
- user wants to buy 10 **wstETH** and has 20 000 in PeUSD, so he calls [`rigidRedemption`](https://github.com/code-423n4/2023-06-lybra/blob/main/contracts/lybra/pools/base/LybraPeUSDVaultBase.sol#L157-L168)
- Now due to congestion on **ETH**, and volatile prices the transaction could remain stuck in the mempool for a long time
- Finally the transaction gets executed, but now **wstETH **price is 2100$, not the original 2000$, so the user receives 9,52 **wstETH** instead of 10 (not counting fees)!
## Tools Used
Manual Review

## Recommended Mitigation Steps
Because of this scenario and other like it, it is recommended to use some sort of slippage protection when users execute trades.
```jsx
    function rigidRedemption(address provider, uint256 eusdAmount,uint minAmountReceived) external virtual {
        require(configurator.isRedemptionProvider(provider), "provider is not a RedemptionProvider");
        require(borrowed[provider] >= eusdAmount, "eusdAmount cannot surpass providers debt");
        uint256 assetPrice = getAssetPrice();
        uint256 providerCollateralRatio = (depositedAsset[provider] * assetPrice * 100) / borrowed[provider];
        require(providerCollateralRatio >= 100 * 1e18, "provider's collateral ratio should more than 100%");
        _repay(msg.sender, provider, eusdAmount);
        uint256 collateralAmount = (((eusdAmount * 1e18) / assetPrice) * (10000 - configurator.redemptionFee())) / 10000;
        depositedAsset[provider] -= collateralAmount;
        totalDepositedAsset -= collateralAmount;
+       require(minAmountReceived <= collateralAmount);
        collateralAsset.transfer(msg.sender, collateralAmount);
        emit RigidRedemption(msg.sender, provider, eusdAmount, collateralAmount, block.timestamp);
    }
```


## [M-05] The function `getExchangeRatio` does not exist 


## Impact
Due to the use of wrong function in `getAssetPrice` -> `getExchangeRatio` (which does not exist) the `LybraRETHVault.getAssetPrice` will revert, causing the whole contract to malfunction.

## Proof of Concept
In [LybraRETHVault](https://github.com/code-423n4/2023-06-lybra/blob/main/contracts/lybra/pools/LybraRETHVault.sol) when users deposit eth and try to mint PeUSD, they call[ depositEtherToMint](https://github.com/code-423n4/2023-06-lybra/blob/main/contracts/lybra/pools/LybraRETHVault.sol#L27-L39). This function calculates their deposit amount and if they want to mint PeUSD it calls [_mintPeUSD](https://github.com/code-423n4/2023-06-lybra/blob/main/contracts/lybra/pools/LybraRETHVault.sol#L35) with  amount and price. 

Now the issue occurs when we call [`getAssetPrice`](https://github.com/code-423n4/2023-06-lybra/blob/main/contracts/lybra/pools/LybraRETHVault.sol#L46-L48) This function gets the ether price, RETH to ETH conversion ration and calculated the RETH price in USD. This implementation is faulty tho, because RETH does not have function  called [getExchangeRatio](https://github.com/code-423n4/2023-06-lybra/blob/main/contracts/lybra/pools/LybraRETHVault.sol#L47), they have [getExchangeRate](https://docs.rocketpool.net/developers/api/js/RETH.html#getexchangerate). This means that  `getAssetPrice` will **call the fallback on RETH**, if they have it, **or revert all together**.
```jsx
    function getAssetPrice() public override returns (uint256) {
     //@audit RETH does not have getExchangeRatio, they have getExchangeRate
         return (_etherPrice() * IRETH(address(collateralAsset)).getExchangeRatio()) / 1e18;
    }
 ```
 One of the developers confirmed in discord that this is a misspelling of the original function, when asked about it. [PIC](https://imgur.com/a/CNBKcck)

> "The correct interface is getExchangeRate(). This was a mistake."

Same interface issue occurs also in [LybraWbETHVault](https://github.com/code-423n4/2023-06-lybra/blob/main/contracts/lybra/pools/LybraWbETHVault.sol#L10), where again **IWBETH** uses [exchangeRatio](), but the WBET contract has [exchangeRate](https://etherscan.io/token/0xa2E3356610840701BDf5611a53974510Ae27E2e1#readProxyContract). Also making `LybraWbETHVault` not usable, since [getAssetPrice](https://github.com/code-423n4/2023-06-lybra/blob/main/contracts/lybra/pools/LybraWbETHVault.sol#L34-L36) will revert again on calling.

## Tools Used
Manual review Foundry and HH
## Recommended Mitigation Steps
Change the [interface](https://github.com/code-423n4/2023-06-lybra/blob/main/contracts/lybra/pools/LybraRETHVault.sol#L9-L11):
```jsx
interface IRETH {
    function getExchangeRate() external view returns (uint256);
}
```
And [getAssetPrice](https://github.com/code-423n4/2023-06-lybra/blob/main/contracts/lybra/pools/LybraRETHVault.sol#L46-L48):
```jsx
    function getAssetPrice() public override returns (uint256) {
        return (_etherPrice() * IRETH(address(collateralAsset)).getExchangeRate()) / 1e18;
    }
```

## [M-06] Wrong logic in `_repay` causes users to be charged lower fees


## Impact
Wrong logic in [`LybraPeUSDVaultBase._repay()`](https://github.com/code-423n4/2023-06-lybra/blob/main/contracts/lybra/pools/base/LybraPeUSDVaultBase.sol) causes users to be charged lower/no fees.

## Proof of Concept
The logic in [`_repay`](https://github.com/code-423n4/2023-06-lybra/blob/main/contracts/lybra/pools/base/LybraPeUSDVaultBase.sol#L192-L210) in implemented wrongly, more precisely, the `totalFee` and `borrowed[_onBehalfOf]` calculation are bolt lowered from the amount, even tho some parts of the amount provided should go as fees and the rest to deduct `borrowed[_onBehalfOf]` . 

```jsx
    function _repay(address _provider, address _onBehalfOf, uint256 _amount) internal virtual {
        try configurator.refreshMintReward(_onBehalfOf) {} catch {}
        _updateFee(_onBehalfOf);
        uint256 totalFee = feeStored[_onBehalfOf];
        uint256 amount = borrowed[_onBehalfOf] + totalFee >= _amount ? _amount : borrowed[_onBehalfOf] + totalFee;
        if(amount >= totalFee) {
            feeStored[_onBehalfOf] = 0;
            PeUSD.transferFrom(_provider, address(configurator), totalFee);
            PeUSD.burn(_provider, amount - totalFee);
        } else {
//@aduit here we deduct from the fee 
            feeStored[_onBehalfOf] = totalFee - amount;
            PeUSD.transferFrom(_provider, address(configurator), amount);     
        }
        try configurator.distributeRewards() {} catch {}
//@audit and here we deduct this from the borrow balance 
        borrowed[_onBehalfOf] -= amount;
        poolTotalPeUSDCirculation -= amount;

        emit Burn(_provider, _onBehalfOf, amount, block.timestamp);
    }
```
Here is a more clearer explanation + an example:

- User has  borrowed 100 **EUSD** =>`borrowed[_onBehalfOf] = 100e18` and a total fee generated of 20 **EUSD** `feeStored[_onBehalfOf] = 20e18`
- He calls `burn()` which in turns calls `_repay()` 
```jsx
    function _repay(address _provider, address _onBehalfOf, uint256 _amount) internal virtual {
        try configurator.refreshMintReward(_onBehalfOf) {} catch {}
        _updateFee(_onBehalfOf);
      // we get the fee == 20e18
        uint256 totalFee = feeStored[_onBehalfOf];
        //  amount = _amount = 100e18
        uint256 amount = borrowed[_onBehalfOf] + totalFee >= _amount ? _amount : borrowed[_onBehalfOf] + totalFee;
        if(amount >= totalFee) {//true
            feeStored[_onBehalfOf] = 0;// we reset the fee 
            PeUSD.transferFrom(_provider, address(configurator), totalFee);// we send it away
            PeUSD.burn(_provider, amount - totalFee);//and burn the rest
        } 
        try configurator.distributeRewards() {} catch {}
        //but here the borrowed[_onBehalfOf] is lowered by the `amount`, not by `amount - totalFee`
        borrowed[_onBehalfOf] -= amount;
        poolTotalPeUSDCirculation -= amount;
```
- Because of this line `borrowed[_onBehalfOf] -= amount` the user borrowed = 100 and paid fees on it, repaying 100 will get his borrow balance to 0, without accounting the fees that he should have payed!

## Tools Used
Manual Review
## Recommended Mitigation Steps
Example how [`_repay`](https://github.com/code-423n4/2023-06-lybra/blob/main/contracts/lybra/pools/base/LybraPeUSDVaultBase.sol#L192-L210) can be fixed:
```jsx
        if(amount >= totalFee) {
            feeStored[_onBehalfOf] = 0;
            PeUSD.transferFrom(_provider, address(configurator), totalFee);
            PeUSD.burn(_provider, amount - totalFee);
+           try configurator.distributeRewards() {} catch {}
+           borrowed[_onBehalfOf] -= (amount - totalFee);
        } else {
            feeStored[_onBehalfOf] = totalFee - amount;
            PeUSD.transferFrom(_provider, address(configurator), amount);
        }
-       try configurator.distributeRewards() {} catch {} 
-       borrowed[_onBehalfOf] -= amount;
        poolTotalPeUSDCirculation -= amount;
```