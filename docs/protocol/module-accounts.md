---
sidebar_position: 4
---

# 模块账户

一些模块拥有自己的模块账户。可以将其视为只能由该模块控制的钱包。
下面是一张模块、它们对应的钱包地址和权限的表格：

## 模块账户列表

| 名称                    | 地址                                               | 权限               |
| :---------------------- | :------------------------------------------------- | :----------------- |
| `claims`                | [evmos15cvq3ljql6utxseh0zau9m8ve2j8erz89m5wkz](https://www.mintscan.io/evmos/account/evmos15cvq3ljql6utxseh0zau9m8ve2j8erz89m5wkz)   | `none`             |
| `erc20`                 | [evmos1glht96kr2rseywuvhhay894qw7ekuc4qg9z5nw](https://www.mintscan.io/evmos/account/evmos1glht96kr2rseywuvhhay894qw7ekuc4qg9z5nw)   | `minter` `burner`  |
| `fee_collector`         | [evmos17xpfvakm2amg962yls6f84z3kell8c5ljcjw34](https://www.mintscan.io/evmos/account/evmos17xpfvakm2amg962yls6f84z3kell8c5ljcjw34)   | `none`             |
| `incentives`            | [evmos1krxwf5e308jmclyhfd9u92kp369l083wn67k4q](https://www.mintscan.io/evmos/account/evmos1krxwf5e308jmclyhfd9u92kp369l083wn67k4q)   | `minter` `burner`  |
| `inflation`             | [evmos1d4e35hk3gk4k6t5gh02dcm923z8ck86qygxf38](https://www.mintscan.io/evmos/account/evmos1d4e35hk3gk4k6t5gh02dcm923z8ck86qygxf38)   | `minter`           |
| `transfer`              | [evmos1yl6hdjhmkf37639730gffanpzndzdpmhv788dt](https://www.mintscan.io/evmos/account/evmos1yl6hdjhmkf37639730gffanpzndzdpmhv788dt)   | `minter` `burner`  |
| `bonded_tokens_pool`    | [evmos1fl48vsnmsdzcv85q5d2q4z5ajdha8yu3h6cprl](https://www.mintscan.io/evmos/account/evmos1fl48vsnmsdzcv85q5d2q4z5ajdha8yu3h6cprl)   | `burner` `staking` |
| `not_bonded_tokens_pool`| [evmos1tygms3xhhs3yv487phx3dw4a95jn7t7lr6ys4t](https://www.mintscan.io/evmos/account/evmos1tygms3xhhs3yv487phx3dw4a95jn7t7lr6ys4t)   | `burner` `staking` |
| `gov`                   | [evmos10d07y265gmmuvt4z0w9aw880jnsr700jcrztvm](https://www.mintscan.io/evmos/account/evmos10d07y265gmmuvt4z0w9aw880jnsr700jcrztvm)   | `burner`           |
| `distribution`          | [evmos1jv65s3grqf6v6jl3dp4t6c9t9rk99cd8974jnh](https://www.mintscan.io/evmos/account/evmos1jv65s3grqf6v6jl3dp4t6c9t9rk99cd8974jnh)   | `none`             |
| `evm`                   | [evmos1vqu8rska6swzdmnhf90zuv0xmelej4lq0n56wq](https://www.mintscan.io/evmos/account/evmos1vqu8rska6swzdmnhf90zuv0xmelej4lq0n56wq)   | `minter` `burner`  |
| `ibc`                   | [evmos1a53udazy8ayufvy0s434pfwjcedzqv345dnt3x](https://www.mintscan.io/evmos/account/evmos1a53udazy8ayufvy0s434pfwjcedzqv345dnt3x)   | `minter` `burner`  |

## 账户权限

* `burner` 权限表示该账户有权限销毁或销毁代币。
* `minter` 权限表示该账户有权限铸造或创建新的代币。
* `staking` 权限表示该账户有权限代表其所有者对代币进行质押。

## IBC 模块账户

此外，与 IBC 转账相关联的模块账户。
对于每个 IBC 连接，当 Evmos 是源链时，都会有一个 `ModuleAccount` 类型的账户用于托管转移的代币。
它们的地址是使用账户名称的 SHA256 校验和的前 20 个字节派生的，并遵循 [ADR 028](https://github.com/cosmos/cosmos-sdk/blob/master/docs/architecture/adr-028-public-key-addresses.md) 中概述的格式：

```go
// accountName is composed by the current version the IBC transfer module supports (in this case, ics20-1), the portID (transfer) and the channelID
accountName := Version + "\0" + portID + "/" + channelID
addr := sha256.Sum256(accountName)[:20]

// example for channel-0
addr := sha256.Sum256("ics20-1\0transfer/channel-0")[:20]
```

可以使用 [`IBC-go` 上的 `GetEscrowAccount` 函数](https://github.com/cosmos/ibc-go/blob/c56f78905a5d2db01d867381d106c403fa9e5c4b/modules/apps/transfer/types/keys.go#L41-L55) 计算。

:::tip
**注意**：执行查询时，这些托管账户不会显示在以下命令的结果中：

```shell
evmosd q auth module-accounts
```

这是因为查询时使用的 [`GetModuleAccount` 函数](https://github.com/cosmos/cosmos-sdk/blob/74d7a0dfcd9f47d8a507205f82c264a269ef0612/x/auth/keeper/keeper.go#L194-L224) 仅考虑 [`AccountKeeper` 的 `permAddrs` 映射](https://github.com/cosmos/cosmos-sdk/blob/74d7a0dfcd9f47d8a507205f82c264a269ef0612/x/auth/keeper/keeper.go#L54-L68) 中的账户。
此地址映射在编译时设置，无法在运行时更改。
:::


---
sidebar_position: 4
---

# Module Accounts

Some modules have their own module account. Think of this as a wallet that can only be controlled by that module.
Below is a table of modules, their respective wallet addresses and permissions:

## List of Module Accounts

| Name                    | Address                                             | Permissions        |
| :---------------------- | :-------------------------------------------------- | :----------------- |
| `claims`                | [evmos15cvq3ljql6utxseh0zau9m8ve2j8erz89m5wkz](https://www.mintscan.io/evmos/account/evmos15cvq3ljql6utxseh0zau9m8ve2j8erz89m5wkz)   | `none`             |
| `erc20`                 | [evmos1glht96kr2rseywuvhhay894qw7ekuc4qg9z5nw](https://www.mintscan.io/evmos/account/evmos1glht96kr2rseywuvhhay894qw7ekuc4qg9z5nw)   | `minter` `burner`  |
| `fee_collector`         | [evmos17xpfvakm2amg962yls6f84z3kell8c5ljcjw34](https://www.mintscan.io/evmos/account/evmos17xpfvakm2amg962yls6f84z3kell8c5ljcjw34)   | `none`             |
| `incentives`            | [evmos1krxwf5e308jmclyhfd9u92kp369l083wn67k4q](https://www.mintscan.io/evmos/account/evmos1krxwf5e308jmclyhfd9u92kp369l083wn67k4q)   | `minter` `burner`  |
| `inflation`             | [evmos1d4e35hk3gk4k6t5gh02dcm923z8ck86qygxf38](https://www.mintscan.io/evmos/account/evmos1d4e35hk3gk4k6t5gh02dcm923z8ck86qygxf38)   | `minter`           |
| `transfer`              | [evmos1yl6hdjhmkf37639730gffanpzndzdpmhv788dt](https://www.mintscan.io/evmos/account/evmos1yl6hdjhmkf37639730gffanpzndzdpmhv788dt)   | `minter` `burner`  |
| `bonded_tokens_pool`    | [evmos1fl48vsnmsdzcv85q5d2q4z5ajdha8yu3h6cprl](https://www.mintscan.io/evmos/account/evmos1fl48vsnmsdzcv85q5d2q4z5ajdha8yu3h6cprl)   | `burner` `staking` |
| `not_bonded_tokens_pool`| [evmos1tygms3xhhs3yv487phx3dw4a95jn7t7lr6ys4t](https://www.mintscan.io/evmos/account/evmos1tygms3xhhs3yv487phx3dw4a95jn7t7lr6ys4t)   | `burner` `staking` |
| `gov`                   | [evmos10d07y265gmmuvt4z0w9aw880jnsr700jcrztvm](https://www.mintscan.io/evmos/account/evmos10d07y265gmmuvt4z0w9aw880jnsr700jcrztvm)   | `burner`           |
| `distribution`          | [evmos1jv65s3grqf6v6jl3dp4t6c9t9rk99cd8974jnh](https://www.mintscan.io/evmos/account/evmos1jv65s3grqf6v6jl3dp4t6c9t9rk99cd8974jnh)   | `none`             |
| `evm`                   | [evmos1vqu8rska6swzdmnhf90zuv0xmelej4lq0n56wq](https://www.mintscan.io/evmos/account/evmos1vqu8rska6swzdmnhf90zuv0xmelej4lq0n56wq)   | `minter` `burner`  |
| `ibc`                   | [evmos1a53udazy8ayufvy0s434pfwjcedzqv345dnt3x](https://www.mintscan.io/evmos/account/evmos1a53udazy8ayufvy0s434pfwjcedzqv345dnt3x)   | `minter` `burner`  |

## Account Permissions

* The `burner` permission means this account has the permission to burn or destroy tokens.
* The `minter` permission means this account has permission to mint or create new tokens.
* The `staking` permission means this account has permission to stake tokens on behalf of its owner.

## IBC Module Accounts

Additionally, there are module accounts associated with IBC transfers.
For each IBC connection, there's an account of type `ModuleAccount` used to escrow the transferred coins when Evmos is the source chain.
Their addresses are derived using the first 20 bytes of the SHA256 checksum of the account name and following the format as outlined in [ADR 028](https://github.com/cosmos/cosmos-sdk/blob/master/docs/architecture/adr-028-public-key-addresses.md):

```go
// accountName is composed by the current version the IBC transfer module supports (in this case, ics20-1), the portID (transfer) and the channelID
accountName := Version + "\0" + portID + "/" + channelID
addr := sha256.Sum256(accountName)[:20]

// example for channel-0
addr := sha256.Sum256("ics20-1\0transfer/channel-0")[:20]
```

This can be calculated with the [`GetEscrowAccount` function on IBC-go](https://github.com/cosmos/ibc-go/blob/c56f78905a5d2db01d867381d106c403fa9e5c4b/modules/apps/transfer/types/keys.go#L41-L55).

:::tip
**Note**: These escrow accounts are not listed when performing the query:

```shell
evmosd q auth module-accounts
```

This happens because the [`GetModuleAccount` function](https://github.com/cosmos/cosmos-sdk/blob/74d7a0dfcd9f47d8a507205f82c264a269ef0612/x/auth/keeper/keeper.go#L194-L224) used on the query considers only the accounts on the [`permAddrs` map of the `AccountKeeper`](https://github.com/cosmos/cosmos-sdk/blob/74d7a0dfcd9f47d8a507205f82c264a269ef0612/x/auth/keeper/keeper.go#L54-L68).
This address map is set at compile time and cannot be changed on runtime.
:::
