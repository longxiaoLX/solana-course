---
title: 创建基础程序，第 1 部分 - 处理指令数据
objectives:
- 在 Rust 中分配可变和不可变变量
- 创建并使用 Rust 结构体和枚举
- 使用 Rust 匹配语句
- 为 Rust 类型添加实现
- 将指令数据反序列化为 Rust 数据类型
- 对不同类型的指令执行不同的程序逻辑
- 解释 Solana 上智能合约的结构
---

# 总结

- 大多数程序支持**多个指令** - 在编写程序时，您可以决定这些指令是什么，以及必须伴随它们的数据是什么
- Rust **枚举**通常用于表示不同的程序指令
- 您可以使用 `borsh` crate 和 `derive` 属性为 Rust 结构体提供 Borsh 反序列化和序列化功能
- Rust `match` 表达式有助于根据提供的指令创建条件代码路径

# 概述

处理指令数据是 Solana 程序中最基本的要素之一。大多数程序支持多个相关功能，并使用指令数据的差异来确定要执行哪条代码路径。例如，传递给程序的指令数据中的两种不同数据格式可能表示创建新数据与删除相同数据的指令。

由于指令数据以字节数组的形式提供给程序的入口点，通常会创建一个 Rust 数据类型来表示指令，以便在整个代码中更方便地使用。本课程将介绍如何设置这种类型，如何将指令数据反序列化为此格式，并根据传递给程序入口点的指令执行适当的代码路径。

## Rust 基础知识

在深入讨论基本的 Solana 程序之前，让我们先了解一下在本课程中将要使用的 Rust 基础知识。

### 变量

在 Rust 中，变量（variable）赋值使用 `let` 关键字。

```rust
let age = 33;
```

在 Rust 中，默认情况下变量是不可变的，这意味着一旦设置了变量的值，就无法更改。为了创建一个我们希望在将来某个时候能够更改的变量，我们使用 `mut` 关键字。使用这个关键字定义的变量意味着其中存储的值是可以改变的。

```rust
// compiler will throw error
let age = 33;
age = 34;

// this is allowed
let mut mutable_age = 33;
mutable_age = 34;
```

Rust 编译器保证不可变变量确实不能更改，这样你就不必自己跟踪它。这使得你的代码更容易理解，并简化了调试过程。

### 结构体

结构体（Struct）是一种自定义数据类型，它允许你将多个相关的数值打包在一起，并为其命名，形成一个有意义的组。结构体中的每个数据都可以是不同的类型，并且每个数据都有与之关联的名称。这些数据片段被称为**字段（fields）**。它们的行为类似于其他语言中的属性（properties）。

```rust
struct User {
    active: bool,
    email: String,
    age: u64
}
```

在我们定义了结构体之后，要使用它，我们需要创建该结构体的实例，通过为每个字段指定具体的值。

```rust
let mut user1 = User {
    active: true,
    email: String::from("test@test.com"),
    age: 36
};
```

要从结构体中获取或设置特定的值，我们使用点符号表示法。

```rust
user1.age = 37;
```

### 枚举类型

枚举类型（Enumerations，或简称枚举，Enums）是一种数据结构，允许您通过列举其可能的变体（variants）来定义一个类型。一个枚举的示例可能如下所示：

```rust
enum LightStatus {
    On,
    Off
}
```

在这种情况下，`LightStatus` 枚举有两个可能的变体：它可以是 `On` 或 `Off`。

您还可以将值嵌入到枚举变体中，类似于向结构体添加字段。

```rust
enum LightStatus {
    On {
        color: String
    },
    Off
}

let light_status = LightStatus::On { color: String::from("red") };
```

在这个例子中，将一个变量设置为 `LightStatus` 的 `On` 变体需要同时设置 `color` 的值。

### 匹配语句

匹配语句（match statements）与 C/C++ 中的 `switch` 语句非常相似。`match` 语句允许你将一个值与一系列模式（patterns）进行比较，然后根据匹配的模式执行代码。模式可以由字面值、变量名、通配符等组成。匹配语句必须包含所有可能的情况，否则代码将无法编译。

