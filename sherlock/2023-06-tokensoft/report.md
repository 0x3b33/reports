| Severity | Title | 
|:--:|:---|
| [H-01](#h-01-anyone-on-the-merkle-tree-can-spam-initializedistributionrecord-and-get-himself-any-number-of-votetokens) | Anyone on the merkle tree can spam `initializeDistributionRecord` and get himself any number of voteTokens |
| [M-01](#m-01-_settotal-will-not-work) | `_setTotal` will not work |



# [H-01] Anyone on the merkle tree can spam initializeDistributionRecord and get himself any number of voteTokens

## Summary
Anyone on the merkle tree can spam `initializeDistributionRecord` and get himself any number of voteTokens

## Vulnerability Detail
Due to the poor checks on [AdvancedDistributor](https://github.com/sherlock-audit/2023-06-tokensoft/blob/main/contracts/contracts/claim/abstract/AdvancedDistributor.sol#L77-L85) anyone on the merkle tree can just call `initializeDistributionRecord` on most of the distributors and get himself minted some vote tokens.

Example is [TrancheVestingMerkle](https://github.com/sherlock-audit/2023-06-tokensoft/blob/main/contracts/contracts/claim/TrancheVestingMerkle.sol) where any user that is on the merkle tree can call [initializeDistributionRecord](https://github.com/sherlock-audit/2023-06-tokensoft/blob/main/contracts/contracts/claim/TrancheVestingMerkle.sol#L39-L49)

```jsx
  function initializeDistributionRecord(
    uint256 index, // the beneficiary's index in the merkle root
    address beneficiary, // the address that will receive tokens
    uint256 amount, // the total claimable by this beneficiary
    bytes32[] calldata merkleProof
  )
    external
    validMerkleProof(keccak256(abi.encodePacked(index, beneficiary, amount)), merkleProof)
  {
    _initializeDistributionRecord(beneficiary, amount);
  }
  ```
  From then on [initializeDistributionRecord](https://github.com/sherlock-audit/2023-06-tokensoft/blob/main/contracts/contracts/claim/abstract/AdvancedDistributor.sol#L85C1-L85C1) will forward the call to [AdvancedDistributor](https://github.com/sherlock-audit/2023-06-tokensoft/blob/main/contracts/contracts/claim/abstract/AdvancedDistributor.sol) where the man issue is. `_initializeDistributionRecord` calls super on the [Distributor](https://github.com/sherlock-audit/2023-06-tokensoft/blob/main/contracts/contracts/claim/abstract/Distributor.sol)'s  [_initializeDistributionRecord](https://github.com/sherlock-audit/2023-06-tokensoft/blob/main/contracts/contracts/claim/abstract/Distributor.sol#L47-L59) where nothing is updated, since the value is the same, and them **AdvancedDistributor**'s [initializeDistributionRecord](https://github.com/sherlock-audit/2023-06-tokensoft/blob/main/contracts/contracts/claim/abstract/AdvancedDistributor.sol#L77-L85) mints the vote tokens, without having the sanity check for if there is any update on to the user reward token balance.
  
  ```jsx
    function _initializeDistributionRecord(
    address beneficiary,
    uint256 totalAmount
  ) internal virtual override {
    super._initializeDistributionRecord(beneficiary, totalAmount);
    _mint(beneficiary, tokensToVotes(totalAmount));//@audit always minting on input
  }
```

| Vulnerable contracts                                                                                                                                               |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| [TrancheVestingMerkle](https://github.com/sherlock-audit/2023-06-tokensoft/blob/main/contracts/contracts/claim/TrancheVestingMerkle.sol)                           |
| [PriceTierVestingMerkle](https://github.com/sherlock-audit/2023-06-tokensoft/blob/main/contracts/contracts/claim/PriceTierVestingMerkle.sol)                       |
| [ContinuousVestingMerkle](https://github.com/sherlock-audit/2023-06-tokensoft/blob/main/contracts/contracts/claim/ContinuousVestingMerkle.sol)                     |
| [CrosschainTrancheVestingMerkle](https://github.com/sherlock-audit/2023-06-tokensoft/blob/main/contracts/contracts/claim/CrosschainTrancheVestingMerkle.sol)       |
| [CrosschainContinuousVestingMerkle](https://github.com/sherlock-audit/2023-06-tokensoft/blob/main/contracts/contracts/claim/CrosschainContinuousVestingMerkle.sol) |                      |


<details> <summary>  PoC </summary>

Place it in [TrancheVestingMerkle.test.ts](https://github.com/sherlock-audit/2023-06-tokensoft/blob/main/contracts/test/distributor/TrancheVestingMerkle.test.ts).
```jsx
  it.only("A buyer can initialize before claimingXXX", async () => {
    const user = eligible2
    const distributor = partiallyVestedDistributor
    const [index, beneficiary, amount] = config.proof.claims[user.address].data.map(d => d.value)
    const proof = config.proof.claims[user.address].proof

    // 50% of tokens have already vested
    const claimable = BigInt(amount) / 2n;
    for(let i = 0;i<5;i++){
      await distributor.initializeDistributionRecord(index, beneficiary, amount, proof)
    }

    let distributionRecord = await distributor.getDistributionRecord(user.address)

    expect(distributionRecord.total.toBigInt()).toEqual(
      BigInt(config.proof.claims[user.address].data[2].value)
    )
    // no votes prior to delegation
    expect((await distributor.getVotes(user.address)).toBigInt()).toEqual(0n)

    // delegate to self
    const myDistributor = await ethers.getContractAt("TrancheVestingMerkle", distributor.address, user);
    await myDistributor.delegate(user.address)

    expect(distributionRecord.total.toBigInt()).toEqual(
      BigInt(config.proof.claims[user.address].data[2].value)
    )

    expect(distributionRecord.initialized).toEqual(true)
    console.log("User vote token Balance: " + (await distributor.balanceOf(user.address)).toBigInt());
    console.log("Tokens that have vested: " + (claimable));
  })
  ```
</details>
   
## Impact
Anyone can mint unlimited number or vote tokens.
## Code Snippet
https://github.com/sherlock-audit/2023-06-tokensoft/blob/main/contracts/contracts/claim/abstract/AdvancedDistributor.sol#L77-L85
## Tool used

Manual Review

## Recommendation
You can make the external callable functions have an initialized modifier (or some prevention from multiple calls) or have a sanity check in the **AdvancedDistributor**



# [M-01] `_setTotal` will not work

## Summary
The `_setTotal` function in [CrosschainDistributor](https://github.com/sherlock-audit/2023-06-tokensoft/blob/main/contracts/contracts/claim/abstract/CrosschainDistributor.sol), [CrosschainContinuousVestingMerkle](https://github.com/sherlock-audit/2023-06-tokensoft/blob/main/contracts/contracts/claim/CrosschainContinuousVestingMerkle.sol) and [CrosschainTrancheVestingMerkle](https://github.com/sherlock-audit/2023-06-tokensoft/blob/main/contracts/contracts/claim/CrosschainTrancheVestingMerkle.sol) will not work.
## Vulnerability Detail
`_setTotal` will simply not work, because is calls [_allowConnext](https://github.com/sherlock-audit/2023-06-tokensoft/blob/main/contracts/contracts/claim/abstract/CrosschainDistributor.sol#L35-L37) which in tern calls ` token.safeApprove`.
```jsx
  function _allowConnext(uint256 amount) internal {
    token.safeApprove(address(connext), amount);
  }
```
Firstly `safeApprove` is deprecated and secondly if there is any left approved amount it will revert.
```jsx
    function safeApprove(
        IERC20 token,
        address spender,
        uint256 value
    ) internal {
        require(
            (value == 0) || (token.allowance(address(this), spender) == 0),
            "SafeERC20: approve from non-zero to non-zero allowance"
        );
        _callOptionalReturn(token, abi.encodeWithSelector(token.approve.selector, spender, value));
```

**Example scenario:**

1. Users are given a total rewards and some claimed and others not. With every claim total approved amount is lowered.
2. `setTotal` is called, but it inevitable reverts since some users have not claimed their rewards and may never claim them.
Thus setting of new Total is impossible.
## Impact

## Code Snippet

## Tool used

Manual Review

## Recommendation
It's recommended to use approve (safeApprove is not safe) or use decrease/increase allowance. 