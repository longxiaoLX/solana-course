---
title: 客户端 Anchor 开发简介
objectives:
- 使用 IDL 从客户端与 Solana 程序交互
- 解释 Anchor 中的 `Provider` 对象
- 解释 Anchor 中的 `Program` 对象
- 使用 Anchor 的 `MethodsBuilder` 构建指令和交易
- 使用 Anchor 获取账户
- 设置一个前端来使用 Anchor 和 IDL 调用指令
---

# 总结

- **IDL** 是表示 Solana 程序结构的文件。使用 Anchor 编写和构建的程序会自动生成相应的 IDL。IDL 是接口描述语言（Interface Description Language）的缩写。
- `@coral-xyz/anchor` 是一个包含与 Anchor 程序交互所需的一切的 TypeScript 客户端。
- **Anchor `Provider`** 对象将连接到集群的 `connection` 和指定的 `wallet` 结合起来，以便进行交易签名。
- **Anchor `Program`** 对象提供了与特定程序交互的自定义 API。您可以使用程序的 IDL 和 `Provider` 创建一个 `Program` 实例。
- **Anchor `MethodsBuilder`** 通过 `Program` 提供了一个简单的接口来构建指令和交易。

# 概述

Anchor 通过提供接口描述语言（Interface Description Language，IDL）文件简化了从客户端与 Solana 程序交互的过程。使用 IDL 与 Anchor 的 Typescript 库（`@coral-xyz/anchor`）结合使用，为构建指令和交易提供了简化的格式。

```tsx
// sends transaction
await program.methods
  .instructionName(instructionDataInputs)
  .accounts({})
  .signers([])
  .rpc()
```

这适用于任何 TypeScript 客户端，无论是前端还是集成测试。在这节课中，我们将介绍如何使用 `@coral-xyz/anchor` 来简化你的客户端程序交互。

## Anchor 客户端结构

让我们从 Anchor 的 Typescript 库的基本结构开始。你将主要使用的对象是 `Program` 对象。`Program` 实例代表一个特定的 Solana 程序，并提供了一个自定义的 API 来读取和写入程序。

要创建 `Program` 的实例，你需要以下内容：

- IDL - 表示程序结构的文件
- `Connection` - 集群连接
- `Wallet` - 用于支付和签名交易的默认密钥对
- `Provider` - 封装了与 Solana 集群的 `Connection` 和一个 `Wallet`
- `ProgramId` - 程序的链上地址

![Anchor structure](../../assets/anchor-client-structure.png)

以上图像显示了如何将这些部分组合在一起来创建一个 `Program` 实例。我们将逐个介绍它们，以更好地了解它们如何联系在一起。

### 接口描述语言（IDL）

当你构建一个 Anchor 程序时，Anchor 会生成一个 JSON 和一个 Typescript 文件，代表你程序的 IDL。IDL 表示程序的结构，客户端可以使用它来推断如何与特定程序交互。

