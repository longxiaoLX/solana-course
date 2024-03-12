---
title: 版本化交易和查找表
objectives:
- 创建版本化交易
- 创建查找表
- 扩展查找表
- 在版本化交易中使用查找表
---

# 总结

- **版本化交易（Versioned Transactions）** 是支持遗留版本和新版本交易格式的一种方式。原始交易格式被称为“遗留（legacy）”，而新的交易版本从版本 0 开始。版本化交易的实施是为了支持地址查找表（Address Lookup Tables，也称为查找表，lookup tables 或 LUT）的使用。
- **地址查找表** 是用于存储其他账户地址的账户，然后可以在版本化交易中使用 1 字节索引而不是每个地址的完整 32 字节进行引用。这使得可以创建比引入 LUT 之前更复杂的交易。

# 概述

按设计，Solana 交易限制为 1232 字节。超过此大小的交易将失败。虽然这样做可以实现许多网络优化，但也会限制网络上可以执行的原子操作类型。

为了帮助解决交易大小限制，Solana 发布了一种新的交易格式，允许支持多个版本的交易格式。在撰写本文时，Solana 支持两个交易版本：

1. `legacy` - 原始交易格式
2. `0` - 包括支持地址查找表的最新交易格式

版本化交易不需要对现有的 Solana 程序进行任何修改，但在版本化交易发布之前创建的任何客户端代码都应进行更新。在本课程中，我们将介绍版本化交易的基础知识以及如何使用它们，包括：

- 创建版本化交易
- 创建和管理查找表
- 在版本化交易中使用查找表

## 版本化交易

Solana 交易中占用最多空间的元素之一是包含完整账户地址。每个地址占用 32 字节，39 个账户将使交易过大。这甚至还没有考虑指令数据。实际上，大多数交易在包含约 20 个账户时就会过大。

Solana 发布了版本化交易，以支持多个交易格式。随着版本化交易的发布，Solana 还发布了版本 0 的交易，以支持地址查找表。查找表是单独的账户，用于存储账户地址，然后允许在交易中使用 1 字节索引来引用它们。这显著减小了交易的大小，因为每个包含的账户现在只需要使用 1 字节而不是 32 字节。

即使您不需要使用查找表，您也需要知道如何在客户端代码中支持版本化交易。幸运的是，您需要处理版本化交易和查找表的所有内容都包含在 `@solana/web3.js` 库中。

### 创建版本化交易

要创建版本化交易，只需使用以下参数创建一个 `TransactionMessage`：

- `payerKey` - 将支付交易费用的账户的公钥
- `recentBlockhash` - 来自网络的最近的区块哈希
- `instructions` - 要包含在交易中的指令

然后，您可以使用 `compileToV0Message()` 方法将此消息对象转换为版本 `0` 的交易。

```typescript
import * as web3 from "@solana/web3.js";

// Example transfer instruction
const transferInstruction = [
    web3.SystemProgram.transfer({
        fromPubkey: payer.publicKey, // Public key of account that will send the funds
        toPubkey: toAccount.publicKey, // Public key of the account that will receive the funds
        lamports: 1 * LAMPORTS_PER_SOL, // Amount of lamports to be transferred
    }),
];

// Get the latest blockhash
let { blockhash } = await connection.getLatestBlockhash();

// Create the transaction message
const message = new web3.TransactionMessage({
    payerKey: payer.publicKey, // Public key of the account that will pay for the transaction
    recentBlockhash: blockhash, // Latest blockhash
    instructions: transferInstruction, // Instructions included in transaction
}).compileToV0Message();
```

最后，您将编译后的消息传递给 `VersionedTransaction` 构造函数，以创建一个新的版本化交易。然后，您的代码可以签名并将交易发送到网络，类似于传统的交易。

```typescript
// Create the versioned transaction using the message
const transaction = new web3.VersionedTransaction(message);

// Sign the transaction
transaction.sign([payer]);

// Send the signed transaction to the network
const transactionSignature = await connection.sendTransaction(transaction);
```

## 地址查找表

地址查找表（也称为查找表或 LUT）是存储其他账户地址查找表的账户。这些 LUT 账户归地址查找表程序（Address Lookup Table Program）所有，用于增加可以包含在单个交易中的账户数量。

