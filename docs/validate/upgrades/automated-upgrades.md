---
sidebar_position: 3
---

# 自动升级

我们强烈建议验证人使用 Cosmovisor 来运行他们的节点。这将使低停机时间的升级更加顺畅，因为验证人不需要在升级期间[手动升级](./manual-upgrades)二进制文件。相反，用户可以[预先安装](#manual-download)新的二进制文件，Cosmovisor 将根据链上的软件升级提案自动更新它们。

>[`cosmovisor`](https://docs.cosmos.network/main/tooling/cosmovisor) 是一个用于监视治理模块中链上升级提案的 Cosmos SDK 应用程序二进制文件的小型进程管理器。如果它看到一个被批准的提案，cosmovisor 可以自动下载新的二进制文件，停止当前的二进制文件，从旧的二进制文件切换到新的二进制文件，最后使用新的二进制文件重新启动节点。

## 先决条件

- [安装 Cosmovisor](https://docs.cosmos.network/main/tooling/cosmovisor#installation)

## 1. 设置 Cosmovisor

设置 Cosmovisor 的环境变量。我们建议在您的 `.profile` 文件中设置这些变量，这样它们会在每个会话中自动设置。

```bash
echo "# Setup Cosmovisor" >> ~/.profile
echo "export DAEMON_NAME=evmosd" >> ~/.profile
echo "export DAEMON_HOME=$HOME/.evmosd" >> ~/.profile
source ~/.profile
```

完成后，您必须在 `DAEMON_HOME` 目录 (`~/.evmosd`) 中创建必要的文件夹，并复制当前的二进制文件到其中。

```bash
mkdir -p ~/.evmosd/cosmovisor
mkdir -p ~/.evmosd/cosmovisor/genesis
mkdir -p ~/.evmosd/cosmovisor/genesis/bin
mkdir -p ~/.evmosd/cosmovisor/upgrades

cp $GOPATH/bin/evmosd ~/.evmosd/cosmovisor/genesis/bin
```

要检查您是否正确执行了这些步骤，请确保您的 `cosmovisor` 和 `evmosd` 的版本相同：

```bash
cosmovisor run version
evmosd version
```

## 2. 下载 Evmos 发行版

### 手动下载

Cosmovisor 将持续轮询 `$DAEMON_HOME/data/upgrade-info.json`，以获取新的升级指令。当一个升级被[发布](https://github.com/evmos/evmos/releases)时，节点操作员需要执行以下操作：

1. 下载（**不要安装**）新版本的二进制文件
2. 将其放置在 `$DAEMON_HOME/cosmovisor/upgrades/<name>/bin` 目录下，其中 `<name>` 是在软件升级计划中指定的升级的 URI 编码名称。

**示例**：对于具有名称 `v3.0.0` 和以下 `upgrade-info.json` 的 `Plan`：

```json
{
    "binaries": {
        "darwin/arm64": "https://github.com/evmos/evmos/releases/download/v3.0.0/evmos_3.0.0_Darwin_arm64.tar.gz",
        "darwin/x86_64": "https://github.com/evmos/evmos/releases/download/v3.0.0/evmos_3.0.0_Darwin_x86_64.tar.gz",
        "linux/arm64": "https://github.com/evmos/evmos/releases/download/v3.0.0/evmos_3.0.0_Linux_arm64.tar.gz",
        "linux/amd64": "https://github.com/evmos/evmos/releases/download/v3.0.0/evmos_3.0.0_Linux_amd64.tar.gz",
        "windows/x86_64": "https://github.com/evmos/evmos/releases/download/v3.0.0/evmos_3.0.0_Windows_x86_64.zip"
    }
}
```

您的 `cosmovisor/` 目录应该如下所示：

```shell
cosmovisor/
├── current/   # either genesis or upgrades/<name>
├── genesis
│   └── bin
│       └── evmosd
└── upgrades
    └── v3.0.0
        ├── bin
        │   └── evmosd
        └── upgrade-info.json
```

### 自动下载

:::warning
**注意**：自动下载不会提前验证二进制文件是否可用。如果下载二进制文件出现任何问题，`cosmovisor` 将停止并不会重新启动链（这可能导致链停止）。
:::

可以让 Cosmovisor [自动下载](https://docs.cosmos.network/main/tooling/cosmovisor#auto-download) 新的二进制文件。验证人可以使用自动下载选项，在升级过程中避免不必要的停机时间。一旦链在提议的 `upgrade-height` 处停止，此选项将自动使用升级二进制文件重新启动链。此选项的主要好处是验证人可以提前准备好升级二进制文件，然后在升级时放松。

要设置自动下载，请设置以下环境变量：

```bash
echo "export DAEMON_ALLOW_DOWNLOAD_BINARIES=true" >> ~/.profile
```

## 3. 启动您的节点

现在一切都设置好了，您可以启动您的节点。

```bash
cosmovisor run start
```

您需要一种方式来始终保持进程运行。如果您使用的是 Linux，可以通过创建一个服务来实现。

```bash
sudo tee /etc/systemd/system/evmosd.service > /dev/null <<EOF
[Unit]
Description=Evmos Daemon
After=network-online.target

[Service]
User=$USER
ExecStart=$(which cosmovisor) start
Restart=always
RestartSec=3
LimitNOFILE=infinity

Environment="DAEMON_HOME=$HOME/.evmosd"
Environment="DAEMON_NAME=evmosd"
Environment="DAEMON_ALLOW_DOWNLOAD_BINARIES=false"
Environment="DAEMON_RESTART_AFTER_UPGRADE=true"

[Install]
WantedBy=multi-user.target
EOF
```

然后更新并启动节点

```bash
sudo -S systemctl daemon-reload
sudo -S systemctl enable evmosd
sudo -S systemctl start evmosd
```

您可以使用以下命令检查状态：

```bash
systemctl status evmosd
```


---
sidebar_position: 3
---

# Automated Upgrades

We highly recommend validators use Cosmovisor to run their nodes. This will make low-downtime upgrades smoother, as validators don't have to [manually upgrade](./manual-upgrades) binaries during the upgrade. Instead users can [pre-install](#manual-download) new binaries, and Cosmovisor will automatically update them based on on-chain Software Upgrade proposals.

>[`cosmovisor`](https://docs.cosmos.network/main/tooling/cosmovisor) is a small process manager for Cosmos SDK application binaries that monitors the governance module for incoming chain upgrade proposals. If it sees a proposal that gets approved, cosmovisor can automatically download the new binary, stop the current binary, switch from the old binary to the new one, and finally restart the node with the new binary.

## Prerequisites

- [Install Cosmovisor](https://docs.cosmos.network/main/tooling/cosmovisor#installation)

## 1. Setup Cosmovisor

Set up the Cosmovisor environment variables. We recommend setting these in your `.profile` so it is automatically set in every session.

```bash
echo "# Setup Cosmovisor" >> ~/.profile
echo "export DAEMON_NAME=evmosd" >> ~/.profile
echo "export DAEMON_HOME=$HOME/.evmosd" >> ~/.profile
source ~/.profile
```

After this, you must make the necessary folders for `cosmosvisor` in your `DAEMON_HOME` directory (`~/.evmosd`) and copy over the current binary.

```bash
mkdir -p ~/.evmosd/cosmovisor
mkdir -p ~/.evmosd/cosmovisor/genesis
mkdir -p ~/.evmosd/cosmovisor/genesis/bin
mkdir -p ~/.evmosd/cosmovisor/upgrades

cp $GOPATH/bin/evmosd ~/.evmosd/cosmovisor/genesis/bin
```

To check that you did this correctly, ensure your versions of `cosmovisor` and `evmosd` are the same:

```bash
cosmovisor run version
evmosd version
```

## 2. Download the Evmos release

### Manual Download

Cosmovisor will continually poll the `$DAEMON_HOME/data/upgrade-info.json` for new upgrade instructions. When an upgrade is [released](https://github.com/evmos/evmos/releases), node operators need to:

1. Download (**NOT INSTALL**) the binary for the new release
2. Place it under `$DAEMON_HOME/cosmovisor/upgrades/<name>/bin`, where `<name>` is the URI-encoded name of the upgrade as specified in the Software Upgrade Plan.

**Example**: for a `Plan` with name `v3.0.0` with the following `upgrade-info.json`:

```json
{
    "binaries": {
        "darwin/arm64": "https://github.com/evmos/evmos/releases/download/v3.0.0/evmos_3.0.0_Darwin_arm64.tar.gz",
        "darwin/x86_64": "https://github.com/evmos/evmos/releases/download/v3.0.0/evmos_3.0.0_Darwin_x86_64.tar.gz",
        "linux/arm64": "https://github.com/evmos/evmos/releases/download/v3.0.0/evmos_3.0.0_Linux_arm64.tar.gz",
        "linux/amd64": "https://github.com/evmos/evmos/releases/download/v3.0.0/evmos_3.0.0_Linux_amd64.tar.gz",
        "windows/x86_64": "https://github.com/evmos/evmos/releases/download/v3.0.0/evmos_3.0.0_Windows_x86_64.zip"
    }
}
```

Your `cosmovisor/` directory should look like this:

```shell
cosmovisor/
├── current/   # either genesis or upgrades/<name>
├── genesis
│   └── bin
│       └── evmosd
└── upgrades
    └── v3.0.0
        ├── bin
        │   └── evmosd
        └── upgrade-info.json
```

### Automatic Download

:::warning
**NOTE**: Auto-download doesn't verify in advance if a binary is available. If there will be any issue with downloading a binary, `cosmovisor` will stop and won't restart an the chain (which could lead it to a halt).
:::

It is possible to have Cosmovisor [automatically download](https://docs.cosmos.network/main/tooling/cosmovisor#auto-download) the new binary. Validators can use the automatic download option to prevent unnecessary downtime during the upgrade process. This option will automatically restart the chain with the upgrade binary once the chain has halted at the proposed `upgrade-height`. The major benefit of this option is that validators can prepare the upgrade binary in advance and then relax at the time of the upgrade.

To set the auto-download use set the following environment variable:

```bash
echo "export DAEMON_ALLOW_DOWNLOAD_BINARIES=true" >> ~/.profile
```

## 3. Start your node

Now that everything is setup and ready to go, you can start your node.

```bash
cosmovisor run start
```

You will need some way to keep the process always running. If you're on linux, you can do this by creating a service.

```bash
sudo tee /etc/systemd/system/evmosd.service > /dev/null <<EOF
[Unit]
Description=Evmos Daemon
After=network-online.target

[Service]
User=$USER
ExecStart=$(which cosmovisor) start
Restart=always
RestartSec=3
LimitNOFILE=infinity

Environment="DAEMON_HOME=$HOME/.evmosd"
Environment="DAEMON_NAME=evmosd"
Environment="DAEMON_ALLOW_DOWNLOAD_BINARIES=false"
Environment="DAEMON_RESTART_AFTER_UPGRADE=true"

[Install]
WantedBy=multi-user.target
EOF
```

Then update and start the node

```bash
sudo -S systemctl daemon-reload
sudo -S systemctl enable evmosd
sudo -S systemctl start evmosd
```

You can check the status with:

```bash
systemctl status evmosd
```