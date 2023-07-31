---
sidebar_position: 9
---
# 多签名

学习如何使用密钥环多签名生成、签名和广播交易。

**多签名账户**是一个具有特殊密钥的 Evmos 账户，可以要求多个签名来签署交易。
这对于增加账户的安全性或要求多个方的同意进行交易非常有用。可以通过指定以下内容来创建多签名账户：

- 所需签名的阈值数量
- 参与签名的公钥

要使用多签名账户进行签名，交易必须由账户指定的不同密钥分别签名。
然后，这些签名将被组合成一个多重签名，用于签署交易。如果存在的签名数量少于所需的阈值数量，则生成的多重签名将被视为无效。

## 生成多签名密钥

```bash
evmosd keys add --multisig=name1,name2,name3[...] --multisig-threshold=K new_key_name
```

`K` 是必须对以公钥地址为签名者的交易进行签名的最小私钥数量。

`--multisig` 标志必须包含将组合成一个公钥的公钥名称，该公钥将生成并存储为本地数据库中的 `new_key_name`。
通过 `--multisig` 提供的所有名称必须已经存在于本地数据库中。

除非设置了 `--nosort` 标志，否则在命令行中提供密钥的顺序无关紧要，即以下命令生成两个相同的密钥：

```bash
evmosd keys add --multisig=p1,p2,p3 --multisig-threshold=2 multisig_address
evmosd keys add --multisig=p2,p3,p1 --multisig-threshold=2 multisig_address
```

多签名地址也可以通过以下命令即时生成并打印：

```bash
evmosd keys show --multisig-threshold=K name1 name2 name3 [...]
```

## 签署交易

### 步骤 1：创建多签名密钥

假设您有 `test1` 和 `test2`，想要与 `test3` 创建一个多签名账户。

首先将 `test3` 的公钥导入到您的密钥环中。

```sh
evmosd keys add \
    test3 \
    --pubkey=evmospub1addwnpepqgcxazmq6wgt2j4rdfumsfwla0zfk8e5sws3p3zg5dkm9007hmfysxas0u2
```

使用2/3阈值生成多签名密钥。

```sh
evmosd keys add \
    multi \
    --multisig=test1,test2,test3 \
    --multisig-threshold=2
```

您可以查看其地址和详细信息：

```sh
evmosd keys show multi

- name: multi
  type: multi
  address: evmos1e0fx0q9meawrcq7fmma9x60gk35lpr4xk3884m
  pubkey: evmospub1ytql0csgqgfzd666axrjzq3mxw59ys6yqcd3ydjvhgs0uzs6kdk5fp4t73gmkl8t6y02yfq7tvfzd666axrjzq3sd69kp5usk492x6nehqjal67ynv0nfqapzrzy3gmdk27la0kjfqfzd666axrjzq6utqt639ka2j3xkncgk65dup06t297ccljmxhvhu3rmk92u3afjuyz9dg9
  mnemonic: ""
  threshold: 0
  pubkeys: []
```

让我们向多签名钱包添加10个EVMOS：

```bash
evmosd tx bank send \
    test1 \
    evmos1e0fx0q9meawrcq7fmma9x60gk35lpr4xk3884m \
    10000000000000000000aevmos \
    --chain-id=evmos_9000-4 \
    --gas=auto \
    --fees=1000000aevmos \
    --broadcast-mode=block
```

### 步骤2：创建多签名交易

我们希望从我们的多签名账户向`evmos1rgjxswhuxhcrhmyxlval0qa70vxwvqn2e0srft`发送5个EVMOS。

```bash
evmosd tx bank send \
    evmos1rgjxswhuxhcrhmyxlval0qa70vxwvqn2e0srft \
    evmos157g6rn6t6k5rl0dl57zha2wx72t633axqyvvwq \
    5000000000000000000aevmos \
    --gas=200000 \
    --fees=1000000aevmos \
    --chain-id=evmos_9000-4 \
    --generate-only > unsignedTx.json
```

文件`unsignedTx.json`包含以JSON编码的未签名交易。