版本化交易可以包含 LUT 账户的地址，然后使用 1 字节索引引用其他账户，而不是包含这些账户的完整地址。这显著减少了交易中用于引用账户的空间量。

为了简化使用 LUT 的过程，`@solana/web3.js` 库包含一个 `AddressLookupTableProgram` 类，该类提供了一组方法来创建管理 LUT 的指令。这些方法包括：

- `createLookupTable` - 创建一个新的 LUT 账户
- `freezeLookupTable` - 使现有的 LUT 无法更改
- `extendLookupTable` - 将地址添加到现有的 LUT 中
- `deactivateLookupTable` - 将 LUT 放入“停用（deactivation）”期，然后可以关闭它
- `closeLookupTable` - 永久关闭一个 LUT 账户

### 创建查找表

您可以使用 `createLookupTable` 方法构建创建查找表的指令。该函数需要以下参数：

- `authority` - 有权修改查找表的账户
- `payer` - 将支付账户创建费用的账户
- `recentSlot` - 用于派生查找表地址的最近 slot

该函数返回创建查找表的指令以及查找表的地址。

```typescript
// Get the current slot
const slot = await connection.getSlot();

// Create an instruction for creating a lookup table
// and retrieve the address of the new lookup table
const [lookupTableInst, lookupTableAddress] =
    web3.AddressLookupTableProgram.createLookupTable({
        authority: user.publicKey, // The authority (i.e., the account with permission to modify the lookup table)
        payer: user.publicKey, // The payer (i.e., the account that will pay for the transaction fees)
        recentSlot: slot - 1, // The recent slot to derive lookup table's address
    });
```

在底层，查找表地址只是使用 `authority` 和 `recentSlot` 作为种子派生的 PDA。

```typescript
const [lookupTableAddress, bumpSeed] = PublicKey.findProgramAddressSync(
    [params.authority.toBuffer(), toBufferLE(BigInt(params.recentSlot), 8)],
    this.programId,
);
```

请注意，有时在发送交易后，使用最近的 slot 会导致错误。为了避免这种情况，您可以使用比最近 slot 早一个的 slot（例如 `recentSlot: slot - 1`）。然而，如果在发送交易时仍然遇到错误，您可以尝试重新发送交易。

```
"Program AddressLookupTab1e1111111111111111111111111 invoke [1]",
"188115589 is not a recent slot",
"Program AddressLookupTab1e1111111111111111111111111 failed: invalid instruction data";
```

### 扩展查找表

您可以使用 `extendLookupTable` 方法创建一个指令，以向现有的查找表添加地址。它接受以下参数：

- `payer` - 将支付交易费用和任何增加的租金的账户
- `authority` - 有权更改查找表的账户
- `lookupTable` - 要扩展的查找表的地址
- `addresses` - 要添加到查找表的地址

该函数返回一个扩展查找表的指令。

```typescript
const addresses = [
    new web3.PublicKey("31Jy3nFeb5hKVdB4GS4Y7MhU7zhNMFxwF7RGVhPc1TzR"),
    new web3.PublicKey("HKSeapcvwJ7ri6mf3HwBtspLFTDKqaJrMsozdfXfg5y2"),
    // add more addresses
];

// Create an instruction to extend a lookup table with the provided addresses
const extendInstruction = web3.AddressLookupTableProgram.extendLookupTable({
    payer: user.publicKey, // The payer (i.e., the account that will pay for the transaction fees)
    authority: user.publicKey, // The authority (i.e., the account with permission to modify the lookup table)
    lookupTable: lookupTableAddress, // The address of the lookup table to extend
    addresses: addresses, // The addresses to add to the lookup table
});
```

请注意，当扩展查找表时，可以在一条指令中添加的地址数量受到交易大小限制的限制，即 1232 字节。这意味着您一次可以向查找表添加 30 个地址。如果需要添加的地址数量超过这个限制，您将需要发送多个交易。每个查找表最多可以存储 256 个地址。

### 发送交易

在创建指令之后，您可以将它们添加到一个交易中并发送到网络。

