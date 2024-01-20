| *Issue* | *Description*                                                                                                                                                                                                                                                                                                                                                              |
|---------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| [L-01]  | Changing `gencore` will effect the mints                                                                                                                                                                                                                                                                                                                                   |
| [L-02]  | [getPrice](https://github.com/code-423n4/2023-10-nextgen/blob/main/smart-contracts/MinterContract.sol#L540) uses different time checks from both [mint](https://github.com/code-423n4/2023-10-nextgen/blob/main/smart-contracts/MinterContract.sol#L202) and [burnToMint](https://github.com/code-423n4/2023-10-nextgen/blob/main/smart-contracts/MinterContract.sol#L260) |
| [L-03]  | People can set their hash to 0x00 |                                                                                                                                                                                                                                                 
| [L-04]  | If randomizer is changed while a mint is happening token hash will be 0 |                                          
| [L-05]  | Emergency functions should not perform external calls |

### [L-01] Changing `gencore` will effect the mints
Admins are able to change the gencore used by the minter with the help of [updateCoreContract](). However such change will effect any minting that are held, as the information inside gencore ([whitelists](https://github.com/code-423n4/2023-10-nextgen/blob/main/smart-contracts/MinterContract.sol#L261), [mint counts](https://github.com/code-423n4/2023-10-nextgen/blob/main/smart-contracts/MinterContract.sol#L213C50-L213C82), [cirSupply](https://github.com/code-423n4/2023-10-nextgen/blob/main/smart-contracts/MinterContract.sol#L220) and so on) is used in the minting process.


### [L-02] [getPrice](https://github.com/code-423n4/2023-10-nextgen/blob/main/smart-contracts/MinterContract.sol#L540) uses different time checks from both [mint](https://github.com/code-423n4/2023-10-nextgen/blob/main/smart-contracts/MinterContract.sol#L202) and [burnToMint](https://github.com/code-423n4/2023-10-nextgen/blob/main/smart-contracts/MinterContract.sol#L260)

While [mint](https://github.com/code-423n4/2023-10-nextgen/blob/main/smart-contracts/MinterContract.sol#L202) and [burnToMint](https://github.com/code-423n4/2023-10-nextgen/blob/main/smart-contracts/MinterContract.sol#L260) use `>=` and `<=` phase 2 under [getPrice](https://github.com/code-423n4/2023-10-nextgen/blob/main/smart-contracts/MinterContract.sol#L540) uses only `>` and `<`.
```solidity
        } else if (
            collectionPhases[_collectionId].salesOption == 2
                && block.timestamp > collectionPhases[_collectionId].allowlistStartTime 
                && block.timestamp < collectionPhases[_collectionId].publicEndTime
        ) {
```
This will lead to a few inconsistency. Example is when users try and mint at `block.timestamp == publicEndTime` Which is possible in both [mint](https://github.com/code-423n4/2023-10-nextgen/blob/main/smart-contracts/MinterContract.sol#L202) and [burnToMint](https://github.com/code-423n4/2023-10-nextgen/blob/main/smart-contracts/MinterContract.sol#L260), however instead of giving the decreased price,it will go into the [else](https://github.com/code-423n4/2023-10-nextgen/blob/main/smart-contracts/MinterContract.sol#L566) and return the normal `collectionMintCost`

### [L-03] People can set their hash to 0x00

When minting tokens [_mintProcessing](https://github.com/code-423n4/2023-10-nextgen/blob/main/smart-contracts/NextGenCore.sol#L227-L232) which calls `randomizer.calculateTokenHash(...)`, afterwards the randomizer calls [setTokenHash](https://github.com/code-423n4/2023-10-nextgen/blob/main/smart-contracts/NextGenCore.sol#L299-L303) to set the hash for this ID . When ChainLink is used as the randomizer the call is returned after a few block. This call can be made to revert on the return call, by simple providing enough gas for the request to reach ChainLink, but not enough for the return call to be successful. If the return call runs out of gas [setTokenHash](https://github.com/code-423n4/2023-10-nextgen/blob/main/smart-contracts/NextGenCore.sol#L299-L303) will never be called, thus leaving `tokenToHash[tokenID] == 0`. This will not cause any damage to the protocol, but may render some NFTs useless, or the opposite - extremely rare, due to their nature.

### [L-04] If randomizer is changed while a mint is happening token hash will be 0
[addRandomizer](https://github.com/code-423n4/2023-10-nextgen/blob/main/smart-contracts/NextGenCore.sol#L170-L174) is used to change the randomizer for its collection. This function can be called at any time by an admin of this collection. When it is called, a new randomizer will be put in place. However, if a call is made to the old randomizer, its return function [setTokenHash](https://github.com/code-423n4/2023-10-nextgen/blob/main/smart-contracts/NextGenCore.sol#L299-L303) will take some time to execute. This means that the NFTs that were minted just before the change will have a hash of 0, rendering them useless.

Example:
1. A user mints an NFT - [_mintProcessing](https://github.com/code-423n4/2023-10-nextgen/blob/main/smart-contracts/NextGenCore.sol#L229) calls `randomizer.calculateTokenHash`, and VRF will take 3 blocks to respond.
2. An admin changes the randomizer.
3. Chainlink responds; however, [setTokenHash](https://github.com/code-423n4/2023-10-nextgen/blob/main/smart-contracts/NextGenCore.sol#L299-L303) reverts, as Chainlink is not the current randomizer.

The NFTs will have a hash of 0.

### [L-05] Emergency functions should not perform external calls 

[emergencyWithdraw](https://github.com/code-423n4/2023-10-nextgen/blob/main/smart-contracts/MinterContract.sol#L461-L466) as the name suggest does a emergent withdraw. Emergent means that most likely a hack has happened and the protocol is looking to rescue the funds. However the function calls `adminsContract.owner()` which in direct contradiction with the emergency state. This may cause an issue if `adminsContract` is compromised. I suggest to have this address in storage or to be an input to the function. 