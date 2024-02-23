---
title: Anchor 开发入门
objectives:
- 使用 Anchor 框架构建一个基本程序
- 描述 Anchor 程序的基本结构
- 解释如何使用 Anchor 实现基本的账户验证和安全检查
---

# 总结

- **Anchor** 是构建 Solana 程序的框架。
- **Anchor 宏**通过抽象掉大量样板代码，加速了构建 Solana 程序的过程。
- Anchor 允许你更轻松地构建**安全的程序**，通过执行某些安全检查，要求账户验证，并提供一种简单的方法来实现额外的检查。

# 概述

## 什么是 Anchor？

Anchor 是一个开发框架，使编写 Solana 程序更加简单、快速和安全。它是 Solana 开发的“首选”框架，有很好的理由。它使得更容易组织和理解你的代码，自动实现常见的安全检查，并抽象掉与编写 Solana 程序相关的大量样板代码。

## Anchor 程序结构

Anchor 使用宏和特征为你生成样板 Rust 代码。这些为你的程序提供了清晰的结构，使你更容易理解你的代码。主要的高级宏和属性包括：

- `declare_id` - 用于声明程序的链上地址的宏
- `#[program]` - 用于表示包含程序指令逻辑的模块的属性宏
- `Accounts` - 应用于表示指令所需账户列表的结构体的特性
- `#[account]` - 用于为程序定义自定义账户类型的属性宏

让我们在将所有部分组合在一起之前讨论一下每一个。

## 声明你的程序 ID

`declare_id` 宏用于指定程序的链上地址（即 `programId`）。当你第一次构建一个 Anchor 程序时，框架将生成一个新的密钥对。这成为默认用于部署程序的密钥对，除非另有说明。相应的公钥应该作为 `declare_id!` 宏中指定的 `programId`。

```rust
declare_id!("Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS");
```

## 定义指令逻辑

`#[program]` 属性宏定义了包含程序所有指令的模块。这是你为程序中每个指令实现业务逻辑的地方。

模块中带有 `#[program]` 属性的每个公共函数将被视为单独的指令。

每个指令函数都需要一个 `Context` 类型的参数，并可以选择包括表示指令数据的额外函数参数。Anchor 将自动处理指令数据的反序列化，以便你可以将指令数据作为 Rust 类型处理。

```rust
#[program]
mod program_module_name {
    use super::*;

    pub fn instruction_one(ctx: Context<InstructionAccounts>, instruction_data: u64) -> Result<()> {
		ctx.accounts.account_name.data = instruction_data;
        Ok(())
    }
}
```

### 指令 `Context`

`Context` 类型向你的指令逻辑暴露了指令的元数据和账户。

```rust
pub struct Context<'a, 'b, 'c, 'info, T> {
    /// Currently executing program id.
    pub program_id: &'a Pubkey,
    /// Deserialized accounts.
    pub accounts: &'b mut T,
    /// Remaining accounts given but not deserialized or validated.
    /// Be very careful when using this directly.
    pub remaining_accounts: &'c [AccountInfo<'info>],
    /// Bump seeds found during constraint validation. This is provided as a
    /// convenience so that handlers don't have to recalculate bump seeds or
    /// pass them in as arguments.
    pub bumps: BTreeMap<String, u8>,
}
```

`Context` 是一个泛型类型，其中 `T` 定义了指令需要的账户列表。当你使用 `Context` 时，你将 `T` 的具体类型指定为一个采用了 `Accounts` 特征的结构体（例如 `Context<AddMovieReviewAccounts>`）。通过这个上下文参数，指令可以访问以下内容：

- 执行程序的程序 ID（`ctx.program_id`）
- 传递给指令的账户（`ctx.accounts`）
- 剩余账户（`ctx.remaining_accounts`）。`remaining_accounts` 是一个向量，包含传递给指令但未在 `Accounts` 结构中声明的所有账户。
- `Accounts` 结构中任何 PDA 账户的 bump（`ctx.bumps`）

## 定义指令账户

`Accounts` 特征定义了一个经过验证的账户数据结构。采用这个特征的结构体定义了给定指令所需的账户列表。然后通过指令的 `Context` 将这些账户暴露出来，这样就不再需要手动迭代和反序列化账户了。

通常，你通过 `derive` 宏（例如 `#[derive(Accounts)]`）应用 `Accounts` 特征。这将在给定的结构体上实现一个 `Accounts` 反序列化器，并消除了手动反序列化每个账户的需要。

`Accounts` 特征的实现负责执行所有必要的约束检查，以确保账户满足程序安全运行所需的条件。使用 `#account(..)` 属性为每个字段提供约束条件（稍后会详细介绍）。

