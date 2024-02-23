---
title: 重新初始化攻击
objectives:
- 解释与重新初始化漏洞相关的安全风险
- 使用长格式 Rust 检查账户是否已经被初始化
- 使用 Anchor 的 `init` 约束来初始化账户，该约束会自动设置一个账户鉴别器，以防止账户的重新初始化
---

# 总结

- 使用账户鉴别器或初始化标志来检查账户是否已经被初始化，以防止账户重新初始化并覆盖现有的账户数据。
- 为了 Rust 中防止账户重新初始化，可以使用一个 `is_initialized` 标志来初始化账户，并在初始化账户时检查该标志是否已经被设置为 true。

  ```rust
  if account.is_initialized {
      return Err(ProgramError::AccountAlreadyInitialized.into());
  }
  ```

为了简化这个过程，可以使用 Anchor 的 `init` 约束通过 CPI 到系统程序创建一个账户，并设置其鉴别器。

# 概述

初始化指的是首次设置新账户的数据。在初始化新账户时，您应该实现一种检查账户是否已经被初始化的方式。如果没有适当的检查，现有账户可能会被重新初始化，并且现有数据会被覆盖。

请注意，初始化账户和创建账户是两个独立的指令。创建账户需要在系统程序上调用 `create_account` 指令，该指令指定了账户所需的空间、分配给账户的租金（以 lamports 计量）、以及账户的程序所有者。初始化是一个指令，用于设置新创建账户的数据。创建和初始化账户可以合并为单个交易。

## 缺少初始化检查

在下面的示例中，对 `user` 账户没有进行任何检查。`initialize` 指令将 `user` 账户的数据反序列化为 `User` 账户类型，设置 `authority` 字段，并将更新后的账户数据序列化到 `user` 账户中。

在 `user` 账户上没有进行检查的情况下，另一方可能会再次将相同的账户传递给 `initialize` 指令，从而覆盖已存储在账户数据上的现有 `authority`。

```rust
use anchor_lang::prelude::*;
use borsh::{BorshDeserialize, BorshSerialize};

declare_id!("Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS");

#[program]
pub mod initialization_insecure  {
    use super::*;

    pub fn initialize(ctx: Context<Initialize>) -> Result<()> {
        let mut user = User::try_from_slice(&ctx.accounts.user.data.borrow()).unwrap();
        user.authority = ctx.accounts.authority.key();
        user.serialize(&mut *ctx.accounts.user.data.borrow_mut())?;
        Ok(())
    }
}

#[derive(Accounts)]
pub struct Initialize<'info> {
		#[account(mut)]
    user: AccountInfo<'info>,
    #[account(mut)]
		authority: Signer<'info>,
}

#[derive(BorshSerialize, BorshDeserialize)]
pub struct User {
    authority: Pubkey,
}
```

## 添加 `is_initialized` 检查

修复这个问题的一种方法是向 `User` 账户类型添加一个额外的 `is_initialized` 字段，并将其用作标志，以检查账户是否已经被初始化。

```jsx
if user.is_initialized {
    return Err(ProgramError::AccountAlreadyInitialized.into());
}
```

通过在 `initialize` 指令中包含检查，只有当 `is_initialized` 字段尚未设置为 true 时，`user` 账户才会被初始化。如果 `is_initialized` 字段已经设置，交易将失败，从而避免了攻击者可以用自己的公钥替换账户授权的情况。

```rust
use anchor_lang::prelude::*;
use borsh::{BorshDeserialize, BorshSerialize};

declare_id!("Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS");

#[program]
pub mod initialization_secure {
    use super::*;

    pub fn initialize(ctx: Context<Initialize>) -> Result<()> {
        let mut user = User::try_from_slice(&ctx.accounts.user.data.borrow()).unwrap();
        if user.is_initialized {
            return Err(ProgramError::AccountAlreadyInitialized.into());
        }

        user.authority = ctx.accounts.authority.key();
        user.is_initialized = true;

        user.serialize(&mut *ctx.accounts.user.data.borrow_mut())?;
        Ok(())
    }
}

#[derive(Accounts)]
pub struct Initialize<'info> {
		#[account(mut)]
    user: AccountInfo<'info>,
    #[account(mut)]
		authority: Signer<'info>,
}

#[derive(BorshSerialize, BorshDeserialize)]
pub struct User {
    is_initialized: bool,
    authority: Pubkey,
}
```

## 使用 Anchor 的 `init` 约束

Anchor 提供了一个 `init` 约束，可以与 `#[account(...)]` 属性一起使用来初始化一个账户。`init` 约束通过 CPI 到系统程序创建账户，并设置账户鉴别器（account discriminator）。

`init` 约束必须与 `payer` 和 `space` 约束结合使用。`payer` 指定支付新账户初始化费用的账户。`space` 指定新账户所需的空间大小，这决定了必须分配给账户的 lamports 量。数据的前 8 个字节被设置为鉴别器，Anchor 自动添加以识别账户类型。

