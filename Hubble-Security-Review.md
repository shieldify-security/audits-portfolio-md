# Findings Summary

| ID     | Title                                                                 | Severity      |
| ------ | --------------------------------------------------------------------- | ------------- |
| [M-01] | `call()` should be used instead of `transfer()` on an address payable | Medium        |
| [M-02] | Centralisation Vulnerability                                          | Medium        |
| [L-01] | Strict sent token value check                                         | Low           |
| [L-02] | Missing input validation                                              | Low           |
| [L-03] | Deprecated `safeApprove()` function                                   | Low           |
| [I-01] | Open TODOs                                                            | Informational |
| [I-02] | Variables can be turned into an immutable                             | Informational |
| [I-03] | Commented out code                                                    | Informational |
| [I-04] | Extract code to memory variable                                       | Informational |
| [I-05] | No event is emitted                                                   | Informational |
| [I-06] | Unused import `IOracle`                                               | Informational |
| [I-07] | Use newer pragma statement                                            | Informational |
| [I-08] | Many files are missing NatSpec                                        | Informational |
| [I-09] | Typos                                                                 | Informational |

# Detailed Findings

# [M-01] `call()` should be used instead of `transfer()` on an address payable

## Severity

Medium Risk

## Context

https://github.com/hubble-exchange/hubble-protocol/blob/shieldify-audit/contracts/HGT.sol#L43

## Description

The `transfer()` and `send()` functions forward a fixed amount of 2300 gas. Historically, it has often been recommended to use these functions for value transfers to guard against reentrancy attacks. However, the gas cost of EVM instructions may change significantly during hard forks which may break already deployed contract systems that make fixed assumptions about gas costs. For example. EIP 1884 broke several existing smart contracts due to a cost increase of the SLOAD instruction.

```solidity
contracts/HGT.sol

payable(_toAddress).transfer(_amount);
```

## Scenario

The use of the deprecated `transfer()` function for an address will inevitably make the transaction fail when:

- The claimer smart contract does not implement a payable function.
- The claimer smart contract does implement a payable fallback which uses more than 2300 gas unit.
- The claimer smart contract implements a payable fallback function that needs less than 2300 gas units but is called through proxy, raising the call's gas usage above 2300.
- Additionally, using higher than 2300 gas might be mandatory for some multisig wallets.

## Recommendation

Use `call()` instead of `transfer()`, but be sure to respect the CEI pattern and/or add re-entrancy guards, as several hacks already happened in the past due to this recommendation not being fully understood.

```diff
-- payable(_toAddress).transfer(_amount);

++ (bool success,) = payable(_toAddress).call{value: _amount}("");
++ require(success, "Transfer failed.");
```

# [M-02] Centralisation Vulnerability

## Severity

Medium Risk

## Context

https://github.com/hubble-exchange/hubble-protocol/blob/shieldify-audit/contracts/VUSD.sol#L45-L47

https://github.com/hubble-exchange/hubble-protocol/blob/shieldify-audit/contracts/VUSD.sol#L56-L76

https://github.com/hubble-exchange/hubble-protocol/blob/shieldify-audit/contracts/HGT.sol#L49-L60

## Description

It's a centralisation vulnerability if an owner can pause not only inbound but outbound functionality. The pausing in `OrderBook` is properly implemented and the outbound function `cancelOrder` can't be paused. However in `VUSD` and `HGT` there is a pausing functionality for such methods.

```solidity
contracts/VUSD.sol

function withdraw(uint amount) external override whenNotPaused {

function processWithdrawals() external override whenNotPaused nonReentrant {
```

```solidity
contracts/HGT.sol

function withdraw(
  uint16 _dstChainId,
  bytes memory _toAddress,
  uint _amount,
  address payable _refundAddress,
  address _zroPaymentAddress,
  bytes memory _adapterParams
) external payable whenNotPaused {
```

## Recommendation

Remove `whenNotPaused` modifier from all withdraw functions in `VUSD` and `HGT`.

# [L-01] Strict sent token value check

## Severity

Low Risk

## Context

https://github.com/hubble-exchange/hubble-protocol/blob/shieldify-audit/contracts/VUSD.sol#L41

## Description

Ensure that the sender has sent at least the expected amount .

