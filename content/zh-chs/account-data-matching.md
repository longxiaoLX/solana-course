---
title: 账户数据匹配
objectives:
- 解释与缺少数据验证检查相关的安全风险
- 使用长格式 Rust 实现数据验证检查
- 使用 Anchor 约束实现数据验证检查
---

# 总结

- 使用**数据验证检查（data validation checks）**来验证账户数据是否与预期值匹配。如果没有适当的数据验证检查，意外的账户可能会被用于指令中。
- 在 Rust 中实现数据验证检查，只需将存储在账户上的数据与预期值进行比较。
    
    ```rust
    if ctx.accounts.user.key() != ctx.accounts.user_data.user {
        return Err(ProgramError::InvalidAccountData.into());
    }
    ```

- 在 Anchor 中，你可以使用 `constraint` 来检查给定表达式是否为 true。或者，你可以使用 `has_one` 来检查存储在账户上的目标账户字段是否与 `Accounts` 结构中的账户的键匹配。

# 概述

账户数据匹配是指用于验证账户存储的数据是否与预期值匹配的数据验证检查。数据验证检查提供了一种包含额外约束的方式，以确保将适当的账户传递到指令中。

当指令所需的账户依赖于其他账户中存储的值，或者指令依赖于账户中存储的数据时，这种方法非常有用。

## 缺少数据验证检查

下面的示例包括一个 `update_admin` 指令，用于更新存储在 `admin_config` 账户上的 `admin` 字段。

该指令缺少数据验证检查，以验证签署交易的 `admin` 账户是否与存储在 `admin_config` 账户上的 `admin` 匹配。这意味着任何签署交易并作为 `admin` 账户传递到指令中的账户都可以更新 `admin_config` 账户。

```rust
use anchor_lang::prelude::*;

declare_id!("Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS");

#[program]
pub mod data_validation {
    use super::*;
    ...
    pub fn update_admin(ctx: Context<UpdateAdmin>) -> Result<()> {
        ctx.accounts.admin_config.admin = ctx.accounts.new_admin.key();
        Ok(())
    }
}

#[derive(Accounts)]
pub struct UpdateAdmin<'info> {
    #[account(mut)]
    pub admin_config: Account<'info, AdminConfig>,
    #[account(mut)]
    pub admin: Signer<'info>,
    pub new_admin: SystemAccount<'info>,
}

#[account]
pub struct AdminConfig {
    admin: Pubkey,
}
```

## 添加数据验证检查

解决这个问题的基本 Rust 方法是简单地将传入的 `admin` 密钥与存储在 `admin_config` 账户中的 `admin` 密钥进行比较，如果它们不匹配，则抛出错误。

```rust
if ctx.accounts.admin.key() != ctx.accounts.admin_config.admin {
    return Err(ProgramError::InvalidAccountData.into());
}
```

通过添加数据验证检查，`update_admin` 指令将仅在交易的 `admin` 签署者与存储在 `admin_config` 账户上的 `admin` 匹配时才会执行。

```rust
use anchor_lang::prelude::*;

declare_id!("Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS");

#[program]
pub mod data_validation {
    use super::*;
    ...
    pub fn update_admin(ctx: Context<UpdateAdmin>) -> Result<()> {
      if ctx.accounts.admin.key() != ctx.accounts.admin_config.admin {
            return Err(ProgramError::InvalidAccountData.into());
        }
        ctx.accounts.admin_config.admin = ctx.accounts.new_admin.key();
        Ok(())
    }
}

#[derive(Accounts)]
pub struct UpdateAdmin<'info> {
    #[account(mut)]
    pub admin_config: Account<'info, AdminConfig>,
    #[account(mut)]
    pub admin: Signer<'info>,
    pub new_admin: SystemAccount<'info>,
}

#[account]
pub struct AdminConfig {
    admin: Pubkey,
}
```

## 使用 Anchor 约束

Anchor 使用 `has_one` 约束简化了这一过程。你可以使用 `has_one` 约束将数据验证检查从指令逻辑移到 `UpdateAdmin` 结构中。

