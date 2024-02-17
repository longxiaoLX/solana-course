---
title: 程序架构
objectives:
- 使用 Box 和 Zero Copy 处理链上的大数据
- 做出更好的 PDA 设计决策
- 使您的程序具有未来的可扩展性
- 处理并发问题
---

# TL;DR

- 如果您的数据账户过大而无法存放在栈（Stack）中，请使用 `Box` 封装它们并分配到堆中。
- 使用 Zero-Copy 来处理那些对于 `Box` 来说过大（小于 10 MB）的账户。
- 账户中字段的大小和顺序很重要；将可变长度的字段放在最后。
- Solana 可以并行处理，但您仍可能遇到瓶颈；请注意所有用户与程序交互都需要写入的“共享”账户。

# 概述

程序架构是业余爱好者和专业人士的分水岭。打造高性能的程序更多地与系统设计有关，而不仅仅是与代码有关。而作为设计者，您需要考虑：

1. 您的代码需要做什么
2. 可能有哪些实现方式
3. 不同实现之间的权衡是什么

在为区块链开发时，这些问题甚至更为重要。资源不仅比典型计算环境更有限，而且您还在处理人们的资产；现在代码是有成本的。

我们将把大部分资产处理讨论留给[安全课程](./security-intro.md)，但重要的是要注意 Solana 开发中资源限制的性质。当然，在典型的开发环境中会有限制，但在区块链和 Solana 开发中存在着独特的限制，例如可以存储在账户中的数据量、存储该数据的成本以及每个交易可用的计算单元（compute units）数量。作为程序设计者，您必须谨记这些限制，以创建既经济、快速、安全又功能齐全的程序。今天，我们将深入探讨在创建 Solana 程序时应考虑的一些更高级的因素。

## 处理大型账户

在现代应用程序编程中，我们通常不必考虑正在使用的数据结构的大小。想要创建一个字符串？如果想避免滥用，您可以将其限制为 4000 个字符，但这可能不是一个问题。想要一个整数？出于方便起见，它们几乎总是 32 位的。

在高级语言中，您可以尽情使用数据！然而，在 Solana 中，我们按字节存储（租金）付费，并且对堆（heap）、栈（stack）和账户大小（account sizes）有限制。我们必须更加巧妙地处理字节。在本节中，我们将着重关注两个主要问题：

1. 由于我们按字节付费，通常希望将我们的占用尽可能地减小。我们将在另一节中更深入地探讨优化，但在这里，我们将向您介绍数据大小的概念。

