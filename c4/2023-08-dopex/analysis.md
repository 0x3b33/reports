# Summary
The codebase quality displayed a solid foundation with intriguing concepts, though there's room for improvement, particularly in mathematical aspects. A variety of mechanisms were reviewed, such as UniV2LiquidityAmo.sol, UniV3LiquidityAmo.sol, and RdpxV2Core.sol, each offering distinctive features and functionalities.  Recommendations emphasized that while the current system is comprehensive, excessive complexity doesn't necessarily lead to improvement. A balanced and refined approach is essential for stability.

| Severity | Occurrences |
|----------|-------------|
| High     | 6           |
| Medium   | 8           |
| Low      | 4           |
| Gas      | 6           |

# Time allocations

|            |            |
|------------|------------|
| Start date | 21.08.2023 |
| End date   | 31.08.2023 |
| Total      | 10 days    |

| Members       | Positions            | Time spent |
|---------------|----------------------|------------|
| **0x3b**      | full time researcher | 70H+       |



# Findings
A wide array of findings were provided for assessment, spanning across categories such as logical errors, mathematical inaccuracies, and code-related mistakes. Additionally, suggestions for enhancements, optimizations, and improvements that developers could implement were also included. Having dedicated ample time to thoroughly examine the code, I delved deeply into various concepts, striving to uncover as many vulnerabilities as possible.


# Codebase quality
The code quality demonstrated a solid foundation, featuring intriguing concepts alongside unconventional mechanics. However, there's room for refinement to further elevate its performance. The mathematical aspects displayed a slight imbalance, stemming from a lack of meticulous refinement. While some instances indicated a correct approach, they fell short of comprehensively achieving the intended outcome. For instance, one of the findings highlighted a 22/78 split in the bond, deviating slightly from the optimal 25/75 ratio that Dopex requires.

# Mechanism review

###### UniV2LiquidityAmo.sol
- This contract links the bond ecosystem with UNIv2.
- It implements necessary standard functions.
- It offers an emergency withdrawal option in case of issues.
- Noteworthy handling of deadlines, demonstrated with [block.timestamp + 1](https://github.com/code-423n4/2023-08-dopex/blob/main/contracts/amo/UniV2LiquidityAmo.sol#L342).

###### UniV3LiquidityAmo.sol
- Essential handler for UNIv3.
- Encompasses fundamental functions.
- Lacks internal slippage calculation function.
- Provides an emergency withdrawal feature in case of emergencies.
- Features an interesting `execute` function that grants administrators versatile control.

###### RdpxV2Core.sol
- Central contract within the system.
- Introduces intriguing bonding mechanics.
- Presents an option for bonding with delegates in the absence of WETH or if the user wants exclusive use of DOPEX.
- Enforces a pegging mechanism, unfortunately under administrative control.
- Features an emergency withdrawal option for unexpected situations.

## RdpxV2Bond.sol
- A straightforward ERC721 minting contract.
- Contains functions for pausing and unpausing.

## RdpxDecayingBonds.sol
- Basic ERC721 contract.
- Offers additional getter functions for data extraction.
- Features an emergency withdrawal option in unforeseen situations.

## DpxEthToken.sol
- Fundamental ERC20 contract.

## PerpetualAtlanticVault.sol
- Manages collateral movement between V2Core and LP vault.
- Incorporates the "drip" mechanism, gradually sending collateral to the LP vault.
- Encompasses a function for purchasing premiums.
- Utilizes the BlackScholes mechanism for premium calculation.
- Implements a mechanism for bond settlement.

## PerpetualAtlanticVaultLP.sol
- A 4626 vault.
- Stores and gradually releases DRPX and collateral.
- Provides regular deposit and redeem functions.
- Evaluated against a 1 wei deposit attack.
- Contains mechanisms to prevent manipulation of shares.

## ReLPContract.sol
- Contract designed for adding/removing liquidity to uniV2.
- Features a function for setting global slippage tolerance.


# Architecture recommendations
Dopex's existing framework is quite comprehensive, encompassing advanced bonds, intricate pegging mechanisms, and the integration of ERC20 tokens. Within the parameters of the current scope, the system appears to be well-equipped. It's worth noting that in solidity, additions don't necessarily equate to improvement. Rather, additional complexity can introduce heigher risks in a non-linear, exponential manner. Therefore, maintaining a balanced and refined approach is key to a stable system.

# Centralization risk
Considering the provided scope, Dopex appeared to show a degree of centralization. This was primarily attributed to the prevalent utilization of access control modifiers such as onlyOwner or similar appointed admin roles. Approximately 50% of the functions were restricted to access by these specific roles. While this centralization was noticeable, it's important to recognize that the system's functionality doesn't inherently necessitate decentralization. As long as the admins maintain a belief in the system's future and remain dedicated to its upkeep and enhancement, its effectiveness can still be existing without the need for complete decentralization.

# Systemic risks
Since this is their first review, it's normal to find some bugs, even though the system is already quite good. There's still room for making it better. Besides the bugs, there's a potential risk related to how the pegging mechanisms are controlled only by the admins.

For systems as complex as this one, it's a good idea to think about doing another review after this first one. This second review could help find any important problems that might have been missed. Complicated systems often need multiple audits to make sure they're safe and working well.