在下面的示例中，`has_one = admin` 指定了签署交易的 `admin` 账户必须与存储在 `admin_config` 账户上的 `admin` 字段匹配。要使用 `has_one` 约束，账户上的数据字段的命名约定必须与账户验证结构中的命名一致。

```rust
use anchor_lang::prelude::*;

declare_id!("Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS");

#[program]
pub mod data_validation {
    use super::*;
    ...
    pub fn update_admin(ctx: Context<UpdateAdmin>) -> Result<()> {
        ctx.accounts.admin_config.admin = ctx.accounts.new_admin.key();
        Ok(())
    }
}

#[derive(Accounts)]
pub struct UpdateAdmin<'info> {
    #[account(
        mut,
        has_one = admin
    )]
    pub admin_config: Account<'info, AdminConfig>,
    #[account(mut)]
    pub admin: Signer<'info>,
    pub new_admin: SystemAccount<'info>,
}

#[account]
pub struct AdminConfig {
    admin: Pubkey,
}
```

或者，你可以使用 `constraint` 手动添加一个必须为 true 的表达式，以便执行可以继续进行。当命名不能保持一致或当你需要一个更复杂的表达式来完全验证传入的数据时，这是非常有用的。

```rust
#[derive(Accounts)]
pub struct UpdateAdmin<'info> {
    #[account(
        mut,
        constraint = admin_config.admin == admin.key()
    )]
    pub admin_config: Account<'info, AdminConfig>,
    #[account(mut)]
    pub admin: Signer<'info>,
    pub new_admin: SystemAccount<'info>,
}
```

# 实验

在这个实验中，我们将创建一个简单的“保险库”程序，类似于我们在签名者授权课程和所有者检查课程中使用的程序。与那些实验室类似，我们将在这个实验室中展示缺少数据验证检查如何导致保险库被清空。

## 1. 起始

