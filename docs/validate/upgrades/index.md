---
sidebar_position: 1
---

# 概述

了解如何管理完整节点和验证节点的链升级。升级可以分为以下三个不同的类别：

- **计划或非计划升级**：可以通过升级提案计划在给定的高度进行计划升级。
- **破坏性或非破坏性升级**：升级可以是API或状态机破坏性的，这会影响向后兼容性。为了解决这个问题，应该在升级前迁移应用程序状态或创世文件。
- **数据重置升级**：某些升级需要完全重置数据以清理状态。这有时会在回滚或硬分叉的情况下发生。

此外，验证节点可以根据其首选选项选择如何管理升级：

- **自动或手动升级**：验证节点可以运行`cosmovisor`进程自动执行升级，也可以手动执行升级。

## 计划升级

计划升级是协调的预定升级，使用[升级模块](https://docs.cosmos.network/main/modules/upgrade)逻辑。这有助于平滑升级Evmos到一个新的（破坏性的）软件版本，因为它会自动处理新版本的状态迁移。

### 治理提案

治理提案是一种机制，用于协调在给定高度或时间上进行升级，使用[`SoftwareProposal`](https://docs.cosmos.network/main/modules/upgrade)。

:::tip
所有治理提案，包括软件升级，在升级执行之前都需要等待投票期结束。在提交软件升级提案时，请考虑此持续时间。
:::

如果提案通过，升级`Plan`将被持久化到区块链状态，并在给定的升级高度上进行计划。可以通过在新提案中更新`Plan.Height`来延迟或加速升级。

### 硬分叉

计划升级的一种特殊类型是[硬分叉](./upgrades/hard-fork-upgrades)。与治理提案不同，硬分叉不需要等待完整的投票期。这使得它们非常适合协调安全漏洞和补丁。

升级（分叉）块高度是在应用程序的 `BeginBlock` 中设置的（即在处理块的事务之前）。一旦区块链达到该高度，它会自动为相同的高度安排一个升级计划 `Plan`，然后触发升级过程。升级完成后，块操作（`BeginBlock`、事务处理和状态 `Commit`）将继续正常进行。

:::tip
为了执行升级硬分叉，首先需要发布一个带有 `BeginBlock` 升级调度逻辑的[补丁版本](#patch-versions)。在超过2/3的验证者升级到新的补丁版本后，它们的节点将自动停止并升级二进制文件。
:::

## 非计划升级

非计划升级是指所有验证者需要在过程的完全相同点上优雅地停止并关闭其节点的升级。这可以通过在运行 `evmosd start` 命令时设置 `--halt-height` 标志来完成。

如果在非计划升级期间存在破坏性更改（见下文），验证者需要在重新启动节点之前迁移状态和创世块。

:::tip
非计划升级的主要考虑因素是需要导出创世状态并重置区块链数据。这主要影响基础设施提供商、工具和客户端（如区块浏览器和客户端），它们必须使用存档节点为升级前的高度提供查询服务。
:::

## 破坏性和非破坏性升级

根据相应软件[发布版本](https://github.com/evmos/evmos/releases)（即 `vX.Y.Z`）的语义化版本（[Semver](https://semver.org/)），升级可以分为破坏性和非破坏性：

- **主版本（`X`）**：不向后兼容的 API 和状态机破坏性更改。
- **次版本（`Y`）**：新的向后兼容功能。这也可能是状态机破坏性的。
- **修订版本（`Z`）**：向后兼容的错误修复、小的重构和改进。

### 主版本

如果您要升级到的新版本有破坏性更改，您需要：

1. 迁移创世 JSON
2. 迁移应用状态
3. 重启节点

这样做是为了防止[双签名或在共识期间停止链](https://docs.tendermint.com/master/spec/consensus/signing.html#double-signing)。

要升级创世文件，您可以从可信源获取，或使用`evmosd export`命令在本地导出。

### 小版本

如果您要升级到的新版本有破坏性更改，您需要：

1. 迁移状态（如果适用）
2. 重启节点

### 补丁版本

为了更新补丁：

1. 停止节点
2. 手动下载新的发布二进制文件
3. 重启节点

## 数据重置升级

数据重置升级要求节点运营者完全重置区块链状态，并从干净状态重新启动节点，但使用相同的验证者密钥。

## 自动或手动升级

随着每个新的软件发布，我们强烈建议全节点和验证者运营者进行软件升级。

您可以通过以下方式升级节点：

- [自动](./upgrades/automated-upgrades)增加软件版本，并在升级完成后重新启动节点，或者
- 下载新的二进制文件并进行[手动升级](./upgrades/manual-upgrades)

按照上述选项中的链接了解如何根据您的首选选项升级节点。


---
sidebar_position: 1
---

# Overview

Learn how to manage chain upgrades for full and validator nodes. There are 3 different categories for upgrades:

- **Planned or Unplanned**: Chain upgrades can be scheduled at a given height through an upgrade proposal plan.
- **Breaking or Non-breaking**: Upgrades can be API or State Machine breaking, which affects backwards compatibility.
To address this, the application state or genesis file would need to be migrated in preparation for the upgrade.
- **Data Reset Upgrades**: Some upgrades will need a full data reset in order to clean the state. This can sometimes
occur in the case of a rollback or hard fork.

Additionally, validators can choose how to manage the upgrade according to their preferred option:

- **Automatic or Manual Upgrades**: Validator can run the `cosmovisor` process to automatically perform the upgrade
or do it manually.

## Planned Upgrades

Planned upgrades are coordinated scheduled upgrades that use the [upgrade module](https://docs.cosmos.network/main/modules/upgrade)
logic. This facilitates smoothly upgrading Evmos to a new (breaking) software version as it automatically handles
the state migration for the new release.

### Governance Proposal

Governance Proposals are a mechanism for coordinating an upgrade at a given height or time using an
[`SoftwareProposal`](https://docs.cosmos.network/main/modules/upgrade).

:::tip
All governance proposals, including software upgrades, need to wait for the voting period to conclude before the
upgrade can be executed. Consider this duration when submitting a software upgrade proposal.
:::

If the proposal passes, the upgrade `Plan`, which targets a specific upgrade logic to migrate the state, is
persisted to the blockchain state and scheduled at the given upgrade height. The upgrade can be delayed or
expedited by updating the `Plan.Height` in a new proposal.

### Hard Forks

A special type of planned upgrades are [hard forks](./upgrades/hard-fork-upgrades). Hard Forks, as opposed to Governance
Proposal, don't require waiting for the full voting
period. This makes them ideal for coordinating security vulnerabilities and patches.

The upgrade (fork) block height is set in the `BeginBlock` of the application (i.e before the transactions
are processed for the block). Once the blockchain reaches that height, it automatically schedules an upgrade
`Plan` for the same height and then triggers the upgrade process. After upgrading, the block operations
(`BeginBlock`, transaction processing and state `Commit`) continue normally.

:::tip
In order to execute an upgrade hard fork, a [patch version](#patch-versions) needs to first be released with
the `BeginBlock` upgrade scheduling logic. After a +2/3 of the validators upgrade to the new patch version,
their nodes will automatically halt and upgrade the binary.
:::

## Unplanned Upgrades

Unplanned upgrades are upgrades where all the validators need to gracefully halt and shut down their nodes at
exactly the same point in the process. This can be done by setting the `--halt-height` flag when running the
`evmosd start` command.

If there are breaking changes during an unplanned upgrade (see below), validators will need to migrate the state
and genesis before restarting their nodes.

:::tip
The main consideration with unplanned upgrades is that the genesis state needs to be exported and the blockchain
data needs to be [reset](#data-reset-upgrades). This mainly affects infrastructure providers, tools and clients
like block explorers and clients, which have to use archival nodes to serve queries for the pre-upgrade heights.
:::

## Breaking and Non-Breaking Upgrades

Upgrades can be categorized as breaking or non-breaking according to the Semantic versioning
([Semver](https://semver.org/)) of the corresponding software [release version](https://github.com/evmos/evmos/releases)
(*i.e* `vX.Y.Z`):

- **Major version (`X`)**: backward incompatible API and state machine breaking changes.
- **Minor version (`Y`)**: new backward compatible features. These can be also be state machine breaking.
- **Patch version (`Z`)**: backwards compatible bug fixes, small refactors and improvements.

### Major Versions

If the new version you are upgrading to has breaking changes, you will have to:

1. Migrate genesis JSON
2. Migrate application state
3. Restart node

This needs to be done to prevent [double signing or halting the chain during consensus](https://docs.tendermint.com/master/spec/consensus/signing.html#double-signing).

To upgrade the genesis file, you can either fetch it from a trusted source or export it locally using the
`evmosd export` command.

### Minor Versions

If the new version you are upgrading to has breaking changes, you will have to:

1. Migrate the state (if applicable)
2. Restart node

### Patch Versions

In order to update a patch:

1. Stop Node
2. Download new release binary manually
3. Restart node

## Data Reset Upgrades

Data Reset upgrades require node operators to fully reset the blockchain state and restart their nodes from a clean
state, but using the same validator keys.

## Automatic or Manual Upgrades

With every new software release, we strongly recommend full nodes and validator operators to perform a software upgrade.

You can upgrade your node by either:

- [automatically](./upgrades/automated-upgrades) bumping the software version and restart the node once the upgrade occurs, or
- download the new binary and perform a [manual upgrade](./upgrades/manual-upgrades)

Follow the links in the options above to learn how to upgrade your node according to your preferred option.
