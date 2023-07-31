# 运行 IBC 中继器

## 什么是 IBC 中继器？

IBC 中继器是一种软件组件，用于在支持 Inter-Blockchain Communication (IBC) 协议的两个不同区块链网络之间进行通信。IBC 协议是一种用于在不同区块链网络之间安全可靠地传输数字资产和数据的标准。

IBC 中继器负责中继 IBC 数据包，这些数据包用于在两个不同的区块链网络之间发送消息和数据。它从一个链接收数据包，验证其真实性和有效性，然后将其中继到接收链。

## 最低要求

- 8 核 (4 物理核心)，x86_64 架构处理器
- 32 GB RAM (或等效的交换文件设置)
- 1 TB+ nVME 驱动器

如果在单个虚拟机上运行多个节点，请[确保增加打开文件限制](https://tecadmin.net/increase-open-files-limit-ubuntu/)。

## 先决条件

在开始之前，请确保在您打算进行中继的同一台机器的后台上运行着一个 Evmos 节点。如果您还没有设置 Evmos 节点，请按照[此指南](./../protocol/evmos-cli/single-node)进行设置。

在本指南中，我们将在 [Evmos (channel-3) 和 Cosmos Hub (channel-292)](https://www.mintscan.io/evmos/relayers) 之间进行中继。
在设置 Evmos 和 Cosmos 全节点时，请确保在各自链的 `app.toml` 和 `config.toml` 文件中偏移使用的端口（此过程将在下面展示）。

在此示例中，将使用 Evmos 的默认端口，并手动更改 Cosmos Hub 节点的端口。

## Evmos 守护程序设置

首先，在 `$HOME/.evmosd/config` 目录下的 `app.toml` 文件中将 `grpc server` 设置为端口 `9090`：

```bash
vim $HOME/.evmosd/config/app.toml
```

```bash
[grpc]

# Enable defines if the gRPC server should be enabled.
enable = true

# Address defines the gRPC server address to bind to.
address = "0.0.0.0:9090"
```

然后，在 `$HOME/.evmosd/config` 目录下的 `config.toml` 文件中将 `pprof_laddr` 设置为端口 `6060`，`rpc laddr` 设置为端口 `26657`，`prp laddr` 设置为 `26656`：

```bash
vim $HOME/.evmosd/config/config.toml
```

```bash
# pprof监听地址（https://golang.org/pkg/net/http/pprof）
pprof_laddr = "localhost:6060"
```

```bash
[rpc]

# TCP or UNIX socket address for the RPC server to listen on
laddr = "tcp://127.0.0.1:26657"
```

```bash
[p2p]

# Address to listen for incoming connections
laddr = "tcp://0.0.0.0:26656"
```

## Cosmos守护进程设置

首先，在`$HOME/.gaiad/config`目录下的`app.toml`文件中将`grpc server`设置为端口`9090`：

```bash
vim $HOME/.gaiad/config/app.toml
```

```bash
[grpc]

# Enable defines if the gRPC server should be enabled.
enable = true

# Address defines the gRPC server address to bind to.
address = "0.0.0.0:9092"
```

然后，在`$HOME/.gaiad/config`目录下的`config.toml`文件中将`pprof_laddr`设置为端口`6062`，`rpc laddr`设置为端口`26757`，`prp laddr`设置为`26756`：

```bash
vim $HOME/.gaiad/config/app.toml
```

```bash
# pprof监听地址（https://golang.org/pkg/net/http/pprof）
pprof_laddr = "localhost:6062"
```

```bash
[rpc]

# TCP or UNIX socket address for the RPC server to listen on
laddr = "tcp://127.0.0.1:26757"
```

```bash
[p2p]

# Address to listen for incoming connections
laddr = "tcp://0.0.0.0:26756"
```

## 安装Rust依赖

安装以下Rust依赖：

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

```bash
source $HOME/.cargo/env
sudo apt-get install pkg-config libssl-dev
```

```bash
sudo apt install librust-openssl-dev build-essential git
```

## 构建和设置Hermes

创建二进制文件将被放置的目录，克隆hermes源代码库，并使用最新版本进行构建。

```bash
mkdir -p $HOME/hermes
git clone https://github.com/informalsystems/ibc-rs.git hermes
cd hermes
git checkout v0.12.0
cargo install ibc-relayer-cli --bin hermes --locked
```

创建hermes的`config`和`keys`目录，并将`config.toml`复制到config目录：

```bash
mkdir -p $HOME/.hermes
mkdir -p $HOME/.hermes/keys
cp config.toml $HOME/.hermes
```

检查hermes的版本和配置目录设置：

```bash
$ hermes version
INFO ThreadId(01) using default configuration from '/home/relay/.hermes/config.toml'
hermes 0.12.0
```

编辑hermes配置（根据上面设置的端口配置使用相应的端口，只添加将要中继的链）：

```bash
vim $HOME/.hermes/config.toml
```

```bash
# In this example, we will set channel-292 on the cosmoshub-4 chain settings and channel-3 on the evmos_9001-2 chain settings:
[[chains]]
id = 'cosmoshub-4'
rpc_addr = 'http://127.0.0.1:26757'
grpc_addr = 'http://127.0.0.1:9092'
websocket_addr = 'ws://127.0.0.1:26757/websocket'
...
[chains.packet_filter]
policy = 'allow'
list = [
   ['transfer', 'channel-292'], # evmos_9001-2
]

[[chains]]
id = 'evmos_9001-2'
rpc_addr = 'http://127.0.0.1:26657'
grpc_addr = 'http://127.0.0.1:9090'
websocket_addr = 'ws://127.0.0.1:26657/websocket'
...
address_type = { derivation = 'ethermint', proto_type = { pk_type = '/ethermint.crypto.v1.ethsecp256k1.PubKey' } }
[chains.packet_filter]
policy = 'allow'
list = [
  ['transfer', 'channel-3'], # cosmoshub-4
]
```

将中继器钱包添加到Hermes的密钥环中（位于`$HOME/.hermes/keys`）

最佳实践是在所有网络上使用相同的助记词。不要将中继地址用于其他任何事情，因为这会导致账户序列错误。

```bash
hermes keys restore cosmoshub-4 -m "24-word mnemonic seed"
hermes keys restore evmos_9001-2 -m "24-word mnemonic seed"
```

确保该钱包在EVMOS和ATOM中都有资金，以支付中继所需的费用。

## 最终检查

验证您的Hermes配置文件：

```bash
$ hermes config validate
INFO ThreadId(01) using default configuration from '/home/relay/.hermes/config.toml'
Success: "validation passed successfully"
```

执行Hermes的`health-check`命令，查看所有连接的节点是否正常运行且同步：

```bash
$ hermes health-check
INFO ThreadId(01) using default configuration from '/home/relay/.hermes/config.toml'
INFO ThreadId(01) telemetry service running, exposing metrics at http://0.0.0.0:3001/metrics
INFO ThreadId(01) starting REST API server listening at http://127.0.0.1:3000
INFO ThreadId(01) [cosmoshub-4] chain is healthy
INFO ThreadId(01) [evmos_9001-2] chain is healthy
```

当您的节点完全同步后，可以启动Hermes守护进程：

```bash
hermes start
```

观察Hermes的输出，以查看成功中继的数据包或任何错误。它将在启动完成后尝试清除任何未接收的数据包。

## 有用的命令

使用以下命令查询Hermes中未接收的数据包和确认（即检查通道是否“清除”）：

```bash
hermes query packet unreceived-packets cosmoshub-4 transfer channel-292
hermes query packet unreceived-acks cosmoshub-4 transfer channel-292
```

```bash
hermes query packet unreceived-packets evmos_9001-2 transfer channel-3
hermes query packet unreceived-acks evmos_9001-2 transfer channel-3
```

使用以下命令查询Hermes中的数据包承诺：

```bash
hermes query packet commitments cosmoshub-4 transfer channel-292
hermes query packet commitments evmos_9001-2 transfer channel-3
```

使用以下命令清除通道（仅适用于Hermes `v0.12.0`及更高版本）：

```bash
hermes clear packets cosmoshub-4 transfer channel-292
hermes clear packets evmos_9001-2 transfer channel-3
```

使用以下命令手动清除未接收的数据包（实验性功能，需要停止Hermes守护进程以防止与账户序列混淆）：

```bash
hermes tx raw packet-recv evmos_9001-2 cosmoshub-4 transfer channel-292
hermes tx raw packet-ack evmos_9001-2 cosmoshub-4 transfer channel-292
hermes tx raw packet-recv cosmoshub-4 evmos_9001-2 transfer channel-3
hermes tx raw packet-ack cosmoshub-4 evmos_9001-2 transfer channel-3
```


---
sidebar_position: 6
---

# Run an IBC Relayer

## What is an IBC Relayer?

An IBC relayer is a software component that facilitates communication between two distinct blockchain networks that
support the Inter-Blockchain Communication (IBC) protocol. The IBC protocol is a standard for the secure and reliable
 transfer of digital assets and data across different blockchain networks.

An IBC relayer is responsible for relaying IBC packets, which are used to send messages and data between two different
 blockchain networks. It receives packets from one chain, verifies their authenticity and validity, and then relays
  them to the receiving chain.

## Minimum Requirements

- 8 core (4 physical core), x86_64 architecture processor
- 32 GB RAM (or equivalent swap file set up)
- 1 TB+ nVME drives

If running many nodes on a single VM, [ensure your open files limit is increased](https://tecadmin.net/increase-open-files-limit-ubuntu/).

## Prerequisites

Before beginning, ensure you have an Evmos node running in the background of the same machine that you intend to relay on.
 Follow [this guide](./../protocol/evmos-cli/single-node) to set up an Evmos node if you have not already.

In this guide, we will be relaying between [Evmos (channel-3) and Cosmos Hub (channel-292)](https://www.mintscan.io/evmos/relayers).
 When setting up your Evmos and Cosmos full nodes, be sure to offset the ports being used in both the `app.toml` and `config.toml`
  files of the respective chains (this process will be shown below).

In this example, the default ports for Evmos will be used, and the ports of the Cosmos Hub node will be manually changed.

## Evmos Daemon Settings

First, set `grpc server` on port `9090` in the `app.toml` file from the `$HOME/.evmosd/config` directory:

```bash
vim $HOME/.evmosd/config/app.toml
```

```bash
[grpc]

# Enable defines if the gRPC server should be enabled.
enable = true

# Address defines the gRPC server address to bind to.
address = "0.0.0.0:9090"
```

Then, set the `pprof_laddr` to port `6060`, `rpc laddr` to port `26657`, and `prp laddr` to `26656` in the `config.toml`
 file from the `$HOME/.evmosd/config` directory:

```bash
vim $HOME/.evmosd/config/config.toml
```

```bash
# pprof listen address (https://golang.org/pkg/net/http/pprof)
pprof_laddr = "localhost:6060"
```

```bash
[rpc]

# TCP or UNIX socket address for the RPC server to listen on
laddr = "tcp://127.0.0.1:26657"
```

```bash
[p2p]

# Address to listen for incoming connections
laddr = "tcp://0.0.0.0:26656"
```

## Cosmos Daemon Settings

First, set `grpc server` to port `9090` in the `app.toml` file from the `$HOME/.gaiad/config` directory:

```bash
vim $HOME/.gaiad/config/app.toml
```

```bash
[grpc]

# Enable defines if the gRPC server should be enabled.
enable = true

# Address defines the gRPC server address to bind to.
address = "0.0.0.0:9092"
```

Then, set the `pprof_laddr` to port `6062`, `rpc laddr` to port `26757`, and `prp laddr` to `26756` in the `config.toml`
 file from the `$HOME/.gaiad/config` directory:

```bash
vim $HOME/.gaiad/config/app.toml
```

```bash
# pprof listen address (https://golang.org/pkg/net/http/pprof)
pprof_laddr = "localhost:6062"
```

```bash
[rpc]

# TCP or UNIX socket address for the RPC server to listen on
laddr = "tcp://127.0.0.1:26757"
```

```bash
[p2p]

# Address to listen for incoming connections
laddr = "tcp://0.0.0.0:26756"
```

## Install Rust Dependencies

Install the following rust dependencies:

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

```bash
source $HOME/.cargo/env
sudo apt-get install pkg-config libssl-dev
```

```bash
sudo apt install librust-openssl-dev build-essential git
```

## Build & Setup Hermes

Create the directory where the binary will be placed, clone the hermes source repository, and build it using the latest release.

```bash
mkdir -p $HOME/hermes
git clone https://github.com/informalsystems/ibc-rs.git hermes
cd hermes
git checkout v0.12.0
cargo install ibc-relayer-cli --bin hermes --locked
```

Make the hermes `config` and `keys` directory, and copy `config.toml` to the config directory:

```bash
mkdir -p $HOME/.hermes
mkdir -p $HOME/.hermes/keys
cp config.toml $HOME/.hermes
```

Check the hermes version and configuration directory setup:

```bash
$ hermes version
INFO ThreadId(01) using default configuration from '/home/relay/.hermes/config.toml'
hermes 0.12.0
```

Edit the hermes configuration (use ports according the port configuration set above, adding only chains that will be relayed):

```bash
vim $HOME/.hermes/config.toml
```

```bash
# In this example, we will set channel-292 on the cosmoshub-4 chain settings and channel-3 on the evmos_9001-2 chain settings:
[[chains]]
id = 'cosmoshub-4'
rpc_addr = 'http://127.0.0.1:26757'
grpc_addr = 'http://127.0.0.1:9092'
websocket_addr = 'ws://127.0.0.1:26757/websocket'
...
[chains.packet_filter]
policy = 'allow'
list = [
   ['transfer', 'channel-292'], # evmos_9001-2
]

[[chains]]
id = 'evmos_9001-2'
rpc_addr = 'http://127.0.0.1:26657'
grpc_addr = 'http://127.0.0.1:9090'
websocket_addr = 'ws://127.0.0.1:26657/websocket'
...
address_type = { derivation = 'ethermint', proto_type = { pk_type = '/ethermint.crypto.v1.ethsecp256k1.PubKey' } }
[chains.packet_filter]
policy = 'allow'
list = [
  ['transfer', 'channel-3'], # cosmoshub-4
]
```

Add your relayer wallet to Hermes' keyring (located in `$HOME/.hermes/keys`)

The best practice is to use the same mnemonic over all networks. Do not use your relaying-addresses for anything else, because it will lead to account sequence errors.

```bash
hermes keys restore cosmoshub-4 -m "24-word mnemonic seed"
hermes keys restore evmos_9001-2 -m "24-word mnemonic seed"
```

Ensure this wallet has funds in both EVMOS and ATOM in order to pay the fees required to relay.

## Final Checks

Validate your hermes configuration file:

```bash
$ hermes config validate
INFO ThreadId(01) using default configuration from '/home/relay/.hermes/config.toml'
Success: "validation passed successfully"
```

Perform the hermes `health-check` to see if all connected nodes are up and synced:

```bash
$ hermes health-check
INFO ThreadId(01) using default configuration from '/home/relay/.hermes/config.toml'
INFO ThreadId(01) telemetry service running, exposing metrics at http://0.0.0.0:3001/metrics
INFO ThreadId(01) starting REST API server listening at http://127.0.0.1:3000
INFO ThreadId(01) [cosmoshub-4] chain is healthy
INFO ThreadId(01) [evmos_9001-2] chain is healthy
```

When your nodes are fully synced, you can start the hermes daemon:

```bash
hermes start
```

Watch hermes' output for successfully relayed packets, or any errors. It will try and clear any unrecieved packets after startup has completed.

## Helpful Commands

Query hermes for unrecieved packets and acknowledgements (ie. check if channels are "clear") with the following:

```bash
hermes query packet unreceived-packets cosmoshub-4 transfer channel-292
hermes query packet unreceived-acks cosmoshub-4 transfer channel-292
```

```bash
hermes query packet unreceived-packets evmos_9001-2 transfer channel-3
hermes query packet unreceived-acks evmos_9001-2 transfer channel-3
```

Query hermes for packet commitments with the following:

```bash
hermes query packet commitments cosmoshub-4 transfer channel-292
hermes query packet commitments evmos_9001-2 transfer channel-3
```

Clear the channel (only works on hermes `v0.12.0` and higher) with the following:

```bash
hermes clear packets cosmoshub-4 transfer channel-292
hermes clear packets evmos_9001-2 transfer channel-3
```

Clear unrecieved packets manually (experimental, will need to stop hermes daemon to prevent confusion with account sequences) with the following:

```bash
hermes tx raw packet-recv evmos_9001-2 cosmoshub-4 transfer channel-292
hermes tx raw packet-ack evmos_9001-2 cosmoshub-4 transfer channel-292
hermes tx raw packet-recv cosmoshub-4 evmos_9001-2 transfer channel-3
hermes tx raw packet-ack cosmoshub-4 evmos_9001-2 transfer channel-3
```
