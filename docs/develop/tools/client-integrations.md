---
sidebar_position: 6
---

# Evmos 客户端集成

客户端集成库在区块链技术中起着至关重要的作用，它使开发人员更容易与区块链网络进行交互。这些库将复杂性抽象化，并提供集成和方法，使开发人员能够以更一致的方式创建产品。

## Evmos 特定的客户端集成

Evmos 特定的库可帮助开发人员加快开发速度，提供接口、类型和方法来进行签名、地址转换（在 `eth` 和 `evmos` 地址之间）、以及 `EIP-712` 交易生成。有两个库绑定，分别是 Javascript/Typescript 和 Python。

- [EvmosJS](https://github.com/evmos/evmosjs) - 是官方的 Evmos 客户端 Typescript 库。该库包含以下几个包：
    - [地址转换器](https://www.npmjs.com/package/@evmos/address-converter)
    - [EIP-712](https://www.npmjs.com/package/@evmos/eip712)
    - [Proto](https://www.npmjs.com/package/@evmos/proto)
    - [提供者](https://www.npmjs.com/package/@evmos/provider)
    - [交易](https://www.npmjs.com/package/@evmos/transactions)
- [PyEvmos](https://github.com/sterliakov/pyevmos) - 是由 [sterliakov](https://github.com/sterliakov) 开发的社区驱动的 Python 库。

## 以太坊客户端集成

EthersJS 和 Web3JS 是 dApp 开发中最常用的两个库。开发人员使用这些库与区块链进行交互并查询 JSON-RPC 数据，例如。此外，这两个库都包含一些实用工具，用于辅助完成任务，如转换大数（BigNumber）。

- [Ethers.js](https://docs.ethers.org/v5/) 是最新的 JS 库，旨在成为与以太坊区块链及其生态系统进行交互的完整而紧凑的库。
- [web3js](https://web3js.readthedocs.io/en/v1.8.2/) 是一组库，允许您使用 HTTP、IPC 或 WebSocket 与本地或远程以太坊节点进行交互。


---
sidebar_position: 6
---

# Evmos Client Integrations

Client integration libraries play a crucial role in blockchain technology by making it easier for developers to interact
with the blockchain network. Libraries abstract away complexities and provide integrations and methods to allow developers
to create product in a more consistent manner.

## Evmos-specific Client Integrations

Evmos-specific libraries are useful in aiding developers speed up development by providing interfaces, types, and methods
to signing, address converter (between `eth` and `evmos` addresses), and `EIP-712` transaction generator. There are two
library bindings, in Javascript/Typescript and Python.

- [EvmosJS](https://github.com/evmos/evmosjs) - is the official Evmos client Typescript library. This library contains
 several packages:
    - [Address Converter](https://www.npmjs.com/package/@evmos/address-converter)
    - [EIP-712](https://www.npmjs.com/package/@evmos/eip712)
    - [Proto](https://www.npmjs.com/package/@evmos/proto)
    - [Provider](https://www.npmjs.com/package/@evmos/provider)
    - [Transactions](https://www.npmjs.com/package/@evmos/transactions)
- [PyEvmos](https://github.com/sterliakov/pyevmos) - is a community-led Python library developed by [sterliakov](https://github.com/sterliakov)

## Ethereum Client Integrations

EthersJS and Web3JS are two most commonly used libraries in dApp development. Developer uses these libraries to interact
with blockchain and query JSON-RPC data, for example. Additionally, both of these libraries contain utilities to aid in
task like converting large numbers (BigNumber).

- [Ethers.js](https://docs.ethers.org/v5/) is the latest JS library that aims to be a complete and compact library for
interacting with the Ethereum Blockchain and its ecosystem.
- [web3js](https://web3js.readthedocs.io/en/v1.8.2/) is a collection of libraries that allow you to interact with a local
or remote ethereum node using HTTP, IPC or WebSocket.
