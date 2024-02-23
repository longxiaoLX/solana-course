---
title: 类型伪装
objectives:
- 解释不检查账户类型所带来的安全风险
- 使用 Rust 的长格式实现账户类型判别器
- 使用 Anchor 的 `init` 约束初始化账户
- 使用 Anchor 的 `Account` 类型进行账户验证
---

# 总结

- 使用鉴别器区分不同类型的账户
- 要在 Rust 中实现一个鉴别器，需要在账户结构体中包含一个字段来表示账户类型。

    ```rust
    #[derive(BorshSerialize, BorshDeserialize)]
    pub struct User {
        discriminant: AccountDiscriminant,
        user: Pubkey,
    }

    #[derive(BorshSerialize, BorshDeserialize, PartialEq)]
    pub enum AccountDiscriminant {
        User,
        Admin,
    }
    ```

- 要在 Rust 中实现鉴别器检查，需要验证反序列化的账户数据的鉴别器是否与预期值匹配。

    ```rust
    if user.discriminant != AccountDiscriminant::User {
        return Err(ProgramError::InvalidAccountData.into());
    }
    ```

- 在 Anchor 中，程序账户类型自动实现了 `Discriminator` 特征，该特征为类型创建了一个 8 字节的唯一标识符。
- 在反序列化账户数据时，使用 Anchor 的 `Account<'info, T>` 类型来自动检查账户的鉴别器。

# 概述

“类型伪装（Type cosplay）”指的是在期望的账户类型位置上使用了意外的账户类型。在幕后，账户数据简单地被存储为字节数组，程序将其反序列化为自定义的账户类型。如果没有实现一种明确区分账户类型的方式，来自意外账户的账户数据可能会导致指令被以意外的方式使用。

## 未经检查的账户（Unchecked account）

在下面的示例中，`AdminConfig` 和 `UserConfig` 账户类型都存储了一个公钥。`admin_instruction` 指令将 `admin_config` 账户反序列化为 `AdminConfig` 类型，然后执行所有者检查和数据验证检查。

然而，`AdminConfig` 和 `UserConfig` 账户类型具有相同的数据结构。这意味着一个 `UserConfig` 账户类型可以作为 `admin_config` 账户传入。只要账户数据中存储的公钥与签名交易的 `admin` 匹配，`admin_instruction` 指令就会继续处理，即使签名者实际上并不是管理员。

请注意，账户类型中存储的字段名称（`admin` 和 `user`）在反序列化账户数据时并不重要。数据是根据字段的顺序而不是名称进行序列化和反序列化的。

```rust
use anchor_lang::prelude::*;
use borsh::{BorshDeserialize, BorshSerialize};

declare_id!("Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS");

#[program]
pub mod type_cosplay_insecure {
    use super::*;

    pub fn admin_instruction(ctx: Context<AdminInstruction>) -> Result<()> {
        let account_data =
            AdminConfig::try_from_slice(&ctx.accounts.admin_config.data.borrow()).unwrap();
        if ctx.accounts.admin_config.owner != ctx.program_id {
            return Err(ProgramError::IllegalOwner.into());
        }
        if account_data.admin != ctx.accounts.admin.key() {
            return Err(ProgramError::InvalidAccountData.into());
        }
        msg!("Admin {}", account_data.admin);
        Ok(())
    }
}

#[derive(Accounts)]
pub struct AdminInstruction<'info> {
    admin_config: UncheckedAccount<'info>,
    admin: Signer<'info>,
}

#[derive(BorshSerialize, BorshDeserialize)]
pub struct AdminConfig {
    admin: Pubkey,
}

#[derive(BorshSerialize, BorshDeserialize)]
pub struct UserConfig {
    user: Pubkey,
}
```

## 添加账户鉴别器

为了解决这个问题，可以为每个账户类型添加一个鉴别器字段，并在初始化账户时设置鉴别器。

下面的示例更新了 `AdminConfig` 和 `UserConfig` 账户类型，添加了一个 `discriminant` 字段。`admin_instruction` 指令还包括了对 `discriminant` 字段的额外数据验证检查。

```rust
if account_data.discriminant != AccountDiscriminant::Admin {
    return Err(ProgramError::InvalidAccountData.into());
}
```

如果作为 `admin_config` 账户传入指令的账户的 `discriminant` 字段与预期的 `AccountDiscriminant` 不匹配，则交易将失败。只需确保在初始化每个账户时设置适当的 `discriminant` 值（在示例中未显示），然后你可以在每个后续指令中包含这些鉴别器检查。

