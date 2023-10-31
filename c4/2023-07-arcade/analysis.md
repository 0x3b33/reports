# Summary
Tho the given scope was not much ~1k, my allocated time was also not enough for a deep dive to be made in every concept used by Arcade. From my time spend on the code and the current understating of it I can say that is seems secure enough, and it's gonna be even more secure after the c4 audit. The protocol implements some interesting topics, like different vaults for keeping track/increasing voting power. It also featured a quite unique concept of **ERC1155** NFTs used to boost the voting power in one of the vaults. Overall the code quality was good and there were some minor bugs and improvements detected by me, but so far nothing protocol breaking.

| Severity | Occurrences |
|----------|------------|
| High     | 0          |
| Medium   | 4          |
| Low      | 4          |
| R        | 1          |
| INFO     | 1          |

# Time allocations

|            |            |
|------------|------------|
| Start date | 27.07.2023 |
| End date   | 29.07.2023 |
| Total      | 2 days     |

| Members       | Positions            | Time spent |
|---------------|----------------------|------------|
| **0x3b**      | full time researcher | ~17H       |

# Findings
A diverse range of findings was presented for review, categories such as logical errors, code errors, and recommendations and even some optimizations and improvements that the devs can do. Nonetheless, the late entry that I had into the competition, ment that there was not enough time to delve deeply into any of the concepts.


# Codebase quality
Code was interesting and of high quality, this may be because there was an audit before the one in c4, but nonetheless it had good and stable concepts, multiple variations of vaults (always good to have an option ), each one with it's own advantages and disadvantages. The code also featured a spending system that was very interesting, although the bad part is that the main voting mechanism were not in scope. It would have been a pleasure for me to dive into them, but  we are gonna save it for the next audit hopefully.

# Mechanism review

###### Vaults
The code featured many different implementations of vaults, each with it's own advantages and disadvantages . As the seeing the differences within the vaults it was presumed that **NFTBoostVault** was the most advanced part, so his allocated share of the time was greatest. Another interesting vault was [ARCDVestingVault](https://github.com/code-423n4/2023-07-arcade/blob/main/contracts/ARCDVestingVault.sol) where grant were given to users. This system was not the most decentralized one, as one's manager needed to allow a user to have voting power, but it too seemed interesting. As always the base as [BaseVotingVault](https://github.com/code-423n4/2023-07-arcade/blob/main/contracts/BaseVotingVault.sol) was the most simple and secure one.

The one that got my eye was [NFTBoostVault](https://github.com/code-423n4/2023-07-arcade/blob/main/contracts/NFTBoostVault.sol), not only because it was the biggest solc, but also because it featured quite unique  boosting mechanism. The mechanism was interesting mainly because it used NFTs, the 1155 type, the idea was good, but there was one negative, and in was in the way that the owner needed manually to set the NTF address and it's own unique ID to a boost percentage. This may be gas costly as many NFT collections have 1k - 10k NFTs in them, but if the arcade collection has less than 500 I think it would not be that big of an issue.

###### Tokens
As there is not much code with the tokens there is not much to say about them. The [distributor](https://github.com/code-423n4/2023-07-arcade/blob/main/contracts/token/ArcadeTokenDistributor.sol) was interesting in a way that is was a single use contract. But it was quite simple, and simple things are generally secure, as there is not much to break within them. [ArcadeToken](https://github.com/code-423n4/2023-07-arcade/blob/main/contracts/token/ArcadeToken.sol) was a nice edition to the system as it featured limited supply minting, as in a way not to inflate the token too much (as most projects do) and with an inflation rate of 2%. Finally [ArcadeAirdrop](https://github.com/code-423n4/2023-07-arcade/blob/main/contracts/token/ArcadeAirdrop.sol) was most out of scope, due to [ArcadeMerkleRewards](https://github.com/code-423n4/2023-07-arcade/blob/main/contracts/libraries/ArcadeMerkleRewards.sol) being out of scope, but nonetheless the contract is simple enough not to leak value.

###### NFTs
As the shortest of them all the NFT part it didn't offer much, as there were only a few function users can use ([mint](https://github.com/code-423n4/2023-07-arcade/blob/main/contracts/nft/ReputationBadge.sol#L98-L120)),  although it used ERC1155 which was a nice improvement that make it different from the rest of projects that use NFTs (as they include them just to show off to the users that they have an NFT). One improvement in [mint](https://github.com/code-423n4/2023-07-arcade/blob/main/contracts/nft/ReputationBadge.sol#L98-L120) that i can suggest is to return the rest of `msg.value` to the users after the mint, as in the current implementation that is not done.

# Architecture recommendations
There is not much to be added to what Arcade already has. As in the current scope that was given there was use of advanced NFTs (ERC1155), ERC20 tokens and a few different implementations of vaults. There is not much more needed for a stable system, as in solidity more is not always better, because more code and more thing introduces more risk, that is not linear, but exponential as complexity increases.

# Centralization risk
From the perspective of the scope that we were given Arcade seemed quite centralized, due to the extensive use of onlyOwner/onlyManager, and close to 50% of the function being only accessible with one of these roles. Since I have not looked deeper into the out of scope system I cannot say for sure how centralized or decentralize the whole system is, however from seeing that some contracts (a brief look on out of scope code) implement a voting mechanism and out give scope if for the contracts that allocate the voting power, I can **assume** that Arcade is at least decentralized enough for any user to feel safe, and more decentralized than the normal protocols.

# Systemic risks
For now the risk factor seems under control, as one audit was already done and currently a second one in c4 is near completion. Generally for systems with lack of intense complexity one, two or maximum three audits should be enough to cover the major faults.
