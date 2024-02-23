---
title: 签名者授权
objectives:
- 解释未执行适当签名者检查所带来的安全风险
- 使用 Rust 的长格式实现签名者检查
- 使用 Anchor 的 `Signer` 类型实现签名者检查
- 使用 Anchor 的 `#[account(signer)]` 约束实现签名者检查
---

# 总结

- 使用**签名者检查**来验证特定账户是否已对交易进行签名。如果没有适当的签名者检查，账户可能会执行它们未被授权执行的指令。
- 要在 Rust 中实现签名者检查，只需检查账户的 `is_signer` 属性是否为 `true`

    ```rust
    if !ctx.accounts.authority.is_signer {
    	return Err(ProgramError::MissingRequiredSignature.into());
    }
    ```
    
- 在 Anchor 中，您可以在账户验证结构体中使用 **`Signer`** 账户类型，让 Anchor 自动对给定账户执行签名者检查
- Anchor 还有一个账户约束，将自动验证给定账户是否已对交易进行签名

# 概述

签名者检查用于验证给定账户的所有者是否已授权交易。如果没有签名者检查，仅应由特定账户执行的操作可能会被任何账户执行。在最坏的情况下，这可能导致攻击者通过传入他们想要的任何账户来执行指令，从而完全耗尽钱包的资金。

## 缺失的签名者检查

下面的示例显示了一个简化版的指令，该指令更新了存储在程序账户上的 `authority` 字段。

请注意，`UpdateAuthority` 账户验证结构体上的 `authority` 字段的类型为 `AccountInfo`。在 Anchor 中，`AccountInfo` 账户类型表示在执行指令之前不对账户进行任何检查。

虽然使用了 `has_one` 约束来验证传入指令的 `authority` 账户是否与 `vault` 账户上存储的 `authority` 字段匹配，但没有检查验证 `authority` 账户是否授权了交易。

这意味着攻击者可以简单地传入 `authority` 账户的公钥和他们自己的公钥作为 `new_authority` 账户，以重新分配自己为 `vault` 账户的新授权者。在那时，他们可以以新的授权者身份与程序进行交互。

```rust
use anchor_lang::prelude::*;

declare_id!("Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS");

#[program]
pub mod insecure_update{
    use super::*;
        ...
        pub fn update_authority(ctx: Context<UpdateAuthority>) -> Result<()> {
        ctx.accounts.vault.authority = ctx.accounts.new_authority.key();
        Ok(())
    }
}

#[derive(Accounts)]
pub struct UpdateAuthority<'info> {
   #[account(
        mut,
        has_one = authority
    )]
    pub vault: Account<'info, Vault>,
    pub new_authority: AccountInfo<'info>,
    pub authority: AccountInfo<'info>,
}

#[account]
pub struct Vault {
    token_account: Pubkey,
    authority: Pubkey,
}
```

## 添加签名者授权检查

要验证 `authority` 账户是否签署了交易，您需要在指令中添加一个签名者检查。这简单地意味着检查 `authority.is_signer` 是否为 `true`，如果为 `false`，则返回一个 `MissingRequiredSignature` 错误。

```tsx
if !ctx.accounts.authority.is_signer {
    return Err(ProgramError::MissingRequiredSignature.into());
}
```

通过添加签名者检查，指令只会在传入的 `authority` 账户也签署了交易时才会执行。如果交易没有由传入的 `authority` 账户签署，那么交易将失败。

```rust
use anchor_lang::prelude::*;

declare_id!("Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS");

#[program]
pub mod secure_update{
    use super::*;
        ...
        pub fn update_authority(ctx: Context<UpdateAuthority>) -> Result<()> {
            if !ctx.accounts.authority.is_signer {
            return Err(ProgramError::MissingRequiredSignature.into());
        }

        ctx.accounts.vault.authority = ctx.accounts.new_authority.key();
        Ok(())
    }
}

#[derive(Accounts)]
pub struct UpdateAuthority<'info> {
    #[account(
        mut,
        has_one = authority
    )]
    pub vault: Account<'info, Vault>,
    pub new_authority: AccountInfo<'info>,
    pub authority: AccountInfo<'info>,
}

#[account]
pub struct Vault {
    token_account: Pubkey,
    authority: Pubkey,
}
```

