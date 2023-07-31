---
sidebar_position: 2
---

# 使用 Docker

有多种方法可以使用 Docker 运行 Evmos。
如果您想在 Docker 设置中运行 Evmos，并且可能将 Docker 容器连接到其他容器化的兼容区块链二进制文件，请查看关于[构建包含 Evmos 二进制文件的 Docker 镜像](#构建包含二进制文件的 Docker 镜像)的指南。
如果您希望生成用于 Docker 外部使用的二进制文件，但是希望通过在 Docker 容器中构建二进制文件来确保使用正确的依赖项，请继续阅读关于[使用 Docker 构建 Evmos 二进制文件](#使用 Docker 构建二进制文件)的部分。

:::note
给定的说明已在 *Ubuntu 18.04.2 LTS* 上使用 *Docker 20.10.2* 和 *macOS 13.2.1* 上使用 *Docker 20.10.22* 进行了测试。
:::

## 先决条件

- [安装 Docker](https://docs.docker.com/get-docker/)

## 通用设置

为了使用 Docker 构建 Evmos 二进制文件，需要执行以下操作：

- 将 Evmos 存储库克隆到本地机器（例如 `git clone git@github.com/evmos/evmos.git`）
- 检出您要构建的提交、分支或发布标签（例如 `git checkout v11.0.2`）

## 构建包含二进制文件的 Docker 镜像

要构建一个包含 Evmos 二进制文件的 Docker 镜像，请进入克隆的存储库，并在终端会话中运行以下命令：

```bash
make build-docker
```

这将创建一个名为 `tharsishq/evmos` 的镜像，并带有版本标签 `latest`。
现在可以在容器中运行 `evmosd` 二进制文件，例如评估其版本：

```bash
docker run -it --rm tharsishq/evmos:latest evmosd version
```

## 使用 Docker 构建二进制文件

可以使用 Docker 确定性地构建 `evmosd` 二进制文件。
Docker 提供的容器系统可以在隔离的环境中创建 Evmos 二进制文件的实例。

### 构建镜像

运行以下命令以启动所有支持的架构（目前为 **linux/amd64**）的构建：

```bash
make distclean build-reproducible
```

构建系统会在 `artifacts` 目录中生成二进制文件和确定性构建报告。
`artifacts/build_report` 文件包含构建产物及其相应的校验和的列表，可用于验证构建的正确性。以下是其内容的示例：

### 构建镜像

[Tendermint构建器Docker镜像](https://github.com/tendermint/images/tree/master/rbuilder)
提供了一个确定性的构建环境，用于构建Cosmos SDK应用程序。
它提供了一种合理确保可执行文件确实是从git源代码构建的方式。
它还确保使用相同的经过测试的依赖项，并将其静态构建到可执行文件中。

----

现在，您已经构建了Evmos二进制文件，无论是用于本地使用还是在Docker容器中，
您将在以下部分找到有关运行节点实例的信息
关于[设置本地网络](./single-node)的信息。


---
sidebar_position: 2
---

# Working with Docker

There are multiple ways to use Evmos with Docker.
If you want to run Evmos inside a Docker setup and possibly connect the Docker container
to other containerized compatible blockchain binaries, check out the guide on
[building a Docker image containing the Evmos binary](#building-a-docker-image-containing-the-binary).
If you instead want to generate a binary for use outside of Docker,
but want to ensure the correct dependencies are used by building the binary inside a Docker container,
then go ahead to the section on [building the Evmos binary with Docker](#building-the-binary-with-docker).

:::note
The given instructions have been tested on *Ubuntu 18.04.2 LTS* with *Docker 20.10.2* and *macOS 13.2.1* with *Docker 20.10.22*.
:::

## Prerequisites

- [Install Docker](https://docs.docker.com/get-docker/)

## General Setup

In order to build Evmos binaries with Docker, it is necessary to

- clone the Evmos repository to your local machine (e.g. `git clone git@github.com/evmos/evmos.git`)
- checkout the commit, branch, or release tag you want to build (e.g. `git checkout v11.0.2`)

## Building A Docker Image Containing The Binary

To build a Docker image, that contains the Evmos binary,
step into the cloned repository and run the following command in a terminal session:

```bash
make build-docker
```

This will create an image with the name `tharsishq/evmos` and the version tag `latest`.
Now it is possible to run the `evmosd` binary in the container, e.g. evaluating its version:

```bash
docker run -it --rm tharsishq/evmos:latest evmosd version
```

## Building The Binary With Docker

It is possible to build the `evmosd` binary deterministically using Docker.
The container system that Docker provides offers the ability
to create an instance of the Evmos binary in an isolated environment.

### Building the Image

Run the following command to launch a build for all supported architectures (currently **linux/amd64**):

```bash
make distclean build-reproducible
```

The build system generates both the binaries and deterministic build report in the `artifacts` directory.
The `artifacts/build_report` file contains the list of the build artifacts and their respective checksums,
and can be used to verify build sanity. An example of its contents follows:

```
App: evmosd
Version: 11.0.2
Commit: 8eeeac7ae42a5b2695fea7f56868f3c6e9bc2378
Files:
 6b5939adfd9a8ce964d78fcaab16091a  evmosd-11.0.2-linux-amd64
 ac503925c535ddb8ee0fbebbb96d0eb9  evmosd-11.0.2.tar.gz
Checksums-Sha256:
 0857d59c285a87b7d354aa6d566db90c56663d938a88d41d35415da490708aea  evmosd-11.0.2-linux-amd64
 5005814fc34abc02d7e30dcfbe67e363c1b593efb774e0c97ebb7ec713baf306  evmosd-11.0.2.tar.gz
```

### Builder Image

The [Tendermint builder Docker image](https://github.com/tendermint/images/tree/master/rbuilder)
provides a deterministic build environment that is used to build Cosmos SDK applications.
It provides a way to be reasonably sure that the executables are really built from the git source.
It also makes sure that the same, tested dependencies are used and statically built into the executable.

----

Now that you have built the Evmos binary, either for local use or in a Docker container,
you'll find information to run a node instance in the following section
on [setting up a local network](./single-node).
