---
title: 共享 PDA
objectives:
- 解释与共享 PDA 相关的安全风险
- 派生具有离散权限域的 PDA
- 使用 Anchor 的 `seeds` 和 `bump` 约束来验证 PDA 账户
---

# 总结

- 将同一个 PDA 用于多个权限域会使您的程序面临用户访问不属于他们的数据和资金的可能性。
- 通过使用用户和/或域特定的种子，防止同一个 PDA 用于多个账户。
- 使用 Anchor 的 `seeds` 和 `bump` 约束来验证 PDA 是否使用了预期的种子和 bump。

# 概述

共享 PDA （PDA sharing）指的是在多个用户或域（domains）之间使用相同的 PDA 作为签名者。特别是在使用 PDAs 进行签名时，使用全局 PDA 代表程序可能看起来是合适的。然而，这会打开一个账户验证通过但用户能够访问不属于他们的资金、转账或数据的可能性。

## 不安全的全局 PDA

在下面的示例中，`vault` 账户的 `authority` 是使用存储在 `pool` 账户上的 `mint` 地址派生的 PDA。该 PDA 作为 `authority` 账户传递到指令中，用于签署从 `vault` 到 `withdraw_destination` 的代币转账。

使用 `mint` 地址作为派生用于签署 `vault` 的 PDA 的种子是不安全的，因为可以为同一个 `vault` 代币账户创建多个 `pool` 账户，但使用不同的 `withdraw_destination`。通过使用 `mint` 作为种子派生用于代币转账的 PDA 进行签署，任何 `pool` 账户都可以签署从 `vault` 代币账户到任意 `withdraw_destination` 的代币转账。

```rust
use anchor_lang::prelude::*;
use anchor_spl::token::{self, Token, TokenAccount};

declare_id!("Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS");

#[program]
pub mod pda_sharing_insecure {
    use super::*;

    pub fn withdraw_tokens(ctx: Context<WithdrawTokens>) -> Result<()> {
        let amount = ctx.accounts.vault.amount;
        let seeds = &[ctx.accounts.pool.mint.as_ref(), &[ctx.accounts.pool.bump]];
        token::transfer(ctx.accounts.transfer_ctx().with_signer(&[seeds]), amount)
    }
}

#[derive(Accounts)]
pub struct WithdrawTokens<'info> {
    #[account(has_one = vault, has_one = withdraw_destination)]
    pool: Account<'info, TokenPool>,
    vault: Account<'info, TokenAccount>,
    withdraw_destination: Account<'info, TokenAccount>,
    authority: AccountInfo<'info>,
    token_program: Program<'info, Token>,
}

impl<'info> WithdrawTokens<'info> {
    pub fn transfer_ctx(&self) -> CpiContext<'_, '_, '_, 'info, token::Transfer<'info>> {
        let program = self.token_program.to_account_info();
        let accounts = token::Transfer {
            from: self.vault.to_account_info(),
            to: self.withdraw_destination.to_account_info(),
            authority: self.authority.to_account_info(),
        };
        CpiContext::new(program, accounts)
    }
}

#[account]
pub struct TokenPool {
    vault: Pubkey,
    mint: Pubkey,
    withdraw_destination: Pubkey,
    bump: u8,
}
```

## 安全的账户特定 PDA

创建账户特定 PDA 的一种方法是使用 `withdraw_destination` 作为种子来派生用作 `vault` 代币账户的 `authority` 的 PDA。这确保了用于 `withdraw_tokens` 指令中 CPI 签名的 PDA 是使用预期的 `withdraw_destination` 代币账户派生的。换句话说，只能将 `vault` 代币账户中的代币提取到最初与 `pool` 账户一起初始化的 `withdraw_destination`。

```rust
use anchor_lang::prelude::*;
use anchor_spl::token::{self, Token, TokenAccount};

declare_id!("Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS");

#[program]
pub mod pda_sharing_secure {
    use super::*;

    pub fn withdraw_tokens(ctx: Context<WithdrawTokens>) -> Result<()> {
        let amount = ctx.accounts.vault.amount;
        let seeds = &[
            ctx.accounts.pool.withdraw_destination.as_ref(),
            &[ctx.accounts.pool.bump],
        ];
        token::transfer(ctx.accounts.transfer_ctx().with_signer(&[seeds]), amount)
    }
}

#[derive(Accounts)]
pub struct WithdrawTokens<'info> {
    #[account(has_one = vault, has_one = withdraw_destination)]
    pool: Account<'info, TokenPool>,
    vault: Account<'info, TokenAccount>,
    withdraw_destination: Account<'info, TokenAccount>,
    authority: AccountInfo<'info>,
    token_program: Program<'info, Token>,
}

impl<'info> WithdrawTokens<'info> {
    pub fn transfer_ctx(&self) -> CpiContext<'_, '_, '_, 'info, token::Transfer<'info>> {
        let program = self.token_program.to_account_info();
        let accounts = token::Transfer {
            from: self.vault.to_account_info(),
            to: self.withdraw_destination.to_account_info(),
            authority: self.authority.to_account_info(),
        };
        CpiContext::new(program, accounts)
    }
}

#[account]
pub struct TokenPool {
    vault: Pubkey,
    mint: Pubkey,
    withdraw_destination: Pubkey,
    bump: u8,
}
```

