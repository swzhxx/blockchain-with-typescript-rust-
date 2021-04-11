# 1：最小的工作区块链

## 预览

区块链的基本概念非常简单：一个分布式数据库，维护一个不断正常的有序记录列表。在本站中，我们将实现这种区块链的玩具版本。在本章的末尾，我们将了解区块链的以下基本功能

- 定义的区块和区块链的结构
- 使用任意数据向区块链添加新区块的方法
- 与其他节点通信并同步区块链节点
- 一个简单的 httpapi 来控制节点

## 块状结构

我们将从定义块结构开始。此时块中只包含最基本的树形

**index**：区块链中区块的高度

**data**：包含在块中的任何数据

**timestamp**：时间戳

**hash**：从块的内容提取的 sha256 哈希

**previousHash**：对上一个块的哈希的引用。此值显示定义上一个块

```ts static
class Block {
  // public index: number
  // public hash: string
  // public previousHash: string
  // public timestamp: number
  // public data: string
  constructor(
    public index: number,
    public hash: string,
    public previousHash: string,
    public timestamp: number,
    public data: string
  )
}
```

## Block Hash

块哈希是块最重要的属性之一。对块的所有数据计算哈希。这意味着，如果块中的任何内容发生更改，原始哈希值将不再有效。块哈希也可以被认为是块的唯一标识符。例如，可以出现具有相同索引的块，但他们都有唯一的哈希

我们使用以下代码计算块的哈希：

```ts static
const calculateHash (index:number , previousHash:string , timestamp:number , data:string) :string => {
  return CryptoJS.SHA256(index + previousHash + timestamp + data).toString()
}
```

