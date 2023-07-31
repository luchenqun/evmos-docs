# 智能合约

自2015年以太坊的推出以来，
通过[智能合约](https://ethereum.org/en/smart-contracts/)控制数字资产的能力
吸引了大量开发者社区
在以太坊虚拟机（EVM）上构建去中心化应用程序。
这个社区不断地创建广泛的工具和引入标准，
进一步提高了EVM兼容技术的采用率。

无论您是在Evmos上构建新的用例
还是将现有的dApp从其他基于EVM的链（例如以太坊）迁移，
您都可以轻松地在Evmos上构建和部署EVM智能合约来实现您的dApp的核心业务逻辑。
Evmos与EVM完全兼容，
因此它允许您使用与EVM上可用的相同工具（Solidity、Remix、Oracles等）
和API（即以太坊JSON-RPC）。

利用Cosmos链的互操作性，
Evmos使您能够在熟悉的EVM环境中构建可扩展的跨链应用程序。
在下面了解在Evmos上构建和部署EVM智能合约时的基本组件。

## 使用Solidity构建

您可以使用[Solidity](https://github.com/ethereum/solidity)在Evmos上开发EVM智能合约。
Solidity也用于在以太坊上构建智能合约。
因此，如果您已经在以太坊（或任何其他EVM兼容链）上部署了智能合约，
您可以在Evmos上使用相同的合约。

作为区块链中最广泛使用的智能合约编程语言，
Solidity具有良好的文档和丰富的语言支持。
请查看我们的工具和IDE插件列表，以帮助您入门。

### EVM扩展

EVM扩展是预编译合约，内置于以太坊虚拟机（EVM）中。
每个扩展提供特定的功能，可以被其他智能合约使用。
通常，它们用于执行常规智能合约无法实现或成本过高的操作，
例如哈希、椭圆曲线加密和模幂运算。

通过向以太坊的基本功能集添加自定义的 EVM 扩展，Evmos 允许开发人员在智能合约中使用以前不可用的功能，如质押和治理操作。这将使得在 Evmos 上可以构建更复杂的智能合约，并进一步提高 Cosmos 和以太坊之间的互操作性。这也是实现 Evmos 成为终极 dApp 链的关键特性，任何 dApp 只需部署一次，用户就可以与各种不同的区块链进行本地交互。

为了启用上述功能，Evmos 引入了所谓的“有状态”预编译智能合约，它们可以执行状态转换，而不同于标准的 Go-Ethereum 实现只能读取状态信息。这是必要的，因为诸如质押代币之类的操作最终会改变链的状态。

在[此处](./list-evm-extensions.md)查看可用的 evm 扩展列表。

### 预言机

区块链预言机为智能合约提供了访问外部信息的方式，例如金融交易所的价格源或碳排放测量数据。它们充当区块链与外部世界之间的桥梁。

请访问我们的[预言机部分](./oracles)了解智能合约如何在 Evmos 上利用预言机进行保险、借贷、贷款或游戏等真实活动。

## 使用以太坊 JSON-RPC 部署

Evmos 完全兼容[Ethereum JSON-RPC](./../../develop/api/ethereum-json-rpc/) API，允许您在 Evmos 上部署和与智能合约交互，并与现有的以太坊兼容的 web3 工具进行连接。这使您可以直接访问以太坊格式的交易或将其发送到网络，这在 Cosmos 链上是不可能的，比如 Evmos。

您可以连接到 Evmos [测试网](./testnet)在部署和测试智能合约之前。

### 区块浏览器

您可以使用[区块浏览器](./block-explorers)查看和调试在 Evmos 上部署的智能合约的交互。区块浏览器索引区块及其交易，以便您可以搜索与区块链相关的实时和历史信息，包括与区块、交易、地址等相关的数据。

### 合约验证

一旦部署，智能合约数据将以非人类可读的EVM字节码形式部署。
您可以使用[合约验证工具](./tools/contract-verifications)
发布和验证您的原始Solidity代码，
以向用户证明他们正在与正确的智能合约进行交互。

## Evmos功能

核心协议团队不断构建功能，
以增强Evmos上的智能合约开发者的体验。
请前往我们的主网部分，了解更多关于这些功能的信息，
例如如何通过您的智能合约赚取[收入](./mainnet#revenue)
或[注册您的ERC-20](./mainnet#token-registration)代币
以进行跨链使用。


# Smart Contracts

Since the introduction of Ethereum in 2015,
the ability to control digital assets through [smart contracts](https://ethereum.org/en/smart-contracts/)
has attracted a large community of developers
to build decentralized applications on the Ethereum Virtual Machine (EVM).
This community is continuously creating extensive tooling and introducing standards,
which are further increasing the adoption rate of EVM-compatible technology.

Whether you are building new use cases on Evmos
or porting an existing dApp from another EVM-based chain (e.g. Ethereum),
you can easily build and deploy EVM smart contracts on Evmos to implement the core business logic of your dApp.
Evmos is fully compatible with the EVM,
so it allows you to use the same tools (Solidity, Remix, Oracles, etc.)
and APIs (i.e. Ethereum JSON-RPC) that are available on the EVM.

Leveraging the interoperability of Cosmos chains,
Evmos enables you to build scalable cross-chain applications within a familiar EVM environment.
Learn about the essential components when building and deploying EVM smart contracts on Evmos below.

## Build with Solidity

You can develop EVM smart contracts on Evmos using [Solidity](https://github.com/ethereum/solidity).
Solidity is also used to build smart contracts on Ethereum.
So if you have deployed smart contracts on Ethereum (or any other EVM-compatible chain)
you can use the same contracts on Evmos.

Since it is the most widely used smart contract programming language in Blockchain,
Solidity comes with well-documented and rich language support.
Head over to our list of Tools and IDE Plugins to help you get started.

### EVM Extensions

EVM Extensions are precompiled contracts that are built into the Ethereum Virtual Machine (EVM).
Each offers specific functionality, that can be used by other smart contracts.
Generally, they are used to perform operations that are either not possible
or would be too expensive to perform with a regular smart contract
implementation, such as hashing, elliptic curve cryptography, and modular exponentiation.

By adding custom EVM extensions to Ethereum's basic feature set,
Evmos allows developers to use previously unavailable functionality in smart contracts, like staking and governance operations.
This will allow more complex smart contracts to be built on Evmos and further improves the interoperability between Cosmos and Ethereum.
It also is a key feature to achieve Evmos' vision of being the definitive dApp
chain, where any dApp can be deployed once and users can interact with
a wide range of different blockchains natively.

To enable the described functionalities, Evmos introduces so-called *stateful* precompiled smart contracts,
which can perform a state transition,
as opposed to those offered by the standard Go-Ethereum implementation,
which can only read state information.
This is necessary because an operation like e.g. staking tokens
will ultimately change the chain state.

View a list of available evm extensions [here](./list-evm-extensions.md).

### Oracles

Blockchain oracles provide a way for smart contracts to access external information,
such as price feeds from financial exchanges or carbon emission measurements.
They serve as bridges between blockchains and the outside world.

Head over to our [Oracles section](./oracles) to find out
how smart contracts can make use of oracles on Evmos for real-life activities
such as insurance, borrowing, lending, or gaming.

## Deploy with Ethereum JSON-RPC

Evmos is fully compatible with the [Ethereum JSON-RPC](./../../develop/api/ethereum-json-rpc/) APIs,
allowing you to deploy and interact with smart contracts on Evmos
and connect with existing Ethereum-compatible web3 tooling.
This gives you direct access to reading Ethereum-formatted transactions
or sending them to the network which otherwise wouldn't be possible on a Cosmos chain, such as Evmos.

You can connect to the Evmos [Testnet](./testnet)
to deploy and test your smart contracts before moving to Mainnet.

### Block Explorers

You can use [block explorers](./block-explorers)
to view and debug interactions with your smart contracts deployed on Evmos.
Block explorers index blocks and their transactions
so that you can search for real-time and historical information about the blockchain,
including data related to blocks, transactions, addresses, and more.

### Contract Verification

Once deployed, smart contract data is deployed as non-human readable EVM bytecode.
You can use [contract verification tools](./tools/contract-verifications)
that publish and verify your original Solidity code
to prove to users that they are interacting with the correct smart contract.

## Evmos Features

The core protocol team is continuously building features
that enhance the experience of smart contract developers on Evmos.
Head over to our Mainnet sections to learn more about these functionalities,
e.g. how to earn [revenue](./mainnet#revenue) with your smart contract
or [register your ERC-20](./mainnet#token-registration) token
to be used cross-chain.
