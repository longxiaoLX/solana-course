---
title: Solana 程序中的环境变量
objectives:
- 在 `Cargo.toml` 文件中定义程序特性
- 使用 Rust 的 `cfg` 属性根据启用或未启用的特性有条件地编译代码
- 使用 Rust 的 `cfg!` 宏根据启用或未启用的特性有条件地编译代码
- 创建一个仅限管理员的指令来设置一个程序账户，该账户可用于存储程序配置数值
---

# TL;DR

- 对于在链上程序中创建不同环境，目前没有现成的解决方案，但如果你具有创造性，可以实现类似于环境变量的功能。
- 你可以使用 Rust 特性 (`#[cfg(feature = ...)]`) 结合 **Rust 特性** 来运行不同的代码或提供不同的变量值。_这是在编译时发生的，不允许在程序部署后交换值_。
- 同样地，你可以使用 `cfg!` **宏** 根据启用的特性来编译不同的代码路径。
- 或者，你可以通过创建只能由程序升级权限访问的账户和指令，实现类似于环境变量的功能，这样可以在部署后修改。

# 概述

工程师在各种软件开发中面临的一个困难是编写可测试的代码并创建用于本地开发、测试、生产等的不同环境。

这在 Solana 程序开发中可能尤为困难。例如，想象创建一个 NFT 质押程序，每个质押的 NFT 每天奖励 10 个奖励代币。当测试在几百毫秒内运行，时间远远不足以赚取奖励时，如何测试领取奖励的能力呢？

传统的 Web 开发通过环境变量（environment variables）解决了部分问题，这些变量的值可以在每个不同的 "环境" 中有所不同。目前，在 Solana 程序中没有正式的环境变量概念。如果有的话，你可以在测试环境中设置奖励为每天 10,000,000 代币，这样测试领取奖励的能力将更容易进行测试。

幸运的是，如果你有创造性，你可以实现类似的功能。最好的方法可能是两者的结合：

1. Rust 特性标志（feature flags），允许你在构建命令中指定构建的 "环境"，并配合相应调整特定值的代码。
2. 仅管理员可访问的程序账户和指令，这些账户和指令只能由程序的升级权限访问。

## Rust 特性标志

创建环境的最简单方法之一是使用 Rust 特性。特性在程序的 `Cargo.toml` 文件的 `[features]` 表中定义。你可以为不同的用例定义多个特性。

```toml
[features]
feature-one = []
feature-two = []
```

需要注意的是，上述仅仅定义了一个特性。在测试程序时启用特性，你可以使用 `anchor test` 命令的 `--features` 标志。

```bash
anchor test -- --features "feature-one"
```

你也可以通过用逗号分隔它们来指定多个特性。

```bash
anchor test -- --features "feature-one", "feature-two"
```

### 使用 `cfg` 属性进行代码条件编译

有了定义的特性，你可以在代码中使用 `cfg` 属性来根据特性是否启用有条件地编译代码。这允许你在程序中包含或排除某些代码。

使用 `cfg` 属性的语法与其他属性宏相同：`#[cfg(feature=[FEATURE_HERE])]`。例如，以下代码在启用 `testing` 特性时编译函数 `function_for_testing`，否则编译 `function_when_not_testing`：

```rust
#[cfg(feature = "testing")]
fn function_for_testing() {
    // code that will be included only if the "testing" feature flag is enabled
}

#[cfg(not(feature = "testing"))]
fn function_when_not_testing() {
    // code that will be included only if the "testing" feature flag is not enabled
}
```

这允许你在编译时通过启用或禁用特性来启用或禁用 Anchor 程序中的某些功能。

可以想象希望使用这个功能来为不同的程序部署创建不同的 "环境"。例如，并非所有代币都在 Mainnet 和 Devnet 上都部署。因此，你可能会为 Mainnet 部署硬编码一个代币地址，但为 Devnet 和 Localnet 部署硬编码一个不同的地址。这样，你就可以在不需要对代码本身进行任何更改的情况下快速切换不同的环境。

下面的代码显示了一个使用 `cfg` 属性的 Anchor 程序示例，以在本地测试时包含不同的代币地址，而与其他部署不同：

