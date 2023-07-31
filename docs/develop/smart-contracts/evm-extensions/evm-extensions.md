# EVM 扩展

核心协议上的有状态 EVM 扩展允许 dApps 和用户访问 EVM 之外的逻辑。
作为一个网关，这些 EVM 扩展定义了智能合约如何执行跨链交易（通过 IBC），
以及如何与 Evmos 链上的核心功能（如质押、投票）进行交互。

:::tip
**注意**：不清楚什么是 EVM 扩展吗？
EVM 扩展的行为类似于在 EVM 中编译和部署的智能合约。
如果您熟悉 EVM，您可能知道它们被称为 Precompiles。
它们具有预定义的地址，并且根据它们的逻辑可以被归类为有状态或无状态。
当它们改变链的状态（交易）或访问状态数据（查询）时，扩展被认为是“有状态”的；
当它们不改变状态时，它们是“无状态”的。
:::

## EVM 扩展文档

在本节中，您将找到当前已实现的 EVM 扩展的概述，包括交易、查询和使用示例：

- [授权接口（如果您对 EVM 扩展不熟悉，请先阅读）](./authorization.md)
- [EVM 扩展共享类型](./types.md)
- [`x/staking` 模块的 EVM 扩展](./staking.md)
- [`x/distribution` 模块的 EVM 扩展](./distribution.md)
- [`ibc/transfer` 模块的 EVM 扩展](./ibc-transfer.md)

:::tip
**注意**：在[Evmos 扩展仓库](https://github.com/evmos/extensions)中找到 EVM 扩展的 Solidity 接口和示例。
:::

## 其他学习资源

- [EVM 扩展 - 质押和分发](https://academy.evmos.org/articles/advanced/evm-extensions-stk-distr)
  学院文章
- [深入 EVM 扩展研讨会（DoraHacks 黑客马拉松）](https://www.youtube.com/live/pJhOfZ0ScAE?feature=share)


# EVM Extensions

Stateful EVM Extensions on the core protocol allow dApps and users to access logic outside of the EVM.
Acting as a gateway, these EVM Extensions define how smart contracts can perform cross-chain transactions
(via IBC) and interact with core functionalities on the Evmos chain (e.g. staking, voting) from the EVM.

:::tip
**Note**: Not sure what EVM extensions are?
EVM extensions behave like smart contracts that are compiled and deployed within the EVM.
If you are familiar with the EVM, you may know them as Precompiles.
These have predefined addresses and, according to their logic, can be classified as stateful or stateless.
When they change the state of the chain (transactions)
or access state data (queries), extensions are considered "stateful";
when they don't, they're "stateless".
:::

## EVM Extensions documentation

Find in this section an outline of the currently implemented EVM extensions with transactions,
queries, and examples of using them:

- [Authorization interface (read first if you're new to EVM extensions)](./authorization.md)
- [EVM Extensions shared types](./types.md)
- [`x/staking` module EVM extension](./staking.md)
- [`x/distribution` module EVM extension](./distribution.md)
- [`ibc/transfer` module EVM extension](./ibc-transfer.md)

:::tip
**Note**: Find the EVM Extensions Solidity interfaces and examples in the [Evmos Extensions repo](https://github.com/evmos/extensions).
:::

## Other Learning Resources

- [EVM Extensions - Staking & Distribution](https://academy.evmos.org/articles/advanced/evm-extensions-stk-distr)
  academy article
- [Diving into EVM Extensions Workshop (DoraHacks Hackathon)](https://www.youtube.com/live/pJhOfZ0ScAE?feature=share)
