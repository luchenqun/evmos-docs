---
sidebar_position: 5
---

# 回滚

了解在链升级失败的情况下如何回滚链版本。

为了恢复先前的链版本，验证人必须恢复以下数据：

- 包含先前链状态的数据库（默认位于 `~/.evmosd/data`）
- 验证人的 `priv_validator_state.json` 文件（也位于 `~/.evmosd/data`）

如果验证人没有自己的数据库数据，其他验证人应该共享数据库的副本。验证人将能够下载数据的副本并在启动节点之前进行验证。如果验证人没有备份的 `priv_validator_state.json` 文件，那么这些验证人在第一个区块上将没有双签保护。

## 恢复状态的步骤

1. 首先，停止你的节点。

2. 然后，将备份数据目录的内容复制回 `EVMOS_HOME/data` 目录（默认情况下应该是 `~/.evmosd/data`）。

```bash
# Assumes backup is stored in "backup" directory
rm -rf ~/.evmosd/data
mv backup/.evmosd/data ~/.evmosd/data
```

3. 接下来，安装先前版本的 Evmos。

```bash
# from evmos directory
git checkout <prev_version>
make install
## verify version
evmosd version --long
```

4. 最后，启动节点。

```bash
evmosd start
```


---
sidebar_position: 5
---

# Rollback

Learn how to rollback the chain version in the case of an unsuccessful chain upgrade.

In order to restore a previous chain version, the following data must be recovered by validators:

- the database that contains the state of the previous chain (in `~/.evmosd/data` by default)
- the `priv_validator_state.json` file of the validator (also in `~/.evmosd/data` by default)

If validators don't possess their database data, another validator should share a copy of the database. Validators will
be able to download a copy of the data and verify it before starting their node. If validators don't have the backup
`priv_validator_state.json` file, then those validators will not have double-sign protection on their first block.

## Restoring State Procedure

1. First, stop your node.

2. Then, copy the contents of your backup data directory back to the `EVMOS_HOME/data` directory (which, by default,
should be `~/.evmosd/data`).

```bash
# Assumes backup is stored in "backup" directory
rm -rf ~/.evmosd/data
mv backup/.evmosd/data ~/.evmosd/data
```

3. Next, install the previous version of Evmos.

```bash
# from evmos directory
git checkout <prev_version>
make install
## verify version
evmosd version --long
```

4. Finally, start the node.

```bash
evmosd start
```
