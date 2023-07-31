---
sidebar_position: 5
---

# 常见问题

## 钱包

<details>

<summary><b>你推荐哪个钱包用于 Evmos？</b></summary>

有很多钱包可供选择，但支持最广泛的顶级钱包是 [Metamask](https://metamask.io/) 和 [Keplr](https://www.keplr.app/)。
Evmos 是基于 Cosmos SDK 构建的 EVM 链，Metamask 不支持非 EVM 特定资产，而 Keplr 钱包支持。Keplr 钱包将很快支持 ERC-20。

</details>

<details>

<summary><b>我可以使用我的 Ledger 设备吗？</b></summary>

当然可以！请查看 [Ledger](./connect-your-wallet/keplr) 获取更多信息。Metamask、Keplr 和 WalletConnect 都与 Ledger 兼容。
在与 Evmos 上的 dApps 和产品进行交互之前，需要进行 Ledger 设置。

</details>

<details>

<summary><b>对于某些钱包，我看到同时显示 bech32 和 hex，而其他钱包只显示 hex 格式的地址，我应该使用哪个？</b></summary>

Evmos 网络支持两种格式：bech32 和 hex。其他 EVM 节点及其生态系统使用 hex 编码，而 Cosmos 本地使用 bech32 格式的地址。
Keplr 是独特的，EVM 兼容链同时显示两种格式。如果您要发送代币（通过 [IBC](https://www.mintscan.io/evmos/relayers)），
您将使用 bech32 格式的地址，除非接收链支持 EVM（即基于 Ethermint 的链）。您可以在此处了解更多详细信息（./../protocol/concepts/accounts）。

</details>

## 入口

<details>

<summary><b>我在哪里可以获取 EVMOS 代币？</b></summary>

用户可以通过以下几种途径获取 EVMOS 代币。

- 去中心化交易所：[Osmosis](https://app.osmosis.zone/?from=ATOM&to=EVMOS)
- [C14 Money](https://pay.c14.money/) 是一个入口服务
- [测试网水龙头](https://faucet.evmos.dev/) 发放少量测试网代币

</details>


---
sidebar_position: 5
---

# Frequently Asked Questions

## Wallets

<details>

<summary><b>Which wallet would you recommend for Evmos?</b></summary>

There are many wallets to select from but the top wallets with the widest support are [Metamask](https://metamask.io/)
and [Keplr](https://www.keplr.app/). Evmos is an EVM chain built on top of the Cosmos SDK and Metamask does not support
non EVM-specific assets while Keplr wallet does. Keplr wallet will soon support ERC-20.

</details>

<details>

<summary><b>Can I use my Ledger device?</b></summary>

Absolutely! Take a look at the [Ledger](./connect-your-wallet/keplr) for more information. Metamask,
Keplr, and WalletConnect all work with Ledger. Ledger setup will be required before engaging with the dApps and products on Evmos.

</details>

<details>

<summary><b>For certain wallets, I see both bech32 and hex while others only show hex formatted addresses, which should
 I use?</b></summary>

The Evmos network supports both formats: bech32 and hex. Other EVM peers and its ecosystem uses hex encoding while
Cosmos-native uses bech32 formatted addresses. Keplr is unique and the EVM-compatible chains shows both formats. If you
are sending tokens (via [IBC](https://www.mintscan.io/evmos/relayers)), you will use bech32 formatted addresses unless
the receiving chain support EVM (i.e. Ethermint-based chains). You can further details [here](./../protocol/concepts/accounts).

</details>

## Onramp

<details>

<summary><b>Where can I acquire EVMOS token?</b></summary>

There are several paths users can take to acquire EVMOS Token.

- Decentralized Exchanges: [Osmosis](https://app.osmosis.zone/?from=ATOM&to=EVMOS)
- [C14 Money](https://pay.c14.money/) is an onramp service
- [Testnet Faucet](https://faucet.evmos.dev/) dispenses a small amount of testnet tokens

</details>
