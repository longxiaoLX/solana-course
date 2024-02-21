---
title: Rust 过程宏
objectives:
- 在 Rust 中创建和使用 **过程宏**
- 解释并使用 Rust 抽象语法树（AST）
- 描述在 Anchor 框架中如何使用过程宏
---

# TL;DR

- **过程宏** 是 Rust 中一种特殊类型的宏，允许程序员根据自定义输入在编译时生成代码。
- 在 Anchor 框架中，过程宏用于生成代码，以减少编写 Solana 程序时所需的样板代码量。
- **抽象语法树（AST）** 是对输入代码的语法和结构的表示，传递给过程宏。创建宏时，您使用 AST 的元素（如 token 和 item）来生成相应的代码。
- **Token** 是 Rust 中编译器可以解析的最小源代码单位。
- **Item** 是定义可以在 Rust 程序中使用的声明，例如结构体、枚举、trait、函数或方法。
- **TokenStream** 是表示源代码片段的一系列 tokens，可以传递给过程宏，使其能够访问和操作代码中的各个 tokens。

# 概述

在 Rust 中，宏（macro）是一段代码，您可以编写一次，然后在编译时“展开”以生成代码。当您需要生成重复或复杂的代码，或者想要在程序中的多个位置使用相同的代码时，这可能很有用。

宏分为两种不同类型：声明性宏（declarative macros）和过程宏（procedural macros）。

- 声明性宏是使用 `macro_rules!` 宏定义的，它允许您根据匹配的模式来匹配代码，并基于匹配模式生成代码。
- Rust 中的过程宏是使用 Rust 代码定义的，并且操作输入 TokenStream 的抽象语法树（AST，abstract syntax tree），这使它们能够在更细节的级别上操作和生成代码。

在本课程中，我们将重点介绍过程宏，这在 Anchor 框架中经常使用。

## Rust 概念

在我们具体讨论宏之前，让我们先谈谈本课程中将要使用的一些重要术语、概念和工具。

### Token

