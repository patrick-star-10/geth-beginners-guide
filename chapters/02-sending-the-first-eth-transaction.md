# 第 2 章 发送第一笔 ETH 交易

本章承接[第 1 章](01-running-a-local-geth-dev-chain.md)的本地开发链环境，继续使用已经运行的 Geth 节点和 JavaScript 控制台。

接下来让开发账户 A 向新创建的收款账户 B 发送 1 ETH。

## 创建收款账户 B

正常来说，创建以太坊账户涉及生成私钥、推导公钥，再从公钥计算账户地址。

根据 [Geth 官方开发模式文档](https://geth.ethereum.org/docs/developers/dapp-developer/dev-mode)，当前教程推荐使用 Clef 创建新账户，而不是依赖旧教程中的 `personal.newAccount()`。本次本地实验使用 Geth 内置的账户管理命令创建一个由密码保护的 Keystore 账户：

```bash
cd ~/ethereum/go-ethereum

./build/bin/geth account new \
  --datadir ~/ethereum/dev-chain
```

按照提示输入密码并再次确认。创建成功后，会返回类似以下信息：

![使用 Geth 创建密码保护的 Keystore 账户](../assets/chapter-02/create-keystore-account.png)

这里有三个知识点：

| 概念 | 解释 |
| --- | --- |
| Private Key | 用来控制账户、签署交易的私钥 |
| Address | 根据公钥推导出的 20 字节地址 |
| Keystore | 使用密码加密保存私钥的文件 |

这个收款账户无需解锁，因为本次实验只让它接收 ETH。

创建完成后，回到 Geth 控制台执行：

```javascript
eth.accounts
```

正常情况下，你应该能看到两个账户：

![查看开发账户和新创建的收款账户](../assets/chapter-02/two-development-accounts.png)

接着定义两个变量：

```javascript
var sender = eth.accounts[0]
var receiver = eth.accounts[1]
```

检查收款账户的初始余额：

```javascript
eth.getBalance(receiver)
```

![新创建的收款账户余额为零](../assets/chapter-02/receiver-initial-balance.png)

新创建且没有获得过 ETH 的账户，余额应该是 `0`。

> 创建账户本身不是一笔链上交易。它主要是在本地生成密钥并保存到 Keystore，因此通常不会导致区块高度增加。

## 发送 1 ETH

在 Geth JavaScript 控制台中执行：

```javascript
var txHash = eth.sendTransaction({
  from: sender,
  to: receiver,
  value: web3.toWei(1, "ether")
})
```

然后查看交易哈希：

```javascript
txHash
```

如果交易成功提交，会返回一个以 `0x` 开头的 32 字节交易哈希。

![发送 1 ETH 并查看交易哈希](../assets/chapter-02/send-one-eth-transaction.png)

> 请保存**你自己的交易**返回的 `txHash`，第 3、5、6 章还会用到它。截图中的哈希只是本次实验的示例，不能直接照抄；重新连接控制台后，需要用自己保存的哈希恢复变量。

### 理解交易对象

刚才提交的交易对象包含三个核心字段：

![交易对象中的 from、to 和 value 字段](../assets/chapter-02/transaction-object-fields.png)

| 字段 | 作用 |
| --- | --- |
| `from` | 发送方地址 |
| `to` | 接收方地址 |
| `value` | 转账金额，单位是 Wei |

为什么不能直接写 `value: 1`？

因为以太坊原生的转账金额单位是 Wei，而不是 ETH。`web3.toWei(1, "ether")` 会将 1 ETH 转换为 `10^18` Wei。

控制台中出现的 `undefined` 并不是报错，而是 JavaScript 的正常行为。例如执行：

```javascript
var sender = eth.accounts[0]
```

这条语句先声明变量 `sender`，再将 `eth.accounts[0]` 的地址赋值给它。但是 `var` 变量声明语句本身不返回具体值，因此控制台显示 `undefined`。

## 查询交易对象

```javascript
eth.getTransaction(txHash)
```

你会看到包含下列字段的交易对象：

![查询已提交交易的完整对象](../assets/chapter-02/transaction-details.png)

重点关注这些字段：

| 字段 | 解释 |
| --- | --- |
| `hash` | 交易哈希 |
| `nonce` | 发送账户的交易序号 |
| `gas` | 允许这笔交易使用的最大 Gas |
| `blockNumber` | 交易被打包进的区块高度 |
| `blockHash` | 包含这笔交易的区块哈希 |
| `transactionIndex` | 交易在区块中的位置 |

其中最值得注意的是 `nonce`。它不是随机数，而是用来区分同一账户发出的不同交易，并帮助防止交易被重复执行。

如果开发账户此前没有发送过交易，第一笔交易的 `nonce` 通常是 `0`，下一笔则是 `1`。

## 查询交易回执

```javascript
eth.getTransactionReceipt(txHash)
```

![查询交易执行回执](../assets/chapter-02/transaction-receipt.png)

实际字段的显示形式可能因控制台版本而不同：

- `status: 1` 表示交易执行成功。
- `gasUsed` 表示实际消耗的 Gas。
- `blockNumber` 指向包含这笔交易的区块。

简单的 ETH 转账通常消耗 21,000 Gas，但执行智能合约时的 Gas 消耗会更加复杂。

## 在最新区块中定位交易

先查询最新区块高度：

```javascript
eth.blockNumber
```

![交易被打包后区块高度变为一](../assets/chapter-02/latest-block-number.png)

然后查询最新区块，并要求 Geth 返回完整交易对象：

```javascript
eth.getBlock("latest", true)
```

![查询包含完整交易对象的最新区块](../assets/chapter-02/latest-block-with-transactions.png)

第二个参数 `true` 非常重要：

- `false`：`transactions` 通常只返回交易哈希。
- `true`：`transactions` 返回完整交易对象。

你应该可以在区块的 `transactions` 数组中找到刚才发送的交易。

## 验证余额与 Nonce 变化

首先查询收款账户余额：

```javascript
web3.fromWei(
  eth.getBalance(receiver),
  "ether"
)
```

![收款账户收到 1 ETH](../assets/chapter-02/receiver-balance-after-transfer.png)

然后查询发送账户已经发出的交易数量：

```javascript
eth.getTransactionCount(sender)
```

![发送账户的交易数量变为一](../assets/chapter-02/sender-transaction-count.png)

如果这是开发账户的第一笔交易，结果通常是 `1`。

---

上一章：[第 1 章 运行本地 Geth 开发节点](01-running-a-local-geth-dev-chain.md) · 下一章：[第 3 章 理解交易生命周期与 Nonce](03-understanding-transaction-lifecycle-and-nonce.md)
