# `revenue`

## 摘要

本文档规定了 Evmos Hub 的内部 `x/revenue` 模块。

`x/revenue` 模块使 Evmos Hub 能够支持将交易费用分配给区块提议者和智能合约部署者。
作为[Evmos Token Model](https://evmos.blog/the-evmos-token-model-edc07014978b)的一部分，
该机制旨在通过为智能合约部署者提供一种新的稳定收入来源，
增加 Evmos Hub 的采用率。
开发人员可以注册他们的智能合约，每当有人与注册的智能合约进行交互时，
合约部署者或其指定的提现账户将获得一部分交易费用。

所有注册的智能合约共同组成了 Evmos dApp Store：
通过内置的共享费用收入模型向开发人员和网络运营商支付他们的服务费用。

## 目录

1. **[概念](#概念)**
2. **[状态](#状态)**
3. **[状态转换](#状态转换)**
4. **[交易](#交易)**
5. **[钩子](#钩子)**
6. **[事件](#事件)**
7. **[参数](#参数)**
8. **[客户端](#客户端)**
9. **[未来改进](#未来改进)**

## 概念

### Evmos dApp Store

Evmos dApp Store 是一种按交易收入计费的模型，允许开发人员在 Evmos 上部署他们的去中心化应用程序（dApps）并获得报酬。
开发人员每次用户在 dApp Store 中与他们的 dApp 进行交互时都会获得收入，从而获得稳定的收入。
用户可以在 dApp Store 中发现新的应用程序，并支付用于资助 dApp 收入的交易费用。
这种以交易费用为代价的 dApp 服务与价值回报交换是通过 `x/revenue` 模块实现的。

### 注册

开发人员通过注册其应用程序的智能合约在 dApp Store 中注册其应用程序。
开发人员可以通过提交签名的交易来注册任何合约。
此交易的签名者必须与合约的部署者地址匹配，
以使注册成功。
交易成功执行后，
当用户与注册的合约进行交互时，开发人员将开始收到支付的交易费用的一部分。

:::tip
**注意**：如果您的合约是开发者项目的一部分，请确保部署合约的账户（或部署合约的工厂）是该项目拥有的账户。
这样可以避免个人部署者离开项目后可能变得恶意。
:::

### 费用分配

如上所述，开发者在注册其合约后将获得交易费的一部分。
为了了解交易费是如何分配的，我们详细看一下以下两个方面：

* 仅有符合条件的[EVM交易](evm.md)（`MsgEthereumTx`）才有资格。
  Cosmos SDK交易目前不符合条件。
* 注册工厂合约（由其他合约部署的智能合约）需要识别原始合约的部署者。
  这是通过地址派生来完成的。

#### EVM交易费

用户支付交易费以与使用EVM与智能合约进行交互。
当执行交易时，整个费用金额（`gasLimit * gasPrice`）将发送到`FeeCollector`模块账户
在[Cosmos SDK AnteHandler](https://docs.cosmos.network/main/modules/auth#antehandlers)执行期间。
在EVM执行交易后，用户将获得退款金额为`(gasLimit - gasUsed) * gasPrice`。
因此，用户为执行支付了总交易费`txFee = gasUsed * gasPrice`。

此交易费根据`x/revenue`模块的参数`DeveloperShares`和`ValidatorShares`分配给开发者和验证者。
此分配通过EVM的[`PostTxProcessing` Hook](#hooks)处理。

#### 地址派生

dApp开发者可能会使用[工厂模式](https://en.wikipedia.org/wiki/Factory_method_pattern)
通过智能合约来实现其应用逻辑。
在这种情况下，智能合约可以由外部拥有的账户（[EOA](https://ethereum.org/en/whitepaper/#ethereum-accounts)）部署，也可以通过其他合约部署。

在这两种情况下，费用分配需要识别一个部署者地址，该地址是一个EOA地址，除非在注册期间由合约部署者设置了提现地址以接收已注册智能合约的交易费用。如果没有设置提现地址，则默认为部署者的地址。

通过地址派生来识别部署者地址。在注册智能合约时，部署者提供一个非空数组，用于[派生合约地址](https://github.com/ethereum/go-ethereum/blob/d8ff53dfb8a516f47db37dbc7fd7ad18a1e8a125/crypto/crypto.go#L107-L111)：

- 如果`MyContract`直接由`DeployerEOA`部署，在具有nonce `5`的事务中发送，则非空数组为`[5]`。
- 如果合约是通过智能合约创建的，通过`CREATE`操作码，我们需要提供从创建路径中的所有非空数组。例如，如果`DeployerEOA`使用nonce `5`部署了一个名为`FactoryA`的智能合约。然后，`DeployerEOA`通过一个事务向`FactoryA`发送一个事务，通过该事务创建了一个名为`FactoryB`的智能合约。如果我们假设`FactoryB`是由`FactoryA`创建的第二个合约，则`FactoryA`的nonce为`2`。然后，`DeployerEOA`通过一个事务向`FactoryB`合约发送一个事务，通过该事务创建了`MyContract`。如果这是由`FactoryB`创建的第一个合约，则nonce为`1`。现在，我们有了一个地址派生路径：`DeployerEOA` -> `FactoryA` -> `FactoryB` -> `MyContract`。为了能够验证`DeployerEOA`可以注册`MyContract`，我们需要提供以下非空数组：`[5, 2, 1]`。

:::tip
**注意**：即使`MyContract`是由与`DeployerEOA`不同的账户通过`FactoryB`创建的事务创建的，只有`DeployerEOA`才能注册`MyContract`。
:::

## 状态

### 状态对象

`x/revenue`模块在状态中保留以下对象：

| 状态对象              | 描述                                 | 键                                                               | 值              | 存储 |
| :-------------------- | :------------------------------------ | :---------------------------------------------------------------- | :----------------- | :---- |
| `Revenue`            | 费用分配的字节码                     | `[]byte{1} + []byte(contract_address)`                            | `[]byte{revenue}` | KV    |
| `DeployerRevenues`   | 由部署者地址的合约字节码 | `[]byte{2} + []byte(deployer_address) + []byte(contract_address)` | `[]byte{1}`        | KV    |
| `WithdrawerRevenues` | 由提现地址的合约字节码 | `[]byte{3} + []byte(withdraw_address) + []byte(contract_address)` | `[]byte{1}`        | KV    |

#### 收入

收入定义了为给定智能合约的所有者组织费用分配条件的实例。

```go
type Revenue struct {
	// hex address of registered contract
	ContractAddress string `protobuf:"bytes,1,opt,name=contract_address,json=contractAddress,proto3" json:"contract_address,omitempty"`
	// bech32 address of contract deployer
	DeployerAddress string `protobuf:"bytes,2,opt,name=deployer_address,json=deployerAddress,proto3" json:"deployer_address,omitempty"`
	// bech32 address of account receiving the transaction fees it defaults to
	// deployer_address
	WithdrawerAddress string `protobuf:"bytes,3,opt,name=withdrawer_address,json=withdrawerAddress,proto3" json:"withdrawer_address,omitempty"`
}
```

#### 合约地址

`ContractAddress` 定义了已注册的费用分配的合约地址。

#### 部署者地址

`DeployerAddress` 是已注册合约的 EOA 地址。

#### 提款者地址

`WithdrawerAddress` 是接收已注册合约的交易费用的地址。

### 创世状态

`x/revenue` 模块的 `GenesisState` 定义了从先前导出的高度初始化链所需的状态。
它包含了模块参数和已注册合约的收入：

```go
// GenesisState defines the module's genesis state.
type GenesisState struct {
	// module parameters
	Params Params `protobuf:"bytes,1,opt,name=params,proto3" json:"params"`
	// active registered contracts for fee distribution
	Revenues []Revenue `protobuf:"bytes,2,rep,name=revenues,json=revenues,proto3" json:"revenues"`
}

```

## 状态转换

`x/revenue` 模块允许三种类型的状态转换：
`RegisterRevenue`、`UpdateRevenue` 和 `CancelRevenue`。
通过 [Hooks](#hooks) 处理交易费用分配的逻辑。

#### 注册费用分配

开发者注册一个合约以接收交易费用，定义合约地址、用于 [地址派生](#concepts) 的非ces数组，
以及一个可选的提款地址以接收费用。
如果未设置提款地址，则默认将费用发送到部署者地址。

1. 用户提交 `RegisterRevenue` 来注册一个合约地址，
   以及他们希望将费用发送到的提款地址
2. 检查以下条件是否满足：
    1. `x/revenue` 模块已启用
    2. 合约之前未注册过
    3. 部署者有一个有效的账户（至少进行过一次交易）且不是智能合约
    4. 存在与合约地址对应的账户，且具有非空的字节码
    5. 可以使用 `CREATE` 操作从部署者的地址和提供的非ces派生合约地址
    6. 合约已经部署
3. 存储提供的费用的实例。

在注册后发送到已注册合约的所有交易的费用将根据全局的 `DeveloperShares` 参数分配给开发者。

#### 更新费用分配

开发者更新已注册合约的提现地址，定义合约地址和新的提现地址。

1. 用户提交一个 `UpdateRevenue` 交易。
2. 检查以下条件是否满足：
    1. `x/revenue` 模块已启用。
    2. 合约已注册。
    3. 交易签名者与合约部署者相同。
3. 使用新的提现地址更新费用。
   注意，如果提现地址为空或与部署者地址相同，则提现地址设置为 `""`。

更新后，开发者将在新的提现地址上收到费用。

#### 取消费用分配

开发者取消已注册合约的费用接收，定义合约地址。

1. 用户提交一个 `CancelRevenue` 交易。
2. 检查以下条件是否满足：
    1. `x/revenue` 模块已启用。
    2. 合约已注册。
    3. 交易签名者与合约部署者相同。
3. 从存储中移除费用。

开发者将不再从发送到该合约的交易中收到费用。

## 交易

本节定义了导致前一节中定义的状态转换的 `sdk.Msg` 具体类型。

### `MsgRegisterRevenue`

定义了由开发者签名的交易，用于注册用于交易费用分配的合约。
发送者必须是与合约部署者地址对应的 EOA。

```go
type MsgRegisterRevenue struct {
	// contract hex address
	ContractAddress string `protobuf:"bytes,1,opt,name=contract_address,json=contractAddress,proto3" json:"contract_address,omitempty"`
	// bech32 address of message sender, must be the same as the origin EOA
	// sending the transaction which deploys the contract
	DeployerAddress string `protobuf:"bytes,2,opt,name=deployer_address,json=deployerAddress,proto3" json:"deployer_address,omitempty"`
	// bech32 address of account receiving the transaction fees
	WithdrawerAddress string `protobuf:"bytes,3,opt,name=withdraw_address,json=withdrawerAddress,proto3" json:"withdraw_address,omitempty"`
	// array of nonces from the address path, where the last nonce is
	// the nonce that determines the contract's address - it can be an EOA nonce
	// or a factory contract nonce
	Nonces []uint64 `protobuf:"varint,4,rep,packed,name=nonces,proto3" json:"nonces,omitempty"`
}
```

如果消息内容的无状态验证失败，则会出现以下情况：

- 合约十六进制地址无效
- 合约十六进制地址为零
- 部署者 bech32 地址无效
- 提现 bech32 地址无效
- Nonces 数组为空

### `MsgUpdateRevenue`

定义了由开发者签名的交易，用于更新已注册用于交易费用分配的合约的提现地址。
发送者必须是与合约部署者地址对应的 EOA。

```go
type MsgUpdateRevenue struct {
	// contract hex address
	ContractAddress string `protobuf:"bytes,1,opt,name=contract_address,json=contractAddress,proto3" json:"contract_address,omitempty"`
	// deployer bech32 address
	DeployerAddress string `protobuf:"bytes,2,opt,name=deployer_address,json=deployerAddress,proto3" json:"deployer_address,omitempty"`
	// new withdraw bech32 address for receiving the transaction fees
	WithdrawerAddress string `protobuf:"bytes,3,opt,name=withdraw_address,json=withdrawerAddress,proto3" json:"withdraw_address,omitempty"`
}
```

如果消息内容的无状态验证失败，则会出现以下情况：

- 合约十六进制地址无效
- 合约十六进制地址为零
- 部署者 bech32 地址无效
- 提现 bech32 地址无效
- 提现 bech32 地址与部署者地址相同

### `MsgCancelRevenue`（取消收入消息）

定义了由开发者签名的交易，用于移除已注册合约的信息。
此智能合约将不再分发交易费用给开发者。
发送者必须是与合约部署者地址对应的EOA。

```go
type MsgCancelRevenue struct {
	// contract hex address
	ContractAddress string `protobuf:"bytes,1,opt,name=contract_address,json=contractAddress,proto3" json:"contract_address,omitempty"`
	// deployer bech32 address
	DeployerAddress string `protobuf:"bytes,2,opt,name=deployer_address,json=deployerAddress,proto3" json:"deployer_address,omitempty"`
}
```

如果以下情况发生，消息内容的无状态验证将失败：

- 合约十六进制地址无效
- 合约十六进制地址为零
- 部署者bech32地址无效

## 钩子函数

费用模块实现了`x/evm`模块中的一个交易钩子，以便在开发者和验证者之间分发费用。

### EVM钩子

[`PostTxProcessing` EVM钩子](evm.md#hooks)在每个成功的EVM交易之后执行自定义逻辑。
用户为交易执行支付的所有费用都会在`AnteHandler`执行期间发送到`FeeCollector`模块账户，
然后再分发给开发者和验证者。

如果`x/revenue`模块被禁用或EVM交易的目标是未注册的合约，
EVM钩子将返回`nil`，不执行任何操作。
在这种情况下，交易费用的100%将保留在`FeeCollector`模块中，以分发给区块提议者。

如果`x/revenue`模块被启用且EVM交易的目标是已注册的合约，
EVM钩子将向该合约设置的提现地址或合约部署者发送交易费用的一部分（由用户支付）。

1. 用户向智能合约提交EVM交易（`MsgEthereumTx`）并成功执行交易
2. 检查以下内容
    * 是否启用了费用模块
    * 智能合约是否已注册以接收费用
3. 根据`DeveloperShares`参数计算开发者费用。
   初始交易消息包括用户支付的燃料价格和交易收据，
   交易收据包括交易使用的燃料。

   ```go
    devFees := receipt.GasUsed * msg.GasPrice * params.DeveloperShares
    ```

4. Transfer developer fee from the `FeeCollector` (Cosmos SDK `auth` module account)
   to the registered withdraw address for that contract.
   If there is no withdraw address, fees are sent to contract deployer's address.
5. Distribute the remaining amount in the `FeeCollector` to validators according to the
   [SDK  Distribution Scheme](https://docs.cosmos.network/main/modules/distribution#the-distribution-scheme).


## Events

The `x/revenue` module emits the following events:

### Register Fee Split

| Type                 | Attribute Key          | Attribute Value           |
| :------------------- | :--------------------- | :------------------------ |
| `register_revenue` | `"contract"`           | `{msg.ContractAddress}`   |
| `register_revenue` | `"sender"`             | `{msg.DeployerAddress}`   |
| `register_revenue` | `"withdrawer_address"` | `{msg.WithdrawerAddress}` |

### Update Fee Split

| Type               | Attribute Key          | Attribute Value           |
| :----------------- | :--------------------- | :------------------------ |
| `update_revenue` | `"contract"`           | `{msg.ContractAddress}`   |
| `update_revenue` | `"sender"`             | `{msg.DeployerAddress}`   |
| `update_revenue` | `"withdrawer_address"` | `{msg.WithdrawerAddress}` |

### Cancel Fee Split

| Type               | Attribute Key | Attribute Value         |
| :----------------- | :------------ | :---------------------- |
| `cancel_revenue` | `"contract"`  | `{msg.ContractAddress}` |
| `cancel_revenue` | `"sender"`    | `{msg.DeployerAddress}` |

## Parameters

The fees module contains the following parameters:

| Key                        | Type    | Default Value |
| :------------------------- | :------ | :------------ |
| `EnableRevenue`           | bool    | `true`        |
| `DeveloperShares`          | sdk.Dec | `50%`         |
| `AddrDerivationCostCreate` | uint64  | `50`          |

### Enable Revenue Module

The `EnableRevenue` parameter toggles all state transitions in the module.
When the parameter is disabled, it will prevent any transaction fees from being distributed to contract deployers
and it will disallow contract registrations, updates or cancellations.

#### Developer Shares Amount

The `DeveloperShares` parameter is the percentage of transaction fees that is sent to the contract deployers.

#### Address Derivation Cost with CREATE opcode

The `AddrDerivationCostCreate` parameter is the gas value charged
for performing an address derivation in the contract registration process.
A flat gas fee is charged for each address derivation iteration.
We allow a maximum number of 20 iterations, and therefore a maximum number of 20 nonces can be given
for deriving the smart contract address from the deployer's address.


## Clients

### CLI

Find below a list of  `evmosd` commands added with the  `x/revenue` module.
You can obtain the full list by using the `evmosd -h` command.
A CLI command can look like this:

```bash
evmosd query revenue params
```

#### 查询

| 命令               | 子命令                | 描述                                 |
| :----------------- | :--------------------- | :--------------------------------------- |
| `query` `revenue` | `params`               | 获取收入参数                          |
| `query` `revenue` | `contract`             | 获取给定合约的收入                  |
| `query` `revenue` | `contracts`            | 获取所有收入                       |
| `query` `revenue` | `deployer-contracts`   | 获取给定部署者的所有收入           |
| `query` `revenue` | `withdrawer-contracts` | 获取给定提现者的所有收入 |

#### 交易

| 命令              | 子命令     | 描述                                       |
| :-------------- | :--------- | :----------------------------------------- |
| `tx` `revenue` | `register` | 注册合约以接收收入                           |
| `tx` `revenue` | `update`   | 更新合约的提款地址                         |
| `tx` `revenue` | `cancel`   | 移除合约的收入                             |

### gRPC

#### 查询

| 动词    | 方法                                              | 描述                                       |
| :----- | :------------------------------------------------ | :----------------------------------------- |
| `gRPC` | `evmos.revenue.v1.Query/Params`                  | 获取收入参数                                 |
| `gRPC` | `evmos.revenue.v1.Query/Revenue`                | 获取给定合约的收入                           |
| `gRPC` | `evmos.revenue.v1.Query/Revenues`               | 获取所有收入                               |
| `gRPC` | `evmos.revenue.v1.Query/DeployerRevenues`       | 获取给定部署者的所有收入                     |
| `gRPC` | `evmos.revenue.v1.Query/WithdrawerRevenues`     | 获取给定提款者的所有收入                     |
| `GET`  | `/evmos/revenue/v1/params`                       | 获取收入参数                                 |
| `GET`  | `/evmos/revenue/v1/revenues/{contract_address}`  | 获取给定合约的收入                           |
| `GET`  | `/evmos/revenue/v1/revenues`                    | 获取所有收入                               |
| `GET`  | `/evmos/revenue/v1/revenues/{deployer_address}` | 获取给定部署者的所有收入                     |
| `GET`  | `/evmos/revenue/v1/revenues/{withdraw_address}` | 获取给定提款者的所有收入                     |

#### 交易

| 动词    | 方法                                     | 描述                                       |
| :----- | :----------------------------------------- | :----------------------------------------- |
| `gRPC` | `evmos.revenue.v1.Msg/RegisterRevenue`   | 注册合约以接收收入                           |
| `gRPC` | `evmos.revenue.v1.Msg/UpdateRevenue`     | 更新合约的提款地址                         |
| `gRPC` | `evmos.revenue.v1.Msg/CancelRevenue`     | 移除合约的收入                             |
| `POST` | `/evmos/revenue/v1/tx/register_revenue` | 注册合约以接收收入                           |
| `POST` | `/evmos/revenue/v1/tx/update_revenue`   | 更新合约的提款地址                         |
| `POST` | `/evmos/revenue/v1/tx/cancel_revenue`   | 移除合约的收入                             |

## 未来的改进

- 可以扩展费用分配注册以注册提现地址
  根据[EIP173](https://eips.ethereum.org/EIPS/eip-173)将其注册给合约的所有者。
- 扩展交易费用分配的支持消息类型到与EVM交互的Cosmos交易
  （例如：ERC20模块，IBC交易）。
- 将内部交易调用的费用分配给其他注册合约。
  目前，我们只将交易费用发送给智能合约的部署者，
  由交易请求的`to`字段（`MyContract`）表示。
  我们不会将费用分配给由`MyContract`内部调用的智能合约。
- 支持`CREATE2`操作码以进行地址派生。
  在注册智能合约时，我们验证其地址是否派生自部署者的地址。
  目前，我们只支持使用`CREATE`操作码的派生路径，这适用于大多数情况。


# `revenue`

## Abstract

This document specifies the internal `x/revenue` module of the Evmos Hub.

The `x/revenue` module enables the Evmos Hub to support splitting transaction fees
between block proposer and smart contract deployers.
As a part of the [Evmos Token Model](https://evmos.blog/the-evmos-token-model-edc07014978b),
this mechanism aims to increase the adoption of the Evmos Hub
by offering a new stable source of income for smart contract deployers.
Developers can register their smart contracts and everytime someone interacts with a registered smart contract,
the contract deployer or their assigned withdrawal account receives a part of the transaction fees.

Together, all registered smart contracts make up the Evmos dApp Store:
paying developers and network operators for their services via built-in shared fee revenue model.

## Contents

1. **[Concepts](#concepts)**
2. **[State](#state)**
3. **[State Transitions](#state-transitions)**
4. **[Transactions](#transactions)**
5. **[Hooks](#hooks)**
6. **[Events](#events)**
7. **[Parameters](#parameters)**
8. **[Clients](#clients)**
9. **[Future Improvements](#future-improvements)**

## Concepts

### Evmos dApp Store

The Evmos dApp store is a revenue-per-transaction model, which allows developers
to get payed for deploying their decentralized application (dApps) on Evmos.
Developers generate revenue, every time a user interacts with their dApp in the dApp store, gaining them a steady income.
Users can discover new applications in the dApp store and pay for the transaction fees that finance the dApp's revenue.
This value-reward exchange of dApp services for transaction fees is implemented by the `x/revenue` module.

### Registration

Developers register their application in the dApp store by registering their application's smart contracts.
Any contract can be registered by a developer by submitting a signed transaction.
The signer of this transaction must match the address of the deployer of the contract
in order for the registration to succeed.
After the transaction is executed successfully,
the developer will start receiving a portion of the transaction fees paid when a user interacts with the registered contract.

:::tip
**NOTE**: If your contract is part of a developer project, please ensure
that the deployer of the contract (or the factory that deployes the contract) is an account
that is owned by that project.
This avoids the situtation, that an individual deployer who leaves your project could become malicious.
:::

### Fee Distribution

As described above, developers will earn a portion of the transaction fee after registering their contracts.
To understand how transaction fees are distributed, we look at the following two things in detail:

* The transactions eligible are only [EVM transactions](evm.md) (`MsgEthereumTx`).
  Cosmos SDK transactions are not eligible at this time.
* The registration of factory contracts (smart contracts that have been deployed by other contracts)
  requires the identification original contract's deployer.
  This is done through address derivation.

#### EVM Transaction Fees

Users pay transaction fees to pay interact with smart contracts using the EVM.
When a transaction is executed, the entire fee amount (`gasLimit * gasPrice`)
is sent to the `FeeCollector` module account
during the [Cosmos SDK AnteHandler](https://docs.cosmos.network/main/modules/auth#antehandlers) execution.
After the EVM executes the transaction, the user receives a refund of `(gasLimit - gasUsed) * gasPrice`.
In result a user pays a total transaction fee of `txFee = gasUsed * gasPrice` for the execution.

This transaction fee is distributed between developers and validators,
in accordance with the `x/revenue` module parameters: `DeveloperShares`, `ValidatorShares`.
This distribution is handled through the EVM's [`PostTxProcessing` Hook](#hooks).

#### Address Derivation

dApp developers might use a [factory pattern](https://en.wikipedia.org/wiki/Factory_method_pattern)
to implement their application logic through smart contracts.
In this case a smart contract can be either deployed by an Externally Owned Account ([EOA](https://ethereum.org/en/whitepaper/#ethereum-accounts):
an account controlled by a private key, that can sign transactions) or through another contract.

In both cases, the fee distribution requires the identification a deployer address that is an EOA address,
unless a withdrawal address is set by the contract deployer during registration
to receive transaction fees for a registered smart contract.
If a withdrawal address is not set, it defaults to the deployer’s address.

The identification of the deployer address is done through address derivation.
When registering a smart contract, the deployer provides an array of nonces, used to [derive the contract’s address](https://github.com/ethereum/go-ethereum/blob/d8ff53dfb8a516f47db37dbc7fd7ad18a1e8a125/crypto/crypto.go#L107-L111):

* If `MyContract` is deployed directly by `DeployerEOA`, in a transaction sent with nonce `5`,
  then the array of nonces is `[5]`.
* If the contract was created by a smart contract, through the `CREATE` opcode,
  we need to provide all the nonces from the creation path.
  E.g.
  if `DeployerEOA` deploys a `FactoryA` smart contract with nonce `5`.
  Then, `DeployerEOA` sends a transaction to `FactoryA` through which a `FactoryB` smart contract is created.
  If we assume `FactoryB` is the second contract created by `FactoryA`, then `FactoryA`'s nonce is `2`.
  Then, `DeployerEOA` sends a transaction to the `FactoryB` contract, through which `MyContract` is created.
  If this is the first contract created by `FactoryB` - the nonce is `1`.
  We now have an address derivation path of `DeployerEOA` -> `FactoryA` -> `FactoryB` -> `MyContract`.
  To be able to verify that `DeployerEOA` can register `MyContract`, we need to provide the following nonces: `[5, 2, 1]`.

:::tip
**Note**: Even if `MyContract` is created from `FactoryB` through a transaction
sent by an account different from `DeployerEOA`, only `DeployerEOA` can register `MyContract`.
:::

## State

### State Objects

The `x/revenue` module keeps the following objects in state:

| State Object          | Description                           | Key                                                               | Value              | Store |
| :-------------------- | :------------------------------------ | :---------------------------------------------------------------- | :----------------- | :---- |
| `Revenue`            | Fee split bytecode                     | `[]byte{1} + []byte(contract_address)`                            | `[]byte{revenue}` | KV    |
| `DeployerRevenues`   | Contract by deployer address bytecode | `[]byte{2} + []byte(deployer_address) + []byte(contract_address)` | `[]byte{1}`        | KV    |
| `WithdrawerRevenues` | Contract by withdraw address bytecode | `[]byte{3} + []byte(withdraw_address) + []byte(contract_address)` | `[]byte{1}`        | KV    |

#### Revenue

A Revenue defines an instance that organizes fee distribution conditions for
the owner of a given smart contract

```go
type Revenue struct {
	// hex address of registered contract
	ContractAddress string `protobuf:"bytes,1,opt,name=contract_address,json=contractAddress,proto3" json:"contract_address,omitempty"`
	// bech32 address of contract deployer
	DeployerAddress string `protobuf:"bytes,2,opt,name=deployer_address,json=deployerAddress,proto3" json:"deployer_address,omitempty"`
	// bech32 address of account receiving the transaction fees it defaults to
	// deployer_address
	WithdrawerAddress string `protobuf:"bytes,3,opt,name=withdrawer_address,json=withdrawerAddress,proto3" json:"withdrawer_address,omitempty"`
}
```

#### ContractAddress

`ContractAddress` defines the contract address that has been registered for fee distribution.

#### DeployerAddress

A `DeployerAddress` is the EOA address for a registered contract.

#### WithdrawerAddress

The `WithdrawerAddress` is the address that receives transaction fees for a registered contract.

### Genesis State

The `x/revenue` module's `GenesisState` defines the state necessary for initializing the chain from a previous exported height.
It contains the module parameters and the revenues for registered contracts:

```go
// GenesisState defines the module's genesis state.
type GenesisState struct {
	// module parameters
	Params Params `protobuf:"bytes,1,opt,name=params,proto3" json:"params"`
	// active registered contracts for fee distribution
	Revenues []Revenue `protobuf:"bytes,2,rep,name=revenues,json=revenues,proto3" json:"revenues"`
}

```

## State Transitions

The `x/revenue` module allows for three types of state transitions:
`RegisterRevenue`, `UpdateRevenue` and `CancelRevenue`.
The logic for distributing transaction fees is handled through [Hooks](#hooks).

#### Register Fee Split

A developer registers a contract for receiving transaction fees, defining the contract address,
an array of nonces for [address derivation](#concepts)
and an optional withdraw address for receiving fees.
If the withdraw address is not set, the fees are sent to the deployer address by default.

1. User submits a `RegisterRevenue` to register a contract address,
   along with a withdraw address that they would like to receive the fees to
2. Check if the following conditions pass:
    1. `x/revenue` module is enabled
    2. the contract was not previously registered
    3. deployer has a valid account (it has done at least one transaction) and is not a smart contract
    4. an account corresponding to the contract address exists, with a non-empty bytecode
    5. contract address can be derived from the deployer’s address and provided nonces using the `CREATE` operation
    6. contract is already deployed
3. Store an instance of the provided fee.

All transactions sent to the registered contract occurring after registration
will have their fees distributed to the developer, according to the global `DeveloperShares` parameter.

#### Update Fee Split

A developer updates the withdraw address for a registered contract,
defining the contract address and the new withdraw address.

1. User submits a `UpdateRevenue`
2. Check if the following conditions pass:
    1. `x/revenue` module is enabled
    2. the contract is registered
    3. the signer of the transaction is the same as the contract deployer
3. Update the fee with the new withdraw address.
   Note that if withdraw address is empty or the same as deployer address, then the withdraw address is set to `""`.

After this update, the developer receives the fees on the new withdraw address.

#### Cancel Fee Split

A developer cancels receiving fees for a registered contract, defining the contract address.

1. User submits a `CancelRevenue`
2. Check if the following conditions pass:
    1. `x/revenue` module is enabled
    2. the contract is registered
    3. the signer of the transaction is the same as the contract deployer
3. Remove fee from storage

The developer no longer receives fees from transactions sent to this contract.

## Transactions

This section defines the `sdk.Msg` concrete types that result in the state transitions defined on the previous section.

### `MsgRegisterRevenue`

Defines a transaction signed by a developer to register a contract for transaction fee distribution.
The sender must be an EOA that corresponds to the contract deployer address.

```go
type MsgRegisterRevenue struct {
	// contract hex address
	ContractAddress string `protobuf:"bytes,1,opt,name=contract_address,json=contractAddress,proto3" json:"contract_address,omitempty"`
	// bech32 address of message sender, must be the same as the origin EOA
	// sending the transaction which deploys the contract
	DeployerAddress string `protobuf:"bytes,2,opt,name=deployer_address,json=deployerAddress,proto3" json:"deployer_address,omitempty"`
	// bech32 address of account receiving the transaction fees
	WithdrawerAddress string `protobuf:"bytes,3,opt,name=withdraw_address,json=withdrawerAddress,proto3" json:"withdraw_address,omitempty"`
	// array of nonces from the address path, where the last nonce is
	// the nonce that determines the contract's address - it can be an EOA nonce
	// or a factory contract nonce
	Nonces []uint64 `protobuf:"varint,4,rep,packed,name=nonces,proto3" json:"nonces,omitempty"`
}
```

The message content stateless validation fails if:

- Contract hex address is invalid
- Contract hex address is zero
- Deployer bech32 address is invalid
- Withdraw bech32 address is invalid
- Nonces array is empty

### `MsgUpdateRevenue`

Defines a transaction signed by a developer to update the withdraw address of a contract registered for transaction fee distribution.
The sender must be an EOA that corresponds to the contract deployer address.

```go
type MsgUpdateRevenue struct {
	// contract hex address
	ContractAddress string `protobuf:"bytes,1,opt,name=contract_address,json=contractAddress,proto3" json:"contract_address,omitempty"`
	// deployer bech32 address
	DeployerAddress string `protobuf:"bytes,2,opt,name=deployer_address,json=deployerAddress,proto3" json:"deployer_address,omitempty"`
	// new withdraw bech32 address for receiving the transaction fees
	WithdrawerAddress string `protobuf:"bytes,3,opt,name=withdraw_address,json=withdrawerAddress,proto3" json:"withdraw_address,omitempty"`
}
```

The message content stateless validation fails if:

- Contract hex address is invalid
- Contract hex address is zero
- Deployer bech32 address is invalid
- Withdraw bech32 address is invalid
- Withdraw bech32 address is same as deployer address

### `MsgCancelRevenue`

Defines a transaction signed by a developer to remove the information for a registered contract.
Transaction fees will no longer be distributed to the developer, for this smart contract.
The sender must be an EOA that corresponds to the contract deployer address.

```go
type MsgCancelRevenue struct {
	// contract hex address
	ContractAddress string `protobuf:"bytes,1,opt,name=contract_address,json=contractAddress,proto3" json:"contract_address,omitempty"`
	// deployer bech32 address
	DeployerAddress string `protobuf:"bytes,2,opt,name=deployer_address,json=deployerAddress,proto3" json:"deployer_address,omitempty"`
}
```

The message content stateless validation fails if:

- Contract hex address is invalid
- Contract hex address is zero
- Deployer bech32 address is invalid

## Hooks

The fees module implements one transaction hook from the `x/evm` module
in order to distribute fees between developers and validators.

### EVM Hook

A [`PostTxProcessing` EVM hook](evm.md#hooks) executes custom logic
after each successful EVM transaction.
All fees paid by a user for transaction execution are sent to the `FeeCollector` module account
during the `AnteHandler` execution before being distributed to developers and validators.

If the `x/revenue` module is disabled or the EVM transaction targets an unregistered contract,
the EVM hook returns `nil`, without performing any actions.
In this case, 100% of the transaction fees remain in the `FeeCollector` module, to be distributed to the block proposer.

If the `x/revenue` module is enabled and a EVM transaction targets a registered contract,
the EVM hook sends a percentage of the transaction fees (paid by the user)
to the withdraw address set for that contract, or to the contract deployer.

1. User submits EVM transaction (`MsgEthereumTx`) to a smart contract and transaction is executed successfully
2. Check if
    * fees module is enabled
    * smart contract is registered to receive fees
3. Calculate developer fees according to the `DeveloperShares` parameter.
   The initial transaction message includes the gas price paid by the user and the transaction receipt,
   which includes the gas used by the transaction.

   ```go
    devFees := receipt.GasUsed * msg.GasPrice * params.DeveloperShares
    ```

4. Transfer developer fee from the `FeeCollector` (Cosmos SDK `auth` module account)
   to the registered withdraw address for that contract.
   If there is no withdraw address, fees are sent to contract deployer's address.
5. Distribute the remaining amount in the `FeeCollector` to validators according to the
   [SDK  Distribution Scheme](https://docs.cosmos.network/main/modules/distribution#the-distribution-scheme).


## Events

The `x/revenue` module emits the following events:

### Register Fee Split

| Type                 | Attribute Key          | Attribute Value           |
| :------------------- | :--------------------- | :------------------------ |
| `register_revenue` | `"contract"`           | `{msg.ContractAddress}`   |
| `register_revenue` | `"sender"`             | `{msg.DeployerAddress}`   |
| `register_revenue` | `"withdrawer_address"` | `{msg.WithdrawerAddress}` |

### Update Fee Split

| Type               | Attribute Key          | Attribute Value           |
| :----------------- | :--------------------- | :------------------------ |
| `update_revenue` | `"contract"`           | `{msg.ContractAddress}`   |
| `update_revenue` | `"sender"`             | `{msg.DeployerAddress}`   |
| `update_revenue` | `"withdrawer_address"` | `{msg.WithdrawerAddress}` |

### Cancel Fee Split

| Type               | Attribute Key | Attribute Value         |
| :----------------- | :------------ | :---------------------- |
| `cancel_revenue` | `"contract"`  | `{msg.ContractAddress}` |
| `cancel_revenue` | `"sender"`    | `{msg.DeployerAddress}` |

## Parameters

The fees module contains the following parameters:

| Key                        | Type    | Default Value |
| :------------------------- | :------ | :------------ |
| `EnableRevenue`           | bool    | `true`        |
| `DeveloperShares`          | sdk.Dec | `50%`         |
| `AddrDerivationCostCreate` | uint64  | `50`          |

### Enable Revenue Module

The `EnableRevenue` parameter toggles all state transitions in the module.
When the parameter is disabled, it will prevent any transaction fees from being distributed to contract deployers
and it will disallow contract registrations, updates or cancellations.

#### Developer Shares Amount

The `DeveloperShares` parameter is the percentage of transaction fees that is sent to the contract deployers.

#### Address Derivation Cost with CREATE opcode

The `AddrDerivationCostCreate` parameter is the gas value charged
for performing an address derivation in the contract registration process.
A flat gas fee is charged for each address derivation iteration.
We allow a maximum number of 20 iterations, and therefore a maximum number of 20 nonces can be given
for deriving the smart contract address from the deployer's address.


## Clients

### CLI

Find below a list of  `evmosd` commands added with the  `x/revenue` module.
You can obtain the full list by using the `evmosd -h` command.
A CLI command can look like this:

```bash
evmosd query revenue params
```

#### Queries

| Command            | Subcommand             | Description                              |
| :----------------- | :--------------------- | :--------------------------------------- |
| `query` `revenue` | `params`               | Get revenue params                          |
| `query` `revenue` | `contract`             | Get the revenue for a given contract   |
| `query` `revenue` | `contracts`            | Get all revenues                       |
| `query` `revenue` | `deployer-contracts`   | Get all revenues of a given deployer   |
| `query` `revenue` | `withdrawer-contracts` | Get all revenues of a given withdrawer |

#### Transactions

| Command         | Subcommand | Description                                |
| :-------------- | :--------- | :----------------------------------------- |
| `tx` `revenue` | `register` | Register a contract for receiving revenue     |
| `tx` `revenue` | `update`   | Update the withdraw address for a contract |
| `tx` `revenue` | `cancel`   | Remove the revenue for a contract        |

### gRPC

#### Queries

| Verb   | Method                                            | Description                              |
| :----- | :------------------------------------------------ | :--------------------------------------- |
| `gRPC` | `evmos.revenue.v1.Query/Params`                  | Get revenue params                          |
| `gRPC` | `evmos.revenue.v1.Query/Revenue`                | Get the revenue for a given contract   |
| `gRPC` | `evmos.revenue.v1.Query/Revenues`               | Get all revenues                       |
| `gRPC` | `evmos.revenue.v1.Query/DeployerRevenues`       | Get all revenues of a given deployer   |
| `gRPC` | `evmos.revenue.v1.Query/WithdrawerRevenues`     | Get all revenues of a given withdrawer |
| `GET`  | `/evmos/revenue/v1/params`                       | Get revenue params                          |
| `GET`  | `/evmos/revenue/v1/revenues/{contract_address}`  | Get the revenue for a given contract   |
| `GET`  | `/evmos/revenue/v1/revenues`                    | Get all revenues                       |
| `GET`  | `/evmos/revenue/v1/revenues/{deployer_address}` | Get all revenues of a given deployer   |
| `GET`  | `/evmos/revenue/v1/revenues/{withdraw_address}` | Get all revenues of a given withdrawer |

#### Transactions

| Verb   | Method                                     | Description                                |
| :----- | :----------------------------------------- | :----------------------------------------- |
| `gRPC` | `evmos.revenue.v1.Msg/RegisterRevenue`   | Register a contract for receiving revenue     |
| `gRPC` | `evmos.revenue.v1.Msg/UpdateRevenue`     | Update the withdraw address for a contract |
| `gRPC` | `evmos.revenue.v1.Msg/CancelRevenue`     | Remove the revenue for a contract        |
| `POST` | `/evmos/revenue/v1/tx/register_revenue` | Register a contract for receiving revenue     |
| `POST` | `/evmos/revenue/v1/tx/update_revenue`   | Update the withdraw address for a contract |
| `POST` | `/evmos/revenue/v1/tx/cancel_revenue`   | Remove the revenue for a contract        |

## Future Improvements

- The fee distribution registration could be extended to register the withdrawal address
  to the owner of the contract according to [EIP173](https://eips.ethereum.org/EIPS/eip-173).
- Extend the supported message types for the transaction fee distribution to Cosmos transactions
  that interact with the EVM (eg: ERC20 module, IBC transactions).
- Distribute fees for internal transaction calls to other registered contracts.
  At this time, we only send transaction fees to the deployer of the smart contract
  represented by the `to` field of the transaction request (`MyContract`).
  We do not distribute fees to smart contracts called internally by `MyContract`.
- `CREATE2` opcode support for address derivation.
  When registering a smart contract, we verify that its address is derived from the deployer’s address.
  At this time, we only support the derivation path using the `CREATE` opcode, which accounts for most cases.


