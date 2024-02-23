---
title: PDAs
objectives:
- 解释程序派生地址（PDAs）
- 解释 PDAs 的各种用例
- 描述 PDAs 的派生方式
- 使用 PDA 派生来定位和检索数据
---

# TL;DR

- **程序派生地址**（PDA）是从**程序 ID**和可选的**种子**列表派生而来的
- PDAs 由其派生的程序拥有和控制
- PDA 派生提供了一种确定性的方式，根据用于派生的种子找到数据
- 种子可用于映射到存储在单独的 PDA 账户中的数据
- 一个程序可以代表从其 ID 派生的 PDAs 签名指令

# 概述

## 程序派生地址是什么？

**程序派生地址**（Program Derived Addresses，PDAs）是设计为由程序签名而不是私钥签名的账户地址。正如名称所示，PDAs 是使用程序 ID 派生的。这些派生账户也可以使用 ID 和一组可选的“种子（seeds）”来查找。稍后我们会详细介绍，但这些种子将在我们使用 PDAs 进行数据存储和检索时发挥重要作用。

PDAs 具有两个主要功能：

1. 为程序提供确定性的方式来查找给定的数据项
2. 授权从中派生 PDA 的程序代表其签名，方式类似于用户使用他们的私钥签名

在本课程中，我们将重点讨论如何使用 PDAs 查找和存储数据。在将来的课程中，我们将更深入地讨论使用 PDA 签名时，涵盖跨程序调用（Cross Program Invocations，CPIs）。

## 查找 PDAs

PDAs 在技术上并非被直接创建。相反，它们是基于程序 ID 和一个或多个输入种子而被*找到*或*派生*的。

Solana 的密钥对可以在称为 Ed25519 椭圆曲线上找到。Ed25519 是 Solana 用于生成相应的公钥和私钥的确定性签名方案，总称为这些密钥对。

另外，PDAs 是位于 Ed25519 曲线*之外*的地址。这意味着 PDAs 不是公钥，并且没有私钥。PDAs 具有这种属性是程序能够代表其签署的关键，但这将在未来的课程中详细讨论。

要在 Solana 程序中查找 PDA，我们将使用 `find_program_address` 函数。该函数接受一个可选的“种子”列表和一个程序 ID 作为输入，然后返回 PDA 和一个 bump 种子（bump seed）。

```rust
let (pda, bump_seed) = Pubkey::find_program_address(&[user.key.as_ref(), user_input.as_bytes().as_ref(), "SEED".as_bytes()], program_id)
```

### 种子

“种子”是在 `find_program_address` 函数中用于派生 PDA 的可选输入。例如，种子可以是任意组合的公钥、用户提供的输入或硬编码的值。一个 PDA 也可以仅使用程序 ID 而不需要额外的种子进行派生。然而，使用种子来查找我们的 PDA 允许我们创建程序可以拥有的任意数量的账户。

虽然您作为开发者确定要传递给 `find_program_address` 函数的种子，但函数本身提供了一个额外的称为“bump 种子”。用于派生 PDA 的加密函数派生的密钥大约有 50% 的概率*位于* Ed25519 曲线上。为了确保结果*不在* Ed25519 曲线上，因此没有私钥，`find_program_address` 函数添加了一个称为碰撞种子（bump seed）的数字种子。

该函数首先使用值 `255` 作为 bump 种子，然后检查输出是否为有效的 PDA。如果结果不是有效的 PDA，则函数将 bump 种子减 1 并重试（`254`，`253`等）。一旦找到有效的 PDA，函数返回 PDA 和用于派生 PDA 的 bump 种子。

### 揭开 `find_program_address` 的内部机制

让我们来看看 `find_program_address` 的源代码。

```rust
 pub fn find_program_address(seeds: &[&[u8]], program_id: &Pubkey) -> (Pubkey, u8) {
    Self::try_find_program_address(seeds, program_id)
        .unwrap_or_else(|| panic!("Unable to find a viable program address bump seed"))
}
```

在内部，`find_program_address` 函数将输入的 `seeds` 和 `program_id` 传递给 `try_find_program_address` 函数。