## Anchor 的 `seeds` 和 `bump` 约束

PDA 可以同时用作账户的地址，并允许程序对其拥有的 PDA 进行签名。

下面的示例使用了使用 `withdraw_destination` 派生的 PDA，既作为 `pool` 账户的地址，又作为 `vault` 代币账户的所有者。这意味着只有与正确的 `vault` 和 `withdraw_destination` 相关联的 `pool` 账户才能在 `withdraw_tokens` 指令中使用。

您可以使用 Anchor 的 `seeds` 和 `bump` 约束与 `#[account(...)]` 属性一起验证 `pool` 账户的 PDA。Anchor 使用指定的 `seeds` 和 `bump` 派生一个 PDA，并与作为 `pool` 账户传递到指令中的账户进行比较。`has_one` 约束用于进一步确保只有正确的账户存储在 `pool` 账户上，并传递到指令中。

```rust
use anchor_lang::prelude::*;
use anchor_spl::token::{self, Token, TokenAccount};

declare_id!("Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS");

#[program]
pub mod pda_sharing_recommended {
    use super::*;

    pub fn withdraw_tokens(ctx: Context<WithdrawTokens>) -> Result<()> {
        let amount = ctx.accounts.vault.amount;
        let seeds = &[
            ctx.accounts.pool.withdraw_destination.as_ref(),
            &[ctx.accounts.pool.bump],
        ];
        token::transfer(ctx.accounts.transfer_ctx().with_signer(&[seeds]), amount)
    }
}

#[derive(Accounts)]
pub struct WithdrawTokens<'info> {
    #[account(
				has_one = vault,
				has_one = withdraw_destination,
				seeds = [withdraw_destination.key().as_ref()],
				bump = pool.bump,
		)]
    pool: Account<'info, TokenPool>,
    vault: Account<'info, TokenAccount>,
    withdraw_destination: Account<'info, TokenAccount>,
    token_program: Program<'info, Token>,
}

impl<'info> WithdrawTokens<'info> {
    pub fn transfer_ctx(&self) -> CpiContext<'_, '_, '_, 'info, token::Transfer<'info>> {
        let program = self.token_program.to_account_info();
        let accounts = token::Transfer {
            from: self.vault.to_account_info(),
            to: self.withdraw_destination.to_account_info(),
            authority: self.pool.to_account_info(),
        };
        CpiContext::new(program, accounts)
    }
}

#[account]
pub struct TokenPool {
    vault: Pubkey,
    mint: Pubkey,
    withdraw_destination: Pubkey,
    bump: u8,
}
```

# 实验

让我们通过创建一个简单的程序来练习，演示共享 PDA 如何允许攻击者提取不属于他们的代币。这个实验通过包含初始化所需程序账户的指令来扩展上面的示例。

## 1. 起步

