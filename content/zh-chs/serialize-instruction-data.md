---
title: 序列化自定义指令数据
objectives:
- 解释交易的内容
- 解释交易指令
- 解释 Solana runtime 优化的基础知识
- 解释 Borsh
- 使用 Borsh 序列化程序数据
---

# 总结

- 交易由一系列指令组成，单个交易可以包含任意数量的指令，每个指令都有指定自己的程序。当提交交易时，Solana runtime 将按顺序和原子方式处理其指令，这意味着如果任何指令由于任何原因失败，整个交易将无法被处理。
- 每个 *指令* 由三个组件组成：目标程序的 ID、涉及的所有账户的数组以及指令数据的字节缓冲区。
- 每个 *交易* 包含：一个数组，其中包含其打算读取或写入的所有账户，一个或多个指令，一个最近的块哈希，以及一个或多个签名。
- 为了将指令数据从客户端传递到服务器端，它必须被序列化为字节缓冲区。为了促进这个序列化过程，我们将使用 [Borsh](https://borsh.io/)。
- 交易可能由于各种原因而无法被区块链处理，我们将在这里讨论一些最常见的原因。

# 概述

## 交易

交易（Transactions）是我们将信息发送到区块链以进行处理的方式。到目前为止，我们已经学会了如何创建具有有限功能的非常基本的交易。但是，交易及其发送到的程序可以设计得更加灵活，并处理远比我们目前处理的更复杂的情况。

### 交易内容

每个交易都包含以下内容：

- 一个数组，其中包含其打算读取或写入的每个账户
- 一个或多个指令
- 一个最近的区块哈希
- 一个或多个签名

`@solana/web3.js` 为您简化了这个过程，因此您实际上只需要关注添加指令和签名。该库基于这些信息构建账户数组，并处理包含最近块哈希的逻辑。

## 指令

每个指令（instruction）包含以下内容：

- 执行目标程序的程序 ID（公钥）
- 列出在执行过程中将被读取或写入的每个账户的数组
- 指令数据的字节缓冲区（byte buffer）

通过其公钥识别程序确保了该指令由正确的程序执行。

包括一个将被读取或写入的每个账户的数组，允许网络执行许多优化，从而实现高交易负载和更快的执行。

字节缓冲区允许您向程序传递外部数据。

您可以在单个交易中包含多个指令。Solana runtime 将按顺序和原子方式处理这些指令。换句话说，如果每个指令都成功，则整个交易将成功，但如果单个指令失败，则整个交易将立即失败，没有任何副作用。

账户数组不仅仅是账户的公钥数组。数组中的每个对象都包含账户的公钥、它是否是交易的签名者以及它是否可写。在执行指令期间包括账户是否可写，使 runtime 能够促进智能合约的并行处理。因为您必须定义哪些账户是只读的，哪些是要写入的，runtime 可以确定哪些交易是不重叠的或只读的，并允许它们并发执行。要了解更多关于 Solana 运行时的信息，请查看这篇[博客文章](https://solana.com/news/sealevel-\--parallel-processing-thousands-of-smart-contracts)。

### 指令数据

向指令添加任意数据的能力确保了程序可以像 HTTP 请求的主体一样动态和灵活，可以适用于广泛的用例。

就像 HTTP 请求的主体结构取决于您打算调用的端点一样，用作指令数据的字节缓冲区的结构完全取决于接收方程序。如果您正在自己构建全栈 dApp，那么您需要将构建程序时使用的相同结构复制到客户端代码中。如果您正在与另一个开发人员合作处理程序开发，您可以协调以确保匹配的缓冲区布局（buffer layouts）。

让我们考虑一个具体的例子。想象一下，你正在开发一个 Web3 游戏，并负责编写与玩家物品栏程序交互的客户端代码。该程序被设计成允许客户端：

- 根据玩家的游戏结果添加物品到物品栏
- 将物品栏的中物品从一个玩家转移到另一个玩家
- 为玩家装备选择的物品栏的物品

该程序将被结构化，使得每个功能都封装在自己的函数中。

然而，每个程序只有一个入口点。您将通过指令数据指示程序运行其中的哪个函数。

您还会在指令数据中包含函数执行所需的任何信息，例如物品栏中物品的 ID、要转移物品的玩家等等。

这些数据的结构将取决于程序的编写方式，但通常情况下，指令数据的第一个字段是程序可以映射到函数的数字，随后的附加字段充当函数参数。

## 序列化

除了知道要在指令数据缓冲区中包含哪些信息之外，您还需要正确地序列化它。在Solana中最常用的序列化工具是[Borsh](https://borsh.io)。根据官网的描述：

> Borsh代表二进制对象表示法序列化器用于哈希。它旨在用于安全关键的项目，因为它优先考虑一致性、安全性、速度；并且附带严格的规范。

Borsh维护了一个[JS库](https://github.com/near/borsh-js)，该库处理将常见类型序列化到缓冲区(buffer)中。 还有一些基于borsh构建的其他包试图使这个过程变得更加简单。我们将使用 `@coral-xyz/borsh`库，可以使用`npm`进行安装。
基于之前的游戏库存示例，让我们来看一个假设性的场景，其中我们指示程序为玩家装备一个给定的物品。假设程序设计为接受一个代表具有以下属性的结构体的缓冲区(buffer)：

1.variant 作为一个无符号的8位整数，指示程序要执行哪个指令或函数。
2.playerId 作为一个无符号的16位整数，代表要装备给定物品的玩家的玩家ID。
3.itemId 作为一个无符号的256位整数，代表将要装备给给定玩家的物品ID。

所有这些都将作为一个字节缓冲区传递，将按顺序读取，所以确保正确的缓冲区布局顺序是至关重要的。你将为上述内容创建缓冲区布局模式或模板，如下所示：

```tsx
import * as borsh from '@coral-xyz/borsh'

const equipPlayerSchema = borsh.struct([
  borsh.u8('variant'),
  borsh.u16('playerId'),
  borsh.u256('itemId')
])
```

接下来，您可以使用这个模式（schema）和encode方法来编码数据。这个方法接受一个代表要序列化数据的对象和一个缓冲区作为参数。在下面的示例中，我们分配了一个比需要的大得多的新缓冲区，然后将数据编码到那个缓冲区中，并将原始缓冲区切割成一个新的、仅与所需大小相等的缓冲区。

```tsx
import * as borsh from '@coral-xyz/borsh'

const equipPlayerSchema = borsh.struct([
  borsh.u8('variant'),
  borsh.u16('playerId'),
  borsh.u256('itemId')
])

const buffer = Buffer.alloc(1000)
equipPlayerSchema.encode({ variant: 2, playerId: 1435, itemId: 737498 }, buffer)

const instructionBuffer = buffer.slice(0, equipPlayerSchema.getSpan(buffer))
```

一旦缓冲区被正确创建并且数据被序列化，剩下的就是构建交易了。这与你在之前的课程中所做的类似。下面的示例假设：

- `player`、`playerInfoAccount`和`PROGRAM_ID`已经在代码片段之外的某个地方定义
- `player`是用户的公钥
- `playerInfoAccount`是将要写入库存变更的账户的公钥
- 在执行指令的过程中将会使用`SystemProgram`。

```tsx
import * as borsh from '@coral-xyz/borsh'
import * as web3 from '@solana/web3.js'

const equipPlayerSchema = borsh.struct([
  borsh.u8('variant'),
  borsh.u16('playerId'),
  borsh.u256('itemId')
])

const buffer = Buffer.alloc(1000)
equipPlayerSchema.encode({ variant: 2, playerId: 1435, itemId: 737498 }, buffer)

const instructionBuffer = buffer.slice(0, equipPlayerSchema.getSpan(buffer))

const endpoint = web3.clusterApiUrl('devnet')
const connection = new web3.Connection(endpoint)

const transaction = new web3.Transaction()
const instruction = new web3.TransactionInstruction({
  keys: [
    {
      pubkey: player.publicKey,
      isSigner: true,
      isWritable: false,
    },
    {
      pubkey: playerInfoAccount,
      isSigner: false,
      isWritable: true,
    },
    {
      pubkey: web3.SystemProgram.programId,
      isSigner: false,
      isWritable: false,
    }
  ],
  data: instructionBuffer,
  programId: PROGRAM_ID
})

transaction.add(instruction)

web3.sendAndConfirmTransaction(connection, transaction, [player]).then((txid) => {
  console.log(`Transaction submitted: https://explorer.solana.com/tx/${txid}?cluster=devnet`)
})
```

# 实验

构建一个电影评论应用是一个很好的实践项目，它允许用户提交电影评论，并将其存储在Solana网络上。我们将在接下来的几节课中逐步构建这个应用，每节课都会添加新的功能。让我们一步一步来实现这个目标。

![电影评论前端界面](../../assets/movie-reviews-frontend.png)

这是我们将要构建的程序的快速图解:

![Solana在PDA中存储数据项，这些数据项可以通过它们的种子找到。](../../assets/movie-review-program.svg)

我们将在这个应用程序中使用的Solana程序的公钥是`CenYq6bDRB7p73EjsPEpiYN7uveyPUTdXkDkgUduboaN`.

### 1. 下载初始代码

在我们开始之前，请先下载[初始代码](https://github.com/Unboxed-Software/solana-movie-frontend/tree/starter).

该项目是一个相对简单的`Next.js`应用程序。它包括我们在钱包课程中创建的`WalletContextProvider`，一个用于展示电影评论的`Card`组件，一个用于以列表形式展示评论的`MovieList`组件，一个用于提交新评论的`Form`组件，以及一个包含`Movie`对象类定义的`Movie.ts`文件。

请注意，目前，在你运行`npm run dev`时页面上显示的电影是模拟数据。在本课中，我们将专注于添加新的评论，但实际上我们将无法看到该评论被展示。在下一课中，我们将专注于从链上账户反序列化自定义数据。

### 2. 创建缓冲区布局

记住，要正确地与Solana程序交互，你需要知道它期望数据如何结构化。我们的电影评论程序期望指令数据包含：

1. `variant`作为一个无符号的8位整数，表示应该执行哪个指令（换句话说，应该调用程序上的哪个函数）。
2. `title`作为一个字符串，代表你正在评论的电影的标题。
3. `rating`作为一个无符号的8位整数，代表你给予正在评论的电影的评分，满分为5分。
4. `description`作为一个字符串，代表你为电影留下的书面评论部分。

让我们在`Movie`类中配置一个`borsh`布局。首先导入`@coral-xyz/borsh`。接下来，创建一个`borshInstructionSchema`属性，并将其设置为包含上述属性的适当`borsh`结构体。

```tsx
import * as borsh from '@coral-xyz/borsh'

export class Movie {
  title: string;
  rating: number;
  description: string;

  ...

  borshInstructionSchema = borsh.struct([
    borsh.u8('variant'),
    borsh.str('title'),
    borsh.u8('rating'),
    borsh.str('description'),
  ])
}
```

请记住，顺序很重要。如果这里属性的*顺序*与程序的结构不同，交易将会失败。

### 3. 创建一个用于序列化数据的方法

现在我们已经设置好了缓冲区布局，让我们在`Movie`中创建一个名为`serialize()`的方法，该方法将返回一个`Buffer`，其中包含`Movie`对象的属性，这些属性被编码到适当的布局中。

```tsx
import * as borsh from '@coral-xyz/borsh'

export class Movie {
  title: string;
  rating: number;
  description: string;

  ...

  borshInstructionSchema = borsh.struct([
    borsh.u8('variant'),
    borsh.str('title'),
    borsh.u8('rating'),
    borsh.str('description'),
  ])

  serialize(): Buffer {
    const buffer = Buffer.alloc(1000)
    this.borshInstructionSchema.encode({ ...this, variant: 0 }, buffer)
    return buffer.slice(0, this.borshInstructionSchema.getSpan(buffer))
  }
}
```

上述方法首先为我们的对象创建了一个足够大的缓冲区，然后将`{ ...this, variant: 0 }`编码进缓冲区。因为`Movie`类定义包含了缓冲区布局所需的4个属性中的3个，并且使用了相同的命名，我们可以直接使用展开运算符并只添加variant属性。最后，该方法返回一个新的缓冲区，省略了原始缓冲区中未使用的部分。

### 4. 用户提交表单时发送交易

现在我们已经有了指令数据的构建块，当用户提交表单时，我们可以创建并发送交易。打开`Form.tsx`并找到`handleTransactionSubmit`函数。每次用户提交电影评论表单时，都会调用`handleSubmit`。

在这个函数内部，我们将创建并发送包含通过表单提交的数据的交易。

首先导入`@solana/web3.js`并从`@solana/wallet-adapter-react`导入`useConnection和useWallet`。

```tsx
import { FC } from 'react'
import { Movie } from '../models/Movie'
import { useState } from 'react'
import { Box, Button, FormControl, FormLabel, Input, NumberDecrementStepper, NumberIncrementStepper, NumberInput, NumberInputField, NumberInputStepper, Textarea } from '@chakra-ui/react'
import * as web3 from '@solana/web3.js'
import { useConnection, useWallet } from '@solana/wallet-adapter-react'
```

接下来，在`handleSubmit`函数之前，调用`useConnection()`来获取一个`connection`对象，并调用`useWallet()`来获取`publicKey`和`sendTransaction`。

```tsx
import { FC } from 'react'
import { Movie } from '../models/Movie'
import { useState } from 'react'
import { Box, Button, FormControl, FormLabel, Input, NumberDecrementStepper, NumberIncrementStepper, NumberInput, NumberInputField, NumberInputStepper, Textarea } from '@chakra-ui/react'
import * as web3 from '@solana/web3.js'
import { useConnection, useWallet } from '@solana/wallet-adapter-react'

const MOVIE_REVIEW_PROGRAM_ID = 'CenYq6bDRB7p73EjsPEpiYN7uveyPUTdXkDkgUduboaN'

export const Form: FC = () => {
  const [title, setTitle] = useState('')
  const [rating, setRating] = useState(0)
  const [message, setMessage] = useState('')

  const { connection } = useConnection();
  const { publicKey, sendTransaction } = useWallet();

  const handleSubmit = (event: any) => {
    event.preventDefault()
    const movie = new Movie(title, rating, description)
    handleTransactionSubmit(movie)
  }

  ...
}
```

在我们实现`handleTransactionSubmit`之前，让我们讨论一下需要做什么。我们需要：

1. 检查`publicKey`是否存在，以确保用户已经连接了他们的钱包。
2. 对`movie`调用`serialize()`来获取代表指令数据的缓冲区。
3. 创建一个新的`Transaction`对象。
4. 获取事务将读取或写入的所有账户。
5. 创建一个新的`Instruction`对象，该对象在`keys`参数中包含所有这些账户，在`data`参数中包含缓冲区，在`programId`参数中包含程序的公钥。
6. 将上一步中的指令添加到交易中。
7. 调用`sendTransaction`，传入组装好的交易。

这确实是很多步骤！但不用担心，做得越多就会越容易。让我们从上述的前3个步骤开始：

```tsx
const handleTransactionSubmit = async (movie: Movie) => {
  if (!publicKey) {
    alert('Please connect your wallet!')
    return
  }

  const buffer = movie.serialize()
  const transaction = new web3.Transaction()
}
```

下一步是获取交易将读取或写入的所有账户。在之前的课程中，将要存储数据的账户已经给出。这一次，账户的地址更加动态，因此需要计算。我们将在下一课中深入讨论这个问题，但现在你可以使用以下内容，其中`pda`是将要存储数据的账户的地址：

```tsx
const [pda] = await web3.PublicKey.findProgramAddress(
  [publicKey.toBuffer(), Buffer.from(movie.title)],
  new web3.PublicKey(MOVIE_REVIEW_PROGRAM_ID)
)
```

除了这个账户之外，程序还需要从`SystemProgram`读取，因此我们的数组还需要包括`web3.SystemProgram.programId`。

有了这些，我们可以完成剩下的步骤：

```tsx
const handleTransactionSubmit = async (movie: Movie) => {
  if (!publicKey) {
    alert('Please connect your wallet!')
    return
  }

  const buffer = movie.serialize()
  const transaction = new web3.Transaction()

  const [pda] = await web3.PublicKey.findProgramAddress(
    [publicKey.toBuffer(), new TextEncoder().encode(movie.title)],
    new web3.PublicKey(MOVIE_REVIEW_PROGRAM_ID)
  )

  const instruction = new web3.TransactionInstruction({
    keys: [
      {
        pubkey: publicKey,
        isSigner: true,
        isWritable: false,
      },
      {
        pubkey: pda,
        isSigner: false,
        isWritable: true
      },
      {
        pubkey: web3.SystemProgram.programId,
        isSigner: false,
        isWritable: false
      }
    ],
    data: buffer,
    programId: new web3.PublicKey(MOVIE_REVIEW_PROGRAM_ID)
  })

  transaction.add(instruction)

  try {
    let txid = await sendTransaction(transaction, connection)
    console.log(`Transaction submitted: https://explorer.solana.com/tx/${txid}?cluster=devnet`)
  } catch (e) {
    alert(JSON.stringify(e))
  }
}
```

就是这样！现在你应该能够使用网站上的表单提交电影评论了。虽然你不会看到用户界面更新以反映新的评论，但你可以在Solana Explorer上查看交易的程序日志，以确认它已成功。

如果你需要更多时间来熟悉这个项目，请查看完整的内容。 [解法代码](https://github.com/Unboxed-Software/solana-movie-frontend/tree/solution-serialize-instruction-data).

# 挑战

现在轮到你独立构建一些东西了。创建一个应用程序，让这门课程的学生们介绍自己！支持此功能的Solana程序位于
`HdE95RSVsdb315jfJtaykXhXY478h53X6okDupVfY9yf`.

![学生介绍前端的截图](../../assets/student-intros-frontend.png)

1. 你可以从头开始构建，或者你可以[下载起始代码](https://github.com/Unboxed-Software/solana-student-intros-frontend/tree/starter)。
2. 在`StudentIntro.ts`中创建指令缓冲区布局。程序期望指令数据包含：
  1. 作为无符号8位整数的variant，代表要运行的指令（应为0）。
  2. 作为字符串的name，代表学生的名字。
  3. 作为字符串的message，代表学生分享关于他们的Solana之旅的信息。
3. 在`StudentIntro.ts`中创建一个方法，使用缓冲区布局序列化一个`StudentIntro`对象。
4. 在`Form`组件中，实现`handleTransactionSubmit`函数，以便它序列化一个`StudentIntro`，构建适当的交易和交易指令，并将交易提交到用户的钱包。
5. 现在你应该能够提交介绍，并且有信息存储在链上！确保记录交易ID，并在`Solana Explorer`中查看它以验证它是否工作。

如果你真的感到非常困惑，你可以[查看解决方案代码](https://github.com/Unboxed-Software/solana-student-intros-frontend/tree/solution-serialize-instruction-data)。

随意发挥这些挑战的创造性，并将它们推进得更远。这些指令不是为了限制你！

## 实验完成了吗?

将你的代码推送到GitHub并[告诉我们你对这节课的看法](https://form.typeform.com/to/IPH0UGz7#answers-lesson=6cb40094-3def-4b66-8a72-dd5f00298f61)!