`try_find_program_address` 函数引入了 `bump_seed`。`bump_seed` 是一个取值范围在 0 到 255 之间的 `u8` 变量。在从 255 开始的递减范围上进行迭代时，将 `bump_seed` 追加到可选的输入种子中，然后将其传递给 `create_program_address` 函数。如果 `create_program_address` 的输出不是有效的 PDA，那么 `bump_seed` 就会减小 1，并且循环会继续，直到找到有效的 PDA 为止。

```rust
pub fn try_find_program_address(seeds: &[&[u8]], program_id: &Pubkey) -> Option<(Pubkey, u8)> {

    let mut bump_seed = [std::u8::MAX];
    for _ in 0..std::u8::MAX {
        {
            let mut seeds_with_bump = seeds.to_vec();
            seeds_with_bump.push(&bump_seed);
            match Self::create_program_address(&seeds_with_bump, program_id) {
                Ok(address) => return Some((address, bump_seed[0])),
                Err(PubkeyError::InvalidSeeds) => (),
                _ => break,
            }
        }
        bump_seed[0] -= 1;
    }
    None

}
```

`create_program_address` 函数执行一系列哈希操作，涉及种子和 `program_id`。这些操作计算一个密钥，然后验证计算得到的密钥是否位于 Ed25519 椭圆曲线上。如果找到有效的 PDA（即一个*不在*曲线上的地址），则返回该 PDA。否则，返回一个错误。

```rust
pub fn create_program_address(
    seeds: &[&[u8]],
    program_id: &Pubkey,
) -> Result<Pubkey, PubkeyError> {

    let mut hasher = crate::hash::Hasher::default();
    for seed in seeds.iter() {
        hasher.hash(seed);
    }
    hasher.hashv(&[program_id.as_ref(), PDA_MARKER]);
    let hash = hasher.result();

    if bytes_are_curve_point(hash) {
        return Err(PubkeyError::InvalidSeeds);
    }

    Ok(Pubkey::new(hash.as_ref()))

}
```

总的来说，`find_program_address` 函数将我们的输入种子和 `program_id` 传递给 `try_find_program_address` 函数。`try_find_program_address` 函数向我们的输入种子添加一个 `bump_seed`（从255开始），然后调用 `create_program_address` 函数，直到找到有效的 PDA 为止。一旦找到，将返回找到的 PDA 和 `bump_seed`。

请注意，对于相同的输入种子，不同的有效的 bump 将生成不同的有效 PDA。`find_program_address` 返回的 `bump_seed` 将始终是找到的第一个有效 PDA。由于该函数从 `bump_seed` 值为255开始向下迭代到零，最终返回的 `bump_seed` 将始终是可能的最大有效 8 位值。这个 `bump_seed` 通常被称为 "*规范（canonical） bump*"。为避免混淆，建议仅使用规范 bump，*并始终验证传递到程序中的每个 PDA*。

需要强调的一点是，`find_program_address` 函数只返回一个由程序派生的地址和用于派生它的 `bump_seed`。`find_program_address` 函数不会初始化新账户，而且该函数返回的 PDA 未必与存储数据的账户相关联。

## 使用 PDA 账户存储数据

由于程序本身是无状态的，程序状态是通过外部账户进行管理的。鉴于您可以使用种子进行映射，并且程序可以代表自己进行签名，使用 PDA 账户来存储与程序相关的数据是一种极为常见的设计选择。虽然程序可以调用系统程序创建非 PDA 账户，并使用这些账户存储数据，但 PDA 通常是更为合适的选择。

如果您需要了解如何在 PDA 中存储数据的详细信息，请查看 [创建基础的程序，第2部分 - 状态管理课程](./program-state-management)。

## 映射到存储在 PDA 账户中的数据

在 PDA 账户中存储数据只是方程的一半。您还需要一种检索数据的方法。我们将讨论两种方法：

1. 创建一个 PDA "映射"账户，该账户存储了各种账户地址，并且指向的这些账户里面存储着数据
2. 有策略地使用种子来定位适当的 PDA 账户并检索必要的数据

### 使用 PDA “映射” 账户映射数据

