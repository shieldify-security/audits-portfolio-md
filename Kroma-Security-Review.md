# 1. About Shieldify

Positioned as the first hybrid Web3 Security company, Shieldify shakes things up with a unique subscription-based auditing model that entitles the customer to unlimited audits within its duration, as well as top-notch service quality thanks to a disruptive 6-layered security approach. The company works with very well-established researchers in the space and have secured multiple millions in TVL across protocols, also can audit codebases written in Solidity, Vyper, Rust, Cairo, Move and Go.

Learn more about us at [`shieldify.org`](https://shieldify.org/).

# 2. Disclaimer

This security review does not guarantee bulletproof protection against a hack or exploit. Smart contracts are a novel technological feat with many known and unknown risks. The protocol, which this report is intended for, indemnifies Shieldify Security against any responsibility for any misbehavior, bugs, or exploits affecting the audited code during any part of the project's life cycle. It is also pivotal to acknowledge that modifications made to the audited code, including fixes for the issues described in this report, may introduce new problems and necessitate additional auditing.

# 3. About Kroma

Kroma is the first universal stage 2 Layer 2 (L2) on Ethereum. Kroma has been developed based on Optimism Bedrock architecture and aims to be a universal ZK rollup. Currently, Kroma operates as an Optimistic rollup with ZK fault proofs, utilizing a zkEVM based on Scroll. The goal of Kroma is to eventually transition to a ZK rollup once the generation of ZK proofs becomes more cost-efficient and faster.

Kroma offers lower transactions fees compared to the Ethereum mainnet, with native Ethereum security and EVM equivalence. Our roadmap to transition from Optimistic rollup to ZK rollup sets Kroma apart from other L2 networks. Additionally, Kroma is advancing towards decentralization. As an initial step for this, Kroma has introduced a permissionless validator and challenge system.

For more detailed information about Kroma, check [Kroma docs](https://docs.kroma.network/) (or [spec docs](https://specs.kroma.network/)).

# 4. Risk Classification

|        Severity        | Impact: High | Impact: Medium | Impact: Low |
| :--------------------: | :----------: | :------------: | :---------: |
|  **Likelihood: High**  |   Critical   |      High      |   Medium    |
| **Likelihood: Medium** |     High     |     Medium     |     Low     |
|  **Likelihood: Low**   |    Medium    |      Low       |     Low     |

## 4.1 Impact

- **High** - results in a significant risk for the protocol’s overall well-being. Affects all or most users
- **Medium** - results in a non-critical risk for the protocol affects all or only a subset of users, but is still
  unacceptable
- **Low** - losses will be limited but bearable - and covers vectors similar to griefing attacks that can be easily repaired

## 4.2 Likelihood

- **High** - almost certain to happen and highly lucrative for execution by malicious actors
- **Medium** - still relatively likely, although only conditionally possible
- **Low** - requires a unique set of circumstances and poses non-lucrative cost-of-execution to rewards ratio for the actor

# 5. Security Review Summary

The security review lasted 7 days with a total of 14 hours dedicated to the audit by two researchers from the Shieldify team.

Overall, the code is well-written and tested. The audit report contributed by identifying an issue that could lead to fund loss for the protocol.

## 5.1 Protocol Summary

| **Project Name**             | Kroma                                                                                                                            |
| ---------------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| **Repository**               | [kroma](https://github.com/kroma-network/kroma/tree/feat/implement-validator-system-v2)                                          |
| **Type of Project**          | Layer 2 on Ethereum                                                                                                              |
| **Security Review Timeline** | 7 days                                                                                                                           |
| **Review Commit Hash**       | [7359dbece7f3523bed3327a0c06a875331fb0de5](https://github.com/kroma-network/kroma/tree/7359dbece7f3523bed3327a0c06a875331fb0de5) |
| **Fixes Review Commit Hash** | N/A                                                                                                                              |

## 5.2 Scope

The following smart contracts were in the scope of the security review:

| File                                                 | nSLOC |
| ---------------------------------------------------- | :---: |
| packages/contracts/contracts/L1/AssetManager.sol     |  621  |
| packages/contracts/contracts/L1/Colosseum.sol        |  440  |
| packages/contracts/contracts/L1/L2OutputORacle.sol   |  202  |
| packages/contracts/contracts/L1/ValidatorManager.sol |  359  |
| packages/contracts/contracts/L1/ValidatorPool.sol    |  278  |
| Total                                                | 1900  |

# 6. Findings Summary

The following number of issues have been identified, sorted by their severity:

- **Critical** and **High** issues: 0
- **Medium** issues: 1
- **Low** issues: 0

| **ID** | **Title**                                                                                                                     | **Severity** | **Status** |
| :----: | ----------------------------------------------------------------------------------------------------------------------------- | :----------: | :--------: |
| [М-01] | Timed Out Challenger Can Front Run Any Calls to the `_challengerTimeout` and Prevent Paying the Tax to the `SECURITY_COUNCIL` |    Medium    |   Fixed    |

# 7. Findings

# [M-01] Timed Out Challenger Can Front Run Any Calls to the `_challengerTimeout` and Prevent Paying the Tax to the `SECURITY_COUNCIL`

## Severity

Medium Risk

## Description

When a new l2 output is submitted on the l2 oracle output contract the function `createBond()` is called to create a bond for the l2Output, the `createBond()` function will decrease the submitter balance and the bond created. The next step that may happen is creating a challenge to prove that the output is not valid. This is possible by making call to the `createChallenge()` function in the `Colosseum.sol` contract.

The issue arises when the challenger is timed out and the function `_challengerTimeout()` gets executed(this function can be called by `challengerTimeout()` function when the challenger is timed out, this will call the `increaseBond()` function which in return it should pay tax to the security council, an attacker can prevent this to happen by calling the `unbond` which in turn `delete bonds[outputIndex]` and sets the value of amount and expired at to zero, this will cause damage to the security council and can lead to prevent sending any taxes to the council during the protocol life(until the `validatorPool` gets deprecated).

## Scenario

Let's see the execution flow:

- First, the l2 output oracle creates a bond through a `submitL2Output()` function.

- The new output is created and so do bonds - `createBond()`.

- Then a challenger calls the `createChallenge()` to make a call to the `addPendingBond()` as show below:

```solidity
function createChallenge(uint256 _outputIndex, bytes32 _l1BlockHash, uint256 _l1BlockNumber, bytes32[] calldata _segments) external {
    // code

    // Switch validator system after validator pool contract terminated.
    if (L2_ORACLE.VALIDATOR_POOL().isTerminated(_outputIndex)) {
        // Only the validators whose status is active can create challenge.
        if (!L2_ORACLE.VALIDATOR_MANAGER().isActive(msg.sender))
            revert ImproperValidatorStatus();
    } else {
        L2_ORACLE.VALIDATOR_POOL().addPendingBond(_outputIndex, msg.sender);
    }

    // code
}
```

- The challenger times is out(finished) and someone or the submitter calls the function below:

```solidity
function _challengerTimeout(uint256 _outputIndex, address _challenger) private {
    // code

    // After output is finalized, the challenger's bond is included in the balance of output submitter.
    if (L2_ORACLE.isFinalized(_outputIndex)) {
        address submitter = L2_ORACLE.getSubmitter(_outputIndex);
        L2_ORACLE.VALIDATOR_POOL().releasePendingBond(_outputIndex, _challenger, submitter);
    } else {
        // Because the challenger lost, the challenger's bond is included in the bond for that output.
        L2_ORACLE.VALIDATOR_POOL().increaseBond(_outputIndex, _challenger);
    }
}
```

- The `increaseBond()` function in return should pay the security council a fair amount of tax:

- In this step, an attacker or the challenger himself can make a call to the `unbond()` function to prevent paying tax to the security council and front run the call to the `increaseBond()` function, this is possible because when `unbond()` called the bonds storage get deleted which mean the `require` check in the `increaseBond()` will revert `require( bond.expiresAt >= block.timestamp)`:

```solidity
function unbond() external {
    bool released = _tryUnbond();
    require(released, "ValidatorPool: no bond that can be unbond");
}

/**
* @notice Attempts to unbond starting from nextUnbondOutputIndex and returns whether at least
*         one unbond is executed. Tries unbond at most MAX_UNBOND number of bonds and sends
*         a reward message to L2 for each unbond.
* @return Whether at least one unbond is executed.
*/
function _tryUnbond() private returns (bool) {
    // code
    if (block.timestamp >= bond.expiresAt && bondAmount > 0) {
        delete bonds[outputIndex]; //@audit set all inside value to zero
        output = L2_ORACLE.getL2Output(outputIndex);
        _increaseBalance(output.submitter, bondAmount);
        emit Unbonded(outputIndex, output.submitter, bondAmount);
        // code
    }
}
```

## Impact

An attacker front-running the call to `increaseBond()` can cause the security council to lose tax revenue and not get paid.

## Recommendation

It's recommended to allow only trusted users to call the `unbond()` function only or add a pausing mechanism to the `unbond()` function when the increase bond function gets called.

## Team Response

N/A
