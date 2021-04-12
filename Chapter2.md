# 2:工作证明（Proof of Work）

## 概述

在本章中，我们将对玩具区块链版本实施一个简单的工作证明方案。在第一章的版本中，任何人都可以免费为链添加块。通过工作证明，我们引入了一个需要解决的计算难题，然后才能将块添加到区块链中。视图解决这个难题通常被称为“采矿”

通过工作证明，我们还可以控制（大约）将块引入区块链的频率间隔。这是通过改变拼图的难度来实现的。如果区块开采过于频繁，谜题的难度就会增加，反之亦然。

需要主要的是，我们在本章中还没有介绍交易。这意味着实际上没有没有激励旷工开采区块。通常在加密货币中，旷工发现区块会得到奖励，但在我们的区块链中还不是这样。

## 难度、暂时性与工作证明难题

我们将为块结构添加两个新属性：`difficulty`和`nonce`。要理解这些的含义，我们必须首先介绍工作证明难题。

工作证明难题是找到一个块散列，它有一个特定数量的零前缀。`difficulty`属性定义块哈希必须有多少前缀零，为了使块有效。从哈希的二进制格式检查前缀零。

检查哈希在难度方面是否正确的代码：

```ts static
const hashMatchDifficulty = (hash: string, difficulty: number): boolean => {
  const hashInBinary: string = hexToBinary(hash)
  const requiredPrefix: string = '0'.repeat(difficulty)
  return hashInBinary.startsWith(requiredPrefix)
}
```

为了找到满足难度的哈希，我们必须能够为块的相同内容计算不同的哈希值。这是通过修改`nonce`参数来实现的。因为 SHA256 是一个哈希函数，每次块中的任何内容发生变化时，哈希值都会完全不同。“挖掘”基本上只是尝试一个不同的暂时性，知道块哈希匹配难度。

现在添加了`difficulty`和`nonce`，块结构如下所示

```ts static
class Block {
  constructor(
    public index: number,
    public hash: string,
    public previousHash: string,
    public timestamp: number,
    public data: string,
    public difficulty: number,
    public nonce: number
  ) {}
}
```

我们还必须记住更新创世区块（gensis block）!

## 找到一个块

如上所述，为了找到一个有效的块哈希，我们必须增加 nonce 知道得到一个有效的散列。找到一个满意的哈希完全是一个随机过程。我们必须循环使用足够的 nonce，直到找到满意的哈希：

```ts static
const findBlock = (
  index: number,
  previousHash: string,
  timestamp: number,
  data: string,
  difficulty: number
): Block => {
  let nonce = 0
  while (true) {
    const hash: string = caculateHash(
      index,
      previousHash,
      timestamp,
      data,
      difficulty,
      nonce
    )

    if (hashMatchesDifficulty(hash, difficulty)) {
      return new Block(
        index,
        hash,
        previousHash,
        timestamp,
        data,
        difficulty,
        nonce
      )
    }
    nonce++
  }
}
```

当快被发现时，它被广播到网络上，就像第一章中的情况一样

## 关于困难的共识

我们现在有办法找到并验证给定难度的哈希，但是如何确定难度呢？必须有一种方法让节点同意当前的困难是什么。为此，我们引入了一些新的规则来计算当前网络的难度。

让我们为网络定义以下新常量

- `BLOCK_GENERATION_INTERVAL` 定义应找到块的频率。（在比特币中，这个值是 10 分钟）

- `DIFFICULTY_ADJUSTMENT_INTERVAL` 定义根据网络哈希速率的增加或减少调整难度的频率。（比特币值为 2016 块）

我们将设置块生成间隔为 10 秒，难度调整为 10 块。这些常数不会随时间而改变，他们是硬编码的。

```ts static
// in seconds
const BLOCK_GENERATION_INTERVAL: number = 10

// in blocks
const DIFFICULTY_ADJUSTMENT_INTERVAL: number = 10
```