```typescript
// Get the latest blockhash
let { blockhash } = await connection.getLatestBlockhash();

// Create the transaction message
const message = new web3.TransactionMessage({
    payerKey: payer.publicKey, // Public key of the account that will pay for the transaction
    recentBlockhash: blockhash, // Latest blockhash
    instructions: [lookupTableInst, extendInstruction], // Instructions included in transaction
}).compileToV0Message();

// Create the versioned transaction using the message
const transaction = new web3.VersionedTransaction(message);

// Sign the transaction
transaction.sign([payer]);

// Send the signed transaction to the network
const transactionSignature = await connection.sendTransaction(transaction);
```

请注意，当您首次创建或扩展查找表时，或者在此之后，需要等待一个 slot 以使查找表或新地址可以在交易中使用。换句话说，您只能使用和访问在当前 slot 之前添加的查找表和其地址。

```typescript
SendTransactionError: failed to send transaction: invalid transaction: Transaction address table lookup uses an invalid index
```

如果您遇到上述错误或在扩展查找表后无法立即访问查找表中的地址，很可能是因为您尝试在预热期结束之前访问查找表或特定地址。为了避免此问题，在扩展查找表后，在引用该表的交易发送之前添加延迟。

### 停用查找表

当不再需要查找表时，您可以停用（deactivate）并关闭它以回收其租金余额。可以随时停用地址查找表，但它们可以继续被交易使用，直到指定的“停用” slot 不再“最近（recent）”。这个“冷却”期确保了在运行中的交易不能被查找表关闭并在同一 slot 中重新创建。停用期约为 513 个 slots。

要停用 LUT，请使用 `deactivateLookupTable` 方法，并传入以下参数：

- `lookupTable` - 要停用的 LUT 的地址
- `authority` - 有权限停用 LUT 的账户

```typescript
const deactivateInstruction =
    web3.AddressLookupTableProgram.deactivateLookupTable({
        lookupTable: lookupTableAddress, // The address of the lookup table to deactivate
        authority: user.publicKey, // The authority (i.e., the account with permission to modify the lookup table)
    });
```

### 关闭查找表

要在其停用期后关闭查找表，请使用 `closeLookupTable` 方法。该方法创建了一个指令，用于关闭已停用的查找表并收回其租金余额。它接受以下参数：

- `lookupTable` - 要关闭的 LUT 的地址
- `authority` - 具有关闭 LUT 权限的账户
- `recipient` - 将接收被收回的租金余额的账户

```typescript
const closeInstruction = web3.AddressLookupTableProgram.closeLookupTable({
    lookupTable: lookupTableAddress, // The address of the lookup table to close
    authority: user.publicKey, // The authority (i.e., the account with permission to modify the lookup table)
    recipient: user.publicKey, // The recipient of closed account lamports
});
```

尝试在查找表完全停用之前关闭它将导致错误。

```
"Program AddressLookupTab1e1111111111111111111111111 invoke [1]",
"Table cannot be closed until it's fully deactivated in 513 blocks",
"Program AddressLookupTab1e1111111111111111111111111 failed: invalid program argument";
```

### 冻结查找表

除了标准的增删改查操作之外，您还可以将查找表“冻结（freeze）”。这将使其成为不可变的，因此它不再能够被扩展、停用或关闭。

您可以使用 `freezeLookupTable` 方法来冻结查找表。它接受以下参数：

- `lookupTable` - 要冻结的 LUT 的地址
- `authority` - 具有冻结 LUT 权限的账户

```typescript
const freezeInstruction = web3.AddressLookupTableProgram.freezeLookupTable({
    lookupTable: lookupTableAddress, // The address of the lookup table to freeze
    authority: user.publicKey, // The authority (i.e., the account with permission to modify the lookup table)
});
```

一旦 LUT 被冻结，任何进一步尝试修改它的操作将导致错误。

```
"Program AddressLookupTab1e1111111111111111111111111 invoke [1]",
"Lookup table is frozen",
"Program AddressLookupTab1e1111111111111111111111111 failed: Account is immutable";
```

### 在版本化交易中使用查找表

要在版本化交易中使用查找表，您需要使用其地址检索查找表账户。

```typescript
const lookupTableAccount = (
    await connection.getAddressLookupTable(lookupTableAddress)
).value;
```

然后，您可以像平常一样将一系列指令包含到交易中。在创建 `TransactionMessage` 时，您可以通过将它们作为数组传递给 `compileToV0Message()` 方法来包含任何查找表账户。您还可以提供多个查找表账户。

