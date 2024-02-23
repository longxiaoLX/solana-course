---
title: Anchor CPIs and Errors
objectives:
- 从 Anchor 程序中进行跨程序调用 (CPIs)
- 使用 `cpi` 功能生成调用现有 Anchor 程序指令的辅助函数
- 在无法使用 CPI 辅助函数的情况下，使用 `invoke` 和 `invoke_signed` 进行 CPIs
- 创建并返回自定义的 Anchor 错误
---

# 总结

- Anchor 提供了一种简化的方法来使用 **`CpiContext`** 创建 CPIs
- Anchor 的 **`cpi`** 功能会为调用现有 Anchor 程序指令生成 CPI 辅助函数
- 如果你无法访问 CPI 辅助函数，你仍然可以直接使用 `invoke` 和 `invoke_signed`
- 使用 **`error_code`** 属性宏可以创建自定义的 Anchor 错误

# 概述

如果回想一下[第一个 CPI 课程](./cpi.md)，你会记得使用纯 Rust 构建 CPIs 可能会变得有些棘手。不过，Anchor 让这个过程变得简单一些，特别是如果你要调用的程序也是一个 Anchor 程序，而且你可以访问其 crate。

在这节课中，你将学习如何构建一个 Anchor CPI。你还将学习如何从 Anchor 程序中抛出自定义错误，这样你就可以开始编写更复杂的 Anchor 程序了。

## 使用 Anchor 进行跨程序调用（CPIs）

作为温习，CPIs 允许程序使用 `invoke` 或 `invoke_signed` 函数调用其他程序的指令。这使得新程序可以构建在现有程序的基础之上（我们称之为可组合性，composability）。

虽然直接使用 `invoke` 或 `invoke_signed` 进行 CPIs 仍然是一种选择，但 Anchor 还提供了一种简化的方式，即使用 `CpiContext` 进行 CPIs。

