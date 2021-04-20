# 4:Wallet(钱包)

## 概述

钱包的目的是为最终用户创建一个更加抽象的界面

最终用户必须能够

- 创建一个新的钱包（=在这种情况下的私钥）
- 查看他的钱包余额
- 将硬币发送到其他地址

以上所有内容都必须有效，以便最终用户不必了解 txIns 或 txOuts 的工作方式。就像比特币：您将硬币发送到地址，并发布自己的地址，其他人可以在其中发送硬币。

## 生成并存储私钥

在本教程中，我们将使用最简单的方式来处理钱包的生成和存储：我们将为文件`node/wallet/private_key`生成未加密的私钥。

```ts static
const privateKeyLocation = `node/wallet/private_key`
const generatePrivateKey = (): string => {
  const keyPair = EC.genKeyPair()
  const privateKey = keyPair.getPrivate()
  return privateKey.toString(16)
}
const initWallet = () => {
  if (existsSync(privateKeyLocation)) {
    return
  }
  const newPrivateKey = generatePrivateKey()
  writeFileSync(privateKeyLocation, newPrivateKey)
  console.log('new wallet with private key created')
}
```

如前所述，可以从私钥中计算出公钥（=地址）

```ts static
const getPublicFromWallet = (): string => {
  const privateKey = getPrivateFromWallet()
  const key = EC.keyFromPrivate(privateKey, 'hex')
  return key.getPublic().encode('hex')
}
```

应该注意的是，以未加密的格式存储私钥是非常不安全的。我们这样做仅是为了保持目前的简单性。此外，该钱包仅支持单个私钥，因此您需要生成一个新的钱包以获取新的公共地址

## 钱包余额

上一章提醒：当您在区块链中有一些硬币时，实际上拥有的是未使用的交易输出列表，其公钥与您拥有的私钥匹配。
这意味着计算给定地址的余额非常简单：您只需对该地址“拥有”的所有未使用交易进行求和：

```ts static
const getBalance = (
  address: string,
  unspentTxOuts: UnspentTxOut[]
): number => {
  return _(unspentTxOuts).filter(uTxO:UnspentTxOut => uTxO.address === address).map((uTxO:UnspentTxOut)=> uTxO.amount)
  .sum()
}
```

如代码所示，不需要私钥来查询地址余额，因此，这意味着任何人都可以解决给定地址的余额。

## 产生交易

在发送硬币时，用户应该能够交易输入和输出的概念。但是，如果用户 A 的余额为 50 个硬币（即在一次交易输出中）并且用户想要向用户 B 发送 10 个硬币，该怎么办？

在这种情况下，解决方案是将 10 个比特币发送到用户 B 的地址，再将 40 个硬币发送回用户 A。全部交易输出必须始终用完，因此在将硬币分配给输出时必须完成拆分部分。

让我们来看一个更复杂的交易场景：

1. 用户 C 最初有 0 个硬币
2. 用户 C 收到 3 笔交易，分别价值 10、20、和 30 个硬币
3. 用户 C 希望向用户 D 发送 55 个硬币。交易会是什么样的

在这种情况下，必须使用所有三个输出，并且输出必须具有对用户 D 55 枚硬币和对用户 C 5 枚硬币的值

```ts static
const findTxOutsForAmount = (
  amount: number,
  myUnspentTxOuts: UnspentTxOut[]
) => {
  let currentAmount = 0
  const includeUnspentTxOuts = []
  for (const myUnspentTxOut of myUnspentTxOuts) {
    includeUnspentTxOuts.push(myUnspentTxOut)
    currentAmount = currentAmount + myUnspentTxOut.amount
    if (currentAmount >= amount) {
      const leftOverAmount = currentAmount - amount
      return { includeUnspentTxOuts, leftOverAmount }
    }
  }
  throw Error('not enough coins to send transaction')
}
```

我们还将计算`leftOverAmount`，这是我们将发送回我们地址的值。

由于我们有未使用的事务输出列表，因此我们可以创建事务的 txIns：

```ts static
const toUnsignedTxIn = (unspentTxOut: UnspentTxOut) => {
  const txIn: TxIn = new TxIn()
  txIn.txOutId = unspentTxOut.txOutId
  txIn.txOutIndex = unspentTxOut.txOutIndex
  return txIn
}
const { includedUnspentTxOuts, leftOverAmount } = findTxOutsForAmount(
  amoutn,
  myUnspentTxouts
)
const unsignedTxIns: TxIn[] = includeUnspentTxOuts.map(toUnsignedTxIn)
```

接下来，创建事务的两个 txOut：一个用于硬币接受者的 txOut，一个用于 leftOverAmount 的 txOut。如果 txIns 恰好具有所需值的确切数量，我们将不创建 leftOver 交易。

```ts static
const createTxOuts = (
  receiverAddress: string,
  myAddress: string,
  amount,
  leftOverAmount: number
) => {
  const txOut1: TxOut = new TxOut(receiverAddress, amount)
  if (leftOverAmount == 0) {
    return [txOut1]
  } else {
    const leftOverTx = new TxOut(myAddress, leftOverAmount)
    return [txOut1, leftOverTx]
  }
}
```

最后，我们计算交易 ID 并签署 txIns：

```ts static
const tx: Transaction = new Transaction()
tx.txIns = unsignedTxIns
tx.txOuts = createTxOuts(receiverAddress, myAddress, amount, leftOverAmount)
tx.id = getTrasactionId(tx)
tx.txIns = tx.txIns.map(txIn:TxIn,index:number)=> {
  txIn.signature = signTxIn(tx,index,privateKey,unspentTxOuts)
  return txIn
}
```

## Using the wallet

我们还要为钱包添加一个有意义的控制端点

```ts static
app.post('/mineTransaction', (req, res) => {
  const address = req.body.address
  const amount = req.body.amount
  const resp = generatenextBlockWithTransaction(address, amount)
  res.send(resp)
})
```

如图所示，最终用户必须仅提供该节点的地址和硬币数量。节点将计算其余部分。

## 结论

我们仅通过简单的交易生成就实现了一个幼稚的未加密钱包。尽管此交易生成算法永远也不会创建具有 2 个以上输出的交易，但应该注意的是，区块链本身支持任何数量的输出。您可以输入 50 个硬币，输出 5，15，和 30 个硬币来创建有效的交易，但是必须使用`/mineRawBlock`接口手动创建这些交易

此外，将所需交易包含在区块链中的唯一方法是自己进行挖掘。节点不交换有关尚未包含在区块链中的交易的信息。这将在下一章解决