```rust
use anchor_lang::prelude::*;
use borsh::{BorshDeserialize, BorshSerialize};

declare_id!("Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS");

#[program]
pub mod type_cosplay_secure {
    use super::*;

    pub fn admin_instruction(ctx: Context<AdminInstruction>) -> Result<()> {
        let account_data =
            AdminConfig::try_from_slice(&ctx.accounts.admin_config.data.borrow()).unwrap();
        if ctx.accounts.admin_config.owner != ctx.program_id {
            return Err(ProgramError::IllegalOwner.into());
        }
        if account_data.admin != ctx.accounts.admin.key() {
            return Err(ProgramError::InvalidAccountData.into());
        }
        if account_data.discriminant != AccountDiscriminant::Admin {
            return Err(ProgramError::InvalidAccountData.into());
        }
        msg!("Admin {}", account_data.admin);
        Ok(())
    }
}

#[derive(Accounts)]
pub struct AdminInstruction<'info> {
    admin_config: UncheckedAccount<'info>,
    admin: Signer<'info>,
}

#[derive(BorshSerialize, BorshDeserialize)]
pub struct AdminConfig {
    discriminant: AccountDiscriminant,
    admin: Pubkey,
}

#[derive(BorshSerialize, BorshDeserialize)]
pub struct UserConfig {
    discriminant: AccountDiscriminant,
    user: Pubkey,
}

#[derive(BorshSerialize, BorshDeserialize, PartialEq)]
pub enum AccountDiscriminant {
    Admin,
    User,
}
```

## 使用 Anchor 的 `Account` 包装器

为每个指令实现这些检查可能很繁琐。幸运的是，Anchor 提供了一个 `#[account]` 属性宏，用于自动实现每个账户应该具有的特征。

标记为 `#[account]` 的结构体可以与 `Account` 一起使用，以验证传入的账户确实是你期望的类型。当初始化一个具有 `#[account]` 属性的结构体表示的账户时，前 8 个字节将自动保留用于该账户类型的唯一鉴别器。在反序列化账户数据时，Anchor 将自动检查账户上的鉴别器是否与预期的账户类型匹配，并在不匹配时抛出错误。

在下面的示例中，`Account<'info, AdminConfig>` 指定了 `admin_config` 账户应该是 `AdminConfig` 类型。然后，Anchor 自动检查账户数据的前 8 个字节是否与 `AdminConfig` 类型的鉴别器匹配。

对于 `admin` 字段的数据验证检查也从指令逻辑移到了账户验证结构中，使用了 `has_one` 约束。`#[account(has_one = admin)]` 指定了 `admin_config` 账户的 `admin` 字段必须与传入指令的 `admin` 账户匹配。请注意，为了使 `has_one` 约束起作用，结构体中账户的命名必须与正在验证的账户上的字段命名相匹配。

```rust
use anchor_lang::prelude::*;
use borsh::{BorshDeserialize, BorshSerialize};

declare_id!("Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS");

#[program]
pub mod type_cosplay_recommended {
    use super::*;

    pub fn admin_instruction(ctx: Context<AdminInstruction>) -> Result<()> {
        msg!("Admin {}", ctx.accounts.admin_config.admin);
        Ok(())
    }
}

#[derive(Accounts)]
pub struct AdminInstruction<'info> {
    #[account(has_one = admin)]
    admin_config: Account<'info, AdminConfig>,
    admin: Signer<'info>,
}

#[account]
pub struct AdminConfig {
    admin: Pubkey,
}

#[account]
pub struct UserConfig {
    user: Pubkey,
}
```

重要的是要注意，当使用 Anchor 时，这其实不是一个你真正需要担心的漏洞 - 这正是其存在的根本目的！在介绍了如果在本地 Rust 程序中没有正确处理可能会被利用的情况后，希望你对 Anchor 账户中账户鉴别器的目的有了更好的理解。Anchor 自动设置和检查这个鉴别器的事实意味着开发者可以花更多时间专注于他们的产品，但仍然非常重要的是要理解 Anchor 在幕后做了什么来开发健壮的 Solana 程序。

# 实验

在这个实验中，我们将创建两个程序来演示类型伪装漏洞。

- 第一个程序将初始化没有鉴别器的程序账户。
- 第二个程序将使用 Anchor 的 `init` 约束来初始化程序账户，该约束会自动设置账户鉴别器。

## 1. 起步

