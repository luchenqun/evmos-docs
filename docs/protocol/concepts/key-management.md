# 密钥管理

助记词短语，也被称为种子短语，是一组用于恢复或恢复加密货币钱包的单词。它作为访问您的数字资产的备份，以防您无法访问原始钱包。该短语通常是一系列由创建钱包时生成的12-24个单词，并且应该保持安全和私密。

助记词短语的重要性在于加密货币的存储方式是分散的，这意味着没有持有或控制您资金的中央机构或机构。这意味着如果您无法访问您的钱包（例如忘记密码，丢失设备），您将无法在没有助记词短语的情况下恢复您的资金。

因此，将助记词短语存储在安全可靠的地方非常重要，例如物理纸张或安全的数字文件。此外，建议制作多个副本并将它们存储在不同的位置，以便在紧急情况下可以访问您的资金。

:::note
**助记词短语与私钥**
种子短语，也称为恢复短语或备份短语，是用于生成私钥的一系列单词。它通常是一组由12或24个单词组成的短语，用于在原始私钥丢失或损坏的情况下恢复或恢复对加密货币钱包的访问。种子短语可以用于生成多个私钥，这些私钥可以用于访问多个加密货币地址和余额。

另一方面，私钥是一个由字符组成的长字符串，用于签署交易并提供对您的加密货币资金的访问。私钥是从种子短语生成的，并且对于每个加密货币地址都是唯一的。它用于为交易创建数字签名，以确保交易是合法的，并且已经得到资金的合法所有者的授权。

总之，您的私钥和助记词短语的安全性至关重要。如果您的私钥受到损害，可能会使所有相关账户面临风险。然而，助记词短语的丢失可能会带来更严重的后果，因为它用于生成多个私钥。因此，为了避免任何灾难性损失，采取适当的措施来保护您的私钥和助记词短语非常重要。

:::

## 从 Evmos CLI 获取助记词

:::note
在使用 CLI 之前，请确保已安装 `evmosd`。安装说明请参考[此处](./../../protocol/evmos-cli/single-node)。
:::

创建新密钥时，您将收到一个助记词短语，可用于恢复该密钥。备份助记词短语：

```bash
evmosd keys add dev0
{
  "name": "dev0",
  "type": "local",
  "address": "evmos1n253dl2tgyhxjm592p580c38r4dn8023ctv28d",
  "pubkey": '{"@type":"/ethermint.crypto.v1.ethsecp256k1.PubKey","key":"ArJhve4v5HkLm+F7ViASU/rAGx7YrwU4+XKV2MNJt+Cq"}',
  "mnemonic": ""
}

**Important** write this mnemonic phrase in a safe place.
It is the only way to recover your account if you ever forget your password.

# <24 word mnemonic phrase>
```

恢复密钥：

```bash
$ evmosd keys add dev0-restored --recover
> Enter your bip39 mnemonic
banner genuine height east ghost oak toward reflect asset marble else explain foster car nest make van divide twice culture announce shuffle net peanut
{
  "name": "dev0-restored",
  "type": "local",
  "address": "evmos1n253dl2tgyhxjm592p580c38r4dn8023ctv28d",
  "pubkey": '{"@type":"/ethermint.crypto.v1.ethsecp256k1.PubKey","key":"ArJhve4v5HkLm+F7ViASU/rAGx7YrwU4+XKV2MNJt+Cq"}'
}
```

## 导出密钥

### Tendermint 格式的私钥

要备份此类型的密钥而不包含助记词短语，请执行以下操作：

```bash
evmosd keys export dev0
Enter passphrase to decrypt your key:
Enter passphrase to encrypt the exported key:
-----BEGIN TENDERMINT PRIVATE KEY-----
kdf: bcrypt
salt: 14559BB13D881A86E0F4D3872B8B2C82
type: secp256k1

# <Tendermint private key>
-----END TENDERMINT PRIVATE KEY-----

$ echo "\
-----BEGIN TENDERMINT PRIVATE KEY-----
kdf: bcrypt
salt: 14559BB13D881A86E0F4D3872B8B2C82
type: secp256k1

# <Tendermint private key>
-----END TENDERMINT PRIVATE KEY-----" > dev0.export
```

