---
sidebar_position: 1
---

# 验证者安全

鼓励每个验证者独立运行其操作，因为多样化的设置增加了网络的弹性。验证者候选人应该留出足够的时间来确保安全地启动验证者。

在本节中，您可以了解有关安全运行验证者的最佳实践，而不会牺牲块签名性能。这包括如何保护您的私钥，运行具有远程访问权限的节点集群，减轻双重签名的风险，并通过哨兵节点为网络的DDoS保护做出贡献。

此外，提供了一个[验证者安全检查清单](./validator-security-checklist.md)，用于对验证者当前的安全措施进行调查。

## Horcrux

Horcrux是一个用于Tendermint节点的多方计算（MPC）签名服务，它在安全性和可用性方面提高了您的验证者基础设施。它提供了以下功能：

- 由一组签名节点组成，取代了远程签名节点，通过容错性实现块签名的高可用性（HA）。
- 通过将验证者私钥分散到多个私有签名节点，使用阈值Ed25519签名来保护您的私钥。
- 在不牺牲块签名性能的情况下增加安全性和可用性。

请查看[此处的文档](https://github.com/strangelove-ventures/horcrux/blob/main/docs/migrating.md)，了解如何使用Horcrux升级您的验证者基础设施。

## 硬件HSM

攻击者不能窃取验证者的密钥是至关重要的。如果这是可能的，那么整个委托给受损验证者的利益都会面临风险。硬件安全模块是减轻这种风险的重要策略。

HSM模块必须支持Evmos的`ed25519`签名。[YubiHSM 2](https://www.yubico.com/products/hardware-security-module/)支持`ed25519`，并且可以与此[YubiKey库](https://github.com/iqlusioninc/yubihsm.rs)一起使用。

:::info
🚨 **重要提示**：YubiHSM可以保护私钥，但**无法确保**在安全环境中不会对同一个块进行两次签名。
:::

## Tendermint KMS

Tendermint KMS是一个支持硬件安全模块（HSMs）的签名服务，例如YubiHSM2和Ledger Nano。它旨在与Cosmos验证器一起运行，最好在单独的物理主机上，为在线验证器签名密钥提供深度防御、双重签名保护，并作为一个中央签名服务，可在多个Cosmos区域中运行多个验证器时使用。

了解如何使用Tendermint KMS为Evmos设置密钥管理系统[在这里](./tendermint-kms)。

## 哨兵节点（DDoS保护）

验证器有责任确保网络能够承受拒绝服务攻击。一种推荐的减轻这些风险的方法是让验证器仔细构建其网络拓扑，采用所谓的哨兵节点架构。

验证器节点应仅连接到它们信任的全节点，因为这些全节点由它们自己操作或由其他验证器社交认可。验证器节点通常在数据中心运行。大多数数据中心提供与主要云提供商的网络直接连接。验证器可以使用这些连接来连接到云中的哨兵节点。这将将拒绝服务的负担从验证器节点直接转移到其哨兵节点，并可能需要快速启动或激活新的哨兵节点以减轻对现有哨兵节点的攻击。

哨兵节点可以快速启动或更改其IP地址。由于与哨兵节点的连接位于私有IP空间中，因此互联网攻击无法直接干扰它们。这将确保验证器的区块提案和投票始终能够传递到网络的其他部分。

:::tip
在[论坛](https://forum.cosmos.network/t/sentry-node-architecture-overview/454)上阅读有关哨兵节点的更多信息。
:::

要设置您的哨兵节点架构，您可以按照以下说明进行操作：

验证器节点应编辑其`config.toml`文件：

```bash
# Comma separated list of nodes to keep persistent connections to
# Do not add private peers to this list if you don't want them advertised
persistent_peers =[list of sentry nodes]

# Set true to enable the peer-exchange reactor
pex = false
```

哨兵节点应编辑其`config.toml`文件：

```bash
# Comma separated list of peer IDs to keep private (will not be gossiped to other peers)
# Example ID: 3e16af0cead27979e1fc3dac57d03df3c7a77acc@3.87.179.235:26656

private_peer_ids = "node_ids_of_private_peers"
```

## 验证器备份

备份您的验证者私钥是**至关重要**的。这是在发生灾难时恢复您的验证者的唯一方法。验证者私钥是一个 Tendermint 密钥：用于签署共识投票的唯一密钥。

要备份您需要恢复验证者的所有内容，请注意，如果您正在使用 "软件签名"（Tendermint 的默认签名方法），则您的 Tendermint 密钥位于：

```bash
~/.evmosd/config/priv_validator_key.json
```

然后执行以下操作：

1. 备份上述提到的 `json` 文件（或备份整个 `config` 文件夹）。
2. 备份自委托钱包。请参阅[使用 Evmos Daemon 备份钱包](./../../protocol/concepts/key-management)。

要查看您验证者关联的公钥：

```bash
evmosd tendermint show-validator
```

要查看您验证者关联的 bech32 地址：

```bash
evmosd tendermint show-address
```

您还可以使用硬件设备更安全地存储您的 Tendermint 密钥，例如[YubiHSM2](https://developers.yubico.com/YubiHSM2/)。

## 环境变量

默认情况下，具有以下前缀的大写环境变量将替换小写命令行标志：

- `EVMOS`（用于 Evmos 标志）
- `TM`（用于 Tendermint 标志）
- `BC`（用于 democli 或 basecli 标志）

例如，环境变量 `EVMOS_CHAIN_ID` 将映射到命令行标志 `--chain-id`。请注意，虽然显式的命令行标志将优先于环境变量，但环境变量将优先于任何配置文件。因此，您必须锁定您的环境，以便任何关键参数都被定义为二进制文件上的标志，或者防止修改任何环境变量。


---
sidebar_position: 1
---

# Validator Security

Each validator is encouraged to run its operations independently, as diverse setups increase the resilience of the network. Validator candidates should put aside meaningful time to guarantee a secure validator launch.

In this section, you can learn about best practices for operating a validator securely without sacrificing block sign performance. This includes information on how to secure your private keys, run a cluster of nodes with remote access, mitigate the risk of double signing and contribute to DDOS protection on the network through sentry nodes.

Also, a [validator security checklist](./validator-security-checklist.md) is provided to conduct a survey on the current security measures of a validator.

## Horcrux

Horcrux is a [multi-party-computation (MPC)](https://en.wikipedia.org/wiki/Secure_multi-party_computation) signing service for Tendermint nodes, that improves your validator infrastructure in terms of security and availability. It offers

- Composed of a cluster of signer nodes in place of the remote signer, enabling High Availability (HA) for block signing through fault tolerance.
- Secure your validator private key by splitting it across multiple private signer nodes using threshold Ed25519 signatures
- Add security and availability without sacrificing block sign performance.

See the documentation [here](https://github.com/strangelove-ventures/horcrux/blob/main/docs/migrating.md) to learn how to upgrade your validator infrastructure with Horcrux.

## Hardware HSM

It is mission-critical that an attacker cannot steal a validator's key. If this is possible, it puts the entire stake delegated to the compromised validator at risk. Hardware security modules are an important strategy for mitigating this risk.

HSM modules must support `ed25519` signatures for Evmos. The [YubiHSM 2](https://www.yubico.com/products/hardware-security-module/) supports `ed25519` and can be used with this YubiKey [library](https://github.com/iqlusioninc/yubihsm.rs).

:::info
🚨 **IMPORTANT**: The YubiHSM can protect a private key but **cannot ensure** in a secure setting that it won't sign the same block twice.
:::

## Tendermint KMS

Tendermint KMS is a signature service with support for Hardware Security Modules (HSMs), such as YubiHSM2 and Ledger 
Nano. It is intended to be run alongside Cosmos Validators, ideally on separate physical hosts, providing 
defense-in-depth for online validator signing keys, double signing protection, and functioning as a central 
signing service that can be used when operating multiple validators in several Cosmos Zones.

Learn how to set up a Key Management System for Evmos with Tendermint KMS [here](./tendermint-kms).

## Sentry Nodes (DDOS Protection)

Validators are responsible for ensuring that the network can sustain denial-of-service attacks. One recommended way 
to mitigate these risks is for validators to carefully structure their network topology in a so-called sentry node architecture.

Validator nodes should only connect to full nodes they trust because they operate them themselves or are run by other validators they know socially. A validator node will typically run in a data center. Most data centers provide direct links to the networks of major cloud providers. The validator can use those links to connect to sentry nodes in the cloud. This shifts the burden of denial-of-service from the validator's node directly to its sentry nodes and may require new sentry nodes to be spun up or activated to mitigate attacks on existing ones.

Sentry nodes can be quickly spun up or change their IP addresses. Because the links to the sentry nodes are in private IP space, an internet-based attack cannot disturb them directly. This will ensure validator block proposals and votes always make it to the rest of the network.

:::tip
Read more about Sentry Nodes on the [forum](https://forum.cosmos.network/t/sentry-node-architecture-overview/454)
:::

To set up your sentry node architecture you can follow the instructions below:

Validator nodes should edit their `config.toml`:

```bash
# Comma separated list of nodes to keep persistent connections to
# Do not add private peers to this list if you don't want them advertised
persistent_peers =[list of sentry nodes]

# Set true to enable the peer-exchange reactor
pex = false
```

Sentry Nodes should edit their config.toml:

```bash
# Comma separated list of peer IDs to keep private (will not be gossiped to other peers)
# Example ID: 3e16af0cead27979e1fc3dac57d03df3c7a77acc@3.87.179.235:26656

private_peer_ids = "node_ids_of_private_peers"
```

## Validator Backup

It is **crucial** to back up your validator's private key. It's the only way to restore your validator in the event of a
 disaster. The validator private key is a Tendermint Key: a unique key used to sign consensus votes.

To backup everything you need to restore your validator, note that if you are using the "software sign" (the default
signing method of Tendermint), your Tendermint key is located at:

```bash
~/.evmosd/config/priv_validator_key.json
```

Then do the following:

1. Back up the `json` file mentioned above (or backup the whole `config` folder).
2. Back up the self-delegator wallet. See [backing up wallets with the Evmos Daemon](./../../protocol/concepts/key-management).

To see your validator's associated public key:

```bash
evmosd tendermint show-validator
```

To see your validator's associated bech32 address:

```bash
evmosd tendermint show-address
```

You can also use hardware to store your Tendermint Key much more safely, such as [YubiHSM2](https://developers.yubico.com/YubiHSM2/).

## Environment Variables

By default, uppercase environment variables with the following prefixes will replace lowercase command-line flags:

- `EVMOS` (for Evmos flags)
- `TM` (for Tendermint flags)
- `BC` (for democli or basecli flags)

For example, the environment variable `EVMOS_CHAIN_ID` will map to the command line flag `--chain-id`. Note that while
explicit command-line flags will take precedence over environment variables, environment variables will take precedence
over any of your configuration files. For this reason,  you must lock down your environment such that any critical
parameters are defined as flags on the binary or prevent modification of any environment variables.
