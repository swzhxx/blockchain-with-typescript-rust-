# 3:交易（Transactions）

## 概述

在本章中，我们将介绍交易的概念。通过这一修改，我们实际上从我们的项目从“通用”区块链转向了加密货币。因此，如果我们能证明我们一开始就拥有硬币，我们就可以把硬币寄到地址。

为了实现这一切，必须提出许多新概念。这包括公钥加密、签名和事务输入和输出。

## 公钥密码和签名

在公钥加密中，您有一个密钥对：密钥和公钥。公钥可以从私钥派生，但是私钥不能从公钥派生。公钥可以安全地共享给任何人。

任何消息都可以使用私钥来创建签名。使用此签名和相应的公钥，任何人都可以验证签名是由于公钥匹配的私钥生成的。

我们使用[elliptic](https://github.com/indutny/elliptic)来生成作为公钥的加密算法，它可以生成椭圆曲线(=[ECDSA](https://en.wikipedia.org/wiki/Elliptic_Curve_Digital_Signature_Algorithm))

最后，在密码货币中，两种不同的加密函数用于不同的目的

- 用于工作证明挖掘的哈希函数（SHA256）（哈希还用于保持块完整性）
- 公钥加密（ECDSA）

## 私钥和公钥（在 ECDSA 中）

有效的私钥是任何随机的 32 字节字符串，列如：
`19f128debc1b9122da0635954488b208b829879cf13b3d6cac5d1260c0fd967c`
有效的公钥是‘04’与 64 字节字符串连接在一起，例如：
`04bfcab8722991ae774db48f934ca79cfb7dd991229153b9f732ba5334aafcd8e7266e47076996b55a14bf9913ee3145ce0cfc1372ada8ada74bd287450313534a`

公钥可以从私钥派生。公钥将在交易中用作硬币的“接受者”

## 交易概述

在编写任何代码之前，我们先概述一下交易的结构。交易包括两个部分：输入和输出。输出指定将硬币发送到的松子，输入可以证明实际发送的硬币首先存在并且由“发件人”拥有。输入始终引用现有（未使用）的输出

## 交易输出

交易输出（txOut）由一个地址和一定数量的硬币组成。该地址是 ECSDSA 公钥。这意味着具有引用的公钥的私钥用户将能够访问硬币

```ts static
class TxOut {
  contructor(public address: string, public amount: number) {}
}
```

## 交易输入

交易输入（txIn）提供了“硬币来自何处”的信息。每个 txIn 都指向一个较早的输出，带有签名的硬币从中被“解锁”。现在，txOut 可以使用这些未锁定的硬币。该签名提供了证明，自由具有所引用的公钥（=地址）的私钥的用户才可以创建交易

```ts static
class TxIn {
  public txOutId: string
  public txOutIndex: number
  public signature: string
}
```

应该注意的是，txIn 仅包含签名（由私钥创建），而不包含私钥本身。区块链包含公钥和签名，从不包含私钥。

作为结论，也可以认为 txIns 可以解锁硬币，而 txOuts 可以“重新锁定”硬币

## 交易结构

交易结构本身非常简单，因为我们现在定义 txIns 和 txOuts

```ts static
class Transaction {
  public id: string
  public txIns: TxIn[]
  public txOuts: TxOut[]
}
```

## 交易 id

交易 Id 是通过从交易内容中获取哈希值来计算的。但是，txId 的签名不包括在事务哈希中，因为稍后会将其添加到事务中。

```ts static
const getTransactionId = (transaction: Transaction): string => {
  const txInContent: string = transaction.txIns
    .map((txIn: TxIn) => txIn.txOutId + txIn.txOutIndex)
    .reduce((a, b) => a + b, '')
  const txOutContent: string = transaction.txOuts
    .map((txOut: TxOut) => txOut.address + txOut.amount)
    .reduce((a, b) => a + b, '')

  return CryptoJS.SHA256(txInContent + txOutContent).toString()
}
```

## 交易签名

重要的是，在签署后，交易内容不能更改。由于交易是公开的，因此任何人都可以访问加油，甚至可以将交易包括在区块链之前。
在对交易输入进行签名时，将仅对 txId 进行签名。如果修改了事务中的任何内容，则必须修改 txId，从而使交易和签名无效。

```ts static
const signTxIn = (
  transation: Transaction,
  txInIndex: number,
  privateKey: string,
  aUnspentTxOuts: UnspentTxOut[]
): string => {
  const txIn: TxIn = transaction.txIns[txIndex]
  const dataToSign = transaction.id
  const referencedUnspentTxOut: UnspentTxOut = findUnspentTxOut(
    txIn,
    txOutId,
    txIn.txOutIndex,
    aUnspentTxOuts
  )
  const referencedAddress = referencedUnspentTxOut.address
  const key = ec.keyFromPrivate(privateKey, 'hex')
  const signature: string = toHexString(key.sign(dataToSign).toDER())
  return signature
}
```

让我们尝试了解如何有人尝试修改交易会发生什么情况：

1. 攻击者运行一个节点并接受一个包含以下内容的交易：“将 10 个硬币从地址 AAA 发送到 BBB”，其 txId 为 0x555...
2. 攻击者将接收者地址更改为 CCC，并将其转发到网络中。现在的交易内容是“从地址 AAA 向 CCC 发送 10 个硬币”
3. 但是随着接收器地址的更改，txId 不再有效。新的有效 txId 为 0x567...
4. 如果 txId 设置为新值，则签名无效。签名仅与原始 txId 0x555 匹配
5. 修改后的事务将不会被其他节点接受，因为无论哪种方式，它都是无效的。

## 未使用的交易输出（Unspent transaction outputs）

交易输入必须始终引用未使用的交易输出。因此，但您在去快乐中拥有一些硬币时，实际拥有的是未使用的交易输出列表，其公钥与您拥有的私钥匹配

在交易验证方面，为了确定交易是否有效，我们只能关注未使用的交易输出列表。未使用的交易输出列表始终可以从当前的区块链中得出。在此实现中，我们将在处理过程中更新未用交易输出的列表，并将交易包括在区块链中。

未使用的交易输出结构如下所示：

```ts static
class UnspentTxOut {
  constructor(
    public readonly txOutId: string,
    public readonly txOutIndex: number,
    public readonly address: string,
    public readonly amount: number
  ) {}
}
```

数据结构本身，只是一个列表：

```ts static
let unspentTxOuts: UnspentTxOut[] = []
```

## 更新未使用的交易输出

每次将新块添加到链中时，我们都必须更新未使用的交易输出列表。这是因为新交易将花费一些现有交易输出并引入新的未使用输出。
为了解决这个问题，我们将首先从新块中检索所有新的未花费的交易输出（newUnspentTxOuts）

```ts static
const newUnspentTxOuts: UnspentTxOut[] = new Transactions.map((t) => {
  return t.txOuts.map(txOut , index) => new UnspentTxOut(t.id , index, txOut.address,txOut.amount)
}).reduce((a,b)=>a.concat(b) , [])
```

我们还需要知道该块的新交易(`consumedTxouts`)消耗了哪些交易输出。这将通过检查新交易的输入来解决：

```ts static
const consumedTxOuts: UnspentTxOut[] = new Transactions.map((t) => t.txIns)
  .reduce((a, b) => a.concat(b), [])
  .map((txIn) => {
    return new UnspentTxOut(txIn, txOutId, txIn.txOutIndex, '', 0)
  })
```

最后，我们可以通过删除 consumedTxOuts 并将 newUnspentTxOuts 添加到我们现有的交易输出中来生成新的未花费的交易输出。

```ts static
const resultingUnspentTxOuts = aUnspentTxOuts
  .filter(
    (uTxO) => !findUnspentTxOut(uTxO.txOutId, uTxO.txOutIndex, consumedTxOuts)
  )
  .concat(newUnspentTxOuts)
```

所描述的代码和功能包含在 updateUnspentTxOuts 方法中。应该注意的是，自由在验证了区块中的交易之后，才调用此方法。

## 交易验证

现在，我们终于可以列出使交易有效的规则：

### 正确的交易结构

交易必须符合`Transaction`，`TxIn`，`TxOut`的已定义类

```ts static
const isValidTransactionStructure = (transaction: Transaction) => {
  if (typeof transaction.id !== 'string') {
    console.log('transactionId missing')
    return false
  }
  //....
}
```

### 有效交易编号

交易中 ID 必须正确计算。

```ts static
if (getTransactionId(transaction) !== transaction.id) {
  console.log(`invalid tx id : ${transaction.id}`)
  return false
}
```

### 有效的 txIns

txIns 中的签名必须有效，并且必须未使用引用的输出。

```ts static
const validateTxIn = (
  txIn: TxIn,
  transaction: Transaction,
  aUnspentTxOuts: UnspentTxOut[]
): boolean => {
  const referencedUTxOut: UnspentTxOut = aUnspentTxOuts.find(
    (uTxO) => uTxO.txOutId === txIn.txOutId && uTxO.txOutId === txIn.txOutId
  )
  if (referencedUTxOut == null) {
    console.log(`referenced txOut not found : ${JSON.stringify(txIn)}`)
    return false
  }
  const address = referencedUTxOut.address
  const key = ec.keyFromPublic(address, 'hex')
  return key.verify(transaction.id, txIn.signature)
}
```

### 有效的 txOut 值

在输出中指定的值的总和必须等于在输入中指定的值的总和。如果引用的输出包含 50 个硬币，则新输出中的值总和也必须为 50 个硬币。

```ts static
const totalTxInValues: number = transaction.txIns
  .map((txIn) => getTxInAmount(txIn, aUnspentTxOuts))
  .reduce((a, b) => a + b, 0)
const totalTxOutValues = transaction.txOuts
  .map((txOut) => txOut.amount)
  .reduce((a, b) => a + b, 0)

if (totalTxOutValues !== totalTxInValues) {
  console.log(`totalTxOutValues !== totalTxInValues in tx : ${transaction.id}`)
  return false
}
```

## Coinbase 交易

交易输入必须始终引用未使用的交易输出，但是初始代币从何处进入区块链？为了解决这个问题，引入了一种特殊类型的交易：coinbase 交易。

Coinbase 交易仅包含输出，但不包括输入。这意味着基于硬币的交易增加了新硬币的流通量。我们制定的 coinbase 输出量为 50 个硬币

```ts static
const COINBASE_AMOUNT: number = 50
```

Coinbase 交易始终是区块中的第一笔交易，并且被区块的矿工包括在内。币值简历是对矿工的将激励：如果您找到该区块，就可以收集 50 个硬币。

我们将区块高度添加到 coinbase 交易的输入中。这是为了确保每个 coinbase 交易都具有唯一的 txId。例如，如果没有此规则，则表示“将 50 个硬币分配给地址 0xabc”的 coinbase 交易将始终具有相同的 txId。

Coinbase 交易的验证于“正常”交易的验证略有不同。

```ts static
const validateCoinbaseTx = (
  transaction: Transaction,
  blockIndex: number
): boolean => {
  if (getTransactionId(transaction) !== transaction.id) {
    console.log(`invalid coinbase tx id : ${transaction.id}`)
    return false
  }
  if (transaction.txIns.lengths !== 1) {
    console.log(`one txIn must be specified in the coinbase transaction`)
    return
  }

  if (transaction.txIns[0].txOutIndex !== blockIndex) {
    console.log(`the txIn index in coinbase tx must be the block height`)
    return false
  }
  if (transaction.txOuts.length !== 1) {
    console.log(`invalid number of txOuts in coinbase transaction`)
    return false
  }
  if (transaction.txOuts[0].amount !== COINBASE_AMOUNT) {
    console.log(`invalid coinbase amount in coinbase transaction`)
  }
  return true
}
```

## 结论

我们将交易的概念纳入了区块链。基本思想很简单：我们在交易输入中应用未使用的输出，并使用签名来显示解锁部分是有效的。然后，我们使用输出将它们“重新锁定”到接收者地址。
但是，创建交易仍然非常困难。我们必须手动创建交易的输入和输出，并使用我们的私钥对其进行签名。当我们在下一章节介绍钱包时，这种情况将会改变。
还没有交易中继：要将交易包含到区块链中，您必须自己进行挖掘。这也是我们尚未引入交易费用概念的原因。
