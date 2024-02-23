---
title: 跨程序调用
objectives:
- 解释跨程序调用（CPIs）
- 描述如何构建和使用 CPIs
- 解释程序如何为 PDA 提供签名
- 避免与 CPIs 相关的常见问题，并排除与之相关的常见错误
---

# 总结

- **跨程序调用（Cross-Program Invocation，CPI）** 是一个程序对另一个程序的调用，针对的是目标程序中的特定指令。
- 使用命令 `invoke` 或 `invoke_signed` 进行 CPI，后者是程序提供 PDA 签名的方式。
- CPI 使得 Solana 生态系统中的程序完全可互操作，因为一个程序的所有公共指令都可以通过 CPI 被另一个程序调用。
- 由于我们无法控制提交给程序的账户和数据，因此验证传递给 CPI 的所有参数以确保程序安全非常重要。

# 概述

## 什么是CPI？

跨程序调用（Cross-Program Invocation，CPI）是一个程序直接调用另一个程序的方式。就像任何客户端都可以使用 JSON RPC 调用任何程序一样，任何程序都可以直接调用任何其他程序。从您的程序中调用另一个程序的指令的唯一要求是正确构造指令。您可以对原生程序（native programs）、您创建的其他程序以及第三方程序进行 CPI。CPI 本质上将整个 Solana 生态系统转变为一个巨大的 API，作为开发者，您可以随意使用。

CPI 的构成与您习惯于在客户端创建的指令类似。根据您使用的是 `invoke` 还是 `invoke_signed`，会有一些复杂性和差异。我们将在本课程的后面分别讨论这两种情况。

## 如何进行 CPI

