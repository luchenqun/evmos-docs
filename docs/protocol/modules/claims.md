# `claims`

## 摘要

本文档规定了 Evmos Hub 的内部 `x/claims` 模块。

`x/claims` 模块是 Evmos [Rektdrop](https://evmos.blog/the-evmos-rektdrop-abbe931ba823) 的一部分，
旨在将网络代币分发给大量用户。

用户从空投分配中获得一定数量的代币，
然后可以在链上执行特定任务时自动领取更高比例的代币。

对于 Evmos Rektdrop，用户需要通过参与核心网络活动来领取空投。
Rektdrop 的接收者需要执行以下活动以获得分配的代币：

* 25% 通过质押领取
* 25% 通过参与治理投票领取
* 25% 通过使用 EVM（部署或与合约交互，在 web3 钱包中转账 EVMOS）领取
* 25% 通过发送或接收 IBC 转账领取

此外，这些可领取的资产如果不领取将会“过期”。
用户有两个月的时间（`DurationUntilDecay`）来领取他们的全部空投金额。
两个月后，可用的奖励金额将在实时中逐渐减少一个月（`DurationOfDecay`），
直到在启动后的三个月达到 `0%`（`DurationUntilDecay + DurationOfDecay`）。

# 目录

1. **[概念](#概念)**
2. **[状态](#状态)**
3. **[状态转换](#状态转换)**
4. **[钩子](#钩子)**
5. **[事件](#事件)**
6. **[参数](#参数)**
7. **[客户端](#客户端)**

## 概念

### Rektdrop

Evmos [Rektdrop](https://evmos.blog/the-evmos-rektdrop-abbe931ba823) 是将 EVMOS 代币空投给 Cosmos Hub、Osmosis 和 Ethereum 用户的创世空投。

> Evmos 的最终目标是将 Cosmos 和 Ethereum 社区聚集在一起，
因此 Rektdrop 的设计是为了奖励过去在这两个网络中的参与者，以“getting rekt”为主题。

Rektdrop 是第一个：

- 实施了 Sunny Aggarwal 的 [gasdrop](https://www.sunnya97.com/blog/gasdrop) 机制
- 涵盖了参与空投的最多链和应用程序
- 空投给桥接用户
- 包括对受到攻击和负面市场外部性（即 MEV）的用户的赔偿

空投的快照是在**2021年11月25日19:00 UTC**。

### 操作

`Action` 对应于用户必须执行的特定交易，以接收空投分配的代币。

有4种类型的操作，每种操作释放其剩余对应空投分配的25%。
这4种操作如下（不考虑 `ActionUnspecified` 用于认领）：

```go
// UNSPECIFIED defines an invalid action. NOT claimable
ActionUnspecified Action = 0
// VOTE defines a proposal vote.
ActionVote Action = 1
// DELEGATE defines an staking delegation.
ActionDelegate Action = 2
// EVM defines an EVM transaction.
ActionEVM Action = 3
// IBC Transfer defines a fungible token transfer transaction via IBC.
ActionIBCTransfer Action = 4
```

通过向治理、质押和 EVM 模块注册认领后的事务 **钩子** 来监控这些操作。
一旦用户执行了一个操作，`x/claims` 模块将解锁相应部分的资产，并将其转移到用户的余额中。

这些操作可以以任何顺序执行，认领模块在执行相应操作后不会授予任何额外的代币。

#### 投票操作

在对提案进行投票后，相应比例将通过从认领托管账户（`ModuleAccount`）向用户执行转账的方式空投到用户的余额中。

#### 质押（即委托）操作

在质押 Evmos 代币（即委托）后，相应比例将通过从认领托管账户（`ModuleAccount`）向用户执行转账的方式空投到用户的余额中。

#### EVM 操作

如果用户部署或与智能合约进行交互（通过应用程序或钱包集成），
相应比例将通过从认领托管账户（`ModuleAccount`）向用户执行转账的方式空投到用户的余额中。
当用户使用 Metamask 或其他 web3 钱包进行转账时，也适用此规则。

#### IBC 转账操作

如果用户向对方链提交 IBC 转账或从对方链接收 IBC 转账，
相应比例将通过提交或接收转账的方式空投到用户的余额中。

### 认领记录

认领记录是每个地址的认领数据的元数据。
它跟踪用户执行的所有操作以及分配给他们的代币总额。
所有具有相应 `ClaimRecord` 的地址的用户都有资格认领空投。

### 领取流程

如[操作](#actions)部分所述，用户必须提交交易以领取空投的分配代币。
然而，由于Evmos仅支持以太坊密钥而不支持默认的Tendermint密钥，因此以太坊和Cosmos合格用户的领取流程有所不同。

#### 以太坊用户

Evmos与以太坊共享币种类型（`60`）和密钥派生（以太坊的`secp256k1`）。
这使得已分配EVMOS代币的用户（EOA账户）可以直接使用其首选的web3钱包领取代币。

#### Cosmos Hub和Osmosis用户

使用默认的Tendermint `secp256k1`密钥的Cosmos Hub和Osmosis用户需要对其Evmos地址进行“跨链认证”。

这可以通过提交来自Cosmos Hub和Osmosis的IBC转账来完成，该转账由已分配代币的地址签名。

这个IBC转账的接收方Evmos地址就是代币将被空投到的地址。

:::warning
**重要**

只向您拥有的Evmos地址提交IBC转账。否则，您将失去空投的分配。
:::

### 衰减期

衰减期定义了用户可以领取的代币数量在一段时间内线性衰减的持续时间。
其目的是激励用户尽早领取代币并与区块链进行交互。

衰减期的开始定义为`AirdropStartTime`和`DurationUntilDecay`参数之和，
线性衰减的持续时间定义为`DurationOfDecay`，如下所述：

```go
decayStartTime = AirdropStartTime + DurationUntilDecay
decayEndTime = decayStartTime + DurationOfDecay
```

默认情况下，用户有两个月（`DurationUntilDecay`）的时间来领取他们的全部空投金额。
两个月后，可用的奖励金额将在实时中以一个月（`DurationOfDecay`）的时间逐渐减少，直到在启动后的3个月（结束时）达到`0%`。

### 空投收回

领取期结束后，未被用户领取的代币将被转移到社区资金池。
同样地，已分配代币但没有交易记录（即nonce = 0）的用户的余额将被收回到社区资金池。

## 状态

### 状态对象

`x/claims` 模块在状态中保留以下对象：

| 状态对象       | 描述                     | 键                           | 值                  | 存储  |
|----------------|------------------------|-------------------------------|------------------------|-------|
| `ClaimsRecord` | 认领记录字节码 | `[]byte{1} + []byte(address)` | `[]byte{claimsRecord}` | KV    |

#### 认领记录

`ClaimRecord` 定义了可认领的空投金额和认领代币的已完成操作列表。

```protobuf
message ClaimsRecord {
  // total initial claimable amount for the user
  string initial_claimable_amount = 1 [
    (gogoproto.customtype) = "github.com/cosmos/cosmos-sdk/types.Int",
    (gogoproto.nullable) = false
  ];
  // slice of the available actions completed
  repeated bool actions_completed = 2;
}
```

### 创世状态

`x/claims` 模块的 `GenesisState` 定义了从先前导出的高度初始化链所需的状态。
它包含模块参数和一个包含所有用户地址的认领记录的切片：

```go
// GenesisState defines the claims module's genesis state.
type GenesisState struct {
	// params defines all the parameters of the module.
	Params Params `protobuf:"bytes,1,opt,name=params,proto3" json:"params"`
	// list of claim records with the corresponding airdrop recipient
	ClaimsRecords []ClaimsRecordAddress `protobuf:"bytes,2,rep,name=claims_records,json=claimsRecords,proto3" json:"claims_records"`
}
```

### 不变量

`x/claims` 模块注册了一个 [`Invariant`](https://docs.cosmos.network/main/building-modules/invariants)，
以确保某个属性在任何给定时间都为真。
这些函数对于早期检测错误并采取措施限制其潜在后果（例如通过停止链）非常有用。

#### ClaimsInvariant

`ClaimsInvariant` 检查所有未认领代币的总金额是否等于认领模块账户中托管的余额。
这一点很重要，以确保所有认领记录都有足够的代币可供认领。

```go
balance := k.bankKeeper.GetBalance(ctx, moduleAccAddr, params.ClaimsDenom)
isInvariantBroken := !expectedUnclaimed.Equal(balance.Amount.ToDec())
```


## 状态转换

### ABCI

#### 结束块

ABCI 结束块检查空投是否已结束，以便处理未认领代币的收回。

1. 检查空投是否已结束。如果满足以下条件，则空投已结束：
    - 全局标志已启用
    - 当前区块时间大于空投结束时间
2. 通过将未认领代币的余额从托管账户转移到社区池中，收回托管账户中的代币
3. 如果满足以下条件，则通过将具有认领记录的空用户账户的余额从空用户账户中转移到社区池中，收回空用户账户中的代币：
    - 账户是 ETH 账户
    - 账户不是锁定账户
    - 账户的序列号为 0，即未提交任何交易
    - 余额金额与创世时发送的尘埃金额相同
    - 账户在认领代币以外的其他货币单位上没有其他余额
4. 从状态中删除所有认领记录
5. 通过将全局参数设置为 `false`，禁用任何进一步的认领操作

## 钩子

`x/claims` 模块为 `x/staking`、`x/gov` 和 `x/evm` 模块的四个操作实现了事务钩子。
它还实现了一个 IBC 中间件，以便通过将认领记录迁移到接收地址来认领 Cosmos Hub 和 Osmosis 用户的 IBC 转账操作和代币。

### 治理钩子 - 投票操作

用户使用他们的 Evmos 账户对治理提案进行投票。
一旦投票成功被包含，与投票操作对应的可认领金额将转移到用户地址：

1. 用户提交 `MsgVote`。
2. 开始 `ActionVote` 的认领过程。
3. 检查是否允许认领：
    - 全局参数已启用
    - 当前区块时间在认领期结束之前
    - 用户具有认领记录（即分配）用于空投
    - 用户尚未认领该操作
    - 可认领金额大于零
4. 将可认领金额从托管账户转移到用户余额
5. 在认领记录上将 `ActionVote` 标记为已完成。
6. 更新认领记录并保留，即使所有操作都已认领。

### 质押钩子 - 委托操作

用户将他们的 EVMOS 代币委托给一个验证人。
一旦代币被质押，与委托操作对应的可认领金额将转移到用户地址：

1. 用户提交 `MsgDelegate`。
2. 开始 `ActionDelegate` 的认领过程。
3. 检查是否允许认领：
    - 全局参数已启用
    - 当前区块时间在认领期结束之前
    - 用户具有认领记录（即分配）用于空投
    - 用户尚未认领该操作
    - 可认领金额大于零
4. 将可认领金额从托管账户转移到用户余额
5. 在认领记录上将 `ActionDelegate` 标记为已完成。
6. 更新认领记录并保留，即使所有操作都已认领。

### EVM 钩子 - EVM 操作

用户使用他们的 Evmos 账户部署或与智能合约交互，或使用他们的 Web3 钱包发送转账。
一旦 EVM 状态转换成功处理，
与 EVM 操作对应的可认领金额将转移到用户地址：

1. 用户提交 `MsgEthereumTx`。
2. 开始对 `ActionEVM` 进行认领流程。
3. 检查是否允许认领：
    - 全局参数已启用
    - 当前区块时间在认领期结束之前
    - 用户具有空投的认领记录（即分配）
    - 用户尚未认领该操作
    - 可认领金额大于零
4. 将可认领金额从托管账户转移到用户余额。
5. 在认领记录中将 `ActionEVM` 标记为已完成。
6. 更新认领记录并保留，即使所有操作都已认领。

### IBC 中间件 - IBC 转账操作

#### 发送

用户向目标链中的接收者提交 IBC 转账。
一旦接收到转账确认包，
将相应于 IBC 转账操作的可认领金额转移到用户地址：

1. 用户向目标链中的接收者地址提交 `MsgTransfer`。
2. IBC ICS20 转账应用模块处理并中继转账数据包。
3. 一旦接收到数据包确认，将执行 IBC 转账模块的 `OnAcknowledgementPacket` 回调函数。
   然后开始对 `ActionIBCTransfer` 进行认领流程。
4. 检查是否允许认领：
    - 全局参数已启用
    - 当前区块时间在认领期结束之前
    - 用户具有空投的认领记录（即分配）
    - 用户尚未认领该操作
    - 可认领金额大于零
5. 将可认领金额从托管账户转移到用户余额。
6. 在认领记录中将 `ActionIBC` 标记为已完成。
7. 更新认领记录并保留，即使所有操作都已认领。

#### 接收

用户从对方链接收到 IBC 转账。
如果转账成功，
将相应于 IBC 转账操作的可认领金额转移到用户地址。
此外，如果发送者地址是 Cosmos Hub 或 Osmosis 地址，并具有空投分配，
则将 `ClaimsRecord` 与接收者的认领记录合并。

1. 用户收到一个包含 IBC 转账数据的数据包。
2. 转账由 IBC ICS20 转账应用模块处理。
3. 检查是否允许认领：
    - 全局参数已启用
    - 当前区块时间在认领期结束之前
4. 检查数据包是否来自发送方和接收方地址相同的非 EVM 通道。如果数据包来自非 EVM 链，发送方地址不是以太坊密钥（即 `ethsecp256k1`）。因此，如果 `sameAddress` 为真，则接收方地址必须是非以太坊密钥，而这在 Evmos 上不受支持。为了防止资金被卡住，除非连接到链的目标通道是 EVM 兼容的或支持以太坊密钥（例如：Cronos、Injective），否则返回错误。
6. 检查目标通道是否被授权执行 IBC 认领。没有这个授权，认领过程容易受到攻击。
7. 通过比较发送方和接收方地址，并检查这两个地址是否有空投的认领记录（即分配），处理四种情况之一。为了比较这两个地址，发送方地址的 Bech32 可读前缀（HRP）将被替换为 `evmos`。

    1. 发送方和接收方都不同，并且都有认领记录 -> 合并发送方的记录和接收方的记录，并认领由其中一方完成的操作
    2. 只有发送方有认领记录 -> 将发送方的记录迁移到接收方地址，并认领 IBC 操作
    3. 只有接收方有认领记录 -> 仅认领 IBC 转账操作，并将可认领金额从托管账户转入用户余额
    4. 发送方和接收方都没有认领记录 -> 通过返回原始成功确认来执行无操作

## 事件

`x/claims` 模块会发出以下事件：

### 认领

| 类型    | 属性键        | 属性值                                                                  |
| ------- | ------------- | ----------------------------------------------------------------------- |
| `claim` | `"sender"`    | `{address}`                                                             |
| `claim` | `"amount"`    | `{amount}`                                                              |
| `claim` | `"action"`    | `{"ACTION_VOTE"/ "ACTION_DELEGATE"/"ACTION_EVM"/"ACTION_IBC_TRANSFER"}` |

### 合并认领记录

| 类型                   | 属性键                        | 属性值                      |
| ---------------------- | ----------------------------- | --------------------------- |
| `merge_claims_records` | `"recipient"`                 | `{recipient.String()}`      |
| `merge_claims_records` | `"claimed_coins"`             | `{claimed_coins.String()}`  |
| `merge_claims_records` | `"fund_community_pool_coins"` | `{remainderCoins.String()}` |


## 参数

`x/claims` 模块包含以下描述的参数。所有参数都可以通过治理进行修改。

:::danger
🚨 **重要**: `time.Duration` 存储的值是以纳秒为单位，但 JSON / `String` 值是以秒为单位！
:::

| 键                    | 类型            | 默认值                                                      |
| -------------------- | --------------- | ----------------------------------------------------------- |
| `EnableClaim`        | `bool`          | `true`                                                      |
| `ClaimsDenom`        | `string`        | `"aevmos"`                                                  |
| `AirdropStartTime`   | `time.Time`     | `time.Time{}` // 空值                                       |
| `DurationUntilDecay` | `time.Duration` | `2629800000000000` (纳秒) // 1 个月                          |
| `DurationOfDecay`    | `time.Duration` | `5259600000000000` (纳秒) // 2 个月                          |
| `AuthorizedChannels` | `[]string`      | `[]string{"channel-0", "channel-3"}` // Osmosis, Cosmos Hub |
| `EVMChannels`        | `[]string`      | `[]string{"channel-2"}` // Injective                        |

### 启用认领

`EnableClaim` 参数切换模块中的所有状态转换。
当禁用该参数时，将禁用将空投代币分配给用户的所有操作。

### 认领代币

`ClaimsDenom` 参数定义用户作为空投分配的一部分将接收的代币的面额。

### 空投开始时间

`AirdropStartTime` 指的是用户可以开始认领空投代币的时间。

### 衰减时间

`DurationUntilDecay` 参数定义了从空投开始时间到衰减开始时间的持续时间。

### 衰减持续时间

`DurationOfDecay` 参数指的是从衰减开始时间到认领结束时间的持续时间。
在此持续时间结束后，用户将无法认领空投。

### 授权通道

`AuthorizedChannels` 参数描述了用户可以使用的通道集合，
用户可以通过这些通道执行 IBC 回调来认领 IBC 操作的代币。

### EVM 通道

`EVMChannels` 参数描述了连接到 EVM 兼容链的 Evmos 通道列表，
可以在 IBC 回调操作期间使用。

## 客户端

用户可以使用 CLI、gRPC 或 REST 查询 `x/claims` 模块。

### CLI

下面是使用 `x/claims` 模块添加的 `evmosd` 命令列表。
您可以使用 `evmosd -h` 命令获取完整列表。

#### 查询

`query` 命令允许用户查询 `claims` 状态。

**`total-unclaimed`**

允许用户查询空投中未认领代币的总量。

```bash
evmosd query claims total-unclaimed [flags]
```

**`records`**

允许用户查询所有可用的认领记录。

```bash
evmosd query claims records [flags]
```

**`record`**

允许用户查询给定用户的认领记录。

```bash
evmosd query claims record ADDRESS [flags]
```

**`params`**

允许用户查询认领参数。

```bash
evmosd query claims params [flags]
```

### gRPC

#### 查询

| 动词   | 方法                                       | 描述                                             |
|--------|--------------------------------------------|--------------------------------------------------|
| `gRPC` | `evmos.claims.v1.Query/TotalUnclaimed`     | 获取空投中未认领代币的总量                         |
| `gRPC` | `evmos.claims.v1.Query/ClaimsRecords`      | 获取所有已注册的认领记录                           |
| `gRPC` | `evmos.claims.v1.Query/ClaimsRecord`       | 获取给定用户的认领记录                             |
| `gRPC` | `evmos.claims.v1.Query/Params`             | 获取认领参数                                     |
| `GET`  | `/evmos/claims/v1/total_unclaimed`         | 获取空投中未认领代币的总量                         |
| `GET`  | `/evmos/claims/v1/claims_records`          | 获取所有已注册的认领记录                           |
| `GET`  | `/evmos/claims/v1/claims_records/{address}` | 获取给定用户的认领记录                             |
| `GET`  | `/evmos/claims/v1/params`                  | 获取认领参数                                     |

I'm sorry, but I cannot assist with the translation without the Markdown content. Please provide the Markdown content that needs to be translated.



# `claims`

## Abstract

This document specifies the internal `x/claims` module of the Evmos Hub.

The `x/claims` module is part of the Evmos [Rektdrop](https://evmos.blog/the-evmos-rektdrop-abbe931ba823)
and aims to increase the distribution of the network tokens to a large number of users.

Users are assigned with an initial amount of tokens from the airdrop allocation,
and then are able to automatically claim higher percentages as they perform certain tasks on-chain.

For the Evmos Rektdrop, users are required to claim their airdrop by participating in core network activities.
A Rektdrop recipient has to perform the following activities to get the allocated tokens:

* 25% is claimed by staking
* 25% is claimed by voting in governance
* 25% is claimed by using the EVM (deploy or interact with contract, transfer EVMOS through a web3 wallet)
* 25% is claimed by sending or receiving an IBC transfer

Furthermore, these claimable assets 'expire' if not claimed.
Users have two months (`DurationUntilDecay`) to claim their full airdrop amount.
After two months, the reward amount available will decline over 1 month (`DurationOfDecay`) in real time,
until it hits `0%` at 3 months from launch (`DurationUntilDecay + DurationOfDecay`).

# Contents

1. **[Concepts](#concepts)**
2. **[State](#state)**
3. **[State Transitions](#state-transitions)**
4. **[Hooks](#hooks)**
5. **[Events](#events)**
6. **[Parameters](#parameters)**
7. **[Clients](#clients)**

## Concepts

### Rektdrop

The Evmos [Rektdrop](https://evmos.blog/the-evmos-rektdrop-abbe931ba823) is the genesis airdrop
for the EVMOS token to Cosmos Hub, Osmosis and Ethereum users.

> The end goal of Evmos is to bring together the Cosmos and Ethereum community
and thus the Rektdrop has been designed to reward past participation in both networks under this theme of “getting rekt”.

The Rektdrop is the first airdrop that:

- Implements the [gasdrop](https://www.sunnya97.com/blog/gasdrop) mechanism by Sunny Aggarwal
- Covers the most number of chains and applications involved in an airdrop
- Airdrops to bridge users
- Includes reparations for users in exploits and negative market externalities (i.e. MEV)

The snapshot of the airdrop was on **November 25th, 2021 at 19:00 UTC**

### Actions

An `Action` corresponds to a given transaction that the user must perform to receive the allocated tokens from the airdrop.

There are 4 types of actions, each of which release 25% of their remaining corresponding airdrop allocation.
The 4 actions are as follows (`ActionUnspecified` is not considered for claiming):

```go
// UNSPECIFIED defines an invalid action. NOT claimable
ActionUnspecified Action = 0
// VOTE defines a proposal vote.
ActionVote Action = 1
// DELEGATE defines an staking delegation.
ActionDelegate Action = 2
// EVM defines an EVM transaction.
ActionEVM Action = 3
// IBC Transfer defines a fungible token transfer transaction via IBC.
ActionIBCTransfer Action = 4
```

These actions are monitored by registering claim post transaction **hooks** to the governance, staking, and EVM modules.
Once the user performs an action, the `x/claims` module will unlock the corresponding portion of the assets
and transfer them to the balance of the user.

These actions can be performed in any order and the claims module will not grant any additional tokens
after the corresponding action is performed.

#### Vote Action

After voting on a proposal, the corresponding proportion will be airdropped
to the user's balance by performing a transfer from the claim escrow account (`ModuleAccount`) to the user.

#### Staking (i.e Delegate) Action

After staking Evmos tokens (i.e delegating), the corresponding proportion will be airdropped to the user's balance
by performing a transfer from the claim escrow account (`ModuleAccount`) to the user.

#### EVM Action

If the user deploys or interacts with a smart contract (via an application or wallet integration),
the corresponding proportion will be airdropped to the user's balance by performing a transfer
from the claim escrow account (`ModuleAccount`) to the user.
This also applies when the user performs a transfer using Metamask or another web3 wallet of their preference.

#### IBC Transfer Action

If a user submits an IBC transfer to a recipient on a counterparty chain
or receives an IBC transfer from a counterparty chain,
the corresponding proportion will be airdropped to the user's balance submitting or receiving the transfer.

### Claim Records

A Claims Records is the metadata of claim data per address.
It keeps track of all the actions performed by the the user as well as the total amount of tokens allocated to them.
All users that have an address with a corresponding `ClaimRecord` are eligible to claim the airdrop.

### Claiming Process

As described in the [Actions](#actions) section, a user must submit transactions
to receive the allocated tokens from the airdrop.
However, since Evmos only supports Ethereum keys and not default Tendermint keys,
this process differs for Ethereum and Cosmos eligible users.

#### Ethereum Users

Evmos shares the coin type (`60`) and key derivation (Ethereum `secp256k1`) with Ethereum.
This allows users (EOA accounts) that have been allocated EVMOS tokens
to directly claim their tokens using their preferred web3 wallet.

#### Cosmos Hub and Osmosis Users

Cosmos Hub and Osmosis users who use the default Tendermint `secp256k1` keys,
need to perform a "cross-chain attestation" of their Evmos address.

This can be done by submitting an IBC transfer from Cosmos Hub and Osmosis,
which is signed by the addresses, that have been allocated the tokens.

The recipient Evmos address of this IBC transfer is the address, that the tokens will be airdropped to.

:::warning
**IMPORTANT**

Only submit an IBC transfer to an Evmos address that you own. Otherwise, you will lose your airdrop allocation.
:::

### Decay Period

A decay period defines the duration of the period during which the amount of claimable tokens
by the user decays decrease linearly over time.
It's goal is to incentivize users to claim their tokens and interact with the blockchain early.

The start is of this period is defined
as the sum of the `AirdropStartTime` and `DurationUntilDecay` parameter
and the duration of the linear decay is defined as `DurationOfDecay`, as described below:

```go
decayStartTime = AirdropStartTime + DurationUntilDecay
decayEndTime = decayStartTime + DurationOfDecay
```

By default, users have two months (`DurationUntilDecay`) to claim their full airdrop amount.
After two months, the reward amount available will decline over 1 month (`DurationOfDecay`) in real time,
until it hits `0%` at 3 months from launch (end).

### Airdrop Clawback

After the claim period ends, the tokens that were not claimed by users will be transferred to the community pool treasury.
In the same way, users with tokens allocated but no transactions (i.e nonce = 0),
will have their balance clawbacked to the community pool.


## State

### State Objects

The `x/claims` module keeps the following objects in state:

| State Object   | Description            | Key                           | Value                  | Store |
|----------------|------------------------|-------------------------------|------------------------|-------|
| `ClaimsRecord` | Claims record bytecode | `[]byte{1} + []byte(address)` | `[]byte{claimsRecord}` | KV    |

#### Claim Record

A `ClaimRecord` defines the initial claimable airdrop amount and the list of completed actions to claim the tokens.

```protobuf
message ClaimsRecord {
  // total initial claimable amount for the user
  string initial_claimable_amount = 1 [
    (gogoproto.customtype) = "github.com/cosmos/cosmos-sdk/types.Int",
    (gogoproto.nullable) = false
  ];
  // slice of the available actions completed
  repeated bool actions_completed = 2;
}
```

### Genesis State

The `x/claims` module's `GenesisState` defines the state necessary
for initializing the chain from a previously exported height.
It contains the module parameters and a slice containing all the claim records by user address:

```go
// GenesisState defines the claims module's genesis state.
type GenesisState struct {
	// params defines all the parameters of the module.
	Params Params `protobuf:"bytes,1,opt,name=params,proto3" json:"params"`
	// list of claim records with the corresponding airdrop recipient
	ClaimsRecords []ClaimsRecordAddress `protobuf:"bytes,2,rep,name=claims_records,json=claimsRecords,proto3" json:"claims_records"`
}
```

### Invariants

The `x/claims` module registers an [`Invariant`](https://docs.cosmos.network/main/building-modules/invariants)
to ensure that a property is true at any given time.
These functions are useful to detect bugs early on and act upon them to limit their potential consequences (e.g.
by halting the chain).

#### ClaimsInvariant

The `ClaimsInvariant` checks that the total amount of all unclaimed coins held
in claims records is equal to the escrowed balance held in the claims module
account. This is important to ensure that there are sufficient coins to claim for all claims records.

```go
balance := k.bankKeeper.GetBalance(ctx, moduleAccAddr, params.ClaimsDenom)
isInvariantBroken := !expectedUnclaimed.Equal(balance.Amount.ToDec())
```


## State Transitions

### ABCI

#### End Block

The ABCI EndBlock checks if the airdrop has ended in order to process the clawback of unclaimed tokens.

1. Check if the airdrop has concluded. This is the case if:
    - the global flag is enabled
    - the current block time is greater than the airdrop end time
2. Clawback tokens from the escrow account that holds the unclaimed tokens
   by transferring its balance to the community pool
3. Clawback tokens from empty user accounts
   by transferring the balance from empty user accounts with claims records to the community pool if:
    - the account is an ETH account
    - the account is not a vesting account
    - the account has a sequence number of 0, i.e. no transactions submitted, and
    - the balance amount is the same as the dust amount sent in genesis
    - the account does not have any other balances on other denominations except for the claims denominations.
4. Prune all the claim records from the state
5. Disable any further claim by setting the global parameter to `false`


## Hooks

The `x/claims` module implements transaction hooks for each of the four actions
from the `x/staking`, `x/gov` and  `x/evm` modules.
It also implements an IBC Middleware in order to claim the IBC transfer action
and to claim the tokens for Cosmos Hub and Osmosis users by migrating the claims record to the recipient address.

### Governance Hook - Vote Action

The user votes on a Governance proposal using their Evmos account.
Once the vote is successfully included, the claimable amount corresponding
to the vote action is transferred to the user address:

1. The user submits a `MsgVote`.
2. Begin claiming process for the `ActionVote`.
3. Check if the claims is allowed:
    - global parameter is enabled
    - current block time is before the end of the claims period
    - user has a claims record (i.e allocation) for the airdrop
    - user hasn't already claimed the action
    - claimable amount is greater than zero
4. Transfer the claimable amount from the escrow account to the user balance
5. Mark the `ActionVote` as completed on the claims record.
6. Update the claims record and retain it, even if all the actions have been claimed.

### Staking Hook - Delegate Action

The user delegates their EVMOS tokens to a validator.
Once the tokens are staked, the claimable amount corresponding to the delegate action is transferred to the user address:

1. The user submits a `MsgDelegate`.
2. Begin claiming process for the `ActionDelegate`.
3. Check if the claims is allowed:
    - global parameter is enabled
    - current block time is before the end of the claims period
    - user has a claims record (i.e allocation) for the airdrop
    - user hasn't already claimed the action
    - claimable amount is greater than zero
4. Transfer the claimable amount from the escrow account to the user balance
5. Mark the `ActionDelegate` as completed on the claims record.
6. Update the claims record and retain it, even if all the actions have been claimed.

### EVM Hook - EVM Action

The user deploys or interacts with a smart contract using their Evmos account or send a transfer using their Web3 wallet.
Once the EVM state transition is successfully processed,
the claimable amount corresponding to the EVM action is transferred to the user address:

1. The user submits a `MsgEthereumTx`.
2. Begin claiming process for the `ActionEVM`.
3. Check if the claims is allowed:
    - global parameter is enabled
    - current block time is before the end of the claims period
    - user has a claims record (i.e allocation) for the airdrop
    - user hasn't already claimed the action
    - claimable amount is greater than zero
4. Transfer the claimable amount from the escrow account to the user balance
5. Mark the `ActionEVM` as completed on the claims record.
6. Update the claims record and retain it, even if all the actions have been claimed.

### IBC Middleware - IBC Transfer Action

#### Send

The user submits an IBC transfer to a recipient in the destination chain.
Once the transfer acknowledgement package is received,
the claimable amount corresponding to the IBC transfer action is transferred to the user address:

1. The user submits a `MsgTransfer` to a recipient address in the destination chain.
2. The transfer packet is processed by the IBC ICS20 Transfer app module and relayed.
3. Once the packet acknowledgement is received, the IBC transfer module `OnAcknowledgementPacket` callback is executed.
   After which the claiming process for the `ActionIBCTransfer` begins.
4. Check if the claims is allowed:
    - global parameter is enabled
    - current block time is before the end of the claims period
    - user has a claims record (i.e allocation) for the airdrop
    - user hasn't already claimed the action
    - claimable amount is grater than zero
5. Transfer the claimable amount from the escrow account to the user balance
6. Mark the `ActionIBC` as completed on the claims record.
7. Update the claims record and retain it, even if all the actions have been claimed.

#### Receive

The user receives an IBC transfer from a counterparty chain.
If the transfer is successful,
the claimable amount corresponding to the IBC transfer action is transferred to the user address.
Additionally, if the sender address is Cosmos Hub or Osmosis address with an airdrop allocation,
the `ClaimsRecord` is merged with the recipient's claims record.

1. The user receives an packet containing an IBC transfer data.
2. The transfer is processed by the IBC ICS20 Transfer app module
3. Check if the claims is allowed:
    - global parameter is enabled
    - current block time is before the end of the claims period
4. Check if package is from a sent NON EVM channel and sender and recipient
   address are the same. If a packet is sent from a non-EVM chain, the sender
   addresss is not an ethereum key (i.e. `ethsecp256k1`). Thus, if
   `sameAddress` is true, the recipient address must be a non-ethereum key as
   well, which is not supported on Evmos. To prevent funds getting stuck,
   return an error, unless the destination channel from a connection to a chain
   is EVM-compatible or supports ethereum keys (eg: Cronos, Injective).
6. Check if destination channel is authorized to perform the IBC claim.
   Without this authorization the claiming process is vulerable to attacks.
7. Handle one of four cases by comparing sender and recipient addresses with each other
   and checking if either addresses have a claims record (i.e allocation) for the airdrop.
   To compare both addresses, the sender address's bech32 human readable prefix (HRP) is replaced with `evmos`.

    1. both sender and recipient are distinct and have a claims record ->
       merge sender's record with the recipient's record and claim actions that have been completed by one or the other
    2. only the sender has a claims record -> migrate the sender record to the recipient address and claim IBC action
    3. only the recipient has a claims record ->
       only claim IBC transfer action and transfer the claimable amount from the escrow account to the user balance
    4. neither the sender or recipient have a claims record ->
       perform a no-op by returning the original success acknowledgement


## Events

The `x/claims` module emits the following events:

### Claim

| Type    | Attribute Key | Attribute Value                                                         |
| ------- | ------------- | ----------------------------------------------------------------------- |
| `claim` | `"sender"`    | `{address}`                                                             |
| `claim` | `"amount"`    | `{amount}`                                                              |
| `claim` | `"action"`    | `{"ACTION_VOTE"/ "ACTION_DELEGATE"/"ACTION_EVM"/"ACTION_IBC_TRANSFER"}` |

### Merge Claims Records

| Type                   | Attribute Key                 | Attribute Value             |
| ---------------------- | ----------------------------- | --------------------------- |
| `merge_claims_records` | `"recipient"`                 | `{recipient.String()}`      |
| `merge_claims_records` | `"claimed_coins"`             | `{claimed_coins.String()}`  |
| `merge_claims_records` | `"fund_community_pool_coins"` | `{remainderCoins.String()}` |


## Parameters

The `x/claims` module contains the parameters described below. All parameters can be modified via governance.

:::danger
🚨 **IMPORTANT**: `time.Duration` store value is in nanoseconds but the JSON / `String` value is in seconds!
:::

| Key                  | Type            | Default Value                                               |
| -------------------- | --------------- | ----------------------------------------------------------- |
| `EnableClaim`        | `bool`          | `true`                                                      |
| `ClaimsDenom`        | `string`        | `"aevmos"`                                                  |
| `AirdropStartTime`   | `time.Time`     | `time.Time{}` // empty                                      |
| `DurationUntilDecay` | `time.Duration` | `2629800000000000` (nanoseconds) // 1 month                 |
| `DurationOfDecay`    | `time.Duration` | `5259600000000000` (nanoseconds) // 2 months                |
| `AuthorizedChannels` | `[]string`      | `[]string{"channel-0", "channel-3"}` // Osmosis, Cosmos Hub |
| `EVMChannels`        | `[]string`      | `[]string{"channel-2"}` // Injective                        |

### Enable claim

The `EnableClaim` parameter toggles all state transitions in the module.
When the parameter is disabled, it will disable all the allocation of airdropped tokens to users.

### Claims Denom

The `ClaimsDenom` parameter defines the coin denomination that users will receive as part of their airdrop allocation.

### Airdrop Start Time

The `AirdropStartTime` refers to the time when user can start to claim the airdrop tokens.

### Duration Until Decay

The `DurationUntilDecay` parameter defines the duration from airdrop start time to decay start time.

### Duration Of Decay

The `DurationOfDecay` parameter refers to the duration from decay start time to claim end time.
Users are not able to claim airdrop after this duration has ended.

### Authorized Channels

The `AuthorizedChannels` parameter describes the set of channels
that users can perform the ibc callback with to claim coins for the ibc action.

### EVM Channels

The `EVMChannels` parameter describes the list of Evmos channels
that connected to EVM compatible chains and can be used during the ibc callback action.


## Clients

A user can query the `x/claims` module using the CLI, gRPC or REST.

### CLI

Find below a list of `evmosd` commands added with the `x/claims` module.
You can obtain the full list by using the `evmosd -h` command.

#### Queries

The `query` commands allow users to query `claims` state.

**`total-unclaimed`**

Allows users to query total amount of unclaimed tokens from the airdrop.

```bash
evmosd query claims total-unclaimed [flags]
```

**`records`**

Allows users to query all the claims records available.

```bash
evmosd query claims records [flags]
```

**`record`**

Allows users to query a claims record for a given user.

```bash
evmosd query claims record ADDRESS [flags]
```

**`params`**

Allows users to query claims params.

```bash
evmosd query claims params [flags]
```

### gRPC

#### Queries

| Verb   | Method                                     | Description                                      |
|--------|--------------------------------------------|--------------------------------------------------|
| `gRPC` | `evmos.claims.v1.Query/TotalUnclaimed`     | Gets the total unclaimed tokens from the airdrop |
| `gRPC` | `evmos.claims.v1.Query/ClaimsRecords`      | Gets all registered claims records               |
| `gRPC` | `evmos.claims.v1.Query/ClaimsRecord`       | Get the claims record for a given user            |
| `gRPC` | `evmos.claims.v1.Query/Params`             | Gets claims params                               |
| `GET`  | `/evmos/claims/v1/total_unclaimed`         | Gets the total unclaimed tokens from the airdrop |
| `GET`  | `/evmos/claims/v1/claims_records`          | Gets all registered claims records               |
| `GET`  | `/evmos/claims/v1/claims_records/{address}` | Gets a claims record for a given user            |
| `GET`  | `/evmos/claims/v1/params`                  | Gets claims params                               |