对于这个教程最重要的是，`init` 约束确保此指令每个账户只能调用一次，因此您可以在指令逻辑中设置账户的初始状态，而不必担心攻击者试图重新初始化账户。

```rust
use anchor_lang::prelude::*;

declare_id!("Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS");

#[program]
pub mod initialization_recommended {
    use super::*;

    pub fn initialize(_ctx: Context<Initialize>) -> Result<()> {
        msg!("GM");
        Ok(())
    }
}

#[derive(Accounts)]
pub struct Initialize<'info> {
    #[account(init, payer = authority, space = 8+32)]
    user: Account<'info, User>,
    #[account(mut)]
    authority: Signer<'info>,
    system_program: Program<'info, System>,
}

#[account]
pub struct User {
    authority: Pubkey,
}
```

## Anchor 的 `init_if_needed` 约束

值得注意的是，Anchor 还有一个 `init_if_needed` 约束。这个约束应该非常谨慎地使用。事实上，它被封锁在一个特性标志（feature flag）后面，这样您就必须有意识地使用它。

`init_if_needed` 约束和 `init` 约束做的事情是一样的，只是如果账户已经被初始化，该指令仍然会运行。

鉴于此，当您使用这个约束时，***非常重要*** 的是要包含确保以避免将账户重置为其初始状态。

例如，如果账户存储了一个 `authority` 字段，该字段在使用 `init_if_needed` 约束的指令中设置，您需要进行检查，以确保在账户已经被初始化后，没有攻击者可以调用该指令并再次设置 `authority` 字段。

在大多数情况下，最安全的做法是为初始化账户数据单独创建一个指令。

# 实验

在这个实验中，我们将创建一个简单的程序，只用来初始化账户。我们将包括两个指令：

- `insecure_initialization` - 初始化一个可以被重新初始化的账户
- `recommended_initialization` - 使用 Anchor 的 `init` 约束初始化一个账户

## 1. 起步

