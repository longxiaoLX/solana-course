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

## Serialization

In addition to knowing what information to include in an instruction data buffer, you also need to serialize it properly. The most common serializer used in Solana is [Borsh](https://borsh.io). Per the website:

> Borsh stands for Binary Object Representation Serializer for Hashing. It is meant to be used in security-critical projects as it prioritizes consistency, safety, speed; and comes with a strict specification.

Borsh maintains a [JS library](https://github.com/near/borsh-js) that handles serializing common types into a buffer. There are also other packages built on top of borsh that try to make this process even easier. We’ll be using the `@coral-xyz/borsh` library which can be installed using `npm`.

Building off of the previous game inventory example, let’s look at a hypothetical scenario where we are instructing the program to equip a player with a given item. Assume the program is designed to accept a buffer that represents a struct with the following properties:

1. `variant` as an unsigned, 8-bit integer that instructs the program which instruction, or function, to execute.
2. `playerId` as an unsigned, 16-bit integer that represents the player ID of the player who is to be equipped with the given item.
3. `itemId` as an unsigned, 256-bit integer that represents the item ID of the item that will be equipped to the given player.

All of this will be passed as a byte buffer that will be read in order, so ensuring proper buffer layout order is crucial. You would create the buffer layout schema or template for the above as follows:

```tsx
import * as borsh from '@coral-xyz/borsh'

const equipPlayerSchema = borsh.struct([
  borsh.u8('variant'),
  borsh.u16('playerId'),
  borsh.u256('itemId')
])
```

You can then encode data using this schema with the `encode` method. This method accepts as arguments an object representing the data to be serialized and a buffer. In the below example, we allocate a new buffer that’s much larger than needed, then encode the data into that buffer and slice the original buffer down into a new buffer that’s only as large as needed.

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

Once a buffer is properly created and the data serialized, all that’s left is building the transaction. This is similar to what you’ve done in previous lessons. The example below assumes that:

- `player`, `playerInfoAccount`, and `PROGRAM_ID` are already defined somewhere outside the code snippet
- `player` is a user’s public key
- `playerInfoAccount` is the public key of the account where inventory changes will be written
- `SystemProgram` will be used in the process of executing the instruction.

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

# Lab

Let’s practice this together by building a Movie Review app that lets users submit a movie review and have it stored on Solana’s network. We’ll build this app a little bit at a time over the next few lessons, adding new functionality each lesson.

![Movie review frontend](../assets/movie-reviews-frontend.png)

Here's a quick diagram of the program we'll build:

![Solana stores data items in PDAs, which can be found by their seeds](../assets/movie-review-program.svg)

The public key of the Solana program we’ll use for this application is `CenYq6bDRB7p73EjsPEpiYN7uveyPUTdXkDkgUduboaN`.

### 1. Download the starter code

Before we get started, go ahead and download the [starter code](https://github.com/Unboxed-Software/solana-movie-frontend/tree/starter).

The project is a fairly simple Next.js application. It includes the `WalletContextProvider` we created in the Wallets lesson, a `Card` component for displaying a movie review, a `MovieList` component that displays reviews in a list, a `Form` component for submitting a new review, and a `Movie.ts` file that contains a class definition for a `Movie` object.

Note that for now, the movies displayed on the page when you run `npm run dev` are mocks. In this lesson, we’ll focus on adding a new review but we won’t actually be able to see that review displayed. Next lesson, we’ll focus on deserializing custom data from onchain accounts.

### 2. Create the buffer layout

Remember that to properly interact with a Solana program, you need to know how it expects data to be structured. Our Movie Review program is expecting instruction data to contain:

1. `variant` as an unsigned, 8-bit integer representing which instruction should be executed (in other words which function on the program should be called).
2. `title` as a string representing the title of the movie that you are reviewing.
3. `rating` as an unsigned, 8-bit integer representing the rating out of 5 that you are giving to the movie you are reviewing.
4. `description` as a string representing the written portion of the review you are leaving for the movie.

Let’s configure a `borsh` layout in the `Movie` class. Start by importing `@coral-xyz/borsh`. Next, create a `borshInstructionSchema` property and set it to the appropriate `borsh` struct containing the properties listed above.

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

Keep in mind that *order matters*. If the order of properties here differs from how the program is structured, the transaction will fail.

### 3. Create a method to serialize data

Now that we have the buffer layout set up, let’s create a method in `Movie` called `serialize()` that will return a `Buffer` with a `Movie` object’s properties encoded into the appropriate layout.

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

The method shown above first creates a large enough buffer for our object, then encodes `{ ...this, variant: 0 }` into the buffer. Because the `Movie` class definition contains 3 of the 4 properties required by the buffer layout and uses the same naming, we can use it directly with the spread operator and just add the `variant` property. Finally, the method returns a new buffer that leaves off the unused portion of the original.

### 4. Send transaction when user submits form

Now that we have the building blocks for the instruction data, we can create and send the transaction when a user submits the form. Open `Form.tsx` and locate the `handleTransactionSubmit` function. This gets called by `handleSubmit` each time a user submits the Movie Review form.

Inside this function, we’ll be creating and sending the transaction that contains the data submitted through the form.

Start by importing `@solana/web3.js` and importing `useConnection` and `useWallet` from `@solana/wallet-adapter-react`.

```tsx
import { FC } from 'react'
import { Movie } from '../models/Movie'
import { useState } from 'react'
import { Box, Button, FormControl, FormLabel, Input, NumberDecrementStepper, NumberIncrementStepper, NumberInput, NumberInputField, NumberInputStepper, Textarea } from '@chakra-ui/react'
import * as web3 from '@solana/web3.js'
import { useConnection, useWallet } from '@solana/wallet-adapter-react'
```

Next, before the `handleSubmit` function, call `useConnection()` to get a `connection` object and call `useWallet()` to get `publicKey` and `sendTransaction`.

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

Before we implement `handleTransactionSubmit`, let’s talk about what needs to be done. We need to:

1. Check that `publicKey` exists to ensure that the user has connected their wallet.
2. Call `serialize()` on `movie` to get a buffer representing the instruction data.
3. Create a new `Transaction` object.
4. Get all of the accounts that the transaction will read or write.
5. Create a new `Instruction` object that includes all of these accounts in the `keys` argument, includes the buffer in the `data` argument, and includes the program’s public key in the `programId` argument.
6. Add the instruction from the last step to the transaction.
7. Call `sendTransaction`, passing in the assembled transaction.

That’s quite a lot to process! But don’t worry, it gets easier the more you do it. Let’s start with the first 3 steps from above:

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

The next step is to get all of the accounts that the transaction will read or write. In past lessons, the account where data will be stored has been given to you. This time, the account’s address is more dynamic, so it needs to be computed. We’ll cover this in depth in the next lesson, but for now you can use the following, where `pda` is the address to the account where data will be stored:

```tsx
const [pda] = await web3.PublicKey.findProgramAddress(
  [publicKey.toBuffer(), Buffer.from(movie.title)],
  new web3.PublicKey(MOVIE_REVIEW_PROGRAM_ID)
)
```

In addition to this account, the program will also need to read from `SystemProgram`, so our array needs to include `web3.SystemProgram.programId` as well.

With that, we can finish the remaining steps:

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

And that’s it! You should now be able to use the form on the site to submit a movie review. While you won’t see the UI update to reflect the new review, you can look at the transaction’s program logs on Solana Explorer to see that it was successful.

If you need a bit more time with this project to feel comfortable, have a look at the complete [solution code](https://github.com/Unboxed-Software/solana-movie-frontend/tree/solution-serialize-instruction-data).

# Challenge

Now it’s your turn to build something independently. Create an application that lets students of this course introduce themselves! The Solana program that supports this is at `HdE95RSVsdb315jfJtaykXhXY478h53X6okDupVfY9yf`.

![Screenshot of Student Intros frontend](../assets/student-intros-frontend.png)

1. You can build this from scratch or you can [download the starter code](https://github.com/Unboxed-Software/solana-student-intros-frontend/tree/starter).
2. Create the instruction buffer layout in `StudentIntro.ts`. The program expects instruction data to contain:
   1. `variant` as an unsigned, 8-bit integer representing the instruction to run (should be 0).
   2. `name` as a string representing the student's name.
   3. `message` as a string representing the message the student is sharing about their Solana journey.
3. Create a method in `StudentIntro.ts` that will use the buffer layout to serialize a `StudentIntro` object.
4. In the `Form` component, implement the `handleTransactionSubmit` function so that it serializes a `StudentIntro`, builds the appropriate transaction and transaction instructions, and submits the transaction to the user's wallet.
5. You should now be able to submit introductions and have the information stored on chain! Be sure to log the transaction ID and look at it in Solana Explorer to verify that it worked.

If you get really stumped, you can [check out the solution code](https://github.com/Unboxed-Software/solana-student-intros-frontend/tree/solution-serialize-instruction-data).

Feel free to get creative with these challenges and take them even further. The instructions aren't here to hold you back!


## Completed the lab?

Push your code to GitHub and [tell us what you thought of this lesson](https://form.typeform.com/to/IPH0UGz7#answers-lesson=6cb40094-3def-4b66-8a72-dd5f00298f61)!