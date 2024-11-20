# 1. About Shieldify

Positioned as the first hybrid Web3 Security company, Shieldify shakes things up with a unique subscription-based auditing model that entitles the customer to unlimited audits within its duration, as well as top-notch service quality thanks to a disruptive 6-layered security approach. The company works with very well-established researchers in the space and has secured multiple millions in TVL across protocols, also can audit codebases written in Solidity, Vyper, Rust, Cairo, Move and Go.

Learn more about us at [`shieldify.org`](https://shieldify.org/).

# 2. Disclaimer

This security review does not guarantee bulletproof protection against a hack or exploit. Smart contracts are a novel technological feat with many known and unknown risks. The protocol, which this report is intended for, indemnifies Shieldify Security against any responsibility for any misbehavior, bugs, or exploits affecting the audited code during any part of the project's life cycle. It is also pivotal to acknowledge that modifications made to the audited code, including fixes for the issues described in this report, may introduce new problems and necessitate additional auditing.

# 3. About Revert

Revert Lend is a decentralized peer-to-pool lending protocol designed for Automated Market Maker Liquidity Providers. It allows users to collateralize their Uniswap v3 liquidity provider positions, in the form of the Uniswap NFT- Manager NFTs, and obtain loans in a protocol-determined ERC-20 token. The protocol is specifically designed to allow Liquidity Providers to maintain control over the capital in their positions while they are collateralized. This feature facilitates uninterrupted management and optimization of LP positions, catering to the dynamic needs of liquidity providers.

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

The security review lasted 7 days with a total of 168 hours dedicated to the audit by three researchers from the Shieldify team.

Overall, the code is well-written. The audit report contributed by providing several recommendations and best practice guidelines.

The Revert team has excelled in their security approach, undergoing several public contests, formal verification, and multiple private security reviews. They have also provided thorough support and responses to all the questions raised by the Shieldify researchers.

## 5.1 Protocol Summary

| **Project Name**             | Revert                                                                                                                           |
| ---------------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| **Repository**               | [revert-finance/lend](https://github.com/revert-finance/lend)                                                                    |
| **Type of Project**          | Decentralized Peer-to-Pool Lending Protocol                                                                                      |
| **Security Review Timeline** | 7 days                                                                                                                           |
| **Review Commit Hash**       | [a9925b2e08bf7776dd7da41ea8d48f62faa4f1ca](https://github.com/revert-finance/lend/tree/a9925b2e08bf7776dd7da41ea8d48f62faa4f1ca) |
| **Fixes Review Commit Hash** | N/A                                                                                                                              |

## 5.2 Scope

The following smart contracts were in the scope of the security review:

| File                                     | nSLOC |
| ---------------------------------------- | :---: |
| src/automators/AutoExit.sol              |  187  |
| src/automators/Automator.sol             |  148  |
| src/interfaces/IV3Oracle.sol             |   3   |
| src/interfaces/IVault.sol                |  22   |
| src/interfaces/IInterestRateModel.sol    |   3   |
| src/V3Vault.sol                          |  843  |
| src/V3Oracle.sol                         |  360  |
| src/transformers/LeverageTransformer.sol |  144  |
| src/transformers/V3Utils.sol             |  687  |
| src/transformers/Transformer.sol         |  43   |
| src/transformers/AutoCompound.sol        |  210  |
| src/transformers/AutoRange.sol           |  242  |
| src/InterestRateModel.sol                |  55   |
| src/utils/Constants.sol                  |  56   |
| src/utils/Swapper.sol                    |  112  |
| src/utils/FlashloanLiquidator.sol        |  90   |
| Total                                    | 3205  |

# 6. Findings Summary

The following number of issues have been identified, sorted by their severity:

- **Critical** and **High** issues: 0
- **Medium** issues: 0
- **Low** issues: 0
- **Info** issues: 2

| **ID** | **Title**                          | **Severity**  | **Status** |
| :----: | ---------------------------------- | :-----------: | :--------: |
| [I-01] | Use Two-Step Role Transfer         | Informational |    N/A     |
| [I-02] | Use Locked Solidity Version Pragma | Informational |    N/A     |

# 7. Findings

# [I-01] Use Two-Step Role Transfer

## Severity

Informational

## Description

The current process for transferring the emergency admin role requires the current owner to call the `setEmergencyAdmin()` function. If the designated EOA account is not valid, there's a risk that the owner might inadvertently assign the emergency admin role to an unmanaged account, thereby losing access to the emergency admin functionalities.

## Location of Affected Code

File: [V3Vault.sol](https://github.com/revert-finance/lend/blob/a9925b2e08bf7776dd7da41ea8d48f62faa4f1ca/src/V3Vault.sol)

File: [V3Oracle.sol](https://github.com/revert-finance/lend/blob/a9925b2e08bf7776dd7da41ea8d48f62faa4f1ca/src/V3Oracle.sol)

## Recommendation

It is recommended to implement a two-step process where the owner nominates an account and the nominated account needs to call an `acceptOwnership()` function for the transfer of the ownership to fully succeed. This ensures the nominated EOA account is a valid and active account.

## Team Response

N/A

# [I-02] Use Locked Solidity Version Pragma

## Severity

Informational

## Description

Currently, version `^0.8.0` is used in the codebase. Contracts should be deployed with the same compiler version that they have been tested with. Locking the pragma helps to ensure that contracts do not accidentally get deployed using, for example, either an outdated compiler version that might introduce bugs or a compiler version too recent that has not been extensively tested yet.

## Location of Affected Code

All smart contracts.

## Recommendation

Consider locking the version pragma to the same Solidity version used during development and testing `(0.8.25)`.

```diff
- pragma solidity ^0.8.0;
+ pragma solidity 0.8.25;
```

## Team Response

N/A
