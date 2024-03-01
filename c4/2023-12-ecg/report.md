| Severity | Title |
|:--------:|:------|
| [M-01](#m-01-there-is-an-inconsistency-between-borrowers-debt-ceiling-and-debtceiling) | There is an inconsistency between borrower's debt ceiling and debtCeiling |
| [M-02](#m-02-gauges-weight-will-still-take-effect-in-markets-where-there-is-only-one-term) | Gauge's weight will still take effect in markets where there is only one term |
| [M-03](#m-03-auctionhouse-whales-can-win-large-auctions-via-block-stuffing) | AuctionHouse Whales can win large auctions via block stuffing |
| [M-04](#m-04-borrowers-can-reduce-the-debt-ceiling-bellow-the-issuance) | Borrowers can reduce the debt ceiling bellow the issuance |
| [M-05](#m-05-liquidators-can-extract-extra-value-with-flash-loans-significantly-reducing-profits-for-other-users) | Liquidators can extract extra value with flash loans, significantly reducing profits for other users |
| [M-06](#m-06-deprecating-a-gauge-can-cause-a-massive-loss-for-lenders) | Deprecating a gauge can cause a massive loss for lenders |



# [M-01] There is an inconsistency between borrower's debt ceiling and debtCeiling

## Impact [M-22](https://github.com/code-423n4/2023-12-ethereumcreditguild-findings/issues/308)
There is an inconsistency between how [borrow](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/loan/LendingTerm.sol#L394-L397) calculates the `debtCeiling` and how it is calculated in [debtCeiling](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/loan/LendingTerm.sol#L270).

## Proof of Concept

Currently, there is an inconsistency in the way [borrow](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/loan/LendingTerm.sol#L394-L397) calculates a smaller value for [debtCeiling](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/loan/LendingTerm.sol#L270) than the actual [debtCeiling](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/loan/LendingTerm.sol#L270) function. This renders [this check useless](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/tokens/GuildToken.sol#L225-L231) since borrow prevents issuance from going even close to the debt ceiling.

```solidity
uint256 debtCeilingAfterDecrement = LendingTerm(gauge).debtCeiling(-int256(weight));
require(issuance <= debtCeilingAfterDecrement, "GuildToken: debt ceiling used");
```

For the example, I am going to use the parameters below, and we will follow with the two calculations to get the results.

| **Prerequisites**      | **Values**                                |
|------------------------|------------------------------------------|
| Issuance               | 20,000                                   |
| Total borrowed credit  | 70,000 (5k when we call borrow / 65k old)|
| Total weight           | 100,000                                  |
| Gauge weight           | 50,000                                   |
| Gauge weight tolerance | 60% (1.2e18)                              |

[borrow](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/loan/LendingTerm.sol#L394-L397) calculates the debt `debtCeiling` with the following formula (found [here](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/loan/LendingTerm.sol#L383-L387)):

$$
\begin{align*}
\text{debtCeiling} &= \frac{{(\text{getGaugeWeight} \cdot (\text{totalBorrowedCredit} + \text{borrowAmount}))}}{{\text{totalWeight}}} \cdot \frac{{1.2e18}}{{1e18}} \\
&= \frac{{50000e18 \cdot (65000e18+5000e18)}}{{100000e18}} \cdot \frac{{1.2e18}}{{1e18}} \\
&= \frac{{50000e18 \cdot 70000e18}}{{100000e18}} \cdot \frac{{1.2e18}}{{1e18}} \\
&= \frac{{35000e18 \cdot 1.2e18}}{{1e18}} \\
&= 42000e18
\end{align*}
$$

At the same time, [debtCeiling](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/loan/LendingTerm.sol#L270) uses the complicated formula below, which we will break down for easier understanding.

```solidity
debtCeiling = ((((totalBorrowedCredit * (getGaugeWeight * 1.2e18)) / totalWeight) - issuance) * totalWeight / otherGaugesWeight) + issuance
```

Feel free to follow with the [code](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/loan/LendingTerm.sol#L270-L331). 

#### [toleratedGaugeWeight](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/loan/LendingTerm.sol#L305)

$$
\begin{align*}
\text{toleratedGaugeWeight} &= \frac{{\text{gaugeWeight} \cdot \text{gaugeWeightTolerance}}}{{1e18}} \\
&= \frac{{50000e18 \cdot 1.2e18}}{{1e18}} \\
&= 60000e18
\end{align*}
$$

#### [debtCeilingBefore](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/loan/LendingTerm.sol#L307-L308)

$$
\begin{align*}
\text{debtCeilingBefore} &= \frac{{\text{totalBorrowedCredit} \cdot \text{toleratedGaugeWeight}}}{{\text{totalWeight}}} \\
&= \frac{{70000e18 \cdot 60000e18}}{{100000e18}} \\
&= 42000e18
\end{align*}
$$

#### [remainingDebtCeiling](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/loan/LendingTerm.sol#L312)

$$
\begin{align*}
\text{remainingDebtCeiling} &= \text{debtCeilingBefore} - \text{issuance} \\
&= 42000e18 - 20000e18 \\
&= 22000e18
\end{align*}
$$

#### [otherGaugesWeight](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/loan/LendingTerm.sol#L319)

$$
\begin{align*}
\text{otherGaugesWeight} &= \text{totalWeight} - \text{toleratedGaugeWeight} \\
&= 100000e18 - 60000e18 \\
&= 40000e18
\end{align*}
$$

#### [maxBorrow](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/loan/LendingTerm.sol#L320-L321)

$$
\begin{align*}
\text{maxBorrow} &= \frac{{\text{remainingDebtCeiling} \cdot \text{totalWeight}}}{{\text{otherGaugesWeight}}} \\
&= \frac{{22000e18 \cdot 100000e18}}{{40000e18}} \\
&= 55000e18
\end{align*}
$$

#### [debtCeiling](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/loan/LendingTerm.sol#L322)

$$
\begin{align*}
\text{debtCeiling} &= \text{issuance} + \text{maxBorrow} \\
&= 55000e18 + 20000e18 \\
&= 75000e18
\end{align*}
$$

Finally, after the long pursuit, we have come to our answer. However, this answer (75k) differs from what we can max borrow (42k).

## Tools Used
Manual review.

## Recommended Mitigation Steps
Implement one function to calculate `debtCeiling`.


# [M-02] Gauge's weight will still take effect in markets where there is only one term

## Impact [M-18](https://github.com/code-423n4/2023-12-ethereumcreditguild-findings/issues/333)
Gauges weight will still take effect in markets where there is only one term. This is not desired, as described by the developers, if there is only one gauge, its weight won't matter.This will make voters for this gauge unable to decrease its weight, even though they should be able to do so.

> `debtCeiling` should not go below issuance because if there is just one term, then 100% of the borrows can happen there regardless of the weight (10 or 1e27, it doesn't matter, still 100%, so loans should be unrestricted).

## Proof of Concept
The function [_decrementGaugeWeight](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/tokens/GuildToken.sol#L207-L234), called by [decrementGauge](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/tokens/ERC20Gauges.sol#L301-L312), checks if the new decrease in gauge weight will lower the debt ceiling below the issuance.

```solidity
uint256 issuance = LendingTerm(gauge).issuance();
if (issuance != 0) {
    uint256 debtCeilingAfterDecrement = LendingTerm(gauge).debtCeiling(-int256(weight));
    require(issuance <= debtCeilingAfterDecrement, "GuildToken: debt ceiling used");
}
```

In [debtCeiling](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/loan/LendingTerm.sol#L274-L277), we get the gauge weight and add the `gaugeWeightDelta` (decrease the weight in this case).

```solidity
uint256 gaugeWeight = GuildToken(_guildToken).getGaugeWeight(address(this));
gaugeWeight = uint256(int256(gaugeWeight) + gaugeWeightDelta);
```

However, because `totalTypeWeight` is updated after [debtCeiling](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/loan/LendingTerm.sol#L274-L277) finishes execution, the math above makes it so 
`gaugeWeight != totalTypeWeight` even though there is only one gauge.

This, in turn, will pass by the first [else if](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/loan/LendingTerm.sol#L287-L292) (which is made for occasions where there is only one gauge) and continue execution below. It will most likely stop at the [if](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/loan/LendingTerm.sol#L309-L311) below, where it will return `debtCeilingBefore`, which will cause [_decrementGaugeWeight](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/tokens/GuildToken.sol#L207-L234) to revert. This is because `debtCeilingBefore` is already smaller than issuance.

```solidity
uint256 toleratedGaugeWeight = (gaugeWeight * gaugeWeightTolerance) / 1e18;
uint256 debtCeilingBefore = (totalBorrowedCredit * toleratedGaugeWeight) / totalWeight;
if (_issuance >= debtCeilingBefore) {
    return debtCeilingBefore;
}
```

The above is likely, as we have mentioned that `gaugeWeight != totalTypeWeight`, which will make this calculation push `totalBorrowedCredit` downwards (i.e., it will make it smaller since `totalWeight > toleratedGaugeWeight`).

```solidity
uint256 debtCeilingBefore = (totalBorrowedCredit * toleratedGaugeWeight) / totalWeight;
```

### Example:
| *Prerequisites*        | *Values*                        |
|------------------------|---------------------------------|
| Issuance               | 20,000                          |
| Total borrowed credit  | 21,000 (issuance + 1,000 interest) |
| Total weight           | 20,000                          |
| Gauge weight           | 20,000                          |
| Gauge weight tolerance | 60%                             |
| Current reduction      | 5,000                           |

With the example above, our current user is trying to unstake 5,000 weight from the gauge. As it's the only gauge in the market, [debtCeiling](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/loan/LendingTerm.sol#L291) should return `hardCap` or `creditMinterBuffer` (whichever is the smaller one). Let's check the math and find out.

1. [gaugeWeight](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/loan/LendingTerm.sol#L274-L277) will be calculated as 15,000.

$$ \text{gaugeWeight} = \text{gaugeWeight} - \text{gaugeWeightDelta} = 20000e18 - 5000e18 = 15000e18 $$


2. `totalWeight` is 20,000 as it's reduced in [decrementGauge](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/tokens/ERC20Gauges.sol#L307-L310) after [_decrementGaugeWeight](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/tokens/GuildToken.sol#L207-L234) finishes.

3. This will skip [else if](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/loan/LendingTerm.sol#L287-L292) since `gaugeWeight != totalWeight`.

4. [toleratedGaugeWeight](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/loan/LendingTerm.sol#L305-L306) will be 18,000.

$$ \text{toleratedGaugeWeight} = \frac{\text{gaugeWeight} \times \text{gaugeWeightTolerance}}{1e18} = \frac{15000e18 \times 1.2e18}{1e18} = 18000e18 $$

5. This will make [debtCeilingBefore](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/loan/LendingTerm.sol#L307-L308) be 18,900.

$$ \text{debtCeilingBefore} = \frac{\text{totalBorrowedCredit} \times \text{toleratedGaugeWeight}}{\text{totalWeight}} = \frac{21000e18 \times 18000e18}{20000e18} = 18900e18 $$

6. We will enter the [if](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/loan/LendingTerm.sol#L309-L311) below where it will return `debtCeilingBefore`.

```solidity
if (_issuance >= debtCeilingBefore) {
    return debtCeilingBefore;
}
```

7. [_decrementGaugeWeight](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/loan/LendingTerm.sol#L309-L311) will revert as `debtCeilingBefore < issuance`.

```solidity
uint256 issuance = LendingTerm(gauge).issuance();
if (issuance != 0) {
    uint256 debtCeilingAfterDecrement = LendingTerm(gauge).debtCeiling(-int256(weight));
    require(issuance <= debtCeilingAfterDecrement, "GuildToken: debt ceiling used");
}
```

Note that, of course, the user can make 5 separate TX and withdraw those 5k, 1k at a time (this is possible as `gaugeWeight * 1.2 > totalWeight` and entering this [if](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/loan/LendingTerm.sol#L313-L318)). However, it will be unnecessary, and the solution I have recommended will fix the issue without asking the user to redo his TX.

## Tools Used
Manual review.

## Recommended Mitigation Steps
Add a check [here](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/loan/LendingTerm.sol#L274-L281), before updating the gauge weight, to see if there is only one gauge.

```diff
        uint256 gaugeWeight = GuildToken(_guildToken).getGaugeWeight(address(this));
+      uint256 totalWeight = GuildToken(_guildToken).totalTypeWeight(gaugeType);
+      if (gaugeWeight == totalWeight) {
+           return _hardCap < creditMinterBuffer ? _hardCap : creditMinterBuffer;
+      }
        gaugeWeight = uint256(int256(gaugeWeight) + gaugeWeightDelta);
        uint256 gaugeType = GuildToken(_guildToken).gaugeType(address(this));
-       uint256 totalWeight = GuildToken(_guildToken).totalTypeWeight(gaugeType);
```



# [M-03] AuctionHouse Whales can win large auctions via block stuffing

## Impact [M-16](https://github.com/code-423n4/2023-12-ethereumcreditguild-findings/issues/463)

Whales (accounts with significant cryptocurrency holdings) can exploit the current auction mechanics to guarantee profits through block stuffing.

This is particularly feasible due to the short duration of auctions, allowing these entities to manipulate auction outcomes in their favor.

## Proof of Concept

The protocol uses a Dutch auction format with two phases:

    First Phase: Bidders must pay the full debt amount. The collateral percentage starts at 0% and increases with each new block until the auction's midpoint.
    Second Phase: The protocol offers the full collateral and decreases the owed debt by a percentage in each new block, reaching 0% at auction's end. This implies that a bidder could eventually receive the collateral for free.

Bidders are disincentivized to participate in the first phase, as it generally results in a net loss unless there are force majeure market conditions.

Whales can use a block stuffing attack to win large auctions and acquire collateral at significantly reduced prices.

Example scenario:

    Alice has the following bad debt which is auctioned off:
        Collateral: 2,000,000 USDC
        Debt: 1,000 WETH (1 WETH = 2,000 USDC)
    There are no bidders in the first phase of the auction since this would result in a loss for the bidder.
    The auction reaches its midpoint. Collateral cost reduces by ~1.14% every block. This is because the second phase is 1150 sec. (as per the GIP_0.sol deployment script), i.e. 88 blocks on Mainnet. The decay rate is thus 100% / 88 ~ 1.14%
    At the auction's midpoint, Bob executes block stuffing attack. To make sure his attack would succeed, he uses a gas price of 250 Gwei.
    After 88 blocks, Bob binds in the final block and wins 2,000,000 USDC at 0 ETH cost.

The attack cost is:
$$
88\ blocks \times 30M\ gas\ \times 250\ Gwei = 66,000,000,000\ Gwei = 660\ ETH ~= 1.5M\ USDC
$$

Thus, Bob has made a profit of 500,000 USDC.

This strategy, while requiring substantial funds, is a feasible and potentially lucrative attack vector.

The severity is set to Medium given its low likelihood but high impact.

## Tools Used

Manual review

## Recommended Mitigation Steps

To mitigate this vulnerability, it is recommended to extend the auction duration. Longer auctions would increase the cost and complexity of block stuffing attacks, reducing the likelihood of such exploits.



# [M-04] Borrowers can reduce the debt ceiling bellow the issuance

## Impact [M-15](https://github.com/code-423n4/2023-12-ethereumcreditguild-findings/issues/135)

The [debtCeiling](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/loan/LendingTerm.sol#L270) function may return incorrect values. The vulnerability surpasses [this requirement](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/tokens/GuildToken.sol#L225-L231), potentially causing borrowers to reduce the debt ceiling below the issuance.

```solidity
if (issuance != 0) {
    uint256 debtCeilingAfterDecrement = LendingTerm(gauge).debtCeiling(-int256(weight));
    require(
        issuance <= debtCeilingAfterDecrement,
        "GuildToken: debt ceiling exceeded"
    );
}
```

## Proof of Concept

The [debtCeiling](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/loan/LendingTerm.sol#L270) final minimum check may be inaccurate, as it doesn't return the actual minimum value. If `_hardCap` < `creditMinterBuffer`, it will still return `creditMinterBuffer` because `creditMinterBuffer` is compared first to `_debtCeiling`.

```solidity
if (creditMinterBuffer < _debtCeiling && creditMinterBuffer < _hardCap) {
    return creditMinterBuffer;
}
if (_hardCap < _debtCeiling) {
    return _hardCap;
}
return _debtCeiling;
```

**Example:**

| *Prerequisites*        | *Values*                     |
|------------------------|------------------------------|
| Hard cap               | 70k                          |
| Issuance               | 70k                          |
| Total borrowed credit  | 100k                         |
| Gauges                 | 80% / 20% - 80k / 20k weight |
| Total weight           | 100k                         |
| Gauge weight tolerance | 60%                          |
| Credit minter buffer   | 100k                         |

With the current parameters, [debtCeiling](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/loan/LendingTerm.sol#L270) will return 100,000 instead of 70,000 (which is the `hardCap`). These parameters are not uncommon, as they are expected for small markets with not much use. A big `buffer` suggests that loans are rarely taken and a small `hardcap` indicates volatility.

Below is the math we need to do to reach the final return value. You can follow along with the code.

#### [toleratedGaugeWeight](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/loan/LendingTerm.sol#L305-L306)

$$
\text{toleratedGaugeWeight} = \frac{\text{gaugeWeight} \times \text{gaugeWeightTolerance}}{1 \times 10^{18}} = \frac{80000 \times 10^{18} \times 1.2 \times 10^{18}}{1 \times 10^{18}} = 96000 \times 10^{18}
$$

#### [debtCeilingBefore](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/loan/LendingTerm.sol#L307-L308)

$$
\text{debtCeilingBefore} = \frac{\text{totalBorrowedCredit} \times \text{toleratedGaugeWeight}}{\text{totalWeight}} = \frac{100000 \times 10^{18} \times 96000 \times 10^{18}}{100000 \times 10^{18}} = 96000 \times 10^{18}
$$

#### [remainingDebtCeiling](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/loan/LendingTerm.sol#L312)

$$
\text{remainingDebtCeiling} = \text{debtCeilingBefore} - \text{issuance} = 96000 \times 10^{18} - 70000 \times 10^{18} = 26000 \times 10^{18}
$$

#### [otherGaugesWeight](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/loan/LendingTerm.sol#L319)

$$
\text{otherGaugesWeight} = \text{totalWeight} - \text{toleratedGaugeWeight} = 100000 \times 10^{18} - 96000 \times 10^{18} = 4000 \times 10^{18}
$$

#### [maxBorrow](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/loan/LendingTerm.sol#L320-L321)

$$
\text{maxBorrow} = \frac{\text{remainingDebtCeiling} \times \text{totalWeight}}{\text{otherGaugesWeight}} = \frac{26000 \times 10^{18} \times 100000 \times 10^{18}}{4000 \times 10^{18}} = 650000 \times 10^{18}
$$

#### [debtCeiling](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/loan/LendingTerm.sol#L322)

$$
\text{debtCeiling} = \text{issuance} + \text{maxBorrow} = 70000 \times 10^{18} + 650000 \times 10^{18} = 720000 \times 10^{18}
$$

```solidity
function debtCeiling(int256 gaugeWeightDelta) public view returns (uint256) {
    ...
    if (creditMinterBuffer < _debtCeiling && creditMinterBuffer < _hardCap) {
        return creditMinterBuffer;
    }
    if (_hardCap < _debtCeiling) {
        return _hardCap;
    }
    return _debtCeiling;
}
```

## Tools Used

Manual review

## Recommended Mitigation Steps

Improve the final [min check](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/loan/LendingTerm.sol#L324-L326).

```diff
function debtCeiling(int256 gaugeWeightDelta) public view returns (uint256) {
    ...
-   if (creditMinterBuffer < _debtCeiling) {
+   if (creditMinterBuffer < _debtCeiling && creditMinterBuffer < _hardCap) {
        return creditMinterBuffer;
    }
    if (_hardCap < _debtCeiling) {
        return _hardCap;
    }
    return _debtCeiling;
}
```



# [M-05] Liquidators can extract extra value with flash loans, significantly reducing profits for other users

## Impact [M-11](https://github.com/code-423n4/2023-12-ethereumcreditguild-findings/issues/144)

Liquidators will be able to flash loan [mint](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/loan/SimplePSM.sol#L103-L112) and [stake](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/loan/SurplusGuildMinter.sol#L114-L155) before liquidating the borrower, extracting maximal potential value. While advantageous for liquidators, this significantly reduces gauge stakers' profits without changing the associated risks.

## Proof of Concept

To extract the maximal possible value, bidders (liquidators) will [mint](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/loan/SimplePSM.sol#L103-L112) with PSM and [stake](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/loan/SurplusGuildMinter.sol#L114-L155) in the gauge they are liquidating. This is because, upon liquidation, [onBid](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/loan/LendingTerm.sol#L789-L798) calls **ProfitManager**'s [notifyPnL](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/governance/ProfitManager.sol#L342-L405), distributing part of the interest to gauge voters. This process is achievable in a single transaction, incentivizing liquidators to do it for every profitable (positive PnL) liquidation.

Example:

| *Prerequisites*                  | *Values*        |
|----------------------------------|-----------------|
| Borrower coll                    | 10,000 USDC     |
| Borrower loan                    | 7,000 USDC      |
| Borrower fees (start + interest) | 1,000 USDC      |
| Gauge weight                     | 10,000          |
| PM split - buffer/credit/gauge   | 20% / 40% / 40% |

1. Borrower misses a payment.
2. [Call](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/loan/LendingTerm.sol#L678-L680) triggers the auction, reaching a profitable point of 8,000 USDC credit for 8,100 USDC collateral.
3. Alice executes a Flash loan transaction:
   - Flash loan 90,000 USDC
   - Mint 90,000 gUSDC
   - Stake 90,000 gUSDC into the gauge
   - Bid on Bob's loan.
   - Call SGM [getRewards](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/loan/SurplusGuildMinter.sol#L216-L290)
   - Unstake 90,000 gUSDC
   - Redeem 90,360 gUSDC
   - Pay the loan

After Alice bids on Bob's loan, [calculations](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/loan/LendingTerm.sol#L751-L768) are performed, and **ProfitManager**'s [notifyPnL](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/governance/ProfitManager.sol#L292) is called with 1,000 USDC to split. PM allocates 400 USDC to the gauge. However, Alice holds 90k out of 100k weight (90%), entitling her to 90% of the gauge's profit (360 USDC).

Alice profits 360 USDC from the FL (460 USDC in total) + the gauge tokens that SGM mints as [rewardsRatio](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/loan/SurplusGuildMinter.sol#L252-L261) (360 with `rewardRatio` of 1), while the remaining gauge stakers split the remaining 40 USDC. This scenario disincentivizes staking for a given gauge, as liquidation becomes a safer and more profitable alternative.


### POC

Gist - https://gist.github.com/0x3b33/cf4349253c7762ab4c3d099ecadbea95
Run it with `forge test --match-test test_flashLoanExtraProfit


## Tools Used

Manual review

## Recommended Mitigation Steps

Implementing a dripping mechanism similar to that used with credit tokens ([here](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/tokens/ERC20RebaseDistributor.sol#L168)) may be the most effective solution, albeit making gauges more complex. Alternatively, pausing [mint](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/loan/SimplePSM.sol#L103-L112) could be considered, but this might only make it more challenging as liquidators can still use flash loans to acquire credits through other means.



# [M-06] Deprecating a gauge can cause a massive loss for lenders

## Impact [M-17](https://github.com/code-423n4/2023-12-ethereumcreditguild-findings/issues/161)
If market conditions change, some markets might consider deprecating specific gauges. However, this action could trigger a bank run on the gauge, leading to permanent losses for some lenders, as their credit tokens would be slashed if a borrower leaves with bad debt.

## Proof of Concept
When [offBoarding](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/governance/LendingTermOffboarding.sol#L153-L170) a gauge, PSM is paused, preventing the so-called "bank runs." Nevertheless, these bank runs are still likely to occur in the case of the removed gauge, where voters will attempt to exit before any bad debt accrues, otherwise, they face potential slashing. This will be feasible for some but not for all, as voters cannot decrease the gauge weight below its issuance. This limitation is enforced by the **GuildToken**'s [_decrementGaugeWeight](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/tokens/GuildToken.sol#L224-L231), which checks if the issuance of the term is below the [debtCeiling](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/loan/LendingTerm.sol#L270).

```solidity
uint256 issuance = LendingTerm(gauge).issuance();
if (issuance != 0) {
    uint256 debtCeilingAfterDecrement = LendingTerm(gauge).debtCeiling(-int256(weight));
    require(
        issuance <= debtCeilingAfterDecrement,
        "GuildToken: debt ceiling used"
    );
}
```

If there are still existing borrowers (whose auctions haven't finished), lenders are not allowed to leave. If one of these borrowers causes bad debt, all lenders will be slashed. The pausing mechanism that aims to stop these bank runs is not working correctly, as they are still likely to happen. In this scenario, the first lenders will win (avoiding credit slashing), while the last will lose (being slashed).

Example:

1. GaugeA is deprecated and every borrower is [_called](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/loan/LendingTerm.sol#L683-L688).
2. Alice calls [decrementGauge](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/tokens/ERC20Gauges.sol#L301-L312) in order to leave the gauge.
3. Bob front-runs Alice and leaves before her.
4. When Alice TX executes [_decrementGaugeWeight](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/tokens/GuildToken.sol#L224-L231) reverts as she is trying to lower the weight bellow the issuance.
5. Eve auction ends and she accrues bad debt for the gauge.
6. Alice gets slashed.

In this example Alice was fast enough to leave, but due to the [issuance check](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/tokens/GuildToken.sol#L224-L231) she still gets slashed. 

## Tools Used
Manual review.

## Recommended Mitigation Steps
One option is to keep PSM paused and skip [this if](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/tokens/GuildToken.sol#L224-L231), if a gauge is removed. This approach will halt the bank run from the gauge and still use the `creditMultiplier` as a tool for splitting bad debt.

```diff
+  if (isGauge(gauge)) { // This will stop deprecated gauges from entering
     uint256 issuance = LendingTerm(gauge).issuance();
     if (issuance != 0) {
          uint256 debtCeilingAfterDecrement = LendingTerm(gauge).debtCeiling(-int256(weight));
          require(
             issuance <= debtCeilingAfterDecrement,
             "GuildToken: debt ceiling used"
          );
     }
+  }
```


