---
title: 使用 Token 程序创建代币
objectives:
- 创建铸币厂
- 创建代币账户
- 铸造代币
- 转移代币
- 销毁代币
---

# 总结

- **SPL-Tokens** 代表 Solana 网络上的所有非原生代币。Solana 上的同质化和非同质化代币（NFT）都是 SPL-Tokens。
- **Token Program** 包含了创建和与 SPL-Tokens 交互的指令。
- **Token Mints（铸币厂）** 是保存关于特定代币的数据的账户，但不保存代币本身。
- **Token Accounts（代币账户）** 用于保存特定铸币厂的代币。
- 创建铸币厂和代币账户需要在 SOL 中分配**租金（rent）**。当账户关闭时，代币账户的租金可以退还，但目前铸币厂无法关闭。

# 概述

Token 程序（Token Program）是 Solana 程序库（Solana Program Library，SPL）提供的众多程序之一。它包含了创建和与 SPL-Tokens 交互的指令。这些代币代表了 Solana 网络上的所有非原生（即非 SOL）代币。

本课程将重点介绍使用 Token 程序创建和管理新的 SPL-Token 的基础知识：
1. 创建新的铸币厂
2. 创建代币账户
3. 铸造
4. 将代币从一个持有者转移到另一个持有者
5. 销毁代币

我们将在客户端使用 `@solana/spl-token` Javascript 库来进行开发和讨论。

## 铸币厂

要创建一个新的 SPL-Token，首先必须创建一个铸币厂（Token Mint）。铸币厂是保存关于特定代币数据的账户。