```json
{
  "body": {
    "messages": [
      {
        "@type": "/cosmos.bank.v1beta1.MsgSend",
        "from_address": "evmos1rgjxswhuxhcrhmyxlval0qa70vxwvqn2e0srft",
        "to_address": "evmos157g6rn6t6k5rl0dl57zha2wx72t633axqyvvwq",
        "amount": [
          {
            "denom": "aevmos",
            "amount": "5000000000000000000"
          }
        ]
      }
    ],
    "memo": "",
    "timeout_height": "0",
    "extension_options": [],
    "non_critical_extension_options": []
  },
  "auth_info": {
    "signer_infos": [],
    "fee": {
      "amount": [
        {
          "denom": "aevmos",
          "amount": "1000000"
        }
      ],
      "gas_limit": "200000",
      "payer": "",
      "granter": ""
    }
  },
  "signatures": []
}
```

### 步骤3：逐个签名

使用`test1`和`test2`进行签名并创建单独的签名。

```sh
evmosd tx sign \
    unsignedTx.json \
    --multisig=evmos1e0fx0q9meawrcq7fmma9x60gk35lpr4xk3884m \
    --from=test1 \
    --output-document=test1sig.json \
    --chain-id=evmos_9000-4
```

```sh
evmosd tx sign \
    unsignedTx.json \
    --multisig=evmos1e0fx0q9meawrcq7fmma9x60gk35lpr4xk3884m \
    --from=test2 \
    --output-document=test2sig.json \
    --chain-id=evmos_9000-4
```

### 步骤4：创建多重签名

合并签名以签署交易。

```sh
evmosd tx multisign \
    unsignedTx.json \
    multi \
    test1sig.json test2sig.json \
    --output-document=signedTx.json \
    --chain-id=evmos_9000-4
```

现在交易已签名：

```json
{
  "body": {
    "messages": [
      {
        "@type": "/cosmos.bank.v1beta1.MsgSend",
        "from_address": "evmos1rgjxswhuxhcrhmyxlval0qa70vxwvqn2e0srft",
        "to_address": "evmos157g6rn6t6k5rl0dl57zha2wx72t633axqyvvwq",
        "amount": [
          {
            "denom": "aevmos",
            "amount": "5000000000000000000"
          }
        ]
      }
    ],
    "memo": "",
    "timeout_height": "0",
    "extension_options": [],
    "non_critical_extension_options": []
  },
  "auth_info": {
    "signer_infos": [
      {
        "public_key": {
          "@type": "/cosmos.crypto.multisig.LegacyAminoPubKey",
          "threshold": 2,
          "public_keys": [
            {
              "@type": "/cosmos.crypto.secp256k1.PubKey",
              "key": "ApCzSG8k7Tr4aM6e4OJRExN7cNtvH21L9azbh+uRrvt4"
            },
            {
              "@type": "/cosmos.crypto.secp256k1.PubKey",
              "key": "Ah91erz8ChNanqLe9ea948rvAiXMCRlR5Ka7EE/c0xUK"
            },
            {
              "@type": "/cosmos.crypto.secp256k1.PubKey",
              "key": "A0OjtIUCFJM3AobJ9HJTWKP9RZV2+WPcwVjLgsAidrZ/"
            }
          ]
        },
        "mode_info": {
          "multi": {
            "bitarray": {
              "extra_bits_stored": 3,
              "elems": "wA=="
            },
            "mode_infos": [
              {
                "single": {
                  "mode": "SIGN_MODE_LEGACY_AMINO_JSON"
                }
              },
              {
                "single": {
                  "mode": "SIGN_MODE_LEGACY_AMINO_JSON"
                }
              }
            ]
          }
        },
        "sequence": "1"
      }
    ],
    "fee": {
      "amount": [
        {
          "denom": "aevmos",
          "amount": "1000000"
        }
      ],
      "gas_limit": "200000",
      "payer": "",
      "granter": ""
    }
  },
  "signatures": [
    "CkCEeIbeGc+I1ipZuhp/0KhVNnWAv2tTlvgo5x61lzk1KHmLPV38m/YFurrFt5cm5+fqIXrn+FlOjrJuzBhw8ogYCkCawm9mpXsBHk0CFsE5618fVnvScEkfrzW0c2jCcjqV8EPuj3ut74UWzZyQkwtJGxUWtro9EgnGsB7Di1Gzizst"
  ]
}
```

### 步骤5：广播交易

```sh
evmosd tx broadcast signedTx.json \
    --chain-id=evmos_9000-4 \
    --broadcast-mode=block
```