首先，下载位于 [此存储库](https://github.com/Unboxed-Software/solana-pda-sharing/tree/starter) 的 `starter` 分支上的起始代码。起始代码包括一个具有两个指令的程序和测试文件的样板设置。

`initialize_pool` 指令初始化一个新的 `TokenPool`，其中存储了 `vault`、`mint`、`withdraw_destination` 和 `bump`。`vault` 是一个代币账户，其权限设置为使用 `mint` 地址派生的 PDA。

`withdraw_insecure` 指令将 `vault` 代币账户中的代币转移到 `withdraw_destination` 代币账户中。

然而，如所写的种子用于签名并不特定于 vault 的提取目的地，因此打开了程序的安全漏洞。在继续之前，请花一点时间熟悉代码。

## 2. 测试 `withdraw_insecure` 指令

测试文件包括调用 `initialize_pool` 指令然后向 `vault` 代币账户铸造 100 个代币的代码。它还包括一个测试，调用 `withdraw_insecure` 使用预期的 `withdraw_destination`。这显示了指令可以按预期使用。

之后，还有两个测试来展示指令容易受到利用的情况。

第一个测试调用 `initialize_pool` 指令创建一个使用相同的 `vault` 代币账户，但不同的 `withdraw_destination` 的“伪造” `pool` 账户。

第二个测试从此池中提取，从 vault 中窃取资金。

```tsx
it("Insecure initialize allows pool to be initialized with wrong vault", async () => {
    await program.methods
      .initializePool(authInsecureBump)
      .accounts({
        pool: poolInsecureFake.publicKey,
        mint: mint,
        vault: vaultInsecure.address,
        withdrawDestination: withdrawDestinationFake,
        payer: walletFake.publicKey,
      })
      .signers([walletFake, poolInsecureFake])
      .rpc()

    await new Promise((x) => setTimeout(x, 1000))

    await spl.mintTo(
      connection,
      wallet.payer,
      mint,
      vaultInsecure.address,
      wallet.payer,
      100
    )

    const account = await spl.getAccount(connection, vaultInsecure.address)
    expect(Number(account.amount)).to.equal(100)
})

it("Insecure withdraw allows stealing from vault", async () => {
    await program.methods
      .withdrawInsecure()
      .accounts({
        pool: poolInsecureFake.publicKey,
        vault: vaultInsecure.address,
        withdrawDestination: withdrawDestinationFake,
        authority: authInsecure,
        signer: walletFake.publicKey,
      })
      .signers([walletFake])
      .rpc()

    const account = await spl.getAccount(connection, vaultInsecure.address)
    expect(Number(account.amount)).to.equal(0)
})
```

运行 `anchor test` 来查看交易是否成功完成，并且 `withdraw_insecure` 指令允许将 `vault` 代币账户的资金转移到伪造的 `pool` 账户上存储的伪造提取目的地。

## 3. 添加 `initialize_pool_secure` 指令

现在让我们为程序添加一个新指令，用于安全初始化池。

这个新的 `initialize_pool_secure` 指令将初始化一个 `pool` 账户，作为使用 `withdraw_destination` 派生的 PDA。它还将初始化一个 `vault` 代币账户，并将权限设置为 `pool` PDA。

```rust
pub fn initialize_pool_secure(ctx: Context<InitializePoolSecure>) -> Result<()> {
    ctx.accounts.pool.vault = ctx.accounts.vault.key();
    ctx.accounts.pool.mint = ctx.accounts.mint.key();
    ctx.accounts.pool.withdraw_destination = ctx.accounts.withdraw_destination.key();
    ctx.accounts.pool.bump = *ctx.bumps.get("pool").unwrap();
    Ok(())
}

...

#[derive(Accounts)]
pub struct InitializePoolSecure<'info> {
    #[account(
        init,
        payer = payer,
        space = 8 + 32 + 32 + 32 + 1,
        seeds = [withdraw_destination.key().as_ref()],
        bump
    )]
    pub pool: Account<'info, TokenPool>,
    pub mint: Account<'info, Mint>,
    #[account(
        init,
        payer = payer,
        token::mint = mint,
        token::authority = pool,
    )]
    pub vault: Account<'info, TokenAccount>,
    pub withdraw_destination: Account<'info, TokenAccount>,
    #[account(mut)]
    pub payer: Signer<'info>,
    pub system_program: Program<'info, System>,
    pub token_program: Program<'info, Token>,
    pub rent: Sysvar<'info, Rent>,
}
```

## 4. 添加 `withdraw_secure` 指令

接下来，添加一个 `withdraw_secure` 指令。该指令将从 `vault` 代币账户提取代币到 `withdraw_destination`。使用 `seeds` 和 `bump` 约束对 `pool` 账户进行验证，以确保提供了正确的 PDA 账户。`has_one` 约束检查提供了正确的 `vault` 和 `withdraw_destination` 代币账户。

```rust
pub fn withdraw_secure(ctx: Context<WithdrawTokensSecure>) -> Result<()> {
    let amount = ctx.accounts.vault.amount;
    let seeds = &[
    ctx.accounts.pool.withdraw_destination.as_ref(),
      &[ctx.accounts.pool.bump],
    ];
    token::transfer(ctx.accounts.transfer_ctx().with_signer(&[seeds]), amount)
}

...

#[derive(Accounts)]
pub struct WithdrawTokensSecure<'info> {
    #[account(
        has_one = vault,
        has_one = withdraw_destination,
        seeds = [withdraw_destination.key().as_ref()],
        bump = pool.bump,
    )]
    pool: Account<'info, TokenPool>,
    #[account(mut)]
    vault: Account<'info, TokenAccount>,
    #[account(mut)]
    withdraw_destination: Account<'info, TokenAccount>,
    token_program: Program<'info, Token>,
}

impl<'info> WithdrawTokensSecure<'info> {
    pub fn transfer_ctx(&self) -> CpiContext<'_, '_, '_, 'info, token::Transfer<'info>> {
        let program = self.token_program.to_account_info();
        let accounts = token::Transfer {
            from: self.vault.to_account_info(),
            to: self.withdraw_destination.to_account_info(),
            authority: self.pool.to_account_info(),
        };
        CpiContext::new(program, accounts)
    }
}
```

## 5. 测试 `withdraw_secure` 指令

最后，返回到测试文件，测试 `withdraw_secure` 指令，并展示通过缩小我们的 PDA 签名权限范围，我们已经移除了漏洞。

在编写一个显示漏洞已经被修复的测试之前，让我们先编写一个简单的测试，显示初始化和提取指令按预期工作：

```typescript
it("Secure pool initialization and withdraw works", async () => {
    const withdrawDestinationAccount = await getAccount(
      provider.connection,
      withdrawDestination
    )

    await program.methods
      .initializePoolSecure()
      .accounts({
        pool: authSecure,
        mint: mint,
        vault: vaultRecommended.publicKey,
        withdrawDestination: withdrawDestination,
      })
      .signers([vaultRecommended])
      .rpc()

    await new Promise((x) => setTimeout(x, 1000))

    await spl.mintTo(
      connection,
      wallet.payer,
      mint,
      vaultRecommended.publicKey,
      wallet.payer,
      100
    )

    await program.methods
      .withdrawSecure()
      .accounts({
        pool: authSecure,
        vault: vaultRecommended.publicKey,
        withdrawDestination: withdrawDestination,
      })
      .rpc()

    const afterAccount = await getAccount(
      provider.connection,
      withdrawDestination
    )

    expect(
      Number(afterAccount.amount) - Number(withdrawDestinationAccount.amount)
    ).to.equal(100)
})
```

现在，我们将测试漏洞不再存在。由于 `vault` 的权限是使用预期的 `withdraw_destination` 代币账户派生的 `pool` PDA，因此不再有办法提取到除预期的 `withdraw_destination` 之外的账户。

添加一个测试，显示您无法使用错误的提取目的地调用 `withdraw_secure`。它可以使用之前测试中创建的池和 vault。

```typescript
  it("Secure withdraw doesn't allow withdraw to wrong destination", async () => {
    try {
      await program.methods
        .withdrawSecure()
        .accounts({
          pool: authSecure,
          vault: vaultRecommended.publicKey,
          withdrawDestination: withdrawDestinationFake,
        })
        .signers([walletFake])
        .rpc()

      assert.fail("expected error")
    } catch (error) {
      console.log(error.message)
      expect(error)
    }
  })
```

最后，由于 `pool` 账户是使用 `withdraw_destination` 代币账户派生的 PDA，我们无法使用相同的 PDA 创建伪造的 `pool` 账户。添加一个测试，显示新的 `initialize_pool_secure` 指令不会让攻击者放入错误的 vault。

```typescript
it("Secure pool initialization doesn't allow wrong vault", async () => {
    try {
      await program.methods
        .initializePoolSecure()
        .accounts({
          pool: authSecure,
          mint: mint,
          vault: vaultInsecure.address,
          withdrawDestination: withdrawDestination,
        })
        .signers([vaultRecommended])
        .rpc()

      assert.fail("expected error")
    } catch (error) {
      console.log(error.message)
      expect(error)
    }
})
```

运行 `anchor test`，查看新的指令不允许攻击者从不属于他们的 vault 中提取资金。

```
  pda-sharing
    ✔ Initialize Pool Insecure (981ms)
    ✔ Withdraw (470ms)
    ✔ Insecure initialize allows pool to be initialized with wrong vault (10983ms)
    ✔ Insecure withdraw allows stealing from vault (492ms)
    ✔ Secure pool initialization and withdraw works (2502ms)
unknown signer: ARjxAsEPj6YsAPKaBfd1AzUHbNPtAeUsqusAmBchQTfV
    ✔ Secure withdraw doesn't allow withdraw to wrong destination
unknown signer: GJcHJLot3whbY1aC9PtCsBYk5jWoZnZRJPy5uUwzktAY
    ✔ Secure pool initialization doesn't allow wrong vault
```

这就是全部！与我们讨论过的其他一些安全漏洞不同，这个漏洞更多是概念性的，不能通过简单地使用特定的 Anchor 类型来修复。您需要仔细思考您的程序架构，并确保您不会跨不同的域共享 PDAs。

如果您想查看最终的解决方案代码，可以在 [相同的存储库](https://github.com/Unboxed-Software/solana-pda-sharing/tree/solution) 的 `solution` 分支上找到它。

# 挑战

与本单元的其他课程一样，您练习避免这种安全漏洞的机会在于审查您自己或其他程序。

花些时间审查至少一个程序，并查找其 PDA 结构中的潜在漏洞。用于签名的 PDA 应尽可能狭窄，专注于单个域。

请记住，如果您在别人的程序中发现了错误或漏洞，请立即通知他们！如果您在自己的程序中发现了错误或漏洞，请务必立即修复。

## 完成实验了吗？

将您的代码推送到 GitHub，并[告诉我们您对本课程的看法](https://form.typeform.com/to/IPH0UGz7#answers-lesson=5744079f-9473-4485-9a14-9be4d31b40d1)!