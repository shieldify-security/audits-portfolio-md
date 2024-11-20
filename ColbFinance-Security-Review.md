# 1. About Shieldify

Positioned as the first hybrid Web3 Security company, Shieldify shakes things up with a unique subscription-based auditing model that entitles the customer to unlimited audits within its duration, as well as top-notch service quality thanks to a disruptive 6-layered security approach. The company works with very well-established researchers in the space and have secured multiple millions in TVL across protocols, also can audit codebases written in Solidity, Vyper, Rust, Cairo, Move and Go.

Learn more about us at [`shieldify.org`](https://shieldify.org/).

# 2. Disclaimer

This security review does not guarantee bulletproof protection against a hack or exploit. Smart contracts are a novel technological feat with many known and unknown risks. The protocol, which this report is intended for, indemnifies Shieldify Security against any responsibility for any misbehavior, bugs, or exploits affecting the audited code during any part of the project's life cycle. It is also pivotal to acknowledge that modifications made to the audited code, including fixes for the issues described in this report, may introduce new problems and necessitate additional auditing.

# 3. About Colb Finance

Colb is the first native non-custodial tokenization solution that enables peerless access to Swiss-grade wealth management strategies, pre-IPO opportunities, and premium investment funds. It offers a bankruptcy remote Trust structure and native ownership of real-world assets, all on-chain. Colb reduces the entrance threshold to such investments by removing constraints such as the minimum investment amount to get exposure to them. The protocol is designed with security at its core, boasting compliance with Swiss regulations and DeFi composability. Colb envisions a future rooted in transparency where every individual has equitable access to premium RWA investments.

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

The security review lasted 3 days with a total of 72 hours dedicated to the audit by three researchers from the Shieldify team.

Overall, the code is well-written. The audit report contributed by identifying a missing validation checkс during the token migration and provided several other recommendations and best practice guidelines.

The Colb Finance team has done a great job with their test suite and provided support and responses to all of the questions that the Shieldify researchers had.

## 5.1 Protocol Summary

| **Project Name**             | Colb Finance                                                       |
| ---------------------------- | ------------------------------------------------------------------ |
| **Repository**               | N/A                                                                |
| **Type of Project**          | ERC20, Migration contract - Burns old tokens and mints new tokens. |
| **Security Review Timeline** | 3 days                                                             |
| **Review Commit Hash**       | N/A                                                                |
| **Fixes Review Commit Hash** | N/A                                                                |

## 5.2 Scope

The following smart contracts were in the scope of the security review:

| File                                      | nSLOC |
| ----------------------------------------- | :---: |
| contracts/stablecoin/USC.sol              |  22   |
| contracts/stablecoin/Migration.sol        |  55   |
| contracts/stablecoin/StableManagement.sol |  56   |
| Total                                     |  133  |

# 6. Findings Summary

The following number of issues have been identified, sorted by their severity:

- **Critical** and **High** issues: 0
- **Medium** issues: 0
- **Low** issues: 0
- **Info** issues: 5

| **ID** | **Title**                                              | **Severity**  | **Status** |
| :----: | ------------------------------------------------------ | :-----------: | :--------: |
| [I-01] | Missing Check By Tokens Migration                      |      Low      |    N/A     |
| [I-02] | Incomplete NatSpec                                     | Informational |    N/A     |
| [I-03] | Missing Event Emission                                 | Informational |    N/A     |
| [I-04] | Use `Ownable2Step` Instead of `Ownable`                | Informational |    N/A     |
| [I-05] | Change Function Visibility from `public` to `external` | Informational |    N/A     |

# 7. Findings

# [I-01] Missing Check By Tokens Migration

## Severity

Informational

## Description

The `Migration.sol` constructor misses a check if `_oldToken!=_newToken`. While not detrimental to the protocol's well-being, this can introduce UX hindrances and the need to redeploy the Migration contract.

## Location of Affected Code

File: contracts/stablecoin/Migration.sol#L42

## Recommendation

Add the following check:

```diff
    constructor(address _oldToken, address _newToken, address _newTokenMgmt) {
        if (
            _oldToken == address(0) ||
            _newToken == address(0) ||
            _newTokenMgmt == address(0) ||
+           _oldToken == _newToken
        ) {
            revert InvalidAddress();
        }
```

## Team Response

Acknowledged.

# [I-02] Incomplete NatSpec

- In the `Migration` contract in the `constructor()` function for the `_newTokenMgmt` property.

## Team Response

Acknowledged.

# [I-03] Missing Event Emission

Events are a method of informing the transaction initiator about the actions taken by the called function. An event logs its emitted parameters in a specific log history, which can be accessed outside of the contract using some filter parameters.

It has been observed that the following important methods are missing an emitting event:

- In the `Migration` contract in the `switchMigration()` function.

## Team Response

Acknowledged.

# [I-04] Use `Ownable2Step` Instead of `Ownable`

The current ownership transfer process for all the contracts inheriting from Ownable involves the current owner calling the `transferOwnership()` function. If the nominated EOA account is not a valid account, it is entirely possible that the owner may accidentally transfer ownership to an uncontrolled account, losing access to all functions with the `onlyOwner` modifier.

It is recommended to implement a two-step process where the owner nominates an account and the nominated account needs to call an `acceptOwnership()` function for the transfer of the ownership to fully succeed. This ensures the nominated EOA account is a valid and active account. This can be easily achieved by using OpenZeppelin’s `Ownable2Step` contract instead of `Ownable`.

## Team Response

Acknowledged.

# [I-05] Change Function Visibility from `public` to `external`

## Description

It is best practice to mark functions that are not called internally as external instead, as this saves gas (especially in the case where the function takes arguments, as external functions can read arguments directly from calldata instead of having to allocate memory).

## Location of Affected Code

All `public` functions in `USC.sol` and `StableManagement.sol`.

## Recommendation

Consider changing the visibility of the function to `external`.

## Team Response

Acknowledged.