```rust
enum Coin {
    Penny,
    Nickel,
    Dime,
    Quarter
}

fn value_in_cents(coin: Coin) -> u8 {
    match coin {
        Coin::Penny => 1,
        Coin::Nickel => 5,
        Coin::Dime => 10,
        Coin::Quarter => 25
    }
}
```

### 实现

在 Rust 中，使用 `impl` 关键字来定义类型的实现（implementations）。函数和常量都可以在实现中定义。

```rust
struct Example {
    number: i32
}

impl Example {
    fn boo() {
        println!("boo! Example::boo() was called!");
    }

    fn answer(&mut self) {
        self.number += 42;
    }

    fn get_number(&self) -> i32 {
        self.number
    }
}
```

这里的函数 `boo` 只能在类型本身上调用，而不能在类型的实例上调用，就像这样：

```rust
Example::boo();
```

与此同时，`answer` 需要一个可变的 `Example` 实例，并且可以使用点语法调用：

```rust
let mut example = Example { number: 3 };
example.answer();
```

### 特征和属性

在这个阶段，您不会创建自己的特征（traits）或属性（attributes），因此我们不会提供详细的解释。然而，您将使用 `borsh` crate 提供的 `derive` 属性宏（attribute macro）和一些特征，因此您需要对每个特征有一个高层次的理解。

特征描述了类型可以实现的抽象接口。如果一个特征定义了一个函数 `bark()`，而一个类型采用了该特征，那么该类型必须实现 `bark()` 函数。

