# Gas和费用

用户在Evmos网络上提交交易时需要支付费用。
由于以太坊和Cosmos处理费用的方式不同，
因此了解Evmos区块链如何实现与Cosmos SDK兼容的以太坊类型费用计算非常重要。

因此，本概述解释了燃气计算的基础知识，
如何为交易提供费用，
以及以太坊类型费用计算如何使用费用市场（EIP1559）来优先处理交易。

此外，请注意，与Evmos上的智能合约交互所支付的费用
可以为智能合约部署者带来收入。有关此信息，
请参阅[develop](./../../develop/mainnet#revenue)。

## 先决条件阅读

- [Cosmos SDK燃气](https://docs.cosmos.network/main/basics/gas-fees.html)
- [以太坊燃气](https://ethereum.org/en/developers/docs/gas/)

## 基础知识

### 为什么交易需要费用？

如果任何人都可以免费向网络提交交易，
网络可能会被少数参与者淹没，
他们发送大量的欺诈性交易，
以阻塞网络并使其停止工作。

解决这个问题的方法是一个称为“燃气”的概念，
它是在交易执行过程中消耗的资源。
实际上，每个代码执行步骤都会消耗少量的燃气，
从而有效地收取验证者资源的使用费用，
并防止恶意参与者随意停止网络。

### 什么是燃气？

一般来说，燃气是衡量特定交易的计算强度的单位，
换句话说，评估和执行该工作所需的工作量有多大。
复杂的多步交易，
例如委托给十几个验证者的Cosmos交易，
比简单的单步交易需要更多的燃气，
例如将代币发送到另一个地址的Cosmos交易。

在提到交易时，
“燃气”指的是交易所需的总燃气数量。
例如，一个交易可能需要执行300,000个燃气单位。

燃气可以被视为房屋或工厂内的电力（千瓦时），
或者汽车的燃料。
这个想法是到达某个地方是需要付出代价的。

关于 Gas 的更多信息：

- [Cosmos Gas 费用](https://docs.cosmos.network/main/basics/gas-fees)
- [Cosmos 交易生命周期](https://docs.cosmos.network/main/basics/tx-lifecycle.html)
- [以太坊 Gas](https://ethereum.org/en/developers/docs/gas/)

### Gas 如何计算？

一般来说，没有办法准确知道一笔交易将花费多少 gas，除非直接运行它。
使用 Cosmos SDK，可以通过[模拟交易](https://docs.cosmos.network/main/run-node/txs#simulating-a-transaction)来实现。
否则，可以根据交易字段和数据的详细信息来估算交易所需的 gas 数量。
例如，在 EVM 中，每个字节码操作都有[对应的 gas 数量](https://ethereum.org/en/developers/docs/evm/opcodes/)。

关于 Gas 计算的更多信息：

- [估算 Gas](https://docs.ethers.org/v5/api/providers/provider/#Provider-estimateGas)
- [执行 EVM 字节码](https://ethereum.org/en/developers/docs/evm/opcodes/)
- [模拟 Cosmos SDK 交易](https://docs.cosmos.network/main/run-node/txs#simulating-a-transaction)

### Gas 与费用的关系是什么？

虽然 gas 指的是执行所需的计算工作量，但费用指的是实际花费执行交易的代币数量。
它们是通过以下公式计算得出的：

```markdown
总费用 = Gas * Gas 价格（每单位 gas 的价格）
```

如果“gas”以千瓦时（kWh）为单位，那么“gas 价格”将是您能源供应商确定的费率（每千瓦时的美元价格），
而“费用”将是您的账单。
就像电力一样，gas 价格可能会在一天内波动，取决于网络流量。

关于 Gas 与费用的更多信息：

- [Cosmos Gas 和费用](https://docs.cosmos.network/main/basics/gas-fees)
- [以太坊 Gas 和费用](https://ethereum.org/en/developers/docs/gas/)

### Cosmos 如何处理费用？

Cosmos 上的 gas 费用相对简单。作为用户，您需要指定两个字段：

1. `GasLimit`，对应于执行 gas 的上限，定义为 `GasWanted`
2. `Fees` 或 `GasPrice` 中的一个，用于指定或计算交易费用

节点将完全消耗提供的费用，然后开始执行交易。如果在执行过程中发现`GasLimit`不足，则交易将失败并回滚任何已进行的更改，而不会退还提供的费用。

基于 Cosmos SDK 的链的验证人可以在选择要包含在区块中的交易时指定他们将强制执行的`min-gas-prices`。因此，费用不足的交易将遇到延迟或直接失败。

在每个区块的开始时，来自上一个区块的费用会被[分配给验证人和委托人](https://docs.cosmos.network/main/modules/distribution)，然后可以提取和使用。

### 以太坊上的费用是如何处理的？

以太坊上的费用包括随着时间推移引入的多个实现。

最初，用户会在交易中指定`GasPrice`和`GasLimit`，就像 Cosmos SDK 交易一样。区块提议者会从每个交易中获得整个燃气费用，并相应地选择要包含的交易。

随着提案 EIP-1559 和伦敦硬分叉的推出，燃气计算发生了变化。上述的`GasPrice`现在被拆分为两个独立的组成部分：`BaseFee`和`PriorityFee`。`BaseFee`根据区块大小自动计算，并在区块被挖掘后被销毁。`PriorityFee`将给予提议者，并代表一个小费，或者是鼓励提议者将交易包含在区块中的激励。

```markdown
燃气价格 = 基础费用 + 优先费用
```

在交易中，用户可以指定`max_fee_per_gas`，对应于总的`GasPrice`，以及`max_priority_fee_per_gas`，对应于最大的`PriorityFee`，除了像以前一样指定`gas_limit`。所有未被执行所需的多余燃气将退还给用户。

有关以太坊费用的更多信息：

- [燃气计算文档](https://ethereum.org/en/developers/docs/gas/)
- [提案 EIP-1559](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-1559.md)

## 实现

### Evmos 上的燃气和费用是如何处理的？

基本上，Evmos是一个Cosmos SDK链，作为Cosmos SDK模块的一部分，实现了EVM的兼容性。由于这种架构，所有的EVM交易最终都被编码为Cosmos SDK交易，并更新了由Cosmos SDK管理的状态。

由于所有的交易都表示为Cosmos SDK交易，因此可以在执行层面上以相同的方式处理交易费用。在实践中，处理费用包括标准的Cosmos SDK逻辑、一些以太坊逻辑和自定义的Evmos逻辑。在大部分情况下，费用由`fee_collector`模块收集，然后支付给验证人和委托人。以下是一些关键区别：

1. 费用市场模块

    为了支持Evmos的EVM层上的EIP-1559燃气和费用计算，Evmos跟踪每个区块的燃气供应量，并使用它来计算未来EVM交易的基础费用，从而实现了EIP-1559规定的EVM动态费用和交易优先级。

    对于EVM交易，每个节点都会绕过它们本地的`min-gas-prices`配置，而是应用EIP-1559的费用逻辑——燃气价格必须大于全局的`min-gas-price`和区块的`BaseFee`，多余的部分被视为优先小费。这使得验证人可以计算以太坊的费用，而不应用Cosmos SDK的费用逻辑。

    与以太坊不同，Evmos上的`BaseFee`不会被销毁，而是分发给验证人和委托人。此外，`BaseFee`由全局的`min-gas-price`下限（目前，全局的`min-gas-price`参数设置为零，尽管可以通过治理进行更新）。

2. EVM燃气退款

    Evmos将EVM交易中未使用的燃气的一部分（默认至少50%）退还，以近似以太坊的当前行为。[为什么不总是100%？](https://github.com/evmos/ethermint/issues/1085)

3. 收入模块

    Evmos开发了收入模块作为一种奖励开发者创建有用的dApp的方式——任何在Evmos的收入模块中注册的合约都会将与该合约交互的每个交易的交易费（目前为95%）的一部分奖励给合约开发者。验证人和委托人获得剩余部分。

### 详细时间线

1. 节点执行上一个区块并运行 `EndBlock` 钩子
    * 作为该钩子的一部分，FeeMarket（EIP-1559）模块跟踪该区块上的交易的总 `TransientGasWanted`。这将用于下一个区块的 `BaseFee`。
2. 节点接收后续区块的交易并将这些交易传播给对等节点
    * 这些交易可以按照包含的费用价格进行排序和优先级排序（使用 EIP-1559 的费用优先级机制用于 EVM 交易 - [代码片段](https://github.com/evmos/ethermint/blob/57ed355c985d9f3116aba6aabfa2ee0f3f38e966/app/ante/eth.go#L137)），
     以便包含在下一个区块中
3. 节点为后续区块运行 `BeginBlock`
    * FeeMarket 模块使用上一个区块的总 `GasWanted` 计算要应用于该区块的 `BaseFee`（[代码片段](https://github.com/evmos/ethermint/blob/89fdd1984826ea524cb9b8feb089a99b6cfe8ace/x/feemarket/keeper/abci.go#L14)）。
    * Distribution 模块将上一个区块的费用奖励分发给验证人和委托人（[分发](https://docs.cosmos.network/main/modules/distribution#begin-block)）
4. 对于将包含在此区块中的每个有效交易，节点执行以下操作：
    * 它们运行与交易类型相对应的 `AnteHandler`。此过程：
        1. 执行基本的交易验证
        2. 验证提供的费用是否大于全局和本地最小验证器值 *且* 大于计算得到的 `BaseFee`
        3. （对于以太坊交易）预先消耗 EVM 交易的 gas
        4. 从用户中扣除交易费用并将其转移到 `fee_collector` 模块
        5. 增加当前区块中的 `TransientGasWanted`，以用于计算下一个区块的 `BaseFee`
    * 然后，对于标准的 Cosmos 交易，节点：
        1. 执行交易并更新状态
        2. 消耗交易的 gas
    * 对于以太坊交易，节点：
        1. 执行交易并更新状态
        2. 计算使用的 gas 并将其与提供的 gas 进行比较，然后退还剩余部分的一部分
        3. 如果交易与已注册的智能合约进行交互，则将使用的费用的一部分作为收入发送给合约开发人员，作为收入模块的一部分
5. 节点为此区块运行 `EndBlock` 并存储区块的 `GasWanted`

{/*try-react*/}

## 详细机制

### Cosmos `Gas`

在 Cosmos SDK 中，`gas` 被跟踪在主要的 `GasMeter` 和 `BlockGasMeter` 中：

- `GasMeter`：跟踪在导致状态转换的执行过程中消耗的 gas。它在每次交易执行时被重置。
- `BlockGasMeter`：跟踪在一个区块中消耗的 gas，并强制确保 gas 不超过预定义的限制。该限制在 Tendermint 共识参数中定义，并可以通过治理参数更改提案进行更改。

由于 gas 的定价是按字节计算的，相同的交互对于较大的参数值比较小的参数值消耗更多的 gas（与以太坊的 `uint256` 值不同，Cosmos SDK 的数值类型使用 [Big.Int](https://pkg.go.dev/math/big#Int) 类型表示，这些类型是动态大小的）。

有关 Cosmos SDK 中 gas 的更多信息，请参阅[此处](https://docs.cosmos.network/main/basics/gas-fees.html)。

### 匹配 EVM 的 Gas 消耗

Evmos 是一个兼容以太坊 EVM 的链，支持以太坊的 Web3 工具。因此，gas 消耗必须与其他 EVM（尤其是以太坊）相等。

EVM 和 Cosmos 状态转换之间的主要区别在于，EVM 使用一个 [gas 表](https://github.com/ethereum/go-ethereum/blob/master/params/protocol_params.go) 来表示每个 OPCODE 的 gas，而 Cosmos 使用一个 `GasConfig`，通过为访问数据库设置固定的每字节费用，为每个 CRUD 操作收取 gas。

+++ https://github.com/cosmos/cosmos-sdk/blob/3fd376bd5659f076a4dc79b644573299fd1ec1bf/store/types/gas.go#L187-L196

为了与 EVM 消耗的 gas 匹配，忽略了 SDK 中的 gas 消耗逻辑，而是通过从消息中定义的 gas 限制中减去状态转换剩余 gas 和退款来计算消耗的 gas。

为了忽略 SDK 的 gas 消耗，我们将交易的 `GasMeter` 计数重置为 0，并手动将其设置为执行结束时 EVM 模块计算的 `gasUsed` 值。

+++ https://github.com/evmos/ethermint/blob/098da6d0cc0e0c4cefbddf632df1057383973e4a/x/evm/keeper/state_transition.go#L188

### `AnteHandler`（事前处理程序）

Cosmos SDK的[`AnteHandler`](https://docs.cosmos.network/main/basics/gas-fees.html#antehandler)在交易执行之前执行基本检查。这些检查通常包括签名验证、交易字段验证、交易费用等。

关于燃气消耗和费用，`AnteHandler`检查用户是否有足够的余额来支付交易成本（金额加上费用），并检查消息中定义的燃气限制是否大于或等于消息的计算内在燃气。

### 燃气退款

在EVM中，可以在执行之前指定燃气。指定的燃气总量在执行开始时消耗（在`AnteHandler`步骤中），如果执行后还有剩余燃气，则将剩余燃气退还给用户。此外，EVM还可以定义要退还给用户的燃气，但根据使用的分叉/版本，这些燃气将被限制为已使用燃气的一部分。

### 零费用交易

在Cosmos中，`AnteHandler`不强制执行最低燃气价格，因为`min-gas-prices`与本地节点/验证器进行了检查。换句话说，接受的最低费用由网络的验证器确定，每个验证器可以为其费用指定不同的最低值。这可能允许最终用户提交0费用交易，如果至少有一个验证器愿意在其提议的区块中包含`0`燃气价格的交易。

出于同样的原因，在Evmos中，除了由`evm`模块定义的交易类型之外，还可以发送`0`费用的交易。EVM模块的交易不能具有`0`费用，因为EVM本质上需要燃气。这个检查由EVM交易的无状态验证（即`ValidateBasic`）函数以及由Evmos定义的自定义`AnteHandler`执行。

### 燃气估算

以太坊提供了一个JSON-RPC端点`eth_estimateGas`，以帮助用户在其交易中设置正确的燃气限制。

因此，在Evmos中实现了一个特定的查询API`EstimateGas`。它将对当前块/状态应用交易，并执行二分搜索，以找到要返回给用户的最佳燃气值（将重复应用相同的交易，直到找到在失败之前所需的最小燃气）。我们需要使用二分搜索的原因是，交易所需的燃气可能高于应用交易后EVM返回的值，因此我们需要尝试，直到找到最佳值。

整个执行过程中将使用缓存上下文，以避免更改被持久化到状态中。

+++ https://github.com/evmos/ethermint/blob/098da6d0cc0e0c4cefbddf632df1057383973e4a/x/evm/keeper/grpc_query.go#L100

对于 Cosmos 交易，开发者可以使用 Cosmos SDK 的[交易模拟](https://docs.cosmos.network/main/run-node/txs#simulating-a-transaction)来创建准确的估算。

### 跨链 Gas 和费用

假设用户通过 IBC 转账将代币从 Chain A 转到 Evmos，并且想要执行一个 Evmos 交易，但是他们没有足够的 Evmos 代币来支付费用。Cosmos SDK 引入了 `Tips` 来解决这个问题；用户可以使用不同的代币（在这种情况下是来自 Chain A 的代币）来支付费用。

为了使用 tip 来支付交易费用，用户可以使用一个带有 tip 但没有费用的交易进行签名，然后将交易发送给费用中继。费用中继将以本地货币（在这种情况下是 Evmos）支付费用，并收取 tip 作为支付，充当中间交易所。

有关 Cosmos Tips 的更多信息：

- [Cosmos Tips 文档](https://docs.cosmos.network/main/core/tips)

## 使用 Evmos CLI 处理 Gas 和费用

在使用 Evmos CLI 客户端广播交易时，用户应该考虑可用的选项。在将交易发送到网络时，有三个标志需要考虑：

- `--fees`：与交易一起支付的费用；例如：10aevmos。默认为所需的费用。
- `--gas`：每笔交易设置的 gas 限制；默认值为 200000。
- `--gas-prices`：用于确定交易费用的 gas 价格（例如：10aevmos）。

然而，并不是每个交易都需要定义所有这些标志。正确的组合方式有：

- `--fees=auto`：自动估算费用和 gas（与 `--gas=auto` 的行为相同）。
  如果使用其他与费用相关的标志（例如 `--gas-prices`、`--fees`）会抛出错误。
- `--gas=auto`：与 `--fees=auto` 的行为相同。
  如果使用其他与费用相关的标志（例如 `--gas-prices`、`--fees`）会抛出错误。
- `--gas={int}`：使用指定的 gas 数量和所需的费用进行交易。
- `--fees={int}{denom}`：使用指定的费用进行交易。使用默认的 gas 值（200000）进行交易。
- `--fees={int}{denom} --gas={int}`：使用指定的 gas 和费用。使用提供的参数计算 gas 价格。
- `--gas-prices={int}{denom}`：使用提供的 gas 价格和默认的 gas 数量（200000）。
- `--gas-prices={int}{denom} --gas={int}`：使用指定的 gas 进行交易，并使用相应的参数计算费用。

请注意，前两个选项为新用户提供了更友好的用户体验，而后两个选项则适用于更高级的用户，他们希望对这些参数有更多的控制。

团队引入了`auto`标志选项，它可以自动计算执行交易所需的燃气和费用。这样，新用户或开发者可以在不定义特定燃气和费用值的情况下执行交易。

使用`auto`标志有时可能无法根据网络流量准确估算出正确的燃气和费用。为了解决这个问题，您可以为`--gas-adjustment`标志使用更高的值。默认情况下，该值设置为`1.2`。当估算值不足时，请尝试使用更高的燃气调整值，例如`--gas-adjustment 1.3`。

无法同时使用`--gas-prices`和`--fees`标志。如果这样做，用户将收到一个错误，指出无法同时提供费用和燃气价格。

请记住，如果提供的费用或燃气数量不足，上述组合可能会失败。如果出现这种情况，CLI将返回一个带有具体原因的错误消息。例如：

```shell
raw_log: 'out of gas in location: submit proposal; gasWanted: 200000, gasUsed: 263940.
  Please retry with a gas (--gas flag) amount higher than gasUsed: out of gas'
```



---
sidebar_position: 4
---

# Gas and Fees

Users need to pay a fee to submit transactions on the Evmos network.
As fees are handled differently on Ethereum and Cosmos,
it is important to understand how the Evmos blockchain implements an Ethereum-type fee calculation,
that is compatible with the Cosmos SDK.

Therefore this overview explains the basics of gas calculation,
how to provide fees for transactions
and how the Ethereum-type fee calculation uses a fee market (EIP1559)
for prioritizing transactions.

Also, note the fees that are paid for interacting with smart contracts on Evmos
can earn smart contract deployers a revenue. For information on this,
head to [develop](./../../develop/mainnet#revenue).

## Prerequisite Readings

- [Cosmos SDK Gas](https://docs.cosmos.network/main/basics/gas-fees.html)
- [Ethereum Gas](https://ethereum.org/en/developers/docs/gas/)

## Basics

### Why do Transactions Need Fees?

If anyone can submit transactions to a network at no cost,
the network can be overrun by a handful of actors
sending large numbers of fraudulent transactions
to clog up the network and stop it from working.

The solution to this is a concept called “gas,"
which is a resource consumed throughout transaction execution.
In practice, a small amount of gas is spent on each step of code execution,
thus effectively charging for use of a validator’s resources
and preventing malicious actors from halting a network at will.

### What is Gas?

In general, gas is a unit that measures the computational intensity of a particular transaction
— in other words, how much work would be required to evaluate and perform the job.
Complex, multi-step transactions,
such as a Cosmos transaction that delegates to a dozen validators,
require more gas than simple, single-step transactions,
such as a Cosmos transaction to send tokens to another address.

When referring to a transaction,
“gas” refers to the total quantity of gas required for the transaction.
For example, a transaction may require 300,000 units of gas to be executed.

Gas can be thought of as electricity (kWh) within a house or factory,
or fuel for automobiles.
The idea is that it costs something to get somewhere.

More on Gas:

- [Cosmos Gas Fees](https://docs.cosmos.network/main/basics/gas-fees)
- [Cosmos Tx Lifecycle](https://docs.cosmos.network/main/basics/tx-lifecycle.html)
- [Ethereum Gas](https://ethereum.org/en/developers/docs/gas/)

### How is Gas Calculated?

In general, there’s no way to know exactly how much gas a transaction will cost
without simply running it.
Using the Cosmos SDK, this can be done by [simulating the Tx](https://docs.cosmos.network/main/run-node/txs#simulating-a-transaction).
Otherwise, there are ways to estimate the amount of gas a transaction will require,
based on the details of the transaction fields, and data.
In the case of the EVM, for example, each bytecode operation
has a [corresponding amount of gas](https://ethereum.org/en/developers/docs/evm/opcodes/).

More on Gas Calculations:

- [Estimate Gas](https://docs.ethers.org/v5/api/providers/provider/#Provider-estimateGas)
- [Executing EVM Bytecode](https://ethereum.org/en/developers/docs/evm/opcodes/)
- [Simulate a Cosmos SDK Tx](https://docs.cosmos.network/main/run-node/txs#simulating-a-transaction)

### How does Gas Relate to Fees?

While gas refers to the computational work required for execution,
fees refer to the amount of the tokens you actually spend to execute the transaction.
They are derived using the following formula:

```markdown
Total Fees = Gas * Gas Price (the price per unit of gas)
```

If “gas” was measured in kWh, the “gas price” would be the rate (in dollars per kWh)
determined by your energy provider,
and the “fees” would be your bill.
Just as with electricity, gas price is liable to fluctuate over a given day,
depending on network traffic.

More on Gas vs. Fees:

- [Cosmos Gas and Fees](https://docs.cosmos.network/main/basics/gas-fees)
- [Ethereum Gas and Fees](https://ethereum.org/en/developers/docs/gas/)

### How are Fees Handled on Cosmos?

Gas fees on Cosmos are relatively straightforward. As a user, you specify two fields:

1. A `GasLimit` corresponding to an upper bound on execution gas, defined as `GasWanted`
2. One of `Fees` or `GasPrice`, which will be used to specify or calculate the transaction fees

The node will entirely consume the fees provided, then begin to execute the transaction. If the `GasLimit` is found to
 be insufficient during execution, the transaction will fail and roll back any changes made, without refunding the fees provided.

Validators for Cosmos SDK-based chains can specify their `min-gas-prices` that they will enforce when selecting
transactions to include in blocks. Thus, transactions with insufficient fees will encounter delays or fail outright.

At the beginning of each block, fees from the previous block are
[allocated to validators and delegators](https://docs.cosmos.network/main/modules/distribution), after which they can
be withdrawn and spent.

### How are Fees Handled on Ethereum?

Fees on Ethereum include multiple implementations that were introduced over time.

Originally, a user would specify a `GasPrice` and `GasLimit` within a transaction—much like a Cosmos SDK transaction. A
block proposer would receive the entire gas fee from each transaction in the block, and they would select transactions
to include accordingly.

With proposal EIP-1559 and the London Hard fork, gas calculation changed. The `GasPrice` from above has now been split
into two separate components: a `BaseFee` and `PriorityFee`. The `BaseFee` is calculated automatically based on the
block size and is burned once the block is mined. The `PriorityFee` goes to the proposer and represents a tip, or an
incentive for a proposer to include the transaction in a block.

```markdown
Gas Price = Base Fee + Priority Fee
```

Within a transaction, users can specify a `max_fee_per_gas` corresponding to the total `GasPrice` and a
`max_priority_fee_per_gas` corresponding to a maximum `PriorityFee`, in addition to specifying the `gas_limit` as before.
 All surplus gas that was not required for execution is refunded to the user.

More on Ethereum Fees:

- [Gas Calculation Docs](https://ethereum.org/en/developers/docs/gas/)
- [Proposal EIP-1559](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-1559.md)

## Implementation

### How are Gas and Fees Handled on Evmos?

Fundamentally, Evmos is a Cosmos SDK chain that enables EVM compatibility as part of a Cosmos SDK module. As a result of
 this architecture, all EVM transactions are ultimately encoded as Cosmos SDK transactions and update a Cosmos
 SDK-managed state.

Since all transactions are represented as Cosmos SDK transactions, transaction fees can be treated identically across
execution layers. In practice, dealing with fees includes standard Cosmos SDK logic, some Ethereum logic, and custom
Evmos logic. For the most part, fees are collected by the `fee_collector` module, then paid out to validators and
delegators. A few key distinctions are as follows:

1. Fee Market Module

    In order to support EIP-1559 gas and fee calculation on Evmos’ EVM layer, Evmos tracks the gas supplied for each
    block and uses that to calculate a base fee for future EVM transactions, thus enabling EVM dynamic fees and
    transaction prioritization as specified by EIP-1559.

    For EVM transactions, each node bypasses their local `min-gas-prices` configuration, and instead applies EIP-1559 fee
     logic—the gas price simply must be greater than both the global `min-gas-price` and the block's `BaseFee`, and the
     surplus is considered a priority tip. This allows validators to compute Ethereum fees without applying Cosmos SDK
     fee logic.

    Unlike on Ethereum, the `BaseFee` on Evmos is not burned, and instead is distributed to validators and delegators.
    Furthermore, the `BaseFee` is lower-bounded by the global `min-gas-price` (currently, the global `min-gas-price`
    parameter is set to zero, although it can be updated via Governance).

2. EVM Gas Refunds

    Evmos refunds a fraction (at least 50% by default) of the unused gas for EVM transactions to approximate the current
    behavior on Ethereum. [Why not always 100%?](https://github.com/evmos/ethermint/issues/1085)

3. Revenue Module

    Evmos developed the Revenue Module as a way to reward developers for creating useful dApps—any contract that is
    registered with Evmos’ Revenue Module rewards a fraction of the transaction fee (currently 95%) from each transaction
     that interacts with the contract to the contract developer. Validators and delegators earn the remaining portion.

### Detailed Timeline

1. Nodes execute the previous block and run the `EndBlock` hook
    * As part of this hook, the FeeMarket (EIP-1559) module tracks the total `TransientGasWanted` from the transactions
    on this block. This will be used for the next block’s `BaseFee`.
2. Nodes receive transactions for a subsequent block and gossip these transactions to peers
    * These can be sorted and prioritized by the included fee price (using EIP-1559 fee priority mechanics for EVM
    transactions - [code snippet](https://github.com/evmos/ethermint/blob/57ed355c985d9f3116aba6aabfa2ee0f3f38e966/app/ante/eth.go#L137)),
     to be included in the next block
3. Nodes run `BeginBlock` for the subsequent block
    * The FeeMarket module calculates the `BaseFee` ([code snippet](https://github.com/evmos/ethermint/blob/89fdd1984826ea524cb9b8feb089a99b6cfe8ace/x/feemarket/keeper/abci.go#L14))
     to be applied for this block using the total `GasWanted` from the previous block.
    * The Distribution module [distributes](https://docs.cosmos.network/main/modules/distribution#begin-block) the
    previous block’s fee rewards to validators and delegators
4. For each valid transaction that will be included in this block, nodes perform the following:
    * They run an `AnteHandler` corresponding to the transaction type. This process:
        1. Performs basic transaction validation
        2. Verifies the fees provided are greater than the global and local minimum validator values *and* greater than
        the `BaseFee` calculated
        3. (For Ethereum transactions) Preemptively consumes gas for the EVM transaction
        4. Deducts the transaction fees from the user and transfers them to the `fee_collector` module
        5. Increments the `TransientGasWanted` in the current block, to be used to calculate the next block’s `BaseFee`
    * Then, for standard Cosmos Transactions, nodes:
        1. Execute the transaction and update the state
        2. Consume gas for the transaction
    * For Ethereum Transactions, nodes:
        1. Execute the transaction and update the state
        2. Calculate the gas used and compare it to the gas supplied, then refund a designated portion of the surplus
        3. Send a fraction of the fees used as revenue to contract developers as part of the Revenue Module, if the
        transaction interacted with a registered smart contract
5. Nodes run `EndBlock` for this block and store the block’s `GasWanted`

## Detailed Mechanics

### Cosmos `Gas`

In the Cosmos SDK, gas is tracked in the main `GasMeter` and the `BlockGasMeter`:

- `GasMeter`: keeps track of the gas consumed during executions that lead to state transitions. It is reset on every
transaction execution.
- `BlockGasMeter`: keeps track of the gas consumed in a block and enforces that the gas does not go over a predefined
 limit. This limit is defined in the Tendermint consensus parameters and can be changed via governance parameter change proposals.

Since gas is priced per-byte, the same interaction is more gas-intensive with larger parameter values than smaller
(unlike Ethereum's `uint256` values, Cosmos SDK numericals are represented using [Big.Int](https://pkg.go.dev/math/big#Int)
 types, which are dynamically sized).

More information regarding gas as part of the Cosmos SDK can be found [here](https://docs.cosmos.network/main/basics/gas-fees.html).

### Matching EVM Gas consumption

Evmos is an EVM-compatible chain that supports Ethereum Web3 tooling. For this reason, gas consumption must be equatable
 with other EVMs, most importantly Ethereum.

The main difference between EVM and Cosmos state transitions, is that the EVM uses a [gas table](https://github.com/ethereum/go-ethereum/blob/master/params/protocol_params.go)
 for each OPCODE, whereas Cosmos uses a `GasConfig` that charges gas for each CRUD operation by setting a flat and
 per-byte cost for accessing the database.

+++ https://github.com/cosmos/cosmos-sdk/blob/3fd376bd5659f076a4dc79b644573299fd1ec1bf/store/types/gas.go#L187-L196

In order to match the gas consumed by the EVM, the gas consumption logic from the SDK is ignored, and instead the gas consumed
 is calculated by subtracting the state transition leftover gas plus refund from the gas limit defined on the message.

To ignore the SDK gas consumption, we reset the transaction `GasMeter` count to 0 and manually set it to the `gasUsed`
 value computed by the EVM module at the end of the execution.

+++ https://github.com/evmos/ethermint/blob/098da6d0cc0e0c4cefbddf632df1057383973e4a/x/evm/keeper/state_transition.go#L188

### `AnteHandler`

The Cosmos SDK [`AnteHandler`](https://docs.cosmos.network/main/basics/gas-fees.html#antehandler)
performs basic checks prior to transaction execution. These checks are usually signature
verification, transaction field validation, transaction fees, etc.

Regarding gas consumption and fees, the `AnteHandler` checks that the user has enough balance to
cover for the tx cost (amount plus fees) as well as checking that the gas limit defined in the
message is greater or equal than the computed intrinsic gas for the message.

### Gas Refunds

In the EVM, gas can be specified prior to execution. The totality of the gas specified is consumed at the beginning of
the execution (during the `AnteHandler` step) and the remaining gas is refunded back to the user if any gas is left over
 after the execution. Additionally the EVM can also define gas to be refunded back to the user but those will be capped
  to a fraction of the used gas depending on the fork/version being used.

### Zero-Fee Transactions

In Cosmos, a minimum gas price is not enforced by the `AnteHandler` as the `min-gas-prices` is
checked against the local node/validator. In other words, the minimum fees accepted are determined
by the validators of the network, and each validator can specify a different minimum value for their fees.
This potentially allows end users to submit 0 fee transactions if there is at least one single
validator that is willing to include transactions with `0` gas price in their blocks proposed.

For this same reason, in Evmos it is possible to send transactions with `0` fees for transaction
types other than the ones defined by the `evm` module. EVM module transactions cannot have `0` fees
as gas is required inherently by the EVM. This check is done by the EVM transactions stateless validation
(i.e `ValidateBasic`) function as well as on the custom `AnteHandler` defined by Evmos.

### Gas Estimation

Ethereum provides a JSON-RPC endpoint `eth_estimateGas` to help users set up a correct gas limit in their transactions.

For that reason, a specific query API `EstimateGas` is implemented in Evmos. It will apply the transaction against the
current block/state and perform a binary search in order to find the optimal gas value to return to the user (the same
transaction will be applied over and over until we find the minimum gas needed before it fails). The reason we need to
 use a binary search is that the gas required for the
transaction might be higher than the value returned by the EVM after applying the transaction, so we need to try until
we find the optimal value.

A cache context will be used during the whole execution to avoid changes be persisted in the state.

+++ https://github.com/evmos/ethermint/blob/098da6d0cc0e0c4cefbddf632df1057383973e4a/x/evm/keeper/grpc_query.go#L100

For Cosmos Tx's, developers can use Cosmos SDK's [transaction simulation](https://docs.cosmos.network/main/run-node/txs#simulating-a-transaction)
 to create an accurate estimate.

### Cross-Chain Gas and Fees

Let’s say a user transfers tokens from Chain A to Evmos via IBC-transfer and wants to execute an Evmos transaction—however,
 they don’t have any Evmos tokens to cover fees. The Cosmos SDK introduced `Tips` as a solution to this issue; a user can
  cover fees using a different token—in this case, tokens from Chain A.

To cover transaction fees using a tip, this user can sign a transaction with a tip and no fees, then send the transaction
 to a fee relayer. The fee relayer will then cover the fee in the native currency (Evmos in this case), and receive the
  tip in payment, behaving as an intermediary exchange.

More on Cosmos Tips:

- [Cosmos Tips Docs](https://docs.cosmos.network/main/core/tips)

## Dealing with gas and fees with the Evmos CLI

When broadcasting a transaction using the Evmos CLI client, users should keep into consideration the options available.
There are three flags to consider when sending a transaction to the network:

- `--fees`: fees to pay along with transaction; eg: 10aevmos. Defaults to the required fees.
- `--gas`: the gas limit to set per-transaction; the default value is 200000.
- `--gas-prices`: gas prices to determine the transaction fee (e.g. 10aevmos).

However, not all of them need to be defined on each transaction.
The correct combinations are:

- `--fees=auto`: estimates fees and gas automatically (same behavior as `--gas=auto`).
  Throws an error if using any other fees-related flag (e.i, `--gas-prices` , `--fees`)
- `--gas=auto`: same behavior as `--fees=auto`.
  Throws an error if using any other fees-related flag (e.i, `--gas-prices` , `--fees`)
- `--gas={int}`: uses the specified gas amount and the required fees for the transaction
- `--fees={int}{denom}`: uses the specified fees for the tx. Uses gas default value (200000) for the tx.
- `--fees={int}{denom} --gas={int}`: uses specified gas and fees. Calculates gas-prices with the provided params
- `--gas-prices={int}{denom}`: uses the provided gas price and the default gas amount (200000)
- `--gas-prices={int}{denom} --gas={int}`: uses the gas specified on for the tx and calculates
  the fee with the corresponding parameters.

The reader should note that the former two options provide a frendlier user experience for new users,
and the latter are for more advanced users, who desire more control over these parameters.

The team introduced the `auto` flag option that calculates automatically the gas and fees required to execute a transaction.
In this way, new users or developers can perform transactions without the hustle of defining specific gas and
fees values.

Using the `auto` flag sometimes may fail on estimating the right gas and fees based on network traffic.
To overcome this, you can use a higher value for the `--gas-adjustment` flag.
By default, this is set to `1.2`.
When the estimated values are insufficient,
retry with a higher gas adjustment, for example, `--gas-adjustment 1.3`.

It is not possible to use the `--gas-prices` and `--fees` flags combined.
If so, the user will get an error stating that cannot provide both fees and gas prices.

Keep in mind that the above combinations may fail if the provided fees or gas amount is insufficient.
If that is the case, the CLI will return an error message with the specific reason. For example:

```shell
raw_log: 'out of gas in location: submit proposal; gasWanted: 200000, gasUsed: 263940.
  Please retry with a gas (--gas flag) amount higher than gasUsed: out of gas'
```
