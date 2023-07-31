# 签名

签名是使用私钥创建数字签名以验证 Evmos 网络上的交易的过程。签名使用特定的加密算法创建，以确保使用[钱包](./../../use/connect-your-wallet)和[CLI](./../evmos-cli)等方法验证交易的真实性和完整性。

有不同的签名方法，但其中一种最常用的方法是[EIP-712](https://eips.ethereum.org/EIPS/eip-712)标准。Evmos 利用 EIP-712 来使 EVM 和 Cosmos 之间的交互更加统一。

## EIP-712

EIP-712 引入了一种以人类可读格式签名 "typed-data" 的标准。该标准使用户能够更轻松地理解他们正在签名的数据，并提供了一种更安全的签名数据的方式，因为它不容易受到钓鱼攻击的影响。EIP-712 不是以太坊的交易类型，而是一种用于签名结构化数据以进行身份验证和对程序逻辑的间接影响的方法。

为了支持签名 Cosmos 交易，Evmos 使用 EIP-712 协议将 Cosmos 交易编码为可以被 Ethereum 签名者（包括 Ledger 硬件钱包）理解和处理的格式。这种方法有助于克服 Ethereum 签名设备的限制，这些设备通常不支持签名任意字节以确保安全性。

该过程的工作原理如下：

1. Cosmos 交易被表示为 JSON sign-doc。
2. JSON sign-doc 被转换为 EIP-712 对象，该对象由类型和消息组成。
3. 使用 Ethereum 签名者（如 MetaMask 或 Ledger 硬件设备）对 EIP-712 对象进行签名。
4. 在节点上执行相同的过程以验证签名。

通过使用 EIP-712 对 Cosmos 交易进行签名，Evmos 确保与流行的 Ethereum 签名工具（如 MetaMask 和 Ledger 设备）以及 Keplr 兼容。这种兼容性使用户更容易与 Ethereum 和 Cosmos 网络进行交互，最终促进了两个生态系统之间的更大互操作性。

:::note
EvmosJS支持使用EIP-712进行签名。有关该库的更多信息，请访问[此处](https://github.com/evmos/evmosjs)。

支持：[Ledger支持](./../../use/connect-your-wallet/ledger)和[CLI命令](./../evmos-cli/cli-commands)。
:::


---
sidebar_position: 11
---

# Signing

Signing is the process of creating a digital signature using a private key to verify a transaction
on the Evmos network. The signature is created using a specific cryptographic algorithm that
ensures the authenticity and integrity of the transaction using methods like
[wallets](./../../use/connect-your-wallet) and the [CLI](./../evmos-cli).

There are different methods for signing, but one of the most commonly used methods is the 
[EIP-712](https://eips.ethereum.org/EIPS/eip-712) standard.
Evmos leverages EIP-712 to homogenize the interaction between the EVM and Cosmos.

## EIP-712

EIP-712 introduces a standard for signing "typed-data" in a human-readable format. This standard allowed users to understand
the data they are signing more easily and provides a more secure way to sign data, as it is less susceptible to phishing
attacks. EIP-712 is not an Ethereum transaction type, but a method for signing structured data that can be used for
authentication and indirect influence on program logic.

To support signing Cosmos transactions, Evmos utilizes the EIP-712 protocol for encoding Cosmos transactions in a
format that can be understood and processed by Ethereum signers, including Ledger hardware wallets. This approach
helps to overcome the limitations of Ethereum signing devices, which often do not support signing arbitrary bytes
for security reasons.

The process works as follows:

1. A Cosmos transaction is represented as a JSON sign-doc.
2. The JSON sign-doc is converted to an EIP-712 object, which consists of types and messages.
3. The EIP-712 object is signed using an Ethereum signer, such as MetaMask or a Ledger hardware device.
4. The same process is performed on the node to verify the signature.

By using EIP-712 for signing Cosmos transactions, Evmos ensures compatibility with popular Ethereum signing tools
like MetaMask and Ledger devices as well as Keplr. This compatibility makes it easier for users to interact with both
Ethereum and Cosmos networks, ultimately fostering greater interoperability between the two ecosystems.

:::note
EvmosJS supports signing with EIP-712. More information about the library can be found [here](https://github.com/evmos/evmosjs).

Supported: [Ledger support](./../../use/connect-your-wallet/ledger) and [CLI Commands](./../evmos-cli/cli-commands).
:::