[属性](https://doc.rust-lang.org/rust-by-example/attribute.html) 可以为类型添加元数据，可以用于许多不同的目的。

当您将 [`derive` 属性](https://doc.rust-lang.org/rust-by-example/trait/derive.html) 添加到一个类型上并提供一个或多个支持的特征时，代码会在幕后生成，自动为该类型实现这些特征。我们很快会提供一个具体的例子。

## 将指令表示为 Rust 数据类型

现在我们已经了解了 Rust 的基础知识，让我们将它们应用到 Solana 程序中。

通常情况下，程序会有多个函数。例如，您可能有一个程序作为记笔记应用的后端。假设该程序接受用于创建新笔记、更新现有笔记和删除现有笔记的指令。

由于指令具有离散的类型，它们通常非常适合用枚举数据类型表示。

```rust
enum NoteInstruction {
    CreateNote {
        title: String,
        body: String,
        id: u64
    },
    UpdateNote {
        title: String,
        body: String,
        id: u64
    },
    DeleteNote {
        id: u64
    }
}
```

请注意，`NoteInstruction` 枚举的每个变体都附带了嵌入数据，程序将使用这些数据来执行创建、更新和删除笔记的任务。

## 反序列化指令数据

指令数据以字节数组的形式传递给程序，因此您需要一种确定性地将该数组转换为指令枚举类型实例的方法。

在之前的单元中，我们使用 Borsh 进行客户端的序列化和反序列化。要在程序端使用 Borsh，我们使用 `borsh` crate。这个 crate 提供了 `BorshDeserialize` 和 `BorshSerialize` 的 traits，您可以使用 `derive` 属性将这些 traits 应用到您的类型上。

为了使反序列化指令数据简单化，您可以创建一个表示数据的结构体，并使用 `derive` 属性将 `BorshDeserialize` trait 应用到结构体上。这将实现 `BorshDeserialize` 中定义的方法，包括我们将使用来反序列化指令数据的 `try_from_slice` 方法。

请记住，结构体本身需要与字节数组中的数据结构相匹配。

```rust
#[derive(BorshDeserialize)]
struct NoteInstructionPayload {
    id: u64,
    title: String,
    body: String
}
```

一旦创建了这个结构体，您可以为指令枚举创建一个实现，用于处理与反序列化指令数据相关的逻辑。通常会在一个名为 `unpack` 的函数内完成这个操作，该函数接受指令数据作为参数，并返回带有反序列化数据的适当枚举实例。

按照标准惯例，您的程序应该预期第一个字节（或其他固定数量的字节）是指示程序应该运行哪个指令的标识符。这可以是整数或字符串标识符。对于本示例，我们将使用第一个字节，并将整数 0、1 和 2 映射到分别表示创建、更新和删除的指令。

```rust
impl NoteInstruction {
    // Unpack inbound buffer to associated Instruction
    // The expected format for input is a Borsh serialized vector
    pub fn unpack(input: &[u8]) -> Result<Self, ProgramError> {
        // Take the first byte as the variant to
        // determine which instruction to execute
        let (&variant, rest) = input.split_first().ok_or(ProgramError::InvalidInstructionData)?;
        // Use the temporary payload struct to deserialize
        let payload = NoteInstructionPayload::try_from_slice(rest).unwrap();
        // Match the variant to determine which data struct is expected by
        // the function and return the TestStruct or an error
        Ok(match variant {
            0 => Self::CreateNote {
                title: payload.title,
                body: payload.body,
                id: payload.id
            },
            1 => Self::UpdateNote {
                title: payload.title,
                body: payload.body,
                id: payload.id
            },
            2 => Self::DeleteNote {
                id: payload.id
            },
            _ => return Err(ProgramError::InvalidInstructionData)
        })
    }
}
```

这个示例包含很多内容，让我们一步一步来看：

1. 这个函数首先在 `input` 参数上使用 `split_first` 函数，返回一个元组。第一个元素 `variant` 是字节数组的第一个字节，第二个元素 `rest` 是字节数组的剩余部分。
2. 然后，函数使用 `NoteInstructionPayload` 上的 `try_from_slice` 方法，将字节数组的剩余部分反序列化为一个名为 `payload` 的 `NoteInstructionPayload` 实例。
3. 最后，函数使用 `match` 表达式在 `variant` 上创建并返回适当的枚举实例，使用 `payload` 中的信息。

请注意，这个函数中有一些我们尚未解释的 Rust 语法。`ok_or` 和 `unwrap` 函数用于错误处理，我们将在另一个课程中详细讨论它们。

## 程序逻辑

有了一种将指令数据反序列化为自定义 Rust 类型的方法，您可以使用适当的控制流来执行程序中的不同代码路径，这取决于传递到程序入口点的指令是什么。

```rust
entrypoint!(process_instruction);

pub fn process_instruction(
    program_id: &Pubkey,
    accounts: &[AccountInfo],
    instruction_data: &[u8]
) -> ProgramResult {
    // Call unpack to deserialize instruction_data
    let instruction = NoteInstruction::unpack(instruction_data)?;
    // Match the returned data struct to what you expect
    match instruction {
        NoteInstruction::CreateNote { title, body, id } => {
            // Execute program code to create a note
        },
        NoteInstruction::UpdateNote { title, body, id } => {
            // Execute program code to update a note
        },
        NoteInstruction::DeleteNote { id } => {
            // Execute program code to delete a note
        }
    }
}
```

对于只有一两个指令需要执行的简单程序，将实际执行逻辑写在 `match` 语句内部可能是可以接受的。对于有许多不同可能指令需要匹配的程序来说，如果为每个指令的逻辑编写一个单独的函数，并且只是从 `match` 语句内部调用这些函数，那么您的代码将会更加可读。

## 程序文件结构

[Hello World 课程](./hello-world-program.md) 的程序足够简单，可以限制在一个文件中。但随着程序复杂度的增加，保持一个易读且可扩展的项目结构变得至关重要。这涉及将代码封装到函数和数据结构中，就像我们迄今所做的那样。但它还涉及将相关的代码分组到单独的文件中。

例如，到目前为止，我们已经处理的大部分代码都与定义和反序列化指令有关。这些代码应该放在自己的文件中，而不是写在与入口点相同的文件中。通过这样做，我们将有两个文件，一个文件包含程序入口点，另一个文件包含指令代码：

- **lib.rs**
- **instruction.rs**

一旦您开始像这样拆分您的程序，您将需要确保将所有文件注册在一个中央位置。我们将在 `lib.rs` 中进行此操作。**您必须像这样注册您程序中的每个文件。**

```rust
// This would be inside lib.rs
pub mod instruction;
```

此外，为了能够在其他文件中通过 `use` 语句调用您的声明（例如，`NoteInstruction`），您需要在这些声明前添加 `pub` 关键词。

```rust
pub enum NoteInstruction { ... }
```

# 实验

在本课程的实验中，我们将扩展第一个模块中使用的电影评论程序的前半部分。该程序存储用户提交的电影评论。

现在，我们将专注于反序列化指令数据。接下来的课程将专注于该程序的后半部分。

## 1. 入口点

我们将再次使用 [Solana Playground](https://beta.solpg.io/) 来构建这个程序。Solana Playground 在您的浏览器中保存状态，因此您之前的所有操作可能仍然存在。如果存在，请清除当前 `lib.rs` 文件中的所有内容。

在 `lib.rs` 文件中，我们将引入以下 crate，并使用 `entrypoint` 宏定义程序的入口点（entry point）。

```rust
use solana_program::{
    entrypoint,
    entrypoint::ProgramResult,
    pubkey::Pubkey,
    msg,
    account_info::AccountInfo,
};

// Entry point is a function call process_instruction
entrypoint!(process_instruction);

// Inside lib.rs
pub fn process_instruction(
    program_id: &Pubkey,
    accounts: &[AccountInfo],
    instruction_data: &[u8]
) -> ProgramResult {

    Ok(())
}
```

## 2. 反序列化指令数据

在继续处理器逻辑之前，我们应该定义我们支持的指令并实现我们的反序列化函数。

为了提高可读性，让我们创建一个名为 `instruction.rs` 的新文件。在这个新文件中，添加对 `BorshDeserialize` 和 `ProgramError` 的 `use` 语句，然后创建一个名为 `MovieInstruction` 的枚举，其中包含一个名为 `AddMovieReview` 的变体。这个变体应该有嵌入值 `title`、`rating` 和 `description`。

```rust
use borsh::{BorshDeserialize};
use solana_program::{program_error::ProgramError};

pub enum MovieInstruction {
    AddMovieReview {
        title: String,
        rating: u8,
        description: String
    }
}
```

接下来，定义一个名为 `MovieReviewPayload` 的结构体。这将充当反序列化的中间类型，因此它应该使用 `derive` 属性宏为 `BorshDeserialize` trait 提供默认实现。

```rust
#[derive(BorshDeserialize)]
struct MovieReviewPayload {
    title: String,
    rating: u8,
    description: String
}
```

最后，为 `MovieInstruction` 枚举创建一个实现，定义并实现一个名为 `unpack` 的函数，该函数以字节数组作为参数，并返回一个 `Result` 类型。这个函数应该：

1. 使用 `split_first` 函数将数组的第一个字节与数组的其余部分分开。
2. 将数组的其余部分反序列化为 `MovieReviewPayload` 实例。
3. 使用 `match` 语句，如果数组的第一个字节为 0，则返回 `MovieInstruction` 的 `AddMovieReview` 变体，否则返回一个程序错误。

```rust
impl MovieInstruction {
    // Unpack inbound buffer to associated Instruction
    // The expected format for input is a Borsh serialized vector
    pub fn unpack(input: &[u8]) -> Result<Self, ProgramError> {
        // Split the first byte of data
        let (&variant, rest) = input.split_first().ok_or(ProgramError::InvalidInstructionData)?;
        // `try_from_slice` is one of the implementations from the BorshDeserialization trait
        // Deserializes instruction byte data into the payload struct
        let payload = MovieReviewPayload::try_from_slice(rest).unwrap();
        // Match the first byte and return the AddMovieReview struct
        Ok(match variant {
            0 => Self::AddMovieReview {
                title: payload.title,
                rating: payload.rating,
                description: payload.description },
            _ => return Err(ProgramError::InvalidInstructionData)
        })
    }
}
```

## 3. 程序逻辑

处理了指令的反序列化之后，我们可以回到 `lib.rs` 文件中处理一些程序逻辑。

请记住，由于我们在不同的文件中添加了代码，我们需要在 `lib.rs` 文件中使用 `pub mod instruction;` 来注册它。然后，我们可以添加一个 `use` 语句来将 `MovieInstruction` 类型引入范围。

```rust
pub mod instruction;
use instruction::{MovieInstruction};
```

接下来，让我们定义一个新函数 `add_movie_review`，它以 `program_id`、`accounts`、`title`、`rating` 和 `description` 作为参数。它还应该返回一个 `ProgramResult` 实例。在这个函数内部，目前让我们简单地记录我们的值，我们将在下一课程中重新访问该函数的其余实现。

```rust
pub fn add_movie_review(
    program_id: &Pubkey,
    accounts: &[AccountInfo],
    title: String,
    rating: u8,
    description: String
) -> ProgramResult {

    // Logging instruction data that was passed in
    msg!("Adding movie review...");
    msg!("Title: {}", title);
    msg!("Rating: {}", rating);
    msg!("Description: {}", description);

    Ok(())
}
```

完成了这一步，我们可以从 `process_instruction`（我们设置为入口点的函数）中调用 `add_movie_review`。为了将所有必需的参数传递给函数，我们首先需要对 `MovieInstruction` 上创建的 `unpack` 进行调用，然后使用 `match` 语句确保我们收到的指令是 `AddMovieReview` 变体。

```rust
pub fn process_instruction(
    program_id: &Pubkey,
    accounts: &[AccountInfo],
    instruction_data: &[u8]
) -> ProgramResult {
    // Unpack called
    let instruction = MovieInstruction::unpack(instruction_data)?;
    // Match against the data struct returned into `instruction` variable
    match instruction {
        MovieInstruction::AddMovieReview { title, rating, description } => {
            // Make a call to `add_move_review` function
            add_movie_review(program_id, accounts, title, rating, description)
        }
    }
}
```

这样一来，当提交交易时，您的程序应该已经足够功能齐备，可以记录传入的指令数据了！

从 Solana Program 构建和部署您的程序，就像上一课程中一样。如果您自上次课程以来没有更改程序 ID，它将自动部署到相同的 ID。如果您希望它有一个单独的地址，您可以在部署之前从 playground 生成一个新的程序 ID。

您可以通过提交包含正确指令数据的交易来测试您的程序。为此，您可以使用[此脚本](https://github.com/Unboxed-Software/solana-movie-client)或[我们在 Serialize Custom Instruction Data lesson 中构建的前端](https://github.com/Unboxed-Software/solana-movie-frontend)。在这两种情况下，请确保将您的程序 ID 复制并粘贴到源代码的适当位置，以确保您测试的是正确的程序。

如果您需要更多时间来完成此实验，请随时这样做！如果您被卡住了，您也可以查看程序的[解决方案代码](https://beta.solpg.io/62aa9ba3b5e36a8f6716d45b)。

# 挑战

对于本课程的挑战，尝试复制模块 1 中的学生介绍程序。回想一下，我们创建了一个前端应用，让学生介绍自己！该程序接受用户的姓名和简短消息作为 `instruction_data`，并创建一个账户将数据存储在链上。

利用您在本课程中学到的知识，将学生介绍程序建立到可以在调用程序时打印用户提供的 `name` 和 `message` 到程序日志的程度。

您可以通过构建我们在[Serialize Custom Instruction Data lesson](serialize-instruction-data)中创建的[前端](https://github.com/Unboxed-Software/solana-student-intros-frontend/tree/solution-serialize-instruction-data)，然后在 Solana Explorer 上检查程序日志来测试您的程序。请记住，将前端代码中的程序 ID 替换为您部署的程序 ID。

如果可能的话，请尝试独立完成！但是如果您遇到困难，请随时参考[解决方案代码](https://beta.solpg.io/62b0ce53f6273245aca4f5b0)。

## 完成了实验吗？

将您的代码推送到 GitHub，并[告诉我们您对本课程的看法](https://form.typeform.com/to/IPH0UGz7#answers-lesson=74a157dc-01a7-4b08-9a5f-27aa51a4346c)！