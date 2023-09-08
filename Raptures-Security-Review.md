# 1. About Shieldify

We are Shieldify Security – a company on a mission to make web3 protocols more secure, cost-efficient and user-friendly. Our team boasts extensive experience in the web3 space as both smart contract auditors and developers that have worked on top 100 blockchain projects with multi-million dollars in market capitalization.

Book an audit and learn more about us at [`shieldify.org`](https://shieldify.org/) or [`@ShieldifySec`](https://twitter.com/ShieldifySec)

# 2. Disclaimer

This security review does not guarantee bulletproof protection against a hack or exploit. Smart contracts are a novel technological feat with many known and unknown risks. The protocol, which this report is intended for, indemnifies Shieldify Security against any responsibility for any misbehavior, bugs, or exploits affecting the audited code during any part of the project's life cycle. It is also pivotal to acknowledge that modifications made to the audited code, including fixes for the issues described in this report, may introduce new problems and necessitate additional auditing.

# 3. About Raptures

Raptures is an innovative Web3 wallet experience, akin to Zapier, that transforms users' social media profiles into gateways for their Web3 wallets. The platform enables seamless composition of blockchain operations through user-linked plugins, simplifying the following objectives:

1. Intuitive Onboarding: Streamlining Web3 adoption for newcomers.
2. Dynamic Dapp Engagement: Fostering interaction and mutual discovery between dapp and target users.

Addressing key concerns:

1. Enhanced Security: Utilizing WebAuthn authentication mechanism for secure and familiar operation authorization and execution.
2. Streamlined Integration: Removing complexities of wallet SDKs, enabling biometric signatures which are then verified on-chain.
3. Simplicity in Operations: Users sign operations with biometric signatures simply through generated links, which then follows the Account Abstraction flow.

# 4. Risk Classification

| Severity               | Impact: High | Impact: Medium | Impact: Low |
| ---------------------- | ------------ | -------------- | ----------- |
| **Likelihood: High**   | Critical     | High           | Medium      |
| **Likelihood: Medium** | High         | Medium         | Low         |
| **Likelihood: Low**    | Medium       | Low            | Low         |

## 4.1 Impact

- **High** - results in a significant risk for the protocol’s overall well-being. Affects all or most users
- **Medium** - results in a non-critical risk for the protocol affects all or only a subset of users, but is still
  unacceptable
- **Low** - losses will be limited but bearable - and covers vectors similar to griefing attacks that can be easily repaired or even gas optimization techniques

## 4.2 Likelihood

- **High** - almost certain to happen and highly lucrative for execution by malicious actors
- **Medium** - still relatively likely, although only conditionally possible.
- **Low** - requires a unique set of circumstances and poses non-lucrative cost-of-execution to rewards ratio for the actor

# 5. Audit Summary

The audit duration lasted 4 days and a total of 96 hours have been spent by the three auditors - [`@ShieldifyMartin`](https://twitter.com/ShieldifyMartin), [`@ShieldifyAnon`](https://twitter.com/ShieldifyAnon) and [`@ShieldifyGhost`](https://twitter.com/ShieldifyGhost). This is the first audit for this specific part of the Account Abstraction Raptures protocol.

The project is a fork of the Account Abstraction architecture, which has been very thoroughly audited. The identified findings are primarily gas-optimization-related, together with some informational ones.

## 5.1 Protocol Summary

| **Project Name**             | Raptures                                                                                                                                                         |
| ---------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Repository**               | [`Account Abstarction Raptures`](https://github.com/madhav-madhusoodanan/account-abstraction-raptures/tree/d5bcc4b550d7a184561aae1c56f7741e90899868)             |
| **Type of Project**          | Project with Account Abstraction Mechanism                                                                                                                       |
| **Audit Timeline**           | 4 days                                                                                                                                                           |
| **Review Commit Hash**       | [`d5bcc4b550d7a184561aae1c56f7741e90899868`](https://github.com/madhav-madhusoodanan/account-abstraction-raptures/tree/d5bcc4b550d7a184561aae1c56f7741e90899868) |
| **Fixes Review Commit Hash** | N/A                                                                                                                                                              |

## 5.2 Scope

The following smart contracts were in the scope of the audit:

| File                                  | nSLOC |
| ------------------------------------- | ----- |
| contracts/samples/WebauthnAccount.sol | 157   |
| Total                                 | 157   |

# 6. Findings Summary

The following number of issues have been identified, sorted by their severity:

- **Critical** and **High** issues: 0
- **Medium** issues: 0
- **Low** issues: 1
- **Informational** issues: 6
- **Gas Optimization** issues: 3

| **ID** | **Title**                                                  | **Severity**     |
| ------ | ---------------------------------------------------------- | ---------------- |
| [L-01] | Missing Zero Address Check For All Functions               | Low              |
| [I-01] | Missing/Incomplete NatSpec Comments                        | Informational    |
| [I-02] | Change Function Visibility from `public` to `external`     | Informational    |
| [I-03] | Unused Variable                                            | Informational    |
| [I-04] | Function Ordering does not Follow the Solidity Style Guide | Informational    |
| [I-05] | Update External Dependency to the Latest Version           | Informational    |
| [I-06] | Use a More Recent Solidity Version                         | Informational    |
| [G-01] | No Need to Initialize Variables with Default Values        | Gas Optimization |
| [G-02] | Array Length Read in Each Iteration of the Loop Wastes Gas | Gas Optimization |
| [G-03] | Use Assembly to `emit` Events, in Order to Save Gas        | Gas Optimization |

# 7.Findings

# [L-01] Missing Zero Address Check For All Functions

## Severity

Low Risk

## Description

Contract `WebauthnAccount.sol` is missing address validation for all functions.
It is possible to configure the `address(0)`, which may cause issues during execution. Although functions like `execute` and `executeBatch` are callable primarily by the`EntryPoint` contract, the `_requireFromEntryPointOrOwner` check allow them to be called by the owner of the smart contract account directly, thus allowing the functions to accept the zero address as an argument for `dest`.

## Location of Affected Code

File: [`contracts/samples/WebauthnAccount.sol#L52`](https://github.com/madhav-madhusoodanan/account-abstraction-raptures/blob/d5bcc4b550d7a184561aae1c56f7741e90899868/contracts/samples/WebauthnAccount.sol#L52)

```solidity
constructor(IEntryPoint anEntryPoint) {
```

File: [`contracts/samples/WebauthnAccount.sol#L65`](https://github.com/madhav-madhusoodanan/account-abstraction-raptures/blob/d5bcc4b550d7a184561aae1c56f7741e90899868/contracts/samples/WebauthnAccount.sol#L86)

```solidity
function execute(
    address dest,
    uint256 value,
    bytes calldata func
) external {
```

File: [`contracts/samples/WebauthnAccount.sol#L73`](https://github.com/madhav-madhusoodanan/account-abstraction-raptures/blob/d5bcc4b550d7a184561aae1c56f7741e90899868/contracts/samples/WebauthnAccount.sol#L73)

```solidity
function executeBatch(
    address[] calldata dest,
    bytes[] calldata func
) external {
```

File: [`contracts/samples/WebauthnAccount.sol#L86`](https://github.com/madhav-madhusoodanan/account-abstraction-raptures/blob/d5bcc4b550d7a184561aae1c56f7741e90899868/contracts/samples/WebauthnAccount.sol#L86)

```solidity
function initialize(address anOwner) public virtual initializer {
```

File: [`contracts/samples/WebauthnAccount.sol#L208`](https://github.com/madhav-madhusoodanan/account-abstraction-raptures/blob/d5bcc4b550d7a184561aae1c56f7741e90899868/contracts/samples/WebauthnAccount.sol#L208)

```solidity
function withdrawDepositTo(
    address payable withdrawAddress,
    uint256 amount
) public onlyOwner {
```

## Recommendation

Add a zero-address check for all functions.

## Team Response

N/A

# [I-01] Missing/Incomplete NatSpec Comments

## Severity

Informational

## Description

(@notice, @dev, @param and @return) are missing in some functions. Given that NatSpec is an important part of code documentation, this affects code comprehension, audibility, and usability.

This might lead to confusion for other auditors/developers that are interacting with the code.

## Location of Affected Code

In all contracts

## Recommendation

Consider adding in full NatSpec comments for all functions where missing to have complete code documentation for future use.

## Team Response

N/A

# [I-02] Change Function Visibility from `public` to `external`

## Severity

Informational

## Description

It is best practice to mark functions that are not called internally as `external` instead.

## Location of Affected Code

Most functions

## Recommendation

Consider changing the visibility of functions that are not used with the contract from `public` to `external`.

## Team Response

N/A

# [I-03] Unused Variable

## Severity

Informational

## Description

`_hashIncluded` variable is imported but never used. Despite the fact that it is intended for future compatibility, it's a best practice to import it when it is needed.

## Location of Affected Code

File: WebauthnAccount

```solidity
(
    uint256 rs_1,
    uint256 rs_2,
    uint256 q_1,
    uint256 q_2,
    bytes32 _hashIncluded,
    bytes memory authenticatorData,
    bytes memory clientDataJSON
) = abi.decode(
    signatureField,
    (uint256, uint256, uint256, uint256, bytes32, bytes, bytes)
);
```

## Recommendation

Consider removing `_hashIncluded`.

## Team Response

N/A

# [I-04] Function Ordering does not Follow the Solidity Style Guide

## Severity

Informational

## Description

One of the guidelines mentioned in the style guide is to order functions in a specific way to improve readability and maintainability. By following this order, you can achieve a consistent and logical structure in your contract code.

## Location of Affected Code

File: EllipticCurve
File: WebauthnAccount

## Recommendation

It is recommended to follow the recommended order of functions in Solidity, as outlined in the [`Solidity style guide (LINK)`](https://docs.soliditylang.org/en/v0.8.17/style-guide.html#order-of-functions).

Functions should be grouped according to their visibility and ordered:

1. constructor
2. receive function (if exists)
3. fallback function (if exists)
4. external
5. public
6. internal
7. private

## Team Response

N/A

# [I-05] Update External Dependency to the Latest Version

## Severity

Informational

## Description

Update the versions `@openzeppelin/contracts` to be the latest in `package.json`.

## Location of Affected Code

According to package.json, `@openzeppelin/contracts`is currently set to `^4.2.0`

## Recommendation

I also recommend double-checking the versions of other dependencies as a precaution, as they may include important bug fixes.

## Team Response

N/A

# [I-06] Use a More Recent Solidity Version

## Severity:

Informational

## Description:

Currently, version `^0.8.12` is used across the whole codebase. Use the latest stable Solidity version to get all compiler features, bug fixes, and optimizations. However, when upgrading to a new Solidity version, it's crucial to carefully review the release notes, consider any breaking changes, and thoroughly test your code to ensure compatibility and correctness. Additionally, be aware that some features or changes may not be backward compatible, requiring adjustments in your code.

## Location of Affected Code

All of the smart contracts use a relatively old solidity version.

## Recommendation

Consider, upgrading all smart contracts to Solidity version `0.8.19`.

## Team Response

N/A

# [G-01] No Need to Initialize Variables with Default Values

## Severity

Gas Optimization

## Description

If a variable is not set/initialized, the default value is assumed (0, false, 0x0 … depending on the data type). Saves 8 gas per instance.

## Location of Affected Code

In all contracts where there is a for loop like this:

```diff
-- for (uint256 i = 0; ....
++ for (uint256 i; ....
```

## Recommendation

Do not initialize variables with their default values.

## Team Response

N/A

# [G-02] Array Length Read in Each Iteration of the Loop Wastes Gas

## Severity

Gas Optimization

## Description

Reading array length at each iteration of the loop takes `6 gas` (3 for mload and 3 to place memory_offset) in the stack.
Caching the array length in the stack saves around `3 gas` per iteration.

## Location of Affected Code

Files: WebauthnAccount.sol

## Recommendation

Cache the array length outside of the loop and use that variable in the loop.

Example:

```diff
+ uint256 _arrayLength = array.length;

- for (uint256 i; i < array.length; ....) {
+ for (uint256 i; i < _arrayLength; ....) {
```

## Team Response

N/A

# [G-03] Use Assembly to `emit` Events, in Order to Save Gas

## Severity

Gas Optimization

## Description

Using the scratch space for event arguments (two words or fewer) will save gas over needing Solidity's full abi memory expansion used for emitting normally.

## Location of Affected Code

Files: WebauthnAccount.sol

## Recommendation

Consider using assembly.

## Team Response

N/A
