# Tendermint RPC

Tendermint RPC允许您查询交易、区块、共识状态、广播交易等。

最新的Tendermint RPC文档可以在[这里](https://docs.tendermint.com/v0.34/rpc/)找到。Tendermint支持以下RPC协议：

- URI over HTTP
- JSON-RPC over HTTP
- JSON-RPC over Websockets

文档将包含一个交互式的Swagger界面。

## URI/HTTP

使用查询参数编码的GET请求：

```
curl localhost:26657/block?height=5
```

## RPC/HTTP

可以通过HTTP将JSONRPC请求POST到根RPC端点。使用Swagger查看支持的Tendermint RPC端点列表[here](../api#clients)。

## RPC/Websocket

### Cosmos和Tendermint事件

`Event`是包含有关应用程序执行的信息的对象，在提交块后触发。它们主要由服务提供商（如区块浏览器和钱包）用于跟踪各种消息的执行和索引交易。您可以在[这里](#list-of-tendermint-events)获取`event`类别和值的完整列表。

有关事件的更多信息：

- [Cosmos SDK事件](https://docs.cosmos.network/main/core/events.html)

### 通过Websocket订阅事件

Tendermint Core提供了一个[Websocket](https://docs.tendermint.com/v0.34/tendermint-core/subscription.html)连接，用于订阅或取消订阅Tendermint `Events`。要启动与Tendermint Websocket的连接，您需要在启动节点时使用`--rpc.laddr`标志定义地址（默认为`tcp://127.0.0.1:26657`）：

```bash
evmosd start --rpc.laddr="tcp://127.0.0.1:26657"
```

然后，使用[ws](https://github.com/hashrocket/ws)启动一个Websocket订阅：

```bash
# connect to tendermint websocket at port 8080
ws ws://localhost:8080/websocket

# subscribe to new Tendermint block headers
> { "jsonrpc": "2.0", "method": "subscribe", "params": ["tm.event='NewBlockHeader'"], "id": 1 }
```

`query`的`type`和`attribute`值允许您过滤特定的`event`。例如，Evmos上的以太坊交易（`MsgEthereumTx`）会触发一个类型为`ethermint`的`event`，并且具有`sender`和`recipient`作为`attributes`。订阅此`event`可以这样做：

```json
{
    "jsonrpc": "2.0",
    "method": "subscribe",
    "id": "0",
    "params": {
        "query": "tm.event='Tx' AND ethereum.recipient='hexAddress'"
    }
}
```

其中 `hexAddress` 是以太坊的十六进制地址（例如：`0x1122334455667788990011223344556677889900`）。

通用的语法如下：

```json
{
    "jsonrpc": "2.0",
    "method": "subscribe",
    "id": "0",
    "params": {
        "query": "tm.event='<event_value>' AND eventType.eventAttribute='<attribute_value>'"
    }
}
```

### Tendermint 事件列表

您可以订阅的主要事件包括：

- `NewBlock`：包含在 `BeginBlock` 和 `EndBlock` 期间触发的 `events`。
- `Tx`：包含在 `DeliverTx`（即交易处理）期间触发的 `events`。
- `ValidatorSetUpdates`：包含区块的验证者集更新。

:::tip
👉 您可以在 [Modules Specification](./../../../../protocol/modules/) 部分找到每个 Cosmos SDK 模块的事件类型和值列表。
请查看 `Events` 页面以获取 Evmos 上每个支持的模块的事件列表。
:::

Tendermint 事件键的列表如下：

|                                                      | 事件类型           | 类别        |
| ---------------------------------------------------- | ---------------- | ----------- |
| 订阅特定事件                                          | `"tm.event"`     | `block`     |
| 订阅特定交易                                          | `"tx.hash"`      | `block`     |
| 订阅特定块高度的交易                                  | `"tx.height"`    | `block`     |
| 索引 `BeginBlock` 和 `Endblock` 事件                  | `"block.height"` | `block`     |
| 订阅 ABCI `BeginBlock` 事件                           | `"begin_block"`  | `block`     |
| 订阅 ABCI `EndBlock` 事件                             | `"end_block"`    | `consensus` |

以下是您可以用于订阅 `tm.event` 类型的值列表：

|                        | 事件值                  | 类别        |
| ---------------------- | ----------------------- | ----------- |
| 新区块                 | `"NewBlock"`            | `block`     |
| 新区块头               | `"NewBlockHeader"`      | `block`     |
| 新拜占庭证据           | `"NewEvidence"`         | `block`     |
| 新交易                 | `"Tx"`                  | `block`     |
| 验证者集更新           | `"ValidatorSetUpdates"` | `block`     |
| 区块同步状态           | `"BlockSyncStatus"`     | `consensus` |
| 锁定                   | `"Lock"`                | `consensus` |
| 新共识轮               | `"NewRound"`            | `consensus` |
| Polka                  | `"Polka"`               | `consensus` |
| 重新锁定               | `"Relock"`              | `consensus` |
| 状态同步状态           | `"StateSyncStatus"`     | `consensus` |
| 提议超时               | `"TimeoutPropose"`      | `consensus` |
| 等待超时               | `"TimeoutWait"`         | `consensus` |
| 解锁                   | `"Unlock"`              | `consensus` |
| 区块有效               | `"ValidBlock"`          | `consensus` |
| 共识投票               | `"Vote"`                | `consensus` |

### 示例

```bash
ws ws://localhost:26657/websocket
> { "jsonrpc": "2.0", "method": "subscribe", "params": ["tm.event='ValidatorSetUpdates'"], "id": 1 }
```

示例响应：

```json
{
    "jsonrpc": "2.0",
    "id": 0,
    "result": {
        "query": "tm.event='ValidatorSetUpdates'",
        "data": {
            "type": "tendermint/event/ValidatorSetUpdates",
            "value": {
              "validator_updates": [
                {
                  "address": "09EAD022FD25DE3A02E64B0FE9610B1417183EE4",
                  "pub_key": {
                    "type": "tendermint/PubKeyEd25519",
                    "value": "ww0z4WaZ0Xg+YI10w43wTWbBmM3dpVza4mmSQYsd0ck="
                  },
                  "voting_power": "10",
                  "proposer_priority": "0"
                }
              ]
            }
        }
    }
}
```

:::tip
**注意：** 在查询以太坊交易和 Cosmos 交易时，交易哈希是不同的。
在查询以太坊交易时，用户需要使用事件查询。
以下是使用 CLI 的示例：

```bash
curl -X GET "http://localhost:26657/tx_search?query=ethereum_tx.ethereumTxHash%3D0x8d43464891fac6c113e809e14dff1a3e608eae124d629799e42ca0e36562d9d7&prove=false&page=1&per_page=30&order_by=asc" -H "accept: application/json"
```

:::


---
sidebar_position: 4
---

# Tendermint RPC

The Tendermint RPC allows you to query transactions, blocks, consensus state, broadcast transactions, etc.

The latest Tendermint RPC documentations can be found [here](https://docs.tendermint.com/v0.34/rpc/). Tendermint
supports the following RPC protocols:

- URI over HTTP
- JSON-RPC over HTTP
- JSON-RPC over Websockets

The docs will contain an interactive Swagger interface.

## URI/HTTP

A GET request with arguments encoded as query parameters:

```
curl localhost:26657/block?height=5
```

## RPC/HTTP

JSONRPC requests can be POST'd to the root RPC endpoint via HTTP. See the list
of supported Tendermint RPC endpoints using Swagger [here](../api#clients).

## RPC/Websocket

### Cosmos and Tendermint Events

`Event`s are objects that contain information about the execution of the application
and are triggered after a block is committed. They are mainly used by service providers
like block explorers and wallet to track the execution of various messages and index transactions.
You can get the full list of `event` categories and values [here](#list-of-tendermint-events).

More on Events:

- [Cosmos SDK Events](https://docs.cosmos.network/main/core/events.html)

### Subscribing to Events via Websocket

Tendermint Core provides a [Websocket](https://docs.tendermint.com/v0.34/tendermint-core/subscription.html) connection to subscribe or unsubscribe to Tendermint `Events`. To start a connection with the Tendermint websocket you need to define the address with the `--rpc.laddr` flag when starting the node (default `tcp://127.0.0.1:26657`):

```bash
evmosd start --rpc.laddr="tcp://127.0.0.1:26657"
```

Then, start a websocket subscription with [ws](https://github.com/hashrocket/ws)

```bash
# connect to tendermint websocket at port 8080
ws ws://localhost:8080/websocket

# subscribe to new Tendermint block headers
> { "jsonrpc": "2.0", "method": "subscribe", "params": ["tm.event='NewBlockHeader'"], "id": 1 }
```

The `type` and `attribute` value of the `query` allow you to filter the specific `event` you are
looking for. For example, an Ethereum transaction on Evmos (`MsgEthereumTx`) triggers an `event` of type `ethermint` and
has `sender` and `recipient` as `attributes`. Subscribing to this `event` would be done like so:

```json
{
    "jsonrpc": "2.0",
    "method": "subscribe",
    "id": "0",
    "params": {
        "query": "tm.event='Tx' AND ethereum.recipient='hexAddress'"
    }
}
```

where `hexAddress` is an Ethereum hex address (eg: `0x1122334455667788990011223344556677889900`).

The generic syntax looks like this:

```json
{
    "jsonrpc": "2.0",
    "method": "subscribe",
    "id": "0",
    "params": {
        "query": "tm.event='<event_value>' AND eventType.eventAttribute='<attribute_value>'"
    }
}
```

### List of Tendermint Events

The main events you can subscribe to are:

- `NewBlock`: Contains `events` triggered during `BeginBlock` and `EndBlock`.
- `Tx`: Contains `events` triggered during `DeliverTx` (i.e. transaction processing).
- `ValidatorSetUpdates`: Contains validator set updates for the block.

:::tip
👉 The list of events types and values for each Cosmos SDK module can be found in the [Modules Specification](./../../../../protocol/modules/) section.
Check the `Events` page to obtain the event list of each supported module on Evmos.
:::

List of all Tendermint event keys:

|                                                      | Event Type       | Categories  |
| ---------------------------------------------------- | ---------------- | ----------- |
| Subscribe to a specific event                        | `"tm.event"`     | `block`     |
| Subscribe to a specific transaction                  | `"tx.hash"`      | `block`     |
| Subscribe to transactions at a specific block height | `"tx.height"`    | `block`     |
| Index `BeginBlock` and `Endblock` events             | `"block.height"` | `block`     |
| Subscribe to ABCI `BeginBlock` events                | `"begin_block"`  | `block`     |
| Subscribe to ABCI `EndBlock` events                  | `"end_block"`    | `consensus` |

Below is a list of values that you can use to subscribe for the `tm.event` type:

|                        | Event Value             | Categories  |
| ---------------------- | ----------------------- | ----------- |
| New block              | `"NewBlock"`            | `block`     |
| New block header       | `"NewBlockHeader"`      | `block`     |
| New Byzantine Evidence | `"NewEvidence"`         | `block`     |
| New transaction        | `"Tx"`                  | `block`     |
| Validator set updated  | `"ValidatorSetUpdates"` | `block`     |
| Block sync status      | `"BlockSyncStatus"`     | `consensus` |
| lock                   | `"Lock"`                | `consensus` |
| New consensus round    | `"NewRound"`            | `consensus` |
| Polka                  | `"Polka"`               | `consensus` |
| Relock                 | `"Relock"`              | `consensus` |
| State sync status      | `"StateSyncStatus"`     | `consensus` |
| Timeout propose        | `"TimeoutPropose"`      | `consensus` |
| Timeout wait           | `"TimeoutWait"`         | `consensus` |
| Unlock                 | `"Unlock"`              | `consensus` |
| Block is valid         | `"ValidBlock"`          | `consensus` |
| Consensus vote         | `"Vote"`                | `consensus` |

### Example

```bash
ws ws://localhost:26657/websocket
> { "jsonrpc": "2.0", "method": "subscribe", "params": ["tm.event='ValidatorSetUpdates'"], "id": 1 }
```

Example response:

```json
{
    "jsonrpc": "2.0",
    "id": 0,
    "result": {
        "query": "tm.event='ValidatorSetUpdates'",
        "data": {
            "type": "tendermint/event/ValidatorSetUpdates",
            "value": {
              "validator_updates": [
                {
                  "address": "09EAD022FD25DE3A02E64B0FE9610B1417183EE4",
                  "pub_key": {
                    "type": "tendermint/PubKeyEd25519",
                    "value": "ww0z4WaZ0Xg+YI10w43wTWbBmM3dpVza4mmSQYsd0ck="
                  },
                  "voting_power": "10",
                  "proposer_priority": "0"
                }
              ]
            }
        }
    }
}
```

:::tip
**Note:** When querying Ethereum transactions versus Cosmos transactions, the transaction hashes are different.
When querying Ethereum transactions, users need to use event query.
Here's an example with the CLI:

```bash
curl -X GET "http://localhost:26657/tx_search?query=ethereum_tx.ethereumTxHash%3D0x8d43464891fac6c113e809e14dff1a3e608eae124d629799e42ca0e36562d9d7&prove=false&page=1&per_page=30&order_by=asc" -H "accept: application/json"
```

:::