### Ethereum 格式的私钥

:::tip
**注意**：这些类型的密钥与 MetaMask 兼容。
:::

要备份此类型的密钥而不包含助记词短语，请执行以下操作：

```bash
evmosd keys unsafe-export-eth-key dev0 > dev0.export
**WARNING** this is an unsafe way to export your unencrypted private key, are you sure? [y/N]: y
Enter keyring passphrase:
```

## 导入密钥

### Tendermint 格式的私钥

```bash
$ evmosd keys import dev0-imported ./dev0.export
输入解密密钥的密码：
```

### Ethereum 格式的私钥

```
$ evmosd keys unsafe-import-eth-key dev0-imported ./dev0.export
输入加密密钥的密码：
```

### 验证

使用以下命令验证已恢复密钥：

```bash
$ evmosd keys list
[
  {
    "name": "dev0-imported",
    "type": "local",
    "address": "evmos1n253dl2tgyhxjm592p580c38r4dn8023ctv28d",
    "pubkey": '{"@type":"/ethermint.crypto.v1.ethsecp256k1.PubKey","key":"ArJhve4v5HkLm+F7ViASU/rAGx7YrwU4+XKV2MNJt+Cq"}'
  },
  {
    "name": "dev0-restored",
    "type": "local",
    "address": "evmos1n253dl2tgyhxjm592p580c38r4dn8023ctv28d",
    "pubkey": '{"@type":"/ethermint.crypto.v1.ethsecp256k1.PubKey","key":"ArJhve4v5HkLm+F7ViASU/rAGx7YrwU4+XKV2MNJt+Cq"}'
  },
  {
    "name": "dev0",
    "type": "local",
    "address": "evmos1n253dl2tgyhxjm592p580c38r4dn8023ctv28d",
    "pubkey": '{"@type":"/ethermint.crypto.v1.ethsecp256k1.PubKey","key":"ArJhve4v5HkLm+F7ViASU/rAGx7YrwU4+XKV2MNJt+Cq"}'
  }
]
```


---
sidebar_position: 6
---

# Key Management

A mnemonic phrase, also known as a seed phrase, is a set of words used to recover or restore a cryptocurrency wallet.
It acts as a backup to access your digital assets in case you lose access to the original wallet. The phrase is
typically a series of 12-24 words that are generated when you create a wallet, and it should be kept secure and
 private.

The importance of mnemonic phrases lies in the fact that cryptocurrencies are stored in a decentralized manner,
meaning that there is no central authority or institution that holds or controls your funds. This means that if
you lose access to your wallet (e.g. forget your password, lose your device), you will not be able to recover
your funds without the mnemonic phrase.

Therefore, it is crucial to store your mnemonic phrase in a safe and secure place, such as a physical paper or
a secure digital file. Additionally, it is recommended to make multiple copies and store them in different
locations, so that you can access your funds in case of any emergency.

:::note
**Mnemonic Phrase vs Private Key**
A seed phrase, also known as a recovery phrase or backup phrase, is a sequence of words used to generate a private
key. It is typically a set of 12 or 24 words, and it's used to recover or restore access to a cryptocurrency wallet
in case the original private key is lost or damaged. A seed phrase can be used to generate multiple private keys,
which can be used to access multiple cryptocurrency addresses and balances.

On the other hand, a private key is a long string of characters that is used to sign transactions and provide access
to your cryptocurrency funds. The private key is generated from the seed phrase and is unique to each cryptocurrency
address. It is used to create digital signatures for transactions, which ensure that the transaction is legitimate
and has been authorized by the rightful owner of the funds.

In conclusion, the security of your private keys and mnemonic phrase is of utmost importance. If your private keys
 are compromised, it can put all associated accounts at risk. However, the loss of your mnemonic phrase can have
 even more severe consequences as it is used to generate multiple private keys. Therefore, it is crucial to take
  proper measures to safeguard both your private keys and mnemonic phrase to avoid any catastrophic loss.

:::

## Mnemonics from the Evmos CLI

:::note
Before proceeding with the CLI, please insure you have `evmosd` installed. Installation instruction are located [here](./../../protocol/evmos-cli/single-node).
:::

