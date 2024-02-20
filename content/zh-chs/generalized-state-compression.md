---
title: 通用状态压缩
objectives:
- 解释 Solana 状态压缩背后的逻辑流程
- 解释 Merkle 树和并发 Merkle 树之间的区别
- 在基本 Solana 程序中实现通用状态压缩
---

# TL;DR

- 在 Solana 上，状态压缩最常用于压缩 NFT，但也可以用于任意数据。
- 状态压缩通过利用 Merkle 树降低了您在链上存储的数据量。
- Merkle 树存储一个单一的哈希值，代表了一个完整的哈希二叉树。每个 Merkle 树的叶子节点都是该叶子节点数据的哈希值。
- 并发 Merkle 树是 Merkle 树的一种特殊版本，允许并发更新。
- 因为状态压缩程序中的数据不存储在链上，所以您必须使用索引器来维护数据的链下缓存，然后将该数据与链上的 Merkle 树进行验证。

# 概述

之前，我们在压缩 NFT 的上下文中讨论了状态压缩（state compression）。在撰写本文时，压缩 NFT 代表了状态压缩的最常见用例，但可以在任何程序中使用状态压缩。在本课程中，我们将更加通用地讨论状态压缩，以便您可以将其应用于您的任何程序中。

## 状态压缩的理论概述

在传统程序中，数据被序列化（通常使用 borsh），然后直接存储在账户中。这使得数据可以通过 Solana 程序轻松读取和写入。您可以“信任”存储在账户中的数据，因为除了通过程序提供的机制外，它不能被修改。

状态压缩实际上断言了这个方程式中最重要的部分是数据的“可信度”。如果我们关心的只是能够相信数据是它所声称的那样，那么实际上我们可以***不***在链上的账户中存储数据。相反，我们可以存储数据的哈希值，其中哈希值可以用于证明或验证数据。数据哈希占用的存储空间远远小于数据本身。然后，我们可以将实际数据存储在更便宜的地方，并在访问数据时小心验证它与链上哈希的匹配性。

Solana 状态压缩程序使用的特定数据结构是一种特殊的二叉树结构，称为**并发 Merkle 树（concurrent merkle tree）**。该树结构以确定性方式将数据片段进行哈希运算，计算出一个单一的最终哈希，并将其存储在链上。这个最终哈希的大小远远小于所有原始数据的总和，因此称为“压缩”。该过程的步骤如下：

1. 获取任意数据片段
2. 对该数据进行哈希运算
3. 将该哈希作为“叶子（leaf）”存储在树的底部
4. 然后将每对叶子进行哈希运算，形成一个“分支（branch）”
5. 然后每对分支进行哈希
6. 继续沿着树向上爬行，并将相邻的分支进行哈希运算
7. 一旦到达树的顶部，就会产生一个最终的“根哈希”
8. 将根哈希作为可验证的证明存储在链上，证明每个叶子内的数据
9. 任何想要验证他们所拥有的数据是否与“真实来源”匹配的人，都可以通过相同的过程进行比较最终的哈希，而无需将所有数据都存储在链上。

这涉及到一些相当严重的开发权衡：

1. 由于数据不再存储在链上的账户中，因此访问数据变得更加困难。
2. 一旦访问了数据，开发人员必须决定他们的应用程序将多频繁地根据链上的哈希验证数据。
3. 对数据的任何更改都将需要将先前已经哈希过的全部数据以及新数据发送到一个指令中。开发人员可能还必须提供与验证原始数据与哈希之间所需的证明相关的额外数据。

在确定是否、何时以及如何实现状态压缩时，每个都将是需要考虑的因素。

### 并发 Merkle 树

**Merkle 树** 是一种由单个哈希表示的二叉树结构。结构中的每个叶子节点都是其内部数据的哈希，而每个分支都是其子叶哈希的哈希。依此类推，分支也会被一起哈希，直到最终剩下一个根哈希。

由于 Merkle 树被表示为单个哈希，对叶子数据的任何修改都会改变根哈希。当同一 slot 中的多个交易尝试修改叶子数据时，这会引发问题。由于这些交易必须按顺序执行，除了第一个交易之外的所有交易都将失败，因为由第一个执行的交易完成后，根哈希和证明（proof）会被修改，后续的传递根哈希和证明将是无效的。换句话说，标准的 Merkle 树每个 slot 只能修改一个叶子。在一个依赖单个 Merkle 树作为其状态的虚拟状态压缩程序中，这严重限制了吞吐量。

这可以通过使用**并发 Merkle 树**来解决。并发 Merkle 树是一种 Merkle 树，它存储了最近更改的安全变更日志，以及其根哈希和用于推导它的证明。当同一 slot 中的多个交易尝试修改叶子数据时，可以使用变更日志作为真实来源，允许对树进行并发更改。

换句话说，虽然存储 Merkle 树的账户只会有根哈希，但并发 Merkle 树还将包含允许后续写入成功发生的附加数据。这包括：

1. 根哈希（root hash） - 与标准 Merkle 树相同的根哈希。
2. 变更日志缓冲区（changelog buffer） - 此缓冲区包含与最近根哈希更改相关的证明数据，以便同一 slot 中的后续写入仍然可以成功。
3. canopy - 在对任何给定叶子进行更新操作时，您需要从该叶子到根哈希的整个证明路径。canopy 存储沿该路径的中间证明节点，因此不必全部从客户端传递到程序中。

作为程序架构师，您直接控制与这三个项目直接相关的三个值。您的选择决定了树的大小、创建树的成本以及可以对树进行的并发更改的数量：

1. 最大深度（Max depth）
2. 最大缓冲区大小（Max buffer size）
3. Canopy depth

- **最大深度（Max depth）** 是从任何叶子到达树根的最大跳数。由于 Merkle 树是二叉树，每个叶子只与另一个叶子相连。最大深度可以逻辑上用于计算树的节点数量，公式为 `2 ^ maxDepth`。

- **最大缓冲区大小（Max buffer size）** 实际上是在单个 slot 中对树进行的并发更改的最大数量。当在同一 slot 中提交多个交易，每个交易都试图更新标准 Merkle 树上的叶子时，只有第一个运行的交易将有效。这是因为该“写入”操作将修改存储在账户中的哈希。同一 slot 中的后续交易将尝试将将现在已过时的哈希来验证他们的数据。并发 Merkle 树具有缓冲区，以便缓冲区可以记录这些修改的运行日志。这允许状态压缩程序在同一 slot 中验证多个数据写入，因为它可以查找缓冲区中的先前哈希，并与适当的哈希进行比较。