开始之前，请从[此存储库的 `starter` 分支](https://github.com/Unboxed-Software/solana-type-cosplay/tree/starter)下载起始代码。起始代码包括一个具有三个指令和一些测试的程序。

这三个指令是：

1. `initialize_admin` - 初始化管理员账户并设置程序的管理员权限
2. `initialize_user` - 初始化标准用户账户
3. `update_admin` - 允许现有的管理员更新程序的管理员权限

查看 `lib.rs` 文件中的这三个指令。最后一个指令只应由与使用 `initialize_admin` 指令初始化的管理员账户上的 `admin` 字段匹配的账户调用。

## 2. 测试不安全的 `update_admin` 指令

然而，两个账户具有相同的字段和字段类型：

```rust
#[derive(BorshSerialize, BorshDeserialize)]
pub struct AdminConfig {
    admin: Pubkey,
}

#[derive(BorshSerialize, BorshDeserialize)]
pub struct User {
    user: Pubkey,
}
```

因此，有可能在 `update_admin` 指令中将一个 `User` 账户替换成 `admin` 账户，从而绕过需要是管理员才能调用此指令的要求。

查看 `tests` 目录中的 `solana-type-cosplay.ts` 文件。它包含一些基本设置和两个测试。一个测试初始化一个用户账户，另一个调用 `update_admin` 并将用户账户替换成管理员账户。

运行 `anchor test`，查看调用 `update_admin` 是否会成功完成。

```bash
  type-cosplay
    ✔ Initialize User Account (233ms)
    ✔ Invoke update admin instruction with user account (487ms)
```

## 3. 创建 `type-checked` 程序

现在，我们将通过在现有 Anchor 程序的根目录中运行 `anchor new type-checked` 来创建一个名为 `type-checked` 的新程序。

现在在你的 `programs` 文件夹中将会有两个程序。运行 `anchor keys list`，你应该会看到新程序的程序 ID。将其添加到 `type-checked` 程序的 `lib.rs` 文件和 `Anchor.toml` 文件中的 `type_checked` 程序中。

接下来，更新测试文件的设置，包括新程序和我们将为新程序初始化的两个新密钥对。

```tsx
import * as anchor from "@coral-xyz/anchor"
import { Program } from "@coral-xyz/anchor"
import { TypeCosplay } from "../target/types/type_cosplay"
import { TypeChecked } from "../target/types/type_checked"
import { expect } from "chai"

describe("type-cosplay", () => {
  const provider = anchor.AnchorProvider.env()
  anchor.setProvider(provider)

  const program = anchor.workspace.TypeCosplay as Program<TypeCosplay>
  const programChecked = anchor.workspace.TypeChecked as Program<TypeChecked>

  const userAccount = anchor.web3.Keypair.generate()
  const newAdmin = anchor.web3.Keypair.generate()

  const userAccountChecked = anchor.web3.Keypair.generate()
  const adminAccountChecked = anchor.web3.Keypair.generate()
})
```

## 4. 实现 `type-checked` 程序

在 `type_checked` 程序中，使用 `init` 约束添加两个指令，以初始化一个 `AdminConfig` 账户和一个 `User` 账户。当使用 `init` 约束初始化新的程序账户时，Anchor 将自动将账户数据的前 8 个字节设置为该账户类型的唯一鉴别器。

我们还将添加一个 `update_admin` 指令，该指令使用 Anchor 的 `Account` 包装器验证 `admin_config` 账户作为 `AdminConfig` 账户类型。对于任何作为 `admin_config` 账户传入的账户，Anchor 将自动检查账户鉴别器是否与预期的账户类型匹配。

```rust
use anchor_lang::prelude::*;

declare_id!("FZLRa6vX64QL6Vj2JkqY1Uzyzjgi2PYjCABcDabMo8U7");

#[program]
pub mod type_checked {
    use super::*;

    pub fn initialize_admin(ctx: Context<InitializeAdmin>) -> Result<()> {
        ctx.accounts.admin_config.admin = ctx.accounts.admin.key();
        Ok(())
    }

    pub fn initialize_user(ctx: Context<InitializeUser>) -> Result<()> {
        ctx.accounts.user_account.user = ctx.accounts.user.key();
        Ok(())
    }

    pub fn update_admin(ctx: Context<UpdateAdmin>) -> Result<()> {
        ctx.accounts.admin_config.admin = ctx.accounts.admin.key();
        Ok(())
    }
}

#[derive(Accounts)]
pub struct InitializeAdmin<'info> {
    #[account(
        init,
        payer = admin,
        space = 8 + 32
    )]
    pub admin_config: Account<'info, AdminConfig>,
    #[account(mut)]
    pub admin: Signer<'info>,
    pub system_program: Program<'info, System>,
}

#[derive(Accounts)]
pub struct InitializeUser<'info> {
    #[account(
        init,
        payer = user,
        space = 8 + 32
    )]
    pub user_account: Account<'info, User>,
    #[account(mut)]
    pub user: Signer<'info>,
    pub system_program: Program<'info, System>,
}

#[derive(Accounts)]
pub struct UpdateAdmin<'info> {
    #[account(
        mut,
        has_one = admin
    )]
    pub admin_config: Account<'info, AdminConfig>,
    pub new_admin: SystemAccount<'info>,
    #[account(mut)]
    pub admin: Signer<'info>,
}

#[account]
pub struct AdminConfig {
    admin: Pubkey,
}

#[account]
pub struct User {
    user: Pubkey,
}
```

## 5. 测试安全的 `update_admin` 指令

在测试文件中，我们将从 `type_checked` 程序初始化一个 `AdminConfig` 账户和一个 `User` 账户。然后我们将调用 `updateAdmin` 指令两次，并传入新创建的账户。

```rust
describe("type-cosplay", () => {
	...

  it("Initialize type checked AdminConfig Account", async () => {
    await programChecked.methods
      .initializeAdmin()
      .accounts({
        adminConfig: adminAccountType.publicKey,
      })
      .signers([adminAccountType])
      .rpc()
  })

  it("Initialize type checked User Account", async () => {
    await programChecked.methods
      .initializeUser()
      .accounts({
        userAccount: userAccountType.publicKey,
        user: provider.wallet.publicKey,
      })
      .signers([userAccountType])
      .rpc()
  })

  it("Invoke update instruction using User Account", async () => {
    try {
      await programChecked.methods
        .updateAdmin()
        .accounts({
          adminConfig: userAccountType.publicKey,
          newAdmin: newAdmin.publicKey,
          admin: provider.wallet.publicKey,
        })
        .rpc()
    } catch (err) {
      expect(err)
      console.log(err)
    }
  })

  it("Invoke update instruction using AdminConfig Account", async () => {
    await programChecked.methods
      .updateAdmin()
      .accounts({
        adminConfig: adminAccountType.publicKey,
        newAdmin: newAdmin.publicKey,
        admin: provider.wallet.publicKey,
      })
      .rpc()
  })
})
```

运行 `anchor test`。对于传入 `User` 账户类型的交易，我们期望该指令返回一个 Anchor 错误，提示账户不是 `AdminConfig` 类型。

```bash
'Program EU66XDppFCf2Bg7QQr59nyykj9ejWaoW93TSkk1ufXh3 invoke [1]',
'Program log: Instruction: UpdateAdmin',
'Program log: AnchorError caused by account: admin_config. Error Code: AccountDiscriminatorMismatch. Error Number: 3002. Error Message: 8 byte discriminator did not match what was expected.',
'Program EU66XDppFCf2Bg7QQr59nyykj9ejWaoW93TSkk1ufXh3 consumed 4765 of 200000 compute units',
'Program EU66XDppFCf2Bg7QQr59nyykj9ejWaoW93TSkk1ufXh3 failed: custom program error: 0xbba'
```

遵循 Anchor 的最佳实践并使用 Anchor 类型将确保你的程序避免此漏洞。在创建账户结构体时始终使用 `#[account]` 属性，初始化账户时使用 `init` 约束，并在账户验证结构中使用 `Account` 类型。

如果你想查看最终解决方案代码，可以在[存储库的 `solution` 分支](https://github.com/Unboxed-Software/solana-type-cosplay/tree/solution)中找到。

# 挑战

与本单元中的其他课程一样，避免这种安全漏洞的机会在于审计你自己或其他程序。

花些时间审查至少一个程序，并确保账户类型具有鉴别器，并且对每个账户和指令进行了检查。由于标准 Anchor 类型会自动处理此检查，因此你更有可能在本地程序中找到漏洞。

请记住，如果你发现了他人程序中的错误或漏洞，请提醒他们！如果你发现了自己程序中的错误或漏洞，请务必立即修补。

## 完成了实验吗？

将你的代码推送到 GitHub，并[告诉我们你对本课程的想法](https://form.typeform.com/to/IPH0UGz7#answers-lesson=37ebccab-b19a-43c6-a96a-29fa7e80fdec)!