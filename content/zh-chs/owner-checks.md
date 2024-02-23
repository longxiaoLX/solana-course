---
title: 所有者检查
objectives:
- 解释不执行适当的所有者检查所带来的安全风险
- 使用长格式 Rust 实现所有者检查
- 使用 Anchor 的 `Account<'info, T>` 包装器和一个账户类型来自动执行所有者检查
- 使用 Anchor 的 `#[account(owner = <expr>)]` 约束来明确定义一个外部程序应该拥有一个账户
---

# 总结

- 使用**所有者检查**来验证账户是否由预期的程序拥有。如果没有适当的所有者检查，那些被意外程序拥有的账户可能会在指令中被使用。
- 要在 Rust 中实现所有者检查，只需检查账户的所有者是否与预期的程序 ID 匹配。

```rust
if ctx.accounts.account.owner != ctx.program_id {
    return Err(ProgramError::IncorrectProgramId.into());
}
```

- Anchor 程序账户类型实现了 `Owner` trait，这允许 `Account<'info, T>` 包装器自动验证程序的所有权。
- Anchor 为您提供了一个选项，即如果账户的所有者不应该是当前执行的程序，则可以明确地定义账户的所有者。

# 概述

**所有者检查（Owner Checks）** 用于验证传入指令的账户是否由预期的程序所拥有。这可以防止由意外程序拥有的账户被用于指令中。

作为提醒，`AccountInfo` 结构包含以下字段。所有者检查是指检查 `AccountInfo` 中的 `owner` 字段是否与预期的程序 ID 匹配。

```rust
/// Account information
#[derive(Clone)]
pub struct AccountInfo<'a> {
    /// Public key of the account
    pub key: &'a Pubkey,
    /// Was the transaction signed by this account's public key?
    pub is_signer: bool,
    /// Is the account writable?
    pub is_writable: bool,
    /// The lamports in the account.  Modifiable by programs.
    pub lamports: Rc<RefCell<&'a mut u64>>,
    /// The data held in this account.  Modifiable by programs.
    pub data: Rc<RefCell<&'a mut [u8]>>,
    /// Program that owns this account
    pub owner: &'a Pubkey,
    /// This account's data contains a loaded program (and is now read-only)
    pub executable: bool,
    /// The epoch at which this account will next owe rent
    pub rent_epoch: Epoch,
}
```

### 缺少所有者检查

下面的示例显示了一个 `admin_instruction`，该指令旨在仅由存储在 `admin_config` 账户上的 `admin` 账户访问。

尽管该指令检查了 `admin` 账户是否在交易中签名，并且与存储在 `admin_config` 账户上的 `admin` 字段匹配，但没有所有者检查来验证传递到指令中的 `admin_config` 账户是否由执行程序拥有。

由于 `admin_config` 未经检查，如 `AccountInfo` 类型所示，因此另一个程序可能拥有一个伪造的 `admin_config` 账户，并将其用于 `admin_instruction`。这意味着攻击者可以创建一个程序，其中的 `admin_config` 数据结构与你的程序的 `admin_config` 匹配，将自己的公钥设置为 `admin`，并将他们的 `admin_config` 账户传递给你的程序。这将让他们欺骗你的程序，以为他们是你的程序的授权管理员。

这个简化的示例仅将 `admin` 打印到程序日志中。然而，你可以想象缺少所有者检查可能会允许假账户利用指令的情况。

```rust
use anchor_lang::prelude::*;

declare_id!("Cft4eTTrt4sJU4Ar35rUQHx6PSXfJju3dixmvApzhWws");

#[program]
pub mod owner_check {
    use super::*;
	...

    pub fn admin_instruction(ctx: Context<Unchecked>) -> Result<()> {
        let account_data = ctx.accounts.admin_config.try_borrow_data()?;
        let mut account_data_slice: &[u8] = &account_data;
        let account_state = AdminConfig::try_deserialize(&mut account_data_slice)?;

        if account_state.admin != ctx.accounts.admin.key() {
            return Err(ProgramError::InvalidArgument.into());
        }
        msg!("Admin: {}", account_state.admin.to_string());
        Ok(())
    }
}

#[derive(Accounts)]
pub struct Unchecked<'info> {
    admin_config: AccountInfo<'info>,
    admin: Signer<'info>,
}

#[account]
pub struct AdminConfig {
    admin: Pubkey,
}
```

