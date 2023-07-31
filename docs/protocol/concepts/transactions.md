---
sidebar_position: 12
---
# 交易

交易是指由账户发起的改变区块链状态的操作。
为了有效地进行状态更改，每个交易都会被广播到整个网络。
任何节点都可以向区块链状态机广播一个交易请求；
在此之后，验证者将验证并执行交易，并将结果状态更改传播到网络的其他节点。

为了处理每个交易，网络上的计算资源会被消耗。
因此，“gas”这个概念出现了，它是指验证者处理交易所需的计算量的参考。
用户必须为此计算支付费用，所有交易都需要相关的费用。
这个费用是根据执行交易所需的gas和gas价格计算得出的。

此外，交易还需要使用发送者的私钥进行签名。
这证明了交易只能来自发送者，并且没有被欺诈性地发送。

简而言之，一旦签名的交易被提交到网络，交易的生命周期如下：

- 生成一个交易哈希。
- 将交易广播到网络，并添加到由所有其他待处理网络交易组成的交易池中。
- 验证者必须选择您的交易，并将其包含在一个区块中，以验证交易并将其视为“成功”。

有关交易生命周期的更详细说明，请参见[相应部分](https://docs.cosmos.network/main/basics/tx-lifecycle)。

交易哈希是一个唯一标识符，可以用来检查交易信息，
例如，发出的事件，是否成功等。

交易可能因各种原因而失败。
例如，提供的gas或费用可能不足。
此外，交易验证可能失败。
每个交易都有特定的条件，必须满足这些条件才能被视为有效。
一个常见的验证条件是发送者是交易的签名者。
在这种情况下，如果您发送的交易的发送者地址与签名者的地址不同，
即使费用足够，交易也会失败。

如今，交易不仅可以在提交的链上执行状态转换，还可以在其他区块链上执行交易。
通过[Inter-Blockchain Communication protocol (IBC)](https://ibcprotocol.org/)，可以实现跨链交易。
在下面的章节中可以找到更详细的解释。

## 交易类型

Evmos支持两种交易类型：

1. Cosmos交易
2. Ethereum交易

这是因为Evmos使用了[Cosmos-SDK](https://docs.cosmos.network/main)，并将[Ethereum Virtual Machine](https://ethereum.org/en/developers/docs/evm/)实现为一个模块。
通过这种方式，Evmos提供了Ethereum和Cosmos链的功能和功能的结合，以及更多。

尽管这两种交易类型的大部分信息是相似的，但它们之间存在差异。
一个重要的区别是，Cosmos交易允许在同一笔交易中包含多个消息。
相反，Ethereum交易没有这个可能性。
为了将这两种类型结合起来，Evmos将Ethereum交易实现为包含在[`auth.StdTx`](https://pkg.go.dev/github.com/cosmos/cosmos-sdk/x/auth#StdTx)中的单个[`sdk.Msg`](https://godoc.org/github.com/cosmos/cosmos-sdk/types#Msg)。
所有相关的Ethereum交易信息都包含在这个消息中。
这包括签名、gas、有效载荷等。

在以下章节中可以找到关于这两种类型的更多信息。

### Cosmos交易

在Cosmos链上，交易由存储在上下文和`sdk.Msg`中的元数据组成，
通过模块的Protobuf [Msg service](https://docs.cosmos.network/main/building-modules/msg-services)触发模块内的状态变化。

当用户想要与应用程序交互并进行状态变更（例如发送硬币）时，他们创建交易。
Cosmos交易可以包含多个`sdk.Msg`。
在将交易广播到网络之前，必须使用与相应账户关联的私钥对每个消息进行签名。

一个 Cosmos 交易包括以下信息：

- `Msgs`：一个包含 `sdk.Msg` 的消息数组
- `GasLimit`：用户选择的计算所需支付的 gas 的选项
- `FeeAmount`：用户愿意支付的最大费用金额
- `TimeoutHeight`：交易有效的区块高度
- `Signatures`：所有签名者的签名数组
- `Memo`：与交易一起发送的注释或备注

要提交 Cosmos 交易，用户必须使用提供的客户端之一。

### 以太坊交易

以太坊交易是指由外部拥有的账户（由人类管理）发起的操作，而不是内部智能合约调用。以太坊交易会改变 EVM 的状态，因此必须广播到整个网络。

以太坊交易还需要支付费用，称为 `gas`。([EIP-1559](https://eips.ethereum.org/EIPS/eip-1559)) 引入了基础费用的概念，以及作为矿工激励的优先费用，用于促使矿工将特定交易包含在区块中。

以太坊交易有几个类别：

- 普通交易：从一个账户发送到另一个账户的交易
- 合约部署交易：没有 `to` 地址的交易，合约代码发送到 `data` 字段中
- 合约执行交易：与已部署智能合约交互的交易，`to` 地址是智能合约地址

一个以太坊交易包括以下信息：

- `recipient`：接收地址
- `signature`：发送者的签名
- `nonce`：账户的交易计数器
- `value`：要转移的 ETH 金额（以 wei 为单位）
- `data`：包含任意数据。在部署智能合约或调用智能合约方法时使用
- `gasLimit`：要消耗的最大 gas 量
- `maxPriorityFeePerGas`：作为奖励给验证者的最大 gas 量
- `maxFeePerGas`：要支付的交易的最大 gas 量

有关以太坊交易和交易生命周期的更多信息，请[点击此处](https://ethereum.org/en/developers/docs/transactions/)。

Evmos支持以下以太坊交易。

:::tip
**注意**: 默认情况下，不支持未保护的旧版交易。
:::

- 动态费用交易（[EIP-1559](https://eips.ethereum.org/EIPS/eip-1559)）
- 访问列表交易（[EIP-2930](https://eips.ethereum.org/EIPS/eip-2930)）
- 旧版交易（[EIP-2718](https://eips.ethereum.org/EIPS/eip-2718)）

Evmos通过将以太坊交易封装在`sdk.Msg`中来处理以太坊交易。
Evmos通过使用`MsgEthereumTx`实现这一目标。
该消息将以太坊交易封装为SDK消息，并包含必要的交易数据字段。

关于`MsgEthereumTx`的一个注意事项是，它同时实现了`sdk.Msg`和`sdk.Tx`接口
（通常SDK消息只实现前者，而后者是一组打包在一起的消息）。
之所以如此，是因为`MsgEthereumTx`不能包含在`auth.StdTx`中
（SDK的标准交易类型），因为它使用来自Geth的以太坊逻辑执行燃气和费用检查
而不是在auth模块`AnteHandler`上执行的Cosmos SDK检查。

#### 以太坊交易类型

Evmos的[Go Ethereum](https://github.com/ethereum/go-ethereum/blob/b946b7a13b749c99979e312c83dce34cac8dd7b1/core/types/transaction.go#L43-L48)实现中使用了三种交易类型，这些类型来自以太坊改进提案（EIPs）：

1. LegacyTxType（EIP-155）：
LegacyTxType代表Ethereum Improvement Proposal（EIP）155之前存在的原始交易格式。
这些交易不包括链ID，这使它们容易受到重放攻击的影响。EIP-155引入了链ID来解决这个问题，
链ID唯一标识特定的以太坊链，以防止跨链重放攻击。

2. AccessListTxType（EIP-2930）：
AccessListTxType是作为柏林升级的一部分与EIP-2930一起引入的。这种新的交易类型允许用户指定访问列表，
即计划访问的地址和存储键的列表。访问列表的主要目标是减轻EIP-2929引入的一些增加的燃气成本，
EIP-2929增加了状态访问操作的燃气成本，以提高防止服务拒绝（DoS）攻击的能力。通过指定访问列表，
用户可以避免在同一交易中对相同地址和存储键进行后续访问时支付更高的燃气成本。

3. 动态费用交易类型（EIP-1559）：
动态费用交易类型是在伦敦升级中作为EIP-1559的一部分引入的。这种交易类型对以太坊的费用市场带来了重大变化，旨在使燃气费更可预测，并改善用户体验。
EIP-1559交易包括两个主要组成部分：基础费用和优先费用（或小费）。基础费用由网络算法确定，而优先费用由用户设置以激励矿工包含他们的交易。
基础费用被销毁，有效地减少了整体以太币的供应，而优先费用作为对矿工工作的奖励。
动态费用交易类型允许更可预测和高效的燃气费管理。

这些交易类型代表了以太坊网络的持续演进和改进，有助于解决与可扩展性、安全性和用户体验相关的挑战。

### 跨链交易

跨链交易是指在两个或多个不同的区块链网络之间转移数字资产或数据。

每个区块链网络都有自己独特的协议和数据结构，直接从一个区块链转移资产或数据到另一个区块链是困难的。
通过使用中介机制或协议，跨链交易允许在不同的区块链之间转移资产和数据。

其中一种机制是跨链桥，它充当不同区块链之间的连接器，实现资产或数据的转移。
跨链桥通常需要某种形式的信任或共识机制，以确保交易的安全性和完整性。

另一种可能性是使用[IBC（Inter-Blockchain Communication）](https://ibcprotocol.org/)协议。
要使用IBC进行跨链交易，用户需要：

- 选择用户希望在之间转移资产或数据的源区块链和目标区块链网络。
- 确保两个区块链网络都已实施IBC协议。
- 确保通过IBC建立了两个区块链网络之间的连接和通道。
- 启动资产或数据的转移：这是通过从源区块链向目标区块链通过IBC通道发送交易来完成的。

跨链交易在不同的区块链网络和应用程序不断增长的数量中变得越来越重要。
它们实现了不同区块链网络的互操作性，使数字资产和数据的转移更加灵活和高效。

## 交易收据

交易收据显示以太坊客户端返回的数据，表示特定交易的结果，
包括交易的哈希值、所在区块的编号、使用的燃料量，
以及在部署智能合约的情况下，合约的地址。
此外，它还包括智能合约中发出的自定义事件的信息。

收据包含以下信息：

- `transactionHash`：交易的哈希值。
- `transactionIndex`：交易在区块中的索引位置。
- `blockHash`：包含此交易的区块的哈希值。
- `blockNumber`：包含此交易的区块编号。
- `from`：发送者的地址。
- `to`：接收者的地址。如果是合约创建交易，则为 null。
- `cumulativeGasUsed`：此交易在区块中执行时使用的总燃料量。
- `effectiveGasPrice`：每单位燃料支付的基本费用和小费的总和。
- `gasUsed`：此特定交易使用的燃料量。
- `contractAddress`：如果交易是合约创建，则为创建的合约地址；否则为 null。
- `logs`：由此交易生成的日志对象数组。
- `logsBloom`：用于轻客户端快速检索相关日志的布隆过滤器。
- `type`：交易类型的整数，0x00 表示传统交易，0x01 表示访问列表类型，0x02 表示动态费用类型。
- `root`：交易状态根（Byzantium 之前）。
- `status`：1 表示成功，0 表示失败。


---
sidebar_position: 12
---
# Transactions

A transaction refers to an action initiated by an account which changes the state of the blockchain.
To effectively perform the state change, every transaction is broadcasted to the whole network.
Any node can broadcast a request for a transaction to be executed on the blockchain state machine;
after this happens, a validator will validate, execute the transaction and propagate the resulting state change
to the rest of the network.

To process every transaction, computation resources on the network are consumed.
Thus, the concept of "gas" arises as a reference to the computation required to process the transaction by a validator.
Users have to pay a fee for this computation, all transactions require an associated fee.
This fee is calculated based on the gas required to execute the transaction and the gas price.

Additionally, a transaction needs to be signed using the sender's private key.
This proves that the transaction could only have come from the sender and was not sent fraudulently.

In a nutshell, the transaction lifecycle once a signed transaction is submitted to the network is the following:

- A transaction hash is cryptographically generated.
- The transaction is broadcasted to the network and added to a transaction pool consisting of all other pending network transactions.
- A validator must pick your transaction and include it in a block in order to verify the transaction and consider it "successful".

For a more detailed explanation of the transaction lifecyle, see [the corresponding section](https://docs.cosmos.network/main/basics/tx-lifecycle).

The transaction hash is a unique identifier and can be used to check transaction information,
for example, the events emitted, if was successful or not.

Transactions can fail for various reasons.
For example, the provided gas or fees may be insufficient.
Also, the transaction validation may fail.
Each transaction has specific conditions that must fullfil to be considered valid.
A widespread validation is that the sender is the transaction signer.
In such a case, if you send a transaction where the sender address is different than the signer's address,
the transation will fail, even if the fees are sufficient.

Nowadays, transactions can not only perform state transitions on the chain in which are submitted,
but also can execute transactions on another blockchains.
Interchain transactions are possible through the [Inter-Blockchain Communication protocol (IBC)](https://ibcprotocol.org/).
Find a more detailed explanation on the section below.

## Transaction Types

Evmos supports two transaction types:

1. Cosmos transactions
2. Ethereum transactions

This is possible because Evmos uses the [Cosmos-SDK](https://docs.cosmos.network/main)
and implements the [Ethereum Virtual Machine](https://ethereum.org/en/developers/docs/evm/) as a module.
In this way, Evmos provides the features and functionalities of Ethereum and Cosmos chains combined, and more.

Although most of the information included on both of these transaction types is similar,
there are differences among them.
An important difference, is that Cosmos transactions allow multiple messages on the same transaction.
Conversely, Ethereum transactions don't have this possibility.
To bring these two types together, Evmos implements Ethereum transactions as a single [`sdk.Msg`](https://godoc.org/github.com/cosmos/cosmos-sdk/types#Msg)
contained in an [`auth.StdTx`](https://pkg.go.dev/github.com/cosmos/cosmos-sdk/x/auth#StdTx).
All relevant Ethereum transaction information is contained in this message.
This includes the signature, gas, payload, etc.

Find more information about these two types on the following sections.

### Cosmos Transactions

On Cosmos chains, transactions are comprised of metadata held in contexts and `sdk.Msg`s
that trigger state changes within a module through the module's Protobuf [Msg service](https://docs.cosmos.network/main/building-modules/msg-services).

When users want to interact with an application and make state changes (e.g. sending coins), they create transactions.
Cosmos transactions can have multiple `sdk.Msg`s.
Each of these must be signed using the private key associated with the appropriate account(s),
before the transaction is broadcasted to the network.

A Cosmos transaction includes the following information:

- `Msgs`: an array of msgs (`sdk.Msg`)
- `GasLimit`: option chosen by the users for how to calculate how much gas they will need to pay
- `FeeAmount`: max amount user is willing to pay in fees
- `TimeoutHeight`: block height until which the transaction is valid
- `Signatures`: array of signatures from all signers of the tx
- `Memo`: a note or comment to send with the transaction

To submit a Cosmos transaction, users must use one of the provided clients.

### Ethereum Transactions

Ethereum transactions refer to actions initiated by EOAs (externally-owned accounts, managed by humans),
rather than internal smart contract calls. Ethereum transactions transform the state of the EVM
and therefore must be broadcasted to the entire network.

Ethereum transactions also require a fee, known as `gas`. ([EIP-1559](https://eips.ethereum.org/EIPS/eip-1559))
introduced the idea of a base fee, along with a priority fee which serves as an incentive
for miners to include specific transactions in blocks.

There are several categories of Ethereum transactions:

- regular transactions: transactions from one account to another
- contract deployment transactions: transactions without a `to` address, where the contract code is sent in the `data` field
- execution of a contract: transactions that interact with a deployed smart contract,
where the `to` address is the smart contract address

An Ethereum transaction includes the following information:

- `recipient`: receiving address
- `signature`: sender's signature
- `nonce`: counter of tx number from account
- `value`: amount of ETH to transfer (in wei)
- `data`: include arbitrary data. Used when deploying a smart contract or making a smart contract method call
- `gasLimit`: max amount of gas to be consumed
- `maxPriorityFeePerGas`: mas gas to be included as tip to validators
- `maxFeePerGas`: max amount of gas to be paid for tx

For more information on Ethereum transactions and the transaction lifecycle, [go here](https://ethereum.org/en/developers/docs/transactions/).

Evmos supports the following Ethereum transactions.

:::tip
**Note**: Unprotected legacy transactions are not supported by default.
:::

- Dynamic Fee Transactions ([EIP-1559](https://eips.ethereum.org/EIPS/eip-1559))
- Access List Transactions ([EIP-2930](https://eips.ethereum.org/EIPS/eip-2930))
- Legacy Transactions ([EIP-2718](https://eips.ethereum.org/EIPS/eip-2718))

Evmos is capable of processing Ethereum transactions by wrapping them on a `sdk.Msg`.
Evmos achieves this by using the `MsgEthereumTx`.
This message encapsulates an Ethereum transaction as an SDK message and contains the necessary transaction data fields.

One remark about the `MsgEthereumTx` is that it implements both the `sdk.Msg` and `sdk.Tx` interfaces
(generally SDK messages only implement the former, while the latter is a group of messages bundled together).
The reason of this, is because the `MsgEthereumTx` must not be included in a `auth.StdTx`
(SDK's standard transaction type) as it performs gas and fee checks using the Ethereum logic
from Geth instead of the Cosmos SDK checks done on the auth module `AnteHandler`.

#### Ethereum Tx Type

There are three types of transaction types used in Evmos's [Go Ethereum](https://github.com/ethereum/go-ethereum/blob/b946b7a13b749c99979e312c83dce34cac8dd7b1/core/types/transaction.go#L43-L48)
implementation that came from Ethereum Improvement Proposals(EIPs):

1. LegacyTxType (EIP-155):
The LegacyTxType represents the original transaction format that existed before Ethereum Improvement Proposal (EIP) 155.
These transactions do not include a chain ID, which makes them vulnerable to replay attacks. EIP-155 was introduced
to solve this problem by incorporating a chain ID, which uniquely identifies a specific Ethereum chain to prevent
cross-chain replay attacks.

2. AccessListTxType (EIP-2930):
AccessListTxType was introduced with EIP-2930 as part of the Berlin upgrade. This new transaction type allows users to
specify an access list – a list of addresses and storage keys that the transaction plans to access. The primary goal
of access lists is to mitigate some of the gas cost increases introduced with EIP-2929, which increased gas costs
for state access operations to improve denial-of-service (DoS) attack resistance. By specifying an access list, users
can avoid paying higher gas costs for subsequent accesses to the same addresses and storage keys within the same transaction.

3. DynamicFeeTxType (EIP-1559):
DynamicFeeTxType was introduced with EIP-1559 as part of the London upgrade. This transaction type brought significant
changes to Ethereum's fee market, with the aim of making gas fees more predictable and improving user experience.
EIP-1559 transactions include two main components: a base fee and a priority fee (or tip). The base fee is algorithmically
determined by the network, while the priority fee is set by users to incentivize miners to include their transaction.
The base fee is burned, effectively reducing the overall ETH supply, while the priority fee goes to miners as a reward
for their work. DynamicFeeTxType transactions allow for more predictable and efficient gas fee management.

These transaction types represent Ethereum's continuous evolution and improvements to its network, helping address
challenges related to scalability, security, and user experience.

### Interchain Transactions

Interchain transactions refer to the transfer of digital assets or data between two or more different blockchain networks.

Each blockchain network has its own unique protocol and data structure,
making it difficult to directly transfer assets or data from one blockchain to another.
Interchain transactions allow for the transfer of assets and data between different blockchains
by using intermediary mechanisms or protocols.

One such mechanism is a cross-chain bridge, which acts as a connector between different blockchains,
enabling the transfer of assets or data.
Cross-chain bridges typically require some form of trust or
consensus mechanism to ensure the security and integrity of the transaction.

Another possibility is to use the [IBC (Inter-Blockchain Communication)](https://ibcprotocol.org/) protocol.
To make an interchain transaction using IBC a user needs to:

- Choose the source and destination blockchain networks that the user wants to transfer assets or data between.
- Ensure that both blockchain networks have implemented the IBC protocol
- Ensure there's a connection and channel established between the two blockchain networks using IBC
- Initiate the transfer of assets or data: this is done by sending a transaction from the source blockchain
to the destination blockchain through the IBC channel

Interchain transactions are becoming increasingly important as the number of different blockchain networks
and applications continues to grow.
They enable the interoperability of different blockchain networks,
allowing for greater flexibility and efficiency in the transfer of digital assets and data.

## Transaction Receipts

A transaction receipt shows data returned by an Ethereum client to represent the result of a particular transaction,
including a hash of the transaction, its block number, the amount of gas used, and,
in case of deployment of a smart contract, the address of the contract.
Additionally, it includes custom information from the events emitted in the smart contract.

A receipt contains the following information:

- `transactionHash` : hash of the transaction.
- `transactionIndex`: integer of the transactions index position in the block.
- `blockHash`: hash of the block where this transaction was in.
- `blockNumber`: block number where this transaction was in.
- `from`: address of the sender.
- `to`: address of the receiver. null when its a contract creation transaction.
- `cumulativeGasUsed` : The total amount of gas used when this transaction was executed in the block.
- `effectiveGasPrice` : The sum of the base fee and tip paid per unit of gas.
- `gasUsed` : The amount of gas used by this specific transaction alone.
- `contractAddress` : The contract address created, if the transaction was a contract creation, otherwise null.
- `logs`: Array of log objects, which this transaction generated.
- `logsBloom`: Bloom filter for light clients to quickly retrieve related logs.
- `type`: integer of the transaction type, 0x00 for legacy transactions, 0x01 for access list types,
0x02 for dynamic fees. It also returns either.
- `root` : transaction stateroot (pre Byzantium)
- `status`: either 1 (success) or 0 (failure)
