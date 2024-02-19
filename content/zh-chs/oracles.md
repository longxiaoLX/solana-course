---
title: 预言机和预言机网络
objectives:
- 解释为什么链上程序无法自行获取现实世界的数据
- 解释预言机如何解决链上获取现实世界数据的问题
- 解释激励预言机网络如何增强数据的可信度
- 有效权衡使用各种类型的预言机的利弊
- 使用预言机从链上程序获取现实世界的数据
---

- 预言机是向区块链网络提供外部数据的服务。
- Solana 上有两个主要的预言机提供者：**Switchboard** 和 **Pyth**。
- 您可以构建自己的预言机来创建自定义数据源。
- 在选择数据源提供者时必须谨慎。

# 概述.

预言机（oracles）是向区块链网络提供外部数据的服务。区块链本质上是封闭的环境，对外部世界一无所知。这一限制固有地限制了去中心化应用（dApps）的用例。预言机通过创建一种去中心化的方式将现实世界的数据带入链上，为解决这一限制提供了解决方案。

预言机可以提供几乎任何类型的链上数据。例如：

- 体育赛事结果
- 天气数据
- 政治选举结果
- 市场数据
- 随机性

虽然具体实现可能因区块链而异，但通常预言机的工作方式如下：

1. 数据来源于链下。
2. 该数据以交易的形式发布到链上，并存储在一个账户中。
3. 程序可以读取存储在账户中的数据，并在其逻辑中使用。

本课程将介绍预言机的基本工作原理、Solana 上的预言机现状以及如何在 Solana 开发中有效使用预言机。

## 信任与预言机网络

预言机需要克服的主要障碍之一是信任问题。由于区块链执行不可逆的金融交易，开发者和用户都需要知道他们可以信任预言机数据的有效性和准确性。信任预言机的第一步是了解其实现方式。

广义上讲，有三种实现类型：

1. 单一的、中心化的预言机在链上发布数据。
    1. 优点：简单明了；有一个真实的数据来源。
    2. 缺点：没有什么阻止预言机提供者提供不准确的数据。
2. 一组预言机发布数据，并使用共识机制确定最终结果。
    1. 优点：共识机制减少了不良数据被推送到链上的可能性。
    2. 缺点：无法阻止不良行为者发布错误数据并试图影响共识。
3. 具有某种权益证明机制的预言机网络。即要求预言机抵押代币以参与共识机制。在每次响应中，如果一个预言机偏离了接受范围的结果，其抵押将被协议收取，他们将不能再报告。
    1. 优点：确保没有单一的预言机可以对最终结果产生太大的影响，同时激励诚实和准确的行为。
    2. 缺点：构建去中心化网络具有挑战性，需要正确设置激励措施，并足够吸引参与等等。

根据预言机的使用案例，以上任何解决方案都可能是正确的方法。例如，您可能完全愿意参与那些利用中心化预言机向链上发布游戏信息的区块链游戏。

另一方面，您可能不太愿意相信为交易应用程序提供价格信息的中心化预言机。

您可能会创建许多独立的预言机来为自己的应用程序提供链下信息访问的途径。然而，这些预言机不太可能被更广泛的社区使用，因为去中心化是其核心原则。您也应该对使用中心化的第三方预言机保持谨慎。

在理想的情况下，所有重要和/或有价值的数据都将通过一个高效的预言机网络和可信赖的权益证明共识机制提供到链上。通过引入抵押机制，预言机提供者有充分的动机确保其数据准确性，以保持其抵押资金。

即使预言机网络声称拥有这样的共识机制，也务必了解使用该网络所涉及的风险。如果下游应用程序所涉及的总价值大于预言机的分配抵押金额，预言机仍然可能有足够的动机串通作弊。

您的工作是了解预言机网络的配置，并判断它们是否可信。一般来说，预言机应仅用于非关键任务，应考虑最坏情况。

## Solana 上的预言机

