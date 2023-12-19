# 1. About Shieldify

We are Shieldify Security – a company on a mission to make web3 protocols more secure, cost-efficient and user-friendly. Our team boasts extensive experience in the web3 space as both smart contract auditors and developers that have worked on top 100 blockchain projects with multi-million dollars in market capitalization.

Book an audit and learn more about us at [`shieldify.org`](https://shieldify.org/) or [`@ShieldifySec`](https://twitter.com/ShieldifySec)

# 2. Disclaimer

This security review does not guarantee bulletproof protection against a hack or exploit. Smart contracts are a novel technological feat with many known and unknown risks. The protocol, which this report is intended for, indemnifies Shieldify Security against any responsibility for any misbehavior, bugs, or exploits affecting the audited code during any part of the project's life cycle. It is also pivotal to acknowledge that modifications made to the audited code, including fixes for the issues described in this report, may introduce new problems and necessitate additional auditing.

# 3. About Dark Mythos

Dark Mythos is a fantasy trading card game that introduces NFT cards with a unique storytelling feature. NFT holders can unlock exclusive stories crafted by fantasy author Marco Dülk, delving deep into the lore of Dark Mythos and offering insights into the characters and their worlds. The game merges the thrill of collecting rare NFT cards with professionally written narratives, creating a personalized and immersive experience for fans of fantasy literature and trading card games. The stories are handcrafted by a skilled author to ensure authenticity and high literary quality, making each one a unique work of art.

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

# 5. Audit Summary

The audit lasted 3 days and a total of 96 hours were spent by the four auditors:

- [`@marcobesier`](https://twitter.com/marcobesier)
- [`@ShieldifyMartin`](https://twitter.com/ShieldifyMartin)
- [`@ShieldifyAnon`](https://twitter.com/ShieldifyAnon)
- [`@ShieldifyGhost`](https://twitter.com/ShieldifyGhost)

This is the first audit for the protocol's smart contract component, which represents a single NFT contract of the ERC-721 standard. Considering the small size codebase, the security review managed to identify issues, tackling the random number generation process and reentrancy, among other informational and gas optimization findings.

The NatSpec is comprehensive. The code's readability could be further improved via the implementation of the Informational findings (not included in the report), which also outline some foundational best practices.

We extend our gratitude to the Dark Mythos's blockchain team for their exemplary responsiveness, offering comprehensive clarifications and detailed responses to our inquiries.

We would also like to point out that the project's network of choice - IOTA's Shimmer, is still a relatively unexplored territory in terms of performance and network-level bugs and issues that might create additional attack surfaces.

## 5.1 Protocol Summary

| **Project Name**             | Dark Mythos                                                                                                                                                             |
| ---------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Repository**               | [`Dark-mythos`](https://gitlab.vettersolutions.de/dark-mythos/web3-academy-smart-contract/-/tree/master)                                                                |
| **Type of Project**          | ERC-721 collection                                                                                                                                                      |
| **Audit Timeline**           | 3 days                                                                                                                                                                  |
| **Review Commit Hash**       | [`45d9a7fbccb7a647f649a3a14f9d3f2bfa1c5f73`](https://gitlab.vettersolutions.de/dark-mythos/web3-academy-smart-contract/-/tree/45d9a7fbccb7a647f649a3a14f9d3f2bfa1c5f73) |
| **Fixes Review Commit Hash** | N/A                                                                                                                                                                     |

## 5.2 Scope

The following smart contracts were in the scope of the audit:

| File                     | nSLOC |
| ------------------------ | :---: |
| contracts/DarkMythos.sol |  143  |
| Total                    |  143  |

# 6. Findings Summary

The following number of issues have been identified, sorted by their severity:

- **Critical** and **High** issues: 0
- **Medium** issues: 3
- **Low** issues: 3

  | **ID** | **Title**                                                               | **Severity** |
  | :----: | ----------------------------------------------------------------------- | :----------: |
  | [M-01] | Insecure Generation of Randomness Used for Token Determination Logic    |    Medium    |
  | [M-02] | Using the `transfer` function of `address payable` is discouraged       |    Medium    |
  | [M-03] | Centralization Risk Due to Trusted Owner                                |    Medium    |
  | [L-01] | Missing Reentrancy Protection For `DarkMythos._mint()` Function         |     Low      |
  | [L-02] | Missing Zero Value Check for `_mintingCost` Might Lead to Loss of Funds |     Low      |
  | [L-03] | Ownership Role Transfer Function Implement Single-Step Role Transfer    |     Low      |

# 7. Findings

# [M-01] Insecure Generation of Randomness Used for Token Determination Logic

## Severity

Medium Risk

## Description

It generates a `uint256` random value that relies on variables like `block.timestamp`, `randomizationNonce` and `msg.sender` as a source of randomness is a common vulnerability, as the outcome can be influenced/predicted by miners/validators, but even normal users can easily replicate these three sources of entropy:

1. `block.timestamp` can be replicated inside an attacker contract if the attack transaction is included in the same block as the PRNG.
2. `randomizationNonce` can be predicted since the value is deterministically incremented by 1, and the previous values are publicly accessible (like any state variable of a smart contract on a public blockchain).
3. `msg.sender` can be replicated since it's simply the attacker contract's address.

## Location of Affected Code

File: [`contracts/DarkMythos.sol#L264-L265`](https://gitlab.vettersolutions.de/dark-mythos/web3-academy-smart-contract/-/blob/45d9a7fbccb7a647f649a3a14f9d3f2bfa1c5f73/contracts/DarkMythos.sol#L264-265)

```solidity
uint256 randomNumber = uint256(keccak256(abi.encodePacked(block.timestamp, randomizationNonce, msg.sender)));
uint256 randomIndex = randomNumber % tokenIdsToStartMinting.length;
```

## Recommendation

Consider using a decentralized oracle for the generation of random numbers, such as [Chainlink VRF](https://docs.chain.link/vrf/v2/introduction). It is important to take into account the `requestConfirmations` variable that will be used in the VRFv2Consumer contract when implementing VRF. The purpose of this value is to specify the minimum number of blocks you wish to wait before receiving randomness from the Chainlink VRF service.

## Team Response

Acknowledged.

# [M-02] Using the `transfer` function of `address payable` is discouraged

## Severity

Medium Risk

## Description

The `transfer()` function only allows the recipient to use `2300` gas. If the recipient uses more than that, transfers will fail. This could, for example, be the case if `vendor` is the address of a multisig or payment splitter that is supposed to execute additional logic after the withdrawal. Furthermore, gas costs might change in the future, increasing the likelihood of that happening. Also, notice that `vendor` is immutable after deployment.

Consider the following scenario:

1. During deployment, Dark Mythos is not aware of this "`transfer()` issue" and sets `vendor` to a contract address (e.g., a payment splitter) that consumes more than `2300` gas.
2. Dark Mythos launches the collection.
3. Users mint the entire collection.
4. Dark Mythos tries to withdraw the revenue from the contract. However, all of their attempts revert because the recipient consumes more than `2300` gas when receiving the funds.

Notice that Dark Mythos cannot reset `vendor` to another address (e.g., an externally owned account) because the contract does not provide such functionality. Therefore, all the revenue that Dark Mythos earned during the mint of the collection is stuck in the contract forever.

## Location of Affected Code

File: [`contracts/DarkMythos.sol#L230`](https://gitlab.vettersolutions.de/dark-mythos/web3-academy-smart-contract/-/blob/45d9a7fbccb7a647f649a3a14f9d3f2bfa1c5f73/contracts/DarkMythos.sol#L230)

```solidity
payable(vendor).transfer(address(this).balance);
```

## Recommendation

While this issue will never occur as long as `vendor` represents an externally owned account, Dark Mythos might _want_ to set `vendor` to a contract address during deployment. Therefore, we recommend using `call()` instead of `transfer()` to withdraw the contract's SMR balance because `call()` will forward all available gas instead of only `2300` gas.

```diff
- payable(vendor).transfer(address(this).balance);

+ (bool success, ) = msg.sender.call{value: address(this).balance}("");
+ require(success, "Withdrawal failed.")
```

## Team Response

Acknowledged and fixed.

# [M-03] Centralization Risk Due to Trusted Owner

## Severity

Medium Risk

## Description

The contract has an owner with the privileged right to pause and unpause most of the contract's functionality and therefore it needs to be trusted. Currently, the contract owner is not prevented from renouncing the ownership while the contract is paused, which could cause any user assets stored in the protocol, to be locked indefinitely.

Both `Dark Mythos` and `Shieldify` have been clear from the get-go that this functionality was only implemented because `EU` law (Article 30 of [`REPORT on the proposal for a regulation`](https://www.europarl.europa.eu/doceo/document/A-9-2023-0031_EN.html)) currently requires `that a mechanism exists to terminate the continued execution of transactions`. Nonetheless, Dark Mythos kindly asked Shieldify to incorporate this (hopefully temporary) issue here in the report to ensure Dark Mythos' users enjoy full transparency. **Dark Mythos intends to keep this privileged role only as long as the legal situation is not fully clarified. Furthermore, Dark Mythos has implemented a dedicated function to revoke their privileged role in the contract as soon as they are certain that they don't violate EU law by doing so.** Dark Mythos will make an effort to investigate this issue further and stay up to date with the latest legal developments.

## Location of Affected Code

File: [`contracts/DarkMythos.sol#L163`](https://gitlab.vettersolutions.de/dark-mythos/web3-academy-smart-contract/-/blob/45d9a7fbccb7a647f649a3a14f9d3f2bfa1c5f73/contracts/DarkMythos.sol#L163)

```solidity
function mint() external payable whenNotPaused {
```

File: [`contracts/DarkMythos.sol#L187`](https://gitlab.vettersolutions.de/dark-mythos/web3-academy-smart-contract/-/blob/45d9a7fbccb7a647f649a3a14f9d3f2bfa1c5f73/contracts/DarkMythos.sol#L187)

```solidity
function mintBulk(uint256 _bulkAmount) external payable whenNotPaused {
```

## Recommendation

It is recommended that the client carefully manages the private key of the controller account to avoid any potential hacking risk. Measures that can be taken are to enhance centralized privileges and roles in the protocol through a decentralized mechanism or module-based accounts with enhanced security practices. We propose to make the owner of `DarkMythos` a multi-sig wallet behind a `Timelock` contract so that users can monitor what transactions are about to be executed by this account and take action if necessary.

## Team Response

Acknowledged.

# [L-01] Missing Reentrancy Protection For `DarkMythos._mint()` Function

## Severity

Low Risk

## Description

A potential threat emerges from the `mint()` function, as it internally calls `safeMint()`, which triggers the `onERC721Received` callback. This could potentially execute malicious code, allowing an attacker to claim all tokens. It's important to be aware that the `mintingCost` will be paid for each iteration.

## Location of Affected Code

File: [`contracts/DarkMythos.sol#L146-149`](https://gitlab.vettersolutions.de/dark-mythos/web3-academy-smart-contract/-/blob/45d9a7fbccb7a647f649a3a14f9d3f2bfa1c5f73/contracts/DarkMythos.sol#L146-149)

```solidity
for (uint256 i = tokenIdToStartMinting; i < tokenIdToStartMinting + numberOfTokensPerMint; i++) {
  _safeMint(msg.sender, i);
  mintedTokenIds[i - tokenIdToStartMinting] = i;
}
```

## Recommendation

To protect against cross-function reentrancy attacks, OpenZeppelin's `nonReentrant` modifier that guards the decorated function with a mutex against reentrancy attacks should be applied. It's also a best practice to follow the CEI (Checks-Effects-Interactions) pattern.

File: [`contracts/DarkMythos.sol#L163`](https://gitlab.vettersolutions.de/dark-mythos/web3-academy-smart-contract/-/blob/45d9a7fbccb7a647f649a3a14f9d3f2bfa1c5f73/contracts/DarkMythos.sol#L163)

```diff
+ import {ReentrancyGuard} from "@openzeppelin/contracts/security/ReentrancyGuard.sol";

+ contract DarkMythos is ERC721, ERC721Enumerable, Pausable, Ownable, ReentrancyGuard {

.
.

- function mint() external payable whenNotPaused
+ function mint() external payable nonReentrant whenNotPaused {

.
.

- function mintBulk(uint256 _bulkAmount) external payable whenNotPaused {
+ function mintBulk(uint256 _bulkAmount) external payable nonReentrant whenNotPaused {
}
```

File: [`contracts/DarkMythos.sol#L146-149`](https://gitlab.vettersolutions.de/dark-mythos/web3-academy-smart-contract/-/blob/45d9a7fbccb7a647f649a3a14f9d3f2bfa1c5f73/contracts/DarkMythos.sol#L146-149)

```diff
for (uint256 i = tokenIdToStartMinting; i < tokenIdToStartMinting + numberOfTokensPerMint; i++) {
- _safeMint(msg.sender, i);
  mintedTokenIds[i - tokenIdToStartMinting] = i;
+ _safeMint(msg.sender, i);
}
```

## Team Response

Acknowledged and fixed.

# [L-02] Missing Zero Value Check for `_mintingCost` Might Lead to Loss of Funds

## Severity

Low Risk

## Description

The `_mintingCost` variable in the constructor is missing a zero-value check. This variable serves as a fundamental parameter in the protocol's operation, dictating the cost associated with minting tokens. If it is set to 0 by mistake, the protocol's business logic could be severely impacted, as there will be no minting tax fees.

## Location of Affected Code

File: [`contracts/DarkMythos.sol#L77`](https://gitlab.vettersolutions.de/dark-mythos/web3-academy-smart-contract/-/blob/45d9a7fbccb7a647f649a3a14f9d3f2bfa1c5f73/contracts/DarkMythos.sol#L77)

```solidity
constructor(
  string memory _name,
  string memory _symbol,
  string memory _baseURI_,
  uint256 _mintingCost,
  uint256 _numberOfTokensPerMint,
  uint256 _maxBulkBuy,
  uint256 _maxMints,
  uint256 _allowMintingAfter,
  address _vendor
)
  ERC721(_name, _symbol)
{

.
.

  mintingCost = _mintingCost;
}
```

## Recommendation

To address this vulnerability, consider adding a check so that it is not possible for `_mintingCost` to be `0`.

```diff
constructor(
  string memory _name,
  string memory _symbol,
  string memory _baseURI_,
  uint256 _mintingCost,
  uint256 _numberOfTokensPerMint,
  uint256 _maxBulkBuy,
  uint256 _maxMints,
  uint256 _allowMintingAfter,
  address _vendor
)
  ERC721(_name, _symbol)
{
+ require(_mintingCost != 0, "@dev: mintingCost must not equal zero");

.
.

  mintingCost = _mintingCost;
}
```

## Team Response

Acknowledged and fixed.

# [L-03] Ownership Role Transfer Function Implement Single-Step Role Transfer

## Severity

Low Risk

## Description

The current ownership transfer process for all the contracts inheriting from `Ownable` involves the current owner calling the `transferOwnership()` function. If the nominated EOA account is not a valid account, it is entirely possible that the owner may accidentally transfer ownership to an uncontrolled account, losing access to all functions with the `onlyOwner` modifier.

## Location of Affected Code

File: [`contracts/DarkMythos.sol#L24`](https://gitlab.vettersolutions.de/dark-mythos/web3-academy-smart-contract/-/blob/45d9a7fbccb7a647f649a3a14f9d3f2bfa1c5f73/contracts/DarkMythos.sol#L24)

## Recommendation

It is recommended to implement a two-step process where the owner nominates an account and the nominated account needs to call an `acceptOwnership()` function for the transfer of the ownership to fully succeed. This ensures the nominated EOA account is a valid and active account. This can be easily achieved by using OpenZeppelin’s `Ownable2Step` contract instead of `Ownable`.

```diff
- import "@openzeppelin/contracts/access/Ownable.sol";
+ import {Ownable2Step} from "@openzeppelin/contracts/access/Ownable2Step.sol";

- contract DarkMythos is ERC721, ERC721Enumerable, Pausable, Ownable {
+ contract DarkMythos is ERC721, ERC721Enumerable, Pausable, Ownable2Step {
```

## Team Response

Acknowledged and fixed.

# [I-01] Missing Event Emission in `withdraw()` function

## Severity

Informational

## Description

It has been observed that important functionality is missing an emitting event for `withdraw()` function.
Events are a method of informing the transaction initiator about the actions taken by the called function. An
event logs its emitted parameters in a specific log history, which can be accessed outside of the contract using some filter parameters.

## Location of Affected Code

File: `contracts/DarkMythos.sol`

```solidity
function withdraw() external {
    require(msg.sender == vendor, "You aren't the vendor");
    payable(vendor).transfer(address(this).balance);
}
```

## Recommendation

For best security practices, consider declaring events as often as possible at the end of a function. Events can be used to detect the end of the operation.

## Team Response

N/A

# [I-02] Update External Dependency to the Latest Version

## Severity

Informational

## Description

Update the versions `@openzeppelin/contracts` to be the latest in `package.json`.

## Location of Affected Code

According to package.json, `@openzeppelin/contracts` is currently set to ^4.8.3.

## Recommendation

I also recommend double-checking the versions of other dependencies as a precaution, as they may include important bug fixes.

## Team Response

N/A

# [I-03] `_mint()` is Redundant

## Severity

Informational

## Description

Calling `mint()` is equivalent to calling `mintBulk(1)`. Therefore, the `mint()` function is redundant.

## Location of Affected Code

File: `contracts/DarkMythos.sol#L163-L171`

```solidity
function mint() external payable whenNotPaused {
    require(block.timestamp >= allowMintingAfter, "Minting is not allowed yet");
    require(msg.value == mintingCost, "User must pay the correct price");

    uint256 totalSupply = totalSupply();
    require(totalSupply < maxTokenSupply, "All tokens have been minted");

    _mint();
}
```

## Recommendation

Remove the `mint()` function. Not only will this make the code cleaner, but it will also make the contract more gas-efficient.

File: `contracts/DarkMythos.sol#L163-L171`

```diff
- function mint() external payable whenNotPaused {
-     require(block.timestamp >= allowMintingAfter, "Minting is not allowed yet");
-     require(msg.value == mintingCost, "User must pay the correct price");
-
-     uint256 totalSupply = totalSupply();
-     require(totalSupply < maxTokenSupply, "All tokens have been minted");
-
-     _mint();
- }
```

## Team Response

N/A

# [I-04] Fix Indentation

## Severity

Informational

## Description

The contract contains several instances of inconsistent indentation.

## Location of Affected Code

File: `contracts/DarkMythos.sol#L139-L142`

```solidity
function _mint() internal {
uint256 randomIndex = _randomIndex();
uint256 tokenIdToStartMinting = tokenIdsToStartMinting[randomIndex];
    tokenIdsToStartMinting[randomIndex] = tokenIdsToStartMinting[tokenIdsToStartMinting.length - 1];
```

File: `contracts/DarkMythos.sol#L187-L198`

```solidity
function mintBulk(uint256 _bulkAmount) external payable whenNotPaused {
    require(block.timestamp >= allowMintingAfter, "Minting is not allowed yet");
    require(_bulkAmount <= maxBulkBuy, "Exceeds max bulk buy limit");
    require(msg.value == mintingCost * _bulkAmount, "User must pay the correct price");

    uint256 newTotalSupply = totalSupply() + (_bulkAmount * numberOfTokensPerMint);
    require(newTotalSupply < maxTokenSupply, "Exceeds max token supply");

    for (uint256 i = 0; i < _bulkAmount; i++) {
        _mint();
    }
  }
```

File: `contracts/DarkMythos.sol#L219-L221`

```solidity
function viewBalance() external view returns (uint256) {
  return address(this).balance;
}
```

## Recommendation

Use consistent indentation (4 spaces) throughout the contract.

File: `contracts/DarkMythos.sol#L139-L142`

```diff
- function _mint() internal {
- uint256 randomIndex = _randomIndex();
- uint256 tokenIdToStartMinting = tokenIdsToStartMinting[randomIndex];
-     tokenIdsToStartMinting[randomIndex] = tokenIdsToStartMinting[tokenIdsToStartMinting.length - 1];
+ function _mint() internal {
+     uint256 randomIndex = _randomIndex();
+     uint256 tokenIdToStartMinting = tokenIdsToStartMinting[randomIndex];
+     tokenIdsToStartMinting[randomIndex] = tokenIdsToStartMinting[tokenIdsToStartMinting.length - 1];
```

File: `contracts/DarkMythos.sol#L187-L198`

```diff
- function mintBulk(uint256 _bulkAmount) external payable whenNotPaused {
-       require(block.timestamp >= allowMintingAfter, "Minting is not allowed yet");
-       require(_bulkAmount <= maxBulkBuy, "Exceeds max bulk buy limit");
-       require(msg.value == mintingCost * _bulkAmount, "User must pay the correct price");
-
-       uint256 newTotalSupply = totalSupply() + (_bulkAmount * numberOfTokensPerMint);
-       require(newTotalSupply < maxTokenSupply, "Exceeds max token supply");
-
-       for (uint256 i = 0; i < _bulkAmount; i++) {
-           _mint();
-       }
-   }
+ function mintBulk(uint256 _bulkAmount) external payable whenNotPaused {
+     require(block.timestamp >= allowMintingAfter, "Minting is not allowed yet");
+     require(_bulkAmount <= maxBulkBuy, "Exceeds max bulk buy limit");
+     require(msg.value == mintingCost * _bulkAmount, "User must pay the correct price");
+
+     uint256 newTotalSupply = totalSupply() + (_bulkAmount * numberOfTokensPerMint);
+     require(newTotalSupply < maxTokenSupply, "Exceeds max token supply");
+
+     for (uint256 i = 0; i < _bulkAmount; i++) {
+         _mint();
+     }
+ }
```

File: `contracts/DarkMythos.sol#L219-L221`

```diff
- function viewBalance() external view returns (uint256) {
-   return address(this).balance;
- }
+ function viewBalance() external view returns (uint256) {
+     return address(this).balance;
+ }
```

## Team Response

N/A

# [I-05] Choose the Proper Functions Visibility

## Severity

Informational

## Description

It is best practice to mark functions that are not called internally as `external` instead of `public` and `private` instead of `internal` when they should only be called inside the given contract.

## Location of Affected Code

File: [`contracts/DarkMythos.sol#L236`](https://gitlab.vettersolutions.de/dark-mythos/web3-academy-smart-contract/-/blob/master/contracts/DarkMythos.sol#L236)

```solidity
function pause() public onlyOwner { _pause(); }
```

File: [`contracts/DarkMythos.sol#L241`](https://gitlab.vettersolutions.de/dark-mythos/web3-academy-smart-contract/-/blob/master/contracts/DarkMythos.sol#L241)

```solidity
function unpause() public onlyOwner { _unpause(); }
```

File: [`contracts/DarkMythos.sol#L139`](https://gitlab.vettersolutions.de/dark-mythos/web3-academy-smart-contract/-/blob/master/contracts/DarkMythos.sol#L139)

```solidity
function _mint() internal {
```

File: [`contracts/DarkMythos.sol#L263`](https://gitlab.vettersolutions.de/dark-mythos/web3-academy-smart-contract/-/blob/master/contracts/DarkMythos.sol#L263)

```solidity
function _randomIndex() internal returns (uint256) {
```

## Recommendation

- Consider using `external` modifier for clarity's sake if the function is not called inside the contract.
- Consider using `private` modifier for clarity's sake if the function can only be called inside the given contract.

```diff
- function pause() public onlyOwner { _pause(); }
- function unpause() public onlyOwner { _unpause(); }
- function _mint() internal {
- function _randomIndex() internal returns (uint256) {

+ function pause() external onlyOwner { _pause(); }
+ function unpause() external onlyOwner { _unpause(); }
+ function _mint() private {
+ function _randomIndex() private returns (uint256) {
```

## Team Response

N/A

# [I-06] Use Locked Solidity Version Pragma

## Severity

Informational

## Description

Currently, version ^0.8.19 is used in the codebase. Contracts should be deployed with the same compiler version that they have been tested with. Locking the pragma helps to ensure that contracts do not accidentally get deployed using, for example, either an outdated compiler version that might introduce bugs or a compiler version too recent that has not been extensively tested yet.

## Location of Affected Code

File: [`contracts/DarkMythos.sol`](https://gitlab.vettersolutions.de/dark-mythos/web3-academy-smart-contract/-/blob/master/contracts/DarkMythos.sol)

## Recommendation

Consider locking the version pragma to the same Solidity version used during development and testing (0.8.19).

## Team Response

N/A

# [G-01] No Need to Initialize Variables with Default Values

## Severity

Gas Optimization

## Description

If a variable is not set/initialized, the default value is assumed (0, false, 0x0 … depending on the data type). Saves 8 gas per instance.

## Location of Affected Code

File: `contracts/DarkMythos.sol`

```solidity
112: for (uint256 i = 0; i < _maxMints; i++) {
195: for (uint256 i = 0; i < _bulkAmount; i++) {
209: for (uint256 i = 0; i < ownerTokenCount; i++) {
```

## Recommendation

Do not initialize variables with their default values.

```diff
- for (uint256 i = 0; ...
+ for (uint256 i; ...
```

## Team Response

N/A

# [G-02] Write gas-optimal for-loops

## Severity

Gas optimization

## Description

Some for-loops are not written in a gas-optimal way.

More specifically, they use post-increment instead of pre-increment and implicitly check for overflows/underflows where they are not possible.

## Location of Affected Code

File: `contracts/DarkMythos.sol#L112-L114`

```solidity
for (uint256 i = 0; i < _maxMints; i++) {
    tokenIdsToStartMinting.push(i * numberOfTokensPerMint + 1);
}
```

File: `contracts/DarkMythos.sol#L146-L149`

```solidity
for (uint256 i = tokenIdToStartMinting; i < tokenIdToStartMinting + numberOfTokensPerMint; i++) {
    _safeMint(msg.sender, i);
    mintedTokenIds[i - tokenIdToStartMinting] = i;
}
```

File: `contracts/DarkMythos.sol#L195-L197`

```solidity
for (uint256 i = 0; i < _bulkAmount; i++) {
    _mint();
}
```

File: `contracts/DarkMythos.sol#L209-L211`

```solidity
for (uint256 i = 0; i < ownerTokenCount; i++) {
    tokenIds[i] = tokenOfOwnerByIndex(_owner, i);
}
```

## Recommendation

A gas-optimal for-loop looks like this:

```solidity
for (uint256 i; i < limit; ) {

    // inside the loop

    unchecked {
        ++i;
    }
}
```

The two differences here from a conventional for loop are that `i++` becomes `++i`, and that the loop variable's increment is unchecked because the `limit` variable ensures it won’t overflow. First, `++i` and `i++` are equivalent operations in the for-loop's logic, but `++i` is more gas-efficient than `i++`. Second, placing the increment inside `unchecked` will skip the superfluous check for an arithmetic overflow. Notice, however, that the general pattern above doesn't include any further optimizations that might be possible _inside_ the loop.

In the specific case of the Dark Mythos contract, one can implement the following changes to save gas:

File: `contracts/DarkMythos.sol#L112-L114`

```diff
- for (uint256 i = 0; i < _maxMints; i++) {
-     tokenIdsToStartMinting.push(i * numberOfTokensPerMint + 1);
- }
+ for (uint256 i; i < _maxMints; ) {
+     unchecked {
+ 	tokenIdsToStartMinting.push(i * numberOfTokensPerMint + 1);
+ 	++i;
+     }
+ }
```

File: `contracts/DarkMythos.sol#L146-L149`

```diff
- for (uint256 i = tokenIdToStartMinting; i < tokenIdToStartMinting + numberOfTokensPerMint; i++) {
-     _safeMint(msg.sender, i);
-     mintedTokenIds[i - tokenIdToStartMinting] = i;
- }
+ for (uint256 i = tokenIdToStartMinting; i < tokenIdToStartMinting + numberOfTokensPerMint; ) {
+     _safeMint(msg.sender, i);
+     unchecked {
+ 	mintedTokenIds[i - tokenIdToStartMinting] = i;
+ 	++i;
+     }
+ }
```

File: `contracts/DarkMythos.sol#L195-L197`

```diff
- for (uint256 i = 0; i < _bulkAmount; i++) {
-     _mint();
- }
+ for (uint256 i; i < _bulkAmount; ) {
+     _mint();
+     unchecked {
+ 	++i;
+     }
+ }
```

File: `contracts/DarkMythos.sol#L209-L211`

```diff
- for (uint256 i = 0; i < ownerTokenCount; i++) {
-     tokenIds[i] = tokenOfOwnerByIndex(_owner, i);
- }
+ for (uint256 i; i < ownerTokenCount; ) {
+     tokenIds[i] = tokenOfOwnerByIndex(_owner, i);
+     unchecked {
+ 	++i;
+     }
+ }
```

Implementing this change will save 1762 gas on method calls and 11812 gas during deployment.

```diff
diff --git a/gas-report.md b/gas-report.md
index 47461ae..32f3f4d 100644
--- a/gas-report.md
+++ b/gas-report.md
@@ -7,11 +7,11 @@
 ···············|················|··············|·············|·············|···············|··············
 |  DarkMythos  ·  approve       ·           -  ·          -  ·      49333  ·            2  ·          -  │
 ···············|················|··············|·············|·············|···············|··············
-|  DarkMythos  ·  mint          ·      744714  ·     761014  ·     754114  ·           20  ·          -  │
+|  DarkMythos  ·  mint          ·      742872  ·     763972  ·     755152  ·           20  ·          -  │
 ···············|················|··············|·············|·············|···············|··············
 |  DarkMythos  ·  pause         ·           -  ·          -  ·      28169  ·            2  ·          -  │
 ···············|················|··············|·············|·············|···············|··············
-|  DarkMythos  ·  transferFrom  ·           -  ·          -  ·      99020  ·            1  ·          -  │
+|  DarkMythos  ·  transferFrom  ·           -  ·          -  ·      96220  ·            1  ·          -  │
 ···············|················|··············|·············|·············|···············|··············
 |  DarkMythos  ·  unpause       ·           -  ·          -  ·      28211  ·            1  ·          -  │
 ···············|················|··············|·············|·············|···············|··············
@@ -19,5 +19,5 @@
 ···············|················|··············|·············|·············|···············|··············
 |  Deployments                  ·                                          ·  % of limit   ·             │
 ································|··············|·············|·············|···············|··············
-|  DarkMythos                   ·           -  ·          -  ·    4201984  ·         14 %  ·          -  │
+|  DarkMythos                   ·           -  ·          -  ·    4190172  ·         14 %  ·          -  │
 ·-------------------------------|--------------|-------------|-------------|---------------|-------------·
```

## Team Response

N/A

# [G-03] Strict Inequalities Can Sometimes Be Cheaper Than Non-strict Inequalities

## Severity

Gas optimization

## Description

The contract has two instances of a non-strict inequality (`>=`) where a strict inequality (`>`) would suffice.

## Location of Affected Code

File: `contracts/DarkMythos.sol#L164`

```solidity
require(block.timestamp >= allowMintingAfter, "Minting is not allowed yet");
```

File: `contracts/DarkMythos.sol#L188`

```solidity
require(block.timestamp >= allowMintingAfter, "Minting is not allowed yet");
```

## Recommendation

It is generally recommended to use strict inequalities (`<`, `>`) over non-strict inequalities (`<=`, `>=`). This is because the compiler will sometimes change `a > b` to `!(a < b)` to accomplish the non-strict inequality. The EVM does not have an opcode for checking less-than-or-equal to or greater-than-or-equal to.

However, one should try both comparisons because it is not always the case that using strict inequality will save gas. This is very dependent on the context of the surrounding opcodes.

In the case of the current version of the Dark Mythos contract, making the changes below results in savings of 3 gas on method calls and 456 gas during deployment.

File: `contracts/DarkMythos.sol#L164`

```diff
- require(block.timestamp >= allowMintingAfter, "Minting is not allowed yet");
+ require(block.timestamp > allowMintingAfter, "Minting is not allowed yet");
```

File: `contracts/DarkMythos.sol#L188`

```diff
- require(block.timestamp >= allowMintingAfter, "Minting is not allowed yet");
+ require(block.timestamp > allowMintingAfter, "Minting is not allowed yet");
```

```diff
diff --git a/gas-report.md b/gas-report.md
index 316681a..ea52700 100644
--- a/gas-report.md
+++ b/gas-report.md
@@ -7,7 +7,7 @@
 ···············|················|··············|·············|·············|···············|··············
 |  DarkMythos  ·  approve       ·           -  ·          -  ·      49333  ·            2  ·          -  │
 ···············|················|··············|·············|·············|···············|··············
-|  DarkMythos  ·  mint          ·      744714  ·     765814  ·     756994  ·           20  ·          -  │
+|  DarkMythos  ·  mint          ·      744711  ·     765811  ·     756991  ·           20  ·          -  │
 ···············|················|··············|·············|·············|···············|··············
 |  DarkMythos  ·  pause         ·           -  ·          -  ·      28169  ·            2  ·          -  │
 ···············|················|··············|·············|·············|···············|··············
@@ -19,5 +19,5 @@
 ···············|················|··············|·············|·············|···············|··············
 |  Deployments                  ·                                          ·  % of limit   ·             │
 ································|··············|·············|·············|···············|··············
-|  DarkMythos                   ·           -  ·          -  ·    4201984  ·         14 %  ·          -  │
+|  DarkMythos                   ·           -  ·          -  ·    4201528  ·         14 %  ·          -  │
 ·-------------------------------|--------------|-------------|-------------|---------------|-------------·
```

## Team Response

N/A

# [G-04] Use Custom Errors Instead of `require` and Split Checks with Boolean Expressions

## Severity

Gas optimization

## Description

The contract uses `require` instead of custom errors. However, custom errors can be both more comprehensive and more gas-efficient. Furthermore, the `require` statements contain boolean `&&` expressions, which are fully evaluated before the revert - even if the first condition of the `&&` already failed.

## Location of Affected Code

File: `contracts/DarkMythos.sol#L90-L103`

```solidity
require(
  _maxMints > 0 && _numberOfTokensPerMint > 0,
  "@dev: maxTokenSupply must not equal zero"
);

require(
    _maxBulkBuy > 0 && _maxBulkBuy <= _maxMints,
    "@dev: maxBulkBuy must be greater than zero and less than or equal to maxMints"
);

require(
    _vendor != address(0),
    "@dev: vendor must not be the zero address"
);
```

File: `contracts/DarkMythos.sol#L164-L168`

```solidity
require(block.timestamp >= allowMintingAfter, "Minting is not allowed yet");
require(msg.value == mintingCost, "User must pay the correct price");

uint256 totalSupply = totalSupply();
require(totalSupply < maxTokenSupply, "All tokens have been minted");
```

File: `contracts/DarkMythos.sol#L188-L193`

```solidity
require(block.timestamp >= allowMintingAfter, "Minting is not allowed yet");
require(_bulkAmount <= maxBulkBuy, "Exceeds max bulk buy limit");
require(msg.value == mintingCost * _bulkAmount, "User must pay the correct price");

uint256 newTotalSupply = totalSupply() + (_bulkAmount * numberOfTokensPerMint);
require(newTotalSupply < maxTokenSupply, "Exceeds max token supply");
```

File: `contracts/DarkMythos.sol#L229`

```solidity
require(msg.sender == vendor, "You aren't the vendor");
```

## Recommendation

Use custom errors instead of `require` statements. Furthermore, avoid boolean `&&` expressions and instead perform the checks separately from one another to ensure a potential revert happens as early as possible.

File: `contracts/DarkMythos.sol#L37`

```diff
event TokensMinted(address indexed owner, uint256[] tokenIds);

+ error maxMintsIsZero();
+ error numberOfTokensPerMintIsZero();
+ error maxBulkBuyIsZero();
+ error maxBulkBuyIsGreaterThanMaxMints();
+ error vendorIsZeroAddress();
+ error mintingNotAllowedYet();
+ error userMustPayCorrectPrice();
+ error allTokensHaveBeenMinted();
+ error requestedBulkAmountExceedsLimit();
+ error notEnoughTokensLeft();
+ error invalidVendor();
```

File: `contracts/DarkMythos.sol#L90-L103`

```diff
- require(
-   _maxMints > 0 && _numberOfTokensPerMint > 0,
-   "@dev: maxTokenSupply must not equal zero"
- );
-
- require(
-     _maxBulkBuy > 0 && _maxBulkBuy <= _maxMints,
-     "@dev: maxBulkBuy must be greater than zero and less than or equal to maxMints"
- );
-
- require(
-     _vendor != address(0),
-     "@dev: vendor must not be the zero address"
- );
+ if (_maxMints == 0) {
+     revert maxMintsIsZero();
+ }
+
+ if (_numberOfTokensPerMint == 0) {
+     revert numberOfTokensPerMintIsZero();
+ }
+
+ if (_maxBulkBuy == 0) {
+     revert maxBulkBuyIsZero();
+ }
+
+ if (_maxBulkBuy > _maxMints) {
+     revert maxBulkBuyIsGreaterThanMaxMints();
+ }
+
+ if (_vendor == address(0)) {
+     revert vendorIsZeroAddress();
+ }
```

File: `contracts/DarkMythos.sol#L164-L168`

```diff
- require(block.timestamp >= allowMintingAfter, "Minting is not allowed yet");
- require(msg.value == mintingCost, "User must pay the correct price");
-
- uint256 totalSupply = totalSupply();
- require(totalSupply < maxTokenSupply, "All tokens have been minted");
+ if (block.timestamp < allowMintingAfter) {
+     revert mintingNotAllowedYet();
+ }
+
+ if (msg.value != mintingCost) {
+     revert userMustPayCorrectPrice();
+ }
+
+ uint256 totalSupply = totalSupply();
+ if (totalSupply == maxTokenSupply) {
+     revert allTokensHaveBeenMinted();
+ }
```

File: `contracts/DarkMythos.sol#L188-L193`

```diff
- require(block.timestamp >= allowMintingAfter, "Minting is not allowed yet");
- require(_bulkAmount <= maxBulkBuy, "Exceeds max bulk buy limit");
- require(msg.value == mintingCost * _bulkAmount, "User must pay the correct price");
-
- uint256 newTotalSupply = totalSupply() + (_bulkAmount * numberOfTokensPerMint);
- require(newTotalSupply < maxTokenSupply, "Exceeds max token supply");
+ if (block.timestamp < allowMintingAfter) {
+     revert mintingNotAllowedYet();
+ }
+
+ if (_bulkAmount > maxBulkBuy) {
+     revert requestedBulkAmountExceedsLimit();
+ }
+
+ if (msg.value != mintingCost * _bulkAmount) {
+     revert userMustPayCorrectPrice();
+ }
+
+ uint256 newTotalSupply = totalSupply() + (_bulkAmount * numberOfTokensPerMint);
+
+ if (newTotalSupply >= maxTokenSupply) {
+     revert notEnoughTokensLeft();
+ }
```

File: `contracts/DarkMythos.sol#L229`

```diff
- require(msg.sender == vendor, "You aren't the vendor");
+ if (msg.sender != vendor) {
+     revert invalidVendor();
+ }
```

Implementing these changes will save 1920 gas on method calls and 161653 gas during deployment.

```diff
diff --git a/gas-report.md b/gas-report.md
index 3af2a0b..df82f1f 100644
--- a/gas-report.md
+++ b/gas-report.md
@@ -7,7 +7,7 @@
 ···············|················|··············|·············|·············|···············|··············
 |  DarkMythos  ·  approve       ·           -  ·          -  ·      49333  ·            2  ·          -  │
 ···············|················|··············|·············|·············|···············|··············
-|  DarkMythos  ·  mint          ·      744714  ·     765814  ·     756034  ·           20  ·          -  │
+|  DarkMythos  ·  mint          ·      744714  ·     761014  ·     754114  ·           20  ·          -  │
 ···············|················|··············|·············|·············|···············|··············
 |  DarkMythos  ·  pause         ·           -  ·          -  ·      28169  ·            2  ·          -  │
 ···············|················|··············|·············|·············|···············|··············
@@ -19,5 +19,5 @@
 ···············|················|··············|·············|·············|···············|··············
 |  Deployments                  ·                                          ·  % of limit   ·             │
 ································|··············|·············|·············|···············|··············
-|  DarkMythos                   ·           -  ·          -  ·    4201984  ·         14 %  ·          -  │
+|  DarkMythos                   ·           -  ·          -  ·    4040331  ·       13.5 %  ·          -  │
 ·-------------------------------|--------------|-------------|-------------|---------------|-------------·
```

## Team Response

N/A

# [G-05] Read and Write Storage Variables Exactly Once

## Severity

Gas optimization

## Description

`randomizationNonce` is read from storage more than once.

## Location of Affected Code

File: `contracts/DarkMythos.sol#L264-L267`

```solidity
uint256 randomNumber = uint256(keccak256(abi.encodePacked(block.timestamp, randomizationNonce, msg.sender)));
uint256 randomIndex = randomNumber % tokenIdsToStartMinting.length;

randomizationNonce++;
```

## Recommendation

Reading from a storage variable costs at least 100 gas as Solidity does not cache the storage read. Writes are considerably more expensive. Therefore, one can save gas by manually caching the variable to do exactly one storage read and exactly one storage write.

We recommend manually caching `randomizationNonce` as follows:

File: `contracts/DarkMythos.sol#L264-L267`

```diff
- uint256 randomNumber = uint256(keccak256(abi.encodePacked(block.timestamp, randomizationNonce, msg.sender)));
- uint256 randomIndex = randomNumber % tokenIdsToStartMinting.length;
-
- randomizationNonce++;
+ uint256 tempRandomizationNonce = randomizationNonce;
+ uint256 randomNumber = uint256(keccak256(abi.encodePacked(block.timestamp, tempRandomizationNonce, msg.sender)));
+ uint256 randomIndex = randomNumber % tokenIdsToStartMinting.length;
+
+ randomizationNonce = tempRandomizationNonce + 1;
```

While implementing this change will increase deployment costs by 192 gas, it will save 2912 gas on the method calls.

```diff
diff --git a/gas-report.md b/gas-report.md
index 3af2a0b..a0dae83 100644
--- a/gas-report.md
+++ b/gas-report.md
@@ -7,7 +7,7 @@
 ···············|················|··············|·············|·············|···············|··············
 |  DarkMythos  ·  approve       ·           -  ·          -  ·      49333  ·            2  ·          -  │
 ···············|················|··············|·············|·············|···············|··············
-|  DarkMythos  ·  mint          ·      744714  ·     765814  ·     756034  ·           20  ·          -  │
+|  DarkMythos  ·  mint          ·      744672  ·     760972  ·     753112  ·           20  ·          -  │
 ···············|················|··············|·············|·············|···············|··············
 |  DarkMythos  ·  pause         ·           -  ·          -  ·      28169  ·            2  ·          -  │
 ···············|················|··············|·············|·············|···············|··············
@@ -19,5 +19,5 @@
 ···············|················|··············|·············|·············|···············|··············
 |  Deployments                  ·                                          ·  % of limit   ·             │
 ································|··············|·············|·············|···············|··············
-|  DarkMythos                   ·           -  ·          -  ·    4201984  ·         14 %  ·          -  │
+|  DarkMythos                   ·           -  ·          -  ·    4202176  ·         14 %  ·          -  │
 ·-------------------------------|--------------|-------------|-------------|---------------|-------------·
```

## Team Response

N/A

# [G-06] Use `!= 0` instead of `> 0` for unsigned integer comparison

## Severity

Gas Optimization

## Description

In the context of unsigned integer types, opting for the `!= 0` comparison over `> 0` is typically more gas-efficient. This preference arises from the compiler's capacity to optimize the `!= 0` comparison into a straightforward bitwise operation. In contrast, the `> 0` comparison necessitates an extra subtraction operation. Consequently, choosing `!= 0` can enhance gas efficiency and contribute to lowering the overall contract expenditure.

## Location of Affected Code

File: `contracts/DarkMythos.sol`

```solidity
91:  _maxMints > 0 && _numberOfTokensPerMint > 0,

91:  _maxMints > 0 && _numberOfTokensPerMint > 0,

96:  _maxBulkBuy > 0 && _maxBulkBuy <= _maxMints,
```

## Recommendation

Consider changing the `>` to `!=` in the places, outlined above.

## Team Response

N/A

# [G-07] Split require statements that have boolean expressions and keep their revert strings smaller than 32 bytes

## Severity

Gas optimization

## Description

Some `require` statements contain a boolean `&&` expression and revert strings that are longer than 32 bytes.

## Location of Affected Code

File: `contracts/DarkMythos.sol#L90-L103`

```solidity
require(
  _maxMints > 0 && _numberOfTokensPerMint > 0,
  "@dev: maxTokenSupply must not equal zero"
);

require(
    _maxBulkBuy > 0 && _maxBulkBuy <= _maxMints,
    "@dev: maxBulkBuy must be greater than zero and less than or equal to maxMints"
);

require(
    _vendor != address(0),
    "@dev: vendor must not be the zero address"
);
```

## Recommendation

Whenever `require` statements contain a boolean `&&`, they can be split into two separate statements. If the first statement evaluates to false, the function will revert immediately and the following `require` statement will not be examined. This will save the gas cost of evaluating the second condition in cases where the first one will fail.

Moreover, we can save additional gas by keeping the revert strings smaller than 32 bytes, i.e., smaller than 32 characters (assuming all of those characters are ASCII).

```diff
require(
  _maxMints > 0,
  "_maxMints cannot be zero"
);

require(
  _numberOfTokensPerMint > 0,
  "maxTokenSupply cannot be zero"
);

require(
    _maxBulkBuy > 0,
    "_maxBulkBuy cannot be zero"
);

require(
    _maxBulkBuy <= _maxMints,
    "_maxBulkBuy must be lt maxMints"
);

require(
    _vendor != address(0),
    "_vendor cannot be zero address"
);
```

Implementing this change will save x gas on method calls and x gas during deployment.

```diff
diff --git a/gas-report.md b/gas-report.md
index 654b086..10e2224 100644
--- a/gas-report.md
+++ b/gas-report.md
@@ -7,11 +7,11 @@
 ···············|················|··············|·············|·············|···············|··············
 |  DarkMythos  ·  approve       ·           -  ·          -  ·      49333  ·            2  ·          -  │
 ···············|················|··············|·············|·············|···············|··············
-|  DarkMythos  ·  mint          ·      744714  ·     761014  ·     755074  ·           20  ·          -  │
+|  DarkMythos  ·  mint          ·      744714  ·     765814  ·     756994  ·           20  ·          -  │
 ···············|················|··············|·············|·············|···············|··············
 |  DarkMythos  ·  pause         ·           -  ·          -  ·      28169  ·            2  ·          -  │
 ···············|················|··············|·············|·············|···············|··············
-|  DarkMythos  ·  transferFrom  ·           -  ·          -  ·      99020  ·            1  ·          -  │
+|  DarkMythos  ·  transferFrom  ·           -  ·          -  ·      96220  ·            1  ·          -  │
 ···············|················|··············|·············|·············|···············|··············
 |  DarkMythos  ·  unpause       ·           -  ·          -  ·      28211  ·            1  ·          -  │
 ···············|················|··············|·············|·············|···············|··············
@@ -19,5 +19,5 @@
 ···············|················|··············|·············|·············|···············|··············
 |  Deployments                  ·                                          ·  % of limit   ·             │
 ································|··············|·············|·············|···············|··············
-|  DarkMythos                   ·           -  ·          -  ·    4201972  ·         14 %  ·          -  │
+|  DarkMythos                   ·           -  ·          -  ·    4204660  ·         14 %  ·          -  │
 ·-------------------------------|--------------|-------------|-------------|---------------|-------------·
```

## Team Response

N/A

# [G-08] Immutables Can be `private`

## Severity

Gas optimization

## Description

Marking immutables as `private` saves gas upon deployment, as the compiler does not have to create getter functions for these values. It is worth noting that a `private` immutable can still be read using either the verified contract source code, the bytecode, or by looking at the contract deployment transaction (since immutables are set in the constructor).

## Location of Affected Code

File: [`contracts/DarkMythos.sol#L26-L31`](https://gitlab.vettersolutions.de/dark-mythos/web3-academy-smart-contract/-/blob/master/contracts/DarkMythos.sol#L26-31)

```solidity
uint256 public immutable mintingCost;
uint256 public immutable maxTokenSupply;
uint256 public immutable numberOfTokensPerMint;
uint256 public immutable maxBulkBuy;
uint256 public immutable allowMintingAfter;
address public immutable vendor;
```

## Recommendation

If you want to prioritize saving gas on deployment, change the visibility modifier of the contract's immutables to `private`. Notice, however, that this change will make some unit tests fail because they can no longer access the implicit getter functions provided by the `public` modifier. Also, while `private` immutables are still publicly accessible, performing this will be more tricky than simply calling a getter function. So, if the added convenience of keeping these getters is worth the additional gas cost, you may want to keep the `public` modifiers.

However, if you decide to change the visibility to `private`, implementing this change will increase method call costs by 78 gas but save 118706 gas during deployment.

```diff
- uint256 public immutable mintingCost;
- uint256 public immutable maxTokenSupply;
- uint256 public immutable numberOfTokensPerMint;
- uint256 public immutable maxBulkBuy;
- uint256 public immutable allowMintingAfter;
- address public immutable vendor;

+ uint256 private immutable mintingCost;
+ uint256 private immutable maxTokenSupply;
+ uint256 private immutable numberOfTokensPerMint;
+ uint256 private immutable maxBulkBuy;
+ uint256 private immutable allowMintingAfter;
+ address private immutable vendor;
```

## Team Response

N/A

# [G-09] Constructor Can be `payable`

## Severity

Gas optimization

## Description

However, making a constructor payable can save over 200 gas on deployment. This is because non-payable functions have an implicit `require(msg.value == 0)` inserted in them. Additionally, fewer bytecode at deployment time means less gas cost due to smaller `calldata`.

There are good reasons to make regular functions non-payable, but generally, a contract is deployed by a privileged address that you can reasonably assume won’t send funds.

Implementing this change will save 4720 gas on method calls and 214 gas during deployment.

## Location of Affected Code

File: [`contracts/DarkMythos.sol#L77`](https://gitlab.vettersolutions.de/dark-mythos/web3-academy-smart-contract/-/blob/master/contracts/DarkMythos.sol#L77)

## Recommendation

Assuming that experienced users are deploying the contract, consider making the constructor `payable`.

## Team Response

N/A
