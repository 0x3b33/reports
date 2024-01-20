| *Issue*    | *Description*                                                                                                                                                                                         |
|------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **[G-01]** | Call functions directly                                                                                                                                                                               |
| **[G-02]** | Save strike in memory earlier                                                                                                                                                                         |
| **[G-03]** | Save equation in memory earlier                                                                                                                                                                       |
| **[G-04]** | Instead of calling it multiple times [nextFundingPaymentTimestamp](https://github.com/code-423n4/2023-08-dopex/blob/main/contracts/perp-vault/PerpetualAtlanticVault.sol#L563-L569) save it in memory |
| **[G-05]** | [_curveSwap](https://github.com/code-423n4/2023-08-dopex/blob/main/contracts/core/RdpxV2Core.sol#L515-L542) input parameter `validate` is useless                                                     |
| **[G-06]** | Check is not need                                                                                                                                                                                     |

### **[G-01]** Call functions directly 
Under [deposit](https://github.com/code-423n4/2023-08-dopex/blob/main/contracts/perp-vault/PerpetualAtlanticVaultLP.sol#L118-L135) you can skip [previewDeposit](https://github.com/code-423n4/2023-08-dopex/blob/main/contracts/perp-vault/PerpetualAtlanticVaultLP.sol#L123) and isntead call directly [convertToShares](https://github.com/code-423n4/2023-08-dopex/blob/main/contracts/perp-vault/PerpetualAtlanticVaultLP.sol#L274-L284), as `previewDeposit` just calls `convertToShares`
Same applies under [redeem](https://github.com/code-423n4/2023-08-dopex/blob/main/contracts/perp-vault/PerpetualAtlanticVaultLP.sol#L159), where you can call [_convertToAssets](https://github.com/code-423n4/2023-08-dopex/blob/main/contracts/perp-vault/PerpetualAtlanticVaultLP.sol#L218-L229) directly

### **[G-02]** Save strike in memory earlier
Under [calculateFunding](https://github.com/code-423n4/2023-08-dopex/blob/main/contracts/perp-vault/PerpetualAtlanticVault.sol#L413-L419) strike can be saved earlier to avoid unnecessary memory checks  

```diff
    for (uint256 i = 0; i < strikes.length; i++) {
+      uint256 strike = strikes[i];
      _validate(optionsPerStrike[strikes[i]] > 0, 4);
      _validate(
        latestFundingPerStrike[strikes[i]] != latestFundingPaymentPointer,
        5
      );
-      uint256 strike = strikes[i];
```

### **[G-03]** Save equation in memory earlier 
Under [updateFunding](https://github.com/code-423n4/2023-08-dopex/blob/main/contracts/perp-vault/PerpetualAtlanticVault.sol#L462-L496) this equation bellow is used 3 times in a row. 
```jsx
(currentFundingRate * (block.timestamp - startTime)) / 1e18
```
This is not needed as the result can be saved in memory. and accessed at will for a minimum gas cost.

### **[G-04]** Instead of calling it multiple times [nextFundingPaymentTimestamp](https://github.com/code-423n4/2023-08-dopex/blob/main/contracts/perp-vault/PerpetualAtlanticVault.sol#L563-L569) save it in memory
Under [_updateFundingRate](https://github.com/code-423n4/2023-08-dopex/blob/main/contracts/perp-vault/PerpetualAtlanticVault.sol#L594-L614) you can save [nextFundingPaymentTimestamp](https://github.com/code-423n4/2023-08-dopex/blob/main/contracts/perp-vault/PerpetualAtlanticVault.sol#L602) before the `if` or `else`, saving a few calls to this function. After it you can use the variable [endTime](https://github.com/code-423n4/2023-08-dopex/blob/main/contracts/perp-vault/PerpetualAtlanticVault.sol#L602) in the calculations much more cheaply than doing multiple function calls.

### **[G-05]** [_curveSwap](https://github.com/code-423n4/2023-08-dopex/blob/main/contracts/core/RdpxV2Core.sol#L515-L542) input parameter `validate` is useless 
[_curveSwap](https://github.com/code-423n4/2023-08-dopex/blob/main/contracts/core/RdpxV2Core.sol#L515-L542) is used in 2 places ( [upperDepeg](https://github.com/code-423n4/2023-08-dopex/blob/main/contracts/core/RdpxV2Core.sol#L1051-L1070) and [lowerDepeg](https://github.com/code-423n4/2023-08-dopex/blob/main/contracts/core/RdpxV2Core.sol#L1080-L1124) ), where they each input `validate` as true, meaning that there is no need for this parameter as it can be hard-coded to constantly validate.

### **[G-06]** Check is not need
[_transfer](https://github.com/code-423n4/2023-08-dopex/blob/main/contracts/core/RdpxV2Core.sol#L624-L685) currently checks if [msg.sender matches the owner of the bondID](https://github.com/code-423n4/2023-08-dopex/blob/main/contracts/core/RdpxV2Core.sol#L638-L641), unfortunately this is done in the gas expensive way as the previous call to [IRdpxDecayingBonds.bonds(bondID)](https://github.com/code-423n4/2023-08-dopex/blob/main/contracts/core/RdpxV2Core.sol#L631-L633) also return the owner of the bond, thus rendering this useless.
```diff
-     (, uint256 expiry, uint256 amount) = IRdpxDecayingBonds(
+     (address owner, uint256 expiry, uint256 amount) = IRdpxDecayingBonds(addresses.rdpxDecayingBonds).bonds(_bondId);

      _validate(amount >= _rdpxAmount, 1);
      _validate(expiry >= block.timestamp, 2);
-     _validate( IRdpxDecayingBonds(addresses.rdpxDecayingBonds).ownerOf(_bondId) == msg.sender,9);
+     _validate(msg.sender == owner, 9);
```