一种组织数据存储的方法是将相关数据存储在它们各自的 PDA 中，然后拥有一个单独的 PDA 账户，用于存储所有数据的映射关系。

例如，您可能有一个记笔记应用程序，其支持程序使用随机种子生成 PDA 账户，并在每个账户中存储一个笔记。该程序还会有一个单一的全局 PDA “映射” 账户，用于存储用户公钥与其笔记存储位置列表之间的映射关系。此映射账户将使用静态种子（例如“GLOBAL_MAPPING”）派生而来。

当需要检索用户的笔记时，您可以查看映射账户，查看与用户公钥相关联的地址列表，然后检索每个地址对应的账户。

虽然这样的解决方案对于传统的 Web 开发人员可能更容易理解，但它确实带有一些特定于 Web3 开发的缺点。由于映射存储在映射账户中的大小会随时间增长，您要么需要在首次创建账户时分配比必要更多的空间，要么每次创建新笔记时都需要重新分配空间。除此之外，最终会达到账户大小上限，即 10 兆字节。

为了在一定程度上缓解这个问题，您可以为每个用户创建一个单独的映射账户。例如，不是为整个程序创建一个单一的 PDA 映射账户，而是为每个用户构建一个 PDA 映射账户。每个这样的映射账户可以使用用户的公钥派生。然后，每个笔记的地址可以存储在相应用户的映射账户中。

这种方法减少了每个映射账户所需的大小，但最终仍然向流程中添加了一个不必要的要求：在能够找到具有相关笔记数据的账户之前，必须 *先* 读取映射账户上的信息。

在某些情况下，使用这种方法可能对您的应用程序有意义，但我们不建议将其作为您的“首选”策略。

### 使用 PDA 派生进行数据映射

如果您在派生 PDA 时选择种子比较有策略，您可以将所需的映射嵌入到种子本身中。这是我们刚刚讨论的记笔记应用程序示例的自然演变。如果您开始使用笔记创建者的公钥作为种子为每个用户创建一个映射账户，那么为何不使用创建者的公钥和其他已知信息一起派生一个用于笔记本身的 PDA 呢？

现在，虽然我们没有明确讨论过，但在整个课程中我们一直在将种子映射到账户。想想我们在前面课程中构建的电影评论程序。该程序使用评论创建者的公钥和他们评论的电影标题来查找*应该*用于存储评论的地址。这种方法使程序能够为每个新评论创建一个唯一的地址，同时在需要时轻松找到评论。当您想要找到用户对“Spiderman”的评论时，您知道它存储在使用用户的公钥和文本“Spiderman”作为种子派生的 PDA 账户的地址上。

```rust
let (pda, bump_seed) = Pubkey::find_program_address(&[
        initializer.key.as_ref(),
        title.as_bytes().as_ref()
    ],
    program_id)
```

### 关联代币账户地址

这种映射类型的另一个实际例子是确定关联代币账户（ATA）地址的方式。代币通常存储在其地址是使用钱包地址和特定代币的铸币地址派生的关联代币账户中。通过使用 `get_associated_token_address` 函数，可以找到关联代币账户的地址，该函数以 `wallet_address` 和 `token_mint_address` 作为输入。

```rust
let associated_token_address = get_associated_token_address(&wallet_address, &token_mint_address);
```

在内部，关联的代币地址是使用 `wallet_address`、`token_program_id` 和 `token_mint_address` 作为种子找到的 PDA。这提供了一种确定性的方法，以找到与特定代币铸造相关的任何钱包地址的代币账户。

```rust
fn get_associated_token_address_and_bump_seed_internal(
    wallet_address: &Pubkey,
    token_mint_address: &Pubkey,
    program_id: &Pubkey,
    token_program_id: &Pubkey,
) -> (Pubkey, u8) {
    Pubkey::find_program_address(
        &[
            &wallet_address.to_bytes(),
            &token_program_id.to_bytes(),
            &token_mint_address.to_bytes(),
        ],
        program_id,
    )
}
```

你使用的种子和 PDA 账户之间的映射将高度依赖于您特定的程序。虽然这不是关于系统设计或架构的课程，但值得注意一些指导原则：

