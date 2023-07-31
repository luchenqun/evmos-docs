---
sidebar_position: 1
---

# 运行验证者

了解如何运行验证者节点。

## 先决条件阅读

- [验证者概述](./../) 
- [验证者安全性](./../security/validator-security) 

:::tip
如果您计划使用密钥管理系统（KMS），您应该首先完成以下步骤：[使用 KMS](./../../validate/security/tendermint-kms)。
:::

## 创建您的验证者

您的节点共识公钥（`evmosvalconspub...`）可以用于通过抵押 EVMOS 代币来创建新的验证者。您可以通过运行以下命令找到您的验证者公钥：

```bash
evmosd tendermint show-validator
```

:::danger
🚨 **危险**：<u>永远不要</u>使用[`test`](./../../protocol/concepts/keyring#testing)密钥后端创建您的主网验证者密钥。这样做可能会导致通过 `eth_sendTransaction` JSON-RPC 端点远程访问您的资金，从而导致资金损失。

参考：[安全公告：配置不安全的 geth 可以使资金远程访问](https://blog.ethereum.org/2015/08/29/security-alert-insecurely-configured-geth-can-make-funds-remotely-accessible/)
:::

要在测试网上创建您的验证者，只需使用以下命令：

```bash
evmosd tx staking create-validator \
  --amount=1000000atevmos \
  --pubkey=$(evmosd tendermint show-validator) \
  --moniker="choose a moniker" \
  --chain-id=<chain_id> \
  --commission-rate="0.05" \
  --commission-max-rate="0.10" \
  --commission-max-change-rate="0.01" \
  --min-self-delegation="1000000" \
  --gas="auto" \
  --gas-prices="0.025atevmos" \
  --from=<key_name>
```

:::tip
在指定佣金参数时，`commission-max-change-rate` 用于测量 `commission-rate` 的 % *point* 变化。例如，从 1% 到 2% 是一个 100% 的比率增加，但只有 1 个百分点。
:::

:::tip
`Min-self-delegation` 是一个严格的正整数，表示您的验证者必须始终具有的最小自委托投票权重。`min-self-delegation` 为 `1000000` 意味着您的验证者的自委托永远不会低于 `1 atevmos`
:::

您可以使用第三方浏览器确认您是否在验证者集中。

## 编辑验证者描述

您可以编辑您的验证者的公共描述。此信息用于识别您的验证者，并将被委托人依赖于此来决定委托给哪些验证者。请确保为下面的每个标志提供输入。如果命令中未包含标志，则该字段将默认为空（`--moniker` 默认为机器名称），如果该字段从未设置过，则保持不变。

<key_name>指定您要编辑的验证器。如果选择不包含某些标志，请记住必须包含--from标志以标识要更新的验证器。

`--identity`可用于与Keybase或UPort等系统验证身份。在与Keybase一起使用时，`--identity`应填入一个16位数字字符串，该字符串是使用[keybase.io](https://keybase.io)帐户生成的。这是一种在多个在线网络上验证您身份的加密安全方法。Keybase API允许我们检索您的Keybase头像。这是您如何向验证器配置文件添加徽标的方法。

```bash
evmosd tx staking edit-validator
  --moniker="choose a moniker" \
  --website="https://evmos.org" \
  --identity=6A0D65E29A4CBC8E \
  --details="To infinity and beyond!" \
  --chain-id=<chain_id> \
  --gas="auto" \
  --gas-prices="0.025atevmos" \
  --from=<key_name> \
  --commission-rate="0.10"
```

**注意**：`commission-rate`的值必须符合以下不变性：

* 必须介于0和验证器的`commission-max-rate`之间
* 不得超过验证器的`commission-max-change-rate`，即每天的最大百分点变化率。换句话说，验证器每天只能更改一次佣金，并且必须在`commission-max-change-rate`的范围内。

## 查看验证器描述

使用以下命令查看验证器的信息：

```bash
evmosd query staking validator <account_cosmos>
```

## 跟踪验证器签名信息

为了跟踪验证器过去的签名，您可以使用`signing-info`命令：

```bash
evmosd query slashing signing-info <validator-pubkey>\
  --chain-id=<chain_id>
```

## 解除验证器的监禁状态

当验证器因停机而被“监禁”时，您必须从操作员帐户提交一个`Unjail`事务，以便能够再次获得区块提议者奖励（取决于区域费用分配）。

```bash
evmosd tx slashing unjail \
  --from=<key_name> \
  --chain-id=<chain_id>
```

## 确认您的验证器正在运行

如果以下命令返回任何内容，则表示您的验证器处于活动状态：

```bash
evmosd query tendermint-validator-set | grep "$(evmosd tendermint show-address)"
```

现在，您应该在Evmos的一个浏览器中看到您的验证器。您需要在`~/.evmosd/config/priv_validator.json`文件中查找`bech32`编码的`address`。

:::warning 注意
要成为验证人集合的一部分，您需要拥有比第100个验证人更多的总投票权。
:::

## 停止您的验证人

当尝试进行常规维护或计划即将进行的协调升级时，有序地停止您的验证人可能会很有用。您可以通过将 `halt-height` 设置为您希望节点关闭的高度，或者通过向 `evmosd` 传递 `--halt-height` 标志来实现这一点。节点将在达到给定高度后提交区块后以零退出代码关闭。

## 常见问题

### 问题＃1：我的验证人的 `voting_power: 0`

您的验证人已被监禁。如果验证人在最近的 `10000` 个区块中没有对 `500` 个区块进行投票，或者如果验证人双重签名，则会被监禁，即从活跃的验证人集合中移除。

如果您因停机而被监禁，您可以将您的投票权恢复到您的验证人。首先，如果 `evmosd` 没有运行，请重新启动它：

```bash
evmosd start
```

等待您的全节点追上最新的区块。然后，您可以[解除监禁您的验证人](#unjail-validator)。

最后，再次检查您的验证人，以查看您的投票权是否恢复。

```bash
evmosd status
```

您可能会注意到您的投票权少于以前。这是因为您因停机而被处罚！

### 问题＃2：我的节点因为 `too many open files` 而崩溃

Linux 每个进程可以打开的文件数的默认值为 `1024`。已知 `evmosd` 会打开超过 `1024` 个文件。这会导致进程崩溃。一个快速的解决方法是运行 `ulimit -n 4096`（增加允许打开的文件数），然后使用 `evmosd start` 重新启动进程。如果您使用 `systemd` 或其他进程管理器来启动 `evmosd`，则可能需要在该级别进行一些配置。以下是一个修复此问题的示例 `systemd` 文件：

```toml
# /etc/systemd/system/evmosd.service
[Unit]
Description=Evmos Node
After=network.target

[Service]
Type=simple
User=ubuntu
WorkingDirectory=/home/ubuntu
ExecStart=/home/ubuntu/go/bin/evmosd start
Restart=on-failure
RestartSec=3
LimitNOFILE=4096

[Install]
WantedBy=multi-user.target
```


---
sidebar_position: 1
---

# Run a Validator

Learn how to run a validator node.

## Prerequisite Readings

- [Validator Overview](./../) 
- [Validator Security](./../security/validator-security) 

:::tip
If you plan to use a Key Management System (KMS), you should go through these steps first: [Using a KMS](./../../validate/security/tendermint-kms).
:::

## Create Your Validator

Your node consensus public key (`evmosvalconspub...`) can be used to create a new validator by staking EVMOS tokens. You can find your validator pubkey by running:

```bash
evmosd tendermint show-validator
```

:::danger
🚨 **DANGER**: <u>Never</u> create your mainnet validator keys using a [`test`](./../../protocol/concepts/keyring#testing) keying backend. Doing so might result in a loss of funds by making your funds remotely accessible via the `eth_sendTransaction` JSON-RPC endpoint.

Ref: [Security Advisory: Insecurely configured geth can make funds remotely accessible](https://blog.ethereum.org/2015/08/29/security-alert-insecurely-configured-geth-can-make-funds-remotely-accessible/)
:::

To create your validator on testnet, just use the following command:

```bash
evmosd tx staking create-validator \
  --amount=1000000atevmos \
  --pubkey=$(evmosd tendermint show-validator) \
  --moniker="choose a moniker" \
  --chain-id=<chain_id> \
  --commission-rate="0.05" \
  --commission-max-rate="0.10" \
  --commission-max-change-rate="0.01" \
  --min-self-delegation="1000000" \
  --gas="auto" \
  --gas-prices="0.025atevmos" \
  --from=<key_name>
```

:::tip
When specifying commission parameters, the `commission-max-change-rate` is used to measure % *point* change over the `commission-rate`. E.g. 1% to 2% is a 100% rate increase, but only 1 percentage point.
:::

:::tip
`Min-self-delegation` is a strictly positive integer that represents the minimum amount of self-delegated voting power your validator must always have. A `min-self-delegation` of `1000000` means your validator will never have a self-delegation lower than `1 atevmos`
:::

You can confirm that you are in the validator set by using a third party explorer.

## Edit Validator Description

You can edit your validator's public description. This info is to identify your validator, and will be relied on by delegators to decide which validators to stake to. Make sure to provide input for every flag below. If a flag is not included in the command the field will default to empty (`--moniker` defaults to the machine name) if the field has never been set or remain the same if it has been set in the past.

The <key_name> specifies which validator you are editing. If you choose to not include certain flags, remember that the --from flag must be included to identify the validator to update.

The `--identity` can be used as to verify identity with systems like Keybase or UPort. When using with Keybase `--identity` should be populated with a 16-digit string that is generated with a [keybase.io](https://keybase.io) account. It's a cryptographically secure method of verifying your identity across multiple online networks. The Keybase API allows us to retrieve your Keybase avatar. This is how you can add a logo to your validator profile.

```bash
evmosd tx staking edit-validator
  --moniker="choose a moniker" \
  --website="https://evmos.org" \
  --identity=6A0D65E29A4CBC8E \
  --details="To infinity and beyond!" \
  --chain-id=<chain_id> \
  --gas="auto" \
  --gas-prices="0.025atevmos" \
  --from=<key_name> \
  --commission-rate="0.10"
```

**Note**: The `commission-rate` value must adhere to the following invariants:

* Must be between 0 and the validator's `commission-max-rate`
* Must not exceed the validator's `commission-max-change-rate` which is maximum
  % point change rate **per day**. In other words, a validator can only change
  its commission once per day and within `commission-max-change-rate` bounds.

## View Validator Description

View the validator's information with this command:

```bash
evmosd query staking validator <account_cosmos>
```

## Track Validator Signing Information

In order to keep track of a validator's signatures in the past you can do so by using the `signing-info` command:

```bash
evmosd query slashing signing-info <validator-pubkey>\
  --chain-id=<chain_id>
```

## Unjail Validator

When a validator is "jailed" for downtime, you must submit an `Unjail` transaction from the operator account in order to be able to get block proposer rewards again (depends on the zone fee distribution).

```bash
evmosd tx slashing unjail \
  --from=<key_name> \
  --chain-id=<chain_id>
```

## Confirm Your Validator is Running

Your validator is active if the following command returns anything:

```bash
evmosd query tendermint-validator-set | grep "$(evmosd tendermint show-address)"
```

You should now see your validator in one of Evmos explorers. You are looking for the `bech32` encoded `address` in the `~/.evmosd/config/priv_validator.json` file.

:::warning Note
To be in the validator set, you need to have more total voting power than the 100th validator.
:::

## Halting Your Validator

When attempting to perform routine maintenance or planning for an upcoming coordinated
upgrade, it can be useful to have your validator systematically and gracefully halt.
You can achieve this by either setting the `halt-height` to the height at which
you want your node to shutdown or by passing the `--halt-height` flag to `evmosd`.
The node will shutdown with a zero exit code at that given height after committing
the block.

## Common Problems

### Problem #1: My validator has `voting_power: 0`

Your validator has become jailed. Validators get jailed, i.e. get removed from the active validator set, if they do not vote on `500` of the last `10000` blocks, or if they double sign.

If you got jailed for downtime, you can get your voting power back to your validator. First, if `evmosd` is not running, start it up again:

```bash
evmosd start
```

Wait for your full node to catch up to the latest block. Then, you can [unjail your validator](#unjail-validator)

Lastly, check your validator again to see if your voting power is back.

```bash
evmosd status
```

You may notice that your voting power is less than it used to be. That's because you got slashed for downtime!

### Problem #2: My node crashes because of `too many open files`

The default number of files Linux can open (per-process) is `1024`. `evmosd` is known to open more than `1024` files. This causes the process to crash. A quick fix is to run `ulimit -n 4096` (increase the number of open files allowed) and then restart the process with `evmosd start`. If you are using `systemd` or another process manager to launch `evmosd` this may require some configuration at that level. A sample `systemd` file to fix this issue is below:

```toml
# /etc/systemd/system/evmosd.service
[Unit]
Description=Evmos Node
After=network.target

[Service]
Type=simple
User=ubuntu
WorkingDirectory=/home/ubuntu
ExecStart=/home/ubuntu/go/bin/evmosd start
Restart=on-failure
RestartSec=3
LimitNOFILE=4096

[Install]
WantedBy=multi-user.target
```