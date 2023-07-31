# `feemarket`

## 摘要

本文档规定了`feemarket`模块，该模块允许为网络定义全局交易费用。

该模块已经设计用于支持cosmos-sdk中的EIP1559。

需要覆盖`x/auth`模块中的`MempoolFeeDecorator`，以检查`baseFee`和`minimal-gas-prices`，从而实现根据网络活动变化的全局费用机制。

有关EIP1559的更多参考信息：

<https://github.com/ethereum/EIPs/blob/master/EIPS/eip-1559.md>

## 目录

1. **[概念](#概念)**
2. **[状态](#状态)**
3. **[开始区块](#开始区块)**
4. **[结束区块](#结束区块)**
5. **[管理器](#管理器)**
6. **[事件](#事件)**
7. **[参数](#参数)**
8. **[客户端](#客户端)**
9. **[Ante处理器](#Ante处理器)**


## 概念

### EIP-1559: 费用市场

[EIP-1559](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-1559.md)描述了以太坊上提出的一种定价机制，用于改进交易费用的计算。
它包括一个每个区块固定的网络费用，该费用会被销毁，并且动态地扩展/收缩区块大小以应对网络拥堵的高峰期。

在EIP-1559之前，交易费用的计算方式为

```
fee = gasPrice * gasLimit
```

其中`gasPrice`是每个gas的价格，`gasLimit`描述了执行交易所需的gas数量。
交易所需的操作越复杂，gas限制就越高
（参见[执行EVM字节码](evm.md#执行EVM字节码)）。
要提交交易，签名者需要指定`gasPrice`。

启用EIP-1559后，交易费用的计算方式为

```
fee = (baseFee + priorityTip) * gasLimit
```

其中`baseFee`是每个gas的固定网络费用
而`priorityTip`是可选设置的每个gas的额外费用。
注意，基础费用和优先费用都是以gas价格计算的。
要使用EIP-1559提交交易，签名者需要指定`gasFeeCap`，
即他们愿意支付的每个gas的最大费用。
可选地，可以指定`priorityTip`，
它涵盖了优先费用和区块的每个gas的网络费用（也称为基础费用）。

:::tip
Cosmos SDK在“gas”方面使用与以太坊不同的术语。
以太坊上称为“gasLimit”的东西，在Cosmos上称为“gasWanted”。
在Evmos上，您可能会遇到这两种术语，因为它在SDK之上构建了以太坊，
例如，当使用不同的钱包（如Cosmos的Keplr和以太坊的Metamask）时。
:::

### 基础费用

每个gas的基础费用（也称为基础费用）是在共识级别上定义的全局gas价格。
它作为模块参数存储，
并在每个区块结束时根据前一个区块中使用的总gas和gas目标（`区块gas限制/弹性乘数`）进行调整：

- 当区块超过gas目标时，它会增加，
- 当区块低于gas目标时，它会减少。

与在以太坊上实现的燃烧基础费用不同，
`feemarket`模块将基础费用分配给常规的[Cosmos SDK费用分配](https://docs.cosmos.network/main/modules/distribution)。

### 优先小费

在EIP-1559中，`max_priority_fee_per_gas`（通常称为小费）是可以添加到`baseFee`中的额外gas价格，以激励事务优先级。
小费越高，事务越有可能被包含在区块中。

然而，在Cosmos SDK版本v0.46之前，没有事务优先级的概念。
因此，在Evmos上，EIP-1559事务的小费应为零（`MaxPriorityFeePerGas` JSON-RPC端点返回`0`）。
请查看[mempool](https://docs.evmos.org/validate/setup-and-configuration/mempool)文档，了解如何利用事务优先级的更多信息。

### 有效的Gas价格

对于EIP-1559事务（动态费用事务），有效的Gas价格描述了事务愿意提供的最高Gas价格。
它由事务参数和基础费用参数派生而来。
根据哪个较小，有效的Gas价格要么是`baseFee + tip`，要么是`gasFeeCap`

```
min(baseFee + gasTipCap, gasFeeCap)
```

### 本地与全局最低Gas价格

最低Gas价格用于丢弃网络中的垃圾事务，
通过提高事务的成本，使垃圾邮件发送者无法从经济上进行操作。
这是通过为接受mempool中的txs定义最低Gas价格来实现的，
对于Cosmos和EVM事务都是如此。
如果事务没有提供这两种类型的最低Gas价格之一，则会从mempool中丢弃该事务。

最低燃气价格用于在网络中丢弃垃圾事务，通过提高事务的成本，使垃圾邮件发送者无法从经济上可行。
这是通过为接受 Cosmos 和 EVM 事务的 mempool 定义最低燃气价格来实现的。
如果事务没有提供以下两种类型的最低燃气价格之一，则将其从 mempool 中丢弃：

1. 本地最低燃气价格，验证人可以在其节点配置中设置。
2. 全局最低燃气价格，作为 `feemarket` 模块中的参数设置，可以通过治理进行更改。

事务的燃气价格下限通过以下三种情况的燃气价格边界比较来确定：

1. 如果有效燃气价格（`有效燃气价格 = 基础费用 + 优先费用`）或本地最低燃气价格低于全局 `MinGasPrice`
   （`min-gas-price (local) < MinGasPrice (global) OR EffectiveGasPrice < MinGasPrice`），
   则使用 `MinGasPrice` 作为下限。
2. 如果事务因燃气价格低于 `MinGasPrice` 被拒绝，
   用户需要使用高于或等于 `MinGasPrice` 的燃气价格重新发送事务。
3. 如果有效燃气价格或本地 `minimum-gas-price` 高于全局 `MinGasPrice`，
   则较大的值将作为下限。
   在 EIP-1559 的情况下，用户必须增加事务的优先费用才能使其有效。

事务燃气价格与下限的比较是通过 AnteHandler 装饰器实现的。
对于 EVM 事务，这是在 `EthMempoolFeeDecorator` 和 `EthMinGasPriceDecorator` `AnteHandler` 中完成的，
对于 Cosmos 事务，这是在 `NewMempoolFeeDecorator` 和 `MinGasPriceDecorator` `AnteHandler` 中完成的。

:::tip
如果基础费用降低到低于全局 `MinGasPrice` 的值，它将被设置为 `MinGasPrice`。
这样实现的目的是，基础费用不能降到不允许事务在 mempool 中被接受的燃气价格，因为存在更高的 `MinGasPrice`。
:::

## 状态

x/feemarket模块保留了用于费用计算的状态变量：

只需要跟踪上一个区块中的BlockGasUsed，以便进行下一个基础费用计算。

|                  | 描述                           | 键              | 值                   | 存储      |
| -----------      | ------------------------------ | ---------------| ------------------- | --------- |
| BlockGasUsed     | 区块中使用的Gas                | `[]byte{1}`    | `[]byte{gas_used}`  | KV        |


## 开始区块

基础费用在每个区块开始时计算。

### 基础费用

#### 禁用基础费用

我们引入了两个参数：`NoBaseFee`和`EnableHeight`

`NoBaseFee`控制着feemarket基础费用的值。
如果设置为true，则不进行计算，并且keeper返回的基础费用为零。

`EnableHeight`控制着我们开始计算的高度。

- 如果`NoBaseFee = false`且`height < EnableHeight`，
  基础费用的值将等于创世块中定义的`base_fee`，
  并且`BeginBlock`将返回而不进行进一步计算。
- 如果`NoBaseFee = false`且`height >= EnableHeight`，
  基础费用将在每个区块的`BeginBlock`中动态计算。

这些参数允许我们在后期引入静态基础费用或激活基础费用。

#### 启用基础费用

要在EVM中启用EIP1559，应设置以下参数：

- NoBaseFee应为false
- EnableHeight应设置为大于或等于升级高度的正整数。
  它定义了链在哪个高度开始进行基础费用调整。
- LondonBlock evm的参数应设置为大于或等于升级高度的正整数。
  它定义了链在哪个高度开始接受EIP1559交易。

#### 计算

基础费用在`EnableHeight`处初始化为创世文件中定义的`InitialBaseFee`值。

然后，根据上一个区块中使用的总Gas进行调整。

```golang
parent_gas_target = parent_gas_limit / ELASTICITY_MULTIPLIER

if EnableHeight == block.number
    base_fee = INITIAL_BASE_FEE
else if parent_gas_used == parent_gas_target:
    base_fee = parent_base_fee
else if parent_gas_used > parent_gas_target:
    gas_used_delta = parent_gas_used - parent_gas_target
    base_fee_delta = max(parent_base_fee * gas_used_delta / parent_gas_target / BASE_FEE_MAX_CHANGE_DENOMINATOR, 1)
    base_fee = parent_base_fee + base_fee_delta
else:
    gas_used_delta = parent_gas_target - parent_gas_used
    base_fee_delta = parent_base_fee * gas_used_delta / parent_gas_target / BASE_FEE_MAX_CHANGE_DENOMINATOR
    base_fee = parent_base_fee - base_fee_delta

```


## 结束区块

`block_gas_used`的值在每个区块结束时更新。

### 区块使用的燃气量

当前区块使用的总燃气量存储在`EndBlock`的KVStore中。

它的初始值为创世块中定义的`block_gas`。


## Keeper

feemarket模块提供了这个导出的keeper，
可以传递给其他模块，
这些模块需要访问基础费用值

```go
type Keeper interface {
    GetBaseFee(ctx sdk.Context) *big.Int
}
```


## 事件

`x/feemarket`模块会发出以下事件：

### BeginBlocker

| 类型       | 属性键   | 属性值 |
| ---------- | --------------- | --------------- |
| fee_market | base_fee        | {baseGasPrices} |

### EndBlocker

| 类型       | 属性键   | 属性值 |
| ---------- | --------------- | --------------- |
| block_gas  | height          | {blockHeight}   |
| block_gas  | amount          | {blockGasUsed}  |


## 参数

`x/feemarket`模块包含以下参数：

| 键                      | 类型    | 默认值 | 描述                                                                                                             |
|--------------------------|---------|----------------|-------------------------------------------------------------------------------------------------------------------------|
| NoBaseFee                | bool    | false          | 控制基础费用调整                                                                                         |
| BaseFeeChangeDenominator | uint32  | 8              | 限制基础费用在区块之间可以变化的数量                                                           |
| ElasticityMultiplier     | uint32  | 2              | 限制基础费用根据前一个区块中使用的总燃气量增加或减少的阈值 |
| BaseFee                  | uint32  | 1000000000     | EIP-1559区块的基础费用                                                                                            |
| EnableHeight             | uint32  | 0              | 启用费用调整的高度                                                                                      |
| MinGasPrice              | sdk.Dec | 0              | 全局最低燃气价格，需要支付以将交易包含在区块中                                      |

## 客户端

### 命令行界面（CLI）

用户可以使用命令行界面（CLI）查询和与 `feemarket` 模块进行交互。

#### 查询

`query` 命令允许用户查询 `feemarket` 的状态。

```bash
evmosd query feemarket --help
```

##### 基础费用

`base-fee` 命令允许用户按高度查询区块的基础费用。

```bash
evmosd query feemarket base-fee [flags]
```

示例：

```bash
evmosd query feemarket base-fee ...
```

示例输出：

```
base_fee: "512908936"
```

##### 区块 Gas

`block-gas` 命令允许用户按高度查询区块的 Gas。

```bash
evmosd query feemarket block-gas [flags]
```

示例：

```bash
evmosd query feemarket block-gas ...
```

示例输出：

```
gas: "21000"
```

##### 参数

`params` 命令允许用户查询模块的参数。

```bash
evmosd query params subspace [subspace] [key] [flags]
```

示例：

```bash
evmosd query params subspace feemarket ElasticityMultiplier ...
```

示例输出：

```
key: ElasticityMultiplier
subspace: feemarket
value: "2"
```

### gRPC

#### 查询

| 动词   | 方法                                    | 描述                   |
|--------|-----------------------------------------|------------------------|
| `gRPC` | `ethermint.feemarket.v1.Query/Params`   | 获取模块参数           |
| `gRPC` | `ethermint.feemarket.v1.Query/BaseFee`  | 获取区块基础费用       |
| `gRPC` | `ethermint.feemarket.v1.Query/BlockGas` | 获取区块使用的 Gas     |
| `GET`  | `/ethermint/feemarket/v1/params`        | 获取模块参数           |
| `GET`  | `/ethermint/feemarket/v1/base_fee`      | 获取区块基础费用       |
| `GET`  | `/ethermint/feemarket/v1/block_gas`     | 获取区块使用的 Gas     |


## AnteHandlers

`x/feemarket` 模块提供了 `AnteDecorator`，这些装饰器被递归链接在一起，形成一个单一的 [`Antehandler`](https://github.com/cosmos/cosmos-sdk/blob/v0.43.0-alpha1/docs/architecture/adr-010-modular-antehandler.md)。
这些装饰器对以太坊或 Cosmos SDK 交易执行基本的有效性检查，以便可以将其从交易内存池中排除。

请注意，`AnteHandler` 在每个交易中运行，并在 `CheckTx` 和 `DeliverTx` 时调用。

### 装饰器

### `MinGasPriceDecorator`

拒绝 Cosmos SDK 交易的交易费低于 `MinGasPrice * GasLimit`。

### `EthMinGasPriceDecorator`

拒绝 EVM 交易的交易费低于 `MinGasPrice * GasLimit`。

- 对于 `LegacyTx` 和 `AccessListTx`，使用 `GasPrice * GasLimit`。
- 对于 EIP-1559（又称为 `DynamicFeeTx`），使用 `EffectivePrice * GasLimit`。

:::tip
**注意**：对于动态交易，
如果 `feemarket` 公式导致 `BaseFee` 降低 `EffectivePrice < MinGasPrices`，
用户必须增加 `GasTipCap`（优先费用），直到 `EffectivePrice > MinGasPrices`。
`MinGasPrices * GasLimit < transaction fee < EffectiveFee` 的交易将被 `feemarket` 的 `AnteHandle` 拒绝。
:::

### `EthGasConsumeDecorator`

根据 EIP-1559 规范计算要扣除的有效费用和交易优先级，
然后在响应中扣除费用并设置交易优先级。

```
effectivePrice = min(baseFee + tipFeeCap, gasFeeCap)
effectiveTipFee = effectivePrice - baseFee
priority = effectiveTipFee / DefaultPriorityReduction
```

当交易中有多个消息时，请选择其中优先级最低的消息。


# `feemarket`

## Abstract

This document specifies the feemarket module which allows to define a global transaction fee for the network.

This module has been designed to support EIP1559 in cosmos-sdk.

The `MempoolFeeDecorator` in `x/auth` module needs to be overwritten
to check the `baseFee` along with the `minimal-gas-prices` allowing
to implement a global fee mechanism which vary depending on the network activity.

For more reference to EIP1559:

<https://github.com/ethereum/EIPs/blob/master/EIPS/eip-1559.md>

## Contents

1. **[Concepts](#concepts)**
2. **[State](#state)**
3. **[Begin Block](#begin-block)**
4. **[End Block](#end-block)**
5. **[Keeper](#keeper)**
6. **[Events](#events)**
7. **[Params](#params)**
8. **[Client](#client)**
9. **[AnteHandlers](#antehandlers)**


## Concepts

### EIP-1559: Fee Market

[EIP-1559](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-1559.md) describes a pricing mechanism
that was proposed on Ethereum to improve to calculation of transaction fees.
It includes a fixed-per-block network fee that is burned
and dynamically expands/contracts block sizes to deal with peaks of network congestion.

Before EIP-1559 the transaction fee is calculated with

```
fee = gasPrice * gasLimit
```

where `gasPrice` is the price per gas and `gasLimit` describes the amount of gas required to perform the transaction.
The more complex operations a transaction requires, the higher the gas limit
(see [Executing EVM bytecode](evm.md#executing-evm-bytecode)).
To submit a transaction, the signer needs to specify the `gasPrice`.

With EIP-1559 enabled, the transaction fee is calculated with

```
fee = (baseFee + priorityTip) * gasLimit
```

where `baseFee` is the fixed-per-block network fee per gas
and `priorityTip` is an additional fee per gas that can be set optionally.
Note, that both the base fee and the priority tip are gas prices.
To submit a transaction with EIP-1559, the signer needs to specify the `gasFeeCap`,
which is the maximum fee per gas they are willing to pay in total.
Optionally, the `priorityTip` can be specified,
which covers both the priority fee and the block's network fee per gas (aka: base fee).

:::tip
The Cosmos SDK uses a different terminology for `gas` than Ethereum.
What is called `gasLimit` on Ethereum is called `gasWanted` on Cosmos.
You might encounter both terminologies on Evmos since it builds Ethereum on top of the SDK,
e.g. when using different wallets like Keplr for Cosmos and Metamask for Ethereum.
:::

### Base Fee

The base fee per gas (aka base fee) is a global gas price defined at the consensus level.
It is stored as a module parameter
and is adjusted at the end of each block
based on the total gas used in the previous block
and gas target (`block gas limit / elasticity multiplier`):

- it increases when blocks are above the gas target,
- it decreases when blocks are below the gas target.

Instead of burning the base fee (as implemented on Ethereum),
the `feemarket` module allocates the base fee
for regular [Cosmos SDK fee distribution](https://docs.cosmos.network/main/modules/distribution).

### Priority Tip

In EIP-1559, the `max_priority_fee_per_gas`, often referred to as `tip`,
is an additional gas price that can be added to the `baseFee` in order to incentivize transaction prioritization.
The higher the tip, the more likely the transaction is included in the block.

Until the Cosmos SDK version v0.46, however, there is no notion of transaction prioritization.
Thus, the tip for an EIP-1559 transaction on Evmos should be zero
(`MaxPriorityFeePerGas` JSON-RPC endpoint returns `0`).
Have a look at the [mempool](https://docs.evmos.org/validate/setup-and-configuration/mempool) docs
to read more about how to leverage transaction prioritization.

### Effective Gas price

For EIP-1559 transactions (dynamic fee transactions) the effective gas price describes the maximum gas price
that a transaction is willing to provide.
It is derived from the transaction arguments and the base fee parameter.
Depending on which one is smaller, the effective gas price is either the `baseFee + tip` or the `gasFeeCap`

```
min(baseFee + gasTipCap, gasFeeCap)
```

### Local vs. Global Minimum Gas Prices

Minimum gas prices are used to discard spam transactions in the network,
by raising the cost of transactions to the point that it is not economically viable for the spammer.
This is achieved by defining a minimum gas price for accepting txs in the mempool
for both Cosmos and EVM transactions.
A transaction is discarded from the mempool
if it doesn't provide at least one of the two types of min gas prices:

Minimum gas prices are used to discard spam transactions in the network,
by raising the cost of transactions to the point that it is not economically viable for the spammer.
This is achieved by defining a minimum gas price for accepting txs in the mempool for both Cosmos and EVM transactions.
A transaction is discarded from the mempool if it doesn't provide at least one of the two types of min gas prices:

1. the local min gas prices that validators can set on their node config and
2. the global min gas price, which is set as a parameter in the `feemarket` module, which can be changed through governance.

The lower bound for a transaction gas price is determined by comparing of gas price bounds according to three cases:

1. If the effective gas price (`effective gas price = base fee + priority tip`)
   or the local minimum gas price is lower than the global `MinGasPrice`
   (`min-gas-price (local) < MinGasPrice (global) OR EffectiveGasPrice < MinGasPrice`),
   then `MinGasPrice` is used as a lower bound.
2. If transactions are rejected due to having a gas price lower than `MinGasPrice`,
   users need to resend the transactions with a gas price higher or equal to `MinGasPrice`.
3. If the effective gas price or the local `minimum-gas-price` is higher than the global `MinGasPrice`,
   then the larger value of the two is used as a lower bound.
   In the case of EIP-1559, users must increase the priority fee for their transactions to be valid.

The comparison of transaction gas price and the lower bound is implemented through AnteHandler decorators.
For EVM transactions, this is done in the `EthMempoolFeeDecorator` and `EthMinGasPriceDecorator` `AnteHandler`
and for Cosmos transactions in `NewMempoolFeeDecorator` and `MinGasPriceDecorator` `AnteHandler`.

:::tip
If the base fee decreases to a value below the global `MinGasPrice`, it is set to the `MinGasPrice`.
This is implemented, so that the base fee can't drop to gas prices
that wouldn't allow transactions to be accepted in the mempool, because of a higher `MinGasPrice`.
:::


## State

The x/feemarket module keeps in the state variable needed to the fee calculation:

Only BlockGasUsed in previous block needs to be tracked in state for the next base fee calculation.

|                  | Description                    | Key            | Value               | Store     |
| -----------      | ------------------------------ | ---------------| ------------------- | --------- |
| BlockGasUsed     | gas used in the block          | `[]byte{1}`    | `[]byte{gas_used}`  | KV        |


## Begin block

The base fee is calculated at the beginning of each block.

### Base Fee

#### Disabling base fee

We introduce two parameters : `NoBaseFee`and `EnableHeight`

`NoBaseFee` controls the feemarket base fee value.
If set to true, no calculation is done and the base fee returned by the keeper is zero.

`EnableHeight` controls the height we start the calculation.

- If `NoBaseFee = false` and `height < EnableHeight`,
  the base fee value will be equal to `base_fee` defined in the genesis
  and the `BeginBlock` will return without further computation.
- If `NoBaseFee = false` and `height >= EnableHeight`,
  the base fee is dynamically calculated upon each block at `BeginBlock`.

Those parameters allow us to introduce a static base fee or activate the base fee at a later stage.

#### Enabling base fee

To enable EIP1559 with the EVM, the following parameters should be set :

- NoBaseFee should be false
- EnableHeight should be set to a positive integer >= upgrade height.
  It defines at which height the chain starts the base fee adjustment
- LondonBlock evm's param should be set to a positive integer >= upgrade height.
  It defines at which height the chain start to accept EIP1559 transactions

#### Calculation

The base fee is initialized at `EnableHeight` to the `InitialBaseFee` value defined in the genesis file.

The base fee is after adjusted according to the total gas used in the previous block.

```golang
parent_gas_target = parent_gas_limit / ELASTICITY_MULTIPLIER

if EnableHeight == block.number
    base_fee = INITIAL_BASE_FEE
else if parent_gas_used == parent_gas_target:
    base_fee = parent_base_fee
else if parent_gas_used > parent_gas_target:
    gas_used_delta = parent_gas_used - parent_gas_target
    base_fee_delta = max(parent_base_fee * gas_used_delta / parent_gas_target / BASE_FEE_MAX_CHANGE_DENOMINATOR, 1)
    base_fee = parent_base_fee + base_fee_delta
else:
    gas_used_delta = parent_gas_target - parent_gas_used
    base_fee_delta = parent_base_fee * gas_used_delta / parent_gas_target / BASE_FEE_MAX_CHANGE_DENOMINATOR
    base_fee = parent_base_fee - base_fee_delta

```


## End block

The `block_gas_used` value is updated at the end of each block.

### Block Gas Used

The total gas used by current block is stored in the KVStore at `EndBlock`.

It is initialized to `block_gas` defined in the genesis.


## Keeper

The feemarket module provides this exported keeper
that can be passed to other modules,
which require access to the base fee value

```go
type Keeper interface {
    GetBaseFee(ctx sdk.Context) *big.Int
}
```


## Events

The `x/feemarket` module emits the following events:

### BeginBlocker

| Type       | Attribute Key   | Attribute Value |
| ---------- | --------------- | --------------- |
| fee_market | base_fee        | {baseGasPrices} |

### EndBlocker

| Type       | Attribute Key   | Attribute Value |
| ---------- | --------------- | --------------- |
| block_gas  | height          | {blockHeight}   |
| block_gas  | amount          | {blockGasUsed}  |


## Parameters

The `x/feemarket` module contains the following parameters:

| Key                      | Type    | Default Values | Description                                                                                                             |
|--------------------------|---------|----------------|-------------------------------------------------------------------------------------------------------------------------|
| NoBaseFee                | bool    | false          | control the base fee adjustment                                                                                         |
| BaseFeeChangeDenominator | uint32  | 8              | bounds the amount the base fee that can change between blocks                                                           |
| ElasticityMultiplier     | uint32  | 2              | bounds the threshold which the base fee will increase or decrease depending on the total gas used in the previous block |
| BaseFee                  | uint32  | 1000000000     | base fee for EIP-1559 blocks                                                                                            |
| EnableHeight             | uint32  | 0              | height which enable fee adjustment                                                                                      |
| MinGasPrice              | sdk.Dec | 0              | global minimum gas price that needs to be paid to include a transaction in a block                                      |


## Client

### CLI

A user can query and interact with the `feemarket` module using the CLI.

#### Queries

The `query` commands allow users to query `feemarket` state.

```bash
evmosd query feemarket --help
```

##### Base Fee

The `base-fee` command allows users to query the block base fee by height.

```bash
evmosd query feemarket base-fee [flags]
```

Example:

```bash
evmosd query feemarket base-fee ...
```

Example Output:

```
base_fee: "512908936"
```

##### Block Gas

The `block-gas` command allows users to query the block gas by height.

```bash
evmosd query feemarket block-gas [flags]
```

Example:

```bash
evmosd query feemarket block-gas ...
```

Example Output:

```
gas: "21000"
```

##### Params

The `params` command allows users to query the module params.

```bash
evmosd query params subspace [subspace] [key] [flags]
```

Example:

```bash
evmosd query params subspace feemarket ElasticityMultiplier ...
```

Example Output:

```
key: ElasticityMultiplier
subspace: feemarket
value: "2"
```

### gRPC

#### Queries

| Verb   | Method                                  | Description            |
|--------|-----------------------------------------|------------------------|
| `gRPC` | `ethermint.feemarket.v1.Query/Params`   | Get the module params  |
| `gRPC` | `ethermint.feemarket.v1.Query/BaseFee`  | Get the block base fee |
| `gRPC` | `ethermint.feemarket.v1.Query/BlockGas` | Get the block gas used |
| `GET`  | `/ethermint/feemarket/v1/params`        | Get the module params  |
| `GET`  | `/ethermint/feemarket/v1/base_fee`      | Get the block base fee |
| `GET`  | `/ethermint/feemarket/v1/block_gas`     | Get the block gas used |


## AnteHandlers

The `x/feemarket` module provides `AnteDecorator`s that are recursively chained together
into a single [`Antehandler`](https://github.com/cosmos/cosmos-sdk/blob/v0.43.0-alpha1/docs/architecture/adr-010-modular-antehandler.md).
These decorators perform basic validity checks on an Ethereum or Cosmos SDK transaction,
such that it could be thrown out of the transaction Mempool.

Note that the `AnteHandler` is run for every transaction
and called on both `CheckTx` and `DeliverTx`.

### Decorators

### `MinGasPriceDecorator`

Rejects Cosmos SDK transactions with transaction fees lower than `MinGasPrice * GasLimit`.

### `EthMinGasPriceDecorator`

Rejects EVM transactions with transactions fees lower than `MinGasPrice * GasLimit`.

- For `LegacyTx` and `AccessListTx`, the `GasPrice * GasLimit` is used.
- For EIP-1559 (*aka.* `DynamicFeeTx`), the `EffectivePrice * GasLimit` is used.

:::tip
**Note**: For dynamic transactions,
if the `feemarket` formula results in a `BaseFee` that lowers `EffectivePrice < MinGasPrices`,
the users must increase the `GasTipCap` (priority fee) until `EffectivePrice > MinGasPrices`.
Transactions with `MinGasPrices * GasLimit < transaction fee < EffectiveFee`
are rejected by the `feemarket` `AnteHandle`.
:::

### `EthGasConsumeDecorator`

Calculates the effective fees to deduct and the tx priority according to EIP-1559 spec,
then deducts the fees and sets the tx priority in the response.

```
effectivePrice = min(baseFee + tipFeeCap, gasFeeCap)
effectiveTipFee = effectivePrice - baseFee
priority = effectiveTipFee / DefaultPriorityReduction
```

When there are multiple messages in the transaction, choose the lowest priority in them.