- **Canopy depth** 是存储在链上的任何给定证明路径的证明节点数。验证任何叶子都需要完整的树的证明路径。完整的证明路径由树的每“层”的一个证明节点组成，即最大深度为 14 意味着有 14 个证明节点。每个传递到程序中的证明节点都会为交易增加 32 字节，因此大型树很快就会超过最大交易大小限制。在 canopy 中在链上缓存证明节点有助于提高程序的组合性。

这三个值中的每一个，最大深度、最大缓冲区大小和 canopy 深度，都伴随着一些权衡。增加任何一个值都会增加用于存储树的账户的大小，从而增加创建树的成本。

选择最大深度相对比较直观，因为它直接关系到叶子的数量，因此也就关系到可以存储的数据量。如果您需要在一个单独的树上存储 100 万个 cNFT，其中每个 cNFT 是树的一个叶子，那么找到使以下表达式成立的最大深度：`2^maxDepth > 100万`。答案是20。

选择最大缓冲区大小实际上是一个关于吞吐量的问题：您需要多少并发写入？缓冲区越大，吞吐量就越高。

最后，canopy 深度将决定您的程序的组合性。状态压缩先驱们已经明确表示省略 canopy 是一个坏主意。如果这样做会使交易大小限制达到上限，那么程序 A 就无法调用您的状态压缩程序 B。请记住，除了需要的证明路径外，程序 A 还需要必要的账户和数据，这些都会占用交易空间。

### 在状态压缩程序中的数据访问

状态压缩账户并不存储数据本身。相反，它存储了上述讨论过的并发 Merkle 树结构。原始数据本身仅存在于区块链更便宜的**账本状态（ledger state）**中。这使得数据访问略微困难，但并非不可能。

Solana 账本是一个包含已签名交易的条目列表。理论上，这可以追溯到创世区块。这实际上意味着任何曾经放入交易中的数据都存在于账本中。

由于状态压缩的哈希处理过程发生在链上，所有数据都存在于账本状态中，理论上可以通过从头开始重新播放整个链状态来从原始交易中检索。然而，更直接（虽然仍然复杂）的方法是让一个**索引器（indexer）**在交易发生时跟踪和索引这些数据。这确保了有一个链下的数据“缓存”，任何人都可以访问，并随后根据链上的根哈希进行验证。

这个过程很复杂，但经过一些实践后会变得很容易理解。

## 状态压缩工具

上面描述的理论对于正确理解状态压缩至关重要。但您并不需要从头开始实现其中任何内容。杰出的工程师们已经为您奠定了大部分基础，以 SPL 状态压缩程序和 Noop 程序的形式。

### SPL 状态压缩和 Noop 程序

SPL 状态压缩程序（SPL State Compression Program）的存在旨在使创建和更新并发 Merkle 树的过程在 Solana 生态系统中可重复和可组合。它提供了初始化 Merkle 树、管理树叶（即添加、更新、删除数据）和验证叶数据的指令。

状态压缩程序还利用了一个单独的“no op”（空操作）程序，其主要目的是通过将叶子数据记录到账本状态中使叶子数据更容易索引。当您想要存储压缩数据时，您将其传递给状态压缩程序，其中它会被哈希化并作为“事件（event）”发送到 Noop 程序。哈希将存储在相应的并发 Merkle 树中，但原始数据仍可通过 Noop 程序的交易日志进行访问。


### 索引数据以便于查找

在正常情况下，您通常会通过获取相应的账户来访问链上数据。然而，在使用状态压缩时，情况并非如此简单。

如上所述，数据现在存在于账本状态中，而不是账户中。找到完整数据最简单的地方是在 Noop 指令的日志中。不幸的是，虽然这些数据从某种意义上说将永远存在于账本状态中，但在一定时间后，它们很可能会被验证节点访问不到。

为了节省空间并提高性能，验证节点不会保留每个交易直到创世区块。您将能够访问与您的数据相关的 Noop 指令日志的具体时间将根据验证节点而异。如果您直接依赖指令日志，最终您将无法访问它。