## 使用 Anchor 的 `Signer` 账户类型

然而，将此检查放入指令函数会混淆账户验证和指令逻辑之间的分离。

幸运的是，Anchor 通过提供 `Signer` 账户类型，使执行签名者检查变得很容易。只需将账户验证结构体中的 `authority` 账户的类型更改为 `Signer` 类型，Anchor 将在运行时检查指定的账户是否为交易的签名者。这通常是我们推荐的方法，因为它允许您将签名者检查与指令逻辑分开。

在下面的示例中，如果 `authority` 账户未对交易进行签名，那么交易将在甚至到达指令逻辑之前失败。

```rust
use anchor_lang::prelude::*;

declare_id!("Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS");

#[program]
pub mod secure_update{
    use super::*;
        ...
        pub fn update_authority(ctx: Context<UpdateAuthority>) -> Result<()> {
        ctx.accounts.vault.authority = ctx.accounts.new_authority.key();
        Ok(())
    }
}

#[derive(Accounts)]
pub struct UpdateAuthority<'info> {
    #[account(
        mut,
        has_one = authority
    )]
    pub vault: Account<'info, Vault>,
    pub new_authority: AccountInfo<'info>,
    pub authority: Signer<'info>,
}

#[account]
pub struct Vault {
    token_account: Pubkey,
    authority: Pubkey,
}
```

请注意，当您使用 `Signer` 类型时，不会执行其他所有权或类型检查。

## 使用 Anchor 的 `#[account(signer)]` 约束

在大多数情况下，`Signer` 账户类型足以确保账户已对交易进行签名，但没有执行其他所有权或类型检查的事实意味着该账户实际上无法用于指令中的其他任何操作。

这就是 `signer` *约束* 发挥作用的地方。`#[account(signer)]` 约束允许您验证账户是否已对交易进行签名，同时还可以享受使用 `Account` 类型的好处，如果您希望访问其底层数据。

举一个这种情况下会有用的例子，想象一下编写一个预期通过 CPI 调用的指令，该指令希望传入的账户之一既是交易的签名者又是数据源。在这里使用 `Signer` 账户类型会删除您将使用 `Account` 类型时自动反序列化和类型检查。这既不方便，因为您需要在指令逻辑中手动反序列化账户数据，而且可能会通过不执行 `Account` 类型的所有权和类型检查使您的程序变得容易受到攻击。

在下面的示例中，您可以安全地编写逻辑来与存储在 `authority` 账户中的数据进行交互，同时验证它是否对交易进行了签名。

```rust
use anchor_lang::prelude::*;

declare_id!("Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS");

#[program]
pub mod secure_update{
    use super::*;
        ...
        pub fn update_authority(ctx: Context<UpdateAuthority>) -> Result<()> {
        ctx.accounts.vault.authority = ctx.accounts.new_authority.key();

        // access the data stored in authority
        msg!("Total number of depositors: {}", ctx.accounts.authority.num_depositors);
        Ok(())
    }
}

#[derive(Accounts)]
pub struct UpdateAuthority<'info> {
    #[account(
        mut,
        has_one = authority
    )]
    pub vault: Account<'info, Vault>,
    pub new_authority: AccountInfo<'info>,
    #[account(signer)]
    pub authority: Account<'info, AuthState>
}

#[account]
pub struct Vault {
    token_account: Pubkey,
    authority: Pubkey,
}
#[account]
pub struct AuthState{
	amount: u64,
	num_depositors: u64,
	num_vaults: u64
}
```

# 实验

让我们通过创建一个简单的程序来练习，演示缺少签名者检查如何允许攻击者提取不属于他们的代币。