在本课程中，你将使用 `anchor_spl` crate 对 SPL Token Program 进行 CPIs。你可以[探索 `anchor_spl` crate 中提供的内容](https://docs.rs/anchor-spl/latest/anchor_spl/#)。

### `CpiContext`

创建 CPI 的第一步是创建一个 `CpiContext` 的实例。`CpiContext` 非常类似于 Anchor 指令函数所需的第一个参数类型 `Context`。它们都声明在同一个模块中，并且共享类似的功能。

`CpiContext` 类型指定了跨程序调用的非参数输入：

- `accounts` - 调用指令所需的账户列表
- `remaining_accounts` - 任何剩余的账户
- `program` - 被调用程序的程序 ID
- `signer_seeds` - 如果 PDA 在签名，则包括派生 PDA 所需的种子

```rust
pub struct CpiContext<'a, 'b, 'c, 'info, T>
where
    T: ToAccountMetas + ToAccountInfos<'info>,
{
    pub accounts: T,
    pub remaining_accounts: Vec<AccountInfo<'info>>,
    pub program: AccountInfo<'info>,
    pub signer_seeds: &'a [&'b [&'c [u8]]],
}
```

在传递原始交易签名时，你可以使用 `CpiContext::new` 来构造一个新实例。

```rust
CpiContext::new(cpi_program, cpi_accounts)
```

```rust
pub fn new(
        program: AccountInfo<'info>,
        accounts: T
    ) -> Self {
    Self {
        accounts,
        program,
        remaining_accounts: Vec::new(),
        signer_seeds: &[],
    }
}
```

当代表 PDA 对 CPI 进行签名时，你可以使用 `CpiContext::new_with_signer` 来构造一个新实例。

```rust
CpiContext::new_with_signer(cpi_program, cpi_accounts, seeds)
```

```rust
pub fn new_with_signer(
    program: AccountInfo<'info>,
    accounts: T,
    signer_seeds: &'a [&'b [&'c [u8]]],
) -> Self {
    Self {
        accounts,
        program,
        signer_seeds,
        remaining_accounts: Vec::new(),
    }
}
```

### CPI 账户

`CpiContext` 的一个主要优点是，`accounts` 参数是一个泛型类型，允许你传入任何实现了 `ToAccountMetas` 和 `ToAccountInfos<'info>` 特征的对象。

这些特征是通过之前创建指令账户表示结构体时使用的 `#[derive(Accounts)]` 属性宏添加的。这意味着你可以使用类似的结构体来与 `CpiContext` 搭配使用。

这有助于代码组织和类型安全。

### 在另一个 Anchor 程序上调用指令

当你要调用的程序是一个已发布 crate 的 Anchor 程序时，Anchor 可以为你生成指令构建器（instruction builders）和 CPI 辅助函数。

只需在你的程序的 `Cargo.toml` 文件中声明你的程序对要调用的程序的依赖，如下所示：

```
[dependencies]
callee = { path = "../callee", features = ["cpi"]}
```

通过添加 `features = ["cpi"]`，你可以启用 `cpi` 功能，你的程序就可以访问 `callee::cpi` 模块了。

`cpi` 模块将 `callee` 的指令暴露为一个 Rust 函数，该函数接受一个 `CpiContext` 和任何额外的指令数据作为参数。这些函数的格式与你的 Anchor 程序中的指令函数相同，只是使用 `CpiContext` 替换了 `Context`。`cpi` 模块还公开了调用这些指令所需的账户结构体。

例如，如果 `callee` 有一个需要在 `DoSomething` 结构体中定义的账户的指令 `do_something`，你可以按如下方式调用 `do_something`：

```rust
use anchor_lang::prelude::*;
use callee;
...

#[program]
pub mod lootbox_program {
    use super::*;

    pub fn call_another_program(ctx: Context<CallAnotherProgram>, params: InitUserParams) -> Result<()> {
        callee::cpi::do_something(
            CpiContext::new(
                ctx.accounts.callee.to_account_info(),
                callee::DoSomething {
                    user: ctx.accounts.user.to_account_info()
                }
            )
        )
        Ok(())
    }
}
...
```

### 在非 Anchor 程序上调用指令

当你要调用的程序 *不是* 一个 Anchor 程序时，有两种可能的选择：

1. 可能该程序的维护者已经发布了一个 crate，其中包含了用于调用他们的程序的自己的辅助函数。例如，`anchor_spl` crate 提供了与 Anchor 程序的 `cpi` 模块在调用站点的角度几乎相同的辅助函数。例如，你可以使用 [`mint_to` 辅助函数](https://docs.rs/anchor-spl/latest/src/anchor_spl/token.rs.html#36-58) 进行铸币，使用 [`MintTo` 账户结构体](https://docs.rs/anchor-spl/latest/anchor_spl/token/struct.MintTo.html)。

    ```rust
    token::mint_to(
        CpiContext::new_with_signer(
            ctx.accounts.token_program.to_account_info(),
            token::MintTo {
                mint: ctx.accounts.mint_account.to_account_info(),
                to: ctx.accounts.token_account.to_account_info(),
                authority: ctx.accounts.mint_authority.to_account_info(),
            },
            &[&[
                "mint".as_bytes(),
                &[*ctx.bumps.get("mint_authority").unwrap()],
            ]]
        ),
        amount,
    )?;
    ```

2. 如果没有针对你需要调用的程序的辅助模块，你可以退而使用 `invoke` 和 `invoke_signed`。事实上，上面引用的 `mint_to` 辅助函数的源代码显示了一个示例，当给定一个 `CpiContext` 时使用 `invoke_signed`。如果你决定使用账户结构体和 `CpiContext` 来组织和准备你的 CPI，你可以遵循类似的模式。

    ```rust
    pub fn mint_to<'a, 'b, 'c, 'info>(
        ctx: CpiContext<'a, 'b, 'c, 'info, MintTo<'info>>,
        amount: u64,
    ) -> Result<()> {
        let ix = spl_token::instruction::mint_to(
            &spl_token::ID,
            ctx.accounts.mint.key,
            ctx.accounts.to.key,
            ctx.accounts.authority.key,
            &[],
            amount,
        )?;
        solana_program::program::invoke_signed(
            &ix,
            &[
                ctx.accounts.to.clone(),
                ctx.accounts.mint.clone(),
                ctx.accounts.authority.clone(),
            ],
            ctx.signer_seeds,
        )
        .map_err(Into::into)
    }
    ```

## 在 Anchor 中抛出错误

我们已经深入到 Anchor 中了，所以知道如何创建自定义错误是很重要的。

最终，所有程序都返回相同的错误类型：[`ProgramError`](https://docs.rs/solana-program/latest/solana_program/program_error/enum.ProgramError.html)。然而，在使用 Anchor 编写程序时，你可以使用 `AnchorError` 作为 `ProgramError` 的抽象。该抽象在程序失败时提供了额外的信息，包括：

- 错误名称和编号
- 抛出错误的代码位置
- 违反约束的账户

```rust
pub struct AnchorError {
    pub error_name: String,
    pub error_code_number: u32,
    pub error_msg: String,
    pub error_origin: Option<ErrorOrigin>,
    pub compared_values: Option<ComparedValues>,
}
```

Anchor 错误可以分为：

- Anchor 内部错误，框架从其自己的代码中返回
- 开发人员可以创建的自定义错误

你可以使用 `error_code` 属性为你的程序添加独特的错误。只需将此属性添加到自定义的 `enum` 类型上。然后，你可以在你的程序中使用 `enum` 的变体作为错误。此外，你还可以使用 `msg` 属性为每个变体添加错误消息。如果发生错误，客户端可以显示此错误消息。

```rust
#[error_code]
pub enum MyError {
    #[msg("MyAccount may only hold data below 100")]
    DataTooLarge
}
```

要返回自定义错误，你可以在指令函数中使用 [err](https://docs.rs/anchor-lang/latest/anchor_lang/macro.err.html) 或 [error](https://docs.rs/anchor-lang/latest/anchor_lang/prelude/macro.error.html) 宏。这些宏会向错误添加文件和行信息，然后由 Anchor 记录下来，帮助你进行调试。

```rust
#[program]
mod hello_anchor {
    use super::*;
    pub fn set_data(ctx: Context<SetData>, data: MyAccount) -> Result<()> {
        if data.data >= 100 {
            return err!(MyError::DataTooLarge);
        }
        ctx.accounts.my_account.set_inner(data);
        Ok(())
    }
}

#[error_code]
pub enum MyError {
    #[msg("MyAccount may only hold data below 100")]
    DataTooLarge
}
```

另外，你还可以使用 [require](https://docs.rs/anchor-lang/latest/anchor_lang/macro.require.html) 宏来简化返回错误。上面的代码可以重构为以下形式：

```rust
#[program]
mod hello_anchor {
    use super::*;
    pub fn set_data(ctx: Context<SetData>, data: MyAccount) -> Result<()> {
        require!(data.data < 100, MyError::DataTooLarge);
        ctx.accounts.my_account.set_inner(data);
        Ok(())
    }
}

#[error_code]
pub enum MyError {
    #[msg("MyAccount may only hold data below 100")]
    DataTooLarge
}
```

# 实验

让我们通过在之前课程中介绍的电影评论程序基础上构建，来练习本课程中学到的概念。

在这个实验中，我们将更新程序，以便在用户提交新的电影评论时为他们铸造代币。

## 1. 起步

要开始实验，我们将使用上一课程中 Anchor 电影评论程序的最终状态。因此，如果你刚刚完成了那节课程，那么你已经准备好了。如果你刚刚开始学习，也没关系，你可以[下载起始代码](https://github.com/Unboxed-Software/anchor-movie-review-program/tree/solution-pdas)。我们将使用 `solution-pdas` 分支作为起点。

## 2. 在 `Cargo.toml` 中添加依赖项

在开始之前，我们需要启用 `init-if-needed` 功能，并在 `Cargo.toml` 的依赖项中添加 `anchor-spl` crate。如果你需要复习 `init-if-needed` 功能，请查看 [Anchor PDAs and Accounts 课程](./anchor-pdas.md)。

```rust
[dependencies]
anchor-lang = { version = "0.25.0", features = ["init-if-needed"] }
anchor-spl = "0.25.0"
```

## 3. 初始化奖励代币

接下来，导航到 `lib.rs` 并创建一个指令来初始化一个新的代币铸币厂。这将是每次用户留下评论时铸造的代币。请注意，我们不需要包含任何自定义指令逻辑，因为初始化完全可以通过 Anchor 约束来处理。

```rust
pub fn initialize_token_mint(_ctx: Context<InitializeMint>) -> Result<()> {
    msg!("Token mint initialized");
    Ok(())
}
```

现在，实现 `InitializeMint` 上下文类型，并列出指令所需的账户和约束。在这里，我们使用字符串 "mint" 作为种子，使用 PDA 初始化一个新的 `Mint` 账户。请注意，我们可以将相同的 PDA 用于 `Mint` 账户和铸币权限的地址。使用 PDA 作为铸币权限使我们的程序能够签署代币的铸造。

为了初始化 `Mint` 账户，我们需要将 `token_program`、`rent` 和 `system_program` 包含在账户列表中。

```rust
#[derive(Accounts)]
pub struct InitializeMint<'info> {
    #[account(
        init,
        seeds = ["mint".as_bytes()],
        bump,
        payer = user,
        mint::decimals = 6,
        mint::authority = mint,
    )]
    pub mint: Account<'info, Mint>,
    #[account(mut)]
    pub user: Signer<'info>,
    pub token_program: Program<'info, Token>,
    pub rent: Sysvar<'info, Rent>,
    pub system_program: Program<'info, System>
}
```

可能有一些约束你还没有见过。添加 `mint::decimals` 和 `mint::authority`，以及 `init` 确保了该账户以适当的小数位和铸币权限设置为初始化为新的代币铸币厂。

## 4. Anchor 错误

接下来，让我们创建一个 Anchor 错误，当验证传递给 `add_movie_review` 或 `update_movie_review` 指令的 `rating` 时使用。

```rust
#[error_code]
enum MovieReviewError {
    #[msg("Rating must be between 1 and 5")]
    InvalidRating
}
```

## 5. 更新 `add_movie_review` 指令

现在我们已经完成了一些设置，让我们更新 `add_movie_review` 指令和 `AddMovieReview` 上下文类型，以便为评论者铸造代币。

接下来，更新 `AddMovieReview` 上下文类型，添加以下账户：

- `token_program` - 我们将使用 Token 程序来铸造代币
- `mint` - 我们将在用户添加电影评论时铸造代币的代币铸币厂账户
- `token_account` - 与上述 `mint` 和评论者关联的代币账户
- `associated_token_program` - 必需的，因为我们将在 `token_account` 上使用 `associated_token` 约束
- `rent` - 必需的，因为我们在 `token_account` 上使用了 `init-if-needed` 约束

```rust
#[derive(Accounts)]
#[instruction(title: String, description: String)]
pub struct AddMovieReview<'info> {
    #[account(
        init,
        seeds=[title.as_bytes(), initializer.key().as_ref()],
        bump,
        payer = initializer,
        space = 8 + 32 + 1 + 4 + title.len() + 4 + description.len()
    )]
    pub movie_review: Account<'info, MovieAccountState>,
    #[account(mut)]
    pub initializer: Signer<'info>,
    pub system_program: Program<'info, System>,
    // ADDED ACCOUNTS BELOW
    pub token_program: Program<'info, Token>,
    #[account(
        seeds = ["mint".as_bytes()]
        bump,
        mut
    )]
    pub mint: Account<'info, Mint>,
    #[account(
        init_if_needed,
        payer = initializer,
        associated_token::mint = mint,
        associated_token::authority = initializer
    )]
    pub token_account: Account<'info, TokenAccount>,
    pub associated_token_program: Program<'info, AssociatedToken>,
    pub rent: Sysvar<'info, Rent>
}
```

同样，上面的一些约束可能对你来说并不熟悉。`associated_token::mint` 和 `associated_token::authority` 约束以及 `init_if_needed` 约束确保，如果账户尚未初始化，它将根据指定铸币厂和权限初始化关联的代币账户。

接下来，让我们更新 `add_movie_review` 指令来执行以下操作：

- 检查 `rating` 是否有效。如果它不是有效的评分，则返回 `InvalidRating` 错误。
- 使用铸币权限 PDA 作为签名者，向代币程序的 `mint_to` 指令进行 CPI。请注意，我们将向用户铸造 10 个代币，但需要根据铸币的小数位进行调整，使其为 `10*10^6`。

幸运的是，我们可以使用 `anchor_spl` crate 来访问辅助函数和类型，如 `mint_to` 和 `MintTo`，以构建我们的 CPI 到 Token 程序。`mint_to` 接受一个 `CpiContext` 和一个整数作为参数，其中整数表示要铸造的代币数量。`MintTo` 可用于铸币指令所需的账户列表。

```rust
pub fn add_movie_review(ctx: Context<AddMovieReview>, title: String, description: String, rating: u8) -> Result<()> {
    msg!("Movie review account created");
    msg!("Title: {}", title);
    msg!("Description: {}", description);
    msg!("Rating: {}", rating);

    require!(rating >= 1 && rating <= 5, MovieReviewError::InvalidRating);

    let movie_review = &mut ctx.accounts.movie_review;
    movie_review.reviewer = ctx.accounts.initializer.key();
    movie_review.title = title;
    movie_review.description = description;
    movie_review.rating = rating;

    mint_to(
        CpiContext::new_with_signer(
            ctx.accounts.token_program.to_account_info(),
            MintTo {
                authority: ctx.accounts.mint.to_account_info(),
                to: ctx.accounts.token_account.to_account_info(),
                mint: ctx.accounts.mint.to_account_info()
            },
            &[&[
                "mint".as_bytes(),
                &[*ctx.bumps.get("mint").unwrap()]
            ]]
        ),
        10*10^6
    )?;

    msg!("Minted tokens");

    Ok(())
}
```

## 6. 更新 `update_movie_review` 指令

在这里，我们只添加了 `rating` 是否有效的检查。

```rust
pub fn update_movie_review(ctx: Context<UpdateMovieReview>, title: String, description: String, rating: u8) -> Result<()> {
    msg!("Movie review account space reallocated");
    msg!("Title: {}", title);
    msg!("Description: {}", description);
    msg!("Rating: {}", rating);

    require!(rating >= 1 && rating <= 5, MovieReviewError::InvalidRating);

    let movie_review = &mut ctx.accounts.movie_review;
    movie_review.description = description;
    movie_review.rating = rating;

    Ok(())
}
```

## 7. 测试

这就是我们需要对程序进行的所有更改！现在，让我们更新我们的测试。

首先确保你的导入和 `describe` 函数看起来像这样：

```typescript
import * as anchor from "@coral-xyz/anchor"
import { Program } from "@coral-xyz/anchor"
import { expect } from "chai"
import { getAssociatedTokenAddress, getAccount } from "@solana/spl-token"
import { AnchorMovieReviewProgram } from "../target/types/anchor_movie_review_program"

describe("anchor-movie-review-program", () => {
  // Configure the client to use the local cluster.
  const provider = anchor.AnchorProvider.env()
  anchor.setProvider(provider)

  const program = anchor.workspace
    .AnchorMovieReviewProgram as Program<AnchorMovieReviewProgram>

  const movie = {
    title: "Just a test movie",
    description: "Wow what a good movie it was real great",
    rating: 5,
  }

  const [movie_pda] = anchor.web3.PublicKey.findProgramAddressSync(
    [Buffer.from(movie.title), provider.wallet.publicKey.toBuffer()],
    program.programId
  )

  const [mint] = anchor.web3.PublicKey.findProgramAddressSync(
    [Buffer.from("mint")],
    program.programId
  )
...
}
```

完成后，为 `initializeTokenMint` 指令添加一个测试：

```typescript
it("Initializes the reward token", async () => {
    const tx = await program.methods.initializeTokenMint().rpc()
})
```

请注意，我们不需要添加 `.accounts`，因为它们可以被推断出来，包括 `mint` 账户（假设你已经启用了种子推断）。

接下来，更新 `addMovieReview` 指令的测试。主要的添加内容是：

1. 获取需要作为无法推断的账户传递给指令的关联代币地址
2. 在测试结束时检查关联代币账户是否有 10 个代币

```typescript
it("Movie review is added`", async () => {
  const tokenAccount = await getAssociatedTokenAddress(
    mint,
    provider.wallet.publicKey
  )
  
  const tx = await program.methods
    .addMovieReview(movie.title, movie.description, movie.rating)
    .accounts({
      tokenAccount: tokenAccount,
    })
    .rpc()
  
  const account = await program.account.movieAccountState.fetch(movie_pda)
  expect(movie.title === account.title)
  expect(movie.rating === account.rating)
  expect(movie.description === account.description)
  expect(account.reviewer === provider.wallet.publicKey)

  const userAta = await getAccount(provider.connection, tokenAccount)
  expect(Number(userAta.amount)).to.equal((10 * 10) ^ 6)
})
```

在此之后，`updateMovieReview` 的测试和 `deleteMovieReview` 的测试都不需要任何更改。

在这一点上，运行 `anchor test`，你应该会看到以下输出:

```console
anchor-movie-review-program
    ✔ Initializes the reward token (458ms)
    ✔ Movie review is added (410ms)
    ✔ Movie review is updated (402ms)
    ✔ Deletes a movie review (405ms)

  5 passing (2s)
```

如果你需要更多时间来理解本课程的概念，或者在学习过程中遇到了困难，可以随时查看[解决方案代码](https://github.com/Unboxed-Software/anchor-movie-review-program/tree/solution-add-tokens)。请注意，这个实验的解决方案位于 `solution-add-tokens` 分支上。

# 挑战

要将本课程中学到的关于 CPIs 的知识应用到学生介绍程序中，可以考虑如何将它们整合到该程序中。你可以类似于我们在这里的实验中所做的事情，为用户在介绍自己时铸造一些代币。

如果可能的话，尽量独立完成这个任务！但如果遇到困难，可以参考这个[解决方案代码](https://github.com/Unboxed-Software/anchor-student-intro-program/tree/cpi-challenge)。请注意，根据你的实现方式，你的代码可能与解决方案代码略有不同。

## 完成了实验吗？

将你的代码推送到 GitHub，并[告诉我们你对这节课的想法](https://form.typeform.com/to/IPH0UGz7#answers-lesson=21375c76-b6f1-4fb6-8cc1-9ef151bc5b0a)！