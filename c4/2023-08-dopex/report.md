
| Severity | Title | 
|:--:|:---|
| [H-01](#h-01-calculatebondcostroundup-messes-up-the-price-causing-the-bond-to-be-more-expensive) | `calculateBondCost::roundUp` messes up the price, causing the bond to be more expensive |
| [H-02](#h-02-perpetualatlanticvaultlp-subtractloss-can-be-dos-by-donation-attack) | **PerpetualAtlanticVaultLP** `subtractLoss` can be DoS by donation attack |
| [M-01](#m-01-relp-under-relpcontract-does-not-return-excess-which-may-lead-to-small-amounts-being-left-stuck-in-the-contract) | `reLP` under **ReLPContract** does not return excess, which may lead to small amounts being left stuck in the contract |
| [M-02](#m-02-addliquidity-under-relpcontract-lacks-a-slippage-check-which-leads-to-losses) | `addLiquidity` under **ReLPContract** lacks a slippage check, which leads to losses |



# [H-01] `calculateBondCost::roundUp` messes up the price, causing the bond to be more expensive

## Impact
[roundUp](https://github.com/code-423n4/2023-08-dopex/blob/main/contracts/perp-vault/PerpetualAtlanticVault.sol#L576-L583) rounds the price up with precision of 1e6, unfortunately the RDPX price also has small decimal number and the rounding actually inflates the price. 

## Proof of Concept
1. RDPX's [price](https://github.com/code-423n4/2023-08-dopex/blob/main/contracts/core/RdpxV2Core.sol#L1160) is not in USD, but in [WETH](https://github.com/code-423n4/2023-08-dopex/blob/main/contracts/core/RdpxV2Core.sol#L1240-L1241) 

2. The oracle returns in [with 8 decimals](https://github.com/code-423n4/2023-08-dopex/blob/main/contracts/core/RdpxV2Core.sol#L1235)

3. [roundUp](https://github.com/code-423n4/2023-08-dopex/blob/main/contracts/perp-vault/PerpetualAtlanticVault.sol#L576-L583) rounds up with [6 decimals of precision](https://github.com/code-423n4/2023-08-dopex/blob/main/contracts/perp-vault/PerpetualAtlanticVault.sol#L104) 

This all combined lead to an issue, because the current price of RDPX is 13.5 USD and the price of WETH is 1677 USD. This means that the oracle will return 0.008e8 as a price of RDPX represented in WETH. Now when we input this price into [the strike formula](https://github.com/code-423n4/2023-08-dopex/blob/main/contracts/core/RdpxV2Core.sol#L1189-L1190) this happens:
```jsx
    uint256 strike = IPerpetualAtlanticVault(addresses.perpetualAtlanticVault)
      .roundUp(rdpxPrice - (rdpxPrice / 4)); // 25% below the current price
```
**(0.008e8 - (0.008e8 / 4)) => 0.006e8**

And when we call [roundUp](https://github.com/code-423n4/2023-08-dopex/blob/main/contracts/perp-vault/PerpetualAtlanticVault.sol#L576-L583) on this value, It will try and perform this equation to get the [remainder](https://github.com/code-423n4/2023-08-dopex/blob/main/contracts/perp-vault/PerpetualAtlanticVault.sol#L577):  

**0.006e8 % 1e6 => 0.6e6 % 1e6 => 0.6e6**

And afterwards it will turn to the [else](https://github.com/code-423n4/2023-08-dopex/blob/main/contracts/perp-vault/PerpetualAtlanticVault.sol#L581) where it will return `_strike - remainder + roundingPrecision` 

**0.6e6 - 0.6e6 + 1e6 => 1e6**

```jsx
 function roundUp(uint256 _strike) public view returns (uint256 strike) {
    uint256 remainder = _strike % roundingPrecision;
    if (remainder == 0) {
      return _strike;
    } else {
      return _strike - remainder + roundingPrecision;
    }
```
This actually makes the bond **from put** to **call**, as it increases the strike price above our current one.

Also found in [purchase](https://github.com/code-423n4/2023-08-dopex/blob/main/contracts/perp-vault/PerpetualAtlanticVault.sol#L270).

## PoC

Place it in [**Unit.t.sol**](https://github.com/code-423n4/2023-08-dopex/blob/main/tests/rdpxV2-core/Unit.t.sol)
```jsx
  function test_calculate_cost() public {
    address user1 = address(111);
    rdpxPriceOracle.updateRdpxPrice(8e5); // 0.008e8
    vm.prank(user1);
    rdpxV2Core.calculateBondCost(1e18, 0);
  }
```
When run with **-vvvv** we see that we input 6e5 (0.008e8 - 25%) into [roundUp](https://github.com/code-423n4/2023-08-dopex/blob/main/contracts/perp-vault/PerpetualAtlanticVault.sol#L576-L583) and get 1e6.
```
    │   ├─ [2876] PerpetualAtlanticVault::roundUp(600000 [6e5]) [staticcall]
    │   │   └─ ← 1000000 [1e6]
```

## Tools Used
Manual review 

## Recommended Mitigation Steps
Use lower precision for the rounding -- 1e4 as an example.



# [H-02] **PerpetualAtlanticVaultLP** `subtractLoss` can be DoS by donation attack

## Impact
[subtractLoss](https://github.com/code-423n4/2023-08-dopex/blob/main/contracts/perp-vault/PerpetualAtlanticVaultLP.sol#L199-L205) `require` statement makes it possible to DoS this function and makes **PerpetualAtlanticVault** [settle](https://github.com/code-423n4/2023-08-dopex/blob/main/contracts/perp-vault/PerpetualAtlanticVault.sol#L315-L369) unusable. 

## Proof of Concept
The `require` statement in [subtractLoss](https://github.com/code-423n4/2023-08-dopex/blob/main/contracts/perp-vault/PerpetualAtlanticVaultLP.sol#L199-L205) allows an attacker to exploit it by donating a small amount the collateral token, causing the `require` condition to fail consistently.

```jsx
    require(
      collateral.balanceOf(address(this)) == _totalCollateral - loss,
      "Not enough collateral was sent out"
    );
```

The issue arises because this statement strictly checks if the collateral held by the contract ( as balance ) matches the storage variable [_totalCollateral](https://github.com/code-423n4/2023-08-dopex/blob/main/contracts/perp-vault/PerpetualAtlanticVaultLP.sol#L58). By donating any amount of the collateral token to the contract, the value of `_totalCollateral` remains unchanged, while `collateral.balanceOf(address(this))` changes.

As this function is used in [settle](https://github.com/code-423n4/2023-08-dopex/blob/main/contracts/perp-vault/PerpetualAtlanticVault.sol#L315-L369) process, it can cause repeated reverting, leading to a DoS scenario and preventing the admin from settling any bond.

## Tools Used
Manual review

## Recommended Mitigation Steps
You can modify the [require](https://github.com/code-423n4/2023-08-dopex/blob/main/contracts/perp-vault/PerpetualAtlanticVaultLP.sol#L200-L203) statement from `==` to `>=`, or use an another check method.

```jsx
    require(
-     collateral.balanceOf(address(this)) == _totalCollateral - loss,
+     collateral.balanceOf(address(this)) >= _totalCollateral - loss,
      "Not enough collateral was sent out"
    );
```


# [M-01] `reLP` under **ReLPContract** does not return excess, which may lead to small amounts being left stuck in the contract

## Impact
The `reLP` function within the **ReLPContract** does not return excess amounts, potentially resulting in small amounts of WETH being left trapped within the contract.

## Proof of Concept
The [`reLP`](https://github.com/code-423n4/2023-08-dopex/blob/main/contracts/reLP/ReLPContract.sol#L202-L307) function includes an [`addliquidity`](https://github.com/code-423n4/2023-08-dopex/blob/main/contracts/reLP/ReLPContract.sol#L286) call to UniSwap v2. Subsequently, the function [returns](https://github.com/code-423n4/2023-08-dopex/blob/main/contracts/reLP/ReLPContract.sol#L302-L305) the remaining tokenA (RDPX) to the **RdpxV2Core**. However, it neglects to return the remaining tokenB (WETH) held within the contract. Over time, although the un-returned WETH accumulates, but remains permanently trapped within the contract.

## Tools Used
Manual review

## Recommended Mitigation Steps
A simple solution would be to modify the code to also return the unused WETH by adding the following changes:

```diff
     IERC20WithBurn(addresses.tokenA).safeTransfer(
       addresses.rdpxV2Core,
       IERC20WithBurn(addresses.tokenA).balanceOf(address(this))
     );
+    IERC20WithBurn(addresses.tokenB).safeTransfer(
+      addresses.rdpxV2Core,
+      IERC20WithBurn(addresses.tokenB).balanceOf(address(this))
+    );
```


# [M-02] `addLiquidity` under **ReLPContract** lacks a slippage check, which leads to losses

## Impact
[addLiquidity](https://github.com/code-423n4/2023-08-dopex/blob/main/contracts/reLP/ReLPContract.sol#L286-L295) has a slippage input that in neglected in the current code, which combined with the non-existent deadline leads to losses and unpredictable trades.

## Proof of Concept
[addLiquidity](https://github.com/code-423n4/2023-08-dopex/blob/main/contracts/reLP/ReLPContract.sol#L286-L295) has 2 parameters (`amountAMin` and `amountBMin`), that account for slippage losses, if the trade executes at a bad time, or protect against MEV. However in the current implementation, the slippage check in not accounted for and in it's place 0 [is used](https://github.com/code-423n4/2023-08-dopex/blob/main/contracts/reLP/ReLPContract.sol#L291-L292), meaning any change in price will satisfy the call. This combined with [no deadline](https://github.com/code-423n4/2023-08-dopex/blob/main/contracts/reLP/ReLPContract.sol#L294C27-L294C27) (as `block.timestamp` means the TX will pass anytime, 1h, 1 day or even 1 week after is sent) means that the trade can execute at unprecedented time, and unknown prices.

A likely scenario would be for the call to remain stuck in the mem-pool for some days, executing at bad prices - **Protocol receives less LP**
 

```jsx
    (, , uint256 lp) = IUniswapV2Router(addresses.ammRouter).addLiquidity(
      addresses.tokenA,
      addresses.tokenB,
      tokenAAmountOut,
      amountB / 2,
      0,
      0,//@audit should should be `amountBMin`
      address(this),
      block.timestamp + 10//@audit why?
    )
```
## Tools Used
Manual review

## Recommended Mitigation Steps
You can either add `minA` and `minB `to the function parameters, or like in other contract implement slippage tolerance calculation. Here is my pseudo-code that you can add [here](https://github.com/code-423n4/2023-08-dopex/blob/main/contracts/reLP/ReLPContract.sol#L285), before the [addLiquidity](https://github.com/code-423n4/2023-08-dopex/blob/main/contracts/reLP/ReLPContract.sol#L286-L295) call:
```jsx
uint minA = tokenAAmountOut - (tokenAAmountOut * slippageTolerance);
uint minB = amountB / 2 - ((amountB / 2) * slippageTolerance);
```