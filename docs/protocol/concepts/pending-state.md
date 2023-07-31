# 待处理状态

当一个交易被提交到以太坊网络时，它首先进入待处理状态，等待节点执行。如果交易的燃气价格设置得非常低，并且节点正在处理其他更高燃气价格的交易，那么交易可能会在待处理状态下停留较长时间。

在待处理状态下，交易发起者可以随时更改交易字段。他们可以通过发送具有相同nonce的另一笔交易来实现。

## 先决条件阅读

- [Cosmos SDK Mempool](https://docs.cosmos.network/main/building-apps/app-mempool)

## Evmos与以太坊的区别

在以太坊中，待处理的区块在矿工排队生产时生成。这些待处理的区块包括矿工选择的待处理交易，选择的依据是支付的燃气费用最高。这种机制存在的原因是以太坊网络上无法实现区块的最终性。区块以概率性的最终性进行提交，这意味着随着时间（和区块）的推移，交易和区块变得越来越不可能被撤销。

Evmos在这方面的设计与以太坊有很大的不同，它没有"待处理状态"的概念。Evmos使用[Tendermint Core](https://docs.tendermint.com/) BFT共识，为交易提供即时的最终性。因此，Ethermint不需要待处理状态机制，因为（如果不是大多数）交易将被提交到下一个区块（Cosmos链的平均区块时间约为8秒）。然而，这在以太坊兼容的Web3查询中会导致一些问题，因为无法查询待处理状态。

与以太坊的另一个重要区别是，区块是由验证者或区块生产者生成的，他们按照先进先出（FIFO）的方式将本地mempool中的交易包含在区块中。在Evmos上，无法从Tendermint节点的mempool中对交易进行排序或挑选。

## 待处理状态查询

Evmos将查询考虑节点事务内存池中的所有未确认交易。进行的待处理状态查询将是主观的，并且查询将在目标节点的内存池上进行。因此，对于两个不同节点的相同查询，待处理状态将不相同。

### 关于待处理交易的JSON-RPC调用

- [`eth_getBalance`](./../../develop/api/ethereum-json-rpc/methods#eth_getbalance)
- [`eth_getTransactionCount`](./../../develop/api/ethereum-json-rpc/methods#eth_gettransactioncount)
- [`eth_getBlockTransactionCountByNumber`](./../../develop/api/ethereum-json-rpc/methods#eth_getblocktransactioncountbynumber)
- [`eth_getBlockByNumber`](./../../develop/api/ethereum-json-rpc/methods#eth_getblockbynumber)
- [`eth_getTransactionByHash`](./../../develop/api/ethereum-json-rpc/methods#eth_gettransactionbyhash)
- [`eth_getTransactionByBlockNumberAndIndex`](./../../develop/api/ethereum-json-rpc/methods#eth_gettransactionbyblockhashandindex)
- [`eth_sendTransaction`](./../../develop/api/ethereum-json-rpc/methods#eth_sendtransaction)


---
sidebar_position: 10
---

# Pending State

When a transaction is submitted to the Ethereum network, it first goes into the pending status, waiting to be executed
by the nodes. A transaction can be in the pending state for a longer duration if the gas price is set very low in the
transaction and the nodes are busy processing other higher gas price transactions.

During the pending state, the transaction initiator is allowed to change the transaction fields at any time. They can do
so by sending another transaction with the same nonce.

## Prerequisite Readings

- [Cosmos SDK Mempool](https://docs.cosmos.network/main/building-apps/app-mempool)

## Evmos vs Ethereum

In Ethereum, pending blocks are generated as they are queued for production by miners. These pending
blocks include pending transactions that are picked out by miners, based on the highest reward paid
in gas. This mechanism exists as block finality is not possible on the Ethereum network. Blocks are
committed with probabilistic finality, which means that transactions and blocks become less likely
to become reverted as more time (and blocks) passes.

Evmos is designed quite differently on this front as there is no concept of a "pending state".
Evmos uses [Tendermint Core](https://docs.tendermint.com/) BFT consensus which provides instant
finality for transaction. For this reason, Ethermint does not require a pending state mechanism, as
all (if not most) of the transactions will be committed to the next block (avg. block time on Cosmos chains is ~8s).
However, this causes a
few hiccups in terms of the Ethereum Web3-compatible queries that can be made to pending state.

Another significant difference with Ethereum, is that blocks are produced by validators or block producers, who include
transactions from their local mempool into blocks in a
first-in-first-out (FIFO) fashion. Transactions on Evmos cannot be ordered or cherry picked out from the Tendermint node
[mempool](https://docs.tendermint.com/v0.34/tendermint-core/mempool.html).

## Pending State Queries

Evmos will make queries which will account for any unconfirmed transactions present in a node's
transaction mempool. A pending state query made will be subjective and the query will be made on the
target node's mempool. Thus, the pending state will not be the same for the same query to two
different nodes.

### JSON-RPC Calls on Pending Transactions

- [`eth_getBalance`](./../../develop/api/ethereum-json-rpc/methods#eth_getbalance)
- [`eth_getTransactionCount`](./../../develop/api/ethereum-json-rpc/methods#eth_gettransactioncount)
- [`eth_getBlockTransactionCountByNumber`](./../../develop/api/ethereum-json-rpc/methods#eth_getblocktransactioncountbynumber)
- [`eth_getBlockByNumber`](./../../develop/api/ethereum-json-rpc/methods#eth_getblockbynumber)
- [`eth_getTransactionByHash`](./../../develop/api/ethereum-json-rpc/methods#eth_gettransactionbyhash)
- [`eth_getTransactionByBlockNumberAndIndex`](./../../develop/api/ethereum-json-rpc/methods#eth_gettransactionbyblockhashandindex)
- [`eth_sendTransaction`](./../../develop/api/ethereum-json-rpc/methods#eth_sendtransaction)
