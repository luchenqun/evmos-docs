---
sidebar_position: 4
slug: '/dapps'
---

# 使用dApps

在使用dApps之前，重要的是要拥有一些Evmos代币的钱包。如果您还没有安装钱包，请访问我们的[钱包](./../use/wallet)部分选择一个合适的钱包。如果您没有任何EVMOS代币，您可以通过阅读[Onramp to Evmos](../transfer-tokens/index.md#onramp-to-evmos)策略来获取它们。

<!-- 添加链接到[dApps](../intro#what-are-dapps) -->

您可以开始使用由Evmos核心开发团队构建和维护的Evmos dApps，或者使用我们的[Ecosystem Page](https://evmos.org/ecosystem)浏览Evmos上丰富的dApps生态系统。

## 查询和交易

一旦您设置好了钱包，就可以开始在Evmos上使用dApps。通常，从网络中读取信息（查询）是免费的，不需要支付交易费用。因此，dApps始终可以显示区块链信息，例如您的资产余额，而无需您支付网络费用。

要与dApp进行交互，您需要使用钱包签署交易并支付一小笔网络费用。交易是在区块链上的价值转移。这可以是将代币简单转移到另一个账户，更改游戏中角色的所有权，或者在借贷资产时接受贷款条款。

与dApp交互的典型流程如下：

1. 将您的钱包连接到Evmos网络。
2. 访问一个dApp并提交一个交易（例如抵押您的代币）。
3. 在您的钱包上，通过支付燃气费用签署和广播交易。
4. 等待交易被确认并成功。
5. 在dApp中查看您的交易所带来的变化。您可以使用区块浏览器查看有关您的交易的更多详细信息。交易会被批处理并与其他交易一起在一个区块中处理，区块浏览器允许任何人查看网络中的所有交易。

## Evmos dApps

除了开发者社区构建的应用程序外，Evmos核心开发团队还构建了自己的[Evmos dApps](https://app.evmos.org)。一个简单的开始方式是抵押并投票支持治理提案。

### 质押

质押在保持 Evmos 网络安全和去中心化方面起着至关重要的作用。Evmos 网络由一种名为权益证明（Proof-of-Stake）的技术运行，该技术激励一组社区成员将其保持在线和去中心化。这些社区成员被称为验证者，负责验证区块中的交易。通常情况下，他们是非常技术性的团队，因为他们需要监控和升级运行 Evmos 网络的软件。验证者通过获得质押奖励来激励他们保持在线。

您可以通过将您的 EVMOS 代币委托给一个验证者来帮助保持网络的安全，并将其质押。这将以每日利息的形式为您赚取新的 EVMOS 代币（也称为质押奖励）。只要您的代币被质押，您就无法将其转移或在其他 dApp 中使用。在任何时候，您都可以取消委托您的代币。在协议中定义了一个解绑期（例如两周），在此期间过后，您可以再次开始使用您的代币。

![evmos_staking.png](/img/evmos_staking.png)

您可以从[Evmos Staking dApp](https://app.evmos.org/staking)的列表中选择验证者来质押您的代币。我们建议选择一位信誉良好的验证者，他们承诺保持在线并且没有很大的投票权（例如不在前20位）。长时间离线或恶意行为的验证者可能会被惩罚，导致您的质押损失。此外，委托给投票权较小的验证者可以使网络更加去中心化。您可以通过重新委托来在验证者之间切换，而无需取消委托您的代币。

### 治理

Evmos 具有链上治理机制，用于对网络协议进行更改，例如更改质押的解绑期限、从社区资金池中支出资金（例如为 Evmos 上的团队提供资金）或升级由验证者运行的 Evmos 软件。

开发人员通过代码更新提出更改建议，社区对是否接受或拒绝提议进行投票。只要他们质押了 EVMOS 代币，每个参与者都可以对提案进行投票。每个投票的权重由质押代币的数量决定。

![evmos_governance.png](/img/evmos_governance.png)

Evmos以其在区块链领域中托管最活跃的治理社区而闻名。任何人都可以参与治理，因此Evmos网络的所有权、规范和文化都分散在社区中。一旦您抵押了您的代币，请前往[Evmos治理dApp](https://app.evmos.org/governance)开始对治理提案进行投票。

:::note
在[Commonwealth](https://commonwealth.im/evmos)上与我们的社区互动，并了解即将推出的提案。如果您有兴趣发起自己的治理提案，请参考[这里](https://academy.evmos.org/community/governance/)的指南。
:::


---
sidebar_position: 4
slug: '/dapps'
---

# Use dApps

Before engaging with dApps, it is important to have a wallet with some Evmos tokens. If you have not installed a wallet,
please visit our [wallet's](./../use/wallet) section to select an appropriate wallet. If you do not have any EVMOS tokens,
you can acquire them by reading the [Onramp to Evmos](../transfer-tokens/index.md#onramp-to-evmos) strategies.

<!-- add link to [dApps](../intro#what-are-dapps) -->

You can start by using the Evmos dApps that are built and maintained by the Evmos core development team or use our
[Ecosystem Page](https://evmos.org/ecosystem) to browse through the rich ecosystem of dApps on Evmos.

## Queries and Transactions

Once you have your wallet set up, you can start using dApps on Evmos. Generally, reading information from the network
(querying) is free and doesn't require you to pay transaction fees. So dApps can always display blockchain information,
such as your asset balances without requiring you to pay a network fee.

To engage with a dApp, you are required to sign transactions with your wallet and pay a small network fee. A transaction
is a transfer of value on the blockchain. This can be anything like a simple transfer of tokens to another account,
changing the ownership of a character in a game or accepting the terms of a loan when lending or borrowing assets.

A typical flow to interact with a dApp can look like this:

1. Connect your wallet to the Evmos network.
2. Visit a dApp and submit a transaction (e.g. to stake your tokens).
3. On your wallet, sign and broadcast the transaction by paying the network fee with gas.
4. Wait until the transaction is confirmed and successful.
5. View the changes from your transaction in the dApp. You can view more details about your transaction using a block
  explorer. Transactions are batched and processed together with other transactions in a block and block explorers allow
  anyone to view all transactions in the network.

## Evmos dApps

Alongside the applications built by the developer community, the Evmos Core Development Team also builds its own
[Evmos dApps](https://app.evmos.org). An easy way to get started is by staking and voting for governance proposals.

### Staking

Staking plays an essential role in keeping the Evmos network secure and decentralized. The Evmos network is run by a
technology that incentivizes a set of community members to keep it online and decentralized, called Proof-of-Stake.
These community members, called validators, are responsible for validating transactions in a block and are usually
very technical teams as they have to monitor and upgrade the software that runs the Evmos network. Validators are
incentivized to stay online as they earn a commission on your stake.

You can help keep the network secure and put your EVMOS tokens at "stake", by delegating them to a validator. This will
earn you a daily interest in form of new EVMOS tokens (aka. staking rewards). As long as your tokens are staked, you
cannot transfer or use them in other dApps. At any given time, you can undelegate your tokens. This will allow you to
start using your tokens again after an unbonding period that is defined in the protocol (e.g. two weeks).

![evmos_staking.png](/img/evmos_staking.png)

You can choose validators to stake your tokens with from a list on the [Evmos Staking dApp](https://app.evmos.org/staking).
We recommend choosing a reputable validator, that can promise to stay online and doesn't have a large voting power
(e.g. is not in the top 20). Validators that are offline for an extended period or act maliciously can get slashed,
resulting in a loss of your stake. Also, delegating to validators with less voting power keeps the network more
decentralized. You can switch between validators without undelegating your tokens by redelegating them.

### Governance

Evmos has an on-chain governance mechanism for making changes to the network protocol, such as changing chain parameters
(like the unbonding period for staking), spending funds from the community pool (e.g. to fund teams to build on Evmos),
or upgrading the Evmos software that is run by validators.

Developers propose changes through code updates and the community votes on whether to accept or reject the proposed change.
Every participant can vote for proposals, as long as they have staked EVMOS tokens. Each vote is weighted by the amount
of staked tokens.

![evmos_governance.png](/img/evmos_governance.png)

Evmos is known for hosting one of the most active governance communities in the blockchain space. Anyone can participate
in governance so the ownership, norms, and culture of the Evmos network are thus spread across the community. Once you
have staked your tokens, head over to the [Evmos Governance dApp](https://app.evmos.org/governance) to start voting on
governance proposals.

:::note
Engage with our community on [Commonwealth](https://commonwealth.im/evmos) and learn about upcoming proposals. If you are
interested in launching your own governance proposals, head over [here](https://academy.evmos.org/community/governance/)
for a guide.
:::
