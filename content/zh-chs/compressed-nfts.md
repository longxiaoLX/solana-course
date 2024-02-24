---
title: 压缩 NFTs
objectives:
- 使用 Metaplex 的 Bubblegum 程序创建一个压缩的 NFT 集合
- 使用 Bubblegum TS SDK 铸造压缩的 NFTs
- 使用 Bubblegum TS SDK 转移压缩的 NFTs
- 使用 Read API 读取压缩的 NFT 数据
---

# 总结

- **压缩 NFTs（cNFTs）** 使用 **状态压缩** 对 NFT 数据进行哈希，并将哈希存储在链上的一个账户中，使用 **并发 Merkle 树** 结构。
- cNFT 数据的哈希不能用于推断 cNFT 数据，但可以用于 **验证** 所见的 cNFT 数据是否正确。
- 支持的 RPC 提供商在 cNFT 铸造时会 **索引** 链下 cNFT 数据，以便您可以使用 **Read API** 访问数据。
- **Metaplex Bubblegum 程序** 是 **状态压缩** 程序之上的一个抽象，使您更简单地创建、铸造和管理 cNFT 集合。

# 概述

压缩 NFTs（Compressed NFTs，cNFTs）正如其名称所示：NFT 的结构占用的账户存储空间（account storage）比传统 NFT 更少。压缩 NFTs 利用了一种称为 **状态压缩（State Compression）** 的概念，以极大地降低成本的方式存储数据。

Solana 的交易成本非常便宜，以至于大多数用户从未考虑过大规模铸造 NFTs 可以是多么昂贵。设置并铸造 100 万个传统 NFTs 的成本约为 24,000 SOL。相比之下，结构化的 cNFTs 可以使得相同的设置和铸造成本降低到 10 SOL 或更低。这意味着任何大规模使用 NFTs 的人都可以通过使用 cNFTs 而不是传统 NFTs 将成本削减超过 1000 倍。

然而，使用 cNFTs 可能有些棘手。最终，处理它们所需的工具将被从底层技术中充分抽象出来，以至于传统 NFTs 和 cNFTs 之间的开发人员体验将可以忽略不计。但是目前，您仍然需要理解底层拼图，所以让我们深入探讨！

## cNFTs 的理论概述

与传统 NFTs 相关的大部分成本归结为账户存储空间（account storage space）。压缩 NFTs 使用一种称为状态压缩的概念，将数据存储在区块链更便宜的 **账本状态（ledger state）** 中，只使用更昂贵的账户空间来存储数据的“指纹（fingerprint）”或 **哈希（hash）**。这个哈希允许您通过加密的方式验证数据是否被篡改。

为了存储哈希并启用验证，我们使用一种称为 **并发 Merkle 树** 的特殊二进制树结构。这个树结构让我们以确定性的方式将数据一起哈希，计算出一个单一的、最终的哈希值，并将其存储在链上。这个最终的哈希值的大小明显小于所有原始数据的总和，因此称之为“压缩”。这个过程的步骤如下：

1. 取任意一段数据。
2. 创建这些数据的哈希值。
3. 将这个哈希值作为一个“叶（leaf）子”存储在树的底部。
4. 然后将每一对叶子进行哈希，创建一个“分支（branch）”。
5. 然后将每个分支进行哈希。
6. 不断地向树上爬行，并将相邻的分支进行哈希。
7. 当到达树的顶部时，产生一个最终的“根哈希（root hash）”。
8. 将根哈希作为可验证的证明存储在链上，证明了每个叶子中的数据。
9. 任何想要验证他们拥有的数据是否与“真相源（source of truth）”匹配的人都可以通过相同的过程进行，并比较最终的哈希值，而无需将所有数据存储在链上。

上述内容未解决的一个问题是，如果无法从账户中获取数据，如何使数据可用。由于此哈希过程发生在链上，所有数据都存在于账本状态中，理论上可以通过重放整个链状态从原点检索原始交易中的所有数据。然而，更简单（尽管仍然复杂）的方法是让一个 **索引器（indexer）** 在交易发生时跟踪和索引这些数据。这确保了有一个链下的数据“缓存（cache）”，任何人都可以访问并随后与链上的根哈希进行验证。

这个过程非常复杂。我们将在下面介绍一些关键概念，但如果您一开始不理解也不用担心。我们将在[状态压缩课程](./generalized-state-compression.md)中讨论更多理论，并主要专注于在本课程中将其应用于 NFTs。即使您没有完全理解状态压缩拼图的每一部分，到本课程结束时，您也将能够处理 cNFTs。

### 并发 Merkle 树

**Merkle 树** 是一种由单个哈希表示的二进制树结构。结构中的每个叶子节点都是其内部数据的哈希，而每个分支都是其子叶子哈希的哈希。依次，分支也被一起哈希，直到最终只剩下一个根哈希。

对叶子数据的任何修改都会改变根哈希。当同一 slot 中的多个交易尝试修改叶子数据时会产生问题。由于这些交易必须按顺序执行，除了第一个交易外，所有交易都会失败，因为第一个执行的交易将使根哈希和传递的证明失效。

**并发 Merkle 树（concurrent merkle tree）** 是一种 Merkle 树，它存储了最近更改的安全变更日志，以及它们的根哈希和用于推导它的证明。当同一 slot 中的多个交易尝试修改叶子数据时，可以使用变更日志作为真相源，以允许对树进行并发更改。

在使用并发 Merkle 树时，有三个变量决定了树的大小、创建树的成本以及可以对树进行的并发更改的数量：

1. 最大深度（Max depth）
2. 最大缓冲区大小（Max buffer size）
3. 林冠深度（Canopy depth）

**最大深度** 是从任意叶子到树根的最大跳数（hops）。由于 Merkle 树是二叉树，每个叶子只连接到另一个叶子。最大深度可以逻辑上用于计算树的节点数，公式为 `2 ^ maxDepth`。

**最大缓冲区大小** 实际上是在单个 slot 内对树进行的最大并发更改次数，同时根哈希仍然有效。

**林冠深度** 是存储在链上的给定证明路径的证明节点数。验证任何叶子都需要完整的树的证明路径。完整的证明路径由树的每一“层”上的一个证明节点组成，即最大深度为 14 意味着有 14 个证明节点。每个证明节点都会在交易中增加 32 字节的大小，如果不在链上缓存证明节点，大型树很快就会超过最大交易大小限制。

这三个值，最大深度、最大缓冲区大小和林冠深度，都存在权衡。增加任何一个值都会增加用于存储树的账户的大小，从而增加创建树的成本。

选择最大深度相对比较直接，因为它直接关系到叶子节点的数量，因此也就关系到可以存储的数据量。如果你需要在单个树上存储100万个 cNFTs，则找到使以下表达式成立的最大深度：`2^maxDepth > 100万`。答案是 20。

选择最大缓冲区大小实际上是一个吞吐量问题：你需要多少并发写入。

### SPL 状态压缩和 Noop 程序

SPL 状态压缩程序（SPL State Compression Program）的存在是为了使上述过程在 Solana 生态系统中可以重复和组合。它提供了初始化 Merkle 树、管理树叶（例如添加、更新、删除数据）和验证叶子数据的指令。

状态压缩程序还利用了一个单独的“无操作”程序（“no op” program），其主要目的是通过将数据记录到账本状态中，使叶子数据更容易被索引。

### 使用账本状态进行存储

Solana 账本是一个包含已签名交易的条目列表。理论上，这可以追溯到创世区块。这实际上意味着任何曾经放入交易中的数据都存在于账本中。

当你想要存储压缩数据时，你将其传递给状态压缩程序，它会对其进行哈希处理，并作为一个“事件（event）”发出到无操作程序。然后，哈希值存储在相应的并发 Merkle 树中。由于数据经过了交易传递，甚至存在于无操作程序的日志中，因此它将永远存在于账本状态中。