### 添加所有者检查

在纯 Rust 中，你可以通过比较账户上的 `owner` 字段与程序 ID 来解决这个问题。如果它们不匹配，你将返回一个 `IncorrectProgramId` 错误。

```rust
if ctx.accounts.admin_config.owner != ctx.program_id {
    return Err(ProgramError::IncorrectProgramId.into());
}
```

添加所有者检查可以防止意外程序拥有的账户被传递为 `admin_config` 账户。如果在 `admin_instruction` 中使用了伪造的 `admin_config` 账户，那么交易将失败。

```rust
use anchor_lang::prelude::*;

declare_id!("Cft4eTTrt4sJU4Ar35rUQHx6PSXfJju3dixmvApzhWws");

#[program]
pub mod owner_check {
    use super::*;
    ...
    pub fn admin_instruction(ctx: Context<Unchecked>) -> Result<()> {
        if ctx.accounts.admin_config.owner != ctx.program_id {
            return Err(ProgramError::IncorrectProgramId.into());
        }

        let account_data = ctx.accounts.admin_config.try_borrow_data()?;
        let mut account_data_slice: &[u8] = &account_data;
        let account_state = AdminConfig::try_deserialize(&mut account_data_slice)?;

        if account_state.admin != ctx.accounts.admin.key() {
            return Err(ProgramError::InvalidArgument.into());
        }
        msg!("Admin: {}", account_state.admin.to_string());
        Ok(())
    }
}

#[derive(Accounts)]
pub struct Unchecked<'info> {
    admin_config: AccountInfo<'info>,
    admin: Signer<'info>,
}

#[account]
pub struct AdminConfig {
    admin: Pubkey,
}
```

### 使用 Anchor 的 `Account<'info, T>`

Anchor 可以通过 `Account` 类型使这变得更简单。

`Account<'info, T>` 是围绕 `AccountInfo` 的一个包装器，它验证了程序的所有权并将底层数据反序列化为指定的账户类型 `T`。这样，你就可以使用 `Account<'info, T>` 轻松验证所有权。

为了更好理解，`#[account]` 属性实现了用于表示账户的数据结构的各种特性。其中之一是 `Owner` 特征，它定义了一个预期拥有账户的地址。所有者设置为在 `declare_id!` 宏中指定的程序 ID。

在下面的示例中，`Account<'info, AdminConfig>` 用于验证 `admin_config`。这将自动执行所有者检查并反序列化账户数据。此外，使用 `has_one` 约束来检查 `admin` 账户是否与存储在 `admin_config` 账户上的 `admin` 字段匹配。

这样，你就不需要在指令逻辑中添加所有者检查，使代码更清晰简洁。

```rust
use anchor_lang::prelude::*;

declare_id!("Cft4eTTrt4sJU4Ar35rUQHx6PSXfJju3dixmvApzhWws");

#[program]
pub mod owner_check {
    use super::*;
	...
    pub fn admin_instruction(ctx: Context<Checked>) -> Result<()> {
        msg!("Admin: {}", ctx.accounts.admin_config.admin.to_string());
        Ok(())
    }
}

#[derive(Accounts)]
pub struct Checked<'info> {
    #[account(
        has_one = admin,
    )]
    admin_config: Account<'info, AdminConfig>,
    admin: Signer<'info>,
}

#[account]
pub struct AdminConfig {
    admin: Pubkey,
}
```

### 使用 Anchor 的 `#[account(owner = <expr>)]` 约束

