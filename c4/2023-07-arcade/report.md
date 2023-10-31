| Severity | Title | 
|:--:|:---|
| [M-01](#m-01-user-who-have-claimed-before-the-update-on-the-merkle-tree-will-have-no-shares-of-the-rewards-afterwards) | User who have claimed before the update on the merkle tree will have no shares of the rewards afterwards |
| [M-02](#m-02-setmultiplier-does-not-sync-the-voting-power-and-voters-can-abuse-it-by-not-refreshing-their-voting-power-before-a-vote) | `setMultiplier` does not sync the voting power, and voters can abuse it by not refreshing their voting power before a vote |



# [M-01] User who have claimed before the update on the merkle tree will have no shares of the rewards afterwards

## Impact
User who have claimed before an update on the merkle tree in [ArcadeAirdrop](https://github.com/code-423n4/2023-07-arcade/blob/main/contracts/token/ArcadeAirdrop.sol) are not gonna be able to claim the extra rewards. The reward tokens will be left stuck in the contract, waiting for the manager to reclaim them.

## Proof of Concept
Due to the nature of how [ArcadeAirdrop](https://github.com/code-423n4/2023-07-arcade/blob/main/contracts/token/ArcadeAirdrop.sol) is made and [the expectation](https://github.com/code-423n4/2023-07-arcade/blob/main/contracts/token/ArcadeAirdrop.sol#L17-L19) that there might be a change in the merkle tree in the near future, users will expect to be able to claim the extra re
wards (if they are increase on merkle root change). However, because there is a problematic [if stamen](https://github.com/code-423n4/2023-07-arcade/blob/main/contracts/libraries/ArcadeMerkleRewards.sol#L106-L107) (that is inherited in [ArcadeAirdrop](https://github.com/code-423n4/2023-07-arcade/blob/main/contracts/token/ArcadeAirdrop.sol) by [ArcadeMerkleRewards](https://github.com/code-423n4/2023-07-arcade/blob/main/contracts/libraries/ArcadeMerkleRewards.sol)) users that have already claimed their rewards will have no share of the future ones.

Example: 
1. Alice and Bob bolt have **100** tokens set to be claimed on the Merkle tree
2. Alice claims her **100** tokens
3. The owner sees that there is a huge activity on the platform and wants to incentivize users, so he increases rewards (by changing the merkle root) to **120** tokens
4. Bob claims his **120** tokens
5. Alice sees that and tries to claim the remainder of her **120** tokens (**20**), but the claim reverts because of this [if](https://github.com/code-423n4/2023-07-arcade/blob/main/contracts/libraries/ArcadeMerkleRewards.sol#L106-L107)

```jsx
    function _validateWithdraw(uint256 totalGrant, bytes32[] memory merkleProof) internal {
        ...
        //@audit if the user has already claimed, he is not able to claim ever again
        if (claimed[msg.sender] != 0) revert AA_AlreadyClaimed();
        claimed[msg.sender] = totalGrant;
    }
```

## Tools Used
Manual review 
## Recommended Mitigation Steps
Change the claim implementation the way how [mint](https://github.com/code-423n4/2023-07-arcade/blob/main/contracts/nft/ReputationBadge.sol#L98-L120) in **ReputationBadge** is done. They have the [total amount](https://github.com/code-423n4/2023-07-arcade/blob/main/contracts/nft/ReputationBadge.sol#L110C47-L110C61) in the merkle tree and track [claimed amount](https://github.com/code-423n4/2023-07-arcade/blob/main/contracts/nft/ReputationBadge.sol#L116C1-L116C1) in a mapping.


# [M-02] `setMultiplier` does not sync the voting power, and voters can abuse it by not refreshing their voting power before a vote

## Impact
[setMultiplier](https://github.com/code-423n4/2023-07-arcade/blob/main/contracts/NFTBoostVault.sol#L363-L371) as the name suggests set a new multiplier for a token address of a given token ID, but it does not sync the voting power afterwards. This can be exploited by voters who after the update have received lower boost (and thus lower voting power), by them not updating their voting power.

## Proof of Concept
[setMultiplier](https://github.com/code-423n4/2023-07-arcade/blob/main/contracts/NFTBoostVault.sol#L363-L371) is a function used by the manager of **NFTBoostVault** to set a new voting power for a NFT and it's corresponding token ID. However after this change no measure is done to enforce it, and voters and delegates are left with the old (before `setMultiplier` was called) voting power. This means if an owner deliberately want to lower to voting power of a certain NFT he can set the boost ration of that NFT to a lower amount, but users with that NFT will still receive the prev. boost, unless they are synced manually.

Example:
1. A given faction has 100 000 tokens and NFTs of type **cow** that gives 150% voting power => 150 000 vote tokens 
2. Owner doesn't like cows (for some reason) so he lowers the voting power of NFT **cow** to 120%
3. The group of user does not care and can vote with the prev. voting power, since the onwer needs to sync their voting power manually, user by user, and thy may cost a fortune

## Tools Used
Manual review

## Recommended Mitigation Steps
Add a modifier that before every call updates the voting power of the `msg.sender`




