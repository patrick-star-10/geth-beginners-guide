# 第 1 章 运行本地 Geth 开发节点

2022 年 9 月 15 日，以太坊完成 The Merge，从 PoW 转换为 PoS。

## 以太坊节点包含什么

合并后的以太坊节点主要由两个客户端组成：

![以太坊节点由执行层和共识层组成](../assets/chapter-01/ethereum-node-architecture.png)

安装的 Geth 负责左边的执行层。

以前在 PoW 时代，单独运行 Geth 就能构成完整节点。今天若想正常运行以太坊主网节点，通常还需要连接一个共识客户端。

这意味着：

- Geth 不负责 PoS 投票和最终确定性。
- Geth 仍然负责验证与执行交易、维护以太坊状态。
- 你不需要质押 32 ETH，也可以运行非验证者的完整节点。
- Geth 目前不提供独立的执行层轻节点同步模式，默认的 Snap 同步也属于全节点同步机制。

## 第一阶段学习目标

运行本地开发链，学会启动 Geth、连接控制台、查看区块和创建交易，理解节点和区块链的基本运行过程。

## 启动第一个 Geth 节点

今天先不接入主网，而是运行一个本地开发节点。

官方提供的 `--dev` 模式允许 Geth 在本机创建一条独立的实验链，不需要下载主网历史数据，也不需要安装真正的 PoS 共识客户端。

### 第一步 切换工作目录

编译完成后，切换到 go-ethereum 工作目录：

```bash
cd ~/ethereum/go-ethereum
```

### 第二步 创建实验数据目录

```bash
mkdir -p ~/ethereum/dev-chain
```

为什么要单独创建目录？

因为 Geth 不只是一个可执行程序，它还是一个需要存储数据的区块链客户端。节点运行期间可能保存区块、链状态、交易索引以及账户密钥等数据。

我们使用独立目录，是为了以后运行主网或测试网时，不会与这条实验链的数据混在一起。

### 第三步 启动开发链

确保仍然处于 `~/ethereum/go-ethereum` 目录，执行：

```bash
./build/bin/geth --dev \
  --datadir ~/ethereum/dev-chain \
  --dev.period 0 \
  --http \
  --http.addr 127.0.0.1 \
  --http.api eth,net,web3
```

逐个理解这些参数：

| 参数 | 含义 |
| --- | --- |
| `--dev` | 启用本地开发链 |
| `--datadir` | 指定节点数据保存目录 |
| `--dev.period 0` | 有待处理交易时按需生成区块 |
| `--http` | 开启 HTTP JSON-RPC 服务 |
| `--http.addr 127.0.0.1` | 仅监听本机 |
| `--http.api` | 指定允许通过 HTTP 使用的 API |

特别注意 `--dev.period 0`。

它意味着这条实验链不会像真实的以太坊 PoS 主网那样，每 12 秒安排一个 slot，而是在需要处理交易时生成区块。

这也说明开发链只是用来学习执行机制的实验环境，并不等于完整模拟了 PoS 主网共识。

启动成功后，终端中会出现 Geth 的日志。这个终端不要关闭。

### 第四步 连接 Geth 控制台

打开第二个 Ubuntu 终端，执行：

```bash
cd ~/ethereum/go-ethereum
./build/bin/geth attach ~/ethereum/dev-chain/geth.ipc
```

如果成功，你会进入一个交互式环境，大致如下：

![成功连接 Geth JavaScript 控制台](../assets/chapter-01/geth-console-connected.png)

这个 `>` 表示你已经进入了 Geth 的 JavaScript 控制台。

你要理解一个很重要的概念：

- 第一个终端在运行区块链节点。
- 第二个终端在与这个节点通信。

它们通过 IPC（进程间通信）连接。这和以后使用 JSON-RPC 与其他程序连接 Geth 是相似的交互思路。

### 第五步 亲自观察区块链

进入 Geth 控制台后，我们先执行下面这些命令。

#### 实验 1 查看当前区块高度

```javascript
eth.blockNumber
```

`blockNumber` 返回当前本地区块链的最新区块高度。

刚创建开发链时，如果还没有执行交易，通常是 `0`，也就是创世区块。

![查看当前区块高度](../assets/chapter-01/block-number.png)

#### 实验 2 查看创世区块

```javascript
eth.getBlock(0)
```

这里会返回一个区块对象，包含区块哈希、状态根、时间戳、Gas 信息等字段。后面学习以太坊数据结构时，我们会逐个研究。

![查看本地开发链的创世区块](../assets/chapter-01/genesis-block.png)

`eth.getBlock(0)` 背后到底发生了什么？

![eth.getBlock 请求在控制台、RPC、Geth 和本地数据库之间的流程](../assets/chapter-01/get-block-request-flow.png)

这就是我们后面要深入理解的内容：一个看起来很简单的 JavaScript 调用，实际上会跨过 RPC、Go 实现代码和区块链数据库等多个层次。

#### 实验 3 查看本地账户

```javascript
eth.accounts
```

在开发模式下，Geth 会准备一个预置资金的开发账户。它不是主网账户里的真实 ETH，而是用于实验的本地测试资金。

![查看 Geth 开发模式提供的本地账户](../assets/chapter-01/development-account.png)

#### 实验 4 查看账户余额

```javascript
web3.fromWei(
  eth.getBalance(eth.accounts[0]),
  "ether"
)
```

`eth.getBalance()` 查询余额，原始单位是 Wei。`web3.fromWei()` 将 Wei 转换成 ETH，方便阅读。

![查看本地开发账户余额](../assets/chapter-01/development-account-balance.png)

## 发送第一笔 ETH 交易

接下来让开发账户 A 向新创建的收款账户 B 发送 1 ETH。

### 创建收款账户 B

正常来说，创建以太坊账户涉及生成私钥、推导公钥，再从公钥计算账户地址。

根据 [Geth 官方开发模式文档](https://geth.ethereum.org/docs/developers/dapp-developer/dev-mode)，当前教程推荐使用 Clef 创建新账户，而不是依赖旧教程中的 `personal.newAccount()`。本次本地实验使用 Geth 内置的账户管理命令创建一个由密码保护的 Keystore 账户：

```bash
cd ~/ethereum/go-ethereum

./build/bin/geth account new \
  --datadir ~/ethereum/dev-chain
```

按照提示输入密码并再次确认。创建成功后，会返回类似以下信息：

![使用 Geth 创建密码保护的 Keystore 账户](../assets/chapter-01/create-keystore-account.png)

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

![查看开发账户和新创建的收款账户](../assets/chapter-01/two-development-accounts.png)

接着定义两个变量：

```javascript
var sender = eth.accounts[0]
var receiver = eth.accounts[1]
```

检查收款账户的初始余额：

```javascript
eth.getBalance(receiver)
```

![新创建的收款账户余额为零](../assets/chapter-01/receiver-initial-balance.png)

新创建且没有获得过 ETH 的账户，余额应该是 `0`。

> 创建账户本身不是一笔链上交易。它主要是在本地生成密钥并保存到 Keystore，因此通常不会导致区块高度增加。