## Recommendation

```diff
contracts/VUSD.sol

-- require(msg.value == amount * SCALING_FACTOR, "vUSD: Insufficient amount transferred");
++ require(msg.value >= amount * SCALING_FACTOR, "vUSD: Insufficient amount transferred");
```

# [L-02] Missing input validation

## Severity

Low Risk

## Context

https://github.com/hubble-exchange/hubble-protocol/blob/shieldify-audit/contracts/AMM.sol#L88

https://github.com/hubble-exchange/hubble-protocol/blob/shieldify-audit/contracts/AMM.sol#L92

```solidity
contracts/AMM.sol

constructor(address _clearingHouse) {

function initialize(
  string memory _name,
  address _underlyingAsset,
  address _oracle,
  uint _minSizeRequirement,
  address _governance
) external initializer {
```

https://github.com/hubble-exchange/hubble-protocol/blob/shieldify-audit/contracts/ClearingHouse.sol#L57

```solidity
contracts/ClearingHouse.sol

function initialize(
  address _governance,
  address _feeSink,
  address _marginAccount,
  address _defaultOrderBook,
  address _vusd,
  address _hubbleReferral
) external
  initializer
{
```

https://github.com/hubble-exchange/hubble-protocol/blob/shieldify-audit/contracts/InsuranceFund.sol#L62

```solidity
contracts/InsuranceFund.sol

function initialize(address _governance) external initializer {
```

https://github.com/hubble-exchange/hubble-protocol/blob/shieldify-audit/contracts/MarginAccountHelper.sol#L22

```solidity
contracts/MarginAccountHelper.sol

function initialize(
  address _governance,
  address _vusd,
  address _marginAccount,
  address _insuranceFund
) external initializer {
```

https://github.com/hubble-exchange/hubble-protocol/blob/shieldify-audit/contracts/Oracle.sol#L20

```solidity
contracts/Oracle.sol

function initialize(address _governance) external initializer {
```

https://github.com/hubble-exchange/hubble-protocol/blob/shieldify-audit/contracts/orderbooks/OrderBookSigned.sol#L60

https://github.com/hubble-exchange/hubble-protocol/blob/shieldify-audit/contracts/orderbooks/OrderBookSigned.sol#L64

```solidity
contracts/OrderBookSigned.sol

constructor(address _clearingHouse) {

function initialize(
  string memory _name,
  string memory _version,
  address _governance
) external initializer {
```

https://github.com/hubble-exchange/hubble-protocol/blob/shieldify-audit/contracts/legos/HubbleBase.sol#L33

```solidity
contracts/MetaHubbleBase.sol

constructor(address _trustedForwarder) ERC2771Context(_trustedForwarder) {}
```

https://github.com/hubble-exchange/hubble-protocol/blob/shieldify-audit/contracts/orderbooks/OrderBook.sol#L52

https://github.com/hubble-exchange/hubble-protocol/blob/shieldify-audit/contracts/orderbooks/OrderBook.sol#L57

```solidity
contracts/OrderBook.sol

constructor(address _clearingHouse, address _marginAccount) {

function initialize(
  string memory _name,
  string memory _version,
  address _governance
) external initializer {
```

https://github.com/hubble-exchange/hubble-protocol/blob/shieldify-audit/contracts/VUSD.sol#L28

```solidity
contracts/VUSD.sol

function initialize(string memory name, string memory symbol) public override virtual {
```

## Description

There are no input parameters validation in `initialize` and `constructor` methods.

## Recommendation

Perform the corresponding input parameters validation checks.

# [L-03] Deprecated `safeApprove()` function

## Severity

Low Risk

## Context

https://github.com/hubble-exchange/hubble-protocol/blob/shieldify-audit/contracts/MarginAccountHelper.sol#L32-L33

## Description

This behavior is not natively present when calling `approve()` and can therefore easily create unintended reverts that lock funds in smart contracts that use this code. The front-running vector is also present. This function was deprecated by OpenZeppelin since it doesn't resolve the fron-running.

```solidity
contracts/MarginAccountHelper.sol

IERC20(_vusd).safeApprove(_marginAccount, type(uint).max);
IERC20(_vusd).safeApprove(_insuranceFund, type(uint).max);
```

## Recommendation