从技术上讲，您可以重新播放交易状态到创世区块，但普通团队不会这样做，而且肯定不会性能良好。[数字资产标准（DAS）](https://docs.helius.dev/compression-and-das-api/digital-asset-standard-das-api)已被许多 RPC 提供商采用，以实现对压缩 NFT 和其他资产的高效查询。然而，在撰写本文时，它不支持任意状态压缩。因此，您有两个主要选择：

1. 使用一个索引提供商，该提供商将为您的程序构建一个自定义索引解决方案，观察发送到 Noop 程序的事件，并将相关数据存储在链下。
2. 创建您自己的伪索引解决方案，将交易数据存储在链下。

对于许多 dApp 来说，选项 2 是很合理的。而规模较大的应用程序可能需要依赖基础设施提供商来处理它们的索引。

## 状态压缩开发流程

### 创建 Rust 类型

与典型的 Anchor 程序一样，您应该首先定义程序的 Rust 类型。然而，在传统的 Anchor 程序中，Rust 类型通常表示账户。而在状态压缩程序中，您的账户状态只会存储 Merkle 树。更“可用”的数据模式将只是被序列化并记录到 Noop 程序中。

该类型应包括存储在叶子节点中的所有数据以及任何解释数据所需的上下文信息。例如，如果您要创建一个简单的消息传递程序，您的 `Message` 结构可能如下所示：

```rust
#[derive(AnchorSerialize)]
pub struct MessageLog {
		leaf_node: [u8; 32], // The leaf node hash
    from: Pubkey,        // Pubkey of the message sender
		to: Pubkey,          // Pubkey of the message recipient
    message: String,     // The message to send
}

impl MessageLog {
    // Constructs a new message log from given leaf node and message
    pub fn new(leaf_node: [u8; 32], from: Pubkey, to: Pubkey, message: String) -> Self {
        Self { leaf_node, from, to, message }
    }
}
```

需要非常清楚地指出，**这不是您将能够从中读取的账户**。您的程序将从指令输入创建此类型的实例，而不是从它读取的账户数据构造此类型的实例。我们将在后面的部分讨论如何读取数据。

### 初始化新树

客户端将通过两个单独的指令创建和初始化 Merkle 树账户。第一个简单地通过调用系统程序来分配账户。第二个将是您在自定义程序上创建的一个指令，用于初始化新账户。此初始化实际上只是记录 Merkle 树的最大深度和缓冲区大小。

这个指令所需要做的就是建立一个 CPI 来调用状态压缩程序上的 `init_empty_merkle_tree` 指令。由于这需要最大深度和最大缓冲区大小，这些将需要作为参数传递给该指令。

请记住，最大深度是指从任何叶子到达树根的最大跳数。最大缓冲区大小是指为存储树更新的变更日志而保留的空间量。这个变更日志用于确保您的树可以支持同一区块内的并发更新。

例如，如果我们正在初始化用于存储用户之间消息的树，该指令可能如下所示：

```rust
pub fn create_messages_tree(
    ctx: Context<MessageAccounts>,
    max_depth: u32, // Max depth of the merkle tree
    max_buffer_size: u32 // Max buffer size of the merkle tree
) -> Result<()> {
    // Get the address for the merkle tree account
    let merkle_tree = ctx.accounts.merkle_tree.key();
    // Define the seeds for pda signing
    let signer_seeds: &[&[&[u8]]] = &[
        &[
            merkle_tree.as_ref(), // The address of the merkle tree account as a seed
            &[*ctx.bumps.get("tree_authority").unwrap()], // The bump seed for the pda
        ],
    ];

    // Create cpi context for init_empty_merkle_tree instruction.
    let cpi_ctx = CpiContext::new_with_signer(
        ctx.accounts.compression_program.to_account_info(), // The spl account compression program
        Initialize {
            authority: ctx.accounts.tree_authority.to_account_info(), // The authority for the merkle tree, using a PDA
            merkle_tree: ctx.accounts.merkle_tree.to_account_info(), // The merkle tree account to be initialized
            noop: ctx.accounts.log_wrapper.to_account_info(), // The noop program to log data
        },
        signer_seeds // The seeds for pda signing
    );

    // CPI to initialize an empty merkle tree with given max depth and buffer size
    init_empty_merkle_tree(cpi_ctx, max_depth, max_buffer_size)?;

    Ok(())
}
```

### 向树中添加哈希值

有了初始化的 Merkle 树，就可以开始添加数据哈希值了。这涉及将未压缩的数据传递给您的程序上的一个指令，该指令将对数据进行哈希处理，记录到 Noop 程序中，并使用状态压缩程序的 `append` 指令将哈希添加到树中。以下详细讨论了您的指令需要做的事情：

1. 使用 `keccak` crate 中的 `hashv` 函数对数据进行哈希处理。在大多数情况下，您还希望对数据的所有者或权限进行哈希处理，以确保它只能由适当的权限修改。
2. 创建一个表示要记录到 Noop 程序的数据的日志对象，然后调用 `wrap_application_data_v1` 发出一个 CPI 到 Noop 程序，其中包含此对象。这样可以确保未压缩的数据对任何正在寻找它的客户端都是可用的。对于广泛的用例，如 cNFT，这将是索引器。您还可以创建自己的观察客户端来模拟索引器的操作，但特定于您的应用程序。
3. 构建并发出一个 CPI 到状态压缩程序的 `append` 指令。这需要在步骤 1 中计算的哈希，并将其添加到您的 Merkle 树上的下一个可用叶子上。就像之前一样，这需要 Merkle 树地址和树权限增量作为签名种子。

当将所有这些内容结合使用消息示例时，看起来是这样的：

```rust
// Instruction for appending a message to a tree.
pub fn append_message(ctx: Context<MessageAccounts>, message: String) -> Result<()> {
    // Hash the message + whatever key should have update authority
    let leaf_node = keccak::hashv(&[message.as_bytes(), ctx.accounts.sender.key().as_ref()]).to_bytes();
    // Create a new "message log" using the leaf node hash, sender, receipient, and message
    let message_log = MessageLog::new(leaf_node.clone(), ctx.accounts.sender.key().clone(), ctx.accounts.receipient.key().clone(), message);
    // Log the "message log" data using noop program
    wrap_application_data_v1(message_log.try_to_vec()?, &ctx.accounts.log_wrapper)?;
    // Get the address for the merkle tree account
    let merkle_tree = ctx.accounts.merkle_tree.key();
    // Define the seeds for pda signing
    let signer_seeds: &[&[&[u8]]] = &[
        &[
            merkle_tree.as_ref(), // The address of the merkle tree account as a seed
            &[*ctx.bumps.get("tree_authority").unwrap()], // The bump seed for the pda
        ],
    ];
    // Create a new cpi context and append the leaf node to the merkle tree.
    let cpi_ctx = CpiContext::new_with_signer(
        ctx.accounts.compression_program.to_account_info(), // The spl account compression program
        Modify {
            authority: ctx.accounts.tree_authority.to_account_info(), // The authority for the merkle tree, using a PDA
            merkle_tree: ctx.accounts.merkle_tree.to_account_info(), // The merkle tree account to be modified
            noop: ctx.accounts.log_wrapper.to_account_info(), // The noop program to log data
        },
        signer_seeds // The seeds for pda signing
    );
    // CPI to append the leaf node to the merkle tree
    append(cpi_ctx, leaf_node)?;
    Ok(())
}
```

### 更新哈希值

要更新数据，您需要创建一个新的哈希值，以替换 Merkle 树上相应叶子处的哈希值。为此，您的程序需要访问四个东西：

1. 要更新的叶子的索引
2. Merkle 树的根哈希
3. 您要修改的原始数据
4. 更新后的数据

在获得这些数据的情况下，一个程序指令可以按照非常类似于将初始数据附加到树上的步骤进行操作：

1. **验证更新权限** - 这是一个新步骤。在大多数情况下，您希望验证更新权限。这通常涉及证明“更新”交易的签名者是给定索引处叶子的真实所有者或权限。由于数据以哈希的形式压缩在叶子上，我们不能简单地将“权限”公钥与存储的值进行比较。相反，我们需要使用旧数据和账户验证结构中列出的“权限”计算先前的哈希。然后，我们构建并发出一个 CPI 到状态压缩程序的 `verify_leaf` 指令，使用我们计算得到的哈希。
2. **对新数据进行哈希处理** - 这一步与附加初始数据的第一步相同。使用 `keccak` crate 中的 `hashv` 函数对新数据和更新权限进行哈希处理，分别使用它们的相应字节表示。
3. **记录新数据** - 这一步与附加初始数据的第二步相同。创建日志结构的实例，并调用 `wrap_application_data_v1` 发出一个 CPI 到 Noop 程序。
4. **替换现有叶子哈希** - 这一步与附加初始数据的最后一步略有不同。构建并发出一个 CPI 到状态压缩程序的 `replace_leaf` 指令。这使用旧哈希、新哈希和叶子索引来用新哈希替换给定哈希处叶子的数据。与之前一样，这需要 Merkle 树地址和树权限增量作为签名种子。

将这个过程合并成一个单一指令，如下所示：

```rust
pub fn update_message(
    ctx: Context<MessageAccounts>,
    index: u32,
    root: [u8; 32],
    old_message: String,
    new_message: String
) -> Result<()> {
    let old_leaf = keccak
        ::hashv(&[old_message.as_bytes(), ctx.accounts.sender.key().as_ref()])
        .to_bytes();

    let merkle_tree = ctx.accounts.merkle_tree.key();

    // Define the seeds for pda signing
    let signer_seeds: &[&[&[u8]]] = &[
        &[
            merkle_tree.as_ref(), // The address of the merkle tree account as a seed
            &[*ctx.bumps.get("tree_authority").unwrap()], // The bump seed for the pda
        ],
    ];

    // Verify Leaf
    {
        if old_message == new_message {
            msg!("Messages are the same!");
            return Ok(());
        }

        let cpi_ctx = CpiContext::new_with_signer(
            ctx.accounts.compression_program.to_account_info(), // The spl account compression program
            VerifyLeaf {
                merkle_tree: ctx.accounts.merkle_tree.to_account_info(), // The merkle tree account to be modified
            },
            signer_seeds // The seeds for pda signing
        );
        // Verify or Fails
        verify_leaf(cpi_ctx, root, old_leaf, index)?;
    }

    let new_leaf = keccak
        ::hashv(&[new_message.as_bytes(), ctx.accounts.sender.key().as_ref()])
        .to_bytes();

    // Log out for indexers
    let message_log = MessageLog::new(new_leaf.clone(), ctx.accounts.sender.key().clone(), ctx.accounts.recipient.key().clone(), new_message);
    // Log the "message log" data using noop program
    wrap_application_data_v1(message_log.try_to_vec()?, &ctx.accounts.log_wrapper)?;

    // replace leaf
    {
        let cpi_ctx = CpiContext::new_with_signer(
            ctx.accounts.compression_program.to_account_info(), // The spl account compression program
            Modify {
                authority: ctx.accounts.tree_authority.to_account_info(), // The authority for the merkle tree, using a PDA
                merkle_tree: ctx.accounts.merkle_tree.to_account_info(), // The merkle tree account to be modified
                noop: ctx.accounts.log_wrapper.to_account_info(), // The noop program to log data
            },
            signer_seeds // The seeds for pda signing
        );
        // CPI to append the leaf node to the merkle tree
        replace_leaf(cpi_ctx, root, old_leaf, new_leaf, index)?;
    }

    Ok(())
}
```

### 删除哈希值

在撰写本文时，状态压缩程序没有提供显式的 `delete` 指令。相反，您可以通过更新叶子数据为指示数据 “已删除（deleted）” 的数据来实现删除。具体的数据将取决于您的用例和安全问题。有些人可能会将所有数据设置为 0，而其他人可能会存储一个所有“已删除”元素都共有的静态字符串。

### 从客户端访问数据

到目前为止，讨论涵盖了4个标准的 CRUD 过程中的3个：创建（Create）、更新（Update）和删除（Delete）。剩下的是状态压缩中更难理解的概念之一：读取数据（Read）。

从客户端访问数据是棘手的，主要是因为数据不是以易于访问的格式存储的。存储在 Merkle 树账户中的数据哈希不能用于重建初始数据，并且记录到 Noop 程序的数据也不是永久可用的。

您最好的选择是两种选项之一：

1. 与索引提供商合作，为您的程序创建一个自定义的索引解决方案，然后根据索引器如何让您访问数据进而编写客户端代码。
2. 创建自己的伪索引器作为一种轻量级解决方案。

如果您的项目确实是完全去中心化的，以至于许多参与者将通过自己的前端以外的方式与您的程序交互，那么选项 2 可能不足以满足需求。然而，根据项目规模或您是否能控制大部分程序访问的情况，它可能是一种可行的方法。

并没有“正确”的方法来做这件事。两种潜在的方法是：

1. 在将数据发送到程序的同时，将原始数据存储在数据库中，并附带数据被哈希并存储到的叶子。
2. 创建一个服务器，观察您程序的交易，查找相关的 Noop 日志，解码日志，并将其存储。

我们在本课程实验中编写测试时将同时使用这两种方法（尽管我们不会将数据持久化到数据库中 - 它只会在测试期间存在于内存中）。

这个设置有点繁琐。给定一个特定的交易，您可以从 RPC 提供商获取交易，获取与 Noop 程序关联的内部指令，使用 `@solana/spl-account-compression` JavaScript 包中的 `deserializeApplicationDataEvent` 函数获取日志，然后使用 Borsh 进行反序列化。下面是一个基于上面使用的消息程序的示例。

```tsx
export async function getMessageLog(connection: Connection, txSignature: string) {
  // Confirm the transaction, otherwise the getTransaction sometimes returns null
  const latestBlockHash = await connection.getLatestBlockhash()
  await connection.confirmTransaction({
    blockhash: latestBlockHash.blockhash,
    lastValidBlockHeight: latestBlockHash.lastValidBlockHeight,
    signature: txSignature,
  })

  // Get the transaction info using the tx signature
  const txInfo = await connection.getTransaction(txSignature, {
    maxSupportedTransactionVersion: 0,
  })

  // Get the inner instructions related to the program instruction at index 0
  // We only send one instruction in test transaction, so we can assume the first
  const innerIx = txInfo!.meta?.innerInstructions?.[0]?.instructions

  // Get the inner instructions that match the SPL_NOOP_PROGRAM_ID
  const noopInnerIx = innerIx.filter(
    (instruction) =>
      txInfo?.transaction.message.staticAccountKeys[
        instruction.programIdIndex
      ].toBase58() === SPL_NOOP_PROGRAM_ID.toBase58()
  )

  let noteLog: NoteLog
  for (let i = noopInnerIx.length - 1; i >= 0; i--) {
    try {
      // Try to decode and deserialize the instruction data
      const applicationDataEvent = deserializeApplicationDataEvent(
        Buffer.from(bs58.decode(noopInnerIx[i]?.data!))
      )

      // Get the application data
      const applicationData = applicationDataEvent.fields[0].applicationData

      // Deserialize the application data into MessageLog instance
      messageLog = deserialize(
        MessageLogBorshSchema,
        MessageLog,
        Buffer.from(applicationData)
      )

      if (messageLog !== undefined) {
        break
      }
    } catch (__) {}
  }

  return messageLog
}
```

## 结论

通用状态压缩可能会很困难，但绝对可以使用现有工具实现。此外，随着时间的推移，工具和程序将会变得更加完善。如果您想出了能够改进开发体验的解决方案，请与社区分享！

# 实验

让我们通过创建一个新的 Anchor 程序来练习通用状态压缩。该程序将使用自定义状态压缩来支持一个简单的记笔记应用。

## 1. 创建项目

首先初始化一个 Anchor 程序：

```bash
anchor init compressed-notes
```

We’ll be using the `spl-account-compression` crate with the `cpi` feature enabled. Let’s add it as a dependency in `programs/compressed-notes/Cargo.toml`.

在 `programs/compressed-notes/Cargo.toml` 中添加依赖项 `spl-account-compression` 并启用 `cpi` 功能。

```toml
[dependencies]
anchor-lang = "0.28.0"
spl-account-compression = { version="0.2.0", features = ["cpi"] }
solana-program = "1.16.0"
```

我们将在本地进行测试，但我们需要从主网获取压缩程序和 Noop 程序。我们需要将它们添加到根目录中的 `Anchor.toml` 中，以便它们被克隆到我们的本地集群。

```toml
[test.validator]
url = "https://api.mainnet-beta.solana.com"

[[test.validator.clone]]
address = "noopb9bkMVfRPU8AsbpTUg8AQkHtKwMYZiFUjNRtMmV"

[[test.validator.clone]]
address = "cmtDvXumGCrqC1Age74AVPhSRVXJMd8PJS91L8KbNCK"
```

最后，让我们为其余的演示准备 `lib.rs` 文件。删除 `initialize` 指令和 `Initialize` 账户结构，然后添加下面代码片段中显示的导入（确保放入***您的***程序ID）：

```rust
use anchor_lang::{
    prelude::*, 
    solana_program::keccak
};
use spl_account_compression::{
    Noop,
    program::SplAccountCompression,
    cpi::{
        accounts::{Initialize, Modify, VerifyLeaf},
        init_empty_merkle_tree, verify_leaf, replace_leaf, append, 
    },
    wrap_application_data_v1, 
};

declare_id!("YOUR_KEY_GOES_HERE");

// STRUCTS GO HERE

#[program]
pub mod compressed_notes {
    use super::*;

	// FUNCTIONS GO HERE
	
}
```

在接下来的演示中，我们将直接在 `lib.rs` 文件中对程序代码进行更新。这样可以简化解释。您可以根据自己的意愿修改结构。

在继续之前，请随意构建。这可以确保您的环境正常工作，并缩短将来的构建时间。

## 2. 定义 `Note` 布局

接下来，我们将定义程序中笔记的结构。笔记应该具有以下属性：

- `leaf_node` - 这应该是一个 32 字节的数组，表示存储在叶节点上的哈希
- `owner` - 笔记所有者的公钥
- `note` - 笔记的字符串表示

```rust
#[derive(AnchorSerialize)]
pub struct NoteLog {
    leaf_node: [u8; 32],  // The leaf node hash
    owner: Pubkey,        // Pubkey of the note owner
    note: String,         // The note message
}

impl NoteLog {
    // Constructs a new note from given leaf node and message
    pub fn new(leaf_node: [u8; 32], owner: Pubkey, note: String) -> Self {
        Self { leaf_node, owner, note }
    }
}
```

在传统的 Anchor 程序中，这将是一个账户结构体，但由于我们使用状态压缩，我们的账户不会完全反映我们的本地结构。由于我们不需要账户的所有功能，我们可以使用 `AnchorSerialize` 派生宏而不是 `account` 宏。

## 3. 定义输入账户和约束

幸运的是，我们的每个指令都将使用相同的账户。我们将为我们的账户验证创建一个单独的 `NoteAccounts` 结构体。它将需要以下账户：

- `owner` - 笔记的创建者和所有者；应在交易中签署
- `tree_authority` - Merkle 树的授权方；用于签署与压缩相关的 CPIs
- `merkle_tree` - 用于存储笔记哈希的 Merkle 树的地址；由于由状态压缩程序验证，因此将不受检查
- `log_wrapper` - Noop 程序的地址
- `compression_program` - 状态压缩程序的地址

```rust
#[derive(Accounts)]
pub struct NoteAccounts<'info> {
    // The payer for the transaction
    #[account(mut)]
    pub owner: Signer<'info>,

    // The pda authority for the merkle tree, only used for signing
    #[account(
        seeds = [merkle_tree.key().as_ref()],
        bump,
    )]
    pub tree_authority: SystemAccount<'info>,

    // The merkle tree account
    /// CHECK: This account is validated by the spl account compression program
    #[account(mut)]
    pub merkle_tree: UncheckedAccount<'info>,

    // The noop program to log data
    pub log_wrapper: Program<'info, Noop>,

    // The spl account compression program
    pub compression_program: Program<'info, SplAccountCompression>,
}
```

## 4. 创建 `create_note_tree` 指令

接下来，让我们创建我们的 `create_note_tree` 指令。请记住，客户端已经分配了 Merkle 树账户，但将使用此指令来初始化它。

此指令只需要构建一个 CPI 来调用状态压缩程序上的 `init_empty_merkle_tree` 指令。为此，它需要 `NoteAccounts` 账户验证结构中列出的账户。它还需要两个额外的参数：

1. `max_depth` - Merkle 树的最大深度
2. `max_buffer_size` - Merkle 树的最大缓冲区大小

这些值是初始化 Merkle 树账户上的数据所需的。请记住，最大深度是指从任何叶子到树根的最大跳数。最大缓冲区大小是指为存储树更新的更改日志保留的空间量。此更改日志用于确保您的树能够支持同一区块内的并发更新。

```rust
#[program]
pub mod compressed_notes {
    use super::*;

    // Instruction for creating a new note tree.
    pub fn create_note_tree(
        ctx: Context<NoteAccounts>,
        max_depth: u32,       // Max depth of the merkle tree
        max_buffer_size: u32, // Max buffer size of the merkle tree
    ) -> Result<()> {
        // Get the address for the merkle tree account
        let merkle_tree = ctx.accounts.merkle_tree.key();

        // Define the seeds for pda signing
        let signer_seeds: &[&[&[u8]]] = &[&[
            merkle_tree.as_ref(), // The address of the merkle tree account as a seed
            &[*ctx.bumps.get("tree_authority").unwrap()], // The bump seed for the pda
        ]];

        // Create cpi context for init_empty_merkle_tree instruction.
        let cpi_ctx = CpiContext::new_with_signer(
            ctx.accounts.compression_program.to_account_info(), // The spl account compression program
            Initialize {
                authority: ctx.accounts.tree_authority.to_account_info(), // The authority for the merkle tree, using a PDA
                merkle_tree: ctx.accounts.merkle_tree.to_account_info(), // The merkle tree account to be initialized
                noop: ctx.accounts.log_wrapper.to_account_info(), // The noop program to log data
            },
            signer_seeds, // The seeds for pda signing
        );

        // CPI to initialize an empty merkle tree with given max depth and buffer size
        init_empty_merkle_tree(cpi_ctx, max_depth, max_buffer_size)?;
        Ok(())
    }

    //...
}
```

确保您在 CPI 上的签名者种子中包括 Merkle 树地址和树授权的 bump。

## 5. 创建 `append_note` 指令

现在，让我们创建我们的 `append_note` 指令。此指令需要将原始笔记作为字符串获取并将其压缩为哈希，我们将在 Merkle 树上存储该哈希。我们还将将笔记记录到 Noop 程序中，以便数据的完整性存在于链的状态中。

以下是操作步骤：

1. 使用 `keccak` crate 中的 `hashv` 函数对笔记和所有者进行哈希，每个都作为其相应的字节表示。非常 ***重要*** 的是，您需要对所有者和笔记进行哈希。这是我们在更新指令中在更新之前验证笔记所有权的方式。
2. 使用步骤 1 中的哈希、所有者的公钥和原始笔记作为字符串创建 `NoteLog` 结构体的实例。然后调用 `wrap_application_data_v1` 来向 Noop 程序发出 CPI，传递 `NoteLog` 的实例。这确保了笔记的完整内容（而不仅仅是哈希）对于任何正在寻找它的客户端都是立即可用的。对于像 cNFT 这样的广泛用例，这可能是索引器。您可能会创建观察客户端来模拟索引器正在做的事情，但是针对您自己的应用程序。
3. 构建并发出 CPI 到状态压缩程序的 `append` 指令。这需要在签名种子中使用 Merkle 树地址和树授权的 bump。

```rust
#[program]
pub mod compressed_notes {
    use super::*;

    //...

    // Instruction for appending a note to a tree.
    pub fn append_note(ctx: Context<NoteAccounts>, note: String) -> Result<()> {
        // Hash the "note message" which will be stored as leaf node in the merkle tree
        let leaf_node =
            keccak::hashv(&[note.as_bytes(), ctx.accounts.owner.key().as_ref()]).to_bytes();
        // Create a new "note log" using the leaf node hash and note.
        let note_log = NoteLog::new(leaf_node.clone(), ctx.accounts.owner.key().clone(), note);
        // Log the "note log" data using noop program
        wrap_application_data_v1(note_log.try_to_vec()?, &ctx.accounts.log_wrapper)?;
        // Get the address for the merkle tree account
        let merkle_tree = ctx.accounts.merkle_tree.key();
        // Define the seeds for pda signing
        let signer_seeds: &[&[&[u8]]] = &[&[
            merkle_tree.as_ref(), // The address of the merkle tree account as a seed
            &[*ctx.bumps.get("tree_authority").unwrap()], // The bump seed for the pda
        ]];
        // Create a new cpi context and append the leaf node to the merkle tree.
        let cpi_ctx = CpiContext::new_with_signer(
            ctx.accounts.compression_program.to_account_info(), // The spl account compression program
            Modify {
                authority: ctx.accounts.tree_authority.to_account_info(), // The authority for the merkle tree, using a PDA
                merkle_tree: ctx.accounts.merkle_tree.to_account_info(), // The merkle tree account to be modified
                noop: ctx.accounts.log_wrapper.to_account_info(), // The noop program to log data
            },
            signer_seeds, // The seeds for pda signing
        );
        // CPI to append the leaf node to the merkle tree
        append(cpi_ctx, leaf_node)?;
        Ok(())
    }

    //...
}
```

## 6. 创建 `update_note` 指令

我们将创建的最后一个指令是 `update_note` 指令。这应该用新的哈希代表新的更新后的笔记数据替换现有的叶子。

为了使其工作，我们将需要以下参数：

1. `index` - 我们将要更新的叶子的索引
2. `root` - Merkle 树的根哈希
3. `old_note` - 我们正在更新的旧笔记的字符串表示形式
4. `new_note` - 我们想要更新为的新笔记的字符串表示形式

请记住，这里的步骤与 `append_note` 类似，但有一些小的添加和修改：

1. 第一步是新的。我们首先需要证明调用此函数的 `owner` 是给定索引处叶子的真实所有者。由于数据被压缩为叶子上的哈希，我们不能简单地将 `owner` 公钥与存储的值进行比较。相反，我们需要使用旧笔记数据和账户验证结构中列出的 `owner` 计算先前的哈希。然后，我们构建并发出 CPI 到状态压缩程序的 `verify_leaf` 指令，使用我们计算的哈希。

2. 这一步与创建 `append_note` 指令的第一步相同。使用 `keccak` crate 中的 `hashv` 函数对新笔记及其所有者进行哈希，每个都作为其相应的字节表示。

3. 这一步与创建 `append_note` 指令的第二步相同。使用步骤 2 中的哈希、所有者的公钥和新笔记作为字符串创建 `NoteLog` 结构体的实例。然后调用 `wrap_application_data_v1` 来向 Noop 程序发出 CPI，传递 `NoteLog` 的实例。

4. 这一步与创建 `append_note` 指令的最后一步略有不同。构建并发出 CPI 到状态压缩程序的 `replace_leaf` 指令。这使用在第 1 步中计算的旧哈希、新哈希和叶子索引，将其添加到 Merkle 树上的下一个可用叶子上。就像以前一样，这需要在签名种子中使用 Merkle 树地址和树授权的增量。

```rust
#[program]
pub mod compressed_notes {
    use super::*;

    //...

		pub fn update_note(
        ctx: Context<NoteAccounts>,
        index: u32,
        root: [u8; 32],
        old_note: String,
        new_note: String,
    ) -> Result<()> {
        let old_leaf =
            keccak::hashv(&[old_note.as_bytes(), ctx.accounts.owner.key().as_ref()]).to_bytes();

        let merkle_tree = ctx.accounts.merkle_tree.key();

        // Define the seeds for pda signing
        let signer_seeds: &[&[&[u8]]] = &[&[
            merkle_tree.as_ref(), // The address of the merkle tree account as a seed
            &[*ctx.bumps.get("tree_authority").unwrap()], // The bump seed for the pda
        ]];

        // Verify Leaf
        {
            if old_note == new_note {
                msg!("Notes are the same!");
                return Ok(());
            }

            let cpi_ctx = CpiContext::new_with_signer(
                ctx.accounts.compression_program.to_account_info(), // The spl account compression program
                VerifyLeaf {
                    merkle_tree: ctx.accounts.merkle_tree.to_account_info(), // The merkle tree account to be modified
                },
                signer_seeds, // The seeds for pda signing
            );
            // Verify or Fails
            verify_leaf(cpi_ctx, root, old_leaf, index)?;
        }

        let new_leaf =
            keccak::hashv(&[new_note.as_bytes(), ctx.accounts.owner.key().as_ref()]).to_bytes();

        // Log out for indexers
        let note_log = NoteLog::new(new_leaf.clone(), ctx.accounts.owner.key().clone(), new_note);
        // Log the "note log" data using noop program
        wrap_application_data_v1(note_log.try_to_vec()?, &ctx.accounts.log_wrapper)?;

        // replace leaf
        {
            let cpi_ctx = CpiContext::new_with_signer(
                ctx.accounts.compression_program.to_account_info(), // The spl account compression program
                Modify {
                    authority: ctx.accounts.tree_authority.to_account_info(), // The authority for the merkle tree, using a PDA
                    merkle_tree: ctx.accounts.merkle_tree.to_account_info(), // The merkle tree account to be modified
                    noop: ctx.accounts.log_wrapper.to_account_info(), // The noop program to log data
                },
                signer_seeds, // The seeds for pda signing
            );
            // CPI to append the leaf node to the merkle tree
            replace_leaf(cpi_ctx, root, old_leaf, new_leaf, index)?;
        }

        Ok(())
    }
}
```

## 7. 客户端测试设置

我们将编写一些测试来确保我们的程序按预期工作。首先，让我们进行一些设置。

我们将使用 `@solana/spl-account-compression` 包。请安装它：

```bash
yarn add @solana/spl-account-compression
```

接下来，我们将给你一个实用工具文件的内容，我们创建了这个文件以便更容易进行测试。在 `tests` 目录下创建一个 `utils.ts` 文件，将下面的内容添加进去，然后我们会解释它。

```tsx
import {
  SPL_NOOP_PROGRAM_ID,
  deserializeApplicationDataEvent,
} from "@solana/spl-account-compression"
import { Connection, PublicKey } from "@solana/web3.js"
import { bs58 } from "@coral-xyz/anchor/dist/cjs/utils/bytes"
import { deserialize } from "borsh"
import { keccak256 } from "js-sha3"

class NoteLog {
  leafNode: Uint8Array
  owner: PublicKey
  note: string

  constructor(properties: {
    leafNode: Uint8Array
    owner: Uint8Array
    note: string
  }) {
    this.leafNode = properties.leafNode
    this.owner = new PublicKey(properties.owner)
    this.note = properties.note
  }
}

// A map that describes the Note structure for Borsh deserialization
const NoteLogBorshSchema = new Map([
  [
    NoteLog,
    {
      kind: "struct",
      fields: [
        ["leafNode", [32]], // Array of 32 `u8`
        ["owner", [32]], // Pubkey
        ["note", "string"],
      ],
    },
  ],
])

export function getHash(note: string, owner: PublicKey) {
  const noteBuffer = Buffer.from(note)
  const publicKeyBuffer = Buffer.from(owner.toBytes())
  const concatenatedBuffer = Buffer.concat([noteBuffer, publicKeyBuffer])
  const concatenatedUint8Array = new Uint8Array(
    concatenatedBuffer.buffer,
    concatenatedBuffer.byteOffset,
    concatenatedBuffer.byteLength
  )
  return keccak256(concatenatedUint8Array)
}

export async function getNoteLog(connection: Connection, txSignature: string) {
  // Confirm the transaction, otherwise the getTransaction sometimes returns null
  const latestBlockHash = await connection.getLatestBlockhash()
  await connection.confirmTransaction({
    blockhash: latestBlockHash.blockhash,
    lastValidBlockHeight: latestBlockHash.lastValidBlockHeight,
    signature: txSignature,
  })

  // Get the transaction info using the tx signature
  const txInfo = await connection.getTransaction(txSignature, {
    maxSupportedTransactionVersion: 0,
  })

  // Get the inner instructions related to the program instruction at index 0
  // We only send one instruction in test transaction, so we can assume the first
  const innerIx = txInfo!.meta?.innerInstructions?.[0]?.instructions

  // Get the inner instructions that match the SPL_NOOP_PROGRAM_ID
  const noopInnerIx = innerIx.filter(
    (instruction) =>
      txInfo?.transaction.message.staticAccountKeys[
        instruction.programIdIndex
      ].toBase58() === SPL_NOOP_PROGRAM_ID.toBase58()
  )

  let noteLog: NoteLog
  for (let i = noopInnerIx.length - 1; i >= 0; i--) {
    try {
      // Try to decode and deserialize the instruction data
      const applicationDataEvent = deserializeApplicationDataEvent(
        Buffer.from(bs58.decode(noopInnerIx[i]?.data!))
      )

      // Get the application data
      const applicationData = applicationDataEvent.fields[0].applicationData

      // Deserialize the application data into NoteLog instance
      noteLog = deserialize(
        NoteLogBorshSchema,
        NoteLog,
        Buffer.from(applicationData)
      )

      if (noteLog !== undefined) {
        break
      }
    } catch (__) {}
  }

  return noteLog
}
```

以上文件主要包含以下三个内容：

1. `NoteLog` - 代表我们在 Noop 程序日志中找到的笔记日志的类。我们还添加了 Borsh 模式作为 `NoteLogBorshSchema` 用于反序列化。
2. `getHash` - 一个函数，用于创建笔记和笔记所有者的哈希值，以便我们可以将其与默克尔树上找到的内容进行比较。
3. `getNoteLog` - 一个函数，用于查找提供的交易日志，找到 Noop 程序日志，然后反序列化并返回相应的笔记日志。

## 8. 编写客户端测试

现在我们已经安装了我们的包并准备好了实用工具文件，让我们深入研究测试本身。我们将创建四个测试：

1. 创建笔记树 - 这将创建我们将用于存储笔记哈希的默克尔树
2. 添加笔记 - 这将调用我们的 `append_note` 指令
3. 添加最大大小的笔记 - 这将调用我们的 `append_note` 指令，并使用一个在单个交易中允许的 1232 字节中达到最大值的笔记
4. 更新第一个笔记 - 这将调用我们的 `update_note` 指令以修改我们添加的第一个笔记

第一个测试主要是为了设置。在最后三个测试中，我们每次都会断言笔记树上的笔记哈希是否与给定的笔记文本和签名者所期望的匹配。

让我们从导入开始。从 Anchor、`@solana/web3.js`、`@solana/spl-account-compression` 和我们自己的实用工具文件中有相当多的导入。

```tsx
import * as anchor from "@coral-xyz/anchor"
import { Program } from "@coral-xyz/anchor"
import { CompressedNotes } from "../target/types/compressed_notes"
import {
  Keypair,
  Transaction,
  PublicKey,
  sendAndConfirmTransaction,
  Connection,
} from "@solana/web3.js"
import {
  ValidDepthSizePair,
  createAllocTreeIx,
  SPL_ACCOUNT_COMPRESSION_PROGRAM_ID,
  SPL_NOOP_PROGRAM_ID,
  ConcurrentMerkleTreeAccount,
} from "@solana/spl-account-compression"
import { getHash, getNoteLog } from "./utils"
import { assert } from "chai"
```

接下来，我们将设置我们在整个测试过程中将使用的状态变量。这包括默认的 Anchor 设置以及生成默克尔树密钥对、树授权和一些笔记。

```tsx
describe("compressed-notes", () => {
  const provider = anchor.AnchorProvider.env()
  anchor.setProvider(provider)
  const connection = new Connection(
    provider.connection.rpcEndpoint,
    "confirmed" // has to be confirmed for some of the methods below
  )

  const wallet = provider.wallet as anchor.Wallet
  const program = anchor.workspace.CompressedNotes as Program<CompressedNotes>

  // Generate a new keypair for the merkle tree account
  const merkleTree = Keypair.generate()

  // Derive the PDA to use as the tree authority for the merkle tree account
  // This is a PDA derived from the Note program, which allows the program to sign for appends instructions to the tree
  const [treeAuthority] = PublicKey.findProgramAddressSync(
    [merkleTree.publicKey.toBuffer()],
    program.programId
  )

	const firstNote = "hello world"
  const secondNote = "0".repeat(917)
  const updatedNote = "updated note"


  // TESTS GO HERE

});
```

最后，让我们从测试本身开始。首先是 `创建笔记树` 测试。该测试将执行两项操作：

1. 为默克尔树分配一个新账户，最大深度为3，最大缓冲区大小为8，梯度深度为0。
2. 使用我们程序的 `createNoteTree` 指令对这个新账户进行初始化。

```tsx
it("Create Note Tree", async () => {
  const maxDepthSizePair: ValidDepthSizePair = {
    maxDepth: 3,
    maxBufferSize: 8,
  }

  const canopyDepth = 0

  // instruction to create new account with required space for tree
  const allocTreeIx = await createAllocTreeIx(
    connection,
    merkleTree.publicKey,
    wallet.publicKey,
    maxDepthSizePair,
    canopyDepth
  )

  // instruction to initialize the tree through the Note program
  const ix = await program.methods
    .createNoteTree(maxDepthSizePair.maxDepth, maxDepthSizePair.maxBufferSize)
    .accounts({
      merkleTree: merkleTree.publicKey,
      treeAuthority: treeAuthority,
      logWrapper: SPL_NOOP_PROGRAM_ID,
      compressionProgram: SPL_ACCOUNT_COMPRESSION_PROGRAM_ID,
    })
    .instruction()

  const tx = new Transaction().add(allocTreeIx, ix)
  await sendAndConfirmTransaction(connection, tx, [wallet.payer, merkleTree])
})
```

接下来，我们将创建 `Add Note` 测试。它应该调用 `append_note` 并传入 `firstNote`，然后检查链上的哈希是否与我们计算的哈希匹配，以及笔记日志是否与我们传入指令的笔记文本匹配。

```tsx
it("Add Note", async () => {
  const txSignature = await program.methods
    .appendNote(firstNote)
    .accounts({
      merkleTree: merkleTree.publicKey,
      treeAuthority: treeAuthority,
      logWrapper: SPL_NOOP_PROGRAM_ID,
      compressionProgram: SPL_ACCOUNT_COMPRESSION_PROGRAM_ID,
    })
    .rpc()
  
  const noteLog = await getNoteLog(connection, txSignature)
  const hash = getHash(firstNote, provider.publicKey)
  
  assert(hash === Buffer.from(noteLog.leafNode).toString("hex"))
  assert(firstNote === noteLog.note)
})
```

接下来，我们将创建 `Add Max Size Note` 测试。与上一个测试相同，但使用的是第二个笔记。

```tsx
it("Add Max Size Note", async () => {
  // Size of note is limited by max transaction size of 1232 bytes, minus additional data required for the instruction
  const txSignature = await program.methods
    .appendNote(secondNote)
    .accounts({
      merkleTree: merkleTree.publicKey,
      treeAuthority: treeAuthority,
      logWrapper: SPL_NOOP_PROGRAM_ID,
      compressionProgram: SPL_ACCOUNT_COMPRESSION_PROGRAM_ID,
    })
    .rpc()
  
  const noteLog = await getNoteLog(connection, txSignature)
  const hash = getHash(secondNote, provider.publicKey)
  
  assert(hash === Buffer.from(noteLog.leafNode).toString("hex"))
  assert(secondNote === noteLog.note)
})
```

最后，我们将创建 `Update First Note` 测试。这比添加一个笔记稍微复杂一些。我们将执行以下步骤：

1. 获取 Merkle 树的根，因为指令需要它。
2. 调用程序的 `update_note` 指令，传入索引 0（对应第一个笔记）、Merkle 树的根、第一个笔记以及更新的数据。请记住，它需要第一个笔记和根，因为程序必须在更新之前验证笔记叶子的完整证明路径。

```tsx
it("Update First Note", async () => {
  const merkleTreeAccount =
    await ConcurrentMerkleTreeAccount.fromAccountAddress(
      connection,
      merkleTree.publicKey
    )
  
  const rootKey = merkleTreeAccount.tree.changeLogs[0].root
  const root = Array.from(rootKey.toBuffer())

  const txSignature = await program.methods
    .updateNote(0, root, firstNote, updatedNote)
    .accounts({
      merkleTree: merkleTree.publicKey,
      treeAuthority: treeAuthority,
      logWrapper: SPL_NOOP_PROGRAM_ID,
      compressionProgram: SPL_ACCOUNT_COMPRESSION_PROGRAM_ID,
    })
    .rpc()
  
  const noteLog = await getNoteLog(connection, txSignature)
  const hash = getHash(updatedNote, provider.publicKey)
  
  assert(hash === Buffer.from(noteLog.leafNode).toString("hex"))
  assert(updatedNote === noteLog.note)
})
```

就是这样，恭喜！继续运行 `anchor test`，你应该会得到四个通过的测试。

如果你遇到了问题，可以随时回顾一些演示或查看 [Compressed Notes 仓库](https://github.com/unboxed-software/anchor-compressed-notes) 中的完整解决方案代码。

# 挑战

现在你已经练习了状态压缩的基础知识，那么请给 Compressed Notes 程序添加一个新指令。这个新指令应该允许用户删除一个已存在的笔记。请记住，你不能从树中移除一个叶子节点，所以你需要决定在你的程序中“删除”是什么样子的。祝你好运！

如果你想要一个非常简单的删除函数示例，请查看 GitHub 上的 [`solution` 分支](https://github.com/Unboxed-Software/anchor-compressed-notes/tree/solution)。

## 完成了实验吗？

将你的代码推送到 GitHub，并[告诉我们你对这节课的看法](https://form.typeform.com/to/IPH0UGz7#answers-lesson=60f6b072-eaeb-469c-b32e-5fea4b72d1d1)！