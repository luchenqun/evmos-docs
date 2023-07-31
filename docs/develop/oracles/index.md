# Oracles（预言机）

Evmos支持多个预言机提供商，以使智能合约能够访问链外数据并与现实世界进行交互（例如价格源或随机性）。这些预言机充当区块链的去中心化、无信任环境与传统互联网的中心化环境之间的桥梁。

预言机是一种从外部源获取数据并将其提供给区块链上的智能合约的软件。这使得智能合约能够响应现实世界的事件，触发自动化操作，并执行其预定的功能。

## 预言机列表

### 主网

| 服务         | 描述                                                                                                                                                                                                                                                                                                                                             | 链接和特点                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **[Pyth](https://docs.pyth.network/)**     | 利用[70多个一方发布者](https://pyth.network/publishers)将金融市场数据发布到多个区块链。他们为各种资产类别提供数据源，例如[美国股票、商品和加密货币](https://pyth.network/price-feeds/)。                                                            | <ul><li>[开发者文档](https://docs.pyth.network/)</li><li>[Linux上的Pyth客户端](https://github.com/pyth-network/pyth-client)</li><li>[Pyth TS客户端NPM](https://www.npmjs.com/package/@pythnetwork/client)</li><li>在[此处](https://github.com/pyth-network/audit-reports)查找审计报告</li></ul>                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| **[Adrastia](https://docs.adrastia.io/)** | 提供一个去中心化和无权限的预言机网络，安全可靠且易于使用。它使用[三种类型的合约](https://docs.adrastia.io/structure/contracts)提供安全的数据源：累加器、中间预言机和聚合预言机。                                                                                                                                     | <ul><li>Evmos的数据和合约地址可以在[此处](https://docs.adrastia.io/deployments/evmos)找到</li></ul>                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| **[DIA](https://docs.diadata.org/introduction/readme)**      | 为传统和数字金融应用程序提供透明且经过验证的数据源的获取、验证和共享。DIA的机构级数据源涵盖资产价格、元宇宙数据、借贷利率等。数据直接从各种链上和链下来源以个别交易级别获取。 | <ul><li>DIA的数据源可以根据来源和方法的组合进行完全定制，从而产生定制的高弹性数据源</li><li>[Evmos](https://docs.diadata.org/documentation/oracle-documentation/deployed-contracts#evmos)主网和测试网合约可供使用。更新频率为2小时。</li><li>[DIA的API链接](https://docs.diadata.org/documentation/api-1)</li><li>DIA有一个[自定义数据源构建器](https://app.diadata.org/feed-builder)，支持的代币对位于[此处](https://docs.diadata.org/documentation/oracle-documentation/deployed-contracts#evmos)</li><li>[DIA团队的Discord](https://go.diadata.org/dev-discord)</li></ul>    |
| **[Redstone](https://docs.redstone.finance/docs/introduction)** | 提供了一个完全不同设计的预言机，以满足现代DeFi协议的需求                                                                                                                                                                                                                                               | <ul><li>数据提供者可以避免连续在链上交付数据的要求</li><li>允许最终用户自行将签名的预言机数据交付到链上</li><li>使用去中心化的Streamr网络将签名的预言机数据交付给最终用户</li><li>使用代币激励来激励数据提供者维护数据完整性和不间断服务</li><li>利用Arweave区块链作为廉价和永久存储来存档预言机数据并维护数据提供者的责任</li><li>Redstone EVM连接器的示例可以在[此处](https://github.com/redstone-finance/redstone-evm-connector-examples/blob/main/contracts/example-custom-urls.sol)找到</li></ul> |
| **[SEDA Network](https://docs.seda.xyz/seda-network/introduction/the-oracle-problem)**     | 基于完全去中心化基础构建的多链本地数据传输协议。SEDA网络是一个基于权益证明的链上数据提供解决方案，允许任何人在所有区块链网络上提供和访问高质量的数据。                                                                                       | <ul><li>[SEDA Rust库](https://github.com/sedaprotocol/seda-rust)</li></ul>                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |

## Oracles是如何工作的？

``` sql
   +------------+         +------------+         +-------------+
   | External   |         |   Oracle   |         |  Smart      |
   | Data Source|         | Service    |         | Contract    |
   +------------+         +------------+         +-------------+
        |                       |                       |
        |  API Call             |                       |
        |---------------------> |                       |
        |                       |    Retrieve External  |
        |                       |    Data via API Call  |
        |                       |---------------------->|
        |                       |                       |
        |                       |    Use External Data  |
        |                       |    in Smart Contract  |
        |                       |<----------------------|
        |                       |                       |
        |                       |    Return Result to   |
        |                       |    Smart Contract     |
        |                       |<----------------------|
        |                       |                       |

```

在这个图表中：

* 外部数据源是指区块链网络之外的数据源，比如股市、天气服务或其他外部API。
* Oracle服务是一个第三方服务，充当外部数据源和智能合约之间的桥梁。它从外部源获取数据并提供给智能合约使用。
* 智能合约是部署在区块链网络上的自执行合约。它使用Oracle提供的数据执行特定的操作，比如释放资金或触发事件。
* API调用是指智能合约向Oracle服务发出的请求，请求所需的外部数据。
* 检索外部数据是指通过API调用从外部数据源检索所请求的数据的过程。
* 使用外部数据是指在智能合约中使用检索到的数据执行操作，比如条件检查和状态更改。
* 返回结果是指将智能合约中执行的操作结果返回给Oracle的过程。


# Oracles

Evmos supports several oracle providers to enable smart contracts to access
off-chain data and interact with the real world (e.g. price feeds or
randomness). These oracles serve as a bridge between the decentralized,
trustless environment of blockchain and the centralized, traditional internet.

An oracle is a piece of software that retrieves data from external sources and feeds it into smart contracts on the blockchain.
This enables smart contracts to respond to real-world events, trigger automated actions, and execute their intended functions.

## List of Oracles

### Mainnet

| Service      | Description                                                                                                                                                                                                                                                                                                                                      | Links & Features                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **[Pyth](https://docs.pyth.network/)**     | Leverages over [70 first-party publishers](https://pyth.network/publishers) to publish financial market data to numerous blockchains. They provide data feeds to various assets classes, such as [US equities, commodities, and cryptocurrencies](https://pyth.network/price-feeds/).                                                            | <ul><li>[Developer Docs](https://docs.pyth.network/)</li><li>[Pyth Client for Linux](https://github.com/pyth-network/pyth-client)</li><li>[Pyth TS client NPM](https://www.npmjs.com/package/@pythnetwork/client)</li><li>Find audit reports [here](https://github.com/pyth-network/audit-reports)</li></ul>                                                                                                                                                                                                                                                                                                                                                                                                    |
| **[Adrastia](https://docs.adrastia.io/)** | Provides a decentralized and permissionless oracle network that is secure, reliable, and easy to use. It uses [three types of contracts](https://docs.adrastia.io/structure/contracts) to provide secure data feeds: Accumulators, Intermediate oracles & Aggregator oracles                                                                     | <ul><li>Evmos data and contract address can be found [here](https://docs.adrastia.io/deployments/evmos)</li></ul>                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               |
| **[DIA](https://docs.diadata.org/introduction/readme)**      | Enables the sourcing, validation and sharing of transparent and verified data feeds for traditional and digital financial applications. DIA’s institutional-grade data feeds cover asset prices, metaverse data, lending rates and more. Data is directly sourced from a broad array of on-chain and off-chain sources at individual trade-level | <ul><li>DIA feeds are fully customizable with regards to the mix of sources and methodologies, resulting in tailor-made, high resilience feeds</li><li>[Evmos](https://docs.diadata.org/documentation/oracle-documentation/deployed-contracts#evmos) Mainnet and Testnet contracts available for use. Update frequency is 2 hrs.</li><li>[Link to DIA's API](https://docs.diadata.org/documentation/api-1)</li><li>DIA has a [custom feed builder](https://app.diadata.org/feed-builder) and the supported token pairs are located [here](https://docs.diadata.org/documentation/oracle-documentation/deployed-contracts#evmos)</li><li>the [DIA team Discord](https://go.diadata.org/dev-discord)</li></ul>    |
| **[Redstone](https://docs.redstone.finance/docs/introduction)** | Offers a radically different design of Oracles catering for the needs of modern Defi protocols                                                                                                                                                                                                                                                   | <ul><li>Data providers can avoid the requirement of continuous on-chain data delivery</li><li>Allow end users to self-deliver signed Oracle data on-chain</li><li>Use the decentralized Streamr network to deliver signed oracle data to the end users</li><li>Use token incentives to motivate data providers to maintain data integrity and uninterrupted service</li><li>Leverage the Arweave blockchain as a cheap and permanent storage for archiving Oracle data and maintaining data providers' accountability</li><li>Examples of Redstone EVM Connector can be found [here](https://github.com/redstone-finance/redstone-evm-connector-examples/blob/main/contracts/example-custom-urls.sol)</li></ul> |
| **[SEDA Network](https://docs.seda.xyz/seda-network/introduction/the-oracle-problem)**     | A multi-chain-native data transmission protocol built on an entirely decentralized foundation. The SEDA network is a Proof-of-Stake on-chain data provision solution that allows anyone to provide and access high-quality data on all blockchain networks                                                                                       | <ul><li>[SEDA Rust library](https://github.com/sedaprotocol/seda-rust)</li></ul>                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |

## How do Oracles work?

``` sql
   +------------+         +------------+         +-------------+
   | External   |         |   Oracle   |         |  Smart      |
   | Data Source|         | Service    |         | Contract    |
   +------------+         +------------+         +-------------+
        |                       |                       |
        |  API Call             |                       |
        |---------------------> |                       |
        |                       |    Retrieve External  |
        |                       |    Data via API Call  |
        |                       |---------------------->|
        |                       |                       |
        |                       |    Use External Data  |
        |                       |    in Smart Contract  |
        |                       |<----------------------|
        |                       |                       |
        |                       |    Return Result to   |
        |                       |    Smart Contract     |
        |                       |<----------------------|
        |                       |                       |

```

In this diagram:

* External Data Source refers to a source of data outside the blockchain network, such as a stock market, weather service, or other external API.
* Oracle Service is a third-party service that acts as a bridge between the external data source and the smart contract. It retrieves the data from the external source and provides it to the smart contract.
* Smart Contract is a self-executing contract that is deployed on the blockchain network. It uses the data provided by the oracle to perform certain actions, such as releasing funds or triggering events.
* API Call refers to the request made by the smart contract to the oracle service, asking for the required external data.
* Retrieve External Data refers to the process of retrieving the requested data from the external data source via the API call.
* Use External Data refers to the process of using the retrieved data in the smart contract to perform actions, such as condition checking and state changes.
* Return Result refers to the process of returning the result of the action performed in the smart contract back to the oracle.
