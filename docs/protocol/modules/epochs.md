# `epochs`

## 摘要

本文档规定了 Evmos Hub 的内部 `x/epochs` 模块。

在使用 [Cosmos SDK](https://github.com/cosmos/cosmos-sdk) 时，我们经常希望定期运行某些代码。

`epochs` 模块的目的是允许其他模块维护它们希望在一段时间内被通知的需求。
因此，另一个模块可以指定它希望在每周的特定时间（UTC 时间 = x）执行某些代码。
`epochs` 创建了一个通用的时代接口，以便其他模块可以更容易地在此类事件发生时被通知。

## 目录

1. **[概念](#概念)**
2. **[状态](#状态)**
3. **[事件](#事件)**
4. **[Keeper](#Keeper)**
5. **[Hooks](#Hooks)**
6. **[查询](#查询)**
7. **[未来改进](#未来改进)**

## 概念

`epochs` 模块定义了在固定时间间隔内执行的链上定时器。
其他 Evmos 模块可以注册逻辑以在定时器滴答时执行。
我们将两个定时器滴答之间的时间段称为“时代”。

每个定时器都有一个唯一的标识符，每个时代都有一个开始时间和结束时间，
其中 `结束时间 = 开始时间 + 定时器间隔`。

## 状态

### 状态对象

`x/epochs` 模块在状态中保留以下对象：

| 状态对象     | 描述               | 键                   | 值                  | 存储  |
|--------------|--------------------|----------------------|---------------------|-------|
| `EpochInfo`  | 时代信息字节码     | `[]byte{identifier}`  | `[]byte{epochInfo}` | KV    |

#### EpochInfo

`EpochInfo` 定义了几个变量：

1. `identifier` 保留了一个时代标识字符串
2. `start_time` 保留了时代计数的开始时间：
   如果块高度超过 `start_time`，则设置 `epoch_counting_started`
3. `duration` 保留了目标时代持续时间
4. `current_epoch` 保留了当前活动时代的编号
5. `current_epoch_start_time` 保留了当前时代的开始时间
6. `epoch_counting_started` 是一个标志，与 `start_time` 一起设置，在此时 `epoch_number` 将被计数
7. `current_epoch_start_height` 保留了当前时代的开始块高度

```protobuf
message EpochInfo {
    string identifier = 1;
    google.protobuf.Timestamp start_time = 2 [
        (gogoproto.stdtime) = true,
        (gogoproto.nullable) = false,
        (gogoproto.moretags) = "yaml:\"start_time\""
    ];
    google.protobuf.Duration duration = 3 [
        (gogoproto.nullable) = false,
        (gogoproto.stdduration) = true,
        (gogoproto.jsontag) = "duration,omitempty",
        (gogoproto.moretags) = "yaml:\"duration\""
    ];
    int64 current_epoch = 4;
    google.protobuf.Timestamp current_epoch_start_time = 5 [
        (gogoproto.stdtime) = true,
        (gogoproto.nullable) = false,
        (gogoproto.moretags) = "yaml:\"current_epoch_start_time\""
    ];
    bool epoch_counting_started = 6;
    reserved 7;
    int64 current_epoch_start_height = 8;
}
```

`epochs` 模块将这些 `EpochInfo` 对象保存在状态中，它们在创世时被初始化，并在开始块或结束块时进行修改。

#### 创世状态

`x/epochs` 模块的 `GenesisState` 定义了初始化链所需的状态，该状态是从先前导出的高度开始的。
它包含一个包含所有保存在状态中的 `EpochInfo` 对象的切片：

```go
// Genesis State defines the epoch module's genesis state
type GenesisState struct {
    // list of EpochInfo structs corresponding to all epochs
	Epochs []EpochInfo `protobuf:"bytes,1,rep,name=epochs,proto3" json:"epochs"`
}
```

## 事件

`x/epochs` 模块会发出以下事件：

### BeginBlocker

| 类型          | 属性键            | 属性值            |
| ------------- | ----------------- | ----------------- |
| `epoch_start` | `"epoch_number"`  | `{epoch_number}`  |
| `epoch_start` | `"start_time"`    | `{start_time}`    |

### EndBlocker

| 类型           | 属性键           | 属性值            |
| ------------- | ----------------- | ----------------- |
| `epoch_end`   | `"epoch_number"`  | `{epoch_number}`  |

## Keepers

`x/epochs` 模块只公开了一个 keeper，即 epochs keeper，可用于管理 epochs。

### Epochs Keeper

目前只公开了一个完全授权的 epochs keeper，
它具有读取和写入所有 epochs 的 `EpochInfo` 的能力，
并且可以迭代所有存储的 epochs。

```go
// Keeper of epoch nodule maintains collections of epochs and hooks.
type Keeper struct {
	cdc      codec.Codec
	storeKey storetypes.StoreKey
	hooks    types.EpochHooks
}
```

```go
// Keeper is the interface for epoch module keeper
type Keeper interface {
  // GetEpochInfo returns epoch info by identifier
  GetEpochInfo(ctx sdk.Context, identifier string) types.EpochInfo

  // SetEpochInfo set epoch info
  SetEpochInfo(ctx sdk.Context, epoch types.EpochInfo)

  // DeleteEpochInfo delete epoch info
  DeleteEpochInfo(ctx sdk.Context, identifier string)

  // IterateEpochInfo iterate through epochs
  IterateEpochInfo(ctx sdk.Context, fn func(index int64, epochInfo types.EpochInfo) (stop bool))

  // Get all epoch infos
  AllEpochInfos(ctx sdk.Context) []types.EpochInfo
}
```

## 钩子

`x/epochs` 模块实现了钩子，以便其他模块可以使用 epochs
以特定的计划运行 [Cosmos SDK](https://github.com/cosmos/cosmos-sdk) 的各个方面。

### 钩子实现

```go
// combine multiple epoch hooks, all hook functions are run in array sequence
type MultiEpochHooks []types.EpochHooks

// AfterEpochEnd is called when epoch is going to be ended, epochNumber is the
// number of epoch that is ending
func (mh MultiEpochHooks) AfterEpochEnd(ctx sdk.Context, epochIdentifier string, epochNumber int64) {...}

// BeforeEpochStart is called when epoch is going to be started, epochNumber is
// the number of epoch that is starting
func (mh MultiEpochHooks) BeforeEpochStart(ctx sdk.Context, epochIdentifier string, epochNumber int64) {...}

// AfterEpochEnd executes the indicated hook after epochs ends
func (k Keeper) AfterEpochEnd(ctx sdk.Context, identifier string, epochNumber int64) {...}

// BeforeEpochStart executes the indicated hook before the epochs
func (k Keeper) BeforeEpochStart(ctx sdk.Context, identifier string, epochNumber int64) {...}
```

### 接收钩子

当其他模块（`x/epochs` 之外的模块）接收到钩子时，
它们需要过滤值 `epochIdentifier`，并仅对特定的 `epochIdentifier` 执行操作。

从 `epochIdentifier` 中过滤出的值可以存储在其他模块的 `Params` 中，
因此它们可以由治理进行修改。

治理可以根据需要将 epoch 周期从 `week` 更改为 `day`。

## 查询

`x/epochs` 模块提供以下查询来检查模块的状态。

```protobuf
service Query {
  // EpochInfos provide running epochInfos
  rpc EpochInfos(QueryEpochsInfoRequest) returns (QueryEpochsInfoResponse) {}
  // CurrentEpoch provide current epoch of specified identifier
  rpc CurrentEpoch(QueryCurrentEpochRequest) returns (QueryCurrentEpochResponse) {}
}
```

## 未来改进

### 正确使用

在当前设计中，每个时代至少应包含两个区块，因为起始区块应与结束区块不同。
因此，每个时代分配的时间将为 `max(block_time x 2, epoch_duration)`。
例如：如果将 `epoch_duration` 设置为 `1s`，`block_time` 设置为 `5s`，则实际时代时间应为 `10s`。

建议将 `epoch_duration` 配置为超过 `block_time` 两倍以上，以正确使用此模块。
如果 `epoch_duration` 与实际时代时间不匹配，如上述示例中，
则模块逻辑可能会变得无效。

### 区块时间漂移

`x/epochs` 模块的实现基于 `block_time` 值具有区块时间漂移。
例如：如果我们有一个以 `t=100` 结束的时代，时代长度为 100 个单位，
并且在 `t=97`、`t=104` 和 `t=110` 有一个区块，那么该时代将在 `t=104` 结束，
新的时代将从 `t=110` 开始。

这里存在时间漂移，大约相差 1-2 个区块时间，这将减慢时代的进程。


# `epochs`

## Abstract

This document specifies the internal `x/epochs` module of the Evmos Hub.

Often, when working with the [Cosmos SDK](https://github.com/cosmos/cosmos-sdk),
we would like to run certain pieces of code every so often.

The purpose of the `epochs` module is to allow other modules to maintain
that they would like to be signaled once in a time period.
So, another module can specify it wants to execute certain code once a week, starting at UTC-time = x.
`epochs` creates a generalized epoch interface to other modules so they can be more easily signaled upon such events.

## Contents

1. **[Concept](#concepts)**
2. **[State](#state)**
3. **[Events](#events)**
4. **[Keeper](#keepers)**
5. **[Hooks](#hooks)**
6. **[Queries](#queries)**
7. **[Future improvements](#future-improvements)**

## Concepts

The `epochs` module defines on-chain timers that execute at fixed time intervals.
Other Evmos modules can then register logic to be executed at the timer ticks.
We refer to the period in between two timer ticks as an "epoch".

Every timer has a unique identifier, and every epoch will have a start time and an end time,
where `end time = start time + timer interval`.


## State

### State Objects

The `x/epochs` module keeps the following `objects in state`:

| State Object | Description         | Key                  | Value               | Store |
|--------------|---------------------|----------------------|---------------------|-------|
| `EpochInfo`  | Epoch info bytecode | `[]byte{identifier}` | `[]byte{epochInfo}` | KV    |

#### EpochInfo

An `EpochInfo` defines several variables:

1. `identifier` keeps an epoch identification string
2. `start_time` keeps the start time for epoch counting:
   if block height passes `start_time`, then `epoch_counting_started` is set
3. `duration` keeps the target epoch duration
4. `current_epoch` keeps the current active epoch number
5. `current_epoch_start_time` keeps the start time of the current epoch
6. `epoch_counting_started` is a flag set with `start_time`, at which point `epoch_number` will be counted
7. `current_epoch_start_height` keeps the start block height of the current epoch

```protobuf
message EpochInfo {
    string identifier = 1;
    google.protobuf.Timestamp start_time = 2 [
        (gogoproto.stdtime) = true,
        (gogoproto.nullable) = false,
        (gogoproto.moretags) = "yaml:\"start_time\""
    ];
    google.protobuf.Duration duration = 3 [
        (gogoproto.nullable) = false,
        (gogoproto.stdduration) = true,
        (gogoproto.jsontag) = "duration,omitempty",
        (gogoproto.moretags) = "yaml:\"duration\""
    ];
    int64 current_epoch = 4;
    google.protobuf.Timestamp current_epoch_start_time = 5 [
        (gogoproto.stdtime) = true,
        (gogoproto.nullable) = false,
        (gogoproto.moretags) = "yaml:\"current_epoch_start_time\""
    ];
    bool epoch_counting_started = 6;
    reserved 7;
    int64 current_epoch_start_height = 8;
}
```

The `epochs` module keeps these `EpochInfo` objects in state, which are initialized at genesis
and are modified on begin blockers or end blockers.

#### Genesis State

The `x/epochs` module's `GenesisState` defines the state necessary for initializing the chain
from a previously exported height.
It contains a slice containing all the `EpochInfo` objects kept in state:

```go
// Genesis State defines the epoch module's genesis state
type GenesisState struct {
    // list of EpochInfo structs corresponding to all epochs
	Epochs []EpochInfo `protobuf:"bytes,1,rep,name=epochs,proto3" json:"epochs"`
}
```

## Events

The `x/epochs` module emits the following events:

### BeginBlocker

| Type          | Attribute Key     | Attribute Value   |
| ------------- | ----------------- | ----------------- |
| `epoch_start` | `"epoch_number"`  | `{epoch_number}`  |
| `epoch_start` | `"start_time"`    | `{start_time}`    |

### EndBlocker

| Type           | Attribute Key    | Attribute Value   |
| ------------- | ----------------- | ----------------- |
| `epoch_end`   | `"epoch_number"`  | `{epoch_number}`  |

## Keepers

The `x/epochs` module only exposes one keeper, the epochs keeper, which can be used to manage epochs.

### Epochs Keeper

Presently only one fully-permissioned epochs keeper is exposed,
which has the ability to both read and write the `EpochInfo` for all epochs,
and to iterate over all stored epochs.

```go
// Keeper of epoch nodule maintains collections of epochs and hooks.
type Keeper struct {
	cdc      codec.Codec
	storeKey storetypes.StoreKey
	hooks    types.EpochHooks
}
```

```go
// Keeper is the interface for epoch module keeper
type Keeper interface {
  // GetEpochInfo returns epoch info by identifier
  GetEpochInfo(ctx sdk.Context, identifier string) types.EpochInfo

  // SetEpochInfo set epoch info
  SetEpochInfo(ctx sdk.Context, epoch types.EpochInfo)

  // DeleteEpochInfo delete epoch info
  DeleteEpochInfo(ctx sdk.Context, identifier string)

  // IterateEpochInfo iterate through epochs
  IterateEpochInfo(ctx sdk.Context, fn func(index int64, epochInfo types.EpochInfo) (stop bool))

  // Get all epoch infos
  AllEpochInfos(ctx sdk.Context) []types.EpochInfo
}
```

## Hooks

The `x/epochs` module implements hooks so that other modules can use epochs
to allow facets of the [Cosmos SDK](https://github.com/cosmos/cosmos-sdk) to run on specific schedules.

### Hooks Implementation

```go
// combine multiple epoch hooks, all hook functions are run in array sequence
type MultiEpochHooks []types.EpochHooks

// AfterEpochEnd is called when epoch is going to be ended, epochNumber is the
// number of epoch that is ending
func (mh MultiEpochHooks) AfterEpochEnd(ctx sdk.Context, epochIdentifier string, epochNumber int64) {...}

// BeforeEpochStart is called when epoch is going to be started, epochNumber is
// the number of epoch that is starting
func (mh MultiEpochHooks) BeforeEpochStart(ctx sdk.Context, epochIdentifier string, epochNumber int64) {...}

// AfterEpochEnd executes the indicated hook after epochs ends
func (k Keeper) AfterEpochEnd(ctx sdk.Context, identifier string, epochNumber int64) {...}

// BeforeEpochStart executes the indicated hook before the epochs
func (k Keeper) BeforeEpochStart(ctx sdk.Context, identifier string, epochNumber int64) {...}
```

### Recieving Hooks

When other modules (outside of `x/epochs`) recieve hooks,
they need to filter the value `epochIdentifier`, and only do executions for a specific `epochIdentifier`.

The filtered values from `epochIdentifier` could be stored in the `Params` of other modules,
so they can be modified by governance.

Governance can change epoch periods from `week` to `day` as needed.


## Queries

The `x/epochs` module provides the following queries to check the module's state.

```protobuf
service Query {
  // EpochInfos provide running epochInfos
  rpc EpochInfos(QueryEpochsInfoRequest) returns (QueryEpochsInfoResponse) {}
  // CurrentEpoch provide current epoch of specified identifier
  rpc CurrentEpoch(QueryCurrentEpochRequest) returns (QueryCurrentEpochResponse) {}
}
```


## Future Improvements

### Correct Usage

In the current design, each epoch should be at least two blocks, as the start block should be different from the endblock.
Because of this, the time allocated to each epoch will be `max(block_time x 2, epoch_duration)`.
For example: if the `epoch_duration` is set to `1s`, and `block_time` is `5s`, actual epoch time should be `10s`.

It is recommended to configure `epoch_duration` to be more than two times the `block_time`, to use this module correctly.
If there is a mismatch between the `epoch_duration` and the actual epoch time, as in the example above,
then module logic could become invalid.

### Block-Time Drifts

This implementation of the `x/epochs` module has block-time drifts based on the value of `block_time`.
For example: if we have an epoch of 100 units that ends at `t=100`,
and we have a block at `t=97` and a block at `t=104` and `t=110`, this epoch ends at `t=104`,
and the new epoch will start at `t=110`.

There are time drifts here, varying about 1-2 blocks time, which will slow down epochs.