应该注意的是，块哈希还没有与挖掘无关，因为没有工作问题的证明（ [proof-of-work](https://en.wikipedia.org/wiki/Proof_of_work)）可以解决。我们使用库唉哈希来保持块的完整性，并显式引用前一个块。

hash 和 previousHash 属性的一个重要结果是，如果不更改每个连续块的 hash,就不能修改块。

在引入工作证明时(proof-of-work)，这是一个特别重要的属性。区块链中的区块越深，修改它就越困难，因为它需要修改每个联系的区块。

## 创世区块(Gensis block)

创世区块是区块链中的第一个区块。它是唯一没有 previousHash 的块。我们见吧 genesis 块硬编码为源代码

```ts static
const genesisBlock: Block = new Block(
  0,
  '816534932c2b7154836da6afc367695e6337db8a921823784c14378abed4f7d7',
  null,
  1465154705,
  'my genesis block!!'
)
```

## 生成块

要生成一个块，我们必须知道前一个块的散列，并创建所需内容的其余部分（=index,hash , data and timestamp）。块数据由最终用户提供，但其余参数将使用以下代码生成

```ts static
const generateNextBlock = (blockData: string): Block => {
  const previousBlock: Block = getLatestBlock()
  const nextIndex: number = previousBlock.index + 1
  const nextTimestamp: number = new Date().getTime() / 1000
  const nextHash: string = calculateHash(
    nextIndex,
    previousBlock.hash,
    nextTimestamp,
    blockData
  )
  const newBlock: Block = new Block(
    nextIndex,
    nextHash,
    previousBlock.hash,
    nextTimestamp,
    blockData
  )
  return newBlock
}
```

## 存储区块链

目前我们只是用内存中的 javascript 数组来存储区块链。这意味着当节点终止时，数据将不会被持久化。

```ts static
const blockchain: Block[] = [genesisBlock]
```

## 验证块的完整性

在任何给定的时间，我们必须能够验证一个块或一个块链在完整性方面是否有效。当我们从其他节点接收到新的块并且必须决定是否接受他们时，尤其如此。

要使块有效，必须应用一下内容：

- 块的索引必须必前一个大一个数字
- 块的前一个哈希值与前一个块的哈希值匹配
- 块本身的哈希必须有效

下面的代码演示了这一点：

```ts static
const isValidNewBlock = (newBlock: Block, previousBlock: Block): boolean => {
  if (previousBlock.index + 1 !== newBlock.index) {
    console.log('invalid index')
    return false
  } else if (previoushBlock.hash !== newBlock.previoushHash) {
    console.log('invalid previoushash')
    return false
  } else if (calculateHashForBlock(newBlock) !== newBlock.hash) {
    console.log(
      typeof newBlock.hash + ' ' + typeof calculateHashForBlock(newBlock)
    )
    console.log(
      'invalid hash: ' + calculateHashForBlock(newBlock) + ' ' + newBlock.hash
    )
    return false
  }
  return true
}
```

我们必须验证块的结构，以便对等方发送的格式错误的内容不会使节点崩溃。

```ts static
const isValidBlockStructure = (block: Block): boolean => {
  return (
    typeof block.index === 'number' &&
    typeof block.hash === 'string' &&
    typeof block.previousHash === 'string' &&
    typeof block.timestamp === 'number' &&
    typeof block.data === 'string'
  )
}
```

现在我们有了验证单个块的方法，我们可以继续验证完整的块链。我们首先检查链中的第一个块是否与 genesis 块匹配。之后，我们使用前面描述的方法验证每个连续块。这是我们使用以下代码演示的：

```ts static
const isValidChain = (blockchainToValidate: Block[]): boolean => {
  const isValidGenesis = (block: Block): boolean => {
    return JSON.stringify(block) === JSON.stringify(genesisBlock)
  }

  if (!isValidGenesis(blockchainToValidate[0])) {
    return false
  }

  for (let i = 1; i < blockchainToValidate.length; i++) {
    if (
      !isValidNewBlock(blockchainToValidate[i], blockchainToValidate[i - 1])
    ) {
      return false
    }
  }
  return true
}
```

## 选择最长的链

在给定的时间，链中应该总是只有一组显式的块。如果发生冲突我们选择具有最长块数的链。在下面的示例中，块 72:a350235b00 中引入的数据将不包括在区块链中，因为它将被较长的链覆盖。

这是使用以下代码实现的逻辑：

```ts static
const replaceChain = (newBlocks: Block[]) => {
  if (isValidChain(newBlocks) && newBlocks.length > getBlockchain().length) {
    console.log(
      'Received blockchain is valid. Replacing current blockchain with received blockchain'
    )
    blockchain = newBlocks
    broadcastLatest()
  } else {
    console.log('Received blockchain invalid')
  }
}
```

## 与其他节点通信

节点的一个重要部分是预其他节点共享和同步区块链。以下规则用于保持网络同步。

- 当一个节点生成一个新的块时，它向网络广播它
- 当一个节点连接到一个新的对等节点时，它会查询最新的块
- 当一个节点遇到一个索引大于当前已知的块时，它要么将该块添加到当前链中，要么查询整个区块链

我们将使用 websockets 进行点对点通信。每个节点的活动套接字存储在 `const sockets:WebSocket[]`变量中。不是用自动对等发现。必须手动添加对等点的位置

## 控制节点

用户必须能够以某种方式控制节点。这是通过设置 HTTP 服务器实现的

```ts static
const initHttpServer = (myHttpPort: number) => {
  const app = express()
  app.use(bodyParser.json())

  app.get('/blocks', (req, res) => {
    res.send(getBlockchain())
  })
  app.post('/mineBlock', (req, res) => {
    const newBlock: Block = generateNextBlock(req.body.data)
    res.send(newBlock)
  })
  app.get('/peers', (req, res) => {
    res.send(
      getSockets().map(
        (s: any) => s._socket.remoteAddress + ':' + s._socket.remotePort
      )
    )
  })
  app.post('/addPeer', (req, res) => {
    connectToPeers(req.body.peer)
    res.send()
  })

  app.listen(myHttpPort, () => {
    console.log('Listening http on port: ' + myHttpPort)
  })
}
```

用户能够通过以下方式与节点交互：

- 列出所有块
- 使用用户给定的内容创建新块
- 列出或添加对等店

## 结构

应该注意的是，节点实际上公开了两个 web 服务器：一个用于用户控制节点（HTTP 服务器）， 另一个用于节点之间的对等通信。（Websocket HTTP 服务器）