现在我们有办法就这一块的难点达成一致。对于生成的每 10 个块，我们检查生成这些块所用的时间是否大于或小于预期时间。预期时间的计算方式如下：`BLOCK_GENERATION_INTERVAL * DIFFICULTY_ADJUSTMENT_INTERVAL`预期时间表示 hashrate 与当前难度完全匹配的情况

如果所花费的时间比预期的难度至少大两倍或下两倍，我们要么增加难度，要么减少难度。难度调整由以下代码处理

```ts static
const getDifficulty = (aBlockchain: Block[]): number => {
  const latestBlock: block = aBlockchain[blockchain.length - 1]
  if (
    latestBlock.index % DIFFICULTY_ADJUSTMENT_INTERVAL == 0 &&
    latestBlock.index !== 0
  ) {
    return getAdjustDifficulty(latestBlock, aBlockchain)
  } else {
    return latestBlock.difficulty
  }
}

const getAdjustDifficulty = (latestBlock: Block, aBlockchain: Block[]) => {
  const prevAdjustmentBlock: Block =
    aBlockchain[blockchain.length - DIFFICULTY_ADJUSTMENT_INTERVAL]

  const timeExpected: number =
    BLOCK_GENERATION_INTERVAL * DIFFICULTY_ADJUSTMENT_INTERVAL

  const timeTaken: number =
    latestBlock.timestamp - prevAdjustmentBlock.timestamp

  if (timeTaken < timeExpected / 2) {
    return prevAdjustmentBlock.difficulty + 1
  } else if (timeTaken > timeExpected * 2) {
    return prevAdjustmentBlock.difficulty - 1
  } else {
    return prevAdjustmentBlock.difficulty
  }
}
```

## 时间戳验证

在区块链的第一章版本中，时间戳没有任何角色或验证。事实上，它可以是客户决定生成的任何内容。由于`timeTaken`变量是基于块的时间戳计算的，因此引入了难度条痕，所以这一点发生了变化。

为了减轻引入假时间戳以操作难度的攻击，引入了一下规则：

- 如何时间戳距离我们感知的时间最多一分钟，则一个块是有效的。
- 如果时间戳在前一个块的过去最多一分钟，则链中的块是有效的。

```ts static
const isValidTimestamp = (newBlock: Block, previousBlock: Block): boolean => {
  return (
    previousBlock.timestamp - 60 < newBlock.timestamp &&
    newBlock.timestamp - 60 < getCurrentTimestamp()
  )
}
```

## 累计难度

在区块链的第一章版本中， 我们总是选择“最长”的区块链作为有效的区块链。既然`difficulty`已经出现，这种情况就必须改变。就目前而言，“正确”的链条不是“最长”的链条，但是链条的累计难度最大，换句话说，正确的链是需要大部分资源(=hashRate \* time)才能产生的链。

为了得到链的累计难度，我们计算每个块的 `2^difficulty`，并取所有这些数字的和。我们必须使用`2^difficulty`因此我们选择`difficulty`来表示必须以二进制格式作为哈希前缀的零的数量。例如，我们比较 5 和 11 的难度，就需要 2^（11-5） = 2^6 倍的工作量能找到后一种难度的块

只有块的难度才重要，而不是实际的哈希。例如难度为 4 且块哈希为 000000a34...(=也满足难度 6)，这在计算累计难度时仅考虑难度 4。

这个属性也被成为中本共识(Nakamoto consensus)，它是佐藤最重要的发明之一，当他发明比特的时候。在分叉的情况下，旷工必须选择他们决定将当前的资源放在哪条链上（=hashRate）。由于生产这种区块链中包含的区块符合矿商的利益，矿商被激励最终选择相同的区块链。

## 结论

工作证明难题必须具备一个重要特性是，它很难解决，但很容易验证。找到特定的 SHA256 散列是解决这类问题的一个简单而好的例子。

我们实现了难度方面，节点现在必须挖掘，以便向链中添加新的块。在下一章中，我们将实现事务