该程序初始化了一个简化的代币“保险库（vault）”账户，并演示了缺少签名者检查如何导致保险库被耗尽。

## 1. 起步

要开始，请从[此存储库](https://github.com/Unboxed-Software/solana-signer-auth/tree/starter)的 `starter` 分支下载起始代码。起始代码包括一个具有两个指令和测试文件的模板程序。

`initialize_vault` 指令初始化了两个新账户：`Vault` 和 `TokenAccount`。`Vault` 账户将使用程序派生地址 (PDA) 进行初始化，并存储一个代币账户的地址和保险库的授权。代币账户的权限将是 `vault` PDA，这使得程序可以签署代币的转移。

`insecure_withdraw` 指令将从 `vault` 账户的代币账户中转移代币到 `withdraw_destination` 代币账户。然而，在 `InsecureWithdraw` 结构中，`authority` 账户的类型是 `UncheckedAccount`。这是 `AccountInfo` 的一个包装，用于明确指示该账户未经检查。

如果没有签名者检查，任何人都可以简单地提供与存储在 `vault` 账户上的 `authority` 匹配的 `authority` 账户的公钥，`insecure_withdraw` 指令将继续处理。

虽然这有些牵强，因为任何具有保险库的 DeFi 程序都比这更复杂，但它将展示没有签名者检查会导致代币被错误的一方提取。

```rust
use anchor_lang::prelude::*;
use anchor_spl::token::{self, Mint, Token, TokenAccount};

declare_id!("Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS");

#[program]
pub mod signer_authorization {
    use super::*;

    pub fn initialize_vault(ctx: Context<InitializeVault>) -> Result<()> {
        ctx.accounts.vault.token_account = ctx.accounts.token_account.key();
        ctx.accounts.vault.authority = ctx.accounts.authority.key();
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
        space = 8 + 32 + 32,
        seeds = [b"vault"],
        bump
    )]
    pub vault: Account<'info, Vault>,
    #[account(
        init,
        payer = authority,
        token::mint = mint,
        token::authority = vault,
    )]
    pub token_account: Account<'info, TokenAccount>,
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
        has_one = token_account,
        has_one = authority
    )]
    pub vault: Account<'info, Vault>,
    #[account(mut)]
    pub token_account: Account<'info, TokenAccount>,
    #[account(mut)]
    pub withdraw_destination: Account<'info, TokenAccount>,
    pub token_program: Program<'info, Token>,
    /// CHECK: demo missing signer check
    pub authority: UncheckedAccount<'info>,
}

#[account]
pub struct Vault {
    token_account: Pubkey,
    authority: Pubkey,
}
```

## 2. 测试 `insecure_withdraw` 指令

测试文件包含了使用 `wallet` 作为保险库上的 `authority` 调用 `initialize_vault` 指令的代码。然后，该代码向 `vault` 代币账户铸造了 100 个代币。理论上，`wallet` 密钥应该是唯一能够从保险库中提取这 100 个代币的密钥。

现在，让我们添加一个测试来调用程序上的 `insecure_withdraw`，以展示当前版本的程序实际上允许第三方提取这 100 个代币。

在测试中，我们仍然将使用 `wallet` 的公钥作为 `authority` 账户，但我们将使用另一个密钥对来签名和发送交易。

```tsx
describe("signer-authorization", () => {
    ...
    it("Insecure withdraw", async () => {
    const tx = await program.methods
      .insecureWithdraw()
      .accounts({
        vault: vaultPDA,
        tokenAccount: tokenAccount.publicKey,
        withdrawDestination: withdrawDestinationFake,
        authority: wallet.publicKey,
      })
      .transaction()

    await anchor.web3.sendAndConfirmTransaction(connection, tx, [walletFake])

    const balance = await connection.getTokenAccountBalance(
      tokenAccount.publicKey
    )
    expect(balance.value.uiAmount).to.eq(0)
  })
})
```

运行 `anchor test` 来查看两个交易都将成功完成。

```bash
signer-authorization
  ✔ Initialize Vault (810ms)
  ✔ Insecure withdraw  (405ms)
```

由于 `authority` 账户没有签名者检查，只要 `authority` 账户的公钥与 `vault` 账户的 `authority` 字段中存储的公钥匹配，`insecure_withdraw` 指令就会将代币从 `vault` 代币账户转移到 `withdrawDestinationFake` 代币账户。显然，`insecure_withdraw` 指令正如其名字所暗示的那样不安全。

## 3. 添加 `secure_withdraw` 指令

让我们在一个新的指令中修复这个问题，称为 `secure_withdraw`。这个指令将与 `insecure_withdraw` 指令相同，只是我们将在 `SecureWithdraw` 结构中的 Accounts 结构中使用 `Signer` 类型来验证 `authority` 账户。如果 `authority` 账户不是交易的签名者，那么我们期望交易将失败并返回一个错误。

```rust
use anchor_lang::prelude::*;
use anchor_spl::token::{self, Mint, Token, TokenAccount};

declare_id!("Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS");

#[program]
pub mod signer_authorization {
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
        has_one = authority
    )]
    pub vault: Account<'info, Vault>,
    #[account(mut)]
    pub token_account: Account<'info, TokenAccount>,
    #[account(mut)]
    pub withdraw_destination: Account<'info, TokenAccount>,
    pub token_program: Program<'info, Token>,
    pub authority: Signer<'info>,
}
```

## 4. 测试 `secure_withdraw` 指令

有了指令之后，回到测试文件来测试 `secure_withdraw` 指令。再次调用 `secure_withdraw` 指令，仍然使用 `wallet` 的公钥作为 `authority` 账户，但使用 `withdrawDestinationFake` 密钥对作为签名者和提取目的地。由于 `authority` 账户是使用 `Signer` 类型进行验证的，我们期望交易将失败签名者检查，并返回一个错误。

```tsx
describe("signer-authorization", () => {
    ...
	it("Secure withdraw", async () => {
    try {
      const tx = await program.methods
        .secureWithdraw()
        .accounts({
          vault: vaultPDA,
          tokenAccount: tokenAccount.publicKey,
          withdrawDestination: withdrawDestinationFake,
          authority: wallet.publicKey,
        })
        .transaction()

      await anchor.web3.sendAndConfirmTransaction(connection, tx, [walletFake])
    } catch (err) {
      expect(err)
      console.log(err)
    }
  })
})
```

运行 `anchor test` 来查看交易现在将返回一个签名验证错误。

```bash
Error: Signature verification failed
```

就是这样！这是一件相当简单的事情可以避免，但却非常重要。确保始终考虑由谁授权指令，并确保每个人都是交易的签名者。

如果你想查看最终的解决方案代码，你可以在[该存储库的`solution`分支](https://github.com/Unboxed-Software/solana-signer-auth/tree/solution)上找到它。

# 挑战

在课程的这一阶段，我们希望你已经开始在课程之外的程序和项目上进行工作。在本课程和余下的安全漏洞课程中，每个课程的挑战将是审计你自己的代码，以查找课程中讨论的安全漏洞。

或者，你可以找到开源程序进行审计。有很多程序可以供你查看。如果你不介意深入研究原生 Rust，那么[SPL程序](https://github.com/solana-labs/solana-program-library)是一个很好的起点。

因此，在本课程中，审查一个程序（无论是你自己的还是在线找到的程序），并对签名者检查进行审计。如果你发现别人程序中的漏洞，请及时通知他们！如果你发现自己程序中的漏洞，请务必立即修补。

## 完成实验了吗？

将你的代码推送到 GitHub，并[告诉我们你对本课程的想法](https://form.typeform.com/to/IPH0UGz7#answers-lesson=26b3f41e-8241-416b-9cfa-05c5ab519d80)！