```rust
use anchor_lang::prelude::*;
use anchor_spl::token::{Mint, Token, TokenAccount};

declare_id!("Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS");

#[cfg(feature = "local-testing")]
pub mod constants {
    use solana_program::{pubkey, pubkey::Pubkey};
    pub const USDC_MINT_PUBKEY: Pubkey = pubkey!("WaoKNLQVDyBx388CfjaVeyNbs3MT2mPgAhoCfXyUvg8");
}

#[cfg(not(feature = "local-testing"))]
pub mod constants {
    use solana_program::{pubkey, pubkey::Pubkey};
    pub const USDC_MINT_PUBKEY: Pubkey = pubkey!("EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v");
}

#[program]
pub mod test_program {
    use super::*;

    pub fn initialize_usdc_token_account(ctx: Context<Initialize>) -> Result<()> {
        Ok(())
    }
}

#[derive(Accounts)]
pub struct Initialize<'info> {
    #[account(
        init,
        payer = payer,
        token::mint = mint,
        token::authority = payer,
    )]
    pub token: Account<'info, TokenAccount>,
    #[account(address = constants::USDC_MINT_PUBKEY)]
    pub mint: Account<'info, Mint>,
    #[account(mut)]
    pub payer: Signer<'info>,
    pub token_program: Program<'info, Token>,
    pub system_program: Program<'info, System>,
    pub rent: Sysvar<'info, Rent>,
}
```

在这个例子中，`cfg` 属性被用来有条件地编译 `constants` 模块的两个不同实现。这使得程序可以根据是否启用 `local-testing` 特性来使用不同的值作为 `USDC_MINT_PUBKEY` 常量。

### 使用 `cfg!` 宏进行代码条件编译

与 Rust 中的 `cfg` 属性类似，`cfg!` **宏** 允许你在运行时检查某些配置标志的值。如果你想根据某些配置标志的值执行不同的代码路径，这可能会很有用。

你可以使用这个方法来绕过或调整我们之前提到的 NFT 质押应用中所需的基于时间的约束。在运行测试时，你可以执行提供比生产版本更高的质押奖励的代码。

要在 Anchor 程序中使用 `cfg!` 宏，只需将 `cfg!` 宏调用添加到相关的条件语句中：

```rust
#[program]
pub mod my_program {
    use super::*;

    pub fn test_function(ctx: Context<Test>) -> Result<()> {
        if cfg!(feature = "local-testing") {
            // This code will be executed only if the "local-testing" feature is enabled
            // ...
        } else {
            // This code will be executed only if the "local-testing" feature is not enabled
            // ...
        }
        // Code that should always be included goes here
        ...
        Ok(())
    }
}
```

在这个例子中，`test_function` 使用 `cfg!` 宏在运行时检查 `local-testing` 特性的值。如果启用了 `local-testing` 特性，则执行第一段代码路径。如果未启用 `local-testing` 特性，则执行第二段代码路径。

## 仅管理员指令

特征标志（feature flags）在编译时调整值和代码路径方面非常有用，但如果在部署程序后需要调整某些内容，则帮助不大。

例如，如果您的 NFT 抵押计划必须转变并使用不同的奖励代币，那么在不重新部署的情况下更新程序将是不可能的。如果只有程序管理员能够更新某些程序值的方法该多好... 好消息是，这是可能的！

首先，您需要将您的程序结构化，将您预期会更改的值存储在一个账户中，而不是硬编码到程序代码中。

接下来，您需要确保此账户只能由某个已知的程序权威（authority）或我们称之为管理员（admin）来更新。这意味着任何修改此账户上数据的指令都需要有限制，限制谁可以为该指令签名。从理论上讲，这听起来相当简单，但存在一个主要问题：程序如何知道谁是授权的管理员？

好吧，有几种解决方案，每种解决方案都有其自己的优点和缺点：

1. 在仅管理员指令约束中硬编码一个可用于的管理员公钥。
2. 将程序的升级权限设为管理员。
3. 将管理员存储在配置账户中，并在 `initialize` 指令中设置第一个管理员。

### 创建配置账户

第一步是向您的程序添加我们将称之为“配置（config）”账户。您可以根据自己的需求进行定制，但我们建议使用单个全局 PDA（Program Derived Address）。在 Anchor 中，这意味着创建一个账户结构，并使用单个种子来派生账户的地址。

```rust
pub const SEED_PROGRAM_CONFIG: &[u8] = b"program_config";

#[account]
pub struct ProgramConfig {
    reward_token: Pubkey,
    rewards_per_day: u64,
}
```

上面的示例显示了我们在整个课程中引用的 NFT 抵押计划示例的假想配置账户。它存储了代表应该用于奖励的代币以及每天抵押的代币数量的数据。

