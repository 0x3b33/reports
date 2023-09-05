# Table of findings

|          |                                                                                    |
|----------|------------------------------------------------------------------------------------|
| **L-01** | `executeFlashloan` does not check if funds are bigger or equal after the execution |
| **L-02** | EUSD can be transferred even when paused.                                          |

**[LOW-01]** `executeFlashloan` does not check if funds are **>=** after the execution.

In [`PeUSDMainnetStableVision`](https://github.com/code-423n4/2023-06-lybra/blob/main/contracts/lybra/token/PeUSDMainnetStableVision.sol), there is an [`executeFlashloan`](https://github.com/code-423n4/2023-06-lybra/blob/main/contracts/lybra/token/PeUSDMainnetStableVision.sol#L129-L139) function that, as the name suggests, is used for users to make flash loans. However, best practices are not followed in this function. It pulls funds from the receiver and does not check if the funds are greater than or equal to the amount before the flash loan.
Add a check for if the funds are more or equal to the ones before the flash loan.
```jsx
    function executeFlashloan(FlashBorrower receiver, uint256 eusdAmount, bytes calldata data) public payable {
        uint256 shareAmount = EUSD.getSharesByMintedEUSD(eusdAmount);
+       uint sharesBefore = balanceOf(address(this); //balance of on EUSD gets the shares of an account 
        EUSD.transferShares(address(receiver), shareAmount);
        receiver.onFlashLoan(shareAmount, data);//audot LOW cuz it does not check balances before and after
        bool success = EUSD.transferFrom(address(receiver), address(this), EUSD.getMintedEUSDByShares(shareAmount));
        require(success, "TF");
+       uint sharesAfter = balanceOf(address(this); 
+       require(sharesBefore <= sharesAfter,"FL failed");

        uint256 burnShare = getFee(shareAmount);
        EUSD.burnShares(address(receiver), burnShare);
        emit Flashloaned(receiver, eusdAmount, burnShare);
    }
```

**[LOW-02]** EUSD can be transferred even when paused.

In the [comments](https://github.com/code-423n4/2023-06-lybra/blob/main/contracts/lybra/token/EUSD.sol#L222), the developers state that a transfers and increase or decrease on allowances  should not be possible if the contract is paused. However, the [function](https://github.com/code-423n4/2023-06-lybra/blob/main/contracts/lybra/token/EUSD.sol#L226) lacks the modifier for pausing transfers. Here's an example of the code:
```jsx
function transferFrom(address from, address to, uint256 amount) public returns (bool) {//audit no pause modifier here?
    address spender = _msgSender();
    if (!configurator.mintVault(spender)) {
        _spendAllowance(from, spender, amount);
    }
    _transfer(from, to, amount);
    return true;
}
```
Include the pause modifier in the functions that should not operate when paused.