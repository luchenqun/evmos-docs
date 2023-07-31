---
sidebar_position: 2
---

# 配置

## 区块时间

节点配置中的 timeout-commit 值定义了在提交一个区块后等待多长时间，
然后开始新的高度
（这给了我们一个机会来接收更多的预提交，即使我们已经有了 +2/3）。
当前的默认值是 `"3s"`。

:::tip
**注意**：从 v6 开始，服务器在初始化节点时会自动处理此问题。
验证人需要确保他们的本地节点配置以加快网络速度到约 4 秒的区块时间。
:::

```toml
# In ~/.evmosd/config/config.toml

#######################################################
###         Consensus Configuration Options         ###
#######################################################
[consensus]

### ... 

# How long we wait after committing a block, before starting on the new
# height (this gives us a chance to receive some more precommits, even
# though we already have +2/3).
timeout_commit = "3s"
```

## 节点

在 `~/.evmosd/config/config.toml` 中，您可以设置您的节点。

请参阅我们文档中的 [添加持久节点部分](./../testnet#add-persistent-peers) 以获取自动化方法，但
字段应该类似于一串逗号分隔的节点（不要复制这个，只是一个示例）：

```bash
persistent_peers = "5576b0160761fe81ccdf88e06031a01bc8643d51@195.201.108.97:24656,13e850d14610f966de38fc2f925f6dc35c7f4bf4@176.9.60.27:26656,38eb4984f89899a5d8d1f04a79b356f15681bb78@18.169.155.159:26656,59c4351009223b3652674bd5ee4324926a5a11aa@51.15.133.26:26656,3a5a9022c8aa2214a7af26ebbfac49b77e34e5c5@65.108.1.46:26656,4fc0bea2044c9fd1ea8cc987119bb8bdff91aaf3@65.21.246.124:26656,6624238168de05893ca74c2b0270553189810aa7@95.216.100.80:26656,9d247286cd407dc8d07502240245f836e18c0517@149.248.32.208:26656,37d59371f7578101dee74d5a26c86128a229b8bf@194.163.172.168:26656,b607050b4e5b06e52c12fcf2db6930fd0937ef3b@95.217.107.96:26656,7a6bbbb6f6146cb11aebf77039089cd038003964@94.130.54.247:26656"
```

### 共享您的节点

您可以使用 `tendermint show-node-id` 命令查看和共享您的节点

```bash
evmosd tendermint show-node-id
ac29d21d0a6885465048a4481d16c12f59b2e58b
```

- **节点格式**：`node-id@ip:port`
- **示例**：`ac29d21d0a6885465048a4481d16c12f59b2e58b@143.198.224.124:26656`

### 健康节点

如果您仅依赖种子节点而没有持久节点或只有少量持久节点，
请增加 `config.toml` 中的以下参数：

```bash
# Maximum number of inbound peers
max_num_inbound_peers = 120

# Maximum number of outbound peers to connect to, excluding persistent peers
max_num_outbound_peers = 60
```

Please paste the Markdown content here.


---
sidebar_position: 2
---

# Configuration

## Block Time

The timeout-commit value in the node config defines how long we wait after committing a block,
before starting on the new height
(this gives us a chance to receive some more pre-commits, even though we already have +2/3).
The current default value is `"3s"`.

:::tip
**Note**: From v6, this is handled automatically by the server when initializing the node.
Validators will need to ensure their local node configurations in order to speed up the network to ~4s block times.
:::

```toml
# In ~/.evmosd/config/config.toml

#######################################################
###         Consensus Configuration Options         ###
#######################################################
[consensus]

### ... 

# How long we wait after committing a block, before starting on the new
# height (this gives us a chance to receive some more precommits, even
# though we already have +2/3).
timeout_commit = "3s"
```

## Peers

In `~/.evmosd/config/config.toml` you can set your peers.

See the [Add persistent peers section](./../testnet#add-persistent-peers) in our docs for an automated method, but
field should look something like a comma separated string of peers (do not copy this, just an example):

```bash
persistent_peers = "5576b0160761fe81ccdf88e06031a01bc8643d51@195.201.108.97:24656,13e850d14610f966de38fc2f925f6dc35c7f4bf4@176.9.60.27:26656,38eb4984f89899a5d8d1f04a79b356f15681bb78@18.169.155.159:26656,59c4351009223b3652674bd5ee4324926a5a11aa@51.15.133.26:26656,3a5a9022c8aa2214a7af26ebbfac49b77e34e5c5@65.108.1.46:26656,4fc0bea2044c9fd1ea8cc987119bb8bdff91aaf3@65.21.246.124:26656,6624238168de05893ca74c2b0270553189810aa7@95.216.100.80:26656,9d247286cd407dc8d07502240245f836e18c0517@149.248.32.208:26656,37d59371f7578101dee74d5a26c86128a229b8bf@194.163.172.168:26656,b607050b4e5b06e52c12fcf2db6930fd0937ef3b@95.217.107.96:26656,7a6bbbb6f6146cb11aebf77039089cd038003964@94.130.54.247:26656"
```

### Sharing your Peer

You can see and share your peer with the `tendermint show-node-id` command

```bash
evmosd tendermint show-node-id
ac29d21d0a6885465048a4481d16c12f59b2e58b
```

- **Peer Format**: `node-id@ip:port`
- **Example**: `ac29d21d0a6885465048a4481d16c12f59b2e58b@143.198.224.124:26656`

### Healthy peers

If you are relying on just seed node and no persistent peers or a low amount of them,
please increase the following params in the `config.toml`:

```bash
# Maximum number of inbound peers
max_num_inbound_peers = 120

# Maximum number of outbound peers to connect to, excluding persistent peers
max_num_outbound_peers = 60
```
