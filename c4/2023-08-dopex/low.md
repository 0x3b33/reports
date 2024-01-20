| *Issue*    | *Description*                                                                                                                                                                                    |
|------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **[L-01]** | [upperDepeg](https://github.com/code-423n4/2023-08-dopex/blob/main/contracts/core/RdpxV2Core.sol#L1051-L1070) has `DEFAULT_ADMIN_ROLE`, although it specifies that it should be called by users. |
| **[L-02]** | Bond discount needs to be change every so often to avoid issues                                                                                                                                  |
| **[L-03]** | Wrong `_validate`                                                                                                                                                                                |
| **[L-04]** | [_stake](https://github.com/code-423n4/2023-08-dopex/blob/main/contracts/core/RdpxV2Core.sol#L566-L583) does not return bondID leaving users to guess                                            |

### **[L-01]** [upperDepeg](https://github.com/code-423n4/2023-08-dopex/blob/main/contracts/core/RdpxV2Core.sol#L1051-L1070) has `DEFAULT_ADMIN_ROLE`, although it specifies that it should be called by users.
The function is specified to allow Users to mint and increase the peg, however the function is noted with `DEFAULT_ADMIN_ROLE`.
```jsx
 /**
   * @notice Lets users mint DpxEth at a 1:1 ratio when DpxEth pegs above 1.01 of the ETH token
   */
  function upperDepeg() external onlyRole(DEFAULT_ADMIN_ROLE) returns (uint256 wethReceived) {
```

### **[L-02]** Bond discount needs to be change every so often to avoid issues
Under [calculateBondCost](https://github.com/code-423n4/2023-08-dopex/blob/main/contracts/core/RdpxV2Core.sol#L1163-L1165) `bondDiscount` is counted for with static value against movable `rdpxReserves` which means that with market turns up and down, this value needs to be changed constantly. This could be annoying and if forgotten or updated lately it can cause issue, either giving too much or too little discount.

### **[L-03]** Wrong `_validate`
`_validate` under [calculateBondCost](https://github.com/code-423n4/2023-08-dopex/blob/main/contracts/core/RdpxV2Core.sol#L1163-L1165) is unnecessary since its check is useless. It check if `bondDiscount < 100e8` and reverts on true. However if `bondDiscount > 50e8` (half of this check) it will underflow in [rdpxRequired](https://github.com/code-423n4/2023-08-dopex/blob/main/contracts/core/RdpxV2Core.sol#L1169-L1173). This is because [RDPX_RATIO_PERCENTAGE](https://github.com/code-423n4/2023-08-dopex/blob/main/contracts/core/RdpxV2Core.sol#L88) is only 25e8 and if the sub-tractor is bigger than 50e8 it will cause an underflow revert. 
```jsx
(RDPX_RATIO_PERCENTAGE - (bondDiscount / 2)
(25e8 - (51e8 / 2) // revert on underflow
```

### **[L-04]** [_stake](https://github.com/code-423n4/2023-08-dopex/blob/main/contracts/core/RdpxV2Core.sol#L566-L583) does not return bondID leaving users to guess 
Simply the functions [bond](https://github.com/code-423n4/2023-08-dopex/blob/main/contracts/core/RdpxV2Core.sol#L894-L933) and [bondWith](https://github.com/code-423n4/2023-08-dopex/blob/main/contracts/core/RdpxV2Core.sol#L819-L885) do not return bondIDs, leaving users who want to reuse their bond to guess that ID