The function `safeApprove` function has been deprecated by OpenZeppelin team. Avoid using it and use `safeIncreaseAllowance` and `safeDecreaseAllowance` instead since USDT is not one of the supported tokens.

# [I-01] Open TODOs

## Severity

Informational

## Context

https://github.com/hubble-exchange/hubble-protocol/blob/shieldify-audit/contracts/GenesisTUP.sol#L13

```solidity
contracts/GenesisTUP.sol

// @todo initializer check
```

https://github.com/hubble-exchange/hubble-protocol/blob/shieldify-audit/contracts/AMM.sol#L233

https://github.com/hubble-exchange/hubble-protocol/blob/shieldify-audit/contracts/AMM.sol#L495

```solidity
contracts/AMM.sol

// @todo calculate oracle twap for exact funding period

// @todo calculate oracle twap for exact funding period
```

https://github.com/hubble-exchange/hubble-protocol/blob/shieldify-audit/contracts/MarginAccount.sol#L287

```solidity
contracts/MarginAccount.sol

@todo consider providing some incentive from insurance fund to execute a liquidation in this scenario.
```

https://github.com/hubble-exchange/hubble-protocol/blob/shieldify-audit/contracts/orderbooks/OrderBook.sol#L131

```solidity
contracts/OrderBook.sol

// @todo move this to precompile
```

## Description

Open TO-DOs can point to architecture or programming issues that still
need to be resolved. Often these kinds of comments indicate areas of
complexity or confusion for developers. This provides value and insight
to an attacker who aims to cause damage to the protocol.

## Recommendation

Consider resolving the TO-DOs before deploying code to a production
context. Use an independent issue tracker or other project management
software to track development tasks.

# [I-02] Variables can be turned into an immutable

## Severity

Informational

## Context

https://github.com/hubble-exchange/hubble-protocol/blob/shieldify-audit/contracts/AMM.sol#L92

## Description & Recommendation

The `name` and `underlyingAsset` can be made `immutable` since their values are only set in the `initialize` method. It's a best practice because it makes the code easier to reason about and less prone to errors, since it guarantees that their values will not change unexpectedly.

# [I-03] Commented out code

## Severity

Informational

## Context

https://github.com/hubble-exchange/hubble-protocol/blob/shieldify-audit/contracts/MinimalForwarder.sol#L6

```solidity
contracts/MinimalForwarder.sol

// import { MinimalForwarderUpgradeable } from "@openzeppelin/contracts-upgradeable/metatx/MinimalForwarderUpgradeable.sol";
```

https://github.com/hubble-exchange/hubble-protocol/blob/shieldify-audit/contracts/orderbooks/OrderBookSigned.sol#L31-L36

```solidity
contracts/OrderBookSigned.sol

// enum OrderExecutionMode {
//     Taker,
//     Maker,
//     SameBlock,
//     Liquidation
// }
```

https://github.com/hubble-exchange/hubble-protocol/blob/shieldify-audit/contracts/orderbooks/OrderBookSigned.sol#L130-L146

```solidity
contracts/OrderBookSigned.sol

// try clearingHouse.openComplementaryPositions(orders, matchInfo, fillAmount, fulfillPrice) {
//     // get openInterestNotional for indexing
//     IAMM amm = clearingHouse.amms(orders[0].ammIndex);
//     uint openInterestNotional = amm.openInterestNotional();
//     // emit OrdersMatched(matchInfo[0].orderHash, matchInfo[1].orderHash, fillAmount.toUint256() /* asserts fillAmount is +ve */, fulfillPrice, openInterestNotional, msg.sender, block.timestamp);
// } catch Error(string memory err) { // catches errors emitted from "revert/require"
//     try this.parseMatchingError(err) returns(bytes32 orderHash, string memory reason) {
//         // emit OrderMatchingError(orderHash, reason);
//     } catch (bytes memory) {
//         // abi.decode failed; we bubble up the original err
//         revert(err);
//     }
//     return;
// } /* catch (bytes memory err) {
//     // we do not any special handling for other generic type errors
//     // they can revert the entire tx as usual
// } */
```

https://github.com/hubble-exchange/hubble-protocol/blob/shieldify-audit/contracts/orderbooks/OrderBook.sol#L143-L146

