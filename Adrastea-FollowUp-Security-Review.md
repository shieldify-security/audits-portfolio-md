# 1. About Shieldify

Positioned as the first hybrid Web3 Security company, Shieldify shakes things up with a unique subscription-based auditing model that entitles the customer to unlimited audits within its duration, as well as top-notch service quality thanks to a disruptive 6-layered security approach. The company works with very well-established researchers in the space and has secured multiple millions in TVL across protocols, also can audit codebases written in Solidity, Vyper, Rust, Cairo, Move and Go.

Learn more about us at [`shieldify.org`](https://shieldify.org/).

# 2. Disclaimer

This security review does not guarantee bulletproof protection against a hack or exploit. Smart contracts are a novel technological feat with many known and unknown risks. The protocol, which this report is intended for, indemnifies Shieldify Security against any responsibility for any misbehavior, bugs, or exploits affecting the audited code during any part of the project's life cycle. It is also pivotal to acknowledge that modifications made to the audited code, including fixes for the issues described in this report, may introduce new problems and necessitate additional auditing.

# 3. About Adra LRT

Adrastea's main purpose is ultra-accessible real yields on Solana. As a restaking protocol, an input token and an output token (LRT) are needed for the restaking pool. The input token can be any form of asset the pool takes and the output token should be the liquid restaking token issued by the restaking pool. Users transfer the input tokens to the restaking pool and should get the output token (LRT) back. Under the hood, the restaking pool delegates authority delegates/undelegates the input token to AVSs and manages the AVS token. Users should be able to withdraw their funds anytime, even if there is not enough input token liquidity in the restaking pool.

## Overview

Adrastea is a Liquid Restaking Protocol where users leverage the full potential of their liquidity by depositing liquid staked tokens to maximize the yield accruing power of their initial assets, whilst also providing security to the Solana ecosystem via delegations to active validation services.
The protocol currently runs on Solayer, where users can deposit their `sSol` to gain an equivalent value in liquid restaking tokens, which represent a share in the entire pool, allowing users to profit on accrued rewards from the endoAVS.

**The Protocol consists of 3 major functionalities for the users.**

1. The Deposit functionality.
2. The Withdraw functionality.
3. The Withdraw Delegated Stake functionality.

---

- **The Deposit** allows users to deposit their liquid staked tokens, where they are minted the liquid restaked tokens.
- **The Withdraw** allows users to withdraw their initial liquid-staked tokens by providing their liquid restaked tokens to be burned. This is used when the pools have liquidity.
- **The Withdraw Delegated Stake** allows users to withdraw their initially staked tokens at any point, further providing decentralization and constant access to liquidity for the restaked tokens. This can be used to withdraw even when there is no liquidity in the pool

---

**The Protocol consists of 3 major functionalities for the Liquid Restaking Pool Delegator.**

1. `Delegate`
2. `Undelegate`
3. `Set Deposit Limit`

- **Delegate** allows the delegator to delegate funds from the `endo avs program`, allowing the pools to accrue yield for the users.
- **Undelegate** allows the delegator to undelegate the funds from the `endo avs program`, providing the users with their initially deposited funds with the accrued profit, so they can always withdraw.
- **Set Deposit Limit** allows the delegator to set a max cap for the amount of staked tokens that can be deposited into a pool.

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

The security review lasted 5 days, with a total of 32 hours dedicated by the Shieldify team.

Overall, the code is well-written. The audit report contributed by identifying two High and one Low severity findings, related to unauthorised and incompatibility issues for the delegate and undelegate functionality.

The protocol did well to ensure validation of seeds and ownership of every account in all the context structs, ensuring security against bypassing certain invariants. The protocol limited centralization by reducing the power of the delegator, ensuring they have no way to steal or siphon the user's funds.
Certain functions are well tested and properly implemented, giving the protocol an edge as its development was properly structured, planned, and executed.

## 5.1 Protocol Summary

| **Project Name**             | Adra LRT - Follow-up Review                                                                                                           |
| ---------------------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| **Repository**               | [adra-lrt](https://github.com/adrasteafinance/adra-lrt)                                                                               |
| **Type of Project**          | DeFi, Liquid restaking protocol                                                                                                       |
| **Security Review Timeline** | 5 days                                                                                                                                |
| **Review Commit Hash**       | [e3b8848c25c7eadfbc8830f515907f1736cdb576](https://github.com/adrasteafinance/adra-lrt/tree/e3b8848c25c7eadfbc8830f515907f1736cdb576) |
| **Fixes Review Commit Hash** | N/A                                                                                                                                   |

## 5.2 Scope

The following files were in the scope of the security review:

| File                                                          | LOC |
| ------------------------------------------------------------- | :-: |
| programs/adra-lrt/src/utils.rs                                |  6  |
| programs/adra-lrt/src/lib.rs                                  | 70  |
| programs/adra-lrt/src/errors.rs                               | 28  |
| programs/adra-lrt/src/state/lrt_pool.rs                       | 12  |
| programs/adra-lrt/src/state/mod.rs                            |  2  |
| programs/adra-lrt/src/contexts/delegate.rs                    | 188 |
| programs/adra-lrt/src/contexts/deposit.rs                     | 110 |
| programs/adra-lrt/src/contexts/initialize.rs                  | 57  |
| programs/adra-lrt/src/contexts/mod.rs                         | 20  |
| programs/adra-lrt/src/contexts/transfer_delegate_authority.rs | 25  |
| programs/adra-lrt/src/contexts/update_mint_limit.rs           | 23  |
| programs/adra-lrt/src/contexts/withdraw.rs                    | 109 |
| programs/adra-lrt/src/contexts/withdraw_stake.rs              | 201 |
| Total                                                         | 851 |

# 6. Findings Summary

The following number of issues have been identified, sorted by their severity:

- **High** issues: 2
- **Low** issues: 1

| **ID** | **Title**                                                                                                                                                            | **Severity** | **Status** |
| :----: | -------------------------------------------------------------------------------------------------------------------------------------------------------------------- | :----------: | :--------: |
| [H-01] | The `withdraw_delegated_stake()` Blindly Undelegates the Stake in the AVS Program, Allowing Malicious Users to Brick Accrual of Yield from the AVS                   |     High     |    N/A     |
| [H-02] | The `withdraw_delegated_stake()` Will Always Incorrectly Evaluate the Amount of Tokens Undelegated from the AVS, and Send Incorrect Input Token Amounts to the Users |     High     |    N/A     |
| [L-01] | The `delegate()`, `undelegate()`, and `withdraw_delegated_stake()` Will Break in the Future, Due to Lack of Compatibility with Other Endo AVS Programs               |     Low      |    N/A     |

# 7. Findings

# [H-01] The `withdraw_delegated_stake()` Blindly Undelegates the Stake in the AVS Program, Allowing Malicious Users to Brick Accrual of Yield from the AVS

## Severity

High Risk

## Description

The `withdraw_delegated_stake()` is used to withdraw liquidity from the pool in cases where there is no active liquidity in the pool. The issue is that `withdraw_delegated_stake()` undelegates the balance to be withdrawn without checking if it has enough liquidity to cover that unstake. This can be exploited to block the accrual of yield from the AVS.

**Exploit Example**

1. User A deposits **100 sSol to gain 100 adrSol**
2. The delegator delegates the deposited sSol (100 total) to the AVS.
3. User B, being malicious, comes and **deposits 100 sSol to gain 100 adrSol too**.
4. But before the delegator delegates 100 sSol to the AVS to enable **yield accrual**.
5. User B comes and calls `withdraw_delegated_stake()` to collect all their balance back.
6. But since the instruction **doesn't check if the vault has enough liquidity to cover that withdrawal**.
7. The instruction **forcefully un-delegates the 100 sSol User A deposited, from the AVS**, and sends the balance to User B.
8. Now the remaining `sSol` stays in the **program's vault without earning yield.**
9. If the delegator ever comes to delegate that balance back to the AVS, User B can always continuously come again and deposit to it, then use `withdraw_delegated_stake()` to un-delegate that stake from the AVS again, which will only cost him gas fees, thereby bricking the accrual of yield by the program's delegation to the AVS.

## Location of Affected Code

File: [src/lib.rs#L40](https://github.com/adrasteafinance/adra-lrt/blob/e3b8848c25c7eadfbc8830f515907f1736cdb576/programs/adra-lrt/src/lib.rs#L40)

```rust
// user can always withdraw stake to get sSol back, even if there is no sSol liquidity in the pool
pub fn withdraw_delegated_stake(ctx: Context<WithdrawStake>, amount: u64) -> Result<()> {
    // burn output token from user first
    ctx.accounts.burn_output_token(amount)?;
    // calculate withdraw amount
    let withdraw_amount = ctx.accounts.calculate_input_token_amount(amount);
    //@audit this function doesn't check if there is liquidity to cover the withdrawal.
    //it blindly always un-delegates from the AVS, even though there is enough in the pool to cover withdrawal
    // undelegate avs token
    ctx.accounts.undelegate(withdraw_amount)?;
    // transfer input token back to the user's vault
    ctx.accounts.unstake(withdraw_amount)?;
    Ok(())
}
```

## Impact

This is high because it allows malicious users to block yield accrual by always forcing the program to un-delegate from the AVS.

## Recommendation

Before calling ` ctx.accounts.undelegate(withdraw_amount)`, ensure to check if **the pool has enough liquidity to cover the withdrawal**; if it has enough, **then simply send to the user rather than un-delegating from the AVS** when there is no need.

## Team Response

N/A

# [H-02] The `withdraw_delegated_stake()` Will Always Incorrectly Evaluate the Amount of Tokens Undelegated from the AVS, and Send Incorrect Input Token Amounts to the Users

## Severity

High Risk

## Description

In `withdraw_delegated_stake()`, the `calculate_input_token_amount()` is used twice and also inappropriately, leading to a **flaw where the amount undelegated will be incorrect**, subsequently forcing an incorrect amount of the pool's input token to be sent to the staker who wishes to withdraw.

## Location of Affected Code

File: [src/lib.rs#L40](https://github.com/adrasteafinance/adra-lrt/blob/e3b8848c25c7eadfbc8830f515907f1736cdb576/programs/adra-lrt/src/lib.rs#L40)

```rust
// user can always withdraw stake to get sSol back even if there is no sSol liquidity in the pool
pub fn withdraw_delegated_stake(ctx: Context<WithdrawStake>, amount: u64) -> Result<()> {
    // burn output token from user first
    ctx.accounts.burn_output_token(amount)?;
    // calculate withdraw amount
    let withdraw_amount = ctx.accounts.calculate_input_token_amount(amount);
    // undelegate avs token
    //@audit the calculated input token amount is used as the amount to be undelegated.
    ctx.accounts.undelegate(withdraw_amount)?;
    // transfer input token back to the user's vault
    ctx.accounts.unstake(withdraw_amount)?;
    Ok(())
}
```

The issue then lies in the **subsequent recalculating of the input token amount from that same previously calculated amount** in the `unstake()` logic during the transfer of the pool's input tokens back to the user.

File: [src/contexts/withdraw.rs#L78](https://github.com/adrasteafinance/adra-lrt/blob/e3b8848c25c7eadfbc8830f515907f1736cdb576/programs/adra-lrt/src/contexts/withdraw.rs#L78)

```rust
pub fn unstake(&mut self, amount: u64) -> Result<()> {
    self.pool_input_token_vault.reload()?;
    if self.pool_input_token_vault.amount < amount {
        return Err(LRTPoolError::InsufficientSSOLFundsForWithdraw.into());
   }

   // code

    transfer_checked(
        ctx,
        self.calculate_input_token_amount(amount),
        self.input_token_mint.decimals,
    )
}
```

This will lead to an incorrect amount of tokens being undelegated from the AVS and also sent to the user.

## Impact

Based on the `calculate_input_token_amount()` business logic, the issue would be catastrophic, whilst here it is without any logic, on addition of the main logic, this will lead to.

1. Incorrect amounts undelegated.
2. Incorrect amounts sent to the stakeholder.
3. Incorrect state checks, which could possibly lead to a DOS on the calls to withdraw.

```rust
  //@audit these kinds of checks will be asserting conditions using incorrect amounts.
  if self.pool_input_token_vault.amount < amount {
            return Err(LRTPoolError::InsufficientSSOLFundsForWithdraw.into());
        }
```

Whilst currently the `calculate_input_token_amount()` returns the same amount, the report assumes the addition of future calculation logic to ensure appropriate withdrawals.

```rust
  // fill this function according to your business logic
    pub fn calculate_input_token_amount(&self, amount: u64) -> u64 {
        amount
    }
```

## Recommendation

Create a separate function to calculate the amount of AVS to be withdrawn based on the output token's amount, rather than using the `calculate_input_token_amount()` twice during the same flow, and **ensure consistency across all calculations**.

## Team Response

N/A

# [L-01] The `delegate()`, `undelegate()`, and `withdraw_delegated_stake()` Will Break in the Future, Due to Lack of Compatibility with Other Endo AVS Programs

## Severity

Low Risk

## Description

The above functions are written specifically for the Solayer Endo AVS program and lack compatibility with other AVS programs, resulting in a denial of service when attempting to delegate or un-delegate the stake to and from another type of Endo AVS program in the future.

## Impact

This is a lack of robustness and compatibility in the current program's implementation, and will lead to a denial of service on trying to interact with other AVS programs, potentially in the future.

## Recommendation

Make those instructions more robust to handle delegations or un-delegations with other AVS programs.
**Fix Instance**

1. Create a delegation type and an un-delegation type enum.
2. When delegations or un-delegations are to be called, the caller should specify the type or the AVS program they intend to interact with.
3. On the execution of the logic, the instructions should take into account the specified type and AVS program to be invoked, and should ensure it handles it properly.

## Team Response

N/A