CPI 是使用 `solana_program` 包中的 [`invoke`](https://docs.rs/solana-program/latest/solana_program/program/fn.invoke.html) 或 [`invoke_signed`](https://docs.rs/solana-program/latest/solana_program/program/fn.invoke_signed.html) 函数进行的。您可以使用 `invoke` 来传递您程序的原始交易签名。您可以使用 `invoke_signed` 为您的程序的 PDA 进行“签名”。

```rust
// Used when there are not signatures for PDAs needed
pub fn invoke(
    instruction: &Instruction,
    account_infos: &[AccountInfo<'_>]
) -> ProgramResult

// Used when a program must provide a 'signature' for a PDA, hence the signer_seeds parameter
pub fn invoke_signed(
    instruction: &Instruction,
    account_infos: &[AccountInfo<'_>],
    signers_seeds: &[&[&[u8]]]
) -> ProgramResult
```

CPI 扩展了调用方对被调用方的权限。如果被调用方程序正在处理的指令包含一个账户，在最初传递给调用方程序时被标记为签名者（signer）或可写（writable）的，那么在被调用的程序中，该账户也将被视为签名者或可写的账户。

重要的是要注意，作为开发者，您可以决定将哪些账户传递给 CPI。您可以将 CPI 视为使用仅传递给您程序的信息，来去从头开始构建另一个指令。

### CPI with `invoke`

```rust
invoke(
    &Instruction {
        program_id: calling_program_id,
        accounts: accounts_meta,
        data,
    },
    &account_infos[account1.clone(), account2.clone(), account3.clone()],
)?;
```

- `program_id` - 要调用的程序的公钥
- `account` - 账户元数据（[AccountMeta](https://docs.rs/solana-sdk/latest/solana_sdk/instruction/struct.Instruction.html)）列表向量。您需要包括被调用程序将要读取或写入的每个账户
- `data` - 表示以向量的形式传递给被调用方程序的数据的字节缓冲区

`Instruction` 类型的定义如下：

```rust
pub struct Instruction {
    pub program_id: Pubkey,
    pub accounts: Vec<AccountMeta>,
    pub data: Vec<u8>,
}
```

根据您要调用的程序，可能会提供一个 crate，其中包含用于创建 `Instruction` 对象的辅助函数。许多个人和组织会在他们的程序旁边创建公开可用的 crate，以公开这些函数，以简化调用他们的程序。这类似于本课程中我们使用的 Typescript 库（例如 [@solana/web3.js](https://solana-labs.github.io/solana-web3.js/)、[@solana/spl-token](https://solana-labs.github.io/solana-program-library/token/js/)）。例如，在本课程的实验中，我们将使用 `spl_token` crate 来创建铸币指令。

在所有其他情况下，您需要从头开始创建 `Instruction` 实例。

虽然 `program_id` 字段相当简单，但 `accounts` 和 `data` 字段需要一些解释。

`accounts` 和 `data` 字段都是 `Vec` 类型，或称为向量。您可以使用 [`vec`](https://doc.rust-lang.org/std/macro.vec.html) 宏来使用数组表示法构造向量，如下所示：

```rust
let v = vec![1, 2, 3];
assert_eq!(v[0], 1);
assert_eq!(v[1], 2);
assert_eq!(v[2], 3);
```

`Instruction` 结构体的 `accounts` 字段期望一个类型为 [`AccountMeta`](https://docs.rs/solana-program/latest/solana_program/instruction/struct.AccountMeta.html) 的向量。`AccountMeta` 结构体的定义如下：

```rust
pub struct AccountMeta {
    pub pubkey: Pubkey,
    pub is_signer: bool,
    pub is_writable: bool,
}
```

将这两个部分组合起来的样子如下所示：

```rust
use solana_program::instruction::AccountMeta;

vec![
    AccountMeta::new(account1_pubkey, true), // metadata for a writable, signer account
    AccountMeta::read_only(account2_pubkey, false), // metadata for a read-only, non-signer account
    AccountMeta::read_only(account3_pubkey, true), // metadata for a read-only, signer account
    AccountMeta::new(account4_pubkey, false), // metadata for a writable, non-signer account
]
```

指令对象的最后一个字段是数据（data），当然是作为字节缓冲区。在 Rust 中，您可以再次使用 `vec` 宏来创建字节缓冲区，该宏具有一个已实现的函数，允许您创建特定长度的向量。一旦初始化了一个空向量，您将构造字节缓冲区，类似于客户端的做法。确定被调用方程序需要的数据和使用的序列化格式，然后编写代码以匹配。请随意查阅一些[此处可用的 `vec` 宏的特性](https://doc.rust-lang.org/alloc/vec/struct.Vec.html#)。

```rust
let mut vec = Vec::with_capacity(3);
vec.push(1);
vec.push(2);
vec.extend_from_slice(&number_variable.to_le_bytes());
```

[`extend_from_slice`](https://doc.rust-lang.org/alloc/vec/struct.Vec.html#method.extend_from_slice) 方法可能是您新接触的。这是向量的一个方法，它接受一个切片作为输入，遍历该切片，克隆每个元素，然后将其附加到 `Vec` 中。

### 传递账户列表

除了指令之外，`invoke` 和 `invoke_signed` 还需要一组 `account_info` 对象。就像您添加到指令中的 `AccountMeta` 对象列表一样，您必须包含您要调用的程序将要读取或写入的所有账户。

在您的程序中进行 CPI 时，您应该已经获取了传递给您的程序的所有 `account_info` 对象，并将它们存储在变量中。您将通过选择要复制和发送的这些账户来构建 CPI 的 `account_info` 对象列表。

您可以使用 `solana_program` 包中实现的 `Clone` trait 来复制需要传递给 CPI 的每个 `account_info` 对象。这个 `Clone` trait 返回 [`account_info`](https://docs.rs/solana-program/latest/solana_program/account_info/struct.AccountInfo.html) 实例的副本。

```rust
&[first_account.clone(), second_account.clone(), third_account.clone()]
```

### 使用 `invoke` 进行 CPI

有了指令和账户列表，您可以调用 `invoke` 进行调用。

```rust
invoke(
    &Instruction {
        program_id: calling_program_id,
        accounts: accounts_meta,
        data,
    },
    &[account1.clone(), account2.clone(), account3.clone()],
)?;
```

不需要包含签名，因为 Solana runtime 会传递给您的程序的原始签名。请记住，如果需要代表 PDA 签名，则 `invoke` 将无法工作。对于这种情况，您需要使用 `invoke_signed`。

### 使用 `invoke_signed` 进行 CPI

使用 `invoke_signed` 有一点不同，因为这里有一个额外的字段，需要用于派生必须对交易进行签名的所有 PDA 的种子。您可能还记得之前的课程中提到过，PDAs 不位于 Ed25519 曲线上，因此没有相应的私钥。您已经知道程序可以为它们的 PDA 提供签名，但是还不知道实际的操作方式，现在您可以了解到了。程序使用 `invoke_signed` 函数为它们的 PDA 提供签名。`invoke_signed` 的前两个字段与 `invoke` 相同，但是在这里还有一个额外的 `signers_seeds` 字段。

```rust
invoke_signed(
    &instruction,
    accounts,
    &[&["First addresses seed"],
        &["Second addresses first seed",
        "Second addresses second seed"]],
)?;
```

虽然 PDAs 没有自己的私钥，但程序可以使用它们来发出一个包含 PDA 作为签名者的指令。runtime 验证 PDA 是否属于调用程序的唯一方法是让调用程序提供用于生成地址的种子，放在 `signers_seeds` 字段中。

Solana runtime 将在内部调用 [`create_program_address`](https://docs.rs/solana-program/latest/solana_program/pubkey/struct.Pubkey.html#method.create_program_address) ，使用提供的种子和调用程序的 `program_id`。然后，它可以将结果与指令中提供的地址进行比较。如果其中地址匹配，则 runtime 知道确实与该地址关联的程序是调用方，因此有权作为签名者。

## 最佳实践和常见陷阱

### 安全检查

在利用 CPI 时，有一些常见的错误和需要记住的事项对于您程序的安全性和健壮性至关重要。首先要记住的是，正如我们现在已经知道的，我们无法控制传递给我们程序的信息。因此，始终验证传递给 CPI 的 `program_id`、账户和数据是非常重要的。如果没有这些安全检查，有人可能会提交一个调用了完全不同程序的指令的交易，这是不理想的。

幸运的是，在 `invoke_signed` 函数中，标记为签名者的任何 PDA 的有效性都会进行内在检查。在进行 CPI 之前，应该在您的程序代码中的某处验证所有其他账户和 `instruction_data`。还要确保您正在调用的程序上的目标指令是预期的。最简单的方法是阅读您将要调用的程序的源代码，就像您在构造来自客户端的指令时所做的那样。

### 常见错误

在执行 CPI 时，您可能会遇到一些常见的错误，它们通常意味着您正在使用不正确的信息构造 CPI。例如，您可能会遇到类似于以下的错误消息：

```text
EF1M4SPfKcchb6scq297y8FPCaLvj5kGjwMzjTM68wjA's signer privilege escalated
Program returned error: "Cross-program invocation with unauthorized signer or writable account"
```

这条消息有点误导性，因为“签名者权限提升（signer privilege escalated）”看起来不像是一个问题，但实际上，它意味着您在消息中错误地为地址签名。如果您使用 `invoke_signed` 并收到此错误消息，则很可能是您提供的种子不正确。您还可以找到[一个因此错误而失败的示例交易](https://explorer.solana.com/tx/3mxbShkerH9ZV1rMmvDfaAhLhJJqrmMjcsWzanjkARjBQurhf4dounrDCUkGunH1p9M4jEwef9parueyHVw6r2Et?cluster=devnet)。

另一个类似的错误是当写入的账户在 `AccountMeta` 结构中没有标记为 `writable` 时抛出。

```text
2qoeXa9fo8xVHzd2h9mVcueh6oK3zmAiJxCTySM5rbLZ's writable privilege escalated
Program returned error: "Cross-program invocation with unauthorized signer or writable account"
```

请记住，任何在执行过程中可能由程序改变数据的账户都必须指定为可写。在执行过程中，向未指定为可写的账户写入数据将导致交易失败。向程序未拥有的账户写入数据将导致交易失败。在执行过程中，任何可能由程序改变余额的账户都必须指定为可写。在执行过程中，改变未指定为可写的账户的余额将导致交易失败。从程序未拥有的账户中减去余额将导致交易失败，但向任何账户添加余额是允许的，只要该账户是可变的。

要查看此操作，请查看[浏览器中的此交易](https://explorer.solana.com/tx/ExB9YQJiSzTZDBqx4itPaa4TpT8VK4Adk7GU5pSoGEzNz9fa7PPZsUxssHGrBbJRnCvhoKgLCWnAycFB7VYDbBg?cluster=devnet)。

## 为什么 CPIs 很重要？

CPIs 是 Solana 生态系统的一个非常重要的特性，它们使所有部署的程序能够互操作。有了 CPIs，开发时就无需重新发明轮子。这为在已有基础上构建新协议和应用程序创造了机会，就像搭积木或乐高积木一样。重要的是要记住，CPIs 是双向的，对于你部署的任何程序都是如此！如果你构建了一些酷炫和有用的东西，开发人员有能力在你所做的基础上构建，或者只是将你的协议插入到他们正在构建的任何东西中。可组合性是使加密货币如此独特的重要因素之一，而 CPIs 则是在 Solana 上实现这一点的关键。

另一个 CPIs 的重要方面是它们允许程序为它们的 PDAs 进行签名。正如你现在可能已经注意到的那样，PDAs 在 Solana 开发中被频繁使用，因为它们允许程序以一种特定的方式控制特定地址，以至于没有外部用户可以为这些地址生成带有有效签名的交易。这对于 Web3 中的许多应用程序（例如 DeFi、NFT 等）非常有用。如果没有 CPIs，PDAs 将不会像现在这样有用，因为程序将无法为涉及它们的交易签名 - 本质上将它们变成黑洞（一旦将某物发送到 PDA，就没有办法在没有 CPIs 的情况下将其取回！）

# 实验

现在让我们通过对电影评论程序进行一些修改，来亲身体验一下 CPIs。如果你没有经历过之前的课程，电影评论程序允许用户提交电影评论并将它们存储在 PDA 账户中。

在上一课中，我们添加了使用 PDAs 在其他电影评论上留下意见的功能。在本课中，我们将致力于每当提交评论或意见时，程序都会向评论者或提出意见者铸造代币。

为了实现这一点，我们将使用 CPI 调用 SPL Token 程序的 `MintTo` 指令。如果你需要对代币、代币铸造和铸造新代币进行复习，请在继续进行此实验之前查看 [Token 程序课程](./token-program.md)。

## 1. 获取起始代码并添加依赖项

要开始，我们将使用之前 PDA 课程中电影评论程序的最终状态。所以，如果你刚刚完成了那节课，那么你已经准备好可以开始了。如果你是刚刚进入这里的，不用担心，你可以[在这里下载起始代码](https://github.com/Unboxed-Software/solana-movie-program/tree/solution-add-comments)。我们将使用 `solution-add-comments` 分支作为我们的起点。

## 2. 在 `Cargo.toml` 中添加依赖项

在开始之前，我们需要在 `Cargo.toml` 文件的 `[dependencies]` 下添加两个新的依赖项。除了现有的依赖项外，我们还将使用 `spl-token` 和 `spl-associated-token-account` crates。

```text
spl-token = { version="~3.2.0", features = [ "no-entrypoint" ] }
spl-associated-token-account = { version="=1.0.5", features = [ "no-entrypoint" ] }
```

在添加上述内容后，在你的控制台中运行 `cargo check` 命令，以让 cargo 解析你的依赖项，并确保你已经准备好继续进行。根据你的设置，你可能需要修改 crate 版本才能继续。

## 3. 在 `add_movie_review` 中添加必要的账户

因为我们希望用户在创建评论时铸造代币，所以在 `add_movie_review` 函数内添加铸造逻辑是合理的。由于我们将铸造代币，因此 `add_movie_review` 指令需要传入一些新的账户：

- `token_mint` - 代币的铸币厂地址
- `mint_auth` - 代币铸造的权限地址
- `user_ata` - 用户与该铸币厂相关联的代币账户（代币将铸造到此处）
- `token_program` - 代币程序的地址

我们将从向函数中迭代传入的账户的区域添加这些新账户：

```rust
// Inside add_movie_review
msg!("Adding movie review...");
msg!("Title: {}", title);
msg!("Rating: {}", rating);
msg!("Description: {}", description);

let account_info_iter = &mut accounts.iter();

let initializer = next_account_info(account_info_iter)?;
let pda_account = next_account_info(account_info_iter)?;
let pda_counter = next_account_info(account_info_iter)?;
let token_mint = next_account_info(account_info_iter)?;
let mint_auth = next_account_info(account_info_iter)?;
let user_ata = next_account_info(account_info_iter)?;
let system_program = next_account_info(account_info_iter)?;
let token_program = next_account_info(account_info_iter)?;
```

对于新功能，不需要额外的 `instruction_data`，因此不需要更改数据反序列化的方式。唯一需要的额外信息是额外的账户。

## 4. 在 `add_movie_review` 中为评论者铸造代币

在我们深入铸造逻辑之前，让我们在文件顶部导入 Token 程序的地址和常量 `LAMPORTS_PER_SOL`。

```rust
// Inside processor.rs
use solana_program::native_token::LAMPORTS_PER_SOL;
use spl_associated_token_account::get_associated_token_address;
use spl_token::{instruction::initialize_mint, ID as TOKEN_PROGRAM_ID};
```

现在我们可以继续处理实际铸造代币的逻辑了！我们将把这部分添加到 `add_movie_review` 函数的最后，在返回 `Ok(())` 之前。

铸造代币需要由铸币厂权限签名。由于程序需要能够铸造代币，所以铸造权限需要是程序可以签名的账户。换句话说，它需要是程序拥有的 PDA 账户。

我们还将构建我们的铸币厂，使得铸币厂是我们可以确定性地派生出来的 PDA 账户。这样，我们就可以始终验证传递给程序的 `token_mint` 账户是否是预期的账户。

让我们继续使用分别为 "token_mint" 和 "token_auth" 的种子，使用 `find_program_address` 函数派生代币铸造地址和铸造权限地址。

```rust
// Mint tokens here
msg!("deriving mint authority");
let (mint_pda, _mint_bump) = Pubkey::find_program_address(&[b"token_mint"], program_id);
let (mint_auth_pda, mint_auth_bump) =
    Pubkey::find_program_address(&[b"token_auth"], program_id);
```

接下来，我们将对传入程序的每个新账户执行安全检查。始终记得验证账户！

```rust
if *token_mint.key != mint_pda {
    msg!("Incorrect token mint");
    return Err(ReviewError::IncorrectAccountError.into());
}

if *mint_auth.key != mint_auth_pda {
    msg!("Mint passed in and mint derived do not match");
    return Err(ReviewError::InvalidPDA.into());
}

if *user_ata.key != get_associated_token_address(initializer.key, token_mint.key) {
    msg!("Incorrect token mint");
    return Err(ReviewError::IncorrectAccountError.into());
}

if *token_program.key != TOKEN_PROGRAM_ID {
    msg!("Incorrect token program");
    return Err(ReviewError::IncorrectAccountError.into());
}
```

最后，我们可以使用 `invoke_signed` 向 Token 程序的 `mint_to` 函数发出一个正确的 CPI，使用正确的账户。`spl_token` crate 提供了一个 `mint_to` 辅助函数来创建铸造指令。这很棒，因为这意味着我们不必手动从头开始构建整个指令。相反，我们只需传入函数所需的参数即可。以下是函数签名：

```rust
// Inside the token program, returns an Instruction object
pub fn mint_to(
    token_program_id: &Pubkey,
    mint_pubkey: &Pubkey,
    account_pubkey: &Pubkey,
    owner_pubkey: &Pubkey,
    signer_pubkeys: &[&Pubkey],
    amount: u64,
) -> Result<Instruction, ProgramError>
```

然后，我们提供 `token_mint`、`user_ata` 和 `mint_auth` 账户的副本。与本课程最相关的是，我们提供了用于找到 `token_auth` 地址的种子，包括 bump 种子。

```rust
msg!("Minting 10 tokens to User associated token account");
invoke_signed(
    // Instruction
    &spl_token::instruction::mint_to(
        token_program.key,
        token_mint.key,
        user_ata.key,
        mint_auth.key,
        &[],
        10*LAMPORTS_PER_SOL,
    )?,
    // Account_infos
    &[token_mint.clone(), user_ata.clone(), mint_auth.clone()],
    // Seeds
    &[&[b"token_auth", &[mint_auth_bump]]],
)?;

Ok(())
```

请注意，我们在这里使用的是 `invoke_signed` 而不是 `invoke`。Token 程序要求 `mint_auth` 账户为此交易签名。由于 `mint_auth` 账户是一个 PDA，只有它派生自的程序才能代表它签名。当调用 `invoke_signed` 时，Solana 运行时使用提供的种子和 bump 调用 `create_program_address`，然后将派生的地址与提供的所有 `AccountInfo` 对象的地址进行比较。如果其中一个地址与派生的地址匹配，runtime 就知道匹配的账户是该程序的一个 PDA，并且该程序正在为该账户签署这个交易。

在这一点上，`add_movie_review` 指令应该是完全可用的，并且在创建评论时将向评论者铸造十个代币。

## 5. 对 `add_comment` 进行重复的操作

我们对 `add_comment` 函数的更新几乎与上面对 `add_movie_review` 函数的操作相同。唯一的区别是，我们将一个意见能够铸造代币数量从十个更改为五个，以便对添加评论和意见进行加权。首先，使用与 `add_movie_review` 函数中相同的四个额外账户更新账户。

```rust
// Inside add_comment
let account_info_iter = &mut accounts.iter();

let commenter = next_account_info(account_info_iter)?;
let pda_review = next_account_info(account_info_iter)?;
let pda_counter = next_account_info(account_info_iter)?;
let pda_comment = next_account_info(account_info_iter)?;
let token_mint = next_account_info(account_info_iter)?;
let mint_auth = next_account_info(account_info_iter)?;
let user_ata = next_account_info(account_info_iter)?;
let system_program = next_account_info(account_info_iter)?;
let token_program = next_account_info(account_info_iter)?;
```

接下来，移动到 `add_comment` 函数的底部，就在 `Ok(())` 之前。然后派生铸币厂和铸造权限账户。请记住，两者都是从种子 "token_mint" 和 "token_authority" 派生出的 PDA。

```rust
// Mint tokens here
msg!("deriving mint authority");
let (mint_pda, _mint_bump) = Pubkey::find_program_address(&[b"token_mint"], program_id);
let (mint_auth_pda, mint_auth_bump) =
    Pubkey::find_program_address(&[b"token_auth"], program_id);
```

接下来，验证每个新账户是否是正确的账户。

```rust
if *token_mint.key != mint_pda {
    msg!("Incorrect token mint");
    return Err(ReviewError::IncorrectAccountError.into());
}

if *mint_auth.key != mint_auth_pda {
    msg!("Mint passed in and mint derived do not match");
    return Err(ReviewError::InvalidPDA.into());
}

if *user_ata.key != get_associated_token_address(commenter.key, token_mint.key) {
    msg!("Incorrect token mint");
    return Err(ReviewError::IncorrectAccountError.into());
}

if *token_program.key != TOKEN_PROGRAM_ID {
    msg!("Incorrect token program");
    return Err(ReviewError::IncorrectAccountError.into());
}
```

最后，使用 `invoke_signed` 向 Token 程序发送 `mint_to` 指令，向提出意见者发送五个代币。

```rust
msg!("Minting 5 tokens to User associated token account");
invoke_signed(
    // Instruction
    &spl_token::instruction::mint_to(
        token_program.key,
        token_mint.key,
        user_ata.key,
        mint_auth.key,
        &[],
        5 * LAMPORTS_PER_SOL,
    )?,
    // Account_infos
    &[token_mint.clone(), user_ata.clone(), mint_auth.clone()],
    // Seeds
    &[&[b"token_auth", &[mint_auth_bump]]],
)?;

Ok(())
```

## 6. 设置铸币厂

我们已经编写了所有需要的代码，以便为评论者和提出意见者铸造代币，但前提是在使用种子 "token_mint" 派生的 PDA 上存在一个铸币厂。为了使这个工作，我们将设置一个额外的指令来初始化铸币厂。它将被编写成只能调用一次，并且谁调用它并不特别重要。

鉴于在本课程中，我们已经多次强调了与 PDAs 和 CPIs 相关的所有概念，我们在这一部分的解释会比之前更少一些。首先，在 `instruction.rs` 中的 `MovieInstruction` 枚举中添加第四个指令变体 `InitializeMint`。

```rust
pub enum MovieInstruction {
    AddMovieReview {
        title: String,
        rating: u8,
        description: String,
    },
    UpdateMovieReview {
        title: String,
        rating: u8,
        description: String,
    },
    AddComment {
        comment: String,
    },
    InitializeMint,
}
```

确保将其添加到同一文件中 `unpack` 函数中的 `match` 语句中，位于变体 `3` 下。

```rust
impl MovieInstruction {
    pub fn unpack(input: &[u8]) -> Result<Self, ProgramError> {
        let (&variant, rest) = input
            .split_first()
            .ok_or(ProgramError::InvalidInstructionData)?;
        Ok(match variant {
            0 => {
                let payload = MovieReviewPayload::try_from_slice(rest).unwrap();
                Self::AddMovieReview {
                    title: payload.title,
                    rating: payload.rating,
                    description: payload.description,
                }
            }
            1 => {
                let payload = MovieReviewPayload::try_from_slice(rest).unwrap();
                Self::UpdateMovieReview {
                    title: payload.title,
                    rating: payload.rating,
                    description: payload.description,
                }
            }
            2 => {
                let payload = CommentPayload::try_from_slice(rest).unwrap();
                Self::AddComment {
                    comment: payload.comment,
                }
            }
            3 => Self::InitializeMint,
            _ => return Err(ProgramError::InvalidInstructionData),
        })
    }
}
```

在 `processor.rs` 文件中的 `process_instruction` 函数中，将新指令添加到 `match` 语句中，并调用一个名为 `initialize_token_mint` 的函数。

```rust
pub fn process_instruction(
    program_id: &Pubkey,
    accounts: &[AccountInfo],
    instruction_data: &[u8],
) -> ProgramResult {
    let instruction = MovieInstruction::unpack(instruction_data)?;
    match instruction {
        MovieInstruction::AddMovieReview {
            title,
            rating,
            description,
        } => add_movie_review(program_id, accounts, title, rating, description),
        MovieInstruction::UpdateMovieReview {
            title,
            rating,
            description,
        } => update_movie_review(program_id, accounts, title, rating, description),
        MovieInstruction::AddComment { comment } => add_comment(program_id, accounts, comment),
        MovieInstruction::InitializeMint => initialize_token_mint(program_id, accounts),
    }
}
```

最后，声明并实现 `initialize_token_mint` 函数。此函数将派生铸币厂和铸造权限 PDAs，创建铸币厂账户，然后初始化铸币厂。我们不会详细解释所有这些，但阅读代码是值得的，特别是考虑到铸币厂和初始化都涉及到 CPIs。再次强调，如果你需要对代币和铸造进行复习，请查看 [Token 程序课程](./token-program.md)。

```rust
pub fn initialize_token_mint(program_id: &Pubkey, accounts: &[AccountInfo]) -> ProgramResult {
    let account_info_iter = &mut accounts.iter();

    let initializer = next_account_info(account_info_iter)?;
    let token_mint = next_account_info(account_info_iter)?;
    let mint_auth = next_account_info(account_info_iter)?;
    let system_program = next_account_info(account_info_iter)?;
    let token_program = next_account_info(account_info_iter)?;
    let sysvar_rent = next_account_info(account_info_iter)?;

    let (mint_pda, mint_bump) = Pubkey::find_program_address(&[b"token_mint"], program_id);
    let (mint_auth_pda, _mint_auth_bump) =
        Pubkey::find_program_address(&[b"token_auth"], program_id);

    msg!("Token mint: {:?}", mint_pda);
    msg!("Mint authority: {:?}", mint_auth_pda);

    if mint_pda != *token_mint.key {
        msg!("Incorrect token mint account");
        return Err(ReviewError::IncorrectAccountError.into());
    }

    if *token_program.key != TOKEN_PROGRAM_ID {
        msg!("Incorrect token program");
        return Err(ReviewError::IncorrectAccountError.into());
    }

    if *mint_auth.key != mint_auth_pda {
        msg!("Incorrect mint auth account");
        return Err(ReviewError::IncorrectAccountError.into());
    }

    let rent = Rent::get()?;
    let rent_lamports = rent.minimum_balance(82);

    invoke_signed(
        &system_instruction::create_account(
            initializer.key,
            token_mint.key,
            rent_lamports,
            82,
            token_program.key,
        ),
        &[
            initializer.clone(),
            token_mint.clone(),
            system_program.clone(),
        ],
        &[&[b"token_mint", &[mint_bump]]],
    )?;

    msg!("Created token mint account");

    invoke_signed(
        &initialize_mint(
            token_program.key,
            token_mint.key,
            mint_auth.key,
            Option::None,
            9,
        )?,
        &[token_mint.clone(), sysvar_rent.clone(), mint_auth.clone()],
        &[&[b"token_mint", &[mint_bump]]],
    )?;

    msg!("Initialized token mint");

    Ok(())
}
```

## 7. 构建和部署

现在我们已经准备好构建和部署我们的程序了！你可以通过运行 `cargo build-bpf` 来构建程序，然后运行返回的命令，它应该类似于 `solana program deploy <PATH>`。

在开始测试是否添加评论或意见会发送代币之前，你需要初始化程序的铸币厂。你可以使用[此脚本](https://github.com/Unboxed-Software/solana-movie-token-client)来执行此操作。一旦你克隆了该存储库，请将 `index.ts` 中的 `PROGRAM_ID` 替换为你程序的ID。然后运行 `npm install`，然后运行 `npm start`。该脚本假设你正在部署到 Devnet。如果你正在本地部署，请确保根据情况调整脚本。

一旦你初始化了铸币厂，你就可以使用[电影评论前端](https://github.com/Unboxed-Software/solana-movie-frontend/tree/solution-add-tokens)来测试添加评论和意见。同样，该代码假设你在 Devnet 上，请相应地行事。

提交评论后，你应该在你的钱包中看到 10 个新的代币！当你添加意见时，你应该收到 5 个代币。由于我们没有为代币添加任何元数据，它们将没有 fancy 的名称或图像，但你明白了吧。

如果你需要更多时间来理解本课程的概念，或者在实践中遇到了困难，请随时[查看解决方案代码](https://github.com/Unboxed-Software/solana-movie-program/tree/solution-add-tokens)。请注意，此实验的解决方案位于 `solution-add-tokens` 分支上。

# 挑战

为了应用你在本课程中学到的关于 CPIs 的知识，考虑一下你如何将它们应用到学生介绍程序中。你可以做一些类似于我们在这里实验中所做的事情，给用户在介绍自己时铸造代币的功能。或者，如果你感到非常有雄心壮志，想一想你如何将到目前为止在课程中学到的所有知识都融合起来，从头开始创建一些全新的东西。

一个很好的例子是构建一个去中心化的 Stack Overflow。该程序可以使用代币来确定用户的总体评级，在问题被正确回答时铸造代币，允许用户投票赞成答案等等。所有这些都是可能的，而你现在已经掌握了足够的技能和知识，可以独立构建类似的项目了！

祝贺你完成了第四模块的学习！欢迎[分享一些快速反馈](https://airtable.com/shrOsyopqYlzvmXSC?prefill_Module=Module%204)，这样我们就可以继续改进课程。

## 完成实验了吗？

将你的代码推送到 GitHub，并[告诉我们你对这节课的看法](https://form.typeform.com/to/IPH0UGz7#answers-lesson=ade5d386-809f-42c2-80eb-a6c04c471f53)！