# Geth Beginner's Guide

一份以实验为主线、面向 PoS 时代的中文以太坊客户端入门指南。从运行本地 Geth 节点开始，亲手观察交易、Gas、状态变化和智能合约执行，逐步理解执行客户端的工作方式，并进一步探索它与共识客户端如何协作。

本项目希望用可复现的实验，帮助读者建立对合并后以太坊的认识，补充中文资料中较少涉及的现行客户端运行机制。

## 内容

1. [运行本地 Geth 开发节点](chapters/01-running-a-local-geth-dev-chain.md)
2. [发送第一笔 ETH 交易](chapters/02-sending-the-first-eth-transaction.md)
3. [理解交易生命周期与 Nonce](chapters/03-understanding-transaction-lifecycle-and-nonce.md)
4. [交易验证与 TxPool](chapters/04-transaction-validation-and-txpool.md)
5. [理解 Gas 与 EIP-1559 交易费用](chapters/05-understanding-gas-and-eip-1559-fees.md)
6. [以太坊状态机与账户状态转换](chapters/06-ethereum-state-machine-and-account-transitions.md)
7. [EVM 与智能合约执行](chapters/07-evm-and-smart-contract-execution.md)
8. [EVM 的 Stack、Memory 与 Storage](chapters/08-evm-stack-memory-and-storage.md)
9. [ABI、函数选择器与交易数据编码](chapters/09-abi-function-selectors-and-calldata.md)

## 阅读说明

本教程面向刚开始接触 Geth 和以太坊节点的读者。内容以动手实验为主，并逐步解释命令背后的执行层概念。

截图和命令输出中的账户或合约地址、交易或区块哈希、State Root 等，均来自一次本地实验。复现时请使用**自己节点实际返回的值**，不必与截图一致；函数签名不变时，相应的函数选择器则可以照用。

笔记仍在持续整理中。每次更新会保留清晰的 Git 提交记录，方便查看学习路径和内容演进。

## 许可

除另有说明外，本项目内容由 [patrick-star-10](https://github.com/patrick-star-10) 依据 [Creative Commons Attribution 4.0 International](https://creativecommons.org/licenses/by/4.0/) 许可发布。转载或改编时请保留署名，并注明是否修改。
