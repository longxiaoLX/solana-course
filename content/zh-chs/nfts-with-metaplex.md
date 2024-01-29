
---
title: 使用Metaplex在Solana创建NFTs
objectives:
- 解释NFTs以及它们在Solana网络中的表示方式
- 解释Metaplex在Solana NFT生态系统中的作用
- 使用Metaplex SDK创建和更新NFTs
- 解释Token Metadata程序、Candy Machine程序和Sugar CLI的基本功能，它们是帮助在Solana上创建和分发NFTs的工具
---

# TL;DR
**非同质化代币（NFTs）**在 Solana 上以 SPL 代币的形式存在，与之关联的是一个元数据账户，它有 0 位小数，并且最大供应量为 1。
**Metaplex** 提供了一系列工具，简化了在 Solana 区块链上创建和分发 NFTs 的过程。
**Token Metadata** 程序标准化了将元数据附加到 SPL 代币的过程。
**Metaplex SDK** 是一个工具，提供了用户友好的 API，帮助开发人员利用 Metaplex 提供的链上工具。
**Candy Machine** 程序是一个用于创建和铸造集合中 NFTs 的 NFT 分发工具。
**Sugar CLI** 是一个工具，简化了上传媒体/元数据文件并为集合创建 Candy Machine 的过程。

## 简介

Solana的非同质化代币（NFTs）是使用 Token 程序创建的 SPL 代币。然而，这些代币还有一个额外的与每个代币铸造相关联的元数据帐户。这允许了广泛的用例，你可以有效地将任何东西令牌化，从游戏库存到艺术品。

在这篇文章中，我们将介绍 Solana 上 NFTs 的基本概念，如何使用Metaplex SDK创建和更新它们，并简要介绍一些工具，这些工具可以帮助你在 Solana 上大规模创建和分发NFTs。

## Solana上的NFTs

Solana上的 NFT 是一种非可分割代币，并附带有元数据。此外，代币的铸造有一个最大供应量为1。

换句话说，NFT是来自 Token 程序的标准代币，但与你可能认为的“标准代币”不同，它具有以下特点：

1. 小数位为0，因此不能分割为部分
2. 来自一个最大供应量为1的代币铸造，因此只有1个这样的代币存在
3. 来自一个授权设置为 `null` 的代币铸造（以确保供应量永远不会改变）
4. 有一个存储元数据的关联帐户

虽然前三个特点是使用SPL Token程序可以实现的功能，但关联的元数据需要一些额外的功能。

通常，NFT的元数据既有链上部分又有链下部分。见下图：

![元数据截图](../assets/solana-nft-metaplex-metadata.png)

- **链上元数据** 存储在与代币铸造厂关联的帐户中。链上元数据包含一个指向链下`.json`文件的 URI 字段。
- **链下元数据** 在 JSON 文件中存储 NFT 的媒体链接（图像、视频、3D文件）、NFT可能具有的特征以及其他元数据。通常使用永久数据存储系统（如Arweave）来存储 NFT 元数据的链下部分。

## Metaplex

