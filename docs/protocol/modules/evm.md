# `evm`

## 摘要

本文档定义了以 Cosmos SDK 模块形式的以太坊虚拟机（EVM）规范。

自从 2015 年以太坊的引入以来，
通过[**智能合约**](https://www.fon.hum.uva.nl/rob/Courses/InformationInSpeech/CDROM/Literature/LOTwinterschool2006/szabo.best.vwh.net/idea.html)控制数字资产的能力
吸引了大量开发者社区
在以太坊虚拟机（EVM）上构建去中心化应用。
这个社区不断创建广泛的工具和引入标准，
进一步提高了 EVM 兼容技术的采用率。

然而，基于 EVM 的链（例如以太坊）揭示了几个可扩展性挑战，
通常被称为[去中心化、安全性和可扩展性的三难题](https://vitalik.ca/general/2021/04/07/sharding.html)。
开发者对高昂的燃气费用、缓慢的交易速度和吞吐量感到沮丧，
以及链特定的治理只能进行缓慢更改
因为已部署的应用程序范围广泛。
需要一种解决方案来消除这些对于开发者的担忧，
他们在熟悉的 EVM 环境中构建应用程序。

`x/evm` 模块在可扩展的高吞吐量的权益证明区块链上提供了这种 EVM 熟悉性。
它是作为 [Cosmos SDK 模块](https://docs.cosmos.network/main/building-modules/intro.html)构建的，
允许部署智能合约，
与 EVM 状态机（状态转换）进行交互，
并使用 EVM 工具。
它可以在 Cosmos 应用特定的区块链上使用，
通过高交易吞吐量缓解了上述担忧
通过 [Tendermint Core](https://github.com/tendermint/tendermint) 快速交易最终性，
以及通过 [IBC](https://ibcprotocol.org/) 横向扩展性。

`x/evm` 模块是 [ethermint 库](https://pkg.go.dev/github.com/evmos/ethermint)的一部分。

## 目录

1. **[概念](#concepts)**
2. **[状态](#state)**
3. **[状态转换](#state-transitions)**
4. **[交易](#transactions)**
5. **[ABCI](#abci)**
6. **[钩子](#hooks)**
7. **[事件](#events)**
8. **[参数](#parameters)**
9. **[客户端](#client)**

## 模块架构

> **注意**: 如果您对SDK模块的整体结构不熟悉，请先阅读此[文档](https://docs.cosmos.network/main/building-modules/structure.html)。

```shell
evm/
├── client
│   └── cli
│       ├── query.go      # CLI query commands for the module
│       └── tx.go         # CLI transaction commands for the module
├── keeper
│   ├── keeper.go         # ABCI BeginBlock and EndBlock logic
│   ├── keeper.go         # Store keeper that handles the business logic of the module and has access to a specific subtree of the state tree.
│   ├── params.go         # Parameter getter and setter
│   ├── querier.go        # State query functions
│   └── statedb.go        # Functions from types/statedb with a passed in sdk.Context
├── types
│   ├── chain_config.go
│   ├── codec.go          # Type registration for encoding
│   ├── errors.go         # Module-specific errors
│   ├── events.go         # Events exposed to the Tendermint PubSub/Websocket
│   ├── genesis.go        # Genesis state for the module
│   ├── journal.go        # Ethereum Journal of state transitions
│   ├── keys.go           # Store keys and utility functions
│   ├── logs.go           # Types for persisting Ethereum tx logs on state after chain upgrades
│   ├── msg.go            # EVM module transaction messages
│   ├── params.go         # Module parameters that can be customized with governance parameter change proposals
│   ├── state_object.go   # EVM state object
│   ├── statedb.go        # Implementation of the StateDb interface
│   ├── storage.go        # Implementation of the Ethereum state storage map using arrays to prevent non-determinism
│   └── tx_data.go        # Ethereum transaction data types
├── genesis.go            # ABCI InitGenesis and ExportGenesis functionality
├── handler.go            # Message routing
└── module.go             # Module setup for the module manager
```

## 概念

### EVM

以太坊虚拟机（EVM）是一个计算引擎，可以将其视为由数千台连接的计算机（节点）维护的单个实体，这些计算机运行着以太坊客户端。
作为一个虚拟机（[VM](https://en.wikipedia.org/wiki/Virtual_machine)），EVM负责在不考虑其环境（硬件和操作系统）的情况下确定性地计算状态的变化。
这意味着每个节点在给定相同的起始状态和交易（tx）的情况下都必须获得完全相同的结果。

EVM被认为是以太坊协议的一部分，负责处理[智能合约](https://ethereum.org/en/developers/docs/smart-contracts/)的部署和执行。
为了明确区分：

* 以太坊协议描述了一个区块链，在该区块链中，所有以太坊账户和智能合约都存在。
  它在链中的任何给定块上只有一个规范状态（一个数据结构，保存所有账户）。
* 然而，EVM是[状态机](https://en.wikipedia.org/wiki/Finite-state_machine)，它定义了从一个块到另一个块计算新的有效状态的规则。
  它是一个隔离的运行时，这意味着在EVM内部运行的代码无法访问网络、文件系统或其他进程（不包括外部API）。

`x/evm`模块将EVM作为Cosmos SDK模块进行了实现。
它允许用户通过提交以太坊交易与EVM进行交互，并在给定状态上执行其中包含的消息以引发状态转换。

#### 状态

以太坊状态是一个数据结构，实现为一个[Merkle Patricia Tree](https://en.wikipedia.org/wiki/Merkle_tree)，用于保存链上的所有账户。
EVM对这个数据结构进行更改，从而产生一个具有不同状态根的新状态。
因此，以太坊可以被看作是一个状态链，通过在块中使用EVM执行交易来从一个状态过渡到另一个状态。
一个新的交易块可以通过其块头（父哈希、块编号、时间戳、随机数、收据等）来描述。

#### 账户

在给定地址上，可以存储两种类型的账户：

* **外部拥有账户（EOA）**：具有nonce（交易计数器）和余额
* **智能合约**：具有nonce、余额、（不可变的）代码哈希和存储根（另一个Merkle Patricia Trie）

智能合约就像区块链上的常规账户一样，
此外，它们还以以太坊特定的二进制格式（称为**EVM字节码**）存储可执行代码。
它们通常是用以太坊高级语言（如Solidity）编写的，
该语言会被编译为EVM字节码，
然后通过使用以太坊客户端提交交易来部署到区块链上。

#### 架构

EVM是基于堆栈的机器。
它的主要架构组件包括：

* 虚拟ROM：在处理交易时，将合约代码加载到这个只读内存中
* 机器状态（易失性）：随着EVM的运行而改变，并在处理每个交易后被清除
    * 程序计数器（PC）
    * Gas：跟踪使用了多少gas
    * 栈和内存：计算状态变化
* 访问账户存储（持久性）

#### 与智能合约的状态转换

通常，智能合约会公开一个公共ABI，
这是一个列出了用户可以与合约交互的支持方式的列表。
要与合约交互并触发状态转换，用户将提交一笔携带任意数量的gas和按照ABI格式化的数据有效负载的交易，
其中指定了交互类型和任何其他参数。
当接收到交易时，EVM使用交易有效负载执行智能合约的EVM字节码。

#### 执行EVM字节码

合约的EVM字节码由基本操作（加法、乘法、存储等）组成，称为**操作码**。
每个操作码的执行都需要用交易支付的gas。
因此，EVM被认为是准图灵完备的，
因为它允许任意计算，
但在合约执行期间的计算量受限于交易中提供的gas数量。
每个操作码的[**gas消耗**](https://www.evm.codes/)反映了在实际计算机硬件上运行这些操作的成本
（例如，`ADD = 3gas`和`SSTORE = 100gas`）。
要计算交易的gas消耗，需要将gas成本乘以**gas价格**，
该价格可能会根据网络需求的变化而变化。
如果网络负载过重，您可能需要支付更高的gas价格来执行您的交易。
如果达到了gas限制（gas用尽异常），则不会对以太坊状态进行任何更改，
除了发送者的nonce增加和其余额减少以支付浪费EVM时间的费用。

智能合约也可以调用其他智能合约。
每次调用新合约都会创建一个新的EVM实例（包括新的堆栈和内存）。
每次调用都将沙盒状态传递给下一个EVM。
如果燃气用尽，所有状态更改都将被丢弃。
否则，它们将被保留。

更多阅读，请参考：

* [EVM](https://eth.wiki/concepts/evm/evm)
* [EVM架构](https://cypherpunks-core.github.io/ethereumbook/13evm.html#evm_architecture)
* [以太坊是什么](https://ethdocs.org/en/latest/introduction/what-is-ethereum.html#what-is-ethereum)
* [操作码](https://www.ethervm.io/)

### Evmos作为Geth实现

Evmos包含了一个[Golang中的以太坊协议实现](https://geth.ethereum.org/docs/getting-started)
（Geth）作为Cosmos SDK模块。
Geth包括一个EVM的实现来计算状态转换。
可以查看[go-ethereum源代码](https://github.com/ethereum/go-ethereum/blob/master/core/vm/instructions.go)
以了解EVM操作码的实现方式。
就像Geth可以作为以太坊节点运行一样，
Evmos可以作为一个节点来使用EVM计算状态转换。
Evmos支持Geth的标准[Ethereum JSON-RPC API](https://docs.evmos.org/develop/api/ethereum-json-rpc/methods)
以实现Web3和EVM的兼容性。

#### JSON-RPC

JSON-RPC是一种无状态、轻量级的远程过程调用（RPC）协议。
主要规定了几种数据结构及其处理规则。
它在传输上是无关的，可以在同一进程内、通过套接字、通过HTTP或在许多不同的消息传递环境中使用这些概念。
它使用JSON（RFC 4627）作为数据格式。

##### JSON-RPC示例：`eth_call`

JSON-RPC方法[`eth_call`](https://docs.evmos.org/develop/api/ethereum-json-rpc/methods#eth-call)允许您执行针对合约的消息。
通常，您需要将交易发送到Geth节点以将其包含在内存池中，
然后节点之间进行传播，最终交易将被包含在一个区块中并执行。
然而，`eth_call`允许您向合约发送数据并查看发生的情况，而无需提交交易。

在Geth实现中，调用端点大致经过以下步骤：

1. `eth_call` 请求被转换为使用 `eth` 命名空间调用 `func (s *PublicBlockchainAPI) Call()` 函数。
2. [`Call()`](https://github.com/ethereum/go-ethereum/blob/master/internal/ethapi/api.go#L982) 接收交易参数、调用的区块以及可选参数来修改调用的状态。然后调用 `DoCall()`。
3. [`DoCall()`](https://github.com/ethereum/go-ethereum/blob/d575a2d3bc76dfbdefdd68b6cffff115542faf75/internal/ethapi/api.go#L891) 将参数转换为 `ethtypes.message`，实例化 EVM，并使用 `core.ApplyMessage` 应用消息。
4. [`ApplyMessage()`](https://github.com/ethereum/go-ethereum/blob/d575a2d3bc76dfbdefdd68b6cffff115542faf75/core/state_transition.go#L180) 调用状态转换的 `TransitionDb()`。
5. [`TransitionDb()`](https://github.com/ethereum/go-ethereum/blob/d575a2d3bc76dfbdefdd68b6cffff115542faf75/core/state_transition.go#L275) 要么 `Create()` 创建一个新的合约，要么 `Call()` 调用一个合约。
6. [`evm.Call()`](https://github.com/ethereum/go-ethereum/blob/d575a2d3bc76dfbdefdd68b6cffff115542faf75/core/vm/evm.go#L168) 运行解释器 `evm.interpreter.Run()` 执行消息。如果执行失败，状态将回滚到执行前的快照，并消耗燃料。
7. [`Run()`](https://github.com/ethereum/go-ethereum/blob/d575a2d3bc76dfbdefdd68b6cffff115542faf75/core/vm/interpreter.go#L116) 执行操作码的循环。

Evmos 实现类似，并利用了包含在 Cosmos SDK 中的 gRPC 查询客户端：

1. `eth_call` 请求被转换为使用 `eth` 命名空间调用 `func (e *PublicAPI) Call` 函数。
2. [`Call()`](https://github.com/evmos/ethermint/blob/main/rpc/namespaces/ethereum/eth/api.go#L639) 调用 `doCall()`。
3. [`doCall()`](https://github.com/evmos/ethermint/blob/main/rpc/namespaces/ethereum/eth/api.go#L656) 将参数转换为 `EthCallRequest`，并使用 evm 模块的查询客户端调用 `EthCall()`。
4. [`EthCall()`](https://github.com/evmos/ethermint/blob/main/x/evm/keeper/grpc_query.go#L212) 将参数转换为 `ethtypes.message`，并调用 `ApplyMessageWithConfig()`。
5. [`ApplyMessageWithConfig()`](https://github.com/evmos/ethermint/blob/d5598932a7f06158b7a5e3aa031bbc94eaaae32c/x/evm/keeper/state_transition.go#L341) 实例化 EVM，并使用 Geth 实现要么 `Create()` 创建一个新的合约，要么 `Call()` 调用一个合约。

#### StateDB

`StateDB`接口来自[go-ethereum](https://github.com/ethereum/go-ethereum/blob/master/core/vm/interface.go)，表示用于完整状态查询的EVM数据库。
通过这个接口，可以实现EVM状态转换，在`x/evm`模块中由`Keeper`实现。
这个接口的实现使得Evmos兼容EVM。

### 共识引擎

使用`x/evm`模块的应用程序通过应用区块链接口（ABCI）与Tendermint Core共识引擎进行交互。
应用程序和Tendermint Core共同构成了运行完整区块链的程序，将业务逻辑与分散式数据存储相结合。

在提交到`x/evm`模块的以太坊交易在执行并改变应用程序状态之前，会参与这个共识过程。
我们鼓励您了解[Tendermint共识引擎](https://docs.tendermint.com/main/introduction/what-is-tendermint.html#intro-to-abci)的基础知识，以便详细了解状态转换。

### 交易日志

在每个`x/evm`交易中，结果包含了状态机执行的以太坊`Log`，这些日志用于JSON-RPC Web3服务器进行过滤查询和处理EVM Hooks。

交易日志在交易执行期间存储在临时存储中，然后在交易处理完成后通过cosmos事件发出。
可以通过gRPC和JSON-RPC进行查询。

### 区块Bloom

Bloom是每个区块的布隆过滤器值，以字节为单位，可用于过滤查询。
区块Bloom值存储在临时存储中，然后在`EndBlock`处理期间通过cosmos事件发出。
可以通过gRPC和JSON-RPC进行查询。

:::tip
👉 **注意**：由于交易日志和区块Bloom不存储在状态中，因此在升级后不会持久化。
用户在升级后必须使用存档节点才能获取旧链事件。
:::

## 状态

本节概述了存储在`x/evm`模块状态中的对象，以及从go-ethereum的`StateDB`接口派生的功能，
以及通过Keeper实现的状态实现和创世状态实现。

### 状态对象

`x/evm` 模块在状态中保留以下对象：

#### 状态

|             | 描述                                                                                                                                     | 键                            | 值                   | 存储      |
|-------------|-----------------------------------------------------------------------------------------------------------------------------------------|-------------------------------|---------------------|-----------|
| Code        | 智能合约的字节码                                                                                                                         | `[]byte{1} + []byte(address)` | `[]byte{code}`      | KV        |
| Storage     | 智能合约的存储                                                                                                                           | `[]byte{2} + [32]byte{key}`   | `[32]byte(value)`   | KV        |
| Block Bloom | 区块布隆过滤器，用于累积当前区块的布隆过滤器，在结束块时发出到事件中。                                                                       | `[]byte{1} + []byte(tx.Hash)` | `protobuf([]Log)`   | Transient |
| Tx Index    | 当前交易在当前区块中的索引                                                                                                                 | `[]byte{2}`                   | `BigEndian(uint64)` | Transient |
| Log Size    | 当前区块中迄今为止发出的日志数量。用于确定后续日志的日志索引。                                                                             | `[]byte{3}`                   | `BigEndian(uint64)` | Transient |
| Gas Used    | 当前 cosmos-sdk 交易中以太坊消息使用的燃料量，在 cosmos-sdk 交易包含多个以太坊消息时是必要的。                                                   | `[]byte{4}`                   | `BigEndian(uint64)` | Transient |

### StateDB

`StateDB` 接口由 `x/evm/statedb` 模块中的 `StateDB` 实现，用于表示用于查询合约和账户的完整状态的 EVM 数据库。
在以太坊协议中，`StateDB` 用于存储 IAVL 树中的任何内容，并负责缓存和存储嵌套状态。

`x/evm`中的`StateDB`提供以下功能：

#### 以太坊账户的CRUD操作

您可以使用提供的地址创建`EthAccount`实例，并使用`createAccount()`将值设置到`AccountKeeper`中进行存储。
如果给定地址的账户已经存在，此函数还会重置与该地址关联的任何预先存在的代码和存储。

通过`BankKeeper`管理账户的货币余额，并可以使用`GetBalance()`进行读取，使用`AddBalance()`和`SubBalance()`进行更新。

- `GetBalance()`返回提供的地址的EVM货币余额。货币单位从模块参数中获取。
- `AddBalance()`将给定金额添加到地址余额中，通过铸造新币并将其转移到地址进行转账。货币单位从模块参数中获取。
- `SubBalance()`从地址余额中减去给定金额，通过将货币转移到托管账户并销毁它们。货币单位从模块参数中获取。
  如果金额为负数或用户没有足够的资金进行转账，则此函数不执行任何操作。

通过身份验证模块的`AccountKeeper`，可以获取账户的nonce（或交易序列）。

- `GetNonce()`检索具有给定地址的账户，并返回交易序列（即nonce）。
  如果找不到账户，则函数不执行任何操作。
- `SetNonce()`将给定的nonce设置为地址账户的序列。
  如果账户不存在，则将从地址创建一个新账户。

包含任意合约逻辑的智能合约字节码存储在`EVMKeeper`中，并可以使用`GetCodeHash()`、`GetCode()`和`GetCodeSize()`进行查询，使用`SetCode()`进行更新。

- `GetCodeHash()`从存储中获取账户并返回其代码哈希。
  如果账户不存在或不是EthAccount类型，则返回空的代码哈希值。
- `GetCode()`返回与给定地址关联的代码字节数组。
  如果账户的代码哈希为空，则此函数返回nil。
- `SetCode()`将代码字节数组存储到应用程序KVStore，并将代码哈希设置为给定账户。
  如果代码为空，则从存储中删除代码。
- `GetCodeSize()`返回与此对象关联的合约代码的大小，如果没有则返回零。

需要跟踪和存储退款的燃料，以便在EVM执行完成后将其加减到燃料使用值中。退款值在每个交易和每个区块结束时被清除。

- `AddRefund()` 将给定数量的燃料添加到内存中的退款值。
- `SubRefund()` 从内存中的退款值中减去给定数量的燃料。
  如果燃料数量大于当前退款值，此函数将引发错误。
- `GetRefund()` 返回交易执行完成后可用于退还的燃料数量。
  此值在每个交易中重置为0。

状态存储在 `EVMKeeper` 上。
可以使用 `GetCommittedState()`、`GetState()` 进行查询，并使用 `SetState()` 进行更新。

- `GetCommittedState()` 返回存储中为给定键哈希设置的值。
  如果键未注册，则此函数返回空哈希。
- `GetState()` 返回给定键哈希的内存中的脏状态，
  如果不存在，则从 KVStore 加载已提交的值。
- `SetState()` 将给定的哈希（键、值）设置为状态。
  如果值哈希为空，则此函数从状态中删除键，
  新值首先保留在脏状态中，并在最后提交到 KVStore 中。

账户也可以设置为自杀状态。
当合约自杀时，账户被标记为自杀状态，
在提交代码、存储和账户之后将其删除
（从下一个区块开始）。

- `Suicide()` 将给定账户标记为自杀状态，并清空 EVM 代币的账户余额。
- `HasSuicided()` 查询内存中的标志，检查账户是否在当前交易中被标记为自杀状态。
  被标记为自杀状态的账户在查询时将返回非空，并在区块提交后被“清除”。

要检查账户是否存在，请使用 `Exist()` 和 `Empty()`。

- `Exist()` 如果给定账户在存储中存在或已被标记为自杀状态，则返回 true。
- `Empty()` 如果地址满足以下条件，则返回 true：
    - nonce 为 0
    - evm 代币的余额为 0
    - 账户代码哈希为空

#### EIP2930功能

支持包含[访问列表](https://eips.ethereum.org/EIPS/eip-2930)的事务类型，该列表包含事务计划访问的地址和存储键。
访问列表状态保存在内存中，并在事务提交后丢弃。

- `PrepareAccessList()` 处理执行状态转换的准备步骤，涉及到 EIP-2929 和 EIP-2930。
  仅当当前编号适用于 Yolov3/Berlin/2929+2930 时才应调用此方法。
    - 将发送者添加到访问列表（EIP-2929）
    - 将目标地址添加到访问列表（EIP-2929）
    - 将预编译合约添加到访问列表（EIP-2929）
    - 将可选的事务访问列表内容添加到访问列表（EIP-2930）
- `AddressInAccessList()` 如果地址已注册，则返回 true。
- `SlotInAccessList()` 检查地址和槽是否已注册。
- `AddAddressToAccessList()` 将给定地址添加到访问列表。
  如果地址已在访问列表中，则此函数不执行任何操作。
- `AddSlotToAccessList()` 将给定的（地址，槽）添加到访问列表。
  如果地址和槽已在访问列表中，则此函数不执行任何操作。

#### 快照状态和还原功能

EVM 使用状态还原异常来处理错误。
这样的异常将撤销当前调用（及其所有子调用）对状态所做的所有更改，
并且调用者可以处理错误并且不会传播。
您可以使用 `Snapshot()` 来标识具有修订的当前状态，
并使用 `RevertToSnapshot()` 将状态还原到给定的修订版本以支持此功能。

- `Snapshot()` 创建一个新的快照并返回标识符。
- `RevertToSnapshot(rev)` 撤销到标识为 `rev` 的快照之前的所有修改。

Evmos 采用了[go-ethereum日志实现](https://github.com/ethereum/go-ethereum/blob/master/core/state/journal.go#L39)来支持此功能，
它使用日志列表记录到目前为止所做的所有状态修改操作，
快照由唯一的标识符和日志列表中的索引组成，
要还原到快照，只需按相反的顺序撤销快照索引之后的日志。

#### 以太坊交易日志

使用 `AddLog()` 函数，您可以将给定的以太坊 `Log` 追加到与当前状态中保存的交易哈希相关联的日志列表中。
在将日志存储之前，此函数还会填充 tx 哈希、块哈希、tx 索引和日志索引字段。

### Keeper

EVM 模块的 `Keeper` 授予对 EVM 模块状态的访问权限，并实现 `statedb.Keeper` 接口以支持 `StateDB` 的实现。
Keeper 包含一个存储键，允许 DB 写入仅由 EVM 模块访问的多存储的具体子树。
Evmos 不使用 trie 和数据库进行查询和持久化（`StateDB` 的实现），而是使用 Cosmos `KVStore`（键值存储）和 Cosmos SDK `Keeper` 来促进状态转换。

为了支持接口功能，它导入了 4 个模块 Keepers：

- `auth`：CRUD 账户
- `bank`：账户余额的会计（供应）和 CRUD
- `staking`：查询历史头部
- `fee market`：在 `ChainConfig` 参数上激活 `London` 硬分叉后，用于处理 `DynamicFeeTx` 的 EIP1559 基础费用

```go
type Keeper struct {
 // Protobuf codec
 cdc codec.BinaryCodec
 // Store key required for the EVM Prefix KVStore. It is required by:
 // - storing account's Storage State
 // - storing account's Code
 // - storing Bloom filters by block height. Needed for the Web3 API.
 // For the full list, check the module specification
 storeKey sdk.StoreKey

 // key to access the transient store, which is reset on every block during Commit
 transientKey sdk.StoreKey

 // module specific parameter space that can be configured through governance
 paramSpace paramtypes.Subspace
 // access to account state
 accountKeeper types.AccountKeeper
 // update balance and accounting operations with coins
 bankKeeper types.BankKeeper
 // access historical headers for EVM state transition execution
 stakingKeeper types.StakingKeeper
 // fetch EIP1559 base fee and parameters
 feeMarketKeeper types.FeeMarketKeeper

 // chain ID number obtained from the context's chain id
 eip155ChainID *big.Int

 // Tracer used to collect execution traces from the EVM transaction execution
 tracer string
 // trace EVM state transition execution. This value is obtained from the `--trace` flag.
 // For more info check https://geth.ethereum.org/docs/dapp/tracing
 debug bool

 // EVM Hooks for tx post-processing
 hooks types.EvmHooks
}
```

### 创世状态

`x/evm` 模块的 `GenesisState` 定义了从先前导出的高度初始化链所需的状态。
它包含了 `GenesisAccounts` 和模块参数。

```go
type GenesisState struct {
  // accounts is an array containing the ethereum genesis accounts.
  Accounts []GenesisAccount `protobuf:"bytes,1,rep,name=accounts,proto3" json:"accounts"`
  // params defines all the parameters of the module.
  Params Params `protobuf:"bytes,2,opt,name=params,proto3" json:"params"`
}
```

### 创世账户

`GenesisAccount` 类型对应于以太坊 `GenesisAccount` 类型的适应版本。
它定义了在创世状态中初始化的账户。

它的主要区别在于 Evmos 上的账户使用了自定义的 `Storage` 类型，
该类型使用切片而不是映射来存储 evm `State`（由于非确定性），
并且它不包含私钥字段。

还需要注意的是，由于 Cosmos SDK 上的 `auth` 模块管理账户状态，
`Address` 字段必须对应于存储在 `auth` 模块的 `Keeper`（即 `AccountKeeper`）中的现有 `EthAccount`。
地址在 `genesis.json` 上使用 **[EIP55](https://eips.ethereum.org/EIPS/eip-55)** 十六进制 **[格式](https://docs.evmos.org/protocol/concepts/accounts#address-formats-for-clients)**。

```go
type GenesisAccount struct {
  // address defines an ethereum hex formated address of an account
  Address string `protobuf:"bytes,1,opt,name=address,proto3" json:"address,omitempty"`
  // code defines the hex bytes of the account code.
  Code string `protobuf:"bytes,2,opt,name=code,proto3" json:"code,omitempty"`
  // storage defines the set of state key values for the account.
  Storage Storage `protobuf:"bytes,3,rep,name=storage,proto3,castrepeated=Storage" json:"storage"`
}
```

## 状态转换

`x/evm` 模块允许用户提交以太坊交易 (`Tx`) 并执行其中的消息，以在给定状态上引发状态转换。

用户在客户端上提交交易以将其广播到网络中。
当交易在共识期间包含在一个区块中时，它在服务器端执行。
我们强烈建议您了解 [Tendermint 共识引擎](https://docs.tendermint.com/main/introduction/what-is-tendermint.html#intro-to-abci) 的基本知识，以详细了解状态转换。

### 客户端

:::tip
👉 这是基于 `eth_sendTransaction` JSON-RPC 的内容
:::

1. 用户通过可用的 JSON-RPC 端点之一提交交易，
   使用兼容以太坊的客户端或钱包（例如 Metamask、WalletConnect、Ledger 等）：
   a. eth (公共) 命名空间：
    - `eth_sendTransaction`
    - `eth_sendRawTransaction`
      b. personal (私有) 命名空间：
    - `personal_sendTransaction`
2. 在填充 RPC 交易并使用 `SetTxDefaults` 填充缺失的 tx 参数的情况下，创建 `MsgEthereumTx` 的实例
3. 使用 `ValidateBasic()` 对 `Tx` 字段进行验证（无状态验证）
4. 使用与发送者地址关联的密钥和最新的以太坊硬分叉 (`London`、`Berlin` 等) 从 `ChainConfig` 中对 `Tx` 进行**签名**
5. 使用 Cosmos Config 构建器从消息字段中**构建** `Tx`
6. 使用 [同步模式](https://docs.cosmos.network/main/run-node/txs.html#broadcasting-a-transaction) **广播** `Tx`，
   以确保等待 [`CheckTx`](https://docs.tendermint.com/main/introduction/what-is-tendermint.html#intro-to-abci) 执行响应。
   交易通过使用 `CheckTx()` 进行应用程序验证，
   然后被添加到共识引擎的内存池中。
7. JSON-RPC 用户收到一个包含交易字段的 [`RLP`](https://eth.wiki/en/fundamentals/rlp) 哈希的响应。
   此哈希与 SDK 交易使用的默认哈希不同，
   后者计算交易字节的 `sha256` 哈希。

### 服务器端

一旦在共识过程中提交了一个包含 `Tx` 的区块，它将在应用程序中以一系列的 ABCI 消息的形式应用。

每个 `Tx` 都通过调用 [`RunTx`](https://docs.cosmos.network/main/core/baseapp.html#runtx) 来由应用程序处理。
在对 `Tx` 中的每个 `sdk.Msg` 进行无状态验证之后，`AnteHandler` 确认 `Tx` 是以太坊还是 SDK 交易。
作为以太坊交易，它包含的消息将由 `x/evm` 模块处理以更新应用程序的状态。

#### AnteHandler

`anteHandler` 会对每个交易运行。
它检查 `Tx` 是否为以太坊交易，并将其路由到内部的 ante 处理程序。
在这里，`Tx` 使用 EthereumTx 扩展选项进行处理，以使其与普通的 Cosmos SDK 交易有所不同。
`antehandler` 会对每个 `Tx` 运行一系列选项及其 `AnteHandle` 函数：

- `EthSetUpContextDecorator()` 是从 cosmos-sdk 的 SetUpContextDecorator 改编而来，
  它通过将 gas 计量器设置为无限来忽略 gas 消耗
- `EthValidateBasicDecorator(evmKeeper)` 验证以太坊类型的 Cosmos `Tx` 消息的字段
- `EthSigVerificationDecorator(evmKeeper)` 验证注册的链 ID 是否与消息上的链 ID 相同，
  并验证签名者地址是否与消息上定义的地址相匹配。
  它不会在 RecheckTx 中被跳过，因为它设置了其他 ante 处理程序所需的关键 `From` 地址。
  在 RecheckTx 失败时，将阻止将交易包含在区块中，特别是在 CheckTx 成功的情况下，
  此时用户将无法看到错误消息。
- `EthAccountVerificationDecorator(ak, bankKeeper, evmKeeper)`
  将验证发送者的余额是否大于总交易成本。
  如果账户不存在（即在存储中找不到），则将设置该账户。
  如果满足以下条件，此 AnteHandler 装饰器将失败：
    - 任何消息不是 MsgEthereumTx
    - 发送者地址为空
    - 账户余额低于交易成本
- `EthNonceVerificationDecorator(ak)` 验证交易的 nonce 是否有效，
  并且与发送者账户的当前 nonce 相等。
- `EthGasConsumeDecorator(evmKeeper)` 验证以太坊交易消息是否足够支付内在 gas（仅在 CheckTx 期间），
  并且发送者有足够的余额支付 gas 费用。
  交易的内在 gas 是在执行交易之前交易使用的 gas 量。
  gas 是一个常量值，加上与交易一起提供的额外数据字节所产生的任何费用。
  如果满足以下条件，此 AnteHandler 装饰器将失败：
    - 交易包含多个消息
    - 消息不是 MsgEthereumTx
    - 找不到发送者账户
    - 交易的 gas 限制低于内在 gas
    - 用户没有足够的余额来扣除交易费用（gas_limit * gas_price）
    - 交易或区块的 gas 计量器用尽
- `CanTransferDecorator(evmKeeper, feeMarketKeeper)` 创建一个 EVM 从消息中，
  并调用 BlockContext 的 CanTransfer 函数来查看地址是否可以执行该交易。
- `EthIncrementSenderSequenceDecorator(ak)` 处理递增签名者（即发送者）的序列。
  如果交易是合约创建，nonce 将在交易执行期间递增，
  而不是在此 AnteHandler 装饰器中递增。

选项 `authante.NewMempoolFeeDecorator()`、`authante.NewTxTimeoutHeightDecorator()` 和 `authante.NewValidateMemoDecorator(ak)` 与 Cosmos `Tx` 相同。点击[此处](https://docs.cosmos.network/main/basics/gas-fees.html#antehandler)了解有关 `anteHandler` 的更多信息。

#### EVM 模块

通过 `antehandler` 进行身份验证后，`Tx` 中的每个 `sdk.Msg`（在本例中为 `MsgEthereumTx`）都会传递到 `x/evm` 模块中的 Msg 处理程序，并按照以下步骤运行：

1. 将 `Msg` 转换为以太坊 `Tx` 类型
2. 使用 `EVMConfig` 应用 `Tx` 并尝试执行状态转换，只有在事务不失败时才会将其持久化（提交）到底层 KVStore 中：
    1. 确认已创建 `EVMConfig`
    2. 使用 `EVMConfig` 中的链配置值创建以太坊签名者
    3. 将以太坊事务哈希设置到（非永久性的）瞬态存储中，以便在 StateDB 函数中也可用
    4. 生成新的 EVM 实例
    5. 确认合约创建（`EnableCreate`）和合约执行（`EnableCall`）的 EVM 参数已启用
    6. 应用消息。如果 `To` 地址为 `nil`，则使用代码作为部署代码创建新合约。否则，使用给定输入作为参数调用给定地址的合约
    7. 计算 EVM 操作使用的 gas
3. 如果 `Tx` 成功应用
    1. 执行 EVM `Tx` 后处理钩子。如果钩子返回错误，则回滚整个 `Tx`
    2. 根据以太坊的 gas 计算规则退还 gas
    3. 使用从事务生成的日志更新块布隆过滤器值
    4. 为事务字段和事务日志发出 SDK 事件

## 交易

本节定义了导致在前一节中定义的状态转换的 `sdk.Msg` 具体类型。

## `MsgEthereumTx`

可以使用 `MsgEthereumTx` 来实现 EVM 状态转换。此消息将以太坊事务数据（`TxData`）封装为 `sdk.Msg`。它包含了必要的事务数据字段。请注意，`MsgEthereumTx` 同时实现了 [`sdk.Msg`](https://github.com/cosmos/cosmos-sdk/blob/v0.39.2/types/tx_msg.go#L7-L29) 和 [`sdk.Tx`](https://github.com/cosmos/cosmos-sdk/blob/v0.39.2/types/tx_msg.go#L33-L41) 接口。通常，SDK 消息只实现前者，而后者是一组打包在一起的消息。

```go
type MsgEthereumTx struct {
 // inner transaction data
 Data *types.Any `protobuf:"bytes,1,opt,name=data,proto3" json:"data,omitempty"`
 // DEPRECATED: encoded storage size of the transaction
 Size_ float64 `protobuf:"fixed64,2,opt,name=size,proto3" json:"-"`
 // transaction hash in hex format
 Hash string `protobuf:"bytes,3,opt,name=hash,proto3" json:"hash,omitempty" rlp:"-"`
 // ethereum signer address in hex format. This address value is checked
 // against the address derived from the signature (V, R, S) using the
 // secp256k1 elliptic curve
 From string `protobuf:"bytes,4,opt,name=from,proto3" json:"from,omitempty"`
}
```

如果满足以下条件，此消息字段验证将失败：

- 如果`From`字段已定义且地址无效
- 如果`TxData`的无状态验证失败

如果满足以下条件，事务执行将失败：

- 如果任何自定义的`AnteHandler`以太坊装饰器检查失败：
    - 事务的最小燃气量要求
    - 事务发送者账户不存在或余额不足以支付费用
    - 账户序列与事务的`Data.AccountNonce`不匹配
    - 消息签名验证失败
- EVM合约创建（即`evm.Create`）失败，或者`evm.Call`失败

#### 转换

`MsgEthreumTx`可以转换为go-ethereum的`Transaction`和`Message`类型，以便创建和调用evm合约。

```go
// AsTransaction creates an Ethereum Transaction type from the msg fields
func (msg MsgEthereumTx) AsTransaction() *ethtypes.Transaction {
 txData, err := UnpackTxData(msg.Data)
 if err != nil {
  return nil
 }

 return ethtypes.NewTx(txData.AsEthereumData())
}

// AsMessage returns the transaction as a core.Message.
func (tx *Transaction) AsMessage(s Signer, baseFee *big.Int) (Message, error) {
 msg := Message{
  nonce:      tx.Nonce(),
  gasLimit:   tx.Gas(),
  gasPrice:   new(big.Int).Set(tx.GasPrice()),
  gasFeeCap:  new(big.Int).Set(tx.GasFeeCap()),
  gasTipCap:  new(big.Int).Set(tx.GasTipCap()),
  to:         tx.To(),
  amount:     tx.Value(),
  data:       tx.Data(),
  accessList: tx.AccessList(),
  isFake:     false,
 }
 // If baseFee provided, set gasPrice to effectiveGasPrice.
 if baseFee != nil {
  msg.gasPrice = math.BigMin(msg.gasPrice.Add(msg.gasTipCap, baseFee), msg.gasFeeCap)
 }
 var err error
 msg.from, err = Sender(s, tx)
 return msg, err
}
```

#### 签名

为了使签名验证有效，`TxData`必须包含来自`Signer`的`v | r | s`值。Sign计算secp256k1 ECDSA签名并对事务进行签名。它接受一个密钥环签名者和链ID，根据EIP155标准对以太坊事务进行签名。此方法会改变事务，因为它填充了事务签名的V、R、S字段。如果消息的发送者地址未定义或发送者未在密钥环上注册，则此函数将失败。

```go
// Sign calculates a secp256k1 ECDSA signature and signs the transaction. It
// takes a keyring signer and the chainID to sign an Ethereum transaction according to
// EIP155 standard.
// This method mutates the transaction as it populates the V, R, S
// fields of the Transaction's Signature.
// The function will fail if the sender address is not defined for the msg or if
// the sender is not registered on the keyring
func (msg *MsgEthereumTx) Sign(ethSigner ethtypes.Signer, keyringSigner keyring.Signer) error {
 from := msg.GetFrom()
 if from.Empty() {
  return fmt.Errorf("sender address not defined for message")
 }

 tx := msg.AsTransaction()
 txHash := ethSigner.Hash(tx)

 sig, _, err := keyringSigner.SignByAddress(from, txHash.Bytes())
 if err != nil {
  return err
 }

 tx, err = tx.WithSignature(ethSigner, sig)
 if err != nil {
  return err
 }

 msg.FromEthereumTx(tx)
 return nil
}
```

### TxData

`MsgEthereumTx`支持来自go-ethereum的3种有效的以太坊事务数据类型：
`LegacyTx`、`AccessListTx`和`DynamicFeeTx`。这些类型被定义为protobuf消息，并打包到`MsgEthereumTx`字段的`proto.Any`接口类型中。

- `LegacyTx`：[EIP-155](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-155.md)事务类型
- `DynamicFeeTx`：[EIP-1559](https://eips.ethereum.org/EIPS/eip-1559)事务类型。在伦敦硬分叉块启用
- `AccessListTx`：[EIP-2930](https://eips.ethereum.org/EIPS/eip-2930)事务类型。在柏林硬分叉块启用

### `LegacyTx`

常规以太坊事务的事务数据。

```go
type LegacyTx struct {
 // nonce corresponds to the account nonce (transaction sequence).
 Nonce uint64 `protobuf:"varint,1,opt,name=nonce,proto3" json:"nonce,omitempty"`
 // gas price defines the value for each gas unit
 GasPrice *github_com_cosmos_cosmos_sdk_types.Int `protobuf:"bytes,2,opt,name=gas_price,json=gasPrice,proto3,customtype=github.com/cosmos/cosmos-sdk/types.Int" json:"gas_price,omitempty"`
 // gas defines the gas limit defined for the transaction.
 GasLimit uint64 `protobuf:"varint,3,opt,name=gas,proto3" json:"gas,omitempty"`
 // hex formatted address of the recipient
 To string `protobuf:"bytes,4,opt,name=to,proto3" json:"to,omitempty"`
 // value defines the unsigned integer value of the transaction amount.
 Amount *github_com_cosmos_cosmos_sdk_types.Int `protobuf:"bytes,5,opt,name=value,proto3,customtype=github.com/cosmos/cosmos-sdk/types.Int" json:"value,omitempty"`
 // input defines the data payload bytes of the transaction.
 Data []byte `protobuf:"bytes,6,opt,name=data,proto3" json:"data,omitempty"`
 // v defines the signature value
 V []byte `protobuf:"bytes,7,opt,name=v,proto3" json:"v,omitempty"`
 // r defines the signature value
 R []byte `protobuf:"bytes,8,opt,name=r,proto3" json:"r,omitempty"`
 // s define the signature value
 S []byte `protobuf:"bytes,9,opt,name=s,proto3" json:"s,omitempty"`
}
```

这个消息字段验证预计会失败，如果：

- `GasPrice` 无效（`nil`，负数或超出 int256 范围）
- `Fee`（gasprice * gaslimit）无效
- `Amount` 无效（负数或超出 int256 范围）
- `To` 地址无效（非有效的以太坊十六进制地址）

### `DynamicFeeTx`

EIP-1559 动态费用交易的交易数据。

```go
type DynamicFeeTx struct {
 // destination EVM chain ID
 ChainID *github_com_cosmos_cosmos_sdk_types.Int `protobuf:"bytes,1,opt,name=chain_id,json=chainId,proto3,customtype=github.com/cosmos/cosmos-sdk/types.Int" json:"chainID"`
 // nonce corresponds to the account nonce (transaction sequence).
 Nonce uint64 `protobuf:"varint,2,opt,name=nonce,proto3" json:"nonce,omitempty"`
 // gas tip cap defines the max value for the gas tip
 GasTipCap *github_com_cosmos_cosmos_sdk_types.Int `protobuf:"bytes,3,opt,name=gas_tip_cap,json=gasTipCap,proto3,customtype=github.com/cosmos/cosmos-sdk/types.Int" json:"gas_tip_cap,omitempty"`
 // gas fee cap defines the max value for the gas fee
 GasFeeCap *github_com_cosmos_cosmos_sdk_types.Int `protobuf:"bytes,4,opt,name=gas_fee_cap,json=gasFeeCap,proto3,customtype=github.com/cosmos/cosmos-sdk/types.Int" json:"gas_fee_cap,omitempty"`
 // gas defines the gas limit defined for the transaction.
 GasLimit uint64 `protobuf:"varint,5,opt,name=gas,proto3" json:"gas,omitempty"`
 // hex formatted address of the recipient
 To string `protobuf:"bytes,6,opt,name=to,proto3" json:"to,omitempty"`
 // value defines the the transaction amount.
 Amount *github_com_cosmos_cosmos_sdk_types.Int `protobuf:"bytes,7,opt,name=value,proto3,customtype=github.com/cosmos/cosmos-sdk/types.Int" json:"value,omitempty"`
 // input defines the data payload bytes of the transaction.
 Data     []byte     `protobuf:"bytes,8,opt,name=data,proto3" json:"data,omitempty"`
 Accesses AccessList `protobuf:"bytes,9,rep,name=accesses,proto3,castrepeated=AccessList" json:"accessList"`
 // v defines the signature value
 V []byte `protobuf:"bytes,10,opt,name=v,proto3" json:"v,omitempty"`
 // r defines the signature value
 R []byte `protobuf:"bytes,11,opt,name=r,proto3" json:"r,omitempty"`
 // s define the signature value
 S []byte `protobuf:"bytes,12,opt,name=s,proto3" json:"s,omitempty"`
}
```

这个消息字段验证预计会失败，如果：

- `GasTipCap` 无效（`nil`，负数或溢出 int256）
- `GasFeeCap` 无效（`nil`，负数或溢出 int256）
- `GasFeeCap` 小于 `GasTipCap`
- `Fee`（gas price * gas limit）无效（溢出 int256）
- `Amount` 无效（负数或溢出 int256）
- `To` 地址无效（非有效的以太坊十六进制地址）
- `ChainID` 是 `nil`

### `AccessListTx`

EIP-2930 访问列表交易的交易数据。

```go
type AccessListTx struct {
 // destination EVM chain ID
 ChainID *github_com_cosmos_cosmos_sdk_types.Int `protobuf:"bytes,1,opt,name=chain_id,json=chainId,proto3,customtype=github.com/cosmos/cosmos-sdk/types.Int" json:"chainID"`
 // nonce corresponds to the account nonce (transaction sequence).
 Nonce uint64 `protobuf:"varint,2,opt,name=nonce,proto3" json:"nonce,omitempty"`
 // gas price defines the value for each gas unit
 GasPrice *github_com_cosmos_cosmos_sdk_types.Int `protobuf:"bytes,3,opt,name=gas_price,json=gasPrice,proto3,customtype=github.com/cosmos/cosmos-sdk/types.Int" json:"gas_price,omitempty"`
 // gas defines the gas limit defined for the transaction.
 GasLimit uint64 `protobuf:"varint,4,opt,name=gas,proto3" json:"gas,omitempty"`
 // hex formatted address of the recipient
 To string `protobuf:"bytes,5,opt,name=to,proto3" json:"to,omitempty"`
 // value defines the unsigned integer value of the transaction amount.
 Amount *github_com_cosmos_cosmos_sdk_types.Int `protobuf:"bytes,6,opt,name=value,proto3,customtype=github.com/cosmos/cosmos-sdk/types.Int" json:"value,omitempty"`
 // input defines the data payload bytes of the transaction.
 Data     []byte     `protobuf:"bytes,7,opt,name=data,proto3" json:"data,omitempty"`
 Accesses AccessList `protobuf:"bytes,8,rep,name=accesses,proto3,castrepeated=AccessList" json:"accessList"`
 // v defines the signature value
 V []byte `protobuf:"bytes,9,opt,name=v,proto3" json:"v,omitempty"`
 // r defines the signature value
 R []byte `protobuf:"bytes,10,opt,name=r,proto3" json:"r,omitempty"`
 // s define the signature value
 S []byte `protobuf:"bytes,11,opt,name=s,proto3" json:"s,omitempty"`
}
```

这个消息字段验证预计会失败，如果：

- `GasPrice` 无效（`nil`，负数或溢出 int256）
- `Fee`（gas price * gas limit）无效（溢出 int256）
- `Amount` 无效（负数或溢出 int256）
- `To` 地址无效（非有效的以太坊十六进制地址）
- `ChainID` 是 `nil`

## ABCI

应用区块链接口（ABCI）允许应用与 Tendermint 共识引擎进行交互。
应用与 Tendermint 维护多个 ABCI 连接。
对于 `x/evm` 来说，最相关的是[提交时的共识连接](https://docs.tendermint.com/v0.33/app-dev/app-development.html#consensus-connection)。
该连接负责块执行，并调用函数 `InitChain`（包含 `InitGenesis`）、`BeginBlock`、`DeliverTx`、`EndBlock`、`Commit`。
`InitChain` 仅在启动新区块链时首次调用，
而 `DeliverTx` 则对块中的每个交易调用。

### InitGenesis

`InitGenesis` 通过将 `GenesisState` 字段设置到存储中来初始化 EVM 模块的创世状态。
特别是，它设置了参数和创世账户（状态和代码）。

### ExportGenesis

`ExportGenesis` ABCI函数用于导出EVM模块的创世状态。
具体来说，它检索所有帐户及其字节码、余额和存储、事务日志以及EVM参数和链配置。

### BeginBlock

在处理事务的状态转换之前，执行EVM模块的`BeginBlock`逻辑。
该函数的主要目标是：

- 为当前块设置上下文，以便`Keeper`在EVM状态转换期间调用`StateDB`函数时可以访问块头、存储、燃气计量等。
- 设置EIP155的`ChainID`编号（从完整的链ID获取），以防在`InitChain`期间尚未设置。

### EndBlock

在执行所有事务的状态转换之后，执行EVM模块的`EndBlock`逻辑。
该函数的主要目标是：

- 发出块布隆事件
    - 这是为了与web3兼容性，因为以太坊头部包含此类型作为字段。
      JSON-RPC服务使用此事件查询从Tendermint头构造以太坊头。
    - 块布隆过滤器值从临时存储获取，然后发出

## Hooks

`x/evm`模块实现了一个扩展和自定义`Tx`处理逻辑的`EvmHooks`接口。

这支持EVM合约通过以下方式调用本机cosmos模块：

1. 定义一个日志签名并从智能合约中发出特定的日志，
2. 在本机tx处理代码中识别这些日志，并
3. 将它们转换为本机模块调用。

为此，该接口包括一个`PostTxProcessing`钩子，用于在`EvmKeeper`中注册自定义的`Tx`钩子。
这些`Tx`钩子在EVM状态转换完成且不失败后进行处理。
请注意，EVM模块中没有实现默认的钩子。

```go
type EvmHooks interface {
 // Must be called after tx is processed successfully, if return an error, the whole transaction is reverted.
 PostTxProcessing(ctx sdk.Context, msg core.Message, receipt *ethtypes.Receipt) error
}
```

## `PostTxProcessing`

`PostTxProcessing`仅在EVM事务成功完成后调用，并将调用委托给底层钩子。
如果没有注册钩子，该函数将返回`nil`错误。

```go
func (k *Keeper) PostTxProcessing(ctx sdk.Context, msg core.Message, receipt *ethtypes.Receipt) error {
 if k.hooks == nil {
  return nil
 }
 return k.hooks.PostTxProcessing(k.Ctx(), msg, receipt)
}
```

它在与EVM事务相同的缓存上下文中执行，
如果返回错误，则整个EVM事务将被回滚，
如果钩子实现者不想回滚事务，他们可以始终返回`nil`。

钩子返回的错误被转换为VM错误`failed to process native logs`，
详细的错误消息存储在返回值中。
该消息以异步方式发送给本机模块，调用者无法捕获和恢复错误。

### 使用案例：在Evmos上调用本机ERC20模块

下面是从Evmos [erc20模块](erc20.md)中摘录的一个示例，
展示了`EVMHooks`如何支持合约调用本机模块
将ERC-20代币转换为Cosmos本机币。
按照上述步骤进行操作。

您可以在智能合约中定义和发出`Transfer`日志签名，如下所示：

```solidity
event Transfer(address indexed from, address indexed to, uint256 value);

function _transfer(address sender, address recipient, uint256 amount) internal virtual {
  require(sender != address(0), "ERC20: transfer from the zero address");
  require(recipient != address(0), "ERC20: transfer to the zero address");

  _beforeTokenTransfer(sender, recipient, amount);

  _balances[sender] = _balances[sender].sub(amount, "ERC20: transfer amount exceeds balance");
  _balances[recipient] = _balances[recipient].add(amount);
  emit Transfer(sender, recipient, amount);
}
```

应用程序将在`EvmKeeper`中注册一个`BankSendHook`。
它识别以太坊tx `Log`并将其转换为对银行模块的`SendCoinsFromAccountToAccount`方法的调用：

```go

const ERC20EventTransfer = "Transfer"

// PostTxProcessing implements EvmHooks.PostTxProcessing
func (k Keeper) PostTxProcessing(
 ctx sdk.Context,
 msg core.Message,
 receipt *ethtypes.Receipt,
) error {
 params := h.k.GetParams(ctx)
 if !params.EnableErc20 || !params.EnableEVMHook {
  // no error is returned to allow for other post-processing txs
  // to pass
  return nil
 }

 erc20 := contracts.ERC20BurnableContract.ABI

 for i, log := range receipt.Logs {
  if len(log.Topics) < 3 {
   continue
  }

  eventID := log.Topics[0] // event ID

  event, err := erc20.EventByID(eventID)
  if err != nil {
   // invalid event for ERC20
   continue
  }

  if event.Name != types.ERC20EventTransfer {
   h.k.Logger(ctx).Info("emitted event", "name", event.Name, "signature", event.Sig)
   continue
  }

  transferEvent, err := erc20.Unpack(event.Name, log.Data)
  if err != nil {
   h.k.Logger(ctx).Error("failed to unpack transfer event", "error", err.Error())
   continue
  }

  if len(transferEvent) == 0 {
   continue
  }

  tokens, ok := transferEvent[0].(*big.Int)
  // safety check and ignore if amount not positive
  if !ok || tokens == nil || tokens.Sign() != 1 {
   continue
  }

  // check that the contract is a registered token pair
  contractAddr := log.Address

  id := h.k.GetERC20Map(ctx, contractAddr)

  if len(id) == 0 {
   // no token is registered for the caller contract
   continue
  }

  pair, found := h.k.GetTokenPair(ctx, id)
  if !found {
   continue
  }

  // check that conversion for the pair is enabled
  if !pair.Enabled {
   // continue to allow transfers for the ERC20 in case the token pair is disabled
   h.k.Logger(ctx).Debug(
    "ERC20 token -> Cosmos coin conversion is disabled for pair",
    "coin", pair.Denom, "contract", pair.Erc20Address,
   )
   continue
  }

  // ignore as the burning always transfers to the zero address
  to := common.BytesToAddress(log.Topics[2].Bytes())
  if !bytes.Equal(to.Bytes(), types.ModuleAddress.Bytes()) {
   continue
  }

  // check that the event is Burn from the ERC20Burnable interface
  // NOTE: assume that if they are burning the token that has been registered as a pair, they want to mint a Cosmos coin

  // create the corresponding sdk.Coin that is paired with ERC20
  coins := sdk.Coins{{Denom: pair.Denom, Amount: sdk.NewIntFromBigInt(tokens)}}

  // Mint the coin only if ERC20 is external
  switch pair.ContractOwner {
  case types.OWNER_MODULE:
   _, err = h.k.CallEVM(ctx, erc20, types.ModuleAddress, contractAddr, true, "burn", tokens)
  case types.OWNER_EXTERNAL:
   err = h.k.bankKeeper.MintCoins(ctx, types.ModuleName, coins)
  default:
   err = types.ErrUndefinedOwner
  }

  if err != nil {
   h.k.Logger(ctx).Debug(
    "failed to process EVM hook for ER20 -> coin conversion",
    "coin", pair.Denom, "contract", pair.Erc20Address, "error", err.Error(),
   )
   continue
  }

  // Only need last 20 bytes from log.topics
  from := common.BytesToAddress(log.Topics[1].Bytes())
  recipient := sdk.AccAddress(from.Bytes())

  // transfer the tokens from ModuleAccount to sender address
  if err := h.k.bankKeeper.SendCoinsFromModuleToAccount(ctx, types.ModuleName, recipient, coins); err != nil {
   h.k.Logger(ctx).Debug(
    "failed to process EVM hook for ER20 -> coin conversion",
    "tx-hash", receipt.TxHash.Hex(), "log-idx", i,
    "coin", pair.Denom, "contract", pair.Erc20Address, "error", err.Error(),
   )
   continue
  }
 }

 return nil
```

最后，在`app.go`中注册钩子：

```go
app.EvmKeeper = app.EvmKeeper.SetHooks(app.Erc20Keeper)
```

## 事件

`x/evm`模块在状态执行后发出Cosmos SDK事件。
EVM模块发出与相关事务字段以及事务日志（以太坊事件）相关的事件。

### MsgEthereumTx

| 类型          | 属性键             | 属性值                  |
| ------------- | ----------------- | ----------------------- |
| ethereum_tx   | `"amount"`        | `{amount}`              |
| ethereum_tx   | `"recipient"`     | `{hex_address}`         |
| ethereum_tx   | `"contract"`      | `{hex_address}`         |
| ethereum_tx   | `"txHash"`        | `{tendermint_hex_hash}` |
| ethereum_tx   | `"ethereumTxHash"`| `{hex_hash}`            |
| ethereum_tx   | `"txIndex"`       | `{tx_index}`            |
| ethereum_tx   | `"txGasUsed"`     | `{gas_used}`            |
| tx_log        | `"txLog"`         | `{tx_log}`              |
| message       | `"sender"`        | `{eth_address}`         |
| message       | `"action"`        | `"ethereum"`            |
| message       | `"module"`        | `"evm"`                 |

此外，EVM 模块在 `EndBlock` 期间为过滤器查询块布隆发出事件。

### ABCI

| 类型        | 属性键       | 属性值              |
| ----------- | ------------- | -------------------- |
| block_bloom | `"bloom"`     | `string(bloomBytes)` |

## 参数

evm 模块包含以下参数：

### Params

| 键              | 类型        | 默认值         |
|----------------|-------------|-----------------|
| `EVMDenom`     | string      | `"aevmos"`      |
| `EnableCreate` | bool        | `true`          |
| `EnableCall`   | bool        | `true`          |
| `ExtraEIPs`    | []int       | TBD             |
| `ChainConfig`  | ChainConfig | See ChainConfig |

### EVM 币种

evm 币种参数定义了在 EVM 状态转换和 EVM 消息的 gas 消耗中使用的代币币种。

例如，在以太坊上，`evm_denom` 将是 `ETH`。
在 Evmos 的情况下，默认币种是 **atto evmos**。
在精度方面，`EVMOS` 和 `ETH` 具有相同的值，
即 `1 EVMOS = 10^18 atto evmos` 和 `1 ETH = 10^18 wei`。

:::tip
注意：希望将 EVM 模块作为依赖项导入的 SDK 应用程序
需要设置自己的 `evm_denom`（而不是 `"aevmos"`）。
:::

### 启用创建

启用创建参数切换使用 `vm.Create` 函数的状态转换。
当禁用该参数时，将阻止所有合约创建功能。

### 启用转账

启用转账参数切换使用 `vm.Call` 函数的状态转换。
当禁用该参数时，将阻止账户之间的转账和执行智能合约调用。

### 额外的 EIPs

额外的 EIPs 参数定义了可激活的以太坊改进提案（**[EIPs](https://ethereum.org/en/eips/)**）
在应用自定义跳转表的以太坊 VM `Config` 上。

:::tip
注意：这些 EIPs 中的一些已经根据硬分叉编号在链配置中启用。
:::

支持的可激活 EIPs 有：

- **[EIP 1344](https://eips.ethereum.org/EIPS/eip-1344)**
- **[EIP 1884](https://eips.ethereum.org/EIPS/eip-1884)**
- **[EIP 2200](https://eips.ethereum.org/EIPS/eip-2200)**
- **[EIP 2315](https://eips.ethereum.org/EIPS/eip-2315)**
- **[EIP 2929](https://eips.ethereum.org/EIPS/eip-2929)**
- **[EIP 3198](https://eips.ethereum.org/EIPS/eip-3198)**
- **[EIP 3529](https://eips.ethereum.org/EIPS/eip-3529)**

### 链配置

`ChainConfig` 是一个包装了 protobuf 类型的结构体，
它包含了与 go-ethereum 的 `ChainConfig` 参数相同的字段，
但是使用 `*sdk.Int` 类型而不是 `*big.Int`。

默认情况下，除了 `ConstantinopleBlock` 字段外，所有的块配置字段在创世块（高度为0）时都是启用的。

#### ChainConfig 默认值

| 名称                | 默认值                                                               |
| ------------------- | -------------------------------------------------------------------- |
| HomesteadBlock      | 0                                                                    |
| DAOForkBlock        | 0                                                                    |
| DAOForkSupport      | `true`                                                               |
| EIP150Block         | 0                                                                    |
| EIP150Hash          | `0x0000000000000000000000000000000000000000000000000000000000000000` |
| EIP155Block         | 0                                                                    |
| EIP158Block         | 0                                                                    |
| ByzantiumBlock      | 0                                                                    |
| ConstantinopleBlock | 0                                                                    |
| PetersburgBlock     | 0                                                                    |
| IstanbulBlock       | 0                                                                    |
| MuirGlacierBlock    | 0                                                                    |
| BerlinBlock         | 0                                                                    |
| LondonBlock         | 0                                                                    |
| ArrowGlacierBlock   | 0                                                                    |
| GrayGlacierBlock    | 0                                                                    |
| MergeNetsplitBlock  | 0                                                                    |
| ShanghaiBlock       | 0                                                                    |
| CancunBlock.        | 0                                                                    |

## 客户端

用户可以使用CLI、JSON-RPC、gRPC或REST与`evm`模块进行查询和交互。

### CLI

以下是使用`x/evm`模块添加的`evmosd`命令列表。
您可以使用`evmosd -h`命令获取完整列表。

#### 查询

`query`命令允许用户查询`evm`状态。

**`code`**

允许用户查询给定地址的智能合约代码。

```bash
evmosd query evm code ADDRESS [flags]
```

```bash
# Example
$ evmosd query evm code 0x7bf7b17da59880d9bcca24915679668db75f9397

# Output
code: "0xef616c92f3cfc9e92dc270d6acff9cea213cecc7020a76ee4395af09bdceb4837a1ebdb5735e11e7d3adb6104e0c3ac55180b4ddf5e54d022cc5e8837f6a4f971b"
```

**`storage`**

允许用户查询具有给定键和高度的帐户的存储。

```bash
evmosd query evm storage ADDRESS KEY [flags]
```

```bash
# Example
$ evmosd query evm storage 0x0f54f47bf9b8e317b214ccd6a7c3e38b893cd7f0 0 --height 0

# Output
value: "0x0000000000000000000000000000000000000000000000000000000000000000"
```

#### 交易

`tx`命令允许用户与`evm`模块进行交互。

**`raw`**

允许用户从原始以太坊交易构建cosmos交易。

```bash
evmosd tx evm raw TX_HEX [flags]
```

```bash
# Example
$ evmosd tx evm raw 0xf9ff74c86aefeb5f6019d77280bbb44fb695b4d45cfe97e6eed7acd62905f4a85034d5c68ed25a2e7a8eeb9baf1b84

# Output
value: "0x0000000000000000000000000000000000000000000000000000000000000000"
```

### JSON-RPC

有关Evmos支持的JSON-RPC方法和命名空间的概述，请参阅[https://docs.evmos.org/develop/api/ethereum-json-rpc/methodsl](https://docs.evmos.org/develop/api/ethereum-json-rpc/methods)

### gRPC

#### 查询

| 动词   | 方法                                               | 描述                                                               |
| ------ | ---------------------------------------------------- | ------------------------------------------------------------------------- |
| `gRPC` | `ethermint.evm.v1.Query/Account`                     | 获取以太坊账户                                                   |
| `gRPC` | `ethermint.evm.v1.Query/CosmosAccount`               | 获取以太坊账户的Cosmos地址                                  |
| `gRPC` | `ethermint.evm.v1.Query/ValidatorAccount`            | 获取以太坊账户的验证者共识地址              |
| `gRPC` | `ethermint.evm.v1.Query/Balance`                     | 获取单个EthAccount的EVM货币余额        |
| `gRPC` | `ethermint.evm.v1.Query/Storage`                     | 获取单个帐户的所有货币余额                         |
| `gRPC` | `ethermint.evm.v1.Query/Code`                        | 获取单个帐户的所有货币余额                         |
| `gRPC` | `ethermint.evm.v1.Query/Params`                      | 获取x/evm模块的参数                                        |
| `gRPC` | `ethermint.evm.v1.Query/EthCall`                     | 实现eth_call rpc api                                           |
| `gRPC` | `ethermint.evm.v1.Query/EstimateGas`                 | 实现eth_estimateGas rpc api                                    |
| `gRPC` | `ethermint.evm.v1.Query/TraceTx`                     | 实现debug_traceTransaction rpc api                             |
| `gRPC` | `ethermint.evm.v1.Query/TraceBlock`                  | 实现debug_traceBlockByNumber和debug_traceBlockByHash rpc api |
| `GET`  | `/ethermint/evm/v1/account/{address}`                | 获取以太坊账户                                                   |
| `GET`  | `/ethermint/evm/v1/cosmos_account/{address}`         | 获取以太坊账户的Cosmos地址                                  |
| `GET`  | `/ethermint/evm/v1/validator_account/{cons_address}` | 获取以太坊账户的验证者共识地址              |
| `GET`  | `/ethermint/evm/v1/balances/{address}`               | 获取单个EthAccount的EVM货币余额        |
| `GET`  | `/ethermint/evm/v1/storage/{address}/{key}`          | 获取单个帐户的所有货币余额                         |
| `GET`  | `/ethermint/evm/v1/codes/{address}`                  | 获取单个帐户的所有货币余额                         |
| `GET`  | `/ethermint/evm/v1/params`                           | 获取x/evm模块的参数                                        |
| `GET`  | `/ethermint/evm/v1/eth_call`                         | 实现eth_call rpc api                                           |
| `GET`  | `/ethermint/evm/v1/estimate_gas`                     | 实现eth_estimateGas rpc api                                    |
| `GET`  | `/ethermint/evm/v1/trace_tx`                         | 实现debug_traceTransaction rpc api                             |
| `GET`  | `/ethermint/evm/v1/trace_block`                      | 实现debug_traceBlockByNumber和debug_traceBlockByHash rpc api |

#### 交易

| 动词   | 方法                              | 描述                           |
| ------ | --------------------------------- | ------------------------------- |
| `gRPC` | `ethermint.evm.v1.Msg/EthereumTx` | 提交以太坊交易                 |
| `POST` | `/ethermint/evm/v1/ethereum_tx`   | 提交以太坊交易                 |


# `evm`

## Abstract

This document defines the specification of the Ethereum Virtual Machine (EVM) as a Cosmos SDK module.

Since the introduction of Ethereum in 2015,
the ability to control digital assets through [**smart contracts**](https://www.fon.hum.uva.nl/rob/Courses/InformationInSpeech/CDROM/Literature/LOTwinterschool2006/szabo.best.vwh.net/idea.html)
has attracted a large community of developers
to build decentralized applications on the Ethereum Virtual Machine (EVM).
This community is continuously creating extensive tooling and introducing standards,
which are further increasing the adoption rate of EVM compatible technology.

The growth of EVM-based chains (e.g.
Ethereum), however, has uncovered several scalability challenges
that are often referred to as the [trilemma of decentralization, security, and scalability](https://vitalik.ca/general/2021/04/07/sharding.html).
Developers are frustrated by high gas fees, slow transaction speed & throughput,
and chain-specific governance that can only undergo slow change
because of its wide range of deployed applications.
A solution is required that eliminates these concerns for developers,
who build applications within a familiar EVM environment.

The `x/evm` module provides this EVM familiarity on a scalable, high-throughput Proof-of-Stake blockchain.
It is built as a [Cosmos SDK module](https://docs.cosmos.network/main/building-modules/intro.html)
which allows for the deployment of smart contracts,
interaction with the EVM state machine (state transitions),
and the use of EVM tooling.
It can be used on Cosmos application-specific blockchains,
which alleviate the aforementioned concerns through high transaction throughput
via [Tendermint Core](https://github.com/tendermint/tendermint), fast transaction finality,
and horizontal scalability via [IBC](https://ibcprotocol.org/).

The `x/evm` module is part of the [ethermint library](https://pkg.go.dev/github.com/evmos/ethermint).

## Contents

1. **[Concepts](#concepts)**
2. **[State](#state)**
3. **[State Transitions](#state-transitions)**
4. **[Transactions](#transactions)**
5. **[ABCI](#abci)**
6. **[Hooks](#hooks)**
7. **[Events](#events)**
8. **[Parameters](#parameters)**
9. **[Client](#client)**

## Module Architecture

> **NOTE:**: If you're not familiar with the overall module structure from
the SDK modules, please check this [document](https://docs.cosmos.network/main/building-modules/structure.html) as
prerequisite reading.

```shell
evm/
├── client
│   └── cli
│       ├── query.go      # CLI query commands for the module
│       └── tx.go         # CLI transaction commands for the module
├── keeper
│   ├── keeper.go         # ABCI BeginBlock and EndBlock logic
│   ├── keeper.go         # Store keeper that handles the business logic of the module and has access to a specific subtree of the state tree.
│   ├── params.go         # Parameter getter and setter
│   ├── querier.go        # State query functions
│   └── statedb.go        # Functions from types/statedb with a passed in sdk.Context
├── types
│   ├── chain_config.go
│   ├── codec.go          # Type registration for encoding
│   ├── errors.go         # Module-specific errors
│   ├── events.go         # Events exposed to the Tendermint PubSub/Websocket
│   ├── genesis.go        # Genesis state for the module
│   ├── journal.go        # Ethereum Journal of state transitions
│   ├── keys.go           # Store keys and utility functions
│   ├── logs.go           # Types for persisting Ethereum tx logs on state after chain upgrades
│   ├── msg.go            # EVM module transaction messages
│   ├── params.go         # Module parameters that can be customized with governance parameter change proposals
│   ├── state_object.go   # EVM state object
│   ├── statedb.go        # Implementation of the StateDb interface
│   ├── storage.go        # Implementation of the Ethereum state storage map using arrays to prevent non-determinism
│   └── tx_data.go        # Ethereum transaction data types
├── genesis.go            # ABCI InitGenesis and ExportGenesis functionality
├── handler.go            # Message routing
└── module.go             # Module setup for the module manager
```

## Concepts

### EVM

The Ethereum Virtual Machine (EVM) is a computation engine
which can be thought of as one single entity maintained by thousands of connected computers (nodes)
running an Ethereum client.
As a virtual machine ([VM](https://en.wikipedia.org/wiki/Virtual_machine)),
the EVM is responsible for computing changes to the state deterministically
regardless of its environment (hardware and OS).
This means that every node has to get the exact same result
given an identical starting state and transaction (tx).

The EVM is considered to be the part of the Ethereum protocol
that handles the deployment and execution of [smart contracts](https://ethereum.org/en/developers/docs/smart-contracts/).
To make a clear distinction:

* The Ethereum protocol describes a blockchain,
  in which all Ethereum accounts and smart contracts live.
  It has only one canonical state (a data structure, which keeps all accounts)
  at any given block in the chain.
* The EVM, however, is the [state machine](https://en.wikipedia.org/wiki/Finite-state_machine)
  that defines the rules for computing a new valid state from block to block.
  It is an isolated runtime, which means
  that code running inside the EVM has no access to network, filesystem, or other processes (not external APIs).

The `x/evm` module implements the EVM as a Cosmos SDK module.
It allows users to interact with the EVM by submitting Ethereum txs
and executing their containing messages on the given state to evoke a state transition.

#### State

The Ethereum state is a data structure,
implemented as a [Merkle Patricia Tree](https://en.wikipedia.org/wiki/Merkle_tree),
that keeps all accounts on the chain.
The EVM makes changes to this data structure resulting in a new state with a different state root.
Ethereum can therefore be seen as a state chain
that transitions from one state to another
by executing transactions in a block using the EVM.
A new block of txs can be described through its block header
(parent hash, block number, time stamp, nonce, receipts,...).

#### Accounts

There are two types of accounts that can be stored in state at a given address:

* **Externally Owned Account (EOA)**: Has nonce (tx counter) and balance
* **Smart Contract**: Has nonce, balance, (immutable) code hash, storage root (another Merkle Patricia Trie)

Smart contracts are just like regular accounts on the blockchain,
which additionally store executable code in an Ethereum-specific binary format,
known as **EVM bytecode**.
They are typically written in an Ethereum high level language, such as Solidity,
which is compiled down to EVM bytecode
and deployed on the blockchain by submitting a transaction using an Ethereum client.

#### Architecture

The EVM operates as a stack-based machine.
It's main architecture components consist of:

* Virtual ROM: contract code is pulled into this read only memory when processing txs
* Machine state (volatile): changes as the EVM runs and is wiped clean after processing each tx
    * Program counter (PC)
    * Gas: keeps track of how much gas is used
    * Stack and Memory: compute state changes
* Access to account storage (persistent)

#### State Transitions with Smart Contracts

Typically smart contracts expose a public ABI,
which is a list of supported ways a user can interact with a contract.
To interact with a contract and invoke a state transition,
a user will submit a tx carrying any amount of gas and a data payload formatted according to the ABI,
specifying the type of interaction and any additional parameters.
When the tx is received, the EVM executes the smart contracts' EVM bytecode using the tx payload.

#### Executing EVM bytecode

A contract's EVM bytecode consists of basic operations (add, multiply, store, etc...), called **Opcodes**.
Each Opcode execution requires gas that needs to be paid with the tx.
The EVM is therefore considered quasi-turing complete,
as it allows any arbitrary computation,
but the amount of computations during a contract execution is limited to the amount of gas provided in the tx.
Each Opcode's [**gas cost**](https://www.evm.codes/) reflects the cost of running these operations on actual computer hardware
(e.g. `ADD = 3gas` and `SSTORE = 100gas`).
To calculate the gas consumption of a tx, the gas cost is multiplied by the **gas price**,
which can change depending on the demand of the network at the time.
If the network is under heavy load, you might have to pay a higher gas price to get your tx executed.
If the gas limit is hit (out of gas exception) no changes to the Ethereum state are applied,
except that the sender's nonce increments and their balance goes down to pay for wasting the EVM's time.

Smart contracts can also call other smart contracts.
Each call to a new contract creates a new instance of the EVM (including a new stack and memory).
Each call passes the sandbox state to the next EVM.
If the gas runs out, all state changes are discarded.
Otherwise, they are kept.

For further reading, please refer to:

* [EVM](https://eth.wiki/concepts/evm/evm)
* [EVM Architecture](https://cypherpunks-core.github.io/ethereumbook/13evm.html#evm_architecture)
* [What is Ethereum](https://ethdocs.org/en/latest/introduction/what-is-ethereum.html#what-is-ethereum)
* [Opcodes](https://www.ethervm.io/)

### Evmos as Geth implementation

Evmos contains an implementation of the [Ethereum protocol in Golang](https://geth.ethereum.org/docs/getting-started)
(Geth) as a Cosmos SDK module.
Geth includes an implementation of the EVM to compute state transitions.
Have a look at the [go-ethereum source code](https://github.com/ethereum/go-ethereum/blob/master/core/vm/instructions.go)
to see how the EVM opcodes are implemented.
Just as Geth can be run as an Ethereum node,
Evmos can be run as a node to compute state transitions with the EVM.
Evmos supports Geth's standard [Ethereum JSON-RPC APIs](https://docs.evmos.org/develop/api/ethereum-json-rpc/methods)
in order to be Web3 and EVM compatible.

#### JSON-RPC

JSON-RPC is a stateless, lightweight remote procedure call (RPC) protocol.
Primarily this specification defines several data structures
and the rules around their processing.
It is transport agnostic in that the concepts can be used within the same process,
over sockets, over HTTP, or in many various message passing environments.
It uses JSON (RFC 4627) as a data format.

##### JSON-RPC Example: `eth_call`

The JSON-RPC method [`eth_call`](https://docs.evmos.org/develop/api/ethereum-json-rpc/methods#eth-call) allows you
to execute messages against contracts.
Usually, you need to send a transaction to a Geth node to include it in the mempool,
then nodes gossip between each other and eventually the transaction is included in a block and gets executed.
`eth_call` however lets you send data to a contract and see what happens without committing a transaction.

In the Geth implementation, calling the endpoint roughly goes through the following steps:

1. The `eth_call` request is transformed to call the `func (s *PublicBlockchainAPI) Call()` function using the `eth` namespace
2. [`Call()`](https://github.com/ethereum/go-ethereum/blob/master/internal/ethapi/api.go#L982) is given
   the transaction arguments, the block to call against and optional arguments that modify the state to call against.
   It then calls `DoCall()`.
3. [`DoCall()`](https://github.com/ethereum/go-ethereum/blob/d575a2d3bc76dfbdefdd68b6cffff115542faf75/internal/ethapi/api.go#L891)
   transforms the arguments into a `ethtypes.message`, instantiates an EVM
   and applies the message with `core.ApplyMessage`
4. [`ApplyMessage()`](https://github.com/ethereum/go-ethereum/blob/d575a2d3bc76dfbdefdd68b6cffff115542faf75/core/state_transition.go#L180)
   calls the state transition `TransitionDb()`
5. [`TransitionDb()`](https://github.com/ethereum/go-ethereum/blob/d575a2d3bc76dfbdefdd68b6cffff115542faf75/core/state_transition.go#L275)
   either `Create()`s a new contract or `Call()`s a contract
6. [`evm.Call()`](https://github.com/ethereum/go-ethereum/blob/d575a2d3bc76dfbdefdd68b6cffff115542faf75/core/vm/evm.go#L168)
   runs the interpreter `evm.interpreter.Run()` to execute the message.
   If the execution fails, the state is reverted to a snapshot taken before the execution and gas is consumed.
7. [`Run()`](https://github.com/ethereum/go-ethereum/blob/d575a2d3bc76dfbdefdd68b6cffff115542faf75/core/vm/interpreter.go#L116)
   performs a loop to execute the opcodes.

The Evmos implementation is similar and makes use of the gRPC query client which is included in the Cosmos SDK:

1. `eth_call` request is transformed to call the `func (e *PublicAPI) Call` function using the `eth` namespace
2. [`Call()`](https://github.com/evmos/ethermint/blob/main/rpc/namespaces/ethereum/eth/api.go#L639) calls `doCall()`
3. [`doCall()`](https://github.com/evmos/ethermint/blob/main/rpc/namespaces/ethereum/eth/api.go#L656)
   transforms the arguments into a `EthCallRequest` and calls `EthCall()` using the query client of the evm module.
4. [`EthCall()`](https://github.com/evmos/ethermint/blob/main/x/evm/keeper/grpc_query.go#L212)
   transforms the arguments into a `ethtypes.message` and calls `ApplyMessageWithConfig()
5. [`ApplyMessageWithConfig()`](https://github.com/evmos/ethermint/blob/d5598932a7f06158b7a5e3aa031bbc94eaaae32c/x/evm/keeper/state_transition.go#L341)
   instantiates an EVM and either `Create()`s a new contract or `Call()`s a contract using the Geth implementation.

#### StateDB

The `StateDB` interface from [go-ethereum](https://github.com/ethereum/go-ethereum/blob/master/core/vm/interface.go)
represents an EVM database for full state querying.
EVM state transitions are enabled by this interface, which in the `x/evm` module is implemented by the `Keeper`.
The implementation of this interface is what makes Evmos EVM compatible.

### Consensus Engine

The application using the `x/evm` module interacts with the Tendermint Core Consensus Engine
over an Application Blockchain Interface (ABCI).
Together, the application and Tendermint Core form the programs that run a complete blockchain
and combine business logic with decentralized data storage.

Ethereum transactions which are submitted to the `x/evm` module take part in this consensus process
before being executed and changing the application state.
We encourage to understand the basics of the [Tendermint consensus engine](https://docs.tendermint.com/main/introduction/what-is-tendermint.html#intro-to-abci)
in order to understand state transitions in detail.

### Transaction Logs

On every `x/evm` transaction, the result contains the Ethereum `Log`s from the state machine execution
that are used by the JSON-RPC Web3 server for filter querying and for processing the EVM Hooks.

The tx logs are stored in the transient store during tx execution
and then emitted through cosmos events after the transaction has been processed.
They can be queried via gRPC and JSON-RPC.

### Block Bloom

Bloom is the bloom filter value in bytes for each block that can be used for filter queries.
The block bloom value is stored in the transient store
and then emitted through a cosmos event during `EndBlock` processing.
They can be queried via gRPC and JSON-RPC.

:::tip
👉 **Note**: Since they are not stored on state, Transaction Logs and Block Blooms are not persisted after upgrades.
A user must use an archival node after upgrades in order to obtain legacy chain events.
:::

## State

This section gives you an overview of the objects stored in the `x/evm` module state,
functionalities that are derived from the go-ethereum `StateDB` interface,
and its implementation through the Keeper as well as the state implementation at genesis.

### State Objects

The `x/evm` module keeps the following objects in state:

#### State

|             | Description                                                                                                                              | Key                           | Value               | Store     |
|-------------|------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------|---------------------|-----------|
| Code        | Smart contract bytecode                                                                                                                  | `[]byte{1} + []byte(address)` | `[]byte{code}`      | KV        |
| Storage     | Smart contract storage                                                                                                                   | `[]byte{2} + [32]byte{key}`   | `[32]byte(value)`   | KV        |
| Block Bloom | Block bloom filter, used to accumulate the bloom filter of current block, emitted to events at end blocker.                              | `[]byte{1} + []byte(tx.Hash)` | `protobuf([]Log)`   | Transient |
| Tx Index    | Index of current transaction in current block.                                                                                           | `[]byte{2}`                   | `BigEndian(uint64)` | Transient |
| Log Size    | Number of the logs emitted so far in current block. Used to decide the log index of following logs.                                      | `[]byte{3}`                   | `BigEndian(uint64)` | Transient |
| Gas Used    | Amount of gas used by ethereum messages of current cosmos-sdk tx, it's necessary when cosmos-sdk tx contains multiple ethereum messages. | `[]byte{4}`                   | `BigEndian(uint64)` | Transient |

### StateDB

The `StateDB` interface is implemented by the `StateDB` in the `x/evm/statedb` module
to represent an EVM database for full state querying of both contracts and accounts.
Within the Ethereum protocol, `StateDB`s are used to store anything
within the IAVL tree and take care of caching and storing nested states.

```go
// github.com/ethereum/go-ethereum/core/vm/interface.go
type StateDB interface {
 CreateAccount(common.Address)

 SubBalance(common.Address, *big.Int)
 AddBalance(common.Address, *big.Int)
 GetBalance(common.Address) *big.Int

 GetNonce(common.Address) uint64
 SetNonce(common.Address, uint64)

 GetCodeHash(common.Address) common.Hash
 GetCode(common.Address) []byte
 SetCode(common.Address, []byte)
 GetCodeSize(common.Address) int

 AddRefund(uint64)
 SubRefund(uint64)
 GetRefund() uint64

 GetCommittedState(common.Address, common.Hash) common.Hash
 GetState(common.Address, common.Hash) common.Hash
 SetState(common.Address, common.Hash, common.Hash)

 Suicide(common.Address) bool
 HasSuicided(common.Address) bool

 // Exist reports whether the given account exists in state.
 // Notably this should also return true for suicided accounts.
 Exist(common.Address) bool
 // Empty returns whether the given account is empty. Empty
 // is defined according to EIP161 (balance = nonce = code = 0).
 Empty(common.Address) bool

 PrepareAccessList(sender common.Address, dest *common.Address, precompiles []common.Address, txAccesses types.AccessList)
 AddressInAccessList(addr common.Address) bool
 SlotInAccessList(addr common.Address, slot common.Hash) (addressOk bool, slotOk bool)
 // AddAddressToAccessList adds the given address to the access list. This operation is safe to perform
 // even if the feature/fork is not active yet
 AddAddressToAccessList(addr common.Address)
 // AddSlotToAccessList adds the given (address,slot) to the access list. This operation is safe to perform
 // even if the feature/fork is not active yet
 AddSlotToAccessList(addr common.Address, slot common.Hash)

 RevertToSnapshot(int)
 Snapshot() int

 AddLog(*types.Log)
 AddPreimage(common.Hash, []byte)

 ForEachStorage(common.Address, func(common.Hash, common.Hash) bool) error
}
```

The `StateDB` in the `x/evm` provides the following functionalities:

#### CRUD of Ethereum accounts

You can create `EthAccount` instances from the provided address
and set the value to store on the  `AccountKeeper`with `createAccount()`.
If an account with the given address already exists,
this function also resets any preexisting code and storage associated with that address.

An account's coin balance can be is managed through the `BankKeeper`
and can be read with `GetBalance()` and updated with `AddBalance()` and `SubBalance()`.

- `GetBalance()` returns the EVM denomination balance of the provided address.
  The denomination is obtained from the module parameters.
- `AddBalance()` adds the given amount to the address balance coin
  by minting new coins and transferring them to the address.
  The coin denomination is obtained from the module parameters.
- `SubBalance()` subtracts the given amount from the address balance
  by transferring the coins to an escrow account and then burning them.
  The coin denomination is obtained from the module parameters.
  This function performs a no-op if the amount is negative or the user doesn't have enough funds for the transfer.

The nonce (or transaction sequence) can be obtained from the Account `Sequence` via the auth module `AccountKeeper`.

- `GetNonce()` retrieves the account with the given address and returns the tx sequence (i.e nonce).
  The function performs a no-op if the account is not found.
- `SetNonce()` sets the given nonce as the sequence of the address' account.
  If the account doesn't exist, a new one will be created from the address.

The smart contract bytecode containing arbitrary contract logic is stored on the `EVMKeeper`
and it can be queried with `GetCodeHash()` ,`GetCode()` & `GetCodeSize()`and updated with `SetCode()`.

- `GetCodeHash()` fetches the account from the store and returns its code hash.
  If the account doesn't exist or is not an EthAccount type, it returns the empty code hash value.
- `GetCode()` returns the code byte array associated with the given address.
  If the code hash from the account is empty, this function returns nil.
- `SetCode()` stores the code byte array to the application KVStore and sets the code hash to the given account.
  The code is deleted from the store if it is empty.
- `GetCodeSize()` returns the size of the contract code associated with this object, or zero if none.

Gas refunded needs to be tracked and stored in a separate variable in
order to add it subtract/add it from/to the gas used value after the EVM
execution has finalized. The refund value is cleared on every transaction and at the end of every block.

- `AddRefund()` adds the given amount of gas to the in-memory refund value.
- `SubRefund()` subtracts the given amount of gas from the in-memory refund value.
  This function will panic if gas amount is greater than the current refund.
- `GetRefund()` returns the amount of gas available for return after the tx execution finalizes.
  This value is reset to 0 on every transaction.

The state is stored on the `EVMKeeper`.
It can be queried with `GetCommittedState()`, `GetState()` and updated with `SetState()`.

- `GetCommittedState()` returns the value set in store for the given key hash.
  If the key is not registered this function returns the empty hash.
- `GetState()` returns the in-memory dirty state for the given key hash,
  if not exist load the committed value from KVStore.
- `SetState()` sets the given hashes (key, value) to the state.
  If the value hash is empty, this function deletes the key from the state,
  the new value is kept in dirty state at first, and will be committed to KVStore in the end.

Accounts can also be set to a suicide state.
When a contract commits suicide, the account is marked as suicided,
when committing the code, storage and account are deleted
(from the next block and forward).

- `Suicide()` marks the given account as suicided and clears the account balance of the EVM tokens.
- `HasSuicided()` queries the in-memory flag to check if the account has been marked as suicided in the current transaction.
  Accounts that are suicided will be returned as non-nil during queries and "cleared" after the block has been committed.

To check account existence use `Exist()` and `Empty()`.

- `Exist()` returns true if the given account exists in store or if it has been
  marked as suicided.
- `Empty()` returns true if the address meets the following conditions:
    - nonce is 0
    - balance amount for evm denom is 0
    - account code hash is empty

#### EIP2930 functionality

Supports a transaction type that contains an [access list](https://eips.ethereum.org/EIPS/eip-2930),
a list of addresses and storage keys, that the transaction plans to access.
The access list state is kept in memory and discarded after the transaction committed.

- `PrepareAccessList()` handles the preparatory steps for executing a state transition
  in regard to both EIP-2929 and EIP-2930.
  This method should only be called if Yolov3/Berlin/2929+2930 is applicable at the current number.
    - Add sender to access list (EIP-2929)
    - Add destination to access list (EIP-2929)
    - Add precompiles to access list (EIP-2929)
    - Add the contents of the optional tx access list (EIP-2930)
- `AddressInAccessList()` returns true if the address is registered.
- `SlotInAccessList()` checks if the address and the slots are registered.
- `AddAddressToAccessList()` adds the given address to the access list.
  If the address is already in the access list, this function performs a no-op.
- `AddSlotToAccessList()` adds the given (address, slot) to the access list.
  If the address and slot are already in the access list, this function performs a no-op.

#### Snapshot state and Revert functionality

The EVM uses state-reverting exceptions to handle errors.
Such an exception will undo all changes made to the state in the current call (and all its sub-calls),
and the caller could handle the error and don't propagate.
You can use `Snapshot()` to identify the current state with a revision
and revert the state to a given revision with `RevertToSnapshot()` to support this feature.

- `Snapshot()` creates a new snapshot and returns the identifier.
- `RevertToSnapshot(rev)` undo all the modifications up to the snapshot identified as `rev`.

Evmos adapted the [go-ethereum journal implementation](https://github.com/ethereum/go-ethereum/blob/master/core/state/journal.go#L39)
to support this, it uses a list of journal logs to record all the state modification operations done so far,
snapshot is consists of a unique id and an index in the log list,
and to revert to a snapshot it just undoes the journal logs after the snapshot index in reversed order.

#### Ethereum Transaction logs

With `AddLog()` you can append the given Ethereum `Log` to the list of logs
associated with the transaction hash kept in the current state.
This function also fills in the tx hash, block hash, tx index and log index fields before setting the log to store.

### Keeper

The EVM module `Keeper` grants access to the EVM module state
and implements `statedb.Keeper` interface to support the `StateDB` implementation.
The Keeper contains a store key that allows the DB
to write to a concrete subtree of the multistore that is only accessible by the EVM module.
Instead of using a trie and database for querying and persistence (the `StateDB` implementation),
Evmos uses the Cosmos `KVStore` (key-value store) and Cosmos SDK `Keeper` to facilitate state transitions.

To support the interface functionality, it imports 4 module Keepers:

- `auth`: CRUD accounts
- `bank`: accounting (supply) and CRUD of balances
- `staking`: query historical headers
- `fee market`: EIP1559 base fee for processing `DynamicFeeTx`
  after the `London` hard fork has been activated on the `ChainConfig` parameters

```go
type Keeper struct {
 // Protobuf codec
 cdc codec.BinaryCodec
 // Store key required for the EVM Prefix KVStore. It is required by:
 // - storing account's Storage State
 // - storing account's Code
 // - storing Bloom filters by block height. Needed for the Web3 API.
 // For the full list, check the module specification
 storeKey sdk.StoreKey

 // key to access the transient store, which is reset on every block during Commit
 transientKey sdk.StoreKey

 // module specific parameter space that can be configured through governance
 paramSpace paramtypes.Subspace
 // access to account state
 accountKeeper types.AccountKeeper
 // update balance and accounting operations with coins
 bankKeeper types.BankKeeper
 // access historical headers for EVM state transition execution
 stakingKeeper types.StakingKeeper
 // fetch EIP1559 base fee and parameters
 feeMarketKeeper types.FeeMarketKeeper

 // chain ID number obtained from the context's chain id
 eip155ChainID *big.Int

 // Tracer used to collect execution traces from the EVM transaction execution
 tracer string
 // trace EVM state transition execution. This value is obtained from the `--trace` flag.
 // For more info check https://geth.ethereum.org/docs/dapp/tracing
 debug bool

 // EVM Hooks for tx post-processing
 hooks types.EvmHooks
}
```

### Genesis State

The `x/evm` module `GenesisState` defines the state necessary for initializing the chain from a previous exported height.
It contains the `GenesisAccounts` and the module parameters

```go
type GenesisState struct {
  // accounts is an array containing the ethereum genesis accounts.
  Accounts []GenesisAccount `protobuf:"bytes,1,rep,name=accounts,proto3" json:"accounts"`
  // params defines all the parameters of the module.
  Params Params `protobuf:"bytes,2,opt,name=params,proto3" json:"params"`
}
```

### Genesis Accounts

The `GenesisAccount` type corresponds to an adaptation of the Ethereum `GenesisAccount` type.
It defines an account to be initialized in the genesis state.

Its main difference is that the one on Evmos uses a custom `Storage` type
that uses a slice instead of maps for the evm `State` (due to non-determinism),
and that it doesn't contain the private key field.

It is also important to note that since the `auth` module on the Cosmos SDK manages the account state,
the `Address` field must correspond to an existing `EthAccount`
that is stored in the `auth`'s module `Keeper` (i.e `AccountKeeper`).
Addresses use the **[EIP55](https://eips.ethereum.org/EIPS/eip-55)** hex **[format](https://docs.evmos.org/protocol/concepts/accounts#address-formats-for-clients)**
on `genesis.json`.

```go
type GenesisAccount struct {
  // address defines an ethereum hex formated address of an account
  Address string `protobuf:"bytes,1,opt,name=address,proto3" json:"address,omitempty"`
  // code defines the hex bytes of the account code.
  Code string `protobuf:"bytes,2,opt,name=code,proto3" json:"code,omitempty"`
  // storage defines the set of state key values for the account.
  Storage Storage `protobuf:"bytes,3,rep,name=storage,proto3,castrepeated=Storage" json:"storage"`
}
```

## State Transitions

The `x/evm` module allows for users to submit Ethereum transactions (`Tx`)
and execute their containing messages to evoke state transitions on the given state.

Users submit transactions client-side to broadcast it to the network.
When the transaction is included in a block during consensus, it is executed server-side.
We highly recommend to understand the basics of the [Tendermint consensus engine](https://docs.tendermint.com/main/introduction/what-is-tendermint.html#intro-to-abci)
to understand the State Transitions in detail.

### Client-Side

:::tip
👉 This is based on the `eth_sendTransaction` JSON-RPC
:::

1. A user submits a transaction via one of the available JSON-RPC endpoints
   using an Ethereum-compatible client or wallet (eg Metamask, WalletConnect, Ledger, etc):
   a. eth (public) namespace:
    - `eth_sendTransaction`
    - `eth_sendRawTransaction`
      b. personal (private) namespace:
    - `personal_sendTransaction`
2. An instance of `MsgEthereumTx` is created after populating the RPC transaction
   using `SetTxDefaults` to fill missing tx arguments with  default values
3. The `Tx` fields are validated (stateless) using `ValidateBasic()`
4. The `Tx` is **signed** using the key associated with the sender address
   and the latest ethereum hard fork (`London`, `Berlin`, etc) from the `ChainConfig`
5. The `Tx` is **built** from the msg fields using the Cosmos Config builder
6. The `Tx` is **broadcast** in [sync mode](https://docs.cosmos.network/main/run-node/txs.html#broadcasting-a-transaction)
   to ensure to wait for
   a [`CheckTx`](https://docs.tendermint.com/main/introduction/what-is-tendermint.html#intro-to-abci) execution response.
   Transactions are validated by the application using `CheckTx()`,
   before being added to the mempool of the consensus engine.
7. JSON-RPC user receives a response with the [`RLP`](https://eth.wiki/en/fundamentals/rlp) hash of the transaction fields.
   This hash is different from the default hash used by SDK Transactions
   that calculates the `sha256` hash of the transaction bytes.

### Server-Side

Once a block (containing the `Tx`) has been committed during consensus,
it is applied to the application in a series of ABCI msgs server-side.

Each `Tx` is handled by the application by calling [`RunTx`](https://docs.cosmos.network/main/core/baseapp.html#runtx).
After a stateless validation on each `sdk.Msg` in the `Tx`,
the `AnteHandler` confirms whether the `Tx` is an Ethereum or SDK transaction.
As an Ethereum transaction it's containing msgs are then handled
by the `x/evm` module to update the application's state.

#### AnteHandler

The `anteHandler` is run for every transaction.
It checks if the `Tx` is an Ethereum transaction and routes it to an internal ante handler.
Here, `Tx`s are handled using EthereumTx extension options to process them differently than normal Cosmos SDK transactions.
The `antehandler` runs through a series of options and their `AnteHandle` functions for each `Tx`:

- `EthSetUpContextDecorator()` is adapted from SetUpContextDecorator from cosmos-sdk,
  it ignores gas consumption by setting the gas meter to infinite
- `EthValidateBasicDecorator(evmKeeper)` validates the fields of an Ethereum type Cosmos `Tx` msg
- `EthSigVerificationDecorator(evmKeeper)` validates that the registered chain id is the same as the one on the message,
  and that the signer address matches the one defined on the message.
  It's not skipped for RecheckTx, because it set `From` address which is critical from other ante handler to work.
  Failure in RecheckTx will prevent tx to be included into block, especially when CheckTx succeed,
  in which case user won't see the error message.
- `EthAccountVerificationDecorator(ak, bankKeeper, evmKeeper)`
  will verify, that the sender balance is greater than the total transaction cost.
  The account will be set to store if it doesn't exist, i.e cannot be found on store.
  This AnteHandler decorator will fail if:
    - any of the msgs is not a MsgEthereumTx
    - from address is empty
    - account balance is lower than the transaction cost
- `EthNonceVerificationDecorator(ak)` validates that the transaction nonces are valid
  and equivalent to the sender account’s current nonce.
- `EthGasConsumeDecorator(evmKeeper)` validates that the Ethereum tx message has enough
  to cover intrinsic gas (during CheckTx only) and that the sender has enough balance to pay for the gas cost.
  Intrinsic gas for a transaction is the amount of gas that the transaction uses before the transaction is executed.
  The gas is a constant value plus any cost incurred by additional bytes of data supplied with the transaction.
  This AnteHandler decorator will fail if:
    - the transaction contains more than one message
    - the message is not a MsgEthereumTx
    - sender account cannot be found
    - transaction's gas limit is lower than the intrinsic gas
    - user doesn't have enough balance to deduct the transaction fees (gas_limit * gas_price)
    - transaction or block gas meter runs out of gas
- `CanTransferDecorator(evmKeeper, feeMarketKeeper)` creates an EVM from the message
  and calls the BlockContext CanTransfer function to see if the address can execute the transaction.
- `EthIncrementSenderSequenceDecorator(ak)`  handles incrementing the sequence of the signer (i.e sender).
  If the transaction is a contract creation, the nonce will be incremented
  during the transaction execution and not within this AnteHandler decorator.

The options `authante.NewMempoolFeeDecorator()`, `authante.NewTxTimeoutHeightDecorator()`
and `authante.NewValidateMemoDecorator(ak)` are the same as for a Cosmos `Tx`.
Click [here](https://docs.cosmos.network/main/basics/gas-fees.html#antehandler) for more on the `anteHandler`.

#### EVM module

After authentication through the `antehandler`,
each `sdk.Msg` (in this case `MsgEthereumTx`) in the `Tx`
is delivered to the Msg Handler in the `x/evm` module
and runs through the following the steps:

1. Convert `Msg` to an ethereum `Tx` type
2. Apply `Tx` with `EVMConfig` and attempt to perform a state transition,
   that will only be persisted (committed) to the underlying KVStore
   if the transaction does not fail:
    1. Confirm that `EVMConfig` is created
    2. Create the ethereum signer using chain config value from `EVMConfig`
    3. Set the ethereum transaction hash to the (impermanent) transient store so
       that it's also available on the StateDB functions
    4. Generate a new EVM instance
    5. Confirm that EVM params for contract creation (`EnableCreate`)
       and contract execution (`EnableCall`) are enabled
    6. Apply message. If `To` address is `nil`, create new contract using code as deployment code.
       Else call contract at given address with the given input as parameters
    7. Calculate gas used by the evm operation
3. If `Tx` applied successfully
    1. Execute EVM `Tx` postprocessing hooks. If hooks return error, revert the whole `Tx`
    2. Refund gas according to Ethereum gas accounting rules
    3. Update block bloom filter value using the logs generated from the tx
    4. Emit SDK events for the transaction fields and tx logs

## Transactions

This section defines the `sdk.Msg` concrete types that result in the state transitions defined on the previous section.

## `MsgEthereumTx`

An EVM state transition can be achieved by using the `MsgEthereumTx`.
This message encapsulates an Ethereum transaction data (`TxData`) as a `sdk.Msg`.
It contains the necessary transaction data fields.
Note, that the `MsgEthereumTx` implements both the [`sdk.Msg`](https://github.com/cosmos/cosmos-sdk/blob/v0.39.2/types/tx_msg.go#L7-L29)
and [`sdk.Tx`](https://github.com/cosmos/cosmos-sdk/blob/v0.39.2/types/tx_msg.go#L33-L41) interfaces.
Normally, SDK messages only implement the former, while the latter is a group of messages bundled together.

```go
type MsgEthereumTx struct {
 // inner transaction data
 Data *types.Any `protobuf:"bytes,1,opt,name=data,proto3" json:"data,omitempty"`
 // DEPRECATED: encoded storage size of the transaction
 Size_ float64 `protobuf:"fixed64,2,opt,name=size,proto3" json:"-"`
 // transaction hash in hex format
 Hash string `protobuf:"bytes,3,opt,name=hash,proto3" json:"hash,omitempty" rlp:"-"`
 // ethereum signer address in hex format. This address value is checked
 // against the address derived from the signature (V, R, S) using the
 // secp256k1 elliptic curve
 From string `protobuf:"bytes,4,opt,name=from,proto3" json:"from,omitempty"`
}
```

This message field validation is expected to fail if:

- `From` field is defined and the address is invalid
- `TxData` stateless validation fails

The transaction execution is expected to fail if:

- Any of the custom `AnteHandler` Ethereum decorators checks fail:
    - Minimum gas amount requirements for transaction
    - Tx sender account doesn't exist or hasn't enough balance for fees
    - Account sequence doesn't match the transaction `Data.AccountNonce`
    - Message signature verification fails
- EVM contract creation (i.e `evm.Create`) fails, or `evm.Call` fails

#### Conversion

The `MsgEthreumTx` can be converted to the go-ethereum `Transaction` and `Message` types
in order to create and call evm contracts.

```go
// AsTransaction creates an Ethereum Transaction type from the msg fields
func (msg MsgEthereumTx) AsTransaction() *ethtypes.Transaction {
 txData, err := UnpackTxData(msg.Data)
 if err != nil {
  return nil
 }

 return ethtypes.NewTx(txData.AsEthereumData())
}

// AsMessage returns the transaction as a core.Message.
func (tx *Transaction) AsMessage(s Signer, baseFee *big.Int) (Message, error) {
 msg := Message{
  nonce:      tx.Nonce(),
  gasLimit:   tx.Gas(),
  gasPrice:   new(big.Int).Set(tx.GasPrice()),
  gasFeeCap:  new(big.Int).Set(tx.GasFeeCap()),
  gasTipCap:  new(big.Int).Set(tx.GasTipCap()),
  to:         tx.To(),
  amount:     tx.Value(),
  data:       tx.Data(),
  accessList: tx.AccessList(),
  isFake:     false,
 }
 // If baseFee provided, set gasPrice to effectiveGasPrice.
 if baseFee != nil {
  msg.gasPrice = math.BigMin(msg.gasPrice.Add(msg.gasTipCap, baseFee), msg.gasFeeCap)
 }
 var err error
 msg.from, err = Sender(s, tx)
 return msg, err
}
```

#### Signing

In order for the signature verification to be valid, the  `TxData` must contain the `v | r | s` values from the `Signer`.
Sign calculates a secp256k1 ECDSA signature and signs the transaction.
It takes a keyring signer and the chainID to sign an Ethereum transaction according to EIP155 standard.
This method mutates the transaction as it populates the V, R, S fields of the Transaction's Signature.
The function will fail if the sender address is not defined for the msg
or if the sender is not registered on the keyring.

```go
// Sign calculates a secp256k1 ECDSA signature and signs the transaction. It
// takes a keyring signer and the chainID to sign an Ethereum transaction according to
// EIP155 standard.
// This method mutates the transaction as it populates the V, R, S
// fields of the Transaction's Signature.
// The function will fail if the sender address is not defined for the msg or if
// the sender is not registered on the keyring
func (msg *MsgEthereumTx) Sign(ethSigner ethtypes.Signer, keyringSigner keyring.Signer) error {
 from := msg.GetFrom()
 if from.Empty() {
  return fmt.Errorf("sender address not defined for message")
 }

 tx := msg.AsTransaction()
 txHash := ethSigner.Hash(tx)

 sig, _, err := keyringSigner.SignByAddress(from, txHash.Bytes())
 if err != nil {
  return err
 }

 tx, err = tx.WithSignature(ethSigner, sig)
 if err != nil {
  return err
 }

 msg.FromEthereumTx(tx)
 return nil
}
```

### TxData

The `MsgEthereumTx` supports the 3 valid Ethereum transaction data types from go-ethereum:
`LegacyTx`, `AccessListTx`  and `DynamicFeeTx`.
These types are defined as protobuf messages
and packed into a `proto.Any` interface type in the `MsgEthereumTx` field.

- `LegacyTx`: [EIP-155](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-155.md) transaction type
- `DynamicFeeTx`: [EIP-1559](https://eips.ethereum.org/EIPS/eip-1559) transaction type.
  Enabled by London hard fork block
- `AccessListTx`: [EIP-2930](https://eips.ethereum.org/EIPS/eip-2930) transaction type.
  Enabled by Berlin hard fork block

### `LegacyTx`

The transaction data of regular Ethereum transactions.

```go
type LegacyTx struct {
 // nonce corresponds to the account nonce (transaction sequence).
 Nonce uint64 `protobuf:"varint,1,opt,name=nonce,proto3" json:"nonce,omitempty"`
 // gas price defines the value for each gas unit
 GasPrice *github_com_cosmos_cosmos_sdk_types.Int `protobuf:"bytes,2,opt,name=gas_price,json=gasPrice,proto3,customtype=github.com/cosmos/cosmos-sdk/types.Int" json:"gas_price,omitempty"`
 // gas defines the gas limit defined for the transaction.
 GasLimit uint64 `protobuf:"varint,3,opt,name=gas,proto3" json:"gas,omitempty"`
 // hex formatted address of the recipient
 To string `protobuf:"bytes,4,opt,name=to,proto3" json:"to,omitempty"`
 // value defines the unsigned integer value of the transaction amount.
 Amount *github_com_cosmos_cosmos_sdk_types.Int `protobuf:"bytes,5,opt,name=value,proto3,customtype=github.com/cosmos/cosmos-sdk/types.Int" json:"value,omitempty"`
 // input defines the data payload bytes of the transaction.
 Data []byte `protobuf:"bytes,6,opt,name=data,proto3" json:"data,omitempty"`
 // v defines the signature value
 V []byte `protobuf:"bytes,7,opt,name=v,proto3" json:"v,omitempty"`
 // r defines the signature value
 R []byte `protobuf:"bytes,8,opt,name=r,proto3" json:"r,omitempty"`
 // s define the signature value
 S []byte `protobuf:"bytes,9,opt,name=s,proto3" json:"s,omitempty"`
}
```

This message field validation is expected to fail if:

- `GasPrice` is invalid (`nil` , negative or out of int256 bound)
- `Fee` (gasprice * gaslimit) is invalid
- `Amount` is invalid (negative or out of int256 bound)
- `To` address is invalid (non valid ethereum hex address)

### `DynamicFeeTx`

The transaction data of EIP-1559 dynamic fee transactions.

```go
type DynamicFeeTx struct {
 // destination EVM chain ID
 ChainID *github_com_cosmos_cosmos_sdk_types.Int `protobuf:"bytes,1,opt,name=chain_id,json=chainId,proto3,customtype=github.com/cosmos/cosmos-sdk/types.Int" json:"chainID"`
 // nonce corresponds to the account nonce (transaction sequence).
 Nonce uint64 `protobuf:"varint,2,opt,name=nonce,proto3" json:"nonce,omitempty"`
 // gas tip cap defines the max value for the gas tip
 GasTipCap *github_com_cosmos_cosmos_sdk_types.Int `protobuf:"bytes,3,opt,name=gas_tip_cap,json=gasTipCap,proto3,customtype=github.com/cosmos/cosmos-sdk/types.Int" json:"gas_tip_cap,omitempty"`
 // gas fee cap defines the max value for the gas fee
 GasFeeCap *github_com_cosmos_cosmos_sdk_types.Int `protobuf:"bytes,4,opt,name=gas_fee_cap,json=gasFeeCap,proto3,customtype=github.com/cosmos/cosmos-sdk/types.Int" json:"gas_fee_cap,omitempty"`
 // gas defines the gas limit defined for the transaction.
 GasLimit uint64 `protobuf:"varint,5,opt,name=gas,proto3" json:"gas,omitempty"`
 // hex formatted address of the recipient
 To string `protobuf:"bytes,6,opt,name=to,proto3" json:"to,omitempty"`
 // value defines the the transaction amount.
 Amount *github_com_cosmos_cosmos_sdk_types.Int `protobuf:"bytes,7,opt,name=value,proto3,customtype=github.com/cosmos/cosmos-sdk/types.Int" json:"value,omitempty"`
 // input defines the data payload bytes of the transaction.
 Data     []byte     `protobuf:"bytes,8,opt,name=data,proto3" json:"data,omitempty"`
 Accesses AccessList `protobuf:"bytes,9,rep,name=accesses,proto3,castrepeated=AccessList" json:"accessList"`
 // v defines the signature value
 V []byte `protobuf:"bytes,10,opt,name=v,proto3" json:"v,omitempty"`
 // r defines the signature value
 R []byte `protobuf:"bytes,11,opt,name=r,proto3" json:"r,omitempty"`
 // s define the signature value
 S []byte `protobuf:"bytes,12,opt,name=s,proto3" json:"s,omitempty"`
}
```

This message field validation is expected to fail if:

- `GasTipCap` is invalid (`nil` , negative or overflows int256)
- `GasFeeCap` is invalid (`nil` , negative or overflows int256)
- `GasFeeCap` is less than `GasTipCap`
- `Fee` (gas price * gas limit) is invalid (overflows int256)
- `Amount` is invalid (negative or overflows int256)
- `To` address is invalid (non-valid ethereum hex address)
- `ChainID` is `nil`

### `AccessListTx`

The transaction data of EIP-2930 access list transactions.

```go
type AccessListTx struct {
 // destination EVM chain ID
 ChainID *github_com_cosmos_cosmos_sdk_types.Int `protobuf:"bytes,1,opt,name=chain_id,json=chainId,proto3,customtype=github.com/cosmos/cosmos-sdk/types.Int" json:"chainID"`
 // nonce corresponds to the account nonce (transaction sequence).
 Nonce uint64 `protobuf:"varint,2,opt,name=nonce,proto3" json:"nonce,omitempty"`
 // gas price defines the value for each gas unit
 GasPrice *github_com_cosmos_cosmos_sdk_types.Int `protobuf:"bytes,3,opt,name=gas_price,json=gasPrice,proto3,customtype=github.com/cosmos/cosmos-sdk/types.Int" json:"gas_price,omitempty"`
 // gas defines the gas limit defined for the transaction.
 GasLimit uint64 `protobuf:"varint,4,opt,name=gas,proto3" json:"gas,omitempty"`
 // hex formatted address of the recipient
 To string `protobuf:"bytes,5,opt,name=to,proto3" json:"to,omitempty"`
 // value defines the unsigned integer value of the transaction amount.
 Amount *github_com_cosmos_cosmos_sdk_types.Int `protobuf:"bytes,6,opt,name=value,proto3,customtype=github.com/cosmos/cosmos-sdk/types.Int" json:"value,omitempty"`
 // input defines the data payload bytes of the transaction.
 Data     []byte     `protobuf:"bytes,7,opt,name=data,proto3" json:"data,omitempty"`
 Accesses AccessList `protobuf:"bytes,8,rep,name=accesses,proto3,castrepeated=AccessList" json:"accessList"`
 // v defines the signature value
 V []byte `protobuf:"bytes,9,opt,name=v,proto3" json:"v,omitempty"`
 // r defines the signature value
 R []byte `protobuf:"bytes,10,opt,name=r,proto3" json:"r,omitempty"`
 // s define the signature value
 S []byte `protobuf:"bytes,11,opt,name=s,proto3" json:"s,omitempty"`
}
```

This message field validation is expected to fail if:

- `GasPrice` is invalid (`nil` , negative or overflows int256)
- `Fee` (gas price * gas limit) is invalid (overflows int256)
- `Amount` is invalid (negative or overflows int256)
- `To` address is invalid (non-valid ethereum hex address)
- `ChainID` is `nil`

## ABCI

The Application Blockchain Interface (ABCI) allows the application to interact with the Tendermint Consensus engine.
The application maintains several ABCI connections with Tendermint.
The most relevant for the  `x/evm` is the [Consensus connection at Commit](https://docs.tendermint.com/v0.33/app-dev/app-development.html#consensus-connection).
This connection is responsible for block execution and calls the functions `InitChain`
(containing `InitGenesis`), `BeginBlock`, `DeliverTx`, `EndBlock`, `Commit` .
`InitChain` is only called the first time a new blockchain is started
and `DeliverTx` is called for each transaction in the block.

### InitGenesis

`InitGenesis` initializes the EVM module genesis state by setting the `GenesisState` fields to the store.
In particular, it sets the parameters and genesis accounts (state and code).

### ExportGenesis

The `ExportGenesis` ABCI function exports the genesis state of the EVM module.
In particular, it retrieves all the accounts with their bytecode, balance and storage, the transaction logs,
and the EVM parameters and chain configuration.

### BeginBlock

The EVM module `BeginBlock` logic is executed prior to handling the state transitions from the transactions.
The main objective of this function is to:

- Set the context for the current block so that the block header, store, gas meter, etc.
  are available to the `Keeper` once one of the `StateDB` functions are called during EVM state transitions.
- Set the EIP155 `ChainID` number (obtained from the full chain-id),
  in case it hasn't been set before during `InitChain`

### EndBlock

The EVM module `EndBlock` logic occurs after executing all the state transitions from the transactions.
The main objective of this function is to:

- Emit Block bloom events
    - This is due for web3 compatibility as the Ethereum headers contain this type as a field.
      The JSON-RPC service uses this event query to construct an Ethereum header from a Tendermint header.
    - The block bloom filter value is obtained from the transient store and then emitted

## Hooks

The `x/evm` module implements an `EvmHooks` interface that extend and customize the `Tx` processing logic externally.

This supports EVM contracts to call native cosmos modules by

1. defining a log signature and emitting the specific log from the smart contract,
2. recognizing those logs in the native tx processing code, and
3. converting them to native module calls.

To do this, the interface includes a  `PostTxProcessing` hook that registers custom `Tx` hooks in the `EvmKeeper`.
These  `Tx` hooks are processed after the EVM state transition is finalized and doesn't fail.
Note that there are no default hooks implemented in the EVM module.

```go
type EvmHooks interface {
 // Must be called after tx is processed successfully, if return an error, the whole transaction is reverted.
 PostTxProcessing(ctx sdk.Context, msg core.Message, receipt *ethtypes.Receipt) error
}
```

## `PostTxProcessing`

`PostTxProcessing` is only called after an EVM transaction finished successfully
and delegates the call to underlying hooks.
If no hook has been registered, this function returns with a `nil` error.

```go
func (k *Keeper) PostTxProcessing(ctx sdk.Context, msg core.Message, receipt *ethtypes.Receipt) error {
 if k.hooks == nil {
  return nil
 }
 return k.hooks.PostTxProcessing(k.Ctx(), msg, receipt)
}
```

It's executed in the same cache context as the EVM transaction,
if it returns an error, the whole EVM transaction is reverted,
if the hook implementor doesn't want to revert the tx, they can always return `nil` instead.

The error returned by the hooks is translated to a VM error `failed to process native logs`,
the detailed error message is stored in the return value.
The message is sent to native modules asynchronously, there's no way for the caller to catch and recover the error.

### Use Case: Call Native ERC20 Module on Evmos

Here is an example taken from the Evmos [erc20 module](erc20.md)
that shows how the `EVMHooks` supports a contract calling a native module
to convert ERC-20 Tokens into Cosmos native Coins.
Following the steps from above.

You can define and emit a `Transfer` log signature in the smart contract like this:

```solidity
event Transfer(address indexed from, address indexed to, uint256 value);

function _transfer(address sender, address recipient, uint256 amount) internal virtual {
  require(sender != address(0), "ERC20: transfer from the zero address");
  require(recipient != address(0), "ERC20: transfer to the zero address");

  _beforeTokenTransfer(sender, recipient, amount);

  _balances[sender] = _balances[sender].sub(amount, "ERC20: transfer amount exceeds balance");
  _balances[recipient] = _balances[recipient].add(amount);
  emit Transfer(sender, recipient, amount);
}
```

The application will register a `BankSendHook` to the `EvmKeeper`.
It recognizes the ethereum tx `Log` and converts it to a call to the bank module's `SendCoinsFromAccountToAccount` method:

```go

const ERC20EventTransfer = "Transfer"

// PostTxProcessing implements EvmHooks.PostTxProcessing
func (k Keeper) PostTxProcessing(
 ctx sdk.Context,
 msg core.Message,
 receipt *ethtypes.Receipt,
) error {
 params := h.k.GetParams(ctx)
 if !params.EnableErc20 || !params.EnableEVMHook {
  // no error is returned to allow for other post-processing txs
  // to pass
  return nil
 }

 erc20 := contracts.ERC20BurnableContract.ABI

 for i, log := range receipt.Logs {
  if len(log.Topics) < 3 {
   continue
  }

  eventID := log.Topics[0] // event ID

  event, err := erc20.EventByID(eventID)
  if err != nil {
   // invalid event for ERC20
   continue
  }

  if event.Name != types.ERC20EventTransfer {
   h.k.Logger(ctx).Info("emitted event", "name", event.Name, "signature", event.Sig)
   continue
  }

  transferEvent, err := erc20.Unpack(event.Name, log.Data)
  if err != nil {
   h.k.Logger(ctx).Error("failed to unpack transfer event", "error", err.Error())
   continue
  }

  if len(transferEvent) == 0 {
   continue
  }

  tokens, ok := transferEvent[0].(*big.Int)
  // safety check and ignore if amount not positive
  if !ok || tokens == nil || tokens.Sign() != 1 {
   continue
  }

  // check that the contract is a registered token pair
  contractAddr := log.Address

  id := h.k.GetERC20Map(ctx, contractAddr)

  if len(id) == 0 {
   // no token is registered for the caller contract
   continue
  }

  pair, found := h.k.GetTokenPair(ctx, id)
  if !found {
   continue
  }

  // check that conversion for the pair is enabled
  if !pair.Enabled {
   // continue to allow transfers for the ERC20 in case the token pair is disabled
   h.k.Logger(ctx).Debug(
    "ERC20 token -> Cosmos coin conversion is disabled for pair",
    "coin", pair.Denom, "contract", pair.Erc20Address,
   )
   continue
  }

  // ignore as the burning always transfers to the zero address
  to := common.BytesToAddress(log.Topics[2].Bytes())
  if !bytes.Equal(to.Bytes(), types.ModuleAddress.Bytes()) {
   continue
  }

  // check that the event is Burn from the ERC20Burnable interface
  // NOTE: assume that if they are burning the token that has been registered as a pair, they want to mint a Cosmos coin

  // create the corresponding sdk.Coin that is paired with ERC20
  coins := sdk.Coins{{Denom: pair.Denom, Amount: sdk.NewIntFromBigInt(tokens)}}

  // Mint the coin only if ERC20 is external
  switch pair.ContractOwner {
  case types.OWNER_MODULE:
   _, err = h.k.CallEVM(ctx, erc20, types.ModuleAddress, contractAddr, true, "burn", tokens)
  case types.OWNER_EXTERNAL:
   err = h.k.bankKeeper.MintCoins(ctx, types.ModuleName, coins)
  default:
   err = types.ErrUndefinedOwner
  }

  if err != nil {
   h.k.Logger(ctx).Debug(
    "failed to process EVM hook for ER20 -> coin conversion",
    "coin", pair.Denom, "contract", pair.Erc20Address, "error", err.Error(),
   )
   continue
  }

  // Only need last 20 bytes from log.topics
  from := common.BytesToAddress(log.Topics[1].Bytes())
  recipient := sdk.AccAddress(from.Bytes())

  // transfer the tokens from ModuleAccount to sender address
  if err := h.k.bankKeeper.SendCoinsFromModuleToAccount(ctx, types.ModuleName, recipient, coins); err != nil {
   h.k.Logger(ctx).Debug(
    "failed to process EVM hook for ER20 -> coin conversion",
    "tx-hash", receipt.TxHash.Hex(), "log-idx", i,
    "coin", pair.Denom, "contract", pair.Erc20Address, "error", err.Error(),
   )
   continue
  }
 }

 return nil
```

Lastly, register the hook in `app.go`:

```go
app.EvmKeeper = app.EvmKeeper.SetHooks(app.Erc20Keeper)
```

## Events

The `x/evm` module emits the Cosmos SDK events after a state execution.
The EVM module emits events of the relevant transaction fields, as well as the transaction logs (ethereum events).

### MsgEthereumTx

| Type        | Attribute Key      | Attribute Value         |
| ----------- | ------------------ | ----------------------- |
| ethereum_tx | `"amount"`         | `{amount}`              |
| ethereum_tx | `"recipient"`      | `{hex_address}`         |
| ethereum_tx | `"contract"`       | `{hex_address}`         |
| ethereum_tx | `"txHash"`         | `{tendermint_hex_hash}` |
| ethereum_tx | `"ethereumTxHash"` | `{hex_hash}`            |
| ethereum_tx | `"txIndex"`        | `{tx_index}`            |
| ethereum_tx | `"txGasUsed"`      | `{gas_used}`            |
| tx_log      | `"txLog"`          | `{tx_log}`              |
| message     | `"sender"`         | `{eth_address}`         |
| message     | `"action"`         | `"ethereum"`            |
| message     | `"module"`         | `"evm"`                 |

Additionally, the EVM module emits an event during `EndBlock` for the filter query block bloom.

### ABCI

| Type        | Attribute Key | Attribute Value      |
| ----------- | ------------- | -------------------- |
| block_bloom | `"bloom"`     | `string(bloomBytes)` |

## Parameters

The evm module contains the following parameters:

### Params

| Key            | Type        | Default Value   |
|----------------|-------------|-----------------|
| `EVMDenom`     | string      | `"aevmos"`      |
| `EnableCreate` | bool        | `true`          |
| `EnableCall`   | bool        | `true`          |
| `ExtraEIPs`    | []int       | TBD             |
| `ChainConfig`  | ChainConfig | See ChainConfig |

### EVM denom

The evm denomination parameter defines the token denomination
used on the EVM state transitions and gas consumption for EVM messages.

For example, on Ethereum, the `evm_denom` would be `ETH`.
In the case of Evmos, the default denomination is the **atto evmos**.
In terms of precision, `EVMOS` and `ETH` share the same value,
*i.e.* `1 EVMOS = 10^18 atto evmos` and `1 ETH = 10^18 wei`.

:::tip
Note: SDK applications that want to import the EVM module as a dependency
will need to set their own `evm_denom` (i.e not `"aevmos"`).
:::

### Enable Create

The enable create parameter toggles state transitions that use the `vm.Create` function.
When the parameter is disabled, it will prevent all contract creation functionality.

### Enable Transfer

The enable transfer toggles state transitions that use the `vm.Call` function.
When the parameter is disabled, it will prevent transfers between accounts and executing a smart contract call.

### Extra EIPs

The extra EIPs parameter defines the set of activateable Ethereum Improvement Proposals (**[EIPs](https://ethereum.org/en/eips/)**)
on the Ethereum VM `Config` that apply custom jump tables.

:::tip
NOTE: some of these EIPs are already enabled by the chain configuration, depending on the hard fork number.
:::

The supported activateable EIPS are:

- **[EIP 1344](https://eips.ethereum.org/EIPS/eip-1344)**
- **[EIP 1884](https://eips.ethereum.org/EIPS/eip-1884)**
- **[EIP 2200](https://eips.ethereum.org/EIPS/eip-2200)**
- **[EIP 2315](https://eips.ethereum.org/EIPS/eip-2315)**
- **[EIP 2929](https://eips.ethereum.org/EIPS/eip-2929)**
- **[EIP 3198](https://eips.ethereum.org/EIPS/eip-3198)**
- **[EIP 3529](https://eips.ethereum.org/EIPS/eip-3529)**

### Chain Config

The `ChainConfig` is a protobuf wrapper type
that contains the same fields as the go-ethereum `ChainConfig` parameters,
but using `*sdk.Int` types instead of `*big.Int`.

By default, all block configuration fields but `ConstantinopleBlock`, are enabled at genesis (height 0).

#### ChainConfig Defaults

| Name                | Default Value                                                        |
| ------------------- | -------------------------------------------------------------------- |
| HomesteadBlock      | 0                                                                    |
| DAOForkBlock        | 0                                                                    |
| DAOForkSupport      | `true`                                                               |
| EIP150Block         | 0                                                                    |
| EIP150Hash          | `0x0000000000000000000000000000000000000000000000000000000000000000` |
| EIP155Block         | 0                                                                    |
| EIP158Block         | 0                                                                    |
| ByzantiumBlock      | 0                                                                    |
| ConstantinopleBlock | 0                                                                    |
| PetersburgBlock     | 0                                                                    |
| IstanbulBlock       | 0                                                                    |
| MuirGlacierBlock    | 0                                                                    |
| BerlinBlock         | 0                                                                    |
| LondonBlock         | 0                                                                    |
| ArrowGlacierBlock   | 0                                                                    |
| GrayGlacierBlock    | 0                                                                    |
| MergeNetsplitBlock  | 0                                                                    |
| ShanghaiBlock       | 0                                                                    |
| CancunBlock.        | 0                                                                    |

## Client

A user can query and interact with the `evm` module using the CLI, JSON-RPC, gRPC or REST.

### CLI

Find below a list of `evmosd` commands added with the `x/evm` module.
You can obtain the full list by using the `evmosd -h` command.

#### Queries

The `query` commands allow users to query `evm` state.

**`code`**

Allows users to query the smart contract code at a given address.

```bash
evmosd query evm code ADDRESS [flags]
```

```bash
# Example
$ evmosd query evm code 0x7bf7b17da59880d9bcca24915679668db75f9397

# Output
code: "0xef616c92f3cfc9e92dc270d6acff9cea213cecc7020a76ee4395af09bdceb4837a1ebdb5735e11e7d3adb6104e0c3ac55180b4ddf5e54d022cc5e8837f6a4f971b"
```

**`storage`**

Allows users to query storage for an account with a given key and height.

```bash
evmosd query evm storage ADDRESS KEY [flags]
```

```bash
# Example
$ evmosd query evm storage 0x0f54f47bf9b8e317b214ccd6a7c3e38b893cd7f0 0 --height 0

# Output
value: "0x0000000000000000000000000000000000000000000000000000000000000000"
```

#### Transactions

The `tx` commands allow users to interact with the `evm` module.

**`raw`**

Allows users to build cosmos transactions from raw ethereum transaction.

```bash
evmosd tx evm raw TX_HEX [flags]
```

```bash
# Example
$ evmosd tx evm raw 0xf9ff74c86aefeb5f6019d77280bbb44fb695b4d45cfe97e6eed7acd62905f4a85034d5c68ed25a2e7a8eeb9baf1b84

# Output
value: "0x0000000000000000000000000000000000000000000000000000000000000000"
```

### JSON-RPC

For an overview on the JSON-RPC methods and namespaces supported on Evmos,
please refer to [https://docs.evmos.org/develop/api/ethereum-json-rpc/methodsl](https://docs.evmos.org/develop/api/ethereum-json-rpc/methods)

### gRPC

#### Queries

| Verb   | Method                                               | Description                                                               |
| ------ | ---------------------------------------------------- | ------------------------------------------------------------------------- |
| `gRPC` | `ethermint.evm.v1.Query/Account`                     | Get an Ethereum account                                                   |
| `gRPC` | `ethermint.evm.v1.Query/CosmosAccount`               | Get an Ethereum account's Cosmos Address                                  |
| `gRPC` | `ethermint.evm.v1.Query/ValidatorAccount`            | Get an Ethereum account's from a validator consensus Address              |
| `gRPC` | `ethermint.evm.v1.Query/Balance`                     | Get the balance of a the EVM denomination for a single EthAccount.        |
| `gRPC` | `ethermint.evm.v1.Query/Storage`                     | Get the balance of all coins for a single account                         |
| `gRPC` | `ethermint.evm.v1.Query/Code`                        | Get the balance of all coins for a single account                         |
| `gRPC` | `ethermint.evm.v1.Query/Params`                      | Get the parameters of x/evm module                                        |
| `gRPC` | `ethermint.evm.v1.Query/EthCall`                     | Implements the eth_call rpc api                                           |
| `gRPC` | `ethermint.evm.v1.Query/EstimateGas`                 | Implements the eth_estimateGas rpc api                                    |
| `gRPC` | `ethermint.evm.v1.Query/TraceTx`                     | Implements the debug_traceTransaction rpc api                             |
| `gRPC` | `ethermint.evm.v1.Query/TraceBlock`                  | Implements the debug_traceBlockByNumber and debug_traceBlockByHash rpc api |
| `GET`  | `/ethermint/evm/v1/account/{address}`                | Get an Ethereum account                                                   |
| `GET`  | `/ethermint/evm/v1/cosmos_account/{address}`         | Get an Ethereum account's Cosmos Address                                  |
| `GET`  | `/ethermint/evm/v1/validator_account/{cons_address}` | Get an Ethereum account's from a validator consensus Address              |
| `GET`  | `/ethermint/evm/v1/balances/{address}`               | Get the balance of a the EVM denomination for a single EthAccount.        |
| `GET`  | `/ethermint/evm/v1/storage/{address}/{key}`          | Get the balance of all coins for a single account                         |
| `GET`  | `/ethermint/evm/v1/codes/{address}`                  | Get the balance of all coins for a single account                         |
| `GET`  | `/ethermint/evm/v1/params`                           | Get the parameters of x/evm module                                        |
| `GET`  | `/ethermint/evm/v1/eth_call`                         | Implements the eth_call rpc api                                           |
| `GET`  | `/ethermint/evm/v1/estimate_gas`                     | Implements the eth_estimateGas rpc api                                    |
| `GET`  | `/ethermint/evm/v1/trace_tx`                         | Implements the debug_traceTransaction rpc api                             |
| `GET`  | `/ethermint/evm/v1/trace_block`                      | Implements the debug_traceBlockByNumber and debug_traceBlockByHash rpc api |

#### Transactions

| Verb   | Method                            | Description                     |
| ------ | --------------------------------- | ------------------------------- |
| `gRPC` | `ethermint.evm.v1.Msg/EthereumTx` | Submit an Ethereum transactions |
| `POST` | `/ethermint/evm/v1/ethereum_tx`   | Submit an Ethereum transactions |
