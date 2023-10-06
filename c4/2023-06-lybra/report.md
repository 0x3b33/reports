
| Severity | Title | 
|:--:|:---|
| [H-01](#h-01-all-the-functions-in-lybraconfigurator-can-be-called-by-malicious-accounts) | All the functions in LybraConfigurator.sol can be called by malicious accounts |
| [M-01](#m-01-the-governance-cant-function-correctly) | The governance can't function correctly |

## [H-01] All the functions in LybraConfigurator can be called by malicious accounts

## Impact
All the functions in LybraConfigurator.sol can be called by malicious accounts. This breaks the whole protocol, as anyone can change the tokens, the fees and the ratios, the borrow apy or mint unlimited amount of tokens, etc..

## Proof of concept
A malicious user calls a **LybraConfigurator** function that is supposed to be executable only by the **DAO** or the **TIMELOCK** . The following modifiers should check if the caller has the needed rights. Both `GovernanceTimelock.checkOnlyRole()`  and `GovernanceTimelock.checkRole() return booleans indicating if a user has the needed role. However, the returned value is not checked, allowing anyone to execute the certain function.

```jsx
 modifier onlyRole(bytes32 role) {
        GovernanceTimelock.checkOnlyRole(role, msg.sender);
        _;
}

    modifier checkRole(bytes32 role) {
        GovernanceTimelock.checkRole(role, msg.sender);
        _;
    }
```
## Tools Used
Manual review

## Recommended Mitigation Steps
In both the modifiers, add a require statement that checks the return value of the GovernanceTimelock function. 
```jsx
modifier onlyRole(bytes32 role) {
        require(GovernanceTimelock.checkOnlyRole(role, msg.sender), "NO_ROLE");
        _;
}


modifier checkRole(bytes32 role) {
    require(GovernanceTimelock.checkRole(role, msg.sender), "NO_ROLE");
    _;
}
```
## [M-01] The governance can't function correctly 

## Impact
Due to inappropriate short `votingPeriod`  and `votingDelay`, it is near impossible for the governance to function correctly.

## Proof of Concept
When making proposals with the [`Governor`](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/governance/Governor.sol#L299-L308) contract OZ uses `votingPeriod`
```jsx
        uint256 snapshot = currentTimepoint + votingDelay();
        uint256 duration = votingPeriod();

        _proposals[proposalId] = ProposalCore({
            proposer: proposer,
            voteStart: SafeCast.toUint48(snapshot),//@audit votingDelay() for when the voting starts
            voteDuration: SafeCast.toUint32(duration),//@audit votingPeriod() for the duration
            executed: false,
            canceled: false
        });
``` 
But currently Lybra has implemented wrong amounts for bolt [`votingPeriod`](https://github.com/code-423n4/2023-06-lybra/blob/main/contracts/lybra/governance/LybraGovernance.sol#L143-L145) and [`votingDelay`](https://github.com/code-423n4/2023-06-lybra/blob/main/contracts/lybra/governance/LybraGovernance.sol#L147-L149), which means proposals from the governance will be near impossible to be voted on.
```jsx
    function votingPeriod() public pure override returns (uint256){
         return 3;//@audit this should be time in seconds 
    }

     function votingDelay() public pure override returns (uint256){
         return 1;//@audit this should be time in seconds 
    }
```
## Tools Used
Manual Review

## Recommended Mitigation Steps
You can implement it as OZ suggest's i their [examples](https://docs.openzeppelin.com/contracts/4.x/governance)
```jsx
    function votingDelay() public pure override returns (uint256) {
        return 7200; // 1 day
    }

    function votingPeriod() public pure override returns (uint256) {
        return 50400; // 1 week
    }
```