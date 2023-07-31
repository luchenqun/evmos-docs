# `vesting`

## 摘要

本文档规定了 Evmos Hub 的内部 `x/vesting` 模块。

`x/vesting` 模块引入了 `ClawbackVestingAccount`，这是一种新的锁仓账户类型，
实现了 Cosmos SDK 的 [`VestingAccount`](https://docs.cosmos.network/main/modules/auth/vesting#vesting-account-types) 接口。
该账户用于分配需要锁仓、冻结和收回的代币。

`ClawbackVestingAccount` 允许任何两方就未来的奖励计划达成协议，
在此计划中，代币会随着时间的推移被授予权限。
各方可以使用该账户来执行法律合同或承诺长期的共同利益。

在这个承诺中，锁仓是逐渐获得转账和委托分配代币权限的机制。
此外，冻结提供了一种机制，防止从账户中转移分配的代币
和执行以太坊交易。
锁仓和冻结都在账户创建时定义为计划。
在任何时候，`ClawbackVestingAccount` 的资助者都可以执行收回操作以取回未锁仓的代币。
应当在合同中约定执行收回操作的情况
（例如智能合约）。

对于 Evmos，`ClawbackVestingAccount` 用于将代币分配给核心团队成员和顾问，
以激励他们长期参与项目。

## 目录

1. **[概念](#概念)**
2. **[状态](#状态转换)**
3. **[状态转换](#状态转换)**
4. **[交易](#交易)**
5. **[AnteHandlers](#AnteHandlers)**
6. **[事件](#事件)**
7. **[客户端](#客户端)**

## 参考资料

- SDK 锁仓规范: [https://docs.cosmos.network/main/modules/auth/vesting](https://docs.cosmos.network/main/modules/auth/vesting)
- SDK 锁仓实现: [https://github.com/cosmos/cosmos-sdk/tree/master/x/auth/vesting](https://github.com/cosmos/cosmos-sdk/tree/master/x/auth/vesting)
- Agoric 的锁仓收回账户: [https://github.com/Agoric/agoric-sdk/issues/4085](https://github.com/Agoric/agoric-sdk/issues/4085)
- Agoric 的 `vestcalc` 工具: [https://github.com/agoric-labs/cosmos-sdk/tree/Agoric/x/auth/vesting/cmd/vestcalc](https://github.com/agoric-labs/cosmos-sdk/tree/Agoric/x/auth/vesting/cmd/vestcalc)

## 概念

### 解锁

解锁是将“未解锁”转换为“已解锁”代币的过程，而不转移这些代币的所有权。
在未解锁状态下，代币不能转移到其他账户，也不能委托给验证者或用于治理。
解锁计划描述了代币解锁的数量和时间。
首次解锁代币的持续时间称为“cliff”。

### 锁定

锁定描述了将代币从“锁定”状态转换为“未锁定”状态的计划。
只要所有代币都被锁定，该账户就无法执行任何使用`x/evm`模块花费EVMOS的以太坊交易。
但是，该账户可以执行不涉及花费EVMOS代币的以太坊交易。
此外，锁定的代币不能转移到其他账户。
如果代币同时处于锁定和解锁状态，则可以将其委托给验证者，但不能转移到其他账户。

下表总结了受到解锁和锁定组合约束的代币所允许的操作：

| 代币状态                | 转账 | 委托 | 投票 | 花费EVMOS的以太坊交易\*\* | 不花费EVMOS的以太坊交易（金额 = 0）\*\* |
| ----------------------- | :--: | :--: | :--: | :-----------------------: | :------------------------------------: |
| `锁定` & `未解锁`        |  ❌   |  ❌   |  ❌   |            ❌              |                  ✅                   |
| `锁定` & `已解锁`        |  ❌   |  ✅   |  ✅   |            ❌              |                  ✅                   |
| `未锁定` & `未解锁`      |  ❌   |  ❌   |  ❌   |            ❌              |                  ✅                   |
| `未锁定` & `已解锁`\*    |  ✅   |  ✅   |  ✅   |            ✅              |                  ✅                   |

\*质押奖励已解锁和已解锁

\*\*仅当EVM交易涉及发送锁定或未解锁的EVMOS代币时，交易才会失败，
例如将EVMOS发送到EOA或智能合约（如果金额> 0，则失败）。

### 计划表

锁定和解锁计划指定了代币解锁或解锁的数量和时间。
它们被定义为[`periods`](https://docs.cosmos.network/main/modules/auth/vesting#period)，
其中每个期间都有自己的长度和数量。
例如，典型的锁定计划从一个一年期开始，表示锁定悬崖，
然后是几个月的锁定期，直到总分配的锁定数量解锁。

可以使用 Agoric 的[`vestcalc`](https://github.com/agoric-labs/cosmos-sdk/tree/Agoric/x/auth/vesting/cmd/vestcalc)工具轻松创建锁定或解锁计划。
例如，
要计算一个为期四年的锁定计划，其中包括一年的悬崖期，并从2022年1月开始，可以运行以下命令：

```bash
vestcalc --write --start=2022-01-01 --coins=200000000000000000000000aevmos --months=48 --cliffs=2023-01-01
```

### 收回

如果`ClawbackVestingAccount`的基础承诺或合约被违反，
收回机制可以将未解锁的资金返还给原始资金提供者。
`ClawbackVestingAccount`的资金提供者是在创建账户时发送代币到该账户的地址。
只有资金提供者才能执行收回操作，将资金返还到他们的账户。
或者，他们可以指定一个目标地址来发送未解锁的资金。

## 状态

### 状态对象

`x/vesting`模块不会在自己的存储中保存对象。
相反，它使用SDK的`auth`模块将账户对象存储在状态中，
使用[账户接口](https://docs.cosmos.network/main/modules/auth#account-interface)。
账户在外部以接口的形式暴露，并在内部作为收回锁定账户存储。

### ClawbackVestingAccount

实现了[锁定账户](https://docs.cosmos.network/main/modules/auth/vesting#vesting-account-types)接口的实例。
它提供了一个可以持有受限制的贡献的账户，
或者可以收回未解锁代币的锁定账户，
或者两者兼而有之（代币解锁，但仍然被锁定）。

```go
type ClawbackVestingAccount struct {
	// base_vesting_account implements the VestingAccount interface. It contains
	// all the necessary fields needed for any vesting account implementation
	*types.BaseVestingAccount `protobuf:"bytes,1,opt,name=base_vesting_account,json=baseVestingAccount,proto3,embedded=base_vesting_account" json:"base_vesting_account,omitempty"`
	// funder_address specifies the account which can perform clawback
	FunderAddress string `protobuf:"bytes,2,opt,name=funder_address,json=funderAddress,proto3" json:"funder_address,omitempty"`
	// start_time defines the time at which the vesting period begins
	StartTime time.Time `protobuf:"bytes,3,opt,name=start_time,json=startTime,proto3,stdtime" json:"start_time"`
	// lockup_periods defines the unlocking schedule relative to the start_time
	LockupPeriods []types.Period `protobuf:"bytes,4,rep,name=lockup_periods,json=lockupPeriods,proto3" json:"lockup_periods"`
	// vesting_periods defines the vesting schedule relative to the start_time
	VestingPeriods []types.Period `protobuf:"bytes,5,rep,name=vesting_periods,json=vestingPeriods,proto3" json:"vesting_periods"`
}
```

#### BaseVestingAccount

实现了 `VestingAccount` 接口。
它包含了任何锁仓账户实现所需的所有必要字段。

#### FunderAddress

指定提供原始代币并能够执行收回的账户。

#### StartTime

定义锁仓和解锁计划开始的时间。

#### LockupPeriods

定义相对于开始时间的解锁计划。

#### VestingPeriods

定义相对于开始时间的锁仓计划。

### 创世状态

`x/vesting` 模块允许在创世时定义 `ClawbackVestingAccounts`。
在这种情况下，账户余额必须记录在 SDK `bank` 模块的余额中，
或通过 `add-genesis-account` CLI 命令自动调整。

## 状态转换

`x/vesting` 模块允许进行状态转换，创建和更新带有 `CreateClawbackVestingAccount` 的收回锁仓账户，
或使用 `Clawback` 收回未解锁的资金。

### 创建收回锁仓账户

资助者创建一个新的收回锁仓账户，定义要资助的地址以及锁仓/解锁计划。
此外，可以使用相同的消息向现有的收回锁仓账户添加新的授权。

1. 资助者通过其中一个客户端提交 `MsgCreateClawbackVestingAccount`。
2. 检查是否
    1. 锁仓账户地址未被阻止
    2. 提供了至少一个锁仓或解锁计划。
       如果其中一个不存在，则默认为即时锁仓或解锁计划。
    3. 锁仓和锁仓总金额相等
3. 创建或更新收回锁仓账户，并将代币从资助者发送到锁仓账户
    1. 如果收回锁仓账户已存在且 `--merge` 设置为 true，
       则将授权添加到现有的总锁仓金额，并更新锁仓和解锁计划。
    2. 否则创建一个新的收回锁仓账户

### 收回

资助地址是唯一能够执行收回的地址。

1. 资助者通过其中一个客户端提交 `MsgClawback`。
2. 检查是否
    1. 给定了目标地址，如果没有则默认为资助者地址
    2. 目标地址未被阻止
    3. 账户存在且为收回锁仓账户
    4. 账户的资助者与消息中的相同
3. 将未解锁的代币从收回锁仓账户转移到目标地址，
   更新解锁计划并移除未来的锁仓事件。

### 更新回购锁定账户的资金提供者

只有当前的资金提供者才能更新现有回购锁定账户的资金提供地址。

1. 资金提供者通过其中一个客户端提交 `MsgUpdateVestingFunder`。
2. 检查以下内容：
    1. 新的资金提供地址未被阻止。
    2. 锁定账户存在且为回购锁定账户。
    3. 账户的资金提供者与消息中的相同。
3. 使用新的资金提供地址更新锁定账户的资金提供者。

### 转换锁定账户

当所有代币都解锁后，锁定账户可以转换为 `ETHAccount`。

1. 锁定账户的所有者通过其中一个客户端提交 `MsgConvertVestingAccount`。
2. 检查以下内容：
    1. 锁定账户存在且为回购锁定账户。
    2. 锁定账户的锁定和解锁计划已经结束。
3. 将锁定账户转换为 `EthAccount`。

## 交易

本节定义了在前一节中定义的状态转换所产生的具体 `sdk.Msg` 类型。

### `CreateClawbackVestingAccount`

```go
type MsgCreateClawbackVestingAccount struct {
	// from_address specifies the account to provide the funds and sign the
	// clawback request
	FromAddress string `protobuf:"bytes,1,opt,name=from_address,json=fromAddress,proto3" json:"from_address,omitempty"`
	// to_address specifies the account to receive the funds
	ToAddress string `protobuf:"bytes,2,opt,name=to_address,json=toAddress,proto3" json:"to_address,omitempty"`
	// start_time defines the time at which the vesting period begins
	StartTime time.Time `protobuf:"bytes,3,opt,name=start_time,json=startTime,proto3,stdtime" json:"start_time"`
	// lockup_periods defines the unlocking schedule relative to the start_time
	LockupPeriods []types.Period `protobuf:"bytes,4,rep,name=lockup_periods,json=lockupPeriods,proto3" json:"lockup_periods"`
	// vesting_periods defines thevesting schedule relative to the start_time
	VestingPeriods []types.Period `protobuf:"bytes,5,rep,name=vesting_periods,json=vestingPeriods,proto3" json:"vesting_periods"`
	// merge specifies a the creation mechanism for existing
	// ClawbackVestingAccounts. If true, merge this new grant into an existing
	// ClawbackVestingAccount, or create it if it does not exist. If false,
	// creates a new account. New grants to an existing account must be from the
	// same from_address.
	Merge bool `protobuf:"varint,6,opt,name=merge,proto3" json:"merge,omitempty"`
}
```

如果消息内容的无状态验证失败，则会出现以下情况：

- `FromAddress` 或 `ToAddress` 无效。
- `LockupPeriods` 和 `VestingPeriods`
    - 包含长度为非正数的期间。
    - 描述相同总金额。

### `Clawback`

```go
type MsgClawback struct {
	// funder_address is the address which funded the account
	FunderAddress string `protobuf:"bytes,1,opt,name=funder_address,json=funderAddress,proto3" json:"funder_address,omitempty"`
	// account_address is the address of the ClawbackVestingAccount to claw back from.
	AccountAddress string `protobuf:"bytes,2,opt,name=account_address,json=accountAddress,proto3" json:"account_address,omitempty"`
	// dest_address specifies where the clawed-back tokens should be transferred
	// to. If empty, the tokens will be transferred back to the original funder of
	// the account.
	DestAddress string `protobuf:"bytes,3,opt,name=dest_address,json=destAddress,proto3" json:"dest_address,omitempty"`
}
```

如果消息内容的无状态验证失败，则会出现以下情况：

- `FunderAddress` 或 `AccountAddress` 无效。
- `DestAddress` 不为空且无效。

### `UpdateVestingFunder`

```go
type MsgUpdateVestingFunder struct {
	// funder_address is the current funder address of the ClawbackVestingAccount
	FunderAddress string `protobuf:"bytes,1,opt,name=funder_address,json=funderAddress,proto3" json:"funder_address,omitempty"`
	// new_funder_address is the new address to replace the existing funder_address
	NewFunderAddress string `protobuf:"bytes,2,opt,name=new_funder_address,json=newFunderAddress,proto3" json:"new_funder_address,omitempty"`
	// vesting_address is the address of the ClawbackVestingAccount being updated
	VestingAddress string `protobuf:"bytes,3,opt,name=vesting_address,json=vestingAddress,proto3" json:"vesting_address,omitempty"`
}
```

如果消息内容的无状态验证失败，则会出现以下情况：

- `FunderAddress`、`NewFunderAddress` 或 `VestingAddress` 无效。

### `ConvertVestingAccount`

```go
type MsgConvertVestingAccount struct {
	// vesting_address is the address of the ClawbackVestingAccount being updated
	VestingAddress string `protobuf:"bytes,2,opt,name=vesting_address,json=vestingAddress,proto3" json:"vesting_address,omitempty"`
}
```

如果消息内容的无状态验证失败，则会出现以下情况：

- `VestingAddress` 无效。

## AnteHandlers

`x/vesting` 模块提供了递归链接在一起的 `AnteDecorator`，形成一个单一的 [`Antehandler`](https://github.com/cosmos/cosmos-sdk/blob/v0.43.0-alpha1/docs/architecture/adr-010-modular-antehandler.md)。
这些装饰器对以太坊或 SDK 交易执行基本的有效性检查，以便可以将其从交易内存池中丢弃。

请注意，`AnteHandler` 在 `CheckTx` 和 `DeliverTx` 阶段都会被调用，因为 Tendermint 提议者目前有能力在他们的提议块中包含失败的 `CheckTx` 事务。

### 装饰器

以下装饰器实现了代币委托的锁定逻辑和执行 EVM 事务。

#### `VestingDelegationDecorator`

验证交易是否包含未解锁代币的委托。如果以下条件之一不满足，此 AnteHandler 装饰器将失败：

- 消息不是 `MsgDelegate`
- 发送方账户不存在
- 发送方账户不是 `ClawbackVestingAccount`
- 委托金额大于已解锁的代币数量

#### `EthVestingTransactionDecorator`

验证是否允许回收锁定账户执行以太坊事务，基于其解锁计划是否已超过锁定期和首个锁定期。还验证账户是否有足够的未锁定代币来执行事务。如果以下条件之一不满足，此 AnteHandler 装饰器将失败：

- 消息不是 `MsgEthereumTx`
- 发送方账户不存在
- 发送方账户不是 `ClawbackVestingAccount`
- 区块时间早于超过锁定期结束时间（无解锁代币）且
- 区块时间早于超过所有锁定期（有锁定代币）
- 发送方账户的未锁定代币不足以执行事务

## 事件

`x/vesting` 模块会发出以下事件：

### 创建回收锁定账户

| 类型                              | 属性键         | 属性值                            |
| --------------------------------- | -------------- | --------------------------------- |
| `create_clawback_vesting_account` | `"from"`       | `{msg.FromAddress}`               |
| `create_clawback_vesting_account` | `"coins"`      | `{vestingCoins.String()}`         |
| `create_clawback_vesting_account` | `"start_time"` | `{msg.StartTime.String()}`        |
| `create_clawback_vesting_account` | `"merge"`      | `{strconv.FormatBool(msg.Merge)}` |
| `create_clawback_vesting_account` | `"amount"`     | `{msg.ToAddress}`                 |

### 撤销

| 类型       | 属性键         | 属性值                |
| ---------- | -------------- | --------------------- |
| `clawback` | `"funder"`     | `{msg.FromAddress}`   |
| `clawback` | `"account"`    | `{msg.AccountAddress}` |
| `clawback` | `"destination"`| `{msg.DestAddress}`   |

### 更新撤销锁定账户的资金提供者

| 类型                    | 属性键      | 属性值                    |
| ----------------------- | -------------- | ------------------------ |
| `update_vesting_funder` | `"funder"`     | `{msg.FromAddress}`      |
| `update_vesting_funder` | `"account"`    | `{msg.VestingAddress}`   |
| `update_vesting_funder` | `"new_funder"` | `{msg.NewFunderAddress}` |

## 客户端

用户可以使用CLI、gRPC或REST查询Evmos的`x/vesting`模块。

### CLI

以下是使用`x/vesting`模块添加到`evmosd`命令的列表。
您可以使用`evmosd -h`命令获取完整列表。

#### 创世

创世配置命令允许用户配置创世`vesting`账户状态。

`add-genesis-account`

允许用户在创世时设置带有代币分配的撤销锁定账户，受到撤销的限制。
必须提供一个锁定期文件（`--lockup`），一个锁定期文件（`--vesting`）或两者都有。

如果两个文件都给出了，它们必须描述相同总金额的计划。
如果省略了一个文件，它将默认为立即解锁或解除锁定整个金额的计划。
描述的代币金额将从`--from`地址转移到锁定账户。
如果代币被锁定或未解锁，则无法将其从账户转出。
只有解锁的代币可以质押。
有关如何设置的示例，请参见[此链接](https://github.com/evmos/evmos/pull/303)。

```go
evmosd add-genesis-account ADDRESS_OR_KEY_NAME COIN... [flags]
```

#### 查询

`query`命令允许用户查询`vesting`账户状态。

**`balances`**

允许用户查询给定锁定账户的已锁定、未解锁和解锁代币。

```go
evmosd查询归属余额地址[标志]
```

#### 交易

`tx`命令允许用户创建和收回`归属`账户状态。

**`create-clawback-vesting-account`**

允许用户创建一个新的归属账户，并提供一定数量的代币用于资金分配，同时可以进行收回。
必须提供一个锁定期文件（--lockup），一个归属期文件（--vesting），或者两者都提供。

如果两个文件都提供，它们必须描述相同总金额的计划。
如果一个文件被省略，它将默认为立即解锁或归属整个金额的计划。
描述的代币数量将从--from地址转移到归属账户。
如果代币被锁定或未归属，将无法从账户中转出。
只有已归属的代币可以进行质押。
有关如何设置的示例，请参见[此链接](https://github.com/evmos/evmos/pull/303)。

```go
evmosd tx vesting create-clawback-vesting-account TO_ADDRESS [标志]
```

**`clawback`**

允许用户将未归属的金额从ClawbackVestingAccount中转出。
必须由原始资金提供者地址（--from）请求，并可以提供目标地址（--dest），否则代币将返回给资金提供者。
委托或取消委托的质押代币将以委托（取消委托）状态转移。
接收者可能会受到惩罚，如果需要，必须采取行动解除质押代币。

```go
evmosd tx vesting clawback ADDRESS [标志]
```

**`update-vesting-funder`**

允许用户更新现有`ClawbackVestingAccount`的资金提供者。
必须由原始资金提供者地址（`--from`）请求。
要执行此操作，用户需要提供两个参数：

1. 新的资金提供者地址
2. 归属账户地址

```go
evmosd tx vesting update-vesting-funder VESTING_ACCOUNT_ADDRESS NEW_FUNDER_ADDRESS --from=FUNDER_ADDRESS [标志]
```

**`convert`**

允许用户将其归属账户转换为链上的默认账户（即`EthAccount`）。
要执行此操作，用户需要提供一个参数：

1. 质押账户地址

```go
evmosd tx vesting convert VESTING_ACCOUNT_ADDRESS [flags]
```

### gRPC

#### 查询

| 动词   | 方法                                 | 描述                            |
| ------ | -------------------------------------- | -------------------------------------- |
| `gRPC` | `evmos.vesting.v1.Query/Balances`      | 获取锁定、未解锁和已解锁的代币 |
| `GET`  | `/evmos/vesting/v1/balances/{address}` | 获取锁定、未解锁和已解锁的代币 |

#### 交易

| 动词   | 方法                                                 | 描述                      |
| ------ | ------------------------------------------------------ | -------------------------------- |
| `gRPC` | `evmos.vesting.v1.Msg/CreateClawbackVestingAccount`    | 创建回收质押账户 |
| `gRPC` | `/evmos.vesting.v1.Msg/Clawback`                       | 执行回收                |
| `gRPC` | `/evmos.vesting.v1.Msg/UpdateVestingFunder`            | 更新质押账户的资金提供者   |
| `GET`  | `/evmos/vesting/v1/tx/create_clawback_vesting_account` | 创建回收质押账户 |
| `GET`  | `/evmos/vesting/v1/tx/clawback`                        | 执行回收                |
| `GET`  | `/evmos/vesting/v1/tx/update_vesting_funder`           | 更新质押账户的资金提供者   |


# `vesting`

## Abstract

This document specifies the internal `x/vesting` module of the Evmos Hub.

The `x/vesting` module introduces the `ClawbackVestingAccount`,  a new vesting account type
that implements the Cosmos SDK [`VestingAccount`](https://docs.cosmos.network/main/modules/auth/vesting#vesting-account-types)
interface.
This account is used to allocate tokens that are subject to vesting, lockup, and clawback.

The `ClawbackVestingAccount` allows any two parties to agree on a future rewarding schedule,
where tokens are granted permissions over time.
The parties can use this account to enforce legal contracts or commit to mutual long-term interests.

In this commitment, vesting is the mechanism for gradually earning permission to transfer and delegate allocated tokens.
Additionally, the lockup provides a mechanism to prevent the right to transfer allocated tokens
and perform Ethereum transactions from the account.
Both vesting and lockup are defined in schedules at account creation.
At any time, the funder of a `ClawbackVestingAccount` can perform a clawback to retrieve unvested tokens.
The circumstances under which a clawback should be performed can be agreed upon in a contract
(e.g. smart contract).

For Evmos, the `ClawbackVestingAccount` is used to allocate tokens to core team members and advisors
to incentivize long-term participation in the project.

## Contents

1. **[Concepts](#concepts)**
2. **[State](#state-transitions)**
3. **[State Transitions](#state-transitions)**
4. **[Transactions](#transactions)**
5. **[AnteHandlers](#antehandlers)**
6. **[Events](#events)**
7. **[Clients](#clients)**

## References

- SDK vesting specification: [https://docs.cosmos.network/main/modules/auth/vesting](https://docs.cosmos.network/main/modules/auth/vesting)
- SDK vesting implementation: [https://github.com/cosmos/cosmos-sdk/tree/master/x/auth/vesting](https://github.com/cosmos/cosmos-sdk/tree/master/x/auth/vesting)
- Agoric’s Vesting Clawback Account: [https://github.com/Agoric/agoric-sdk/issues/4085](https://github.com/Agoric/agoric-sdk/issues/4085)
- Agoric’s `vestcalc` tool: [https://github.com/agoric-labs/cosmos-sdk/tree/Agoric/x/auth/vesting/cmd/vestcalc](https://github.com/agoric-labs/cosmos-sdk/tree/Agoric/x/auth/vesting/cmd/vestcalc)


## Concepts

### Vesting

Vesting describes the process of converting `unvested` into `vested` tokens
without transferring the ownership of those tokens.
In an unvested state, tokens cannot be transferred to other accounts, delegated to validators, or used for governance.
A vesting schedule describes the amount and time at which tokens are vested.
The duration until which the first tokens are vested is called the `cliff`.

### Lockup

The lockup describes the schedule by which tokens are converted from a `locked` to an `unlocked` state.
As long as all tokens are locked, the account cannot perform any Ethereum transactions
that spend EVMOS using the `x/evm` module.
However, the account can perform Ethereum transactions that don't spend EVMOS tokens.
Additionally, locked tokens cannot be transferred to other accounts.
In the case in which tokens are both locked and vested at the same time,
it is possible to delegate them to validators, but not transfer them to other accounts.

The following table summarizes the actions that are allowed for tokens
that are subject to the combination of vesting and lockup:

| Token Status            | Transfer | Delegate | Vote | Eth Txs that spend EVMOS\*\* | Eth Txs that don't spend EVMOS (amount = 0)\*\* |
| ----------------------- | :------: | :------: | :--: | :--------------------------: | :---------------------------------------------: |
| `locked` & `unvested`   |    ❌    |    ❌    |  ❌  |              ❌              |                       ✅                        |
| `locked` & `vested`     |    ❌    |    ✅    |  ✅  |              ❌              |                       ✅                        |
| `unlocked` & `unvested` |    ❌    |    ❌    |  ❌  |              ❌              |                       ✅                        |
| `unlocked` & `vested`\* |    ✅    |    ✅    |  ✅  |              ✅              |                       ✅                        |

\*Staking rewards are unlocked and vested

\*\*EVM transactions only fail if they involve sending locked or unvested EVMOS tokens,
e.g. send EVMOS to EOA or Smart Contract (fails if amount > 0 ).

### Schedules

Vesting and lockup schedules specify the amount and time at which tokens are vested or unlocked.
They are defined as [`periods`](https://docs.cosmos.network/main/modules/auth/vesting#period)
where each period has its own length and amount.
A typical vesting schedule for instance would be defined starting with a one-year period to represent the vesting cliff,
followed by several monthly vesting periods until the total allocated vesting amount is vested.

Vesting or lockup schedules can be easily created
with Agoric’s [`vestcalc`](https://github.com/agoric-labs/cosmos-sdk/tree/Agoric/x/auth/vesting/cmd/vestcalc) tool.
E.g.
to calculate a four-year vesting schedule with a one year cliff, starting in January 2022, you can run vestcalc with:

```bash
vestcalc --write --start=2022-01-01 --coins=200000000000000000000000aevmos --months=48 --cliffs=2023-01-01
```

### Clawback

In case a `ClawbackVestingAccount`'s underlying commitment or contract is breached,
the clawback provides a mechanism to return unvested funds to the original funder.
The funder of the `ClawbackVestingAccount` is the address that sends tokens to the account at account creation.
Only the funder can perform the clawback to return the funds to their account.
Alternatively, they can specify a destination address to send unvested funds to.

## State

### State Objects

The `x/vesting` module does not keep objects in its own store.
Instead, it uses the SDK `auth` module to store account objects in state
using the [Account Interface](https://docs.cosmos.network/main/modules/auth#account-interface).
Accounts are exposed externally as an interface and stored internally as a clawback vesting account.

### ClawbackVestingAccount

An instance that implements
the [Vesting Account](https://docs.cosmos.network/main/modules/auth/vesting#vesting-account-types) interface.
It provides an account that can hold contributions subject to lockup,
or vesting which is subject to clawback of unvested tokens,
or a combination (tokens vest, but are still locked).

```go
type ClawbackVestingAccount struct {
	// base_vesting_account implements the VestingAccount interface. It contains
	// all the necessary fields needed for any vesting account implementation
	*types.BaseVestingAccount `protobuf:"bytes,1,opt,name=base_vesting_account,json=baseVestingAccount,proto3,embedded=base_vesting_account" json:"base_vesting_account,omitempty"`
	// funder_address specifies the account which can perform clawback
	FunderAddress string `protobuf:"bytes,2,opt,name=funder_address,json=funderAddress,proto3" json:"funder_address,omitempty"`
	// start_time defines the time at which the vesting period begins
	StartTime time.Time `protobuf:"bytes,3,opt,name=start_time,json=startTime,proto3,stdtime" json:"start_time"`
	// lockup_periods defines the unlocking schedule relative to the start_time
	LockupPeriods []types.Period `protobuf:"bytes,4,rep,name=lockup_periods,json=lockupPeriods,proto3" json:"lockup_periods"`
	// vesting_periods defines the vesting schedule relative to the start_time
	VestingPeriods []types.Period `protobuf:"bytes,5,rep,name=vesting_periods,json=vestingPeriods,proto3" json:"vesting_periods"`
}
```

#### BaseVestingAccount

Implements the `VestingAccount` interface.
It contains all the necessary fields needed for any vesting account implementation.

#### FunderAddress

Specifies the account which provides the original tokens and can perform clawback.

#### StartTime

Defines the time at which the vesting and lockup schedules begin.

#### LockupPeriods

Defines the unlocking schedule relative to the start time.

#### VestingPeriods

Defines the vesting schedule relative to the start time.

### Genesis State

The `x/vesting` module allows the definition of `ClawbackVestingAccounts` at genesis.
In this case, the account balance must be logged in the SDK `bank` module balances
or automatically adjusted through the `add-genesis-account` CLI command.

## State Transitions

The `x/vesting` module allows for state transitions that create
and update a clawback vesting account with `CreateClawbackVestingAccount`
or perform a clawback of unvested funds with `Clawback`.

### Create Clawback Vesting Account

A funder creates a new clawback vesting account defining the address to fund as well as the vesting/lockup schedules.
Additionally, new grants can be added to existing clawback vesting accounts with the same message.

1. Funder submits a `MsgCreateClawbackVestingAccount` through one of the clients.
2. Check if
    1. the vesting account address is not blocked
    2. there is at least one vesting or lockup schedule provided.
       If one of them is absent, default to instant vesting or unlock schedule.
    3. lockup and vesting total amounts are equal
3. Create or update a clawback vesting account and send coins from the funder to the vesting account
    1. if the clawback vesting account already exists and `--merge` is set to true,
       add a grant to the existing total vesting amount and update the vesting and lockup schedules.
    2. else create a new clawback vesting account

### Clawback

The funding address is the only address that can perform the clawback.

1. Funder submits a `MsgClawback` through one of the clients.
2. Check if
    1. a destination address is given and default to funder address if not
    2. the destination address is not blocked
    3. the account exists and is a clawback vesting account
    4. account funder is same as in msg
3. Transfer unvested tokens from the clawback vesting account to the destination address,
   update the lockup schedule and remove future vesting events.

### Update Clawback Vesting Account Funder

The funding address of an existing clawback vesting account can be updated only by the current funder.

1. Funder submits a `MsgUpdateVestingFunder` through one of the clients.
2. Check if
    1. the new funder address is not blocked
    2. the vesting account exists and is a clawback vesting account
    3. account funder is same as in msg
3. Update the vesting account funder with the new funder address.

### Convert Vesting Account

Once all tokens are vested, the vesting account can be converted to an `ETHAccount`

1. Owner of vesting account submits a `MsgConvertVestingAccount` through one of the clients.
2. Check if
    1. the vesting account exists and is a clawback vesting account
    2. the vesting account's vesting and locked schedules have concluded
3. Convert the vesting account to an `EthAccount`

## Transactions

This section defines the concrete `sdk.Msg`  types that result in the state transitions defined on the previous section.

### `CreateClawbackVestingAccount`

```go
type MsgCreateClawbackVestingAccount struct {
	// from_address specifies the account to provide the funds and sign the
	// clawback request
	FromAddress string `protobuf:"bytes,1,opt,name=from_address,json=fromAddress,proto3" json:"from_address,omitempty"`
	// to_address specifies the account to receive the funds
	ToAddress string `protobuf:"bytes,2,opt,name=to_address,json=toAddress,proto3" json:"to_address,omitempty"`
	// start_time defines the time at which the vesting period begins
	StartTime time.Time `protobuf:"bytes,3,opt,name=start_time,json=startTime,proto3,stdtime" json:"start_time"`
	// lockup_periods defines the unlocking schedule relative to the start_time
	LockupPeriods []types.Period `protobuf:"bytes,4,rep,name=lockup_periods,json=lockupPeriods,proto3" json:"lockup_periods"`
	// vesting_periods defines thevesting schedule relative to the start_time
	VestingPeriods []types.Period `protobuf:"bytes,5,rep,name=vesting_periods,json=vestingPeriods,proto3" json:"vesting_periods"`
	// merge specifies a the creation mechanism for existing
	// ClawbackVestingAccounts. If true, merge this new grant into an existing
	// ClawbackVestingAccount, or create it if it does not exist. If false,
	// creates a new account. New grants to an existing account must be from the
	// same from_address.
	Merge bool `protobuf:"varint,6,opt,name=merge,proto3" json:"merge,omitempty"`
}
```

The msg content stateless validation fails if:

- `FromAddress` or `ToAddress` are invalid
- `LockupPeriods` and `VestingPeriods`
    - include period with a non-positive length
    - describe the same total amount

### `Clawback`

```go
type MsgClawback struct {
	// funder_address is the address which funded the account
	FunderAddress string `protobuf:"bytes,1,opt,name=funder_address,json=funderAddress,proto3" json:"funder_address,omitempty"`
	// account_address is the address of the ClawbackVestingAccount to claw back from.
	AccountAddress string `protobuf:"bytes,2,opt,name=account_address,json=accountAddress,proto3" json:"account_address,omitempty"`
	// dest_address specifies where the clawed-back tokens should be transferred
	// to. If empty, the tokens will be transferred back to the original funder of
	// the account.
	DestAddress string `protobuf:"bytes,3,opt,name=dest_address,json=destAddress,proto3" json:"dest_address,omitempty"`
}
```

The msg content stateless validation fails if:

- `FunderAddress` or `AccountAddress` are invalid
- `DestAddress` is not empty and invalid

### `UpdateVestingFunder`

```go
type MsgUpdateVestingFunder struct {
	// funder_address is the current funder address of the ClawbackVestingAccount
	FunderAddress string `protobuf:"bytes,1,opt,name=funder_address,json=funderAddress,proto3" json:"funder_address,omitempty"`
	// new_funder_address is the new address to replace the existing funder_address
	NewFunderAddress string `protobuf:"bytes,2,opt,name=new_funder_address,json=newFunderAddress,proto3" json:"new_funder_address,omitempty"`
	// vesting_address is the address of the ClawbackVestingAccount being updated
	VestingAddress string `protobuf:"bytes,3,opt,name=vesting_address,json=vestingAddress,proto3" json:"vesting_address,omitempty"`
}
```

The msg content stateless validation fails if:

- `FunderAddress`, `NewFunderAddress` or `VestingAddress` are invalid

### `ConvertVestingAccount`

```go
type MsgConvertVestingAccount struct {
	// vesting_address is the address of the ClawbackVestingAccount being updated
	VestingAddress string `protobuf:"bytes,2,opt,name=vesting_address,json=vestingAddress,proto3" json:"vesting_address,omitempty"`
}
```

The msg content stateless validation fails if:

- `VestingAddress` is invalid

## AnteHandlers

The `x/vesting` module provides `AnteDecorator`s that are recursively chained together
into a single [`Antehandler`](https://github.com/cosmos/cosmos-sdk/blob/v0.43.0-alpha1/docs/architecture/adr-010-modular-antehandler.md).
These decorators perform basic validity checks on an Ethereum or SDK transaction,
such that it could be thrown out of the transaction Mempool.

Note that the `AnteHandler` is called on both `CheckTx` and `DeliverTx`,
as Tendermint proposers presently have the ability to include in their proposed block transactions that fail `CheckTx`.

### Decorators

The following decorators implement the vesting logic for token delegation and performing EVM transactions.

#### `VestingDelegationDecorator`

Validates if a transaction contains a staking delegation of unvested coins. This AnteHandler decorator will fail if:

- the message is not a `MsgDelegate`
- sender account cannot be found
- sender account is not a `ClawbackVestingAccount`
- the bond amount is greater than the coins already vested

#### `EthVestingTransactionDecorator`

Validates if a clawback vesting account is permitted to perform Ethereum transactions,
based on if it has its vesting schedule has surpassed the vesting cliff and first lockup period.
Also, validates if the account has sufficient unlocked tokens to execute the transaction.
This AnteHandler decorator will fail if:

- the message is not a `MsgEthereumTx`
- sender account cannot be found
- sender account is not a `ClawbackVestingAccount`
- block time is before surpassing vesting cliff end (with zero vested coins) AND
- block time is before surpassing all lockup periods (with non-zero locked coins)
- sender account has insufficient unlocked tokens to execute the transaction

## Events

The `x/vesting` module emits the following events:

### Create Clawback Vesting Account

| Type                              | Attibute Key   | Attibute Value                    |
| --------------------------------- | -------------- | --------------------------------- |
| `create_clawback_vesting_account` | `"from"`       | `{msg.FromAddress}`               |
| `create_clawback_vesting_account` | `"coins"`      | `{vestingCoins.String()}`         |
| `create_clawback_vesting_account` | `"start_time"` | `{msg.StartTime.String()}`        |
| `create_clawback_vesting_account` | `"merge"`      | `{strconv.FormatBool(msg.Merge)}` |
| `create_clawback_vesting_account` | `"amount"`     | `{msg.ToAddress}`                 |

### Clawback

| Type       | Attibute Key    | Attibute Value         |
| ---------- | --------------- | ---------------------- |
| `clawback` | `"funder"`      | `{msg.FromAddress}`    |
| `clawback` | `"account"`     | `{msg.AccountAddress}` |
| `clawback` | `"destination"` | `{msg.DestAddress}`    |

### Update Clawback Vesting Account Funder

| Type                    | Attibute Key   | Attibute Value           |
| ----------------------- | -------------- | ------------------------ |
| `update_vesting_funder` | `"funder"`     | `{msg.FromAddress}`      |
| `update_vesting_funder` | `"account"`    | `{msg.VestingAddress}`   |
| `update_vesting_funder` | `"new_funder"` | `{msg.NewFunderAddress}` |

## Clients

A user can query the Evmos `x/vesting` module using the CLI, gRPC, or REST.

### CLI

Find below a list of `evmosd` commands added with the `x/vesting` module.
You can obtain the full list by using the `evmosd -h` command.

#### Genesis

The genesis configuration commands allow users to configure the genesis `vesting` account state.

`add-genesis-account`

Allows users to set up clawback vesting accounts at genesis, funded with an allocation of tokens, subject to clawback.
Must provide a lockup periods file (`--lockup`), a vesting periods file (`--vesting`), or both.

If both files are given, they must describe schedules for the same total amount.
If one file is omitted, it will default to a schedule that immediately unlocks or vests the entire amount.
The described amount of coins will be transferred from the --from address to the vesting account.
Unvested coins may be "clawed back" by the funder with the clawback command.
Coins may not be transferred out of the account if they are locked or unvested.
Only vested coins may be staked.
For an example of how to set this see [this link](https://github.com/evmos/evmos/pull/303).

```go
evmosd add-genesis-account ADDRESS_OR_KEY_NAME COIN... [flags]
```

#### Queries

The `query` commands allow users to query `vesting` account state.

**`balances`**

Allows users to query the locked, unvested and vested tokens for a given vesting account

```go
evmosd query vesting balances ADDRESS [flags]
```

#### Transactions

The `tx` commands allow users to create and clawback `vesting` account state.

**`create-clawback-vesting-account`**

Allows users to create a new vesting account funded with an allocation of tokens, subject to clawback.
Must provide a lockup periods file (--lockup), a vesting periods file (--vesting), or both.

If both files are given, they must describe schedules for the same total amount.
If one file is omitted, it will default to a schedule that immediately unlocks or vests the entire amount.
The described amount of coins will be transferred from the --from address to the vesting account.
Unvested coins may be "clawed back" by the funder with the clawback command.
Coins may not be transferred out of the account if they are locked or unvested.
Only vested coins may be staked.
For an example of how to set this see [this link](https://github.com/evmos/evmos/pull/303).

```go
evmosd tx vesting create-clawback-vesting-account TO_ADDRESS [flags]
```

**`clawback`**

Allows users to create a transfer unvested amount out of a ClawbackVestingAccount.
Must be requested by the original funder address (--from) and may provide a destination address (--dest),
otherwise the coins return to the funder.
Delegated or undelegating staking tokens will be transferred in the delegated (undelegating) state.
The recipient is vulnerable to slashing, and must act to unbond the tokens if desired.

```go
evmosd tx vesting clawback ADDRESS [flags]
```

**`update-vesting-funder`**

Allows users to update the funder of an existent `ClawbackVestingAccount`.
Must be requested by the original funder address (`--from`).
To perform this action, the user needs to provide two arguments:

1. the new funder address
2. the vesting account address

```go
evmosd tx vesting update-vesting-funder VESTING_ACCOUNT_ADDRESS NEW_FUNDER_ADDRESS --from=FUNDER_ADDRESS [flags]
```

**`convert`**

Allows users to convert their vesting account to the chain's default account (i.e `EthAccount`).
To perform this action a user needs to provide one argument:

1.the vesting account address

```go
evmosd tx vesting convert VESTING_ACCOUNT_ADDRESS [flags]
```

### gRPC

#### Queries

| Verb   | Method                                 | Description                            |
| ------ | -------------------------------------- | -------------------------------------- |
| `gRPC` | `evmos.vesting.v1.Query/Balances`      | Gets locked, unvested and vested coins |
| `GET`  | `/evmos/vesting/v1/balances/{address}` | Gets locked, unvested and vested coins |

#### Transactions

| Verb   | Method                                                 | Description                      |
| ------ | ------------------------------------------------------ | -------------------------------- |
| `gRPC` | `evmos.vesting.v1.Msg/CreateClawbackVestingAccount`    | Creates clawback vesting account |
| `gRPC` | `/evmos.vesting.v1.Msg/Clawback`                       | Performs clawback                |
| `gRPC` | `/evmos.vesting.v1.Msg/UpdateVestingFunder`            | Updates vesting account funder   |
| `GET`  | `/evmos/vesting/v1/tx/create_clawback_vesting_account` | Creates clawback vesting account |
| `GET`  | `/evmos/vesting/v1/tx/clawback`                        | Performs clawback                |
| `GET`  | `/evmos/vesting/v1/tx/update_vesting_funder`           | Updates vesting account funder   |

