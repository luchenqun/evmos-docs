---
sidebar_position: 4
---
# 状态同步

了解Tendermint Core状态同步和Cosmos SDK提供的支持。

:::tip
**注意**: 只对如何将节点与网络同步感兴趣？跳转到[此部分](#state-syncing-a-node)。
:::

## Tendermint Core状态同步

状态同步允许新节点通过获取网络在最近高度的状态快照来加入网络，而不是获取和重放所有历史区块。
由于应用状态比所有区块的组合要小，并且恢复状态比重放区块更快，这将将与网络同步的时间从几天减少到几分钟。

本文档的此部分提供了Tendermint状态同步协议的简要概述以及如何同步节点的方法。
有关更多详细信息，请参阅
[ABCI应用程序指南](https://docs.tendermint.com/master/spec/abci/apps.html#state-sync)
和[ABCI参考文档](https://docs.tendermint.com/master/spec/abci/abci.html)。

### 状态同步快照

在设计Tendermint状态同步时的一个指导原则是尽可能给应用程序提供灵活性。
因此，Tendermint不关心快照包含什么，如何获取快照，以及如何恢复快照。
它只关心在网络中发现现有快照，获取它们，并通过ABCI将它们传递给应用程序。
Tendermint使用轻客户端验证来检查恢复的应用程序的最终应用哈希与链应用哈希是否一致，但任何进一步的验证都必须由应用程序在恢复过程中完成。

快照由任意格式的二进制块组成。
块的大小不能超过16 MB，但没有其他限制。
通过ABCI和P2P交换的[快照元数据](https://docs.tendermint.com/master/spec/abci/abci.html#snapshot)包含以下字段：

- `height` (`uint64`): 快照所在的高度
- `format` (`uint32`): 任意应用程序特定的格式标识符（例如版本）
- `chunks` (`uint32`): 快照中的二进制块数量
- `hash` (`bytes`): 用于在节点之间比较快照的任意快照哈希
- `metadata` (`bytes`): 用于应用程序使用的任意二进制快照元数据

`format`字段允许应用以向后兼容的方式更改快照格式，通过提供多种格式的快照，并在恢复过程中选择接受哪些格式。当例如更改序列化或压缩格式时，这非常有用：因为节点可能能够向运行旧版本的对等方提供快照，或者在使用新版本启动时使用旧快照。

`hash`字段包含任意的快照哈希。具有相同`metadata`字段（包括`hash`）的快照在节点之间被视为相同，并且`chunks`将从这些节点中获取。`hash`不能被信任，并且不会被Tendermint本身验证，这可以防止快照生成中的无意的非确定性。应用程序可以验证`hash`。

`metadata`字段可以包含应用程序所需的任意元数据。例如，应用程序可能希望包含块校验和以丢弃损坏的`chunks`，或者使用[Merkle证明](https://ethereum.org/en/developers/tutorials/merkle-proofs-for-offline-data-integrity/)来针对链上应用哈希验证每个块的完整性。在[Protobuf](https://developers.google.com/protocol-buffers/docs/overview)编码形式中，快照`metadata`消息的大小不能超过4 MB。

### 生成和提供快照

为了启用状态同步，网络中的一些节点必须生成和提供快照。当对等方尝试进行状态同步时，现有的Tendermint节点将调用应用程序上的以下ABCI方法，向该对等方提供快照数据：

- [`ListSnapshots`](https://docs.tendermint.com/master/spec/abci/abci.html#listsnapshots)：返回可用快照的列表，包括元数据
- [`LoadSnapshotChunk`](https://docs.tendermint.com/master/spec/abci/abci.html#loadsnapshotchunk)：返回二进制块数据

通常应定期生成快照而不是按需生成：这样可以提高状态同步性能，因为快照生成可能很慢，并且避免了敌对方以此方式淹没节点的拒绝服务攻击。通常可以删除旧的快照，但保留至少最近的两个快照可能是有用的，以避免在节点恢复旧快照时删除先前的快照。

它完全取决于应用程序如何进行快照，但应该努力满足以下保证：

- **异步**：快照不应阻塞块处理，因此应异步进行，例如在单独的线程中进行
- **一致**：快照应在隔离的高度上进行，并且不应受到并发写入的影响，例如由于主线程中的块处理
- **确定性**：给定`高度`和`格式`，快照`chunks`和`metadata`应该是相同的（在字节级别），以确保`chunks`的良好可用性

例如，可以按照以下方式实现：

1. 使用支持事务和快照隔离的数据存储，例如RocksDB或BadgerDB。
2. 在提交块后，在主线程中启动一个只读数据库事务。
3. 将数据库事务句柄传递给新生成的线程。
4. 按确定性顺序（例如按键排序）迭代所有数据项。
5. 序列化数据项（例如使用[Protobuf](https://developers.google.com/protocol-buffers/docs/overview)），并将其写入字节流。
6. 对字节流进行哈希处理，并将其拆分为固定大小的`chunks`（例如10 MB）。
7. 将`chunks`作为单独的文件存储在文件系统中。
8. 将快照元数据（包括字节流哈希）写入数据库或文件。
9. 关闭数据库事务并退出线程。

应用程序还可以采取其他额外的步骤，例如压缩数据，对`chunks`进行校验和，为增量验证生成证明，以及删除旧的快照。

### 恢复快照

当Tendermint启动时，它将检查本地节点是否具有任何状态（即`LastBlockHeight == 0`），如果没有，则将通过P2P网络开始发现快照。
这些快照将通过以下ABCI调用提供给本地应用程序：

- [`OfferSnapshot(snapshot, apphash)`](https://docs.tendermint.com/master/spec/abci/abci.html#offersnapshot)：向应用程序提供发现的快照
- [`ApplySnapshotChunk(index, chunk, sender)`](https://docs.tendermint.com/master/spec/abci/abci.html#applysnapshotchunk)：应用快照`chunk`

发现的快照提供给应用程序，应用程序可以通过接受快照、拒绝快照、拒绝格式、拒绝发送者、中止状态同步等方式进行响应。

一旦快照被接受，Tendermint将从可用的节点中获取块，并按顺序将它们应用于应用程序，应用程序可以选择接受块、重新获取块、拒绝快照、拒绝发送者、中止状态同步等。

一旦所有块都被应用，Tendermint将调用应用程序的 [`Info` ABCI 方法](https://docs.tendermint.com/master/spec/abci/abci.html#info)，并检查应用哈希和高度是否与链上的可信值相对应。然后，它将切换到快速同步以获取任何剩余的块（如果启用），最后加入正常的共识操作。

快照的实际恢复方式完全取决于应用程序，但通常与生成方式相反。然而，请注意，Tendermint仅在所有块都恢复后验证快照，并不会自行拒绝任何P2P节点。只要可信哈希和应用程序代码正确，对手无法使状态同步的节点在加入共识时具有不正确的状态，但应用程序需要采取措施来对抗状态同步的拒绝服务（例如，通过实施增量验证、拒绝无效的节点）。

请注意，状态同步的节点将从恢复快照的高度开始具有截断的块历史记录，并且目前没有[填充所有块数据的功能](https://github.com/tendermint/tendermint/issues/4629)。网络应考虑到这一点的更广泛影响，并可能希望确保至少有几个存档节点保留完整的块历史记录，以便进行审计和备份。

## Cosmos SDK 状态同步

[Cosmos SDK](https://github.com/cosmos/cosmos-sdk) v0.40+ 包含对状态同步的自动支持，因此应用程序开发人员只需启用它即可享受其好处。他们无需自己实现在[Tendermint](#tendermint-core-state-sync)中描述的状态同步协议。

### 状态同步快照

Tendermint Core负责发现、交换和验证状态同步的状态数据的大部分繁重工作，但应用程序必须定期对其状态进行快照，并通过ABCI调用将其提供给Tendermint，并在同步新节点时能够恢复这些快照。

Cosmos SDK将应用程序状态存储在名为[IAVL](https://github.com/cosmos/iavl)的数据存储中，每个模块可以设置自己的IAVL存储。在规定的高度间隔（可配置）上，Cosmos SDK将导出每个存储在该高度上的内容，对其进行[Protobuf](https://developers.google.com/protocol-buffers/docs/overview)编码和压缩，并将其保存到本地文件系统中的快照存储中。由于IAVL保留了数据的历史版本，这些快照可以与正在执行的新块同时生成。然后，当新节点进行状态同步时，Tendermint将通过ABCI获取这些快照。

请注意，只有由Cosmos SDK管理的IAVL存储可以进行快照。如果应用程序将其他数据存储在外部数据存储中，目前没有机制将其包含在状态同步快照中，因此应用程序无法通过SDK自动进行状态同步。但是，它可以自由地根据[ABCI文档](https://docs.tendermint.com/master/spec/abci/apps.html#state-sync)中描述的方式实现状态同步协议。

当进行状态同步的新节点时，Tendermint将从网络中的对等节点获取快照，并将其提供给本地（空）应用程序，该应用程序将其导入其IAVL存储中。然后，Tendermint使用轻客户端验证将应用程序的应用哈希与主区块链进行验证，并继续执行块。请注意，状态同步的节点仅会恢复快照所拍摄的高度的应用程序状态，并不包含历史数据或历史块。

### 启用状态同步快照

要启用状态同步快照，使用CosmosSDK的`BaseApp`的应用程序需要设置一个快照存储（包括数据库和文件系统目录），并配置快照间隔和保留的历史快照数量。以下是一个最简示例：

```bash
snapshotDir := filepath.Join(
  cast.ToString(appOpts.Get(flags.FlagHome)), "data", "snapshots")
snapshotDB, err := sdk.NewLevelDB("metadata", snapshotDir)
if err != nil {
  panic(err)
}
snapshotStore, err := snapshots.NewStore(snapshotDB, snapshotDir)
if err != nil {
  panic(err)
}
app := baseapp.NewBaseApp(
  "app", logger, db, txDecoder,
  baseapp.SetSnapshotStore(snapshotStore),
  baseapp.SetSnapshotInterval(cast.ToUint64(appOpts.Get(
    server.FlagStateSyncSnapshotInterval))),
  baseapp.SetSnapshotKeepRecent(cast.ToUint32(appOpts.Get(
    server.FlagStateSyncSnapshotKeepRecent))),
)
```

在使用适当的标志启动应用程序时，
（例如 `--state-sync.snapshot-interval 1000 --state-sync.snapshot-keep-recent 2`）
它应该生成快照并输出日志消息：

```bash
Creating state snapshot    module=main height=3000
Completed state snapshot   module=main height=3000 format=1
```

请注意，快照间隔目前必须是 `pruning-keep-every` 的倍数（默认为100），
以防止在进行快照时删除高度。
通常还应保留至少2个最近的快照，
这样在节点尝试使用先前的快照进行状态同步时，不会删除它。

## 同步节点状态

:::tip
寻找快照或归档节点以与您的节点同步？请查看[此页面](./../../develop/api/snapshots-archives)。
:::

一旦网络中的几个节点已经进行了状态同步快照，新节点就可以使用状态同步加入网络。
要做到这一点，节点首先应按照通常的方式进行配置，
并且必须获取以下信息以进行轻客户端验证：

- 两个可用的RPC服务器（至少）
- 可信高度
- 可信高度的区块ID哈希

可信哈希必须从可信源（例如区块浏览器）获取，但是RPC服务器不需要受信任。
Tendermint将使用哈希从区块链中获取可信的应用程序哈希，
以验证恢复的应用程序快照。
应用程序哈希和相应的高度是在恢复快照时可以信任的唯一信息。
其他所有信息都可以被对手伪造。

在本指南中，我们使用Ubuntu 20.04

### 准备系统

更新系统

```bash
sudo apt update -y
```

升级系统

```bash
sudo apt upgrade -y
```

安装依赖项

```bash
sudo apt-get install ca-certificates curl gnupg lsb-release make gcc git jq wget -y
```

安装Go

```bash
wget -q -O - https://raw.githubusercontent.com/canha/golang-tools-install-script/master/goinstall.sh | bash
source ~/.bashrc
```

设置节点名称

```bash
moniker="节点名称"
```

## 使用以下命令设置测试网

```bash
SNAP_RPC1="http://bd-evmos-testnet-state-sync-node-01.bdnodes.net:26657"
SNAP_RPC="http://bd-evmos-testnet-state-sync-node-02.bdnodes.net:26657"
CHAIN_ID="evmos_9000-4"
PEER="3a6b22e1569d9f85e9e97d1d204a1c457d860926@bd-evmos-testnet-seed-node-01.bdnodes.net:26656"
wget -O $HOME/genesis.json https://archive.evmos.dev/evmos_9000-4/genesis.json 
```

## 使用以下命令设置主网

```bash
SNAP_RPC1="http://bd-evmos-mainnet-state-sync-us-01.bdnodes.net:26657"
SNAP_RPC="http://bd-evmos-mainnet-state-sync-eu-01.bdnodes.net:26657"
CHAIN_ID="evmos_9001-2"
PEER="96557e26aabf3b23e8ff5282d03196892a7776fc@bd-evmos-mainnet-state-sync-us-01.bdnodes.net:26656,dec587d55ff38827ebc6312cedda6085c59683b6@bd-evmos-mainnet-state-sync-eu-01.bdnodes.net:26656"
wget -O $HOME/genesis.json https://archive.evmos.org/mainnet/genesis.json 
```

### 安装 evmosd

```bash
git clone https://github.com/evmos/evmos.git && \ 
cd evmos && \ 
make install
```

### 配置

节点初始化

```bash
evmosd init $moniker --chain-id $CHAIN_ID
```

将创世文件移动到 .evmosd/config 文件夹

```bash
mv $HOME/genesis.json ~/.evmosd/config/
```

重置节点

```bash
evmosd tendermint unsafe-reset-all --home $HOME/.evmosd
```

更改配置文件（设置节点名称，添加持久节点，设置 indexer = "null"）

```bash
sed -i -e "s%^moniker *=.*%moniker = \"$moniker\"%; " $HOME/.evmosd/config/config.toml
sed -i -e "s%^indexer *=.*%indexer = \"null\"%; " $HOME/.evmosd/config/config.toml
sed -i -e "s%^persistent_peers *=.*%persistent_peers = \"$PEER\"%; " $HOME/.evmosd/config/config.toml
```

设置从快照启动的变量

:::tip
**注意**：通常，在其他 Cosmos 链上，用户被指示使用最新高度减去默认的 2000 作为可信高度。
然而，Evmos 链的快照生成时间较长，导致快照高度与当前最新高度之间的差距超过 2000。
这会导致上下文超时错误：

```
5:33PM ERR error on light block request from witness, removing... error="post failed: Post \"http://bd-evmos-mainnet-state-sync-us-01.bdnodes.net:26657\": context deadline exceeded" module=light primary={} server=node
```

为避免此问题，只需选择一个接近快照高度的可信高度。
例如，如果您知道在高度 `13286000` 存在一个快照，请考虑从块 `13284000` 开始选择一个可信哈希。

您可以从 [Polkachu](https://polkachu.com/tendermint_snapshots/evmos) 获取最新的快照高度。
否则，您的节点将找到可用的快照。
您将看到类似于以下日志：

```
5:48PM INF Discovered new snapshot format=2 hash="�z���Մ��^��Q\x1a\\I_�\x0f�OT!�(jM�$!��" height=13286000 module=statesync server=node
```

:::

```bash
LATEST_HEIGHT=$(curl -s $SNAP_RPC/block | jq -r .result.block.header.height); \
BLOCK_HEIGHT=$((LATEST_HEIGHT - 40000)); \
TRUST_HASH=$(curl -s "$SNAP_RPC/block?height=$BLOCK_HEIGHT" | jq -r .result.block_id.hash)
```

检查

```bash
echo $LATEST_HEIGHT $BLOCK_HEIGHT $TRUST_HASH
```

输出示例（数字将不同）：

```bash
376080 374080 F0C78FD4AE4DB5E76A298206AE3C602FF30668C521D753BB7C435771AEA47189
```

如果输出正常，请执行下一步

```bash
sed -i.bak -E "s|^(enable[[:space:]]+=[[:space:]]+).*$|\1true| ; \

s|^(rpc_servers[[:space:]]+=[[:space:]]+).*$|\1\"$SNAP_RPC,$SNAP_RPC1\"| ; \

s|^(trust_height[[:space:]]+=[[:space:]]+).*$|\1$BLOCK_HEIGHT| ; \

s|^(trust_hash[[:space:]]+=[[:space:]]+).*$|\1\"$TRUST_HASH\"| ; \

s|^(seeds[[:space:]]+=[[:space:]]+).*$|\1\"\"|" ~/.evmosd/config/config.toml
```

### 创建 evmosd 服务

```bash
echo "[Unit]
Description=Evmosd Node
After=network.target
#
[Service]
User=$USER
Type=simple
ExecStart=$(which evmosd) start
Restart=on-failure
LimitNOFILE=65535
#
[Install]
WantedBy=multi-user.target" > $HOME/evmosd.service; sudo mv $HOME/evmosd.service /etc/systemd/system/
```

```bash
sudo systemctl enable evmosd.service && sudo systemctl daemon-reload
```

### 运行 evmosd

```bash
systemctl start evmosd
```

### 检查日志

```bash
journalctl -u evmosd -f
```

当节点启动时，它将尝试在网络中找到一个状态同步快照，并恢复它：

```bash
Started node                   module=main nodeInfo="..."
Discovering snapshots for 20s
Discovered new snapshot        height=3000 format=1 hash=0F14A473
Discovered new snapshot        height=2000 format=1 hash=C6209AF7
Offering snapshot to ABCI app  height=3000 format=1 hash=0F14A473
Snapshot accepted, restoring   height=3000 format=1 hash=0F14A473
Fetching snapshot chunk        height=3000 format=1 chunk=0 total=3
Fetching snapshot chunk        height=3000 format=1 chunk=1 total=3
Fetching snapshot chunk        height=3000 format=1 chunk=2 total=3
Applied snapshot chunk         height=3000 format=1 chunk=0 total=3
Applied snapshot chunk         height=3000 format=1 chunk=1 total=3
Applied snapshot chunk         height=3000 format=1 chunk=2 total=3
Verified ABCI app              height=3000 appHash=F7D66BC9
Snapshot restored              height=3000 format=1 hash=0F14A473
Executed block                 height=3001 validTxs=16 invalidTxs=0
Committed state                height=3001 txs=16 appHash=0FDBB0D5F
Executed block                 height=3002 validTxs=25 invalidTxs=0
Committed state                height=3002 txs=25 appHash=40D12E4B3
```

节点现在已经完成状态同步，仅需几秒钟即可加入网络

### 使用此命令在节点完全同步后关闭状态同步模式，以避免未来节点重启时出现问题！

```bash
sed -i.bak -E "s|^(enable[[:space:]]+=[[:space:]]+).*$|\1false|" $HOME/.evmosd/config/config.toml
```

:::tip

**注意**：本文档中的信息来自[Erik Grinaker](https://medium.com/@erikgrinaker)，
具体来自他的[Tendermint Core](https://medium.com/tendermint/tendermint-core-state-sync-for-developers-70a96ba3ee35)和[Cosmos SDK](https://medium.com/cosmos-blockchain/cosmos-sdk-state-sync-guide-99e4cf43be2f)的状态同步指南。
:::


---
sidebar_position: 4
---
# State Sync

Learn about Tendermint Core state sync and support offered by the Cosmos SDK.

:::tip
**Note**: Only curious about how to sync a node with the network? Skip to [this section](#state-syncing-a-node).
:::

## Tendermint Core State Sync

State sync allows a new node to join a network by fetching a snapshot of the network state at a recent height,
instead of fetching and replaying all historical blocks.
Since application state is smaller than the combination of all blocks,
and restoring state is faster than replaying blocks, this reduces the time to sync with the network from days to minutes.

This section of the document provides a brief overview of the Tendermint state sync protocol, and how to sync a node.
For more details, refer to the
[ABCI Application Guide](https://docs.tendermint.com/master/spec/abci/apps.html#state-sync)
and the [ABCI Reference Documentation](https://docs.tendermint.com/master/spec/abci/abci.html).

### State Sync Snapshots

A guiding principle when designing Tendermint state sync was to give applications as much flexibility as possible.
Therefore, Tendermint does not care what snapshots contain, how they are taken, or how they are restored.
It is only concerned with discovering existing snapshots in the network, fetching them,
and passing them to applications via ABCI.
Tendermint uses light client verification to check the final app hash of a restored
application against the chain app hash, but any further verification must be done by the application itself during restoration.

Snapshots consist of binary chunks in an arbitrary format.
Chunks cannot be larger than 16 MB, but otherwise there are no restrictions.
[Snapshot metadata](https://docs.tendermint.com/master/spec/abci/abci.html#snapshot),
exchanged via ABCI and P2P, contains the following fields:

- `height` (`uint64`): height at which the snapshot was taken
- `format` (`uint32`): arbitrary application-specific format identifier (eg. version)
- `chunks` (`uint32`): number of binary chunks in the snapshot
- `hash` (`bytes`): arbitrary snapshot hash for comparing snapshots across nodes
- `metadata` (`bytes`): arbitrary binary snapshot metadata for use by applications

The `format` field allows applications to change their snapshot format in a backwards-compatible manner,
by providing snapshots in multiple formats, and choosing which formats to accept during restoration.
This is useful when, for example, changing serialization or compression formats: as nodes may be able to provide
snapshots to peers running older verions, or make use of old snapshots when starting up with a newer version.

The `hash` field contains an arbitrary snapshot hash. Snapshots that have identical `metadata` fields (including `hash`)
across nodes are considered identical, and `chunks` will be fetched from any of these nodes.
The `hash` cannot be trusted, and is not verified by Tendermint itself,
which guards against inadvertent nondeterminism in snapshot generation.
The `hash` may be verified by the application instead.

The `metadata` field can contain any arbitrary metadata needed by the application.
For example, the application may want to include chunk checksums to discard damaged `chunks`,
or [Merkle proofs](https://ethereum.org/en/developers/tutorials/merkle-proofs-for-offline-data-integrity/)
to verify each chunk individually against the chain app hash.
In [Protobuf](https://developers.google.com/protocol-buffers/docs/overview)-encoded form,
snapshot `metadata` messages cannot exceed 4 MB.

### Taking, Serving Snapshots

To enable state sync, some nodes in the network must take and serve snapshots. When a peer is attempting to state sync,
an existing Tendermint node will call the following ABCI methods on the application to provide snapshot data to this peer:

- [`ListSnapshots`](https://docs.tendermint.com/master/spec/abci/abci.html#listsnapshots):
  returns a list of available snapshots, with metadata
- [`LoadSnapshotChunk`](https://docs.tendermint.com/master/spec/abci/abci.html#loadsnapshotchunk): returns binary chunk data

Snapshots should typically be generated at regular intervals rather than on-demand: this improves state sync performance,
since snapshot generation can be slow, and avoids a denial-of-service vector where an adversary
floods a node with such requests. Older snapshots can usually be removed, but it may be useful to keep at least
the two most recent to avoid deleting the previous snapshot while a node is restoring it.

It is entirely up to the application to decide how to take snapshots, but it should strive to satisfy the following guarantees:

- **Asynchronous**: snapshotting should not halt block processing, and it should therefore happen asynchronously,
  eg. in a separate thread
- **Consistent**: snapshots should be taken at isolated heights, and should not be affected by concurrent writes,
  eg. due to block processing in the main thread
- **Deterministic**: snapshot `chunks` and `metadata` should be identical (at the byte level)
  across all nodes for a given `height` and `format`, to ensure good availability of `chunks`

As an example, this can be implemented as follows:

1. Use a data store that supports transactions with snapshot isolation, such as RocksDB or BadgerDB.
2. Start a read-only database transaction in the main thread after committing a block.
3. Pass the database transaction handle into a newly spawned thread.
4. Iterate over all data items in a deterministic order (eg. sorted by key)
5. Serialize data items (eg. using [Protobuf](https://developers.google.com/protocol-buffers/docs/overview)),
   and write them to a byte stream.
6. Hash the byte stream, and split it into fixed-size chunks (eg. of 10 MB)
7. Store the chunks in the file system as separate files.
8. Write the snapshot metadata to a database or file, including the byte stream hash.
9. Close the database transaction and exit the thread.

Applications may want to take additional steps as well, such as compressing the data, checksumming chunks,
generating proofs for incremental verification, and removing old snapshots.

### Restoring Snapshots

When Tendermint starts, it will check whether the local node has any state (ie. whether `LastBlockHeight == 0`),
and if it doesn't, it will begin discovering snapshots via the P2P network.
These snapshots will be provided to the local application via the following ABCI calls:

- [`OfferSnapshot(snapshot, apphash)`](https://docs.tendermint.com/master/spec/abci/abci.html#offersnapshot):
  offers a discovered snapshot to the application
- [`ApplySnapshotChunk(index, chunk, sender)`](https://docs.tendermint.com/master/spec/abci/abci.html#applysnapshotchunk):
  applies a snapshot chunk

Discovered snapshots are offered to the application and it can respond by accepting the snapshot,
rejecting it, rejecting the format, rejecting the senders, aborting state sync, and so on.

Once a snapshot is accepted, Tendermint will fetch chunks from across available peers,
and apply them sequentially to the application, which can choose to accept the chunk,
refetch it, reject the snapshot, reject the sender, abort state sync, and so on.

Once all chunks have been applied,
Tendermint will call the [`Info` ABCI method](https://docs.tendermint.com/master/spec/abci/abci.html#info)
on the application, and check that the app hash and height correspond to the trusted values from the chain.
It will then switch to fast sync to fetch any remaining blocks (if enabled), before finally joining normal consensus operation.

How snapshots are actually restored is entirely up to the application,
but will generally be the inverse of how they are generated.
Note, however, that Tendermint only verifies snapshots after all chunks have been restored,
and does not reject any P2P peers on its own. As long as the trusted hash and application code are correct,
it is not possible for an adversary to cause a state synced node to have incorrect state when joining consensus,
but it is up to the application to counteract state sync denial-of-service
(eg. by implementing incremental verification, rejecting invalid peers).

Note that state synced nodes will have a truncated block history starting at the height of the restored snapshot,
and there is currently no [backfill of all block data](https://github.com/tendermint/tendermint/issues/4629).
Networks should consider broader implications of this, and may want to ensure at least a few archive nodes
retain a complete block history, for both auditability and backup.

## Cosmos SDK State Sync

[Cosmos SDK](https://github.com/cosmos/cosmos-sdk) v0.40+ includes automatic support for state sync,
so application developers only need to enable it to take advantage.
They will not need to implement the state sync protocol described in the
[above section on Tendermint](#tendermint-core-state-sync) themselves.

### State Sync Snapshots

Tendermint Core handles most of the grunt work of discovering, exchanging,
and verifying state data for state sync, but the application must take snapshots of its state at regular intervals,
and make these available to Tendermint via ABCI calls, and be able to restore these when syncing a new node.

The Cosmos SDK stores application state in a data store called [IAVL](https://github.com/cosmos/iavl),
and each module can set up its own IAVL stores. At regular height intervals (which are configurable),
the Cosmos SDK will export the contents of each store at that height,
[Protobuf](https://developers.google.com/protocol-buffers/docs/overview)-encode and compress it,
and save it to a snapshot store in the local filesystem. Since IAVL keeps historical versions of data,
these snapshots can be generated simultaneously with new blocks being executed.
These snapshots will then be fetched by Tendermint via ABCI when a new node is state syncing.

Note that only IAVL stores that are managed by the Cosmos SDK can be snapshotted.
If the application stores additional data in external data stores,
there is currently no mechanism to include these in state sync snapshots,
so the application therefore cannot make use of automatic state sync via the SDK.
However, it is free to implement the state sync protocol itself as described in the
[ABCI Documentation](https://docs.tendermint.com/master/spec/abci/apps.html#state-sync).

When a new node is state synced, Tendermint will fetch a snapshot from peers in the network
and provide it to the local (empty) application, which will import it into its IAVL stores.
Tendermint then verifies the application's app hash against the main blockchain using light client verification,
and proceeds to execute blocks as usual.
Note that a state synced node will only restore the application state for the height the snapshot was taken at,
and will not contain historical data nor historical blocks.

### Enabling State Sync Snapshots

To enable state sync snapshots, an application using the CosmosSDK `BaseApp` needs to set up a snapshot store
(with a database and filesystem directory) and configure the snapshotting interval
and the number of historical snapshots to keep. A minimal exmaple of this follows:

```bash
snapshotDir := filepath.Join(
  cast.ToString(appOpts.Get(flags.FlagHome)), "data", "snapshots")
snapshotDB, err := sdk.NewLevelDB("metadata", snapshotDir)
if err != nil {
  panic(err)
}
snapshotStore, err := snapshots.NewStore(snapshotDB, snapshotDir)
if err != nil {
  panic(err)
}
app := baseapp.NewBaseApp(
  "app", logger, db, txDecoder,
  baseapp.SetSnapshotStore(snapshotStore),
  baseapp.SetSnapshotInterval(cast.ToUint64(appOpts.Get(
    server.FlagStateSyncSnapshotInterval))),
  baseapp.SetSnapshotKeepRecent(cast.ToUint32(appOpts.Get(
    server.FlagStateSyncSnapshotKeepRecent))),
)
```

When starting the application with the appropriate flags,
(eg. `--state-sync.snapshot-interval 1000 --state-sync.snapshot-keep-recent 2`)
it should generate snapshots and output log messages:

```bash
Creating state snapshot    module=main height=3000
Completed state snapshot   module=main height=3000 format=1
```

Note that the snapshot interval must currently be a multiple of the `pruning-keep-every` (defaults to 100),
to prevent heights from being pruned while taking snapshots.
It's also usually a good idea to keep at least 2 recent snapshots,
such that the previous snapshot isn't removed while a node is attempting to state sync using it.

## State Syncing a Node

:::tip
Looking for snapshots or archive nodes to sync your node with? Check out [this page](./../../develop/api/snapshots-archives).
:::

Once a few nodes in a network have taken state sync snapshots, new nodes can join the network using state sync.
To do this, the node should first be configured as usual,
and the following pieces of information must be obtained for light client verification:

- Two available RPC servers (at least)
- Trusted height
- Block ID hash of trusted height

The trusted hash must be obtained from a trusted source (eg. a block explorer),
but the RPC servers do not need to be trusted.
Tendermint will use the hash to obtain trusted app hashes from the blockchain
in order to verify restored application snapshots.
The app hash and corresponding height are the only pieces of information that
can be trusted when restoring snapshots.
Everything else can be forged by adversaries.

In this guide we use Ubuntu 20.04

### Prepare system

Update system

```bash
sudo apt update -y
```

Upgrade system

```bash
sudo apt upgrade -y
```

Install dependencies

```bash
sudo apt-get install ca-certificates curl gnupg lsb-release make gcc git jq wget -y
```

Install Go

```bash
wget -q -O - https://raw.githubusercontent.com/canha/golang-tools-install-script/master/goinstall.sh | bash
source ~/.bashrc
```

Set the node name

```bash
moniker="NODE_NAME"
```

## Use commands below for Testnet setup

```bash
SNAP_RPC1="http://bd-evmos-testnet-state-sync-node-01.bdnodes.net:26657"
SNAP_RPC="http://bd-evmos-testnet-state-sync-node-02.bdnodes.net:26657"
CHAIN_ID="evmos_9000-4"
PEER="3a6b22e1569d9f85e9e97d1d204a1c457d860926@bd-evmos-testnet-seed-node-01.bdnodes.net:26656"
wget -O $HOME/genesis.json https://archive.evmos.dev/evmos_9000-4/genesis.json 
```

## Use commands below for Mainnet setup

```bash
SNAP_RPC1="http://bd-evmos-mainnet-state-sync-us-01.bdnodes.net:26657"
SNAP_RPC="http://bd-evmos-mainnet-state-sync-eu-01.bdnodes.net:26657"
CHAIN_ID="evmos_9001-2"
PEER="96557e26aabf3b23e8ff5282d03196892a7776fc@bd-evmos-mainnet-state-sync-us-01.bdnodes.net:26656,dec587d55ff38827ebc6312cedda6085c59683b6@bd-evmos-mainnet-state-sync-eu-01.bdnodes.net:26656"
wget -O $HOME/genesis.json https://archive.evmos.org/mainnet/genesis.json 
```

### Install evmosd

```bash
git clone https://github.com/evmos/evmos.git && \ 
cd evmos && \ 
make install
```

### Configuration

Node init

```bash
evmosd init $moniker --chain-id $CHAIN_ID
```

Move genesis file to .evmosd/config folder

```bash
mv $HOME/genesis.json ~/.evmosd/config/
```

Reset the node

```bash
evmosd tendermint unsafe-reset-all --home $HOME/.evmosd
```

Change config files (set the node name, add persistent peers, set indexer = "null")

```bash
sed -i -e "s%^moniker *=.*%moniker = \"$moniker\"%; " $HOME/.evmosd/config/config.toml
sed -i -e "s%^indexer *=.*%indexer = \"null\"%; " $HOME/.evmosd/config/config.toml
sed -i -e "s%^persistent_peers *=.*%persistent_peers = \"$PEER\"%; " $HOME/.evmosd/config/config.toml
```

Set the variables for start from snapshot

:::tip
**Note**: Usually, on other cosmos chains, the user is instructed to use as trusted height
the latest height minus 2000 by default.
However, snapshots of the Evmos chain take a long time to generate, which makes the gap between the snapshot height and
the current latest height be more than 2000.
This results in a context timeout error:

```
5:33PM ERR error on light block request from witness, removing... error="post failed: Post \"http://bd-evmos-mainnet-state-sync-us-01.bdnodes.net:26657\": context deadline exceeded" module=light primary={} server=node
```

To avoid this issue, simply pick a trusted height close to snapshot height.
For example, if you know there's a snapshot at height `13286000`, consider a trust hash from block `13284000`.

You can get the latest snapshot height from [Polkachu here.](https://polkachu.com/tendermint_snapshots/evmos).
Otherwise, your node will find the available snapshots.
You will see logs similar to this:

```
5:48PM INF Discovered new snapshot format=2 hash="�z���Մ��^��Q\x1a\\I_�\x0f�OT!�(jM�$!��" height=13286000 module=statesync server=node
```

:::

```bash
LATEST_HEIGHT=$(curl -s $SNAP_RPC/block | jq -r .result.block.header.height); \
BLOCK_HEIGHT=$((LATEST_HEIGHT - 40000)); \
TRUST_HASH=$(curl -s "$SNAP_RPC/block?height=$BLOCK_HEIGHT" | jq -r .result.block_id.hash)
```

Check

```bash
echo $LATEST_HEIGHT $BLOCK_HEIGHT $TRUST_HASH
```

Output example (numbers will be different):

```bash
376080 374080 F0C78FD4AE4DB5E76A298206AE3C602FF30668C521D753BB7C435771AEA47189
```

If output is OK do next

```bash
sed -i.bak -E "s|^(enable[[:space:]]+=[[:space:]]+).*$|\1true| ; \

s|^(rpc_servers[[:space:]]+=[[:space:]]+).*$|\1\"$SNAP_RPC,$SNAP_RPC1\"| ; \

s|^(trust_height[[:space:]]+=[[:space:]]+).*$|\1$BLOCK_HEIGHT| ; \

s|^(trust_hash[[:space:]]+=[[:space:]]+).*$|\1\"$TRUST_HASH\"| ; \

s|^(seeds[[:space:]]+=[[:space:]]+).*$|\1\"\"|" ~/.evmosd/config/config.toml
```

### Create evmosd service

```bash
echo "[Unit]
Description=Evmosd Node
After=network.target
#
[Service]
User=$USER
Type=simple
ExecStart=$(which evmosd) start
Restart=on-failure
LimitNOFILE=65535
#
[Install]
WantedBy=multi-user.target" > $HOME/evmosd.service; sudo mv $HOME/evmosd.service /etc/systemd/system/
```

```bash
sudo systemctl enable evmosd.service && sudo systemctl daemon-reload
```

### Run evmosd

```bash
systemctl start evmosd
```

### Check logs

```bash
journalctl -u evmosd -f
```

When the node is started it will then attempt to find a state sync snapshot in the network, and restore it:

```bash
Started node                   module=main nodeInfo="..."
Discovering snapshots for 20s
Discovered new snapshot        height=3000 format=1 hash=0F14A473
Discovered new snapshot        height=2000 format=1 hash=C6209AF7
Offering snapshot to ABCI app  height=3000 format=1 hash=0F14A473
Snapshot accepted, restoring   height=3000 format=1 hash=0F14A473
Fetching snapshot chunk        height=3000 format=1 chunk=0 total=3
Fetching snapshot chunk        height=3000 format=1 chunk=1 total=3
Fetching snapshot chunk        height=3000 format=1 chunk=2 total=3
Applied snapshot chunk         height=3000 format=1 chunk=0 total=3
Applied snapshot chunk         height=3000 format=1 chunk=1 total=3
Applied snapshot chunk         height=3000 format=1 chunk=2 total=3
Verified ABCI app              height=3000 appHash=F7D66BC9
Snapshot restored              height=3000 format=1 hash=0F14A473
Executed block                 height=3001 validTxs=16 invalidTxs=0
Committed state                height=3001 txs=16 appHash=0FDBB0D5F
Executed block                 height=3002 validTxs=25 invalidTxs=0
Committed state                height=3002 txs=25 appHash=40D12E4B3
```

The node is now state synced, having joined the network in seconds

### Use this command to switch off your State Sync mode, after node fully synced to avoid problems in future node restarts!

```bash
sed -i.bak -E "s|^(enable[[:space:]]+=[[:space:]]+).*$|\1false|" $HOME/.evmosd/config/config.toml
```

:::tip

**Note**: Information included in this document is sourced from [Erik Grinaker](https://medium.com/@erikgrinaker),
specifically his state sync guides for
[Tendermint Core](https://medium.com/tendermint/tendermint-core-state-sync-for-developers-70a96ba3ee35) and the [Cosmos SDK](https://medium.com/cosmos-blockchain/cosmos-sdk-state-sync-guide-99e4cf43be2f).
:::
