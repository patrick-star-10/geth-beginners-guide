# 准备篇 实验环境与 Geth 安装

本篇先记录教程使用的实验环境，再在 Ubuntu 上从源码编译 Geth。完成后即可进入[第 1 章](01-running-a-local-geth-dev-chain.md)，启动一条独立的本地开发链。这里的安装步骤用于学习实验，不是主网节点的部署指南。

## 作者的实验环境

以下是作者本次实验的记录，**不是读者必须安装的版本清单**：

```text
Geth/v1.17.8-unstable-b10c4ecc-20261008/linux-amd64/go1.25.8
```

| 项目 | 作者实验所用配置 |
| --- | --- |
| 执行客户端 | Go Ethereum（Geth） |
| Geth 版本 | `v1.17.8-unstable` |
| 源码提交标识 | `b10c4ecc` |
| 版本标识中的日期 | `2026-10-08` |
| 操作系统 | Ubuntu 24.04.1 LTS（Linux） |
| CPU 架构 | `amd64`（x86-64） |
| 编译所用 Go 版本 | `go1.25.8` |
| 安装方式 | 从 GitHub 获取源码，在本地编译 |
| 实验数据所用数据库引擎 | Pebble |

版本串中的日期只是该标识的一部分，不必把它理解为自己编译二进制文件的时间。`unstable` 表示这是开发版本；[Geth 官方安装文档](https://geth.ethereum.org/docs/getting-started/installing-geth)建议大多数用户使用经过测试的稳定版本。读者不必追求与上表完全一致，应记录自己实际使用的 Geth 版本、源码提交和 Go 版本。不同版本的日志、默认设置和实验输出可能不同。

Pebble 是这次实验的数据目录所用的数据库引擎，**不能从上面的 Geth 版本串读出**。本教程前几章的开发链实验不依赖必须使用 Pebble；若要核对自己的实际引擎，请查看节点启动时有关数据库的日志，不要仅凭版本号推断。

## 安装构建依赖

在 Ubuntu 终端执行作者当时使用的命令，安装 Git 和常用编译工具；`build-essential` 也会安装 Make：

```bash
sudo apt-get update
sudo apt-get install -y git build-essential
```

## 克隆源码

[Geth 官方安装文档](https://geth.ethereum.org/docs/getting-started/installing-geth#build-from-source-code)使用 `git clone` 获取源码。这里把源码放在 `~/ethereum/go-ethereum`，与后续章节的命令保持一致：

```bash
mkdir -p ~/ethereum
cd ~/ethereum
git clone https://github.com/ethereum/go-ethereum.git
cd go-ethereum
```

直接克隆默认分支会得到克隆当时的源码，版本可能是 `unstable`，不会自动与作者的 `b10c4ecc` 相同。如果需要稳定版本或可重复的特定版本，请先从 [官方发布页](https://github.com/ethereum/go-ethereum/releases)选择并切换到相应的发布标签，再运行 `make geth`。不要把作者的开发版提交当作所有读者必须使用的版本。

## 检查源码和 Go 版本要求

在 `go-ethereum` 目录中，先查看源码版本标识、`go.mod` 声明的 Go 版本要求，以及本机的 Go 版本：

```bash
git describe --tags --always
grep '^go ' go.mod
go version
```

`git describe` 给出与当前源码相关的标签描述，不一定是精确的提交标识；需要记录具体提交时，可使用下面的 `git rev-parse`。如果尚未安装 Go，或现有 Go 版本不满足所选源码的要求，请先按 [Go 官方安装说明](https://go.dev/doc/install)准备相应工具链，再编译。作者使用 `go1.25.8`，但读者不必锁定这个版本。

## 编译并核对版本

确认 Go 工具链可用后，按 [Geth 官方源码构建流程](https://geth.ethereum.org/docs/getting-started/installing-geth#build-from-source-code)编译：

```bash
make geth
```

官方构建命令会把可执行文件放在 `build/bin/geth`。检查编译出的版本，并记录自己使用的源码提交：

```bash
./build/bin/geth version
git rev-parse --short HEAD
```

如果找不到 `./build/bin/geth`，先检查 `make geth` 是否报错、Go 版本是否满足源码要求，以及当前目录是否为 `~/ethereum/go-ethereum`。之后的章节均用 `./build/bin/geth` 运行，不要求把程序安装到系统目录。

## 关于 Pebble 与开发链数据目录

第 1 章会创建独立的 `~/ethereum/dev-chain` 数据目录。若希望在**全新的数据目录**中明确使用 Pebble，可在第 1 章的第一次 Geth 启动命令中加上 `--db.engine=pebble`；不加这个参数也能完成本教程的实验。[Geth 数据库文档](https://geth.ethereum.org/docs/fundamentals/databases#pebble)说明，如果数据目录已有 LevelDB 数据，启动时添加该参数不会把旧数据库转换成 Pebble。不要为了切换引擎而删除已有实验数据；需要另建数据目录时，也要重新取得该链上的账户、地址和交易哈希。

---

下一章：[第 1 章 运行本地 Geth 开发节点](01-running-a-local-geth-dev-chain.md)
