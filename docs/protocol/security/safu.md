# 资金上传的简单安排（SAFU）

[资金上传的简单安排（"SAFU"）](https://github.com/evmos/evmos/tree/main/SAFU.pdf)概述了Evmos区块链中活跃漏洞的事后政策。
SAFU旨在面向白帽黑客，
并概述了返回资金和计算奖励的流程，
以及网络中发现的漏洞。
简而言之，SAFU规定如下：

* 如果黑客按照SAFU的规定行事，他们不会面临法律行动的风险。
* 黑客有一个宽限期将任何被利用的资金退还到指定的投递地址，并可以获得奖励，
  奖励金额为所保护的总资金的奖励百分比，最高不超过奖励上限。
* 奖励将在网络的下一次升级期间分发。
* 如果奖励金额超过指定的阈值金额，白帽黑客应该经过了解客户/了解业务（KYC/KYB）的过程。
* 以恶意目的利用漏洞将使黑客无资格获得任何奖励。
* 对于来自"Out of Scope Projects"（被黑客利用但没有自己的SAFU计划的其他项目）的资金，
  白帽黑客没有权利获得团队或网络的任何奖励。

获取更多信息，请访问[SAFU协议](https://github.com/evmos/evmos/tree/main/SAFU.pdf)。

## 投递地址

投递地址是协议从中提取资金的地址。
在奖励分发的情况下，
白帽黑客的奖励将从该地址的账户余额中支付。

:::tip
投递地址不受团队或任何个人控制，而是由Evmos协议控制。
:::

以下投递地址可在Evmos区块链上使用：

**Bech32格式的投递地址**：

```shell
evmos1c6jdy4gy86s69auueqwfjs86vse7kz3grxm9h2
```

**Hex格式的投递地址**：

```shell
0xc6A4d255043ea1A2F79CC81c9940FA6433eb0A28
```

### 地址派生

上面提供的 Dropbox 地址是通过以下算法从 `“safu”` 字符串的 SHA256 校验和的前 20 个字节进行密码学派生的：

```shell
address = shaSum256([]byte("safu"))[:20])
```

## 如何保护受攻击资金

在黑客攻击的宽限期内，白帽子应该将资金转移到 Dropbox 地址以确保安全。

## 如何领取奖励

奖励分发将在下一次链升级时手动进行。
如果奖励金额超过一定的阈值，
白帽子黑客应该经过了解客户/了解业务（KYC/KYB）的流程。

## dApp 的安全建议

如前所述，来自被黑客攻击的 dApp 的安全资金奖励不包含在协议的 SAFU 中。
对于这种情况，我们鼓励 Evmos 上的所有 dApp 自行实施 SAFU。
我们建议参考 Jump Crypto 的 [SAFU.sol](https://github.com/JumpCrypto/Safu/) 合约实现。


---
sidebar_position: 3
---

# Simple Arrangement for Funding Upload (SAFU)

The [Simple Arrangement for Funding Upload (the "SAFU")](https://github.com/evmos/evmos/tree/main/SAFU.pdf)
outlines the post-exploit policy for active vulnerabilities in the Evmos blockchain.
The SAFU is intended for white hat hackers
and outlines the process for returning funds and calculating rewards
for vulnerabilities found in the network.
In summary, the SAFU states the following:

* Hackers are not at risk of legal action if they act in accordance
  with the SAFU.
* Hackers have a Grace Period to return any exploited funds
  to a specified dropbox address and can claim a reward of
  a Bounty Percent of the total funds secured up to the Bounty Cap.
* The rewards are distributed during the next upgrade of the network.
* If the reward is valued above a specified threshold amount,
  white hat hackers should go through
  a Know Your Clients/Know Your Business (KYC/KYB) process.
* Exploiting vulnerabilities for malicious purposes
  will make a hacker ineligible for any rewards.
* White hat hackers are not entitled to any rewards from the team or network
  for funds from "Out of Scope Projects" (other projects that were exploited
  by hackers but do not have their own SAFU program).

For more information,
visit [the SAFU agreement](https://github.com/evmos/evmos/tree/main/SAFU.pdf).<!-- markdown-link-check-disable-line -->

## Dropbox Address

The Dropbox Address is an address to which funds are taken from
the protocol should be deposited.
In the event of a bounty distribution,
the bounty for white hat hackers will be paid out
from the account balance of this address.

:::tip
The dropbox address is not controlled by the team
or any individual, it is controlled by the Evmos protocol.
:::

The following dropbox address is available on the Evmos blockchain:

**Dropbox Address in Bech32 Format**:

```shell
evmos1c6jdy4gy86s69auueqwfjs86vse7kz3grxm9h2
```

**Dropbox Address in Hex Format**:

```shell
0xc6A4d255043ea1A2F79CC81c9940FA6433eb0A28
```

### Address Derivation

The dropbox address provided above is derived cryptographically from the
first 20 bytes of the SHA256 sum for the `“safu”` string,
using the following algorithm:

```shell
address = shaSum256([]byte("safu"))[:20])
```

## How To Secure Vulnerable Funds

Within the Grace Period of a hack,
white hats should secure the funds by transferring them to the dropbox address.

## How To Claim The Reward

Rewards distribution will be done manually on the next chain upgrade.
If the reward is valued above a certain threshold amount,
white hat hackers should go through a
Know Your Clients/Know Your Business (KYC/KYB) process.

## Security recommendations for dApps

As previously stated, rewards for secured funds from hacked dApps
are not included in the protocol's SAFU.
For such a case, we encourage all dApps on Evmos
to have their own SAFU implementation.
We recommend taking the [SAFU.sol](https://github.com/JumpCrypto/Safu/)
contract implementation from Jump Crypto as a reference.