When you create a new key, you'll receive a mnemonic phrase that can be used to restore that key. Backup the mnemonic phrase:

```bash
evmosd keys add dev0
{
  "name": "dev0",
  "type": "local",
  "address": "evmos1n253dl2tgyhxjm592p580c38r4dn8023ctv28d",
  "pubkey": '{"@type":"/ethermint.crypto.v1.ethsecp256k1.PubKey","key":"ArJhve4v5HkLm+F7ViASU/rAGx7YrwU4+XKV2MNJt+Cq"}',
  "mnemonic": ""
}

**Important** write this mnemonic phrase in a safe place.
It is the only way to recover your account if you ever forget your password.

# <24 word mnemonic phrase>
```

To restore the key:

```bash
$ evmosd keys add dev0-restored --recover
> Enter your bip39 mnemonic
banner genuine height east ghost oak toward reflect asset marble else explain foster car nest make van divide twice culture announce shuffle net peanut
{
  "name": "dev0-restored",
  "type": "local",
  "address": "evmos1n253dl2tgyhxjm592p580c38r4dn8023ctv28d",
  "pubkey": '{"@type":"/ethermint.crypto.v1.ethsecp256k1.PubKey","key":"ArJhve4v5HkLm+F7ViASU/rAGx7YrwU4+XKV2MNJt+Cq"}'
}
```

## Export Key

### Tendermint-Formatted Private Keys

To backup this type of key without the mnemonic phrase, do the following:

```bash
evmosd keys export dev0
Enter passphrase to decrypt your key:
Enter passphrase to encrypt the exported key:
-----BEGIN TENDERMINT PRIVATE KEY-----
kdf: bcrypt
salt: 14559BB13D881A86E0F4D3872B8B2C82
type: secp256k1

# <Tendermint private key>
-----END TENDERMINT PRIVATE KEY-----

$ echo "\
-----BEGIN TENDERMINT PRIVATE KEY-----
kdf: bcrypt
salt: 14559BB13D881A86E0F4D3872B8B2C82
type: secp256k1

# <Tendermint private key>
-----END TENDERMINT PRIVATE KEY-----" > dev0.export
```

### Ethereum-Formatted Private Keys

:::tip
**Note**: These types of keys are MetaMask-compatible.
:::

To backup this type of key without the mnemonic phrase, do the following:

```bash
evmosd keys unsafe-export-eth-key dev0 > dev0.export
**WARNING** this is an unsafe way to export your unencrypted private key, are you sure? [y/N]: y
Enter keyring passphrase:
```

## Import Key

### Tendermint-Formatted Private Keys

```bash
$ evmosd keys import dev0-imported ./dev0.export
Enter passphrase to decrypt your key:
```

### Ethereum-Formatted Private Keys

```
$ evmosd keys unsafe-import-eth-key dev0-imported ./dev0.export
Enter passphrase to encrypt your key:
```

### Verification

Verify that your key has been restored using the following command:

```bash
$ evmosd keys list
[
  {
    "name": "dev0-imported",
    "type": "local",
    "address": "evmos1n253dl2tgyhxjm592p580c38r4dn8023ctv28d",
    "pubkey": '{"@type":"/ethermint.crypto.v1.ethsecp256k1.PubKey","key":"ArJhve4v5HkLm+F7ViASU/rAGx7YrwU4+XKV2MNJt+Cq"}'
  },
  {
    "name": "dev0-restored",
    "type": "local",
    "address": "evmos1n253dl2tgyhxjm592p580c38r4dn8023ctv28d",
    "pubkey": '{"@type":"/ethermint.crypto.v1.ethsecp256k1.PubKey","key":"ArJhve4v5HkLm+F7ViASU/rAGx7YrwU4+XKV2MNJt+Cq"}'
  },
  {
    "name": "dev0",
    "type": "local",
    "address": "evmos1n253dl2tgyhxjm592p580c38r4dn8023ctv28d",
    "pubkey": '{"@type":"/ethermint.crypto.v1.ethsecp256k1.PubKey","key":"ArJhve4v5HkLm+F7ViASU/rAGx7YrwU4+XKV2MNJt+Cq"}'
  }
]
```