```solidity
contracts/OrderBook.sol

/* catch (bytes memory err) {
  // we do not any special handling for other generic type errors
  // they can revert the entire tx as usual
} */
```

https://github.com/hubble-exchange/hubble-protocol/blob/shieldify-audit/contracts/orderbooks/OrderBook.sol#L183

```solidity
contracts/OrderBook.sol

// require(order.validUntil == 0, "OB_expiring_orders_not_supported");
```

https://github.com/hubble-exchange/hubble-protocol/blob/shieldify-audit/contracts/orderbooks/OrderBook.sol#L300-L303

```solidity
contracts/OrderBook.sol

/* catch (bytes memory err) {
  // we do not any special handling for other generic type errors
  // they can revert the entire tx as usual
} */
```

https://github.com/hubble-exchange/hubble-protocol/blob/shieldify-audit/contracts/Oracle.sol#L169

```solidity
contracts/Oracle.sol

// AggregatorV3Interface(chainLinkAggregatorMap[underlying]).latestRoundData(); // sanity check
```

## Description

There are instances within the code where commented code is left from
previous development iterations. While this does not cause any security
concerns, it makes the contract less readable.

## Recommendation

Shieldify recommends that the code is removed to improve the readability of the code.

# [I-04] Extract code to memory variable

## Severity

Informational

## Context

https://github.com/hubble-exchange/hubble-protocol/blob/shieldify-audit/contracts/AMM.sol#L129

## Description

It's suggested to extract code to memory variable to improve readability in `openPosition` function in `AMM`.

## Recommendation

```diff
contracts/AMM.sol

-- if (isNewPosition || (position.size > 0 ? Side.LONG : Side.SHORT) == side) {

++ Side positionSide = position.size > 0 ? Side.LONG : Side.SHORT;
++ if (isNewPosition || positionSide == side) {
```

# [I-05] No event is emitted

## Severity

Informational

## Context

https://github.com/hubble-exchange/hubble-protocol/blob/shieldify-audit/contracts/MarginAccount.sol#L662

```solidity
contracts/MarginAccount.sol

function whitelistCollateral(address _coin, uint _weight) external onlyGovernance {
```

https://github.com/hubble-exchange/hubble-protocol/blob/shieldify-audit/contracts/MarginAccount.sol#L667

```solidity
contracts/MarginAccount.sol

function changeCollateralWeight(uint idx, uint _weight) external onlyGovernance {
```

https://github.com/hubble-exchange/hubble-protocol/blob/shieldify-audit/contracts/MarginAccount.sol#L673

```solidity
contracts/MarginAccount.sol

function updateParams(uint _minAllowableMargin) external onlyClearingHouse {
```

https://github.com/hubble-exchange/hubble-protocol/blob/shieldify-audit/contracts/AMM.sol#L180-L192

```solidity
contracts/AMM.sol

function updatePosition(address trader)
  override
  external
  onlyClearingHouse
  returns(int256 fundingPayment, int256 latestCumulativePremiumFraction)
{
```

https://github.com/hubble-exchange/hubble-protocol/blob/shieldify-audit/contracts/AMM.sol#L539

```solidity
contracts/AMM.sol

function changeOracle(address _oracle) public onlyGovernance {
```

https://github.com/hubble-exchange/hubble-protocol/blob/shieldify-audit/contracts/AMM.sol#L543

```solidity
contracts/AMM.sol

function setPriceSpreadParams(uint _maxOracleSpreadRatio, uint /* dummy for backwards compatibility */) external onlyGovernance {
```

https://github.com/hubble-exchange/hubble-protocol/blob/shieldify-audit/contracts/AMM.sol#L547

```solidity
contracts/AMM.sol

function setLiquidationParams(uint _maxLiquidationRatio, uint _maxLiquidationPriceSpread) external onlyGovernance {
```

https://github.com/hubble-exchange/hubble-protocol/blob/shieldify-audit/contracts/AMM.sol#L552

```solidity
contracts/AMM.sol

function setMinSizeRequirement(uint _minSizeRequirement) external onlyGovernance {
```

https://github.com/hubble-exchange/hubble-protocol/blob/shieldify-audit/contracts/AMM.sol#L556

