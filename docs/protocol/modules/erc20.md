# `erc20`

:::tip
**注意：**正在处理与 ERC-20 模块相关的治理提案吗？
请确保查看 [Evmos 治理](https://academy.evmos.org/articles/advanced/governance/)，
特别是 [最佳实践](https://academy.evmos.org/articles/advanced/governance/best-practices)。
:::

## 摘要

本文档规定了 Evmos Hub 的内部 `x/erc20` 模块。

`x/erc20` 模块使 Evmos Hub 能够支持 Evmos 的 EVM 和 Cosmos 运行时之间的令牌的双向、无信任、链上转换，
具体来说是 `x/evm` 和 `x/bank` 模块。
这使得 Evmos 上的令牌持有人可以即时将其本地 Cosmos `sdk.Coins`（在本文档中称为 "Coin(s)"）转换为 ERC-20（也称为 "Token(s)"），
反之亦然，同时保持与发行环境/运行时（EVM 或 Cosmos）上的原始资产的可替代性，并保留 ERC-20 合约的所有权。

此转换功能完全由本地 EVMOS 令牌持有人管理，
他们管理着规范的 `TokenPair` 注册（即 ERC20 ←→ Coin 映射）。
此治理功能使用 Cosmos-SDK 的 `gov` 模块实现，
使用自定义提案类型分别进行规范的映射注册和更新。

为什么这很重要？Cosmos 和 EVM 是两个默认情况下不兼容的运行时。
本地 Cosmos Coins 无法在需要 ERC-20 标准的应用程序中使用。
Cosmos coins 存在于 `x/bank` 模块（可以访问模块方法，如查询供应量或余额），
而 ERC-20 令牌存在于智能合约中。
这个问题类似于 [wETH](https://coinmarketcap.com/alexandria/article/what-is-wrapped-ethereum-weth)，
不同之处在于它不仅适用于燃料令牌（如 EVMOS），
还适用于所有 Cosmos Coins（IBC 代金券、质押和治理代币等）。

通过 `x/erc20`，Evmos 用户可以：

- 在基于 EVM 的链上使用现有的本地 Cosmos 资产（如 OSMO 或 ATOM），例如
用于在 DeFi 协议上交易 IBC 代币，购买 NFT 等。
- 将现有的以太坊和其他基于 EVM 的链上的令牌转移到 Evmos，
以利用 Cosmos 生态系统中的特定应用链。
- 构建基于 ERC-20 智能合约的新应用程序，并访问 Cosmos 生态系统。

## 目录

1. **[概念](#概念)**
2. **[状态](#状态)**
3. **[状态转换](#状态转换)**
4. **[交易](#交易)**
5. **[钩子](#钩子)**
6. **[事件](#事件)**
7. **[参数](#参数)**
8. **[客户端](#客户端)**

## 概念

### 代币对

`x/erc20` 模块维护了一个原生 Cosmos Coin 与 ERC20 代币合约地址之间的一对一映射关系（即 `sdk.Coin` ←→ ERC20），称为 `TokenPair`。
给定一对代币，可以通过治理机制启用或禁用 ERC20 代币与 Cosmos Coin 之间的转换。

### 代币对注册

用户可以通过治理模块注册一个新的代币对提案，并发起投票将该代币对纳入模块中。
根据先存在的是代币还是币种，可以注册 Cosmos Coin 或 ERC20 代币来创建代币对。
一个提案可以包含多个代币对。

当提案通过时，erc20 模块会在应用的存储中注册 Cosmos Coin 和 ERC20 代币的映射关系。

#### 注册 Cosmos Coin

原生的 Cosmos Coin 对应于 bank 模块中的 `sdk.Coin`。
它可以是原生的质押/燃料币种（例如 EVMOS、ATOM 等），
也可以是 IBC 可互换代币凭证（即 denom 格式为 `ibc/{hash}`）。

当为现有的原生 Cosmos Coin 发起提案时，
erc20 模块将部署一个 ERC20 合约工厂，
代表该代币对的 ERC20 代币，
并使模块拥有该合约的所有权。

#### 注册 ERC20 代币

也可以发起一个现有（已部署）ERC20 合约的提案。
在这种情况下，ERC20 代币保留合约的原始所有者，
并使用类似于 [ICS20 - 可互换代币转移](https://github.com/cosmos/ibc/blob/master/spec/app/ics-020-fungible-token-transfer) 规范定义的托管和铸造/销毁和解托机制。
代币对由原始的 ERC20 代币和相应的原生 Cosmos Coin 币种组成。

#### 代币详情和元数据

代币的元数据是从ERC20代币的详情（名称、符号、小数位数）派生出来的，反之亦然。
还有一种特殊情况是描述了IBC可互换代币（ICS20）的ERC20表示。

#### 代币元数据到ERC20详情

在注册Cosmos代币时，使用以下银行`元数据`来部署ERC20合约：

- **名称**
- **符号**
- **小数位数**

与ERC20相比，原生的Cosmos代币包含更详细的元数据，
并包含了转换为ERC20代币所需的所有必要细节，
无需额外填充数据。

#### IBC凭证元数据到ERC20详情

IBC凭证应符合以下标准：

- **名称**：`{名称} channel-{通道}`
- **符号**：`ibc{名称}-{通道}`
- **小数位数**：从银行`元数据`派生

#### ERC20详情到代币元数据

在注册ERC20代币时，代币元数据是从ERC20元数据和银行元数据派生出来的：

- **描述**：`Cosmos代币的{合约地址}表示`
- **DenomUnits**：
    - 代币：`0`
    - ERC20：`{uint32(erc20Data.Decimals)}`
- **基础**：`{"erc20/%s", 地址}`
- **显示**：`{erc20Data.Name}`
- **名称**：`{types.CreateDenom(strContract)}`
- **符号**：`{erc20Data.Symbol}`

### 代币对修饰符

可以通过几个治理提案修改有效的代币对。
通过`ToggleTokenConversionProposal`，可以切换代币对的内部转换，
从而可以启用或禁用代币对之间的转换。

### 代币转换

一旦代币对提案通过，模块允许对该代币对进行转换。
Evmos链上的原生Cosmos代币和IBC凭证持有者可以将其代币转换为ERC20代币，
然后可以在Evmos EVM中使用，通过创建`ConvertCoin`交易来实现。
反之，`ConvertERC20`交易允许Evmos链上的ERC20代币持有者
将ERC20代币转换回其原生的Cosmos代币表示。

根据ERC20合约的所有权情况，
在转换过程中，ERC20代币要么遵循燃烧/铸造机制，要么遵循转账/托管机制。

### 恶意合约

ERC20标准是一个接口，它定义了一组方法签名（名称、参数和输出），但没有定义方法的内部逻辑。
因此，开发人员有可能部署包含隐藏恶意行为的合约。
例如，ERC20的`transfer`方法负责将一定数量的代币发送给指定的接收者，但它可能包含代码，将一部分代币转移到由恶意合约部署者拥有的不同预定义账户中。

更复杂的恶意实现可能还会继承自包含恶意行为的自定义ERC20合约的代码。
有关更详细的示例，请参阅x/erc20审计，第`IF-EVMOS-06: IERC20 Contracts may execute arbitrary code`节。

由于`x/erc20`模块允许通过治理注册任意的ERC20合约，因此在投票阶段，提议者或投票者需要手动验证所提议的合约是否使用了默认的ERC20.sol实现。

以下是我们对审核过程的建议：

- 合约的Solidity代码应该经过验证并可访问（例如使用区块链浏览器）
- 合约应该由可信的审计机构进行审计
- 继承的合约需要进行正确性验证


## 状态

### 状态对象

`x/erc20`模块在状态中保存以下对象：

| 状态对象           | 描述                                       | 键                          | 值                | 存储    |
| ------------------ | ------------------------------------------ | --------------------------- | ------------------ | --- |
| `TokenPair`        | 代币对的字节码                             | `[]byte{1} + []byte(id)`    | `[]byte{tokenPair}` | KV    |
| `TokenPairByERC20` | 通过ERC20合约字节码的代币对id字节码         | `[]byte{2} + []byte(erc20)` | `[]byte(id)`       | KV    |
| `TokenPairByDenom` | 通过denom字符串的代币对id字节码             | `[]byte{3} + []byte(denom)` | `[]byte(id)`       | KV    |

#### 代币对

将本机 Cosmos 币种与 ERC20 代币合约地址进行一对一映射（即 `sdk.Coin` ←→ ERC20）。

```go
type TokenPair struct {
	// address of ERC20 contract token
	Erc20Address string `protobuf:"bytes,1,opt,name=erc20_address,json=erc20Address,proto3" json:"erc20_address,omitempty"`
	// cosmos base denomination to be mapped to
	Denom string `protobuf:"bytes,2,opt,name=denom,proto3" json:"denom,omitempty"`
	// shows token mapping enable status
	Enabled bool `protobuf:"varint,3,opt,name=enabled,proto3" json:"enabled,omitempty"`
	// ERC20 owner address ENUM (0 invalid, 1 ModuleAccount, 2 external address
	ContractOwner Owner `protobuf:"varint,4,opt,name=contract_owner,json=contractOwner,proto3,enum=evmos.erc20.v1.Owner" json:"contract_owner,omitempty"`
}
```

#### 代币对 ID

通过使用以下函数获取 ERC20 十六进制合约地址和币种的 SHA256 哈希来获得 `TokenPair` 的唯一标识符：

```tsx
tokenPairId = sha256(erc20 + "|" + denom)
```

#### 代币来源

`ConvertCoin` 和 `ConvertERC20` 功能使用所有者字段来检查所使用的代币是本机 Coin 还是本机 ERC20。
该字段基于代币注册提案类型（`RegisterCoinProposal` = 1，`RegisterERC20Proposal` = 2）。

`Owner` 枚举了 ERC20 合约的所有权。

```go
type Owner int32

const (
	// OWNER_UNSPECIFIED defines an invalid/undefined owner.
	OWNER_UNSPECIFIED Owner = 0
	// OWNER_MODULE erc20 is owned by the erc20 module account.
	OWNER_MODULE Owner = 1
	// EXTERNAL erc20 is owned by an external account.
	OWNER_EXTERNAL Owner = 2
)
```

可以使用以下辅助函数检查 `Owner`：

```go
// IsNativeCoin returns true if the owner of the ERC20 contract is the
// erc20 module account
func (tp TokenPair) IsNativeCoin() bool {
	return tp.ContractOwner == OWNER_MODULE
}

// IsNativeERC20 returns true if the owner of the ERC20 contract not the
// erc20 module account
func (tp TokenPair) IsNativeERC20() bool {
	return tp.ContractOwner == OWNER_EXTERNAL
}
```

#### 根据 ERC20 和币种查询代币对

`TokenPairByERC20` 和 `TokenPairByDenom` 是用于查询代币对 ID 的附加状态对象。

### 创世状态

`x/erc20` 模块的 `GenesisState` 定义了从先前导出的高度初始化链所需的状态。
它包含模块参数和已注册的代币对：

```go
// GenesisState defines the module's genesis state.
type GenesisState struct {
	// module parameters
	Params Params `protobuf:"bytes,1,opt,name=params,proto3" json:"params"`
	// registered token pairs
	TokenPairs []TokenPair `protobuf:"bytes,2,rep,name=token_pairs,json=tokenPairs,proto3" json:"token_pairs"`
}
```


## 状态转换

erc20 模块允许两种类型的注册状态转换。
根据代币对是使用 `RegisterCoinProposal` 还是 `RegisterERC20Proposal` 注册，有四种可能的转换状态。

### 代币对注册

Cosmos 币和 ERC20 代币注册都允许使用一个提案注册多个代币对。
为简单起见，以下描述仅描述了每个提案注册一个代币对的情况。

#### 1. 注册 Coin

用户注册一个本机 Cosmos Coin。
一旦提案通过（即由治理批准），
ERC20 模块使用工厂模式部署 Cosmos Coin 的 ERC20 代币合约表示。
请注意，无法注册本机 Evmos 币，
因为任何币种中包含 "evm" 的币种都无法注册。
相反，Evmos 代币可以通过 Nomand 的包装 Evmos（WEVMOS）合约进行转换。

1. 用户提交一个 `RegisterCoinProposal`。
2. Evmos Hub 的验证者使用 `MsgVote` 对提案进行投票，提案通过。
3. 如果 Cosmos 币或 IBC 凭证存在于银行模块的供应中，
   在基于 ERC20Mintable 接口的 EVM 上创建 [ERC20 代币合约](https://github.com/evmos/evmos/blob/main/contracts/ERC20MinterBurnerDecimals.sol)，
   基于银行模块提案内容中的 `Metadata` 字段派生出代币的详细信息（名称、符号、小数位数等）。

#### 2. 注册 ERC20

用户注册一个已经部署在 EVM 模块上的 ERC20 代币合约。
一旦提案通过（即由治理机构批准），
ERC20 模块将创建一个 ERC20 代币的 Cosmos 币表示。

1. 用户提交一个 `RegisterERC20Proposal`。
2. EVMOS 链的验证者使用 `MsgVote` 对提案进行投票，提案通过。
3. 如果 ERC-20 合约已经部署在 EVM 模块上，从 ERC20 的详细信息创建一个银行币的 `Metadata`。

### 代币对转换

可以通过以下方式进行已注册的 `TokenPair` 的转换：

- Cosmos 交易（`ConvertCoin` 和 `ConvertERC20`）
- 以太坊交易（即发送一个利用 EVM 钩子的 `MsgEthereumTx`）

#### 1. 已注册的币种

:::tip
👉 **上下文：** 通过 `RegisterCoinProposal` 治理提案创建了一个 `TokenPair`。
该提案创建了一个 ERC20 合约
（[ERC20Mintable by openzeppelin](https://github.com/OpenZeppelin/openzeppelin-contracts/tree/master/contracts/token/ERC20)），
将其分配为合约的 `owner`，从而赋予其调用 ERC20 的 `mint()` 和 `burnFrom()` 方法的权限。
:::

##### 不变量

- 只有 `ModuleAccount` 应该具有 ERC20 的 Minter 角色。否则，
  用户可以单方面地铸造无限供应的 ERC20 代币，
  然后将其转换为本地币种
- 用户和 `ModuleAccount`（所有者）应该是 Cosmos 币的 Burn 角色的唯一持有者
- 不应该存在任何不属于治理机构所有的本地 Cosmos 币 ERC20 合约（例如 Evmos、Atom、Osmo ERC20 合约）
- 代币/币种供应始终保持不变：
    - 总币种供应 = 币种 + 托管币种
    - 总代币供应 = 托管币种 = 铸造的代币

##### 1.1 Coin转ERC20

1. 用户提交`ConvertCoin`交易
2. 检查是否允许进行该交易对的转换，包括发送者和接收者
    - 全局参数已启用
    - 代币对已启用
    - 发送者代币未锁定（在银行模块中进行检查）
    - 接收者地址未列入黑名单
3. 如果Coin是原生的Cosmos Coin，并且Token Owner是`ModuleAccount`
    1. 通过将它们发送到erc20模块账户来托管Cosmos Coin
    2. 调用`mint()`从`ModuleAccount`地址铸造ERC20代币，并将铸造的代币发送到接收者地址
4. 检查代币余额是否增加了指定数量

##### 1.2 ERC20转Coin

1. 用户提交`ConvertERC20`交易
2. 检查是否允许进行该交易对的转换，包括发送者和接收者（参见[1.1 Coin转ERC20](#11-coin转erc20)）
3. 如果代币是ERC20代币，并且Token Owner是`ModuleAccount`
    1. 在ERC20上调用`burnCoins()`来销毁用户余额中的ERC20代币
    2. 从模块向接收者地址发送Coins（之前托管的，参见[1.1 Coin转ERC20](#11-coin转erc20)）
4. 检查
    - Coin余额是否增加了指定数量
    - 代币余额是否减少了指定数量

#### 2. 已注册的ERC20代币

:::tip
👉 **背景：** 通过`RegisterERC20Proposal`治理提案创建了一个`TokenPair`。
`ModuleAccount`不是合约的所有者，因此无法代表用户铸造新代币或销毁代币。
下面描述的机制遵循ICS20标准的相同模型，使用托管和铸造/销毁和解除托管逻辑。
:::

##### 不变量

- EVM运行时的ERC20代币供应始终保持不变：
    - 托管的ERC20代币 + 铸造的Cosmos Coin代表ERC20 = 销毁的ERC20代币的Cosmos Coin代表 + 未托管的ERC20代币
        - 将10个ERC20转换为Coin，总供应增加10个。在Cosmos侧进行铸造，EVM上不会有供应变化
        - 将10个Coin转换为ERC20，总供应减少10个。在Cosmos侧进行销毁，EVM上供应不变
    - 总ERC20代币供应 = 未托管的代币 + 托管的代币（在模块账户地址上）
    - 原生ERC20的总Coin供应 = 模块账户上托管的ERC20代币（即余额） = 铸造的Coins

##### 2.1 ERC20转Coin

1. 用户提交`ConvertERC20`交易
2. 检查是否允许对该配对、发送者和接收者进行转换（参见[1.1 Coin转ERC20](#11-coin-to-erc20)）
3. 如果代币是ERC20代币且代币所有者**不是**`ModuleAccount`
    1. 通过将它们发送到ERC20模块账户来托管ERC20代币
    2. 铸造相应配对面额的Cosmos币，并将币发送到接收者地址
4. 检查
    - 币余额增加了指定数量
    - 代币余额减少了指定数量
5. 如果在日志中发现了意外的`Approval`事件，则失败，以防止恶意合约行为

##### 2.2 Coin转ERC20

1. 用户提交`ConvertCoin`交易
2. 检查是否允许对该配对、发送者和接收者进行转换
3. 如果币是原生的Cosmos币且代币所有者**不是**`ModuleAccount`
    1. 通过将它们发送到ERC20模块账户来托管Cosmos币
    2. 通过将托管的ERC20发送到接收者地址，解锁托管的ERC20
    3. 销毁托管的Cosmos币
4. 检查代币余额增加了指定数量
5. 如果在日志中发现了意外的`Approval`事件，则失败，以防止恶意合约行为


## 交易

本节定义了导致前一节中定义的状态转换的`sdk.Msg`具体类型。

### `RegisterCoinProposal`

一种gov `Content`类型，用于从Cosmos币注册一个代币配对。
治理用户对此提案进行投票，
并在投票通过时自动执行`RegisterCoinProposal`的自定义处理程序。

```go
type RegisterCoinProposal struct {
	// title of the proposal
	Title string `protobuf:"bytes,1,opt,name=title,proto3" json:"title,omitempty"`
	// proposal description
	Description string `protobuf:"bytes,2,opt,name=description,proto3" json:"description,omitempty"`
	// metadata slice of the native Cosmos coins
	Metadata []types.Metadata `protobuf:"bytes,3,rep,name=metadata,proto3" json:"metadata"`
}
```

如果提案内容的状态验证失败，则有以下情况：

- 标题无效（长度或字符）
- 描述无效（长度或字符）
- 元数据无效
    - 名称和符号不能为空
    - 基础和显示单位是有效的币单位
    - 基础和显示单位在DenomUnit切片中存在
    - 基础单位的指数为0
    - 单位的命名按升序排序
    - 单位的命名不重复

### `RegisterERC20Proposal`

一种gov `Content`类型，用于从ERC20代币注册一个代币配对。
治理用户对此提案进行投票，
并在投票通过时自动执行`RegisterERC20Proposal`的自定义处理程序。

```go
type RegisterERC20Proposal struct {
	// title of the proposal
	Title string `protobuf:"bytes,1,opt,name=title,proto3" json:"title,omitempty"`
	// proposal description
	Description string `protobuf:"bytes,2,opt,name=description,proto3" json:"description,omitempty"`
	// contract addresses of ERC20 tokens
	Erc20Addresses []string `protobuf:"bytes,3,rep,name=erc20addresses,proto3" json:"erc20addresses,omitempty"`
}
```

如果以下情况发生，提案内容的无状态验证将失败：

- 标题无效（长度或字符）
- 描述无效（长度或字符）
- ERC20 地址无效

### `MsgConvertCoin`

用户广播一个 `MsgConvertCoin` 消息，将 Cosmos Coin 转换为 ERC20 代币。

```go
type MsgConvertCoin struct {
	// Cosmos coin which denomination is registered on erc20 bridge.
	// The coin amount defines the total ERC20 tokens to convert.
	Coin types.Coin `protobuf:"bytes,1,opt,name=coin,proto3" json:"coin"`
	// recipient hex address to receive ERC20 token
	Receiver string `protobuf:"bytes,2,opt,name=receiver,proto3" json:"receiver,omitempty"`
	// cosmos bech32 address from the owner of the given ERC20 tokens
	Sender string `protobuf:"bytes,3,opt,name=sender,proto3" json:"sender,omitempty"`
}
```

如果以下情况发生，消息的无状态验证将失败：

- Coin 无效（无效的 denom 或非正数金额）
- 接收者的十六进制地址无效
- 发送者的 bech32 地址无效

### `MsgConvertERC20`

用户广播一个 `MsgConvertERC20` 消息，将 ERC20 代币转换为本机的 Cosmos coin。

```go
type MsgConvertERC20 struct {
	// ERC20 token contract address registered on erc20 bridge
	ContractAddress string `protobuf:"bytes,1,opt,name=contract_address,json=contractAddress,proto3" json:"contract_address,omitempty"`
	// amount of ERC20 tokens to mint
	Amount github_com_cosmos_cosmos_sdk_types.Int `protobuf:"bytes,2,opt,name=amount,proto3,customtype=github.com/cosmos/cosmos-sdk/types.Int" json:"amount"`
	// bech32 address to receive SDK coins.
	Receiver string `protobuf:"bytes,3,opt,name=receiver,proto3" json:"receiver,omitempty"`
	// sender hex address from the owner of the given ERC20 tokens
	Sender string `protobuf:"bytes,4,opt,name=sender,proto3" json:"sender,omitempty"`
}
```

如果以下情况发生，消息的无状态验证将失败：

- 合约地址无效
- 金额不是正数
- 接收者的 bech32 地址无效
- 发送者的十六进制地址无效

### `ToggleTokenConversionProposal`

一种用于切换令牌对内部转换的 gov 内容类型。

```go
type ToggleTokenConversionProposal struct {
	// title of the proposal
	Title string `protobuf:"bytes,1,opt,name=title,proto3" json:"title,omitempty"`
	// proposal description
	Description string `protobuf:"bytes,2,opt,name=description,proto3" json:"description,omitempty"`
	// token identifier can be either the hex contract address of the ERC20 or the
	// Cosmos base denomination
	Token string `protobuf:"bytes,3,opt,name=token,proto3" json:"token,omitempty"`
}
```


## Hooks

erc20 模块实现了来自 EVM 的交易钩子，以触发令牌对的转换。

### EVM 钩子

EVM 钩子允许用户通过向模块账户地址发送以太坊 tx 转账来将 ERC20 转换为 Cosmos Coin。
这使得通过 Metamask 和启用了 EVM 的钱包对已通过本机 Cosmos coin 或 ERC20 代币注册的令牌进行本机转换成为可能。
请注意，为了防止恶意合约行为，此处无法执行发送方和接收方的额外币/代币余额检查
（与 [`ConvertERC20` msg](#state-transitions) 中执行的操作相同），
因为事务之前的余额在钩子中不可用。

#### 注册的 Coin：ERC20 到 Coin

1. 用户将 ERC20 代币转账到 `ModuleAccount` 地址以进行托管
2. 通过查看[以太坊事件日志](https://medium.com/mycrypto/understanding-event-logs-on-the-ethereum-blockchain-f4ae7ba50378#:~:text=A%20log%20record%20can%20be,or%20a%20change%20of%20ownership.&text=Each%20log%20record%20consists%20of,going%20on%20in%20an%20event)，
   检查从发送者转账的 ERC20 代币是本机 ERC20 还是本机 Cosmos Coin
3. 如果代币合约地址对应于本机 Cosmos Coin 的 ERC20 表示形式
    1. 从 `ModuleAccount` 调用 `burn()` ERC20 方法。
       请注意，这与 1.2 相同，但由于代币已经在 `ModuleAccount` 余额中，
       我们从模块地址燃烧代币，而不是调用 `burnFrom()`。
       还请注意，我们不需要铸造，
       因为 [1.1 coin to erc20](#state-transitions) 将代币托管
    2. 将 Cosmos Coin 转账到发送者十六进制地址的 bech32 账户地址

#### 注册的ERC20：ERC20转Coin

1. 用户将代币转移到`ModuleAccount`以托管它们
2. 检查转移的ERC20代币是原生的ERC20还是原生的cosmos币
3. 如果代币合约地址是原生的ERC20代币
    1. 铸造Cosmos币
    2. 将Cosmos币转移到发送者的bech32账户地址的十六进制形式


## 事件

`x/erc20`模块会发出以下事件：

### 注册Coin提案

| 类型            | 属性键           | 属性值             |
| --------------- | --------------- | ----------------- |
| `register_coin` | `"cosmos_coin"` | `{denom}`         |
| `register_coin` | `"erc20_token"` | `{erc20_address}` |

### 注册ERC20提案

| 类型             | 属性键           | 属性值             |
| ---------------- | --------------- | ----------------- |
| `register_erc20` | `"cosmos_coin"` | `{denom}`         |
| `register_erc20` | `"erc20_token"` | `{erc20_address}` |

### 切换代币转换

| 类型                      | 属性键           | 属性值             |
| ------------------------- | --------------- | ----------------- |
| `toggle_token_conversion` | `"erc20_token"` | `{erc20_address}` |
| `toggle_token_conversion` | `"cosmos_coin"` | `{denom}`         |

### 转换Coin

| 类型           | 属性键           | 属性值                      |
| -------------- | --------------- | ---------------------------- |
| `convert_coin` | `"sender"`      | `{msg.Sender}`               |
| `convert_coin` | `"receiver"`    | `{msg.Receiver}`             |
| `convert_coin` | `"amount"`      | `{msg.Coin.Amount.String()}` |
| `convert_coin` | `"cosmos_coin"` | `{denom}`                    |
| `convert_coin` | `"erc20_token"` | `{erc20_address}`            |

### 转换ERC20

| 类型            | 属性键           | 属性值                |
| --------------- | --------------- | ----------------------- |
| `convert_erc20` | `"sender"`      | `{msg.Sender}`          |
| `convert_erc20` | `"receiver"`    | `{msg.Receiver}`        |
| `convert_erc20` | `"amount"`      | `{msg.Amount.String()}` |
| `convert_erc20` | `"cosmos_coin"` | `{denom}`               |
| `convert_erc20` | `"erc20_token"` | `{msg.ContractAddress}` |

## 参数

erc20 模块包含以下参数：

| 键                     | 类型          | 默认值                 |
| ----------------------- | ------------- | ----------------------------- |
| `EnableErc20`    | bool          | `true`                        |
| `EnableEVMHook`         | bool          | `true`                        |

### 启用 ERC20

`EnableErc20` 参数用于切换模块中的所有状态转换。
当禁用该参数时，将阻止所有代币对注册和转换功能。

### 启用 EVM Hook

`EnableEVMHook` 参数启用 EVM Hook，通过将代币通过 `MsgEthereumTx` 转移到 `ModuleAddress` 以太坊地址，将 ERC20 代币转换为 Cosmos Coin。

## 客户端

### CLI

下面是使用 `x/erc20` 模块添加的 `evmosd` 命令列表。
您可以使用 `evmosd -h` 命令获取完整列表。
CLI 命令的示例：

```bash
evmosd query erc20 params
```

#### 查询

| 命令         | 子命令    | 描述                    |
| --------------- | ------------- | ------------------------------ |
| `query` `erc20` | `params`      | 获取 erc20 参数               |
| `query` `erc20` | `token-pair`  | 获取已注册的代币对      |
| `query` `erc20` | `token-pairs` | 获取所有已注册的代币对 |

#### 交易

| 命令      | 子命令      | 描述                    |
| ------------ | --------------- | ------------------------------ |
| `tx` `erc20` | `convert-coin`  | 将 Cosmos Coin 转换为 ERC20 |
| `tx` `erc20` | `convert-erc20` | 将 ERC20 转换为 Cosmos Coin |

#### 提案

`tx gov submit-legacy-proposal` 命令允许用户使用治理模块的 CLI 创建提案：

**`register-coin`**

允许用户提交 `RegisterCoinProposal`。
提交一个提案，将 Cosmos coin 注册到 erc20，并附带初始存款。
通过 JSON 文件提供提案详细信息。

```bash
evmosd tx gov submit-legacy-proposal register-coin METADATA_FILE [flags]
```

METADATA_FILE 包含以下内容（示例）：

```json
{
  "metadata": [
    {
			"description": "The native staking and governance token of the Osmosis chain",
			"denom_units": [
				{
						"denom": "ibc/<HASH>",
						"exponent": 0,
						"aliases": ["ibcuosmo"]
				},
				{
						"denom": "OSMO",
						"exponent": 6
				}
			],
			"base": "ibc/<HASH>",
			"display": "OSMO",
			"name": "Osmo",
			"symbol": "OSMO"
		}
	]
}
```

**`register-erc20`**

允许用户提交 `RegisterERC20Proposal`。
提交一个注册 ERC20 代币的提案，同时附带初始存款。
要在一个提案中注册多个代币，请依次传递它们，例如 `register-erc20 <contract-address1> <contract-address2>`。

```bash
evmosd tx gov submit-legacy-proposal register-erc20 ERC20_ADDRESS... [flags]
```

**`toggle-token-conversion`**

允许用户提交 `ToggleTokenConversionProposal`。

```bash
evmosd tx gov submit-legacy-proposal toggle-token-conversion TOKEN [flags]
```

**`param-change`**

允许用户提交 `ParameterChangeProposal`。

```bash
evmosd tx gov submit-legacy-proposal param-change PROPOSAL_FILE [flags]
```

### gRPC

#### 查询

| 动词   | 方法                             | 描述                       |
| ------ | -------------------------------- | ------------------------- |
| `gRPC` | `evmos.erc20.v1.Query/Params`     | 获取 ERC20 参数            |
| `gRPC` | `evmos.erc20.v1.Query/TokenPair`  | 获取已注册的代币对         |
| `gRPC` | `evmos.erc20.v1.Query/TokenPairs` | 获取所有已注册的代币对     |
| `GET`  | `/evmos/erc20/v1/params`          | 获取 ERC20 参数            |
| `GET`  | `/evmos/erc20/v1/token_pair`      | 获取已注册的代币对         |
| `GET`  | `/evmos/erc20/v1/token_pairs`     | 获取所有已注册的代币对     |

#### 交易

| 动词   | 方法                              | 描述                       |
| ------ | --------------------------------- | ------------------------- |
| `gRPC` | `evmos.erc20.v1.Msg/ConvertCoin`   | 将 Cosmos Coin 转换为 ERC20 |
| `gRPC` | `evmos.erc20.v1.Msg/ConvertERC20`  | 将 ERC20 转换为 Cosmos Coin |
| `GET`  | `/evmos/erc20/v1/tx/convert_coin`  | 将 Cosmos Coin 转换为 ERC20 |
| `GET`  | `/evmos/erc20/v1/tx/convert_erc20` | 将 ERC20 转换为 Cosmos Coin |


# `erc20`

:::tip
**Note:** Working on a governance proposal related to the ERC-20 Module?
Make sure to look at [Evmos Governance](https://academy.evmos.org/articles/advanced/governance/),
and specifically the [best practices](https://academy.evmos.org/articles/advanced/governance/best-practices).
:::

## Abstract

This document specifies the internal `x/erc20` module of the Evmos Hub.

The `x/erc20` module enables the Evmos Hub to support a trustless, on-chain bidirectional internal conversion of tokens
between Evmos' EVM and Cosmos runtimes, specifically the `x/evm` and `x/bank` modules.
This allows token holders on Evmos to instantaneously convert their native Cosmos `sdk.Coins`
(in this document referred to as "Coin(s)") to ERC-20 (aka "Token(s)") and vice versa,
while retaining fungibility with the original asset on the issuing environment/runtime (EVM or Cosmos)
and preserving ownership of the ERC-20 contract.

This conversion functionality is fully governed by native EVMOS token holders
who manage the canonical `TokenPair` registrations (ie, ERC20 ←→ Coin mappings).
This governance functionality is implemented using the Cosmos-SDK `gov` module
with custom proposal types for registering and updating the canonical mappings respectively.

Why is this important? Cosmos and the EVM are two runtimes that are not compatible by default.
The native Cosmos Coins cannot be used in applications that require the ERC-20 standard.
Cosmos coins are held on the `x/bank` module (with access to module methods like querying the supply or balances)
and ERC-20 Tokens live on smart contracts.
This problem is similar to [wETH](https://coinmarketcap.com/alexandria/article/what-is-wrapped-ethereum-weth),
with the difference, that it not only applies to gas tokens (like EVMOS),
but to all Cosmos Coins (IBC vouchers, staking and gov coins, etc.) as well.

With the `x/erc20` users on Evmos can

- use existing native cosmos assets (like OSMO or ATOM) on EVM-based chains, e.g.
for Trading IBC tokens on DeFi protocols, buying NFT, etc.
- transfer existing tokens on Ethereum and other EVM-based chains to Evmos
to take advantage of application-specific chains in the Cosmos ecosystem
- build new applications that are based on ERC-20 smart contracts and have access to the Cosmos ecosystem.

## Contents

1. **[Concepts](#concepts)**
2. **[State](#state)**
3. **[State Transitions](#state-transitions)**
4. **[Transactions](#transactions)**
5. **[Hooks](#hooks)**
6. **[Events](#events)**
7. **[Parameters](#parameters)**
8. **[Clients](#clients)**

## Concepts

### Token Pair

The `x/erc20` module maintains a canonical one-to-one mapping of native Cosmos Coin denomination
to ERC20 Token contract addresses (i.e `sdk.Coin` ←→ ERC20), called `TokenPair`.
The conversion of the ERC20 tokens ←→ Coin of a given pair can be enabled or disabled via governance.

### Token Pair Registration

Users can register a new token pair proposal through the governance module
and initiate a vote to include the token pair in the module.
Depending on which exists first, the coin or the token,
you can either register a Cosmos Coin or a ERC20 Token to create a token pair.
One proposal can inculde several token pairs.

When the proposal passes, the erc20 module registers the Cosmos Coin and ERC20 Token mapping on the application's store.

#### Registration of a Cosmos Coin

A native Cosmos Coin corresponds to an `sdk.Coin` that is native to the bank module.
It can be either the native staking/gas denomination (e.g. EVMOS, ATOM, etc)
or an IBC fungible token voucher (i.e. with denom format of `ibc/{hash}`).

When a proposal is initiated for an existing native Cosmos Coin,
the erc20 module will deploy a factory ERC20 contract,
representing the ERC20 token for the token pair,
giving the module ownership of that contract.

#### Registration of an ERC20 token

A proposal for an existing (i.e already deployed) ERC20 contract can be initiated too.
In this case, the ERC20 maintains the original owner of the contract
and uses an escrow & mint / burn & unescrow mechanism similar to the one defined by the
[ICS20 - Fungible Token Transfer](https://github.com/cosmos/ibc/blob/master/spec/app/ics-020-fungible-token-transfer)
specification.
The token pair is composed of the original ERC20 token and a corresponding native Cosmos coin denomination.

#### Token details and metadata

Coin metadata is derived from the ERC20 token details (name, symbol, decimals) and vice versa.
A special case is also described below that for the ERC20 representation of IBC fungible token (ICS20) vouchers.

#### Coin Metadata to ERC20 details

During the registration of a Cosmos Coin the following bank `Metadata` is used to deploy a ERC20 contract:

- **Name**
- **Symbol**
- **Decimals**

The native Cosmos Coin contains a more extensive metadata than the ERC20
and includes all necessary details for the conversion into a ERC20 Token,
which requires no additional population of data.

#### IBC voucher Metadata to ERC20 details

IBC vouchers should comply to the following standard:

- **Name**: `{NAME} channel-{channel}`
- **Symbol**:  `ibc{NAME}-{channel}`
- **Decimals**:  derived from bank `Metadata`

#### ERC20 details to Coin Metadata

During the Registration of an ERC20 Token the Coin metadata is derived from the ERC20 metadata and the bank metadata:

- **Description**: `Cosmos coin token representation of {contractAddress}`
- **DenomUnits**:
    - Coin: `0`
    - ERC20: `{uint32(erc20Data.Decimals)}`
- **Base**: `{"erc20/%s", address}`
- **Display**: `{erc20Data.Name}`
- **Name**: `{types.CreateDenom(strContract)}`
- **Symbol:** `{erc20Data.Symbol}`

### Token Pair Modifiers

A valid token pair can be modified through several governance proposals.
The internal conversion of a token pair can be toggled with `ToggleTokenConversionProposal`,
so that the conversions between the token pair's tokens can be enabled or disabled.

### Token Conversion

Once a token pair proposal passes, the module allows for the conversion of that token pair.
Holders of native Cosmos coins and IBC vouchers on the Evmos chain can convert their Coin into ERC20 Tokens,
which can then be used in Evmos EVM, by creating a `ConvertCoin` Tx.
Vice versa, the `ConvertERC20` Tx allows holders of ERC20 tokens on the Evmos chain
to convert ERC-20 tokens back to their native Cosmos Coin representation.

Depending on the ownership of the ERC20 contract,
the ERC20 tokens either follow a burn/mint or a transfer/escrow mechanism during conversion.

### Malicious Contracts

The ERC20 standard is an interface that defines a set of method signatures (name, arguments and output)
without defining its methods' internal logic.
Therefore it is possible for developers to deploy contracts that contain hidden malicious behaviour within those methods.
For instance, the ERC20 `transfer` method,
which is responsible for sending an `amount` of tokens to a given `recipient` could include code
to siphon some amount of tokens intended for the recipient into a different predefined account,
which is owned by the malicious contract deployer.

More sophisticated malicious implementations
might also inherit code from customized ERC20 contracts that include malicous behaviour.
For an overview of more extensive examples,
please review the x/erc20 audit, section `IF-EVMOS-06: IERC20 Contracts may execute arbitrary code`.

As the `x/erc20` module allows any arbitrary ERC20 contract to be registered through governance,
it is essential that the proposer or the voters manually verify during voting phase
that the proposed contract uses the default ERC20.sol implementation.

Here are our recommendations for the reviewing process:

- contract solidity code should be verified and accessable (e.g. using an explorer)
- contract should be audited by a reputabele auditor
- inherited contracts need to be verified for correctness


## State

### State Objects

The `x/erc20` module keeps the following objects in state:

| State Object       | Description                                    | Key                         | Value               | Store    |
| ------------------ | ---------------------------------------------- | --------------------------- | ------------------- | --- |
| `TokenPair`        | Token Pair bytecode                            | `[]byte{1} + []byte(id)`    | `[]byte{tokenPair}` | KV    |
| `TokenPairByERC20` | Token Pair id bytecode by erc20 contract bytes | `[]byte{2} + []byte(erc20)` | `[]byte(id)`        | KV    |
| `TokenPairByDenom` | Token Pair id bytecode by denom string         | `[]byte{3} + []byte(denom)` | `[]byte(id)`        | KV    |

#### Token Pair

One-to-one mapping of native Cosmos coin denomination to ERC20 token contract addresses (i.e `sdk.Coin` ←→ ERC20).

```go
type TokenPair struct {
	// address of ERC20 contract token
	Erc20Address string `protobuf:"bytes,1,opt,name=erc20_address,json=erc20Address,proto3" json:"erc20_address,omitempty"`
	// cosmos base denomination to be mapped to
	Denom string `protobuf:"bytes,2,opt,name=denom,proto3" json:"denom,omitempty"`
	// shows token mapping enable status
	Enabled bool `protobuf:"varint,3,opt,name=enabled,proto3" json:"enabled,omitempty"`
	// ERC20 owner address ENUM (0 invalid, 1 ModuleAccount, 2 external address
	ContractOwner Owner `protobuf:"varint,4,opt,name=contract_owner,json=contractOwner,proto3,enum=evmos.erc20.v1.Owner" json:"contract_owner,omitempty"`
}
```

#### Token pair ID

The unique identifier of a `TokenPair` is obtained by obtaining the SHA256 hash of the ERC20 hex contract address
and the Coin denomination using the following function:

```tsx
tokenPairId = sha256(erc20 + "|" + denom)
```

#### Token Origin

The `ConvertCoin` and `ConvertERC20` functionalities use the owner field to check
whether the token being used is a native Coin or a native ERC20.
The field is based on the token registration proposal type (`RegisterCoinProposal` = 1, `RegisterERC20Proposal` = 2).

The `Owner` enumerates the ownership of a ERC20 contract.

```go
type Owner int32

const (
	// OWNER_UNSPECIFIED defines an invalid/undefined owner.
	OWNER_UNSPECIFIED Owner = 0
	// OWNER_MODULE erc20 is owned by the erc20 module account.
	OWNER_MODULE Owner = 1
	// EXTERNAL erc20 is owned by an external account.
	OWNER_EXTERNAL Owner = 2
)
```

The `Owner` can be checked with the following helper functions:

```go
// IsNativeCoin returns true if the owner of the ERC20 contract is the
// erc20 module account
func (tp TokenPair) IsNativeCoin() bool {
	return tp.ContractOwner == OWNER_MODULE
}

// IsNativeERC20 returns true if the owner of the ERC20 contract not the
// erc20 module account
func (tp TokenPair) IsNativeERC20() bool {
	return tp.ContractOwner == OWNER_EXTERNAL
}
```

#### Token Pair by ERC20 and by Denom

`TokenPairByERC20` and `TokenPairByDenom` are additional state objects for querying a token pair id.

### Genesis State

The `x/erc20` module's `GenesisState` defines the state necessary
for initializing the chain from a previous exported height.
It contains the module parameters and the registered token pairs :

```go
// GenesisState defines the module's genesis state.
type GenesisState struct {
	// module parameters
	Params Params `protobuf:"bytes,1,opt,name=params,proto3" json:"params"`
	// registered token pairs
	TokenPairs []TokenPair `protobuf:"bytes,2,rep,name=token_pairs,json=tokenPairs,proto3" json:"token_pairs"`
}
```


## State Transitions

The erc20 modules allows for two types of registration state transitions.
Depending on how token pairs are registered, with `RegisterCoinProposal` or `RegisterERC20Proposal`,
there are four possible conversion state transitions.

### Token Pair Registration

Both the Cosmos coin and the ERC20 token registration allow for registering several token pairs with one proposal.
For simplicity, the following description describes the registration of only one token pair per proposal.

#### 1. Register Coin

A user registers a native Cosmos Coin.
Once the proposal passes (i.e is approved by governance),
the ERC20 module uses a factory pattern to deploy an ERC20 token contract representation of the Cosmos Coin.
Note that the native Evmos coin cannot be registered,
as any coin including "evm" in its denomination cannot be registered.
Instead, the Evmos token can be converted by Nomand's wrapped Evmos (WEVMOS) contract.

1. User submits a `RegisterCoinProposal`
2. Validators of the Evmos Hub vote on the proposal using `MsgVote` and proposal passes
3. If Cosmos coin or IBC voucher exist on the bank module supply,
   create the [ERC20 token contract](https://github.com/evmos/evmos/blob/main/contracts/ERC20MinterBurnerDecimals.sol)
   on the EVM based on the ERC20Mintable
   ([ERC20Mintable by openzeppelin](https://github.com/OpenZeppelin/openzeppelin-contracts/tree/master/contracts/token/ERC20))
   interface
    - Initial supply: 0
    - Token details (Name, Symbol, Decimals, etc) are derived
      from the bank module `Metadata` field on the proposal content.

#### 2. Register ERC20

A user registers a ERC20 token contract that is already deployed on the EVM module.
Once the proposal passes (i.e. is approved by governance),
the ERC20 module creates a Cosmos coin representation of the ERC20 token.

1. User submits a `RegisterERC20Proposal`
2. Validators of the EVMOS chain vote on the proposal using `MsgVote` and proposal passes
3. If ERC-20 contract is deployed on the EVM module, create a bank coin `Metadata` from the ERC20 details.

### Token Pair Conversion

Conversion of a registered `TokenPair` can be done via:

- Cosmos transaction (`ConvertCoin` and `ConvertERC20)`
- Ethereum transaction (i.e sending a `MsgEthereumTx` that leverages the EVM hook)

#### 1. Registered Coin

:::tip
👉 **Context:** A `TokenPair` has been created through a `RegisterCoinProposal` governance proposal.
The proposal created an `ERC20` contract
([ERC20Mintable by openzeppelin](https://github.com/OpenZeppelin/openzeppelin-contracts/tree/master/contracts/token/ERC20))
of the ERC20 token representation of the Coin from the `ModuleAccount`,
assigning it as the `owner` of the contract
and thus granting it the permission to call the `mint()` and `burnFrom()` methods of the ERC20.
:::

##### Invariants

- Only the `ModuleAccount` should have the Minter Role on the ERC20. Otherwise,
  the user could unilaterally mint an infinite supply of the ERC20 token and
  then convert them to the native Coin
- The user and the `ModuleAccount` (owner) should be the only ones that have the
  Burn Role for a Cosmos Coin
- There shouldn't exist any native Cosmos Coin ERC20 Contract (eg Evmos, Atom,
  Osmo ERC20 contracts) that is not owned by the governance
- Token/Coin supply is maintained at all times:
    - Total Coin supply = Coins + Escrowed Coins
    - Total Token supply = Escrowed Coins = Minted Tokens

##### 1.1 Coin to ERC20

1. User submits `ConvertCoin` Tx
2. Check if conversion is allowed for the pair, sender and recipient
    - global parameter is enabled
    - token pair is enabled
    - sender tokens are not vesting (checked in the bank module)
    - recipient address is not blacklisted
3. If Coin is a native Cosmos Coin and Token Owner is `ModuleAccount`
    1. Escrow Cosmos coin by sending them to the erc20 module account
    2. Call `mint()` ERC20 tokens from the `ModuleAccount` address and send minted tokens to recipient address
4. Check if token balance increased by amount

##### 1.2 ERC20 to Coin

1. User submits a `ConvertERC20` Tx
2. Check if conversion is allowed for the pair, sender and recipient (see [1.1 Coin to ERC20](#11-coin-to-erc20))
3. If token is a ERC20 and Token Owner is `ModuleAccount`
    1. Call `burnCoins()` on ERC20 to burn ERC20 tokens from the user balance
    2. Send Coins (previously escrowed, see [1.1 Coin to ERC20](#11-coin-to-erc20))
       from module to the recipient address.
4. Check if
    - Coin balance increased by amount
    - Token balance decreased by amount

#### 2. Registered ERC20

:::tip
👉 **Context:** A `TokenPair` has been created through a `RegisterERC20Proposal` governance proposal.
The `ModuleAccount` is not the owner of the contract, so it can't mint new tokens or burn on behalf of the user.
The mechanism described below follows the same model as the ICS20 standard,
by using escrow & mint / burn & unescrow logic.
:::

##### Invariants

- ERC20 Token supply on the EVM runtime is maintained at all times:
    - Escrowed ERC20 + Minted Cosmos Coin representation of ERC20 = Burned Cosmos Coin representation of ERC20 +
      Unescrowed ERC20
        - Convert 10 ERC20 → Coin, the total supply increases by 10. Mint on Cosmos side, no changes on EVM
        - Convert 10 Coin → ERC20, the total supply decreases by 10. Burn on Cosmos side , no changes of supply on EVM
    - Total ERC20 token supply = Non Escrowed Tokens + Escrowed Tokens (on Module account address)
    - Total Coin supply for the native ERC20 = Escrowed ERC20 Tokens on module account  (i.e balance) = Minted Coins

##### 2.1 ERC20 to Coin

1. User submits a `ConvertERC20` Tx
2. Check if conversion is allowed for the pair, sender and recipient (See [1.1 Coin to ERC20](#11-coin-to-erc20))
3. If token is a ERC20 and Token Owner is **not** `ModuleAccount`
    1. Escrow ERC20 token by sending them to the erc20 module account
    2. Mint Cosmos coins of the corresponding token pair denomination and send coins to the recipient address
4. Check if
    - Coin balance increased by amount
    - Token balance decreased by amount
5. Fail if unexpected `Approval` event found in logs to prevent malicious contract behaviour

##### 2.2 Coin to ERC20

1. User submits `ConvertCoin` Tx
2. Check if conversion is allowed for the pair, sender and recipient
3. If coin is a native Cosmos coin and Token Owner is **not** `ModuleAccount`
    1. Escrow Cosmos Coins by sending them to the erc20 module account
    2. Unlock escrowed ERC20 from the module address by sending it to the recipient
    3. Burn escrowed Cosmos coins
4. Check if token balance increased by amount
5. Fail if unexpected `Approval` event found in logs to prevent malicious contract behaviour


## Transactions

This section defines the `sdk.Msg` concrete types that result in the state transitions defined on the previous section.

### `RegisterCoinProposal`

A gov `Content` type to register a token pair from a Cosmos Coin.
Governance users vote on this proposal
and it automatically executes the custom handler for `RegisterCoinProposal` when the vote passes.

```go
type RegisterCoinProposal struct {
	// title of the proposal
	Title string `protobuf:"bytes,1,opt,name=title,proto3" json:"title,omitempty"`
	// proposal description
	Description string `protobuf:"bytes,2,opt,name=description,proto3" json:"description,omitempty"`
	// metadata slice of the native Cosmos coins
	Metadata []types.Metadata `protobuf:"bytes,3,rep,name=metadata,proto3" json:"metadata"`
}
```

The proposal content stateless validation fails if:

- Title is invalid (length or char)
- Description is invalid (length or char)
- Metadata is invalid
    - Name and Symbol are not blank
    - Base and Display denominations are valid coin denominations
    - Base and Display denominations are present in the DenomUnit slice
    - Base denomination has exponent 0
    - Denomination units are sorted in ascending order
    - Denomination units not duplicated

### `RegisterERC20Proposal`

A gov `Content` type to register a token pair from an ERC20 Token.
Governance users vote on this proposal
and it automatically executes the custom handler for `RegisterERC20Proposal` when the vote passes.

```go
type RegisterERC20Proposal struct {
	// title of the proposal
	Title string `protobuf:"bytes,1,opt,name=title,proto3" json:"title,omitempty"`
	// proposal description
	Description string `protobuf:"bytes,2,opt,name=description,proto3" json:"description,omitempty"`
	// contract addresses of ERC20 tokens
	Erc20Addresses []string `protobuf:"bytes,3,rep,name=erc20addresses,proto3" json:"erc20addresses,omitempty"`
}
```

The proposal Content stateless validation fails if:

- Title is invalid (length or char)
- Description is invalid (length or char)
- ERC20Addresses is invalid

### `MsgConvertCoin`

A user broadcasts a `MsgConvertCoin` message to convert a Cosmos Coin to a ERC20 token.

```go
type MsgConvertCoin struct {
	// Cosmos coin which denomination is registered on erc20 bridge.
	// The coin amount defines the total ERC20 tokens to convert.
	Coin types.Coin `protobuf:"bytes,1,opt,name=coin,proto3" json:"coin"`
	// recipient hex address to receive ERC20 token
	Receiver string `protobuf:"bytes,2,opt,name=receiver,proto3" json:"receiver,omitempty"`
	// cosmos bech32 address from the owner of the given ERC20 tokens
	Sender string `protobuf:"bytes,3,opt,name=sender,proto3" json:"sender,omitempty"`
}
```

Message stateless validation fails if:

- Coin is invalid (invalid denom or non-positive amount)
- Receiver hex address is invalid
- Sender bech32 address is invalid

### `MsgConvertERC20`

A user broadcasts a `MsgConvertERC20` message to convert a ERC20 token to a native Cosmos coin.

```go
type MsgConvertERC20 struct {
	// ERC20 token contract address registered on erc20 bridge
	ContractAddress string `protobuf:"bytes,1,opt,name=contract_address,json=contractAddress,proto3" json:"contract_address,omitempty"`
	// amount of ERC20 tokens to mint
	Amount github_com_cosmos_cosmos_sdk_types.Int `protobuf:"bytes,2,opt,name=amount,proto3,customtype=github.com/cosmos/cosmos-sdk/types.Int" json:"amount"`
	// bech32 address to receive SDK coins.
	Receiver string `protobuf:"bytes,3,opt,name=receiver,proto3" json:"receiver,omitempty"`
	// sender hex address from the owner of the given ERC20 tokens
	Sender string `protobuf:"bytes,4,opt,name=sender,proto3" json:"sender,omitempty"`
}
```

Message stateless validation fails if:

- Contract address is invalid
- Amount is not positive
- Receiver bech32 address is invalid
- Sender hex address is invalid

### `ToggleTokenConversionProposal`

A gov Content type to toggle the internal conversion of a token pair.

```go
type ToggleTokenConversionProposal struct {
	// title of the proposal
	Title string `protobuf:"bytes,1,opt,name=title,proto3" json:"title,omitempty"`
	// proposal description
	Description string `protobuf:"bytes,2,opt,name=description,proto3" json:"description,omitempty"`
	// token identifier can be either the hex contract address of the ERC20 or the
	// Cosmos base denomination
	Token string `protobuf:"bytes,3,opt,name=token,proto3" json:"token,omitempty"`
}
```


## Hooks

The erc20 module implements transaction hooks from the EVM in order to trigger token pair conversion.

### EVM Hooks

The EVM hooks allows users to convert ERC20s to Cosmos Coins
by sending an Ethereum tx transfer to the module account address.
This enables native conversion of tokens via Metamask and EVM-enabled wallets for both token pairs
that have been registered through a native Cosmos coin or an ERC20 token.
Note that additional coin/token balance checks for sender and receiver to prevent malicious contract behaviour
(as performed in the [`ConvertERC20` msg](#state-transitions)) cannot be done here,
as the balance prior to the transaction is not available in the hook.

#### Registered Coin: ERC20 to Coin

1. User transfers ERC20 tokens to the `ModuleAccount` address to escrow them
2. Check if the ERC20 Token that was transferred from the sender is a native ERC20
   or a native Cosmos Coin by looking at the
   [Ethereum event logs](https://medium.com/mycrypto/understanding-event-logs-on-the-ethereum-blockchain-f4ae7ba50378#:~:text=A%20log%20record%20can%20be,or%20a%20change%20of%20ownership.&text=Each%20log%20record%20consists%20of,going%20on%20in%20an%20event)
3. If the token contract address corresponds to the ERC20 representation of a native Cosmos Coin
    1. Call `burn()` ERC20 method from the  `ModuleAccount`.
       Note that this is the same as 1.2, but since the tokens are already on the ModuleAccount balance,
       we burn the tokens from the module address instead of calling `burnFrom()`.
       Also note that we don't need to mint
       because [1.1 coin to erc20](#state-transitions) escrows the coin
    2. Transfer Cosmos Coin to the bech32 account address of the sender hex address

#### Registered ERC20: ERC20 to Coin

1. User transfers coins to the`ModuleAccount` to escrow them
2. Check if the ERC20 Token that was transferred is a native ERC20 or a native cosmos coin
3. If the token contract address is a native ERC20 token
    1. Mint Cosmos Coin
    2. Transfer Cosmos Coin to the bech32 account address of the sender hex


## Events

The `x/erc20` module emits the following events:

### Register Coin Proposal

| Type            | Attribute Key   | Attribute Value   |
| --------------- | --------------- | ----------------- |
| `register_coin` | `"cosmos_coin"` | `{denom}`         |
| `register_coin` | `"erc20_token"` | `{erc20_address}` |

### Register ERC20 Proposal

| Type             | Attribute Key   | Attribute Value   |
| ---------------- | --------------- | ----------------- |
| `register_erc20` | `"cosmos_coin"` | `{denom}`         |
| `register_erc20` | `"erc20_token"` | `{erc20_address}` |

### Toggle Token Conversion

| Type                      | Attribute Key   | Attribute Value   |
| ------------------------- | --------------- | ----------------- |
| `toggle_token_conversion` | `"erc20_token"` | `{erc20_address}` |
| `toggle_token_conversion` | `"cosmos_coin"` | `{denom}`         |

### Convert Coin

| Type           | Attribute Key   | Attribute Value              |
| -------------- | --------------- | ---------------------------- |
| `convert_coin` | `"sender"`      | `{msg.Sender}`               |
| `convert_coin` | `"receiver"`    | `{msg.Receiver}`             |
| `convert_coin` | `"amount"`      | `{msg.Coin.Amount.String()}` |
| `convert_coin` | `"cosmos_coin"` | `{denom}`                    |
| `convert_coin` | `"erc20_token"` | `{erc20_address}`            |

### Convert ERC20

| Type            | Attribute Key   | Attribute Value         |
| --------------- | --------------- | ----------------------- |
| `convert_erc20` | `"sender"`      | `{msg.Sender}`          |
| `convert_erc20` | `"receiver"`    | `{msg.Receiver}`        |
| `convert_erc20` | `"amount"`      | `{msg.Amount.String()}` |
| `convert_erc20` | `"cosmos_coin"` | `{denom}`               |
| `convert_erc20` | `"erc20_token"` | `{msg.ContractAddress}` |


## Parameters

The erc20 module contains the following parameters:

| Key                     | Type          | Default Value                 |
| ----------------------- | ------------- | ----------------------------- |
| `EnableErc20`    | bool          | `true`                        |
| `EnableEVMHook`         | bool          | `true`                        |

### Enable ERC20

The `EnableErc20` parameter toggles all state transitions in the module.
When the parameter is disabled, it will prevent all token pair registration and conversion functionality.

### Enable EVM Hook

The `EnableEVMHook` parameter enables the EVM hook to convert an ERC20 token
to a Cosmos Coin by transferring the Tokens through a `MsgEthereumTx`  to the `ModuleAddress` Ethereum address.


## Clients

### CLI

Find below a list of  `evmosd` commands added with the  `x/erc20` module.
You can obtain the full list by using the `evmosd -h` command.
A CLI command can look like this:

```bash
evmosd query erc20 params
```

#### Queries

| Command         | Subcommand    | Description                    |
| --------------- | ------------- | ------------------------------ |
| `query` `erc20` | `params`      | Get erc20 params               |
| `query` `erc20` | `token-pair`  | Get registered token pair      |
| `query` `erc20` | `token-pairs` | Get all registered token pairs |

#### Transactions

| Command      | Subcommand      | Description                    |
| ------------ | --------------- | ------------------------------ |
| `tx` `erc20` | `convert-coin`  | Convert a Cosmos Coin to ERC20 |
| `tx` `erc20` | `convert-erc20` | Convert a ERC20 to Cosmos Coin |

#### Proposals

The `tx gov submit-legacy-proposal` commands allow users to query create a proposal using the governance module CLI:

**`register-coin`**

Allows users to submit a `RegisterCoinProposal`.
Submit a proposal to register a Cosmos coin to the erc20 along with an initial deposit.
Upon passing, the proposal details must be supplied via a JSON file.

```bash
evmosd tx gov submit-legacy-proposal register-coin METADATA_FILE [flags]
```

Where METADATA_FILE contains (example):

```json
{
  "metadata": [
    {
			"description": "The native staking and governance token of the Osmosis chain",
			"denom_units": [
				{
						"denom": "ibc/<HASH>",
						"exponent": 0,
						"aliases": ["ibcuosmo"]
				},
				{
						"denom": "OSMO",
						"exponent": 6
				}
			],
			"base": "ibc/<HASH>",
			"display": "OSMO",
			"name": "Osmo",
			"symbol": "OSMO"
		}
	]
}
```

**`register-erc20`**

Allows users to submit a `RegisterERC20Proposal`.
Submit a proposal to register ERC20 tokens along with an initial deposit.
To register multiple tokens in one proposal pass them after each other e.g.
`register-erc20 <contract-address1> <contract-address2>`.

```bash
evmosd tx gov submit-legacy-proposal register-erc20 ERC20_ADDRESS... [flags]
```

**`toggle-token-conversion`**

Allows users to submit a `ToggleTokenConversionProposal`.

```bash
evmosd tx gov submit-legacy-proposal toggle-token-conversion TOKEN [flags]
```

**`param-change`**

Allows users to submit a `ParameterChangeProposal``.

```bash
evmosd tx gov submit-legacy-proposal param-change PROPOSAL_FILE [flags]
```

### gRPC

#### Queries

| Verb   | Method                            | Description                    |
| ------ | --------------------------------- | ------------------------------ |
| `gRPC` | `evmos.erc20.v1.Query/Params`     | Get erc20 params               |
| `gRPC` | `evmos.erc20.v1.Query/TokenPair`  | Get registered token pair      |
| `gRPC` | `evmos.erc20.v1.Query/TokenPairs` | Get all registered token pairs |
| `GET`  | `/evmos/erc20/v1/params`          | Get erc20 params               |
| `GET`  | `/evmos/erc20/v1/token_pair`      | Get registered token pair      |
| `GET`  | `/evmos/erc20/v1/token_pairs`     | Get all registered token pairs |

#### Transactions

| Verb   | Method                             | Description                    |
| ------ | ---------------------------------- | ------------------------------ |
| `gRPC` | `evmos.erc20.v1.Msg/ConvertCoin`   | Convert a Cosmos Coin to ERC20 |
| `gRPC` | `evmos.erc20.v1.Msg/ConvertERC20`  | Convert a ERC20 to Cosmos Coin |
| `GET`  | `/evmos/erc20/v1/tx/convert_coin`  | Convert a Cosmos Coin to ERC20 |
| `GET`  | `/evmos/erc20/v1/tx/convert_erc20` | Convert a ERC20 to Cosmos Coin |
