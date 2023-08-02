# 轻客户端验证

轻客户端通过与全节点通信，实现对区块链中的[头部][TMBC-HEADER-link]的读取操作。由于一些全节点可能存在故障，因此必须以容错的方式实现此功能。

在Tendermint区块链中，验证者集合可能会随着每个新区块的产生而发生变化。质押和解绑机制引入了一个[安全模型][TMBC-FM-2THIRDS-link]：从[头部][TMBC-HEADER-link]的时间*Time*开始，对于*TrustedPeriod*的持续时间内，下一个新区块的验证者中超过三分之二是正确的。容错的读取操作是针对这个安全模型设计的。

本文所解决的挑战是，轻客户端可能具有高度为*h1*的区块，并且需要读取高度大于*h1*的区块高度为*h2*的区块。检查从*h1*到*h2*的所有区块头可能会过于昂贵（例如，对于移动设备而言，会消耗过多的能量）。本规范试图通过利用[安全模型][TMBC-FM-2THIRDS-link]提供的保证，减少需要检查的中间区块的数量。

# 状态

本文档经过了彻底的审查，并且协议已经在TLA+中进行了形式化和模型检查。

## 需要解决的问题

由于它是较大轻节点的一部分，其数据结构和函数与轻客户端的分叉检测功能进行交互。由于在[Pull Request 479](https://github.com/informalsystems/tendermint-rs/pull/479)的工作结果，我们确定了在[Issue 499](https://github.com/informalsystems/tendermint-rs/issues/499)中需要对数据结构进行更新。这不会改变验证逻辑，但会记录有关验证的信息，这些信息可以在分叉检测中使用（特别是在更高效地计算分叉证明时）。

# 大纲

- [第一部分](#part-i---tendermint-blockchain)：介绍Tendermint区块链的相关术语。

- [第二部分](#part-ii---sequential-definition-of-the-verification-problem)：介绍轻客户端验证协议所解决的问题。
    - [验证非正式问题陈述](#Verification-Informal-Problem-statement)：面向一般受众，即希望从宏观角度了解组件工作的工程师。
    - [顺序问题陈述](#Sequential-Problem-statement)：以其顺序形式提供问题陈述的数学定义，即忽略区块链实现的分布式方面。

- [第三部分](#part-iii---light-client-as-distributed-system): 轻客户端的分布式方面，系统假设和时间逻辑规范。

    - [激励机制](#incentives): 错误的全节点如何从恶意行为中获益，正确的全节点如何从合作中获益。
  
    - [计算模型](#Computational-Model):
      时间和正确性假设。

    - [分布式问题陈述](#Distributed-Problem-Statement):
      在分布式环境中形式化安全性和活性属性的时间属性。

- [第四部分](#part-iv---light-client-verification-protocol):
  协议的规范。

    - [定义](#Definitions): 描述协议使用的输入、输出、变量和辅助函数。

    - [核心验证](#core-verification): 概述解决方案，并详细说明使用的函数（包括前提条件、后置条件、错误条件）。

    - [活性场景](#liveness-scenarios): 轻客户端的进展取决于区块链验证者集合的变化。我们讨论一些典型的场景。

- [第五部分](#part-v---supporting-the-ibc-relayer): 上述部分关注的是最后一个验证的区块高度为 *h1*，而请求的高度 *h2* 满足 *h2 > h1* 的常见情况。对于 IBC，存在一些不满足这种情况的场景。在本部分中，我们提供了一些支持的初步工作。由于目前还不清楚 IBC 要求的所有细节，我们暂时不提供完整的规范。我们用"Open Question"标记需要解决的问题，以便最终完成这个规范。需要注意的是，在第四部分中指定的情况是技术上最具挑战性的情况。

在本文档中，我们广泛使用标签，以便能够在未来的交流中引用假设、不变量等。在这些标签中，我们经常使用以下简写形式：

- TMBC: Tendermint 区块链
- SEQ: 顺序规范
- LCV: 轻客户端验证
- LIVE: 活性
- SAFE: 安全性
- FUNC: 函数
- INV: 不变量
- A: 假设

# 第一部分 - Tendermint区块链

## 轻客户端所需的标题字段

#### **[TMBC-HEADER.1]**

一组区块链交易存储在一个称为*块*的数据结构中，其中包含一个称为*标题*的字段。（数据结构*块*的定义在[这里][block]）。由于标题包含到块的相关字段的哈希值，因此在本规范中，我们将假设区块链是标题的列表，而不是块的列表。

#### **[TMBC-HASH-UNIQUENESS.1]**

我们假设标题中的每个哈希标识其哈希的数据。因此，在本规范中，我们不区分哈希和它们所代表的数据。

#### **[TMBC-HEADER-FIELDS.1]**

标题包含以下字段：

- `Height`：非负整数
- `Time`：时间（整数）
- `LastBlockID`：哈希值
- `LastCommit`：DomainCommit
- `Validators`：DomainVal
- `NextValidators`：DomainVal
- `Data`：DomainTX
- `AppState`：DomainApp
- `LastResults`：DomainRes

#### **[TMBC-SEQ.1]**

Tendermint区块链是标题的列表*chain*。

#### **[TMBC-VALIDATOR-PAIR.1]**

给定一个完整节点，*验证器对*是一个对*(peerID, voting_power)*，其中

- *peerID*是完整节点的PeerID（公钥），
- *voting_power*是一个整数（表示完整节点在某个共识实例中的投票权重）。

> 在Golang实现中，*验证器对*的数据类型称为`Validator`

#### **[TMBC-VALIDATOR-SET.1]**

*验证器集*是一组验证器对。对于验证器集*vs*，我们用*TotalVotingPower(vs)*表示其验证器对的投票权重之和。

#### **[TMBC-VOTE.1]**

*投票*包含在[共识][arXiv]执行期间由验证器节点发送和签名的`prevote`或`precommit`消息。每个消息包含以下字段

- `Type`：prevote或precommit
- `Height`：正整数
- `Round`：正整数
- `BlockID`：一个块的哈希值（不一定是链的块）

#### **[TMBC-COMMIT.1]**

提交是一组`precommit`消息。

## Tendermint故障模型

#### **[TMBC-AUTH-BYZ.1]**

我们假设在认证的拜占庭容错模型中，没有节点（无论是故障的还是正确的）可以破坏数字签名，但除此之外，对于故障节点的内部行为没有做出任何额外的假设。也就是说，故障节点只受到一个限制，即它们不能伪造消息。

#### **[TMBC-TIME-PARAMS.1]**

Tendermint区块链具有以下配置参数：

- *unbondingPeriod*：一个时间段。
- *trustingPeriod*：一个小于*unbondingPeriod*的时间段。

#### **[TMBC-CORRECT.1]**

我们定义一个谓词*correctUntil(n, t)*，其中*n*是一个节点，*t*是一个时间点。
当且仅当节点*n*遵循所有协议（至少）直到时间*t*时，谓词*correctUntil(n, t)*为真。

#### **[TMBC-FM-2THIRDS.1]**

如果一个块*h*在链中，
那么存在*h.NextValidators*的一个子集*CorrV*，满足以下条件：

- *TotalVotingPower(CorrV) > 2/3
    TotalVotingPower(h.NextValidators)*；参见[TMBC-VALIDATOR-SET.1]
- 对于*CorrV*中的每对验证者*(n,p)*，满足*correctUntil(n,
    h.Time + trustingPeriod)*；参见[TMBC-CORRECT.1]

> 正确的定义
> [**[TMBC-CORRECT.1]**][TMBC-CORRECT-link]是指实时性，而在这里使用的是*Time*和*trustingPeriod*，它们是“硬件时间”。我们在这里不做区分。

#### **[TMBC-CORR-FULL.1]**

每个正确的全节点本地存储了来自[**[TMBC-SEQ.1]**][TMBC-SEQ-link]的当前头部列表的前缀。

## 轻客户端检查的内容

> 从[TMBC-FM-2THIRDS.1]我们直接得出以下观察结果：

#### **[TMBC-VAL-CONTAINS-CORR.1]**

给定区块链的（可信任的）块*tb*，给定的全节点集合*N*在实时*t*上包含一个正确的节点，如果

- *t - trustingPeriod < tb.Time < t*
- *N*中节点在*tb.NextValidators*中的投票权超过*TotalVotingPower(tb.NextValidators)*的1/3

> 以下描述了给定块*b*的提交应该是什么样子。

#### **[TMBC-SOUND-DISTR-POSS-COMMIT.1]**

对于一个块 *b*，*PossibleCommit(b)* 的每个元素 *pc* 都满足以下条件：

- *pc* 只包含来自 *b.Validators* 中的验证人的投票（参见[TMBC-VOTE.1]）
- *pc* 中投票权的总和大于 *TotalVotingPower(b.Validators)* 的 2/3
- 存在一个 *r*，使得 *pc* 中的每个投票 *v* 都满足以下条件：
    - v.Type = precommit
    - v.Height = b.Height
    - v.Round = r
    - v.blockID = hash(b)

> 以下属性来自于共识的有效性：如果新的（待决定的）块的 `BlockID` 等于上一个块的哈希值，那么正确的验证人节点只会发送 `prevote` 或 `precommit`。

#### **[TMBC-VAL-COMMIT.1]**

如果对于一个块 *b*，存在一个提交 *c* 满足以下条件：

- 包含至少一个验证人对 *(v,p)*，其中 *v* 是一个**正确的**验证人节点，并且
- *c* 包含在 *PossibleCommit(b)* 中

那么块 *b* 就在区块链上。

## 本文档的背景

在本文档中，我们详细说明了轻客户端验证组件，称为*核心验证*。*核心验证*与完整节点进行通信。由于完整节点可能存在故障，轻客户端不能信任接收到的信息，而是必须检查接收到的头部是否与Tendermint共识生成的头部一致。

[[TMBC-VAL-CONTAINS-CORR.1]][TMBC-VAL-CONTAINS-CORR-link] 和 [[TMBC-VAL-COMMIT]][TMBC-VAL-COMMIT-link] 这两个属性规范化了此规范所执行的检查：
给定一个可信的块 *tb* 和一个带有提交 *cub* 的不可信块 *ub*，必须检查 *cub* 是否在 *PossibleCommit(ub)* 中，并且 *cub* 是否包含使用 *tb* 的正确节点。

# 第二部分 - 验证问题的顺序定义

## 验证问题的非正式陈述

给定一个输入高度 *targetHeight*，*Verifier* 最终会在本地存储一个高度为 *targetHeight* 的头部 *h*。这个头部 *h* 是由Tendermint [区块链][block] 生成的。特别地，不应该存储未由区块链生成的头部。

## 顺序问题的陈述

#### **[LCV-SEQ-LIVE.1]**

*Verifier*接收一个高度为*targetHeight*的输入，并最终存储区块链中高度为*targetHeight*的区块头。

#### **[LCV-SEQ-SAFE.1]**

*Verifier*永远不会存储不在区块链中的区块头。

# 第三部分 - 轻客户端作为分布式系统

## 激励机制

有缺陷的全节点可能通过向轻客户端提供与Tendermint共识生成的区块不一致（例如，包含额外交易）的区块来欺骗轻客户端。
使用轻客户端的用户可能会因接受伪造的区块头而受到伤害。

轻客户端的[分叉检测器][fork-detector]可以帮助正确的全节点判断其区块头是否正确。
因此，结合轻客户端检测器，正确的全节点有动机进行响应。我们可以基于正确的全节点可靠地与轻客户端通信的假设来进行活性论证。

## 计算模型

#### **[LCV-A-PEER.1]**

验证器与称为*primary*的全节点进行通信。对全节点不做任何假设（可能是正确的或有缺陷的）。

#### **[LCV-A-COMM.1]**

轻客户端与正确的全节点之间的通信是可靠且有时间限制的。可靠的通信意味着消息不会丢失、重复，并最终被传递。存在一个（已知的）端到端延迟*Delta*，如果消息在时间*t*发送，则在时间*t + Delta*之前被接收和处理。这意味着我们需要至少*2 Delta*的超时时间来确保正确的对等方的响应在超时之前到达。

#### **[LCV-A-TFM.1]**

Tendermint区块链满足Tendermint故障模型[**[TMBC-FM-2THIRDS.1]**][TMBC-FM-2THIRDS-link]。

#### **[LCV-A-VAL.1]**

系统满足[**[TMBC-AUTH-BYZ.1]**][TMBC-Auth-Byz-link]和[**[TMBC-FM-2THIRDS.1]**][TMBC-FM-2THIRDS-link]。因此，存在一个满足完整性要求（即[[block]]中的验证规则）的区块链。

## 分布式问题陈述

### 两种终止方式

我们不假设*primary*是正确的。在这种假设下，没有协议可以保证顺序属性的组合。因此，在（不可靠的）分布式环境中，我们考虑两种终止方式（成功和失败），并且我们将在下面指定*核心验证*在什么（有利的）条件下保证成功终止，并满足顺序问题陈述的要求：

#### **[LCV-DIST-TERM.1]**

*核心验证*要么*成功终止*，要么*以失败终止*。

### 设计选择

#### **[LCV-DIST-STORE.1]**

*核心验证*有一个名为*LightStore*的本地数据结构，其中包含轻区块（包含一个头部）。对于每个轻区块，我们记录它是否已经验证。

#### **[LCV-DIST-PRIMARY.1]**

*核心验证*有一个名为*primary*的本地变量，其中包含一个完整节点的PeerID。

#### **[LCV-DIST-INIT.1]**

*LightStore*使用由Tendermint共识正确生成的头部*trustedHeader*进行初始化。我们称*trustedHeader*已经验证。

### 时间属性

#### **[LCV-DIST-SAFE.1]**

始终存在这样的情况，即*LightStore*中的每个已验证头部都是由Tendermint共识的一个实例生成的。

#### **[LCV-DIST-LIVE.1]**

不时地，使用一个高度*targetHeight*大于*LightStore*中任何头部的高度调用一个新的*核心验证*实例。每个实例最终必须终止。

- 如果
    - *primary*是正确的（并且本地具有*targetHeight*的块），并且
    - *LightStore*始终包含一个已验证头部，其年龄小于信任期限，
    那么*核心验证*将一个高度为*targetHeight*的已验证头部*hd*添加到*LightStore*中，并且**成功终止**

> 这些定义意味着，如果primary是有故障的，可能会或可能不会将头部添加到*LightStore*中。无论如何，必须满足[**[LCV-DIST-SAFE.1]**](#lcv-vc-inv)。
> 不变式[**[LCV-DIST-SAFE.1]**](#lcv-dist-safe)和活性要求[**[LCV-DIST-LIVE.1]**](#lcv-dist-life)
> 允许将已验证头部添加到*LightStore*中，其高度未传递给验证器（例如，在二分法中使用的中间头部；见下文）。
> 请注意，对于活性，最初在*trustinPeriod*内具有一个*trustedHeader*是不足够的。然而，由于该规范在下载中间头部的顺序方面留有一些自由度，因此我们在这里不给出更精确的活性规范。在给出协议规范之后，我们将讨论一些活性场景[下文](#liveness-scenarios)。

### 解决顺序规范

该规范提供了对顺序规范的部分解决方案。
*验证器*解决了顺序部分的不变式

[**[LCV-DIST-SAFE.1]**](#lcv-vc-inv) => [**[LCV-SEQ-SAFE.1]**](#lcv-seq-inv)

在主节点正确的情况下，并且*LightStore*中存在最近的头部，验证器满足活性要求。

⋀ *主节点正确*  
⋀ 总是 ∃ 在LightStore中验证的头部。*header.Time* > *now* - *trustingPeriod*  
⋀ [**[LCV-A-Comm.1]**](#lcv-a-comm) ⋀ (
       ( [**[TMBC-CorrFull.1]**][TMBC-CorrFull-link] ⋀
         [**[LCV-DIST-LIVE.1]**](#lcv-vc-live) )
       ⟹ [**[LCV-SEQ-LIVE.1]**](#lcv-seq-live)
)

# 第四部分 - 轻客户端验证协议

我们提供了轻客户端验证的规范。本地验证代码由一个顺序函数`VerifyToTarget`表示，以突出显示此功能的控制流程。我们注意到，如果对实现考虑了不同的并发模型，则可以使用互斥锁等实现函数的顺序流程。然而，轻客户端验证被分为三个块，可以独立实现和测试：

- `FetchLightBlock`用于从对等方下载给定高度的轻块（头部）。
- `ValidAndVerified`是一个本地代码，用于检查头部。
- `Schedule`决定下一个要尝试验证的高度。我们将其保留为未指定，因为不同的实现（目前在Goland和Rust中）可能在此处实现不同的优化。我们只提供高度可能如何发展的必要条件。

<!-- > `ValidAndVerified`是在IBC上下文中有时称为"轻客户端"的函数。 -->

## 定义

### 数据类型

该协议的核心数据结构是LightBlock。

#### **[LCV-DATA-LIGHTBLOCK.1]**

```go
type LightBlock struct {
  Header          Header
  Commit          Commit
  Validators      ValidatorSet
}
```

#### **[LCV-DATA-LIGHTSTORE.1]**

LightBlocks存储在一个结构中，该结构存储了从初始化或从对等方接收的所有LightBlock。

```go
type LightStore struct {
 ...
}

```

每个LightBlock处于以下状态之一：

```go
type VerifiedState int

const (
 StateUnverified = iota + 1
 StateVerified
 StateFailed
 StateTrusted
)
```

> 只有检测器模块将LightBlock状态设置为`StateTrusted`，且仅在之前为`StateVerified`时。

LightStore提供以下函数来查询存储的LightBlocks。

#### **[LCV-FUNC-GET.1]**

```go
func (ls LightStore) Get(height Height) (LightBlock, bool)
```

- 预期后置条件
    - 返回给定高度的LightBlock，如果LightStore不包含指定的LightBlock，则返回false作为第二个参数。

#### **[LCV-FUNC-LATEST-VERIF.1]**

```go
func (ls LightStore) LatestVerified() LightBlock
```

- 预期后置条件
    - 返回状态为`StateVerified`或`StateTrusted`的最高LightBlock

#### **[LCV-FUNC-UPDATE.2]**

```go
func (ls LightStore) Update(lightBlock LightBlock, 
                            verfiedState VerifiedState
       verifiedBy Height)
```

- 预期后置条件
    - LightBlock的状态设置为*verifiedState*。
    - Lightblock的verifiedBy设置为*Height*

> 以下函数仅在检测器规范中使用，为了完整性在此列出。

#### **[LCV-FUNC-LATEST-TRUSTED.1]**

```go
func (ls LightStore) LatestTrusted() LightBlock
```

- 预期后置条件
    - 返回已经经过验证和检测器检查的最高LightBlock。

#### **[LCV-FUNC-FILTER.1]**

```go
func (ls LightStore) FilterVerified() LightSTore
```

- 预期后置条件
    - 仅返回状态为verified的LightBlocks。

### 输入

- *lightStore*：存储已下载并通过验证的轻区块。最初，它包含一个具有*trustedHeader*的轻区块。
- *primary*：peerID
- *targetHeight*：所需头的高度

### 配置参数

- *trustThreshold*：浮点数。如果正确性不应基于更多的投票权和1/3，则可以使用。
- *trustingPeriod*：时间持续时间[**[TMBC-TIME_PARAMS.1]**][TMBC-TIME_PARAMS-link]。
- *clockDrift*：时间持续时间。纠正参数，仅处理大致同步的时钟。

### 变量

- *nextHeight*: 最初为*targetHeight*
  > *nextHeight* 应被视为“我们需要下载和验证的下一个头的高度”

### 假设

#### **[LCV-A-INIT.1]**

- *trustedHeader* 来自区块链

- *targetHeight > LightStore.LatestVerified.Header.Height*

### 不变量

#### **[LCV-INV-TP.1]**

*LightStore.LatestTrusted.Header.Time > now - trustingPeriod* 总是成立。

> 如果不变量被违反，轻客户端将没有一个可以信任的头。可信任的头必须从外部获得，其信任只能基于社会共识。

### 使用的远程函数

我们使用由[Tendermint的RPC客户端][RPC]提供的`commit`和`validators`函数。

```go
func Commit(height int64) (SignedHeader, error)
```

- 实现备注
    - RPC到完整节点*n*
    - 发送的JSON：

```javascript
// POST /commit
{
 "jsonrpc": "2.0",
 "id": "ccc84631-dfdb-4adc-b88c-5291ea3c2cfb", // UUID v4, unique per request
 "method": "commit",
 "params": {
  "height": 1234
 }
}
```

- 预期前提条件
    - 区块链上存在高度为`height`的头
- 预期后置条件
    - 如果*n*是正确的：如果通信及时（没有超时），则返回区块链上高度为`height`的签名头
    - 如果*n*是错误的：返回任意内容的签名头
- 错误条件
    - 如果*n*是正确的：前提条件违反或超时
    - 如果*n*是错误的：任意错误

----

```go
func Validators(height int64) (ValidatorSet, error)
```

- 实现备注
    - RPC到完整节点*n*
    - 发送的JSON：

```javascript
// POST /validators
{
 "jsonrpc": "2.0",
 "id": "ccc84631-dfdb-4adc-b88c-5291ea3c2cfb", // UUID v4, unique per request
 "method": "validators",
 "params": {
  "height": 1234
 }
}
```

- 预期前提条件
    - 区块链上存在高度为`height`的头
- 预期后置条件
    - 如果*n*是正确的：如果通信及时（没有超时），则返回区块链上高度为`height`的验证者集合
    - 如果*n*是错误的：返回任意验证者集合
- 错误条件
    - 如果*n*是正确的：前提条件违反或超时
    - 如果*n*是错误的：任意错误

----

### 通信函数

#### **[LCV-FUNC-FETCH.1]**


  ```go
func FetchLightBlock(peer PeerID, height Height) LightBlock
```

- Implementation remark
    - RPC to peer at *PeerID*
    - calls `Commit` for *height* and `Validators` for *height* and *height+1*
- Expected precondition
    - `height` is less than or equal to height of the peer **[LCV-IO-PRE-HEIGHT.1]**
- Expected postcondition:
    - if *node* is correct:
        - Returns the LightBlock *lb* of height `height`
      that is consistent with the blockchain
        - *lb.provider = peer* **[LCV-IO-POST-PROVIDER.1]**
        - *lb.Header* is a header consistent with the blockchain
        - *lb.Validators* is the validator set of the blockchain at height *nextHeight*
        - *lb.NextValidators* is the validator set of the blockchain at height *nextHeight + 1*
    - if *node* is faulty: Returns a LightBlock with arbitrary content
    [**[TMBC-AUTH-BYZ.1]**][TMBC-Auth-Byz-link]
- Error condition
    - if *n* is correct: precondition violated
    - if *n* is faulty: arbitrary error
    - if *lb.provider != peer*
    - times out after 2 Delta (by assumption *n* is faulty)

----

## Core Verification

### Outline

The `VerifyToTarget` is the main function and uses the following functions.

- `FetchLightBlock` is called to download the next light block. It is
  the only function that communicates with other nodes
- `ValidAndVerified` checks whether header is valid and checks if a
  new lightBlock should be trusted
  based on a previously verified lightBlock.
- `Schedule` decides which height to try to verify next

In the following description of `VerifyToTarget` we do not deal with error
handling. If any of the above function returns an error, VerifyToTarget just
passes the error on.

#### **[LCV-FUNC-MAIN.1]**

```go
func VerifyToTarget(primary PeerID, lightStore LightStore,
                    targetHeight Height) (LightStore, Result) {

    nextHeight := targetHeight

    for lightStore.LatestVerified.height < targetHeight {

        // Get next LightBlock for verification
        current, found := lightStore.Get(nextHeight)
        if !found {
            current = FetchLightBlock(primary, nextHeight)
            lightStore.Update(current, StateUnverified)
        }

        // Verify
        verdict = ValidAndVerified(lightStore.LatestVerified, current)

        // Decide whether/how to continue
        if verdict == SUCCESS {
            lightStore.Update(current, StateVerified)
        }
        else if verdict == NOT_ENOUGH_TRUST {
            // do nothing
   // the light block current passed validation, but the validator
            // set is too different to verify it. We keep the state of
   // current at StateUnverified. For a later iteration, Schedule
   // might decide to try verification of that light block again.
        }
        else {
            // verdict is some error code
            lightStore.Update(current, StateFailed)
            // possibly remove all LightBlocks from primary
            return (lightStore, ResultFailure)
        }
        nextHeight = Schedule(lightStore, nextHeight, targetHeight)
    }
    return (lightStore, ResultSuccess)
}
```

- Expected precondition
    - *lightStore* contains a LightBlock within the *trustingPeriod*  **[LCV-PRE-TP.1]**
    - *targetHeight* is greater than the height of all the LightBlocks in *lightStore*
- Expected postcondition:
    - returns *lightStore* that contains a LightBlock that corresponds to a block
     of the blockchain of height *targetHeight*
     (that is, the LightBlock has been added to *lightStore*) **[LCV-POST-LS.1]**
- Error conditions
    - if the precondition is violated
    - if `ValidAndVerified` or `FetchLightBlock` report an error
    - if [**[LCV-INV-TP.1]**](#LCV-INV-TP.1) is violated
  
### Details of the Functions

#### **[LCV-FUNC-VALID.1]**

```go
func ValidAndVerified(trusted LightBlock, untrusted LightBlock) Result
```

- Expected precondition:
    - *untrusted* is valid, that is, satisfies the soundness [checks][block]
    - *untrusted* is **well-formed**, that is,
        - *untrusted.Header.Time < now + clockDrift*
        - *untrusted.Validators = hash(untrusted.Header.Validators)*
        - *untrusted.NextValidators = hash(untrusted.Header.NextValidators)*
    - *trusted.Header.Time > now - trustingPeriod*
    - *trusted.Commit* is a commit for the header
     *trusted.Header*, i.e., it contains
     the correct hash of the header, and +2/3 of signatures
    - the `Height` and `Time` of `trusted` are smaller than the Height and
  `Time` of `untrusted`, respectively
    - the *untrusted.Header* is well-formed (passes the tests from
     [[block]]), and in particular
        - if the untrusted header `unstrusted.Header` is the immediate
   successor  of  `trusted.Header`, then it holds that
            - *trusted.Header.NextValidators =
    untrusted.Header.Validators*, and
    moreover,
            - *untrusted.Header.Commit*
                - contains signatures by more than two-thirds of the validators
                - contains no signature from nodes that are not in *trusted.Header.NextValidators*
- Expected postcondition:
    - Returns `SUCCESS`:
        - if *untrusted* is the immediate successor of *trusted*, or otherwise,
        - if the signatures of a set of validators that have more than
             *max(1/3,trustThreshold)* of voting power in
             *trusted.Nextvalidators* is contained in
             *untrusted.Commit* (that is, header passes the tests
             [**[TMBC-VAL-CONTAINS-CORR.1]**][TMBC-VAL-CONTAINS-CORR-link]
             and [**[TMBC-VAL-COMMIT.1]**][TMBC-VAL-COMMIT-link])
    - Returns `NOT_ENOUGH_TRUST` if:
        - *untrusted* is *not* the immediate successor of
           *trusted*
     and the  *max(1/3,trustThreshold)* threshold is not reached
           (that is, if
      [**[TMBC-VAL-CONTAINS-CORR.1]**][TMBC-VAL-CONTAINS-CORR-link]
      fails and header is does not violate the soundness
         checks [[block]]).
- Error condition:
    - if precondition violated

----

#### **[LCV-FUNC-SCHEDULE.1]**

```go
func Schedule(lightStore, nextHeight, targetHeight) Height
```

- Implementation remark: If picks the next height to be verified.
  We keep the precise choice of the next header under-specified. It is
  subject to performance optimizations that do not influence the correctness
- Expected postcondition: **[LCV-SCHEDULE-POST.1]**
   Return *H* s.t.
   1. if *lightStore.LatestVerified.Height = nextHeight* and
      *lightStore.LatestVerified < targetHeight* then  
   *nextHeight < H <= targetHeight*
   2. if *lightStore.LatestVerified.Height < nextHeight* and
      *lightStore.LatestVerified.Height < targetHeight* then  
   *lightStore.LatestVerified.Height < H < nextHeight*
   3. if *lightStore.LatestVerified.Height = targetHeight* then  
     *H =  targetHeight*

> Case i. captures the case where the light block at height *nextHeight*
> has been verified, and we can choose a height closer to the *targetHeight*.
> As we get the *lightStore* as parameter, the choice of the next height can
> depend on the *lightStore*, e.g., we can pick a height for which we have
> already downloaded a light block.
> In Case ii. the header of *nextHeight* could not be verified, and we need to pick a smaller height.
> In Case iii. is a special case when we have verified the *targetHeight*.

### Solving the distributed specification

*trustedStore* is implemented by the light blocks in lightStore that
have the state *StateVerified*.

#### Argument for [**[LCV-DIST-SAFE.1]**](#lcv-dist-safe)

- `ValidAndVerified` implements the soundness checks and the checks
  [**[TMBC-VAL-CONTAINS-CORR.1]**][TMBC-VAL-CONTAINS-CORR-link] and
  [**[TMBC-VAL-COMMIT.1]**][TMBC-VAL-COMMIT-link] under
  the assumption [**[TMBC-FM-2THIRDS.1]**][TMBC-FM-2THIRDS-link]
- Only if `ValidAndVerified` returns with `SUCCESS`, the state of a light block is
  set to *StateVerified*.

#### Argument for [**[LCV-DIST-LIVE.1]**](#lcv-dist-life)

- If *primary* is correct,
    - `FetchLightBlock` will always return a light block consistent
      with the blockchain
    - `ValidAndVerified` either verifies the header using the trusting
      period or falls back to sequential
      verification
    - If [**[LCV-INV-TP.1]**](#LCV-INV-TP.1) holds, eventually every
   header will be verified and core verification **terminates successfully**.
    - successful termination depends on the age of *lightStore.LatestVerified*
      (for instance, initially on the age of  *trustedHeader*) and the
      changes of the validator sets on the blockchain.
   We will give some examples [below](#liveness-scenarios).
- If *primary* is faulty,
    - it either provides headers that pass all the tests, and we
      return with the header
    - it provides one header that fails a test, core verification
      **terminates with failure**.
    - it times out and core verification
      **terminates with failure**.

## Liveness Scenarios

The liveness argument above assumes [**[LCV-INV-TP.1]**](#LCV-INV-TP.1)

which requires that there is a header that does not expire before the
target height is reached. Here we discuss scenarios to ensure this.

Let *startHeader* be *LightStore.LatestVerified* when core
verification is called (*trustedHeader*) and *startTime* be the time
core verification is invoked.

In order to ensure liveness, *LightStore* always needs to contain a
verified (or initially trusted) header whose time is within the
trusting period. To ensure this, core verification needs to add new
headers to *LightStore* and verify them, before all headers in
*LightStore* expire.

#### Many changes in validator set

 Let's consider `Schedule` implements
 bisection, that is, it halves the distance.
 Assume the case where the validator set changes completely in each
block. Then the
 method in this specification needs to
sequentially verify all headers. That is, for

- *W = log_2 (targetHeight - startHeader.Height)*,

*W* headers need to be downloaded and checked before the
header of height *startHeader.Height + 1* is added to *LightStore*.

- Let *Comp*
  be the local computation time needed to check headers and signatures
  for one header.
- Then we need in the worst case *Comp + 2 Delta* to download and
  check one header.
- Then the first time a verified header could be added to *LightStore* is
  startTime + W * (Comp + 2 Delta)
- [TP.1] However, it can only be added if we still have a header in
  *LightStore*,
  which is not
  expired, that is only the case if
    - startHeader.Time > startTime + WCG * (Comp + 2 Delta) -
      trustingPeriod,
    - that is, if core verification is started at  
   startTime < startHeader.Time + trustingPeriod -  WCG * (Comp + 2 Delta)

- one may then do an inductive argument from this point on, depending
  on the implementation of `Schedule`. We may have to account for the
  headers that are already
  downloaded, but they are checked against the new *LightStore.LatestVerified*.

> We observe that
> the worst case time it needs to verify the header of height
> *targetHeight* depends mainly on how frequent the validator set on the
> blockchain changes. That core verification terminates successfully
> crucially depends on the check [TP.1], that is, that the headers in
> *LightStore* do not expire in the time needed to download more
> headers, which depends on the creation time of the headers in
> *LightStore*. That is, termination of core verification is highly
> depending on the data stored in the blockchain.
> The current light client core verification protocol exploits that, in
> practice, changes in the validator set are rare. For instance,
> consider the following scenario.

#### No change in validator set

If on the blockchain the validator set of the block at height
*targetHeight* is equal to *startHeader.NextValidators*:

- there is one round trip in `FetchLightBlock` to download the light
 block
 of height
  *targetHeight*, and *Comp* to check it.
- as the validator sets are equal, `Verify` returns `SUCCESS`, if
  *startHeader.Time > now - trustingPeriod*.
- that is, if *startTime < startHeader.Header.Time + trustingPeriod -
  2 Delta - Comp*, then core verification terminates successfully

# Part V - Supporting the IBC Relayer

The above specification focuses on the most common case, which also
constitutes the most challenging task: using the Tendermint [security
model][TMBC-FM-2THIRDS-link] to verify light blocks without
downloading all intermediate blocks. To focus on this challenge, above
we have restricted ourselves to the case where  *targetHeight* is
greater than the height of any trusted header. This simplified
presentation of the algorithm as initially
`lightStore.LatestVerified()` is less than *targetHeight*, and in the
process of verification `lightStore.LatestVerified()` increases until
*targetHeight* is reached.

For [IBC][ibc-rs] it might be that some "older" header is
needed, that is,  *targetHeight < lightStore.LatestVerified()*. In this section we present a preliminary design, and we mark some
remaining open questions.
If  *targetHeight < lightStore.LatestVerified()* our design separates
the following cases:

- A previous instance of `VerifyToTarget` has already downloaded the
  light block of *targetHeight*. There are two cases
    - the light block has been verified
    - the light block has not been verified yet
- No light block of *targetHeight* had been downloaded before. There
  are two cases:
    - there exists a verified light block of height less than  *targetHeight*
    - otherwise. In this case we need to do "backwards verification"
     using the hash of the previous block in the `LastBlockID` field
     of a header.
  
**Open Question:** what are the security assumptions for backward
verification. Should we check that the light block we verify from
(and/or the checked light block) is within the trusting period?

The design just presents the above case
distinction as a function, and defines some auxiliary functions in the
same way the protocol was presented in
[Part IV](#part-iv---light-client-verification-protocol).

```go
func (ls LightStore) LatestPrevious(height Height) (LightBlock, bool)
```

- Expected postcondition
    - returns a light block *lb* that satisfies:
        - *lb* is in lightStore
        - *lb* is verified and not expired
        - *lb.Header.Height < height*
        - for all *b* in lightStore s.t. *b* is verified and not expired it
          holds *lb.Header.Height >= b.Header.Height*
    - *false* in the second argument if
      the LightStore does not contain such an *lb*.

```go
func (ls LightStore) MinVerified() (LightBlock, bool)
```

- Expected postcondition
    - returns a light block *lb* that satisfies:
        - *lb* is in lightStore
        - *lb* is verified **Open Question:** replace by trusted?
        - *lb.Header.Height* is minimal in the lightStore
        - **Open Question:** according to this, it might be expired (outside the
          trusting period). This approach appears safe. Are there reasons we
          should not do that?
    - *false* in the second argument if
      the LightStore does not contain such an *lb*.

If a height that is smaller than the smallest height in the lightstore
is required, we check the hashes backwards. This is done with the
following function:

#### **[LCV-FUNC-BACKWARDS.1]**

```go
func Backwards (primary PeerID, lightStore LightStore, targetHeight Height)
               (LightStore, Result) {
  
    lb,res = lightStore.MinVerified()
    if res = false {
        return (lightStore, ResultFailure)
    }

    latest := lb.Header
    for i := lb.Header.height - 1; i >= targetHeight; i-- {
        // here we download height-by-height. We might first download all
        // headers down to targetHeight and then check them.
        current := FetchLightBlock(primary,i)
        if (hash(current) != latest.Header.LastBlockId) {
            return (lightStore, ResultFailure)
        }
        else {
            lightStore.Update(current, StateVerified)
            // **Open Question:** Do we need a new state type for
            // backwards verified light blocks?
        }
        latest = current
    }
    return (lightStore, ResultSuccess)
}
```

The following function just decided based on the required height which
method should be used.

#### **[LCV-FUNC-IBCMAIN.1]**

```go
func Main (primary PeerID, lightStore LightStore, targetHeight Height)
          (LightStore, Result) {

    b1, r1 = lightStore.Get(targetHeight)
    if r1 = true and b1.State = StateVerified {
        // block already there
        return (lightStore, ResultSuccess)
    }

    if targetHeight > lightStore.LatestVerified.height {
     // case of Part IV
        return VerifyToTarget(primary, lightStore, targetHeight)
    }
    else {
        b2, r2 = lightStore.LatestPrevious(targetHeight);
        if r2 = true {
            // make auxiliary lightStore auxLS to call VerifyToTarget.
   // VerifyToTarget uses LatestVerified of the given lightStore
            // For that we need:
            // auxLS.LatestVerified = lightStore.LatestPrevious(targetHeight)
            auxLS.Init;
            auxLS.Update(b2,StateVerified);
            if r1 = true {
                // we need to verify a previously downloaded light block.
                // we add it to the auxiliary store so that VerifyToTarget
                // does not download it again
                auxLS.Update(b1,b1.State);
            }
            auxLS, res2 = VerifyToTarget(primary, auxLS, targetHeight)
            // move all lightblocks from auxLS to lightStore,
            // maintain state
   // we do that whether VerifyToTarget was successful or not
            for i, s range auxLS {
                lighStore.Update(s,s.State)
            }
            return (lightStore, res2)
        }
        else {
            return Backwards(primary, lightStore, targetHeight)
        }
    }
}
```
<!-- - Expected postcondition: -->
<!--   - if targetHeight > lightStore.LatestVerified.height then -->
<!--     return VerifyToTarget(primary, lightStore, targetHeight) -->
<!--   - if targetHeight = lightStore.LatestVerified.height then -->
<!--     return (lightStore, ResultSuccess) -->
<!--   - if targetHeight < lightStore.LatestVerified.height -->
<!--      - let b2 be in lightStore  -->
<!--         - that is verified and not expired -->
<!-- 	    - b2.Header.Height < targetHeight -->
<!-- 	    - for all b in lightStore s.t. b  is verified and not expired it -->
<!--         holds b2.Header.Height >= b.Header.Height -->
<!-- 	 - if b2 does not exists -->
<!--          return Backwards(primary, lightStore, targetHeight) -->
<!-- 	 - if b2 exists -->
<!--           - make auxiliary light store auxLS containing only b2 -->
  
<!-- 	       VerifyToTarget(primary, auxLS, targetHeight) -->
<!--      - if b2  -->

# References

[[block]] Specification of the block data structure.

[[RPC]] RPC client for Tendermint

[[fork-detector]] The specification of the light client fork detector.

[[fullnode]] Specification of the full node API

[[ibc-rs]] Rust implementation of IBC modules and relayer.

[[lightclient]] The light client ADR [77d2651 on Dec 27, 2019].

[RPC]: https://docs.tendermint.com/v0.34/rpc/

[block]: https://github.com/tendermint/spec/blob/d46cd7f573a2c6a2399fcab2cde981330aa63f37/spec/core/data_structures.md

[TMBC-HEADER-link]: #tmbc-header1
[TMBC-SEQ-link]: #tmbc-seq1
[TMBC-CorrFull-link]: #tmbc-corr-full1
[TMBC-Auth-Byz-link]: #tmbc-auth-byz1
[TMBC-TIME_PARAMS-link]: #tmbc-time-params1
[TMBC-FM-2THIRDS-link]: #tmbc-fm-2thirds1
[TMBC-VAL-CONTAINS-CORR-link]: #tmbc-val-contains-corr1
[TMBC-VAL-COMMIT-link]: #tmbc-val-commit1
[TMBC-SOUND-DISTR-POSS-COMMIT-link]: #tmbc-sound-distr-poss-commit1

[lightclient]: https://github.com/interchainio/tendermint-rs/blob/e2cb9aca0b95430fca2eac154edddc9588038982/docs/architecture/adr-002-lite-client.md
[fork-detector]: https://github.com/informalsystems/tendermint-rs/blob/master/docs/spec/lightclient/detection.md
[fullnode]: https://github.com/tendermint/tendermint/blob/v0.34.x/spec/blockchain/fullnode.md

[ibc-rs]:https://github.com/informalsystems/ibc-rs

[FN-LuckyCase-link]: https://github.com/tendermint/tendermint/blob/v0.34.x/spec/blockchain/fullnode.md#fn-luckycase

[blockchain-validator-set]: https://github.com/tendermint/tendermint/blob/v0.34.x/spec/blockchain/blockchain.md#data-structures
[fullnode-data-structures]: https://github.com/tendermint/tendermint/blob/v0.34.x/spec/blockchain/fullnode.md#data-structures

[FN-ManifestFaulty-link]: https://github.com/tendermint/tendermint/blob/v0.34.x/spec/blockchain/fullnode.md#fn-manifestfaulty

[arXiv]: https://arxiv.org/abs/1807.04938



# Light Client Verification

The light client implements a read operation of a
[header][TMBC-HEADER-link] from the [blockchain][TMBC-SEQ-link], by
communicating with full nodes.  As some full nodes may be faulty, this
functionality must be implemented in a fault-tolerant way.

In the Tendermint blockchain, the validator set may change with every
new block.  The staking and unbonding mechanism induces a [security
model][TMBC-FM-2THIRDS-link]: starting at time *Time* of the
[header][TMBC-HEADER-link],
more than two-thirds of the next validators of a new block are correct
for the duration of *TrustedPeriod*. The fault-tolerant read
operation is designed for this security model.

The challenge addressed here is that the light client might have a
block of height *h1* and needs to read the block of height *h2*
greater than *h1*.  Checking all headers of heights from *h1* to *h2*
might be too costly (e.g., in terms of energy for mobile devices).
This specification tries to reduce the number of intermediate blocks
that need to be checked, by exploiting the guarantees provided by the
[security model][TMBC-FM-2THIRDS-link].

# Status

This document is thoroughly reviewed, and the protocol has been
formalized in TLA+ and model checked.

## Issues that need to be addressed

As it is part of the larger light node, its data structures and
functions interact with the fork dectection functionality of the light
client. As a result of the work on
[Pull Request 479](https://github.com/informalsystems/tendermint-rs/pull/479) we
established the need for an update in the data structures in [Issue 499](https://github.com/informalsystems/tendermint-rs/issues/499). This
will not change the verification logic, but it will record information
about verification that can be used in fork detection (in particular
in computing more efficiently the proof of fork).

# Outline

- [Part I](#part-i---tendermint-blockchain): Introduction of
 relevant terms of the Tendermint
blockchain.

- [Part II](#part-ii---sequential-definition-of-the-verification-problem): Introduction
of the problem addressed by the Lightclient Verification protocol.
    - [Verification Informal Problem
      statement](#Verification-Informal-Problem-statement): For the general
      audience, that is, engineers who want to get an overview over what
      the component is doing from a bird's eye view.
    - [Sequential Problem statement](#Sequential-Problem-statement):
      Provides a mathematical definition of the problem statement in
      its sequential form, that is, ignoring the distributed aspect of
      the implementation of the blockchain.

- [Part III](#part-iii---light-client-as-distributed-system): Distributed
  aspects of the light client, system assumptions and temporal
  logic specifications.

    - [Incentives](#incentives): how faulty full nodes may benefit from
    misbehaving and how correct full nodes benefit from cooperating.
  
    - [Computational Model](#Computational-Model):
      timing and correctness assumptions.

    - [Distributed Problem Statement](#Distributed-Problem-Statement):
      temporal properties that formalize safety and liveness
      properties in the distributed setting.

- [Part IV](#part-iv---light-client-verification-protocol):
  Specification of the protocols.

    - [Definitions](#Definitions): Describes inputs, outputs,
       variables used by the protocol, auxiliary functions

    - [Core Verification](#core-verification): gives an outline of the solution,
       and details of the functions used (with preconditions,
       postconditions, error conditions).

    - [Liveness Scenarios](#liveness-scenarios): when the light
       client makes progress depends heavily on the changes in the
       validator sets of the blockchain. We discuss some typical scenarios.

- [Part V](#part-v---supporting-the-ibc-relayer): The above parts
  focus on a common case where the last verified block has height *h1*
  and the
  requested height *h2* satisfies *h2 > h1*. For IBC, there are
  scenarios where this might not be the case. In this part, we provide
  some preliminaries for supporting this. As not all details of the
  IBC requirements are clear by now, we do not provide a complete
  specification at this point. We mark with "Open Question" points
  that need to be addressed in order to finalize this specification.
  It should be noted that the technically
  most challenging case is the one specified in Part IV.

In this document we quite extensively use tags in order to be able to
reference assumptions, invariants, etc. in future communication. In
these tags we frequently use the following short forms:

- TMBC: Tendermint blockchain
- SEQ: for sequential specifications
- LCV: Lightclient Verification
- LIVE: liveness
- SAFE: safety
- FUNC: function
- INV: invariant
- A: assumption

# Part I - Tendermint Blockchain

## Header Fields necessary for the Light Client

#### **[TMBC-HEADER.1]**

A set of blockchain transactions is stored in a data structure called
*block*, which contains a field called *header*. (The data structure
*block* is defined [here][block]).  As the header contains hashes to
the relevant fields of the block, for the purpose of this
specification, we will assume that the blockchain is a list of
headers, rather than a list of blocks.

#### **[TMBC-HASH-UNIQUENESS.1]**

We assume that every hash in the header identifies the data it hashes.
Therefore, in this specification, we do not distinguish between hashes and the
data they represent.

#### **[TMBC-HEADER-FIELDS.1]**

A header contains the following fields:

- `Height`: non-negative integer
- `Time`: time (integer)
- `LastBlockID`: Hashvalue
- `LastCommit` DomainCommit
- `Validators`: DomainVal
- `NextValidators`: DomainVal
- `Data`: DomainTX
- `AppState`: DomainApp
- `LastResults`: DomainRes

#### **[TMBC-SEQ.1]**

The Tendermint blockchain is a list *chain* of headers.

#### **[TMBC-VALIDATOR-PAIR.1]**

Given a full node, a
*validator pair* is a pair *(peerID, voting_power)*, where

- *peerID* is the PeerID (public key) of a full node,
- *voting_power* is an integer (representing the full node's
  voting power in a certain consensus instance).
  
> In the Golang implementation the data type for *validator
pair* is called `Validator`

#### **[TMBC-VALIDATOR-SET.1]**

A *validator set* is a set of validator pairs. For a validator set
*vs*, we write *TotalVotingPower(vs)* for the sum of the voting powers
of its validator pairs.

#### **[TMBC-VOTE.1]**

A *vote* contains a `prevote` or `precommit` message sent and signed by
a validator node during the execution of [consensus][arXiv]. Each
message contains the following fields

- `Type`: prevote or precommit
- `Height`: positive integer
- `Round` a positive integer
- `BlockID` a Hashvalue of a block (not necessarily a block of the chain)

#### **[TMBC-COMMIT.1]**

A commit is a set of `precommit` message.

## Tendermint Failure Model

#### **[TMBC-AUTH-BYZ.1]**

We assume the authenticated Byzantine fault model in which no node (faulty or
correct) may break digital signatures, but otherwise, no additional
assumption is made about the internal behavior of faulty
nodes. That is, faulty nodes are only limited in that they cannot forge
messages.

#### **[TMBC-TIME-PARAMS.1]**

A Tendermint blockchain has the following configuration parameters:

- *unbondingPeriod*: a time duration.
- *trustingPeriod*: a time duration smaller than *unbondingPeriod*.

#### **[TMBC-CORRECT.1]**

We define a predicate *correctUntil(n, t)*, where *n* is a node and *t* is a
time point.
The predicate *correctUntil(n, t)* is true if and only if the node *n*
follows all the protocols (at least) until time *t*.

#### **[TMBC-FM-2THIRDS.1]**

If a block *h* is in the chain,
then there exists a subset *CorrV*
of *h.NextValidators*, such that:

- *TotalVotingPower(CorrV) > 2/3
    TotalVotingPower(h.NextValidators)*; cf. [TMBC-VALIDATOR-SET.1]
- For every validator pair *(n,p)* in *CorrV*, it holds *correctUntil(n,
    h.Time + trustingPeriod)*; cf. [TMBC-CORRECT.1]

> The definition of correct
> [**[TMBC-CORRECT.1]**][TMBC-CORRECT-link] refers to realtime, while it
> is used here with *Time* and *trustingPeriod*, which are "hardware
> times".  We do not make a distinction here.

#### **[TMBC-CORR-FULL.1]**

Every correct full node locally stores a prefix of the
current list of headers from [**[TMBC-SEQ.1]**][TMBC-SEQ-link].

## What the Light Client Checks

> From [TMBC-FM-2THIRDS.1] we directly derive the following observation:

#### **[TMBC-VAL-CONTAINS-CORR.1]**

Given a (trusted) block *tb* of the blockchain, a given set of full nodes
*N* contains a correct node at a real-time *t*, if

- *t - trustingPeriod < tb.Time < t*
- the voting power in tb.NextValidators of nodes in *N* is more
     than 1/3 of *TotalVotingPower(tb.NextValidators)*

> The following describes how a commit for a given block *b* must look
> like.

#### **[TMBC-SOUND-DISTR-POSS-COMMIT.1]**

For a block *b*, each element *pc* of *PossibleCommit(b)* satisfies:

- *pc* contains only votes (cf. [TMBC-VOTE.1])
  by validators from *b.Validators*
- the sum of the voting powers in *pc* is greater than 2/3
  *TotalVotingPower(b.Validators)*
- and there is an *r* such that  each vote *v* in *pc* satisfies
    - v.Type = precommit
    - v.Height = b.Height
    - v.Round = r
    - v.blockID = hash(b)

> The following property comes from the validity of the [consensus][arXiv]: A
> correct validator node only sends `prevote` or `precommit`, if
> `BlockID` of the new (to-be-decided) block is equal to the hash of
> the last block.

#### **[TMBC-VAL-COMMIT.1]**

If for a block *b*,  a commit *c*

- contains at least one validator pair *(v,p)* such that *v* is a
    **correct** validator node, and
- is contained in *PossibleCommit(b)*
  
then the block *b* is on the blockchain.

## Context of this document

In this document we specify the light client verification component,
called *Core Verification*.  The *Core Verification* communicates with
a full node.  As full nodes may be faulty, it cannot trust the
received information, but the light client has to check whether the
header it receives coincides with the one generated by Tendermint
consensus.

The two
 properties [[TMBC-VAL-CONTAINS-CORR.1]][TMBC-VAL-CONTAINS-CORR-link] and
[[TMBC-VAL-COMMIT]][TMBC-VAL-COMMIT-link]  formalize the checks done
 by this specification:
Given a trusted block *tb* and an untrusted block *ub* with a commit *cub*,
one has to check that *cub* is in *PossibleCommit(ub)*, and that *cub*
contains a correct node using *tb*.

# Part II - Sequential Definition of the Verification Problem

## Verification Informal Problem statement

Given a height *targetHeight* as an input, the *Verifier* eventually
stores a header *h* of height *targetHeight* locally.  This header *h*
is generated by the Tendermint [blockchain][block]. In
particular, a header that was not generated by the blockchain should
never be stored.

## Sequential Problem statement

#### **[LCV-SEQ-LIVE.1]**

The *Verifier* gets as input a height *targetHeight*, and eventually stores the
header of height *targetHeight* of the blockchain.

#### **[LCV-SEQ-SAFE.1]**

The *Verifier* never stores a header which is not in the blockchain.

# Part III - Light Client as Distributed System

## Incentives

Faulty full nodes may benefit from lying to the light client, by making the
light client accept a block that deviates (e.g., contains additional
transactions) from the one generated by Tendermint consensus.
Users using the light client might be harmed by accepting a forged header.

The [fork detector][fork-detector] of the light client may help the
correct full nodes to understand whether their header is a good one.
Hence, in combination with the light client detector, the correct full
nodes have the incentive to respond.  We can thus base liveness
arguments on the assumption that correct full nodes reliably talk to
the light client.

## Computational Model

#### **[LCV-A-PEER.1]**

The verifier communicates with a full node called *primary*. No assumption is made about the full node (it may be correct or faulty).

#### **[LCV-A-COMM.1]**

Communication between the light client and a correct full node is
reliable and bounded in time. Reliable communication means that
messages are not lost, not duplicated, and eventually delivered. There
is a (known) end-to-end delay *Delta*, such that if a message is sent
at time *t* then it is received and processes by time *t + Delta*.
This implies that we need a timeout of at least *2 Delta* for remote
procedure calls to ensure that the response of a correct peer arrives
before the timeout expires.

#### **[LCV-A-TFM.1]**

The Tendermint blockchain satisfies the Tendermint failure model [**[TMBC-FM-2THIRDS.1]**][TMBC-FM-2THIRDS-link].

#### **[LCV-A-VAL.1]**

The system satisfies [**[TMBC-AUTH-BYZ.1]**][TMBC-Auth-Byz-link] and
[**[TMBC-FM-2THIRDS.1]**][TMBC-FM-2THIRDS-link]. Thus, there is a
blockchain that satisfies the soundness requirements (that is, the
validation rules in [[block]]).

## Distributed Problem Statement

### Two Kinds of Termination

We do not assume that *primary* is correct. Under this assumption no
protocol can guarantee the combination of the sequential
properties. Thus, in the (unreliable) distributed setting, we consider
two kinds of termination (successful and failure) and we will specify
below under what (favorable) conditions *Core Verification* ensures to
terminate successfully, and satisfy the requirements of the sequential
problem statement:

#### **[LCV-DIST-TERM.1]**

*Core Verification* either *terminates
successfully* or it *terminates with failure*.

### Design choices

#### **[LCV-DIST-STORE.1]**

*Core Verification* has a local data structure called *LightStore* that
contains light blocks (that contain a header). For each light block we
record whether it is verified.

#### **[LCV-DIST-PRIMARY.1]**

*Core Verification* has a local variable *primary* that contains the PeerID of a full node.

#### **[LCV-DIST-INIT.1]**

*LightStore* is initialized with a header *trustedHeader* that was correctly
generated by the Tendermint consensus. We say *trustedHeader* is verified.

### Temporal Properties

#### **[LCV-DIST-SAFE.1]**

It is always the case that every verified header in *LightStore* was
generated by an instance of Tendermint consensus.

#### **[LCV-DIST-LIVE.1]**

From time to time, a new instance of *Core Verification* is called with a
height *targetHeight* greater than the height of any header in *LightStore*.
Each instance must eventually terminate.

- If
    - the  *primary* is correct (and locally has the block of
       *targetHeight*), and
    - *LightStore* always contains a verified header whose age is less than the
        trusting period,  
    then *Core Verification* adds a verified header *hd* with height
    *targetHeight* to *LightStore* and it **terminates successfully**

> These definitions imply that if the primary is faulty, a header may or
> may not be added to *LightStore*. In any case,
> [**[LCV-DIST-SAFE.1]**](#lcv-vc-inv) must hold.
> The invariant [**[LCV-DIST-SAFE.1]**](#lcv-dist-safe) and the liveness
> requirement [**[LCV-DIST-LIVE.1]**](#lcv-dist-life)
> allow that verified headers are added to *LightStore* whose
> height was not passed
> to the verifier (e.g., intermediate headers used in bisection; see below).
> Note that for liveness, initially having a *trustedHeader* within
> the *trustinPeriod* is not sufficient. However, as this
> specification will leave some freedom with respect to the strategy
> in which order to download intermediate headers, we do not give a
> more precise liveness specification here. After giving the
> specification of the protocol, we will discuss some liveness
> scenarios [below](#liveness-scenarios).

### Solving the sequential specification

This specification provides a partial solution to the sequential specification.
The *Verifier* solves the invariant of the sequential part

[**[LCV-DIST-SAFE.1]**](#lcv-vc-inv) => [**[LCV-SEQ-SAFE.1]**](#lcv-seq-inv)

In the case the primary is correct, and there is a recent header in *LightStore*, the verifier satisfies the liveness requirements.

⋀ *primary is correct*  
⋀ always ∃ verified header in LightStore. *header.Time* > *now* - *trustingPeriod*  
⋀ [**[LCV-A-Comm.1]**](#lcv-a-comm) ⋀ (
       ( [**[TMBC-CorrFull.1]**][TMBC-CorrFull-link] ⋀
         [**[LCV-DIST-LIVE.1]**](#lcv-vc-live) )
       ⟹ [**[LCV-SEQ-LIVE.1]**](#lcv-seq-live)
)

# Part IV - Light Client Verification Protocol

We provide a specification for Light Client Verification. The local
code for verification is presented by a sequential function
`VerifyToTarget` to highlight the control flow of this functionality.
We note that if a different concurrency model is considered for
an implementation, the sequential flow of the function may be
implemented with mutexes, etc. However, the light client verification
is partitioned into three blocks that can be implemented and tested
independently:

- `FetchLightBlock` is called to download a light block (header) of a
  given height from a peer.
- `ValidAndVerified` is a local code that checks the header.
- `Schedule` decides which height to try to verify next. We keep this
  underspecified as different implementations (currently in Goland and
  Rust) may implement different optimizations here. We just provide
  necessary conditions on how the height may evolve.
  
<!-- > `ValidAndVerified` is the function that is sometimes called "Light -->
<!-- > Client" in the IBC context. -->

## Definitions

### Data Types

The core data structure of the protocol is the LightBlock.

#### **[LCV-DATA-LIGHTBLOCK.1]**

```go
type LightBlock struct {
  Header          Header
  Commit          Commit
  Validators      ValidatorSet
}
```

#### **[LCV-DATA-LIGHTSTORE.1]**

LightBlocks are stored in a structure which stores all LightBlock from
initialization or received from peers.

```go
type LightStore struct {
 ...
}

```

Each LightBlock is in one of the following states:

```go
type VerifiedState int

const (
 StateUnverified = iota + 1
 StateVerified
 StateFailed
 StateTrusted
)
```

> Only the detector module sets a lightBlock state to `StateTrusted`
> and only if it was `StateVerified` before.

The LightStore exposes the following functions to query stored LightBlocks.

#### **[LCV-FUNC-GET.1]**

```go
func (ls LightStore) Get(height Height) (LightBlock, bool)
```

- Expected postcondition
    - returns a LightBlock at a given height or false in the second argument if
    the LightStore does not contain the specified LightBlock.

#### **[LCV-FUNC-LATEST-VERIF.1]**

```go
func (ls LightStore) LatestVerified() LightBlock
```

- Expected postcondition
    - returns the highest light block whose state is `StateVerified`
     or `StateTrusted`

#### **[LCV-FUNC-UPDATE.2]**

```go
func (ls LightStore) Update(lightBlock LightBlock, 
                            verfiedState VerifiedState
       verifiedBy Height)
```

- Expected postcondition
    - The state of the LightBlock is set to *verifiedState*.
    - verifiedBy of the Lightblock is set to *Height*

> The following function is used only in the detector specification
> listed here for completeness.

#### **[LCV-FUNC-LATEST-TRUSTED.1]**

```go
func (ls LightStore) LatestTrusted() LightBlock
```

- Expected postcondition
    - returns the highest light block that has been verified and
     checked by the detector.

#### **[LCV-FUNC-FILTER.1]**

```go
func (ls LightStore) FilterVerified() LightSTore
```

- Expected postcondition
    - returns only the LightBlocks with state verified.

### Inputs

- *lightStore*: stores light blocks that have been downloaded and that
    passed verification. Initially it contains a light block with
 *trustedHeader*.
- *primary*: peerID
- *targetHeight*: the height of the needed header

### Configuration Parameters

- *trustThreshold*: a float. Can be used if correctness should not be based on more voting power and 1/3.
- *trustingPeriod*: a time duration [**[TMBC-TIME_PARAMS.1]**][TMBC-TIME_PARAMS-link].
- *clockDrift*: a time duration. Correction parameter dealing with only approximately synchronized clocks.

### Variables

- *nextHeight*: initially *targetHeight*
  > *nextHeight* should be thought of the "height of the next header we need
  > to download and verify"

### Assumptions

#### **[LCV-A-INIT.1]**

- *trustedHeader* is from the blockchain

- *targetHeight > LightStore.LatestVerified.Header.Height*

### Invariants

#### **[LCV-INV-TP.1]**

It is always the case that *LightStore.LatestTrusted.Header.Time > now - trustingPeriod*.

> If the invariant is violated, the light client does not have a
> header it can trust. A trusted header must be obtained externally,
> its trust can only be based on social consensus.

### Used Remote Functions

We use the functions `commit` and `validators` that are provided
by the [RPC client for Tendermint][RPC].

```go
func Commit(height int64) (SignedHeader, error)
```

- Implementation remark
    - RPC to full node *n*
    - JSON sent:

```javascript
// POST /commit
{
 "jsonrpc": "2.0",
 "id": "ccc84631-dfdb-4adc-b88c-5291ea3c2cfb", // UUID v4, unique per request
 "method": "commit",
 "params": {
  "height": 1234
 }
}
```

- Expected precondition
    - header of `height` exists on blockchain
- Expected postcondition
    - if *n* is correct: Returns the signed header of height `height`
  from the blockchain if communication is timely (no timeout)
    - if *n* is faulty: Returns a signed header with arbitrary content
- Error condition
    - if *n* is correct: precondition violated or timeout
    - if *n* is faulty: arbitrary error

----

```go
func Validators(height int64) (ValidatorSet, error)
```

- Implementation remark
    - RPC to full node *n*
    - JSON sent:

```javascript
// POST /validators
{
 "jsonrpc": "2.0",
 "id": "ccc84631-dfdb-4adc-b88c-5291ea3c2cfb", // UUID v4, unique per request
 "method": "validators",
 "params": {
  "height": 1234
 }
}
```

- Expected precondition
    - header of `height` exists on blockchain
- Expected postcondition
    - if *n* is correct: Returns the validator set of height `height`
  from the blockchain if communication is timely (no timeout)
    - if *n* is faulty: Returns arbitrary validator set
- Error condition
    - if *n* is correct: precondition violated or timeout
    - if *n* is faulty: arbitrary error

----

### Communicating Function

#### **[LCV-FUNC-FETCH.1]**

  ```go
func FetchLightBlock(peer PeerID, height Height) LightBlock
```

- Implementation remark
    - RPC to peer at *PeerID*
    - calls `Commit` for *height* and `Validators` for *height* and *height+1*
- Expected precondition
    - `height` is less than or equal to height of the peer **[LCV-IO-PRE-HEIGHT.1]**
- Expected postcondition:
    - if *node* is correct:
        - Returns the LightBlock *lb* of height `height`
      that is consistent with the blockchain
        - *lb.provider = peer* **[LCV-IO-POST-PROVIDER.1]**
        - *lb.Header* is a header consistent with the blockchain
        - *lb.Validators* is the validator set of the blockchain at height *nextHeight*
        - *lb.NextValidators* is the validator set of the blockchain at height *nextHeight + 1*
    - if *node* is faulty: Returns a LightBlock with arbitrary content
    [**[TMBC-AUTH-BYZ.1]**][TMBC-Auth-Byz-link]
- Error condition
    - if *n* is correct: precondition violated
    - if *n* is faulty: arbitrary error
    - if *lb.provider != peer*
    - times out after 2 Delta (by assumption *n* is faulty)

----

## Core Verification

### Outline

The `VerifyToTarget` is the main function and uses the following functions.

- `FetchLightBlock` is called to download the next light block. It is
  the only function that communicates with other nodes
- `ValidAndVerified` checks whether header is valid and checks if a
  new lightBlock should be trusted
  based on a previously verified lightBlock.
- `Schedule` decides which height to try to verify next

In the following description of `VerifyToTarget` we do not deal with error
handling. If any of the above function returns an error, VerifyToTarget just
passes the error on.

#### **[LCV-FUNC-MAIN.1]**

```go
func VerifyToTarget(primary PeerID, lightStore LightStore,
                    targetHeight Height) (LightStore, Result) {

    nextHeight := targetHeight

    for lightStore.LatestVerified.height < targetHeight {

        // Get next LightBlock for verification
        current, found := lightStore.Get(nextHeight)
        if !found {
            current = FetchLightBlock(primary, nextHeight)
            lightStore.Update(current, StateUnverified)
        }

        // Verify
        verdict = ValidAndVerified(lightStore.LatestVerified, current)

        // Decide whether/how to continue
        if verdict == SUCCESS {
            lightStore.Update(current, StateVerified)
        }
        else if verdict == NOT_ENOUGH_TRUST {
            // do nothing
   // the light block current passed validation, but the validator
            // set is too different to verify it. We keep the state of
   // current at StateUnverified. For a later iteration, Schedule
   // might decide to try verification of that light block again.
        }
        else {
            // verdict is some error code
            lightStore.Update(current, StateFailed)
            // possibly remove all LightBlocks from primary
            return (lightStore, ResultFailure)
        }
        nextHeight = Schedule(lightStore, nextHeight, targetHeight)
    }
    return (lightStore, ResultSuccess)
}
```

- Expected precondition
    - *lightStore* contains a LightBlock within the *trustingPeriod*  **[LCV-PRE-TP.1]**
    - *targetHeight* is greater than the height of all the LightBlocks in *lightStore*
- Expected postcondition:
    - returns *lightStore* that contains a LightBlock that corresponds to a block
     of the blockchain of height *targetHeight*
     (that is, the LightBlock has been added to *lightStore*) **[LCV-POST-LS.1]**
- Error conditions
    - if the precondition is violated
    - if `ValidAndVerified` or `FetchLightBlock` report an error
    - if [**[LCV-INV-TP.1]**](#LCV-INV-TP.1) is violated
  
### Details of the Functions

#### **[LCV-FUNC-VALID.1]**

```go
func ValidAndVerified(trusted LightBlock, untrusted LightBlock) Result
```

- Expected precondition:
    - *untrusted* is valid, that is, satisfies the soundness [checks][block]
    - *untrusted* is **well-formed**, that is,
        - *untrusted.Header.Time < now + clockDrift*
        - *untrusted.Validators = hash(untrusted.Header.Validators)*
        - *untrusted.NextValidators = hash(untrusted.Header.NextValidators)*
    - *trusted.Header.Time > now - trustingPeriod*
    - *trusted.Commit* is a commit for the header
     *trusted.Header*, i.e., it contains
     the correct hash of the header, and +2/3 of signatures
    - the `Height` and `Time` of `trusted` are smaller than the Height and
  `Time` of `untrusted`, respectively
    - the *untrusted.Header* is well-formed (passes the tests from
     [[block]]), and in particular
        - if the untrusted header `unstrusted.Header` is the immediate
   successor  of  `trusted.Header`, then it holds that
            - *trusted.Header.NextValidators =
    untrusted.Header.Validators*, and
    moreover,
            - *untrusted.Header.Commit*
                - contains signatures by more than two-thirds of the validators
                - contains no signature from nodes that are not in *trusted.Header.NextValidators*
- Expected postcondition:
    - Returns `SUCCESS`:
        - if *untrusted* is the immediate successor of *trusted*, or otherwise,
        - if the signatures of a set of validators that have more than
             *max(1/3,trustThreshold)* of voting power in
             *trusted.Nextvalidators* is contained in
             *untrusted.Commit* (that is, header passes the tests
             [**[TMBC-VAL-CONTAINS-CORR.1]**][TMBC-VAL-CONTAINS-CORR-link]
             and [**[TMBC-VAL-COMMIT.1]**][TMBC-VAL-COMMIT-link])
    - Returns `NOT_ENOUGH_TRUST` if:
        - *untrusted* is *not* the immediate successor of
           *trusted*
     and the  *max(1/3,trustThreshold)* threshold is not reached
           (that is, if
      [**[TMBC-VAL-CONTAINS-CORR.1]**][TMBC-VAL-CONTAINS-CORR-link]
      fails and header is does not violate the soundness
         checks [[block]]).
- Error condition:
    - if precondition violated

----

#### **[LCV-FUNC-SCHEDULE.1]**

```go
func Schedule(lightStore, nextHeight, targetHeight) Height
```

- Implementation remark: If picks the next height to be verified.
  We keep the precise choice of the next header under-specified. It is
  subject to performance optimizations that do not influence the correctness
- Expected postcondition: **[LCV-SCHEDULE-POST.1]**
   Return *H* s.t.
   1. if *lightStore.LatestVerified.Height = nextHeight* and
      *lightStore.LatestVerified < targetHeight* then  
   *nextHeight < H <= targetHeight*
   2. if *lightStore.LatestVerified.Height < nextHeight* and
      *lightStore.LatestVerified.Height < targetHeight* then  
   *lightStore.LatestVerified.Height < H < nextHeight*
   3. if *lightStore.LatestVerified.Height = targetHeight* then  
     *H =  targetHeight*

> Case i. captures the case where the light block at height *nextHeight*
> has been verified, and we can choose a height closer to the *targetHeight*.
> As we get the *lightStore* as parameter, the choice of the next height can
> depend on the *lightStore*, e.g., we can pick a height for which we have
> already downloaded a light block.
> In Case ii. the header of *nextHeight* could not be verified, and we need to pick a smaller height.
> In Case iii. is a special case when we have verified the *targetHeight*.

### Solving the distributed specification

*trustedStore* is implemented by the light blocks in lightStore that
have the state *StateVerified*.

#### Argument for [**[LCV-DIST-SAFE.1]**](#lcv-dist-safe)

- `ValidAndVerified` implements the soundness checks and the checks
  [**[TMBC-VAL-CONTAINS-CORR.1]**][TMBC-VAL-CONTAINS-CORR-link] and
  [**[TMBC-VAL-COMMIT.1]**][TMBC-VAL-COMMIT-link] under
  the assumption [**[TMBC-FM-2THIRDS.1]**][TMBC-FM-2THIRDS-link]
- Only if `ValidAndVerified` returns with `SUCCESS`, the state of a light block is
  set to *StateVerified*.

#### Argument for [**[LCV-DIST-LIVE.1]**](#lcv-dist-life)

- If *primary* is correct,
    - `FetchLightBlock` will always return a light block consistent
      with the blockchain
    - `ValidAndVerified` either verifies the header using the trusting
      period or falls back to sequential
      verification
    - If [**[LCV-INV-TP.1]**](#LCV-INV-TP.1) holds, eventually every
   header will be verified and core verification **terminates successfully**.
    - successful termination depends on the age of *lightStore.LatestVerified*
      (for instance, initially on the age of  *trustedHeader*) and the
      changes of the validator sets on the blockchain.
   We will give some examples [below](#liveness-scenarios).
- If *primary* is faulty,
    - it either provides headers that pass all the tests, and we
      return with the header
    - it provides one header that fails a test, core verification
      **terminates with failure**.
    - it times out and core verification
      **terminates with failure**.

## Liveness Scenarios

The liveness argument above assumes [**[LCV-INV-TP.1]**](#LCV-INV-TP.1)

which requires that there is a header that does not expire before the
target height is reached. Here we discuss scenarios to ensure this.

Let *startHeader* be *LightStore.LatestVerified* when core
verification is called (*trustedHeader*) and *startTime* be the time
core verification is invoked.

In order to ensure liveness, *LightStore* always needs to contain a
verified (or initially trusted) header whose time is within the
trusting period. To ensure this, core verification needs to add new
headers to *LightStore* and verify them, before all headers in
*LightStore* expire.

#### Many changes in validator set

 Let's consider `Schedule` implements
 bisection, that is, it halves the distance.
 Assume the case where the validator set changes completely in each
block. Then the
 method in this specification needs to
sequentially verify all headers. That is, for

- *W = log_2 (targetHeight - startHeader.Height)*,

*W* headers need to be downloaded and checked before the
header of height *startHeader.Height + 1* is added to *LightStore*.

- Let *Comp*
  be the local computation time needed to check headers and signatures
  for one header.
- Then we need in the worst case *Comp + 2 Delta* to download and
  check one header.
- Then the first time a verified header could be added to *LightStore* is
  startTime + W * (Comp + 2 Delta)
- [TP.1] However, it can only be added if we still have a header in
  *LightStore*,
  which is not
  expired, that is only the case if
    - startHeader.Time > startTime + WCG * (Comp + 2 Delta) -
      trustingPeriod,
    - that is, if core verification is started at  
   startTime < startHeader.Time + trustingPeriod -  WCG * (Comp + 2 Delta)

- one may then do an inductive argument from this point on, depending
  on the implementation of `Schedule`. We may have to account for the
  headers that are already
  downloaded, but they are checked against the new *LightStore.LatestVerified*.

> We observe that
> the worst case time it needs to verify the header of height
> *targetHeight* depends mainly on how frequent the validator set on the
> blockchain changes. That core verification terminates successfully
> crucially depends on the check [TP.1], that is, that the headers in
> *LightStore* do not expire in the time needed to download more
> headers, which depends on the creation time of the headers in
> *LightStore*. That is, termination of core verification is highly
> depending on the data stored in the blockchain.
> The current light client core verification protocol exploits that, in
> practice, changes in the validator set are rare. For instance,
> consider the following scenario.

#### No change in validator set

If on the blockchain the validator set of the block at height
*targetHeight* is equal to *startHeader.NextValidators*:

- there is one round trip in `FetchLightBlock` to download the light
 block
 of height
  *targetHeight*, and *Comp* to check it.
- as the validator sets are equal, `Verify` returns `SUCCESS`, if
  *startHeader.Time > now - trustingPeriod*.
- that is, if *startTime < startHeader.Header.Time + trustingPeriod -
  2 Delta - Comp*, then core verification terminates successfully

# Part V - Supporting the IBC Relayer

The above specification focuses on the most common case, which also
constitutes the most challenging task: using the Tendermint [security
model][TMBC-FM-2THIRDS-link] to verify light blocks without
downloading all intermediate blocks. To focus on this challenge, above
we have restricted ourselves to the case where  *targetHeight* is
greater than the height of any trusted header. This simplified
presentation of the algorithm as initially
`lightStore.LatestVerified()` is less than *targetHeight*, and in the
process of verification `lightStore.LatestVerified()` increases until
*targetHeight* is reached.

For [IBC][ibc-rs] it might be that some "older" header is
needed, that is,  *targetHeight < lightStore.LatestVerified()*. In this section we present a preliminary design, and we mark some
remaining open questions.
If  *targetHeight < lightStore.LatestVerified()* our design separates
the following cases:

- A previous instance of `VerifyToTarget` has already downloaded the
  light block of *targetHeight*. There are two cases
    - the light block has been verified
    - the light block has not been verified yet
- No light block of *targetHeight* had been downloaded before. There
  are two cases:
    - there exists a verified light block of height less than  *targetHeight*
    - otherwise. In this case we need to do "backwards verification"
     using the hash of the previous block in the `LastBlockID` field
     of a header.
  
**Open Question:** what are the security assumptions for backward
verification. Should we check that the light block we verify from
(and/or the checked light block) is within the trusting period?

The design just presents the above case
distinction as a function, and defines some auxiliary functions in the
same way the protocol was presented in
[Part IV](#part-iv---light-client-verification-protocol).

```go
func (ls LightStore) LatestPrevious(height Height) (LightBlock, bool)
```

- Expected postcondition
    - returns a light block *lb* that satisfies:
        - *lb* is in lightStore
        - *lb* is verified and not expired
        - *lb.Header.Height < height*
        - for all *b* in lightStore s.t. *b* is verified and not expired it
          holds *lb.Header.Height >= b.Header.Height*
    - *false* in the second argument if
      the LightStore does not contain such an *lb*.

```go
func (ls LightStore) MinVerified() (LightBlock, bool)
```

- Expected postcondition
    - returns a light block *lb* that satisfies:
        - *lb* is in lightStore
        - *lb* is verified **Open Question:** replace by trusted?
        - *lb.Header.Height* is minimal in the lightStore
        - **Open Question:** according to this, it might be expired (outside the
          trusting period). This approach appears safe. Are there reasons we
          should not do that?
    - *false* in the second argument if
      the LightStore does not contain such an *lb*.

If a height that is smaller than the smallest height in the lightstore
is required, we check the hashes backwards. This is done with the
following function:

#### **[LCV-FUNC-BACKWARDS.1]**

```go
func Backwards (primary PeerID, lightStore LightStore, targetHeight Height)
               (LightStore, Result) {
  
    lb,res = lightStore.MinVerified()
    if res = false {
        return (lightStore, ResultFailure)
    }

    latest := lb.Header
    for i := lb.Header.height - 1; i >= targetHeight; i-- {
        // here we download height-by-height. We might first download all
        // headers down to targetHeight and then check them.
        current := FetchLightBlock(primary,i)
        if (hash(current) != latest.Header.LastBlockId) {
            return (lightStore, ResultFailure)
        }
        else {
            lightStore.Update(current, StateVerified)
            // **Open Question:** Do we need a new state type for
            // backwards verified light blocks?
        }
        latest = current
    }
    return (lightStore, ResultSuccess)
}
```

The following function just decided based on the required height which
method should be used.

#### **[LCV-FUNC-IBCMAIN.1]**

```go
func Main (primary PeerID, lightStore LightStore, targetHeight Height)
          (LightStore, Result) {

    b1, r1 = lightStore.Get(targetHeight)
    if r1 = true and b1.State = StateVerified {
        // block already there
        return (lightStore, ResultSuccess)
    }

    if targetHeight > lightStore.LatestVerified.height {
     // case of Part IV
        return VerifyToTarget(primary, lightStore, targetHeight)
    }
    else {
        b2, r2 = lightStore.LatestPrevious(targetHeight);
        if r2 = true {
            // make auxiliary lightStore auxLS to call VerifyToTarget.
   // VerifyToTarget uses LatestVerified of the given lightStore
            // For that we need:
            // auxLS.LatestVerified = lightStore.LatestPrevious(targetHeight)
            auxLS.Init;
            auxLS.Update(b2,StateVerified);
            if r1 = true {
                // we need to verify a previously downloaded light block.
                // we add it to the auxiliary store so that VerifyToTarget
                // does not download it again
                auxLS.Update(b1,b1.State);
            }
            auxLS, res2 = VerifyToTarget(primary, auxLS, targetHeight)
            // move all lightblocks from auxLS to lightStore,
            // maintain state
   // we do that whether VerifyToTarget was successful or not
            for i, s range auxLS {
                lighStore.Update(s,s.State)
            }
            return (lightStore, res2)
        }
        else {
            return Backwards(primary, lightStore, targetHeight)
        }
    }
}
```
<!-- - Expected postcondition: -->
<!--   - if targetHeight > lightStore.LatestVerified.height then -->
<!--     return VerifyToTarget(primary, lightStore, targetHeight) -->
<!--   - if targetHeight = lightStore.LatestVerified.height then -->
<!--     return (lightStore, ResultSuccess) -->
<!--   - if targetHeight < lightStore.LatestVerified.height -->
<!--      - let b2 be in lightStore  -->
<!--         - that is verified and not expired -->
<!-- 	    - b2.Header.Height < targetHeight -->
<!-- 	    - for all b in lightStore s.t. b  is verified and not expired it -->
<!--         holds b2.Header.Height >= b.Header.Height -->
<!-- 	 - if b2 does not exists -->
<!--          return Backwards(primary, lightStore, targetHeight) -->
<!-- 	 - if b2 exists -->
<!--           - make auxiliary light store auxLS containing only b2 -->
  
<!-- 	       VerifyToTarget(primary, auxLS, targetHeight) -->
<!--      - if b2  -->

# References

[[block]] Specification of the block data structure.

[[RPC]] RPC client for Tendermint

[[fork-detector]] The specification of the light client fork detector.

[[fullnode]] Specification of the full node API

[[ibc-rs]] Rust implementation of IBC modules and relayer.

[[lightclient]] The light client ADR [77d2651 on Dec 27, 2019].

[RPC]: https://docs.tendermint.com/v0.34/rpc/

[block]: https://github.com/tendermint/spec/blob/d46cd7f573a2c6a2399fcab2cde981330aa63f37/spec/core/data_structures.md

[TMBC-HEADER-link]: #tmbc-header1
[TMBC-SEQ-link]: #tmbc-seq1
[TMBC-CorrFull-link]: #tmbc-corr-full1
[TMBC-Auth-Byz-link]: #tmbc-auth-byz1
[TMBC-TIME_PARAMS-link]: #tmbc-time-params1
[TMBC-FM-2THIRDS-link]: #tmbc-fm-2thirds1
[TMBC-VAL-CONTAINS-CORR-link]: #tmbc-val-contains-corr1
[TMBC-VAL-COMMIT-link]: #tmbc-val-commit1
[TMBC-SOUND-DISTR-POSS-COMMIT-link]: #tmbc-sound-distr-poss-commit1

[lightclient]: https://github.com/interchainio/tendermint-rs/blob/e2cb9aca0b95430fca2eac154edddc9588038982/docs/architecture/adr-002-lite-client.md
[fork-detector]: https://github.com/informalsystems/tendermint-rs/blob/master/docs/spec/lightclient/detection.md
[fullnode]: https://github.com/tendermint/tendermint/blob/v0.34.x/spec/blockchain/fullnode.md

[ibc-rs]:https://github.com/informalsystems/ibc-rs

[FN-LuckyCase-link]: https://github.com/tendermint/tendermint/blob/v0.34.x/spec/blockchain/fullnode.md#fn-luckycase

[blockchain-validator-set]: https://github.com/tendermint/tendermint/blob/v0.34.x/spec/blockchain/blockchain.md#data-structures
[fullnode-data-structures]: https://github.com/tendermint/tendermint/blob/v0.34.x/spec/blockchain/fullnode.md#data-structures

[FN-ManifestFaulty-link]: https://github.com/tendermint/tendermint/blob/v0.34.x/spec/blockchain/fullnode.md#fn-manifestfaulty

[arXiv]: https://arxiv.org/abs/1807.04938