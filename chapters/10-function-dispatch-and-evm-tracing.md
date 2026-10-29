# 第 10 章 EVM 如何找到并执行正确的合约函数

第 9 章把 `set(100)` 的调用数据拆成了 `set(uint256)` 的四字节选择器和参数 `100` 的 32 字节 ABI 编码。交易到达合约地址后，EVM 看到的仍是一段字节；它并不认识 Solidity 函数名。那么，为什么这段数据最终会进入 `set`，而不是 `get`？本章先理解合约运行时代码中的函数分发逻辑，再用 Geth 的执行轨迹验证它。

## 合约运行时代码负责分发

继续使用第 7 章部署的 `Storage` 合约：

```solidity
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

Solidity 编译器生成的**运行时代码**包含处理外部调用的分发逻辑：读取调用数据开头的函数选择器，与合约支持的选择器比较，匹配后跳到对应的代码位置。选择器的计算方法已在[第 9 章](09-abi-function-selectors-and-calldata.md)介绍；它不是 EVM 内置的“函数表”，而是编译后的字节码执行了这些比较和跳转。[Solidity ABI 规范](https://docs.soliditylang.org/en/latest/abi-spec.html#function-selector)规定调用数据的前四字节用于指定函数。

![Storage 合约按函数选择器分发调用数据](../assets/chapter-10/function-dispatch-overview.png)

示意图中的 `set()` 路径会解码参数并写入 Storage，`get()` 路径会读取 Storage 并编码返回值。具体生成哪些指令、按什么顺序比较，会受编译器版本和优化设置影响；不能把示意图当成所有合约的固定字节码。

## 从调用数据中取出选择器

以下四个操作码是理解这个实验的关键；表格是[以太坊操作码参考](https://ethereum.org/developers/docs/evm/opcodes/)所定义语义的简化说明：

![函数分发相关的四个 EVM 操作码](../assets/chapter-10/dispatcher-opcodes.png)

以 `set(100)` 为例，调用数据开头是 `0x60fe47b1`，后面跟着参数的 ABI 编码。`CALLDATALOAD` 从指定偏移量读取 **32 字节**；如果偏移量是 0，读到的第一个 32 字节字（word）以选择器开头，后面还有参数编码的前 28 字节，**并不只是选择器本身**。在本例中，这些参数高位都是零。

为了提取最左边的 4 字节，可以把这个 256 位字右移 `256 − 4 × 8 = 224` 位。`PUSH1 0xe0` 可将十六进制的 `0xe0`（十进制 `224`）压栈，`SHR` 再完成右移，结果就是 `0x60fe47b1`。编译后的代码可能用 `PUSH0` 提供偏移量 0，也可能用其他等价指令；这里说明的是提取原理，不要求读者在所有版本里找到完全相同的指令序列。

随后，`EQ` 将提取出的选择器与已知选择器比较，`JUMPI` 在条件满足时跳到对应的 `JUMPDEST`。例如 `set(uint256)` 对应 `0x60fe47b1`，`get()` 对应 `0x6d4ce63c`。可以把分发过程概念化为：

```javascript
if (selector === "0x60fe47b1") {
  // 进入 set(uint256) 的执行路径
} else if (selector === "0x6d4ce63c") {
  // 进入 get() 的执行路径
} else {
  // 若没有可处理该调用的 fallback，调用失败
}
```

这只是帮助理解的伪代码，不是 Solidity 编译器输出的 JavaScript 或逐字对应的 EVM 指令。接下来要用真实轨迹观察一次 `get()` 调用。

## 实验：用 Geth 找到函数分发过程

本实验沿用第 7 章部署的 `Storage` 合约，以及保存其地址的 `contractAddress` 变量。若已重新连接控制台，请把它设为**你自己部署的合约地址**；截图中的 `0xdb7d…` 是作者本地实验的地址，不能直接照抄。先检查该地址确实有合约运行时代码：

```javascript
eth.getCode(contractAddress)
```

若返回 `"0x"`，说明当前链上这个地址没有代码，应检查是否连接了原来的开发链、是否使用了正确的数据目录和合约地址。

### 准备 `get()` 的调用数据

在 Geth JavaScript 控制台计算 `get()` 的选择器；它没有输入参数，因此这就是完整的调用数据：

```javascript
var getData = web3.sha3("get()").slice(0, 10)
getData
```

![在 Geth 控制台计算 get 的调用数据](../assets/chapter-10/get-selector-console.png)

结果应为 `0x6d4ce63c`。这个值由函数签名决定，不需要替换成个人实验中的哈希。

### 模拟并追踪调用

使用第 8 章已经接触过的 `debug.traceCall`：

```javascript
var trace = debug.traceCall(
  {to: contractAddress, data: getData},
  "latest",
  {}
)
trace.failed
```

![Geth 模拟 get 调用且追踪结果未失败](../assets/chapter-10/get-trace-failed-false.png)

本次实验返回 `false`，表示**这次模拟调用**没有失败；`debug.traceCall` 不会把它作为新交易写入链上。[Geth 的 `debug_traceCall` 文档](https://geth.ethereum.org/docs/interacting-with-geth/rpc/ns-debug#debug_tracecall)说明它会在给定区块状态下执行调用并返回追踪结果。

先浏览开头的指令：

```javascript
trace.structLogs.slice(0, 25).map(function(s) {
  return {pc: s.pc, op: s.op, stack: s.stack}
})
```

`pc` 是程序计数器，`op` 是操作码，`stack` 是该步附近的栈内容。可以留意 `CALLDATASIZE`、`CALLDATALOAD`、`SHR`、`EQ` 和 `JUMPI`：`CALLDATASIZE` 取得调用数据的字节长度，编译器可能先据此检查输入是否足够长。前 25 步不一定包含完整分发过程，所以再从完整轨迹中筛选：

```javascript
var ops = ["CALLDATALOAD", "SHR", "EQ", "JUMPI"]
trace.structLogs.filter(function(s) {
  return ops.indexOf(s.op) !== -1
}).map(function(s) {
  return {pc: s.pc, op: s.op, stack: s.stack}
})
```

![筛选出 get 调用轨迹中的函数分发指令](../assets/chapter-10/dispatcher-ops-trace.png)

截图中的 `CALLDATALOAD` 读取调用数据，`SHR` 将开头四字节移到低位，后续 `EQ` 比较选择器，匹配时由 `JUMPI` 进入 `get()` 的路径。因为 `getData` 只有四字节，读取 32 字节时其余位置会补零。轨迹中还可能出现用于其他检查的 `JUMPI`，不能把每个跳转都当作函数匹配；应结合 `pc` 和 `stack` 判断。[Geth 的基础追踪文档](https://geth.ethereum.org/docs/developers/evm-tracing/basic-traces)介绍了这些逐条操作码记录。不同编译设置下，`pc`、栈值和分支顺序不必与截图完全相同。

## 实验：输入未匹配的选择器

如果调用数据开头的选择器找不到对应函数，执行路径取决于合约是否定义了能够处理它的 `fallback` 函数。本章的 `Storage` 合约没有定义 `fallback`，因此可以用一个错误选择器观察回退路径；其他合约未必会得到相同结果。[Solidity 文档中的 `fallback` 说明](https://docs.soliditylang.org/en/latest/contracts.html#fallback-function)解释了这种特殊函数何时被调用。

仍使用上面的 `contractAddress`，模拟发送 `0x12345678`：

```javascript
var badTrace = debug.traceCall(
  {to: contractAddress, data: "0x12345678"},
  "latest",
  {}
)
badTrace.failed
```

![错误选择器的模拟调用失败](../assets/chapter-10/unknown-selector-trace-failed.png)

本次实验返回 `true`。这表示**模拟执行失败**，不是某笔已上链交易的回执。查看轨迹末尾：

```javascript
badTrace.structLogs.slice(-5).map(function(s) {
  return {pc: s.pc, op: s.op}
})
```

![错误选择器的执行轨迹以 REVERT 结束](../assets/chapter-10/unknown-selector-revert-trace.png)

截图中最后一条指令是 `REVERT`：运行时代码没有找到匹配的函数入口，也没有可接收这段数据的 `fallback`，于是撤销本次调用。不同编译版本下，末尾的其他指令和 `pc` 数值可能不同，重点是结合 `failed` 与执行路径判断结果。

至此可以把两件事连起来：ABI 规则决定调用数据怎样编码；EVM 则执行合约的运行时代码，由其中的分发逻辑读取选择器、比较并跳转。**EVM 不会凭函数名直接调用 Solidity 函数。**

---

上一章：[第 9 章 ABI、函数选择器与交易数据编码](09-abi-function-selectors-and-calldata.md)
