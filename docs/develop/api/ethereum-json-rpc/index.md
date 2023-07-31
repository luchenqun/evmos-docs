---
sidebar_position: 1
---

# 以太坊 JSON-RPC

JSON-RPC 服务器提供了一个 API，允许您连接到 Evmos 区块链并与 EVM 进行交互。这使您可以直接访问以太坊格式的交易或将其发送到网络，而这在 Cosmos 链（如 Evmos）上是不可能的。

[JSON-RPC](http://www.jsonrpc.org/specification) 是一种无状态、轻量级的远程过程调用（RPC）协议。它定义了几种数据结构及其处理规则。JSON-RPC 可以在多个传输方式上提供。Evmos 支持通过 HTTP 和 WebSocket 进行 JSON-RPC。传输方式必须通过命令行标志或 `app.toml` 配置文件启用。它使用 JSON（[RFC 4627](https://www.ietf.org/rfc/rfc4627.txt)）作为数据格式。

有关以太坊 JSON-RPC 的更多信息：

- [EthWiki JSON-RPC API](https://eth.wiki/json-rpc/API)
- [Geth JSON-RPC Server](https://geth.ethereum.org/docs/interacting-with-geth/rpc)
- [以太坊的 PubSub JSON-RPC API](https://geth.ethereum.org/docs/interacting-with-geth/rpc/pubsub)

:::note
请访问我们的学院，了解有关我们的 [Geth JavaScript 控制台指南](https://academy.evmos.org/articles/advanced/geth-js-console) 的更多信息。
:::

## HTTP 上的 JSON-RPC

Evmos 支持大多数标准的 web3 JSON-RPC API，以便通过 HTTP 连接到现有的与以太坊兼容的 web3 工具。以太坊 JSON-RPC API 使用命名空间系统。RPC 方法根据其用途分组到多个类别中。所有方法名称由命名空间、下划线和命名空间内的实际方法名称组成。例如，`eth_call` 方法位于 eth 命名空间中。可以按命名空间启用对 RPC 方法的访问。

以下是 Evmos 支持的 JSON-RPC 命名空间，或者请访问 [JSON-RPC 方法](./methods.md) 页面上的文档，了解各个 API 端点及其相应的 curl 命令。

| 命名空间                                           | 描述                                                                                                                                                                                                                      | 支持 | 默认启用 |
| ------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | --- | -------- |
| [`eth`](./ethereum-json-rpc/methods#eth-methods)           | Evmos 提供了几个扩展到标准 `eth` JSON-RPC 命名空间的方法。                                                                                                                                                              | ✔  | ✔        |
| [`web3`](./ethereum-json-rpc/methods#web3-methods)         | `web3` API 提供了用于 web3 客户端的实用函数。                                                                                                                                                                           | ✔  | ✔        |
| [`net`](./ethereum-json-rpc/methods#net-methods)           | `net` API 提供了对节点的网络信息的访问。                                                                                                                                                                                 | ✔  | ✔        |
| `clique`                                          | `clique` API 提供了对 clique 共识引擎状态的访问。您可以使用此 API 管理签名者投票并检查私有网络的健康状况。                                                                                                                  | 🚫  |           |
| `debug`                                           | `debug` API 允许您访问几个非标准的 RPC 方法，这些方法允许您在运行时检查、调试和设置某些调试标志。                                                                                                                            | ✔  |           |
| `les`                                             | `les` API 允许您管理 LES 服务器设置，包括优先客户端的客户端参数和支付设置。它还提供了在服务器和客户端模式下查询检查点信息的功能。                                                                                                 | 🚫  |           |
| [`miner`](./ethereum-json-rpc/methods#miner-methods)       | `miner` API 允许您远程控制节点的挖矿操作并设置各种挖矿特定设置。                                                                                                                                                           | ✔  | 🚫        |
| [`txpool`](./ethereum-json-rpc/methods#txpool-methods)     | `txpool` API 允许您访问几个非标准的 RPC 方法，以检查包含当前所有待处理事务以及排队等待未来处理的事务的事务池的内容。                                                                                                         | ✔  | 🚫        |
| `admin`                                           | `admin` API 允许您访问几个非标准的 RPC 方法，这些方法允许您对节点实例进行细粒度的控制，包括但不限于网络对等方和 RPC 端点管理。                                                                                                 | 🚫  |           |
| [`personal`](./ethereum-json-rpc/methods#personal-methods) | `personal` API 管理密钥库中的私钥。                                                                                                                                                                                    | ✔  | 🚫        |

## 订阅以太坊事件

### 过滤器

Evmos还支持以太坊的[JSON-RPC](./ethereum-json-rpc/methods)过滤器调用，用于订阅[状态日志](https://eth.wiki/json-rpc/API#eth_newfilter)、[区块](https://eth.wiki/json-rpc/API#eth_newblockfilter)或[待处理交易](https://eth.wiki/json-rpc/API#eth_newpendingtransactionfilter)的变化。

在底层，它使用Tendermint RPC客户端的事件系统来处理订阅，然后将其格式化为与以太坊兼容的事件。

```bash
curl -X POST --data '{"jsonrpc":"2.0","method":"eth_newBlockFilter","params":[],"id":1}' -H "Content-Type: application/json" http://localhost:8545

{"jsonrpc":"2.0","id":1,"result":"0x3503de5f0c766c68f78a03a3b05036a5"}
```

然后，您可以使用[`eth_getFilterChanges`](https://eth.wiki/json-rpc/API#eth_getfilterchanges)调用来检查状态的变化：

```bash
curl -X POST --data '{"jsonrpc":"2.0","method":"eth_getFilterChanges","params":["0x3503de5f0c766c68f78a03a3b05036a5"],"id":1}' -H "Content-Type: application/json" http://localhost:8545

{"jsonrpc":"2.0","id":1,"result":["0x7d44dceff05d5963b5bc81df7e9f79b27e777b0a03a6feca09f3447b99c6fa71","0x3961e4050c27ce0145d375255b3cb829a5b4e795ac475c05a219b3733723d376","0xd7a497f95167d63e6feca70f344d9f6e843d097b62729b8f43bdcd5febf142ab","0x55d80a4ba6ef54f2a8c0b99589d017b810ed13a1fda6a111e1b87725bc8ceb0e","0x9e8b92c17280dd05f2562af6eea3285181c562ebf41fc758527d4c30364bcbc4","0x7353a4b9d6b35c9eafeccaf9722dd293c46ae2ffd4093b2367165c3620a0c7c9","0x026d91bda61c8789c59632c349b38fd7e7557e6b598b94879654a644cfa75f30","0x73e3245d4ddc3bba48fa67633f9993c6e11728a36401fa1206437f8be94ef1d3"]}
```

### 以太坊 Websocket

以太坊 Websocket 允许您订阅以太坊日志和智能合约中发出的事件。这样，当您需要特定信息时，您就不需要不断地发出请求。

由于 Evmos 是使用 Cosmos SDK 框架构建的，并使用 Tendermint Core 作为其共识引擎，它继承了它们的[event format](./tendermint-rpc#subscribing-to-cosmos-and-tendermint-events)。然而，为了支持[Ethereum 的 PubSubAPI](https://geth.ethereum.org/docs/interacting-with-geth/rpc/pubsub)的本机 Web3 兼容性，Evmos 需要将检索到的 Tendermint 响应转换为以太坊类型。

您可以在启动节点时使用 `--json-rpc.ws-address` 标志与以太坊 Websocket 建立连接（默认为 `"0.0.0.0:8546"`）：

```bash
evmosd start --json-rpc.address="0.0.0.0:8545" --json-rpc.ws-address="0.0.0.0:8546" --json-rpc.api="eth,web3,net,txpool,debug" --json-rpc.enable
```

然后，使用 [`ws`](https://github.com/hashrocket/ws) 开始一个 Websocket 订阅：

```bash
# connect to tendermint websocket at port 8546 as defined above
ws ws://localhost:8546/

# subscribe to new Ethereum-formatted block Headers
> {"id": 1, "method": "eth_subscribe", "params": ["newHeads", {}]}
< {"jsonrpc":"2.0","result":"0x44e010cb2c3161e9c02207ff172166ef","id":1}
```

## 进一步考虑

### HEX 值编码

目前，有两种关键数据类型通过 JSON 传递：

* **数量**和
* **未格式化的字节数组**。

两者都使用十六进制编码，但格式要求不同。

当编码数量（整数、数字）时，请使用十六进制编码，前缀为`"0x"`，使用最紧凑的表示形式（稍有例外：零应表示为`"0x0"`）。示例：

- `0x41`（十进制为65）
- `0x400`（十进制为1024）
- 错误：`0x`（应始终至少有一位数字 - 零为`"0x0"`）
- 错误：`0x0400`（不允许前导零）
- 错误：`ff`（必须加前缀`0x`）

当编码未格式化的数据（字节数组、账户地址、哈希、字节码数组）时，请使用十六进制编码，前缀为`"0x"`，每个字节两个十六进制数字。示例：

- `0x41`（大小为1，`"A"`）
- `0x004200`（大小为3，`"\0B\0"`）
- `0x`（大小为0，`""`）
- 错误：`0xf0f0f`（必须为偶数位数字）
- 错误：`004200`（必须加前缀`0x`）

### 默认块参数

以下方法有一个额外的默认块参数：

- [`eth_getBalance`](./ethereum-json-rpc/methods#eth_getbalance)
- [`eth_getCode`](./ethereum-json-rpc/methods#eth_getcode)
- [`eth_getTransactionCount`](./ethereum-json-rpc/methods#eth_gettransactioncount)
- [`eth_getStorageAt`](./ethereum-json-rpc/methods#eth_getstorageat)
- [`eth_call`](./ethereum-json-rpc/methods#eth_call)

当发出对 Evmos 状态进行操作的请求时，最后一个默认块参数确定块的高度。

`defaultBlock` 参数可以有以下选项：

- `HEX 字符串` - 整数块号
- 字符串 `"earliest"` 表示最早/创世块
- 字符串 `"latest"` - 表示最新的已挖掘块
- 字符串 `"pending"` - 表示待处理状态/交易


---
sidebar_position: 1
---

# Ethereum JSON-RPC

The JSON-PRC Server provides an API that allows you to connect to the Evmos blockchain and interact with the EVM. This
gives you direct access to reading Ethereum-formatted transactions or sending them to the network which otherwise
wouldn't be possible on a Cosmos chain, such as Evmos.

[JSON-RPC](http://www.jsonrpc.org/specification) is a stateless, light-weight remote procedure call (RPC) protocol. It
defines several data structures and the rules around their processing. JSON-RPC is provided on multiple transports.
Evmos supports JSON-RPC over HTTP and WebSocket. Transports must be enabled through command-line flags or through the
`app.toml` configuration file. It uses JSON ([RFC 4627](https://www.ietf.org/rfc/rfc4627.txt)) as data format.

More on Ethereum JSON-RPC:

- [EthWiki JSON-RPC API](https://eth.wiki/json-rpc/API)
- [Geth JSON-RPC Server](https://geth.ethereum.org/docs/interacting-with-geth/rpc)
- [Ethereum's PubSub JSON-RPC API](https://geth.ethereum.org/docs/interacting-with-geth/rpc/pubsub)

:::note
Head over to our Academy to learn about our [Geth Javascript Console Guide](https://academy.evmos.org/articles/advanced/geth-js-console).
:::

## JSON-RPC over HTTP

Evmos supports most of the standard web3 JSON-RPC APIs to connect with existing Ethereum-compatible web3 tooling over
HTTP. Ethereum JSON-RPC APIs use a namespace system. RPC methods are grouped into several categories depending on
their purpose. All method names are composed of the namespace, an underscore, and the actual method name within
the namespace. For example, the `eth_call` method resides in the eth namespace. Access to RPC methods can be enabled
on a per-namespace basis.

Find below the JSON-RPC namespaces supported on Evmos or head over to the documentation for the individual API endpoints
and their respective curl commands on the [JSON-RPC Methods](./methods.md) page.

| Namespace                                         | Description                                                                                                                                                                                                                  | Supported | Enabled by Default |
| ------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------- | ------------------ |
| [`eth`](./ethereum-json-rpc/methods#eth-methods)           | Evmos provides several extensions to the standard `eth` JSON-RPC namespace.                                                                                                                                                  | ✔        | ✔                 |
| [`web3`](./ethereum-json-rpc/methods#web3-methods)         | The `web3` API provides utility functions for the web3 client.                                                                                                                                                               | ✔        | ✔                 |
| [`net`](./ethereum-json-rpc/methods#net-methods)           | The `net` API provides access to network information of the node                                                                                                                                                             | ✔        | ✔                 |
| `clique`                                          | The `clique` API provides access to the state of the clique consensus engine. You can use this API to manage signer votes and to check the health of a private network.                                                      | 🚫        |                    |
| `debug`                                           | The `debug` API gives you access to several non-standard RPC methods, which will allow you to inspect, debug and set certain debugging flags during runtime.                                                                 | ✔        |                    |
| `les`                                             | The `les` API allows you to manage LES server settings, including client parameters and payment settings for prioritized clients. It also provides functions to query checkpoint information in both server and client mode. | 🚫        |                    |
| [`miner`](./ethereum-json-rpc/methods#miner-methods)       | The `miner` API allows you to remote control the node’s mining operation and set various mining specific settings.                                                                                                           | ✔        | 🚫                 |
| [`txpool`](./ethereum-json-rpc/methods#txpool-methods)     | The `txpool` API gives you access to several non-standard RPC methods to inspect the contents of the transaction pool containing all the currently pending transactions as well as the ones queued for future processing.    | ✔        | 🚫                 |
| `admin`                                           | The `admin` API gives you access to several non-standard RPC methods, which will allow you to have a fine grained control over your node instance, including but not limited to network peer and RPC endpoint management.     | 🚫        |                    |
| [`personal`](./ethereum-json-rpc/methods#personal-methods) | The `personal` API manages private keys in the key store.                                                                                                                                                                    | ✔        | 🚫                 |

## Subscribing to Ethereum Events

### Filters

Evmos also supports the Ethereum [JSON-RPC](./ethereum-json-rpc/methods) filters calls to
subscribe to [state logs](https://eth.wiki/json-rpc/API#eth_newfilter),
[blocks](https://eth.wiki/json-rpc/API#eth_newblockfilter) or [pending transactions](https://eth.wiki/json-rpc/API#eth_newpendingtransactionfilter) changes.

Under the hood, it uses the Tendermint RPC client's event system to process subscriptions that are
then formatted to Ethereum-compatible events.

```bash
curl -X POST --data '{"jsonrpc":"2.0","method":"eth_newBlockFilter","params":[],"id":1}' -H "Content-Type: application/json" http://localhost:8545

{"jsonrpc":"2.0","id":1,"result":"0x3503de5f0c766c68f78a03a3b05036a5"}
```

Then you can check if the state changes with the [`eth_getFilterChanges`](https://eth.wiki/json-rpc/API#eth_getfilterchanges) call:

```bash
curl -X POST --data '{"jsonrpc":"2.0","method":"eth_getFilterChanges","params":["0x3503de5f0c766c68f78a03a3b05036a5"],"id":1}' -H "Content-Type: application/json" http://localhost:8545

{"jsonrpc":"2.0","id":1,"result":["0x7d44dceff05d5963b5bc81df7e9f79b27e777b0a03a6feca09f3447b99c6fa71","0x3961e4050c27ce0145d375255b3cb829a5b4e795ac475c05a219b3733723d376","0xd7a497f95167d63e6feca70f344d9f6e843d097b62729b8f43bdcd5febf142ab","0x55d80a4ba6ef54f2a8c0b99589d017b810ed13a1fda6a111e1b87725bc8ceb0e","0x9e8b92c17280dd05f2562af6eea3285181c562ebf41fc758527d4c30364bcbc4","0x7353a4b9d6b35c9eafeccaf9722dd293c46ae2ffd4093b2367165c3620a0c7c9","0x026d91bda61c8789c59632c349b38fd7e7557e6b598b94879654a644cfa75f30","0x73e3245d4ddc3bba48fa67633f9993c6e11728a36401fa1206437f8be94ef1d3"]}
```

### Ethereum Websocket

The Ethereum Websocket allows you to subscribe to Ethereum logs and events emitted in smart contracts. This way you
don't need to continuously make requests when you want specific information.

Since Evmos is built with the Cosmos SDK framework and uses Tendermint Core as it's consensus Engine, it inherits the
[event format](./tendermint-rpc#subscribing-to-cosmos-and-tendermint-events) from them. However, in order to support the
native Web3 compatibility for websockets of the [Ethereum's PubSubAPI](https://geth.ethereum.org/docs/interacting-with-geth/rpc/pubsub),
Evmos needs to cast the Tendermint
responses retrieved into the Ethereum types.

You can start a connection with the Ethereum websocket using the `--json-rpc.ws-address` flag when starting
the node (default `"0.0.0.0:8546"`):

```bash
evmosd start --json-rpc.address="0.0.0.0:8545" --json-rpc.ws-address="0.0.0.0:8546" --json-rpc.api="eth,web3,net,txpool,debug" --json-rpc.enable
```

Then, start a websocket subscription with [`ws`](https://github.com/hashrocket/ws)

```bash
# connect to tendermint websocket at port 8546 as defined above
ws ws://localhost:8546/

# subscribe to new Ethereum-formatted block Headers
> {"id": 1, "method": "eth_subscribe", "params": ["newHeads", {}]}
< {"jsonrpc":"2.0","result":"0x44e010cb2c3161e9c02207ff172166ef","id":1}
```

## Further Considerations

### HEX value encoding

At present there are two key datatypes that are passed over JSON:

* **quantities** and
* **unformatted byte arrays**.

Both are passed with a hex encoding, however with different requirements to formatting.

When encoding quantities (integers, numbers), encode as hex, prefix with `"0x"`, the most compact representation (slight
exception: zero should be represented as `"0x0"`). Examples:

- `0x41` (65 in decimal)
- `0x400` (1024 in decimal)
- WRONG: `0x` (should always have at least one digit - zero is `"0x0"`)
- WRONG: `0x0400` (no leading zeroes allowed)
- WRONG: `ff` (must be prefixed `0x`)

When encoding unformatted data (byte arrays, account addresses, hashes, bytecode arrays), encode as hex, prefix with `"0x"`,
two hex digits per byte. Examples:

- `0x41` (size 1, `"A"`)
- `0x004200` (size 3, `"\0B\0"`)
- `0x` (size 0, `""`)
- WRONG: `0xf0f0f` (must be even number of digits)
- WRONG: `004200` (must be prefixed `0x`)

### Default block parameter

The following methods have an extra default block parameter:

- [`eth_getBalance`](./ethereum-json-rpc/methods#eth_getbalance)
- [`eth_getCode`](./ethereum-json-rpc/methods#eth_getcode)
- [`eth_getTransactionCount`](./ethereum-json-rpc/methods#eth_gettransactioncount)
- [`eth_getStorageAt`](./ethereum-json-rpc/methods#eth_getstorageat)
- [`eth_call`](./ethereum-json-rpc/methods#eth_call)

When requests are made that act on the state of Evmos, the last default block parameter determines the height of the block.

The following options are possible for the `defaultBlock` parameter:

- `HEX String` - an integer block number
- `String "earliest"` for the earliest/genesis block
- `String "latest"` - for the latest mined block
- `String "pending"` - for the pending state/transactions
