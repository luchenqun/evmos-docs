---
sidebar_position: 5
---

# 内存池

了解 Tendermint 中可用的内存池选项。

## 先进先出（FIFO）内存池

内存池保存未提交的交易，这些交易尚未包含在区块中。
Tendermint 区块链的默认内存池实现遵循先进先出（FIFO）原则，
这意味着交易的排序仅取决于它们到达节点的顺序。
首个接收到的交易将是首个被处理的交易。
这对于将接收到的交易传播给其他节点以及将其包含在区块中都是如此。

## 优先级内存池

从 [Tendermint v0.35](https://github.com/tendermint/tendermint/blob/v0.35.0/CHANGELOG.md)
（也已回溯到 [v0.34.20](https://github.com/tendermint/tendermint/blob/17c94bb0dcb354c57f49cdcd1e62f4742752c803/UPGRADING.md?plain=1#L54)）开始，
可以使用优先级内存池实现。
这允许验证人根据相关费用或其他激励机制选择交易。
通过在每个 [`CheckTx` 响应](https://github.com/tendermint/tendermint/blob/17c94bb0dcb354c57f49cdcd1e62f4742752c803/proto/tendermint/abci/types.proto#L234) 中传递一个 `priority` 字段来实现。

Evmos 通过其 [feemarket 模块](../../protocol/modules/feemarket) 支持 [EIP-1559](https://eips.ethereum.org/EIPS/eip-1559#simple-summary) EVM 交易。
该交易类型使用基础费用和可选择的优先级小费，二者相加得到总交易费用。
优先级内存池提供了一种自动利用此机制生成区块的选项。

在使用优先级内存池时，下一个生成的区块的交易会按照它们的优先级（即费用）从高到低进行选择。
如果内存池已满，优先级实现允许删除优先级最低的交易，直到有足够的磁盘空间来容纳新到达的、优先级更高的交易（有关更多详细信息，请参见 [v1/mempool.go](https://github.com/tendermint/tendermint/blob/17c94bb0dcb354c57f49cdcd1e62f4742752c803/mempool/v1/mempool.go#L505C2-L576) 实现）。

:::tip
即使交易处理可以按优先级排序，交易的传播始终按照先进先出（FIFO）的顺序进行。
:::

## 配置

要使用优先级内存池，请在节点配置文件 `~/.evmosd/config/config.toml` 中将 `version = "v1"` 进行调整。
默认值 `"v0"` 表示传统的先进先出（FIFO）内存池。

:::tip
记得**重启**节点以使更改生效。
:::

以下是 `config.toml` 的相关摘录：

```toml
#######################################################
###          Mempool Configuration Option          ###
#######################################################
[mempool]

# Mempool version to use:
#   1) "v0" - (default) FIFO mempool.
#   2) "v1" - prioritized mempool.
version = "v1"
```

## 资源

更详细的信息可以在以下位置找到：

- [Tendermint ADR-067 - Mempool Refactor](https://github.com/tendermint/tendermint/blob/main/docs/architecture/adr-067-mempool-refactor.md).
- [博文：Tendermint v0.35 Announcement](https://medium.com/tendermint/tendermint-v0-35-introduces-prioritized-mempool-a-makeover-to-the-peer-to-peer-network-more-61eea6ec572d)
- [EIP-1559：ETH 1.0 链的费用市场变更](https://eips.ethereum.org/EIPS/eip-1559)
- [EIP-1559 FAQ](https://notes.ethereum.org/@vbuterin/eip-1559-faq)
- [博文：什么是 EIP-1559？它将如何改变以太坊？](https://consensys.net/blog/quorum/what-is-eip-1559-how-will-it-change-ethereum/)


---
sidebar_position: 5
---

# Mempool

Learn about the available mempool options in Tendermint.

## FIFO Mempool

The mempool holds uncommitted transactions, which are not yet included in a block.
The default mempool implementation for Tendermint blockchains follows a first-in-first-out (FIFO) principle,
which means the ordering of transactions depends solely on the order in which they arrive at the node.
The first transaction to be received will be the first transaction to be processed.
This is true for gossiping the received transactions to the rest of the peers as well as including them in a block.

## Prioritized Mempool

Starting with [Tendermint v0.35](https://github.com/tendermint/tendermint/blob/v0.35.0/CHANGELOG.md)
(has also been backported to [v0.34.20](https://github.com/tendermint/tendermint/blob/17c94bb0dcb354c57f49cdcd1e62f4742752c803/UPGRADING.md?plain=1#L54))
it is possible to use a prioritized mempool implementation.
This allows validators to choose transactions based on the associated fees or other incentive mechanisms.
It is achieved by passing a `priority` field with each [`CheckTx` response](https://github.com/tendermint/tendermint/blob/17c94bb0dcb354c57f49cdcd1e62f4742752c803/proto/tendermint/abci/types.proto#L234),
which is run on any transaction trying to enter the mempool.

Evmos supports [EIP-1559](https://eips.ethereum.org/EIPS/eip-1559#simple-summary) EVM transactions through its
[feemarket module](../../protocol/modules/feemarket).
This transaction type uses a base fee and a selectable priority tip that add up to the total transaction fees.
The prioritized mempool presents an option to automatically make use of this mechanism regarding block generation.

When using the prioritized mempool, transactions for the next produced block are chosen
by order of their priority (i.e. their fees) from highest to lowest.
Should the mempool be full, the prioritized implementation allows
to remove the transactions with the lowest priority until enough disk space is available for
an incoming, higher-priority transaction (see [v1/mempool.go](https://github.com/tendermint/tendermint/blob/17c94bb0dcb354c57f49cdcd1e62f4742752c803/mempool/v1/mempool.go#L505C2-L576) implementation for more details).

:::tip
Even though the transaction processing can be ordered by priority, the gossiping of transactions will always be according to FIFO.
:::

## Configuration

To use the a prioritized mempool, adjust `version = "v1"` in the node configuration at `~/.evmosd/config/config.toml`.
The default value `"v0"` indicates the traditional FIFO mempool.

:::tip
Remember to **restart** the node for the changes to take effect.
:::

See the relevant excerpt from `config.toml` here:

```toml
#######################################################
###          Mempool Configuration Option          ###
#######################################################
[mempool]

# Mempool version to use:
#   1) "v0" - (default) FIFO mempool.
#   2) "v1" - prioritized mempool.
version = "v1"
```

## Resources

More detailed information can be found here:

- [Tendermint ADR-067 - Mempool Refactor](https://github.com/tendermint/tendermint/blob/main/docs/architecture/adr-067-mempool-refactor.md).
- [Blogpost: Tendermint v0.35 Announcement](https://medium.com/tendermint/tendermint-v0-35-introduces-prioritized-mempool-a-makeover-to-the-peer-to-peer-network-more-61eea6ec572d)
- [EIP-1559: Fee market change for ETH 1.0 chain](https://eips.ethereum.org/EIPS/eip-1559)
- [EIP-1559 FAQ](https://notes.ethereum.org/@vbuterin/eip-1559-faq)
- [Blogpost: What is EIP-1559? How will it change Ethereum?](https://consensys.net/blog/quorum/what-is-eip-1559-how-will-it-change-ethereum/)