# `recovery`

恢复被卡在不支持的Evmos账户上的代币。

## 摘要

本文档规定了Evmos Hub的 `x/recovery` 模块。

`x/recovery` 模块使得在Evmos上的用户能够恢复被锁定的资金，
这些资金被转移到了在Evmos上不支持的账户中。
这种情况特别发生在Evmos的初始发布（`v1.1.2`）之后，
用户通过IBC将代币转移到了一个 `secp256k1` 的Evmos地址上，
以便[领取他们的空投](claims.md)。
为了与EVM兼容，
[Evmos上的密钥](https://docs.evmos.org/protocol/concepts/accounts#evmos-accounts)是使用 `eth_secp256k1` 密钥类型生成的，
这导致了与其他Cosmos链使用的 `secp256k1` 密钥类型不同的地址派生方式。

在Evmos重新启动时，
不支持账户上锁定的代币价值为36,291.28美元的OSMO代币和268.86美元的ATOM代币，
根据[Mintscan](https://www.mintscan.io/evmos/assets)区块浏览器的数据。
通过 `x/recovery` 模块，用户可以通过从授权的IBC通道（即OSMO的Osmosis，ATOM的Cosmos Hub）执行IBC转账，
将这些代币恢复到其原始链上的自己的地址。

## 目录

1. **[概念](#概念)**
2. **[钩子](#钩子)**
3. **[事件](#事件)**
4. **[参数](#参数)**
5. **[客户端](#客户端)**

## 概念

### 密钥生成

`secp256k1` 是用于生成加密公钥的椭圆曲线的参数。
与比特币一样，像Cosmos链这样的IBC兼容链使用 `secp256k1` 进行公钥生成。

一些链使用不同的椭圆曲线来生成公钥。
一个例子是以太坊和Evmos链使用的 `eth_secp256k1` 用于生成公钥。

```go
// Generate new random ethsecp256k1 private key and address

ethPrivKey, err := ethsecp256k1.GenerateKey()
ethsecpAddr := sdk.AccAddress(ethPrivKey.PubKey().Address())

// Bech32 "evmos" address
ethsecpAddrEvmos := sdk.AccAddress(ethPk.PubKey().Address()).String()

// We can also change the HRP to use "cosmos"
ethsecpAddrCosmos := sdk.MustBech32ifyAddressBytes(sdk.Bech32MainPrefix, ethsecpAddr)
```

上面的示例代码演示了在Evmos上创建一个简单的用户账户。
在第二行，使用 `eth_secp256k1` 曲线生成了一个私钥，
该私钥用于创建一个可读的 `PubKey` 字符串。
有关账户的更详细信息，请查看官方Evmos文档中的[账户部分](https://docs.evmos.org/protocol/concepts/accounts#evmos-accounts)。

### 被卡住的资金

`x/recovery` 模块的主要用途是恢复发送到不支持的 Evmos 地址的代币。
这些代币被称为“被卡住的”，因为账户的所有者无法签署将代币转移到其他账户的交易。
所有者只持有用于在 Evmos 上签署交易的 `eth_secp256k1` 公钥的私钥，而不是其他不受支持的密钥（即 `secp256k1` 密钥）。
由于椭圆曲线的不兼容性，他们无法使用发送代币的账户的密钥来转移代币。

### 恢复

在初始的 Evmos 发布（`v1.1.2`）之后，从具有和不具有认领记录（空投分配）的账户中卡住了代币：

1. Osmosis/Cosmos Hub 账户没有认领记录，将 IBC 转账发送到 Evmos 的 `secp256k1` 接收地址

   **后果**

    - IBC 转账的凭证被卡在接收者的余额中

   **恢复步骤**

    - 接收者可以从他们的 Osmosis / Cosmos Hub 账户（即 `osmo1...` 或 `cosmos1...`）发送 IBC 转账
      到相同的 Evmos 账户（`evmos1...`）以恢复代币，
      并将它们转发到相应的发送链（Osmosis 或 Cosmos Hub）

2. Osmosis/Cosmos Hub 账户具有认领记录，将 IBC 转账发送到 Evmos 的 `secp256k1` 接收地址

   **后果**

    - IBC 转账的凭证被卡在接收者的余额中
    - IBC 转账操作已被认领，
      并且 EVMOS 奖励已转移到接收者的 Evmos `secp256k1` 账户，
      导致 EVMOS 代币被卡住。
    - 发送者的认领记录已迁移到接收者的 Evmos `secp256k1` 账户

   **恢复步骤**

    - 接收者可以从他们的 Osmosis / Cosmos Hub 账户（即 `osmo1...` 或 `cosmos1...`）发送 IBC 转账
      到相同的 Evmos 账户（`evmos1...`）以恢复代币，
      并将它们转发到相应的发送链（Osmosis 或 Cosmos Hub）
    - 再次将认领记录迁移到有效的账户，以便可以认领剩余的 3 个操作
    - 使用恢复后的认领记录重新启动链

### IBC中间件堆栈

#### 中间件排序

IBC中间件在核心IBC和底层应用程序之间添加自定义逻辑。
中间件被实现为堆栈，以便应用程序可以定义多个自定义行为层。

中间件的顺序很重要。
从IBC核心到应用程序的函数调用从顶级中间件到底层中间件，然后到应用程序，
而从应用程序到IBC核心的函数调用首先经过底层中间件，然后按顺序经过顶级中间件，最后到核心IBC处理程序。
因此，相同的中间件集以不同的顺序放置可能会产生不同的效果。

在数据包执行期间，堆栈中的每个中间件将按照创建时定义的顺序执行（从顶部到底部）。

对于Evmos，中间件堆栈的排序如下（从顶部到底部）：

1. IBC转账
2. 认领中间件
3. 恢复中间件

这意味着首先执行IBC转账，然后尝试认领，最后执行恢复。
通过按照这个顺序执行操作，我们允许用户收回用于触发恢复的代币。

**示例执行顺序**

1. 用户尝试恢复在Evmos链上被卡住的`1000aevmos`。
2. 用户通过IBC交易从Osmosis发送`100uosmo`到Evmos。
3. Evmos接收到交易，并经过IBC堆栈：
    1. **IBC转账**：将`100uosmo`的IBC凭证添加到evmos上的用户余额中。
    2. **认领中间件**：由于`sender=receiver` -> 执行无操作
    3. **恢复中间件**：由于`sender=receiver` -> 通过在Osmosis链上从`receiver`向`sender`发送IBC转账来恢复用户余额（`1000aevmos`和`100uosmo`）。
4. 用户在Osmosis上收到`100uosmo`和`1000aevmos`（IBC凭证）。

#### 执行错误

在堆栈执行的任何点上，IBC交易可能失败，
在这种情况下，恢复将不会被交易触发，因为它将回滚到先前的状态。

所以，如果在任何时候，无论是 IBC 转账还是索赔中间件返回错误，
那么恢复中间件将不会被执行。


## 钩子

`x/recovery` 模块允许状态转换，将之前转移到 EVMOS 的 IBC 代币返回到源链的源账户中，
并使用 `Keeper.OnRecvPacket` 回调函数。源链必须经过授权。

### 提取

用户执行 IBC 转账，将之前转移到他们的 Cosmos `secp256k1` 地址的代币返回，
而不是 Ethereum `ethsecp256k1` 地址。该行为是使用 IBC `OnRecvPacket` 回调函数实现的。

1. 用户通过从授权链上的地址（例如 `cosmos1...`）向他们的 evmos `secp2561` 地址（即 `evmos1`）执行 IBC 转账，
   该地址持有被卡住的代币。这是通过使用 [`FungibleTokenPacket`](https://github.com/cosmos/ibc/blob/master/spec/app/ics-020-fungible-token-transfer/README.md) IBC 数据包来完成的。

2. 检查是否满足提取条件，如果有任何条件不满足，则跳过到下一个中间件：

    1. 全局启用了恢复功能
    2. 通道已经授权
    3. 通道不是 EVM 通道（因为 EVM 支持 `eth_secp256k1` 密钥，代币不会被卡住）
    4. 发送者和接收者地址属于同一个账户，因为只有在向 Evmos 上的发送者自己的账户转账时才能进行恢复。
       因此，发送者和接收者地址都会从 `bech32` 转换为 `sdk.AccAddress`。
    5. 发送者/接收者账户不是锁定或模块账户
    6. 接收者公钥不是受支持的密钥（`eth_secp256k1`、`amino multisig`、`ed25519`），
       因为在这种情况下，代币不会被卡住，不需要进行恢复

3. 检查发送者/接收者地址是否被 `x/bank` 模块阻止，
   并抛出一个确认错误，以防止进一步执行 IBC 中间件堆栈
4. 执行恢复操作，使用 IBC `OnRecvPacket` 回调函数将接收者的余额转回发送者地址。
   有两种情况：

    1. 从授权的源链首先转移：
        1. 将源链中起源的IBC代币退回
        2. 转移所有的Evmos原生代币
    2. 从其他授权的源链进行第二次及后续转移：
        1. 只退回源链中起源的IBC代币

5. 如果接收者没有任何余额，则不进行代币恢复


## 事件

`x/recovery` 模块会触发以下事件：

### 恢复

| 类型       |    属性键     |             属性值 |
| :--------- | :------------------- | :-------------------------- |
| `recovery` |       `sender`       |              `senderBech32` |
| `recovery` |      `receiver`      |           `recipientBech32` |
| `recovery` |       `amount`       |                    `amtStr` |
| `recovery` | `packet_src_channel` |      `packet.SourceChannel` |
| `recovery` |  `packet_src_port`   |         `packet.SourcePort` |
| `recovery` | `packet_dst_channel` |    `packet.DestinationPort` |
| `recovery` |  `packet_dst_port`   | `packet.DestinationChannel` |


## 参数

`x/recovery` 模块包含以下参数：

| 键                     |      类型       |             默认值 |
| :---------------------- | :-------------- | :------------------------ |
| `EnableRecovery`        |     `bool`      |                    `true` |
| `PacketTimeoutDuration` | `time.Duration` | `14400000000000`  // 4小时 |

### 启用恢复

`EnableRecovery` 参数用于切换恢复IBC中间件。
当该参数被禁用时，将禁止将卡住的代币恢复给用户。

### 数据包超时时长

`PacketTimeoutDuration` 参数是IBC数据包超时的时长，
当数据包超时时，交易将在对方链上回滚。


## 客户端

用户可以使用CLI、gRPC或REST查询`x/recovery` 模块。

### CLI

以下是添加了 `x/recovery` 模块的 `evmosd` 命令列表。
您可以使用 `evmosd -h` 命令获取完整列表。

#### 查询

查询命令允许用户查询恢复状态。

**`params`**
允许用户查询模块参数。

```bash
evmosd query recovery params [flags]
```

### gRPC

#### 查询

| 动词   |              方法              |           描述 |
| :----- | :------------------------------- | :-------------------- |
| `gRPC` | `evmos.recovery.v1.Query/Params` | `获取恢复参数` |
| `GET`  |   `/evmos/recovery/v1/params`    | `获取恢复参数` |


# `recovery`

Recover tokens that are stuck on unsupported Evmos accounts.

## Abstract

This document specifies the  `x/recovery` module of the Evmos Hub.

The `x/recovery` module enables users on Evmos to recover locked funds
that were transferred to accounts whose keys are not supported on Evmos.
This happened in particular after the initial Evmos launch (`v1.1.2`),
where users transferred tokens to a `secp256k1` Evmos address via IBC
in order to [claim their airdrop](claims.md).
To be EVM compatible,
[keys on Evmos](https://docs.evmos.org/protocol/concepts/accounts#evmos-accounts) are generated
using the `eth_secp256k1` key type which results in a different address derivation
than e.g. the `secp256k1` key type used by other Cosmos chains.

At the time of Evmos’ relaunch,
the value of locked tokens on unsupported accounts sits at $36,291.28 worth of OSMO and $268.86 worth of ATOM tokens
according to the [Mintscan](https://www.mintscan.io/evmos/assets) block explorer.
With the `x/recovery` module, users can recover these tokens back to their own addresses
in the originating chains by performing IBC transfers from authorized IBC channels
(i.e. Osmosis for OSMO, Cosmos Hub for ATOM).

## Contents

1. **[Concepts](#concepts)**
2. **[Hooks](#hooks)**
3. **[Events](#events)**
4. **[Parameters](#parameters)**
5. **[Clients](#clients)**

## Concepts

### Key generation

`secp256k1` refers to the parameters of the elliptic curve used in generating cryptographic public keys.
Like Bitcoin, IBC compatible chains like the Cosmos chain use `secp256k1` for public key generation.

Some chains use different elliptic curves for generating public keys.
An example is the`eth_secp256k1`used by Ethereum and Evmos chain for generating public keys.

```go
// Generate new random ethsecp256k1 private key and address

ethPrivKey, err := ethsecp256k1.GenerateKey()
ethsecpAddr := sdk.AccAddress(ethPrivKey.PubKey().Address())

// Bech32 "evmos" address
ethsecpAddrEvmos := sdk.AccAddress(ethPk.PubKey().Address()).String()

// We can also change the HRP to use "cosmos"
ethsecpAddrCosmos := sdk.MustBech32ifyAddressBytes(sdk.Bech32MainPrefix, ethsecpAddr)
```

The above example code demonstrates a simple user account creation on Evmos.
On the second line, a private key is generated using the `eth_secp256k1` curve,
which is used to create a human readable `PubKey` string.
For more detailed info on accounts,
please check the [accounts section](https://docs.evmos.org/protocol/concepts/accounts#evmos-accounts)
in the official Evmos documentation.

### Stuck funds

The primary use case of the `x/recovery` module is to enable the recovery of tokens,
that were sent to unsupported Evmos addresses.
These tokens are termed “stuck”, as the account’s owner cannot sign transactions
that transfer the tokens to other accounts.
The owner only holds the private key to sign transactions for its `eth_secp256k1` public keys on Evmos,
not other unsupported keys (i.e. `secp256k1` keys).
They are unable to transfer the tokens using the keys of the accounts through which they were sent
due to the incompatibility of their elliptic curves.

### Recovery

After the initial Evmos launch (`v1.1.2`), tokens got stuck from accounts with
and without claims records (airdrop allocation):

1. Osmosis/Cosmos Hub account without claims record sent IBC transfer to Evmos `secp256k1` receiver address

   **Consequences**

    - IBC vouchers from IBC transfer got stuck in the receiver’s balance

   **Recovery procedure**

    - The receiver can send an IBC transfer from their Osmosis / Cosmos Hub account (i.e `osmo1...` or `cosmos1...`)
      to its same Evmos account (`evmos1...`) to recover the tokens
      by forwarding them to the corresponding sending chain (Osmosis or Cosmos Hub)

2. Osmosis/Cosmos Hub account with claims record sent IBC transfer to Evmos `secp256k1` receiver address

   **Consequences**

    - IBC vouchers  from IBC transfer got stuck in the receiver’s balance
    - IBC Transfer Action was claimed
      and the EVMOS rewards were transferred to the receiver’s Evmos `secp256k1` account,
      resulting in stuck EVMOS tokens.
    - Claims record of the sender was migrated to the receiver’s Evmos `secp256k1` account

   **Recovery procedure**

    - The receiver can send an IBC transfer from their Osmosis / Cosmos Hub  account (i.e `osmo1...` or `cosmos1...`)
      to its same Evmos account (`evmos1...`) to recover the tokens
      by forwarding them to the corresponding sending chain (Osmosis or Cosmos Hub)
    - Migrate once again the claims record to a valid account so that the remaining 3 actions can be claimed
    - Chain is restarted with restored Claims records

### IBC Middleware Stack

#### Middleware ordering

The IBC middleware adds custom logic between the core IBC and the underlying application.
Middlewares are implemented as stacks so that applications can define multiple layers of custom behavior.

The order of middleware matters.
Function calls from IBC core to the application travel from top-level middleware to the bottom middleware
and then to the application,
whereas function calls from the application to IBC core go through the bottom middleware first
and then in order to the top middleware and then to core IBC handlers.
Thus, the same set of middleware put in different orders may produce different effects.

During packet execution each middleware in the stack will be executed in the order defined on creation
(from top to bottom).

For Evmos the middleware stack ordering is defined as follows (from top to bottom):

1. IBC Transfer
2. Claims Middleware
3. Recovery Middleware

This means that the IBC transfer will be executed first, then the claim will be attempted
and lastly the recovery will be executed.
By performing the actions in this order we allow the users to receive back the coins used to trigger the recover.

**Example execution order**

1. User attempts to recover `1000aevmos` that are stuck on the Evmos chain.
2. User sends `100uosmo` from Osmosis to Evmos through an IBC transaction.
3. Evmos receives the transaction, and goes through the IBC stack:
    1. **IBC transfer**: the `100uosmo` IBC vouchers are added to the user balance on evmos.
    2. **Claims Middleware**: since `sender=receiver` -> perform no-op
    3. **Recovery Middleware**: since `sender=receiver` -> recover user balance (`1000aevmos` and `100uosmo`)
       by sending an IBC transfer from `receiver` to the `sender` on the Osmosis chain.
4. User receives `100uosmo` and `1000aevmos` (IBC voucher) on Osmosis.

#### Execution errors

It is possible that the IBC transaction fails in any point of the stack execution
and in that case the recovery will not be triggered by the transaction, as it will rollback to the previous state.

So if at any point either the IBC transfer or the claims middleware return an error,
then the recovery middleware will not be executed.


## Hooks

The `x/recovery` module allows for state transitions that return IBC tokens
that were previously transferred to EVMOS back to the source chains into the source accounts
with the `Keeper.OnRecvPacket` callback.
The source chain must be authorized.

### Withdraw

A user performs an IBC transfer to return the tokens that they previously transferred
to their Cosmos `secp256k1` address instead of the Ethereum `ethsecp256k1` address.
The behavior is implemented using an IBC`OnRecvPacket` callback.

1. A user performs an IBC transfer to their own account by sending tokens from their address on an authorized chain
   (e.g. `cosmos1...`) to their evmos `secp2561` address (i.e. `evmos1`)  which holds the stuck tokens.
   This is done using a
   [`FungibleTokenPacket`](https://github.com/cosmos/ibc/blob/master/spec/app/ics-020-fungible-token-transfer/README.md)
   IBC packet.

2. Check that the withdrawal conditions are met and skip to the next middleware if any condition is not satisfied:

    1. recovery is enabled globally
    2. channel is authorized
    3. channel is not an EVM channel (as an EVM supports `eth_secp256k1` keys and tokens are not stuck)
    4. sender and receiver address belong to the same account as recovery
       is only possible for transfers to a sender's own account on Evmos.
       Both sender and recipient addresses are therefore converted from `bech32` to `sdk.AccAddress`.
    5. the sender/recipient account is a not vesting or module account
    6. recipient pubkey is not a supported key (`eth_secp256k1`, `amino multisig`, `ed25519`),
       as in this case tokens are not stuck and don’t require recovery

3. Check if sender/recipient address is blocked by the `x/bank` module
   and throw an acknowledgment error to prevent further execution along with the IBC middleware stack
4. Perform recovery to transfer the recipient’s balance back to the sender address with the IBC `OnRecvPacket` callback.
   There are two cases:

    1. First transfer from authorized source chain:
        1. sends back IBC tokens that originated from the source chain
        2. sends over all Evmos native tokens
    2. Second and further transfers from a different authorized source chain
        1. only sends back IBC tokens that originated from the source chain

5. If the recipient does not have any balance, return without recovering tokens


## Events

The `x/recovery` module emits the following event:

### Recovery

| Type       |    Attribute Key     |             Attribute Value |
| :--------- | :------------------- | :-------------------------- |
| `recovery` |       `sender`       |              `senderBech32` |
| `recovery` |      `receiver`      |           `recipientBech32` |
| `recovery` |       `amount`       |                    `amtStr` |
| `recovery` | `packet_src_channel` |      `packet.SourceChannel` |
| `recovery` |  `packet_src_port`   |         `packet.SourcePort` |
| `recovery` | `packet_dst_channel` |    `packet.DestinationPort` |
| `recovery` |  `packet_dst_port`   | `packet.DestinationChannel` |


## Parameters

The `x/recovery` module contains the following parameters:

| Key                     |      Type       |             Default Value |
| :---------------------- | :-------------- | :------------------------ |
| `EnableRecovery`        |     `bool`      |                    `true` |
| `PacketTimeoutDuration` | `time.Duration` | `14400000000000`  // 4hrs |

### Enable Recovery

The `EnableRecovery` parameter toggles Recovery IBC middleware.
When the parameter is disabled, it will disable the recovery of stuck tokens to users.

### Packet Timeout Duration

The `PacketTimeoutDuration` parameter is the duration before the IBC packet timeouts
and the transaction is reverted on the counter party chain.


## Clients

A user can query the `x/recovery` module using the CLI, gRPC or REST.

### CLI

Find below a list of `evmosd` commands added with the `x/recovery` module.
You can obtain the full list by using the `evmosd` -h command.

#### Queries

The query commands allow users to query Recovery state.

**`params`**
Allows users to query the module parameters.

```bash
evmosd query recovery params [flags]
```

### gRPC

#### Queries

| Verb   |              Method              |           Description |
| :----- | :------------------------------- | :-------------------- |
| `gRPC` | `evmos.recovery.v1.Query/Params` | `Get Recovery params` |
| `GET`  |   `/evmos/recovery/v1/params`    | `Get Recovery params` |

