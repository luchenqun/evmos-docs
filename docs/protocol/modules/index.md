# 模块列表

以下是可以在 Evmos 区块链上使用的所有生产级模块，以及它们各自的文档：

- [claims](claims.md) - 主网发布的奖励状态和认领流程。
- [epochs](epochs.md) - 每个周期（也称为时期）执行自定义状态转换。
- [erc20](erc20.md) - 在 Evmos 的 EVM 和 Cosmos 运行时之间进行可信、链上的双向内部代币转换。
- [evm](evm.md) - 在 Cosmos 上部署和执行智能合约。
- [feemarket](feemarket.md) - 基于 EIP1559 规范的费用市场实现。
- [revenue](revenue.md) - 将 EVM 交易费用分配给区块提议者和智能合约开发者。
- [incentives](incentives.md) - 激励用户与经过治理批准的智能合约进行交互。
- [inflation](inflation.md) - 铸造代币并将其分配给质押奖励、使用激励和社区资金池。
- [recovery](recovery.md) - 恢复在不支持的 Evmos 账户上被卡住的代币。
- [vesting](vesting.md) - 具有锁定和收回功能的解锁账户。

## Cosmos SDK

Evmos 使用以下 Cosmos SDK 模块：

- [auth](https://docs.cosmos.network/main/modules/auth) - 用于 Cosmos SDK 应用程序的账户和交易认证。
- [authz](https://docs.cosmos.network/main/modules/authz) - 允许账户代表其他账户执行操作的授权。
- [bank](https://docs.cosmos.network/main/modules/bank) - 代币转账功能。
- [capability](https://docs.cosmos.network/v0.47/modules/capability) - 对象能力实现。
- [crisis](https://docs.cosmos.network/main/modules/crisis) - 在特定情况下（例如，不变量被破坏）停止区块链。
- [distribution](https://docs.cosmos.network/main/modules/distribution) - 费用分配和质押代币供应分配。
- [evidence](https://docs.cosmos.network/main/modules/evidence) - 处理双重签名、恶意行为等证据。
- [feegrant](https://docs.cosmos.network/main/modules/feegrant) - 授予执行交易的费用津贴。
- [genutil](https://github.com/cosmos/cosmos-sdk/tree/main/x/genutil) - 用于区块链应用程序内部使用的各种创世工具功能。
- [gov](https://docs.cosmos.network/main/modules/gov) - 链上提案和投票。
- [params](https://docs.cosmos.network/main/modules/params) - 全局可用的参数存储。
- [slashing](https://docs.cosmos.network/main/modules/slashing) - 验证人惩罚机制。
- [staking](https://docs.cosmos.network/main/modules/staking) - 公共区块链的权益证明层。
- [upgrade](https://docs.cosmos.network/main/modules/upgrade) - 软件升级处理和协调。

## IBC

Evmos在SDK中使用以下IBC模块：

- [interchain-accounts](https://ibc.cosmos.network/main/apps/interchain-accounts/overview.html)
- [transfer](https://ibc.cosmos.network/main/apps/transfer/overview.html)


---
sidebar_position: 3
---

# List of Modules

Here is a list of all production-grade modules that can be used on the Evmos blockchain, along with their respective documentation:

- [claims](claims.md) - Rewards status and claiming process for the mainnet release.
- [epochs](epochs.md) - Executes custom state transitions every period (*aka* epoch).
- [erc20](erc20.md) - Trustless, on-chain bidirectional internal conversion of tokens
  between Evmos' EVM and Cosmos runtimes.
- [evm](evm.md) - Smart Contract deployment and execution on Cosmos
- [feemarket](feemarket.md) - Fee market implementation based on the EIP1559 specification.
- [revenue](revenue.md) - Split EVM transaction fees between block proposer and smart contract developers.
- [incentives](incentives.md) - Incentivize user interaction with governance-approved smart contracts.
- [inflation](inflation.md) - Mint tokens and allocate them to staking rewards,
  usage incentives and community pool.
- [recovery](recovery.md) - Recover tokens that are stuck on unsupported Evmos accounts.  
- [vesting](vesting.md) - Vesting accounts with lockup and clawback capabilities.

## Cosmos SDK

Evmos uses the following Cosmos SDK modules:

- [auth](https://docs.cosmos.network/main/modules/auth) - Authentication of accounts and transactions for Cosmos SDK applications.
- [authz](https://docs.cosmos.network/main/modules/authz) - Authorization for accounts to perform actions on behalf of other accounts.
- [bank](https://docs.cosmos.network/main/modules/bank) - Token transfer functionalities.
- [capability](https://docs.cosmos.network/v0.47/modules/capability) - Object capability implementation.
- [crisis](https://docs.cosmos.network/main/modules/crisis) - Halting the blockchain under certain circumstances (e.g. if an invariant is broken).
- [distribution](https://docs.cosmos.network/main/modules/distribution) - Fee distribution, and staking token provision distribution.
- [evidence](https://docs.cosmos.network/main/modules/evidence) - Evidence handling for double signing, misbehaviour, etc.
- [feegrant](https://docs.cosmos.network/main/modules/feegrant) - Grant fee allowances for executing transactions.
- [genutil](https://github.com/cosmos/cosmos-sdk/tree/main/x/genutil) - variaety of genesis utility functionalities for usage within a blockchain application
- [gov](https://docs.cosmos.network/main/modules/gov) - On-chain proposals and voting.
- [params](https://docs.cosmos.network/main/modules/params) - Globally available parameter store.
- [slashing](https://docs.cosmos.network/main/modules/slashing) - Validator punishment mechanisms.
- [staking](https://docs.cosmos.network/main/modules/staking) - Proof-of-Stake layer for public blockchains.
- [upgrade](https://docs.cosmos.network/main/modules/upgrade) - Software upgrades handling and coordination.

## IBC

Evmos uses the following the IBC modules for the SDK:

- [interchain-accounts](https://ibc.cosmos.network/main/apps/interchain-accounts/overview.html)
- [transfer](https://ibc.cosmos.network/main/apps/transfer/overview.html)