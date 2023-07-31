---
sidebar_position: 6
---

# 指标

Evmos节点可以启用[Cosmos SDK遥测](https://docs.cosmos.network/main/core/telemetry.html)，
以便观察和收集有关Evmos应用程序的见解。
在内部，它使用[`go-metrics`](https://github.com/hashicorp/go-metrics)包
和Prometheus客户端库来公开不同类型的指标，如仪表和计数器。
有关如何使用不同类型的指标的最佳实践，请查看此[博客文章](https://blog.pvincent.io/2017/12/prometheus-blog-series-part-2-metric-types/)。

下面是支持的Evmos模块及其自定义指标和遥测的列表。
使用这些指标，您可以运行性能概要文件
并在[Grafana](https://grafana.com/)仪表板中显示它们。

## 支持的指标

| 指标                                          | 描述                                                                               | 单位        | 类型    |
| :--------------------------------------------- | :---------------------------------------------------------------------------------- | :---------- | :------ |
| `feemarket_base_fee`                           | 每个EIP-1559块的基础费用金额                                                       | 代币        | 仪表    |
| `feemarket_block_gas`                          | 在EIP-1559块中使用的气体量                                                         | 代币        | 仪表    |
| `recovery_ibc_on_recv_total`                   | 使用ibc `onRecvPacket`回调函数进行恢复的总次数                                      | 恢复    | 计数器 |
| `recovery_ibc_on_recv_token_total`             | 使用ibc `onRecvPacket`回调函数恢复的代币总量                                        | 代币        | 计数器 |
| `tx_msg_convert_coin_amount_total`             | 使用`ConvertCoin`消息转换的代币总量                                                | 代币        | 计数器 |
| `tx_msg_convert_coin_total`                    | 具有`ConvertCoin`消息的交易总数                                                     | 交易        | 计数器 |
| `tx_msg_convert_erc20_amount_total`            | 使用`ConvertERC20`消息转换的erc20代币总量                                           | 代币        | 计数器 |
| `tx_msg_convert_erc20_total`                   | 具有`ConvertERC20`消息的交易总数                                                    | 交易        | 计数器 |
| `tx_msg_ethereum_tx_total`                     | 通过EVM处理的交易总数                                                               | 交易        | 计数器 |
| `tx_msg_ethereum_tx_gas_used_total`            | 以太坊交易使用的总气体量                                                           | 代币        | 计数器 |
| `tx_msg_ethereum_tx_gas_limit_per_gas_used`    | 以太坊交易的气体限制与使用的气体量的比率                                           | 比率        | 仪表    |
| `tx_msg_ethereum_tx_incentives_total`          | 通过EVM处理的具有激励合约的交易总数                                                 | 交易        | 计数器 |
| `tx_msg_ethereum_tx_incentives_gas_used_total` | 通过EVM处理的具有激励合约的交易使用的总气体量                                       | 代币        | 计数器 |
| `incentives_distribute_reward_total`           | 分发给所有激励参与者的奖励总量                                                     | 代币        | 计数器 |
| `inflation_allocate_total`                     | 通过通胀分配的代币总量                                                             | 代币        | 计数器 |
| `inflation_allocate_staking_total`             | 通过通胀将代币分配给质押总量                                                       | 代币        | 计数器 |
| `inflation_allocate_incentives_total`          | 通过通胀将代币分配给激励总量                                                       | 代币        | 计数器 |
| `inflation_allocate_community_pool_total`      | 通过通胀将代币分配给社区池总量                                                     | 代币        | 计数器 |

I'm sorry, but as an AI text-based model, I am unable to receive or process any files or attachments. However, you can copy and paste the Markdown content here, and I will do my best to translate it for you.


---
sidebar_position: 6
---

# Metrics

Evmos nodes can enable [Cosmos SDK telemetry](https://docs.cosmos.network/main/core/telemetry.html)
to allow for observing and gathering insights about the Evmos application.
Under the hood, it uses the [`go-metrics`](https://github.com/hashicorp/go-metrics) package
and the Prometheus client library to expose different [types of metrics](https://prometheus.io/docs/concepts/metric_types/)
like gauges and counters.
For best practices on how to use different metrics types,
check this [blog article](https://blog.pvincent.io/2017/12/prometheus-blog-series-part-2-metric-types/).

Find below a list of supported Evmos modules with custom metrics and telemetry.
Using the metrics you can e.g. run performance profiles
and display them in a [Grafana](https://grafana.com/) dashboard.

## Supported Metrics

| Metric                                         | Description                                                                         | Unit        | Type    |
| :--------------------------------------------- | :---------------------------------------------------------------------------------- | :---------- | :------ |
| `feemarket_base_fee`                           | Amount of base fee per EIP-1559 block                                               | token       | gauge   |
| `feemarket_block_gas`                          | Amount of gas used in an EIP-1559 block                                             | token       | gauge   |
| `recovery_ibc_on_recv_total`                   | Total number of recoveries using the ibc `onRecvPacket` callback                    | recovery    | counter |
| `recovery_ibc_on_recv_token_total`             | Total amount of tokens recovered using the ibc `onRecvPacket` callback              | token       | counter |
| `tx_msg_convert_coin_amount_total`             | Total amount of converted coins using a `ConvertCoin` msg                           | token       | counter |
| `tx_msg_convert_coin_total`                    | Total number of txs with a `ConvertCoin` msg                                        | tx          | counter |
| `tx_msg_convert_erc20_amount_total`            | Total amount of converted erc20 using a `ConvertERC20` msg                          | token       | counter |
| `tx_msg_convert_erc20_total`                   | Total number of txs with a `ConvertERC20` msg                                       | tx          | counter |
| `tx_msg_ethereum_tx_total`                     | Total number of txs processed via the EVM                                           | tx          | counter |
| `tx_msg_ethereum_tx_gas_used_total`            | Total amount of gas used by an ethereum tx                                          | token       | counter |
| `tx_msg_ethereum_tx_gas_limit_per_gas_used`    | Ratio of gas limit to gas used for an ethereum tx                                   | ratio       | gauge   |
| `tx_msg_ethereum_tx_incentives_total`          | Total number of txs with an incentivized contract processed via the EVM             | tx          | counter |
| `tx_msg_ethereum_tx_incentives_gas_used_total` | Total amount of gas used by txs with an incentivized contract processed via the EVM | token       | counter |
| `incentives_distribute_reward_total`           | Total amount of rewards that are distributed to all incentives' participants        | token       | counter |
| `inflation_allocate_total`                     | Total amount of tokens allocated through inflation                                  | token       | counter |
| `inflation_allocate_staking_total`             | Total amount of tokens allocated through inflation to staking                       | token       | counter |
| `inflation_allocate_incentives_total`          | Total amount of tokens allocated through inflation to incentives                    | token       | counter |
| `inflation_allocate_community_pool_total`      | Total amount of tokens allocated through inflation to community pool                | token       | counter |