在 Rust 编程的上下文中，[token](https://doc.rust-lang.org/reference/tokens.html)是语言语法的基本元素，如标识符或字面值。token 表示 Rust 编译器识别的源代码的最小单位，它们用于构建程序中更复杂的表达式和语句。

Rust token 的示例包括：

- [关键字（Keywords）](https://doc.rust-lang.org/reference/keywords.html)，如 `fn`、`let` 和 `match`，是 Rust 语言中具有特殊含义的保留字。
- [标识符（Identifiers）](https://doc.rust-lang.org/reference/identifiers.html)，如变量和函数名称，用于引用值和函数。
- [标点符号（Punctuation）](https://doc.rust-lang.org/reference/tokens.html#punctuation)，如 `{`、`}` 和 `;`，用于结构化和界定代码块。
- [字面值（Literals）](https://doc.rust-lang.org/reference/tokens.html#literals)，如数字和字符串，表示 Rust 程序中的常量值。

您可以[阅读更多关于 Rust token 的信息](https://doc.rust-lang.org/reference/tokens.html)。

### Item

在 Rust 中，item 是命名的、自包含的代码片段。它们提供了一种将相关代码组合在一起并通过名称引用该组的方法。这使您可以以模块化的方式重用和组织您的代码。

有几种不同类型的 item，例如：

-   函数
-   结构体
-   枚举
-   Traits
-   模块（Modules）
-   宏

您可以[阅读更多关于 Rust item 的信息](https://doc.rust-lang.org/reference/items.html)。

### Token Streams

`TokenStream` 类型是表示一系列 token 的数据类型。此类型在 `proc_macro` crate 中定义，并作为一种基于代码库中其他代码编写宏的方式呈现出来。

在定义过程宏时，宏输入作为 `TokenStream` 传递给宏，然后可以根据需要解析和转换。然后，宏生成的 `TokenStream` 可以扩展为宏生成的最终代码输出。

```rust
use proc_macro::TokenStream;

#[proc_macro]
pub fn my_macro(input: TokenStream) -> TokenStream {
    ...
}
```

### 抽象语法树

在 Rust 过程宏的上下文中，抽象语法树（abstract syntax tree，AST）是一种数据结构，表示输入 token 在 Rust 语言中的分层结构和含义。它通常被用作输入的中间表示，可以轻松地被过程宏处理和转换。

宏可以使用 AST 分析输入代码并对其进行更改，例如添加或删除 token，或以某种方式转换代码的含义。然后，它可以使用这个转换后的 AST 生成新的代码，该代码可以作为过程宏的输出返回。

### `syn` crate

`syn` crate 可用于将 token 流解析为过程宏代码可以遍历和操作的 AST。当在 Rust 程序中调用过程宏时，宏函数被调用时会将 token 流作为输入。解析此输入是几乎任何宏的第一步。

以使用 `my_macro!` 调用的过程宏为例：

```rust
my_macro!("hello, world");
```

当上述代码被执行时，Rust 编译器将输入的 token（`"hello, world"`）作为 `TokenStream` 传递给 `my_macro` 过程宏。

```rust
use proc_macro::TokenStream;
use syn::parse_macro_input;

#[proc_macro]
pub fn my_macro(input: TokenStream) -> TokenStream {
    let ast = parse_macro_input!(input as syn::LitStr);
    eprintln! {"{:#?}", ast};
    ...
}
```

在过程宏内部，代码使用 `syn` crate 中的 `parse_macro_input!` 宏将输入的 `TokenStream` 解析为抽象语法树（AST）。具体来说，这个例子将其解析为 Rust 中表示字符串字面值的 `LitStr` 的实例。然后，`eprintln!` 宏用于打印 `LitStr` AST 以进行调试目的。

```rust
LitStr {
    token: Literal {
        kind: Str,
        symbol: "hello, world",
        suffix: None,
        span: #0 bytes(172..186),
    },
}
```

`eprintln!` 宏的输出显示了从输入 token 生成的 `LitStr` AST 的结构。它显示了字符串字面值（`"hello, world"`）以及关于 token 的其他元数据，如其种类（`Str`）、后缀（`None`）和跨度（span）。

### `quote` crate

另一个重要的 crate 是 `quote` crate。在宏的代码生成部分，这个 crate 是至关重要的。

一旦过程宏完成了对 AST 的分析和转换，它可以使用 `quote` crate 或类似的代码生成库将 AST 转换回 token 流。然后，它返回 `TokenStream`，Rust 编译器将使用它来替换源代码中的原始流。

看下面的 `my_macro` 示例：

```rust
use proc_macro::TokenStream;
use syn::parse_macro_input;
use quote::quote;

#[proc_macro]
pub fn my_macro(input: TokenStream) -> TokenStream {
    let ast = parse_macro_input!(input as syn::LitStr);
    eprintln! {"{:#?}", ast};
    let expanded = {
        quote! {println!("The input is: {}", #ast)}
    };
    expanded.into()
}
```

这个示例使用 `quote!` 宏生成一个新的 `TokenStream`，其中包含一个以 `LitStr` AST 作为参数的 `println!` 宏调用。

请注意，`quote!` 宏生成的 `TokenStream` 类型为 `proc_macro2::TokenStream`。要将此 `TokenStream` 返回给 Rust 编译器，您需要使用 `.into()` 方法将其转换为 `proc_macro::TokenStream`。然后，Rust 编译器将使用此 `TokenStream` 来替换源代码中的原始过程宏调用。

```text
The input is: hello, world
```

这使您能够创建执行强大的代码生成和元编程任务的过程宏。

## 过程宏

Rust 中的过程宏是一种强大的扩展语言并创建自定义语法的方式。这些宏是用 Rust 编写的，并与其他代码一起编译。有三种类型的过程宏：

-   函数式宏（Function-like macros） - `custom!(...)`
-   派生宏（erive macros） - `#[derive(CustomDerive)]`
-   属性宏（Attribute macros） - `#[CustomAttribute]`

本节将讨论这三种类型的过程宏，并提供一个示例实现。编写过程宏的过程在这三种类型中保持一致，因此所提供的示例可以适用于其他类型。

### 函数式宏

函数式过程宏是三种类型的过程宏中最简单的一种。这些宏是使用一个带有 `#[proc_macro]` 属性的函数定义的。该函数必须以 `TokenStream` 作为输入，并返回一个新的 `TokenStream` 作为输出，以替换原始代码。

```rust
#[proc_macro]
pub fn my_macro(input: TokenStream) -> TokenStream {
	...
}
```

这些宏是通过函数名称后跟 `!` 操作符来调用的。它们可以在 Rust 程序的各种位置中使用，例如表达式（expressions）、语句（statements）和函数定义中。

```rust
my_macro!(input);
```

函数式过程宏最适合简单的代码生成任务，只需要一个输入和输出流。它们易于理解和使用，并提供了一种直接的方法来在编译时生成代码。

### 属性宏

属性宏定义了附加到 Rust 程序中 items（如函数和结构体）的新属性。

```rust
#[my_macro]
fn my_function() {
	...
}
```

属性宏是使用一个带有 `#[proc_macro_attribute]` 属性的函数来定义的。该函数需要两个 token 流作为输入，并返回一个 `TokenStream` 作为输出，用任意数量的新 itemss 替换原始 item。

```rust
#[proc_macro_attribute]
pub fn my_macro(attr: TokenStream, input: TokenStream) -> TokenStream {
    ...
}
```

第一个 token 流输入表示属性参数。第二个 token 流是附加到属性的 item 的剩余部分，包括可能存在的任何其他属性。

```rust
#[my_macro(arg1, arg2)]
fn my_function() {
    ...
}
```

例如，属性宏可以处理传递给属性的参数以启用或禁用某些功能，然后使用第二个 token 流以某种方式修改原始项。通过访问两个 token 流，属性宏可以提供比仅使用单个 token 流更大的灵活性和功能。

### 派生宏

派生宏是通过在结构体、枚举或联合体上使用 `#[derive]` 属性来调用的，通常用于为输入类型自动实现 traits。

```rust
#[derive(MyMacro)]
struct Input {
	field: String
}
```

派生宏是通过在函数前使用 `#[proc_macro_derive]` 属性来定义的。它们仅限于为结构体、枚举和联合体生成代码。它们接受一个 token 流作为输入，并返回一个 token 流作为输出。

与其他过程宏不同，返回的 token 流不会替换原始代码。相反，返回的 token 流被附加到原始 item 所属的模块或块中。这允许开发人员扩展原始 item 的功能，而不修改原始代码。

```rust
#[proc_macro_derive(MyMacro)]
pub fn my_macro(input: TokenStream) -> TokenStream {
	...
}
```

除了实现 traits 之外，派生宏还可以定义辅助属性（helper attributes）。辅助属性可以在应用派生宏的 item 的范围内使用，并自定义代码生成过程。

```rust
#[proc_macro_derive(MyMacro, attributes(helper))]
pub fn my_macro(body: TokenStream) -> TokenStream {
    ...
}
```

辅助属性是惰性的，这意味着它们本身没有任何效果，它们的唯一目的是作为定义它们的派生宏的输入使用。

```rust
#[derive(MyMacro)]
struct Input {
    #[helper]
    field: String
}
```

例如，一个派生宏可以定义一个辅助属性，根据属性的存在来执行额外的操作。这允许开发人员进一步扩展派生宏的功能，并以更灵活的方式自定义它们生成的代码。

### 过程宏示例

这个示例展示了如何使用派生过程宏来自动生成结构体的 `describe()` 方法的实现。

```rust
use example_macro::Describe;

#[derive(Describe)]
struct MyStruct {
    my_string: String,
    my_number: u64,
}

fn main() {
    MyStruct::describe();
}
```

`describe()` 方法将打印结构体字段的描述到控制台。

```text
MyStruct is a struct with these named fields: my_string, my_number.
```

首先是使用 `#[proc_macro_derive]` 属性定义过程宏。使用 `parse_macro_input!()` 宏解析输入的 `TokenStream`，以提取结构体的标识符和数据。

```rust
use proc_macro::{self, TokenStream};
use quote::quote;
use syn::{parse_macro_input, DeriveInput, FieldsNamed};

#[proc_macro_derive(Describe)]
pub fn describe_struct(input: TokenStream) -> TokenStream {
    let DeriveInput { ident, data, .. } = parse_macro_input!(input);
    ...
}
```

接下来的步骤是使用 `match` 关键字对 `data` 值进行模式匹配，以提取结构体中字段的名称。

第一个 `match` 有两个分支：一个用于 `syn::Data::Struct` 变体，另一个用于处理 `syn::Data` 的所有其他变体的“通配符” `_` 分支。

第二个 `match` 也有两个分支：一个用于 `syn::Fields::Named` 变体，另一个用于处理 `syn::Fields` 的所有其他变体的“通配符” `_` 分支。

`#(#idents), *` 语法指定了 `idents` 迭代器将被“展开”，以创建迭代器中元素的逗号（,）分隔列表。

```rust
use proc_macro::{self, TokenStream};
use quote::quote;
use syn::{parse_macro_input, DeriveInput, FieldsNamed};

#[proc_macro_derive(Describe)]
pub fn describe_struct(input: TokenStream) -> TokenStream {
    let DeriveInput { ident, data, .. } = parse_macro_input!(input);

    let field_names = match data {
        syn::Data::Struct(s) => match s.fields {
            syn::Fields::Named(FieldsNamed { named, .. }) => {
                let idents = named.iter().map(|f| &f.ident);
                format!(
                    "a struct with these named fields: {}",
                    quote! {#(#idents), *},
                )
            }
            _ => panic!("The syn::Fields variant is not supported"),
        },
        _ => panic!("The syn::Data variant is not supported"),
    };
    ...
}
```

最后一步是为结构体实现一个 `describe()` 方法。使用 `quote!` 宏和 `impl` 关键字定义了 `expanded` 变量，以创建存储在 `#ident` 变量中的结构体名称的实现。

该实现定义了 `describe()` 方法，该方法使用 `println!` 宏打印结构体的名称和其字段名称。

最后，使用 `into()` 方法将 `expanded` 变量转换为 `TokenStream`。

```rust
use proc_macro::{self, TokenStream};
use quote::quote;
use syn::{parse_macro_input, DeriveInput, FieldsNamed};

#[proc_macro_derive(Describe)]
pub fn describe(input: TokenStream) -> TokenStream {
    let DeriveInput { ident, data, .. } = parse_macro_input!(input);

    let field_names = match data {
        syn::Data::Struct(s) => match s.fields {
            syn::Fields::Named(FieldsNamed { named, .. }) => {
                let idents = named.iter().map(|f| &f.ident);
                format!(
                    "a struct with these named fields: {}",
                    quote! {#(#idents), *},
                )
            }
            _ => panic!("The syn::Fields variant is not supported"),
        },
        _ => panic!("The syn::Data variant is not supported"),
    };

    let expanded = quote! {
        impl #ident {
            fn describe() {
            println!("{} is {}.", stringify!(#ident), #field_names);
            }
        }
    };

    expanded.into()
}
```

现在，当将 `#[derive(Describe)]` 属性添加到一个结构体时，Rust 编译器会自动生成一个 `describe()` 方法的实现，可以调用它来打印结构体的名称和其字段的名称。

```rust
#[derive(Describe)]
struct MyStruct {
    my_string: String,
    my_number: u64,
}
```

`cargo expand` 命令来自 `cargo-expand` crate，可以用来展开使用过程宏的 Rust 代码。例如，使用 `#[derive(Describe)]` 属性生成的 `MyStruct` 结构体的代码如下所示：

```rust
struct MyStruct {
    my_string: String,
    my_number: f64,
}
impl MyStruct {
    fn describe() {
        {
            ::std::io::_print(
                ::core::fmt::Arguments::new_v1(
                    &["", " is ", ".\n"],
                    &[
                        ::core::fmt::ArgumentV1::new_display(&"MyStruct"),
                        ::core::fmt::ArgumentV1::new_display(
                            &"a struct with these named fields: my_string, my_number",
                        ),
                    ],
                ),
            );
        };
    }
}
```

## Anchor 过程宏

过程宏是 Anchor 库背后的魔力，它在 Solana 开发中经常被使用。Anchor 宏允许更简洁的代码、常见的安全检查等。让我们通过几个示例来看看 Anchor 如何使用过程宏。

### 函数式宏

`declare_id` 宏展示了在 Anchor 中如何使用函数式宏。该宏接受表示程序 ID 的字符字符串作为输入，并将其转换为可以在 Anchor 程序中使用的 `Pubkey` 类型。

```rust
declare_id!("G839pmstFmKKGEVXRGnauXxFgzucvELrzuyk6gHTiK7a");
```

`declare_id` 宏是使用 `#[proc_macro]` 属性定义的，表示它是一个函数式过程宏。

```rust
#[proc_macro]
pub fn declare_id(input: proc_macro::TokenStream) -> proc_macro::TokenStream {
    let id = parse_macro_input!(input as id::Id);
    proc_macro::TokenStream::from(quote! {#id})
}
```

### 派生宏

`#[derive(Accounts)]` 是 Anchor 中使用的众多派生宏之一的示例。

`#[derive(Accounts)]` 宏生成了实现给定结构体的 `Accounts` trait 的代码。这个 trait 做了许多事情，包括验证和反序列化传递给指令的账户。这使得结构体可以被用作 Anchor 程序中指令所需的账户列表。

通过 `#[account(..)]` 属性对字段指定的任何约束都会在反序列化过程中应用。`#[instruction(..)]` 属性也可以添加以指定指令的参数，并使它们对宏可见。

```rust
#[derive(Accounts)]
#[instruction(input: String)]
pub struct Initialize<'info> {
    #[account(init, payer = payer, space = 8 + input.len())]
    pub data_account: Account<'info, MyData>,
    #[account(mut)]
    pub payer: Signer<'info>,
    pub system_program: Program<'info, System>,
}
```

这个宏是使用 `proc_macro_derive` 属性定义的，它允许它被用作派生宏，可以应用到一个结构体上。`#[proc_macro_derive(Accounts, attributes(account, instruction))]` 表示这是一个处理 `account` 和 `instruction` 辅助属性的派生宏。

```rust
#[proc_macro_derive(Accounts, attributes(account, instruction))]
pub fn derive_anchor_deserialize(item: TokenStream) -> TokenStream {
    parse_macro_input!(item as anchor_syn::AccountsStruct)
        .to_token_stream()
        .into()
}
```

### 属性宏 `#[program]`

`#[program]` 属性宏是 Anchor 中使用的一个属性宏的示例，用于定义包含 Solana 程序指令处理程序的模块。

```rust
#[program]
pub mod my_program {
    use super::*;

    pub fn initialize(ctx: Context<Initialize>) -> Result<()> {
        ...
    }
}
```

在这种情况下，`#[program]` 属性被应用到一个模块上，它用于指定该模块包含了 Solana 程序的指令处理程序。

```rust
#[proc_macro_attribute]
pub fn program(
    _args: proc_macro::TokenStream,
    input: proc_macro::TokenStream,
) -> proc_macro::TokenStream {
    parse_macro_input!(input as anchor_syn::Program)
        .to_token_stream()
        .into()
}
```

总的来说，在 Anchor 中使用过程宏大大减少了 Solana 开发人员需要编写的重复代码量。通过减少样板代码的数量，开发人员可以专注于程序的核心功能，避免由手动重复引起的错误。这最终导致了更快、更高效的开发过程。

# 实验

让我们通过创建一个新的派生宏来练习一下！我们的新宏将允许我们自动为 Anchor 程序中的每个账户字段生成更新指令逻辑。

## 1. 起始

要开始，请从[此存储库](https://github.com/Unboxed-Software/anchor-custom-macro/tree/starter)的 `starter` 分支下载起始代码（这个仓库好像没有了）。

起始代码包括一个简单的 Anchor 程序，允许您初始化和更新一个 `Config` 账户。这类似于我们在[环境变量课程](./env-variables.md)中所做的事情。

涉及的账户结构如下：

```rust
use anchor_lang::prelude::*;

#[account]
pub struct Config {
    pub auth: Pubkey,
    pub bool: bool,
    pub first_number: u8,
    pub second_number: u64,
}

impl Config {
    pub const LEN: usize = 8 + 32 + 1 + 1 + 8;
}
```

`programs/admin/src/lib.rs` 文件包含了程序的入口点，其中定义了程序的指令。当前，该程序有用于初始化此账户的指令，然后是每个账户字段更新的一个指令。

`programs/admin/src/admin_config` 目录包含程序的指令逻辑和状态。浏览每个文件。您会注意到，每个字段的指令逻辑都在每个指令中重复。

此实验的目标是实现一个过程宏，让我们能够替换所有指令逻辑函数，并自动生成每个指令的函数。

## 2. 设置自定义宏声明

首先，让我们创建一个单独的 crate 来实现我们的自定义宏。在项目的根目录中，运行 `cargo new custom-macro`。这将创建一个新的 `custom-macro` 目录，其中包含自己的 `Cargo.toml`。更新新的 `Cargo.toml` 文件如下：

```text
[package]
name = "custom-macro"
version = "0.1.0"
edition = "2021"

[lib]
proc-macro = true

[dependencies]
syn = "1.0.105"
quote = "1.0.21"
proc-macro2 = "0.4"
anchor-lang = "0.25.0"
```

第一行的 `proc-macro = true` 表示该 crate 包含一个过程宏。依赖项是我们将用来创建宏的派生所有 crate。

接下来，将 `src/main.rs` 更改为 `src/lib.rs`。

接下来，更新项目根目录下的 `Cargo.toml` 文件的 `members` 字段，包括 `"custom-macro"`：

```text
[workspace]
members = [
    "programs/*",
    "custom-macro"
]
```

现在我们的 crate 已经设置好并且准备就绪了。但在继续之前，让我们再创建一个 crate，在根目录下，我们可以用它来测试我们创建的宏。在项目根目录下使用 `cargo new custom-macro-test`。然后更新新创建的 `Cargo.toml`，将 `anchor-lang` 和 `custom-macro` crate 添加为依赖项：

```text
[package]
name = "custom-macro-test"
version = "0.1.0"
edition = "2021"

[dependencies]
anchor-lang = "0.25.0"
custom-macro = { path = "../custom-macro" }
```

接下来，像之前一样，更新根项目的 `Cargo.toml` 文件，将新的 `custom-macro-test` crate 包含进来：

```text
[workspace]
members = [
    "programs/*",
    "custom-macro",
    "custom-macro-test"
]
```

最后，用以下代码替换 `custom-macro-test/src/main.rs` 中的代码。我们稍后会用这个来进行测试：

```rust
use anchor_lang::prelude::*;
use custom_macro::InstructionBuilder;

#[derive(InstructionBuilder)]
pub struct Config {
    pub auth: Pubkey,
    pub bool: bool,
    pub first_number: u8,
    pub second_number: u64,
}
```

## 3. 定义自定义宏

现在，在 `custom-macro/src/lib.rs` 文件中，让我们添加我们新宏的声明。在这个文件中，我们将使用 `parse_macro_input!` 宏来解析输入的 `TokenStream` 并从 `DeriveInput` 结构中提取 `ident` 和 `data` 字段。然后，我们将使用 `eprintln!` 宏来打印 `ident` 和 `data` 的值。暂时，我们将使用 `TokenStream::new()` 来返回一个空的 `TokenStream`。

```rust
use proc_macro::TokenStream;
use quote::*;
use syn::*;

#[proc_macro_derive(InstructionBuilder)]
pub fn instruction_builder(input: TokenStream) -> TokenStream {
    let DeriveInput { ident, data, .. } = parse_macro_input!(input);

    eprintln! {"{:#?}", ident};
    eprintln! {"{:#?}", data};

    TokenStream::new()
}
```

让我们测试一下打印出的内容。为此，您首先需要通过运行 `cargo install cargo-expand` 安装 `cargo-expand` 命令。您还需要通过运行 `rustup install nightly` 安装 Rust 的 Nightly 版本。

完成以上步骤后，您可以通过转到 `custom-macro-test` 目录并运行 `cargo expand` 来查看上述代码的输出。

这个命令会展开 crate 中的宏。由于 `main.rs` 文件使用了新创建的 `InstructionBuilder` 宏，这将在控制台打印出结构体的 `ident` 和 `data` 的语法树。一旦您确认输入的 `TokenStream` 被正确解析，可以随时移除 `eprintln!` 语句。

## 4. 获取结构体的字段

接下来，让我们使用 `match` 语句从结构体的 `data` 中获取命名字段。然后，我们将使用 `eprintln!` 宏来打印字段的值。

```rust
use proc_macro::TokenStream;
use quote::*;
use syn::*;

#[proc_macro_derive(InstructionBuilder)]
pub fn instruction_builder(input: TokenStream) -> TokenStream {
    let DeriveInput { ident, data, .. } = parse_macro_input!(input);

    let fields = match data {
        syn::Data::Struct(s) => match s.fields {
            syn::Fields::Named(n) => n.named,
            _ => panic!("The syn::Fields variant is not supported: {:#?}", s.fields),
        },
        _ => panic!("The syn::Data variant is not supported: {:#?}", data),
    };

    eprintln! {"{:#?}", fields};

    TokenStream::new()
}
```

再次在终端中使用 `cargo expand` 命令来查看此代码的输出。一旦您确认字段被正确提取并打印出来，您就可以删除 `eprintln!` 语句。

## 5. 构建更新指令

接下来，让我们遍历结构体的字段，并为每个字段生成一个更新指令。指令将使用 `quote!` 宏生成，并包括字段的名称和类型，以及一个新的函数名称用于更新指令。

```rust
use proc_macro::TokenStream;
use quote::*;
use syn::*;

#[proc_macro_derive(InstructionBuilder)]
pub fn instruction_builder(input: TokenStream) -> TokenStream {
    let DeriveInput { ident, data, .. } = parse_macro_input!(input);

    let fields = match data {
        syn::Data::Struct(s) => match s.fields {
            syn::Fields::Named(n) => n.named,
            _ => panic!("The syn::Fields variant is not supported: {:#?}", s.fields),
        },
        _ => panic!("The syn::Data variant is not supported: {:#?}", data),
    };

    let update_instruction = fields.into_iter().map(|f| {
        let name = &f.ident;
        let ty = &f.ty;
        let fname = format_ident!("update_{}", name.clone().unwrap());

        quote! {
            pub fn #fname(ctx: Context<UpdateAdminAccount>, new_value: #ty) -> Result<()> {
                let admin_account = &mut ctx.accounts.admin_account;
                admin_account.#name = new_value;
                Ok(())
            }
        }
    });

    TokenStream::new()
}
```

## 6. 返回新的 `TokenStream`

最后，让我们使用 `quote!` 宏为由 `ident` 变量指定的结构体生成一个实现。该实现包括为结构体中的每个字段生成的更新指令。然后，使用 `into()` 方法将生成的代码转换为 `TokenStream`，并将其作为宏的结果返回。

```rust
use proc_macro::TokenStream;
use quote::*;
use syn::*;

#[proc_macro_derive(InstructionBuilder)]
pub fn instruction_builder(input: TokenStream) -> TokenStream {
    let DeriveInput { ident, data, .. } = parse_macro_input!(input);

    let fields = match data {
        syn::Data::Struct(s) => match s.fields {
            syn::Fields::Named(n) => n.named,
            _ => panic!("The syn::Fields variant is not supported: {:#?}", s.fields),
        },
        _ => panic!("The syn::Data variant is not supported: {:#?}", data),
    };

    let update_instruction = fields.into_iter().map(|f| {
        let name = &f.ident;
        let ty = &f.ty;
        let fname = format_ident!("update_{}", name.clone().unwrap());

        quote! {
            pub fn #fname(ctx: Context<UpdateAdminAccount>, new_value: #ty) -> Result<()> {
                let admin_account = &mut ctx.accounts.admin_account;
                admin_account.#name = new_value;
                Ok(())
            }
        }
    });

    let expanded = quote! {
        impl #ident {
            #(#update_instruction)*
        }
    };
    expanded.into()
}
```

为了验证宏是否生成了正确的代码，使用 `cargo expand` 命令来查看宏的展开形式。其输出如下所示：

```rust
use anchor_lang::prelude::*;
use custom_macro::InstructionBuilder;
pub struct Config {
    pub auth: Pubkey,
    pub bool: bool,
    pub first_number: u8,
    pub second_number: u64,
}
impl Config {
    pub fn update_auth(
        ctx: Context<UpdateAdminAccount>,
        new_value: Pubkey,
    ) -> Result<()> {
        let admin_account = &mut ctx.accounts.admin_account;
        admin_account.auth = new_value;
        Ok(())
    }
    pub fn update_bool(ctx: Context<UpdateAdminAccount>, new_value: bool) -> Result<()> {
        let admin_account = &mut ctx.accounts.admin_account;
        admin_account.bool = new_value;
        Ok(())
    }
    pub fn update_first_number(
        ctx: Context<UpdateAdminAccount>,
        new_value: u8,
    ) -> Result<()> {
        let admin_account = &mut ctx.accounts.admin_account;
        admin_account.first_number = new_value;
        Ok(())
    }
    pub fn update_second_number(
        ctx: Context<UpdateAdminAccount>,
        new_value: u64,
    ) -> Result<()> {
        let admin_account = &mut ctx.accounts.admin_account;
        admin_account.second_number = new_value;
        Ok(())
    }
}
```

## 7. 更新程序以使用您的新宏

要使用新的宏为 `Config` 结构体生成更新指令，首先将 `custom-macro` crate 添加为程序的依赖项，添加到其 `Cargo.toml` 文件中：

```text
[dependencies]
anchor-lang = "0.25.0"
custom-macro = { path = "../../custom-macro" }
```

然后，转到 Anchor 程序中的 `state.rs` 文件，并使用以下代码进行更新：

```rust
use crate::admin_update::UpdateAdminAccount;
use anchor_lang::prelude::*;
use custom_macro::InstructionBuilder;

#[derive(InstructionBuilder)]
#[account]
pub struct Config {
    pub auth: Pubkey,
    pub bool: bool,
    pub first_number: u8,
    pub second_number: u64,
}

impl Config {
    pub const LEN: usize = 8 + 32 + 1 + 1 + 8;
}
```

接下来，转到 `admin_update.rs` 文件，并删除现有的更新指令。这样应该只剩下该文件中的 `UpdateAdminAccount` 上下文结构体。

```rust
use crate::state::Config;
use anchor_lang::prelude::*;

#[derive(Accounts)]
pub struct UpdateAdminAccount<'info> {
    pub auth: Signer<'info>,
    #[account(
        mut,
        has_one = auth,
    )]
    pub admin_account: Account<'info, Config>,
}
```

接下来，更新 Anchor 程序中的 `lib.rs`，以使用由 `InstructionBuilder` 宏生成的更新指令。

```rust
use anchor_lang::prelude::*;
mod admin_config;
use admin_config::*;

declare_id!("Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS");

#[program]
pub mod admin {
    use super::*;

    pub fn initialize(ctx: Context<Initialize>) -> Result<()> {
        Initialize::initialize(ctx)
    }

    pub fn update_auth(ctx: Context<UpdateAdminAccount>, new_value: Pubkey) -> Result<()> {
        Config::update_auth(ctx, new_value)
    }

    pub fn update_bool(ctx: Context<UpdateAdminAccount>, new_value: bool) -> Result<()> {
        Config::update_bool(ctx, new_value)
    }

    pub fn update_first_number(ctx: Context<UpdateAdminAccount>, new_value: u8) -> Result<()> {
        Config::update_first_number(ctx, new_value)
    }

    pub fn update_second_number(ctx: Context<UpdateAdminAccount>, new_value: u64) -> Result<()> {
        Config::update_second_number(ctx, new_value)
    }
}
```

最后，转到 `admin` 目录并运行 `anchor test`，以验证由 `InstructionBuilder` 宏生成的更新指令是否正常工作。

```
  admin
    ✔ Is initialized! (160ms)
    ✔ Update bool! (409ms)
    ✔ Update u8! (403ms)
    ✔ Update u64! (406ms)
    ✔ Update Admin! (405ms)


  5 passing (2s)
```

干得好！到此为止，您可以创建过程宏来帮助您的开发过程。我们鼓励您充分利用 Rust 语言，并在合适的情况下使用宏。但即使您不使用宏，了解它们的工作原理也有助于理解 Anchor 在幕后发生了什么。

如果您需要花更多时间来研究解决方案代码，请随时参考[存储库](https://github.com/Unboxed-Software/anchor-custom-macro/tree/solution)的 `solution` 分支。

# 挑战

为了巩固所学的知识，继续创建另一个过程宏。考虑一下您编写过的代码，看看哪些地方可以通过宏来简化或改进，然后尝试一下！由于这仍然是练习，如果结果不如您所期望，也没有关系。只需开始尝试并进行实验！

## 完成了实验吗？

将您的代码推送到 GitHub，并[告诉我们您对这节课的看法](https://form.typeform.com/to/IPH0UGz7#answers-lesson=eb892157-3014-4635-beac-f562af600bf8)！