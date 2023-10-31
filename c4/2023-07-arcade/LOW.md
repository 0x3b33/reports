### **[LOW-01]** An owner can mint lower than the max allowed amount an thus reduce the supply
 When doing a second mint in `ArcadeToken` the owner is checked if the imputed amount [is lower](https://github.com/code-423n4/2023-07-arcade/blob/main/contracts/token/ArcadeToken.sol#L155-L157) than inflation cap and it it is the TX passes, supply is minted and  `mintingAllowedAfter` is set for the next year. But the issue here is that if the owner inputs any small amount, like 1 token or 1 wei, the TX will pass and mint that amount, meaning this year's mint is basically skipped (because the amount is is insignificant compared to the totalSupply)

### **[R-01]** In [addGrantAndDelegate](https://github.com/code-423n4/2023-07-arcade/blob/main/contracts/ARCDVestingVault.sol#L115) if the manager has not provided enough tokens the TX will revert.
When  a manger calls  [addGrantAndDelegate](https://github.com/code-423n4/2023-07-arcade/blob/main/contracts/ARCDVestingVault.sol#L115), he needs to make sure if there are enough tokens stored in `unassigned.data`, if there are not enough he needs to call [deposit](https://github.com/code-423n4/2023-07-arcade/blob/main/contracts/ARCDVestingVault.sol#L197-L202) and then [addGrantAndDelegate](https://github.com/code-423n4/2023-07-arcade/blob/main/contracts/ARCDVestingVault.sol#L115) and this is an inconvenience for him, and waste of gas. 
I am quite sure   [addGrantAndDelegate](https://github.com/code-423n4/2023-07-arcade/blob/main/contracts/ARCDVestingVault.sol#L115) can be made to transfer te remainder of tokens, if `unassigned.data` is not enough.
Example improvement:
```jsx
    function addGrantAndDelegate() external onlyManager {
    ...
        Storage.Uint256 storage unassigned = _unassigned();
        if (unassigned.data < amount) {
            token.transferFrom(msg.sender,address(this), amount - unassigned.data);
            unassigned.data = 0;
        }
    ...
```

### **[INFO-01]** expiration and cliff should be marked as block.number instead of timestamp
As they [are used](https://github.com/code-423n4/2023-07-arcade/blob/main/contracts/ARCDVestingVault.sol#L234) in the code as block.number, but described as [timestamp](https://github.com/code-423n4/2023-07-arcade/blob/main/contracts/ARCDVestingVault.sol#L86-L87)