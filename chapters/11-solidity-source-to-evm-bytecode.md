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

## 实验一：读取链上的 Runtime Bytecode

我们继续使用前面部署的 `Storage` 合约。

### 第一步：连接 Geth 控制台

如果你的 Geth 节点仍在运行，使用此前的连接命令：

```bash
cd ~/ethereum/go-ethereum
./build/bin/geth attach ~/ethereum/dev-chain/geth.ipc
```

如果节点已经停止，需要先按[第 1 章](01-running-a-local-geth-dev-chain.md)的方式启动开发链，再连接控制台。

### 第二步：定义合约地址

在 Geth JavaScript 控制台执行，把引号中的内容替换为**你自己部署的合约地址**；截图中的地址只是作者的实验结果：

```javascript
var contractAddress = "填入你自己部署的合约地址"
```

### 第三步：读取合约运行时代码

```javascript
var runtimeCode = eth.getCode(contractAddress)
runtimeCode
```

你应该看到一串很长的十六进制字符：

![从合约地址读取链上的 Runtime Bytecode](../assets/chapter-11/runtime-code-console.png)

这是链上保存的 EVM 运行时字节码。如果得到的只是 `"0x"`，先检查开发链和合约地址是否正确。

### 第四步：计算运行时代码的字节长度

执行：

```javascript
(runtimeCode.length - 2) / 2
```

![计算 Runtime Bytecode 的字节长度](../assets/chapter-11/runtime-code-length.png)

为什么这样计算？因为 `0x` 是十六进制前缀，不是字节码内容；每两个十六进制字符表示一个字节。截图中的 `297` 是这次实验的结果，你的合约可能不同。

这里要注意，**字节长度不等于 EVM 指令数量**，因为像 `PUSH1`、`PUSH2` 等指令后面还会携带立即数。

## 实验二：认识真正的 EVM Opcode

现在来看合约字节码开头的一段：`6080604052`。

把它拆开：

| 字节 | Opcode | 含义 |
| --- | --- | --- |
| `60 80` | `PUSH1 0x80` | 将 128 压入栈 |
| `60 40` | `PUSH1 0x40` | 将 64 压入栈 |
| `52` | `MSTORE` | 将数值写入内存 |

这段代码在 EVM 中的执行过程是：

![PUSH1 0x80 把 0x80 压入栈](../assets/chapter-11/push-0x80.png)

![PUSH1 0x40 把 0x40 压入栈顶](../assets/chapter-11/push-0x40.png)

![MSTORE 把 0x80 写入 Memory 的 0x40 位置](../assets/chapter-11/mstore-free-memory-pointer.png)

这段指令的常见用途是初始化 Solidity 的空闲内存指针（Free Memory Pointer）。执行结束后，内存偏移量 `0x40` 处保存的值是 `0x80`，意味着编译器约定可以从偏移量 `0x80` 开始安排后续动态内存分配。

注意：这是 Solidity 编译器采用的[内存布局约定](https://docs.soliditylang.org/en/latest/internals/layout_in_memory.html)，不是 EVM 强制所有程序必须遵守的规则。

## 实验三：将字节码与真实执行指令对应起来

我们现在知道字节码开头的含义了。但是，如何证明 Geth 确实执行了这些指令？

我们使用上一课已经学过的 `debug.traceCall()`。

### 第一步：生成 `get()` 的函数选择器

```javascript
var getData = web3.sha3("get()").slice(0, 10)
```

### 第二步：追踪执行过程

```javascript
var trace = debug.traceCall({to: contractAddress, data: getData}, "latest", {})
```

### 第三步：查看前 15 条指令

```javascript
trace.structLogs.slice(0, 15).map(function(s) {
  return {pc: s.pc, op: s.op, stack: s.stack}
})
```

![追踪 get 调用时前几条 EVM 指令及其栈内容](../assets/chapter-11/runtime-trace-first-15-a.png)

![追踪 get 调用时后续指令及其栈内容](../assets/chapter-11/runtime-trace-first-15-b.png)

预计可以看到类似的执行指令：

```text
PUSH1
PUSH1
MSTORE
CALLVALUE
DUP1
ISZERO
PUSH1
JUMPI
...
```

具体内容以你的实际 Trace 为准。这里的 `pc` 是指令在字节码中的**字节偏移量**，不是第几条指令，也不是源代码行号。[Geth 的 Opcode Tracer](https://geth.ethereum.org/docs/developers/evm-tracing/basic-traces)通过记录 PC、操作码、Gas 和 Stack 等信息，帮助开发者还原 EVM 的执行过程。

![Solidity、字节码、Opcode 和 Geth 解释器的作用对照](../assets/chapter-11/bytecode-concepts-table.png)

### 分析指令

#### 第一阶段：初始化 EVM Memory

观察到的前三条指令：

```text
PC 0  PUSH1
PC 2  PUSH1
PC 4  MSTORE
```

对应 Runtime Bytecode 的开头：`60 80 60 40 52`。它们实际上是：

```text
PUSH1 0x80
PUSH1 0x40
MSTORE
```

这三条指令将 `0x80` 存入 Memory 偏移量 `0x40` 开始的 32 字节区域，设置 Solidity 的空闲内存指针。

你需要记住：Memory 初始化并不是 `get()` 函数的业务逻辑，而是编译器生成的运行时准备工作。
