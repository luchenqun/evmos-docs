---
sidebar_position: 8
---

# 主网

在真实用户开始在您的 dApp 上使用实际资金进行交易之前，将其部署到主网时需要考虑一些因素。尽管将您的 dApp 从测试网迁移到主网可能是一个简单的过程，只需将合约部署到主网网络，但确保成功并提高您的 dApp 的曝光度可能需要额外的业务发展工作。以下是一些需要考虑的关键因素的概述。

:::note
Evmos 验证者 Stakely.io 运行一个[水龙头](https://stakely.io/en/faucet/evmos-evm)，供开发者申请测试币。
:::

## 部署

您可以使用 [JSON-RPC](../develop/smart-contracts#deploy-with-ethereum-json-rpc) 在主网上部署您的合约。这与在测试网上的过程相同，只是将目标指向[主网网络端点](./../develop/api/networks)。在这样做之前，请考虑以下几点。

### 安全性

在 Evmos 测试网上对您的智能合约进行彻底测试，以确保其按预期运行。全面的测试应该确保所有功能、错误处理和边界情况都得到覆盖。

在可能的情况下，对智能合约代码进行全面的安全审计，以识别和消除任何潜在的漏洞或弱点。这对于主网尤为重要，因为代码将对所有人可见，任何安全漏洞可能导致巨大的损失。了解常见的漏洞和合约安全实践。外部审计员也可以帮助优化您的合约性能。

确保正确管理合约所有权，并考虑实施多重签名机制以增加安全性。这将使您能够在团队内保持部署所有权，而不是一个可能离开团队的特定所有者。

最后但同样重要的是，验证合约使用的任何外部库或依赖项是否是最新且安全的。

### 合约可升级性

考虑将来升级合约的可能性，并在需要时实施升级机制。这将使您能够对合约进行更改，而无需重新部署它，创建一个新的合约。

### 成本

评估与部署和执行智能合约相关的燃气成本，包括部署合约和执行其功能的成本是否足够或高效，以满足最终用户的需求。

### 合约文档

为合约提供清晰全面的文档，包括其目的、功能和潜在风险。这将帮助用户了解如何使用合约，并使其他开发人员更容易审查和贡献代码。

## 代币分发

您的 dApp 可能会发行一个 ERC-20 代币，例如为代币持有人提供额外的福利。在这种情况下，您需要决定如何分发这些代币以及您想要创建的叙述方式。

### 空投

一种选择是通过空投将代币分发给用户。如果您想了解如何选择空投的合格接收者，请参考[Evmos Rektdrop](https://medium.com/evmos/the-evmos-rektdrop-abbe931ba823)。

### 代币注册

Evmos 允许 ERC-20 代币在跨链上使用。一旦您的一些代币被铸造出来，您可以通过治理注册一个代币对，这将允许用户在不同链上发送您的代币。请前往我们的学院，了解如何[注册您的 ERC-20 代币](https://academy.evmos.org/articles/advanced/erc20-registration)。

## 收入

在 Evmos 上，每当用户与您的 dApp 交互时，您都可以产生收入，为您带来稳定的收入。开发人员可以注册他们的智能合约，每当有人与注册的智能合约进行交互时，合约部署者或其指定的提款账户将获得一部分交易费用。

要了解如何[注册您的智能合约](https://academy.evmos.org/articles/advanced/incentives-registration)，请前往我们的学院。如果您想了解协议上如何实现这一点，请查看[收入模块规范](./../protocol/modules/incentives)。

## 社区

在 Evmos 社区中发出声音，解释您的 dApp 提供了什么价值。构建 dApp 的一个重要部分是与社区联系，展示他们如何参与或开始为您的项目做出贡献。这不仅有助于提高您的 dApp 的可见性，还可能导致一个新的用户社区，他们希望改进您的 dApp。

请前往[Evmos Discord](https://discord.gg/evmos)频道与社区和贡献者联系，并在下一次社区通话中展示您的dApp。我们还有一个[Telegram群组](https://t.me/EvmosBuilders)供我们的开发者使用。


---
sidebar_position: 8
---

# Mainnet

Before real users begin to transact with actual funds on your dApp, it is important to take into account certain factors
when launching it on Mainnet. Although moving your dApp from Testnet to Mainnet may be a straightforward process of
deploying your contracts to the Mainnet network, ensuring success and enhancing the exposure of your dApp may require
additional business development efforts. Here is an overview of some of the key factors to consider.

:::note
An Evmos validator, Stakely.io, runs a [faucet](https://stakely.io/en/faucet/evmos-evm) for builders to request dust.
:::

## Deployment

You can deploy your contracts on Mainnet using the [JSON-RPC](../develop/smart-contracts#deploy-with-ethereum-json-rpc).
This is the same procedure as on Testnet, but instead targeting the [Mainnet network endpoints](./../develop/api/networks).
Before you do so, have a look at the following considerations.

### Security

Thoroughly test your smart contracts on the Evmos Testnet to ensure it operates as intended. Comprehensive tests should
ensure all functions, error handling, and edge cases are covered.

Whenever possible, perform a comprehensive security audit of the smart contract code to identify and eliminate any
potential vulnerabilities or weaknesses. This is especially vital for the mainnet, as the code will be accessible to
everyone and any security flaws could result in substantial losses. Learn about the common vulnerabilities and contract
security practices. External auditors can also help optimize your contract's performance.

Ensure proper management of contract ownership and consider implementing a multi-sig mechanism for increased security.
This will allow you to maintain deployment ownership within your team instead of one specific owner that might leave the
team.

Last but not least, verify that any external libraries or dependencies used by the contract are up-to-date and secure.

### Contract upgradeability

Consider the possibility of upgrading the contract in the future and implement upgrade mechanisms if needed. This will
enable you to make changes to the contract without having to redeploy it, creating a new contract.

### Costs

Evaluate the gas costs associated with deploying and executing the smart contract, including the cost of deploying the
contract and executing its functions are sufficient or efficient for end users.

### Contract documentation

Provide clear and comprehensive documentation for the contract, including its purpose, functions, and potential risks.
This will assist users in understanding how to use the contract and make it easier for other developers to review and
contribute to the code.

## Token distribution

You're dApp might issue an ERC-20 token, e.g. to give token holders additional benefits. In this case, you will need to
decide on how to distribute them and what kind of narrative you want to create.

### Airdrop

One option is to distribute tokens to users through an airdrop. For some inspiration on how to select
eligible receivers of an airdrop have a look at the [Evmos Rektdrop](https://medium.com/evmos/the-evmos-rektdrop-abbe931ba823).

### Token Registration

Evmos allows for ERC-20 tokens to be used cross-chain. Once some of your tokens have been minted, you can register a token
pair through governance, which will allow users to send your tokens across chains. Head over to our Academy to learn how
to [register your ERC-20 token](https://academy.evmos.org/articles/advanced/erc20-registration).

## Revenue

On Evmos, you can generate revenue, every time a user interacts with your dApp,
gaining you a steady income.
Developers can register their smart contracts
and every time someone interacts with a registered smart contract,
the contract deployer
or their assigned withdrawal account receives a part of the transaction fees.

To learn about how to [register your smart contracts](https://academy.evmos.org/articles/advanced/incentives-registration),
head over to our Academy.
If you are curious on how this is implemented on the protocol,
check out the [revenue module specification](./../protocol/modules/incentives).

## Community

Make yourself heard in the Evmos community and explain what value your dApp provides.
An essential part of building a dApp is getting in touch with the community to showcase how they can take ownership or
start contributing to your project. This will not only help your dApp's visibility but might result in a new community
of users, that want to improve your dApp.

Head over to the [Evmos Discord](https://discord.gg/evmos) channel get in touch with the community and contributors and
showcase your dApp on one of the next community calls. We also have a [Telegram group](https://t.me/EvmosBuilders) for
our builders.