虽然它不是自动的，但你也可以使用诸如 Metaplex 的 [shank](https://github.com/metaplex-foundation/shank) 这样的工具从本机 Solana 程序生成 IDL。

为了了解 IDL 提供的信息，这里是你之前构建的计数器程序的 IDL：

```json
{
  "version": "0.1.0",
  "name": "counter",
  "instructions": [
    {
      "name": "initialize",
      "accounts": [
        { "name": "counter", "isMut": true, "isSigner": true },
        { "name": "user", "isMut": true, "isSigner": true },
        { "name": "systemProgram", "isMut": false, "isSigner": false }
      ],
      "args": []
    },
    {
      "name": "increment",
      "accounts": [
        { "name": "counter", "isMut": true, "isSigner": false },
        { "name": "user", "isMut": false, "isSigner": true }
      ],
      "args": []
    }
  ],
  "accounts": [
    {
      "name": "Counter",
      "type": {
        "kind": "struct",
        "fields": [{ "name": "count", "type": "u64" }]
      }
    }
  ]
}
```

检查 IDL，你会发现这个程序包含两个指令（`initialize` 和 `increment`）。

注意，除了指定指令外，它还为每个指令指定了账户和输入。`initialize` 指令需要三个账户：

1. `counter` - 在指令中初始化的新账户
2. `user` - 交易和初始化的付款方
3. `systemProgram` - 系统程序被调用来初始化一个新账户

而 `increment` 指令需要两个账户：

1. `counter` - 要递增计数字段的现有账户
2. `user` - 交易的付款方

查看 IDL，你会发现在两个指令中，`user` 都作为签名者是必需的，因为 `isSigner` 标志被标记为 `true`。此外，由于 `args` 部分对于两者都是空白，因此两个指令都不需要任何额外的指令数据。

继续查看 `accounts` 部分，你会发现程序包含一个名为 `Counter` 的账户类型，具有一个类型为 `u64` 的单个 `count` 字段。

虽然 IDL 不提供每个指令的实现细节，但我们可以大致了解链上程序希望指令如何构造，以及程序账户的结构。

无论你如何获取它，使用 `@coral-xyz/anchor` 包与程序交互时 *都需要* 一个 IDL 文件。要使用 IDL，你需要将 IDL 文件包含在你的项目中，然后导入该文件。

```tsx
import idl from "./idl.json"
```

### Provider

在使用 IDL 创建 `Program` 对象之前，您首先需要创建一个 Anchor `Provider` 对象。

`Provider` 对象结合了两个东西：

- `Connection` - 与 Solana 集群的连接（例如 localhost、devnet、mainnet）
- `Wallet` - 用于支付和签署交易的指定地址

然后，`Provider` 能够代表 `Wallet` 向 Solana 区块链发送交易，通过将钱包的签名包含到输出交易中。当在带有 Solana 钱包提供程序的前端中使用时，所有输出交易仍然必须由用户通过其钱包浏览器扩展审批。

设置 `Wallet` 和 `Connection` 会像这样：

```tsx
import { useAnchorWallet, useConnection } from "@solana/wallet-adapter-react"

const { connection } = useConnection()
const wallet = useAnchorWallet()
```

要设置连接，您可以使用 `@solana/wallet-adapter-react` 中的 `useConnection` 钩子（hook）来获取与 Solana 集群的连接。

请注意，`@solana/wallet-adapter-react` 提供的 `useWallet` 钩子提供的 `Wallet` 对象与 Anchor `Provider` 预期的 `Wallet` 对象不兼容。然而，`@solana/wallet-adapter-react` 还提供了一个 `useAnchorWallet` 钩子。

为了比较，这里是来自 `useAnchorWallet` 的 `AnchorWallet`：

```tsx
export interface AnchorWallet {
  publicKey: PublicKey
  signTransaction(transaction: Transaction): Promise<Transaction>
  signAllTransactions(transactions: Transaction[]): Promise<Transaction[]>
}
```

以及来自 `useWallet` 的 `WalletContextState`：

```tsx
export interface WalletContextState {
  autoConnect: boolean
  wallets: Wallet[]
  wallet: Wallet | null
  publicKey: PublicKey | null
  connecting: boolean
  connected: boolean
  disconnecting: boolean
  select(walletName: WalletName): void
  connect(): Promise<void>
  disconnect(): Promise<void>
  sendTransaction(
    transaction: Transaction,
    connection: Connection,
    options?: SendTransactionOptions
  ): Promise<TransactionSignature>
  signTransaction: SignerWalletAdapterProps["signTransaction"] | undefined
  signAllTransactions:
    | SignerWalletAdapterProps["signAllTransactions"]
    | undefined
  signMessage: MessageSignerWalletAdapterProps["signMessage"] | undefined
}
```

`WalletContextState` 提供了比 `AnchorWallet` 更多的功能，但是 `AnchorWallet` 是设置 `Provider` 对象所必需的。

要创建 `Provider` 对象，你可以使用 `@coral-xyz/anchor` 中的 `AnchorProvider`。

`AnchorProvider` 构造函数接受三个参数：

- `connection` - Solana 集群的 `Connection`
- `wallet` - `Wallet` 对象
- `opts` - 可选参数，用于指定确认选项，如果未提供则使用默认设置

创建了 `Provider` 对象后，你可以使用 `setProvider` 将其设置为默认提供程序。

```tsx
import { useAnchorWallet, useConnection } from "@solana/wallet-adapter-react"
import { AnchorProvider, setProvider } from "@coral-xyz/anchor"

const { connection } = useConnection()
const wallet = useAnchorWallet()
const provider = new AnchorProvider(connection, wallet, {})
setProvider(provider)
```

### Program

一旦你有了 IDL 和 provider，你就可以创建一个 `Program` 实例了。构造函数需要三个参数：

- `idl` - 类型为 `Idl` 的 IDL
- `programId` - 程序的链上地址，作为 `string` 或 `PublicKey`
- `Provider` - 在前面一节中讨论的 provider

`Program` 对象创建了一个自定义的 API，你可以用它来与 Solana 程序交互。这个 API 是与链上程序通信相关的一站式服务。除其他外，你可以发送交易、获取反序列化账户、解码指令数据、订阅账户变化，以及监听事件。你也可以[了解更多关于 `Program` 类的信息](https://coral-xyz.github.io/anchor/ts/classes/Program.html#constructor)。

要创建 `Program` 对象，首先从 `@coral-xyz/anchor` 导入 `Program` 和 `Idl`。`Idl` 是你在使用 TypeScript 时可以使用的类型。

接下来，指定程序的 `programId`。我们必须明确指定 `programId`，因为可以有多个具有相同 IDL 结构的程序（即如果使用不同地址多次部署相同的程序）。在创建 `Program` 对象时，如果未明确指定 provider，则使用默认的 provider。

总的来说，最终的设置看起来像这样：

```tsx
import idl from "./idl.json"
import { useAnchorWallet, useConnection } from "@solana/wallet-adapter-react"
import {
  Program,
  Idl,
  AnchorProvider,
  setProvider,
} from "@coral-xyz/anchor"

const { connection } = useConnection()
const wallet = useAnchorWallet()

const provider = new AnchorProvider(connection, wallet, {})
setProvider(provider)

const programId = new PublicKey("JPLockxtkngHkaQT5AuRYow3HyUv5qWzmhwsCPd653n")
const program = new Program(idl as Idl, programId)
```

## Anchor `MethodsBuilder`

一旦设置了 `Program` 对象，你就可以使用 Anchor Methods Builder 来构建与程序相关的指令和交易。`MethodsBuilder` 使用 IDL 提供了一个简化的格式，用于构建调用程序指令的交易。

请注意，与在 Rust 中编写程序时使用的蛇形命名（snake case naming）约定相比，与客户端交互时使用的是驼峰命名（camel case naming）约定。

基本的 `MethodsBuilder` 格式如下所示：

```tsx
// sends transaction
await program.methods
  .instructionName(instructionDataInputs)
  .accounts({})
  .signers([])
  .rpc()
```

逐步进行，你需要：

1. 在 `program` 上调用 `methods` - 这是用于创建与程序的 IDL 相关的指令调用的构建器 API。
2. 调用指令名为 `.instructionName(instructionDataInputs)` - 使用点语法简单地调用指令，使用点语法和指令的名称，将任何指令参数作为逗号分隔的值传递进去。
3. 调用 `accounts` - 使用点语法，调用 `.accounts`，传入一个对象，包含基于 IDL 指令所需的每个账户。
4. 可选地调用 `signers` - 使用点语法，调用 `.signers`，传入一个由指令需要的额外签名者组成的数组。
5. 调用 `rpc` - 此方法创建并发送一个带有指定指令的已签名交易，并返回一个 `TransactionSignature`。在使用 `.rpc` 时，`Provider` 中的 `Wallet` 会自动作为一个签名者包含在内，并不需要显式列出。

请注意，如果指令除了由 `Provider` 指定的 `Wallet` 外不需要额外的签名者，那么可以将 `.signer([])` 行排除在外。

你也可以直接构建交易，只需将 `.rpc()` 更改为 `.transaction()`。这将使用指定的指令构建一个 `Transaction` 对象。

```tsx
// creates transaction
const transaction = await program.methods
  .instructionName(instructionDataInputs)
  .accounts({})
  .transaction()

await sendTransaction(transaction, connection)
```

类似地，你可以使用相同的格式来使用 `.instruction()` 构建一个指令，然后手动将指令添加到一个新的交易中。这将使用指定的指令构建一个 `TransactionInstruction` 对象。

```tsx
// creates first instruction
const instructionOne = await program.methods
  .instructionOneName(instructionOneDataInputs)
  .accounts({})
  .instruction()

// creates second instruction
const instructionTwo = await program.methods
  .instructionTwoName(instructionTwoDataInputs)
  .accounts({})
  .instruction()

// add both instruction to one transaction
const transaction = new Transaction().add(instructionOne, instructionTwo)

// send transaction
await sendTransaction(transaction, connection)
```

总之，Anchor `MethodsBuilder` 提供了一种简化且更灵活的方式来与链上程序进行交互。你可以使用基本相同的格式构建指令、交易，或构建并发送交易，而无需手动序列化或反序列化账户或指令数据。

## 获取程序账户

`Program` 对象还允许你轻松地获取和过滤程序账户。只需在 `program` 上调用 `account`，然后指定在 IDL 上反映的账户类型的名称。Anchor 然后根据指定的方式对所有账户进行反序列化并返回。

下面的示例显示了如何获取 Counter 程序的所有现有 `counter` 账户。

```tsx
const accounts = await program.account.counter.all()
```

你还可以通过使用 `memcmp` 应用过滤器，然后指定一个 `offset` 和要过滤的 `bytes` 来进行过滤。

下面的示例获取了所有 `count` 为 0 的 `counter` 账户。请注意，`offset` 为 8 是因为 Anchor 使用的 8 个字节的鉴别器是用来识别账户类型的。第 9 个字节是 `count` 字段开始的地方。你可以参考 IDL，看到下一个字节存储着类型为 `u64` 的 `count` 字段。Anchor 然后过滤并返回所有在相同位置具有匹配字节的账户。

```tsx
const accounts = await program.account.counter.all([
    {
        memcmp: {
            offset: 8,
            bytes: bs58.encode((new BN(0, 'le')).toArray()),
        },
    },
])
```

或者，如果你知道你要查找的账户的地址，也可以使用 `fetch` 获取特定账户的反序列化账户数据。

```tsx
const account = await program.account.counter.fetch(ACCOUNT_ADDRESS)
```

同样地，你可以使用 `fetchMultiple` 获取多个账户。

```tsx
const accounts = await program.account.counter.fetchMultiple([ACCOUNT_ADDRESS_ONE, ACCOUNT_ADDRESS_TWO])
```

# 实验

让我们一起练习，为上节课的计数器程序构建一个前端界面。作为提醒，计数器程序有两个指令：

- `initialize` - 初始化一个新的 `Counter` 账户并将 `count` 设置为 `0`
- `increment` - 递增现有的 `Counter` 账户上的 `count`

## 1. 下载起始代码

下载[此项目的起始代码](https://github.com/Unboxed-Software/anchor-ping-frontend/tree/starter)。一旦你获得了起始代码，请先四处看看。使用 `npm install` 安装依赖项，然后使用 `npm run dev` 运行应用程序。

这个项目是一个简单的 Next.js 应用程序。它包括我们在[钱包课程](./interact-with-wallets.md)中创建的 `WalletContextProvider`，计数器程序的 `idl.json` 文件，以及我们将在整个实验中构建的 `Initialize` 和 `Increment` 组件。我们将调用的程序的 `programId` 也包含在起始代码中。

## 2. `Initialize`

首先，让我们完成在 `Initialize.tsx` 组件中创建 `Program` 对象的设置。

记住，我们需要一个 `Program` 实例来使用 Anchor `MethodsBuilder` 来调用我们程序的指令。为此，我们需要一个 Anchor 钱包和一个连接，我们可以从 `useAnchorWallet` 和 `useConnection` 钩子获取。让我们还创建一个 `useState` 来捕获程序实例。

```tsx
export const Initialize: FC<Props> = ({ setCounter }) => {
  const [program, setProgram] = useState("")

  const { connection } = useConnection()
  const wallet = useAnchorWallet()

  ...
}
```

有了这个，我们可以开始创建实际的 `Program` 实例。让我们在一个 `useEffect` 中完成这个操作。

首先，我们需要获取默认的 provider（如果它已经存在），或者如果不存在的话就创建它。我们可以通过在 try/catch 块中调用 `getProvider` 来实现。如果抛出错误，那意味着没有默认的提供程序，我们需要创建一个。

一旦我们有了提供程序，我们就可以构造一个 `Program` 实例。

```tsx
useEffect(() => {
  let provider: anchor.Provider

  try {
    provider = anchor.getProvider()
  } catch {
    provider = new anchor.AnchorProvider(connection, wallet, {})
    anchor.setProvider(provider)
  }

  const program = new anchor.Program(idl as anchor.Idl, PROGRAM_ID)
  setProgram(program)
}, [])
```

现在我们已经完成了 Anchor 的设置，我们可以实际调用程序的 `initialize` 指令了。我们将在 `onClick` 函数内部完成这个操作。

首先，我们需要为新的 `Counter` 账户生成一个新的 `Keypair`，因为我们是第一次初始化账户。

然后，我们可以使用 Anchor 的 `MethodsBuilder` 来创建并发送一个新的交易。请记住，Anchor 可以推断出一些所需的账户，如 `user` 和 `systemAccount` 账户。但是它不能推断出 `counter` 账户，因为我们是动态生成的，所以你需要使用 `.accounts` 添加它。你还需要将该密钥对添加为签名者，使用 `.signers`。最后，你可以使用 `.rpc()` 将交易提交给用户的钱包。

一旦交易完成，调用 `setUrl` 并传入浏览器 URL，然后调用 `setCounter`，传入计数器账户。

```tsx
const onClick = async () => {
  const sig = await program.methods
    .initialize()
    .accounts({
      counter: newAccount.publicKey,
      user: wallet.publicKey,
      systemAccount: anchor.web3.SystemProgram.programId,
    })
    .signers([newAccount])
    .rpc()

    setTransactionUrl(`https://explorer.solana.com/tx/${sig}?cluster=devnet`)
    setCounter(newAccount.publicKey)
}
```

## 3. `Increment`

接下来，让我们转到 `Increment.tsx` 组件。就像之前一样，完成创建 `Program` 对象的设置。除了调用 `setProgram` 外，`useEffect` 应该调用 `refreshCount`。

添加以下代码进行初始设置：

```tsx
export const Increment: FC<Props> = ({ counter, setTransactionUrl }) => {
  const [count, setCount] = useState(0)
  const [program, setProgram] = useState<anchor.Program>()
  const { connection } = useConnection()
  const wallet = useAnchorWallet()

  useEffect(() => {
    let provider: anchor.Provider

    try {
      provider = anchor.getProvider()
    } catch {
      provider = new anchor.AnchorProvider(connection, wallet, {})
      anchor.setProvider(provider)
    }

    const program = new anchor.Program(idl as anchor.Idl, PROGRAM_ID)
    setProgram(program)
    refreshCount(program)
  }, [])
  ...
}
```

接下来，让我们使用 Anchor 的 `MethodsBuilder` 来构建一个新的指令，调用 `increment` 指令。同样，Anchor 可以从钱包推断出 `user` 账户，因此我们只需要包含 `counter` 账户。

```tsx
const onClick = async () => {
  const sig = await program.methods
    .increment()
    .accounts({
      counter: counter,
      user: wallet.publicKey,
    })
    .rpc()

  setTransactionUrl(`https://explorer.solana.com/tx/${sig}?cluster=devnet`)
}
```

## 5. 显示正确的计数

现在我们可以初始化计数器程序并递增计数，我们需要让我们的界面显示存储在计数器账户中的计数。

我们将在以后的课程中展示如何观察账户变化，但现在我们只有一个按钮调用 `refreshCount`，所以你可以在每次 `increment` 调用后点击它来显示新的计数。

在 `refreshCount` 中，让我们使用 `program` 来获取计数器账户，然后使用 `setCount` 将计数设置为存储在程序上的数字：

```tsx
const refreshCount = async (program) => {
  const counterAccount = await program.account.counter.fetch(counter)
  setCount(counterAccount.count.toNumber())
}
```

使用 Anchor 真的非常简单！

## 5. 测试前端

到目前为止，一切都应该正常工作！你可以通过运行 `npm run dev` 来测试前端。

1. 连接你的钱包，你应该会看到 `Initialize Counter` 按钮
2. 点击 `Initialize Counter` 按钮，然后批准交易
3. 然后你应该在屏幕底部看到一个指向 Solana Explorer 的链接，用于找到 `initialize` 交易。`Increment Counter` 按钮，`Refresh Count` 按钮和计数也应该都出现。
4. 点击 `Increment Counter` 按钮，然后批准交易
5. 等待几秒钟，然后点击 `Refresh Count`。计数应该在屏幕上递增。

![Gif of Anchor Frontend Demo](../../assets/anchor-frontend-demo.gif)

请随时点击链接检查每个交易的程序日志！

![Screenshot of Initialize Program Log](../../assets/anchor-frontend-initialize.png)

![Screenshot of Increment Program Log](../../assets/anchor-frontend-increment.png)

恭喜你，你现在知道如何设置前端来调用使用 Anchor IDL 的 Solana 程序。

如果你需要更多时间来熟悉这些概念，可以在继续之前查看 [solution-increment 分支上的解决方案代码](https://github.com/Unboxed-Software/anchor-ping-frontend/tree/solution-increment)。

# 挑战

现在轮到你独立构建一些东西了。在实验中所做的基础上，尝试在前端构建一个新组件，实现一个按钮来递减计数器。

在前端构建组件之前，你需要：

1. 构建并部署一个新程序，实现一个 `decrement` 指令
2. 使用新程序的 IDL 文件更新前端的 IDL 文件
3. 使用新程序的 `programId` 更新前端的 `programId`

如果需要帮助，可以参考[此程序](https://github.com/Unboxed-Software/anchor-counter-program/tree/solution-decrement)。

尽量独立完成这项任务！但如果遇到困难，可以参考[解决方案代码](https://github.com/Unboxed-Software/anchor-ping-frontend/tree/solution-decrement)。


## 完成实验了吗？

将你的代码推送到 GitHub，然后[告诉我们你对这节课的看法](https://form.typeform.com/to/IPH0UGz7#answers-lesson=774a4023-646d-4394-af6d-19724a6db3db)！