除了 `Account` 类型之外，你还可以使用一个 `owner` 约束。`owner` 约束允许你定义对账户拥有所有权的程序，如果它与当前执行的程序不同的话。这在你编写一个指令，该指令期望一个账户是从不同程序派生的 PDA 时非常方便。你可以使用 `seeds` 和 `bump` 约束，并定义 `owner` 来正确派生和验证传入的账户的地址。

要使用 `owner` 约束，你必须可以访问你期望拥有账户的程序的公钥。你可以将该程序作为额外账户传入，也可以将公钥硬编码到程序的某个地方。

```rust
use anchor_lang::prelude::*;

declare_id!("Cft4eTTrt4sJU4Ar35rUQHx6PSXfJju3dixmvApzhWws");

#[program]
pub mod owner_check {
    use super::*;
    ...
    pub fn admin_instruction(ctx: Context<Checked>) -> Result<()> {
        msg!("Admin: {}", ctx.accounts.admin_config.admin.to_string());
        Ok(())
    }
}

#[derive(Accounts)]
pub struct Checked<'info> {
    #[account(
        has_one = admin,
    )]
    admin_config: Account<'info, AdminConfig>,
    admin: Signer<'info>,
    #[account(
            seeds = b"test-seed",
            bump,
            owner = token_program.key()
    )]
    pda_derived_from_another_program: AccountInfo<'info>,
    token_program: Program<'info, Token>
}

#[account]
pub struct AdminConfig {
    admin: Pubkey,
}
```

# 实验

在这个实验室中，我们将使用两个程序来演示缺少所有者检查如何允许一个伪造账户从一个简化的代币“保险库”账户中提取代币（请注意，这与签名者授权课程中的实验室非常相似）。

为了帮助说明这一点，一个程序将在提取代币到保险库账户时缺少账户所有者检查。

第二个程序将是第一个程序的直接克隆，由一个恶意用户创建，以创建一个与第一个程序的保险库账户相同的账户。

没有所有者检查，这个恶意用户将能够传入由他们“伪造”的程序拥有的保险库账户，而原始程序仍然会执行。

## 1. 起步