有了配置账户定义后，只需确保您的其余代码在使用这些值时引用该账户。这样，如果账户中的数据发生变化，程序就会相应地进行调整。

### 将配置更新限制为硬编码的管理员

您需要一种方法来初始化和更新配置账户数据。这意味着您需要有一个或多个只有管理员才能调用的指令。最简单的方法是在您的代码中硬编码一个管理员的公钥，然后在您的指令的账户验证中添加一个简单的签名者检查，将签名者与此公钥进行比较。

在 Anchor 中，将 `update_program_config` 指令限制为只能由硬编码的管理员使用可能如下所示：

```rust
#[program]
mod my_program {
    pub fn update_program_config(
        ctx: Context<UpdateProgramConfig>,
        reward_token: Pubkey,
        rewards_per_day: u64
    ) -> Result<()> {
        ctx.accounts.program_config.reward_token = reward_token;
        ctx.accounts.program_config.rewards_per_day = rewards_per_day;

        Ok(())
    }
}

pub const SEED_PROGRAM_CONFIG: &[u8] = b"program_config";

#[constant]
pub const ADMIN_PUBKEY: Pubkey = pubkey!("ADMIN_WALLET_ADDRESS_HERE");

#[derive(Accounts)]
pub struct UpdateProgramConfig<'info> {
    #[account(mut, seeds = SEED_PROGRAM_CONFIG, bump)]
    pub program_config: Account<'info, ProgramConfig>,
    #[account(constraint = authority.key() == ADMIN_PUBKEY)]
    pub authority: Signer<'info>,
}
```

甚至在指令逻辑执行之前，将执行一个检查，确保指令的签名者与硬编码的 `ADMIN_PUBKEY` 匹配。请注意，上面的示例没有显示初始化配置账户的指令，但它应该具有类似的约束，以确保攻击者无法使用意外的值初始化账户。

虽然这种方法可行，但这意味着除了跟踪程序的升级权限外，还需要跟踪管理员钱包。通过几行代码，您可以简单地限制一个指令只能由升级权限调用。唯一棘手的部分是获取程序的升级权限进行比较。

### 将配置更新限制为程序的升级权限

幸运的是，每个程序都有一个程序数据账户，对应于 Anchor 的 `ProgramData` 账户类型，并具有 `upgrade_authority_address` 字段。程序本身将此账户的地址存储在其数据中的 `programdata_address` 字段中。

因此，除了硬编码管理员示例中指令所需的两个账户外，该指令还需要 `program` 和 `program_data` 账户。

然后，账户需要以下约束：

1. 对 `program` 的约束，确保提供的 `program_data` 账户与程序的 `programdata_address` 字段匹配。
2. 对 `program_data` 账户的约束，确保指令的签名者与 `program_data` 账户的 `upgrade_authority_address` 字段匹配。

完成后，代码如下所示：

```rust
...

#[derive(Accounts)]
pub struct UpdateProgramConfig<'info> {
    #[account(mut, seeds = SEED_PROGRAM_CONFIG, bump)]
    pub program_config: Account<'info, ProgramConfig>,
    #[account(constraint = program.programdata_address()? == Some(program_data.key()))]
    pub program: Program<'info, MyProgram>,
    #[account(constraint = program_data.upgrade_authority_address == Some(authority.key()))]
    pub program_data: Account<'info, ProgramData>,
    pub authority: Signer<'info>,
}
```

再次强调，上面的示例没有显示初始化配置账户的指令，但它应该具有相同的约束，以确保攻击者无法使用意外的值初始化账户。

