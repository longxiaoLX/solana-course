---
title: Anchor PDAs 和账户
objectives:
- 在 Anchor 中使用 PDA 账户中利用 `seeds` 和 `bump` 约束
- 启用并使用 `init_if_needed` 约束
- 使用 `realloc` 约束在现有账户上重新分配空间
- 使用 `close` 约束关闭现有账户
---

# TL;DR

- `seeds` 和 `bump` 约束用于在 Anchor 中初始化和验证 PDA 账户
- `init_if_needed` 约束用于有条件地初始化新账户
- `realloc` 约束用于在现有账户上重新分配空间
- `close` 约束用于关闭一个账户并退还其租金

# 概述

在本课程中，您将学习如何在 Anchor 中处理 PDAs，重新分配账户和关闭账户。

回顾一下，Anchor 程序将指令逻辑与账户验证（account validation）分离。账户验证主要发生在表示给定指令所需账户列表的结构体中。结构体的每个字段代表一个不同的账户，并且您可以使用 `#[account(...)]` 属性宏自定义对账户的验证。

除了使用约束进行账户验证之外，一些约束还可以处理重复的任务，否则这些任务会在我们的指令逻辑中产生大量样板代码。本课程将介绍 `seeds`、`bump`、`realloc` 和 `close` 约束，以帮助您初始化和验证 PDAs，重新分配账户和关闭账户。

## 使用 Anchor 的 PDAs

