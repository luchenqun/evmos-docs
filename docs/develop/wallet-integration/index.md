# 钱包集成

钱包集成是dApp开发的重要方面，它允许用户安全地与基于区块链的应用程序进行交互。以下是关于dApp开发中钱包集成的一些关键要点，这些要点来自各种来源：

- dApp开发者的集成实施清单包括三个类别：前端功能、交易和钱包交互等。启用dApp上的交易时，开发者需要确定用户的钱包类型，创建交易，从相应的钱包请求签名，最后进行广播。

- 使用Evmos与Keplr、Metamask、Ledger、WalletConnect等钱包进行集成。最新的钱包位于[这里](./../../../use/wallet)。

- 前往我们的[Evmos客户端集成](./../../develop/tools/client-integrations)页面，以利用我们的TypeScript或Python库。

## Gas和估算

在Evmos上开发和运行dApps时，钱包配置将尝试计算用户签名所需的正确的燃气数量。[燃气和费用](./../../../protocol/concepts/gas-and-fees)详细解释了这些概念。我们有一个名为[feemarket](./../../../protocol/modules/feemarket#concepts)的模块，描述了我们在Cosmos SDK 0.46之前没有此类实现的事务优先级模块的实现方式。


# Wallet Integration

Wallet integration is an essential aspect of dApp development that allows users to securely interact with blockchain-based
applications. Here are some key points from various sources on wallet integration in dApp development:

- The integration implementation checklist for dApp developers consists of three categories: frontend features,
transactions and wallet interactions, and more. Developers enabling transactions on their dApp have to determine
the wallet type of the user, create the transaction, request signatures from the corresponding wallet, and finally broadcast.

- Leverage Keplr, Metamask, Ledger, WalletConnect and more with Evmos. The latest wallets are located [here](./../../../use/wallet).

- Head over to our [Evmos Client Integrations](./../../develop/tools/client-integrations)
  to leverage our Typescript or Python libraries.

## Gas & Estimation

When developing and running dApps on Evmos, the wallet configuration will attempt to calculate the correct gas amount
for user's to sign. [Gas and Fees](./../../../protocol/concepts/gas-and-fees) breaks down these concepts in more detail.
We have a module called [feemarket](./../../../protocol/modules/feemarket#concepts) that describes our module implementation
of transaction prioritization since prior to Cosmos SDK 0.46 it did not have such implementation.
