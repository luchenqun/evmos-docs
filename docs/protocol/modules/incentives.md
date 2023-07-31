# `激励`

## 摘要

本文档规定了 Evmos Hub 的内部 `x/incentives` 模块。

`x/incentives` 模块是 Evmos 代币经济的一部分，旨在通过向与激励智能合约进行交互的用户分发奖励来增加网络的增长。
这些奖励鼓励用户与 Evmos 上的应用程序进行交互，并将奖励再投资于网络中的更多服务。

激励的使用来自区块奖励的发行（通胀），并在激励模块账户（托管地址）中汇集起来。
激励功能完全由原生的 EVMOS 代币持有者管理，
他们负责注册 `Incentives`，因此原生的 EVMOS 代币持有者决定哪些应用程序应该成为使用激励的一部分。
这个治理功能是使用 Cosmos-SDK 的 `gov` 模块实现的，
使用自定义的提案类型来注册激励。

用户通过向一个有激励的合约提交交易来参与激励。
该模块会记录参与者在交易中花费的燃料量，并将其存储在燃料计量器中。
根据他们的燃料计量器，激励参与者会在固定的时间间隔（时期）内获得奖励。

## 内容

1. **[概念](#概念)**
2. **[状态](#状态)**
3. **[状态转换](#状态转换)**
4. **[交易](#交易)**
5. **[钩子](#钩子)**
6. **[事件](#事件)**
7. **[参数](#参数)**
8. **[客户端](#客户端)**

## 概念

### 激励

`x/incentives` 模块的目的是为与智能合约进行交互的用户提供激励。
激励允许用户获得最多 `rewards = k * sum(tx fees)` 的奖励，
其中 `k` 是一个奖励缩放参数，通过将其与用户在当前时期中花费的交易费用总和相乘，
限制了分配给单个用户的激励。

`激励` 描述了为给定智能合约分配和分发奖励的条件。
在每个时期结束时，奖励从通胀池中分配给激励的参与者，
取决于每个参与者花费的燃料量和缩放参数。

给定智能合约的激励可以通过治理来启用或禁用。

### 通胀池

通胀池持有可以分配给激励的“奖励”。
在每个区块上，通胀奖励会被铸造并添加到通胀池中。
此外，奖励也可以在通胀之外转移到通胀池中。
有关如何将奖励添加到通胀池的详细信息，请参阅`x/inflation`模块。

### 纪元

为智能合约交互奖励用户是按照纪元进行组织的。
一个“纪元”是一个固定的时间段，在此期间奖励将被添加到通胀池中，并记录智能合约的交互。
在纪元结束时，奖励将被分配并分发给所有参与者。
这样，用户可以定期检查他们的余额以获取新的奖励（例如每天的同一时间）。

### 分配

在奖励分发给用户之前，每个激励都从通胀池中分配奖励。
“分配”描述了通胀池中分配给指定币种激励的奖励部分。

用户可以以多种币种获得奖励。
这些币种以“分配”方式组织。
一个分配包括币种面额和从通胀池中分配的奖励百分比。

- 每个分配的奖励百分比有一个上限。
  它由链参数定义，并可以通过治理进行修改。
- 激励的数量受限于所有活跃的激励合约分配的总和。
  如果总和大于100％，则无法提出更多的激励，直到另一个分配变为非活跃状态。

### 分发

激励的分配奖励根据参与者在纪元期间与合约交互所消耗的燃料量进行分发。
每个地址使用的燃料量通过事务钩子记录并存储在KV存储中。
在纪元结束时，激励中分配的奖励将通过将其转移到参与者的账户上进行分发。

:::tip
💡我们使用钩子（hooks）而不是交易哈希来测量消耗的 gas，因为钩子可以访问实际消耗的 gas，而哈希只包括 gas 限制。
:::


## 状态

### 状态对象

`x/incentives` 模块在状态中保存以下对象：

| 状态对象         | 描述                                          | 键                                                     | 值                   | 存储  |
| --------------- | --------------------------------------------- | ------------------------------------------------------ | ------------------- | ----- |
| Incentive       | 激励合约字节码                                | `[]byte{1} + []byte(contract)`                         | `[]byte{incentive}` | KV    |
| GasMeter        | 根据 erc20 合约字节和参与者的激励 id 字节码     | `[]byte{2} + []byte(contract) + []byte(participant)` | `[]byte{gasMeter}`  | KV    |
| AllocationMeter | 根据代币字节的总分配字节                       | `[]byte{3} + []byte(denom)`                            | `[]byte{sdk.Dec}`   | KV    |

#### Incentive

为给定的智能合约组织分配条件的实例。

```go
type Incentive struct {
	// contract address
	Contract string `protobuf:"bytes,1,opt,name=contract,proto3" json:"contract,omitempty"`
	// denoms and percentage of rewards to be allocated
	Allocations github_com_cosmos_cosmos_sdk_types.DecCoins `protobuf:"bytes,2,rep,name=allocations,proto3,castrepeated=github.com/cosmos/cosmos-sdk/types.DecCoins" json:"allocations"`
	// number of remaining epochs
	Epochs uint32 `protobuf:"varint,3,opt,name=epochs,proto3" json:"epochs,omitempty"`
	// distribution start time
	StartTime time.Time `protobuf:"bytes,4,opt,name=start_time,json=startTime,proto3,stdtime" json:"start_time"`
	// cumulative gas spent by all gasmeters of the incentive during the epoch
	TotalGas uint64 `protobuf:"varint,5,opt,name=total_gas,json=totalGas,proto3" json:"total_gas,omitempty"`
}
```

只要激励还有剩余的时期，它就会根据其分配来分发奖励。
分配以 `sdk.DecCoins` 的形式存储，其中每个包含 [`sdk.DecCoin`](https://github.com/cosmos/cosmos-sdk/blob/master/types/dec_coin.go) 的对象描述了分配给合约的奖励的百分比（`Amount`），
该百分比分配给给定币种（`Denom`）。
一个激励可以包含多个分配，导致用户以多个不同的币种形式接收奖励。

#### GasMeter

跟踪每个参与者在一个时期内在合约中消耗的累积 gas。

```go
type GasMeter struct {
	// hex address of the incentivized contract
	Contract string `protobuf:"bytes,1,opt,name=contract,proto3" json:"contract,omitempty"`
	// participant address that interacts with the incentive
	Participant string `protobuf:"bytes,2,opt,name=participant,proto3" json:"participant,omitempty"`
	// cumulative gas spent during the epoch
	CumulativeGas uint64 `protobuf:"varint,3,opt,name=cumulative_gas,json=cumulativeGas,proto3" json:"cumulative_gas,omitempty"`
}
```

#### AllocationMeter

分配计量器存储了给定币种的所有已注册激励的分配总和，并用于限制已注册激励的数量。

假设有几个激励已经为 EVMOS 代币注册了分配，并且 EVMOS 的分配计量器为 97%。
那么一个新的激励提案只能包括最多 3% 的 EVMOS 分配，
从通胀池中的 EVMOS 奖励中获取剩余的分配容量。

### 创世状态

`x/incentives` 模块的 `GenesisState` 定义了从先前导出的高度初始化链所需的状态。它包含了模块参数以及活跃激励和相应的燃气计量器的列表：

```go
// GenesisState defines the module's genesis state.
type GenesisState struct {
	// module parameters
	Params Params `protobuf:"bytes,1,opt,name=params,proto3" json:"params"`
	// active incentives
	Incentives []Incentive `protobuf:"bytes,2,rep,name=incentives,proto3" json:"incentives"`
	// active Gasmeters
	GasMeters []GasMeter `protobuf:"bytes,3,rep,name=gas_meters,json=gasMeters,proto3" json:"gas_meters"`
}
```


## 状态转换

`x/incentive` 模块允许两种类型的注册状态转换：`RegisterIncentiveProposal` 和 `CancelIncentiveProposal`。*燃气计量*和*奖励分发*的逻辑通过[Hooks](#hooks)处理。

### 激励注册

用户注册一个激励，定义合约、分配和时期数量。一旦提案通过（即由治理机构批准），激励模块将创建激励并分发奖励。

1. 用户提交一个 `RegisterIncentiveProposal`。
2. Evmos Hub 的验证人使用 `MsgVote` 对提案进行投票，并通过。
3. 如果满足以下条件，则为合约创建激励，`TotalGas = 0`，并将其 `startTime` 设置为 `ctx.Blocktime`：
    1. 全局启用了激励参数
    2. 激励尚未注册
    3. 除了铸币单位之外，通胀池中的余额对于每个分配货币都大于 0。
       我们知道铸币单位（例如 EVMOS）的金额将添加到每个区块中，
       但对于其他货币单位（使用 `x/erc20` 模块的 IBC 凭证、ERC20 代币），
       模块账户需要有正数金额来分发激励。
    4. 每个货币单位的所有已注册分配的总和（当前 + 提议）小于 100%


## 交易

本节定义了导致前一节中定义的状态转换的 `sdk.Msg` 具体类型。

## `RegisterIncentiveProposal`

用于为给定合约注册激励的治理 `Content` 类型，持续一定数量的时期。治理用户对此提案进行投票，
并在投票通过时自动执行 `RegisterIncentiveProposal` 的自定义处理程序。

```go
type RegisterIncentiveProposal struct {
	// title of the proposal
	Title string `protobuf:"bytes,1,opt,name=title,proto3" json:"title,omitempty"`
	// proposal description
	Description string `protobuf:"bytes,2,opt,name=description,proto3" json:"description,omitempty"`
	// contract address
	Contract string `protobuf:"bytes,3,opt,name=contract,proto3" json:"contract,omitempty"`
	// denoms and percentage of rewards to be allocated
	Allocations github_com_cosmos_cosmos_sdk_types.DecCoins `protobuf:"bytes,4,rep,name=allocations,proto3,castrepeated=github.com/cosmos/cosmos-sdk/types.DecCoins" json:"allocations"`
	// number of remaining epochs
	Epochs uint32 `protobuf:"varint,5,opt,name=epochs,proto3" json:"epochs,omitempty"`
}
```

提案内容的无状态验证失败的情况包括：

- 标题无效（长度或字符）
- 描述无效（长度或字符）
- 合约地址无效
- 分配无效
    - 分配中没有包含任何分配
    - 至少一个分配的金额无效（低于0或高于1）
- 周期无效（为零）

## `CancelIncentiveProposal`

一种用于移除激励的治理 `Content` 类型。
当投票通过时，治理用户对此提案进行投票，并自动执行 `CancelIncentiveProposal` 的自定义处理程序。

```go
type CancelIncentiveProposal struct {
	// title of the proposal
	Title string `protobuf:"bytes,1,opt,name=title,proto3" json:"title,omitempty"`
	// proposal description
	Description string `protobuf:"bytes,2,opt,name=description,proto3" json:"description,omitempty"`
	// contract address
	Contract string `protobuf:"bytes,3,opt,name=contract,proto3" json:"contract,omitempty"`
}
```

提案内容的无状态验证失败的情况包括：

- 标题无效（长度或字符）
- 描述无效（长度或字符）
- 合约地址无效


## 钩子函数

`x/incentives` 模块实现了来自 `x/evm` 和 `x/epoch` 模块的两个事务钩子。

### EVM 钩子 - Gas 计量

EVM 钩子更新日志，用于跟踪在一个周期内与激励合约的交互中使用的 gas 数量。
在每个成功的 evm 事务之后，[EVM 钩子](evm.md#hooks) 执行自定义逻辑。
在这种情况下，它更新了激励的总 gas 数量和参与者自己的 gas 数量。

1. 用户向激励智能合约提交 EVM 事务，并成功完成该事务。
2. EVM 钩子的 `PostTxProcessing` 方法在 incentives 模块上被调用。
   它接收一个事务收据，其中包括事务发送者为支付 gas 费用而使用的累计 gas 数量。
   钩子执行以下操作：
    1. 将 `gasUsed` 添加到激励的累计 `totalGas` 中
    2. 将 `gasUsed` 添加到参与者的 gas 计量器的累计 gas 使用量中

### Epoch 钩子 - 奖励分配

Epoch 钩子在每个周期（一天或一周）结束时触发所有已注册激励的使用奖励分配。
该分配过程首先 1) 从分配池为每个激励分配奖励，然后 2) 将这些奖励分配给每个激励的所有参与者。

1. `RegisterIncentiveProposal` 通过，并为提议的合约创建了一个 `incentive`。
2. 一个 `epoch` 开始，并且每个区块用于通胀而铸造的奖励（EVMOS 和其他代币）被添加到通胀池中。
3. 用户提交交易并调用激励智能合约上的函数进行交互，通过 EVM 钩子记录 gas。
4. 一个区块，标志着一个 `epoch` 的结束，被提议，并通过 `AfterEpochEnd` 钩子调用 `DistributeIncentives` 方法。
   该方法执行以下操作：
    1. 从通胀池中分配要分发的金额
    2. 将奖励分配给所有参与者。
       每个参与者的奖励受当前周期内他们在交易费用上花费的 gas 量和奖励缩放参数的限制。
    3. 删除合约的所有 gas 计量器
    4. 更新每个激励的剩余周期。
       如果一个激励的剩余周期为零，则移除该激励并更新分配计量器。
    5. 为下一个周期将累计的 totalGas 设置为零
5. 如果某个币种的分配容量未完全用尽，并且所有活动的激励合约的分配总和小于 100%，则该币种的奖励将在通胀池中累积。
   累积的奖励将在下一个周期的分配中添加。

## 事件

`x/incentives` 模块会触发以下事件：

### 注册激励提案

| 类型                 | 属性键       | 属性值                                      |
| -------------------- | ------------ | ------------------------------------------- |
| `register_incentive` | `"contract"` | `{erc20_address}`                           |
| `register_incentive` | `"epochs"`   | `{strconv.FormatUint(uint64(in.Epochs), 10)}` |

### 取消激励提案

| 类型               | 属性键       | 属性值                |
| ------------------ | ------------ | --------------------- |
| `cancel_incentive` | `"contract"` | `{erc20_address}`     |

### 激励分发

| 类型                    | 属性键       | 属性值                                      |
| ----------------------- | ------------ | ------------------------------------------- |
| `distribute_incentives` | `"contract"` | `{erc20_address}`                           |
| `distribute_incentives` | `"epochs"`   | `{strconv.FormatUint(uint64(in.Epochs), 10)}` |


## 参数

`x/incentives` 模块包含以下参数。所有参数都可以通过治理进行修改。

| 键                           | 类型    | 默认值                            |
| --------------------------- | ------- | ---------------------------------- |
| `EnableIncentives`          | bool    | `true`                             |
| `AllocationLimit`           | sdk.Dec | `sdk.NewDecWithPrec(5,2)` // 5%    |
| `IncentivesEpochIdentifier` | string  | `week`                             |
| `rewardScaler`              | sdk.Dec | `sdk.NewDecWithPrec(12,1)` // 120% |

### 启用激励

`EnableIncentives` 参数用于切换模块中的所有状态转换。
当该参数被禁用时，将阻止所有激励的注册、取消和分发功能。

### 分配限制

`AllocationLimit` 参数定义了每个激励在每个币种中可以定义的最大分配额度。
例如，如果 `AllocationLimit` 为 5%，
则每个币种最多可以有 20 个活跃的激励，如果它们都达到了限制。

奖励百分比每个分配都有上限。

### 激励周期标识符

`IncentivesEpochIdentifier`参数指定了一个周期的长度。
这是激励奖励定期分发的间隔。

### 奖励缩放器

`rewardScaler`参数相对于其使用的燃气定义了每个参与者的奖励限制。
激励允许用户获得最多`rewards = k * sum(txFees)`的奖励，
其中`k`定义了奖励缩放器参数，将其乘以他们在当前周期中花费的交易费用总和，以限制分配给单个用户的激励。

## 客户端

用户可以使用CLI、JSON-RPC、gRPC或REST查询`x/incentives`模块。

### CLI

下面是使用`x/incentives`模块添加的`evmosd`命令列表。
您可以使用`evmosd -h`命令获取完整列表。

#### 查询

`query`命令允许用户查询`incentives`状态。

**`incentives`**

允许用户查询所有已注册的激励。

```go
evmosd query incentives incentives [flags]
```

**`incentive`**

允许用户查询给定合约的激励。

```go
evmosd query incentives incentive CONTRACT_ADDRESS [flags]
```

**`gas-meters`**

允许用户查询给定激励的所有燃气计量器。

```bash
evmosd query incentives gas-meters CONTRACT_ADDRESS [flags]
```

**`gas-meter`**

允许用户查询给定激励和用户的燃气计量器。

```go
evmosd query incentives gas-meter CONTRACT_ADDRESS PARTICIPANT_ADDRESS [flags]
```

**`params`**

允许用户查询激励参数。

```bash
evmosd query incentives params [flags]
```

#### 提议

`tx gov submit-legacy-proposal`命令允许用户使用治理模块CLI创建提议：

**`register-incentive`**

允许用户提交`RegisterIncentiveProposal`。

```bash
evmosd tx gov submit-legacy-proposal register-incentive CONTRACT_ADDRESS ALLOCATION EPOCHS [flags]
```

**`cancel-incentive`**

允许用户提交`CanelIncentiveProposal`。

```bash
evmosd tx gov submit-legacy-proposal cancel-incentive CONTRACT_ADDRESS [flags]
```

**`param-change`**

允许用户提交`ParameterChangeProposal`。

```bash
evmosd tx gov submit-legacy-proposal param-change PROPOSAL_FILE [flags]
```

### gRPC

#### 查询

| 动词   | 方法                                                       | 描述                                         |
| ------ | ---------------------------------------------------------- | --------------------------------------------- |
| `gRPC` | `evmos.incentives.v1.Query/Incentives`                     | 获取所有注册的激励                           |
| `gRPC` | `evmos.incentives.v1.Query/Incentive`                      | 获取给定合约的激励                           |
| `gRPC` | `evmos.incentives.v1.Query/GasMeters`                      | 获取给定激励的燃气计量器                     |
| `gRPC` | `evmos.incentives.v1.Query/GasMeter`                       | 获取给定激励和用户的燃气计量器               |
| `gRPC` | `evmos.incentives.v1.Query/AllocationMeters`               | 获取所有分配计量器                           |
| `gRPC` | `evmos.incentives.v1.Query/AllocationMeter`                | 获取给定货币单位的分配计量器                 |
| `gRPC` | `evmos.incentives.v1.Query/Params`                         | 获取激励参数                                 |
| `GET`  | `/evmos/incentives/v1/incentives`                          | 获取所有注册的激励                           |
| `GET`  | `/evmos/incentives/v1/incentives/{contract}`               | 获取给定合约的激励                           |
| `GET`  | `/evmos/incentives/v1/gas_meters`                          | 获取给定激励的燃气计量器                     |
| `GET`  | `/evmos/incentives/v1/gas_meters/{contract}/{participant}` | 获取给定激励和用户的燃气计量器               |
| `GET`  | `/evmos/incentives/v1/allocation_meters`                   | 获取所有分配计量器                           |
| `GET`  | `/evmos/incentives/v1/allocation_meters/{denom}`           | 获取给定货币单位的分配计量器                 |
| `GET`  | `/evmos/incentives/v1/params`                              | 获取激励参数                                 |
```

I'm sorry, but as an AI text-based model, I am unable to receive or process any files or attachments. However, you can copy and paste the Markdown content here, and I will do my best to translate it for you.


# `incentives`

## Abstract

This document specifies the internal `x/incentives` module of the Evmos Hub.

The `x/incentives` module is part of the Evmos tokenomics and aims
to increase the growth of the network by distributing rewards
to users who interact with incentivized smart contracts.
The rewards drive users to interact with applications on Evmos and reinvest their rewards in more services in the network.

The usage incentives are taken from block reward emission (inflation)
and are pooled up in the Incentives module account (escrow address).
The incentives functionality is fully governed by native EVMOS token holders
who manage the registration of `Incentives`,
so that native EVMOS token holders decide which application should be part of the usage incentives.
This governance functionality is implemented using the Cosmos-SDK `gov` module
with custom proposal types for registering the incentives.

Users participate in incentives by submitting transactions to an incentivized contract.
The module keeps a record of how much gas the participants spent on their transactions and stores these in gas meters.
Based on their gas meters, participants in the incentive are rewarded in regular intervals (epochs).

## Contents

1. **[Concepts](#concepts)**
2. **[State](#state)**
3. **[State Transitions](#state-transitions)**
4. **[Transactions](#transactions)**
5. **[Hooks](#hooks)**
6. **[Events](#events)**
7. **[Parameters](#parameters)**
8. **[Clients](#clients)**

## Concepts

### Incentive

The purpose of the `x/incentives` module is to provide incentives to users who interact with smart contracts.
An incentive allows users to earn rewards up to `rewards = k * sum(tx fees)`,
where `k` defines a reward scaler parameter that caps the incentives allocated to a single user
by multiplying it with the sum of transaction fees
that they’ve spent in the current epoch.

An `incentive` describes the conditions under which rewards are allocated and distributed for a given smart contract.
At the end of every epoch, rewards are allocated from an Inflation pool
and distributed to participants of the incentive, depending on how much gas every participant spent and the scaling parameter.

The incentive for a given smart contract can be enabled or disabled via governance.

### Inflation Pool

The inflation pool holds `rewards` that can be allocated to incentives.
On every block, inflation rewards are minted and added to the inflation pool.
Additionally, rewards may also be transferred to the inflation pool on top of inflation.
The details of how rewards are added to the inflation pool are described in the `x/inflation` module.

### Epoch

Rewarding users for smart contract interaction is organized in epochs.
An `epoch` is a fixed duration in which rewards are added to the inflation pool and smart contract interaction is logged.
At the end of an epoch, rewards are allocated and distributed to all participants.
This creates a user experience, where users check their balance for new rewards regularly (e.g.
every day at the same time).

### Allocation

Before rewards are distributed to users, each incentive allocates rewards from the inflation pool.
The `allocation` describes the portion of rewards in the inflation pool,
that is allocated to an incentive for a specified coin.

Users can be rewarded in several coin denominations.
These are organized in `allocations`.
An allocation includes the coin denomination and the percentage of rewards that are allocated from the inflation pool.

- There is a cap on how high the reward percentage can be per allocation.
  It is defined via the chain parameters and can be modified via governance
- The amount of incentives is limited by the sum of all active incentivized contracts' allocations.
  If the sum is > 100%, no further incentive can be proposed until another allocation becomes inactive.

### Distribution

The allocated rewards for an incentive are distributed
according to how much gas participants spend on interaction with the contract during an epoch.
The gas used per address is recorded using transaction hooks and stored on the KV store.
At the end of an epoch, the allocated rewards in the incentive are distributed
by transferring them to the participants accounts.

:::tip
💡 We use hooks instead of the transaction hash to measure the gas spent
because the hook has access to the actual gas spent and the hash only includes the gas limit.
:::


## State

### State Objects

The `x/incentives` module keeps the following objects in state:

| State Object    | Description                                   | Key                                                    | Value               | Store |
| --------------- | --------------------------------------------- | ------------------------------------------------------ | ------------------- | ----- |
| Incentive       | Incentive bytecode                            | `[]byte{1} + []byte(contract)`                         | `[]byte{incentive}` | KV    |
| GasMeter        | Incentive id bytecode by erc20 contract bytes | `[]byte{2} + []byte(contract) + []byte(participant)` | `[]byte{gasMeter}`  | KV    |
| AllocationMeter | Total allocation bytes by denom bytes         | `[]byte{3} + []byte(denom)`                            | `[]byte{sdk.Dec}`   | KV    |

#### Incentive

An instance that organizes distribution conditions for a given smart contract.

```go
type Incentive struct {
	// contract address
	Contract string `protobuf:"bytes,1,opt,name=contract,proto3" json:"contract,omitempty"`
	// denoms and percentage of rewards to be allocated
	Allocations github_com_cosmos_cosmos_sdk_types.DecCoins `protobuf:"bytes,2,rep,name=allocations,proto3,castrepeated=github.com/cosmos/cosmos-sdk/types.DecCoins" json:"allocations"`
	// number of remaining epochs
	Epochs uint32 `protobuf:"varint,3,opt,name=epochs,proto3" json:"epochs,omitempty"`
	// distribution start time
	StartTime time.Time `protobuf:"bytes,4,opt,name=start_time,json=startTime,proto3,stdtime" json:"start_time"`
	// cumulative gas spent by all gasmeters of the incentive during the epoch
	TotalGas uint64 `protobuf:"varint,5,opt,name=total_gas,json=totalGas,proto3" json:"total_gas,omitempty"`
}
```

As long as an incentive has remaining epochs, it distributes rewards according to its allocations.
The allocations are stored as `sdk.DecCoins` where each containing
[`sdk.DecCoin`](https://github.com/cosmos/cosmos-sdk/blob/master/types/dec_coin.go) describes the percentage of rewards
(`Amount`) that are allocated to the contract for a given coin denomination (`Denom`).
An incentive can contain several allocations, resulting in users to receive rewards in form of several different denominations.

#### GasMeter

Tracks the cumulative gas spent in a contract per participant during one epoch.

```go
type GasMeter struct {
	// hex address of the incentivized contract
	Contract string `protobuf:"bytes,1,opt,name=contract,proto3" json:"contract,omitempty"`
	// participant address that interacts with the incentive
	Participant string `protobuf:"bytes,2,opt,name=participant,proto3" json:"participant,omitempty"`
	// cumulative gas spent during the epoch
	CumulativeGas uint64 `protobuf:"varint,3,opt,name=cumulative_gas,json=cumulativeGas,proto3" json:"cumulative_gas,omitempty"`
}
```

#### AllocationMeter

An allocation meter stores the sum of all registered incentives’ allocations for a given denomination
and is used to limit the amount of registered incentives.

Say, there are several incentives that have registered an allocation for the EVMOS coin
and the allocation meter for EVMOS is at 97%.
Then a new incentve proposal can only include an EVMOS allocation at up to 3%,
claiming the last remaining allocation capacity from the EVMOS rewards in the inflation pool.

### Genesis State

The `x/incentives` module's `GenesisState` defines the state
necessary for initializing the chain from a previously exported height.
It contains the module parameters and the list of active incentives and their corresponding gas meters:

```go
// GenesisState defines the module's genesis state.
type GenesisState struct {
	// module parameters
	Params Params `protobuf:"bytes,1,opt,name=params,proto3" json:"params"`
	// active incentives
	Incentives []Incentive `protobuf:"bytes,2,rep,name=incentives,proto3" json:"incentives"`
	// active Gasmeters
	GasMeters []GasMeter `protobuf:"bytes,3,rep,name=gas_meters,json=gasMeters,proto3" json:"gas_meters"`
}
```


## State Transitions

The `x/incentive` module allows for two types of registration state transitions:
`RegisterIncentiveProposal` and `CancelIncentiveProposal`.
The logic for *gas metering* and *distributing rewards* is handled through [Hooks](#hooks).

### Incentive Registration

A user registers an incentive defining the contract, allocations, and number of epochs.
Once the proposal passes (i.e is approved by governance),
the incentive module creates the incentive and distributes rewards.

1. User submits a `RegisterIncentiveProposal`.
2. Validators of the Evmos Hub vote on the proposal using `MsgVote` and proposal passes.
3. Create incentive for the contract with a `TotalGas = 0` and set its `startTime` to `ctx.Blocktime`
   if the following conditions are met:
    1. Incentives param is globally enabled
    2. Incentive is not yet registered
    3. Balance in the inflation pool is > 0 for each allocation denom except for the mint denomination.
       We know that the amount of the minting denom (e.g. EVMOS) will be added to every block
       but for other denominations (IBC vouchers, ERC20 tokens using the `x/erc20` module)
       the module account needs to have a positive amount to distribute the incentives
    4. The sum of all registered allocations for each denom (current + proposed) is < 100%


## Transactions

This section defines the `sdk.Msg` concrete types that result in the state transitions defined on the previous section.

## `RegisterIncentiveProposal`

A gov `Content` type to register an Incentive for a given contract for the duration of a certain number of epochs.
Governance users vote on this proposal
and it automatically executes the custom handler for `RegisterIncentiveProposal` when the vote passes.

```go
type RegisterIncentiveProposal struct {
	// title of the proposal
	Title string `protobuf:"bytes,1,opt,name=title,proto3" json:"title,omitempty"`
	// proposal description
	Description string `protobuf:"bytes,2,opt,name=description,proto3" json:"description,omitempty"`
	// contract address
	Contract string `protobuf:"bytes,3,opt,name=contract,proto3" json:"contract,omitempty"`
	// denoms and percentage of rewards to be allocated
	Allocations github_com_cosmos_cosmos_sdk_types.DecCoins `protobuf:"bytes,4,rep,name=allocations,proto3,castrepeated=github.com/cosmos/cosmos-sdk/types.DecCoins" json:"allocations"`
	// number of remaining epochs
	Epochs uint32 `protobuf:"varint,5,opt,name=epochs,proto3" json:"epochs,omitempty"`
}
```

The proposal content stateless validation fails if:

- Title is invalid (length or char)
- Description is invalid (length or char)
- Contract address is invalid
- Allocations are invalid
    - no allocation included in Allocations
    - invalid amount of at least one allocation (below 0 or above 1)
- Epochs are invalid (zero)

## `CancelIncentiveProposal`

A gov `Content` type to remove an Incentive.
Governance users vote on this proposal
and it automatically executes the custom handler for `CancelIncentiveProposal` when the vote passes.

```go
type CancelIncentiveProposal struct {
	// title of the proposal
	Title string `protobuf:"bytes,1,opt,name=title,proto3" json:"title,omitempty"`
	// proposal description
	Description string `protobuf:"bytes,2,opt,name=description,proto3" json:"description,omitempty"`
	// contract address
	Contract string `protobuf:"bytes,3,opt,name=contract,proto3" json:"contract,omitempty"`
}
```

The proposal content stateless validation fails if:

- Title is invalid (length or char)
- Description is invalid (length or char)
- Contract address is invalid


## Hooks

The `x/incentives` module implements two transaction hooks from the `x/evm` and `x/epoch` modules.

### EVM Hook - Gas Metering

The EVM hook updates the logs that keep track of much gas was used
for interacting with an incentivized contract during one epoch.
An [EVM hook](evm.md#hooks) executes custom logic
after each successful evm transaction.
In this case it updates the incentive’s total gas count and the participant's own gas count.

1. User submits an EVM transaction to an incentivized smart contract and the transaction is finished successfully.
2. The EVM hook’s `PostTxProcessing` method is called on the incentives module.
   It is passed a transaction receipt
   that includes the cumulative gas used by the transaction sender to pay for the gas fees.
   The hook
    1. adds `gasUsed` to an incentive's cumulated `totalGas` and
    2. adds `gasUsed` to a participant's gas meter's cumulative gas used.

### Epoch Hook - Distribution of Rewards

The Epoch hook triggers the distribution of usage rewards for all registered incentives at the end of each epoch
(one day or one week).
This distribution process first 1) allocates the rewards for each incentive from the allocation pool
and then 2) distributes these rewards to all participants of each incentive.

1. A `RegisterIncentiveProposal` passes and an `incentive` for the proposed contract is created.
2. An `epoch` begins and `rewards` (EVMOS and other denoms) that are minted on every block for inflation
   are added to the inflation pool every block.
3. Users submit transactions and call functions on the incentivized smart contracts to interact
   and gas gets logged through the EVM Hook.
4. A block, which signalizes the end of an `epoch`, is proposed
   and the `DistributeIncentives` method is called through `AfterEpochEnd` hook.
   This method:
    1. Allocates the amount to be distributed from the inflation pool
    2. Distributes the rewards to all participants.
       The rewards of each participant are limited by the amount of gas they spent on transaction fees
       during the current epoch and the reward scaler parameter.
    3. Deletes all gas meters for the contract
    4. Updates the remaining epochs of each incentive.
       If an incentive’s remaining epochs equals to zero,
       the incentive is removed and the allocation meters are updated.
    5. Sets the cumulative totalGas to zero for the next epoch
5. Rewards for a given denomination accumulate in the inflation pool
   if the denomination’s allocation capacity is not fully exhausted
   and the sum of all active incentivized contracts' allocation is < 100%.
   The accumulated rewards are added to the allocation in the following epoch.


## Events

The `x/incentives` module emits the following events:

### Register Incentive Proposal

| Type                 | Attribute Key | Attribute Value                                |
| -------------------- | ------------ | --------------------------------------------- |
| `register_incentive` | `"contract"` | `{erc20_address}`                             |
| `register_incentive` | `"epochs"`   | `{strconv.FormatUint(uint64(in.Epochs), 10)}` |

### Cancel Incentive Proposal

| Type               | Attribute Key | Attribute Value    |
| ------------------ | ------------ | ----------------- |
| `cancel_incentive` | `"contract"` | `{erc20_address}` |

### Incentive Distribution

| Type                    | Attribute Key | Attribute Value                                |
| ----------------------- | ------------ | --------------------------------------------- |
| `distribute_incentives` | `"contract"` | `{erc20_address}`                             |
| `distribute_incentives` | `"epochs"`   | `{strconv.FormatUint(uint64(in.Epochs), 10)}` |


## Parameters

The `x/incentives` module contains the parameters described below. All parameters can be modified via governance.

| Key                         | Type    | Default Value                      |
| --------------------------- | ------- | ---------------------------------- |
| `EnableIncentives`          | bool    | `true`                             |
| `AllocationLimit`           | sdk.Dec | `sdk.NewDecWithPrec(5,2)` // 5%    |
| `IncentivesEpochIdentifier` | string  | `week`                             |
| `rewardScaler`              | sdk.Dec | `sdk.NewDecWithPrec(12,1)` // 120% |

### Enable Incentives

The `EnableIncentives` parameter toggles all state transitions in the module.
When the parameter is disabled, it will prevent all Incentive registration and cancellation and distribution functionality.

### Allocation Limit

The `AllocationLimit` parameter defines the maximum allocation that each incentive can define per denomination.
For example, with an `AllocationLimit` of 5%,
there can be at most 20 active incentives per denom if they all max out the limit.

There is a cap on how high the reward percentage can be per allocation.

### Incentives Epoch Identifier

The `IncentivesEpochIdentifier` parameter specifies the length of an epoch.
It is the interval at which incentive rewards are regularly distributed.

### Reward Scaler

The `rewardScaler` parameter defines  each participant’s reward limit, relative to their gas used.
An incentive allows users to earn rewards up to `rewards = k * sum(txFees)`,
where `k` defines the reward scaler parameter that caps the incentives allocated to a single user
by multiplying it to the sum of transaction fees that they’ve spent in the current epoch.


## Clients

A user can query the `x/incentives` module using the CLI, JSON-RPC, gRPC or REST.

### CLI

Find below a list of `evmosd` commands added with the `x/incentives` module.
You can obtain the full list by using the `evmosd -h` command.

#### Queries

The `query` commands allow users to query `incentives` state.

**`incentives`**

Allows users to query all registered incentives.

```go
evmosd query incentives incentives [flags]
```

**`incentive`**

Allows users to query an incentive for a given contract.

```go
evmosd query incentives incentive CONTRACT_ADDRESS [flags]
```

**`gas-meters`**

Allows users to query all gas meters for a given incentive.

```bash
evmosd query incentives gas-meters CONTRACT_ADDRESS [flags]
```

**`gas-meter`**

Allows users to query a gas meter for a given incentive and user.

```go
evmosd query incentives gas-meter CONTRACT_ADDRESS PARTICIPANT_ADDRESS [flags]
```

**`params`**

Allows users to query incentives params.

```bash
evmosd query incentives params [flags]
```

#### Proposals

The `tx gov submit-legacy-proposal` commands allow users to query create a proposal using the governance module CLI:

**`register-incentive`**

Allows users to submit a `RegisterIncentiveProposal`.

```bash
evmosd tx gov submit-legacy-proposal register-incentive CONTRACT_ADDRESS ALLOCATION EPOCHS [flags]
```

**`cancel-incentive`**

Allows users to submit a `CanelIncentiveProposal`.

```bash
evmosd tx gov submit-legacy-proposal cancel-incentive CONTRACT_ADDRESS [flags]
```

**`param-change`**

Allows users to submit a `ParameterChangeProposal``.

```bash
evmosd tx gov submit-legacy-proposal param-change PROPOSAL_FILE [flags]
```

### gRPC

#### Queries

| Verb   | Method                                                     | Description                                   |
| ------ | ---------------------------------------------------------- | --------------------------------------------- |
| `gRPC` | `evmos.incentives.v1.Query/Incentives`                     | Gets all registered incentives                |
| `gRPC` | `evmos.incentives.v1.Query/Incentive`                      | Gets incentive for a given contract           |
| `gRPC` | `evmos.incentives.v1.Query/GasMeters`                      | Gets gas meters for a given incentive         |
| `gRPC` | `evmos.incentives.v1.Query/GasMeter`                       | Gets gas meter for a given incentive and user |
| `gRPC` | `evmos.incentives.v1.Query/AllocationMeters`               | Gets all allocation meters                    |
| `gRPC` | `evmos.incentives.v1.Query/AllocationMeter`                | Gets allocation meter for a denom             |
| `gRPC` | `evmos.incentives.v1.Query/Params`                         | Gets incentives params                        |
| `GET`  | `/evmos/incentives/v1/incentives`                          | Gets all registered incentives                |
| `GET`  | `/evmos/incentives/v1/incentives/{contract}`               | Gets incentive for a given contract           |
| `GET`  | `/evmos/incentives/v1/gas_meters`                          | Gets gas meters for a given incentive         |
| `GET`  | `/evmos/incentives/v1/gas_meters/{contract}/{participant}` | Gets gas meter for a given incentive and user |
| `GET`  | `/evmos/incentives/v1/allocation_meters`                   | Gets all allocation meters                    |
| `GET`  | `/evmos/incentives/v1/allocation_meters/{denom}`           | Gets allocation meter for a denom             |
| `GET`  | `/evmos/incentives/v1/params`                              | Gets incentives params                        |