回想一下，[PDAs](https://github.com/Unboxed-Software/solana-course/blob/main/content/pda) 是使用一系列可选种子、一个 bump 种子和一个程序 ID 派生而来的。Anchor 提供了一种方便的方式来使用 `seeds` 和 `bump` 约束来验证一个 PDA。

```rust
#[derive(Accounts)]
struct ExampleAccounts {
  #[account(
    seeds = [b"example_seed"],
    bump
  )]
  pub pda_account: Account<'info, AccountType>,
}
```

在账户验证过程中，Anchor 将使用 `seeds` 约束中指定的种子派生出一个 PDA，并验证传入指令的账户是否与使用指定 `seeds` 找到的 PDA 匹配。

当包含 `bump` 约束但没有指定特定的 bump 时，Anchor 将默认使用规范 bump（canonical，派生一个有效 PDA 的第一个 bump）。在大多数情况下，您应该使用规范 bump。

您可以从约束中访问结构体中的其他字段，因此您可以指定依赖于其他账户的种子，比如签名者的公钥。

您还可以通过将 `#[instruction(...)]` 属性宏添加到结构体中来引用反序列化的指令数据。

例如，以下示例显示了一个包含 `pda_account` 和 `user` 的账户列表。`pda_account` 受到约束，种子必须是字符串 "example_seed"、`user` 的公钥和作为 `instruction_data` 传递给指令的字符串。

```rust
#[derive(Accounts)]
#[instruction(instruction_data: String)]
pub struct Example<'info> {
    #[account(
        seeds = [b"example_seed", user.key().as_ref(), instruction_data.as_ref()],
        bump
    )]
    pub pda_account: Account<'info, AccountType>,
    #[account(mut)]
    pub user: Signer<'info>
}
```

如果客户端提供的 `pda_account` 地址与使用指定种子和规范 bump 派生的 PDA 不匹配，则账户验证将失败。

### 使用带有 `init` 约束的 PDAs

您可以将 `seeds` 和 `bump` 约束与 `init` 约束结合使用，以使用 PDA 初始化账户。

请注意，必须将 `init` 约束与 `payer` 和 `space` 约束结合使用，以指定支付账户初始化的账户和要在新账户上分配的空间。此外，您必须在账户验证结构的字段中包含 `system_program`。

```rust
#[derive(Accounts)]
pub struct InitializePda<'info> {
    #[account(
        init,
        seeds = [b"example_seed", user.key().as_ref()],
        bump,
        payer = user,
        space = 8 + 8
    )]
    pub pda_account: Account<'info, AccountType>,
    #[account(mut)]
    pub user: Signer<'info>,
    pub system_program: Program<'info, System>,
}

#[account]
pub struct AccountType {
    pub data: u64,
}
```

当使用 `init` 初始化**非** PDA 账户时，Anchor 默认将初始化的账户的所有者（owner）设置为当前执行指令的程序。也可以添加 `owner` [约束](https://docs.rs/anchor-lang/latest/anchor_lang/derive.Accounts.html)来指定其他程序。

然而，当将 `init` 与 `seeds` 和 `bump` 结合使用时，所有者必须是执行指令的程序。这是因为为这个 PDA 初始化一个账户需要一个签名，而这个签名只有这个可执行程序可以提供。换句话说，如果用于派生 PDA 的程序 ID 与执行程序的程序 ID 不匹配，则 PDA 账户初始化的签名验证将失败。

在确定为由执行 Anchor 程序拥有的账户初始化的空间值时，请记住，前 8 个字节保留用于账户识别器。这是一个 8 字节的值，Anchor 计算并使用它来识别程序账户类型。您可以使用此 [参考](https://www.anchor-lang.com/docs/space) 来计算应为账户分配多少空间。

### Seed 推断

指令的账户列表对于某些程序可能会变得非常长。为了简化调用 Anchor 程序指令时的客户端体验，我们可以启用种子推断。

种子推断会向 IDL 添加有关 PDA 种子的信息，以便 Anchor 可以从现有的调用站点信息中推断出 PDA 种子。在前面的示例中，种子是 `b"example_seed"` 和 `user.key()`。第一个是静态的，因此是已知的，第二个是已知的，因为 `user` 是交易签名者。

如果在构建程序时使用种子推断，那么只要使用 Anchor 调用程序，您就不需要显式派生和传递 PDA。相反，Anchor 库将为您完成。

您可以在 `Anchor.toml` 文件中使用 `[features]` 下的 `seeds = true` 来启用种子推断功能。

```
[features]
seeds = true
```

### 使用 `#[instruction(...)]` 属性宏

在继续之前，让我们简要地了解一下 `#[instruction(...)]` 属性宏。使用 `#[instruction(...)]` 时，您在参数列表中提供的指令数据必须与指令参数匹配，并且顺序相同。您可以省略列表末尾未使用的参数，但必须包括直到您将使用的最后一个参数为止的所有参数。

例如，假设一条指令有参数 `input_one`、`input_two` 和 `input_three`。如果您的账户约束需要引用 `input_one` 和 `input_three`，则需要在 `#[instruction(...)]` 属性宏中列出所有三个参数。

但是，如果您的约束只引用 `input_one` 和 `input_two`，则可以省略 `input_three`。

```rust
pub fn example_instruction(
    ctx: Context<Example>,
    input_one: String,
    input_two: String,
    input_three: String,
) -> Result<()> {
    ...
    Ok(())
}

#[derive(Accounts)]
#[instruction(input_one:String, input_two:String)]
pub struct Example<'info> {
    ...
}
```

此外，如果您按错误的顺序列出输入，则会收到错误消息：

```rust
#[derive(Accounts)]
#[instruction(input_two:String, input_one:String)]
pub struct Example<'info> {
    ...
}
```

## Init-if-needed

Anchor 提供了 `init_if_needed` 约束，可用于在帐户尚未初始化时初始化该帐户。

这个功能是通过一个特性标志来控制的，以确保您有意识地使用它。出于安全原因，最好避免让一条指令分支到多个逻辑路径上。正如其名称所示，`init_if_needed` 根据问题帐户的状态执行两种可能的代码路径之一。

在使用 `init_if_needed` 时，您需要确保正确保护您的程序免受重新初始化攻击。您需要在代码中包含检查，以确保初始化的帐户在第一次初始化后不能被重置为其初始设置。

要使用 `init_if_needed`，您必须首先在 `Cargo.toml` 中启用该功能。

```rust
[dependencies]
anchor-lang = { version = "0.25.0", features = ["init-if-needed"] }
```

一旦您启用了该功能，就可以在 `#[account(…)]` 属性宏中包含约束。下面的示例演示了如何使用 `init_if_needed` 约束来初始化一个新的关联代币账户，如果该账户不存在的话。

```rust
#[program]
mod example {
    use super::*;
    pub fn initialize(ctx: Context<Initialize>) -> Result<()> {
        Ok(())
    }
}

#[derive(Accounts)]
pub struct Initialize<'info> {
    #[account(
        init_if_needed,
        payer = payer,
        associated_token::mint = mint,
        associated_token::authority = payer
    )]
    pub token_account: Account<'info, TokenAccount>,
    pub mint: Account<'info, Mint>,
     #[account(mut)]
    pub payer: Signer<'info>,
    pub system_program: Program<'info, System>,
    pub token_program: Program<'info, Token>,
    pub associated_token_program: Program<'info, AssociatedToken>,
    pub rent: Sysvar<'info, Rent>,
}
```

在上一个示例中调用 `initialize` 指令时，Anchor 将检查 `token_account` 是否存在，如果不存在，则进行初始化。如果它已经存在，则该指令将继续执行而不初始化该账户。就像使用 `init` 约束一样，如果账户是 PDA，您可以将 `init_if_needed` 与 `seeds` 和 `bump` 结合使用。

## 重新分配空间

`realloc` 约束提供了一种简单的方法来为现有账户重新分配空间。

`realloc` 约束必须与以下约束结合使用：

- `mut` - 账户必须设置为可变的
- `realloc::payer` - 根据重新分配是减少还是增加账户空间，要减少或添加 lamports 的账户
- `realloc::zero` - 用于指定新内存是否应进行零初始化的布尔值

与 `init` 一样，当使用 `realloc` 时，您必须在账户验证结构中将 `system_program` 包括为其中一个账户。

以下是重新分配存储 `String` 类型 `data` 字段的账户空间的示例。

```rust
#[derive(Accounts)]
#[instruction(instruction_data: String)]
pub struct ReallocExample<'info> {
    #[account(
        mut,
        seeds = [b"example_seed", user.key().as_ref()],
        bump,
        realloc = 8 + 4 + instruction_data.len(),
        realloc::payer = user,
        realloc::zero = false,
    )]
    pub pda_account: Account<'info, AccountType>,
    #[account(mut)]
    pub user: Signer<'info>,
    pub system_program: Program<'info, System>,
}

#[account]
pub struct AccountType {
    pub data: String,
}
```

请注意，`realloc` 设置为 `8 + 4 + instruction_data.len()`。其分解如下：
- `8` 用于账户鉴别器
- `4` 用于 BORSH 用于存储字符串长度的 4 字节空间
- `instruction_data.len()` 是字符串本身的长度

如果账户数据长度的变化是增加的，lamports 将从 `realloc::payer` 转移到账户，以保持免租金。同样，如果变化是减少的，则 lamports 将从账户转移到 `realloc::payer`。

`realloc::zero` 约束是必需的，以确定重新分配后的新内存是否应进行零初始化。在您预期账户的内存会多次收缩和扩展的情况下，应将此约束设置为 true。这样可以清除否则会显示为陈旧数据的空间。

## 关闭

`close` 约束提供了一种简单且安全的方式来关闭现有的账户。

`close` 约束在指令执行结束时将账户标记为关闭状态，方法是将其鉴别器设置为`CLOSED_ACCOUNT_DISCRIMINATOR`，并将其 lamports 发送到指定的账户。将鉴别器设置为特殊变体（special variant）使得账户复活攻击（在随后的指令中再次添加免租金 lamports）变得不可能。如果有人尝试重新初始化账户，则重新初始化将失败鉴别器检查，并被程序视为无效。

下面的示例使用 `close` 约束来关闭 `data_account`，并将为租金分配的 lamports 发送到 `receiver` 账户。

```rust
pub fn close(ctx: Context<Close>) -> Result<()> {
    Ok(())
}

#[derive(Accounts)]
pub struct Close<'info> {
    #[account(mut, close = receiver)]
    pub data_account: Account<'info, AccountType>,
    #[account(mut)]
    pub receiver: Signer<'info>
}
```

# 实验

让我们通过使用 Anchor 框架创建一个电影评论程序来练习我们在本课程中讨论的概念。

该程序将允许用户：

- 使用 PDA 初始化一个新的电影评论账户以存储评论
- 更新现有电影评论账户的内容
- 关闭现有电影评论账户

## 1. 创建一个新的 Anchor 项目

首先，让我们使用 `anchor init` 创建一个新的项目。

```console
anchor init anchor-movie-review-program
```

接下来，转到 `programs` 文件夹中的 `lib.rs` 文件，您应该会看到以下起始代码。

```rust
use anchor_lang::prelude::*;

declare_id!("Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS");

#[program]
pub mod anchor_movie_review_program {
    use super::*;

    pub fn initialize(ctx: Context<Initialize>) -> Result<()> {
        Ok(())
    }
}

#[derive(Accounts)]
pub struct Initialize {}
```

请继续移除 `initialize` 指令和 `Initialize` 类型。

```rust
use anchor_lang::prelude::*;

declare_id!("Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS");

#[program]
pub mod anchor_movie_review_program {
    use super::*;

}
```

## 2. `MovieAccountState`

首先，让我们使用 `#[account]` 属性宏定义 `MovieAccountState`，它将表示电影评论账户的数据结构。作为提醒，`#[account]` 属性宏实现了各种特性，有助于账户的序列化和反序列化，设置账户的鉴别器，并将新账户的所有者设置为在 `declare_id!` 宏中定义的程序 ID。

在每个电影评论账户中，我们将存储以下信息：

- `reviewer` - 创建评论的用户
- `rating` - 电影的评分
- `title` - 电影的标题
- `description` - 评论的内容

```rust
use anchor_lang::prelude::*;

declare_id!("Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS");

#[program]
pub mod anchor_movie_review_program {
    use super::*;

}

#[account]
pub struct MovieAccountState {
    pub reviewer: Pubkey,    // 32
    pub rating: u8,          // 1
    pub title: String,       // 4 + len()
    pub description: String, // 4 + len()
}
```

## 3. 添加电影评论

接下来，让我们实现 `add_movie_review` 指令。`add_movie_review` 指令将需要一个我们稍后将实现的类型为 `AddMovieReview` 的 `Context`。

该指令将需要三个额外的参数作为评论者提供的指令数据：

- `title` - 电影的标题，类型为 `String`
- `description` - 评论的详细信息，类型为 `String`
- `rating` - 电影的评分，类型为 `u8`

在指令逻辑中，我们将使用指令数据填充新的 `movie_review` 账户的数据。我们还将 `reviewer` 字段设置为指令上下文中的 `initializer` 账户。

```rust
#[program]
pub mod movie_review{
    use super::*;

    pub fn add_movie_review(
        ctx: Context<AddMovieReview>,
        title: String,
        description: String,
        rating: u8,
    ) -> Result<()> {
        msg!("Movie Review Account Created");
        msg!("Title: {}", title);
        msg!("Description: {}", description);
        msg!("Rating: {}", rating);

        let movie_review = &mut ctx.accounts.movie_review;
        movie_review.reviewer = ctx.accounts.initializer.key();
        movie_review.title = title;
        movie_review.rating = rating;
        movie_review.description = description;
        Ok(())
    }
}
```

接下来，让我们创建我们在指令上下文中用作泛型的 `AddMovieReview` 结构体。此结构体将列出 `add_movie_review` 指令所需的账户。

请记住，您将需要以下宏：

- `#[derive(Accounts)]` 宏用于反序列化和验证结构体中指定的账户列表
- `#[instruction(...)]` 属性宏用于访问传递给指令的指令数据
- 然后，`#[account(...)]` 属性宏指定账户的附加约束

`movie_review` 账户是一个需要初始化的 PDA，因此我们将添加 `seeds` 和 `bump` 约束，以及带有所需的 `payer` 和 `space` 约束的 `init` 约束。

对于 PDA 种子，我们将使用电影标题和评论者的公钥。初始化的支付方应为评论者，账户分配的空间应足够存放账户鉴别器、评论者的公钥以及电影评论的评分、标题和描述。

```rust
#[derive(Accounts)]
#[instruction(title:String, description:String)]
pub struct AddMovieReview<'info> {
    #[account(
        init,
        seeds = [title.as_bytes(), initializer.key().as_ref()],
        bump,
        payer = initializer,
        space = 8 + 32 + 1 + 4 + title.len() + 4 + description.len()
    )]
    pub movie_review: Account<'info, MovieAccountState>,
    #[account(mut)]
    pub initializer: Signer<'info>,
    pub system_program: Program<'info, System>,
}
```

## 4. 更新电影评论

接下来，让我们使用上下文的泛型类型为 `UpdateMovieReview` 实现 `update_movie_review` 指令。

与之前一样，该指令将需要三个额外的参数作为评论者提供的指令数据：

- `title` - 电影的标题
- `description` - 评论的详细信息
- `rating` - 电影的评分

在指令逻辑中，我们将更新存储在 `movie_review` 账户上的 `rating` 和 `description`。

虽然 `title` 在指令函数本身中没有被使用，但我们将在下一步中需要它来验证 `movie_review` 账户。

```rust
#[program]
pub mod anchor_movie_review_program {
    use super::*;

		...

    pub fn update_movie_review(
        ctx: Context<UpdateMovieReview>,
        title: String,
        description: String,
        rating: u8,
    ) -> Result<()> {
        msg!("Movie review account space reallocated");
        msg!("Title: {}", title);
        msg!("Description: {}", description);
        msg!("Rating: {}", rating);

        let movie_review = &mut ctx.accounts.movie_review;
        movie_review.rating = rating;
        movie_review.description = description;

        Ok(())
    }

}
```

接下来，让我们创建 `UpdateMovieReview` 结构体来定义 `update_movie_review` 指令所需的账户。

由于此时 `movie_review` 账户已经被初始化，我们不再需要 `init` 约束。然而，由于 `description` 的值现在可能不同，我们需要使用 `realloc` 约束来重新分配账户上的空间。除此之外，我们还需要 `mut`、`realloc::payer` 和 `realloc::zero` 约束。

我们仍然需要 `seeds` 和 `bump` 约束，就像在 `AddMovieReview` 中一样。

```rust
#[derive(Accounts)]
#[instruction(title:String, description:String)]
pub struct UpdateMovieReview<'info> {
    #[account(
        mut,
        seeds = [title.as_bytes(), initializer.key().as_ref()],
        bump,
        realloc = 8 + 32 + 1 + 4 + title.len() + 4 + description.len(),
        realloc::payer = initializer,
        realloc::zero = true,
    )]
    pub movie_review: Account<'info, MovieAccountState>,
    #[account(mut)]
    pub initializer: Signer<'info>,
    pub system_program: Program<'info, System>,
}
```

请注意，`realloc` 约束设置为了基于更新后的 `description` 值，能够使 `movie_review` 账户拥有新的空间。

另外，`realloc::payer` 约束指定了所需的额外 lamports 将来自于或发送到 `initializer` 账户。

最后，我们将 `realloc::zero` 约束设置为 `true`，因为 `movie_review` 账户可能会被多次更新，无论是缩小还是扩大分配给该账户的空间。

## 5. 删除电影评论

最后，让我们实现 `delete_movie_review` 指令以关闭现有的 `movie_review` 账户。

我们将使用泛型类型为 `DeleteMovieReview` 的上下文，不会包含任何额外的指令数据。由于我们只是关闭一个账户，实际上我们不需要在函数体内添加任何指令逻辑。关闭操作本身将由 `DeleteMovieReview` 类型中的 Anchor 约束处理。

```rust
#[program]
pub mod anchor_movie_review_program {
    use super::*;

		...

    pub fn delete_movie_review(_ctx: Context<DeleteMovieReview>, title: String) -> Result<()> {
        msg!("Movie review for {} deleted", title);
        Ok(())
    }

}
```

接下来，让我们实现 `DeleteMovieReview` 结构体。

```rust
#[derive(Accounts)]
#[instruction(title: String)]
pub struct DeleteMovieReview<'info> {
    #[account(
        mut,
        seeds=[title.as_bytes(), initializer.key().as_ref()],
        bump,
        close=initializer
    )]
    pub movie_review: Account<'info, MovieAccountState>,
    #[account(mut)]
    pub initializer: Signer<'info>,
    pub system_program: Program<'info, System>
}
```

在这里，我们使用 `close` 约束来指定我们正在关闭 `movie_review` 账户，并且租金应退还给 `initializer` 账户。我们还包括了 `movie_review` 账户的 `seeds` 和 `bump` 约束以进行验证。然后 Anchor 处理了安全关闭账户所需的额外逻辑。

## 6. 测试

程序应该已经准备就绪！现在让我们来测试一下。转到 `anchor-movie-review-program.ts`，并用以下内容替换默认的测试代码。

在这里我们：

- 为电影评论指令数据创建默认值
- 推导电影评论账户的 PDA
- 创建测试的占位符

```typescript
import * as anchor from "@project-serum/anchor"
import { Program } from "@project-serum/anchor"
import { assert, expect } from "chai"
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

  const [moviePda] = anchor.web3.PublicKey.findProgramAddressSync(
    [Buffer.from(movie.title), provider.wallet.publicKey.toBuffer()],
    program.programId
  )

  it("Movie review is added`", async () => {})

  it("Movie review is updated`", async () => {})

  it("Deletes a movie review", async () => {})
})
```

接下来，让我们为 `addMovieReview` 指令创建第一个测试。请注意，我们没有显式添加 `.accounts`。这是因为来自 `AnchorProvider` 的 `Wallet` 自动包括为签名者，Anchor 可以推断某些账户，如 `SystemProgram`，而且 Anchor 也可以从指令参数的 `title` 和签名者的公钥推断出 `movieReview` PDA。

一旦指令运行，我们就获取 `movieReview` 账户，并检查账户上存储的数据是否与预期值匹配。

```typescript
it("Movie review is added`", async () => {
  // Add your test here.
  const tx = await program.methods
    .addMovieReview(movie.title, movie.description, movie.rating)
    .rpc()

  const account = await program.account.movieAccountState.fetch(moviePda)
  expect(movie.title === account.title)
  expect(movie.rating === account.rating)
  expect(movie.description === account.description)
  expect(account.reviewer === provider.wallet.publicKey)
})
```

接下来，让我们按照之前的流程为 `updateMovieReview` 指令创建测试。

```typescript
it("Movie review is updated`", async () => {
  const newDescription = "Wow this is new"
  const newRating = 4

  const tx = await program.methods
    .updateMovieReview(movie.title, newDescription, newRating)
    .rpc()

  const account = await program.account.movieAccountState.fetch(moviePda)
  expect(movie.title === account.title)
  expect(newRating === account.rating)
  expect(newDescription === account.description)
  expect(account.reviewer === provider.wallet.publicKey)
})
```

接下来，为 `deleteMovieReview` 指令创建测试。

```typescript
it("Deletes a movie review", async () => {
  const tx = await program.methods
    .deleteMovieReview(movie.title)
    .rpc()
})
```

最后，运行 `anchor test`，你应该在控制台中看到以下输出。

```console
  anchor-movie-review-program
    ✔ Movie review is added` (139ms)
    ✔ Movie review is updated` (404ms)
    ✔ Deletes a movie review (403ms)


  3 passing (950ms)
```

如果你需要更多时间来熟悉这些概念，可以随时在继续之前查看[解决方案代码](https://github.com/Unboxed-Software/anchor-movie-review-program/tree/solution-pdas)。

# 挑战

现在轮到你独立构建一些东西了。利用本课程介绍的概念，尝试使用 Anchor 框架重新创建我们之前使用过的学生介绍程序。

学生介绍程序是一个 Solana 程序，允许学生介绍自己。该程序以用户的姓名和简短消息作为指令数据，并创建一个账户将数据存储在链上。

利用本课程学到的知识，构建这个程序。该程序应包括以下指令：

1. 为每个学生初始化一个 PDA 账户，存储学生的姓名和简短消息
2. 更新现有账户上的消息
3. 关闭现有账户

如果可能的话，请尝试独立完成！但如果遇到困难，请随时参考 [解决方案代码](https://github.com/Unboxed-Software/anchor-student-intro-program)。

## 完成了实验吗？

将你的代码推送到 GitHub，并[告诉我们你对这节课的看法](https://form.typeform.com/to/IPH0UGz7#answers-lesson=f58108e9-94a0-45b2-b0d5-44ada1909105)！