要开始，请从 [此存储库](https://github.com/Unboxed-Software/solana-reinitialization-attacks/tree/starter) 的 `starter` 分支下载起始代码。起始代码包括一个带有一个指令的程序以及测试文件的样板设置。

`insecure_initialization` 指令初始化一个新的 `user` 账户，该账户存储着一个 `authority` 的公钥。在这个指令中，账户预期在客户端被分配，然后传递到程序指令中。一旦传递到程序中，就没有检查来查看 `user` 账户的初始状态是否已经设置。这意味着同一个账户可以第二次被传递进来，从而覆盖已存在的 `user` 账户上存储的 `authority`。

```rust
use anchor_lang::prelude::*;
use borsh::{BorshDeserialize, BorshSerialize};

declare_id!("Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS");

#[program]
pub mod initialization {
    use super::*;

    pub fn insecure_initialization(ctx: Context<Unchecked>) -> Result<()> {
        let mut user = User::try_from_slice(&ctx.accounts.user.data.borrow()).unwrap();
        user.authority = ctx.accounts.authority.key();
        user.serialize(&mut *ctx.accounts.user.data.borrow_mut())?;
        Ok(())
    }
}

#[derive(Accounts)]
pub struct Unchecked<'info> {
    #[account(mut)]
    /// CHECK:
    user: UncheckedAccount<'info>,
    authority: Signer<'info>,
}

#[derive(BorshSerialize, BorshDeserialize)]
pub struct User {
    authority: Pubkey,
}
```

## 2. 测试 `insecure_initialization` 指令

测试文件包括设置，通过调用系统程序来创建一个账户，然后两次调用 `insecure_initialization` 指令使用同一个账户。

由于没有检查来验证账户数据是否已经初始化，`insecure_initialization` 指令会两次成功完成，尽管第二次调用提供了一个*不同*的授权账户。

```tsx
import * as anchor from "@coral-xyz/anchor"
import { Program } from "@coral-xyz/anchor"
import { expect } from "chai"
import { Initialization } from "../target/types/initialization"

describe("initialization", () => {
  const provider = anchor.AnchorProvider.env()
  anchor.setProvider(provider)

  const program = anchor.workspace.Initialization as Program<Initialization>

  const wallet = anchor.workspace.Initialization.provider.wallet
  const walletTwo = anchor.web3.Keypair.generate()

  const userInsecure = anchor.web3.Keypair.generate()
  const userRecommended = anchor.web3.Keypair.generate()

  before(async () => {
    const tx = new anchor.web3.Transaction().add(
      anchor.web3.SystemProgram.createAccount({
        fromPubkey: wallet.publicKey,
        newAccountPubkey: userInsecure.publicKey,
        space: 32,
        lamports: await provider.connection.getMinimumBalanceForRentExemption(
          32
        ),
        programId: program.programId,
      })
    )

    await anchor.web3.sendAndConfirmTransaction(provider.connection, tx, [
      wallet.payer,
      userInsecure,
    ])

    await provider.connection.confirmTransaction(
      await provider.connection.requestAirdrop(
        walletTwo.publicKey,
        1 * anchor.web3.LAMPORTS_PER_SOL
      ),
      "confirmed"
    )
  })

  it("Insecure init", async () => {
    await program.methods
      .insecureInitialization()
      .accounts({
        user: userInsecure.publicKey,
      })
      .rpc()
  })

  it("Re-invoke insecure init with different auth", async () => {
    const tx = await program.methods
      .insecureInitialization()
      .accounts({
        user: userInsecure.publicKey,
        authority: walletTwo.publicKey,
      })
      .transaction()
    await anchor.web3.sendAndConfirmTransaction(provider.connection, tx, [
      walletTwo,
    ])
  })
})
```

运行 `anchor test`，您会看到两个交易都会成功完成。

```bash
initialization
  ✔ Insecure init (478ms)
  ✔ Re-invoke insecure init with different auth (464ms)
```

## 3. 添加 `recommended_initialization` 指令

让我们创建一个新的指令，称为 `recommended_initialization`，来解决这个问题。与之前不安全的指令不同，这个指令应该使用 Anchor 的 `init` 约束来处理用户账户的创建和初始化。

该约束指示程序通过 CPI 到系统程序创建账户，因此不再需要在客户端创建账户。该约束还设置了账户的辨别器。然后，您的指令逻辑可以设置账户的初始状态。

通过这样做，您确保对同一个用户账户进行的任何后续调用都会失败，而不是重置账户的初始状态。

```rust
use anchor_lang::prelude::*;
use borsh::{BorshDeserialize, BorshSerialize};

declare_id!("Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS");

#[program]
pub mod initialization {
    use super::*;
		...
    pub fn recommended_initialization(ctx: Context<Checked>) -> Result<()> {
        ctx.accounts.user.authority = ctx.accounts.authority.key();
        Ok(())
    }
}

#[derive(Accounts)]
pub struct Checked<'info> {
    #[account(init, payer = authority, space = 8+32)]
    user: Account<'info, User>,
    #[account(mut)]
    authority: Signer<'info>,
    system_program: Program<'info, System>,
}
```

## 4. 测试 `recommended_initialization` 指令

为了测试 `recommended_initialization` 指令，我们将像之前一样两次调用该指令。但是这一次，当我们尝试第二次初始化相同的账户时，我们希望交易失败。

```tsx
describe("initialization", () => {
  ...
  it("Recommended init", async () => {
    await program.methods
      .recommendedInitialization()
      .accounts({
        user: userRecommended.publicKey,
      })
      .signers([userRecommended])
      .rpc()
  })

  it("Re-invoke recommended init with different auth, expect error", async () => {
    try {
      // Add your test here.
      const tx = await program.methods
        .recommendedInitialization()
        .accounts({
          user: userRecommended.publicKey,
          authority: walletTwo.publicKey,
        })
        .transaction()
      await anchor.web3.sendAndConfirmTransaction(provider.connection, tx, [
        walletTwo,
        userRecommended,
      ])
    } catch (err) {
      expect(err)
      console.log(err)
    }
  })
})
```

运行 `anchor test`，您会看到尝试两次初始化相同账户的第二个交易现在会返回一个错误，指示账户地址已经在使用中。

```bash
'Program CpozUgSwe9FPLy9BLNhY2LTGqLUk1nirUkMMA5RmDw6t invoke [1]',
'Program log: Instruction: RecommendedInitialization',
'Program 11111111111111111111111111111111 invoke [2]',
'Allocate: account Address { address: EMvbwzrs4VTR7G1sNUJuQtvRX1EuvLhqs4PFqrtDcCGV, base: None } already in use',
'Program 11111111111111111111111111111111 failed: custom program error: 0x0',
'Program CpozUgSwe9FPLy9BLNhY2LTGqLUk1nirUkMMA5RmDw6t consumed 4018 of 200000 compute units',
'Program CpozUgSwe9FPLy9BLNhY2LTGqLUk1nirUkMMA5RmDw6t failed: custom program error: 0x0'
```

如果您使用了 Anchor 的 `init` 约束，那通常就足以保护您免受重新初始化攻击！请记住，尽管修复这些安全漏洞的方法很简单，但这并不意味着它不重要。每次初始化账户时，请确保您要么使用 `init` 约束，要么有其他检查来避免重置现有账户的初始状态。

如果您想查看最终解决方案代码，可以在 [此存储库](https://github.com/Unboxed-Software/solana-reinitialization-attacks/tree/solution) 的 `solution` 分支找到。

# 挑战

与本单元的其他课程一样，您可以通过审核自己或其他程序来练习避免此安全漏洞。

花些时间审查至少一个程序，并确保指令已经得到适当的保护，以防止重新初始化攻击。

请记住，如果您发现别人的程序中存在漏洞或漏洞，请立即通知他们！如果您在自己的程序中发现了问题，请务必立即修复。

## 完成了实验吗？

将您的代码推送到 GitHub，并[告诉我们您对这节课的看法](https://form.typeform.com/to/IPH0UGz7#answers-lesson=652c68aa-18d9-464c-9522-e531fd8738d5)！