为了开始，请从[该存储库的`starter`分支](https://github.com/Unboxed-Software/solana-account-data-matching)下载起始代码。起始代码包括一个具有两个指令的程序和测试文件的样板设置。

`initialize_vault` 指令初始化一个新的 `Vault` 账户和一个新的 `TokenAccount`。`Vault` 账户将存储一个代币账户的地址、保险库的授权以及一个提取目标代币账户。

新代币账户的授权将设置为程序的 PDA `vault`。这允许 `vault` 账户签署从代币账户转移代币的操作。

`insecure_withdraw` 指令将 `vault` 账户中的所有代币转移到一个 `withdraw_destination` 代币账户中。

请注意，这个指令确实对 `authority` 进行了签名检查，并对 `vault` 进行了所有者检查。然而，在账户验证或指令逻辑中，没有代码检查传递到指令中的 `authority` 账户是否与 `vault` 上的 `authority` 账户匹配。

```rust
use anchor_lang::prelude::*;
use anchor_spl::token::{self, Mint, Token, TokenAccount};

declare_id!("Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS");

#[program]
pub mod account_data_matching {
    use super::*;

    pub fn initialize_vault(ctx: Context<InitializeVault>) -> Result<()> {
        ctx.accounts.vault.token_account = ctx.accounts.token_account.key();
        ctx.accounts.vault.authority = ctx.accounts.authority.key();
        ctx.accounts.vault.withdraw_destination = ctx.accounts.withdraw_destination.key();
        Ok(())
    }

    pub fn insecure_withdraw(ctx: Context<InsecureWithdraw>) -> Result<()> {
        let amount = ctx.accounts.token_account.amount;

        let seeds = &[b"vault".as_ref(), &[*ctx.bumps.get("vault").unwrap()]];
        let signer = [&seeds[..]];

        let cpi_ctx = CpiContext::new_with_signer(
            ctx.accounts.token_program.to_account_info(),
            token::Transfer {
                from: ctx.accounts.token_account.to_account_info(),
                authority: ctx.accounts.vault.to_account_info(),
                to: ctx.accounts.withdraw_destination.to_account_info(),
            },
            &signer,
        );

        token::transfer(cpi_ctx, amount)?;
        Ok(())
    }
}

#[derive(Accounts)]
pub struct InitializeVault<'info> {
    #[account(
        init,
        payer = authority,
        space = 8 + 32 + 32 + 32,
        seeds = [b"vault"],
        bump,
    )]
    pub vault: Account<'info, Vault>,
    #[account(
        init,
        payer = authority,
        token::mint = mint,
        token::authority = vault,
        seeds = [b"token"],
        bump,
    )]
    pub token_account: Account<'info, TokenAccount>,
    pub withdraw_destination: Account<'info, TokenAccount>,
    pub mint: Account<'info, Mint>,
    #[account(mut)]
    pub authority: Signer<'info>,
    pub token_program: Program<'info, Token>,
    pub system_program: Program<'info, System>,
    pub rent: Sysvar<'info, Rent>,
}

#[derive(Accounts)]
pub struct InsecureWithdraw<'info> {
    #[account(
        seeds = [b"vault"],
        bump,
    )]
    pub vault: Account<'info, Vault>,
    #[account(
        mut,
        seeds = [b"token"],
        bump,
    )]
    pub token_account: Account<'info, TokenAccount>,
    #[account(mut)]
    pub withdraw_destination: Account<'info, TokenAccount>,
    pub token_program: Program<'info, Token>,
    pub authority: Signer<'info>,
}

#[account]
pub struct Vault {
    token_account: Pubkey,
    authority: Pubkey,
    withdraw_destination: Pubkey,
}
```

## 2. 测试 `insecure_withdraw` 指令

为了证明这是一个问题，让我们编写一个测试，在保险库之外的账户尝试从保险库中提取。

测试文件包括使用提供者钱包作为 `authority` 调用 `initialize_vault` 指令的代码，然后将 100 个代币铸造到 `vault` 代币账户中。

添加一个测试来调用 `insecure_withdraw` 指令。使用 `withdrawDestinationFake` 作为 `withdrawDestination` 账户，并使用 `walletFake` 作为 `authority`。然后使用 `walletFake` 发送交易。

由于没有检查来验证传递到指令中的 `authority` 账户是否与在第一个测试中初始化的 `vault` 账户上存储的值匹配，因此该指令将成功执行，并且代币将被转移到 `withdrawDestinationFake` 账户中。

```tsx
describe("account-data-matching", () => {
  ...
  it("Insecure withdraw", async () => {
    const tx = await program.methods
      .insecureWithdraw()
      .accounts({
        vault: vaultPDA,
        tokenAccount: tokenPDA,
        withdrawDestination: withdrawDestinationFake,
        authority: walletFake.publicKey,
      })
      .transaction()

    await anchor.web3.sendAndConfirmTransaction(connection, tx, [walletFake])

    const balance = await connection.getTokenAccountBalance(tokenPDA)
    expect(balance.value.uiAmount).to.eq(0)
  })
})
```

运行 `anchor test`，查看两个交易是否都成功完成。

```bash
account-data-matching
  ✔ Initialize Vault (811ms)
  ✔ Insecure withdraw (403ms)
```

## 3. 添加 `secure_withdraw` 指令

让我们来实现一个安全版本的该指令，称为 `secure_withdraw`。

该指令将与 `insecure_withdraw` 指令相同，只是我们将在账户验证结构（`SecureWithdraw`）中使用 `has_one` 约束来检查传递到指令中的 `authority` 账户是否与 `vault` 账户上的 `authority` 账户匹配。这样只有正确的授权账户才能提取保险库的代币。

```rust
use anchor_lang::prelude::*;
use anchor_spl::token::{self, Mint, Token, TokenAccount};

declare_id!("Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS");

#[program]
pub mod account_data_matching {
    use super::*;
    ...
    pub fn secure_withdraw(ctx: Context<SecureWithdraw>) -> Result<()> {
        let amount = ctx.accounts.token_account.amount;

        let seeds = &[b"vault".as_ref(), &[*ctx.bumps.get("vault").unwrap()]];
        let signer = [&seeds[..]];

        let cpi_ctx = CpiContext::new_with_signer(
            ctx.accounts.token_program.to_account_info(),
            token::Transfer {
                from: ctx.accounts.token_account.to_account_info(),
                authority: ctx.accounts.vault.to_account_info(),
                to: ctx.accounts.withdraw_destination.to_account_info(),
            },
            &signer,
        );

        token::transfer(cpi_ctx, amount)?;
        Ok(())
    }
}

#[derive(Accounts)]
pub struct SecureWithdraw<'info> {
    #[account(
        seeds = [b"vault"],
        bump,
        has_one = token_account,
        has_one = authority,
        has_one = withdraw_destination,

    )]
    pub vault: Account<'info, Vault>,
    #[account(
        mut,
        seeds = [b"token"],
        bump,
    )]
    pub token_account: Account<'info, TokenAccount>,
    #[account(mut)]
    pub withdraw_destination: Account<'info, TokenAccount>,
    pub token_program: Program<'info, Token>,
    pub authority: Signer<'info>,
}
```

## 4. 测试 `secure_withdraw` 指令

现在让我们使用两个测试来测试 `secure_withdraw` 指令：一个使用 `walletFake` 作为授权者，另一个使用 `wallet` 作为授权者。我们期望第一个调用返回错误，而第二个调用成功。

```tsx
describe("account-data-matching", () => {
  ...
  it("Secure withdraw, expect error", async () => {
    try {
      const tx = await program.methods
        .secureWithdraw()
        .accounts({
          vault: vaultPDA,
          tokenAccount: tokenPDA,
          withdrawDestination: withdrawDestinationFake,
          authority: walletFake.publicKey,
        })
        .transaction()

      await anchor.web3.sendAndConfirmTransaction(connection, tx, [walletFake])
    } catch (err) {
      expect(err)
      console.log(err)
    }
  })

  it("Secure withdraw", async () => {
    await spl.mintTo(
      connection,
      wallet.payer,
      mint,
      tokenPDA,
      wallet.payer,
      100
    )

    await program.methods
      .secureWithdraw()
      .accounts({
        vault: vaultPDA,
        tokenAccount: tokenPDA,
        withdrawDestination: withdrawDestination,
        authority: wallet.publicKey,
      })
      .rpc()

    const balance = await connection.getTokenAccountBalance(tokenPDA)
    expect(balance.value.uiAmount).to.eq(0)
  })
})
```

运行 `anchor test`，查看使用不正确的授权账户的交易是否现在返回 Anchor 错误，而使用正确账户的交易是否成功完成。

```bash
'Program Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS invoke [1]',
'Program log: Instruction: SecureWithdraw',
'Program log: AnchorError caused by account: vault. Error Code: ConstraintHasOne. Error Number: 2001. Error Message: A has one constraint was violated.',
'Program log: Left:',
'Program log: DfLZV18rD7wCQwjYvhTFwuvLh49WSbXFeJFPQb5czifH',
'Program log: Right:',
'Program log: 5ovvmG5ntwUC7uhNWfirjBHbZD96fwuXDMGXiyMwPg87',
'Program Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS consumed 10401 of 200000 compute units',
'Program Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS failed: custom program error: 0x7d1'
```

请注意，Anchor 在日志中指定导致错误的账户（`AnchorError caused by account: vault`）。

```bash
✔ Secure withdraw, expect error (77ms)
✔ Secure withdraw (10073ms)
```

就是这样，你已经堵住了安全漏洞。大部分潜在漏洞都是相当简单的。然而，随着你的程序范围和复杂性的增加，很容易忽略可能的漏洞。养成编写运行失败的指令测试的习惯是非常好的。测试越多越好。这样你就能在部署之前发现问题。

如果你想查看最终的解决方案代码，你可以在[该存储库的`solution`分支](https://github.com/Unboxed-Software/solana-account-data-matching/tree/solution)上找到它。

# 挑战

和本单元的其他课程一样，你有机会练习避免这种安全漏洞，方法是审计你自己或其他程序。

花一些时间审查至少一个程序，并确保已经放置了适当的数据检查以避免安全漏洞。

请记住，如果你在别人的程序中发现了漏洞或利用漏洞，请立即通知他们！如果你在自己的程序中发现了漏洞，请务必立即修补。

## 完成了实验吗？

将你的代码推送到 GitHub，并[告诉我们你对本课程的想法](https://form.typeform.com/to/IPH0UGz7#answers-lesson=a107787e-ad33-42bb-96b3-0592efc1b92f)！