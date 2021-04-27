# 6.钱包 UI 和区块链资源管理器（Wallet UI and blockchain explorer）

## 概述

在本章中，我们将为钱包添加一个 UI，并为我们的区块链创建区块链浏览器。我们的节点已经使用 HTTP 端点公开了其功能，因此我们将创建一个网页，向这些端点发出请求并可视化结果。

为了实现所有这些，我们必须添加一些额外的断点和节点逻辑，例如：

- 查询有关区块和交易的信息
- 查询有关特定地址的信息

## 新端点

让我们添加一个端点，用户可以从中查询特定的块（如果已知哈希值）。

```ts static
app.get('/block/:hash', (req, res) => {
  const block = _.find(getBlockchain(), { hash: req.params.hash })
  res.send(block)
})
```

查询特定交易也是如此

```ts static
app.get('/transaction/:id', (req, res) => {
  const tx = _(getBlockchain())
    .map((blocks) => blocks.data)
    .flatten()
    .find({ id: req.params.id })
  res.send(tx)
})
```

我们还想显示有关特定地址的信息。现在、我们返回该地址未用完的输出列表，因为根据此信息，我们可以例如计算该地址的总余额。

```ts static
app.get('/address/:address', (req, res) => {
  const unspentTxOuts: UnspentTxOut[] = _.filter(
    getUnspentTxOuts(),
    (uTxO) => uTxO.address === req.params.address
  )
  res.send({ unspentTxOuts: unspentTxOuts })
})
```

我们还可以添加有关给定地址的已用交易输出的信息，以可视化给定地址的完整历史记录。

## 区块链资源管理器

区块链资源管理器是一个用于可视化区块链状态的网站。区块链浏览器的典型用例是轻松检查给定地址的余额或验证给定交易是否包含在区块链中。

在我们的例子中，我们只是向节点发出一个 http 请求，并以某种有意义的方式显示响应。我们从不发出任何修改区块链状态的请求，因此构建区块链资源管理器就是将节点有意义的方式提供的信息可视化。
