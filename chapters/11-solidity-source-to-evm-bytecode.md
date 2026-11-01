# 第 11 章 从 Solidity 源代码到 EVM 字节码

上一课，我们研究了 EVM 的函数分发机制（Function Dispatcher），理解了 `CALLDATALOAD`、`SHR`、`EQ` 和 `JUMPI` 如何帮助合约找到正确的函数。

今天我们再深入一层：**Solidity 写出来的智能合约，究竟是怎样变成 EVM 可以执行的字节码的？**

这也是我们从学习 EVM 基础，逐渐过渡到研究 Geth 源码的重要一步。

## Solidity 代码并不是直接在 EVM 中运行

之前编写的合约：

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract Storage {
    uint256 private number;

    function set(uint256 _number) public {
        number = _number;
    }

    function get() public view returns (uint256) {
        return number;
    }
}
```

EVM 并不认识 `uint256`、`function`、`public` 这些 Solidity 语法。它只认识一个个字节构成的指令。

![Solidity 源代码经过 solc 编译后由 Geth 的 EVM 解释器执行](../assets/chapter-11/solidity-to-evm-flow.png)

这里要区分两个角色：

1. **Solidity 编译器**：负责生成字节码。
2. **Geth**：负责在交易执行过程中按照 EVM 规则执行字节码。

Geth 并不需要读取 Solidity 源代码才能执行你的合约。

## 为什么存在两种合约字节码？

这是本节第一个重要知识点。之前在 Remix 编译合约时，可能注意到两个概念：

1. Creation Bytecode（部署字节码）
2. Runtime Bytecode（运行时字节码）

它们不是同一回事。

**Creation Bytecode** 在创建合约的交易中执行，负责初始化合约，并返回需要部署的运行时代码。它只在部署阶段执行。

**Runtime Bytecode** 在部署后保存在合约账户中，未来调用 `set()`、`get()` 时执行的就是这部分代码。它可以在后续交易和调用中重复执行。

[Solidity 官方文档](https://docs.soliditylang.org/en/latest/contracts.html#constructors)也说明：创建时执行初始化代码，最终保存在链上的代码不包含仅用于部署的构造逻辑。