- 使用在 PDA 推导时需要提前知道种子
- 谨慎考虑将哪些数据组合到一个单一账户中
- 谨慎考虑每个账户内使用的数据结构
- 通常情况下，简单通常更好

# 实验

让我们一起练习一下我们在之前课程中共同完成的电影评论程序。如果您刚刚开始学习而没有完成先前的课程，也不用担心，无论哪种方式，都应该能够跟上。

作为提醒，电影评论程序允许用户创建电影评论。这些评论被存储在一个账户中，该账户是使用初始化程序的公钥和他们正在评论的电影的标题生成的 PDA。

之前，我们已经完成了以安全方式更新电影评论的功能的实现。在这个实验中，我们将添加用户对电影评论（review）进行提出意见（comment）的功能。我们将利用构建此功能的机会，深入研究如何使用 PDA 账户结构化地存储意见。

## 1. 获取起始代码

首先，您可以在 `starter` 分支上找到[电影程序的起始代码](https://github.com/Unboxed-Software/solana-movie-program/tree/starter)。

如果您一直在参与电影评论实验，您会注意到这是我们迄今为止构建的程序。之前，我们使用 [Solana Playground](https://beta.solpg.io/) 来编写、构建和部署我们的代码。在本课程中，我们将在本地构建和部署程序。

打开文件夹，然后运行 `cargo-build-bpf` 命令来构建程序。`cargo-build-bpf` 命令将输出部署程序的指令。

```sh
cargo-build-bpf
```

部署该程序，通过复制 `cargo-build-bpf` 的输出，并运行 `solana program deploy` 命令。

```sh
solana program deploy <PATH>
```

你可以使用电影评论[前端](https://github.com/Unboxed-Software/solana-movie-frontend/tree/solution-update-reviews)来测试该程序，并使用刚刚部署的程序 ID 更新。确保你使用 `solution-update-reviews` 分支。

## 2. 规划账户结构

添加意见意味着我们需要就如何存储与每条意见相关的数据做出一些决策。这里一个良好结构的标准是：

- 不要过于复杂
- 数据容易检索
- 每条评论都有与之关联的内容

为此，我们将创建两种新的账户类型：

- 意见计数器（comment counter）账户
- 意见（commet）账户

每个意见计数器账户与一个评论关联，并且每个意见账户与一个意见关联。意见计数器账户将通过使用评论的地址作为查找意见计数器 PDA 的种子来与特定的评论关联。它还将使用静态字符串 "comment" 作为种子。

意见账户将以相同的方式与评论关联。但是，它不会将 "comment" 字符串作为种子，而是将 *实际意见计数* 作为种子。这样，客户端可以通过以下方式轻松地检索给定评论的意见：

1. 读取意见计数器账户上的数据，以确定在一个评论上意见的数量。
2. 其中 `n` 是意见的总数量，循环 `n` 次。循环的每次迭代将使用评论地址和当前数字作为种子派生一个 PDA。结果是 `n` 个 PDA，每个都是存储意见的账户地址。
3. 获取每个评论 `n` 个 PDA 的意见账户，并读取其中存储的数据。

这确保了我们的每个账户都可以使用预先已知的数据被确定性地检索到。

为了实现这些更改，我们需要执行以下操作：

- 定义结构体来表示意见计数器和意见账户
- 更新现有的 `MovieAccountState` 来包含一个区分器（discriminator，稍后详细介绍）
- 添加一个指令变体来表示 `add_comment` 指令
- 更新现有的 `add_movie_review` 指令处理函数，以包括创建意见计数器账户
- 创建一个新的 `add_comment` 指令处理函数

## 3. 定义 `MovieCommentCounter` 和 `MovieComment` 结构体

回想一下，`state.rs` 文件定义了我们的程序用于填充新账户的数据字段的结构体。

为了启用对电影评论提出意见功能，我们需要定义两个新的结构体。

1. `MovieCommentCounter` - 用于存储与评论相关联的意见计数器
2. `MovieComment` - 用于存储与每条意见相关的数据

首先，让我们定义我们程序将使用的结构体。请注意，我们正在为每个结构体添加一个 `discriminator` 字段，包括现有的 `MovieAccountState`。由于我们现在有多种账户类型，我们需要一种方法来仅从客户端获取我们需要的账户类型。这个鉴别器是一个字符串，可以用来在获取程序账户时过滤账户。

```rust
#[derive(BorshSerialize, BorshDeserialize)]
pub struct MovieAccountState {
    pub discriminator: String,
    pub is_initialized: bool,
    pub reviewer: Pubkey,
    pub rating: u8,
    pub title: String,
    pub description: String,
}

#[derive(BorshSerialize, BorshDeserialize)]
pub struct MovieCommentCounter {
    pub discriminator: String,
    pub is_initialized: bool,
    pub counter: u64
}

#[derive(BorshSerialize, BorshDeserialize)]
pub struct MovieComment {
    pub discriminator: String,
    pub is_initialized: bool,
    pub review: Pubkey,
    pub commenter: Pubkey,
    pub comment: String,
    pub count: u64
}

impl Sealed for MovieAccountState {}

impl IsInitialized for MovieAccountState {
    fn is_initialized(&self) -> bool {
        self.is_initialized
    }
}

impl IsInitialized for MovieCommentCounter {
    fn is_initialized(&self) -> bool {
        self.is_initialized
    }
}

impl IsInitialized for MovieComment {
    fn is_initialized(&self) -> bool {
        self.is_initialized
    }
}
```

因为我们向现有的结构体添加了一个新的 `discriminator` 字段，所以账户大小计算需要进行更改。让我们利用这个机会稍微清理一下我们的代码。我们将为上述的三个结构体中的每一个添加一个常量 `DISCRIMINATOR` 和一个常量 `SIZE` 或函数 `get_account_size` 的实现，这样我们在初始化账户时可以快速获取所需的大小。

```rust
impl MovieAccountState {
    pub const DISCRIMINATOR: &'static str = "review";

    pub fn get_account_size(title: String, description: String) -> usize {
        return (4 + MovieAccountState::DISCRIMINATOR.len())
            + 1
            + 1
            + (4 + title.len())
            + (4 + description.len());
    }
}

impl MovieCommentCounter {
    pub const DISCRIMINATOR: &'static str = "counter";
    pub const SIZE: usize = (4 + MovieCommentCounter::DISCRIMINATOR.len()) + 1 + 8;
}

impl MovieComment {
    pub const DISCRIMINATOR: &'static str = "comment";

    pub fn get_account_size(comment: String) -> usize {
        return (4 + MovieComment::DISCRIMINATOR.len()) + 1 + 32 + 32 + (4 + comment.len()) + 8;
    }
}
```

现在，我们可以在需要使用鉴别器或账户大小的任何地方使用这个实现，而不必担心意外的拼写错误。

## 4. 创建 `AddComment` 指令

回想一下，`instruction.rs` 文件定义了我们的程序将接受的指令以及如何为每个指令反序列化数据。我们需要为添加意见添加一个新的指令变体。让我们首先将一个新的变体 `AddComment` 添加到 `MovieInstruction` 枚举中。

```rust
pub enum MovieInstruction {
    AddMovieReview {
        title: String,
        rating: u8,
        description: String
    },
    UpdateMovieReview {
        title: String,
        rating: u8,
        description: String
    },
    AddComment {
        comment: String
    }
}
```

接下来，让我们创建一个 `CommentPayload` 结构体来表示与这个新指令相关的指令数据。我们将包含在账户中的大部分数据，都是与传递到程序中的账户相关的公钥，所以我们实际上只需要一个字段来表示评论文本。

```rust
#[derive(BorshDeserialize)]
struct CommentPayload {
    comment: String
}
```

现在让我们更新如何解包指令数据。请注意，我们已经将指令数据的反序列化移到了每个匹配的情况中，使用了每个指令的关联 payload 结构体。

```rust
impl MovieInstruction {
    pub fn unpack(input: &[u8]) -> Result<Self, ProgramError> {
        let (&variant, rest) = input.split_first().ok_or(ProgramError::InvalidInstructionData)?;
        Ok(match variant {
            0 => {
                let payload = MovieReviewPayload::try_from_slice(rest).unwrap();
                Self::AddMovieReview {
                title: payload.title,
                rating: payload.rating,
                description: payload.description }
            },
            1 => {
                let payload = MovieReviewPayload::try_from_slice(rest).unwrap();
                Self::UpdateMovieReview {
                    title: payload.title,
                    rating: payload.rating,
                    description: payload.description
                }
            },
            2 => {
                let payload = CommentPayload::try_from_slice(rest).unwrap();
                Self::AddComment {
                    comment: payload.comment
                }
            }
            _ => return Err(ProgramError::InvalidInstructionData)
        })
    }
}
```

最后，让我们更新 `processor.rs` 中的 `process_instruction` 函数，以使用我们创建的新指令变体。

在 `processor.rs` 中，将来自 `state.rs` 的新结构引入范围。

```rust
use crate::state::{MovieAccountState, MovieCommentCounter, MovieComment};
```

然后在 `process_instruction` 函数中，让我们将反序列化的 `AddComment` 指令数据与我们即将实现的 `add_comment` 函数进行匹配。

```rust
pub fn process_instruction(
    program_id: &Pubkey,
    accounts: &[AccountInfo],
    instruction_data: &[u8]
) -> ProgramResult {
    let instruction = MovieInstruction::unpack(instruction_data)?;
    match instruction {
        MovieInstruction::AddMovieReview { title, rating, description } => {
            add_movie_review(program_id, accounts, title, rating, description)
        },
        MovieInstruction::UpdateMovieReview { title, rating, description } => {
            update_movie_review(program_id, accounts, title, rating, description)
        },

        MovieInstruction::AddComment { comment } => {
            add_comment(program_id, accounts, comment)
        }
    }
}
```

## 5. 更新 `add_movie_review` 来创建意见计数器账户

在我们实现 `add_comment` 函数之前，我们需要更新 `add_movie_review` 函数来创建意见计数器账户。

请记住，该账户将跟踪与关联的电影评论相关的意见总数。它的地址将使用电影评论地址和单词 “comment” 作为种子派生而来。请注意，我们存储计数器的方式仅是一种设计选择。我们也可以在原始电影评论账户中添加一个 “counter” 字段。

在 `add_movie_review` 函数内部，让我们添加一个 `pda_counter` 来表示我们将初始化的新计数器账户，以及电影评论账户。这意味着我们现在通过 `accounts` 参数希望将四个账户传递给 `add_movie_review` 函数。

```rust
let account_info_iter = &mut accounts.iter();

let initializer = next_account_info(account_info_iter)?;
let pda_account = next_account_info(account_info_iter)?;
let pda_counter = next_account_info(account_info_iter)?;
let system_program = next_account_info(account_info_iter)?;
```

接下来，有一个检查，确保 `total_len` 小于 1000 字节，但是 `total_len` 不再准确，因为我们添加了鉴别器。让我们用一个调用 `MovieAccountState::get_account_size` 来替换 `total_len`：

```rust
let account_len: usize = 1000;

if MovieAccountState::get_account_size(title.clone(), description.clone()) > account_len {
    msg!("Data length is larger than 1000 bytes");
    return Err(ReviewError::InvalidDataLength.into());
}
```

注意，这也需要在 `update_movie_review` 函数中进行更新，以使该指令正常工作。

一旦我们初始化了评论账户，我们还需要使用 `MovieAccountState` 结构中指定的新字段更新 `account_data`。

```rust
account_data.discriminator = MovieAccountState::DISCRIMINATOR.to_string();
account_data.reviewer = *initializer.key;
account_data.title = title;
account_data.rating = rating;
account_data.description = description;
account_data.is_initialized = true;
```

最后，让我们在 `add_movie_review` 函数中添加初始化计数器账户的逻辑。这意味着：

1. 计算计数器账户免的租金金额
2. 使用评论地址和字符串 "comment" 作为种子派生计数器 PDA
3. 调用系统程序来创建账户
4. 设置初始计数器值
5. 序列化账户数据并从函数中返回

所有这些都应该在 `Ok(())` 之前添加到 `add_movie_review` 函数的末尾。

```rust
msg!("create comment counter");
let rent = Rent::get()?;
let counter_rent_lamports = rent.minimum_balance(MovieCommentCounter::SIZE);

let (counter, counter_bump) =
    Pubkey::find_program_address(&[pda.as_ref(), "comment".as_ref()], program_id);
if counter != *pda_counter.key {
    msg!("Invalid seeds for PDA");
    return Err(ProgramError::InvalidArgument);
}

invoke_signed(
    &system_instruction::create_account(
        initializer.key,
        pda_counter.key,
        counter_rent_lamports,
        MovieCommentCounter::SIZE.try_into().unwrap(),
        program_id,
    ),
    &[
        initializer.clone(),
        pda_counter.clone(),
        system_program.clone(),
    ],
    &[&[pda.as_ref(), "comment".as_ref(), &[counter_bump]]],
)?;
msg!("comment counter created");

let mut counter_data =
    try_from_slice_unchecked::<MovieCommentCounter>(&pda_counter.data.borrow()).unwrap();

msg!("checking if counter account is already initialized");
if counter_data.is_initialized() {
    msg!("Account already initialized");
    return Err(ProgramError::AccountAlreadyInitialized);
}

counter_data.discriminator = MovieCommentCounter::DISCRIMINATOR.to_string();
counter_data.counter = 0;
counter_data.is_initialized = true;
msg!("comment count: {}", counter_data.counter);
counter_data.serialize(&mut &mut pda_counter.data.borrow_mut()[..])?;
```

现在，当创建新评论时，会初始化两个账户：

1. 第一个是存储评论内容的评论账户。这与我们开始时的程序版本没有变化。
2. 第二个账户存储意见计数器

## 6. 实现 `add_comment`

最后，让我们实现我们的 `add_comment` 函数来创建新的意见账户。

当为一条评论创建新意见时，我们将递增意见计数器 PDA 账户上的计数，并使用评论地址和当前计数派生评论账户的 PDA。

与其他指令处理函数一样，我们将从传递到程序中的账户中进行迭代。然后，在我们执行任何其他操作之前，我们需要反序列化计数器账户，这样我们就可以访问当前的评论计数：

```rust
pub fn add_comment(
    program_id: &Pubkey,
    accounts: &[AccountInfo],
    comment: String
) -> ProgramResult {
    msg!("Adding Comment...");
    msg!("Comment: {}", comment);

    let account_info_iter = &mut accounts.iter();

    let commenter = next_account_info(account_info_iter)?;
    let pda_review = next_account_info(account_info_iter)?;
    let pda_counter = next_account_info(account_info_iter)?;
    let pda_comment = next_account_info(account_info_iter)?;
    let system_program = next_account_info(account_info_iter)?;

    let mut counter_data = try_from_slice_unchecked::<MovieCommentCounter>(&pda_counter.data.borrow()).unwrap();

    Ok(())
}
```

现在我们可以继续进行剩余的步骤：

1. 计算新意见账户的免租金金额
2. 使用评论地址和当前意见计数作为种子派生评论账户的 PDA
3. 调用系统程序来创建新的意见账户
4. 设置新创建的账户的适当值
5. 序列化账户数据并从函数返回

```rust
pub fn add_comment(
    program_id: &Pubkey,
    accounts: &[AccountInfo],
    comment: String
) -> ProgramResult {
    msg!("Adding Comment...");
    msg!("Comment: {}", comment);

    let account_info_iter = &mut accounts.iter();

    let commenter = next_account_info(account_info_iter)?;
    let pda_review = next_account_info(account_info_iter)?;
    let pda_counter = next_account_info(account_info_iter)?;
    let pda_comment = next_account_info(account_info_iter)?;
    let system_program = next_account_info(account_info_iter)?;

    let mut counter_data = try_from_slice_unchecked::<MovieCommentCounter>(&pda_counter.data.borrow()).unwrap();

    let account_len = MovieComment::get_account_size(comment.clone());

    let rent = Rent::get()?;
    let rent_lamports = rent.minimum_balance(account_len);

    let (pda, bump_seed) = Pubkey::find_program_address(&[pda_review.key.as_ref(), counter_data.counter.to_be_bytes().as_ref(),], program_id);
    if pda != *pda_comment.key {
        msg!("Invalid seeds for PDA");
        return Err(ReviewError::InvalidPDA.into())
    }

    invoke_signed(
        &system_instruction::create_account(
        commenter.key,
        pda_comment.key,
        rent_lamports,
        account_len.try_into().unwrap(),
        program_id,
        ),
        &[commenter.clone(), pda_comment.clone(), system_program.clone()],
        &[&[pda_review.key.as_ref(), counter_data.counter.to_be_bytes().as_ref(), &[bump_seed]]],
    )?;

    msg!("Created Comment Account");

    let mut comment_data = try_from_slice_unchecked::<MovieComment>(&pda_comment.data.borrow()).unwrap();

    msg!("checking if comment account is already initialized");
    if comment_data.is_initialized() {
        msg!("Account already initialized");
        return Err(ProgramError::AccountAlreadyInitialized);
    }

    comment_data.discriminator = MovieComment::DISCRIMINATOR.to_string();
    comment_data.review = *pda_review.key;
    comment_data.commenter = *commenter.key;
    comment_data.comment = comment;
    comment_data.is_initialized = true;
    comment_data.serialize(&mut &mut pda_comment.data.borrow_mut()[..])?;

    msg!("Comment Count: {}", counter_data.counter);
    counter_data.counter += 1;
    counter_data.serialize(&mut &mut pda_counter.data.borrow_mut()[..])?;

    Ok(())
}
```

## 7. 构建和部署

我们已经准备好构建和部署我们的程序了！

通过运行 `cargo-build-bpf` 来构建更新的程序。然后通过运行在控制台打印的 `solana program deploy` 命令来部署程序。

你可以通过提交具有正确指令数据的交易来测试你的程序。你可以创建自己的脚本，或者随意使用[此前端](https://github.com/Unboxed-Software/solana-movie-frontend/tree/solution-add-comments)。请确保使用 `solution-add-comments` 分支，并在 `utils/constants.ts` 中用你程序的ID替换 `MOVIE_REVIEW_PROGRAM_ID`，否则前端将无法与你的程序一起使用。

请注意，我们对评论账户进行了重大更改（即添加了一个鉴别器）。如果您在部署此程序时使用了之前使用过的相同程序ID，则由于数据不匹配，之前创建的所有评论都不会显示在此前端中。

如果你需要更多时间来熟悉这些概念，请在继续之前查看[解决方案代码](https://github.com/Unboxed-Software/solana-movie-program/tree/solution-add-comments)。请注意，链接存储库的解决方案代码位于所链接存储库的 `solution-add-comments` 分支。

# 挑战

现在轮到你独立构建一些东西了！继续使用我们在过去课程中使用过的学生介绍程序。学生介绍程序是一个 Solana 程序，它允许学生介绍自己。该程序将用户的姓名和简短消息作为 `instruction_data`，并在链上创建一个账户来存储数据。对于这个挑战，你应该：

1. 添加一条指令，允许其他用户回复介绍。
2. 在本地构建和部署该程序。

如果你没有跟随过去的课程，或者没有保存之前的工作，请随时使用[此存储库](https://github.com/Unboxed-Software/solana-student-intro-program/tree/starter)的 `starter` 分支上的起始代码。

如果可能的话，请尝试独立完成！但是如果你遇到困难，请随时参考[解决方案代码](https://github.com/Unboxed-Software/solana-student-intro-program/tree/solution-add-replies)。请注意，解决方案代码位于 `solution-add-replies` 分支，你的代码可能会略有不同。

## 实验完成了吗？

将您的代码推送到 GitHub，并告诉我们您对这节课的看法！ [点击此处填写反馈表单](https://form.typeform.com/to/IPH0UGz7#answers-lesson=89d367b4-5102-4237-a7f4-4f96050fe57e)！