---
sidebar_position: 2
---

# 硬分叉升级

正常的升级过程[via governance](./../../validate/upgrades#governance-proposal)的一个重要限制是需要等待整个投票期的时间。这个时间长度使得它不适用于涉及安全漏洞修补程序或其他关键组件的自动化升级。

使用治理的更快替代方法是创建一个硬分叉程序。这个程序[自动](./automated-upgrades)应用升级计划中的更改，允许在给定的区块高度上执行这些更改，而无需创建治理提案。

协调升级的高级策略如下：

1. 在包含破坏性更改的私有分支上修复漏洞。
2. 需要创建一个新的补丁发布版本（例如 `v8.0.0` -> `v8.0.1`），其中包含硬分叉逻辑，并在预定义的区块高度上升级到下一个破坏性版本（例如 `v9.0.0`）。
3. 验证者将其节点升级到补丁发布版本（例如 `v8.0.1`）。为了成功进行硬分叉，重要的是足够多的验证者升级到补丁发布版本，以便它们占总验证者投票权的至少2/3。
4. 在升级时间的一小时之前（对应于升级的区块高度），发布包含漏洞修复的新主要版本（例如 `v9.0.0`）。

:::info
**重要**: 发布需要提前1小时创建，因为发布二进制文件需要大约30分钟的时间，并且验证者需要缓冲时间来下载并更新他们的[cosmovisor](./automated-upgrades#using-cosmovisor)设置。
:::


---
sidebar_position: 2
---

# Hard Fork Upgrades

One of the significant limitations of the normal upgrade procedure
[via governance](./../../validate/upgrades#governance-proposal) is that it requires waiting for the
entire duration of the voting period. This duration makes it unsuitable for
automated upgrades that involve patches for security vulnerabilities or other
critical components.

A faster alternative to using governance is to create a Hard Fork procedure.
This procedure [automatically](./automated-upgrades) applies the changes from an upgrade plan, allowing
them to be executed at a given block height without the need of having to create
a governance proposal.

The high-level strategy for coordinating an upgrade is as follows:

1. The vulnerability is fixed on a private branch that contains breaking
   changes.
2. A new patch release (e.g. `v8.0.0` -> `v8.0.1`) needs to be created that
   contains a hard fork logic and performs an upgrade to the next breaking
   version (e.g. `v9.0.0`) at a predefined block height.
3. Validators upgrade their nodes to the patch release (e.g. `v8.0.1`). In order to perform the
   hard fork successfully, it’s important that enough validators upgrade to the
   patch release so that they make up at least 2/3 of the total validator voting
   power.
4. One hour before the upgrade time (corresponding to the upgrade block height),
   the new major release (e.g. `v9.0.0`) including the vulnerability fix is
   published.

:::info
**Important**: The release needs to be created with 1hr anticipation because the
release binaries take ~30min to be created and validators need a buffer time to
download them and update their
[cosmovisor](./automated-upgrades#using-cosmovisor) settings.
:::