2. 当操作较大数据时，我们会遇到[栈 Stack](https://docs.solana.com/developing/on-chain-programs/faq#stack)和[堆 Heap](https://docs.solana.com/developing/on-chain-programs/faq#heap-size)约束 - 为了解决这些问题，我们将看看如何使用 Box 和 Zero-Copy。

### 大小

在 Solana 上，交易的付费者为存储在链上的每个字节支付费用。我们称之为[租金](https://docs.solana.com/developing/intro/rent)。顺便说一句：租金有点名不副实，因为它实际上从来没有被永久性地收取过。一旦您将租金存入账户，该数据可以永远留在那里，或者如果您关闭账户，则可以退还租金。租金曾经是一个实际的东西，但现在有了强制的最低租金豁免（minimum rent exemption）。您可以在[Solana 文档](https://docs.solana.com/developing/intro/rent)中了解更多信息。

抛开租金的词源学不谈，将数据存储在区块链上可能很昂贵。这就是为什么 NFT 属性和相关文件，如图像，被存储在链外的原因。您最终希望在保持程序高度功能性的同时达到平衡，而不至于变得过于昂贵，以至于您的用户不愿意支付以打开数据账户。

在您开始为程序中的空间进行优化之前，您需要知道每个结构体的大小。以下是来自 [Anchor Book](https://book.anchor-lang.com/anchor_references/space.html) 的一份非常有用的列表。

| 类型 | 字节大小/bytes | 详细信息/示例 |
| --- | --- | --- |
| bool | 1 | 实际上只需要 1 位，但仍使用 1 字节 |
| u8/i8 | 1 |  |
| u16/i16 | 2 |  |
| u32/i32 | 4 |  |
| u64/i64 | 8 |  |
| u128/i128 | 16 |  |
| [T;amount] | space(T) * amount | 例如，space([u16;32]) = 2 * 32 = 64 |
| Pubkey | 32 |  |
| Vec<T> | 4 + (space(T) * amount) | 账户大小是固定的，因此账户应该从一开始就以足够的空间初始化 |
| String | 4 + 字符串长度（以字节为单位） | 账户大小是固定的，因此账户应该从一开始就以足够的空间初始化 |
| Option<T> | 1 + (space(T)) |  |
| Enum | 1 + 最大变体大小 | 例如，枚举 { A, B { val: u8 }, C { val: u16 } } -> 1 + space(u16) = 3 |
| f32 | 4 | 序列化将在 NaN 时失败 |
| f64 | 8 | 序列化将在 NaN 时失败 |
| Accounts | 8 + space(T) | #[account()] <br> pub struct T { … } |
| 数据结构 | space(T) | #[derive(Clone, AnchorSerialize, AnchorDeserialize)]<br> pub struct T { … } |

了解了这些，开始考虑您可能在程序中采取的一些小优化措施。例如，如果您有一个整数字段，它永远不会超过 100，那么不要使用 u64/i64，而应该使用 u8。为什么？因为一个 u64 占用 8 个字节，最大值为 2^64 或 1.84 * 10^19。这是一种空间浪费，因为您只需要容纳最大值为 100 的数字。一个字节将给您一个最大值为 255，在这种情况下已经足够了。类似地，如果永远不会出现负数，那就没有理由使用 i8。

不过，小的数值类型要格外注意。由于溢出，您可能会很快遇到意外行为。例如，一个逐步递增的 u8 类型将达到 255，然后返回到 0，而不是 256。要了解更多现实世界的背景，请查看**[Y2K bug](https://www.nationalgeographic.org/encyclopedia/Y2K-bug/#:~:text=As%20the%20year%202000%20approached%2C%20computer%20programmers%20realized%20that%20computers,would%20be%20damaged%20or%20flawed.)**。

如果您想进一步了解 Anchor 大小，请参阅 [Sec3 的博客文章](https://www.sec3.dev/blog/all-about-anchor-account-size)。

### Box

现在您已经了解了一些关于数据大小的知识，让我们更进一步，看看如果您想处理更大的数据账户时可能会遇到的问题。假设您有以下数据账户：

```rust
#[account]
pub struct SomeBigDataStruct {
    pub big_data: [u8; 5000],
}  

#[derive(Accounts)]
pub struct SomeFunctionContext<'info> {
    pub some_big_data: Account<'info, SomeBigDataStruct>,
}
```

如果您尝试将 `SomeBigDataStruct` 与 `SomeFunctionContext` 上下文一起传递给函数，您将会遇到以下编译器警告：

```
Stack offset of XXXX exceeded max offset of 4096 by XXXX bytes, please minimize large stack variables
```

如果尝试运行程序，它将会挂起并失败。

为什么会这样呢？

这与栈有关。在 Solana 中，每次调用函数时，都会获得一个 4KB 的栈帧（stack frame）。这是用于局部变量的静态内存分配。整个 `SomeBigDataStruct` 存储在内存中的位置就是在这里，由于其大小为 5000 字节，即 5KB，超过了 4KB 的限制，因此会引发栈错误。那么我们该如何解决这个问题呢？

答案就是 **`Box<T>`** 类型！

```rust
#[account]
pub struct SomeBigDataStruct {
    pub big_data: [u8; 5000],
}  

#[derive(Accounts)]
pub struct SomeFunctionContext<'info> {
    pub some_big_data: Box<Account<'info, SomeBigDataStruct>>, // <- Box Added!
}
```

在 Anchor 中，**`Box<T>`** 用于将账户分配到堆，而不是栈。这很棒，因为堆为我们提供了 32KB 的空间。最好的部分是，在函数内部您无需进行任何不同的操作。您只需要在所有大数据账户周围添加 `Box<…>` 即可。

但是，Box 也并非完美无缺。对于足够大的账户，仍然可能会溢出堆栈。我们将在下一节学习如何解决这个问题。

### Zero Copy

好的，现在您可以使用 `Box` 处理中等大小的账户。但是如果您需要使用最大大小为 10MB 的非常大的账户怎么办？根据以下内容为例：

```rust
#[account]
pub struct SomeReallyBigDataStruct {
    pub really_big_data: [u128; 1024], // 16,384 bytes
}
```

这种账户即使被包装在 `Box` 中，也会导致您的程序失败。为了解决这个问题，您可以使用 `zero_copy` 和 `AccountLoader`。只需在您的账户结构体中添加 `zero_copy`，在账户验证结构体中将 `zero` 添加为一个约束条件，并将账户验证结构体中的账户类型封装在 `AccountLoader` 中。

```rust
#[account(zero_copy)]
pub struct SomeReallyBigDataStruct {
    pub really_big_data: [u128; 1024], // 16,384 bytes
}

pub struct ConceptZeroCopy<'info> {
    #[account(zero)]
    pub some_really_big_data: AccountLoader<'info, SomeReallyBigDataStruct>,
}
```

要理解这里发生的事情，请查看[rust Anchor 文档](https://docs.rs/anchor-lang/latest/anchor_lang/attr.account.html)

> 除了更有效率之外，[`zero_copy`]提供的最显着的好处是能够定义大于最大栈或堆大小的账户类型。当使用 borsh 时，账户必须被复制并反序列化为新的数据结构，因此受到 BPF VM 强加的栈和堆限制的约束。使用零拷贝反序列化时，所有来自账户的 `RefCell<&mut [u8]>` 的字节都简单地被重新解释为对数据结构的引用。不需要分配或复制。因此能够绕过栈和堆的限制。

基本上，您的程序实际上从未将零拷贝账户数据加载到栈或堆中。它实际上是获取指向原始数据的指针访问。`AccountLoader` 确保这并不会太大地改变您从代码中与账户交互的方式。

使用 `zero_copy` 有几个注意事项。首先，您不能像您可能习惯的那样在账户验证结构体中使用 `init` 约束。这是因为账户大于 10KB 时存在 CPI 限制。

```rust
pub struct ConceptZeroCopy<'info> {
    #[account(zero, init)] // <- Can't do this
    pub some_really_big_data: AccountLoader<'info, SomeReallyBigDataStruct>,
}
```

相反，您的客户端必须在单独的指令中创建大型账户并为其支付租金。

```tsx
const accountSize = 16_384 + 8
const ix = anchor.web3.SystemProgram.createAccount({
  fromPubkey: wallet.publicKey,
  newAccountPubkey: someReallyBigData.publicKey,
  lamports: await program.provider.connection.getMinimumBalanceForRentExemption(accountSize),
  space: accountSize,
  programId: program.programId,
});

const txHash = await program.methods.conceptZeroCopy().accounts({
  owner: wallet.publicKey,
  someReallyBigData: someReallyBigData.publicKey,
}).signers([
  someReallyBigData,
]).preInstructions([
  ix
])
.rpc()
```

第二个注意事项是您将不得不在 Rust 指令函数内部调用以下方法之一来加载账户：

- 当首次初始化账户时，使用 `load_init`（这将忽略仅在用户指令代码之后添加的缺失的账户鉴别器）
- 当账户不可变时使用 `load`
- 当账户是可变的时使用 `load_mut`

例如，如果您想要初始化并操作上述的 `SomeReallyBigDataStruct`，您将在函数中调用以下方法：

```rust
let some_really_big_data = &mut ctx.accounts.some_really_big_data.load_init()?;
```

在您完成这些操作之后，您就可以像处理普通账户一样对待该账户了！请随意在代码中尝试这些内容，以查看它们的实际效果！

为了更好地理解所有这些是如何工作的，Solana 制作了一个非常好的[视频](https://www.youtube.com/watch?v=zs_yU0IuJxc&feature=youtu.be)和[代码](https://github.com/solana-developers/anchor-zero-copy-example)，解释了在普通的 Solana 中如何使用 Box 和 Zero-Copy。

## 处理账户

现在您已经了解了 Solana 上空间考虑的基本知识，让我们来看一些更高层次的考虑。在 Solana 中，一切都是账户，因此在接下来的几节中，我们将看一些账户架构概念。

### 数据顺序

这个考虑相对简单。作为一个经验法则，将所有可变长度字段保持在账户的末尾。看一下以下内容：

```rust
#[account] // Anchor hides the account discriminator
pub struct BadState {
    pub flags: Vec<u8>, // 0x11, 0x22, 0x33 ...
    pub id: u32         // 0xDEAD_BEEF
}
```

`flags` 字段是可变长度的。这使得通过 `id` 字段查找特定账户变得非常困难，因为对 `flags` 中数据的更新会改变 `id` 在内存映射中的位置。

为了更清楚地说明这一点，观察一下当 `flags` 中的向量有四个元素和八个元素时，此账户数据在链上是什么样子的。如果您调用 `solana account ACCOUNT_KEY`，您将获得以下数据转储：

```rust
0000:   74 e4 28 4e    d9 ec 31 0a  -> Account Discriminator (8)
0008:	04 00 00 00    11 22 33 44  -> Vec Size (4) | Data 4*(1)
0010:   DE AD BE EF                 -> id (4)

--- vs ---

0000:   74 e4 28 4e    d9 ec 31 0a  -> Account Discriminator (8)
0008:	08 00 00 00    11 22 33 44  -> Vec Size (8) | Data 4*(1)
0010:   55 66 77 88    DE AD BE EF  -> Data 4*(1) | id (4)
```

在这两种情况下，前八个字节是 Anchor 账户鉴别器（account discriminator）。在第一种情况下，接下来的四个字节表示 `flags` 向量的大小，紧接着是另外四个字节用于数据，最后是 `id` 字段的数据。

在第二种情况下，`id` 字段从地址 0x0010 移动到了 0x0014，因为 `flags` 字段中的数据占用了额外的四个字节。

这样做的主要问题在于查找。当您查询 Solana 时，您使用的是查看账户原始数据的过滤器。这些称为 `memcmp` 过滤器，或内存比较过滤器。您给过滤器提供一个 `offset` 和 `bytes`，然后过滤器直接查看内存，通过您提供的 `offset` 从开始位置偏移，并将内存中的字节与您提供的 `bytes` 进行比较。

例如，您知道 `flags` 结构体将始终从地址 0x0008 开始，因为前八个字节包含账户鉴别器。查询所有 `flags` 长度等于四的账户是可能的，因为我们*知道*地址 0x0008 处的四个字节表示 `flags` 中的数据长度。

```typescript
const states = await program.account.badState.all([
  {memcmp: {
    offset: 8,
    bytes: bs58.encode([0x04])
  }}
]);
```

然而，如果您想要根据 `id` 进行查询，您就不知道该将什么放在 `offset` 中，因为 `id` 的位置是根据 `flags` 的长度变化的。这似乎并不是很有帮助。ID 通常是用来帮助查询的！简单的解决方法是颠倒顺序。

```rust
#[account] // Anchor hides the account disriminator
pub struct GoodState {
	pub id: u32         // 0xDEAD_BEEF
    pub flags: Vec<u8>, // 0x11, 0x22, 0x33 ...
}
```

将可变长度字段放在结构体的末尾，您就可以始终根据第一个可变长度字段之前的所有字段查询账户。再次强调本节的开头：作为一个经验法则，将所有可变长度的结构体放在账户的末尾。

### 为将来使用

在某些情况下，考虑向您的账户添加额外的未使用字节。这些字节被保留用于灵活性和向后兼容性。看下面的例子：

```rust
#[account]
pub struct GameState {
    pub health: u64,
    pub mana: u64,
    pub event_log: Vec<string>
}
```

在这个简单的游戏状态中，角色有 `health`、`mana` 和一个事件日志。如果在某个时刻您正在进行游戏改进，并希望添加一个 `experience` 字段，您会遇到一些问题。`experience` 字段应该是一个像 `u64` 这样的数字，这很容易添加。您可以[重新分配账户](./anchor-pdas.md#realloc)并添加空间。

然而，为了保持动态长度字段，比如 `event_log`，在结构体的末尾，您需要对所有重新分配的账户进行一些内存操作，以移动 `event_log` 的位置。这可能会很复杂，并且使得查询账户变得更加困难。您将会处于这样一种状态：未迁移的账户的 `event_log` 位于一个位置，而已迁移的账户位于另一个位置。没有 `experience` 的旧 `GameState` 和新的带有 `experience` 的 `GameState` 不再兼容。旧账户在期望使用新账户的位置时将无法序列化。查询将变得更加困难。您可能需要创建一个迁移系统和持续的逻辑来维护向后兼容性。最终，这开始看起来像是一个不好的主意。

幸运的是，如果您提前考虑，您可以添加一个 `for_future_use` 字段，保留一些字节，以备不时之需。

```rust
#[account]
pub struct GameState { //V1
    pub health: u64,
    pub mana: u64,
	pub for_future_use: [u8; 128],
    pub event_log: Vec<string>
}
```

这样，当您添加 `experience` 或类似的字段时，情况就变成了这样，旧账户和新账户都是兼容的。

```rust
#[account]
pub struct GameState { //V2
    pub health: u64,
    pub mana: u64,
	pub experience: u64,
	pub for_future_use: [u8; 120],
    pub event_log: Vec<string>
}
```

因此，这些额外的字节确实会增加使用您的程序的成本。然而，在大多数情况下，这似乎是非常值得的。

因此，作为一个一般的经验法则：每当您认为您的账户类型有可能以一种需要某种复杂迁移的方式发生变化时，请添加一些 `for_future_use` 字节。

### 数据优化

这里的思想是要注意浪费的位。例如，如果您有一个表示年份的字段，请不要使用 `u64`。一年只有 12 个月。使用一个 `u8`。更好的做法是，使用一个 `u8` 枚举并标记月份。

要更加积极地节省位，小心处理布尔值。看下面由八个布尔标志组成的结构体。虽然布尔值 *可以* 用单个位表示，但 borsh 反序列化将为这些字段中的每一个分配整个字节。这意味着八个布尔值将变成八个字节，而不是八个位，大小增加了八倍。

```rust
#[account]
pub struct BadGameFlags { // 8 bytes
    pub is_frozen: bool,
    pub is_poisoned: bool,
    pub is_burning: bool,
    pub is_blessed: bool,
    pub is_cursed: bool,
    pub is_stunned: bool,
    pub is_slowed: bool,
    pub is_bleeding: bool,
}
```

为了优化这个问题，您可以使用一个 `u8` 字段。然后，您可以使用位操作来查看每个位，并确定它是否被“切换”。

```rust
const IS_FROZEN_FLAG: u8 = 1 << 0;
const IS_POISONED_FLAG: u8 = 1 << 1;
const IS_BURNING_FLAG: u8 = 1 << 2;
const IS_BLESSED_FLAG: u8 = 1 << 3;
const IS_CURSED_FLAG: u8 = 1 << 4;
const IS_STUNNED_FLAG: u8 = 1 << 5;
const IS_SLOWED_FLAG: u8 = 1 << 6;
const IS_BLEEDING_FLAG: u8 = 1 << 7;
const NO_EFFECT_FLAG: u8 = 0b00000000;
#[account]
pub struct GoodGameFlags { // 1 byte
    pub status_flags: u8, 
} 
```

这样可以节省 7 个字节的数据！当然，这样做的代价是现在您必须执行位操作。但这是值得放入您的工具箱中的技巧。

### 索引

这个最后的账户概念很有趣，展示了 PDA 的强大之处。在创建程序账户时，您可以指定用于派生 PDA 的种子。这非常强大，因为它允许您派生您的账户地址而不是存储它们。

最好的例子就是 Associated Token Accounts (ATAs)！

```typescript
function findAssociatedTokenAddress(
  walletAddress: PublicKey,
  tokenMintAddress: PublicKey
): PublicKey {
  return PublicKey.findProgramAddressSync(
    [
      walletAddress.toBuffer(),
      TOKEN_PROGRAM_ID.toBuffer(),
      tokenMintAddress.toBuffer(),
    ],
    SPL_ASSOCIATED_TOKEN_ACCOUNT_PROGRAM_ID
  )[0];
}
```

这是大多数 SPL 代币存储的方式。您无需保留 SPL 代币账户地址的数据库表，您只需知道您的钱包地址和代币地址。ATA 地址可以通过将它们哈希在一起来计算，然后您就有了您的代币账户地址。

根据种子，您可以创建各种关系：

- One-Per-Program (全局账户) - 如果您创建一个带有确定的 `seeds=[b"ONE PER PROGRAM"]` 的账户，在该程序中该种子只能存在一个。例如，如果您的程序需要一个查找表，您可以使用 `seeds=[b"Lookup"]` 这个种子。只要小心提供适当的访问限制。
- One-Per-Owner (每个所有者一个) - 假设您正在创建一个电子游戏玩家账户，并且您希望每个钱包只有一个玩家账户。那么您会使用 `seeds=[b"PLAYER", owner.key().as_ref()]` 这个种子来派生账户。这样，您总是知道在哪里查找钱包的玩家账户 **而且** 只能有一个这样的账户。
- Multiple-Per-Owner (每个所有者多个) - 好了，但是如果您希望每个钱包有多个账户怎么办？假设您想铸造播客剧集（podcast episodes）。然后，您可以像这样对 `Podcast` 账户进行种子：`seeds=[b"Podcast", owner.key().as_ref(), episode_number.to_be_bytes().as_ref()]`。现在，如果您想要查找特定钱包的第 50 集，您可以！而且您可以拥有任意数量的剧集。
- One-Per-Owner-Per-Account (每个所有者每个账户一个) - 这实际上就是我们上面看到的 ATA 的例子。在这里，我们每个钱包和代币账户只有一个令牌账户。`seeds=[b"Mock ATA", owner.key().as_ref(), mint.key().as_ref()]`

从那里，您可以以各种聪明的方式进行混合和匹配！但是前面的列表应该足以让您开始。

对于真正关注设计这个方面的一个巨大好处就是解决“索引”问题。如果没有 PDAs 和种子，所有用户都必须跟踪他们曾经使用过的所有账户的所有地址。这对用户来说是不可行的，因此他们必须依赖一个集中式实体来将他们的地址存储在数据库中。在许多方面，这背离了全球分布式网络的目的。PDAs 是一个更好的解决方案。

为了加强这一点，这里是一个来自产生播客程序的方案的示例。该程序需要以下账户：

- **频道（channel）账户**
    - 名称
    - 创建的剧集数 (Episodes, u64)
- **播客（podcast）账户**
    - 名称
    - 音频 URL（Audio）

为了正确索引每个账户地址，账户使用以下种子：

```rust
// Channel Account
seeds=[b"Channel", owner.key().as_ref()]

// Podcast Account
seeds=[b"Podcast", channel_account.key().as_ref(), episode_number.to_be_bytes().as_ref()]
```

您始终可以找到特定所有者的频道账户。而且，由于频道存储了创建的剧集数，您始终知道在哪里搜索出可以查询的上限。此外，您始终知道要在哪个索引处创建新剧集：`index = episodes_created`。

```rust
Podcast 0: seeds=[b"Podcast", channel_account.key().as_ref(), 0.to_be_bytes().as_ref()] 
Podcast 1: seeds=[b"Podcast", channel_account.key().as_ref(), 1.to_be_bytes().as_ref()] 
Podcast 2: seeds=[b"Podcast", channel_account.key().as_ref(), 2.to_be_bytes().as_ref()] 
...
Podcast X: seeds=[b"Podcast", channel_account.key().as_ref(), X.to_be_bytes().as_ref()] 
```

## 处理并发

选择 Solana 作为您的区块链环境的主要原因之一是其并行事务执行能力。也就是说，只要这些交易不试图向同一个账户写入数据，Solana 就可以并行运行交易。这在出厂时就提高了程序的吞吐量，但通过一些适当的规划，您可以避免并发问题，真正提升程序的性能。

### 共享账户

如果您在加密货币领域呆了一段时间，您可能经历过一个大规模的 NFT 铸造事件。一个新的 NFT 项目即将推出，每个人都对此感到非常兴奋，然后糖果机开始运行。大家争先恐后地点击 `接受交易`，尽可能快地点击。如果您足够聪明，您可能已经编写了一个机器人，以比网站 UI 更快的速度输入交易。这种疯狂的铸造狂潮导致了大量的失败交易。但是为什么呢？因为每个人都试图向同一个糖果机账户写入数据。

看一个简单的例子：

Alice 和 Bob 分别尝试支付他们的朋友 Carol 和 Dean。所有四个账户都会发生变化，但彼此之间没有依赖关系。两笔交易可以同时运行。

```rust
Alice -- pays --> Carol

Bob ---- pays --> Dean
```

但如果 Alice 和 Bob 同时尝试支付给 Carol，他们会遇到问题。

```rust
Alice -- pays --> |
						-- > Carol
Bob   -- pays --- |
```

由于这两笔交易都要向 Carol 的代币账户写入数据，因此一次只能处理其中一笔。幸运的是，Solana 处理速度非常快，所以它们可能看起来几乎同时进行了支付。但是如果不止 Alice 和 Bob 尝试支付给 Carol 呢？

```rust
Alice -- pays --> |
						-- > Carol
x1000 -- pays --- | 
Bob   -- pays --- |
```

如果有 1000 人同时尝试向 Carol 支付，那么这 1000 条指令将排队顺序运行。对于其中的一些人来说，支付似乎会立即完成。他们将是那些早期被包含的幸运者。但是有些人将会等待相当长的时间。对于一些人来说，他们的交易将会简单地失败。

虽然有 1000 人同时支付给 Carol 的情况似乎不太可能发生，但实际上，经常会发生像 NFT 铸造这样的事件，许多人同时尝试向同一个账户写入数据。

想象一下，您创建了一个超级受欢迎的程序，并且您希望在每笔交易中收取一笔费用。出于会计原因，您希望所有这些费用都流入一个钱包。使用这种设置，当用户激增时，您的协议将变得缓慢或变得不可靠。这并不理想。那么解决方案是什么呢？将数据交易与费用交易分开。

例如，假设您有一个名为 `DonationTally` 的数据账户。它的唯一功能是记录您向一个特定的硬编码社区钱包捐赠了多少。

```rust
#[account]
pub struct DonationTally {
    is_initialized: bool,
    lamports_donated: u64,
    lamports_to_redeem: u64,
    owner: Pubkey,
}
```

首先让我们看一下次优解决方案。

```rust
pub fn run_concept_shared_account_bottleneck(
    ctx: Context<ConceptSharedAccountBottleneck>,
    lamports_to_donate: u64,
) -> Result<()> {
    let donation_tally = &mut ctx.accounts.donation_tally;

    if !donation_tally.is_initialized {
        donation_tally.is_initialized = true;
        donation_tally.owner = ctx.accounts.owner.key();
        donation_tally.lamports_donated = 0;
        donation_tally.lamports_to_redeem = 0;
    }

    let cpi_context = CpiContext::new(
        ctx.accounts.system_program.to_account_info(),
        Transfer {
            from: ctx.accounts.owner.to_account_info(),
            to: ctx.accounts.community_wallet.to_account_info(),
        },
    );
    transfer(cpi_context, lamports_to_donate)?;

    donation_tally.lamports_donated = donation_tally
        .lamports_donated
        .checked_add(lamports_to_donate)
        .unwrap();
    donation_tally.lamports_to_redeem = 0;

    Ok(())
}
```

您可以看到，向硬编码的 `community_wallet` 进行转账是在更新统计信息的同一个函数中完成的。这是最直接的解决方案，但是如果您运行本节的测试，您会看到性能下降。

现在让我们看一下优化后的解决方案：

```rust
pub fn run_concept_shared_account(
    ctx: Context<ConceptSharedAccount>,
    lamports_to_donate: u64,
) -> Result<()> {
    let donation_tally = &mut ctx.accounts.donation_tally;

    if !donation_tally.is_initialized {
        donation_tally.is_initialized = true;
        donation_tally.owner = ctx.accounts.owner.key();
        donation_tally.lamports_donated = 0;
        donation_tally.lamports_to_redeem = 0;
    }

    let cpi_context = CpiContext::new(
        ctx.accounts.system_program.to_account_info(),
        Transfer {
            from: ctx.accounts.owner.to_account_info(),
            to: donation_tally.to_account_info(),
        },
    );
    transfer(cpi_context, lamports_to_donate)?;

    donation_tally.lamports_donated = donation_tally
        .lamports_donated
        .checked_add(lamports_to_donate)
        .unwrap();
    donation_tally.lamports_to_redeem = donation_tally
        .lamports_to_redeem
        .checked_add(lamports_to_donate)
        .unwrap();

    Ok(())
}

pub fn run_concept_shared_account_redeem(
    ctx: Context<ConceptSharedAccountRedeem>,
) -> Result<()> {
    let transfer_amount: u64 = ctx.accounts.donation_tally.lamports_donated;

    // Decrease balance in donation_tally account
    **ctx
        .accounts
        .donation_tally
        .to_account_info()
        .try_borrow_mut_lamports()? -= transfer_amount;

    // Increase balance in community_wallet account
    **ctx
        .accounts
        .community_wallet
        .to_account_info()
        .try_borrow_mut_lamports()? += transfer_amount;

    // Reset lamports_donated and lamports_to_redeem
    ctx.accounts.donation_tally.lamports_to_redeem = 0;

    Ok(())
}
```

在 `run_concept_shared_account` 函数中，我们将转账目标从 `community_wallet` 改为 `donation_tally` PDA。这样，我们只影响捐赠者的账户和他们的 PDA，因此没有瓶颈！此外，我们在内部记录了需要赎回（redeem）的 lamports 总数，即以后需要从 PDA 转账到社区钱包的 lamports。在将来的某个时间点，社区钱包将清理所有剩余的 lamports（可能是 [clockwork](https://www.clockwork.xyz/) 的一项良好工作）。重要的是，任何人都应该能够对赎回函数进行签名，因为 PDA 对自身具有权限。

如果您想尽一切办法避免瓶颈，这是一种处理方式。最终这是一个设计决策，对于某些程序来说，更简单、不太优化的解决方案可能是可以接受的。但是，如果您的程序将会有很高的流量，那么值得尝试优化。您始终可以运行模拟来查看最坏、最好和中位数的情况。

## 看它实际运作

本课程中的所有代码片段都是我们创建的一个 [Solana 程序](https://github.com/Unboxed-Software/advanced-program-architecture.git)，用于说明这些概念。每个概念都有相应的程序和测试文件。例如，**Sizes** 概念可以在以下位置找到：

**程序 -** `programs/architecture/src/concepts/sizes.rs`

**测试 -** `cd tests/sizes.ts`

既然您已经阅读了每个概念，就可以随意进入代码中进行一些实验。您可以更改现有值，尝试破坏程序，并尝试理解一切是如何工作的。

您可以从 Github 上 fork 或 clone [此程序](https://github.com/Unboxed-Software/advanced-program-architecture.git) 来开始。在构建和运行测试套件之前，请记得使用您的本地程序 ID 更新 `lib.rs` 和 `Anchor.toml`。

您可以运行整个测试套件，或者在特定测试文件的 `describe` 调用中添加 `.only`，以仅运行该文件的测试。随意自定义并使其成为您自己的。

## 结论

我们讨论了许多程序架构方面的考虑：字节、账户、瓶颈等等。无论您是否遇到了这些具体的考虑因素，希望这些示例和讨论都能激发一些思考。归根结底，您是系统的设计者。您的工作是权衡各种解决方案的利弊。要有远见，但要务实。没有什么是“一种好的方法”来设计任何东西的。只需要知道各种选择的权衡。

# 实验

让我们利用所有这些概念来创建一个简单但优化的 Solana RPG 游戏引擎。该程序将具有以下功能：
- 允许用户创建游戏（`Game` 账户）并成为“游戏管理者”（对游戏的管理权限）
- 游戏管理者负责他们游戏的配置
- 任何公众都可以作为玩家加入游戏 - 每个玩家/游戏组合将拥有一个 `Player` 账户
- 玩家可以花费行动点生成和对抗怪物（`Monster` 账户）；我们将使用 lamports 作为行动点
- 花费的行动点将作为游戏账户中列出的游戏资金库存

在我们进行过程中，我们将逐步讨论各种设计决策的权衡，以便让您了解我们为什么这样做。让我们开始吧！

## 1. 程序设置

我们将从零开始构建这个项目。首先创建一个新的 Anchor 项目：

```powershell
anchor init rpg
```

接下来，运行 `anchor keys list`，将显示的程序 ID 替换 `programs/rpg/lib.rs` 和 `Anchor.toml` 中的程序 ID。

最后，在 `lib.rs` 文件中建立程序的框架。为了更容易跟随，我们将所有内容都放在一个文件中。我们将使用部分注释来更好地组织和导航。在我们开始之前，将以下内容复制到您的文件中：

```rust
use anchor_lang::prelude::*;
use anchor_lang::system_program::{Transfer, transfer};
use anchor_lang::solana_program::log::sol_log_compute_units;

declare_id!("YOUR_KEY_HERE__YOUR_KEY_HERE");

// ----------- ACCOUNTS ----------

// ----------- GAME CONFIG ----------

// ----------- STATUS ----------

// ----------- INVENTORY ----------

// ----------- HELPER ----------

// ----------- CREATE GAME ----------

// ----------- CREATE PLAYER ----------

// ----------- SPAWN MONSTER ----------

// ----------- ATTACK MONSTER ----------

// ----------- REDEEM TO TREASURY ----------

#[program]
pub mod rpg {
    use super::*;

}
```

## 2. 创建账户结构

现在我们的初始设置已准备就绪，让我们创建我们的账户。我们将有 3 个账户：

1. `Game` - 这个账户代表并管理一个游戏。它包括游戏参与者支付的库藏和游戏管理者可以用来自定义游戏的配置结构。它应包括以下字段：
    - `game_master` - 实际上是所有者/权限
    - `treasury` - 玩家将发送行动点数的库藏（我们将仅使用行动点数的 lamports）
    - `action_points_collected` - 跟踪库藏收集的行动点数的数量
    - `game_config` - 用于自定义游戏的配置结构
2. `Player` - 一个 PDA 账户，其地址是使用游戏账户地址和玩家的钱包地址作为种子派生而来的。它有很多字段用于跟踪玩家的游戏状态：
    - `player` - 玩家的公钥
    - `game` - 相应游戏账户的地址
    - `action_points_spent` - 消耗的行动点数
    - `action_points_to_be_collected` - 尚需收集的行动点数
    - `status_flag` - 玩家的状态标志
    - `experience` - 玩家的经验
    - `kills` - 杀死的怪物数量
    - `next_monster_index` - 下一个面对的怪物的索引
    - `for_future_use` - 保留供将来使用的 256 字节
    - `inventory` - 玩家物品清单的向量
3. `Monster` - 一个 PDA 账户，其地址是使用游戏账户地址、玩家的钱包地址和索引（存储在 `Player` 账户中作为 `next_monster_index`）派生而来的。
    - `player` - 怪物正在面对的玩家
    - `game` - 与怪物相关联的游戏
    - `hitpoints` - 怪物剩余的生命值

当添加到程序中时，账户应如下所示：

```rust
// ----------- ACCOUNTS ----------
#[account]
pub struct Game { // 8 bytes
    pub game_master: Pubkey,            // 32 bytes
    pub treasury: Pubkey,               // 32 bytes

    pub action_points_collected: u64,   // 8 bytes
    
    pub game_config: GameConfig,
}

#[account]
pub struct Player { // 8 bytes
    pub player: Pubkey,                 // 32 bytes
    pub game: Pubkey,                   // 32 bytes

    pub action_points_spent: u64,               // 8 bytes
    pub action_points_to_be_collected: u64,     // 8 bytes

    pub status_flag: u8,                // 8 bytes
    pub experience: u64,                 // 8 bytes
    pub kills: u64,                     // 8 bytes
    pub next_monster_index: u64,        // 8 bytes

    pub for_future_use: [u8; 256],      // Attack/Speed/Defense/Health/Mana?? Metadata??

    pub inventory: Vec<InventoryItem>,  // Max 8 items
}

#[account]
pub struct Monster { // 8 bytes
    pub player: Pubkey,                 // 32 bytes
    pub game: Pubkey,                   // 32 bytes

    pub hitpoints: u64,                 // 8 bytes
}
```

这里没有太多复杂的设计决策，但让我们谈谈 `Player` 结构中的 `inventory` 和 `for_future_use` 字段。由于 `inventory` 长度可变，我们决定将其放在账户末尾，以便查询更容易。我们还决定多花一点钱来获得 `for_future_use` 字段中保留的 256 字节空间。如果我们需要在将来添加字段，我们可以排除这个字段并简单地重新分配账户，但现在添加它可以简化未来的工作。

如果我们选择在将来重新分配，我们需要编写更复杂的查询，并且可能无法根据 `inventory` 进行单一调用的查询。重新分配并添加字段将移动 `inventory` 的内存位置，使我们需要编写复杂的逻辑来查询具有不同结构的账户。

## 3. 创建辅助类型

接下来，我们需要添加一些我们还没有创建的账户引用的类型。

让我们从游戏配置结构开始。技术上，这个结构可以放在 `Game` 账户中，但是分开和封装一些东西会更好。这个结构应该存储每个玩家允许的最大物品数量以及一些供将来使用的字节。同样，这里的将来使用的字节帮助我们避免了未来的复杂性。重新分配账户最好在在账户末尾添加字段而不是在中间添加字段。如果您预期在现有数据中间添加字段，可能最好在前面添加一些“将来使用（future use）”字节。

```rust
// ----------- GAME CONFIG ----------

#[derive(Clone, AnchorSerialize, AnchorDeserialize)]
pub struct GameConfig {
    pub max_items_per_player: u8,
    pub for_future_use: [u64; 16], // Health of Enemies?? Experience per item?? Action Points per Action??
}
```

接下来，让我们创建我们的状态标志。请记住，我们*可以*将我们的标志存储为布尔值，但通过将多个标志存储在单个字节中，我们可以节省空间。每个标志占据字节中的不同位。我们可以使用 `<<` 运算符将 `1` 放置在正确的位上。

```rust
// ----------- STATUS ----------

const IS_FROZEN_FLAG: u8 = 1 << 0;
const IS_POISONED_FLAG: u8 = 1 << 1;
const IS_BURNING_FLAG: u8 = 1 << 2;
const IS_BLESSED_FLAG: u8 = 1 << 3;
const IS_CURSED_FLAG: u8 = 1 << 4;
const IS_STUNNED_FLAG: u8 = 1 << 5;
const IS_SLOWED_FLAG: u8 = 1 << 6;
const IS_BLEEDING_FLAG: u8 = 1 << 7;
const NO_EFFECT_FLAG: u8 = 0b00000000;
```

最后，让我们创建我们的 `InventoryItem`。它应该有物品的名称、数量和一些保留供将来使用的字节。

```rust
// ----------- INVENTORY ----------

#[derive(Clone, AnchorSerialize, AnchorDeserialize)]
pub struct InventoryItem {
    pub name: [u8; 32], // Fixed Name up to 32 bytes
    pub amount: u64,
    pub for_future_use: [u8; 128], // Metadata?? // Effects // Flags?
}
```

## 4. 创建用于消耗行动点数的辅助函数

在编写程序说明之前，我们将做的最后一件事是创建一个用于消耗行动点数的辅助函数。玩家将发送行动点数（lamports）到游戏库藏作为在游戏中执行操作的支付。

由于将 lamports 发送到库藏需要向该库藏账户写入数据，如果许多玩家尝试同时向同一个库藏写入数据，我们很容易会遇到性能瓶颈（参见[处理并发](#处理并发)）。

相反，我们将把它们发送到玩家的 PDA 账户，并创建一个指令，该指令会一举将 lamports 从该账户发送到库藏。这样做可以缓解任何并发问题，因为每个玩家都有自己的账户，但也允许程序随时检索这些 lamports。

```rust
// ----------- HELPER ----------

pub fn spend_action_points<'info>(
    action_points: u64, 
    player_account: &mut Account<'info, Player>,
    player: &AccountInfo<'info>, 
    system_program: &AccountInfo<'info>, 
) -> Result<()> {

    player_account.action_points_spent = player_account.action_points_spent.checked_add(action_points).unwrap();
    player_account.action_points_to_be_collected = player_account.action_points_to_be_collected.checked_add(action_points).unwrap();

    let cpi_context = CpiContext::new(
        system_program.clone(), 
        Transfer {
            from: player.clone(),
            to: player_account.to_account_info().clone(),
        });
    transfer(cpi_context, action_points)?;

    msg!("Minus {} action points", action_points);

    Ok(())
}
```

## 5. 创建游戏

我们的第一个指令将创建 `game` 账户。任何人都可以成为 `game_master` 并创建自己的游戏，但一旦创建了游戏，就会有一些限制。

首先，`game` 账户是一个使用其 `treasury` 钱包的 PDA。这确保了相同的 `game_master` 如果为每个游戏使用不同的库藏，可以运行多个游戏。

还请注意，`treasury` 是指令中的签名者。这是为了确保创建游戏的人拥有 `treasury` 的私钥。这是一个设计决定，而不是“正确的方法”。最终，这是一项安全措施，以确保游戏管理者能够检索他们的资金。

```rust
// ----------- CREATE GAME ----------

#[derive(Accounts)]
pub struct CreateGame<'info> {
    #[account(
        init, 
        seeds=[b"GAME", treasury.key().as_ref()],
        bump,
        payer = game_master, 
        space = std::mem::size_of::<Game>()+ 8
    )]
    pub game: Account<'info, Game>,

    #[account(mut)]
    pub game_master: Signer<'info>,

    /// CHECK: Need to know they own the treasury
    pub treasury: Signer<'info>,
    pub system_program: Program<'info, System>,
}

pub fn run_create_game(ctx: Context<CreateGame>, max_items_per_player: u8) -> Result<()> {

    ctx.accounts.game.game_master = ctx.accounts.game_master.key().clone();
    ctx.accounts.game.treasury = ctx.accounts.treasury.key().clone();

    ctx.accounts.game.action_points_collected = 0;
    ctx.accounts.game.game_config.max_items_per_player = max_items_per_player;

    msg!("Game created!");

    Ok(())
}
```

## 6. 创建玩家

我们的第二个指令将创建 `player` 账户。有三个关于此指令需要注意的权衡：

1. 玩家账户是使用 `game` 和 `player` 钱包派生的 PDA 账户。这样可以让玩家参与多个游戏，但每个游戏只能有一个玩家账户。
2. 我们用一个 `Box` 将 `game` 账户包装起来，将其放在堆上，确保我们不会耗尽栈。
3. 玩家进行的第一个操作是生成自己，因此我们调用了 `spend_action_points`。现在我们将 `action_points_to_spend` 硬编码为 100 lamports，但这可能是将来游戏配置中添加的内容。

```rust
// ----------- CREATE PLAYER ----------
#[derive(Accounts)]
pub struct CreatePlayer<'info> {
    pub game: Box<Account<'info, Game>>,

    #[account(
        init, 
        seeds=[
            b"PLAYER", 
            game.key().as_ref(), 
            player.key().as_ref()
        ], 
        bump, 
        payer = player, 
        space = std::mem::size_of::<Player>() + std::mem::size_of::<InventoryItem>() * game.game_config.max_items_per_player as usize + 8)
    ]
    pub player_account: Account<'info, Player>,

    #[account(mut)]
    pub player: Signer<'info>,

    pub system_program: Program<'info, System>,
}

pub fn run_create_player(ctx: Context<CreatePlayer>) -> Result<()> {

    ctx.accounts.player_account.player = ctx.accounts.player.key().clone();
    ctx.accounts.player_account.game = ctx.accounts.game.key().clone();

    ctx.accounts.player_account.status_flag = NO_EFFECT_FLAG;
    ctx.accounts.player_account.experience = 0;
    ctx.accounts.player_account.kills = 0;

    msg!("Hero has entered the game!");

    {   // Spend 100 lamports to create player
        let action_points_to_spend = 100;

        spend_action_points(
            action_points_to_spend, 
            &mut ctx.accounts.player_account,
            &ctx.accounts.player.to_account_info(), 
            &ctx.accounts.system_program.to_account_info()
        )?;
    }

    Ok(())
}
```

## 7. 生成怪物

现在我们有了创建玩家的方法，我们需要一种方法来生成他们要对抗的怪物。这个指令将创建一个新的 `Monster` 账户，其地址是使用 `game` 账户、`player` 账户和一个表示玩家已经面对的怪物数量的索引派生的 PDA。这里有两个设计决策我们应该谈谈：
1. PDA 种子让我们能够跟踪玩家生成的所有怪物
2. 我们用 `Box` 包装了 `game` 和 `player` 账户，将它们分配到堆上

```rust
// ----------- SPAWN MONSTER ----------
#[derive(Accounts)]
pub struct SpawnMonster<'info> {
    pub game: Box<Account<'info, Game>>,

    #[account(mut,
        has_one = game,
        has_one = player,
    )]
    pub player_account: Box<Account<'info, Player>>,

    #[account(
        init, 
        seeds=[
            b"MONSTER", 
            game.key().as_ref(), 
            player.key().as_ref(),
            player_account.next_monster_index.to_le_bytes().as_ref()
        ], 
        bump, 
        payer = player, 
        space = std::mem::size_of::<Monster>() + 8)
    ]
    pub monster: Account<'info, Monster>,

    #[account(mut)]
    pub player: Signer<'info>,

    pub system_program: Program<'info, System>,
}

pub fn run_spawn_monster(ctx: Context<SpawnMonster>) -> Result<()> {

    {
        ctx.accounts.monster.player = ctx.accounts.player.key().clone();
        ctx.accounts.monster.game = ctx.accounts.game.key().clone();
        ctx.accounts.monster.hitpoints = 100;

        msg!("Monster Spawned!");
    }

    {
        ctx.accounts.player_account.next_monster_index = ctx.accounts.player_account.next_monster_index.checked_add(1).unwrap();
    }

    {   // Spend 5 lamports to spawn monster
        let action_point_to_spend = 5;

        spend_action_points(
            action_point_to_spend, 
            &mut ctx.accounts.player_account,
            &ctx.accounts.player.to_account_info(), 
            &ctx.accounts.system_program.to_account_info()
        )?;
    }

    Ok(())
}
```

## 8. 攻击怪物

现在！让我们攻击那些怪物，开始获得一些经验！

这里的逻辑如下：
- 玩家花费 1 个 `action_point` 进行攻击，并获得 1 点 `experience`
- 如果玩家杀死怪物，他们的 `kill` 计数会增加

至于设计决策，我们已经用 `Box` 包装了每个 rpg 账户，将它们分配到堆上。此外，当增加经验和击杀计数时，我们使用了 `saturating_add`。

`saturating_add` 函数确保数字永远不会溢出。假设 `kills` 是一个 u8，我的当前击杀数是 255（0xFF）。如果我再杀死一个并正常添加，例如 `255 + 1 = 0（0xFF + 0x01 = 0x00）= 0`，击杀数将变为 0。如果即将溢出，`saturating_add` 将保持在最大值，因此 `255 + 1 = 255`。如果即将溢出，`checked_add` 函数将抛出错误。在进行 Rust 中的数学运算时请记住这一点。即使 `kills` 是一个 u64，并且在当前编程中永远不会滚动，使用安全的数学运算并考虑滚动也是一个好习惯。

```rust
// ----------- ATTACK MONSTER ----------
#[derive(Accounts)]
pub struct AttackMonster<'info> {

    #[account(
        mut,
        has_one = player,
    )]
    pub player_account: Box<Account<'info, Player>>,

    #[account(
        mut,
        has_one = player,
        constraint = monster.game == player_account.game
    )]
    pub monster: Box<Account<'info, Monster>>,

    #[account(mut)]
    pub player: Signer<'info>,

    pub system_program: Program<'info, System>,
}

pub fn run_attack_monster(ctx: Context<AttackMonster>) -> Result<()> {

    let mut did_kill = false;

    {
        let hp_before_attack =  ctx.accounts.monster.hitpoints;
        let hp_after_attack = ctx.accounts.monster.hitpoints.saturating_sub(1);
        let damage_dealt = hp_before_attack - hp_after_attack;
        ctx.accounts.monster.hitpoints = hp_after_attack;

        

        if hp_before_attack > 0 && hp_after_attack == 0 {
            did_kill = true;
        }

        if  damage_dealt > 0 {
            msg!("Damage Dealt: {}", damage_dealt);
        } else {
            msg!("Stop it's already dead!");
        }
    }

    {
        ctx.accounts.player_account.experience = ctx.accounts.player_account.experience.saturating_add(1);
        msg!("+1 EXP");

        if did_kill {
            ctx.accounts.player_account.kills = ctx.accounts.player_account.kills.saturating_add(1);
            msg!("You killed the monster!");
        }
    }

    {   // Spend 1 lamports to attack monster
        let action_point_to_spend = 1;

        spend_action_points(
            action_point_to_spend, 
            &mut ctx.accounts.player_account,
            &ctx.accounts.player.to_account_info(), 
            &ctx.accounts.system_program.to_account_info()
        )?;
    }

    Ok(())
}
```

## 赎回到库藏

这是我们的最后一条指令。这个指令允许任何人将已花费的 `action_points` 发送到 `treasury` 钱包。

同样，让我们用 `Box` 包装 rpg 账户，并使用安全的数学运算。

```rust
// ----------- REDEEM TO TREASUREY ----------
#[derive(Accounts)]
pub struct CollectActionPoints<'info> {

    #[account(
        mut,
        has_one=treasury
    )]
    pub game: Box<Account<'info, Game>>,

    #[account(
        mut,
        has_one=game
    )]
    pub player: Box<Account<'info, Player>>,

    #[account(mut)]
    /// CHECK: It's being checked in the game account
    pub treasury: AccountInfo<'info>,

    pub system_program: Program<'info, System>,
}

// literally anyone who pays for the TX fee can run this command - give it to a clockwork bot
pub fn run_collect_action_points(ctx: Context<CollectActionPoints>) -> Result<()> {
    let transfer_amount: u64 = ctx.accounts.player.action_points_to_be_collected;

    **ctx.accounts.player.to_account_info().try_borrow_mut_lamports()? -= transfer_amount;
    **ctx.accounts.treasury.to_account_info().try_borrow_mut_lamports()? += transfer_amount;

    ctx.accounts.player.action_points_to_be_collected = 0;

    ctx.accounts.game.action_points_collected = ctx.accounts.game.action_points_collected.checked_add(transfer_amount).unwrap();

    msg!("The treasury collected {} action points to treasury", transfer_amount);

    Ok(())
}
```

## 将所有内容整合起来

现在我们已经编写了所有指令逻辑，让我们将这些函数添加到程序中的实际指令中。对于每个指令记录计算单位也可能会有所帮助。

```rust
#[program]
pub mod rpg {
    use super::*;

    pub fn create_game(ctx: Context<CreateGame>, max_items_per_player: u8) -> Result<()> {
        run_create_game(ctx, max_items_per_player)?;
        sol_log_compute_units();
        Ok(())
    }

    pub fn create_player(ctx: Context<CreatePlayer>) -> Result<()> {
        run_create_player(ctx)?;
        sol_log_compute_units();
        Ok(())
    }

    pub fn spawn_monster(ctx: Context<SpawnMonster>) -> Result<()> {
        run_spawn_monster(ctx)?;
        sol_log_compute_units();
        Ok(())
    }

    pub fn attack_monster(ctx: Context<AttackMonster>) -> Result<()> {
        run_attack_monster(ctx)?;
        sol_log_compute_units();
        Ok(())
    }

    pub fn deposit_action_points(ctx: Context<CollectActionPoints>) -> Result<()> {
        run_collect_action_points(ctx)?;
        sol_log_compute_units();
        Ok(())
    }

}
```

如果你正确添加了所有部分，你应该能够成功构建。

```shell
anchor build
```

## 测试

现在，让我们看看这个程序是如何工作的！

让我们设置 `tests/rpg.ts` 文件。我们将逐个填写每个测试。但首先，我们需要设置几个不同的账户。主要是 `gameMaster` 和 `treasury`。

```tsx
import * as anchor from "@coral-xyz/anchor";
import { Program } from "@coral-xyz/anchor";
import { Rpg, IDL } from "../target/types/rpg";
import { assert } from "chai";
import NodeWallet from "@coral-xyz/anchor/dist/cjs/nodewallet";

describe("RPG", () => {
  // Configure the client to use the local cluster.
  anchor.setProvider(anchor.AnchorProvider.env());

  const program = anchor.workspace.Rpg as Program<Rpg>;
  const wallet = anchor.workspace.Rpg.provider.wallet
    .payer as anchor.web3.Keypair;
  const gameMaster = wallet;
  const player = wallet;

  const treasury = anchor.web3.Keypair.generate();

it("Create Game", async () => {});

it("Create Player", async () => {});

it("Spawn Monster", async () => {});

it("Attack Monster", async () => {});

it("Deposit Action Points", async () => {});

});
```

现在让我们添加 `Create Game` 测试。只需调用 `createGame` 并传入八个元素，确保传入所有账户，并确保 `treasury` 账户签署交易。

```tsx
it("Create Game", async () => {
    const [gameKey] = anchor.web3.PublicKey.findProgramAddressSync(
      [Buffer.from("GAME"), treasury.publicKey.toBuffer()],
      program.programId
    );

    const txHash = await program.methods
      .createGame(
        8, // 8 Items per player
      )
      .accounts({
        game: gameKey,
        gameMaster: gameMaster.publicKey,
        treasury: treasury.publicKey,
        systemProgram: anchor.web3.SystemProgram.programId,
      })
      .signers([treasury])
      .rpc();

    await program.provider.connection.confirmTransaction(txHash);

    // Print out if you'd like
    // const account = await program.account.game.fetch(gameKey);

  });
```

请继续检查你的测试是否运行：

```tsx
yarn install
anchor test
```

**临时解决方案：** 如果由于某种原因，`yarn install` 命令生成了一些 `.pnp.*` 文件而没有 `node_modules` 文件夹，你可以尝试执行 `rm -rf .pnp.*`，然后执行 `npm i`，最后再运行 `yarn install`。这应该可以解决问题。

现在一切都运行正常，让我们实现 `Create Player`、`Spawn Monster` 和 `Attack Monster` 测试。在完成每个测试后运行测试以确保一切顺利进行。

```typescript
it("Create Player", async () => {
    const [gameKey] = anchor.web3.PublicKey.findProgramAddressSync(
      [Buffer.from("GAME"), treasury.publicKey.toBuffer()],
      program.programId
    );

    const [playerKey] = anchor.web3.PublicKey.findProgramAddressSync(
      [Buffer.from("PLAYER"), gameKey.toBuffer(), player.publicKey.toBuffer()],
      program.programId
    );

    const txHash = await program.methods
      .createPlayer()
      .accounts({
        game: gameKey,
        playerAccount: playerKey,
        player: player.publicKey,
        systemProgram: anchor.web3.SystemProgram.programId,
      })
      .rpc();

    await program.provider.connection.confirmTransaction(txHash);

    // Print out if you'd like
    // const account = await program.account.player.fetch(playerKey);

});

it("Spawn Monster", async () => {
    const [gameKey] = anchor.web3.PublicKey.findProgramAddressSync(
      [Buffer.from("GAME"), treasury.publicKey.toBuffer()],
      program.programId
    );

    const [playerKey] = anchor.web3.PublicKey.findProgramAddressSync(
      [Buffer.from("PLAYER"), gameKey.toBuffer(), player.publicKey.toBuffer()],
      program.programId
    );

    const playerAccount = await program.account.player.fetch(playerKey);

    const [monsterKey] = anchor.web3.PublicKey.findProgramAddressSync(
      [Buffer.from("MONSTER"), gameKey.toBuffer(), player.publicKey.toBuffer(), playerAccount.nextMonsterIndex.toBuffer('le', 8)],
      program.programId
    );

    const txHash = await program.methods
      .spawnMonster()
      .accounts({
        game: gameKey,
        playerAccount: playerKey,
        monster: monsterKey,
        player: player.publicKey,
        systemProgram: anchor.web3.SystemProgram.programId,
      })
      .rpc();

    await program.provider.connection.confirmTransaction(txHash);

    // Print out if you'd like
    // const account = await program.account.monster.fetch(monsterKey);

});

it("Attack Monster", async () => {
    const [gameKey] = anchor.web3.PublicKey.findProgramAddressSync(
      [Buffer.from("GAME"), treasury.publicKey.toBuffer()],
      program.programId
    );

    const [playerKey] = anchor.web3.PublicKey.findProgramAddressSync(
      [Buffer.from("PLAYER"), gameKey.toBuffer(), player.publicKey.toBuffer()],
      program.programId
    );
      
    // Fetch the latest monster created
    const playerAccount = await program.account.player.fetch(playerKey);
    const [monsterKey] = anchor.web3.PublicKey.findProgramAddressSync(
      [Buffer.from("MONSTER"), gameKey.toBuffer(), player.publicKey.toBuffer(), playerAccount.nextMonsterIndex.subn(1).toBuffer('le', 8)],
      program.programId
    );

    const txHash = await program.methods
      .attackMonster()
      .accounts({
        playerAccount: playerKey,
        monster: monsterKey,
        player: player.publicKey,
        systemProgram: anchor.web3.SystemProgram.programId,
      })
      .rpc();

    await program.provider.connection.confirmTransaction(txHash);

    // Print out if you'd like
    // const account = await program.account.monster.fetch(monsterKey);

    const monsterAccount = await program.account.monster.fetch(monsterKey);
    assert(monsterAccount.hitpoints.eqn(99));
});
```

请注意我们选择攻击的怪物是 `playerAccount.nextMonsterIndex.subn(1).toBuffer('le', 8)`。这样我们就可以攻击最近生成的怪物。`nextMonsterIndex` 以下的任何怪物都应该是可以攻击的。最后，由于种子只是字节的数组，我们必须将索引转换为 u64，即小端 `le` 8 字节。

运行 `anchor test` 来造成一些伤害！

最后，让我们编写一个测试来收集所有已存入的行动点数。这个测试可能对于它所做的事情来说感觉有些复杂。这是因为我们正在生成一些新的账户来展示任何人都可以调用 `depositActionPoints` 函数。我们使用诸如 `clockwork` 之类的名称，因为如果这个游戏持续运行，可能使用类似 [clockwork](https://www.clockwork.xyz/) 定期任务是有意义的。

```tsx
it("Deposit Action Points", async () => {
    const [gameKey] = anchor.web3.PublicKey.findProgramAddressSync(
      [Buffer.from("GAME"), treasury.publicKey.toBuffer()],
      program.programId
    );

    const [playerKey] = anchor.web3.PublicKey.findProgramAddressSync(
      [Buffer.from("PLAYER"), gameKey.toBuffer(), player.publicKey.toBuffer()],
      program.programId
    );
      
    // To show that anyone can deposit the action points
    // Ie, give this to a clockwork bot
    const clockworkWallet = anchor.web3.Keypair.generate();

    // To give it a starting balance
    const clockworkProvider = new anchor.AnchorProvider(
        program.provider.connection,
        new NodeWallet(clockworkWallet),
        anchor.AnchorProvider.defaultOptions(),
    )
    const clockworkProgram = new anchor.Program<Rpg>(
        IDL,
        program.programId,
        clockworkProvider,
    )

    // Have to give the accounts some lamports else the tx will fail
    const amountToInitialize = 10000000000;

    const clockworkAirdropTx = await clockworkProgram.provider.connection.requestAirdrop(clockworkWallet.publicKey, amountToInitialize);
    await program.provider.connection.confirmTransaction(clockworkAirdropTx, "confirmed");

    const treasuryAirdropTx = await clockworkProgram.provider.connection.requestAirdrop(treasury.publicKey, amountToInitialize);
    await program.provider.connection.confirmTransaction(treasuryAirdropTx, "confirmed");

    const txHash = await clockworkProgram.methods
      .depositActionPoints()
      .accounts({
        game: gameKey,
        player: playerKey,
        treasury: treasury.publicKey,
        systemProgram: anchor.web3.SystemProgram.programId,
      })
      .rpc();

    await program.provider.connection.confirmTransaction(txHash);

    const expectedActionPoints = 100 + 5 + 1; // Player Create ( 100 ) + Monster Spawn ( 5 ) + Monster Attack ( 1 )
    const treasuryBalance = await program.provider.connection.getBalance(treasury.publicKey);
    assert(
        treasuryBalance == 
        (amountToInitialize + expectedActionPoints) // Player Create ( 100 ) + Monster Spawn ( 5 ) + Monster Attack ( 1 )
    );

    const gameAccount = await program.account.game.fetch(gameKey);
    assert(gameAccount.actionPointsCollected.eqn(expectedActionPoints));

    const playerAccount = await program.account.player.fetch(playerKey);
    assert(playerAccount.actionPointsSpent.eqn(expectedActionPoints));
    assert(playerAccount.actionPointsToBeCollected.eqn(0));

});
```

最后，运行 `anchor test` 来查看一切是否正常运行。

恭喜你！这是的内容非常多，但现在你拥有了一个迷你的 RPG 游戏引擎。如果事情还没有完全工作，请回顾一下实验并找出你出错的地方。如果需要，你可以参考[解决方案代码的 `main` 分支](https://github.com/Unboxed-Software/anchor-rpg)。

确保在你自己的程序中将这些概念付诸实践。每一个小优化都是一笔宝贵的收获！

# 挑战

现在轮到你独立练习了。回顾实验代码，寻找可以进行的额外优化和/或扩展。思考你会添加哪些新系统和功能，以及如何对它们进行优化。

你可以在[RPG 代码库的 `challenge-solution` 分支](https://github.com/Unboxed-Software/anchor-rpg/tree/challenge-solution)上找到一些示例修改。

最后，检查你自己的程序之一，思考你可以进行哪些优化，以改进内存管理、存储大小和/或并发性能。

## 完成了实验吗？

将你的代码推送到 GitHub，并[告诉我们你对这节课的想法](https://form.typeform.com/to/IPH0UGz7#answers-lesson=4a628916-91f5-46a9-8eb0-6ba453aa6ca6)!