# 5：交易中继（Transaction relaying）

在本章中，我们将实现此类交易的中继，而这些交易尚未包含在区块链中。在比特币中，这些交易也称为“未经确认的交易”。通常，当某人想要将交易包括到区块链中（=将硬币发送到某个地址）时，他会将交易广播到网络，并希望某个节点将交易挖掘到区块链中。

此功能对于正在运行的加密货币非常重要，因为它意味着您无需自己挖出一个区块即可将交易包括到区块链中。

因此，当节点彼此通信时，他们现在将共享两种类型的数据：

- 区块链的状态（=包含在区块链中的区块和交易）
- 未确认的交易（=包含在区块链中的交易）

## 交易池（Transaction pool）

我们会将未经确认的交易存储在名为“交易池”（在比特币中也成为“内存池”）的新实体中。交易池是一个包含我们节点只到的所有“未经确认交易”的结构。在这个简单的实现中，我们将只使用好一个列表。

```ts static
let transactionPool: Transaction[] = []
```

我们还将向节点引入一个新的断点：`POST/sendTransaction`。此方法根据现有的钱包功能向本地交易池创建交易。现在，当我们要向区块链包括新交易时，我们将使用此方法作为“首选”接口

```ts static
 app.post('/sendTransaction', (req, res) => {
        ...
    })
```

我们就像在第四章中一样创建交易。我们只是将创建的交易添加到池中，而不是立即尝试挖掘一个块：

```ts static
const sendTransaction = (address: string, amount: number): Transaction => {
  const tx: Transaction = createTransaction(
    address,
    amount,
    getPrivateFromWallet(),
    getUnspentTxOuts(),
    getTransactionPool()
  )
  addToTransactionPool(tx, getUnspentTxOuts())
  return tx
}
```

## 广播（Broadcasting）

未经确认交易的全部要点是他们将散布在整个网络中，最终某个节点会将交易挖掘到区块链。

- 当节点接收到从未出现过的未确认交易时，它将向所有对等方广播完整的交易池
- 当一个节点首次连接到另一个节点时，它将查询该节点的交易池

为此，我们将添加两个新的 MessageType:`QUERY_TRANSACTION_POOL`和`RESPONSE_TRANSACTION_POOL`。现在，MessageType 枚举将看起来像这样

```ts static
enum MessageType {
  QUERY_LATEST = 0,
  QUERY_ALL = 1,
  RESPONSE_BLOCKCHAIN = 2,
  QUERY_TRANSACTION_POOL = 3,
  RESPONSE_TRANSACTION_POOL = 4,
}
```

交易池消息将通过以下方式创建：

```ts static
const responseTransactionPoolMsg = (): Message => ({
  type: MessageType.RESPONSE_TRANSACTION_POOL,
  data: JSON.stringify(getTransactionPool()),
})

const queryTransactionPoolMsg = (): Message => ({
  type: MessageType.QUERY_TRANSACTION_POOL,
  data: null,
})
```

为了实现所描述的交易广播逻辑，我们添加代码来处理`MessageType.RESPONSE_TRANSACTION_POOL` 消息类型。每当我们收到未确认的交易时，我们都会尝试将其添加到我们的交易池中。如果我们设法将交易添加到池中，则意味着该交易有效，并且我们的节点之前从未看到该交易。在这种情况下，我们将自己的交易池广播给所有对等方。

```ts static
case MessageType.RESPONSE_TRANSACTION_POOL:
    const receivedTransactions: Transaction[] = JSONToObject<Transaction[]>(message.data);
    receivedTransactions.forEach((transaction: Transaction) => {
        try {
            handleReceivedTransaction(transaction);
            //if no error is thrown, transaction was indeed added to the pool
            //let's broadcast transaction pool
            broadCastTransactionPool();
        } catch (e) {
            //unconfirmed transaction not valid (we probably already have it in our pool)
        }
    });
```

## 验证收到的未确认的交易（Validating received unconfirmed transactions）

由于对等方可以向我们发送任何类型的交易，因此我们必须先验证交易，然后才能将其添加到交易池中。所有现有的交易验证规则均适用。例如，交易必须正确格式化，并且交易输入，输出和签名必须匹配。

除了现有规则之外，我们还添加了一条新规则：如果在现有交易池中已经找到了任何交易输入，这无法将交易添加到池中，以下代码体现了这一规则：

```ts static
const isValidTxForPool = (
  tx: Transaction,
  aTtransactionPool: Transaction[]
): boolean => {
  const txPoolIns: TxIn[] = getTxPoolIns(aTtransactionPool)

  const containsTxIn = (txIns: TxIn[], txIn: TxIn) => {
    return _.find(txPoolIns, (txPoolIn) => {
      return (
        txIn.txOutIndex === txPoolIn.txOutIndex &&
        txIn.txOutId === txPoolIn.txOutId
      )
    })
  }

  for (const txIn of tx.txIns) {
    if (containsTxIn(txPoolIns, txIn)) {
      console.log('txIn already found in the txPool')
      return false
    }
  }
  return true
}
```

没有明确的方法可以从交易池中删除交易。但是，每次找到新块时，交易池都会更新。

## 从交易池到区块链

接下来，让我们为未确认的交易实现一种方法，以找到其从本地交易池到同一节点开采的区块的方式。这很简单：当节点开始挖掘区块时，它将包括从事务池到新区块候选者的交易。

```ts static
const gererateNextBlock = () => {
  const coinbaseTx: Transaction = getCoinbaseTransaction(
    getPublicFromWallet(),
    getLatestBlock().index + 1
  )
  const blockData: Transaction[] = [coinbaseTx].concat(getTransactionPool())
  return generateRawNextBlock(blockData)
}
```

由于已经验证了事务，因此在将它们添加到池之前，我们现在不会进行任何进一步验证。

## 更新交易池（Updaing the transaction pool）

随着具有交易的新区块被挖掘到区块链中，每次发现新区块时，我们都必须重新验证交易池。新块可能包含池中的某些交易无效的交易。例如，如果发生以下情况，可能会发生这种情况：

- 池中的交易已被挖掘（由节点本身或其他人挖掘）
- 在未确认的交易中应用的未使用的交易输出由其他交易使用

交易池将使用以下代码更新:

```ts static
const updateTransactionPool = (unspentTxOuts: UnspentTxOut[]) => {
  const invalidTxs = []
  for (const tx of transactionPool) {
    for (const txIn of tx.txIns) {
      if (!hasTxIn(txIn, unspentTxOuts)) {
        invalidTxs.push(tx)
        break
      }
    }
  }
  if (invalidTxs.length > 0) {
    console.log(
      'removing the following transactions from txPool: %s',
      JSON.stringify(invalidTxs)
    )
    transactionPool = _.without(transactionPool, ...invalidTxs)
  }
}
```

可以看出，我们只需要知道当前未使用的交易输出，就可以决定是否从池中删除事务。

结论：

现在，我们可以将交易包括到区块链中，而无需实际挖掘区块本身。但是，由于我们没有实现交易费用的概念，因此没有动力要求节点将受到的交易包括在区块中。
