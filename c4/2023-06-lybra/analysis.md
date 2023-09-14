# Summary
We spent a week conducting a thorough research of the Lybra protocol, identifying areas for improvement in terms of logical and code errors, diving into different parts of the protocol, including it's used dependencies. We were impressed by Lybra's staking features and support for ETH derivatives.

| Severity | Occurences |
|----------|------------|
| High     | 5          |
| Medium   | 6          |
| Low      | 2          |
| Gas      | 1          |

# Time allocations

|            |            |
|------------|------------|
| Start date | 25.06.2023 |
| End date   | 2.07.2023  |
| Total      | 7 days     |

| Members       | Positions            | Time spent |
|---------------|----------------------|------------|
| **0x3b**      | full time researcher | 50H+       |
| **ZdravkoHr** | part time researcher | 20H+       |

# Findings and how we found them

As always, various types of findings were submitted for review. These findings can be categorized as logical errors, code errors, and recommendations. However, due to the nature of the protocol and the multitude of concepts Lybra implemented simultaneously, the code was not fully finished when it was presented to c4. As a result, we discovered some incorrect modifier checks, issues with stuck rewards, improperly implemented voting periods, and other types of vulnerabilities.

# Codebase quality
The code although faulty, was up to a good level of quality! However, the developers, most likely under pressure, did not have enough time to polish it. It lacked sufficient nat-spec and did not include any tests to ensure it's functionality and reliability. In addition, some functions that use the camelCase naming convention have a capital letter in the middle of their names. At some places in the code there are magic values witch can be assigned to a constant or a variable. Despite that, the code structure was impressive, with simple implementation, well though out math, and a DAO to lower the centralization risk.

# Mechanism review

###### Staking && Rewards
Lybra has implemented some amazing staking features. The code even features [staking](https://github.com/code-423n4/2023-06-lybra/blob/main/contracts/lybra/miner/EUSDMiningIncentives.sol) for borrowers of EUSD, which is quite innovative. It is really appreciated when a protocol, does not include rewards, just to lure users to stake, but it actually offers some future potential for it's users.

One bad side of it though is the long exit cycle of 90 days. Stakers should have a period after the deposit where the funds are locked, this will prevent any users who are here just to trade it's coins and hop on money printer, but the one that Lybra implements is too long. This may lead to user frustration, because the system is locking their funds for a whole 3 months! This is true even without the users using any boosts. As a suggestion, you can add a functionality that lets the users skip or shorten the waiting period after a long boost.

###### ETH and it's derivatives
We were pleasantly surprised to discover that Lybra doesn't merely include tokens that have no meaningful purpose beyond staking. Instead, they provide support for well-established liquid stakers like Lido and Rocketpool by offering ETH. The vault structure is thoughtfully designed, allowing users to borrow against their stake and even earn rewards on the borrowed amount, specifically with EUSD. It's worth noting that the protocol ensures exceptional stability in terms of liquidations due to its high over-collateralization of 150%, making it highly unlikely to encounter bad debt on system's side.

###### DAO
We have explained in detail about the DAO in the centralization risks.

###### Stable coins
Lybra's stable coin implementation is on spot. hey have introduced PeUSD, a multi-chain stablecoin, which is well-executed. The overspecialization ratio of 150% is in line with Compound, and Lybra goes a step further by offering the option to adjust it slightly. This feature provides additional safety in volatile markets and allows for more capital allocation in safer markets.

# Centralization Risks
DAOs provide the promise of decentralization, however, it is evident that a small group of influential individuals often control the majority of proposals. A notable illustration is Compound's recent proposal-[165](https://compound.finance/governance/proposals/165) where the top nine voters held 99% (400 million votes) of the total voting power. These same addresses consistently dominate the proposal landscape. To enhance decentralization further, we recommend that Lybra considers implementing [quadratic voting](https://blog.tally.xyz/a-simple-guide-to-quadratic-voting-327b52addde1). This approach would distribute power more equally and contribute to a more decentralized system.

Another drawback of DAOs is that, despite their decentralized nature, the system may struggle to thrive in practice. Common users often lack interest in participating in the voting process if it does not directly improve their APY.

# Systemic risks 
Lybra faces significant systemic risks due to the intricate nature of its operations, encompassing various components such as DAO, staking, ETH derivatives, and stable coins, all functioning as a cohesive unit. This complexity introduces multiple potential points of vulnerability. The system becomes susceptible to attacks from various angles, necessitating thorough security measures. In order to be secure Lybra's development team needs to follow common practices and do not only one, but a couple of audits.