如果这是您第一次听说程序数据账户，建议阅读[此Notion文档](https://www.notion.so/29780c48794c47308d5f138074dd9838) 关于程序部署的内容。

### 将配置更新限制为提供的管理员

前面两个选项都相当安全，但也不够灵活。如果您想将管理员更改为其他人怎么办？为此，您可以将管理员存储在配置账户上。

```rust
pub const SEED_PROGRAM_CONFIG: &[u8] = b"program_config";

#[account]
pub struct ProgramConfig {
    admin: Pubkey,
    reward_token: Pubkey,
    rewards_per_day: u64,
}
```

然后，您可以通过与配置账户的 `admin` 字段进行签名者检查来限制您的“更新”指令。

```rust
...

pub const SEED_PROGRAM_CONFIG: &[u8] = b"program_config";

#[derive(Accounts)]
pub struct UpdateProgramConfig<'info> {
    #[account(mut, seeds = SEED_PROGRAM_CONFIG, bump)]
    pub program_config: Account<'info, ProgramConfig>,
    #[account(constraint = authority.key() == program_config.admin)]
    pub authority: Signer<'info>,
}
```

这里有一个要注意的地方：在部署程序和初始化配置账户之间的时间内，_没有管理员_。这意味着初始化配置账户的指令不能被限制为仅允许管理员调用者。这意味着可能会被攻击者调用，试图将自己设置为管理员。

虽然听起来很糟糕，但实际上这只意味着在您自己初始化配置账户并验证账户上列出的管理员是否是您预期的账户之前，您不应将您的程序视为“已初始化”。如果您的部署脚本部署后立即调用 `initialize`，那么攻击者甚至不太可能知道您的程序的存在，更不用说试图将自己设为管理员了。如果某些不幸的事情发生，有人“拦截”了您的程序，您可以使用升级权限关闭程序并重新部署。

# 实验

现在让我们一起来尝试一下。在这个实验室中，我们将使用一个简单的程序来启用 USDC 支付。该程序收取一小笔费用来促成转账。请注意，这有点牵强，因为您可以在没有中间合约的情况下进行直接转账，但这模拟了一些复杂的 DeFi 程序的工作方式。

我们将在测试我们的程序时快速了解到，通过管理员控制的配置账户和一些特征标志，程序可以获得更灵活的好处。

## 1. 初始设置

从 [此存储库的 starter 分支](https://github.com/Unboxed-Software/solana-admin-instructions/tree/starter) 下载初始代码。该代码包含一个带有单个指令和单个测试的程序，位于 `tests` 目录中。

让我们快速浏览一下程序的工作原理。

`lib.rs` 文件包括一个用于 USDC 地址的常量和一个单独的 `payment` 指令。`payment` 指令简单地调用了 `instructions/payment.rs` 文件中的 `payment_handler` 函数，该函数包含了指令逻辑。

`instructions/payment.rs` 文件包含了 `payment_handler` 函数以及代表 `payment` 指令所需账户的 `Payment` 账户验证结构体。`payment_handler` 函数从支付金额中计算出 1% 的手续费，将手续费转移到指定的代币账户，并将剩余金额转移到支付接收者。

最后，`tests` 目录中有一个单独的测试文件 `config.ts`，它简单地调用了 `payment` 指令，并断言相应的代币账户余额已经按预期进行了借记和贷记。

在继续之前，花几分钟时间熟悉这些文件及其内容。

## 2. 运行现有测试

让我们从运行现有测试开始。

确保使用 `yarn` 或 `npm install` 安装 `package.json` 文件中列出的依赖项。然后确保运行 `anchor keys list`，将程序的公钥打印到控制台上。这取决于您本地拥有的密钥对，因此您需要更新 `lib.rs` 和 `Anchor.toml` 来使用*您的*密钥。

最后，运行 `anchor test` 开始测试。它应该会失败，并显示以下输出：

```
Error: failed to send transaction: Transaction simulation failed: Error processing Instruction 0: incorrect program id for instruction
```

这个错误的原因是我们试图使用在程序的 `lib.rs` 文件中硬编码的主网 USDC 代币地址，但该代币在本地环境中并不存在。

## 3. 添加一个 `local-testing` 特性

为了解决这个问题，我们需要一个可以在本地使用的代币，并且将其硬编码到程序中。由于本地环境在测试期间经常被重置，您需要存储一个密钥对，以便每次都可以重新创建相同的代币地址。

此外，您不希望在本地和主网版本之间更改硬编码的地址，因为这可能会引入人为错误（而且很烦人）。因此，我们将创建一个 `local-testing` 特性，当启用时，程序将使用我们的本地代币，否则将使用生产环境的 USDC 代币。

运行 `solana-keygen grind` 生成一个新的密钥对。运行以下命令生成一个以 "env" 开头的公钥。

```
solana-keygen grind --starts-with env:1
```

一旦找到密钥对，您应该会看到类似以下的输出：

```
Wrote keypair to env9Y3szLdqMLU9rXpEGPqkjdvVn8YNHtxYNvCKXmHe.json
```

密钥对被写入到您的工作目录中的一个文件中。现在我们有了一个占位的 USDC 地址，让我们修改 `lib.rs` 文件。使用 `cfg` 属性根据 `local-testing` 特性是否启用来定义 `USDC_MINT_PUBKEY` 常量。请记住，将 `local-testing` 特性的 `USDC_MINT_PUBKEY` 常量设置为前面步骤中生成的常量，而不是复制以下的常量。

```rust
use anchor_lang::prelude::*;
use solana_program::{pubkey, pubkey::Pubkey};
mod instructions;
use instructions::*;

declare_id!("BC3RMBvVa88zSDzPXnBXxpnNYCrKsxnhR3HwwHhuKKei");

#[cfg(feature = "local-testing")]
#[constant]
pub const USDC_MINT_PUBKEY: Pubkey = pubkey!("...");

#[cfg(not(feature = "local-testing"))]
#[constant]
pub const USDC_MINT_PUBKEY: Pubkey = pubkey!("EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v");

#[program]
pub mod config {
    use super::*;

    pub fn payment(ctx: Context<Payment>, amount: u64) -> Result<()> {
        instructions::payment_handler(ctx, amount)
    }
}
```

接下来，在位于 `/programs` 的 `Cargo.toml` 文件中添加 `local-testing` 特性。

```
[features]
...
local-testing = []
```

接下来，更新 `config.ts` 测试文件以使用生成的密钥对创建一个代币。首先删除 `mint` 常量。

```typescript
const mint = new anchor.web3.PublicKey(
    "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
);
```

接下来，更新测试以使用密钥对创建一个代币，这样我们就可以在每次运行测试时重用相同的代币地址。请记住，用前面步骤中生成的文件名替换下面的文件名。

```typescript
let mint: anchor.web3.PublicKey

before(async () => {
  let data = fs.readFileSync(
    "env9Y3szLdqMLU9rXpEGPqkjdvVn8YNHtxYNvCKXmHe.json"
  )

  let keypair = anchor.web3.Keypair.fromSecretKey(
    new Uint8Array(JSON.parse(data))
  )

  const mint = await spl.createMint(
    connection,
    wallet.payer,
    wallet.publicKey,
    null,
    0,
    keypair
  )
...
```

最后，使用启用了 `local-testing` 特性运行测试。

```
anchor test -- --features "local-testing"
```

您应该会看到以下输出：

```
config
  ✔ Payment completes successfully (406ms)


1 passing (3s)
```

嗯，就是这样，您使用特性在不同的环境中运行了两条不同的代码路径。

## 4. 程序配置

特性对于在编译时设置不同的值非常有用，但如果您想要能够动态更新程序使用的费率百分比呢？让我们通过创建一个程序配置账户来实现这一点，从而使我们能够在不升级程序的情况下更新费率。

让我们首先更新 `lib.rs` 文件：

1. 包含一个 `SEED_PROGRAM_CONFIG` 常量，该常量将用于生成程序配置账户的PDA。
2. 包含一个 `ADMIN` 常量，该常量将在初始化程序配置账户时用作约束。运行 `solana address` 命令以获取您的地址，然后将其用作常量的值。
3. 包含一个 `state` 模块，我们将在稍后实现。
4. 包含 `initialize_program_config` 和 `update_program_config` 指令以及对它们的“handlers”的调用，这两者我们将在另一个步骤中实现。

```rust
use anchor_lang::prelude::*;
use solana_program::{pubkey, pubkey::Pubkey};
mod instructions;
mod state;
use instructions::*;

declare_id!("BC3RMBvVa88zSDzPXnBXxpnNYCrKsxnhR3HwwHhuKKei");

#[cfg(feature = "local-testing")]
#[constant]
pub const USDC_MINT_PUBKEY: Pubkey = pubkey!("envgiPXWwmpkHFKdy4QLv2cypgAWmVTVEm71YbNpYRu");

#[cfg(not(feature = "local-testing"))]
#[constant]
pub const USDC_MINT_PUBKEY: Pubkey = pubkey!("EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v");

pub const SEED_PROGRAM_CONFIG: &[u8] = b"program_config";

#[constant]
pub const ADMIN: Pubkey = pubkey!("...");

#[program]
pub mod config {
    use super::*;

    pub fn initialize_program_config(ctx: Context<InitializeProgramConfig>) -> Result<()> {
        instructions::initialize_program_config_handler(ctx)
    }

    pub fn update_program_config(
        ctx: Context<UpdateProgramConfig>,
        new_fee: u64,
    ) -> Result<()> {
        instructions::update_program_config_handler(ctx, new_fee)
    }

    pub fn payment(ctx: Context<Payment>, amount: u64) -> Result<()> {
        instructions::payment_handler(ctx, amount)
    }
}
```

## 5. 程序配置状态

接下来，让我们为 `ProgramConfig` 状态定义结构。此账户将存储管理员、收取费用的代币账户以及费率。我们还将指定存储此结构所需的字节数。

在 `/src` 目录中创建一个名为 `state.rs` 的新文件，并添加以下代码。

```rust
use anchor_lang::prelude::*;

#[account]
pub struct ProgramConfig {
    pub admin: Pubkey,
    pub fee_destination: Pubkey,
    pub fee_basis_points: u64,
}

impl ProgramConfig {
    pub const LEN: usize = 8 + 32 + 32 + 8;
}
```

## 6. 添加初始化程序配置账户指令

现在让我们为初始化程序配置账户创建指令逻辑。它应该只能由使用 `ADMIN` 密钥签名的交易调用，并且应该在 `ProgramConfig` 账户上设置所有属性。

在路径 `/src/instructions/program_config` 下创建一个名为 `program_config` 的文件夹。该文件夹将存储与程序配置账户相关的所有指令。

在 `program_config` 文件夹中，创建一个名为 `initialize_program_config.rs` 的文件，并添加以下代码。

```rust
use crate::state::ProgramConfig;
use crate::ADMIN;
use crate::SEED_PROGRAM_CONFIG;
use crate::USDC_MINT_PUBKEY;
use anchor_lang::prelude::*;
use anchor_spl::token::TokenAccount;

#[derive(Accounts)]
pub struct InitializeProgramConfig<'info> {
    #[account(init, seeds = [SEED_PROGRAM_CONFIG], bump, payer = authority, space = ProgramConfig::LEN)]
    pub program_config: Account<'info, ProgramConfig>,
    #[account( token::mint = USDC_MINT_PUBKEY)]
    pub fee_destination: Account<'info, TokenAccount>,
    #[account(mut, address = ADMIN)]
    pub authority: Signer<'info>,
    pub system_program: Program<'info, System>,
}

pub fn initialize_program_config_handler(ctx: Context<InitializeProgramConfig>) -> Result<()> {
    ctx.accounts.program_config.admin = ctx.accounts.authority.key();
    ctx.accounts.program_config.fee_destination = ctx.accounts.fee_destination.key();
    ctx.accounts.program_config.fee_basis_points = 100;
    Ok(())
}
```

## 7. 添加更新程序配置费率指令

接下来，实现更新配置账户的指令逻辑。该指令应要求签名者与 `program_config` 账户中存储的 `admin` 匹配。

在 `program_config` 文件夹中，创建一个名为 `update_program_config.rs` 的文件，并添加以下代码。

```rust
use crate::state::ProgramConfig;
use crate::SEED_PROGRAM_CONFIG;
use crate::USDC_MINT_PUBKEY;
use anchor_lang::prelude::*;
use anchor_spl::token::TokenAccount;

#[derive(Accounts)]
pub struct UpdateProgramConfig<'info> {
    #[account(mut, seeds = [SEED_PROGRAM_CONFIG], bump)]
    pub program_config: Account<'info, ProgramConfig>,
    #[account( token::mint = USDC_MINT_PUBKEY)]
    pub fee_destination: Account<'info, TokenAccount>,
    #[account(
        mut,
        address = program_config.admin,
    )]
    pub admin: Signer<'info>,
    /// CHECK: arbitrarily assigned by existing admin
    pub new_admin: UncheckedAccount<'info>,
}

pub fn update_program_config_handler(
    ctx: Context<UpdateProgramConfig>,
    new_fee: u64,
) -> Result<()> {
    ctx.accounts.program_config.admin = ctx.accounts.new_admin.key();
    ctx.accounts.program_config.fee_destination = ctx.accounts.fee_destination.key();
    ctx.accounts.program_config.fee_basis_points = new_fee;
    Ok(())
}
```

## 8. 添加 mod.rs 并更新 instructions.rs

接下来，让我们暴露我们创建的指令处理程序，以便 `lib.rs` 的调用不会显示错误。首先，在 `program_config` 文件夹中添加一个名为 `mod.rs` 的文件。添加以下代码以使两个模块 `initialize_program_config` 和 `update_program_config` 可访问。

```rust
mod initialize_program_config;
pub use initialize_program_config::*;

mod update_program_config;
pub use update_program_config::*;
```

现在，请更新路径为 `/src/instructions.rs` 的 `instructions.rs` 文件。添加下面的代码以使两个模块 `program_config` 和 `payment` 可访问。

```rust
mod program_config;
pub use program_config::*;

mod payment;
pub use payment::*;
```

## 9. 更新支付指令

最后，让我们更新支付指令，检查指令中的 `fee_destination` 账户是否与程序配置账户中存储的 `fee_destination` 匹配。然后，根据程序配置账户中存储的 `fee_basis_point` 更新指令的费用计算。

```rust
use crate::state::ProgramConfig;
use crate::SEED_PROGRAM_CONFIG;
use crate::USDC_MINT_PUBKEY;
use anchor_lang::prelude::*;
use anchor_spl::token::{self, Token, TokenAccount};

#[derive(Accounts)]
pub struct Payment<'info> {
    #[account(
        seeds = [SEED_PROGRAM_CONFIG],
        bump,
        has_one = fee_destination
    )]
    pub program_config: Account<'info, ProgramConfig>,
    #[account(
        mut,
        token::mint = USDC_MINT_PUBKEY
    )]
    pub fee_destination: Account<'info, TokenAccount>,
    #[account(
        mut,
        token::mint = USDC_MINT_PUBKEY
    )]
    pub sender_token_account: Account<'info, TokenAccount>,
    #[account(
        mut,
        token::mint = USDC_MINT_PUBKEY
    )]
    pub receiver_token_account: Account<'info, TokenAccount>,
    pub token_program: Program<'info, Token>,
    #[account(mut)]
    pub sender: Signer<'info>,
}

pub fn payment_handler(ctx: Context<Payment>, amount: u64) -> Result<()> {
    let fee_amount = amount
        .checked_mul(ctx.accounts.program_config.fee_basis_points)
        .unwrap()
        .checked_div(10000)
        .unwrap();
    let remaining_amount = amount.checked_sub(fee_amount).unwrap();

    msg!("Amount: {}", amount);
    msg!("Fee Amount: {}", fee_amount);
    msg!("Remaining Transfer Amount: {}", remaining_amount);

    token::transfer(
        CpiContext::new(
            ctx.accounts.token_program.to_account_info(),
            token::Transfer {
                from: ctx.accounts.sender_token_account.to_account_info(),
                authority: ctx.accounts.sender.to_account_info(),
                to: ctx.accounts.fee_destination.to_account_info(),
            },
        ),
        fee_amount,
    )?;

    token::transfer(
        CpiContext::new(
            ctx.accounts.token_program.to_account_info(),
            token::Transfer {
                from: ctx.accounts.sender_token_account.to_account_info(),
                authority: ctx.accounts.sender.to_account_info(),
                to: ctx.accounts.receiver_token_account.to_account_info(),
            },
        ),
        remaining_amount,
    )?;

    Ok(())
}
```

## 10. 测试

现在我们已经完成了新程序配置结构和指令的实现，让我们开始测试我们更新后的程序。首先，将程序配置账户的 PDA 添加到测试文件中。

```typescript
describe("config", () => {
  ...
  const programConfig = findProgramAddressSync(
    [Buffer.from("program_config")],
    program.programId
  )[0]
...
```

接下来，更新测试文件，添加三个更多的测试，分别测试：

1. 程序配置账户是否正确初始化
2. 支付指令是否按预期功能
3. 管理员能够成功更新配置账户
4. 除管理员外，其他人无法更新配置账户

第一个测试初始化程序配置账户，并验证正确的费用是否设置，以及程序配置账户上存储的正确管理员。

```typescript
it("Initialize Program Config Account", async () => {
  const tx = await program.methods
    .initializeProgramConfig()
    .accounts({
      programConfig: programConfig,
      feeDestination: feeDestination,
      authority: wallet.publicKey,
      systemProgram: anchor.web3.SystemProgram.programId,
    })
    .rpc()

  assert.strictEqual(
    (
      await program.account.programConfig.fetch(programConfig)
    ).feeBasisPoints.toNumber(),
    100
  )
  assert.strictEqual(
    (
      await program.account.programConfig.fetch(programConfig)
    ).admin.toString(),
    wallet.publicKey.toString()
  )
})
```

第二个测试验证支付指令是否正确运行，包括将费用发送到费用目标账户，并将剩余余额转移到接收方。在这里，我们更新现有的测试以包括 `programConfig` 账户。

```typescript
it("Payment completes successfully", async () => {
  const tx = await program.methods
    .payment(new anchor.BN(10000))
    .accounts({
      programConfig: programConfig,
      feeDestination: feeDestination,
      senderTokenAccount: senderTokenAccount,
      receiverTokenAccount: receiverTokenAccount,
      sender: sender.publicKey,
    })
    .transaction()

  await anchor.web3.sendAndConfirmTransaction(connection, tx, [sender])

  assert.strictEqual(
    (await connection.getTokenAccountBalance(senderTokenAccount)).value
      .uiAmount,
    0
  )

  assert.strictEqual(
    (await connection.getTokenAccountBalance(feeDestination)).value.uiAmount,
    100
  )

  assert.strictEqual(
    (await connection.getTokenAccountBalance(receiverTokenAccount)).value
      .uiAmount,
    9900
  )
})
```

第三个测试尝试更新程序配置账户上的费用，这应该是成功的。

```typescript
it("Update Program Config Account", async () => {
  const tx = await program.methods
    .updateProgramConfig(new anchor.BN(200))
    .accounts({
      programConfig: programConfig,
      admin: wallet.publicKey,
      feeDestination: feeDestination,
      newAdmin: sender.publicKey,
    })
    .rpc()

  assert.strictEqual(
    (
      await program.account.programConfig.fetch(programConfig)
    ).feeBasisPoints.toNumber(),
    200
  )
})
```

第四个测试尝试更新程序配置账户上的费用，其中管理员不是存储在程序配置账户上的管理员，这应该会失败。

```typescript
it("Update Program Config Account with unauthorized admin (expect fail)", async () => {
  try {
    const tx = await program.methods
      .updateProgramConfig(new anchor.BN(300))
      .accounts({
        programConfig: programConfig,
        admin: sender.publicKey,
        feeDestination: feeDestination,
        newAdmin: sender.publicKey,
      })
      .transaction()

    await anchor.web3.sendAndConfirmTransaction(connection, tx, [sender])
  } catch (err) {
    expect(err)
  }
})
```

最后，使用以下命令运行测试：

```
anchor test -- --features "local-testing"
```

您应该会看到以下输出：

```
config
  ✔ Initialize Program Config Account (199ms)
  ✔ Payment completes successfully (405ms)
  ✔ Update Program Config Account (403ms)
  ✔ Update Program Config Account with unauthorized admin (expect fail)

4 passing (8s)
```

这就是全部内容了！您已经使得程序在今后的工作中变得更加容易。如果您想查看最终的解决方案代码，可以在[相同的存储库](https://github.com/Unboxed-Software/solana-admin-instructions/tree/solution)的 `solution` 分支中找到。

# 挑战

现在是您独立完成一些工作的时候了。我们提到可以使用程序的升级权限作为初始管理员。请继续更新实验的 `initialize_program_config`，以便只有升级权限才能调用它，而不是硬编码 `ADMIN`。

请注意，当在本地网络上运行 `anchor test` 命令时，它会启动一个新的测试验证器，使用 `solana-test-validator`。这个测试验证器使用了一个不可升级的加载器（non-upgradeable loader）。不可升级的加载器会导致当验证器启动时，程序的 `program_data` 账户未被初始化。您会从课程中记得，这个账户是我们如何从程序中访问升级权限的。

为了解决这个问题，您可以在测试文件中添加一个 `deploy` 函数，该函数运行使用可升级加载器的程序部署命令。要使用它，请运行 `anchor test --skip-deploy`，并在测试中调用 `deploy` 函数，在测试验证器启动后运行部署命令。

```typescript
import { execSync } from "child_process"

...

const deploy = () => {
  const deployCmd = `solana program deploy --url localhost -v --program-id $(pwd)/target/deploy/config-keypair.json $(pwd)/target/deploy/config.so`
  execSync(deployCmd)
}

...

before(async () => {
  ...
  deploy()
})
```

例如，运行带有特性的测试的命令如下所示：

```
anchor test --skip-deploy -- --features "local-testing"
```

尝试独立完成这个任务，但如果遇到困难，请随时参考[相同存储库](https://github.com/Unboxed-Software/solana-admin-instructions/tree/challenge)的 `challenge` 分支，查看一个可能的解决方案。

## 完成了实验了吗？

将您的代码推送到 GitHub，并[告诉我们您对这节课的想法](https://form.typeform.com/to/IPH0UGz7#answers-lesson=02a7dab7-d9c1-495b-928c-a4412006ec20)!