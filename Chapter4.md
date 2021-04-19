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