[Pyth](https://pyth.network) 和 [Switchboard](https://switchboard.xyz) 是今天 Solana 上的两个主要预言机提供者。它们各自独特，并遵循略有不同的设计选择。

**Pyth** 主要专注于来自一流金融机构发布的金融数据。Pyth 的数据提供者发布市场数据更新。然后，Pyth 程序将这些更新进行聚合，并发布到链上。Pyth 获取的数据并非完全去中心化，因为只有经批准的数据提供者才能发布数据。Pyth 的卖点在于，其数据直接由平台审核，并来源于金融机构，确保了更高的质量。

**Switchboard** 是一个完全去中心化的预言机网络，并提供各种类型的数据。您可以在[它们的网站](https://app.switchboard.xyz/solana/devnet/explore)上查看所有数据源。此外，任何人都可以运行 Switchboard 预言机，任何人都可以使用它们的数据。这意味着您需要对数据源进行认真研究。我们稍后会更详细地讨论该如何选择。

Switchboard 采用了前文第三种选项中描述的权益加权预言机网络的变体。它通过引入所谓的 TEE（可信执行环境）来实现这一点。TEE 是与系统其余部分隔离的安全环境，可以在其中执行敏感代码。简单来说，给定一个程序和一个输入，TEE 可以执行并生成一个输出以及一个证明。如果您想了解更多关于 TEE 的信息，请阅读 [Switchboard 的文档](https://docs.switchboard.xyz/functions)。

通过在权益加权预言机之上引入 TEE，Switchboard 能够验证每个预言机的软件，以允许其参与网络。如果预言机运营商恶意行为并试图更改已批准代码的运行，数据引用验证将失败。这使得 Switchboard 预言机能够进行超出量化价值报告的操作，例如函数运行链下的自定义和保密计算。

## Switchboard 预言机

Switchboard 预言机使用数据源（data feeds）在 Solana 上存储数据。这些数据源也称为聚合器（aggregators），每个都是一系列作业（jobs）的集合，这些作业被聚合以产生单个结果。这些聚合器在链上表示为由 Switchboard 程序管理的普通 Solana 账户。当一个预言机更新时，它会直接将数据写入这些账户。让我们了解一下一些术语，以理解 Switchboard 的工作原理：

- **[聚合器 Aggregator（数据源 Data Feed）](https://github.com/switchboard-xyz/sbv2-solana/blob/0b5e0911a1851f9ca37042e6ff88db4cd840067b/rust/switchboard-solana/src/oracle_program/accounts/aggregator.rs#L60)** - 包含数据源的配置信息，规定数据源链上更新行为如何从其分配的外部资源中，获取请求（get requested）、更新（updated）和解析。聚合器是由 Switchboard Solana 程序拥有的账户，是数据在链上发布的位置。
- **[作业（Job）](https://github.com/switchboard-xyz/sbv2-solana/blob/0b5e0911a1851f9ca37042e6ff88db4cd840067b/rust/switchboard-solana/src/oracle_program/accounts/job.rs)** - 每个数据源应该对应一个作业账户。作业账户是一组 Switchboard 任务的集合，用于指示预言机如何获取和转换数据。换句话说，它存储了针对特定数据源如何从链下获取数据的蓝图。
- **预言机（Oracle）** - 一个独立的程序，位于互联网和区块链之间，促进信息的流动。预言机读取数据源的任务定义，计算结果，并在链上提交其响应。
- **预言机队列（Oracle Queue）** - 一组预言机，按照轮询方式分配更新请求。队列中的预言机必须在链上积极发送心跳以提供更新。这个队列的数据和配置存储在一个由 Switchboard 程序拥有的[链上账户](https://github.com/switchboard-xyz/solana-sdk/blob/9dc3df8a5abe261e23d46d14f9e80a7032bb346c/javascript/solana.js/src/generated/oracle-program/accounts/OracleQueueAccountData.ts#L8)中。
- **预言机共识（Oracle Consensus）** - 决定预言机如何就链上接受的结果达成一致。Switchboard 预言机使用中位预言机（median oracle response）响应作为接受的结果。数据源管理者可以控制要求多少预言机以及需要多少预言机响应来影响其安全性。

Switchboard 预言机有动机更新数据源，因为他们会因准确更新而获得奖励。每个数据源都有一个 `LeaseContract` 账户。租约合约（lease contract）是一个预先资助的托管账户（escrow account），用于奖励预言机履行更新请求。只有预先定义的 `leaseAuthority` 可以从合约中提取资金，但任何人都可以为其贡献资金。当对数据源请求新一轮的更新时，请求更新的用户将从托管账户中获得奖励。这旨在激励用户和转动曲柄者（crank turners，任何运行软件以系统性地向预言机发送更新请求的人）根据数据源的配置保持数据源更新。一旦更新请求成功被队列中的预言机在链上提交，预言机也会从托管账户中获得奖励。这些支付确保了活跃的参与者，请求更新的用户和预言机可以在其中获得收益。

此外，预言机在能够服务更新请求并在链上提交响应之前必须抵押代币。如果一个预言机在链上提交了超出队列配置参数的结果，他们的抵押将被减少（如果队列启用了 `slashingEnabled`）。这有助于确保预言机以诚信的方式做出回应，并提供准确的信息。

现在您已经了解了术语和经济学，让我们看看如何将数据发布到链上：

1. 预言机队列设置 - 当从队列请求更新时，下一批 `N` 个预言机被分配到更新请求，并被移到队列的末尾。Switchboard 网络中的每个预言机队列都是独立的，并维护其自己的配置。配置影响其安全级别。这种设计选择使用户能够根据其特定的用例定制预言机队列的行为。预言机队列以账户的形式存储在链上，并包含有关队列的元数据。通过在 Switchboard Solana 程序上调用 [oracleQueueInit 指令](https://github.com/switchboard-xyz/solana-sdk/blob/9dc3df8a5abe261e23d46d14f9e80a7032bb346c/javascript/solana.js/src/generated/oracle-program/instructions/oracleQueueInit.ts#L13) 来创建队列。

    一些相关的预言机队列配置：
    1. `oracle_timeout` - 预言机未能发送心跳时而被移除的时间间隔。
    2. `reward` - 提供给预言机和在该队列中打开轮次的奖励。
    3. `min_stake` - 预言机必须提供的最低抵押金额，以保持在队列中。
    4. `size` - 队列中当前的预言机数量。
    5. `max_size` - 队列可以支持的最大预言机数量。

2. 聚合器/数据源设置 - 创建聚合器/数据源账户。每个数据源属于单个预言机队列。数据源的配置决定了如何通过网络发起和路由更新请求。

3. 作业账户设置 - 除了数据源外，必须为每个数据源设置一个作业账户。这定义了预言机如何履行数据源的更新请求。这包括定义预言机应从数据源获取数据的位置。

4. 请求分配 - 一旦使用数据源账户请求了更新，预言机队列将请求分配给队列中的不同预言机/节点来履行。预言机将从每个数据源（feed，链上）的作业账户中定义的数据源（source，链下）中获取数据。每个作业账户都有一个相关联的权重。预言机将计算所有任务的结果的加权中位数。

5. 收到 `minOracleResults` 个响应后，链上程序使用预言机响应的中位数计算结果。在队列配置的参数范围内响应的预言机会获得奖励，而响应超出此阈值的预言机会被削减（如果队列启用了 `slashingEnabled`）。

6. 更新后的结果存储在数据源账户中，以便在链上读取/使用。

### 如何使用 Switchboard 预言机

要使用 Switchboard 预言机并将链下数据纳入 Solana 程序中，首先必须找到提供所需数据的数据源（链上）。Switchboard 的数据源是公开的，并且有许多[可供选择的已经可用的数据源](https://app.switchboard.xyz/solana/devnet/explore)。在寻找数据源时，您必须决定您希望数据源的准确性/可靠性如何，您希望从何处获取数据，以及数据源的更新频率。当使用公开可用的数据源时，您无法控制这些事情，因此请谨慎选择！

例如，有一个由 Switchboard 赞助的 [BTC_USD 数据源](https://app.switchboard.xyz/solana/devnet/feed/8SXvChNYFhRq4EZuZvnhjrB3jJRQCv4k3P4W6hesH3Ee)。该数据源在 Solana 的 devnet/mainnet 上都可用，其公钥为 `8SXvChNYFhRq4EZuZvnhjrB3jJRQCv4k3P4W6hesH3Ee`。它提供了比特币当前的美元价格。

Switchboard 数据源账户的实际链上数据看起来有点像这样：

```rust
// from the switchboard solana program
// https://github.com/switchboard-xyz/sbv2-solana/blob/0b5e0911a1851f9ca37042e6ff88db4cd840067b/rust/switchboard-solana/src/oracle_program/accounts/aggregator.rs#L60

pub struct AggregatorAccountData {
    /// Name of the aggregator to store on-chain.
    pub name: [u8; 32],
    ...
		...
    /// Pubkey of the queue the aggregator belongs to.
    pub queue_pubkey: Pubkey,
    ...
    /// Minimum number of oracle responses required before a round is validated.
    pub min_oracle_results: u32,
    /// Minimum number of job results before an oracle accepts a result.
    pub min_job_results: u32,
    /// Minimum number of seconds required between aggregator rounds.
    pub min_update_delay_seconds: u32,
    ...
    /// Change percentage required between a previous round and the current round. If variance percentage is not met, reject new oracle responses.
    pub variance_threshold: SwitchboardDecimal,
    ...
		/// Latest confirmed update request result that has been accepted as valid. This is where you will find the data you are requesting in latest_confirmed_round.result
	  pub latest_confirmed_round: AggregatorRound,
		...
    /// The previous confirmed round result.
    pub previous_confirmed_round_result: SwitchboardDecimal,
    /// The slot when the previous confirmed round was opened.
    pub previous_confirmed_round_slot: u64,
		...
}
```

您可以在[此处查看 Switchboard 程序中完整的数据结构代码](https://github.com/switchboard-xyz/sbv2-solana/blob/0b5e0911a1851f9ca37042e6ff88db4cd840067b/rust/switchboard-solana/src/oracle_program/accounts/aggregator.rs#L60)。

`AggregatorAccountData` 类型的一些相关字段和配置如下：

- `min_oracle_results` - 在验证轮次之前需要的最少预言机响应数。
- `min_job_results` - 在预言机接受结果之前需要的最少作业结果数。
- `variance_threshold` - 前一轮和当前轮之间所需的变化百分比。如果未达到变化百分比，将拒绝新的预言机响应。
- `latest_confirmed_round` - 最新确认的更新请求结果，已被接受为有效。这是您将在 `latest_confirmed_round.result` 中找到数据的地方。
- `min_update_delay_seconds` - 聚合器轮次之间所需的最少秒数。

以上列出的前三个配置与数据源的准确性和可靠性直接相关。

`min_job_results` 字段表示预言机在可以在链上提交响应之前必须从数据源接收到的成功响应的最小数量。换句话说，如果 `min_job_results` 是三，那么每个预言机必须从三个作业源拉取数据。这个数字越高，数据源上的数据就越可靠和准确。这也限制了单个数据源对结果的影响。

`min_oracle_results` 字段是一轮成功所需的最少预言机响应数量。请记住，队列中的每个预言机都从被定义为作业的每个数据源（链下）中拉取数据。然后，预言机计算所有数据源的响应的加权中位数，并将该中位数提交到链上。然后程序等待加权中位数的 `min_oracle_results`，并对其进行中位数计算，该计算结果是存储在数据源账户中的最终结果。

`min_update_delay_seconds` 字段与数据源的更新频率直接相关。在 Switchboard 程序接受结果之前，必须在更新的轮次之间经过 `min_update_delay_seconds` 秒。

这时候看一下 Switchboard 浏览器中的数据源的作业标签可能会有所帮助。例如，您可以查看[浏览器中的 BTC_USD 数据源](https://app.switchboard.xyz/solana/devnet/feed/8SXvChNYFhRq4EZuZvnhjrB3jJRQCv4k3P4W6hesH3Ee)。列出的每个作业定义了预言机将从中获取数据的来源以及每个来源的权重。您可以查看提供此特定数据源数据的实际 API 端点。在确定要在您的程序中使用哪个数据源时，诸如此类的事项非常重要。

以下是与 BTC_USD 数据源相关的两个作业的屏幕截图。它显示了两个数据源：[MEXC](https://www.mexc.com/) 和 [Coinbase](https://www.coinbase.com/)。

![Oracle Jobs](../../assets/oracle-jobs.png)

一旦您选择了要使用的数据源，您就可以开始读取该数据源中的数据。您可以通过简单地对帐户中存储的状态进行反序列化和读取来完成此操作。最简单的方法是在您的程序中利用我们上面在`switchboard_v2` crate中定义的 `AggregatorAccountData` 结构体。

```rust
// import anchor and switchboard crates
use {
    anchor_lang::prelude::*,
    switchboard_v2::AggregatorAccountData,
};

...

#[derive(Accounts)]
pub struct ConsumeDataAccounts<'info> {
	// pass in data feed account and deserialize to AggregatorAccountData
	pub feed_aggregator: AccountLoader<'info, AggregatorAccountData>,
	...
}
```

请注意，在这里我们使用 `AccountLoader` 类型来反序列化聚合器账户，而不是普通的 `Account` 类型。由于 `AggregatorAccountData` 的大小，该账户使用了所谓的零拷贝。这与 `AccountLoader` 结合使用，防止了账户被加载到内存中，并且使我们的程序直接访问数据。当使用 `AccountLoader` 时，我们可以通过以下三种方式之一访问存储在账户中的数据：

- 在初始化账户后使用 `load_init`（这将忽略仅在用户的指令代码之后添加的缺失账户识别符）
- 当账户不可变时使用 `load`
- 当账户可变时使用 `load_mut`

如果您想了解更多信息，请查看[高级程序架构课程](./program-architecture.md)，我们在其中涉及了 `Zero-Copy` 和 `AccountLoader`。

通过将聚合器账户传递到您的程序中，您可以使用它来获取最新的 Oracle 结果。具体来说，您可以使用该类型的 `get_result()` 方法：

```rust
// inside an Anchor program
...

let feed = &ctx.accounts.feed_aggregator.load()?;
// get result
let val: f64 = feed.get_result()?.try_into()?;
```

`AggregatorAccountData` 结构体上定义的 `get_result()` 方法比通过 `latest_confirmed_round.result` 获取数据更安全，因为 Switchboard 已经实现了一些巧妙的安全检查。

```rust
// from switchboard program
// https://github.com/switchboard-xyz/sbv2-solana/blob/0b5e0911a1851f9ca37042e6ff88db4cd840067b/rust/switchboard-solana/src/oracle_program/accounts/aggregator.rs#L195

pub fn get_result(&self) -> anchor_lang::Result<SwitchboardDecimal> {
    if self.resolution_mode == AggregatorResolutionMode::ModeSlidingResolution {
        return Ok(self.latest_confirmed_round.result);
    }
    let min_oracle_results = self.min_oracle_results;
    let latest_confirmed_round_num_success = self.latest_confirmed_round.num_success;
    if min_oracle_results > latest_confirmed_round_num_success {
        return Err(SwitchboardError::InvalidAggregatorRound.into());
    }
    Ok(self.latest_confirmed_round.result)
}
```

你也可以在 TypeScript 中客户端查看存储在 `AggregatorAccountData` 账户中的当前值。

```tsx
import { AggregatorAccount, SwitchboardProgram} from '@switchboard-xyz/solana.js'

...
...
// create keypair for test user
let user = new anchor.web3.Keypair()

// fetch switchboard devnet program object
switchboardProgram = await SwitchboardProgram.load(
  "devnet",
  new anchor.web3.Connection("https://api.devnet.solana.com"),
  user
)

// pass switchboard program object and feed pubkey into AggregatorAccount constructor
aggregatorAccount = new AggregatorAccount(switchboardProgram, solUsedSwitchboardFeed)

// fetch latest SOL price
const solPrice: Big | null = await aggregatorAccount.fetchLatestValue()
if (solPrice === null) {
  throw new Error('Aggregator holds no value')
}
```

记住，Switchboard 数据源只是由第三方（预言机）更新的账户。鉴于此，你可以对这些账户执行与你程序的外部账户通常能做的任何操作。

### 最佳实践和常见陷阱

在将 Switchboard 数据源纳入你的程序时，有两组关注点需要考虑：选择一个数据源，以及实际使用该数据源的数据。

在决定将某个数据源纳入程序之前，始终要审查该数据源的配置。像 **Min Update Delay**、**Min job Results** 和 **Min Oracle Results** 这样的配置可以直接影响最终存储在聚合器账户上链的数据。例如，查看 [BTC_USD 数据源](https://app.switchboard.xyz/solana/devnet/feed/8SXvChNYFhRq4EZuZvnhjrB3jJRQCv4k3P4W6hesH3Ee) 的配置部分，你可以看到其相关配置。

![Oracle Configs](../../assets/oracle-configs.png)

BTC_USD 数据源的 Min Update Delay = 6 秒。这意味着在该数据源上，BTC 的价格至少每 6 秒更新一次。对于大多数用例来说，这可能是可以接受的，但如果你想将此数据源用于对延迟敏感的应用，那么它可能不是一个好选择。

审计数据源的来源也很值得注意，在预言机资源浏览器的 Jobs 部分可以找到。由于在链上持久化的值是预言机从每个来源提取的加权中位数结果，来源直接影响存储在数据源中的内容。检查是否有可疑的链接，可能需要自行运行 API 一段时间以增强信心。

一旦找到符合需求的数据源，仍然需要确保正确使用该数据源。例如，仍然应在传递给指令的账户上实施必要的安全检查。任何账户都可以传递给程序的指令，因此应验证它是否是预期的账户。

在 Anchor 中，如果你将账户反序列化为 `switchboard_v2` crate 中的 `AggregatorAccountData` 类型，Anchor 会检查该账户是否由 Switchboard 程序拥有。如果你的程序期望只有特定的数据源会传递到指令中，那么你也可以验证传递给指令的账户的公钥是否与预期的一致。一种方法是在程序中某处硬编码地址，并使用账户约束来验证传入的地址是否与预期的地址匹配。

```rust
use {
  anchor_lang::prelude::*,
  solana_program::{pubkey, pubkey::Pubkey},
	switchboard_v2::{AggregatorAccountData},
};

pub static BTC_USDC_FEED: Pubkey = pubkey!("8SXvChNYFhRq4EZuZvnhjrB3jJRQCv4k3P4W6hesH3Ee");

...
...

#[derive(Accounts)]
pub struct TestInstruction<'info> {
	// Switchboard SOL feed aggregator
	#[account(
	    address = BTC_USDC_FEED
	)]
	pub feed_aggregator: AccountLoader<'info, AggregatorAccountData>,
}
```

除了确保数据源账户是你期望的账户外，你还可以在程序的指令逻辑中对数据源中存储的数据进行一些检查。两个常见的检查项是数据的陈旧性（staleness）和置信区间（confidence interval）。

每个数据源在被预言机触发时更新其中存储的当前值。这意味着更新取决于分配给它的预言机队列中的预言机。根据你打算使用数据源的用途，验证存储在账户中的值是否最近更新可能会有利。例如，需要确定贷款抵押品是否低于某个水平的借贷协议可能需要数据的寿命不超过几秒钟。你可以让你的代码检查聚合器账户中最近更新的时间戳。以下代码片段检查数据源上最近更新的时间戳是否不超过 30 秒前。

```rust
use {
    anchor_lang::prelude::*,
    anchor_lang::solana_program::clock,
    switchboard_v2::{AggregatorAccountData, SwitchboardDecimal},
};

...
...

let feed = &ctx.accounts.feed_aggregator.load()?;
if (clock::Clock::get().unwrap().unix_timestamp - feed.latest_confirmed_round.round_open_timestamp) <= 30{
      valid_transfer = true;
  }
```

`AggregatorAccountData` 结构体上的 `latest_confirmed_round` 字段的类型是 `AggregatorRound`，其定义如下：

```rust
// https://github.com/switchboard-xyz/sbv2-solana/blob/0b5e0911a1851f9ca37042e6ff88db4cd840067b/rust/switchboard-solana/src/oracle_program/accounts/aggregator.rs#L17

pub struct AggregatorRound {
    /// Maintains the number of successful responses received from nodes.
    /// Nodes can submit one successful response per round.
    pub num_success: u32,
    /// Number of error responses.
    pub num_error: u32,
    /// Whether an update request round has ended.
    pub is_closed: bool,
    /// Maintains the `solana_program::clock::Slot` that the round was opened at.
    pub round_open_slot: u64,
    /// Maintains the `solana_program::clock::UnixTimestamp;` the round was opened at.
    pub round_open_timestamp: i64,
    /// Maintains the current median of all successful round responses.
    pub result: SwitchboardDecimal,
    /// Standard deviation of the accepted results in the round.
    pub std_deviation: SwitchboardDecimal,
    /// Maintains the minimum node response this round.
    pub min_response: SwitchboardDecimal,
    /// Maintains the maximum node response this round.
    pub max_response: SwitchboardDecimal,
    /// Pubkeys of the oracles fulfilling this round.
    pub oracle_pubkeys_data: [Pubkey; 16],
    /// Represents all successful node responses this round. `NaN` if empty.
    pub medians_data: [SwitchboardDecimal; 16],
    /// Current rewards/slashes oracles have received this round.
    pub current_payout: [i64; 16],
    /// Keep track of which responses are fulfilled here.
    pub medians_fulfilled: [bool; 16],
    /// Keeps track of which errors are fulfilled here.
    pub errors_fulfilled: [bool; 16],
}
```

在 Aggregator 账户中还有一些其他相关字段可能会引起你的兴趣，例如 `num_success`、`medians_data`、`std_deviation` 等。`num_success` 是在此轮更新中从预言机接收到的成功响应的数量。`medians_data` 是在此轮更新中从预言机接收到的所有成功响应的数组。这是用于推导中位数和最终结果的数据集。`std_deviation` 是此轮接受结果的标准偏差。你可能想要检查标准偏差较低，这意味着所有的预言机响应都很相似。Switchboard 程序负责在每次从预言机接收到更新时更新该结构体上的相关字段。

`AggregatorAccountData` 还有一个 `check_confidence_interval()` 方法，你可以将其用作对存储在数据源中的数据的另一种验证。该方法允许你传入一个 `max_confidence_interval`。如果从预言机接收到的结果的标准偏差大于给定的 `max_confidence_interval`，它将返回一个错误。

```rust
// https://github.com/switchboard-xyz/sbv2-solana/blob/0b5e0911a1851f9ca37042e6ff88db4cd840067b/rust/switchboard-solana/src/oracle_program/accounts/aggregator.rs#L228

pub fn check_confidence_interval(
    &self,
    max_confidence_interval: SwitchboardDecimal,
) -> anchor_lang::Result<()> {
    if self.latest_confirmed_round.std_deviation > max_confidence_interval {
        return Err(SwitchboardError::ConfidenceIntervalExceeded.into());
    }
    Ok(())
}
```

你可以像下面这样将这些内容整合到你的程序中：

```rust
use {
    crate::{errors::*},
    anchor_lang::prelude::*,
    std::convert::TryInto,
    switchboard_v2::{AggregatorAccountData, SwitchboardDecimal},
};

...
...

let feed = &ctx.accounts.feed_aggregator.load()?;

// check feed does not exceed max_confidence_interval
feed.check_confidence_interval(SwitchboardDecimal::from_f64(max_confidence_interval))
    .map_err(|_| error!(ErrorCode::ConfidenceIntervalExceeded))?;
```

最后，重要的是要在你的程序中为最坏情况进行规划。计划数据源变得陈旧和计划数据源账户关闭。

## 结论

如果你想要能够根据真实世界的数据执行操作的功能性程序，你必须使用预言机。幸运的是，有一些值得信赖的预言机网络，比如 Switchboard，它们使使用预言机比起其他方式来说更容易。然而，确保对你使用的预言机进行尽职调查是非常重要的。最终，你对程序的行为负有责任！

# 实验

让我们来练习使用预言机！我们将构建一个名为 “Michael Burry Escrow” 的程序，该程序将在锁定 SOL 于托管账户中，直到 SOL 的价值超过某个特定的美元价值。这个名字来源于投资者 [Michael Burry](https://en.wikipedia.org/wiki/Michael_Burry)，他因成功预测了 2008 年的房地产市场崩溃而闻名。

我们将使用 Switchboard 的 devnet [SOL_USD](https://app.switchboard.xyz/solana/devnet/feed/GvDMxPzN1sCj7L26YDK2HnMRXEQmQ2aemov8YBtPS7vR) 预言机。该程序将有两个主要指令：

- 存款（Deposit） - 锁定 SOL 并设置一个 USD 价格以解锁。
- 取款（Withdraw） - 检查 USD 价格，如果价格达到，则取出 SOL。

## 1. 程序设置

首先，让我们使用以下命令创建程序：

```bash
anchor init burry-escrow
```

接下来，将 `lib.rs` 和 `Anchor.toml` 中的程序 ID 替换为运行 `anchor keys list` 时显示的程序 ID。

然后，将以下内容添加到 Anchor.toml 文件的末尾。这将告诉 Anchor 如何配置我们的本地测试环境。这样我们就可以在本地测试我们的程序，而无需部署到 devnet 并发送交易。

```toml
// bottom of Anchor.toml
[test.validator]
url="https://api.devnet.solana.com"

[test]
startup_wait = 10000

[[test.validator.clone]] # sbv2 devnet programID
address = "SW1TCH7qEPTdLsDHRgPuMQjbQxKdH2aBStViMFnt64f"

[[test.validator.clone]] # sbv2 devnet IDL
address = "Fi8vncGpNKbq62gPo56G4toCehWNy77GgqGkTaAF5Lkk"

[[test.validator.clone]] # sbv2 SOL/USD Feed
address="GvDMxPzN1sCj7L26YDK2HnMRXEQmQ2aemov8YBtPS7vR"
```

此外，我们希望在我们的 `Cargo.toml` 文件中导入 `switchboard-v2` crate。确保你的依赖项如下所示：

```toml
[dependencies]
anchor-lang = "0.28.0"
switchboard-v2 = "0.4.0"
```

在开始编写逻辑之前，让我们先来了解一下程序的结构。对于小型程序来说，将所有的智能合约代码都添加到一个 `lib.rs` 文件中并且结束工作是非常容易的。然而，为了保持更有条理性，将其分散在不同的文件中会更有帮助。我们的程序将在 `programs/src` 目录下包含以下文件：

`/instructions/deposit.rs`

`/instructions/withdraw.rs`

`/instructions/mod.rs`

`errors.rs`

`state.rs`

`lib.rs`

`lib.rs` 文件仍然将作为程序的入口点，但是每个指令的逻辑将包含在它们各自的单独文件中。请按照上述描述创建程序结构，我们将开始编写逻辑。

## 2. `lib.rs`

在编写任何逻辑之前，我们将设置所有的样板信息。从 `lib.rs` 开始。我们的实际逻辑将存放在 `/instructions` 目录中。

`lib.rs` 文件将作为我们程序的入口点。它将定义所有交易必须经过的 API 端点。

```rust
use anchor_lang::prelude::*;
use instructions::deposit::*;
use instructions::withdraw::*;
use state::*;

pub mod instructions;
pub mod state;
pub mod errors;

declare_id!("YOUR_PROGRAM_KEY_HERE");

#[program]
mod burry_oracle_program {

    use super::*;

    pub fn deposit(ctx: Context<Deposit>, escrow_amt: u64, unlock_price: u64) -> Result<()> {
        deposit_handler(ctx, escrow_amt, unlock_price)
    }

    pub fn withdraw(ctx: Context<Withdraw>) -> Result<()> {
        withdraw_handler(ctx)
    }
}
```

## 3. `state.rs`

接下来，让我们定义这个程序的数据账户：`EscrowState`。我们的数据账户将存储两个信息：

- `unlock_price` - 当 SOL 价格达到该 USD 价格时可以提取；你可以将其硬编码为你想要的任何值（例如 $21.53）
- `escrow_amount` - 跟踪存储在托管账户中的 lamports 数量

我们还将定义我们的 PDA 种子为 `"MICHAEL BURRY"`，以及我们硬编码的 SOL_USD 预言机公钥为 `SOL_USDC_FEED`。

```rust
// in state.rs
use anchor_lang::prelude::*;

pub const ESCROW_SEED: &[u8] = b"MICHAEL BURRY";
pub const SOL_USDC_FEED: &str = "GvDMxPzN1sCj7L26YDK2HnMRXEQmQ2aemov8YBtPS7vR";

#[account]
pub struct EscrowState {
    pub unlock_price: f64,
    pub escrow_amount: u64,
}
```

## 4. 错误

让我们定义在整个程序中将使用的自定义错误。在 `errors.rs` 文件中，粘贴以下内容：

```rust
use anchor_lang::prelude::*;

#[error_code]
#[derive(Eq, PartialEq)]
pub enum EscrowErrorCode {
    #[msg("Not a valid Switchboard account")]
    InvalidSwitchboardAccount,
    #[msg("Switchboard feed has not been updated in 5 minutes")]
    StaleFeed,
    #[msg("Switchboard feed exceeded provided confidence interval")]
    ConfidenceIntervalExceeded,
    #[msg("Current SOL price is not above Escrow unlock price.")]
    SolPriceAboveUnlockPrice,
}
```

## 5. `mod.rs`

让我们设置我们的 `instructions/mod.rs` 文件。

```rust
// inside mod.rs
pub mod deposit;
pub mod withdraw;
```

## 6. 存款

既然我们已经完成了所有的样板，让我们继续进行我们的存款指令。这将存放在 `/src/instructions/deposit.rs` 文件中。当用户存款时，应该创建一个 PDA，其中包含字符串 “MICHAEL BURRY” 和用户的公钥作为种子。这意味着用户一次只能打开一个托管账户。该指令应在此 PDA 上初始化一个账户，并将用户想要锁定的 SOL 金额发送到该账户。用户将需要成为签名者。

首先让我们构建存款上下文结构。为此，我们需要考虑为这个指令需要哪些账户。我们从以下内容开始：

```rust
//inside deposit.rs
use crate::state::*;
use anchor_lang::prelude::*;
use anchor_lang::solana_program::{
    system_instruction::transfer,
    program::invoke
};

#[derive(Accounts)]
pub struct Deposit<'info> {
    // user account
    #[account(mut)]
    pub user: Signer<'info>,
    #[account(
      init,
      seeds = [ESCROW_SEED, user.key().as_ref()],
      bump,
      payer = user,
      space = std::mem::size_of::<EscrowState>() + 8
    )]
    pub escrow_account: Account<'info, EscrowState>,
		// system program
    pub system_program: Program<'info, System>,
}
```

请注意我们添加到账户的约束：
- 因为我们将从用户账户转移 SOL 到 `escrow_state` 账户，它们两者都需要是可变的。
- 我们知道 `escrow_account` 应该是由字符串 “MICHAEL BURRY” 和用户的公钥派生而来的 PDA。我们可以使用 Anchor 账户约束来保证传入的地址确实满足这个要求。
- 我们也知道我们必须在这个 PDA 上初始化一个账户来存储程序的一些状态。我们在这里使用 `init` 约束。

让我们继续进行实际的逻辑。我们需要做的就是初始化 `escrow_state` 账户的状态并转移 SOL。我们期望用户传入他们想要锁定的 SOL 金额和解锁的价格。我们将这些值存储在 `escrow_state` 账户中。

之后，方法应该执行转移。这个程序将锁定原生的 SOL。因此，我们不需要使用代币账户或 Solana 代币程序。我们将使用 `system_program` 来转移用户想要锁定的 lamports，并调用转移指令。

```rust
pub fn deposit_handler(ctx: Context<Deposit>, escrow_amt: u64, unlock_price: u64) -> Result<()> {
		msg!("Depositing funds in escrow...");

    let escrow_state = &mut ctx.accounts.escrow_account;
    escrow_state.unlock_price = unlock_price;
    escrow_state.escrow_amount = escrow_amount;

    let transfer_ix = transfer(
      &ctx.accounts.user.key(),
      &escrow_state.key(),
      escrow_amount
    );

    invoke(
        &transfer_ix,
        &[
            ctx.accounts.user.to_account_info(),
            ctx.accounts.escrow_account.to_account_info(),
            ctx.accounts.system_program.to_account_info()
        ]
    )?;

    msg!("Transfer complete. Escrow will unlock SOL at {}", &ctx.accounts.escrow_account.unlock_price);
}
```

这就是存款指令的要点！`deposit.rs` 文件的最终结果应如下所示：

```rust
use crate::state::*;
use anchor_lang::prelude::*;
use anchor_lang::solana_program::{
    system_instruction::transfer,
    program::invoke
};

pub fn deposit_handler(ctx: Context<Deposit>, escrow_amount: u64, unlock_price: f64) -> Result<()> {
    msg!("Depositing funds in escrow...");

    let escrow_state = &mut ctx.accounts.escrow_account;
    escrow_state.unlock_price = unlock_price;
    escrow_state.escrow_amount = escrow_amount;

    let transfer_ix = transfer(
        &ctx.accounts.user.key(),
        &escrow_state.key(),
        escrow_amount
    );

    invoke(
        &transfer_ix,
        &[
            ctx.accounts.user.to_account_info(),
            ctx.accounts.escrow_account.to_account_info(),
            ctx.accounts.system_program.to_account_info()
        ]
    )?;

    msg!("Transfer complete. Escrow will unlock SOL at {}", &ctx.accounts.escrow_account.unlock_price);

    Ok(())
}

#[derive(Accounts)]
pub struct Deposit<'info> {
    // user account
    #[account(mut)]
    pub user: Signer<'info>,
    // account to store SOL in escrow
    #[account(
        init,
        seeds = [ESCROW_SEED, user.key().as_ref()],
        bump,
        payer = user,
        space = std::mem::size_of::<EscrowState>() + 8
    )]
    pub escrow_account: Account<'info, EscrowState>,

    pub system_program: Program<'info, System>,
}
```

**取款**

取款指令将需要与存款指令相同的三个账户，以及 SOL_USDC Switchboard 供应商账户。此代码将放在 `withdraw.rs` 文件中。

```rust
use crate::state::*;
use crate::errors::*;
use std::str::FromStr;
use anchor_lang::prelude::*;
use switchboard_v2::AggregatorAccountData;
use anchor_lang::solana_program::clock::Clock;

#[derive(Accounts)]
pub struct Withdraw<'info> {
    // user account
    #[account(mut)]
    pub user: Signer<'info>,
    // escrow account
    #[account(
        mut,
        seeds = [ESCROW_SEED, user.key().as_ref()],
        bump,
        close = user
    )]
    pub escrow_account: Account<'info, EscrowState>,
    // Switchboard SOL feed aggregator
    #[account(
        address = Pubkey::from_str(SOL_USDC_FEED).unwrap()
    )]
    pub feed_aggregator: AccountLoader<'info, AggregatorAccountData>,
    pub system_program: Program<'info, System>,
}
```

请注意，我们正在使用关闭约束，因为一旦交易完成，我们希望关闭 `escrow_account`。账户中用作租金的 SOL 将被转移到 `user` 账户。

我们还使用地址约束来验证传入的数据源账户实际上是 `usdc_sol` 数据源账户，而不是其他数据源账户（我们已经在代码中硬编码了 SOL_USDC_FEED 地址）。此外，我们从 Switchboard rust crate 反序列化的 AggregatorAccountData 结构体。它验证了给定的账户是由 switchboard 程序拥有的，并允许我们轻松查看其值。您会注意到它被包裹在 `AccountLoader` 中。这是因为数据源账户实际上是一个相当大的账户，需要零拷贝。

现在让我们实现取款指令的逻辑。首先，我们检查数据源账户是否过时。然后，我们获取存储在 `feed_aggregator` 账户中的 SOL 的当前价格。最后，我们希望检查当前价格是否高于 escrow `unlock_price`。如果是，则将 SOL 从 escrow 账户转回用户账户并关闭账户。如果不是，则指令应该完成并返回错误。

```rust
pub fn withdraw_handler(ctx: Context<Withdraw>, params: WithdrawParams) -> Result<()> {
    let feed = &ctx.accounts.feed_aggregator.load()?;
    let escrow_state = &ctx.accounts.escrow_account;

    // get result
    let val: f64 = feed.get_result()?.try_into()?;

    // check whether the feed has been updated in the last 300 seconds
    feed.check_staleness(Clock::get().unwrap().unix_timestamp, 300)
    .map_err(|_| error!(EscrowErrorCode::StaleFeed))?;

    msg!("Current feed result is {}!", val);
    msg!("Unlock price is {}", escrow_state.unlock_price);

    if val < escrow_state.unlock_price as f64 {
        return Err(EscrowErrorCode::SolPriceAboveUnlockPrice.into())
    }

	....
}
```

为了完成逻辑，我们将执行转账，这次我们将以不同的方式转移资金。因为我们正在从一个还保存有数据的账户转移，所以我们不能像之前那样使用 `system_program::transfer` 方法。如果我们尝试这样做，指令将无法执行，并显示以下错误。

```zsh
'Transfer: `from` must not carry data'
```

为了解决这个问题，我们将在每个账户上使用 `try_borrow_mut_lamports()` 方法，并添加/减去每个账户中存储的 lamports 数量。

```rust
// 'Transfer: `from` must not carry data'
  **escrow_state.to_account_info().try_borrow_mut_lamports()? = escrow_state
      .to_account_info()
      .lamports()
      .checked_sub(escrow_state.escrow_amount)
      .ok_or(ProgramError::InvalidArgument)?;

  **ctx.accounts.user.to_account_info().try_borrow_mut_lamports()? = ctx.accounts.user
      .to_account_info()
      .lamports()
      .checked_add(escrow_state.escrow_amount)
      .ok_or(ProgramError::InvalidArgument)?;
```

`withdraw.rs` 文件中的最终取款方法应该是这样的：

```rust
use crate::state::*;
use crate::errors::*;
use std::str::FromStr;
use anchor_lang::prelude::*;
use switchboard_v2::AggregatorAccountData;
use anchor_lang::solana_program::clock::Clock;

pub fn withdraw_handler(ctx: Context<Withdraw>) -> Result<()> {
    let feed = &ctx.accounts.feed_aggregator.load()?;
    let escrow_state = &ctx.accounts.escrow_account;

    // get result
    let val: f64 = feed.get_result()?.try_into()?;

    // check whether the feed has been updated in the last 300 seconds
    feed.check_staleness(Clock::get().unwrap().unix_timestamp, 300)
    .map_err(|_| error!(EscrowErrorCode::StaleFeed))?;

    msg!("Current feed result is {}!", val);
    msg!("Unlock price is {}", escrow_state.unlock_price);

    if val < escrow_state.unlock_price as f64 {
        return Err(EscrowErrorCode::SolPriceAboveUnlockPrice.into())
    }

    // 'Transfer: `from` must not carry data'
    **escrow_state.to_account_info().try_borrow_mut_lamports()? = escrow_state
        .to_account_info()
        .lamports()
        .checked_sub(escrow_state.escrow_amount)
        .ok_or(ProgramError::InvalidArgument)?;

    **ctx.accounts.user.to_account_info().try_borrow_mut_lamports()? = ctx.accounts.user
        .to_account_info()
        .lamports()
        .checked_add(escrow_state.escrow_amount)
        .ok_or(ProgramError::InvalidArgument)?;

    Ok(())
}

#[derive(Accounts)]
pub struct Withdraw<'info> {
    // user account
    #[account(mut)]
    pub user: Signer<'info>,
    // escrow account
    #[account(
        mut,
        seeds = [ESCROW_SEED, user.key().as_ref()],
        bump,
        close = user
    )]
    pub escrow_account: Account<'info, EscrowState>,
    // Switchboard SOL feed aggregator
    #[account(
        address = Pubkey::from_str(SOL_USDC_FEED).unwrap()
    )]
    pub feed_aggregator: AccountLoader<'info, AggregatorAccountData>,
    pub system_program: Program<'info, System>,
}
```

至此，程序就完成了！此时，您应该能够无错误地运行 `anchor build`。

注意：如果您看到类似下面所示的错误，您可以安全地忽略它。

```bash
Compiling switchboard-v2 v0.4.0
Error: Function _ZN86_$LT$switchboard_v2..aggregator..AggregatorAccountData$u20$as$u20$core..fmt..Debug$GT$3fmt17hea9f7644392c2647E Stack offset of 4128 exceeded max offset of 4096 by 32 bytes, please minimize large stack variables
```

## 7. 测试

让我们编写一些测试。我们应该有四个测试：

- 创建一个解锁价格***低于***当前 SOL 价格的 Escrow，以便我们可以测试取款
- 从上述 Escrow 中取款并关闭
- 创建一个解锁价格***高于***当前 SOL 价格的 Escrow，以便我们可以测试取款
- 从上述 Escrow 中取款并失败

请注意，每个用户只能有一个 Escrow，因此上述顺序很重要。

我们将在一个代码片段中提供所有测试代码。在运行 `anchor test` 之前，请仔细查看以确保您理解它。

```typescript
// tests/burry-escrow.ts

import * as anchor from "@coral-xyz/anchor";
import { Program } from "@coral-xyz/anchor";
import { BurryEscrow } from "../target/types/burry_escrow";
import { Big } from "@switchboard-xyz/common";
import { AggregatorAccount, AnchorWallet, SwitchboardProgram } from "@switchboard-xyz/solana.js"
import { assert } from "chai";

export const solUsedSwitchboardFeed = new anchor.web3.PublicKey("GvDMxPzN1sCj7L26YDK2HnMRXEQmQ2aemov8YBtPS7vR")

describe("burry-escrow", () => {
  // Configure the client to use the local cluster.
  anchor.setProvider(anchor.AnchorProvider.env());
  const provider = anchor.AnchorProvider.env()
  const program = anchor.workspace.BurryEscrow as Program<BurryEscrow>;
  const payer = (provider.wallet as AnchorWallet).payer

  it("Create Burry Escrow Below Price", async () => {
    // fetch switchboard devnet program object
    const switchboardProgram = await SwitchboardProgram.load(
      "devnet",
      new anchor.web3.Connection("https://api.devnet.solana.com"),
      payer
    )
    const aggregatorAccount = new AggregatorAccount(switchboardProgram, solUsedSwitchboardFeed)

    // derive escrow state account
    const [escrowState] = await anchor.web3.PublicKey.findProgramAddressSync(
      [Buffer.from("MICHAEL BURRY"), payer.publicKey.toBuffer()],
      program.programId
    )

    // fetch latest SOL price
    const solPrice: Big | null = await aggregatorAccount.fetchLatestValue()
    if (solPrice === null) {
      throw new Error('Aggregator holds no value')
    }
    const failUnlockPrice = solPrice.minus(10).toNumber()
    const amountToLockUp = new anchor.BN(100)

    // Send transaction
    try {
      const tx = await program.methods.deposit(
        amountToLockUp, 
        failUnlockPrice
      )
      .accounts({
        user: payer.publicKey,
        escrowAccount: escrowState,
        systemProgram: anchor.web3.SystemProgram.programId
      })
      .signers([payer])
      .rpc()

      await provider.connection.confirmTransaction(tx, "confirmed")

      // Fetch the created account
      const newAccount = await program.account.escrowState.fetch(
        escrowState
      )

      const escrowBalance = await provider.connection.getBalance(escrowState, "confirmed")
      console.log("On-chain unlock price:", newAccount.unlockPrice)
      console.log("Amount in escrow:", escrowBalance)

      // Check whether the data on-chain is equal to local 'data'
      assert(failUnlockPrice == newAccount.unlockPrice)
      assert(escrowBalance > 0)
    } catch (e) {
      console.log(e)
      assert.fail(e)
    }
  })

  it("Withdraw from escrow", async () => {
    // derive escrow address
    const [escrowState] = await anchor.web3.PublicKey.findProgramAddressSync(
      [Buffer.from("MICHAEL BURRY"), payer.publicKey.toBuffer()],
      program.programId
    )
    
    // send tx
    const tx = await program.methods.withdraw()
    .accounts({
      user: payer.publicKey,
      escrowAccount: escrowState,
      feedAggregator: solUsedSwitchboardFeed,
      systemProgram: anchor.web3.SystemProgram.programId
  })
    .signers([payer])
    .rpc()

    await provider.connection.confirmTransaction(tx, "confirmed")

    // assert that the escrow account has been closed
    let accountFetchDidFail = false;
    try {
      await program.account.escrowState.fetch(escrowState)
    } catch(e){
      accountFetchDidFail = true;
    }

    assert(accountFetchDidFail)
 
  })

  it("Create Burry Escrow Above Price", async () => {
    // fetch switchboard devnet program object
    const switchboardProgram = await SwitchboardProgram.load(
      "devnet",
      new anchor.web3.Connection("https://api.devnet.solana.com"),
      payer
    )
    const aggregatorAccount = new AggregatorAccount(switchboardProgram, solUsedSwitchboardFeed)

    // derive escrow state account
    const [escrowState] = await anchor.web3.PublicKey.findProgramAddressSync(
      [Buffer.from("MICHAEL BURRY"), payer.publicKey.toBuffer()],
      program.programId
    )
    console.log("Escrow Account: ", escrowState.toBase58())

    // fetch latest SOL price
    const solPrice: Big | null = await aggregatorAccount.fetchLatestValue()
    if (solPrice === null) {
      throw new Error('Aggregator holds no value')
    }
    const failUnlockPrice = solPrice.plus(10).toNumber()
    const amountToLockUp = new anchor.BN(100)

    // Send transaction
    try {
      const tx = await program.methods.deposit(
        amountToLockUp, 
        failUnlockPrice
      )
      .accounts({
        user: payer.publicKey,
        escrowAccount: escrowState,
        systemProgram: anchor.web3.SystemProgram.programId
      })
      .signers([payer])
      .rpc()

      await provider.connection.confirmTransaction(tx, "confirmed")
      console.log("Your transaction signature", tx)

      // Fetch the created account
      const newAccount = await program.account.escrowState.fetch(
        escrowState
      )

      const escrowBalance = await provider.connection.getBalance(escrowState, "confirmed")
      console.log("On-chain unlock price:", newAccount.unlockPrice)
      console.log("Amount in escrow:", escrowBalance)

      // Check whether the data on-chain is equal to local 'data'
      assert(failUnlockPrice == newAccount.unlockPrice)
      assert(escrowBalance > 0)
    } catch (e) {
      console.log(e)
      assert.fail(e)
    }
  })

  it("Attempt to withdraw while price is below UnlockPrice", async () => {
    let didFail = false;

    // derive escrow address
    const [escrowState] = await anchor.web3.PublicKey.findProgramAddressSync(
      [Buffer.from("MICHAEL BURRY"), payer.publicKey.toBuffer()],
      program.programId
    )
    
    // send tx
    try {
      const tx = await program.methods.withdraw()
      .accounts({
        user: payer.publicKey,
        escrowAccount: escrowState,
        feedAggregator: solUsedSwitchboardFeed,
        systemProgram: anchor.web3.SystemProgram.programId
    })
      .signers([payer])
      .rpc()

      await provider.connection.confirmTransaction(tx, "confirmed")
      console.log("Your transaction signature", tx)

    } catch (e) {
      // verify tx returns expected error
      didFail = true;
      console.log(e.error.errorMessage)
      assert(e.error.errorMessage == 'Current SOL price is not above Escrow unlock price.')
    }

    assert(didFail)
  })
});
```

如果您对测试逻辑感到有信心，请在您选择的 shell 中运行 `anchor test`。您应该会得到四个通过的测试。

如果出现了问题，请返回实验室并确保您的一切都是正确的。请特别注意代码背后的意图，而不仅仅是复制/粘贴。您还可以随时查看 [GitHub 仓库的 `main` 分支上的工作代码](https://github.com/Unboxed-Software/michael-burry-escrow)。

## 挑战

作为一个独立的挑战，创建一个备用方案，以防数据源出现故障。如果 Oracle 队列在 X 时间内没有更新聚合器账户，或者数据源账户不再存在，则提取用户的托管资金。

可以在 [GitHub 仓库的 `challenge-solution` 分支上找到这个挑战的一个潜在解决方案](https://github.com/Unboxed-Software/michael-burry-escrow/tree/challenge-solution)。

## 完成实验了吗？

将您的代码推送到 GitHub，并[告诉我们您对这节课的看法](https://form.typeform.com/to/IPH0UGz7#answers-lesson=1a5d266c-f4c1-4c45-b986-2afd4be59991)!