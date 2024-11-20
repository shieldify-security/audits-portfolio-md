# 1. About Shieldify

Positioned as the first hybrid Web3 Security company, Shieldify shakes things up with a unique subscription-based auditing model that entitles the customer to unlimited audits within its duration, as well as top-notch service quality thanks to a disruptive 6-layered security approach. The company works with very well-established researchers in the space and have secured multiple millions in TVL across protocols, also can audit codebases written in Solidity, Vyper, Rust, Cairo, Move and Go.

Learn more about us at [`shieldify.org`](https://shieldify.org/).

# 2. Disclaimer

This security review does not guarantee bulletproof protection against a hack or exploit. Smart contracts are a novel technological feat with many known and unknown risks. The protocol, which this report is intended for, indemnifies Shieldify Security against any responsibility for any misbehavior, bugs, or exploits affecting the audited code during any part of the project's life cycle. It is also pivotal to acknowledge that modifications made to the audited code, including fixes for the issues described in this report, may introduce new problems and necessitate additional auditing.

# 3. About Hana Network

Hana is the first privacy layer between Ethereum and Bitcoin, complying with regulations, and enabling on-chain privacy on existing chains and for arbitrary assets. We are selected for Binance Labs Incubation and won the 2nd prize at the Zkday pitch competition in 2023.

HanaChain is an EVM-compatible L1 blockchain that enables omnichain, generic smart contracts and messaging between any blockchain.

## Prerequisites

- [Go](https://golang.org/doc/install) 1.20
- [Docker](https://docs.docker.com/install/) and
  [Docker Compose](https://docs.docker.com/compose/install/) (optional, for
  running tests locally)
- [buf](https://buf.build/) (optional, for processing protocol buffer files)
- [jq](https://stedolan.github.io/jq/download/) (optional, for running scripts)

## Components of HanaChain

HanaChain is built with [Cosmos SDK](https://github.com/cosmos/cosmos-sdk), a
modular framework for building blockchain and
[Ethermint](https://github.com/evmos/ethermint), a module that implements
EVM-compatibility.

- [hana-node](https://github.com/hana-network/node) (this repository)
  contains the source code for the HanaChain node (`hanacored`) and the
  HanaChain client (`hanaclientd`).
- [protocol-contracts](https://github.com/zeta-chain/protocol-contracts)
  contains the source code for the Solidity smart contracts that implement the
  core functionality of HanaChain.

# 4. Risk Classification

|     Severity      |                                                                                                                                                          Description                                                                                                                                                           |
| :---------------: | :----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------: |
|     **High**      |                                                                                             A serious and exploitable vulnerability that can lead to loss of funds, unrecoverable locked funds, or catastrophic denial of service.                                                                                             |
|    **Medium**     |                                                                                                  A vulnerability or bug that can affect the correct functioning of the system, lead to incorrect states or denial of service.                                                                                                  |
|      **Low**      |                                                                    A violation of common best practices or incorrect usage of primitives, which may not currently have a major impact on security, but may do so in the future or introduce inefficiencies.                                                                    |
| **Informational** | Comments and recommendations of design decisions or potential optimizations, that are not relevant to security. Their application may improve aspects, such as user experience or readability, but is not strictly necessary. This category may also include opinionated recommendations that the project team might not share |

# 5. Security Review Summary

The security review lasted 1 week with a total of 224 hours dedicated to the audit by four researchers from the Shieldify team.

Overall, the code is well-written and leverages Zetachain's codebase as a foundation. The audit report contributed by identifying two issues of low severity, covering the usage of whitelist approach, as well as using Cosmos SDK and Go versions that are prone to vulnerabilities.

We would like to extend our gratitude to the Hana Network team for their fast and meticulous responses.

## 5.1 Protocol Summary

| **Project Name**             | Hana Network                                                                                                                   |
| ---------------------------- | ------------------------------------------------------------------------------------------------------------------------------ |
| **Repository**               | [hana-node](https://github.com/Hana-Network/node)                                                                              |
| **Type of Project**          | EVM-compatible L1 blockchain, Cosmos SDK, Go 1.20                                                                              |
| **Security Review Timeline** | 5 days                                                                                                                         |
| **Review Commit Hash**       | [273dea4c1e73025501c8b8c0f9c7713c4d152be2](https://github.com/Hana-Network/node/tree/273dea4c1e73025501c8b8c0f9c7713c4d152be2) |
| **Fixes Review Commit Hash** | N/A                                                                                                                            |

## 5.2 Scope

The following smart contracts were in the scope of the security review:

| File Name                                                             |
| --------------------------------------------------------------------- |
| app/ante/ante.go                                                      |
| app/ante/ante_test.go                                                 |
| app/ante/handler_options.go                                           |
| app/app.go                                                            |
| changelog.md                                                          |
| contrib/localnet/scripts/start-hanacored.sh                           |
| contrib/seeding-devnet/README.md                                      |
| contrib/seeding-devnet/scripts/start-hanacored-1.sh                   |
| contrib/seeding-devnet/scripts/start-hanacored-2.sh                   |
| contrib/seeding-devnet/scripts/start-hanacored-for-generating-node.sh |
| contrib/seeding-devnet/scripts/start-hanacored-for-sync-node.sh       |

# 6. Findings Summary

The following number of issues have been identified, sorted by their severity:

- **Critical** and **High** issues: 0
- **Medium** issues: 0
- **Low** issues: 2

| **ID** | **Title**                                                | **Severity** |  **Status**  |
| :----: | -------------------------------------------------------- | :----------: | :----------: |
| [L-01] | Consider Using the Whitelist Approach Over the Blacklist |     Low      | Acknowledged |
| [L-02] | Vulnerable Cosmos SDK and Go-Ethereum Version Used       |     Low      |    Fixed     |

# 7. Findings

# [L-01] Consider Using the Whitelist Approach Over the Blacklist

## Severity

Low Risk

## Description

The `blockedReceivingModAcc` mapping configures module accounts that cannot receive tokens. This approach means that future-added modules can receive tokens from users by default, which may be problematic if the modules are not expected to receive tokens and cannot be sent out.

## Location of Affected Code

File: [app/app.go#L208-L214](https://github.com/Hana-Network/node/blob/273dea4c1e73025501c8b8c0f9c7713c4d152be2/app/app.go#L208-L214)

```go
blockedReceivingModAcc = map[string]bool{
	distrtypes.ModuleName:          true,
	authtypes.FeeCollectorName:     true,
	stakingtypes.BondedPoolName:    true,
	stakingtypes.NotBondedPoolName: true,
	govtypes.ModuleName:            true,
}
```

## Recommendation

Consider implementing a whitelist approach of what modules can receive tokens over a blacklist module with an `allowedReceivingModAcc` mapping.

## Team Response

Acknowledged.

# [L-02] Vulnerable Cosmos SDK and Go-Ethereum Version Used

## Severity

Low Risk

## Description

The codebase uses a `cosmos-sdk` version of v0.46.13, vulnerable to the “[ASA-2024-005: Potential slashing evasion during re-delegation](https://github.com/cosmos/cosmos-sdk/security/advisories/GHSA-86h5-xcpx-cfqc)” issue.

Additionally, the `go-ethereum` version used is v1.10.26, which is vulnerable to denial of service via malicious p2p messages, as indicated in [GHSA-ppjg-v974-84cm](https://github.com/ethereum/go-ethereum/security/advisories/GHSA-ppjg-v974-84cm) and [GHSA-4xc9-8hmq-j652](https://github.com/advisories/GHSA-4xc9-8hmq-j652).

## Location of Affected Code

File: [go.mod#L6-L7](https://github.com/Hana-Network/node/blob/273dea4c1e73025501c8b8c0f9c7713c4d152be2/go.mod#L6-L7)

```go
require (
	github.com/cosmos/cosmos-sdk v0.46.13
    github.com/ethereum/go-ethereum v1.10.26
```

## Recommendation

Consider updating `cosmos-sdk` and `go-ethereum` to their latest stable version.

## Team Response

Will be fixed as proposed.
