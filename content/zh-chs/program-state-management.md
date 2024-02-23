---
title: 创建基本程序，第2部分 - 状态管理
objectives:
- 描述使用程序派生地址（PDA）创建新账户的过程
- 使用种子来派生 PDA
- 使用账户所需的空间来计算用户必须分配的租金数量（以 lamports 计）
- 使用跨程序调用（CPI）将 PDA 作为新账户的地址初始化账户
- 解释如何更新存储在新账户上的数据
---

# 总结

- 程序状态存储在其他账户中，而不是在程序本身中。
- 程序派生地址（PDA）是从程序 ID 和一个可选的种子列表派生而来的。一旦派生完成，PDAs 随后被用作存储账户的地址。
- 创建一个账户需要计算所需的空间以及分配给新账户的相应租金。
- 创建一个新账户需要通过对系统程序上的 `create_account` 指令进行跨程序调用（CPI）。
- 更新账户上的数据字段需要将数据序列化（转换为字节数组）到账户中。

# 概述

Solana 通过使程序无状态（stateless）来保持速度、效率和可扩展性。程序不会将状态存储在程序本身上，而是使用 Solana 的账户模型从单独的 PDA 账户中读取状态并写入状态。

虽然这是一个非常灵活的模型，但如果不熟悉的话，这也是一个可能难以使用的范式。但别担心！在这个课程中，我们将从简单的开始，逐步深入到下一个单元中更复杂的程序。

在本课程中，我们将学习 Solana 程序的状态管理基础知识，包括将状态表示为 Rust 类型、使用程序派生地址创建账户以及序列化账户数据。

## 程序状态

所有的 Solana 账户都有一个 `data` 字段，其中保存着一个字节数组。这使得账户像计算机上的文件一样灵活。你可以在一个账户中存储任何东西（只要账户有足够的存储空间）。

就像传统文件系统中的文件遵循特定的数据格式，比如 PDF 或 MP3，存储在 Solana 账户中的数据也需要遵循某种模式，以便可以检索和反序列化成可用的内容。

### 将状态表示为 Rust 类型

在 Rust 中编写程序时，我们通常通过定义一个 Rust 数据类型来创建这种“格式（format）”。如果你已经完成了[本课程的第一部分](./deserialize-instruction-data.md)，那么这与我们创建枚举来表示离散指令时非常相似。

虽然这种类型应该反映你的数据结构，但对于大多数用例来说，一个简单的结构体就足够了。例如，一个将笔记存储在单独账户中的记笔记程序可能会有标题、正文和可能某种 ID 的数据。我们可以创建一个结构体来表示如下：

```rust
struct NoteState {
    title: String,
    body: String,
    id: u64
}
```

### 使用 Borsh 进行序列化和反序列化

就像对指令数据一样，我们需要一种机制来将 Rust 数据类型转换为字节数组，反之亦然。**序列化（Serialization）**是将对象转换为字节数组的过程。**反序列化（Deserialization）**是从字节数组重建对象的过程。

我们将继续使用 Borsh 进行序列化和反序列化。在 Rust 中，我们可以使用 `borsh` crate 来获取 `BorshSerialize` 和 `BorshDeserialize` traits。然后，我们可以使用 `derive` 属性宏来应用这些 traits。

```rust
use borsh::{BorshSerialize, BorshDeserialize};

#[derive(BorshSerialize, BorshDeserialize)]
struct NoteState {
    title: String,
    body: String,
    id: u64
}
```

这些 traits 将为 `NoteState` 提供我们可以根据需要用来序列化和反序列化数据的方法。

## 创建账户

在我们能够更新账户的数据字段之前，我们必须首先创建该账户。

要在我们的程序中创建一个新账户，我们必须：

1. 计算账户所需的空间和租金
2. 有一个地址来分配给新账户
3. 调用系统程序（system program）来创建新账户

### 空间和租金

回想一下，在 Solana 网络上存储数据需要用户分配租金，以 lamports 的形式。新账户所需的租金数量取决于您希望为该账户分配的空间量。这意味着在创建账户之前，我们需要知道要分配多少空间。