```solidity
contracts/AMM.sol

function setFundingParams(
  uint _fundingPeriod,
  uint _fundingBufferPeriod,
  int256 _maxFundingRate,
  uint _spotPriceTwapInterval
) external onlyGovernance {
```

https://github.com/hubble-exchange/hubble-protocol/blob/shieldify-audit/contracts/ClearingHouse.sol#L113-L131

```solidity
contracts/ClearingHouse.sol

function updatePositions(address trader) override public whenNotPaused {
```

https://github.com/hubble-exchange/hubble-protocol/blob/shieldify-audit/contracts/Oracle.sol#L163

```solidity
contracts/Oracle.sol

function setAggregator(address underlying, address aggregator) external onlyGovernance {
```

https://github.com/hubble-exchange/hubble-protocol/blob/shieldify-audit/contracts/Oracle.sol#L172

```solidity
contracts/Oracle.sol

function setStablePrice(address underlying, int256 price) external onlyGovernance {
```

https://github.com/hubble-exchange/hubble-protocol/blob/shieldify-audit/contracts/orderbooks/OrderBookSigned.sol#L215

```solidity
contracts/OrderBookSigned.sol

function setValidatorStatus(address validator, bool status) external onlyGovernance {
```

https://github.com/hubble-exchange/hubble-protocol/blob/shieldify-audit/contracts/orderbooks/OrderBookSigned.sol#L219

```solidity
contracts/OrderBookSigned.sol

function initializeMinSize(int minSize) external onlyGovernance {
```

https://github.com/hubble-exchange/hubble-protocol/blob/shieldify-audit/contracts/orderbooks/OrderBookSigned.sol#L223

```solidity
contracts/OrderBookSigned.sol

function updateMinSize(uint ammIndex, int minSize) external onlyGovernance {
```

https://github.com/hubble-exchange/hubble-protocol/blob/shieldify-audit/contracts/orderbooks/OrderBook.sol#L422

```solidity
contracts/OrderBook.sol

function setValidatorStatus(address validator, bool status) external onlyGovernance {
```

https://github.com/hubble-exchange/hubble-protocol/blob/shieldify-audit/contracts/orderbooks/OrderBook.sol#L426

```solidity
contracts/OrderBook.sol

function initializeMinSize(int minSize) external onlyGovernance {
```

https://github.com/hubble-exchange/hubble-protocol/blob/shieldify-audit/contracts/orderbooks/OrderBook.sol#L430

```solidity
contracts/OrderBook.sol

function updateMinSize(uint ammIndex, int minSize) external onlyGovernance {
```

https://github.com/hubble-exchange/hubble-protocol/blob/shieldify-audit/contracts/orderbooks/OrderBook.sol#L434

```solidity
contracts/OrderBook.sol

function setBibliophile(address _bibliophile) external onlyGovernance {
```

https://github.com/hubble-exchange/hubble-protocol/blob/shieldify-audit/contracts/orderbooks/OrderBook.sol#L438

```solidity
contracts/OrderBook.sol

function updateParams(uint _minAllowableMargin, uint _takerFee) external {
```

## Description

Doesn't emit a event even though is makes important state change.

## Recommendation

Emit an event

# [I-06] Unused import `IOracle`

## Severity

Informational

## Context

https://github.com/hubble-exchange/hubble-protocol/blob/shieldify-audit/contracts/AMM.sol#L9

## Description

There is an unused import in the codebase. The import should be
cleared from the code if no target. Clearing this import
will increase the readability of contracts.

## Recommendation

Consider removing import from the code

# [I-07] Use newer pragma statement

## Severity

Informational

## Description & Recommendation

All of the contracts use version `0.8.9` while the latest version is `0.8.20`. Consider upgrading the version to a newer one to use bugfixes and optimizations in the compiler.

# [I-08] Many files are missing NatSpec

## Severity

Informational

## Description & Recommendation

Following the Solidity Style Guide sections, try to write as detailed comments as possible to make it easier for auditors, developers or other technical guys to understand the purpose of each particular feature and its parameters.

# [I-09] Typos

## Severity

Informational

## Context

https://github.com/hubble-exchange/hubble-protocol/blob/shieldify-audit/contracts/AMM.sol#L51

## Recommendation

`allowd` -> `allowed`