要开始实验，请从[该存储库的`starter`分支](https://github.com/Unboxed-Software/solana-owner-checks/tree/starter)下载起始代码。起始代码包括两个程序 `clone` 和 `owner_check`，以及测试文件的样板设置。

`owner_check` 程序包括两个指令：

- `initialize_vault` 初始化一个简化的保险库账户，该账户存储了代币账户和授权账户的地址
- `insecure_withdraw` 从代币账户提取代币，但是缺少对保险库账户的所有者检查

Let me know if you need further assistance!

```rust
use anchor_lang::prelude::*;
use anchor_spl::token::{self, Mint, Token, TokenAccount};

declare_id!("HQYNznB3XTqxzuEqqKMAD9XkYE5BGrnv8xmkoDNcqHYB");

#[program]
pub mod owner_check {
    use super::*;

    pub fn initialize_vault(ctx: Context<InitializeVault>) -> Result<()> {
        ctx.accounts.vault.token_account = ctx.accounts.token_account.key();
        ctx.accounts.vault.authority = ctx.accounts.authority.key();
        Ok(())
    }

    pub fn insecure_withdraw(ctx: Context<InsecureWithdraw>) -> Result<()> {
        let account_data = ctx.accounts.vault.try_borrow_data()?;
        let mut account_data_slice: &[u8] = &account_data;
        let account_state = Vault::try_deserialize(&mut account_data_slice)?;

        if account_state.authority != ctx.accounts.authority.key() {
            return Err(ProgramError::InvalidArgument.into());
        }

        let amount = ctx.accounts.token_account.amount;

        let seeds = &[
            b"token".as_ref(),
            &[*ctx.bumps.get("token_account").unwrap()],
        ];
        let signer = [&seeds[..]];

        let cpi_ctx = CpiContext::new_with_signer(
            ctx.accounts.token_program.to_account_info(),
            token::Transfer {
                from: ctx.accounts.token_account.to_account_info(),
                authority: ctx.accounts.token_account.to_account_info(),
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
    )]
    pub vault: Account<'info, Vault>,
    #[account(
        init,
        payer = authority,
        token::mint = mint,
        token::authority = token_account,
        seeds = [b"token"],
        bump,
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
    /// CHECK:
    pub vault: UncheckedAccount<'info>,
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
}
```

`clone` 程序包括一个指令：

- `initialize_vault` 初始化一个“保险库”账户，模仿了 `owner_check` 程序的保险库账户。它存储了真实保险库的代币账户地址，但允许恶意用户放入他们自己的授权账户。

```rust
use anchor_lang::prelude::*;
use anchor_spl::token::TokenAccount;

declare_id!("DUN7nniuatsMC7ReCh5eJRQExnutppN1tAfjfXFmGDq3");

#[program]
pub mod clone {
    use super::*;

    pub fn initialize_vault(ctx: Context<InitializeVault>) -> Result<()> {
        ctx.accounts.vault.token_account = ctx.accounts.token_account.key();
        ctx.accounts.vault.authority = ctx.accounts.authority.key();
        Ok(())
    }
}

#[derive(Accounts)]
pub struct InitializeVault<'info> {
    #[account(
        init,
        payer = authority,
        space = 8 + 32 + 32,
    )]
    pub vault: Account<'info, Vault>,
    pub token_account: Account<'info, TokenAccount>,
    #[account(mut)]
    pub authority: Signer<'info>,
    pub system_program: Program<'info, System>,
}

#[account]
pub struct Vault {
    token_account: Pubkey,
    authority: Pubkey,
}
```

## 2. 测试 `insecure_withdraw` 指令

测试文件包括一个测试，使用提供者钱包作为 `authority` 调用 `owner_check` 程序的 `initialize_vault` 指令，然后将 100 个代币铸造到代币账户。

测试文件还包括一个测试，调用 `clone` 程序的 `initialize_vault` 指令，初始化一个假的 `vault` 账户，存储相同的 `tokenPDA` 账户，但是一个不同的 `authority`。请注意，这里不会铸造新的代币。

我们将添加一个测试来调用 `insecure_withdraw` 指令。这个测试应该传入克隆的保险库和虚假的授权。由于没有所有者检查来验证 `vaultClone` 账户是否由 `owner_check` 程序拥有，指令的数据验证检查将通过，并显示 `walletFake` 为有效的授权。然后，代币将从 `tokenPDA` 账户提取到 `withdrawDestinationFake` 账户。

```tsx
describe("owner-check", () => {
	...
    it("Insecure withdraw", async () => {
    const tx = await program.methods
        .insecureWithdraw()
        .accounts({
            vault: vaultClone.publicKey,
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

运行 `anchor test`，查看 `insecure_withdraw` 是否成功完成。

```bash
owner-check
  ✔ Initialize Vault (808ms)
  ✔ Initialize Fake Vault (404ms)
  ✔ Insecure withdraw (409ms)
```

请注意，即使 Anchor 在自动初始化新账户时使用唯一的 8 字节区分器，并在反序列化账户时检查该区分器，`vaultClone` 仍然成功反序列化。这是因为区分器是账户类型名称的哈希。

```rust
#[account]
pub struct Vault {
    token_account: Pubkey,
    authority: Pubkey,
}
```

由于两个程序初始化了相同的账户，并且两个结构体都命名为 `Vault`，即使它们由不同的程序拥有，这些账户也具有相同的区分器。

## 3. 添加 `secure_withdraw` 指令

让我们关闭这个安全漏洞。

在 `owner_check` 程序的 `lib.rs` 文件中，添加一个 `secure_withdraw` 指令和一个 `SecureWithdraw` 账户结构体。

在 `SecureWithdraw` 结构体中，让我们使用 `Account<'info, Vault>` 来确保对 `vault` 账户执行所有者检查。我们还将使用 `has_one` 约束来检查传递到指令的 `token_account` 和 `authority` 是否与存储在 `vault` 账户上的值匹配。

```rust
#[program]
pub mod owner_check {
    use super::*;
	...

	pub fn secure_withdraw(ctx: Context<SecureWithdraw>) -> Result<()> {
        let amount = ctx.accounts.token_account.amount;

        let seeds = &[
            b"token".as_ref(),
            &[*ctx.bumps.get("token_account").unwrap()],
        ];
        let signer = [&seeds[..]];

        let cpi_ctx = CpiContext::new_with_signer(
            ctx.accounts.token_program.to_account_info(),
            token::Transfer {
                from: ctx.accounts.token_account.to_account_info(),
                authority: ctx.accounts.token_account.to_account_info(),
                to: ctx.accounts.withdraw_destination.to_account_info(),
            },
            &signer,
        );

        token::transfer(cpi_ctx, amount)?;
        Ok(())
    }
}
...

#[derive(Accounts)]
pub struct SecureWithdraw<'info> {
    #[account(
       has_one = token_account,
       has_one = authority
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

为了测试 `secure_withdraw` 指令，我们将两次调用该指令。首先，我们将使用 `vaultClone` 账户调用该指令，我们期望失败。然后，我们将使用正确的 `vault` 账户调用该指令，以检查该指令是否按预期工作。

```tsx
describe("owner-check", () => {
	...
	it("Secure withdraw, expect error", async () => {
        try {
            const tx = await program.methods
                .secureWithdraw()
                .accounts({
                    vault: vaultClone.publicKey,
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
            vault: vault.publicKey,
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

运行 `anchor test`，查看使用 `vaultClone` 账户的交易是否现在返回 Anchor 错误，而使用 `vault` 账户的交易是否成功完成。

```bash
'Program HQYNznB3XTqxzuEqqKMAD9XkYE5BGrnv8xmkoDNcqHYB invoke [1]',
'Program log: Instruction: SecureWithdraw',
'Program log: AnchorError caused by account: vault. Error Code: AccountOwnedByWrongProgram. Error Number: 3007. Error Message: The given account is owned by a different program than expected.',
'Program log: Left:',
'Program log: DUN7nniuatsMC7ReCh5eJRQExnutppN1tAfjfXFmGDq3',
'Program log: Right:',
'Program log: HQYNznB3XTqxzuEqqKMAD9XkYE5BGrnv8xmkoDNcqHYB',
'Program HQYNznB3XTqxzuEqqKMAD9XkYE5BGrnv8xmkoDNcqHYB consumed 5554 of 200000 compute units',
'Program HQYNznB3XTqxzuEqqKMAD9XkYE5BGrnv8xmkoDNcqHYB failed: custom program error: 0xbbf'
```

这里我们看到了如何使用 Anchor 的 `Account<'info, T>` 类型可以简化账户验证过程，自动进行所有者检查。另外，请注意，Anchor 错误可以指定导致错误的账户（例如，上面日志的第三行显示 `AnchorError caused by account: vault`）。这在调试时非常有用。

```bash
✔ Secure withdraw, expect error (78ms)
✔ Secure withdraw (10063ms)
```

这就是你确保在账户上检查所有者所需的全部内容！就像其他一些漏洞一样，这是相当简单的事情可以避免，但非常重要。确保始终考虑哪些账户应该由哪些程序拥有，并确保添加适当的验证。

如果你想查看最终的解决方案代码，你可以在[该存储库的`solution`分支](https://github.com/Unboxed-Software/solana-owner-checks/tree/solution)上找到它。

# 挑战

就像本单元中的其他课程一样，你有机会练习避免这种安全漏洞，方法是审计你自己或其他程序。

花一些时间审查至少一个程序，并确保在传递给每个指令的账户上执行了适当的所有者检查。

请记住，如果你发现别人程序中的漏洞或利用漏洞，请立即通知他们！如果你在自己的程序中发现了漏洞，务必立即修补它。

## 完成实验了吗？

将你的代码推送到 GitHub，并[告诉我们你对本课程的想法](https://form.typeform.com/to/IPH0UGz7#answers-lesson=e3069010-3038-4984-b9d3-2dc6585147b1)！