举个例子，让我们看一下[在 Solana Explorer 上的 USD Coin (USDC)](https://explorer.solana.com/address/EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v)。USDC 的铸币厂地址是 `EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v`。通过浏览器，我们可以查看有关 USDC 的铸币厂的特定详细信息，例如代币的当前供应量、铸造和冻结权限的地址以及代币的小数精度：

![USDC 铸币厂的截图](../../assets/token-program-usdc-mint.png)

要创建一个新的铸币厂，您需要向 Token 程序发送正确的交易指令。为此，我们将使用 `@solana/spl-token` 中的 `createMint` 函数。

```tsx
const tokenMint = await createMint(
  connection,
  payer,
  mintAuthority,
  freezeAuthority,
  decimal
);
```

`createMint` 函数返回新铸币厂的 `publicKey`。该函数需要以下参数：

- `connection` - 连接到集群的 JSON-RPC 连接
- `payer` - 交易的付款人的公钥
- `mintAuthority` - 授权执行从铸币厂中实际铸造代币的账户。
- `freezeAuthority` - 授权冻结功能的账户，可以冻结相关代币的代币账户。如果不需要冻结功能，则可以将参数设置为 null。
- `decimals` - 指定代币的所需小数精度。

当从具有访问您的私钥的脚本中创建新的铸币厂时，您可以简单地使用 `createMint` 函数。然而，如果您要构建一个网站，允许用户创建新的铸币厂，您需要在不让用户暴露其私钥给浏览器的情况下执行此操作。在这种情况下，您需要构建并提交一个包含正确指令的交易。

在底层，`createMint` 函数实际上是创建一个包含两个指令的交易：
1. 创建一个新的账户
2. 初始化一个新的铸币厂

这看起来如下所示：

```tsx
import * as web3 from '@solana/web3'
import * as token from '@solana/spl-token'

async function buildCreateMintTransaction(
  connection: web3.Connection,
  payer: web3.PublicKey,
  decimals: number
): Promise<web3.Transaction> {
  const lamports = await token.getMinimumBalanceForRentExemptMint(connection);
  const accountKeypair = web3.Keypair.generate();
  const programId = token.TOKEN_PROGRAM_ID

  const transaction = new web3.Transaction().add(
    web3.SystemProgram.createAccount({
      fromPubkey: payer,
      newAccountPubkey: accountKeypair.publicKey,
      space: token.MINT_SIZE,
      lamports,
      programId,
    }),
    token.createInitializeMintInstruction(
      accountKeypair.publicKey,
      decimals,
      payer,
      payer,
      programId
    )
  );

  return transaction
}
```

当手动构建指令来创建一个新的铸币厂时，请确保将创建账户和初始化铸币厂的指令添加到*同一笔交易*中。如果您将每个步骤分别放在单独的交易中，理论上其他人可能会获取您创建的账户并为其自己的铸造进行初始化。

### 租金和免租金
注意，在上一个代码片段的函数体的第一行中包含了对 `getMinimumBalanceForRentExemptMint` 的调用，其结果被传递到 `createAccount` 函数中。这是账户初始化的一部分，称为免租金（rent exemption）。

直到最近，Solana 上的所有账户都需要做以下其中一项以避免被回收：
1. 在特定时间间隔支付租金
2. 在初始化时存入足够的 SOL 以被视为免租金

最近，第一种选项被取消，而变为在初始化新账户时必须存入足够的 SOL 以免租金。

在这种情况下，我们正在为铸币厂创建一个新账户，因此我们使用 `@solana/spl-token` 库中的 `getMinimumBalanceForRentExemptMint`。然而，这个概念适用于所有账户，对于您可能需要创建的其他账户，您可以在 `Connection` 上使用更通用的 `getMinimumBalanceForRentExemption` 方法。

## 代币账户

在您可以铸造代币（发行新供应量）之前，您需要一个代币账户（token account）来持有新发行的代币。

代币账户持有特定“铸币厂”的代币，并有指定该账户的“所有者（owner）”。只有所有者才能授权减少代币账户余额（转账、销毁等），而任何人都可以向代币账户发送代币以增加其余额。

您可以使用 `spl-token` 库的 `createAccount` 函数来创建新的代币账户：

```tsx
const tokenAccount = await createAccount(
  connection,
  payer,
  mint,
  owner,
  keypair
);
```

`createAccount` 函数返回新代币账户的 `publicKey`。该函数需要以下参数：

- `connection` - 连接到集群的 JSON-RPC 连接
- `payer` - 交易的付款人账户
- `mint` - 新代币账户关联的铸币厂
- `owner` - 新代币账户的所有者账户
- `keypair` - 这是一个可选参数，用于指定新代币账户的地址。如果没有提供 keypair，`createAccount` 函数将默认从关联的 `mint` 和 `owner` 账户派生。

请注意，这里的 `createAccount` 函数与我们在查看 `createMint` 函数底层时上面展示的 `createAccount` 函数是不同的。以前，我们使用 `SystemProgram` 上的 `createAccount` 函数返回创建所有账户的指令。这里的 `createAccount` 函数是 `spl-token` 库中的一个辅助函数，用于提交包含两个指令的事务。第一个指令创建账户，第二个指令将账户初始化为代币账户。

与创建铸币厂一样，如果我们需要手动构建 `createAccount` 的交易，我们可以复制函数在底层所做的操作：
1. 使用 `getMint` 检索与 `mint` 相关联的数据
2. 使用 `getAccountLenForMint` 计算代币账户所需的空间
3. 使用 `getMinimumBalanceForRentExemption` 计算用于免租金的 lamports
4. 使用 `SystemProgram.createAccount` 和 `createInitializeAccountInstruction` 创建一个新的交易。注意，这里的 `createAccount` 来自 `@solana/web3.js`，用于创建一个通用的新账户。`createInitializeAccountInstruction` 使用这个新账户来初始化新的代币账户。

```tsx
import * as web3 from '@solana/web3'
import * as token from '@solana/spl-token'

async function buildCreateTokenAccountTransaction(
  connection: web3.Connection,
  payer: web3.PublicKey,
  mint: web3.PublicKey
): Promise<web3.Transaction> {
  const mintState = await token.getMint(connection, mint)
  const accountKeypair = await web3.Keypair.generate()
  const space = token.getAccountLenForMint(mintState);
  const lamports = await connection.getMinimumBalanceForRentExemption(space);
  const programId = token.TOKEN_PROGRAM_ID

  const transaction = new web3.Transaction().add(
    web3.SystemProgram.createAccount({
      fromPubkey: payer,
      newAccountPubkey: accountKeypair.publicKey,
      space,
      lamports,
      programId,
    }),
    token.createInitializeAccountInstruction(
      accountKeypair.publicKey,
      mint,
      payer,
      programId
    )
  );

  return transaction
}
```

### 关联代币账户

关联代币账户（associated token account）是一个代币账户，其地址是通过所有者的公钥和一个铸币厂来派生的。关联代币账户提供了一种确定性的方法，用于找到由特定 `publicKey` 拥有的特定铸币厂的代币账户。

大多数情况下，您创建代币账户时，希望它是一个关联代币账户。
- 如果不是关联代币账户，用户可能会拥有属于同一铸币厂的许多代币账户，导致不知道要将代币发送到哪里。
- 关联代币账户允许用户将代币发送给另一个用户，如果接收方尚未拥有该铸币厂的代币账户。

![ATAs are PDAs](../../assets/atas-are-pdas.svg)

Similar to above, you can create an associated token account using the `spl-token` library's `createAssociatedTokenAccount` function.

```tsx
const associatedTokenAccount = await createAssociatedTokenAccount(
  connection,
	payer,
	mint,
	owner,
);
```

这个函数返回新关联代币账户的 `publicKey`，并需要以下参数：

- `connection` - 连接到集群的 JSON-RPC 连接
- `payer` - 交易的付款人账户
- `mint` - 新关联的代币账户所关联的铸币厂
- `owner` - 新关联代币账户的所有者账户

您还可以使用 `getOrCreateAssociatedTokenAccount` 来获取与给定地址关联的代币账户，如果该账户不存在则创建它。例如，如果您要编写代码向给定用户进行空投代币，您可能会使用此函数来确保与给定用户关联的代币账户在不存在时被创建。

在底层，`createAssociatedTokenAccount` 执行两个操作：

1. 使用 `getAssociatedTokenAddress` 从 `mint` 和 `owner` 派生关联代币账户地址
2. 使用 `createAssociatedTokenAccountInstruction` 中的指令构建一个交易

```tsx
import * as web3 from '@solana/web3'
import * as token from '@solana/spl-token'

async function buildCreateAssociatedTokenAccountTransaction(
  payer: web3.PublicKey,
  mint: web3.PublicKey
): Promise<web3.Transaction> {
  const associatedTokenAddress = await token.getAssociatedTokenAddress(mint, payer, false);

  const transaction = new web3.Transaction().add(
    token.createAssociatedTokenAccountInstruction(
      payer,
      associatedTokenAddress,
      payer,
      mint
    )
  )

  return transaction
}
```

## 铸造代币

铸造代币是将新代币发行到流通中的过程。当您铸造代币时，您增加了铸币厂的供应量，并将新铸造的代币存入代币账户。只有铸币厂的铸造权限才被允许铸造新代币。

要使用 `spl-token` 库铸造代币，您可以使用 `mintTo` 函数。

```tsx
const transactionSignature = await mintTo(
  connection,
  payer,
  mint,
  destination,
  authority,
  amount
);
```

`mintTo` 函数返回一个 `TransactionSignature`，可以在 Solana 浏览器上查看。`mintTo` 函数需要以下参数：

- `connection` - 连接到集群的 JSON-RPC 连接
- `payer` - 交易的付款人账户
- `mint` - 新的代币账户关联的铸币厂
- `destination` - 将代币从铸造厂铸造到的代币账户
- `authority` - 被授权铸造代币的账户
- `amount` - 铸造代币的原始数量，不考虑小数，例如，如果 Scrooge Coin 的小数属性设置为 2，则需要将此属性设置为 100 才能获得 1 个完整的 Scrooge Coin。

在铸币厂后将铸造权限更新为 null 并不罕见。这会设置一个最大供应量，并确保将来无法再铸造代币。相反，铸造权限可以授予一个程序，以便根据常规间隔或可编程条件自动铸造代币。

在底层，`mintTo` 函数简单地创建一个交易，其中包含从 `createMintToInstruction` 函数获取的指令。

```tsx
import * as web3 from '@solana/web3'
import * as token from '@solana/spl-token'

async function buildMintToTransaction(
  authority: web3.PublicKey,
  mint: web3.PublicKey,
  amount: number,
  destination: web3.PublicKey
): Promise<web3.Transaction> {
  const transaction = new web3.Transaction().add(
    token.createMintToInstruction(
      mint,
      destination,
      authority,
      amount
    )
  )

  return transaction
}
```

## 转移代币

SPL-Token 转移要求发送方和接收方都必须拥有铸币厂的代币账户。代币从发送方的代币账户转移到接收方的代币账户。

在获取接收方的关联代币账户时，您可以使用 `getOrCreateAssociatedTokenAccount` 确保其代币账户在转移之前已存在。只需记住，如果该账户尚不存在，此函数将创建该账户，并且交易的付款人将被扣除创建账户所需的 lamports。

一旦您知道接收方的代币账户地址，您可以使用 `spl-token` 库的 `transfer` 函数来转移代币。

```tsx
const transactionSignature = await transfer(
  connection,
  payer,
  source,
  destination,
  owner,
  amount
)
```

`transfer` 函数返回一个 `TransactionSignature`，可以在 Solana Explorer 上查看。`transfer` 函数需要以下参数：

- `connection` - 连接到集群的 JSON-RPC 连接
- `payer` - 交易的付款人账户
- `source` - 发送代币的代币账户
- `destination` - 接收代币的代币账户
- `owner` - `source` 代币账户的所有者账户
- `amount` - 要转移的代币数量

在底层，`transfer` 函数简单地创建一个交易，其中包含从 `createTransferInstruction` 函数获取的指令：

```tsx
import * as web3 from '@solana/web3'
import * as token from '@solana/spl-token'

async function buildTransferTransaction(
  source: web3.PublicKey,
  destination: web3.PublicKey,
  owner: web3.PublicKey,
  amount: number
): Promise<web3.Transaction> {
  const transaction = new web3.Transaction().add(
    token.createTransferInstruction(
      source,
      destination,
      owner,
      amount,
    )
  )

  return transaction
}
```

## 销毁代币

销毁代币是减少特定铸币厂的代币供应量的过程。销毁代币会将它们从给定的代币账户和更广泛的流通中移除。

要使用 `spl-token` 库销毁代币，您可以使用 `burn` 函数。

```tsx
const transactionSignature = await burn(
  connection,
  payer,
  account,
  mint,
  owner,
  amount
)
```

`burn` 函数返回一个 `TransactionSignature`，可以在 Solana Explorer 上查看。`burn` 函数需要以下参数：

- `connection` - 连接到集群的 JSON-RPC 连接
- `payer` - 交易的付款人账户
- `account` - 要销毁代币的代币账户
- `mint` - 与代币账户关联的铸币厂
- `owner` - 代币账户的所有者账户
- `amount` - 要销毁的代币数量

在底层，`burn` 函数创建一个，交易其中包含从 `createBurnInstruction` 函数获取的指令。

```tsx
import * as web3 from '@solana/web3'
import * as token from '@solana/spl-token'

async function buildBurnTransaction(
  account: web3.PublicKey,
  mint: web3.PublicKey,
  owner: web3.PublicKey,
  amount: number
): Promise<web3.Transaction> {
  const transaction = new web3.Transaction().add(
    token.createBurnInstruction(
      account,
      mint,
      owner,
      amount
    )
  )

  return transaction
}
```

## 批准委托

批准委托（approve delegate）是代币账户授权另一个账户转移或销毁代币的过程。当使用委托时，代币账户的权限仍由原始所有者控制。委托账户可以转移或销毁的代币的最大数量在代币账户的所有者批准委托时指定。请注意，一个代币账户在任何给定时间只能关联一个委托账户。

要使用 `spl-token` 库批准委托，您可以使用 `approve` 函数。

```tsx
const transactionSignature = await approve(
  connection,
  payer,
  account,
  delegate,
  owner,
  amount
  )
```

`approve` 函数返回一个 `TransactionSignature`，可以在 Solana Explorer 上查看。`approve` 函数需要以下参数：

- `connection` - 连接到集群的 JSON-RPC 连接
- `payer` - 交易的付款人账户
- `account` - 要委托代币的代币账户
- `delegate` - 所有者授权进行代币转移或销毁的账户
- `owner` - 代币账户的所有者账户
- `amount` - 委托账户可以转移或销毁的代币的最大数量

在底层，`approve` 函数创建一个事务，其中包含从 `createApproveInstruction` 函数获取的指令。

```tsx
import * as web3 from '@solana/web3'
import * as token from '@solana/spl-token'

async function buildApproveTransaction(
  account: web3.PublicKey,
  delegate: web3.PublicKey,
  owner: web3.PublicKey,
  amount: number
): Promise<web3.Transaction> {
  const transaction = new web3.Transaction().add(
    token.createApproveInstruction(
      account,
      delegate,
      owner,
      amount
    )
  )

  return transaction
}
```

## 撤销委托

之前为代币账户批准的委托可以稍后被撤销（revoke）。一旦委托被撤销，委托就无法再从所有者的代币账户转移代币。之前批准的金额中剩余未转移的任何金额也无法由委托转移。

要使用 `spl-token` 库撤销委托，您可以使用 `revoke` 函数。

```tsx
const transactionSignature = await revoke(
  connection,
  payer,
  account,
  owner,
  )
```

`revoke` 函数返回一个 `TransactionSignature`，可以在 Solana Explorer 上查看。`revoke` 函数需要以下参数：

- `connection` - 连接到集群的 JSON-RPC 连接
- `payer` - 交易的付款人账户
- `account` - 要撤销委托权限的代币账户
- `owner` - 代币账户的所有者账户

在底层，`revoke` 函数创建一个交易，其中包含从 `createRevokeInstruction` 函数获取的指令。

```tsx
import * as web3 from '@solana/web3'
import * as token from '@solana/spl-token'

async function buildRevokeTransaction(
  account: web3.PublicKey,
  owner: web3.PublicKey,
): Promise<web3.Transaction> {
  const transaction = new web3.Transaction().add(
    token.createRevokeInstruction(
      account,
      owner,
    )
  )

  return transaction
}
```

# 实验

我们将创建一个脚本，与代币程序中的指令进行交互。我们将创建一个铸币厂，创建代币账户，铸造代币，批准委托，转移代币和销毁代币。

## 1. 基本结构

让我们从一些基本的结构开始。您可以根据自己的需求设置项目，但我们将使用一个简单的 TypeScript 项目，依赖于 `@solana/web3.js` 和 `@solana/spl-token` 包。

您可以在命令行中使用 `npx create-solana-client [INSERT_NAME_HERE] --initialize-keypair` 克隆我们将从中开始的模板。或者您可以[手动克隆模板](https://github.com/Unboxed-Software/solana-npx-client-template/tree/with-keypair-env)。请注意，如果您直接使用 git 存储库作为起点，我们将从 `with-keypair-env` 分支开始。

然后，您需要添加对 `@solana/spl-token` 的依赖。在新创建的目录中，使用命令行执行 `npm install @solana/spl-token`。

## 2. 创建铸币厂

我们将使用 `@solana/spl-token` 库，所以让我们首先在文件顶部导入它。

```tsx
import * as token from '@solana/spl-token'
```

接下来，声明一个名为 `createNewMint` 的新函数，带有参数 `connection`、`payer`、`mintAuthority`、`freezeAuthority` 和 `decimals`。

在函数体内：
从 `@solana/spl-token` 中导入 `createMint`，然后创建一个调用 `createMint` 的函数：

```tsx
async function createNewMint(
  connection: web3.Connection,
  payer: web3.Keypair,
  mintAuthority: web3.PublicKey,
  freezeAuthority: web3.PublicKey,
  decimals: number
): Promise<web3.PublicKey> {

  const tokenMint = await token.createMint(
    connection,
    payer,
    mintAuthority,
    freezeAuthority,
    decimals
  );

  console.log(
    `Token Mint: https://explorer.solana.com/address/${tokenMint}?cluster=devnet`
  );

  return tokenMint;
}
```

完成了该函数后，从 `main` 函数的主体中调用它，将 `user` 设置为 `payer`、`mintAuthority` 和 `freezeAuthority`。

创建新的铸币厂后，让我们使用 `getMint` 函数获取账户数据，并将其存储在名为 `mintInfo` 的变量中。稍后我们将使用此数据来调整铸币厂的小数精度。

```tsx
async function main() {
  const connection = new web3.Connection(web3.clusterApiUrl("devnet"))
  const user = await initializeKeypair(connection)

  const mint = await createNewMint(
    connection,
    user,
    user.publicKey,
    user.publicKey,
    2
  )

  const mintInfo = await token.getMint(connection, mint);
}
```

## 3. 创建代币账户

现在我们已经创建了铸币厂，让我们创建一个新的代币账户，将 `user` 指定为 `owner`。

`createAccount` 函数可以创建一个带有自定义 Token 账户地址作为新 Token 账户。请注意，如果不提供地址，`createAccount` 将默认使用使用 `mint` 和 `owner` 推导出的关联 Token 账户。

或者，`createAssociatedTokenAccount` 函数也可以创建一个具有与 `mint` 和 `owner` 公钥推导出的相同地址的关联 Token 账户。

对于我们的演示，我们将使用 `getOrCreateAssociatedTokenAccount` 函数来创建我们的代币账户。该函数如果代币账户已存在，则获取 Token 账户的地址。如果不存在，则将在适当的地址创建一个新的关联 Token 账户。

```tsx
async function createTokenAccount(
  connection: web3.Connection,
  payer: web3.Keypair,
  mint: web3.PublicKey,
  owner: web3.PublicKey
) {
  const tokenAccount = await token.getOrCreateAssociatedTokenAccount(
    connection,
    payer,
    mint,
    owner
  )

  console.log(
    `Token Account: https://explorer.solana.com/address/${tokenAccount.address}?cluster=devnet`
  )

  return tokenAccount
}
```

在 `main` 函数中添加一个调用 `createTokenAccount` 的语句，将我们在上一步中创建的铸币厂传递进去，并将 `user` 设置为 `payer` 和 `owner`。

```tsx
async function main() {
  const connection = new web3.Connection(web3.clusterApiUrl("devnet"))
  const user = await initializeKeypair(connection)

  const mint = await createNewMint(
    connection,
    user,
    user.publicKey,
    user.publicKey,
    2
  )

  const mintInfo = await token.getMint(connection, mint);

  const tokenAccount = await createTokenAccount(
    connection,
    user,
    mint,
    user.publicKey
  )
}
```

## 4. 铸造代币

现在我们有了一个铸币厂和一个代币账户，让我们向代币账户铸造代币。请注意，只有 `mintAuthority` 才能向代币账户铸造新的代币。回想一下，我们将 `user` 设置为我们创建的 `mint` 的 `mintAuthority`。

创建一个名为 `mintTokens` 的函数，使用 `spl-token` 函数 `mintTo` 来铸造代币：

```tsx
async function mintTokens(
  connection: web3.Connection,
  payer: web3.Keypair,
  mint: web3.PublicKey,
  destination: web3.PublicKey,
  authority: web3.Keypair,
  amount: number
) {
  const transactionSignature = await token.mintTo(
    connection,
    payer,
    mint,
    destination,
    authority,
    amount
  )

  console.log(
    `Mint Token Transaction: https://explorer.solana.com/tx/${transactionSignature}?cluster=devnet`
  )
}
```

让我们在 `main` 中使用之前创建的 `mint` 和 `tokenAccount` 调用该函数。

请注意，我们必须调整输入的 `amount`，以适应铸币厂的小数精度。我们的 `mint` 代币的小数精度为 2。如果我们只指定 100 作为输入的 `amount`，那么只会向我们的代币账户铸造 1 个代币。

```tsx
async function main() {
  const connection = new web3.Connection(web3.clusterApiUrl("devnet"))
  const user = await initializeKeypair(connection)

  const mint = await createNewMint(
    connection,
    user,
    user.publicKey,
    user.publicKey,
    2
  )

  const mintInfo = await token.getMint(connection, mint);

  const tokenAccount = await createTokenAccount(
    connection,
    user,
    mint,
    user.publicKey
  )

  await mintTokens(
    connection,
    user,
    mint,
    tokenAccount.address,
    user,
    100 * 10 ** mintInfo.decimals
  )
}
```

## 5. 批准委托

现在我们有了一个铸币厂和一个代币账户，让我们授权一个委托人代表我们转移代币。

创建一个名为 `approveDelegate` 的函数，使用 `spl-token` 函数 `approve` 来批准委托：

```tsx
async function approveDelegate(
  connection: web3.Connection,
  payer: web3.Keypair,
  account: web3.PublicKey,
  delegate: web3.PublicKey,
  owner: web3.Signer | web3.PublicKey,
  amount: number
) {
  const transactionSignature = await token.approve(
    connection,
    payer,
    account,
    delegate,
    owner,
    amount
  )

  console.log(
    `Approve Delegate Transaction: https://explorer.solana.com/tx/${transactionSignature}?cluster=devnet`
  )
}
```

在 `main` 中，让我们生成一个新的 `Keypair` 来代表委托账户。然后，让我们调用我们的新的 `approveDelegate` 函数，并授权委托人从 `user` 代币账户转移最多 50 个代币。请记住根据铸币厂的小数精度调整 `amount`。

```tsx
async function main() {
  const connection = new web3.Connection(web3.clusterApiUrl("devnet"))
  const user = await initializeKeypair(connection)

  const mint = await createNewMint(
    connection,
    user,
    user.publicKey,
    user.publicKey,
    2
  )

  const mintInfo = await token.getMint(connection, mint);

  const tokenAccount = await createTokenAccount(
    connection,
    user,
    mint,
    user.publicKey
  )

  await mintTokens(
    connection,
    user,
    mint,
    tokenAccount.address,
    user,
    100 * 10 ** mintInfo.decimals
  )

  const delegate = web3.Keypair.generate();

  await approveDelegate(
    connection,
    user,
    tokenAccount.address,
    delegate.publicKey,
    user.publicKey,
    50 * 10 ** mintInfo.decimals
  )
}
```

## 6. 转移代币

接下来，让我们使用 `spl-token` 库的 `transfer` 函数转移我们刚刚铸造的一些代币。

```tsx
async function transferTokens(
  connection: web3.Connection,
  payer: web3.Keypair,
  source: web3.PublicKey,
  destination: web3.PublicKey,
  owner: web3.Keypair,
  amount: number
) {
  const transactionSignature = await token.transfer(
    connection,
    payer,
    source,
    destination,
    owner,
    amount
  )

  console.log(
    `Transfer Transaction: https://explorer.solana.com/tx/${transactionSignature}?cluster=devnet`
  )
}
```

在调用这个新函数之前，我们需要知道要将代币转移至哪个账户。

在 `main` 函数中，让我们生成一个新的 `Keypair` 作为接收者（但请记住，这只是模拟有人可以发送代币给的情况 - 在真实的应用中，您需要知道接收代币的人的钱包地址）。

然后，为接收者创建一个代币账户。最后，让我们调用我们的新 `transferTokens` 函数，将代币从 `user` 代币账户转移到 `receiver` 代币账户。我们将使用在前一步中批准的 `delegate` 代表我们执行转移操作。

```tsx
async function main() {
  const connection = new web3.Connection(web3.clusterApiUrl("devnet"))
  const user = await initializeKeypair(connection)

  const mint = await createNewMint(
    connection,
    user,
    user.publicKey,
    user.publicKey,
    2
  )

  const tokenAccount = await createTokenAccount(
    connection,
    user,
    mint,
    user.publicKey
  )

  const mintInfo = await token.getMint(connection, mint);

  await mintTokens(
    connection,
    user,
    mint,
    tokenAccount.address,
    user,
    100 * 10 ** mintInfo.decimals
  )

  const receiver = web3.Keypair.generate().publicKey
  const receiverTokenAccount = await createTokenAccount(
    connection,
    user,
    mint,
    receiver
  )

  const delegate = web3.Keypair.generate();
  await approveDelegate(
    connection,
    user,
    tokenAccount.address,
    delegate.publicKey,
    user.publicKey,
    50 * 10 ** mintInfo.decimals
  )

  await transferTokens(
    connection,
    user,
    tokenAccount.address,
    receiverTokenAccount.address,
    delegate,
    50 * 10 ** mintInfo.decimals
  )
}
```

## 7. 撤销委托

现在我们已经完成了代币转移，让我们使用 `spl-token` 库的 `revoke` 函数撤销 `delegate`。

```tsx
async function revokeDelegate(
  connection: web3.Connection,
  payer: web3.Keypair,
  account: web3.PublicKey,
  owner: web3.Signer | web3.PublicKey,
) {
  const transactionSignature = await token.revoke(
    connection,
    payer,
    account,
    owner,
  )

  console.log(
    `Revote Delegate Transaction: https://explorer.solana.com/tx/${transactionSignature}?cluster=devnet`
  )
}
```

撤销操作将会将代币账户的委托设置为 null，并将委托的数量重置为 0。我们只需要代币账户和用户的信息来执行这个函数。让我们调用我们的新 `revokeDelegate` 函数，从 `user` 代币账户中撤销委托。

```tsx
async function main() {
  const connection = new web3.Connection(web3.clusterApiUrl("devnet"))
  const user = await initializeKeypair(connection)

  const mint = await createNewMint(
    connection,
    user,
    user.publicKey,
    user.publicKey,
    2
  )

  const mintInfo = await token.getMint(connection, mint);

  const tokenAccount = await createTokenAccount(
    connection,
    user,
    mint,
    user.publicKey
  )

  await mintTokens(
    connection,
    user,
    mint,
    tokenAccount.address,
    user,
    100 * 10 ** mintInfo.decimals
  )

  const receiver = web3.Keypair.generate().publicKey
  const receiverTokenAccount = await createTokenAccount(
    connection,
    user,
    mint,
    receiver
  )

  const delegate = web3.Keypair.generate();
  await approveDelegate(
    connection,
    user,
    tokenAccount.address,
    delegate.publicKey,
    user.publicKey,
    50 * 10 ** mintInfo.decimals
  )

  await transferTokens(
    connection,
    user,
    tokenAccount.address,
    receiverTokenAccount.address,
    delegate,
    50 * 10 ** mintInfo.decimals
  )

  await revokeDelegate(
    connection,
    user,
    tokenAccount.address,
    user.publicKey,
  )
}
```

## 8. 销毁代币

最后，让我们通过销毁一些代币来从流通中移除它们。

创建一个名为 `burnTokens` 的函数，使用 `spl-token` 库的 `burn` 函数将你的代币总量减少一半。

```tsx
async function burnTokens(
  connection: web3.Connection,
  payer: web3.Keypair,
  account: web3.PublicKey,
  mint: web3.PublicKey,
  owner: web3.Keypair,
  amount: number
) {
  const transactionSignature = await token.burn(
    connection,
    payer,
    account,
    mint,
    owner,
    amount
  )

  console.log(
    `Burn Transaction: https://explorer.solana.com/tx/${transactionSignature}?cluster=devnet`
  )
}
```

现在在 `main` 函数中调用这个新函数，以销毁用户的 25 个代币。记得根据 `mint` 的小数精度来调整 `amount`。

```tsx
async function main() {
  const connection = new web3.Connection(web3.clusterApiUrl("devnet"))
  const user = await initializeKeypair(connection)

  const mint = await createNewMint(
    connection,
    user,
    user.publicKey,
    user.publicKey,
    2
  )

  const mintInfo = await token.getMint(connection, mint);

  const tokenAccount = await createTokenAccount(
    connection,
    user,
    mint,
    user.publicKey
  )

  await mintTokens(
    connection,
    user,
    mint,
    tokenAccount.address,
    user,
    100 * 10 ** mintInfo.decimals
  )

  const receiver = web3.Keypair.generate().publicKey
  const receiverTokenAccount = await createTokenAccount(
    connection,
    user,
    mint,
    receiver
  )

  const delegate = web3.Keypair.generate();
  await approveDelegate(
    connection,
    user,
    tokenAccount.address,
    delegate.publicKey,
    user.publicKey,
    50 * 10 ** mintInfo.decimals
  )

  await transferTokens(
    connection,
    user,
    tokenAccount.address,
    receiverTokenAccount.address,
    delegate,
    50 * 10 ** mintInfo.decimals
  )

  await revokeDelegate(
    connection,
    user,
    tokenAccount.address,
    user.publicKey,
  )

  await burnTokens(
    connection, 
    user, 
    tokenAccount.address, 
    mint, user, 
    25 * 10 ** mintInfo.decimals
  )
}
```

## 9. 测试一切

完成后，运行 `npm start`。你应该会在控制台上看到一系列的 Solana Explorer 链接被记录下来。点击它们，查看每个步骤的情况！你创建了一个新的铸币厂，创建了一个代币账户，铸造了 100 个代币，批准了一个委托，使用委托转移了 50 个代币，撤销了委托，并且销毁了额外的 25 个代币。你正在成为一个代币专家的路上。

如果你需要更多时间来熟悉这个项目，请查看完整的 [解决方案代码](https://github.com/Unboxed-Software/solana-token-client)。

# 挑战

现在轮到你独立构建一个项目了。创建一个应用程序，允许用户创建一个新的铸币厂（mint）、创建一个代币账户，并铸造代币。

请注意，你将无法直接使用我们在实验中讨论过的辅助函数。为了使用 Phantom 钱包适配器与 Token 程序进行交互，你需要手动构建每个交易，并将交易提交给 Phantom 进行批准。

![Screenshot of Token Program Challenge Frontend](../../assets/token-program-frontend.png)

1. 你可以从头开始构建这个项目，或者你可以[下载起始代码](https://github.com/Unboxed-Software/solana-token-frontend/tree/starter)。
2. 在 `CreateMint` 组件中创建一个新的铸币厂。如果你需要关于如何向钱包发送交易以进行批准的提示，请查看[钱包课程](./interact-with-wallets.md)。

    在创建新的铸币厂时，新生成的 `Keypair` 也必须签署交易。当除了连接的钱包之外还需要额外的签名者时，请使用以下格式：

    ```tsx
    sendTransaction(transaction, connection, {
      signers: [Keypair],
    })
    ```
3. 在 `CreateTokenAccount` 组件中创建一个新的代币账户。
4. 在 `MintToForm` 组件中铸造代币。

如果你遇到困难，请随时参考[解决方案代码](https://github.com/ZYJLiu/solana-token-frontend)。

记住，挑战自己，发挥创造力，让这些为己所用！

## 完成了实验吗？

将你的代码推送到 GitHub，并[告诉我们你对这节课的想法](https://form.typeform.com/to/IPH0UGz7#answers-lesson=72cab3b8-984b-4b09-a341-86800167cfc7)！