[Metaplex](https://www.metaplex.com/)是一个提供一套工具的组织，比如[Metaplex SDK](https://docs.metaplex.com/sdks/js/)，它简化了在 Solana 区块链上创建和分发 NFTs 的过程。这些工具适用于各种用例，并且可以轻松地管理创建和铸造 NFT 集合的整个过程。

更具体地说，Metaplex SDK旨在帮助开发人员利用 Metaplex 提供的链上工具。它提供了用户友好的API，专注于流行的用例，并允许与第三方插件轻松集成。要了解Metaplex SDK的功能，请参阅[README](https://github.com/metaplex-foundation/js#readme)。

Metaplex提供的一个重要程序是Token Metadata程序。Token Metadata程序将元数据附加到 SPL 代币的过程标准化。使用 Metaplex 创建 NFT 时，Token Metadata程序使用代币铸造作为种子，使用程序派生地址（PDA）创建元数据帐户。这使得任何 NFT 的元数据帐户都可以通过代币铸造厂的地址确定性地定位。要了解有关Token Metadata程序的更多信息，请参阅Metaplex [文档](https://docs.metaplex.com/programs/token-metadata/)。

在接下来的部分，我们将介绍使用Metaplex SDK准备素材、创建NFT、更新 NFT 以及将 NFT 与更大的集合关联的基础知识。

### Metaplex实例

`Metaplex`实例作为访问Metaplex SDK API的入口点。此实例接受用于与集群通信的连接。此外，开发人员可以通过指定“Identity Driver”和“Storage Driver”来定制 SDK 的交互。

Identity Driver实际上是一个可用于签署交易的密钥对，在创建 NFT 时是必需的。Storage Driver用于指定上传素材的存储服务。 `bundlrStorage`驱动程序是默认选项，它将素材上传到Arweave，这是一个永久和去中心化的存储服务。

以下是如何为 devnet 设置`Metaplex`实例的示例。

```tsx
import {
  Metaplex,
  keypairIdentity,
  bundlrStorage,
} from "@metaplex-foundation/js";
import { Connection, clusterApiUrl, Keypair } from "@solana/web3.js";

const connection = new Connection(clusterApiUrl("devnet"));
const wallet = Keypair.generate();

const metaplex = Metaplex.make(connection)
  .use(keypairIdentity(wallet))
  .use(
    bundlrStorage({
      address: "https://devnet.bundlr.network",
      providerUrl: "https://api.devnet.solana.com",
      timeout: 60000,
    }),
  );
```

### 上传素材

在创建 NFT 之前，您需要准备并上传您计划与 NFT 关联的任何素材。虽然这不一定是一个图像，但大多数 NFT 都与图像相关联。

准备和上传图像涉及将图像转换为 buffer，使用`toMetaplexFile`函数将其转换为 Metaplex 格式，然后将其上传到指定的 Storage Driver。

Metaplex SDK 支持从本地计算机上的文件或通过浏览器上传的文件创建新的 Metaplex 文件。您可以通过使用`fs.readFileSync`来读取图像文件，然后使用`toMetaplexFile`将其转换为 Metaplex 文件。最后

，使用您的`Metaplex`实例调用`storage().upload(file)`来上传文件。该函数的返回值将是存储图像的URI。

```tsx
const buffer = fs.readFileSync("/path/to/image.png");
const file = toMetaplexFile(buffer, "image.png");

const imageUri = await metaplex.storage().upload(file);
```

### 上传元数据

在上传图像之后，是时候使用`nfts().uploadMetadata`函数上传链下 JSON 元数据了。这将返回存储 JSON 元数据的URI。

请记住，元数据的链下部分包括诸如图像 URI 以及 NFT 的名称和描述等附加信息。虽然你可以在这个 JSON 对象中包含任何你想要的东西，但在大多数情况下，你应该遵循[NFT标准](https://docs.metaplex.com/programs/token-metadata/token-standard#the-non-fungible-standard)以确保与钱包、程序和应用程序的兼容性。

要创建元数据，请使用 SDK 提供的`uploadMetadata`方法。此方法接受一个元数据对象，并返回一个指向上传的元数据的URI。

```tsx
const { uri } = await metaplex.nfts().uploadMetadata({
  name: "My NFT",
  description: "My description",
  image: imageUri,
});
```

### 创建NFT

上传 NFT 的元数据后，您可以在网络上创建 NFT。Metaplex SDK 的`create`方法允许您使用最小化配置来创建一个新的NFT。该方法将创建代币铸造厂帐户、代币帐户、元数据帐户和主版本帐户。提供给此方法的数据将表示 NFT 元数据的链上部分。您可以探索 SDK 以查看此方法可以可选提供的所有其他输入。

```tsx
const { nft } = await metaplex.nfts().create(
  {
    uri: uri,
    name: "My NFT",
    sellerFeeBasisPoints: 0,
  },
  { commitment: "finalized" },
);
```

此方法返回一个包含新创建的 NFT 信息的对象。默认情况下，SDK将`isMutable`属性设置为true，允许对 NFT 的元数据进行更新。但是，您可以选择将`isMutable`设置为false，使 NFT 的元数据不可变。

### 更新NFT

如果您将`isMutable`设置为true，您就可以更新 NFT 的元数据。SDK的`update`方法允许您更新 NFT 的链上和链下元数据。要更新链下元数据，您需要重复之前提到的上传新图像和元数据 URI 的步骤，然后将新的元数据 URI 提供给此方法。这将更改链上元数据指向的URI，从而有效地更新链下元数据。

```tsx
const nft = await metaplex.nfts().findByMint({ mintAddress });

const { response } = await metaplex.nfts().update(
  {
    nftOrSft: nft,
    name: "Updated Name",
    uri: uri,
    sellerFeeBasisPoints: 100,
  },
  { commitment: "finalized" },
);
```

请注意，您在调用`update`时不包括的任何字段将保持不变。

### 将 NFT 添加到集合

[Certified Collection](https://docs.metaplex.com/programs/token-metadata/certified-collections#introduction)是一个可以包含单个 NFT 的NFT。想象一个大型的 NFT 集合，比如Solana Monkey Business。如果您查看单个 NFT 的[Metadata](https://explorer.solana.com/address/C18YQWbfwjpCMeCm2MPGTgfcxGeEDPvNaGpVjwYv33q1/metadata)，您将看到一个`collection`字段，其中的`key`指向`Certified Collection` [NFT](https://explorer.solana.com/address/SMBH3wF6baUj6JWtzYvqcKuj2XCKWDqQxzspY12xPND/)。简而言之，作为集合的一部分的 NFT 与另一个代表集合本身的 NFT 相关联。

要将 NFT 添加到集合中，首先必须创建Collection NFT。此过程与之前相同，除了在我们的 NFT 元数据上包括一个附加字段：`isCollection`。此字段告诉 token 程序，此 NFT 是Collection NFT。

```tsx
const { collectionNft } = await metaplex.nfts().create(
  {
    uri: uri,
    name: "My NFT Collection",
    sellerFeeBasisPoints: 0,
    isCollection: true
  },
  { commitment: "finalized" },
);
```

然后，您将集合的铸造地址列为我们新 NFT 中的`collection`字段的参考。

```tsx
const { nft } = await metaplex.nfts().create(
  {
    uri: uri,
    name: "My NFT",
    sellerFeeBasisPoints: 0,
    collection: collectionNft.mintAddress
  },
  { commitment: "finalized" },
);
```

当您查看新创建的 NFT 的元数据时，您应该看到一个类似如下的`collection`字段：

```JSON
"collection":{
  "verified": false,
  "key": "SMBH3wF6baUj6JWtzYvqcKuj2XCKWDqQxzspY12xPND"
}
```

您需要做的最后一件事是验证NFT。这实际上只是将上述`verified`字段设置为true，但这非常重要。这是让消费程序和应用程序知道您的 NFT 实际上是集合的一部分的关键。您可以使用`verifyCollection`函数来执行此操作：

```tsx
await metaplex.nfts().verifyCollection({
  mintAddress: nft.address,
  collectionMintAddress: collectionNft.address,
  isSizedCollection: true,
})
```

### 糖果机（Candy Machine）

当创建和分发大量的 NFT 时，Metaplex通过他们的[Candy Machine](https://docs.metaplex.com/programs/candy-machine/overview)程序和[Sugar CLI](https://docs.metaplex.com/developer-tools/sugar/overview)使这变得简单。

Candy Machine实际上是一个铸造和分发程序，帮助启动 NFT 收藏品。Sugar是一个命令行界面，帮助您创建糖果机，准备素材，并批量创建NFT。上面介绍的创建 NFT 的步骤，如果一次创建数千个 NFT 将会非常繁琐。Candy Machine和 Sugar 通过提供多项保障来解决这个问题，并确保公平启动。

我们不会深入介绍这些工具，但可以查看[Metaplex文档中关于Candy Machine和 Sugar 如何协同工作的内容](https://docs.metaplex.com/developer-tools/sugar/overview/introduction)。

要探索 Metaplex 提供的全部工具，请访问 GitHub 上的[Metaplex代码库](https://github.com/metaplex-foundation/metaplex)。

# 实验室

在这个实验室中，我们将逐步介绍使用Metaplex SDK创建NFT的步骤，然后在事后更新NFT的元数据，最后将NFT与一个集合关联起来。到最后，你将对如何在Solana上使用Metaplex SDK与NFT进行交互有一个基本的了解。

### 1. 起步

首先，从[此代码库](https://github.com/Unboxed-Software/solana-metaplex/tree/starter)的`starter`分支下载起始代码。

项目中的`src`目录包含我们将用于NFT的两个图像。

此外，在`index.ts`文件中，你会找到以下代码片段，其中包含我们将要创建和更新的NFT的示例数据。

```tsx
interface NftData {
  name: string;
  symbol: string;
  description: string;
  sellerFeeBasisPoints: number;
  imageFile: string;
}

interface CollectionNftData {
  name: string
  symbol: string
  description: string
  sellerFeeBasisPoints: number
  imageFile: string
  isCollection: boolean
  collectionAuthority: Signer
}

// 新NFT的示例数据
const nftData = {
  name: "名称",
  symbol: "SYMBOL",
  description: "描述",
  sellerFeeBasisPoints: 0,
  imageFile: "solana.png",
}

// 更新现有NFT的示例数据
const updateNftData = {
  name: "更新",
  symbol: "UPDATE",
  description: "更新描述",
  sellerFeeBasisPoints: 100,
  imageFile: "success.png",
}

async function main() {
  // 创建到集群API的新连接
  const connection = new Connection(clusterApiUrl("devnet"));

  // 为用户初始化一个密钥对
  const user = await initializeKeypair(connection);

  console.log("公钥:", user.publicKey.toBase58());
}
```

在命令行中运行`npm install`安装必要的依赖项。

接下来，通过运行`npm start`执行代码。这将创建一个新的密钥对，将其写入`.env`文件，并向密钥对空投devnet SOL。

```
Current balance is 0
Airdropping 1 SOL...
New balance is 1
PublicKey: GdLEz23xEonLtbmXdoWGStMst6C9o3kBhb7nf7A1Fp6F
Finished successfully
```

### 2. 设置Metaplex

在我们开始创建和更新NFT之前，我们需要设置Metaplex实例。更新`main()`函数如下：

```tsx
async function main() {
  // 创建到集群API的新连接
  const connection = new Connection(clusterApiUrl("devnet"));

  // 为用户初始化一个密钥对
  const user = await initializeKeypair(connection);

  console.log("公钥:", user.publicKey.toBase58());

  // Metaplex设置
  const metaplex = Metaplex.make(connection)
    .use(keypairIdentity(user))
    .use(
      bundlrStorage({
        address: "https://devnet.bundlr.network",
        providerUrl: "https://api.devnet.solana.com",
        timeout: 60000,
      }),
    );
}
```

### 3. `uploadMetadata` 辅助函数

接下来，让我们创建一个辅助函数来处理上传图像和元数据的过程，并返回元数据URI。此函数将以Metaplex实例和NFT数据作为输入，并返回元数据URI作为输出。

```tsx
// 辅助函数，上传图像和元数据
async function uploadMetadata(
  metaplex: Metaplex,
  nftData: NftData,
): Promise<string> {
  // 文件转换为 buffer
  const buffer = fs.readFileSync("src/" + nftData.imageFile);

  // buffer 转换为Metaplex文件
  const file = toMetaplexFile(buffer, nftData.imageFile);

  // 上传图像并获取图像URI
  const imageUri = await metaplex.storage().upload(file);
  console.log("图像URI:", imageUri);

  // 上传元数据并获取元数据URI（离链元数据）
  const { uri } = await metaplex.nfts().uploadMetadata({
    name: nftData.name,
    symbol: nftData.symbol,
    description: nftData.description,
    image: imageUri,
  });

  console.log("元数据URI:", uri);
  return uri;
}
```

该函数将读取一个图像文件，将其转换为buffer，然后上传它以获取图像URI。然后，它将上传NFT元数据，其中包括名称、符号、描述和图像URI，并获取一个元数据URI。这个URI是链下元数据。该函数还会记录图像URI和元数据URI以供参考。

### 5. `createNft` 辅助函数

接下来，让我们创建一个辅助函数来处理创建NFT的过程。此函数接受Metaplex实例、元数据URI和NFT数据作为输入。它使用SDK的`create`方法创建NFT，将元数据URI、名称、版权费(sellerFeeBasisPoints)和符号作为参数传递。

```tsx
// 辅助函数，创建NFT
async function createNft(
  metaplex: Metaplex,
  uri: string,
  nftData: NftData,
): Promise<NftWithToken> {
  const { nft } = await metaplex.nfts().create(
    {
      uri: uri, // 元数据URI
      name: nftData.name,
      sellerFeeBasisPoints: nftData.sellerFeeBasisPoints,
      symbol: nftData.symbol,
    },
    { commitment: "finalized" },
  );

  console.log(
    `Token Mint: https://explorer.solana.com/address/${nft.address.toString()}?cluster=devnet`,
  );

  return nft;
}
```

`createNft`函数记录了Token Mint的URL，并返回包含有关新创建NFT信息的`nft`对象。NFT的地址与设置Metaplex实例时使用的用户Identity Driver程序相对应。

### 6. 创建NFT

现在我们已经设置了Metaplex实例，并创建了用于上传元数据和创建NFT的辅助函数，我们可以通过创建一个NFT来测试这些函数。在`main()`函数中，调用`uploadMetadata`函数上传NFT数据并获取元数据的URI。然后，使用`createNft`函数和元数据URI来创建一个NFT。

```tsx
async function main() {
	...

  // 上传NFT数据并获取元数据的URI
  const uri = await uploadMetadata(metaplex, nftData)

  // 使用辅助函数和元数据中的URI创建一个NFT
  const nft = await createNft(metaplex, uri, nftData)
}
```

在命令行中运行`npm start`来执行`main`函数。你应该会看到类似以下的输出：

```tsx
Current balance is 1.770520342
PublicKey: GdLEz23xEonLtbmXdoWGStMst6C9o3kBhb7nf7A1Fp6F
image uri: https://arweave.net/j5HcSX8qttSgJ_ZDLmbuKA7VGUo7ZLX-xODFU4LFYew
metadata uri: https://arweave.net/ac5fwNfRckuVMXiQW_EAHc-xKFCv_9zXJ-1caY08GFE
Token Mint: https://explorer.solana.com/address/QdK4oCUZ1zMroCd4vqndnTH7aPAsr8ApFkVeGYbvsFj?cluster=devnet
Finished successfully
```

你可以检查生成的图像和元数据的URI，并通过访问输出中提供的URL在Solana浏览器中查看NFT。

### 7. `updateNftUri` 辅助函数

接下来，让我们创建一个辅助函数来处理更新现有NFT的URI。这个函数将接受Metaplex实例、元数据URI和NFT的铸造地址作为输入。它使用SDK的`findByMint`方法使用铸造地址获取现有NFT数据，然后使用`update`方法使用新URI更新元数据。最后，它将记录令牌铸造的URL和交易签名以供参考。

```tsx
// 辅助函数，更新NFT
async function updateNftUri(
  metaplex: Metaplex,
  uri: string,
  mintAddress: PublicKey,
) {
  // 使用铸造厂地址获取NFT数据
  const nft = await metaplex.nfts().findByMint({ mintAddress });

  // 更新NFT元数据
  const { response } = await metaplex.nfts().update(
    {
      nftOrSft: nft,
      uri: uri,
    },
    { commitment: "finalized" },
  );

  console.log(
    `Token Mint: https://explorer.solana.com/address/${nft.address.toString()}?cluster=devnet`,
  );

  console.log(
    `Transaction: https://explorer.solana.com/tx/${response.signature}?cluster=devnet`,
  );
}
```

### 8. 更新NFT

要更新现有的NFT，我们首先需要上传新的NFT元数据并获取对应的URI。在`main()`函数中，再次调用`uploadMetadata`函数上传更新的NFT数据并获取元数据的新URI。然后，我们可以使用`updateNftUri`辅助函数，传入Metaplex实例、来自元数据的新URI以及NFT的铸造厂地址。`nft.address`是`createNft`函数的输出。

```tsx
async function main() {
	...

  // 上传更新后的NFT数据并获取元数据的新URI
  const updatedUri = await uploadMetadata(metaplex, updateNftData)

  // 使用辅助函数和来自元数据的新URI更新NFT
  await updateNftUri(metaplex, updatedUri, nft.address)
}
```

在命令行中运行`npm start`来执行`main`函数。你应该会看到类似以下的额外输出：

```tsx
...
Token Mint: https://explorer.solana.com/address/6R9egtNxbzHr5ksnGqGNHXzKuKSgeXAbcrdRUsR1fkRM?cluster=devnet
Transaction: https://explorer.solana.com/tx/5VkG47iGmECrqD11zbF7psaVqFkA4tz3iZar21cWWbeySd66fTkKg7ni7jiFkLqmeiBM6GzhL1LvNbLh4Jh6ozpU?cluster=devnet
成功完成
```

你也可以通过从`.env`文件中导入`PRIVATE_KEY`到Phantom钱包中查看NFT。

### 9. 创建NFT集合

太棒了，你现在知道如何在Solana区块链上创建单个NFT并对其进行更新了！但是，如何将它添加到一个集合中呢？

首先，让我们创建一个名为`createCollectionNft`的辅助函数。请注意，它与`createNft`非常相似，但确保`isCollection`设置为true，并且数据符合集合的要求。

```tsx
async function createCollectionNft(
  metaplex: Metaplex,
  uri: string,
  data: CollectionNftData
): Promise<NftWithToken> {
  const { nft } = await metaplex.nfts().create(
    {
      uri: uri,
      name: data.name,
      sellerFeeBasisPoints: data.sellerFeeBasisPoints,
      symbol: data.symbol,
      isCollection: true,
    },
    { commitment: "finalized" }
  )

  console.log(
    `Collection Mint: https://explorer.solana.com/address/${nft.address.toString()}?cluster=devnet`
  )

  return nft
}
```

接下来，我们需要创建集合的链下数据。在`main` *之前* 对创建NFT的现有代码进行修改，添加以下`collectionNftData`：

```tsx
const collectionNftData = {
  name: "TestCollectionNFT",
  symbol: "TEST",
  description: "Test Description Collection",
  sellerFeeBasisPoints: 100,
  imageFile: "success.png",
  is

Collection: true,
  collectionAuthority: user,
}
```

现在，让我们调用`uploadMetadata`与`collectionNftData`，然后调用`createCollectionNft`。再次强调，这应该在创建NFT的代码之前进行。

```tsx
async function main() {
  ...

  // 上传集合NFT的数据并获取元数据的URI
  const collectionUri = await uploadMetadata(metaplex, collectionNftData)

  // 使用辅助函数和元数据的URI创建集合NFT
  const collectionNft = await createCollectionNft(
    metaplex,
    collectionUri,
    collectionNftData
  )
}
```

这将返回我们集合的铸造地址，因此我们可以使用它来将NFT添加到集合中。

### 10. 将NFT添加到集合

现在我们有了一个集合，让我们修改现有代码，使新创建的NFT可以添加到集合中。首先，让我们修改`createNft`函数，以便在调用`nfts().create`时包含`collection`字段。然后，添加调用`verifyCollection`的代码，以便将链上元数据的`verified`字段设置为true。这样，消费程序和应用程序可以确保NFT确实属于集合。

```tsx
async function createNft(
  metaplex: Metaplex,
  uri: string,
  nftData: NftData
): Promise<NftWithToken> {
  const { nft } = await metaplex.nfts().create(
    {
      uri: uri, // 元数据URI
      name: nftData.name,
      sellerFeeBasisPoints: nftData.sellerFeeBasisPoints,
      symbol: nftData.symbol,
    },
    { commitment: "finalized" }
  )

  console.log(
    `Token Mint: https://explorer.solana.com/address/${nft.address.toString()}? cluster=devnet`
  )

  //这是我们将集合验证为认证集合的方法
  await metaplex.nfts().verifyCollection({  
    mintAddress: nft.mint.address,
    collectionMintAddress: collectionMint,
    isSizedCollection: true,
  })

  return nft
}
```

现在，运行`npm start`，完成！如果你按照新的NFT链接并查看元数据选项卡，你将看到一个包含你集合铸造地址的`collection`字段。

恭喜！你已成功学会使用Metaplex SDK创建、更新和验证NFT，并将其作为集合的一部分。这就是你需要为几乎任何用例构建自己的集合所需的一切。你可以建立一个TicketMaster的竞争对手，改进Costco的会员计划，甚至将你学校的学生ID系统数字化。可能性是无限的！

如果你想查看最终解决方案代码，你可以在同一个[代码库](https://github.com/Unboxed-Software/solana-metaplex/tree/solution)的解决方案分支中找到它。

# 挑战

为了加深对Metaplex工具的理解，深入研究Metaplex文档，熟悉Metaplex提供的各种程序和工具。你可以深入了解糖果机程序以了解其功能。

一旦你了解了糖果机程序的工作原理，就可以通过使用Sugar CLI为自己的集合创建一个糖果机来将你的知识付诸实践。这种亲身体验不仅会加强你对工具的理解，还会增强你未来有效使用它们的信心。

玩得开心！这将是你第一次独立创建的NFT集合！有了这个，你就完成了第二模块。希望你能感受到这个过程！随时[分享一些快速反馈](https://airtable.com/shrOsyopqYlzvmXSC?prefill_Module=Module%202)，以便我们继续改进课程！

## 完成了实验吗？

将你的代码推送到GitHub，并[告诉我们你对这堂课的看法](https://form.typeform.com/to/IPH0UGz7#answers-lesson=296745ac-503c-4b14-b3a6-b51c5004c165)！