---
sidebar_position: 9
---
# Multisig

Learn how to generate, sign and broadcast a transaction using the keyring multisig.

A **multisig account** is an Evmos account with a special key that can require more than one signature to sign transactions.
 This can be useful for increasing the security of the account or for requiring the consent of multiple parties to make
  transactions. Multisig accounts can be created by specifying:

- threshold number of signatures required
- the public keys involved in signing

To sign with a multisig account, the transaction must be signed individually by the different keys specified for the account.
 Then, the signatures will be combined into a multi-signature which can be used to sign the transaction. If fewer than the
  threshold number of signatures needed are present, the resultant multi-signature is considered invalid.

## Generate a Multisig key

```bash
evmosd keys add --multisig=name1,name2,name3[...] --multisig-threshold=K new_key_name
```

`K` is the minimum number of private keys that must have signed the transactions that carry the public key's address as signer.

The `--multisig` flag must contain the name of public keys that will be combined into a public key that will be
generated and stored as `new_key_name` in the local database. All names supplied through `--multisig` must already exist
 in the local database.

Unless the flag `--nosort` is set, the order in which the keys are supplied on the command line does not matter, i.e. the
 following commands generate two identical keys:

```bash
evmosd keys add --multisig=p1,p2,p3 --multisig-threshold=2 multisig_address
evmosd keys add --multisig=p2,p3,p1 --multisig-threshold=2 multisig_address
```

Multisig addresses can also be generated on-the-fly and printed through the which command:

```bash
evmosd keys show --multisig-threshold=K name1 name2 name3 [...]
```

## Signing a transaction

### Step 1: Create the multisig key

Let's assume that you have `test1` and `test2` want to make a multisig account with `test3`.

First import the public keys of `test3` into your keyring.

```sh
evmosd keys add \
    test3 \
    --pubkey=evmospub1addwnpepqgcxazmq6wgt2j4rdfumsfwla0zfk8e5sws3p3zg5dkm9007hmfysxas0u2
```

Generate the multisig key with 2/3 threshold.

```sh
evmosd keys add \
    multi \
    --multisig=test1,test2,test3 \
    --multisig-threshold=2
```

You can see its address and details:

```sh
evmosd keys show multi

- name: multi
  type: multi
  address: evmos1e0fx0q9meawrcq7fmma9x60gk35lpr4xk3884m
  pubkey: evmospub1ytql0csgqgfzd666axrjzq3mxw59ys6yqcd3ydjvhgs0uzs6kdk5fp4t73gmkl8t6y02yfq7tvfzd666axrjzq3sd69kp5usk492x6nehqjal67ynv0nfqapzrzy3gmdk27la0kjfqfzd666axrjzq6utqt639ka2j3xkncgk65dup06t297ccljmxhvhu3rmk92u3afjuyz9dg9
  mnemonic: ""
  threshold: 0
  pubkeys: []
```

Let's add 10 EVMOS to the multisig wallet:

```bash
evmosd tx bank send \
    test1 \
    evmos1e0fx0q9meawrcq7fmma9x60gk35lpr4xk3884m \
    10000000000000000000aevmos \
    --chain-id=evmos_9000-4 \
    --gas=auto \
    --fees=1000000aevmos \
    --broadcast-mode=block
```

### Step 2: Create the multisig transaction

We want to send 5 EVMOS from our multisig account to `evmos1rgjxswhuxhcrhmyxlval0qa70vxwvqn2e0srft`.

```bash
evmosd tx bank send \
    evmos1rgjxswhuxhcrhmyxlval0qa70vxwvqn2e0srft \
    evmos157g6rn6t6k5rl0dl57zha2wx72t633axqyvvwq \
    5000000000000000000aevmos \
    --gas=200000 \
    --fees=1000000aevmos \
    --chain-id=evmos_9000-4 \
    --generate-only > unsignedTx.json
```

The file `unsignedTx.json` contains the unsigned transaction encoded in JSON.