例如，`instruction_one` 需要一个类型为 `InstructionAccounts` 的 `Context` 参数。使用 `#[derive(Accounts)]` 宏来实现 `InstructionAccounts` 结构体，其中包括三个账户：`account_name`、`user` 和 `system_program`。

```rust
#[program]
mod program_module_name {
    use super::*;
    pub fn instruction_one(ctx: Context<InstructionAccounts>, instruction_data: u64) -> Result<()> {
		...
        Ok(())
    }
}

#[derive(Accounts)]
pub struct InstructionAccounts {
    #[account(init, payer = user, space = 8 + 8)]
    pub account_name: Account<'info, AccountStruct>,
    #[account(mut)]
    pub user: Signer<'info>,
    pub system_program: Program<'info, System>,

}
```

当调用 `instruction_one` 时，程序会执行以下操作：

- 检查传递给指令的账户是否与 `InstructionAccounts` 结构中指定的账户类型匹配。
- 根据指定的任何额外约束条件检查账户。

如果传递给 `instruction_one` 的任何账户未能通过 `InstructionAccounts` 结构中指定的账户验证或安全检查，则该指令在达到程序逻辑之前就会失败。

## 账户验证

你可能已经注意到在前面的示例中，`InstructionAccounts` 中的一个账户类型是 `Account`，一个是 `Signer`，另一个是 `Program`。

