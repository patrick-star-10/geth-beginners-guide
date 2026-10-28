# 第 9 章 ABI、函数选择器与交易数据编码

上一章追踪了 `Storage.get()` 中的 `SLOAD`。但 Geth 收到一段十六进制调用数据后，合约如何知道应执行 `get()`，还是前面用过的 `set(100)`？本章继续使用同一个 `Storage` 合约，从函数选择器和参数编码入手，最后拆解一次真实交易的 `input`。

## ABI 规定了什么？

在这个合约中，我们已经使用过两种调用：

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

| Solidity 调用 | 调用数据的组成 |
| --- | --- |
| `set(100)` | `0x60fe47b1` + 参数 `100` 的 ABI 编码 |
| `get()` | 只有 `0x6d4ce63c` |

这两个开头的四字节值是**函数选择器**。Solidity ABI（Application Binary Interface，应用二进制接口）规定外部调用如何编码函数及参数、如何解释返回数据。它是合约交互时遵循的格式约定；部署到链上的仍是编译后的 EVM 字节码，而不是 ABI 文本。编译器生成的函数分发逻辑会读取调用数据的开头，选择相应的执行路径。[Solidity ABI 规范](https://docs.soliditylang.org/en/latest/abi-spec.html)定义了这些编码规则。

![从 Solidity 调用到交易 input 和合约执行](../assets/chapter-09/abi-call-flow.png)

## 四字节函数选择器从哪里来？

按照 [ABI 规范中的函数选择器规则](https://docs.soliditylang.org/en/latest/abi-spec.html#function-selector)，取函数签名的 Keccak-256 哈希的前四字节：

![函数选择器的计算方式](../assets/chapter-09/function-selector-formula.png)

`set(uint256 _number)` 的规范函数签名是 `set(uint256)`：只保留函数名和参数类型，不包含参数名、`public`、`external` 或返回值类型。因此，把参数名 `_number` 改成 `value`，或者在合适的合约接口中把可见性从 `public` 改为 `external`，不会改变这个函数的选择器；改变函数名或参数类型则会改变签名。

在 Geth JavaScript 控制台中，可以亲自计算：

```javascript
web3.sha3("set(uint256)")
```

![计算 set 函数签名的哈希](../assets/chapter-09/set-signature-hash.png)

结果是 32 字节哈希。虽然这个旧版控制台方法叫 `sha3`，这里计算的是以太坊使用的 **Keccak-256**，不是标准化的 SHA3-256；[Ethereum JSON-RPC 文档](https://ethereum.org/developers/docs/apis/json-rpc/#web3_sha3)也作了这一区分。

再取前四字节：

```javascript
web3.sha3("set(uint256)").slice(0, 10)
```

![取出 set 的函数选择器](../assets/chapter-09/set-function-selector.png)

预期得到 `0x60fe47b1`。这里截取 10 个字符，是因为 `0x` 占 2 个字符，后面 8 个十六进制字符表示 4 字节。同样计算 `get()`：

```javascript
web3.sha3("get()").slice(0, 10)
```

![计算 get 的函数选择器](../assets/chapter-09/get-function-selector.png)

预期得到 `0x6d4ce63c`。这些选择器由函数签名决定；使用相同签名的读者不需要把它们换成自己的交易哈希。

## `set(100)` 的参数怎样编码？

第 7 章曾手动构造调用数据：

```javascript
var callData = "0x60fe47b1" + "64".padStart(64, "0")
```

![set 100 调用数据的组成](../assets/chapter-09/set-call-data-layout.png)

`0x60fe47b1` 是四字节选择器；`100` 的十六进制表示是 `64`，作为 `uint256` 参数被左侧补零到 32 字节。两段合起来是 **36 字节**，开头的 `0x` 只是十六进制字符串前缀，不计入字节长度。这里只是一个静态参数的简单例子；[ABI 规范](https://docs.soliditylang.org/en/latest/abi-spec.html#formal-specification-of-the-encoding)对动态类型还规定了偏移量和数据区域。

## 为什么 `get()` 只有选择器？

`get()` 没有输入参数，所以调用数据只有四字节选择器 `0x6d4ce63c`。它的返回值是另一回事：如果 Slot 0 存着 `100`，`eth.call` 返回的是按 ABI 编码的 `uint256`，即末尾为 `64` 的 32 字节数据：

```text
0x0000000000000000000000000000000000000000000000000000000000000064
```

函数调用的输入编码与函数执行后的返回值编码是两个相关但不同的过程。下一节会从已发送的 `set(100)` 交易中，把选择器和参数再拆出来核对。

## 实验：拆解自己交易的 `input`

第 7 章执行 `set(100)` 时，`eth.sendTransaction` 返回的交易哈希保存在 `setTx` 中。继续使用**你自己那笔交易**的哈希：如果仍在原控制台，可直接使用 `setTx`；如果重新连接过控制台，请先用当时保存的实际哈希恢复 `setTx`。截图中的 `0x1b7e…` 只是作者本地实验的结果，不能直接照抄。使用了新的 `--datadir` 或重新初始化开发链时，也必须重新发送交易并取得新哈希。

### 读取交易调用数据

```javascript
var tx = eth.getTransaction(setTx)
tx.input
```

![读取 set 100 交易的 input](../assets/chapter-09/set-transaction-input.png)

预期得到前缀为 `0x60fe47b1`、末尾为 `64` 的十六进制数据，与第 7 章构造的 `callData` 相同。如果 `tx` 是 `null`，先检查 `setTx` 是否为自己的交易哈希，以及当前连接的是否是当时使用的开发链。交易对象的 `input` 字段承载调用数据，可参考 [Ethereum JSON-RPC 的交易对象说明](https://ethereum.org/developers/docs/apis/json-rpc/#eth_gettransactionbyhash)。

### 核对函数选择器

前四字节是选择器。由于字符串还包含 `0x` 前缀，截取前 10 个字符即可：

```javascript
tx.input.slice(0, 10)
tx.input.slice(0, 10) === web3.sha3("set(uint256)").slice(0, 10)
```

![核对交易 input 的函数选择器](../assets/chapter-09/selector-match-check.png)

这里应该得到 `0x60fe47b1` 和 `true`。这验证了交易的 `input` 开头确实对应 `set(uint256)`，而不是 `get()`。

### 解出静态参数

跳过前面的 10 个字符，剩下的是参数 `100` 的 32 字节编码：

```javascript
var parameter = tx.input.slice(10)
web3.toDecimal("0x" + parameter)
```

![把 32 字节参数解码为 100](../assets/chapter-09/decode-uint256-argument.png)

本次应得到 `100`。这里用 `web3.toDecimal` 是为了观察这个很小的数；一般的 `uint256` 可能超过 JavaScript 安全整数范围，不应把这种手工转换当作通用 ABI 解码器。

![把交易 input 拆成选择器和参数](../assets/chapter-09/transaction-input-breakdown.png)

这个实验只处理一个静态类型 `uint256`。遇到 `string`、动态数组或更复杂的结构时，编码还涉及偏移量及动态数据区域，应按 [Solidity ABI 规范](https://docs.soliditylang.org/en/latest/abi-spec.html#formal-specification-of-the-encoding)解析，或使用可靠的 ABI 解码工具。

## 本章结论

`set(100)` 的交易 `input` 由四字节函数选择器和一个 32 字节参数组成。函数选择器来自 `set(uint256)` 的 Keccak-256 哈希，与读者自己的交易哈希不同：前者在函数签名不变时相同，后者必须使用自己发送交易后得到的值。通过实际交易的 `input`，我们把手动构造的调用数据与链上记录核对了一遍。

## 章节自测

先不看答案，试着用自己的话解释每一题；然后对照下面的参考答案，检查自己是否分清了函数选择器、参数编码、交易输入和执行结果。

### 题目

1. `set(100)` 的函数选择器是怎么得到的？计算选择器时，数字 `100` 会参与哈希吗？
2. 如果调用从 `set(100)` 改成 `set(200)`，函数选择器会变吗？调用数据的哪一部分会变？
3. 为什么 `get()` 的调用数据只有 `0x6d4ce63c`？如果它读出了 `100`，这个 `100` 会放在调用数据里吗？
4. 本章从 `set(100)` 那笔交易的 `tx.input` 中，分别检查哪两部分？每一部分要验证什么？
5. 选择器只有 4 字节，为什么代码用 `tx.input.slice(0, 10)` 取 10 个字符？
6. 部署合约时交易的 `input` 是 Creation Bytecode；调用 `set(100)` 时，交易的 `input` 是什么？它还是合约字节码吗？
7. 如果一笔发往这个合约的交易，`input` 以 `0x60fe47b1` 开头，后面 32 字节参数的末尾是 `c8`，它想调用哪个函数、传入什么十进制参数？光看 `input`，能确定调用已经成功修改了状态吗？
8. 实际应用要大量调用主网上的合约时，还需要每次手工计算选择器并拼接参数吗？

### 参考答案

1. 对规范函数签名 `set(uint256)` 计算 Keccak-256，取哈希的前 4 字节，得到 `0x60fe47b1`。签名只包含函数名和参数类型，不包含本次传入的 `100`；`100` 会单独进行 ABI 编码。
2. 不会变，因为函数签名仍是 `set(uint256)`。变化的是选择器后面的参数编码：`100` 的十六进制末尾是 `64`，`200` 的是 `c8`，它们都会按 `uint256` 左侧补零至 32 字节。参数部分不是参数的哈希值。
3. `get()` 没有输入参数，所以调用数据只有它的选择器。读出的 `100` 属于返回数据，不会放进调用数据。
4. 先检查开头 4 字节是否等于 `set(uint256)` 的选择器，确认调用的是 `set` 而不是 `get`；再把后面的 32 字节参数解码，确认值为 `100`。
5. 1 字节对应 2 个十六进制字符，所以 4 字节选择器需要 8 个字符；再加上 `0x` 前缀的 2 个字符，共 10 个字符。
6. 调用时的 `input` 是按 ABI 编码的调用数据，即函数选择器加参数编码；交易的 `to` 字段是合约地址。它不是部署用的 Creation Bytecode。ABI 是编码规则，不是 `input` 本身。
7. 它打算调用 `set(200)`，因为 `0x60fe47b1` 对应 `set(uint256)`，`0xc8` 是十进制的 `200`。`input` 只表明调用意图，不能证明执行成功；应查看这笔交易回执的 `status`，再核对执行后的状态。只在后来调用 `get()` 读到 `200`，还不能单独证明就是这笔交易将值改成了 `200`，因为原值也可能已经是 `200`。
8. 不需要。手工拼接是为了理解编码过程；实际开发通常把合约 ABI、函数名和参数交给库，让它自动编码调用数据，例如 viem 的 [readContract](https://viem.sh/docs/contract/readContract) 和 [writeContract](https://viem.sh/docs/contract/writeContract)。读操作可直接取得解码后的返回值；写操作仍须发送交易、支付 Gas，并通过回执确认执行结果，不能把返回的交易哈希当作执行成功的证明。

---

上一章：[第 8 章 EVM 的 Stack、Memory 与 Storage](08-evm-stack-memory-and-storage.md)
