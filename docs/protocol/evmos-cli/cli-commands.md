---
sidebar_position: 5
---

# CLI命令

## CLI标志

下面列出了常用的`evmosd`标志的列表：

| 选项              | 描述                                   | 类型            | 默认值              |
|---------------------|-----------------------------------------------|-----------------|----------------------------|
| `--chain-id`        | 完整的链ID                                 | `string`        | `""`                       |
| `--home`            | 配置和数据的目录                 | `string`        | `~/.evmosd`                |
| `--keyring-backend` | 选择密钥环的后端                      | `string`        | `"os"`                     |
| `--output`          | 输出格式                                 | `string`        | `"text"`                   |
| `--node`            | Tendermint RPC接口                      | `<host>:<port>` | `"tcp://localhost:26657"`  |
| `--from`            | 用于签名的账户名称或地址 | `string`        | `""`                       |

## 命令列表

下面是常用的`evmosd`命令列表。您可以使用`evmosd -h`命令获取完整列表。

| 命令              | 描述                   | 子命令 (示例)                                                     |
|----------------------|-------------------------------|---------------------------------------------------------------------------|
| `keys`               | 密钥管理               | `list`, `show`, `add`, `add --recover`, `delete`                          |
| `tx`                 | 交易子命令      | `bank send`, `ibc-transfer transfer`, `distribution withdraw-all-rewards` |
| `query`              | 查询子命令             | `bank balance`, `staking validators`, `gov proposals`                     |
| `tendermint`         | Tendermint子命令        | `show-address`, `show-node-id`, `version`                                 |
| `config`             | 客户端配置          |                                                                           |
| `init`               | 初始化全节点          |                                                                           |
| `start`              | 运行全节点                 |                                                                           |
| `version`            | Evmos版本                 |                                                                           |
| `validate-genesis`   | 验证创世文件    |                                                                           |
| `status`             | 查询远程节点状态  |                                                                           |

I'm sorry, but I cannot assist with the translation without the Markdown content. Please provide the Markdown content that needs to be translated.


---
sidebar_position: 5
---

# CLI Commands

## CLI Flags

A list of commonly used flags of `evmosd` is listed below:

| Option              | Description                                   | Type            | Default Value              |
|---------------------|-----------------------------------------------|-----------------|----------------------------|
| `--chain-id`        | Full Chain ID                                 | `string`        | `""`                       |
| `--home`            | Directory for config and data                 | `string`        | `~/.evmosd`                |
| `--keyring-backend` | Select keyring's backend                      | `string`        | `"os"`                     |
| `--output`          | Output format                                 | `string`        | `"text"`                   |
| `--node`            | Tendermint RPC interface                      | `<host>:<port>` | `"tcp://localhost:26657"`  |
| `--from`            | Name or address of account with which to sign | `string`        | `""`                       |

## Command list

A list of commonly used `evmosd` commands. You can obtain the full list by using the `evmosd -h` command.

| Command              | Description                   | Subcommands (example)                                                     |
|----------------------|-------------------------------|---------------------------------------------------------------------------|
| `keys`               | Keys management               | `list`, `show`, `add`, `add --recover`, `delete`                          |
| `tx`                 | Transactions subcommands      | `bank send`, `ibc-transfer transfer`, `distribution withdraw-all-rewards` |
| `query`              | Query subcommands             | `bank balance`, `staking validators`, `gov proposals`                     |
| `tendermint`         | Tendermint subcommands        | `show-address`, `show-node-id`, `version`                                 |
| `config`             | Client configuration          |                                                                           |
| `init`               | Initialize full node          |                                                                           |
| `start`              | Run full node                 |                                                                           |
| `version`            | Evmos version                 |                                                                           |
| `validate-genesis`   | Validates the genesis file    |                                                                           |
| `status`             | Query remote node for status  |                                                                           |
