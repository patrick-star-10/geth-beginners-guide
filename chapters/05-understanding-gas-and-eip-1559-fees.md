# 第 5 章 理解 Gas 与 EIP-1559 交易费用

本章承接[第 2 章](02-sending-the-first-eth-transaction.md)已经完成的转账实验，继续使用其中的交易和回执理解 Gas、实际手续费以及 EIP-1559 费用结构。

## Gas 与 Gas Price

首先要区分两个概念：

| 概念 | 含义 |
| --- | --- |
| Gas | 衡量以太坊执行交易所消耗的计算资源 |
| Gas Price | 执行每单位 Gas 需要支付的 ETH 数量 |

两者不是同一个东西。可以把 Gas 理解为计算资源的计量单位，把 Gas Price 理解为每单位资源的价格。

以太坊执行交易需要消耗网络计算资源。引入 Gas，一方面用于计算交易费用，另一方面可以限制一笔交易能够消耗的资源，避免无限计算。

## 计算交易手续费

一笔交易的实际手续费可以表示为：

![交易手续费等于 Gas Used 乘以 Effective Gas Price](../assets/chapter-05/transaction-fee-formula.png)

假设一笔普通 ETH 转账实际消耗 `21,000` Gas，每单位 Gas 的实际价格为 `2` Gwei，那么手续费为：

![使用 21000 Gas 和 2 Gwei 计算手续费](../assets/chapter-05/example-transaction-fee.png)

也就是 `0.000042 ETH`。

根据[以太坊 Gas 官方文档](https://ethereum.org/developers/docs/gas/)，Gas Price 通常使用 Gwei 表示，`1 Gwei = 10^-9 ETH`。

## Gas Limit 与 Gas Used

这是初学者最容易混淆的一组概念。假设你准备执行一笔普通 ETH 转账，并为交易设置：

```javascript
gas: 50000
```

![将交易 Gas Limit 设置为 50000](../assets/chapter-05/gas-limit-50000.png)

这表示最多允许这笔交易消耗 `50,000` Gas，并不代表实际一定会消耗 `50,000` Gas。如果交易最终只消耗 `21,000` Gas，剩余的 Gas 额度不会作为已消耗 Gas 收费。

![Gas Limit 与 Gas Used 的区别](../assets/chapter-05/gas-limit-vs-gas-used.png)

| 概念 | 含义 |
| --- | --- |
| Gas Limit | 允许交易使用的最大 Gas |
| Gas Used | 交易实际消耗的 Gas |
| Gas Price | 每单位 Gas 的实际价格 |
| Transaction Fee | `Gas Used × Effective Gas Price` |

之前查询创世区块时出现了 `gasLimit: 11500000` 和 `gasUsed: 0`。这里要注意：区块的 Gas Limit 和单笔交易的 Gas Limit 是两个不同层次的限制。创世区块的 `gasUsed` 为 `0`，是因为当时没有执行普通交易。

## EIP-1559 费用结构

现在常见的 EIP-1559 动态费用交易，会将每单位 Gas 的实际费用拆分为两部分：

![EIP-1559 的 Base Fee 与 Priority Fee](../assets/chapter-05/eip1559-fee-components.png)

- **Base Fee**：由协议根据区块使用情况确定，并在交易执行后销毁。
- **Priority Fee**：实际支付给区块提议者的优先费用，也常被称为小费。

例如 Base Fee 是 `10` Gwei，实际 Priority Fee 是 `2` Gwei，那么实际 Gas Price 是 `12` Gwei。

EIP-1559 交易还可以设置两个上限：

| 字段 | 含义 |
| --- | --- |
| `maxFeePerGas` | 每单位 Gas 愿意支付的最高总价 |
| `maxPriorityFeePerGas` | 每单位 Gas 愿意支付的最高优先费用 |

对于普通 EIP-1559 动态费用交易，每单位 Gas 的实际价格可以表示为：

```text
effectiveGasPrice = min(maxFeePerGas, baseFeePerGas + maxPriorityFeePerGas)
```

![EIP-1559 Effective Gas Price 计算公式](../assets/chapter-05/effective-gas-price-formula.png)

`maxFeePerGas` 是价格上限，不是一定会支付的价格。只要上限足够覆盖 Base Fee 和实际 Priority Fee，最终就按实际价格收费。

基础费用会销毁，实际优先费用会归区块提议者。这套机制由 [EIP-1559](https://eips.ethereum.org/EIPS/eip-1559) 引入，并不是以太坊转向 PoS 时才出现的。
