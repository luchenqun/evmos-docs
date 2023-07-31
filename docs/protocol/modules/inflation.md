# `通胀`

## 摘要

`x/inflation` 模块根据 [Evmos Token Model](https://evmos.blog/the-evmos-token-model-edc07014978b) 的分配规则，每日生成新的 Evmos 代币并将其分配给以下部分：

* 抵押奖励 `40%`,
* 团队锁定 `25%`,
* 使用激励 `25%`,
* 社区资金池 `10%`。

该模块取代了当前使用的 Cosmos SDK `x/mint` 模块。

新币的分配鼓励 Evmos 网络中的特定行为。通胀将资金分配给 1) `费用收集账户`（在 sdk `x/auth` 模块中）以增加抵押奖励，2) `x/incentives` 模块账户以提供使用激励的供应，以及 3) 社区资金池（由 sdk `x/distr` 模块管理）以资助支出提案。

## 内容

1. **[概念](#概念)**
2. **[状态](#状态)**
3. **[钩子](#钩子)**
4. **[事件](#事件)**
5. **[参数](#参数)**
6. **[客户端](#客户端)**

## 概念

### 通胀

在权益证明（PoS）区块链中，通胀被用作激励参与网络的工具。通胀会创建并分发新的代币给参与者，参与者可以使用这些代币与协议进行交互，或者将其抵押以赚取奖励并投票支持治理提案。

特别是在网络的早期阶段，抵押奖励较高且与网络进行交互的可能性较少时，通胀可以作为主要工具来激励抵押，从而确保网络的安全性。

随着更多的抵押者加入，网络变得越来越稳定和去中心化。它变得*稳定*，因为资产被锁定而不是通过交易引起价格波动。它变得*去中心化*，因为对治理提案的投票权力分散在更多的人手中。

### Evmos Token Model

Evmos Token Model 通过用户、开发者和验证者之间的平衡激励利益来确保 Evmos 网络的安全性。在该模型中，通胀在维持这种平衡中起着重要作用。初始供应为 2 亿枚代币，第一年通过通胀发行超过 3 亿枚代币，该模型建议通胀呈指数下降，以在前 4 年内发行 10 亿枚 Evmos 代币。

我们实现了两种不同的通胀机制来支持代币模型：

1. 线性通胀用于团队解禁
2. 指数通胀用于质押奖励、使用激励和社区资金池

#### 线性通胀 - 团队解禁

在代币模型中，团队解禁的分发是以最小化应税事件的方式实现的。在创世时，有200M的初始供应分配给“解禁账户”。这个数量等于团队解禁四年后的总通胀分配（`20% * 1B = 200M`）。随着时间的推移，这些账户上的“未解禁”代币以线性速率转换为“解禁”代币。团队成员在“未解禁”代币解锁之前不能委托、转移或执行以太坊交易，而只能使用“解禁”代币。

#### 指数通胀 - **半衰期**

质押奖励、使用激励和社区资金池的通胀分发是通过指数公式实现的，也称为半衰期。

通胀按日进行时期（epoch）进行铸币分发。在365个时期（一年）的时间内，每天都会铸造一定数量的 Evmos 代币，并分配给质押奖励、使用激励和社区资金池。时期的铸币数量取决于模块参数，并在每个时期结束时重新计算。

时期铸币数量的计算按照以下公式进行：

```latex
periodProvision = exponentialDecay       *  bondingIncentive
f(x)            = (a * (1 - r) ^ x + c)  *  (1 + maxVariance * (1 - bondedRatio / bondingTarget))


epochProvision = periodProvision / epochsPerPeriod

where (with default values):
x = variable    = year
a = 300,000,000 = initial value
r = 0.5         = decay factor
c = 9,375,000   = long term supply

bondedRatio   = variable  = fraction of the staking tokens which are currently bonded
maxVariance   = 0.0       = the max amount to increase inflation
bondingTarget = 0.66      = our optimal bonded ratio
```

```latex
Example with bondedRatio = bondingTarget:

period  periodProvision  cumulated      epochProvision
f(0)    309 375 000      309 375 000	 847 602
f(1)    159 375 000      468 750 000	 436 643
f(2)     84 375 000      553 125 000	 231 164
f(3)     46 875 000      600 000 000	 128 424
```

## 状态

### 状态对象

`x/inflation` 模块在状态中保存以下对象：

| 状态对象          | 描述                          | 键           | 值                           | 存储  |
| ----------------- | ----------------------------- | ------------ | ----------------------------- | ----- |
| 时期计数器        | 时期计数器                    | `[]byte{1}`  | `[]byte{period}`             | KV    |
| 时期标识符字节    | 时期标识符字节                | `[]byte{3}`  | `[]byte{epochIdentifier}`    | KV    |
| 每个时期的时期数  | 每个时期的时期数字节          | `[]byte{4}`  | `[]byte{epochsPerPeriod}`    | KV    |
| 跳过的时期数字节  | 跳过的时期数字节              | `[]byte{5}`  | `[]byte{skippedEpochs}`      | KV    |

#### 周期

计数器，用于跟踪过去周期的数量，基于每个周期的时期数。

#### 时期标识符

触发时期钩子的标识符。

#### 每个周期的时期数

一个周期中的时期数。

### 创世状态

`x/inflation` 模块的 `GenesisState` 定义了从先前导出的高度初始化链所需的状态。它包含模块参数和活动激励以及它们对应的燃气计量器的列表：

```go
type GenesisState struct {
	// params defines all the parameters of the module.
	Params Params `protobuf:"bytes,1,opt,name=params,proto3" json:"params"`
	// amount of past periods, based on the epochs per period param
	Period uint64 `protobuf:"varint,2,opt,name=period,proto3" json:"period,omitempty"`
	// inflation epoch identifier
	EpochIdentifier string `protobuf:"bytes,3,opt,name=epoch_identifier,json=epochIdentifier,proto3" json:"epoch_identifier,omitempty"`
	// number of epochs after which inflation is recalculated
	EpochsPerPeriod int64 `protobuf:"varint,4,opt,name=epochs_per_period,json=epochsPerPeriod,proto3" json:"epochs_per_period,omitempty"`
	// number of epochs that have passed while inflation is disabled
	SkippedEpochs uint64 `protobuf:"varint,5,opt,name=skipped_epochs,json=skippedEpochs,proto3" json:"skipped_epochs,omitempty"`
}
```

## 钩子

`x/inflation` 模块实现了 `x/epoch` 模块的 `AfterEpochEnd` 钩子，以分配通胀。

### 时期钩子：通胀

时期钩子处理通胀逻辑，它在每个时期结束时运行。它负责铸造和分配时期铸币供应，并进行更新：

1. 检查通胀是否已禁用。如果已禁用，则跳过通胀，增加跳过的时期数，并在不进行下一步的情况下返回。
2. 提交一个区块，表示一个 `时期` 已结束（区块 `header.Time` 超过了 `epoch_start` + `epochIdentifier`）。
3. 铸造金额为计算得到的 `epochMintProvision` 的币，并根据通胀分配到质押奖励、使用激励和社区池。
4. 如果当前时期结束，将周期增加 `1` 并将新值设置到存储中。

## 事件

`x/inflation` 模块发出以下事件：

### 通胀

| 类型        | 属性键               | 属性值                                         |
| ----------- |----------------------|-----------------------------------------------|
| `inflation` | `"epoch_provisions"` | `{fmt.Sprintf("%d", epochNumber)}`            |
| `inflation` | `"epoch_number"`     | `{strconv.FormatUint(uint64(in.Epochs), 10)}` |
| `inflation` | `"amount"`           | `{mintedCoin.Amount.String()}`                |

## 参数

`x/inflation` 模块包含以下描述的参数。所有参数都可以通过治理进行修改。

| 键                                   | 类型                   | 默认值                                                                       |
| ------------------------              | ---------------------- | ----------------------------------------------------------------------------- |
| `ParamStoreKeyMintDenom`              | string                 | `evm.DefaultEVMDenom` // “aevmos”                                             |
| `ParamStoreKeyExponentialCalculation` | ExponentialCalculation | `A: sdk.NewDec(int64(300_000_000))`                                           |
|                                       |                        | `R: sdk.NewDecWithPrec(50, 2)`                                                |
|                                       |                        | `C: sdk.NewDec(int64(9_375_000))`                                             |
|                                       |                        | `BondingTarget: sdk.NewDecWithPrec(66, 2)`                                    |
|                                       |                        | `MaxVariance: sdk.ZeroDec()`                                                  |
| `ParamStoreKeyInflationDistribution`  | InflationDistribution  | `StakingRewards: sdk.NewDecWithPrec(533333334, 9)`  // 0.53 = 40% / (1 - 25%) |
|                                       |                        | `UsageIncentives: sdk.NewDecWithPrec(333333333, 9)` // 0.33 = 25% / (1 - 25%) |
|                                       |                        | `CommunityPool: sdk.NewDecWithPrec(133333333, 9)`  // 0.13 = 10% / (1 - 25%)  |
| `ParamStoreKeyEnableInflation`        | bool                   | `true`                                                                        |

### 铸币单位

`ParamStoreKeyMintDenom` 参数设置了新币铸造的单位。

### 指数计算

`ParamStoreKeyExponentialCalculation` 参数保存了计算 `epochMintProvision` 所需的所有值。值 `A`、`R` 和 `C` 描述了通胀随时间的减少情况。`BondingTarget` 和 `MaxVariance` 允许通胀增加，这是由网络中抵押代币的比例自动调节的。具体的计算公式可以在[概念](#concepts)中找到。

### 通胀分配

`ParamStoreKeyInflationDistribution` 参数定义了通胀在每个时期通过铸币分配的方式（`stakingRewards`、`usageIncentives`、`CommunityPool`）。`x/inflation` 不包括团队锁定分配，因为团队锁定在创世时只铸造一次。为了反映这一点，Evmos Token Model 的分配被重新计算为不包括团队锁定的分配。请注意，这不会改变 Evmos Token Model 中提议的通胀。每个 `InflationDistribution` 可以按照以下方式计算：

```markdown
stakingRewards = evmosTokenModelDistribution / (1 - teamVestingDistribution)
0.5333333      = 40%                         / (1 - 25%)
```

### 启用通胀

`ParamStoreKeyEnableInflation` 参数启用每日通胀。如果禁用了通胀，则不会铸造任何代币，并且每个经过的时期的跳过次数会增加。

## 客户端

用户可以使用 CLI、JSON-RPC、gRPC 或 REST 查询 `x/incentives` 模块。

### CLI

以下是添加了 `x/inflation` 模块的 `evmosd` 命令列表。您可以使用 `evmosd -h` 命令获取完整列表。

#### 查询

`query` 命令允许用户查询 `inflation` 状态。

**`period`**

允许用户查询当前通胀周期。

```bash
evmosd query inflation period [flags]
```

**`epoch-mint-provision`**

允许用户查询当前通胀时期的 provisions 值。

```bash
evmosd查询通胀周期铸币供应量[标志]

**`skipped-epochs`**

允许用户查询当前跳过的周期数。

```bash
evmosd查询通胀跳过的周期[标志]
```

**`total-supply`**

允许用户查询流通中的代币总供应量。

```bash
evmosd查询通胀总供应量[标志]
```

**`inflation-rate`**

允许用户查询当前周期的通胀率。

```bash
evmosd查询通胀通胀率[标志]
```

**`params`**

允许用户查询当前通胀参数。

```bash
evmosd查询通胀参数[标志]
```

#### 提案

`tx gov submit-legacy-proposal`命令允许用户使用治理模块CLI创建提案：

**`param-change`**

允许用户提交`ParameterChangeProposal`。

```bash
evmosd tx gov submit-legacy-proposal param-change [proposal-file] [标志]
```

### gRPC

#### 查询

| 动词   | 方法                                          | 描述                                       |
| ------ | --------------------------------------------- | ------------------------------------------ |
| `gRPC` | `evmos.inflation.v1.Query/Period`             | 获取当前通胀周期                          |
| `gRPC` | `evmos.inflation.v1.Query/EpochMintProvision` | 获取当前通胀周期的铸币供应量              |
| `gRPC` | `evmos.inflation.v1.Query/Params`             | 获取当前通胀参数                          |
| `gRPC` | `evmos.inflation.v1.Query/SkippedEpochs`      | 获取当前跳过的周期数                      |
| `gRPC` | `evmos.inflation.v1.Query/TotalSupply`        | 获取当前总供应量                          |
| `gRPC` | `evmos.inflation.v1.Query/InflationRate`      | 获取当前通胀率                            |
| `GET`  | `/evmos/inflation/v1/period`                  | 获取当前通胀周期                          |
| `GET`  | `/evmos/inflation/v1/epoch_mint_provision`    | 获取当前通胀周期的铸币供应量              |
| `GET`  | `/evmos/inflation/v1/skipped_epochs`          | 获取当前跳过的周期数                      |
| `GET`  | `/evmos/inflation/v1/total_supply`          | 获取当前总供应量                          |
| `GET`  | `/evmos/inflation/v1/inflation_rate`          | 获取当前通胀率                            |
| `GET`  | `/evmos/inflation/v1/params`                  | 获取当前通胀参数                          |
```


Please paste the Markdown content here.


# `inflation`

## Abstract

The `x/inflation` module mints new Evmos tokens and allocates them in daily
epochs according to the [Evmos Token
Model](https://evmos.blog/the-evmos-token-model-edc07014978b) distribution to

* Staking Rewards `40%`,
* Team Vesting `25%`,
* Usage Incentives: `25%`,
* Community Pool `10%`.

It replaces the currently used Cosmos SDK `x/mint` module.

The allocation of new coins incentivizes specific behaviour in the Evmos
network. Inflation allocates funds to 1) the `Fee Collector account` (in the sdk
`x/auth` module) to increase staking rewards, 2) the  `x/incentives` module
account  to provide supply for usage incentives and 3) the community pool
(managed by sdk `x/distr` module) to fund spending proposals.

## Contents

1. **[Concepts](#concepts)**
2. **[State](#state)**
3. **[Hooks](#hooks)**
4. **[Events](#events)**
5. **[Parameters](#parameters)**
6. **[Clients](#clients)**

## Concepts

### Inflation

In a Proof of Stake (PoS) blockchain, inflation is used as a tool to incentivize
participation in the network. Inflation creates and distributes new tokens to
participants who can use their tokens to either interact with the protocol or
stake their assets to earn rewards and vote for governance proposals.

Especially in an early stage of a network, where staking rewards are high and
there are fewer possibilities to interact with the network, inflation can be
used as the major tool to incentivize staking and thereby securing the network.

With more stakers, the network becomes increasingly stable and decentralized. It
becomes *stable*, because assets are locked up instead of causing price changes
through trading. And it becomes *decentralized,* because the power to vote for
governance proposals is distributed amongst more people.

### Evmos Token Model

The Evmos Token Model outlines how the Evmos network is secured through a
balanced incentivized interest from users, developers and validators. In this
model, inflation plays a major role in sustaining this balance. With an initial
supply of 200 million and over 300 million tokens being issued through inflation
during the first year, the model suggests an exponential decline in inflation to
issue 1 billion Evmos tokens within the first 4 years.

We implement two different inflation mechanisms to support the token model:

1. linear inflation for team vesting and
2. exponential inflation for staking rewards, usage incentives and community
   pool.

#### Linear Inflation - Team Vesting

The Team Vesting distribution in the Token Model is implemented in a way that
minimized the amount of taxable events. An initial supply of 200M allocated to
`vesting accounts` at genesis. This amount is equal to the total inflation
allocated for team vesting after 4 years (`20% * 1B = 200M`). Over time,
`unvested` tokens on these accounts are converted into `vested` tokens at a
linear rate. Team members cannot delegate, transfer or execute Ethereum
transaction with `unvested` tokens until they are unlocked represented as
`vested` tokens.

#### Exponential Inflation - **The Half Life**

The inflation distribution for staking, usage incentives and community pool is
implemented through an exponential formula, a.k.a. the Half Life.

Inflation is minted in daily epochs. During a period of 365 epochs (one year), a
daily provision (`epochProvison`) of Evmos tokens is minted and allocated to staking rewards,
usage incentives and the community pool.
The epoch provision depends on module parameters and is recalculated at the end of every epoch.

The calculation of the epoch provision is done according to the following formula:

```latex
periodProvision = exponentialDecay       *  bondingIncentive
f(x)            = (a * (1 - r) ^ x + c)  *  (1 + maxVariance * (1 - bondedRatio / bondingTarget))


epochProvision = periodProvision / epochsPerPeriod

where (with default values):
x = variable    = year
a = 300,000,000 = initial value
r = 0.5         = decay factor
c = 9,375,000   = long term supply

bondedRatio   = variable  = fraction of the staking tokens which are currently bonded
maxVariance   = 0.0       = the max amount to increase inflation
bondingTarget = 0.66      = our optimal bonded ratio
```

```latex
Example with bondedRatio = bondingTarget:

period  periodProvision  cumulated      epochProvision
f(0)    309 375 000      309 375 000	 847 602
f(1)    159 375 000      468 750 000	 436 643
f(2)     84 375 000      553 125 000	 231 164
f(3)     46 875 000      600 000 000	 128 424
```


## State

### State Objects

The `x/inflation` module keeps the following objects in state:

| State Object       | Description                    | Key         | Value                        | Store |
| ------------------ | ------------------------------ | ----------- | ---------------------------- | ----- |
| Period             | Period Counter                 | `[]byte{1}` | `[]byte{period}`             | KV    |
| EpochIdentifier    | Epoch identifier bytes         | `[]byte{3}` | `[]byte{epochIdentifier}`    | KV    |
| EpochsPerPeriod    | Epochs per period bytes        | `[]byte{4}` | `[]byte{epochsPerPeriod}`    | KV    |
| SkippedEpochs      | Number of skipped epochs bytes | `[]byte{5}` | `[]byte{skippedEpochs}`      | KV    |

#### Period

Counter to keep track of amount of past periods, based on the epochs per period.

#### EpochIdentifier

Identifier to trigger epoch hooks.

#### EpochsPerPeriod

Amount of epochs in one period

### Genesis State

The `x/inflation` module's `GenesisState` defines the state necessary for
initializing the chain from a previously exported height. It contains the module
parameters and the list of active incentives and their corresponding gas meters
:

```go
type GenesisState struct {
	// params defines all the parameters of the module.
	Params Params `protobuf:"bytes,1,opt,name=params,proto3" json:"params"`
	// amount of past periods, based on the epochs per period param
	Period uint64 `protobuf:"varint,2,opt,name=period,proto3" json:"period,omitempty"`
	// inflation epoch identifier
	EpochIdentifier string `protobuf:"bytes,3,opt,name=epoch_identifier,json=epochIdentifier,proto3" json:"epoch_identifier,omitempty"`
	// number of epochs after which inflation is recalculated
	EpochsPerPeriod int64 `protobuf:"varint,4,opt,name=epochs_per_period,json=epochsPerPeriod,proto3" json:"epochs_per_period,omitempty"`
	// number of epochs that have passed while inflation is disabled
	SkippedEpochs uint64 `protobuf:"varint,5,opt,name=skipped_epochs,json=skippedEpochs,proto3" json:"skipped_epochs,omitempty"`
}
```


## Hooks

The `x/inflation` module implements the `AfterEpochEnd`  hook from the
`x/epoch` module in order to allocate inflation.

### Epoch Hook: Inflation

The epoch hook handles the inflation logic which is run at the end of each
epoch. It is responsible for minting and allocating the epoch mint provision as
well as updating it:

1. Check if inflation is disabled. If it is, skip inflation, increment number
   of skipped epochs and return without proceeding to the next steps.
2. A block is committed, that signalizes that an `epoch` has ended (block
   `header.Time` has surpassed `epoch_start` + `epochIdentifier`).
3. Mint coin in amount of calculated `epochMintProvision` and allocate according to
   inflation distribution to staking rewards, usage incentives and community
   pool.
4. If a period ends with the current epoch, increment the period by `1` and set new value to the store.


## Events

The `x/inflation` module emits the following events:

### Inflation

| Type        | Attribute Key        | Attribute Value                               |
| ----------- |----------------------|-----------------------------------------------|
| `inflation` | `"epoch_provisions"` | `{fmt.Sprintf("%d", epochNumber)}`            |
| `inflation` | `"epoch_number"`     | `{strconv.FormatUint(uint64(in.Epochs), 10)}` |
| `inflation` | `"amount"`           | `{mintedCoin.Amount.String()}`                |


## Parameters

The `x/inflation` module contains the parameters described below. All parameters
can be modified via governance.

| Key                                   | Type                   | Default Value                                                                 |
| ------------------------              | ---------------------- | ----------------------------------------------------------------------------- |
| `ParamStoreKeyMintDenom`              | string                 | `evm.DefaultEVMDenom` // “aevmos”                                             |
| `ParamStoreKeyExponentialCalculation` | ExponentialCalculation | `A: sdk.NewDec(int64(300_000_000))`                                           |
|                                       |                        | `R: sdk.NewDecWithPrec(50, 2)`                                                |
|                                       |                        | `C: sdk.NewDec(int64(9_375_000))`                                             |
|                                       |                        | `BondingTarget: sdk.NewDecWithPrec(66, 2)`                                    |
|                                       |                        | `MaxVariance: sdk.ZeroDec()`                                                  |
| `ParamStoreKeyInflationDistribution`  | InflationDistribution  | `StakingRewards: sdk.NewDecWithPrec(533333334, 9)`  // 0.53 = 40% / (1 - 25%) |
|                                       |                        | `UsageIncentives: sdk.NewDecWithPrec(333333333, 9)` // 0.33 = 25% / (1 - 25%) |
|                                       |                        | `CommunityPool: sdk.NewDecWithPrec(133333333, 9)`  // 0.13 = 10% / (1 - 25%)  |
| `ParamStoreKeyEnableInflation`        | bool                   | `true`                                                                        |

### Mint Denom

The `ParamStoreKeyMintDenom` parameter sets the denomination in which new coins are minted.

### Exponential Calculation

The `ParamStoreKeyExponentialCalculation` parameter holds all values required for the
calculation of the `epochMintProvision`. The values `A`, `R` and `C` describe
the decrease of inflation over time. The `BondingTarget` and `MaxVariance`
allow for an increase in inflation, which is automatically regulated by the
`bonded ratio`, the portion of staked tokens in the network. The exact formula
can be found under
[Concepts](#concepts).

### Inflation Distribution

The `ParamStoreKeyInflationDistribution` parameter defines the distribution in which
inflation is allocated through minting on each epoch (`stakingRewards`,
`usageIncentives`,  `CommunityPool`). The `x/inflation` excludes the team
vesting distribution, as team vesting is minted once at genesis. To reflect this
the distribution from the Evmos Token Model is recalculated into a distribution
that excludes team vesting. Note, that this does not change the inflation
proposed in the Evmos Token Model. Each `InflationDistribution` can be
calculated like this:

```markdown
stakingRewards = evmosTokenModelDistribution / (1 - teamVestingDistribution)
0.5333333      = 40%                         / (1 - 25%)
```

### Enable Inflation

The `ParamStoreKeyEnableInflation` parameter enables the daily inflation. If it is disabled,
no tokens are minted and the number of skipped epochs increases for each passed
epoch.


## Clients

A user can query the `x/incentives` module using the CLI, JSON-RPC, gRPC or
REST.

### CLI

Find below a list of `evmosd` commands added with the `x/inflation` module. You
can obtain the full list by using the `evmosd -h` command.

#### Queries

The `query` commands allow users to query `inflation` state.

**`period`**

Allows users to query the current inflation period.

```bash
evmosd query inflation period [flags]
```

**`epoch-mint-provision`**

Allows users to query the current inflation epoch provisions value.

```bash
evmosd query inflation epoch-mint-provision [flags]
```

**`skipped-epochs`**

Allows users to query the current number of skipped epochs.

```bash
evmosd query inflation skipped-epochs [flags]
```

**`total-supply`**

Allows users to query the total supply of tokens in circulation.

```bash
evmosd query inflation total-supply [flags]
```

**`inflation-rate`**

Allows users to query the inflation rate of the current period.

```bash
evmosd query inflation inflation-rate [flags]
```

**`params`**

Allows users to query the current inflation parameters.

```bash
evmosd query inflation params [flags]
```

#### Proposals

The `tx gov submit-legacy-proposal` commands allow users to query create a proposal
using the governance module CLI:

**`param-change`**

Allows users to submit a `ParameterChangeProposal`.

```bash
evmosd tx gov submit-legacy-proposal param-change [proposal-file] [flags]
```

### gRPC

#### Queries

| Verb   | Method                                        | Description                                   |
| ------ | --------------------------------------------- | --------------------------------------------- |
| `gRPC` | `evmos.inflation.v1.Query/Period`             | Gets current inflation period                 |
| `gRPC` | `evmos.inflation.v1.Query/EpochMintProvision` | Gets current inflation epoch provisions value |
| `gRPC` | `evmos.inflation.v1.Query/Params`             | Gets current inflation parameters             |
| `gRPC` | `evmos.inflation.v1.Query/SkippedEpochs`      | Gets current number of skipped epochs         |
| `gRPC` | `evmos.inflation.v1.Query/TotalSupply`        | Gets current total supply                     |
| `gRPC` | `evmos.inflation.v1.Query/InflationRate`      | Gets current inflation rate                   |
| `GET`  | `/evmos/inflation/v1/period`                  | Gets current inflation period                 |
| `GET`  | `/evmos/inflation/v1/epoch_mint_provision`    | Gets current inflation epoch provisions value |
| `GET`  | `/evmos/inflation/v1/skipped_epochs`          | Gets current number of skipped epochs         |
| `GET`  | `/evmos/inflation/v1/total_supply`          | Gets current total supply                     |
| `GET`  | `/evmos/inflation/v1/inflation_rate`          | Gets current inflation rate                   |
| `GET`  | `/evmos/inflation/v1/params`                  | Gets current inflation parameters             |
