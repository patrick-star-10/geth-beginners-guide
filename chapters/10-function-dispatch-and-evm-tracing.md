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
