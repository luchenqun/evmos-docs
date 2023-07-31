# 代币

## EVMOS 代币

在 EVM 上，用于质押、治理和燃气消耗的代币是 EVMOS。EVMOS 提供以下功能：
- 保护权益证明链
- 用于治理提案的代币
- 将费用分配给验证者和用户
- 作为在 EVM 上运行智能合约的燃气

Evmos 使用 [Atto](https://en.wikipedia.org/wiki/Atto-) EVMOS 作为基本单位，以与以太坊保持一致。有三种类型的资产：

- 本地的 Evmos 代币
- IBC 代币（通过 IBC）
- 以太坊类型的代币，例如 ERC-20

`1 evmos = 10<sup>18</sup> aevmos`

这与以太坊的单位相匹配：

`1 ETH = 10<sup>18</sup> wei`

## Cosmos 代币

账户可以拥有 Cosmos 代币，用于与其他 Cosmos 和交易进行操作。例如，可以使用这些代币进行质押、IBC 转账、治理存款和 EVM 操作。

## EVM 代币

Evmos 兼容 ERC20 代币和其他非同质化代币标准（EIP721、EIP1155），这些标准是 EVM 原生支持的。

## Evmos 资产页面

通过 Evmos 仪表板上的 [单一代币表示](https://app.evmos.org/assets) 功能，我们展示了 ERC-20 代币和 Cosmos IBC 代币的表示方式。Evmos 启用了此功能，以帮助用户简化资产类型，并让他们专注于交互。该协议通过处理转换来简化流程，用户可以获得资产的简化表示和数量。列出的代币仍然需要进行治理和注册（[Cosmos](https://academy.evmos.org/articles/advanced/cosmos-coin-registration) 或 [ERC-20](https://academy.evmos.org/articles/advanced/erc20-registration)）。

![evmos-dashboard-assets])(/img/dashboard-assets.png)

有关我们如何处理代币注册的更多信息，请访问[此处](./../../develop/mainnet#token-registration)。


---
sidebar_position: 11
---

# Tokens

## The EVMOS Token

The denomination used for staking, governance and gas consumption on the EVM is the EVMOS. The EVMOS provides the utility
of: securing the Proof-of-Stake chain, token used for governance proposals, distribution of fees to validator and users,
and as a mean of gas for running smart contracts on the EVM.

Evmos uses [Atto](https://en.wikipedia.org/wiki/Atto-) EVMOS as the base denomination to maintain parity with Ethereum.
There are three types of assets:

- The native Evmos token
- IBC Coins (via the IBC)
- Ethereum-typed tokens, e.g. ERC-20

`1 evmos = 10<sup>18</sup> aevmos`

This matches Ethereum denomination of:

`1 ETH = 10<sup>18</sup> wei`

## Cosmos Coins

Accounts can own Cosmos coins in their balance, which are used for operations with other Cosmos and transactions. Examples
of these are using the coins for staking, IBC transfers, governance deposits and EVM.

## EVM Tokens

Evmos is compatible with ERC20 tokens and other non-fungible token standards (EIP721, EIP1155)
that are natively supported by the EVM.

## Evmos Assets Page

Check out how we represent ERC-20 tokens and Cosmos IBC Coins through our [Single Token Representation](https://app.evmos.org/assets)
feature on the Evmos Dashboard. Evmos enabled this feature to help with the user experiences by obfuscating the assets
types away from the users and allow them to focus on the interaction. The protocol simplifies the process by handling the
conversion and users are given the simplification of denomination and the amount of assets they hold. Tokens listed still
require governance and registering the tokens ([Cosmos](https://academy.evmos.org/articles/advanced/cosmos-coin-registration)
or [ERC-20](https://academy.evmos.org/articles/advanced/erc20-registration)).

![evmos-dashboard-assets])(/img/dashboard-assets.png)

For more information on how we handle token registration, head over [here](./../../develop/mainnet#token-registration).
