---
sidebar_position: 0
---

# 技术架构

Evmos是一个可扩展的权益证明区块链，与以太坊虚拟机（EVM）完全兼容和互操作。它是使用[Cosmos SDK](https://github.com/cosmos/cosmos-sdk/)构建的，
该SDK运行在[CometBFT](https://github.com/cometbft/cometbft)（[Tendermint Core](https://docs.tendermint.com/)的分叉）共识引擎之上，
以实现快速确定性、高交易吞吐量和短块时间（约2秒）。

这种架构允许用户执行Cosmos和EVM格式的交易，
开发人员通过[IBC](https://cosmos.network/ibc)跨链扩展EVM dApps，
网络中的代币和资产可以来自不同的独立来源。

Evmos通过以下方式实现了这些关键功能：

* 利用[Cosmos SDK](https://docs.cosmos.network/)实现的[模块](https://docs.cosmos.network/main/building-modules/intro.html)和其他机制。
* 实现CometBFT的应用区块链接口（[ABCI](https://docs.tendermint.com/master/spec/abci/)）来管理区块链。
* 利用[`geth`](https://github.com/ethereum/go-ethereum)作为库来促进代码重用和提高可维护性。
* 提供一个完全兼容的Web3 [JSON-RPC](./../develop/api/ethereum-json-rpc/methods)层，用于与现有的以太坊客户端和工具（Metamask、Remix、Truffle等）进行交互。

这些功能的总和使开发人员能够利用现有的以太坊生态系统工具和软件，
无缝部署与Cosmos[生态系统](https://cosmos.network/ecosystem)其他部分交互的智能合约。

## Cosmos SDK

Evmos实现了[Cosmos SDK](https://docs.cosmos.network/)的完全组合性和模块化性。
作为一个Cosmos链，Evmos是一个拥有自己本地代币的主权区块链，
可以通过IBC与其他链进行连接。它包括来自Cosmos SDK的标准模块，
与Evmos核心开发团队构建的Evmos特定模块并行工作。
请查看[模块列表](modules/index.md)以了解每个模块的职责概述。

## CometBFT & ABCI

[CometBFT](https://github.com/cometbft/cometbft)由两个主要技术组件组成：
区块链共识引擎和通用应用接口。
共识引擎确保相同的交易以相同的顺序记录在每台机器上。
名为[Application Blockchain Interface (ABCI)](https://docs.tendermint.com/master/spec/abci/)的应用接口使得可以使用任何编程语言处理交易。

CometBFT已经发展成为一个通用的区块链共识引擎，可以托管任意应用状态。
由于它可以复制任意应用程序，因此可以用作其他区块链共识引擎的即插即用替代品。
Evmos是一个通过CometBFT的共识引擎替代以太坊的PoW的ABCI应用的示例。

另一个构建在CometBFT上的加密货币应用示例是Cosmos网络。
CometBFT可以通过在应用程序进程和共识进程之间提供一个非常简单的API（即ABCI）来分解区块链设计。

## EVM兼容性

Evmos通过实现各种组件实现了EVM兼容性，
这些组件共同支持所有EVM状态转换，
同时确保与以太坊相同的开发者体验：

- 以Cosmos SDK `Tx` 和 `Msg` 接口的形式实现了以太坊的交易格式
- 以Cosmos Keyring的`secp256k1`曲线实现
- 用于状态更新和查询的`StateDB`接口
- 用于与EVM交互的[JSON-RPC](../develop/api/ethereum-json-rpc)客户端

大多数组件都是在[EVM模块](modules/evm.md)中实现的。
然而，为了实现无缝的开发者体验，其中一些组件是在模块之外实现的。

如果您想了解Evmos作为Cosmos链如何实现EVM兼容性的更多信息，
我们建议您了解以下概念：

* [账户](./concepts/accounts.md)
* [Gas和费用](./concepts/gas-and-fees.md)
* [代币表示](./concepts/tokens.md)
* [交易](./concepts/transactions.md)

## 贡献

有几种方式可以为Evmos核心协议做出贡献。为了获得一些实践经验，我们建议您使用[Evmos CLI](./protocol/evmos-cli)在本地启动一个Evmos节点，并通过查询和交易使用支持的[客户端](../develop/api#clients)与其进行交互。

然后，如果您对此感兴趣，您可以：

* 在GitHub上为[问题](https://github.com/evmos/evmos/issues)做出开源贡献，使用[Evmos贡献者指南](https://github.com/evmos/evmos/blob/main/CONTRIBUTING.md)
* 申请[Evmos的开放职位](https://boards.eu.greenhouse.io/evmos)
* 搜索[错误并获得赏金](bugs.md)


---
sidebar_position: 0
---

# Technical Architecture

Evmos is a scalable Proof-of-Stake blockchain that is fully compatible and
interoperable with the Ethereum Virtual Machine (EVM). It is built using the [Cosmos SDK](https://github.com/cosmos/cosmos-sdk/)
which runs on top of the [CometBFT](https://github.com/cometbft/cometbft) (a fork of [Tendermint Core](https://docs.tendermint.com/)) consensus engine,
to accomplish fast finality, high transaction throughput and short block times (~2 seconds).

This architecture allows users to perform both Cosmos and EVM formatted transactions,
developers to scale EVM dApps cross-chain via [IBC](https://cosmos.network/ibc),
and tokens and assets in the network to come from different independent sources.

Evmos enables these key features by:

* Leveraging [modules](https://docs.cosmos.network/main/building-modules/intro.html) and other mechanisms implemented by the [Cosmos SDK](https://docs.cosmos.network/).
* Implementing CometBFT's Application Blockchain Interface ([ABCI](https://docs.tendermint.com/master/spec/abci/)) to manage the blockchain.
* Utilizing [`geth`](https://github.com/ethereum/go-ethereum) as a library to promote code reuse and improve maintainability.
* Exposing a fully compatible Web3 [JSON-RPC](./../develop/api/ethereum-json-rpc/methods) layer for interacting with existing Ethereum clients and tooling (Metamask, Remix, Truffle, etc).

The sum of these features allows developers to leverage existing Ethereum ecosystem tooling and
software to seamlessly deploy smart contracts which interact with the rest of the Cosmos
[ecosystem](https://cosmos.network/ecosystem).

## Cosmos SDK

Evmos enables the full composability and modularity of the [Cosmos SDK](https://docs.cosmos.network/).
As a Cosmos chain, Evmos is a sovereign blockchain with its own native token,
that can connect to other chains through IBC. It includes standard modules from the Cosmos SDK,
that work side to side with Evmos-specific modules, built by the Evmos core development team.
Check out the [list of modules](modules/index.md) to get an overview of what each module is responsible for.

## CometBFT & ABCI

[CometBFT](https://github.com/cometbft/cometbft) consists of two chief technical components:
a blockchain consensus engine and a generic application interface.
The consensus engine ensures that the same transactions
are recorded on every machine in the same order.
The application interface, called the [Application Blockchain Interface (ABCI)](https://docs.tendermint.com/master/spec/abci/),
enables the transactions to be processed in any programming language.

CometBFT has evolved to be a general-purpose blockchain consensus engine that
can host arbitrary application states. Since it can replicate arbitrary
applications, it can be used as a plug-and-play replacement for the consensus
engines of other blockchains. Evmos is an example of an ABCI application
replacing Ethereum's PoW via CometBFT's consensus engine.

Another example of a cryptocurrency application built on CometBFT is the Cosmos
network. CometBFT can decompose the blockchain design by offering a very
simple API (ie. the ABCI) between the application process and consensus process.

## EVM Compatibility

Evmos enables EVM compatibility by implementing various components
that together support all the EVM state transitions
while ensuring the same developer experience as Ethereum:

- Ethereum's transaction format as a Cosmos SDK `Tx` and `Msg` interface
- Ethereum's `secp256k1` curve for the Cosmos Keyring
- `StateDB` interface for state updates and queries
- [JSON-RPC](../develop/api/ethereum-json-rpc) client for interacting with the EVM

Most components are implemented in the [EVM module](modules/evm.md)
To achieve a seamless developer UX, however, some of the components are implemented
outside of the module.

If you want to learn more about how Evmos achieves EVM compatibility as a Cosmos chain,
we recommend understanding the following concepts:

* [Accounts](./concepts/accounts.md)
* [Gas and Fees](./concepts/gas-and-fees.md)
* [Token representations](./concepts/tokens.md)
* [Transactions](./concepts/transactions.md)

## Contributing

There are several ways to contribute to the Evmos core protocol. To get some hands-on experience,
we recommend you spin up a local Evmos node using the [Evmos CLI](./protocol/evmos-cli)
and interact with it through queries and transactions using the supported [clients](../develop/api#clients).

Then if you're hooked you can

* Contribute open-source to [issues on GitHub](https://github.com/evmos/evmos/issues)
using the [Evmos Contributor Guideline](https://github.com/evmos/evmos/blob/main/CONTRIBUTING.md)
* Apply to [open positions at Evmos](https://boards.eu.greenhouse.io/evmos)
* Search for [bugs and earn a bounty](bugs.md)