需要注意的是，租金更像是一笔押金。当关闭一个账户时，分配给租金的所有 lamports 都可以完全退还。此外，现在所有新账户都需要是[免租金（rent-exempt）](https://twitter.com/jacobvcreech/status/1524790032938287105)，这意味着不会随着时间的推移从账户中扣除 lamports。如果账户持有至少 2 年的租金，该账户就被视为免租金。换句话说，账户将永久存储在链上，直到所有者（owner）关闭账户并提取租金。

在我们的记笔记应用示例中，`NoteState` 结构指定了需要存储在账户中的三个字段：`title`、`body` 和 `id`。要计算账户需要的大小，您只需将每个字段中存储数据所需的大小相加即可。

对于动态数据，比如字符串，Borsh 在开头会额外添加 4 字节来存储该特定字段的长度。这意味着 `title` 和 `body` 分别是 4 字节加上它们各自的大小。`id` 字段是一个 64 位整数，或者 8 字节。

您可以将这些长度相加，然后使用 `solana_program` crate 的 `rent` 模块中的 `minimum_balance` 函数计算所需空间的租金。

```rust
// Calculate account size required for struct NoteState
let account_len: usize = (4 + title.len()) + (4 + body.len()) + 8;

// Calculate rent required
let rent = Rent::get()?;
let rent_lamports = rent.minimum_balance(account_len);
```

### 程序派生地址（PDA）

在创建账户之前，我们还需要有一个地址来分配给该账户。对于程序能够拥有所有权的账户来说（即账户的 owner 字段为自己的程序地址），这将是使用 `find_program_address` 函数找到的程序派生地址（PDA）。

正如其名称所示，PDAs 是使用程序 ID（创建账户的程序的地址）和一个可选的“种子（seeds）”列表派生而来的。可选的种子是在 `find_program_address` 函数中使用的额外输入，用于派生 PDA。用于派生 PDAs 的函数在给定相同的输入时，每次都会返回相同的地址。这使我们能够创建任意数量的 PDA 账户，并以确定性的方式找到每个账户。

除了您提供用于派生 PDA 的种子之外，`find_program_address` 函数还会提供一个额外的“增量种子（bump seed）”。使 PDAs 与其他 Solana 账户地址不同的是，它们没有相应的私钥。这确保只有拥有该地址的程序才能代表 PDA 进行签名。当 `find_program_address` 函数尝试使用提供的种子派生 PDA 时，它会将数字 255 作为“增量种子”传入。如果生成的地址无效（即具有相应的私钥），则该函数会将增量种子减少 1，并使用新的增量种子派生新的 PDA。一旦找到有效的 PDA，该函数就会返回 PDA 和用于派生 PDA 的增量（bump）。

对于我们的记笔记应用程序，我们将使用笔记创建者的公钥和 ID 作为可选的种子来派生 PDA。通过这种方式派生 PDA，我们可以确定性地找到每个笔记的账户。

```rust
let (note_pda_account, bump_seed) = Pubkey::find_program_address(&[note_creator.key.as_ref(), id.as_bytes().as_ref(),], program_id);
```

### 跨程序调用（CPI）

一旦我们计算出了账户所需的租金，并找到了一个有效的 PDA 作为新账户的地址，我们最终可以创建该账户了。在我们的程序中创建一个新账户需要进行跨程序调用（Cross Program Invocation，CPI）。CPI 是一个程序调用另一个程序上的指令的过程。要在我们的程序中创建一个新账户，我们将在系统程序上调用 `create_account` 指令。

CPI 可以使用 `invoke` 或 `invoke_signed` 来完成。

```rust
pub fn invoke(
    instruction: &Instruction,
    account_infos: &[AccountInfo<'_>]
) -> ProgramResult
```

```rust
pub fn invoke_signed(
    instruction: &Instruction,
    account_infos: &[AccountInfo<'_>],
    signers_seeds: &[&[&[u8]]]
) -> ProgramResult
```

在本课程中，我们将使用 `invoke_signed`。与常规签名不同，常规签名使用私钥进行签名，而 `invoke_signed` 使用可选的种子、增量种子和程序 ID 来派生一个 PDA 并签署一条指令。这是通过将派生的 PDA 与传递给指令的所有账户进行比较来完成的。如果其中的账户与 PDA 匹配，则该账户的签名字段将被设置为 true。

程序可以通过这种方式安全地签署交易，因为 `invoke_signed` 在调用的过程中，runtime 会重新根据种子（可选种子和增量种子）和调用程序的 program ID 重新派生 PDA，如果账户与 [account_info](https://docs.rs/solana-program/latest/solana_program/program/fn.invoke_signed.html) 中的一个账户匹配，那么将认为该用户已经“签名”（signed）。因此，一个程序无法生成一个与使用另一个程序 ID 派生的 PDA 对应的匹配 PDA，以便为账户签署。

```rust
invoke_signed(
    // instruction
    &system_instruction::create_account(
        note_creator.key,
        note_pda_account.key,
        rent_lamports,
        account_len.try_into().unwrap(),
        program_id,
    ),
    // account_infos
    &[note_creator.clone(), note_pda_account.clone(), system_program.clone()],
    // signers_seeds
    &[&[note_creator.key.as_ref(), note_id.as_bytes().as_ref(), &[bump_seed]]],
)?;
```

## 序列化和反序列化账户数据

一旦我们创建了一个新账户，我们就需要访问并更新该账户的数据字段。这意味着将其字节数组反序列化为我们创建的类型的实例，更新该实例上的字段，然后将该实例重新序列化为字节数组。

### 反序列化账户数据

更新账户数据的第一步是将其 `data` 字节数组反序列化为其 Rust 类型。您可以通过首先借用账户上的数据字段来实现这一点。这样可以让您在不获取所有权的情况下访问数据。

然后，您可以使用 `try_from_slice_unchecked` 函数，使用您创建的表示数据的类型的格式，来反序列化借用账户的数据字段。这将为您提供一个您的 Rust 类型的实例，以便您可以使用点符号轻松更新字段。如果我们要使用我们一直在使用的记事应用示例进行操作，代码将如下所示：

```rust
let mut account_data = try_from_slice_unchecked::<NoteState>(note_pda_account.data.borrow()).unwrap();

account_data.title = title;
account_data.body = rating;
account_data.id = id;
```

### 序列化账户数据

一旦表示账户数据的 Rust 实例已经使用适当的值进行更新，您就可以将数据“保存”在账户上。

这是通过您创建的 Rust 类型实例上的 `serialize` 函数来完成的。您需要传递一个对账户数据的可变引用。这里的语法有点复杂，如果您不完全理解也不要担心。借用和引用是 Rust 中最难的概念之一。

```rust
account_data.serialize(&mut &mut note_pda_account.data.borrow_mut()[..])?;
```

上述示例将 `account_data` 对象转换为字节数组，并将其设置为 `note_pda_account` 的 `data` 属性。这样就将更新后的 `account_data` 变量保存到了新账户的数据字段中。现在，当用户获取 `note_pda_account` 并对数据进行反序列化时，将显示我们已序列化到账户中的更新数据。

## 迭代器

在前面的示例中，您可能已经注意到我们引用了 `note_creator`，但没有显示它是从哪里来的。

为了访问这个以及其他账户，我们使用了一个[迭代器（Iterator）](https://doc.rust-lang.org/std/iter/trait.Iterator.html)。迭代器是 Rust 中用于顺序访问值集合中每个元素的 trait。在 Solana 程序中，迭代器被用于安全地迭代传递到程序入口点的账户列表，这是通过 `accounts` 参数传入的。

### Rust 迭代器

迭代器模式允许您对一系列元素执行某些任务。`iter()` 方法创建一个引用集合的迭代器对象。迭代器负责迭代每个元素并确定序列何时结束的逻辑。在 Rust 中，迭代器是惰性的，这意味着在调用消耗迭代器的方法之前，它们不会产生任何效果。一旦创建了迭代器，您必须调用 `next()` 函数来获取下一个项。

```rust
let v1 = vec![1, 2, 3];

// create the iterator over the vec
let v1_iter = v1.iter();

// use the iterator to get the first item
let first_item = v1_iter.next();

// use the iterator to get the second item
let second_item = v1_iter.next();
```

### Solana 账户迭代器

回想一下，所有指令所需的 `AccountInfo` 都通过单个 `accounts` 参数传递。为了解析这些账户并在我们的指令中使用它们，我们需要创建一个具有对 `accounts` 的可变引用的迭代器。

在这一点上，我们不直接使用迭代器，而是将其传递给由 `solana_program` crate 提供的 `account_info` 模块中的 `next_account_info` 函数。

例如，一个记笔记程序中创建新笔记的指令至少需要包括创建笔记的用户、用于存储笔记的 PDA，以及 `system_program` 来初始化新账户的账户。所有这三个账户都通过 `accounts` 参数传递到程序入口点。然后，使用 `accounts` 的迭代器来分离与每个账户相关联的 `AccountInfo`，以处理指令。

请注意，`&mut` 表示对 `accounts` 参数的可变引用。您可以阅读有关 [Rust 中的引用](https://doc.rust-lang.org/book/ch04-02-references-and-borrowing.html) 和 [mut 关键字](https://doc.rust-lang.org/std/keyword.mut.html) 的更多信息。

```rust
// Get Account iterator
let account_info_iter = &mut accounts.iter();

// Get accounts
let note_creator = next_account_info(account_info_iter)?;
let note_pda_account = next_account_info(account_info_iter)?;
let system_program = next_account_info(account_info_iter)?;
```

# 实验

本概述涵盖了许多新概念。让我们通过继续改进上一课程中的电影评论程序来共同练习它们。如果您刚刚开始学习本课程而没有完成上一课程，也不用担心，您应该可以跟着学习。我们将使用 [Solana Playground](https://beta.solpg.io) 来编写、构建和部署我们的代码。

作为复习，我们正在构建一个 Solana 程序，让用户对电影进行评论。在上一课中，我们反序列化了用户传入的指令数据，但我们尚未将这些数据存储在账户中。现在让我们更新我们的程序，创建新账户来存储用户的电影评论。

## 1. 获取起始代码

如果您没有完成上一课的实验室，或者只是想确保您没有遗漏任何内容，您可以参考[起始代码](https://beta.solpg.io/6295b25b0e6ab1eb92d947f7)。

我们的程序目前包括 `instruction.rs` 文件，我们用它来反序列化传递给程序入口点的 `instruction_data`。我们还完成了 `lib.rs` 文件，以至于我们可以使用 `msg!` 宏将反序列化的指令数据打印到程序日志中。

## 2. 创建表示账户数据的结构体

让我们首先创建一个名为 `state.rs` 的新文件。

这个文件将会：

1. 定义我们的程序用来填充新账户的数据字段的结构体
2. 为这个结构体添加 `BorshSerialize` 和 `BorshDeserialize` 特性

首先，让我们从 `borsh` crate 中引入我们需要的所有内容。

```rust
use borsh::{BorshSerialize, BorshDeserialize};
```

接下来，让我们创建我们的 `MovieAccountState` 结构体。这个结构体将定义每个新电影评论账户在其数据字段中存储的参数。我们的 `MovieAccountState` 结构体将需要以下参数：

- `is_initialized` - 显示账户是否已初始化
- `rating` - 用户对电影的评分
- `description` - 用户对电影的描述
- `title` - 用户正在评论的电影的标题

```rust
#[derive(BorshSerialize, BorshDeserialize)]
pub struct MovieAccountState {
    pub is_initialized: bool,
    pub rating: u8,
    pub title: String,
    pub description: String  
}
```

## 3. 更新 `lib.rs`

接下来，让我们更新我们的 `lib.rs` 文件。首先，我们将引入我们需要完成电影评论程序的所有内容。您可以从 [solana_program crate 的文档](https://docs.rs/solana-program/latest/solana_program/) 中了解我们正在使用的每个引用项目的详细信息。

```rust
use solana_program::{
    entrypoint,
    entrypoint::ProgramResult,
    pubkey::Pubkey,
    msg,
    account_info::{next_account_info, AccountInfo},
    system_instruction,
    program_error::ProgramError,
    sysvar::{rent::Rent, Sysvar},
    program::{invoke_signed},
    borsh::try_from_slice_unchecked,
};
use std::convert::TryInto;
pub mod instruction;
pub mod state;
use instruction::MovieInstruction;
use state::MovieAccountState;
use borsh::BorshSerialize;
```

## 4. 遍历 `accounts`

接下来，让我们继续构建我们的 `add_movie_review` 函数。回想一下，一个账户数组通过单个 `accounts` 参数传递到 `add_movie_review` 函数中。为了处理我们的指令，我们需要遍历（iterate） `accounts` 并将每个账户的 `AccountInfo` 分配给它自己的变量。

```rust
// Get Account iterator
let account_info_iter = &mut accounts.iter();

// Get accounts
let initializer = next_account_info(account_info_iter)?;
let pda_account = next_account_info(account_info_iter)?;
let system_program = next_account_info(account_info_iter)?;
```

## 5. 派生 PDA

接下来，在我们的 `add_movie_review` 函数中，让我们独立地派生我们期望用户传入的 PDA。我们将在稍后提供派生的增量种子，所以即使 `pda_account` 应该引用相同的账户，我们仍然需要调用 `find_program_address`。

请注意，我们使用初始化程序的公钥和电影标题作为可选种子来为每个新账户派生 PDA。以这种方式设置 PDA 限制了每个用户仅能为任何一个电影标题撰写一次评论。但是，它仍允许同一用户评论具有不同标题的电影，也允许不同用户评论具有相同标题的电影。

```rust
// Derive PDA
let (pda, bump_seed) = Pubkey::find_program_address(&[initializer.key.as_ref(), title.as_bytes().as_ref(),], program_id);
```

## 6. 计算空间和租金

接下来，让我们计算我们的新账户需要的租金。回想一下，租金是用户必须分配给 Solana 网络上的一个账户以存储数据的 lamports 数量。为了计算租金，我们必须首先计算我们的新账户需要的空间量。

`MovieAccountState` 结构体有四个字段。我们将为 `rating` 和 `is_initialized` 每个分配 1 个字节的空间。对于 `title` 和 `description`，我们将分配空间，等于 4 字节加上字符串的长度。

```rust
// Calculate account size required
let account_len: usize = 1 + 1 + (4 + title.len()) + (4 + description.len());

// Calculate rent required
let rent = Rent::get()?;
let rent_lamports = rent.minimum_balance(account_len);
```

## 7. 创建新账户

一旦我们计算出了租金并验证了 PDA，我们就可以准备创建我们的新账户了。为了创建一个新账户，我们必须从系统程序调用 `create_account` 指令。我们使用跨程序调用（CPI）来完成这个操作，使用 `invoke_signed` 函数。我们使用 `invoke_signed` 是因为我们正在使用 PDA 创建账户，并且需要电影评论程序“签署”指令。

```rust
// Create the account
invoke_signed(
    &system_instruction::create_account(
        initializer.key,
        pda_account.key,
        rent_lamports,
        account_len.try_into().unwrap(),
        program_id,
    ),
    &[initializer.clone(), pda_account.clone(), system_program.clone()],
    &[&[initializer.key.as_ref(), title.as_bytes().as_ref(), &[bump_seed]]],
)?;

msg!("PDA created: {}", pda);
```

## 8. 更新账户数据

现在我们已经创建了一个新账户，我们准备使用来自我们的 `state.rs` 文件中 `MovieAccountState` 结构体的格式来更新新账户的数据字段。我们首先使用 `try_from_slice_unchecked` 从 `pda_account` 反序列化账户数据，然后设置每个字段的值。

```rust
msg!("unpacking state account");
let mut account_data = try_from_slice_unchecked::<MovieAccountState>(&pda_account.data.borrow()).unwrap();
msg!("borrowed account data");

account_data.title = title;
account_data.rating = rating;
account_data.description = description;
account_data.is_initialized = true;
```

最后，我们将更新后的 `account_data` 序列化到我们的 `pda_account` 的数据字段中。

```rust
msg!("serializing account");
account_data.serialize(&mut &mut pda_account.data.borrow_mut()[..])?;
msg!("state account serialized");
```

## 9. 构建和部署

我们已经准备好构建和部署我们的程序了！

![Gif Build and Deploy Program](../../assets/movie-review-pt2-build-deploy.gif)

你可以通过提交带有正确指令数据的交易来测试你的程序。为此，你可以使用我们在[反序列化程序数据课程](./deserialize-custom-data.md)中构建的[此脚本](https://github.com/Unboxed-Software/solana-movie-client)或[前端](https://github.com/Unboxed-Software/solana-movie-frontend)。在这两种情况下，请确保将你的程序的程序 ID 复制并粘贴到源代码的适当位置，以确保你测试的是正确的程序。

如果你使用前端，请在 `MovieList.tsx` 和 `Form.tsx` 组件中将 `MOVIE_REVIEW_PROGRAM_ID` 替换为你部署的程序的地址。然后运行前端，提交一份评论，并刷新浏览器以查看评论。

如果你需要更多时间来熟悉这些概念，可以先查看[解决方案代码](https://beta.solpg.io/62b23597f6273245aca4f5b4)。

# 挑战

现在轮到你独立构建一些东西了。在本课程中介绍的概念的支持下，你现在已经掌握了重新创建 Module 1 中的整个学生介绍程序所需的一切。

学生介绍程序是一个 Solana 程序，允许学生介绍自己。该程序将用户的姓名和简短消息作为 `instruction_data`，并创建一个账户将数据存储在链上。

利用本课程中学到的知识，构建这个程序。除了接受姓名和简短消息作为指令数据之外，该程序还应该：

1. 为每个学生创建一个单独的账户
2. 在每个账户中存储 `is_initialized` 作为布尔值，`name` 作为字符串，`msg` 作为字符串

你可以通过构建我们在 [分页，排序和筛选程序数据课程](./paging-ordering-filtering-data.md) 中创建的[前端](https://github.com/Unboxed-Software/solana-student-intros-frontend) 来测试你的程序。记得用你部署的程序 ID 替换前端代码中的 ID。

如果可能的话，尽量独立完成！但如果遇到困难，可以参考[解决方案代码](https://beta.solpg.io/62b11ce4f6273245aca4f5b2)。

## 完成实验了吗？

将你的代码推送到 GitHub，并[告诉我们你对本课程的看法](https://form.typeform.com/to/IPH0UGz7#answers-lesson=8320fc87-2b6d-4b3a-8b1a-54b55afed781)！