```json
{
  "body": {
    "messages": [
      {
        "@type": "/cosmos.bank.v1beta1.MsgSend",
        "from_address": "evmos1rgjxswhuxhcrhmyxlval0qa70vxwvqn2e0srft",
        "to_address": "evmos157g6rn6t6k5rl0dl57zha2wx72t633axqyvvwq",
        "amount": [
          {
            "denom": "aevmos",
            "amount": "5000000000000000000"
          }
        ]
      }
    ],
    "memo": "",
    "timeout_height": "0",
    "extension_options": [],
    "non_critical_extension_options": []
  },
  "auth_info": {
    "signer_infos": [],
    "fee": {
      "amount": [
        {
          "denom": "aevmos",
          "amount": "1000000"
        }
      ],
      "gas_limit": "200000",
      "payer": "",
      "granter": ""
    }
  },
  "signatures": []
}
```

### Step 3: Sign individually

Sign with `test1` and `test2` and create individual signatures.

```sh
evmosd tx sign \
    unsignedTx.json \
    --multisig=evmos1e0fx0q9meawrcq7fmma9x60gk35lpr4xk3884m \
    --from=test1 \
    --output-document=test1sig.json \
    --chain-id=evmos_9000-4
```

```sh
evmosd tx sign \
    unsignedTx.json \
    --multisig=evmos1e0fx0q9meawrcq7fmma9x60gk35lpr4xk3884m \
    --from=test2 \
    --output-document=test2sig.json \
    --chain-id=evmos_9000-4
```

### Step 4: Create multisignature

Combine signatures to sign transaction.

```sh
evmosd tx multisign \
    unsignedTx.json \
    multi \
    test1sig.json test2sig.json \
    --output-document=signedTx.json \
    --chain-id=evmos_9000-4
```

The TX is now signed:

```json
{
  "body": {
    "messages": [
      {
        "@type": "/cosmos.bank.v1beta1.MsgSend",
        "from_address": "evmos1rgjxswhuxhcrhmyxlval0qa70vxwvqn2e0srft",
        "to_address": "evmos157g6rn6t6k5rl0dl57zha2wx72t633axqyvvwq",
        "amount": [
          {
            "denom": "aevmos",
            "amount": "5000000000000000000"
          }
        ]
      }
    ],
    "memo": "",
    "timeout_height": "0",
    "extension_options": [],
    "non_critical_extension_options": []
  },
  "auth_info": {
    "signer_infos": [
      {
        "public_key": {
          "@type": "/cosmos.crypto.multisig.LegacyAminoPubKey",
          "threshold": 2,
          "public_keys": [
            {
              "@type": "/cosmos.crypto.secp256k1.PubKey",
              "key": "ApCzSG8k7Tr4aM6e4OJRExN7cNtvH21L9azbh+uRrvt4"
            },
            {
              "@type": "/cosmos.crypto.secp256k1.PubKey",
              "key": "Ah91erz8ChNanqLe9ea948rvAiXMCRlR5Ka7EE/c0xUK"
            },
            {
              "@type": "/cosmos.crypto.secp256k1.PubKey",
              "key": "A0OjtIUCFJM3AobJ9HJTWKP9RZV2+WPcwVjLgsAidrZ/"
            }
          ]
        },
        "mode_info": {
          "multi": {
            "bitarray": {
              "extra_bits_stored": 3,
              "elems": "wA=="
            },
            "mode_infos": [
              {
                "single": {
                  "mode": "SIGN_MODE_LEGACY_AMINO_JSON"
                }
              },
              {
                "single": {
                  "mode": "SIGN_MODE_LEGACY_AMINO_JSON"
                }
              }
            ]
          }
        },
        "sequence": "1"
      }
    ],
    "fee": {
      "amount": [
        {
          "denom": "aevmos",
          "amount": "1000000"
        }
      ],
      "gas_limit": "200000",
      "payer": "",
      "granter": ""
    }
  },
  "signatures": [
    "CkCEeIbeGc+I1ipZuhp/0KhVNnWAv2tTlvgo5x61lzk1KHmLPV38m/YFurrFt5cm5+fqIXrn+FlOjrJuzBhw8ogYCkCawm9mpXsBHk0CFsE5618fVnvScEkfrzW0c2jCcjqV8EPuj3ut74UWzZyQkwtJGxUWtro9EgnGsB7Di1Gzizst"
  ]
}
```

### Step 5: Broadcast transaction

```sh
evmosd tx broadcast signedTx.json \
    --chain-id=evmos_9000-4 \
    --broadcast-mode=block
```