```typescript
const message = new web3.TransactionMessage({
    payerKey: payer.publicKey, // The payer (i.e., the account that will pay for the transaction fees)
    recentBlockhash: blockhash, // The blockhash of the most recent block
    instructions: instructions, // The instructions to include in the transaction
}).compileToV0Message([lookupTableAccount]); // Include lookup table accounts

// Create the versioned transaction using the message
const transaction = new web3.VersionedTransaction(message);

// Sign the transaction
transaction.sign([payer]);

// Send the signed transaction to the network
const transactionSignature = await connection.sendTransaction(transaction);
```

# 实验

让我们继续练习使用查找表！

这个实验将引导您完成创建、扩展和在版本化交易中使用查找表的步骤。

## 1. 获取初始代码

首先，从此[存储库](https://github.com/Unboxed-Software/solana-versioned-transactions/tree/starter)的起始分支下载初始代码。一旦您获得了初始代码，请在终端中运行 `npm install` 来安装所需的依赖项。

初始代码包含一个示例，创建一个原始交易（legacy transaction），意图将 SOL 原子地转移到 22 个接收者手中。该交易包含 22 条指令，每条指令将 SOL 从签名者转移到不同的接收者手中。

初始代码的目的是说明原始交易中可以包含的地址数量的限制。在初始代码中构建的交易预计在发送时会失败。

以下初始代码可以在 `index.ts` 文件中找到。

```typescript
import { initializeKeypair } from "./initializeKeypair";
import * as web3 from "@solana/web3.js";

async function main() {
    // Connect to the devnet cluster
    const connection = new web3.Connection(web3.clusterApiUrl("devnet"));

    // Initialize the user's keypair
    const user = await initializeKeypair(connection);
    console.log("PublicKey:", user.publicKey.toBase58());

    // Generate 22 addresses
    const recipients = [];
    for (let i = 0; i < 22; i++) {
        recipients.push(web3.Keypair.generate().publicKey);
    }

    // Create an array of transfer instructions
    const transferInstructions = [];

    // Add a transfer instruction for each address
    for (const address of recipients) {
        transferInstructions.push(
            web3.SystemProgram.transfer({
                fromPubkey: user.publicKey, // The payer (i.e., the account that will pay for the transaction fees)
                toPubkey: address, // The destination account for the transfer
                lamports: web3.LAMPORTS_PER_SOL * 0.01, // The amount of lamports to transfer
            }),
        );
    }

    // Create a transaction and add the transfer instructions
    const transaction = new web3.Transaction().add(...transferInstructions);

    // Send the transaction to the cluster (this will fail in this example if addresses > 21)
    const txid = await connection.sendTransaction(transaction, [user]);

    // Get the latest blockhash and last valid block height
    const { lastValidBlockHeight, blockhash } =
        await connection.getLatestBlockhash();

    // Confirm the transaction
    await connection.confirmTransaction({
        blockhash: blockhash,
        lastValidBlockHeight: lastValidBlockHeight,
        signature: txid,
    });

    // Log the transaction URL on the Solana Explorer
    console.log(`https://explorer.solana.com/tx/${txid}?cluster=devnet`);
}
```

要执行代码，请运行 `npm start`。这将创建一个新的密钥对，将其写入 `.env` 文件，将 devnet SOL 空投到该密钥对，并发送在初始代码中构建的交易。预计该交易将失败，并显示错误消息 `Transaction too large`。

```
Creating .env file
Current balance is 0
Airdropping 1 SOL...
New balance is 1
PublicKey: 5ZZzcDbabFHmoZU8vm3VzRzN5sSQhkf91VJzHAJGNM7B
Error: Transaction too large: 1244 > 1232
```

在接下来的步骤中，我们将介绍如何使用带有版本化交易的查找表来增加可以包含在单个交易中的地址数量。

在开始之前，请继续删除 `main` 函数的内容，只留下以下内容：

```typescript
async function main() {
    // Connect to the devnet cluster
    const connection = new web3.Connection(web3.clusterApiUrl("devnet"));

    // Initialize the user's keypair
    const user = await initializeKeypair(connection);
    console.log("PublicKey:", user.publicKey.toBase58());

    // Generate 22 addresses
    const addresses = [];
    for (let i = 0; i < 22; i++) {
        addresses.push(web3.Keypair.generate().publicKey);
    }
}
```

## 2. 创建 `sendV0Transaction` 辅助函数

我们将发送多个“版本 0（version 0）”交易，因此让我们创建一个辅助函数来简化此过程。

该函数应该接受连接、用户的密钥对、交易指令数组以及可选的查找表账户数组作为参数。

然后，该函数执行以下任务：

-   从 Solana 网络获取最新的区块哈希和最后一个有效的区块高度
-   使用提供的指令创建一个新的事务消息（transaction message）
-   使用用户的密钥对对交易进行签名
-   将交易发送到 Solana 网络
-   确认交易
-   在 Solana Explorer 上记录交易的 URL

```typescript
async function sendV0Transaction(
    connection: web3.Connection,
    user: web3.Keypair,
    instructions: web3.TransactionInstruction[],
    lookupTableAccounts?: web3.AddressLookupTableAccount[],
) {
    // Get the latest blockhash and last valid block height
    const { lastValidBlockHeight, blockhash } =
        await connection.getLatestBlockhash();

    // Create a new transaction message with the provided instructions
    const messageV0 = new web3.TransactionMessage({
        payerKey: user.publicKey, // The payer (i.e., the account that will pay for the transaction fees)
        recentBlockhash: blockhash, // The blockhash of the most recent block
        instructions, // The instructions to include in the transaction
    }).compileToV0Message(
        lookupTableAccounts ? lookupTableAccounts : undefined,
    );

    // Create a new transaction object with the message
    const transaction = new web3.VersionedTransaction(messageV0);

    // Sign the transaction with the user's keypair
    transaction.sign([user]);

    // Send the transaction to the cluster
    const txid = await connection.sendTransaction(transaction);

    // Confirm the transaction
    await connection.confirmTransaction(
        {
            blockhash: blockhash,
            lastValidBlockHeight: lastValidBlockHeight,
            signature: txid,
        },
        "finalized",
    );

    // Log the transaction URL on the Solana Explorer
    console.log(`https://explorer.solana.com/tx/${txid}?cluster=devnet`);
}
```

## 3. 创建 `waitForNewBlock` 辅助函数

回想一下，查找表及其中包含的地址在创建或扩展后无法立即引用。这意味着我们需要等待新的区块生成，然后再提交引用新创建或扩展查找表的交易。为了简化后续的操作，让我们创建一个 `waitForNewBlock` 辅助函数，在发送交易之间用于等待查找表激活。

该函数将接受连接和目标区块高度作为参数。然后，它会启动一个间隔，每 1000 毫秒检查一次网络的当前区块高度。一旦新的区块高度超过目标高度，间隔将被清除，并且承诺将被解析。

```typescript
function waitForNewBlock(connection: web3.Connection, targetHeight: number) {
    console.log(`Waiting for ${targetHeight} new blocks`);
    return new Promise(async (resolve: any) => {
        // Get the last valid block height of the blockchain
        const { lastValidBlockHeight } = await connection.getLatestBlockhash();

        // Set an interval to check for new blocks every 1000ms
        const intervalId = setInterval(async () => {
            // Get the new valid block height
            const { lastValidBlockHeight: newValidBlockHeight } =
                await connection.getLatestBlockhash();
            // console.log(newValidBlockHeight)

            // Check if the new valid block height is greater than the target block height
            if (newValidBlockHeight > lastValidBlockHeight + targetHeight) {
                // If the target block height is reached, clear the interval and resolve the promise
                clearInterval(intervalId);
                resolve();
            }
        }, 1000);
    });
}
```

## 4. 创建一个名为 `initializeLookupTable` 的函数

现在我们已经准备好一些辅助函数，声明一个名为 `initializeLookupTable` 的函数。该函数具有参数 `user`、`connection` 和 `addresses`。该函数将：

1. 获取当前的 slot
2. 生成一个创建查找表的指令
3. 生成一个将提供的地址扩展（extending）到查找表的指令
4. 发送并确认带有创建和扩展查找表指令的交易
5. 返回查找表的地址

```typescript
async function initializeLookupTable(
    user: web3.Keypair,
    connection: web3.Connection,
    addresses: web3.PublicKey[],
): Promise<web3.PublicKey> {
    // Get the current slot
    const slot = await connection.getSlot();

    // Create an instruction for creating a lookup table
    // and retrieve the address of the new lookup table
    const [lookupTableInst, lookupTableAddress] =
        web3.AddressLookupTableProgram.createLookupTable({
            authority: user.publicKey, // The authority (i.e., the account with permission to modify the lookup table)
            payer: user.publicKey, // The payer (i.e., the account that will pay for the transaction fees)
            recentSlot: slot - 1, // The recent slot to derive lookup table's address
        });
    console.log("lookup table address:", lookupTableAddress.toBase58());

    // Create an instruction to extend a lookup table with the provided addresses
    const extendInstruction = web3.AddressLookupTableProgram.extendLookupTable({
        payer: user.publicKey, // The payer (i.e., the account that will pay for the transaction fees)
        authority: user.publicKey, // The authority (i.e., the account with permission to modify the lookup table)
        lookupTable: lookupTableAddress, // The address of the lookup table to extend
        addresses: addresses.slice(0, 30), // The addresses to add to the lookup table
    });

    await sendV0Transaction(connection, user, [
        lookupTableInst,
        extendInstruction,
    ]);

    return lookupTableAddress;
}
```

## 5. 修改 `main` 以使用查找表

现在我们可以使用所有接收者地址初始化查找表，让我们更新 `main` 以使用版本化交易和查找表。我们需要：

1. 调用 `initializeLookupTable`
2. 调用 `waitForNewBlock`
3. 使用 `connection.getAddressLookupTable` 获取查找表
4. 为每个收件人创建转账指令
5. 发送带有所有转账指令的 v0 交易

```typescript
async function main() {
    // Connect to the devnet cluster
    const connection = new web3.Connection(web3.clusterApiUrl("devnet"));

    // Initialize the user's keypair
    const user = await initializeKeypair(connection);
    console.log("PublicKey:", user.publicKey.toBase58());

    // Generate 22 addresses
    const recipients = [];
    for (let i = 0; i < 22; i++) {
        recipients.push(web3.Keypair.generate().publicKey);
    }

    const lookupTableAddress = await initializeLookupTable(
        user,
        connection,
        recipients,
    );

    await waitForNewBlock(connection, 1);

    const lookupTableAccount = (
        await connection.getAddressLookupTable(lookupTableAddress)
    ).value;

    if (!lookupTableAccount) {
        throw new Error("Lookup table not found");
    }

    const transferInstructions = recipients.map((recipient) => {
        return web3.SystemProgram.transfer({
            fromPubkey: user.publicKey, // The payer (i.e., the account that will pay for the transaction fees)
            toPubkey: recipient, // The destination account for the transfer
            lamports: web3.LAMPORTS_PER_SOL * 0.01, // The amount of lamports to transfer
        });
    });

    await sendV0Transaction(connection, user, transferInstructions, [
        lookupTableAccount,
    ]);
}
```

请注意，尽管我们创建了查找表，但您仍然使用完整的接收人地址创建转账指令。这是因为通过在版本化交易中包含查找表，您告诉 `web3.js` 框架将与查找表中的地址匹配的任何收件人地址替换为指向查找表的指针。在交易发送到网络时，存在于查找表中的地址将由单个字节引用，而不是完整的32字节。

在命令行中使用 `npm start` 来执行 `main` 函数。您应该会看到类似以下的输出：

```bash
Current balance is 1.38866636
PublicKey: 8iGVBt3dcJdp9KfyTRcKuHY6gXCMFdnSG2F1pAwsUTMX
lookup table address: Cc46Wp1mtci3Jm9EcH35JcDQS3rLKBWzy9mV1Kkjjw7M
https://explorer.solana.com/tx/4JvCo2azy2u8XK2pU8AnJiHAucKTrZ6QX7EEHVuNSED8B5A8t9GqY5CP9xB8fZpTNuR7tbUcnj2MiL41xRJnLGzV?cluster=devnet
Waiting for 1 new blocks
https://explorer.solana.com/tx/rgpmxGU4QaAXw9eyqfMUqv8Lp6LHTuTyjQqDXpeFcu1ijQMmCH2V3Sb54x2wWAbnWXnMpJNGg4eLvuy3r8izGHt?cluster=devnet
Finished successfully
```

控制台中的第一个交易链接表示用于创建和扩展查找表的交易。第二个交易表示向所有接收人的转账。可以随意在区块链浏览器中检查这些交易。

请记住，在您首次下载起始代码时，此相同交易是失败的。现在我们使用查找表，可以在单个交易中执行所有 22 次转账。

## 6. 向查找表添加更多地址

请记住，到目前为止我们提出的解决方案只支持向多达 30 个帐户进行转账，因为我们只扩展了一次查找表。考虑到转账指令的大小，实际上可以通过额外添加 27 个地址来扩展查找表，并完成对多达 57 个接收者的原子转账。现在让我们继续添加对此的支持！

我们只需进入 `initializeLookupTable` 并执行两项操作：

1. 修改现有的调用 `extendLookupTable` 的代码，仅添加前30个地址（超过这个数量会导致交易过大）
2. 添加一个循环，每次扩展 30 个地址的查找表，直到所有地址都被添加完毕。

```typescript
async function initializeLookupTable(
    user: web3.Keypair,
    connection: web3.Connection,
    addresses: web3.PublicKey[],
): Promise<web3.PublicKey> {
    // Get the current slot
    const slot = await connection.getSlot();

    // Create an instruction for creating a lookup table
    // and retrieve the address of the new lookup table
    const [lookupTableInst, lookupTableAddress] =
        web3.AddressLookupTableProgram.createLookupTable({
            authority: user.publicKey, // The authority (i.e., the account with permission to modify the lookup table)
            payer: user.publicKey, // The payer (i.e., the account that will pay for the transaction fees)
            recentSlot: slot - 1, // The recent slot to derive lookup table's address
        });
    console.log("lookup table address:", lookupTableAddress.toBase58());

    // Create an instruction to extend a lookup table with the provided addresses
    const extendInstruction = web3.AddressLookupTableProgram.extendLookupTable({
        payer: user.publicKey, // The payer (i.e., the account that will pay for the transaction fees)
        authority: user.publicKey, // The authority (i.e., the account with permission to modify the lookup table)
        lookupTable: lookupTableAddress, // The address of the lookup table to extend
        addresses: addresses.slice(0, 30), // The addresses to add to the lookup table
    });

    await sendV0Transaction(connection, user, [
        lookupTableInst,
        extendInstruction,
    ]);

    var remaining = addresses.slice(30);

    while (remaining.length > 0) {
        const toAdd = remaining.slice(0, 30);
        remaining = remaining.slice(30);
        const extendInstruction =
            web3.AddressLookupTableProgram.extendLookupTable({
                payer: user.publicKey, // The payer (i.e., the account that will pay for the transaction fees)
                authority: user.publicKey, // The authority (i.e., the account with permission to modify the lookup table)
                lookupTable: lookupTableAddress, // The address of the lookup table to extend
                addresses: toAdd, // The addresses to add to the lookup table
            });

        await sendV0Transaction(connection, user, [extendInstruction]);
    }

    return lookupTableAddress;
}
```

恭喜！如果您对这个实验感觉良好，那么您可能已经准备好独立使用查找表和版本化交易了。如果您想查看最终的解决方案代码，您可以在[解决方案分支上找到它](https://github.com/Unboxed-Software/solana-versioned-transactions/tree/solution)。

# 挑战

作为挑战，尝试对查找表进行停用（deactivating）、关闭（closing）和冻结（freezing）操作。请记住，您需要等待查找表完成停用操作，然后才能关闭它。另外，如果一个查找表被冻结，它就不能被修改（停用或关闭），因此您需要分别进行测试或使用不同的查找表。

1. 创建一个用于停用查找表的函数。
2. 创建一个用于关闭查找表的函数。
3. 创建一个用于冻结查找表的函数。
4. 在 `main()` 函数中调用这些函数来测试它们。

您可以重用我们在实验中创建的用于发送事务和等待查找表激活/停用的函数。随意参考这个[解决方案代码](https://github.com/Unboxed-Software/versioned-transaction/tree/challenge)。

## 实验完成了吗？

将您的代码推送到 GitHub，并[告诉我们您对这节课的看法](https://form.typeform.com/to/IPH0UGz7#answers-lesson=b58fdd00-2b23-4e0d-be55-e62677d351ef)!