### 为了方便查找而索引数据

在正常情况下，通常会通过获取适当的账户来访问链上数据。然而，当使用状态压缩时，情况就不那么简单了。

如上所述，数据现在存在于账本状态中，而不是一个账户中。找到完整数据的最容易的地方是在无操作指令的日志中，但是尽管这些数据在某种程度上将永远存在于账本状态中，但在一段时间后，它们可能会在验证节点中变得无法访问。

为了节省空间并提高性能，验证节点不会保留每个交易直到创世区块。你能够访问与你的数据相关的无操作指令日志的具体时间将根据验证节点而异，但是如果你直接依赖于指令日志，最终你将失去对其的访问权限。

从技术上讲，你*可以*将交易状态回溯到创世区块，但普通团队不会这样做，而且肯定不会是高效的。相反，你应该使用一个索引器，观察发送到无操作程序的事件，并将相关数据存储在链下。这样你就不必担心旧数据变得无法访问。

## 创建 cNFT 集合

现在我们已经了解了理论知识，让我们将注意力转向本课程的重点：如何创建 cNFT 集合。

幸运的是，你可以使用由 Solana 基金会、Solana 开发者社区和 Metaplex 创建的工具来简化这个过程。具体来说，我们将使用 `@solana/spl-account-compression` SDK、Metaplex Bubblegum 程序以及 Bubblegum 程序的相应 TS SDK `@metaplex-foundation/mpl-bugglegum`。

<aside>

💡 在撰写本文时，Metaplex 团队正在过渡到一个新的 Bubblegum 客户端 SDK，该 SDK 支持 umi，这是他们用于构建和使用 Solana 程序的模块化框架。在本课程中，我们将不使用 SDK 的 umi 版本。相反，我们将把我们的依赖硬编码到版本0.7（`@metaplex-foundation/mpl-bubblegum@0.7`）。该版本提供了用于构建 Bubblegum 指令的简单辅助函数。

</aside>

### 准备元数据

在开始之前，您将准备您的 NFT 元数据，类似于如果您使用糖果机时所做的方式。在本质上，NFT 只是一个带有符合 NFT 标准的元数据的代币。换句话说，它的结构应该类似于下面这样：

```json
{
  "name": "12_217_47",
  "symbol": "RGB",
  "description": "Random RGB Color",
  "seller_fee_basis_points": 0,
  "image": "https://raw.githubusercontent.com/ZYJLiu/rgb-png-generator/master/assets/12_217_47/12_217_47.png",
  "attributes": [
    {
      "trait_type": "R",
      "value": "12"
    },
    {
      "trait_type": "G",
      "value": "217"
    },
    {
      "trait_type": "B",
      "value": "47"
    }
  ]
}
```

根据您的用例，您可以动态生成这些数据，或者您可能希望提前为每个 cNFT 准备一个 JSON 文件。您还需要任何其他在 JSON 中引用的资产，例如上面示例中显示的 `image` URL。

### 创建集合 NFT

如果您希望您的 cNFTs 成为集合的一部分，在开始铸造 cNFTs 之前，您需要创建一个集合 NFT。这是一个传统的 NFT，作为将您的 cNFTs 绑定到单个集合中的参考。您可以使用 `@metaplex-foundation/js` 库创建此 NFT。只需确保将 `isCollection` 设置为 `true` 即可。

```tsx
const collectionNft = await metaplex.nfts().create({
    uri: someUri,
    name: "Collection NFT",
    sellerFeeBasisPoints: 0,
    updateAuthority: somePublicKey,
    mintAuthority: somePublicKey,
    tokenStandard: 0,
    symbol: "Collection",
    isMutable: true,
    isCollection: true,
})
```

### 创建 Merkle 树账户

现在我们开始偏离创建传统 NFTs 时所使用的流程。您用于状态压缩的链上存储机制是一个代表并发 Merkle 树的账户。这个 Merkle 树账户属于 SPL 状态压缩程序。在您执行与 cNFT 相关的任何操作之前，您需要创建一个空的 Merkle 树账户，其大小适当。

影响账户大小的变量包括：

1. 最大深度
2. 最大缓冲区大小
3. 林冠深度

第一个和第二个变量必须从现有的一组有效配对中选择。下表显示了有效配对以及这些值可以创建的 cNFT 的数量。

| 最大深度 | 最大缓冲区大小 | 最大 cNFT 数量 |
| --- | --- | --- |
| 3 | 8 | 8 |
| 5 | 8 | 32 |
| 14 | 64 | 16,384 |
| 14 | 256 | 16,384 |
| 14 | 1,024 | 16,384 |
| 14 | 2,048 | 16,384 |
| 15 | 64 | 32,768 |
| 16 | 64 | 65,536 |
| 17 | 64 | 131,072 |
| 18 | 64 | 262,144 |
| 19 | 64 | 524,288 |
| 20 | 64 | 1,048,576 |
| 20 | 256 | 1,048,576 |
| 20 | 1,024 | 1,048,576 |
| 20 | 2,048 | 1,048,576 |
| 24 | 64 | 16,777,216 |
| 24 | 256 | 16,777,216 |
| 24 | 512 | 16,777,216 |
| 24 | 1,024 | 16,777,216 |
| 24 | 2,048 | 16,777,216 |
| 26 | 512 | 67,108,864 |
| 26 | 1,024 | 67,108,864 |
| 26 | 2,048 | 67,108,864 |
| 30 | 512 | 1,073,741,824 |
| 30 | 1,024 | 1,073,741,824 |
| 30 | 2,048 | 1,073,741,824 |

请注意，可以存储在树上的 cNFT 的数量完全取决于最大深度，而缓冲区大小将确定在同一时间段内对树进行的并发更改（铸造、转移等）。换句话说，选择与您需要树保存的 NFT 数量相对应的最大深度，然后根据您预计需要支持的流量，选择缓冲区大小的其中选项之一。

接下来，选择树冠深度。增加树冠深度会增加您的 cNFT 的可组合性。在未来，每当您或其他开发人员的代码尝试验证 cNFT 时，代码将不得不传递与树中“层”一样多的证明节点。因此，对于最大深度为 20，您需要传递 20 个证明节点。这不仅很繁琐，而且由于每个证明节点都是 32 字节，很容易迅速达到交易大小的上限。

举个例子，如果你的树的树冠深度很低，一个 NFT 市场可能只能支持简单的 NFT 转移，而无法支持你的 cNFT 的链上竞标系统。树冠有效地将证明节点缓存在链上，这样你就不必将它们全部传递到交易中，从而允许更复杂的交易。

增加这三个值中的任何一个都会增加账户的大小，从而增加创建账户的相关成本。在选择这些值时，请根据利弊权衡。

一旦你知道了这些值，你可以使用 `@solana/spl-account-compression` TypeScript SDK 中的 `createAllocTreeIx` 辅助函数来创建空账户的指令。

```tsx
import { createAllocTreeIx } from "@solana/spl-account-compression"

const treeKeypair = Keypair.generate()

const allocTreeIx = await createAllocTreeIx(
  connection,
  treeKeypair.publicKey,
  payer.publicKey,
  { maxDepth: 20; maxBufferSize: 256 },
  canopyDepth
)
```

请注意，这仅仅是一个辅助函数，用于计算账户所需的大小并创建发送给系统程序以分配账户的指令。这个函数目前还没有与任何特定于压缩的程序交互。

### 使用 Bubblegum 初始化您的树

创建了空的树账户之后，接下来您可以使用 Bubblegum 程序来初始化树。除了 Merkle 树账户之外，Bubblegum 还会创建一个树配置账户，用于添加 cNFT 特定的跟踪和功能。

`@metaplex-foundation/mpl-bubblegum` TypeScript SDK 的 0.7 版本提供了一个辅助函数 `createCreateTreeInstruction`，用于调用 Bubblegum 程序上的 `create_tree` 指令。作为调用的一部分，您需要派生程序所期望的 `treeAuthority` PDA。这个 PDA 使用树的地址作为种子。