Anchor 提供了许多账户类型，可以用来表示账户。每种类型都实现了不同的账户验证。我们将介绍一些你可能会遇到的常见类型，但请确保浏览 [账户类型的完整列表](https://docs.rs/anchor-lang/latest/anchor_lang/accounts/index.html)。

### `Account`

`Account` 是围绕着 `AccountInfo` 的一个包装器（wrapper），它验证程序所有权，即验证这个账户是否自己定义的合约，并将底层数据反序列化为 Rust 类型。

```rust
// Deserializes this info
pub struct AccountInfo<'a> {
    pub key: &'a Pubkey,
    pub is_signer: bool,
    pub is_writable: bool,
    pub lamports: Rc<RefCell<&'a mut u64>>,
    pub data: Rc<RefCell<&'a mut [u8]>>,    // <---- deserializes account data
    pub owner: &'a Pubkey,    // <---- checks owner program
    pub executable: bool,
    pub rent_epoch: u64,
}
```

回想一下前面的示例，其中 `InstructionAccounts` 有一个名为 `account_name` 的字段：

```rust
pub account_name: Account<'info, AccountStruct>
```

这里的 `Account` 包装器执行以下操作：

- 将账户 `data` 反序列化为类型 `AccountStruct` 的格式。
- 检查账户的程序所有者是否与指定给 `AccountStruct` 类型的程序所有者匹配。

当在同一个 crate 中使用 `#[account]` 属性宏定义了 `Account` 包装器中指定的账户类型时，程序所有权检查是针对 `declare_id!` 宏中定义的 `programId` 进行的。

执行的检查如下：

```rust
// Checks
Account.info.owner == T::owner()
!(Account.info.owner == SystemProgram && Account.info.lamports() == 0)
```

### `Signer`

`Signer` 类型验证给定的账户是否对交易进行了签名。不执行其他所有权或类型检查。只有在指令中不需要底层账户数据时才应使用 `Signer`。

对于前面示例中的 `user` 账户，`Signer` 类型指定了 `user` 账户必须是指令的签名者。

为你执行以下检查：

```rust
// Checks
Signer.info.is_signer == true
```

### `Program`

`Program` 类型验证账户是否是某个特定程序。

对于前面示例中的 `system_program` 账户，`Program` 类型用于指定该程序应该是系统程序。Anchor 提供了一个 `System` 类型，其中包括要检查的系统程序的 `programId`。

为你执行以下检查：

```rust
//Checks
account_info.key == expected_program
account_info.executable == true
```

## 使用 `#[account(..)]` 添加约束

`#[account(..)]` 属性宏用于对账户应用约束。在本课程和未来的课程中，我们将介绍一些约束的示例，但是请确保最终查看完整的[可能的约束列表](https://docs.rs/anchor-lang/latest/anchor_lang/derive.Accounts.html)。

再次回想一下 `InstructionAccounts` 示例中的 `account_name` 字段。

```rust
#[account(init, payer = user, space = 8 + 8)]
pub account_name: Account<'info, AccountStruct>,
#[account(mut)]
pub user: Signer<'info>,
```

请注意，`#[account(..)]` 属性包含三个逗号分隔的值：

- `init` - 通过对系统程序执行 CPI 来创建账户并初始化它（设置它的账户辨别器，account discriminator）
- `payer` - 指定账户初始化的付款人为结构体中定义的 `user` 账户
- `space`- 指定为账户分配的空间应为 `8 + 8` 字节。前 8 字节用于 Anchor 自动添加的辨别器，以标识账户类型。接下来的 8 字节为在 `AccountStruct` 类型中定义的账户存储的数据分配空间。

对于 `user`，我们使用 `#[account(..)]` 属性来指定给定的账户是可变的。`user` 账户必须标记为可变，因为将从该账户中扣除 lamports 以支付 `account_name` 的初始化。

```rust
#[account(mut)]
pub user: Signer<'info>,
```

请注意，对 `account_name` 施加的 `init` 约束自动包含了一个 `mut` 约束，以便 `account_name` 和 `user` 都是可变账户。

## `#[account]`

`#[account]` 属性应用于表示 Solana 账户数据结构的结构体。它实现了以下特征：

- `AccountSerialize`
- `AccountDeserialize`
- `AnchorSerialize`
- `AnchorDeserialize`
- `Clone`
- `Discriminator`
- `Owner`

你可以阅读更多关于[每个特征的详细信息](https://docs.rs/anchor-lang/latest/anchor_lang/attr.account.html)。然而，大部分你需要知道的是，`#[account]` 属性使得序列化和反序列化成为可能，并为账户实现了辨别器和所有者特征。

辨别器是一个 8 字节的唯一标识符，用于表示账户类型，它是由账户类型名称的前 8 个字节的 SHA256 哈希派生而来的。在实现账户序列化特征时，前 8 个字节被保留用于账户辨别器。

因此，任何调用 `AccountDeserialize` 的 `try_deserialize` 都将检查这个辨别器。如果不匹配，表示提供了无效的账户，账户反序列化将以错误退出。

`#[account]` 属性还为使用 `declareId` 声明的 `programId` 实现了 `Owner` 特性。换句话说，使用 `#[account]` 属性定义的账户类型初始化的所有账户也都归程序所有。

举个例子，让我们看看 `InstructionAccounts` 中的 `account_name` 使用的 `AccountStruct`。

```rust
#[derive(Accounts)]
pub struct InstructionAccounts {
    #[account(init, payer = user, space = 8 + 8)]
    pub account_name: Account<'info, AccountStruct>,
    ...
}

#[account]
pub struct AccountStruct {
    data: u64
}
```

`#[account]` 属性确保它可以作为 `InstructionAccounts` 中的一个账户使用。

当 `account_name` 账户被初始化时：

- 前 8 字节被设置为 `AccountStruct` 的辨别器
- 账户的数据字段将匹配 `AccountStruct`
- 账户所有者被设置为 `declare_id` 中的 `programId`

## 将所有 Anchor 类型组合起来

当你将所有这些 Anchor 类型结合起来时，你就得到了一个完整的程序。以下是一个基本的 Anchor 程序示例，只包含一个指令，该指令实现了以下功能：

- 初始化一个新账户
- 使用传入指令的数据更新账户上的数据字段

```rust
// Use this import to gain access to common anchor features
use anchor_lang::prelude::*;

// Program onchain address
declare_id!("Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS");

// Instruction logic
#[program]
mod program_module_name {
    use super::*;
    pub fn instruction_one(ctx: Context<InstructionAccounts>, instruction_data: u64) -> Result<()> {
        ctx.accounts.account_name.data = instruction_data;
        Ok(())
    }
}

// Validate incoming accounts for instructions
#[derive(Accounts)]
pub struct InstructionAccounts<'info> {
    #[account(init, payer = user, space = 8 + 8)]
    pub account_name: Account<'info, AccountStruct>,
    #[account(mut)]
    pub user: Signer<'info>,
    pub system_program: Program<'info, System>,

}

// Define custom program account type
#[account]
pub struct AccountStruct {
    data: u64
}
```

你现在已经准备好使用 Anchor 框架构建自己的 Solana 程序了！

# 实验

在我们开始之前，请根据 [Anchor 文档中的步骤](https://www.anchor-lang.com/docs/installation) 安装 Anchor。

在这个实验中，我们将创建一个简单的计数器程序，具有两个指令：

- 第一个指令将初始化一个计数器账户
- 第二个指令将增加存储在计数器账户上的计数

## 1. 设置

通过运行 `anchor init` 创建一个名为 `anchor-counter` 的新项目：

```console
anchor init anchor-counter
```

进入新目录，然后运行 `anchor build`。

```console
cd anchor-counter
anchor build
```

`Anchor build` 还会为您的新程序生成一个密钥对 - 这些密钥保存在 `target/deploy` 目录中。

打开 `lib.rs` 文件，查看 `declare_id!`：

```rust
declare_id!("BouTUP7a3MZLtXqMAm1NrkJSKwAjmid8abqiNjUyBJSr");
```

运行 `anchor keys sync`

```console
anchor keys sync
```

你会看到 Anchor 更新了两个地方：

- `lib.rs` 中的 `declare_id!()` 中使用的密钥
- `Anchor.toml` 中的密钥

以匹配 `anchor build` 过程中生成的密钥：

```console
Found incorrect program id declaration in "anchor-counter/programs/anchor-counter/src/lib.rs"
Updated to BouTUP7a3MZLtXqMAm1NrkJSKwAjmid8abqiNjUyBJSr

Found incorrect program id declaration in Anchor.toml for the program `anchor_counter`
Updated to BouTUP7a3MZLtXqMAm1NrkJSKwAjmid8abqiNjUyBJSr

All program id declarations are synced.
```

最后，删除 `lib.rs` 中的默认代码，直到只剩下以下内容：

```rust
use anchor_lang::prelude::*;

declare_id!("your-private-key");

#[program]
pub mod anchor_counter {
    use super::*;

}
```

## 2. 添加 `initialize` 指令

首先，让我们在 `#[program]` 内实现 `initialize` 指令。该指令需要一个类型为 `Initialize` 的 `Context`，并且不需要额外的指令数据。在指令逻辑中，我们只是将 `counter` 账户的 `count` 字段设置为 `0`。

```rust
pub fn initialize(ctx: Context<Initialize>) -> Result<()> {
    let counter = &mut ctx.accounts.counter;
    counter.count = 0;
    msg!("Counter Account Created");
    msg!("Current Count: { }", counter.count);
    Ok(())
}
```

## 3. 实现 `Context` 类型 `Initialize`

接下来，使用 `#[derive(Accounts)]` 宏，让我们实现 `Initialize` 类型，列出并验证 `initialize` 指令使用的账户。它将需要以下账户：

- `counter` - 在指令中初始化的计数器账户
- `user` - 用于初始化的付款方
- `system_program` - 初始化任何新账户都需要系统程序

```rust
#[derive(Accounts)]
pub struct Initialize<'info> {
    #[account(init, payer = user, space = 8 + 8)]
    pub counter: Account<'info, Counter>,
    #[account(mut)]
    pub user: Signer<'info>,
    pub system_program: Program<'info, System>,
}
```

## 4. 实现 `Counter`

接下来，使用 `#[account]` 属性来定义一个新的 `Counter` 账户类型。`Counter` 结构定义了一个 `count` 字段，类型为 `u64`。这意味着我们可以期望任何以 `Counter` 类型初始化的新账户都具有匹配的数据结构。`#[account]` 属性还会自动为新账户设置鉴别器，并将账户的所有者设置为 `declare_id!` 宏中的 `programId`。

```rust
#[account]
pub struct Counter {
    pub count: u64,
}
```

## 5. 添加 `increment` 指令

在 `#[program]` 中，让我们实现一个 `increment` 指令，以便在第一个指令初始化 `counter` 账户后递增 `count`。该指令需要一个类型为 `Update` 的 `Context`（在下一步中实现），并且不需要额外的指令数据。在指令逻辑中，我们只是将现有的 `counter` 账户的 `count` 字段增加 `1`。

```rust
pub fn increment(ctx: Context<Update>) -> Result<()> {
    let counter = &mut ctx.accounts.counter;
    msg!("Previous counter: {}", counter.count);
    counter.count = counter.count.checked_add(1).unwrap();
    msg!("Counter incremented. Current count: {}", counter.count);
    Ok(())
}
```

## 6. 实现 `Context` 类型 `Update`

最后，再次使用 `#[derive(Accounts)]` 宏，让我们创建列出 `increment` 指令所需账户的 `Update` 类型。它将需要以下账户：

- `counter` - 要递增的现有计数器账户
- `user` - 交易费用的付款方

同样，我们需要使用 `#[account(..)]` 属性指定任何约束：

```rust
#[derive(Accounts)]
pub struct Update<'info> {
    #[account(mut)]
    pub counter: Account<'info, Counter>,
    pub user: Signer<'info>,
}
```

## 7. 构建

所有内容放在一起，完整的程序将如下所示：

```rust
use anchor_lang::prelude::*;

declare_id!("BouTUP7a3MZLtXqMAm1NrkJSKwAjmid8abqiNjUyBJSr");

#[program]
pub mod anchor_counter {
    use super::*;

    pub fn initialize(ctx: Context<Initialize>) -> Result<()> {
        let counter = &mut ctx.accounts.counter;
        counter.count = 0;
        msg!("Counter account created. Current count: {}", counter.count);
        Ok(())
    }

    pub fn increment(ctx: Context<Update>) -> Result<()> {
        let counter = &mut ctx.accounts.counter;
        msg!("Previous counter: {}", counter.count);
        counter.count = counter.count.checked_add(1).unwrap();
        msg!("Counter incremented. Current count: {}", counter.count);
        Ok(())
    }
}

#[derive(Accounts)]
pub struct Initialize<'info> {
    #[account(init, payer = user, space = 8 + 8)]
    pub counter: Account<'info, Counter>,
    #[account(mut)]
    pub user: Signer<'info>,
    pub system_program: Program<'info, System>,
}

#[derive(Accounts)]
pub struct Update<'info> {
    #[account(mut)]
    pub counter: Account<'info, Counter>,
    pub user: Signer<'info>,
}

#[account]
pub struct Counter {
    pub count: u64,
}
```

运行 `anchor build` 来构建程序。

## 8. 测试

Anchor 测试通常是使用 mocha 测试框架的 Typescript 集成测试。我们稍后将学习更多关于测试的内容，但现在请导航到 `anchor-counter.ts` 并用以下内容替换默认的测试代码：

```typescript
import * as anchor from "@coral-xyz/anchor"
import { Program } from "@coral-xyz/anchor"
import { expect } from "chai"
import { AnchorCounter } from "../target/types/anchor_counter"

describe("anchor-counter", () => {
  // Configure the client to use the local cluster.
  const provider = anchor.AnchorProvider.env()
  anchor.setProvider(provider)

  const program = anchor.workspace.AnchorCounter as Program<AnchorCounter>

  const counter = anchor.web3.Keypair.generate()

  it("Is initialized!", async () => {})

  it("Incremented the count", async () => {})
})
```

以上代码为我们将要初始化的 `counter` 账户生成了一个新的密钥对，并为每个指令的测试创建了占位符。

接下来，创建 `initialize` 指令的第一个测试：

```typescript
it("Is initialized!", async () => {
  // Add your test here.
  const tx = await program.methods
    .initialize()
    .accounts({ counter: counter.publicKey })
    .signers([counter])
    .rpc()

  const account = await program.account.counter.fetch(counter.publicKey)
  expect(account.count.toNumber() === 0)
})
```

接下来，创建 `increment` 指令的第二个测试：

```typescript
it("Incremented the count", async () => {
  const tx = await program.methods
    .increment()
    .accounts({ counter: counter.publicKey, user: provider.wallet.publicKey })
    .rpc()

  const account = await program.account.counter.fetch(counter.publicKey)
  expect(account.count.toNumber() === 1)
})
```

最后，运行 `anchor test`，你应该会看到以下输出：

```console
anchor-counter
✔ Is initialized! (290ms)
✔ Incremented the count (403ms)


2 passing (696ms)
```

运行 `anchor test` 会自动启动一个本地测试验证器（local test validator），部署你的程序，并对其运行你的 mocha 测试。如果你现在对测试感到困惑，不用担心 - 我们稍后会更深入地了解。

恭喜你，你刚刚使用 Anchor 框架构建了一个 Solana 程序！如果你需要更多时间，可以随时参考[解决方案代码](https://github.com/Unboxed-Software/anchor-counter-program/tree/solution-increment)。

# 挑战

现在轮到你独立构建了。因为我们从非常简单的程序开始，所以你的程序几乎和我们刚刚创建的一样。尝试达到可以不参考先前代码就能够从头编写的程度是很有用的，所以尽量不要在这里复制粘贴。

1. 编写一个新程序，初始化一个 `counter` 账户
2. 实现 `increment` 和 `decrement` 指令
3. 像我们在实验中那样构建和部署你的程序
4. 测试你新部署的程序，并使用 Solana Explorer 检查程序日志

像往常一样，在这些挑战中发挥创造力，并超越基本的指令，如果你愿意的话 - 并且要玩得开心！

如果可以的话，请尽量独立完成！但如果遇到困难，请随时参考 [解决方案代码](https://github.com/Unboxed-Software/anchor-counter-program/tree/solution-decrement)。

## 完成实验了吗？

将你的代码推送到GitHub，并[告诉我们你对这节课的看法](https://form.typeform.com/to/IPH0UGz7#answers-lesson=334874b7-b152-4473-b5a5-5474c3f8f3f1)！