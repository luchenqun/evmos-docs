---
sidebar_position: 3
---
# 磁盘使用优化

自定义配置设置以降低验证节点的磁盘需求。

区块链数据库随时间增长，取决于区块速度和交易数量等因素。对于 Evmos，前两周的磁盘使用量接近 100GB。

有一些配置可以显著减少所需的磁盘使用量。其中一些更改只有在配置并从头开始同步时才会完全生效。

## 索引

如果您不需要从特定节点查询交易，可以禁用索引。在 `config.toml` 中设置

```toml
indexer = "null"
```

如果您在已同步的节点上执行此操作，收集的索引不会自动清除，您需要手动删除它。索引位于数据库目录下，名称为 `data/tx_index.db/`。

## 状态同步快照

我相信 Evmos 默认情况下已禁用了此功能，但在此处列出以防万一。在 `app.toml` 中设置

```toml
snapshot-interval = 0
```

请注意，如果状态同步在网络上启用并正常工作，它将允许在几分钟内同步新节点。但是，该节点将没有历史记录。

## 配置修剪

默认情况下，每隔 500 个状态和最后 100 个状态会被保留。这会在长期运行中占用大量磁盘空间，并且可以通过以下自定义配置进行优化：

```toml
pruning = "custom"
pruning-keep-recent = "100"
pruning-keep-every = "0"
pruning-interval = "10"
```

配置 `pruning-keep-recent = "0"` 可能听起来很诱人，但如果 `evmosd` 因任何原因被终止，这将导致数据库损坏。因此，建议保留最近的几个状态。

## 日志记录

默认情况下，日志记录级别设置为 `info`，这会产生大量日志。当启动节点以确保同步正常进行时，此日志级别可能很好。然而，在您看到同步顺利进行后，可以将日志级别降低为 `warn`（或 `error`）。在 `config.toml` 中设置如下内容

```toml
log_level = "warn"
```

还要确保正确配置日志轮换。

## 结果

以下是 Evmos Arsia Mons 测试网运行两周后的磁盘使用情况。默认配置下，磁盘使用量为90GB。

```bash
5.3G    ./state.db
70G     ./application.db
20K     ./snapshots/metadata.db
24K     ./snapshots
9.0G    ./blockstore.db
20K     ./evidence.db
1018M   ./cs.wal
4.7G    ./tx_index.db
90G     .
```

通过优化配置，磁盘使用量减少到17GB。

```bash
17G     .
1.1G    ./cs.wal
946M    ./application.db
20K     ./evidence.db
9.1G    ./blockstore.db
24K     ./snapshots
20K     ./snapshots/metadata.db
5.3G    ./state.db
```


---
sidebar_position: 3
---
# Disk Usage Optimization

Customize the configuration settings to lower the disk requirements for your validator node.

Blockchain database tends to grow over time, depending e.g. on block
speed and transaction amount. For Evmos, we are talking about close to
100GB of disk usage in first two weeks.

There are few configurations that can be done to reduce the required
disk usage quite significantly. Some of these changes take full effect
only when you do the configuration and start syncing from start with
them in use.

## Indexing

If you do not need to query transactions from the specific node, you can
disable indexing. On `config.toml` set

```toml
indexer = "null"
```

If you do this on already synced node, the collected index is not purged
automatically, you need to delete it manually. The index is located
under the database directory with name `data/tx_index.db/`.

## State-sync snapshots

I believe this was disabled by default on Evmos, but listing it in any
case here. On `app.toml` set

```toml
snapshot-interval = 0
```

Note that if state-sync was enabled on the network and working properly,
it would allow one to sync a new node in few minutes. But this node
would not have the history.

## Configure pruning

By default every 500th state, and the last 100 states are kept. This
consumes a lot of disk space on long run, and can be optimized with
following custom configuration:

```toml
pruning = "custom"
pruning-keep-recent = "100"
pruning-keep-every = "0"
pruning-interval = "10"
```

Configuring `pruning-keep-recent = "0"` might sound tempting, but this
will risk database corruption if the `evmosd` is killed for any reason.
Thus, it is recommended to keep the few latest states.

## Logging

By default the logging level is set to `info`, and this produces a lot of
logs. This log level might be good when starting up to see that the
node starts syncing properly. However, after you see the syncing is
going smoothly, you can lower the log level to `warn` (or `error`). On
`config.toml` set the following

```toml
log_level = "warn"
```

Also ensure your log rotation is configured properly.

## Results

Below is the disk usage after two weeks of Evmos Arsia Mons testnet. The default
configuration results in disk usage of 90GB.

```bash
5.3G    ./state.db
70G     ./application.db
20K     ./snapshots/metadata.db
24K     ./snapshots
9.0G    ./blockstore.db
20K     ./evidence.db
1018M   ./cs.wal
4.7G    ./tx_index.db
90G     .
```

This optimized configuration has reduced the disk usage to 17 GB.

```bash
17G     .
1.1G    ./cs.wal
946M    ./application.db
20K     ./evidence.db
9.1G    ./blockstore.db
24K     ./snapshots
20K     ./snapshots/metadata.db
5.3G    ./state.db
```