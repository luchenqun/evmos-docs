---
sidebar_position: 7
---

# 密钥环

使用 CLI 密钥环创建、导入、导出和删除密钥。

密钥环保存用于与节点交互的私钥/公钥对。例如，在运行节点之前需要设置验证器密钥，以便正确签名区块。私钥可以存储在不同的位置，称为["后端"](#keyring-backends)，例如文件或操作系统自己的密钥存储。

:::tip
如果需要关于私钥和密钥管理的复习，请参考我们的[密钥管理](./key-management)。
:::

## 添加密钥

您可以使用以下命令来获取有关`keys`命令的帮助以及有关特定子命令的更多信息：

```bash
evmosd keys
```

```bash
evmosd keys [command] --help
```

要在密钥环中创建一个新密钥，请使用`add`子命令和`<key_name>`参数。您需要为新生成的密钥提供密码。此密钥将在下一节中使用。

```bash
evmosd keys add dev0

# Put the generated address in a variable for later use.
MY_VALIDATOR_ADDRESS=$(evmosd keys show dev0 -a)
```

此命令生成一个新的24个单词的助记词短语，将其持久化到相关后端，并输出有关密钥对的信息。如果此密钥对将用于持有带有价值的代币，请务必将助记词短语记录在安全的地方！

默认情况下，密钥环生成一个`eth_secp256k1`密钥。密钥环还支持`ed25519`密钥，可以通过传递`--algo`标志来创建。密钥环当然可以同时持有这两种类型的密钥。

:::tip
**注意**：通过获取类型为`eth_secp256k1`的完整以太坊公钥，计算`Keccak-256`哈希，并截断前12个字节，可以派生与公钥关联的以太坊地址。
:::

:::warning
**注意**：由于与以太坊交易的兼容性问题，Evmos不支持 Cosmos `secp256k1`密钥。
:::

## 密钥环后端

### 操作系统

:::tip
**`os`** 是默认选项，因为操作系统的默认凭据管理器旨在满足用户的常见需求，并为他们提供舒适的体验，而不会影响安全性。
:::

`os` 后端依赖于操作系统特定的默认设置来安全处理密钥存储。通常，操作系统的凭据子系统根据用户的密码策略处理密码提示、私钥存储和用户会话。以下是一些最流行的操作系统及其相应的密码管理器：

- macOS（自 Mac OS 8.6 起）：[Keychain](https://support.apple.com/en-gb/guide/keychain-access/welcome/mac)
- Windows：[Credentials Management API](https://docs.microsoft.com/en-us/windows/win32/secauthn/credentials-management)
- GNU/Linux：
  - [libsecret](https://gitlab.gnome.org/GNOME/libsecret)
  - [kwallet](https://api.kde.org/frameworks/kwallet/html/index.html)

默认使用 GNOME 作为桌面环境的 GNU/Linux 发行版通常会预装 [Seahorse](https://wiki.gnome.org/Apps/Seahorse)。而基于 KDE 的发行版通常会提供 [KDE Wallet Manager](https://userbase.kde.org/KDE_Wallet_Manager)。前者实际上是 `libsecret` 的便捷前端，而后者是 `kwallet` 的客户端。

对于无头环境，推荐使用 `file` 和 `pass` 作为后端。

### File

`file` 将密钥环以加密形式存储在应用程序的配置目录中。每次访问该密钥环时都会要求输入密码，可能在单个命令中多次访问，导致重复的密码提示。如果使用 bash 脚本执行使用 `file` 选项的命令，您可能希望使用以下格式处理多个提示：

```bash
# assuming that KEYPASSWD is set in the environment
yes $KEYPASSWD | evmosd keys add me
yes $KEYPASSWD | evmosd keys show me
# start evmosd with keyring-backend flag
evmosd --keyring-backend=file start
```

:::tip
第一次向空密钥环添加密钥时，会提示您两次输入密码。
:::

### Password Store

`pass` 后端使用 [pass](https://www.passwordstore.org/) 实用程序来管理密钥的敏感数据和元数据的磁盘加密。密钥存储在应用程序特定目录中的 `gpg` 加密文件中。`pass` 可用于最流行的 UNIX 操作系统和 GNU/Linux 发行版。请参阅其手册以获取有关如何下载和安装的信息。

:::tip
**`pass`** 使用 [GnuPG](https://gnupg.org/) 进行加密。`gpg` 在执行时会自动调用 `gpg-agent` 守护进程，用于缓存 GnuPG 凭据。请参考 `gpg-agent` 的手册以获取有关如何配置缓存参数（如凭据的生存时间和密码过期时间）的更多信息。
:::

首次使用前必须设置密码存储库：

```sh
pass init <GPG_KEY_ID>
```

将 `<GPG_KEY_ID>` 替换为您的 GPG 密钥 ID。您可以使用个人 GPG 密钥或者您希望专门用于加密密码存储库的其他密钥。

### KDE 钱包管理器

`kwallet` 后端使用 `KDE 钱包管理器`，该管理器默认安装在以 KDE 为默认桌面环境的 GNU/Linux 发行版上。请参考
[KWallet 手册](https://docs.kde.org/stable5/en/kwalletmanager/kwallet5/) 以获取更多信息。

### 测试

`test` 后端是 `file` 后端的无密码变体。密钥以**未加密**的形式存储在磁盘上。此密钥环仅供<u>测试目的</u>使用。请自行承担风险！

:::danger
🚨 **危险**：<u>绝对不要</u>使用 `test` 密钥环后端创建您的主网验证人密钥。这样做可能导致通过 `eth_sendTransaction` JSON-RPC 端点远程访问您的资金，从而导致资金损失。

参考：[安全公告：配置不安全的 geth 可以使资金远程访问](https://blog.ethereum.org/2015/08/29/security-alert-insecurely-configured-geth-can-make-funds-remotely-accessible/)
:::

### 内存中

`memory` 后端将密钥存储在内存中。程序退出后，密钥将立即被删除。

:::danger
**重要**：仅供测试目的提供。**不建议**在生产环境中使用 `memory` 后端。请自行承担风险！
:::


---
sidebar_position: 7
---

# Keyring

Create, import, export and delete keys using the CLI keyring.

The keyring holds the private/public keypairs used to interact with the node. For instance, a validator key needs to be
set up before running the node, so that blocks can be correctly signed. The private key can be stored in different 
locations, called ["backends"](#keyring-backends), such as a file or the operating system's own key storage.

:::tip
In case, you need a refresher on private key and key management, please reference our [Key Management](./key-management).
:::

## Add keys

You can use the following commands for help with the `keys` command and for more information about a particular subcommand,
respectively:

```bash
evmosd keys
```

```bash
evmosd keys [command] --help
```

To create a new key in the keyring, run the `add` subcommand with a `<key_name>` argument. You will have to provide a password
for the newly generated key. This key will be used in the next section.

```bash
evmosd keys add dev0

# Put the generated address in a variable for later use.
MY_VALIDATOR_ADDRESS=$(evmosd keys show dev0 -a)
```

This command generates a new 24-word mnemonic phrase, persists it to the relevant backend, and outputs information about
the keypair. If this keypair will be used to hold value-bearing tokens, be sure to write down the mnemonic phrase 
somewhere safe!

By default, the keyring generates a `eth_secp256k1` key. The keyring also supports `ed25519` keys, which may be created 
by passing the `--algo` flag. A keyring can of course hold both types of keys simultaneously.

:::tip
**Note**: The Ethereum address associated with a public key can be derived by taking the full Ethereum public key of type 
`eth_secp256k1`, computing the `Keccak-256` hash, and truncating the first twelve bytes.
:::

:::warning
**NOTE**: Cosmos `secp256k1` keys are not supported on Evmos due to compatibility issues with Ethereum transactions.
:::

## Keyring Backends

### OS

:::tip
**`os`** is the default option since operating system's default credentials managers are
designed to meet users' most common needs and provide them with a comfortable
experience without compromising on security.
:::

The `os` backend relies on operating system-specific defaults to handle key storage
securely. Typically, an operating system's credential sub-system handles password prompts,
private keys storage, and user sessions according to the user's password policies. Here
is a list of the most popular operating systems and their respective passwords manager:

- macOS (since Mac OS 8.6): [Keychain](https://support.apple.com/en-gb/guide/keychain-access/welcome/mac)
- Windows: [Credentials Management API](https://docs.microsoft.com/en-us/windows/win32/secauthn/credentials-management)
- GNU/Linux:
- [libsecret](https://gitlab.gnome.org/GNOME/libsecret)
- [kwallet](https://api.kde.org/frameworks/kwallet/html/index.html)

GNU/Linux distributions that use GNOME as default desktop environment typically come with
[Seahorse](https://wiki.gnome.org/Apps/Seahorse). Users of KDE based distributions are
commonly provided with [KDE Wallet Manager](https://userbase.kde.org/KDE_Wallet_Manager).
Whilst the former is in fact a `libsecret` convenient frontend, the latter is a `kwallet`
client.

The recommended backends for headless environments are `file` and `pass`.

### File

The `file` stores the keyring encrypted within the app's configuration directory. This
keyring will request a password each time it is accessed, which may occur multiple
times in a single command resulting in repeated password prompts. If using bash scripts
to execute commands using the `file` option you may want to utilize the following format
for multiple prompts:

```bash
# assuming that KEYPASSWD is set in the environment
yes $KEYPASSWD | evmosd keys add me
yes $KEYPASSWD | evmosd keys show me
# start evmosd with keyring-backend flag
evmosd --keyring-backend=file start
```

:::tip
The first time you add a key to an empty keyring, you will be prompted to type the password twice.
:::

### Password Store

The `pass` backend uses the [pass](https://www.passwordstore.org/) utility to manage on-disk
encryption of keys' sensitive data and metadata. Keys are stored inside `gpg` encrypted files
within app-specific directories. `pass` is available for the most popular UNIX
operating systems as well as GNU/Linux distributions. Please refer to its manual page for
information on how to download and install it.

:::tip
**`pass`** uses [GnuPG](https://gnupg.org/) for encryption. `gpg` automatically invokes the `gpg-agent`
daemon upon execution, which handles the caching of GnuPG credentials. Please refer to `gpg-agent`
man page for more information on how to configure cache parameters such as credentials TTL and
passphrase expiration.
:::

The password store must be set up prior to first use:

```sh
pass init <GPG_KEY_ID>
```

Replace `<GPG_KEY_ID>` with your GPG key ID. You can use your personal GPG key or an alternative
one you may want to use specifically to encrypt the password store.

### KDE Wallet Manager

The `kwallet` backend uses `KDE Wallet Manager`, which comes installed by default on the
GNU/Linux distributions that ships KDE as default desktop environment. Please refer to
[KWallet Handbook](https://docs.kde.org/stable5/en/kwalletmanager/kwallet5/) for more
information.

### Testing

The `test` backend is a password-less variation of the `file` backend. Keys are stored
**unencrypted** on disk. This keyring is provided for <u>testing purposes only</u>. Use at your own risk!

:::danger
🚨 **DANGER**: <u>Never</u> create your mainnet validator keys using a `test` keying backend. Doing so might result in
a loss of funds by making your funds remotely accessible via the `eth_sendTransaction` JSON-RPC endpoint.

Ref: [Security Advisory: Insecurely configured geth can make funds remotely accessible](https://blog.ethereum.org/2015/08/29/security-alert-insecurely-configured-geth-can-make-funds-remotely-accessible/)
:::

### In Memory

The `memory` backend stores keys in memory. The keys are immediately deleted after the program has exited.

:::danger
**IMPORTANT**: Provided for testing purposes only. The `memory` backend is **not** recommended for use in production
environments. Use at your own risk!
:::