```tsx
import {
	createAllocTreeIx,
	SPL_ACCOUNT_COMPRESSION_PROGRAM_ID,
  SPL_NOOP_PROGRAM_ID,
} from "@solana/spl-account-compression"
import {
  PROGRAM_ID as BUBBLEGUM_PROGRAM_ID,
  createCreateTreeInstruction,
} from "@metaplex-foundation/mpl-bubblegum"

...

const [treeAuthority, _bump] = PublicKey.findProgramAddressSync(
  [treeKeypair.publicKey.toBuffer()],
  BUBBLEGUM_PROGRAM_ID
)

const createTreeIx = createCreateTreeInstruction(
  {
    treeAuthority,
    merkleTree: treeKeypair.publicKey,
    payer: payer.publicKey,
    treeCreator: payer.publicKey,
    logWrapper: SPL_NOOP_PROGRAM_ID,
    compressionProgram: SPL_ACCOUNT_COMPRESSION_PROGRAM_ID,
  },
  {
    maxBufferSize: 256,
    maxDepth: 20,
    public: false,
  },
  BUBBLEGUM_PROGRAM_ID
)
```

下面列出了这个辅助函数所需的输入：

- `accounts` - 一个表示指令所需账户的对象。包括：
    - `treeAuthority` - Bubblegum 期望这是一个使用 Merkle 树地址作为种子派生出的 PDA。
    - `merkleTree` - Merkle 树账户。
    - `payer` - 支付交易费用、租金等的地址。
    - `treeCreator` - 列为树创建者的地址。
    - `logWrapper` - 用于通过日志向索引器公开数据的程序；除非您有其他自定义实现，否则应该是 SPL Noop 程序的地址。
    - `compressionProgram` - 用于初始化 Merkle 树的压缩程序；除非您有其他自定义实现，否则应该是 SPL State Compression 程序的地址。
- `args` - 表示指令所需的额外参数的对象。包括：
    - `maxBufferSize` - Merkle 树的最大缓冲区大小。
    - `maxDepth` - Merkle 树的最大深度。
    - `public` - 当设置为 `true` 时，任何人都可以从树中铸造 cNFT；当设置为 `false` 时，只有树创建者或树委托者可以从树中铸造 cNFT。

提交后，这将调用 Bubblegum 程序上的 `create_tree` 指令。该指令执行三个操作：

1. 创建树配置 PDA 账户。
2. 使用适当的初始值初始化树配置账户。
3. 向 State Compression 程序发出 CPI，以初始化空的 Merkle 树账户。

