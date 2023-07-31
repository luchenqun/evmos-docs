---
sidebar_position: 1
---
# 概述

## 在 Evmos 上进行验证

Evmos 基于 [CometBFT](https://github.com/cometbft/cometbft)，
它依赖一组验证者来负责在区块链中提交新的区块。这些验证者通过广播包含每个验证者私钥签名的加密签名的投票来参与共识协议。

验证者候选人可以抵押自己的质押代币，并通过代币持有人将代币“委托”或抵押给他们。EVMOS 是 Evmos 的原生代币。在初始阶段，Evmos 启动时有 150 个验证者。验证者是由谁被委托的质押代币最多来决定的 - 委托代币最多的前 150 个验证者候选人将成为活跃的 Evmos 验证者集合的一部分。

验证者及其委托人将通过执行 Tendermint 共识协议获得 EVMOS 作为区块奖励和代币作为交易费用。最初，交易费用将以 EVMOS 支付，但在将来，只要经过治理白名单批准，Cosmos 生态系统中的任何代币都可以作为费用支付的有效代币。请注意，验证者可以对其委托人获得的费用设置佣金作为额外激励。

## 注意事项

如果验证者双重签名、经常离线或不参与治理，他们抵押的 EVMOS（包括委托给他们的用户的 EVMOS）可能会被削减。惩罚的严重程度取决于违规行为的严重程度。

## 硬件

验证者应该建立一个受限访问的物理操作环境。一个很好的起点，例如，是在安全的数据中心进行共同定位。

验证者应该预期将其数据中心位置配备冗余电源、连接和存储备份。预计需要几个冗余的网络盒子用于光纤、防火墙和交换机，然后是带有冗余硬盘和故障转移的小型服务器。硬件可以从数据中心设备的低端开始。

我们预计初始时网络要求较低。随着网络的增长，带宽、CPU 和内存要求将增加。建议使用大容量硬盘来存储多年的区块链历史记录。

### 支持的操作系统

我们正式支持以下架构的 macOS 和 Linux：

* `darwin/arm64`
* `darwin/x86_64`
* `linux/arm64`
* `linux/amd64`

### 最低要求

要运行主网或测试网验证节点，您需要一台具备以下最低硬件要求的机器：

* 至少 4 个物理 CPU 核心
* 至少 500GB 的 SSD 磁盘存储空间
* 至少 32GB 的内存（RAM）
* 至少 100mbps 的网络带宽

随着区块链的使用增长，服务器要求也可能增加，因此您应该有一个更新服务器的计划。

## 参与进来

:::tip
如果您打算运行验证节点，请寻求法律建议。
:::

设置一个专用的验证节点网站，社交资料（例如：Twitter），并在 Discord 上表明您成为验证节点的意图。这很重要，因为用户希望了解他们抵押 EVMOS 的实体的信息。

## 社区

在我们的 [Discord](https://discord.gg/evmos) 上讨论成为验证节点的细节，并向其他验证节点社区寻求建议。


---
sidebar_position: 1
---
# Overview

## Validating on Evmos

Evmos is based on [CometBFT](https://github.com/cometbft/cometbft),
which relies on a set of validators that are responsible for committing new blocks in the blockchain. These validators
participate in the consensus protocol by broadcasting votes which contain cryptographic signatures signed by each
validator's private key.

Validator candidates can bond their own staking tokens and have the tokens "delegated", or staked, to them by token
holders. The EVMOS is Evmos's native token. At its onset, Evmos launche with 150 validators. The validators are
determined by who has the most stake delegated to them - the top 150 validator candidates with the most stake
become part of the active Evmos validator set.

Validators and their delegators will earn EVMOS as block provisions and tokens as transaction fees through execution of
the Tendermint consensus protocol. Initially, transaction fees will be paid in EVMOS but in the future, any token in the
Cosmos ecosystem will be valid as fee tender if it is whitelisted by governance. Note that validators can set commission
on the fees their delegators receive as additional incentive.

## Pitfalls

If validators double sign, are frequently offline or do not participate in governance, their staked EVMOS (including
EVMOS of users that delegated to them) can be slashed. The penalty depends on the severity of the violation.

## Hardware

Validators should set up a physical operation secured with restricted access. A good starting place, for example,
would be co-locating in secure data centers.

Validators should expect to equip their datacenter location with redundant power, connectivity, and storage backups.
Expect to have several redundant networking boxes for fiber, firewall and switching and then small servers with redundant
hard drive and failover. Hardware can be on the low end of datacenter gear to start out with.

We anticipate that network requirements will be low initially. Bandwidth, CPU and memory requirements will rise as
the network grows. Large hard drives are recommended for storing years of blockchain history.

### Supported OS

We officially support macOS and Linux only in the following architectures:

* `darwin/arm64`
* `darwin/x86_64`
* `linux/arm64`
* `linux/amd64`

### Minimum Requirements

To run mainnet or testnet validator nodes, you will need a machine with the following minimum hardware requirements:

* 4 or more physical CPU cores
* At least 500GB of SSD disk storage
* At least 32GB of memory (RAM)
* At least 100mbps network bandwidth

As the usage of the blockchain grows, the server requirements may increase as well, so you should have a plan for
updating your server as well.

## Get Involved

:::tip
Seek legal advice if you intend to run a validator.
:::

Set up a dedicated validator's website, social profile (eg: Twitter) and signal your intention to become a validator on
Discord. This is important since users will want to have information about the entity they are staking their EVMOS to.

## Community

Discuss the finer details of being a validator and seek advise from the rest of the validator community on our
[Discord](https://discord.gg/evmos).
