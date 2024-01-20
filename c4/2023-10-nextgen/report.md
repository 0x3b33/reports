| Severity | Title |
|:--------:|:------|
| [H-01](#h-01-multiple-mints-can-brick-any-form-of-salesoption-3-mintings) | Multiple mints can brick any form of salesOption 3 mintings |
| [H-02](#h-02-winner-and-bidders-can-double-refund-their-bids) | Winner and bidders can double refund their bids |
| [H-03](#h-03-users-can-re-enter-with-mint-in-order-to-mint-as-many-nfts-as-they-want) | Users can re-enter with mint in order to mint as many NFTs as they want |
| [M-01](#m-01-bidder-can-accidentally-bid-after-the-auction-has-ended-leaving-his-bid-stuck-in-the-contract) | Bidder can accidentally bid after the auction has ended, leaving his bid stuck in the contract |
| [M-02](#m-02-getprice-salesoption-2-can-round-down-to-the-lower-barrier-skipping-the-last-time-period) | `getPrice` `salesOption` 2 can round down to the lower barrier, skipping the last time period |
| [M-03](#m-03-if-an-airdrop-happens-before-a-mint-the-price-could-skyrocket) | If an airdrop happens before a mint, the price could skyrocket |
| [M-04](#m-04-the-winner-can-dos-the-auction-and-stuck-the-nft-and-users-funds-in-the-contract) | The winner can DoS the auction and stuck the NFT and users funds in the contract |



# [H-01] Multiple mints can brick any form of salesOption 3 mintings

## Impact
As explained by the sponsor, some collections might want to conduct multiple mints on different days. However, due to the way `salesOption` 3 works, these multiple mints might encounter issues.

## Proof of Concept
A collection has completed its first mint, where it minted 500 NFTs. However, the collection consists of 1000 NFTs, so the owner plans to schedule another mint, this time using sales option 3.

| Values              |                          |
|---------------------|--------------------------|
| `allowlistStartTime`| 4 PM                     |
| `allowlistEndTime`  | 7 PM                     |
| `publicStartTime`   | 7 PM                     |
| `publicEndTime`     | 1 day after public start |
| `timePeriod`        | 1 min                    |

The first user's mint will proceed smoothly since `timeOfLastMint` falls within the previous mint period. However, the second user's mint will fail. The same applies to all other whitelisted users. This issue arises due to the [following block](https://github.com/code-423n4/2023-10-nextgen/blob/main/smart-contracts/MinterContract.sol#L252):

```solidity
lastMintDate[col] = collectionPhases[col].allowlistStartTime
                + (collectionPhases[col].timePeriod * (gencore.viewCirSupply(col) - 1));
```

This [calculation](https://github.com/code-423n4/2023-10-nextgen/blob/main/smart-contracts/MinterContract.sol#L252) extends the allowed time significantly, granting the second minter an allowed time of `allowlistStartTime + 1 min * (500-1) = allowlistStartTime + 499 min`, which is equivalent to 8 hours and 19 minutes after `allowlistStartTime`. This enables the second user to mint at **12:19 AM**, long after the whitelist has ended and in the middle of the public sale. And if anyone tries to mint his call will revert with `underflow` error, as `timeOfLastMint` > `block.timestamp`.

```solidity
uint256 tDiff = (block.timestamp - timeOfLastMint) / collectionPhases[col].timePeriod;
```

It's worth noting that some collections may disrupt the whitelist, while others could brick the entire mint process, especially if there are more minted NFTs or a longer minting period.

## POC
Gits - https://gist.github.com/0x3b33/677f86f30603dfa213541cf764bbc0e8
Add to remappings - `contracts/=smart-contracts/`
Run it with `forge test --match-test test_multipleMints --lib-paths ../smart-contracts`

## Tools Used
Manual review.

## Recommended Mitigation Steps
For this fix I am unable to give any suggestion as big parts of the protocol need to be re-done. I can only point out the root cause of the problem, which is `(gencore.viewCirSupply(col) - 1)` in the [snippet below](https://github.com/code-423n4/2023-10-nextgen/blob/main/smart-contracts/MinterContract.sol#L252).

```solidity
lastMintDate[col] = collectionPhases[col].allowlistStartTime + (collectionPhases[col].timePeriod * (gencore.viewCirSupply(col) - 1));
```


# [H-02] Winner and bidders can double refunds their bids

## Impact
Due to [claimAuction](https://github.com/code-423n4/2023-10-nextgen/blob/main/smart-contracts/AuctionDemo.sol#L104-L120) and [cancelBid](https://github.com/code-423n4/2023-10-nextgen/blob/main/smart-contracts/AuctionDemo.sol#L124-L130) both using `=` in their time equations, bidders will be able to double refunds their bids.
 
```solidity
function claimAuction(...) public WinnerOrAdminRequired(_tokenid,this.claimAuction.selector){
require(block.timestamp >= minter.getAuctionEndTime(_tokenid)...);

function cancelBid(...) public {...
require(block.timestamp <= minter.getAuctionEndTime(_tokenid)...);
```


## Proof of Concept
The winner is able to [claim the auction](https://github.com/code-423n4/2023-10-nextgen/blob/main/smart-contracts/AuctionDemo.sol#L105) at it's **exact auction end time**.
```solidity
function claimAuction(...) public WinnerOrAdminRequired(_tokenid,this.claimAuction.selector){
require(block.timestamp >= minter.getAuctionEndTime(_tokenid)...);
```
However bidders are also able [to cancel their bids](https://github.com/code-423n4/2023-10-nextgen/blob/main/smart-contracts/AuctionDemo.sol#L125) at **exact auction end time**.
```solidity
function cancelBid(...) public {...
require(block.timestamp <= minter.getAuctionEndTime(_tokenid)...);
```

This means that the winner can claim, and him or some bidder to double refunds their bids. This is done by first using the refund in [claimAuction](https://github.com/code-423n4/2023-10-nextgen/blob/main/smart-contracts/AuctionDemo.sol#L115-L117) and then canceling his bid with [cancelBid](https://github.com/code-423n4/2023-10-nextgen/blob/main/smart-contracts/AuctionDemo.sol#L124-L130).

A few example scenarios occur, where in order to do this you must have some failed (not winning) bids:

1. The winner can make multiple bids and either re-enter [here](https://github.com/code-423n4/2023-10-nextgen/blob/main/smart-contracts/AuctionDemo.sol#L116) after every not winning bid and refund it, or simply schedule in one TX [claimAuction](https://github.com/code-423n4/2023-10-nextgen/blob/main/smart-contracts/AuctionDemo.sol#L104-L120) and [cancelBid](https://github.com/code-423n4/2023-10-nextgen/blob/main/smart-contracts/AuctionDemo.sol#L124-L130) on all his not winning bids.

2. Users can re-enter their failed bids and refunds them.

3. Seller of the NFT can re-enter and claim his failed bids.

## Tools Used
Manual review 

## Recommended Mitigation Steps
Even if the re-entracy is fixed the winner would still be able to schedule [claimAuction](https://github.com/code-423n4/2023-10-nextgen/blob/main/smart-contracts/AuctionDemo.sol#L104-L120) and [cancelBid](https://github.com/code-423n4/2023-10-nextgen/blob/main/smart-contracts/AuctionDemo.sol#L124-L130) in one TX and double refund his other bids. The main issue is `<=` which needs to be made `<` in [cancelBid](https://github.com/code-423n4/2023-10-nextgen/blob/main/smart-contracts/AuctionDemo.sol#L124-L130).



# [H-03] Users can re-enter with mint in order to mint as many NFTs as they want

## Impact
Whitelisted users can mint before the public mint starts, but they are restricted in the number of NFTs they can mint. However, they will be able to re-enter the [mint]() function and mint as many NFTs as they want.

## Proof of Concept
When the mint function is called, users need to verify their order with the Merkle tree by inputting their correct `_maxAllowance` and `tokData`. After they are verified, [gencore.mint](https://github.com/code-423n4/2023-10-nextgen/blob/main/smart-contracts/MinterContract.sol#L236) is called to mint their tokens. However, before increasing `tokensMintedAllowlistAddress`, [_mintProcessing](https://github.com/code-423n4/2023-10-nextgen/blob/main/smart-contracts/NextGenCore.sol#L193-L196) is called.

```solidity
            _mintProcessing(mintIndex, _mintTo, _tokenData, _collectionID, _saltfun_o);
            if (phase == 1) {
                tokensMintedAllowlistAddress[_collectionID][_mintingAddress] += 1;
            }
```
This, in turn, mints the user their NFT.

```solidity
    function _mintProcessing(...) internal {
        tokenData[_mintIndex] = _tokenData;
        collectionAdditionalData[_collectionID].randomizer.calculateTokenHash(_collectionID, _mintIndex, _saltfun_o);
        tokenIdsToCollectionIds[_mintIndex] = _collectionID;
        _safeMint(_recipient, _mintIndex);
    }
```
Because it uses `safeMint`, users are called by the 721 Contract with [_checkOnERC721Received](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC721/ERC721.sol#L312-L315). This call can be caught in the attacker's contract, and from there, reentrancy begins. After the mint is done, the attacker can call `MinterContract.mint` again, and the requirement that checks their max mint limit will pass. 

```solidity
    } else {
         node = keccak256(abi.encodePacked(msg.sender, _maxAllowance, tokData));
         //@audit if we re-enter `retrieveTokensMintedALPerAddress` will return the not yet updated value
         require(_maxAllowance >= gencore.retrieveTokensMintedALPerAddress(col, msg.sender) + _numberOfTokens, "AL limit");
         mintingAddress = msg.sender;
    }
```
This is because we re-entered before `tokensMintedAllowlistAddress` had the chance to increase, which means that [gencore.retrieveTokensMintedALPerAddress](https://github.com/code-423n4/2023-10-nextgen/blob/main/smart-contracts/MinterContract.sol#L217) still has the old value.

Due to this reentrancy, whitelisted users can mint more NFTs than they should. This will also have an effect on phase 3 sales, enabling users to mint NFTs before [the price has had a chance to update](https://github.com/code-423n4/2023-10-nextgen/blob/main/smart-contracts/MinterContract.sol#L240-L254). Overall, this breaks the concept of users being restricted in the number of NFTs they can mint.

## Tools Used
Manual review.

## Recommended Mitigation Steps
Add [nonReentrant](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/utils/ReentrancyGuard.sol#L55-L59) in the minting functions.



# [M-01] Bidder can acidentally bid after the auction has ended, leaving his bid stuck in the contract

## Impact
[claimAuction](https://github.com/code-423n4/2023-10-nextgen/blob/main/smart-contracts/AuctionDemo.sol#L104-L120) and [participateToAuction](https://github.com/code-423n4/2023-10-nextgen/blob/main/smart-contracts/AuctionDemo.sol#L58) both use `<=`, which can result in bidders accidentally bidding immediately after the auction ends. This can lead to their bids becoming stuck in the contract.

```solidity
function participateToAuction(...) public {...
require(block.timestamp <= minter.getAuctionEndTime(_tokenid)...);

function claimAuction(...) public WinnerOrAdminRequired(_tokenid,this.claimAuction.selector){
require(block.timestamp >= minter.getAuctionEndTime(_tokenid)...);
```

## Proof of Concept
Shortly after the winner of an auction ends it by calling [claimAuction](https://github.com/code-423n4/2023-10-nextgen/blob/main/smart-contracts/AuctionDemo.sol#L104-L120), a bidder's transaction can execute and bid for the just-ended auction. This is possible because bidders can bid at the exact closing time, which should not be possible.

```solidity
function participateToAuction(uint256 _tokenid) public payable {
    require(.. && block.timestamp <= minter.getAuctionEndTime(_tokenid) && ...);
```

Example scenario:
1. Bob sends a bid a few seconds before the closing time.
2. The bid transaction remains in the mempool.
3. The winner closes the auction.
4. Right after [claimAuction](https://github.com/code-423n4/2023-10-nextgen/blob/main/smart-contracts/AuctionDemo.sol#L104-L120), Bob's [participateToAuction](https://github.com/code-423n4/2023-10-nextgen/blob/main/smart-contracts/AuctionDemo.sol#L57-L61) gets executed.

The winner (which should have been the last bidder - Bob, but it's the guy before him) receives his NFT, and all of the users get refunded on their bids, except for Bob. His bid remains stuck in the contract.

## Tools Used
Manual review

## Recommended Mitigation Steps
Change from `<=` to `<` in [participateToAuction](https://github.com/code-423n4/2023-10-nextgen/blob/main/smart-contracts/AuctionDemo.sol#L57-L61) to make it impossible for bidders to bid after an auction has ended.

```diff
function participateToAuction(uint256 _tokenid) public payable {
-   require(msg.value > returnHighestBid(_tokenid) && block.timestamp <= minter.getAuctionEndTime(_tokenid) && minter.getAuctionStatus(_tokenid) == true);
+   require(msg.value > returnHighestBid(_tokenid) && block.timestamp < minter.getAuctionEndTime(_tokenid) && minter.getAuctionStatus(_tokenid) == true);
    auctionInfoStru memory newBid = auctionInfoStru(msg.sender, msg.value, true);
    auctionInfoData[_tokenid].push(newBid);
}
```


# [M-02] getPrice `salesOption` 2 can round down to the lower barrier, skipping the last time period

## Impact
[getPrice](https://github.com/code-423n4/2023-10-nextgen/blob/main/smart-contracts/MinterContract.sol#L530) in `salesOption` 2 can round down to the lower barrier, effectively skipping the last time period. In simpler terms, if the price is scheduled to decrease over 4 hours, it will decrease for 2.999... hours and then jump to the price at the end of the 4th hour.

## Proof of Concept
[getPrice](https://github.com/code-423n4/2023-10-nextgen/blob/main/smart-contracts/MinterContract.sol#L530) uses [this else-if calculation](https://github.com/code-423n4/2023-10-nextgen/blob/main/smart-contracts/MinterContract.sol#L553) to determine whether it should return the price as an equation or just the final price.

```solidity
if (((collectionMintCost - collectionEndMintCost) / rate) > tDiff) {
    price = collectionPhases[_collectionId].collectionMintCost - (tDiff * collectionPhases[_collectionId].rate);
} else {
    price = collectionPhases[_collectionId].collectionEndMintCost;
}
```

The if statement's method of division by `rate` is incorrect, as it results in rounding down and incorrect price movements. This can cause the price to jump over the last time period, effectively decreasing faster than intended.

Example:

| Values                  |                   |
|-------------------------|-------------------|
| `collectionMintCost`    | 0.49 ETH          |
| `collectionEndMintCost` | 0.1 ETH           |
| `rate`                  | 0.1 ETH (per hour)|
| `timePeriod`            | 1 hour - 3600 sec |

Hour 0 to 1 - 0.49 ETH
Hour 1 to 2 - 0.39 ETH - as it crosses the 1st hour, the price becomes 0.39 ETH
Hour 2 to 3 - 0.29 ETH - same here
It should go from 3 to 4 to 0.19 ETH, but that step is missed, and it goes directly to 0.1 ETH - the final price.
Hour 3 to 4 - 0.10 ETH - the final price

## Math
The equation for calculating `tDiff` using the provided variables is:

$$
tDiff = \frac{{\text{block.timestamp} - \text{collectionPhases}[\text{collectionId}].\text{allowlistStartTime}}}{{\text{collectionPhases}[\text{collectionId}].\text{timePeriod}}}
$$



We want the crossover when hour 2 switches to 3, which is when `(block.timestamp - collectionPhases[_collectionId].allowlistStartTime)` = 3 h or 10800 sec.

$$
tDiff = \frac{10800}{3600} = 3
$$

We plug in `tDiff` and the other variables into the [if](https://github.com/code-423n4/2023-10-nextgen/blob/main/smart-contracts/MinterContract.sol#L553).

```solidity
if (((collectionPhases[_collectionId].collectionMintCost - collectionPhases[_collectionId].collectionEndMintCost) 
                / (collectionPhases[_collectionId].rate)) > tDiff) {
```

$$
\frac{0.49 \times 10^{18} - 0.1 \times 10^{18}}{0.1 \times 10^{18}} = \frac{0.39 \times 10^{18}}{0.1 \times 10^{18}} = 3
$$

We obtain 3 as the answer, which is due to rounding down, as we divide 0.39 by 0.1. Solidity only works with whole numbers, causing the [if](https://github.com/code-423n4/2023-10-nextgen/blob/main/smart-contracts/MinterContract.sol#L553) to fail as `3 > 3` is false. This leads to entering the else statement where the last price of 0.1 ETH is returned, effectively missing one step of the 4-hour decrease.

## PoC
gist - https://gist.github.com/0x3b33/ab8a384f9979c4a9b7c4777be00a78de.
Add `contracts/=smart-contracts/` to mappings and run it with `forge test --match-test test_changeProof  --lib-paths ../smart-contracts -vv`. 

Logs: 
```
[PASS] test_changeProof() (gas: 745699)
Logs:
  4900000000000000000
  3900000000000000000
  2900000000000000000
  1000000000000000000
```

## Tools Used
Manual review.

## Recommended Mitigation Steps
A simple yet effective suggestion is to add `=` in the [if](https://github.com/code-423n4/2023-10-nextgen/blob/main/smart-contracts/MinterContract.sol#L553).

```diff
-  if (...) > tDiff){ 
+  if (...) >= tDiff){ 
```



# [M-03] If an airdrop happens before a mint the price could skyrocket

## Impact
As explained by the [docs](https://seize-io.gitbook.io/nextgen/nextgen-smart-contracts/features#minting-process), several steps can occur during the minting process. However, an airdrop before `salesOption` 3 can lead to price inflation.

## Proof of Concept
Under `salesOption` 3, [getPrice](https://github.com/code-423n4/2023-10-nextgen/blob/main/smart-contracts/MinterContract.sol#L536) returns:
```solidity
return collectionPhases[_collectionId].collectionMintCost
                    + ((collectionPhases[_collectionId].collectionMintCost / collectionPhases[_collectionId].rate) * gencore.viewCirSupply(_collectionId));
```
This is the increased rate based on the NFTs **already in circulation**. If an airdrop occurs before a mint with `salesOption` 3, the price will be much higher than intended.

Example:

| Steps            | Option     | NFTs    | Price | Rate                  |
|------------------|------------|---------|-------|-----------------------|
| 1 OG users       | Airdropped | 20 NFTs | free  | -                     |
| 2 Whitelisted    | Sales 3    | 10 NFTs | 1 ETH | 10 (0.1 ETH increase) |
| 3 Public         | Sales 1    | 70 NFTs | 2 ETH | -                     |

With the current three steps, after the airdrop, `salesOption` 3 should start at 1 ETH and gradually increase to 2 ETH. Afterward, it should mint at a constant rate of 2 ETH.

However, when sales option 3 starts, [getPrice](https://github.com/code-423n4/2023-10-nextgen/blob/main/smart-contracts/MinterContract.sol#L536) will return 3 ETH instead of 1 ETH. This will cause the initial users to pay an inflated price, which was not intended by the owner and can harm their reputation. It's also unfair to the users, as these so-called special (whitelisted) users will pay increased prices.

$$
\begin{align*}
\text{collectionMintCost} + \left(\frac{\text{collectionMintCost}}{\text{rate}}\right) \times \text{cirSupply}  \\
&= 1 \text{eth} + 0.1 \text{eth} \times 20 \\
&= 1 \text{eth} + 2 \text{eth} \\
&= 3 \text{eth}
\end{align*}
$$

## POC
Gist - https://gist.github.com/0x3b33/558b919a57101e7a0942e557a464078a
Add to remappings - `contracts/=smart-contracts/` 
Run it with `forge test --match-test test_airdrop  --lib-paths ../smart-contracts`

## Tools Used
Manual review.

## Recommended Mitigation Steps
You can implement a mapping with the airdropped NFTs and deduct this value from `gencore.viewCirSupply(_collectionId)` to avoid disrupting the minting process.



# [M-04] The winner can DoS the auction and stuck the NFT and users funds in the contract

## Impact
Winners will be able to DoS auctions and lock both their NFTs and all of the funds of other bidders. This issue arises because **AuctionDemo**'s [claimAuction](https://github.com/code-423n4/2023-10-nextgen/blob/main/smart-contracts/AuctionDemo.sol#L104-L120) uses push instead of a pull.

The winner does not need to be a user, it can also be a competitive entity. A competitor against the collection or against NextGen.

## Proof of Concept
Once the auction has ended, the winner can invoke [claimAuction](https://github.com/code-423n4/2023-10-nextgen/blob/main/smart-contracts/AuctionDemo.sol#L104-L120). And to revert on his `_checkOnERC721Received`.

The core issue is in this [if if](https://github.com/code-423n4/2023-10-nextgen/blob/main/smart-contracts/AuctionDemo.sol#L115-L118), within this `for` loop, where, if the current bidder is not the winner, their funds are refunded.
```solidity
for (uint256 i=0; i< auctionInfoData[_tokenid].length; i++) {
    ...
    } else if (auctionInfoData[_tokenid][i].status == true) {
        (bool success, ) = payable(auctionInfoData[_tokenid][i].bidder).call{value: auctionInfoData[_tokenid][i].bid}("");
        emit Refund(auctionInfoData[_tokenid][i].bidder, _tokenid, success, highestBid);
    } else {}
}
```

Example:
1. NFT is scheduled for auction with start of 0.01ETH
3. 50 more bidders bid - 0.011, 0.015... 0.29.
2. Attacker bids last - 0.3ETH (1 wei higher than the bid before him) using a contract that reverts when it receives the NFT.
4. Auction ends.
5. [claimAuction](https://github.com/code-423n4/2023-10-nextgen/blob/main/smart-contracts/AuctionDemo.sol#L104-L120) reverts as the call made to the attacker contract reverts.

Attacker loses some ETH, however 50 bidders lose their bids (the last one is the same as the attaker) and the NFT will remain stuck in the contract.


## Tools Used
Manual review 

## Recommended Mitigation Steps
Utilize the pull method instead of the push method. This can be achieved by creating another function that allows users to manually withdraw their funds after the auction concludes.


