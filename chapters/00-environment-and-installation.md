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

## 在 Ubuntu 上准备编译工具

在 Ubuntu 终端安装 Git、Make 和常用编译工具：

```bash
sudo apt update
sudo apt install -y git make build-essential
```

还需要安装 Go。请按 [Go 官方安装说明](https://go.dev/doc/install)安装与自己所选 Geth 源码兼容的 Go 工具链；作者用的是 `go1.25.8`，读者不必锁定这个版本。安装后检查：

```bash
go version
git --version
make --version
```

如编译时提示 Go 版本不满足要求，以所选源码的 `go.mod` 和编译错误信息为准，再更新 Go 工具链。

## 从源码编译 Geth

[Geth 官方文档](https://geth.ethereum.org/docs/getting-started/installing-geth#build-from-source-code)给出的 Linux 源码构建流程是克隆 `go-ethereum` 仓库，然后运行 `make geth`。这里把源码放在 `~/ethereum/go-ethereum`，与后续章节的命令保持一致：

```bash
mkdir -p ~/ethereum
cd ~/ethereum
git clone https://github.com/ethereum/go-ethereum.git
cd go-ethereum
make geth
```

直接克隆默认分支会得到克隆当时的源码，版本可能是 `unstable`，不会自动与作者的 `b10c4ecc` 相同。如果需要稳定版本或可重复的特定版本，请先从 [官方发布页](https://github.com/ethereum/go-ethereum/releases)选择并切换到相应的发布标签，再运行 `make geth`。不要把作者的开发版提交当作所有读者必须使用的版本。

官方构建命令会把可执行文件放在 `build/bin/geth`。检查构建结果，并记录自己的源码提交：

```bash
./build/bin/geth version
git rev-parse --short HEAD
```

如果找不到 `./build/bin/geth`，先检查 `make geth` 是否报错，以及当前目录是否为 `~/ethereum/go-ethereum`。之后的章节均用 `./build/bin/geth` 运行，不要求把程序安装到系统目录。

## 关于 Pebble 与开发链数据目录

第 1 章会创建独立的 `~/ethereum/dev-chain` 数据目录。若希望在**全新的数据目录**中明确使用 Pebble，可在第 1 章的第一次 Geth 启动命令中加上 `--db.engine=pebble`；不加这个参数也能完成本教程的实验。[Geth 数据库文档](https://geth.ethereum.org/docs/fundamentals/databases#pebble)说明，如果数据目录已有 LevelDB 数据，启动时添加该参数不会把旧数据库转换成 Pebble。不要为了切换引擎而删除已有实验数据；需要另建数据目录时，也要重新取得该链上的账户、地址和交易哈希。

---

下一章：[第 1 章 运行本地 Geth 开发节点](01-running-a-local-geth-dev-chain.md)
