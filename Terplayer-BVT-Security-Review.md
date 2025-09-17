# 1. About Shieldify

Positioned as the first hybrid Web3 Security company, Shieldify shakes things up with a unique subscription-based auditing model that entitles the customer to unlimited audits within its duration, as well as top-notch service quality thanks to a disruptive 6-layered security approach. The company works with very well-established researchers in the space and has secured multiple millions in TVL across protocols, also can audit codebases written in Solidity, Vyper, Rust, Cairo, Move and Go.

Learn more about us at [`shieldify.org`](https://shieldify.org/).

# 2. Disclaimer

This security review does not guarantee bulletproof protection against a hack or exploit. Smart contracts are a novel technological feat with many known and unknown risks. The protocol, which this report is intended for, indemnifies Shieldify Security against any responsibility for any misbehavior, bugs, or exploits affecting the audited code during any part of the project's life cycle. It is also pivotal to acknowledge that modifications made to the audited code, including fixes for the issues described in this report, may introduce new problems and necessitate additional auditing.

# 3. About Terplayer - BeraBTC Vault Token (BVT)

## Token Overview

BeraBTC Vault Token (BVT) is an ERC20 standard token contract designed specifically for the BTC vault system on the Bera network.

## Token Information

- Token Name: BeraBTC Vault Token
- Token Symbol: BVT
- Decimals: 18
- Total Supply: 1,000,000,000 BVT (1 billion tokens)
- Network: Bera Network
- Standard: ERC20

## Contract Features

The BVT token contract is built on OpenZeppelin's ERC20 implementation and provides the following features:

- Standard ERC20 token functionality (transfer, approve, balance queries, etc.)
- Automatic minting of the total supply to the deployer upon deployment
- Fixed total supply with no additional minting capability

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

The security review lasted 1 day, with a total of 4 hours dedicated by 2 researchers from the Shieldify team.

Overall, the code is very well-written. The audit report found no vulnerabilities, as the contract is a fixed-supply ERC20 built on OpenZeppelin’s standard implementation that mints 1 billion tokens to the deployer at deployment.

The Terplayer team has done a great job with the development and has been highly responsive to the Shieldify research team’s inquiries.

## 5.1 Protocol Summary

| **Project Name**             | Terplayer - BVT                                                                                                                   |
| ---------------------------- | --------------------------------------------------------------------------------------------------------------------------------- |
| **Repository**               | [bvt-erc20](https://github.com/batoshidao/bvt-erc20)                                                                              |
| **Type of Project**          | ERC20, Vault Token                                                                                                                |
| **Security Review Timeline** | 0.5 days                                                                                                                          |
| **Review Commit Hash**       | [4d5c505b7cb29c7fd655aa1cf96ee089c52c0889](https://github.com/batoshidao/bvt-erc20/tree/4d5c505b7cb29c7fd655aa1cf96ee089c52c0889) |

## 5.2 Scope

The following smart contracts were in the scope of the security review:

| File                      | nSLOC |
| ------------------------- | :---: |
| src/BeraBTCVaultToken.sol |   8   |
| Total                     |   8   |