欢迎查看 Bubblegum 程序代码[这里](https://github.com/metaplex-foundation/mpl-bubblegum/blob/main/programs/bubblegum/program/src/lib.rs#L887)。

### 铸造 cNFTs

有了初始化的 Merkle 树账户及其对应的 Bubblegum 树配置账户，就可以向树中铸造 cNFTs。要使用的 Bubblegum 指令将是 `mint_v1` 或 `mint_to_collection_v1`，具体取决于您是否希望铸造的 cNFT 成为集合的一部分。

`@metaplex-foundation/mpl-bubblegum` TypeScript SDK 的 0.7 版本提供了辅助函数 `createMintV1Instruction` 和 `createMintToCollectionV1Instruction`，以便您更轻松地创建指令。

这两个函数都需要您传递 NFT 元数据和用于铸造 cNFT 所需的账户列表。以下是向集合中铸造的示例：

```tsx
const mintWithCollectionIx = createMintToCollectionV1Instruction(
  {
    payer: payer.publicKey,
    merkleTree: treeAddress,
    treeAuthority,
    treeDelegate: payer.publicKey,
    leafOwner: destination,
    leafDelegate: destination,
    collectionAuthority: payer.publicKey,
    collectionAuthorityRecordPda: BUBBLEGUM_PROGRAM_ID,
    collectionMint: collectionDetails.mint,
    collectionMetadata: collectionDetails.metadata,
    editionAccount: collectionDetails.masterEditionAccount,
    compressionProgram: SPL_ACCOUNT_COMPRESSION_PROGRAM_ID,
    logWrapper: SPL_NOOP_PROGRAM_ID,
    bubblegumSigner,
    tokenMetadataProgram: TOKEN_METADATA_PROGRAM_ID,
  },
  {
    metadataArgs: Object.assign(nftMetadata, {
      collection: { key: collectionDetails.mint, verified: false },
    }),
  }
)
```

请注意，辅助函数有两个参数：`accounts` 和 `args`。`args` 参数只是 NFT 元数据，而 `accounts` 则是列出指令所需的账户的对象。诚然，其中有很多：

- `payer` - 支付交易费用、租金等的账户。
- `merkleTree` - Merkle 树账户。
- `treeAuthority` - 树的授权账户；应该与之前派生的 PDA 相同。
- `treeDelegate` - 树的委托账户；通常与树创建者相同。
- `leafOwner` - 正在铸造的压缩 NFT 的期望所有者。
- `leafDelegate` - 正在铸造的压缩 NFT 的期望委托者；通常与叶所有者相同。
- `collectionAuthority` - 集合 NFT 的授权账户。
- `collectionAuthorityRecordPda` - 可选的集合授权记录 PDA；通常没有，此时应将 Bubblegum 程序地址放在这里。
- `collectionMint` - 集合 NFT 的铸币账户。
- `collectionMetadata` - 集合 NFT 的元数据账户。
- `editionAccount` - 集合 NFT 的主版本账户。
- `compressionProgram` - 要使用的压缩程序；除非有其他自定义实现，否则应该是 SPL State Compression 程序的地址。
- `logWrapper` - 用于通过日志向索引器公开数据的程序；除非有其他自定义实现，否则应该是 SPL Noop 程序的地址。
- `bubblegumSigner` - Bubblegrum 程序用于处理集合验证的 PDA。
- `tokenMetadataProgram` - 用于集合 NFT 的代币元数据程序；通常始终是 Metaplex 令牌元数据程序。

如果不使用集合进行铸造，则所需的账户较少，其中没有一个是专门用于不使用集合进行铸造的。您可以查看下面的示例。

```tsx
const mintWithoutCollectionIx = createMintV1Instruction(
  {
    payer: payer.publicKey,
    merkleTree: treeAddress,
    treeAuthority,
    treeDelegate: payer.publicKey,
    leafOwner: destination,
    leafDelegate: destination,
    compressionProgram: SPL_ACCOUNT_COMPRESSION_PROGRAM_ID,
    logWrapper: SPL_NOOP_PROGRAM_ID,
  },
  {
    message: nftMetadata,
  }
)
```

## 与 cNFTs 交互

需要注意的是，cNFTs *不是* SPL 代币。这意味着您的代码需要遵循不同的约定，以处理 cNFT 的功能，如获取、查询、转移等。

### 获取 cNFT 数据

从现有的 cNFT 获取数据的最简单方法是使用 [Digital Asset Standard Read API](https://docs.solana.com/developing/guides/compressed-nfts#reading-compressed-nfts-metadata)（读取 API）。请注意，这与标准的 JSON RPC 是分开的。要使用读取 API，您需要使用支持的 RPC 提供商。Metaplex 维护了一个（可能不完整的）[RPC 提供商列表](https://developers.metaplex.com/bubblegum/rpcs)，支持读取 API。在本课程中，我们将使用 [Helius](https://docs.helius.dev/compression-and-das-api/digital-asset-standard-das-api)，因为他们免费支持 Devnet。

要使用读取 API 获取特定的 cNFT，您需要拥有该 cNFT 的资产 ID。然而，在铸造 cNFTs 后，您最多只会有两个信息：

1. 交易签名
2. 叶子索引（可能）

唯一的真正保证是您将拥有交易签名。**可能**从中定位叶子索引，但这涉及到一些相当复杂的解析。简而言之，您必须从 Noop 程序中检索相关的指令日志，并解析它们以找到叶子索引。我们将在以后的课程中更深入地讨论这个问题。现在，我们假设您知道叶子索引。

对于大多数铸造来说，这是一个合理的假设，因为铸造将由您的代码控制，并且可以按顺序设置，以便您的代码可以跟踪每个铸造将使用的索引。例如，第一个铸造将使用索引 0，第二个索引 1，依此类推。

一旦您有了叶子索引，就可以派生出相应的 cNFT 资产 ID。在使用 Bubblegum 时，资产 ID 是使用 Bubblegum 程序 ID 和以下种子派生出来的：

1. 静态字符串 `asset`，以 utf8 编码表示
2. Merkle 树地址
3. 叶子索引

索引器基本上观察 Noop 程序的交易日志，当 Noop 程序在 Merkle 树发生存储经过哈希的 cNFT 元数据。这使他们能够在响应请求时提供该数据。这个资产 ID 是索引器用来标识特定资产的。

为了简单起见，您可以直接使用 Bubblegum SDK 中的 `getLeafAssetId` 辅助函数。有了资产 ID，获取 cNFT 就相当简单了。只需使用支持的 RPC 提供程序提供的 `getAsset` 方法：

```tsx
const assetId = await getLeafAssetId(treeAddress, new BN(leafIndex))
const response = await fetch(process.env.RPC_URL, {
	method: "POST",
	headers: { "Content-Type": "application/json" },
	body: JSON.stringify({
		jsonrpc: "2.0",
		id: "my-id",
		method: "getAsset",
		params: {
			id: assetId,
		},
	}),
})

const { result } = await response.json()
console.log(JSON.stringify(result, null, 2))
```

这将返回一个 JSON 对象，其中包含传统 NFT 的链上和链下元数据的综合信息。例如，您可以在 `content.metadata.attributes` 找到 cNFT 的属性，或者在 `content.files.uri` 找到图像。

### 查询 cNFTs

读取 API 还包括获取多个资产、按所有者、创建者等查询的方法。例如，Helius 支持以下方法：

- `getAsset`
- `getSignaturesForAsset`
- `searchAssets`
- `getAssetProof`
- `getAssetsByOwner`
- `getAssetsByAuthority`
- `getAssetsByCreator`
- `getAssetsByGroup`

我们不会直接讨论其中的大部分内容，但一定要仔细阅读 [Helius 文档](https://docs.helius.dev/compression-and-das-api/digital-asset-standard-das-api)，以了解如何正确使用它们。

### 转移 cNFTs

与标准的 SPL 代币转移一样，安全性是至关重要的。然而，SPL 代币转移使验证转移授权变得非常容易。它已内置于 SPL 代币程序和标准签名中。压缩代币的所有权验证更加困难。实际的验证将在程序端进行，但您的客户端代码需要提供额外的信息以使其成为可能。

虽然 Bubblegum 提供了 `createTransferInstruction` 辅助函数，但与通常情况下相比，需要进行更多的组装。具体来说，Bubblegum 程序需要验证客户端断言的整个 cNFT 数据是否正确，然后才能进行转移。整个 cNFT 数据已经被哈希并存储为 Merkle 树上的单个叶子，而 Merkle 树只是所有叶子和分支的哈希。因此，您不能简单地告诉程序要查看哪个账户，并将该账户的 `authority` 或 `owner` 字段与交易签名者进行比较。

相反，您需要提供整个 cNFT 数据以及在树冠中未存储的任何 Merkle 树的证明信息。这样，程序可以独立证明所提供的 cNFT 数据，因此 cNFT 的所有者是准确的。只有在这种情况下，程序才能安全地确定是否应该允许交易签名者转移 cNFT。

在广义上，这涉及到一个五个步骤的过程：

1. 从索引器获取 cNFT 的资产数据
2. 从索引器获取 cNFT 的证明
3. 从 Solana 区块链获取 Merkle 树账户
4. 准备资产证明作为 `AccountMeta` 对象的列表
5. 构建并发送 Bubblegum 转移指令

前两个步骤非常相似。使用您的支持 RPC 提供商，分别使用 `getAsset` 和 `getAssetProof` 方法来获取资产数据和证明。

```tsx
const assetDataResponse = await fetch(process.env.RPC_URL, {
	method: "POST",
	headers: { "Content-Type": "application/json" },
	body: JSON.stringify({
		jsonrpc: "2.0",
		id: "my-id",
		method: "getAsset",
			params: {
				id: assetId,
			},
		}),
	})
const assetData = (await assetDataResponse.json()).result

const assetProofResponse = await fetch(process.env.RPC_URL, {
	method: "POST",
	headers: { "Content-Type": "application/json" },
	body: JSON.stringify({
		jsonrpc: "2.0",
		id: "my-id",
		method: "getAssetProof",
			params: {
				id: assetId,
			},
		}),
	})
const assetProof = (await assetProofResponse.json()).result
```

第三步是获取 Merkle 树账户。最简单的方法是使用 `@solana/spl-account-compression` 中的 `ConcurrentMerkleTreeAccount` 类型：

```tsx
const treePublicKey = new PublicKey(assetData.compression.tree)

const treeAccount = await ConcurrentMerkleTreeAccount.fromAccountAddress(
	connection,
	treePublicKey
)
```

第四步是最具概念性挑战的步骤。利用收集到的三个信息，您需要组装 cNFT 对应叶子的证明路径。证明路径被表示为传递给程序指令的账户。程序使用每个账户地址作为证明节点，以证明叶子数据与您所说的一致。

索引器提供了完整的证明，就像上面的 `assetProof` 中所示。然而，您可以从证明中排除与树冠深度相同数量的尾部账户。

```tsx
const canopyDepth = treeAccount.getCanopyDepth() || 0

const proofPath: AccountMeta[] = assetProof.proof
	.map((node: string) => ({
	pubkey: new PublicKey(node),
	isSigner: false,
	isWritable: false
}))
.slice(0, assetProof.proof.length - canopyDepth)
```

最后，您可以组装转移指令。指令辅助函数 `createTransferInstruction` 需要以下参数：

以下是指令辅助函数 `createTransferInstruction` 需要的参数：

- `accounts` - 预期的指令账户列表；它们如下所示：
    - `merkleTree` - Merkle 树账户
    - `treeAuthority` - Merkle 树的授权账户
    - `leafOwner` - 问题叶子（cNFT）的所有者
    - `leafDelegate` - 问题叶子（cNFT）的委托者；如果没有添加委托者，则应与 `leafOwner` 相同
    - `newLeafOwner` - 转移后的新所有者的地址
    - `logWrapper` - 用于通过日志向索引器公开数据的程序；除非有其他自定义实现，否则应该是 SPL Noop 程序的地址
    - `compressionProgram` - 要使用的压缩程序；除非有其他自定义实现，否则应该是 SPL State Compression 程序的地址
    - `anchorRemainingAccounts` - 这是您添加证明路径的地方
- `args` - 指令所需的额外参数；它们是：
    - `root` - 从资产证明中提取的 Merkle 树根节点；这是由索引器提供的字符串，必须先转换为字节数组
    - `dataHash` - 从索引器检索的资产数据的哈希；这是由索引器提供的字符串，必须先转换为字节数组
    - `creatorHash` - 从索引器检索的 cNFT 创建者的哈希；这是由索引器提供的字符串，必须先转换为字节数组
    - `nonce` - 用于确保没有两个叶子具有相同的哈希；此值应与 `index` 相同
    - `index` - cNFT 的叶子位于 Merkle 树上的索引位置

以下是示例。请注意，代码的前三行获取了先前嵌套在对象中的附加信息，因此当组装指令本身时，它们已准备就绪。

```tsx
const treeAuthority = treeAccount.getAuthority()
const leafOwner = new PublicKey(assetData.ownership.owner)
const leafDelegate = assetData.ownership.delegate
	? new PublicKey(assetData.ownership.delegate)
	: leafOwner

const transferIx = createTransferInstruction(
	{
		merkleTree: treePublicKey,
		treeAuthority,
		leafOwner,
		leafDelegate,
		newLeafOwner: receiver,
		logWrapper: SPL_NOOP_PROGRAM_ID,
		compressionProgram: SPL_ACCOUNT_COMPRESSION_PROGRAM_ID,
		anchorRemainingAccounts: proofPath,
	},
	{
		root: [...new PublicKey(assetProof.root.trim()).toBytes()],
		dataHash: [...new PublicKey(assetData.compression.data_hash.trim()).toBytes()],
		creatorHash: [
			...new PublicKey(assetData.compression.creator_hash.trim()).toBytes(),
		],
		nonce: assetData.compression.leaf_id,
		index: assetData.compression.leaf_id,
	}
)
```

## 结论

我们已经介绍了与 cNFTs 交互所需的主要技能，但并不全面。您还可以使用 Bubblegum 进行烧毁、验证、委托等操作。我们不会详细介绍这些内容，但这些指令与铸造和转移过程类似。如果您需要这些额外的功能，请查看 [Bubblegum 客户端源代码](https://github.com/metaplex-foundation/mpl-bubblegum/tree/main/clients/js-solita) 并利用它提供的辅助函数。

请记住，压缩技术相当新颖。可用的工具将迅速发展，但您在本课程中学到的原则可能会保持不变。这些原则也可以扩展到任意状态的压缩，因此请确保在这里掌握它们，以便为未来课程中的更有趣的内容做好准备！

# 实验

让我们开始练习创建和操作 cNFTs。我们将一起构建尽可能简单的脚本，以便我们可以从 Merkle 树铸造一个 cNFT 集合。

## 1. 获取起始代码

首先，从我们的[cNFT实验室存储库](https://github.com/Unboxed-Software/solana-cnft-demo)的`starter`分支中克隆起始代码。

`git clone https://github.com/Unboxed-Software/solana-cnft-demo.git`

`cd solana-cnft-demo`

`npm install`

花一些时间熟悉提供的起始代码。最重要的是 `utils.ts` 中提供的辅助函数和 `uri.ts` 中提供的 URI。

`uri.ts` 文件提供了一万个用于您的 NFT 元数据的链下部分的 URI。当然，您也可以创建自己的元数据。但是，本课程并不明确涉及准备元数据，因此我们为您提供了一些。

`utils.ts` 文件中有一些辅助函数，可以帮助您减少不必要的样板代码。它们如下：

- `getOrCreateKeypair`：将为您创建一个新的密钥对，并将其保存到 `.env` 文件中，或者如果 `.env` 文件中已经有私钥，则将从该私钥初始化一个密钥对。
- `airdropSolIfNeeded`：如果指定地址的余额低于 1 SOL，则将一些 Devnet SOL 空投到该地址。
- `createNftMetadata`：将为给定的创建者公钥和索引创建 NFT 元数据。它获取的元数据只是使用 `uri.ts` 中提供的 URI 列表中相应索引对应的 URI 创建的虚拟元数据。
- `getOrCreateCollectionNFT`：将从 `.env` 中指定的地址获取集合 NFT，如果没有，则创建一个新的集合 NFT，并将地址添加到 `.env`。

最后，在 `index.ts` 中有一些样板代码，它创建一个新的 Devnet 连接，调用 `getOrCreateKeypair` 来初始化一个“钱包”，并调用 `airdropSolIfNeeded` 来为钱包提供资金，如果其余额较低的话。

我们将在 `index.ts` 中编写我们的所有代码。

## 2. 创建 Merkle 树账户

我们将开始创建 Merkle 树账户。让我们将这个过程封装在一个函数中，该函数最终将创建 *并且* 初始化账户。我们将把它放在 `index.ts` 文件中 `main` 函数的下面。我们将其命名为 `createAndInitializeTree`。为了使该函数正常工作，它将需要以下参数：

- `connection` - 用于与网络交互的 `Connection`。
- `payer` - 将支付交易费用的 `Keypair`。
- `maxDepthSizePair` - 一个 `ValidDepthSizePair`。此类型来自 `@solana/spl-account-compression`。它是一个简单的对象，具有 `maxDepth` 和 `maxBufferSize` 两个属性，强制执行这两个值的有效组合。
- `canopyDepth` - 用于树冠深度的数字。

在函数体内，我们将为树生成一个新地址，然后通过调用 `@solana/spl-account-compression` 中的 `createAllocTreeIx` 创建一个新的 Merkle 树账户的指令。
    

```tsx
async function createAndInitializeTree(
  connection: Connection,
  payer: Keypair,
  maxDepthSizePair: ValidDepthSizePair,
  canopyDepth: number
) {
	const treeKeypair = Keypair.generate()

	const allocTreeIx = await createAllocTreeIx(
    connection,
    treeKeypair.publicKey,
    payer.publicKey,
    maxDepthSizePair,
    canopyDepth
  )
}
```

## 3. 使用 Bubblegum 初始化 Merkle 树并创建树配置账户

有了创建树的指令准备好了，我们可以创建一个指令来调用 Bubblegum 程序中的 `create_tree`。这将初始化 Merkle 树账户 *并且* 在 Bubblegum 程序上创建一个新的树配置账户。

这个指令需要我们提供以下内容：

- `accounts` - 一个包含所需账户的对象；这包括：
    - `treeAuthority` - 这应该是一个由 Merkle 树地址和 Bubblegum 程序派生的 PDA。
    - `merkleTree` - Merkle 树的地址。
    - `payer` - 交易费支付者。
    - `treeCreator` - 树创建者的地址；我们将其设置为与 `payer` 相同。
    - `logWrapper` - 将其设置为 `SPL_NOOP_PROGRAM_ID`。
    - `compressionProgram` - 将其设置为 `SPL_ACCOUNT_COMPRESSION_PROGRAM_ID`。
- `args` - 一个指令参数列表；这包括：
    - `maxBufferSize` - 来自我们函数的 `maxDepthSizePair` 参数的缓冲区大小。
    - `maxDepth` - 来自我们函数的 `maxDepthSizePair` 参数的最大深度。
    - `public` - 树是否应该是公开的；我们将其设置为 `false`。

最后，我们可以将这两个指令添加到一个交易中并提交交易。请记住，交易需要由 `payer` 和 `treeKeypair` 签名。

```tsx
async function createAndInitializeTree(
  connection: Connection,
  payer: Keypair,
  maxDepthSizePair: ValidDepthSizePair,
  canopyDepth: number
) {
	const treeKeypair = Keypair.generate()

	const allocTreeIx = await createAllocTreeIx(
    connection,
    treeKeypair.publicKey,
    payer.publicKey,
    maxDepthSizePair,
    canopyDepth
  )

	const [treeAuthority, _bump] = PublicKey.findProgramAddressSync(
    [treeKeypair.publicKey.toBuffer()],
    BUBBLEGUM_PROGRAM_ID
  )

	const createTreeIx = createCreateTreeInstruction(
    {
      treeAuthority,
      merkleTree: treeKeypair.publicKey,
      payer: payer.publicKey,
      treeCreator: payer.publicKey,
      logWrapper: SPL_NOOP_PROGRAM_ID,
      compressionProgram: SPL_ACCOUNT_COMPRESSION_PROGRAM_ID,
    },
    {
      maxBufferSize: maxDepthSizePair.maxBufferSize,
      maxDepth: maxDepthSizePair.maxDepth,
      public: false,
    }
  )

	const tx = new Transaction().add(allocTreeIx, createTreeIx)
  tx.feePayer = payer.publicKey
  
  try {
    const txSignature = await sendAndConfirmTransaction(
      connection,
      tx,
      [treeKeypair, payer],
      {
        commitment: "confirmed",
        skipPreflight: true,
      }
    )

    console.log(`https://explorer.solana.com/tx/${txSignature}?cluster=devnet`)

    console.log("Tree Address:", treeKeypair.publicKey.toBase58())

    return treeKeypair.publicKey
  } catch (err: any) {
    console.error("\nFailed to create merkle tree:", err)
    throw err
  }
}
```

如果你想测试到目前为止的内容，可以随时从 `main` 调用 `createAndInitializeTree`，并为最大深度和最大缓冲区大小提供小的值。

```tsx
async function main() {
  const connection = new Connection(clusterApiUrl("devnet"), "confirmed")
  const wallet = await getOrCreateKeypair("Wallet_1")
  await airdropSolIfNeeded(wallet.publicKey)

  const maxDepthSizePair: ValidDepthSizePair = {
    maxDepth: 3,
    maxBufferSize: 8,
  }

  const canopyDepth = 0

  const treeAddress = await createAndInitializeTree(
    connection,
    wallet,
    maxDepthSizePair,
    canopyDepth
  )
}
```

请记住，Devnet SOL 受到限制，如果测试次数过多，可能会在我们进行铸币之前用完 Devnet SOL。要进行测试，请在您的终端中运行以下命令：

`npm run start`

## 4. 将 cNFT 铸造到您的树中

信不信由你，这就是你设置树来压缩 NFT 所需做的一切！现在让我们把注意力转向铸造。

首先，让我们声明一个名为 `mintCompressedNftToCollection` 的函数。它将需要以下参数：

- `connection` - 用于与网络交互的 `Connection`。
- `payer` - 将支付交易费用的 `Keypair`。
- `treeAddress` - Merkle 树的地址。
- `collectionDetails` - 类型为 `utils.ts` 中的 `CollectionDetails` 的集合详情。
- `amount` - 要铸造的 cNFT 数量。

这个函数的主体将执行以下操作：

1. 像之前一样派生树权限。同样，这是一个由 Merkle 树地址和 Bubblegum 程序派生的 PDA。
2. 派生 `bubblegumSigner`。这是一个由字符串 `"collection_cpi"` 和 Bubblegum 程序派生的 PDA，并且对于向集合铸造是必不可少的。
3. 通过调用我们的 `utils.ts` 文件中的 `createNftMetadata` 创建 cNFT 元数据。
4. 通过调用 Bubblegum SDK 中的 `createMintToCollectionV1Instruction` 创建铸造指令。
5. 构建并发送一个包含铸造指令的交易。
6. 重复步骤 3-6 `amount` 次。

`createMintToCollectionV1Instruction` 接受两个参数：`accounts` 和 `args`。后者简单地是 NFT 元数据。与所有复杂指令一样，主要难点在于知道要提供哪些账户。因此，让我们快速浏览一下它们：

- `payer` - 支付交易费、租金等的账户。
- `merkleTree` - Merkle 树账户。
- `treeAuthority` - 树授权方；应该与之前派生的 PDA 相同。
- `treeDelegate` - 树委托方；通常与树创建者相同。
- `leafOwner` - 正在铸造的压缩 NFT 的期望所有者。
- `leafDelegate` - 正在铸造的压缩 NFT 的期望委托者；通常与叶子所有者相同。
- `collectionAuthority` - 集合 NFT 的授权方。
- `collectionAuthorityRecordPda` - 可选的集合授权记录 PDA；通常不存在，如果不存在，则应将 Bubblegum 程序地址放在此处。
- `collectionMint` - 集合 NFT 的铸造账户。
- `collectionMetadata` - 集合 NFT 的元数据账户。
- `editionAccount` - 集合 NFT 的主版本账户。
- `compressionProgram` - 要使用的压缩程序；除非您有其他自定义实现，否则应该是 SPL 状态压缩程序的地址。
- `logWrapper` - 用于通过日志向索引器公开数据的程序；除非您有其他自定义实现，否则应该是 SPL Noop 程序的地址。
- `bubblegumSigner` - Bubblegum 程序用于处理集合验证的 PDA。
- `tokenMetadataProgram` - 用于集合 NFT 的代币元数据程序；通常始终是 Metaplex 代币元数据程序。

将所有内容整合在一起，如下所示：

```tsx
async function mintCompressedNftToCollection(
  connection: Connection,
  payer: Keypair,
  treeAddress: PublicKey,
  collectionDetails: CollectionDetails,
  amount: number
) {
  // Derive the tree authority PDA ('TreeConfig' account for the tree account)
  const [treeAuthority] = PublicKey.findProgramAddressSync(
    [treeAddress.toBuffer()],
    BUBBLEGUM_PROGRAM_ID
  )

  // Derive the bubblegum signer, used by the Bubblegum program to handle "collection verification"
  // Only used for `createMintToCollectionV1` instruction
  const [bubblegumSigner] = PublicKey.findProgramAddressSync(
    [Buffer.from("collection_cpi", "utf8")],
    BUBBLEGUM_PROGRAM_ID
  )

  for (let i = 0; i < amount; i++) {
    // Compressed NFT Metadata
    const compressedNFTMetadata = createNftMetadata(payer.publicKey, i)

    // Create the instruction to "mint" the compressed NFT to the tree
    const mintIx = createMintToCollectionV1Instruction(
      {
        payer: payer.publicKey, // The account that will pay for the transaction
        merkleTree: treeAddress, // The address of the tree account
        treeAuthority, // The authority of the tree account, should be a PDA derived from the tree account address
        treeDelegate: payer.publicKey, // The delegate of the tree account, should be the same as the tree creator by default
        leafOwner: payer.publicKey, // The owner of the compressed NFT being minted to the tree
        leafDelegate: payer.publicKey, // The delegate of the compressed NFT being minted to the tree
        collectionAuthority: payer.publicKey, // The authority of the "collection" NFT
        collectionAuthorityRecordPda: BUBBLEGUM_PROGRAM_ID, // Must be the Bubblegum program id
        collectionMint: collectionDetails.mint, // The mint of the "collection" NFT
        collectionMetadata: collectionDetails.metadata, // The metadata of the "collection" NFT
        editionAccount: collectionDetails.masterEditionAccount, // The master edition of the "collection" NFT
        compressionProgram: SPL_ACCOUNT_COMPRESSION_PROGRAM_ID,
        logWrapper: SPL_NOOP_PROGRAM_ID,
        bubblegumSigner,
        tokenMetadataProgram: TOKEN_METADATA_PROGRAM_ID,
      },
      {
        metadataArgs: Object.assign(compressedNFTMetadata, {
          collection: { key: collectionDetails.mint, verified: false },
        }),
      }
    )

    try {
      // Create new transaction and add the instruction
      const tx = new Transaction().add(mintIx)

      // Set the fee payer for the transaction
      tx.feePayer = payer.publicKey

      // Send the transaction
      const txSignature = await sendAndConfirmTransaction(
        connection,
        tx,
        [payer],
        { commitment: "confirmed", skipPreflight: true }
      )

      console.log(
        `https://explorer.solana.com/tx/${txSignature}?cluster=devnet`
      )
    } catch (err) {
      console.error("\nFailed to mint compressed NFT:", err)
      throw err
    }
  }
}
```

这是一个很好的时机来测试一个小树。只需更新 `main` 函数来调用 `getOrCreateCollectionNFT` 然后调用 `mintCompressedNftToCollection`：

```tsx
async function main() {
  const connection = new Connection(clusterApiUrl("devnet"), "confirmed")
  const wallet = await getOrCreateKeypair("Wallet_1")
  await airdropSolIfNeeded(wallet.publicKey)

  const maxDepthSizePair: ValidDepthSizePair = {
    maxDepth: 3,
    maxBufferSize: 8,
  }

  const canopyDepth = 0

  const treeAddress = await createAndInitializeTree(
    connection,
    wallet,
    maxDepthSizePair,
    canopyDepth
  )

  const collectionNft = await getOrCreateCollectionNFT(connection, wallet)

  await mintCompressedNftToCollection(
    connection,
    wallet,
    treeAddress,
    collectionNft,
    2 ** maxDepthSizePair.maxDepth
  )
}
```

再次运行，请在您的终端中键入：`npm run start`

## 5. 读取现有的 cNFT 数据

既然我们已经编写了用于铸造 cNFT 的代码，让我们看看是否可以实际获取它们的数据。这有点棘手，因为链上数据只是 merkle 树账户，其中的数据可用于验证现有信息的准确性，但在传达信息是无用的。

让我们从声明一个名为 `logNftDetails` 的函数开始，该函数接受 `treeAddress` 和 `nftsMinted` 作为参数。

在这一点上，我们实际上没有任何直接的标识符指向我们的 cNFT。为了获取它，我们需要知道在铸造 cNFT 时使用的叶索引。然后，我们可以使用这个索引来导出由 Read API 使用的资产 ID，随后使用 Read API 获取我们的 cNFT 数据。

在我们的情况下，我们创建了一个非公开的树并铸造了 8 个 cNFT，因此我们知道使用的叶索引是 0-7。有了这个，我们可以使用 `@metaplex-foundation/mpl-bubblegum` 中的 `getLeafAssetId` 函数来获取资产 ID。

最后，我们可以使用支持 [Read API](https://docs.solana.com/developing/guides/compressed-nfts#reading-compressed-nfts-metadata) 的 RPC 来获取资产。我们将使用 [Helius](https://docs.helius.dev/compression-and-das-api/digital-asset-standard-das-api)，但请随意选择您自己的 RPC 提供程序。要使用 Helius，您需要从 [他们的网站](https://dev.helius.xyz/) 获取免费的 API 密钥。然后将您的 `RPC_URL` 添加到您的 `.env` 文件中。例如：

```bash
# Add this
RPC_URL=https://devnet.helius-rpc.com/?api-key=YOUR_API_KEY
```

然后，只需向您提供的 RPC URL 发出一个 POST 请求，并将 `getAsset` 信息放入请求体中：

```tsx
async function logNftDetails(treeAddress: PublicKey, nftsMinted: number) {
  for (let i = 0; i < nftsMinted; i++) {
    const assetId = await getLeafAssetId(treeAddress, new BN(i))
    console.log("Asset ID:", assetId.toBase58())
    const response = await fetch(process.env.RPC_URL, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        jsonrpc: "2.0",
        id: "my-id",
        method: "getAsset",
        params: {
          id: assetId,
        },
      }),
    })
    const { result } = await response.json()
    console.log(JSON.stringify(result, null, 2))
  }
}
```

Helius 基本上会在交易日志发生时观察并存储 NFT 元数据，这些元数据已经被哈希并存储在默克尔树中。这使得他们在被请求时能够提供这些数据。

如果我们在 `main` 的结尾添加一个调用这个函数的操作，并重新运行您的脚本，那么在控制台中我们得到的数据会非常全面。它包括了传统 NFT 的链上和链下部分中您所期望的所有数据。您可以找到 cNFT 的属性、文件、所有权和创建者信息等等。

```json
{
  "interface": "V1_NFT",
  "id": "48Bw561h1fGFK4JGPXnmksHp2fpniEL7hefEc6uLZPWN",
  "content": {
    "$schema": "https://schema.metaplex.com/nft1.0.json",
    "json_uri": "https://raw.githubusercontent.com/Unboxed-Software/rgb-png-generator/master/assets/183_89_78/183_89_78.json",
    "files": [
      {
        "uri": "https://raw.githubusercontent.com/Unboxed-Software/rgb-png-generator/master/assets/183_89_78/183_89_78.png",
        "cdn_uri": "https://cdn.helius-rpc.com/cdn-cgi/image//https://raw.githubusercontent.com/Unboxed-Software/rgb-png-generator/master/assets/183_89_78/183_89_78.png",
        "mime": "image/png"
      }
    ],
    "metadata": {
      "attributes": [
        {
          "value": "183",
          "trait_type": "R"
        },
        {
          "value": "89",
          "trait_type": "G"
        },
        {
          "value": "78",
          "trait_type": "B"
        }
      ],
      "description": "Random RGB Color",
      "name": "CNFT",
      "symbol": "CNFT"
    },
    "links": {
      "image": "https://raw.githubusercontent.com/Unboxed-Software/rgb-png-generator/master/assets/183_89_78/183_89_78.png"
    }
  },
  "authorities": [
    {
      "address": "DeogHav5T2UV1zf5XuH4DTwwE5fZZt7Z4evytUUtDtHd",
      "scopes": [
        "full"
      ]
    }
  ],
  "compression": {
    "eligible": false,
    "compressed": true,
    "data_hash": "3RsXHMBDpUPojPLZuMyKgZ1kbhW81YSY3PYmPZhbAx8K",
    "creator_hash": "Di6ufEixhht76sxutC9528H7PaWuPz9hqTaCiQxoFdr",
    "asset_hash": "2TwWjQPdGc5oVripPRCazGBpAyC5Ar1cia8YKUERDepE",
    "tree": "7Ge8nhDv2FcmnpyfvuWPnawxquS6gSidum38oq91Q7vE",
    "seq": 8,
    "leaf_id": 7
  },
  "grouping": [
    {
      "group_key": "collection",
      "group_value": "9p2RqBUAadMznAFiBEawMJnKR9EkFV98wKgwAz8nxLmj"
    }
  ],
  "royalty": {
    "royalty_model": "creators",
    "target": null,
    "percent": 0,
    "basis_points": 0,
    "primary_sale_happened": false,
    "locked": false
  },
  "creators": [
    {
      "address": "HASk3AoTPAvC1KnXSo6Qm73zpkEtEhbmjLpXLgvyKBkR",
      "share": 100,
      "verified": false
    }
  ],
  "ownership": {
    "frozen": false,
    "delegated": false,
    "delegate": null,
    "ownership_model": "single",
    "owner": "HASk3AoTPAvC1KnXSo6Qm73zpkEtEhbmjLpXLgvyKBkR"
  },
  "supply": {
    "print_max_supply": 0,
    "print_current_supply": 0,
    "edition_nonce": 0
  },
  "mutable": false,
  "burnt": false
}
```

请记住，读取 API 还包括获取多个资产、按所有者、创建者等进行查询等功能。务必查看 [Helius 文档](https://docs.helius.dev/compression-and-das-api/digital-asset-standard-das-api) 以了解可用的功能。

## 6. 转移 cNFT

我们要向脚本中添加的最后一项功能是 cNFT 的转移。与标准的 SPL 代币转移一样，安全性至关重要。然而，与标准的 SPL 代币转移不同的是，在构建任何类型的具有状态压缩的安全转移时，执行转移的程序需要整个资产数据。

在这种情况下，即 Bubblegum 需要提供整个已哈希并存储在相应叶子上的数据，并且需要提供有关所讨论叶子的“证明路径”。这使得 cNFT 的转移比 SPL 代币的转移复杂一些。

请记住，一般的步骤如下：

1. 从索引器获取 cNFT 的资产数据
2. 从索引器获取 cNFT 的证明
3. 从 Solana 区块链获取 Merkle 树账户
4. 准备资产证明，以 `AccountMeta` 对象列表的形式
5. 构建并发送 Bubblegum 转账指令

让我们首先声明一个名为 `transferNft` 的函数，该函数接受以下参数：

- `connection` - 一个 `Connection` 对象
- `assetId` - 一个 `PublicKey` 对象
- `sender` - 一个 `Keypair` 对象，以便我们可以签署交易
- `receiver` - 一个表示新所有者的 `PublicKey` 对象

在该函数内部，让我们再次获取资产数据，然后也获取资产证明。为了保险起见，让我们将所有内容都包裹在 `try catch` 中。

```tsx
async function transferNft(
  connection: Connection,
  assetId: PublicKey,
  sender: Keypair,
  receiver: PublicKey
) {
  try {
    const assetDataResponse = await fetch(process.env.RPC_URL, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        jsonrpc: "2.0",
        id: "my-id",
        method: "getAsset",
        params: {
          id: assetId,
        },
      }),
    })
    const assetData = (await assetDataResponse.json()).result

    const assetProofResponse = await fetch(process.env.RPC_URL, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        jsonrpc: "2.0",
        id: "my-id",
        method: "getAssetProof",
        params: {
          id: assetId,
        },
      }),
    })
    const assetProof = (await assetProofResponse.json()).result
	} catch (err: any) {
    console.error("\nFailed to transfer nft:", err)
    throw err
	}
}
```

接下来，让我们从链上获取 Merkle 树账户，获取树冠深度，并组装证明路径。我们通过将从 Helius 获取的资产证明映射到一个 `AccountMeta` 对象列表来完成这一步，然后移除任何已在树冠上缓存的证明节点。

```tsx
async function transferNft(
  connection: Connection,
  assetId: PublicKey,
  sender: Keypair,
  receiver: PublicKey
) {
  try {
    ...

    const treePublicKey = new PublicKey(assetData.compression.tree)

    const treeAccount = await ConcurrentMerkleTreeAccount.fromAccountAddress(
      connection,
      treePublicKey
    )

    const canopyDepth = treeAccount.getCanopyDepth() || 0

    const proofPath: AccountMeta[] = assetProof.proof
      .map((node: string) => ({
        pubkey: new PublicKey(node),
        isSigner: false,
        isWritable: false,
      }))
      .slice(0, assetProof.proof.length - canopyDepth)
  } catch (err: any) {
    console.error("\nFailed to transfer nft:", err)
    throw err
  }
}
```

最后，我们使用 `createTransferInstruction` 构建指令，将其添加到交易中，然后签名并发送交易。完成后，`transferNft` 函数的整体形式如下：

```tsx
async function transferNft(
  connection: Connection,
  assetId: PublicKey,
  sender: Keypair,
  receiver: PublicKey
) {
  try {
    const assetDataResponse = await fetch(process.env.RPC_URL, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        jsonrpc: "2.0",
        id: "my-id",
        method: "getAsset",
        params: {
          id: assetId,
        },
      }),
    })
    const assetData = (await assetDataResponse.json()).result

    const assetProofResponse = await fetch(process.env.RPC_URL, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        jsonrpc: "2.0",
        id: "my-id",
        method: "getAssetProof",
        params: {
          id: assetId,
        },
      }),
    })
    const assetProof = (await assetProofResponse.json()).result

    const treePublicKey = new PublicKey(assetData.compression.tree)

    const treeAccount = await ConcurrentMerkleTreeAccount.fromAccountAddress(
      connection,
      treePublicKey
    )

    const canopyDepth = treeAccount.getCanopyDepth() || 0

    const proofPath: AccountMeta[] = assetProof.proof
      .map((node: string) => ({
        pubkey: new PublicKey(node),
        isSigner: false,
        isWritable: false,
      }))
      .slice(0, assetProof.proof.length - canopyDepth)

    const treeAuthority = treeAccount.getAuthority()
    const leafOwner = new PublicKey(assetData.ownership.owner)
    const leafDelegate = assetData.ownership.delegate
      ? new PublicKey(assetData.ownership.delegate)
      : leafOwner

    const transferIx = createTransferInstruction(
      {
        merkleTree: treePublicKey,
        treeAuthority,
        leafOwner,
        leafDelegate,
        newLeafOwner: receiver,
        logWrapper: SPL_NOOP_PROGRAM_ID,
        compressionProgram: SPL_ACCOUNT_COMPRESSION_PROGRAM_ID,
        anchorRemainingAccounts: proofPath,
      },
      {
        root: [...new PublicKey(assetProof.root.trim()).toBytes()],
        dataHash: [
          ...new PublicKey(assetData.compression.data_hash.trim()).toBytes(),
        ],
        creatorHash: [
          ...new PublicKey(assetData.compression.creator_hash.trim()).toBytes(),
        ],
        nonce: assetData.compression.leaf_id,
        index: assetData.compression.leaf_id,
      }
    )

    const tx = new Transaction().add(transferIx)
    tx.feePayer = sender.publicKey
    const txSignature = await sendAndConfirmTransaction(
      connection,
      tx,
      [sender],
      {
        commitment: "confirmed",
        skipPreflight: true,
      }
    )
    console.log(`https://explorer.solana.com/tx/${txSignature}?cluster=devnet`)
  } catch (err: any) {
    console.error("\nFailed to transfer nft:", err)
    throw err
  }
}
```

让我们将我们的第一个压缩 NFT（索引为 0）转移给其他人。首先，我们需要启动另一个带有一些资金的钱包，然后使用 `getLeafAssetId` 获取索引为 0 的资产ID。接下来，我们将执行转移操作。最后，我们将使用我们的函数 `logNftDetails` 打印出整个收藏品。您将注意到，索引为 0 的 NFT 现在将在 `ownership` 字段中属于我们的新钱包。

```tsx
async function main() {
  const connection = new Connection(clusterApiUrl("devnet"), "confirmed")
  const wallet = await getOrCreateKeypair("Wallet_1")
  await airdropSolIfNeeded(wallet.publicKey)

  const maxDepthSizePair: ValidDepthSizePair = {
    maxDepth: 3,
    maxBufferSize: 8,
  }

  const canopyDepth = 0

  const treeAddress = await createAndInitializeTree(
    connection,
    wallet,
    maxDepthSizePair,
    canopyDepth
  )

  const collectionNft = await getOrCreateCollectionNFT(connection, wallet)

  await mintCompressedNftToCollection(
    connection,
    wallet,
    treeAddress,
    collectionNft,
    2 ** maxDepthSizePair.maxDepth
  )

  const recieverWallet = await getOrCreateKeypair("Wallet_2")
  const assetId = await getLeafAssetId(treeAddress, new BN(0))
  await airdropSolIfNeeded(recieverWallet.publicKey)

  console.log(`Transfering ${assetId.toString()} from ${wallet.publicKey.toString()} to ${recieverWallet.publicKey.toString()}`)

  await transferNft(
    connection,
    assetId,
    wallet,
    recieverWallet.publicKey
  )

  await logNftDetails(treeAddress, 8)
}
```

请执行您的脚本。整个过程应该可以顺利执行，而且几乎只需 0.01 SOL！

恭喜！现在您知道如何铸造、读取和转移 cNFT 了。如果您愿意，您可以将最大深度、最大缓冲区大小和 Canopy 深度更新为更大的值，只要您有足够的 Devnet SOL，这个脚本就可以让您铸造多达 10,000 个 cNFT，成本仅为铸造 10,000 个传统 NFT 所需成本的一小部分（注：如果您计划铸造大量的 NFT，您可能希望尝试将这些指令批处理为更少的总交易）。

如果您需要更多时间来完成这个实验，请随时再次阅读或查看[实验仓库](https://github.com/Unboxed-Software/solana-cnft-demo/tree/solution)中 `solution` 分支上的解决方案代码。

## 挑战

现在轮到您自己尝试这些概念了！我们不会在这一点上过于具体地规定，但以下是一些想法：

1. 创建您自己的生产 cNFT 收藏品
2. 为本课实验构建一个 UI，使您可以铸造 cNFT 并将其显示出来
3. 看看您是否可以在链上程序中复制实验脚本的一些功能，即编写一个可以铸造 cNFT 的程序


## 完成了实验？

将您的代码推送到 GitHub，并[告诉我们您对本课程的想法](https://form.typeform.com/to/IPH0UGz7#answers-lesson=db156789-2400-4972-904f-40375582384a)！
