---
sidebar_position: 8
---

# 状态导出/导入

Evmos可以将整个应用程序状态导出为JSON文件。
除了[升级](../../validate/upgrades)之外，
这对于在给定高度对状态进行手动分析非常有用。

## 导出状态

使用以下命令导出状态：

```bash
evmosd export > new_genesis.json
```

您还可以从特定高度导出状态
（在处理该高度的块结束时）：

```bash
evmosd export --height [height] > new_genesis.json
```

如果您计划从导出的状态开始一个新的网络，高度为0（即创世块），
请使用`--for-zero-height`标志导出：

```bash
evmosd export --height [height] --for-zero-height > new_genesis.json
```

## 手动迁移状态

如果您想手动迁移状态，例如进行本地测试。
请注意，对于常规链升级，不需要手动迁移状态。

在将状态导出到json文件后，
您可以用`new_genesis.json`替换旧的`genesis.json`。

```bash
cp -f genesis.json new_genesis.json
mv new_genesis.json genesis.json
```

此时，您可能希望运行一个脚本
将导出的创世状态更新为与新版本兼容的创世状态。

您可以使用`migrate`命令
从给定版本迁移到下一个版本（例如：`v0.X.X`到`v1.X.X`）：

```bash
evmosd migrate TARGET_VERSION GENESIS_FILE --chain-id=<new_chain_id> --genesis-time=<yyyy-mm-ddThh:mm:ssZ>
```


---
sidebar_position: 8
---

# State Export/Import

Evmos can dump the entire application state to a JSON file.
This, besides [upgrades](../../validate/upgrades),
can be useful for manual analysis of the state at a given height.

## Export State

Export state with:

```bash
evmosd export > new_genesis.json
```

You can also export state from a particular height
(at the end of processing the block of that height):

```bash
evmosd export --height [height] > new_genesis.json
```

If you plan to start a new network for 0 height (i.e genesis) from the exported state,
export with the `--for-zero-height` flag:

```bash
evmosd export --height [height] --for-zero-height > new_genesis.json
```

## Manually Migrate State

If you want to migrate state manually, e.g. for local testing purpose.
Note that for regular chain upgrades, a manual state migration is not required.

After exporting your state into a json file,
you can replace the old `genesis.json` with `new_genesis.json`.

```bash
cp -f genesis.json new_genesis.json
mv new_genesis.json genesis.json
```

At this point, you might want to run a script
to update the exported genesis into a genesis state
that is compatible with your new version.

You can use the `migrate` command to
migrate from a given version to the next one (eg: `v0.X.X` to `v1.X.X`):

```bash
evmosd migrate TARGET_VERSION GENESIS_FILE --chain-id=<new_chain_id> --genesis-time=<yyyy-mm-ddThh:mm:ssZ>
```
