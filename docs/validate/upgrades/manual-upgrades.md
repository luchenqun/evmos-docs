---
sidebar_position: 4
---

# 手动升级

了解如何手动升级您的节点。

## 先决条件

- [安装 evmosd](../../protocol/evmos-cli)

## 1. 升级 Evmos 版本

在升级 Evmos 版本之前，请使用 `Ctrl/Cmd+C` 停止 `evmosd` 实例。

然后，将软件升级到所需的发布版本。请查看 Evmos 的[发布页面](https://github.com/evmos/evmos/releases)以获取每个发布版本的详细信息。

:::danger
确保安装的版本与您正在运行的网络（主网或测试网）所需的版本匹配。
:::

```bash
cd evmos
git fetch --all && git checkout <new_version>
make install
```

:::tip
如果在此步骤遇到问题，请检查您是否安装了最新稳定版本的[Golang](https://golang.org/dl/)。
:::

通过使用 `version` 命令验证您已成功在系统上安装了 Evmos：

```bash
$ evmosd version --long

name: evmos
server_name: evmosd
version: 3.0.0
commit: fe9df43332800a74a163c014c69e62765d8206e3
build_tags: netgo,ledger
go: go version go1.20 darwin/amd64
...
```

:::tip
如果软件版本不匹配，请检查您的 `$PATH`，以确保正确的 `evmosd` 正在运行。
:::

## 2. 替换 Genesis 文件

:::tip
您可以在以下存储库中找到主网或测试网的最新 `genesis.json` 文件：

- **主网**：[github.com/evmos/mainnet](https://github.com/evmos/mainnet)
- **测试网**：[github.com/evmos/testnets](https://github.com/evmos/testnets)
:::

将新的 genesis 保存为 `new_genesis.json`。然后，将位于 `config/` 目录中的旧 `genesis.json` 替换为 `new_genesis.json`：

```bash
cd $HOME/.evmosd/config
cp -f genesis.json new_genesis.json
mv new_genesis.json genesis.json
```

:::tip
我们建议使用 `sha256sum` 来检查下载的 genesis 的哈希值是否与预期的 genesis 相符。

```bash
cd ~/.evmosd/config
echo "<expected_hash>  genesis.json" | sha256sum -c
```

:::

## 3. 数据重置

:::danger
如果您要升级的版本需要进行数据重置（硬分叉），请在[此处](./list-of-upgrades)检查。如果不需要进行数据重置，
您可以跳过[重新启动](https://docs.cosmos.network/main/modules/upgrade)。
:::

删除过时的文件并重置数据：

```bash
rm $HOME/.evmosd/config/addrbook.json
evmosd tendermint unsafe-reset-all --home $HOME/.evmosd
```

您的节点现在处于原始状态，同时保留了原始的 `priv_validator.json` 和 `config.toml`。如果您之前设置了任何哨兵节点或全节点，
您的节点仍然会尝试连接到它们，但如果它们没有进行升级，可能会失败。

:::danger
🚨 **重要** 🚨

确保每个节点都有一个唯一的 `priv_validator.json` 文件。**不要**将旧节点的 `priv_validator.json` 文件复制到多个新节点上。使用相同的 `priv_validator.json` 文件运行两个节点会导致[双签名](https://docs.tendermint.com/master/spec/consensus/signing.html#double-signing)。
:::

## 4. 重启节点

在新的创世块更新后，使用 `start` 命令来重启节点：

```bash
evmosd start
```


---
sidebar_position: 4
---

# Manual Upgrades

Learn how to manually upgrade your node.

## Prerequisites

- [Install evmosd](../../protocol/evmos-cli)

## 1. Upgrade the Evmos version

Before upgrading the Evmos version. Stop your instance of `evmosd` using `Ctrl/Cmd+C`.

Next, upgrade the software to the desired release version. Check the Evmos [releases page](https://github.com/evmos/evmos/releases)
for details on each release.

:::danger
Ensure that the version installed matches the one needed for the network you are running (mainnet or testnet).
:::

```bash
cd evmos
git fetch --all && git checkout <new_version>
make install
```

:::tip
If you have issues at this step, please check that you have the latest stable version of
[Golang](https://golang.org/dl/) installed.
:::

Verify that you've successfully installed Evmos on your system by using the `version` command:

```bash
$ evmosd version --long

name: evmos
server_name: evmosd
version: 3.0.0
commit: fe9df43332800a74a163c014c69e62765d8206e3
build_tags: netgo,ledger
go: go version go1.20 darwin/amd64
...
```

:::tip
If the software version does not match, then please check your `$PATH` to ensure the correct `evmosd` is running.
:::

## 2. Replace Genesis file

:::tip
You can find the latest `genesis.json` file for mainnet or testnet in the following repositories:

- **Mainnet**: [github.com/evmos/mainnet](https://github.com/evmos/mainnet)
- **Testnet**: [github.com/evmos/testnets](https://github.com/evmos/testnets)
:::

Save the new genesis as `new_genesis.json`. Then, replace the old `genesis.json` located in your `config/` directory with `new_genesis.json`:

```bash
cd $HOME/.evmosd/config
cp -f genesis.json new_genesis.json
mv new_genesis.json genesis.json
```

:::tip
We recommend using `sha256sum` to check the hash of the downloaded genesis against the expected genesis.

```bash
cd ~/.evmosd/config
echo "<expected_hash>  genesis.json" | sha256sum -c
```

:::

## 3. Data Reset

:::danger
Check [here](./list-of-upgrades) if the version you are upgrading require a data reset (hard fork). If this is not the
case, you can skip to [Restart](https://docs.cosmos.network/main/modules/upgrade).
:::

Remove the outdated files and reset the data:

```bash
rm $HOME/.evmosd/config/addrbook.json
evmosd tendermint unsafe-reset-all --home $HOME/.evmosd
```

Your node is now in a pristine state while keeping the original `priv_validator.json` and `config.toml`. If you had any
sentry nodes or full nodes setup before,
your node will still try to connect to them, but may fail if they haven't also
been upgraded.

:::danger
🚨 **IMPORTANT** 🚨

Make sure that every node has a unique `priv_validator.json`. **DO NOT** copy the `priv_validator.json` from an old node
to multiple new nodes. Running two nodes with the same `priv_validator.json` will cause you to [double sign](https://docs.tendermint.com/master/spec/consensus/signing.html#double-signing).
:::

## 4. Restart Node

To restart your node once the new genesis has been updated, use the `start` command:

```bash
evmosd start
```
