---
sidebar_position: 6
---

# 验证者常见问题

查看有关在Evmos上运行验证者的常见问题。

## 一般概念

<details>

<summary><b>什么是验证者？</b></summary>

Evmos由[Tendermint](https://docs.tendermint.com/v0.34/introduction/what-is-tendermint.html) Core提供支持，
它依赖一组验证者来保护网络安全。验证者运行完整节点并参与共识，通过广播包含由其私钥签名的加密签名的投票。验证者在区块链中提交新的区块，并获得收入作为回报。他们还通过对治理提案进行投票参与协议内的财政治理。验证者的投票影响力根据其总质押进行加权。

</details>

<details>

<summary><b>什么是“质押”？</b></summary>

Evmos是一个公共的权益证明（PoS）区块链，这意味着验证者的权重由作为抵押品的质押代币（EVMOS）的数量决定。这些质押代币可以直接由验证者质押，也可以由EVMOS持有者委托给他们。

系统中的任何用户都可以通过发送`create-validator`交易来声明成为验证者的意图。从那时起，他们成为验证者。

验证者的权重（即总质押或投票权）决定了它是否是活跃验证者，以及此节点需要提出区块的频率和获得的收入量。最初，只有权重最高的150个验证者将成为活跃验证者。如果验证者双重签名或经常离线，他们的质押代币（包括用户委托的EVMOS）可能会被协议“惩罚性削减”，以惩罚疏忽和不当行为。

</details>

<details>

<summary><b>什么是完整节点？</b></summary>

完整节点是一个完全验证区块链交易和区块的程序。它与仅处理区块头和一小部分交易的轻客户端节点不同。运行完整节点需要比轻客户端更多的资源，但这是成为验证者所必需的。实际上，运行完整节点只意味着运行一个未被破坏且更新的软件版本，具有低网络延迟且无停机时间。

当然，任何用户都可以运行全节点，即使他们不打算成为验证者。

</details>

<details>

<summary><b>什么是委托人？</b></summary>

委托人是持有 EVMOS 的用户，他们不能或不想自己运行验证者操作。用户可以将 EVMOS 委托给验证者，并以此获得其收入的一部分（有关收入分配的详细信息，请参见下面的“什么是质押的激励？”和“验证者的佣金是什么？”部分）。

由于他们与验证者共享收入，委托人也承担责任。如果验证者行为不端，每个委托人的质押份额将被部分削减。这就是为什么委托人在委托之前应该对验证者进行尽职调查，并通过将他们的质押份额分散到多个验证者来实现多样化。

委托人在系统中扮演着关键的角色，因为他们负责选择验证者。请注意，成为委托人并不是一种被动的角色。委托人有义务保持警惕，并积极监控他们的验证者的行动，如果验证者未能负责任地行事，委托人应该及时切换。

</details>

## 成为验证者

<details>

<summary><b>如何成为验证者？</b></summary>

网络中的任何参与者都可以通过创建验证者并注册其验证者配置文件来表明他们成为验证者的意图。为此，候选人广播一个 `create-validator` 交易，在其中必须提交以下信息：

- **验证者的 PubKey**：验证者操作员可以为验证和持有流动资金使用不同的账户。提交的 PubKey 必须与验证者打算用于签署 *prevotes* 和 *precommits* 的私钥相关联。
- **验证者的地址**：`evmosvaloper1-` 地址。这是用于公开标识您的验证者的地址。与此地址相关联的私钥用于绑定、解绑和领取奖励。
- **验证者的名称**（也称为 **moniker**）
- **验证者的网站**（可选）
- **验证者的描述**（可选）
- **初始佣金率**：对于区块奖励和收费给委托人的区块提供的佣金率。
- **最大佣金率**：此验证者允许收取的最高佣金率。
- **佣金变化率**：验证者佣金的最大每日增加率。
- **最小自我绑定金额**：验证者需要始终绑定的最低金额的 EVMOS。如果验证者自我绑定的质押份额低于此限制，其整个质押池将被解绑。
- **初始自我绑定金额**：验证者希望自我绑定的 EVMOS 的初始金额。

```bash
evmosd tx staking create-validator
--pubkey evmosvalconspub1zcjduepqs5s0vddx5m65h5ntjzwd0x8g3245rgrytpds4ds7vdtlwx06mcesmnkzly
--amount "2aevmos"
--from tmp
--commission-rate="0.20"
--commission-max-rate="1.00"
--commission-max-change-rate="0.01"
--min-self-delegation "1"
--moniker "validator"
--chain-id "evmos_9000-4"
--gas auto
--node tcp://127.0.0.1:26647
```

:::danger
🚨 **危险**: <u>绝对不要</u>使用[`test`](./../../protocol/concepts/keyring#testing)密钥后端创建您的主网验证人密钥。这样做可能导致通过`eth_sendTransaction` JSON-RPC端点远程访问您的资金，从而导致资金损失。

参考: [安全公告：配置不安全的geth可能使资金远程访问](https://blog.ethereum.org/2015/08/29/security-alert-insecurely-configured-geth-can-make-funds-remotely-accessible/)
:::

一旦创建并注册了验证人，EVMOS持有人可以委托EVMOS给验证人，有效地为其添加质押。验证人的总质押是验证人操作者自质押的EVMOS和外部委托人质押的EVMOS之和。

**只有前150个质押最多的验证人被视为活跃验证人**，成为**质押验证人**。如果某个验证人的总质押低于前150名，该验证人将失去验证人特权（即不会产生奖励），不再作为活跃集合的一部分（即不参与共识），进入**解绑模式**，最终变为**解绑状态**。

</details>

## 验证人密钥和状态

<details>

<summary><b>有哪些不同类型的密钥？</b></summary>

简而言之，有两种类型的密钥：

- **Tendermint密钥**：这是用于签署区块哈希的唯一密钥。它与公钥`evmosvalconspub`相关联。
    - 在使用`evmosd init`创建节点时生成。
    - 使用`evmosd tendermint show-validator`获取此值，例如`evmosvalconspub1zcjduc3qcyj09qc03elte23zwshdx92jm6ce88fgc90rtqhjx8v0608qh5ssp0w94c`

- **应用密钥**：这些密钥是从应用程序创建的，用于签署交易。作为验证人，您可能会使用一个密钥来签署与质押相关的交易，另一个密钥来签署与预言机相关的交易。应用密钥与公钥`evmospub-`和地址`evmos-`相关联，两者都是由`evmosd keys add`生成的帐户密钥派生的。
```

:::warning
验证人的操作员密钥直接与应用密钥关联，但仅为此目的使用保留前缀：`evmosvaloper`和`evmosvaloperpub`。
:::

</details>

<details>

<summary><b>验证人可以处于哪些不同的状态？</b></summary>

在使用`create-validator`交易创建验证人之后，它可以处于三种状态之一：

- `bonded`：验证人在活跃集中并参与共识。验证人可以获得奖励，并且可能因为不当行为而被惩罚。
- `unbonding`：验证人不在活跃集中，不参与共识。验证人不获得奖励，但仍可能因为不当行为而被惩罚。这是从`bonded`到`unbonded`的过渡状态。如果验证人在`unbonding`模式下没有发送`rebond`交易，状态转换将需要两周时间才能完成。
- `unbonded`：验证人不在活跃集中，因此不签名区块。未绑定的验证人无法被惩罚，但也不会从其操作中获得任何奖励。仍然可以将 EVMOS 委托给此验证人。从`unbonded`验证人中解除委托是立即生效的。

委托人的状态与其验证人相同。

:::warning
委托不一定是已绑定的。EVMOS 可以被委托并绑定、委托并解绑、委托并未绑定，或者是流动的。
:::

</details>

<details>

<summary><b>什么是“自我绑定”？如何增加我的“自我绑定”？</b></summary>

验证人操作员的“自我绑定”是指委托给自己的 EVMOS 质押金额。您可以通过向您的验证人账户委托更多的 EVMOS 来增加自我绑定。

</details>

<details>

<summary><b>是否有测试网的水龙头？</b></summary>

如果您想要在测试网上获取代币，可以使用[水龙头](https://faucet.evmos.dev/)。

</details>

<details>

<summary><b>成为活跃（已绑定）验证人是否需要最低数量的 EVMOS 质押？</b></summary>

没有最低数量要求。总质押量最高的前150个验证人（其中`总质押量 = 自我绑定的质押量 + 委托人的质押量`）将成为活跃验证人。

</details>

<details>

<summary><b>委托人将如何选择他们的验证者？</b></summary>

委托人可以根据自己的主观标准选择验证者。预计重要的标准包括：

- **自我绑定的EVMOS数量：** 验证者自我绑定到其质押池的EVMOS数量。自我绑定的EVMOS数量越多，验证者在游戏中的利益越大，使其对其行为更加负责。

- **委托的EVMOS数量：** 委托给验证者的EVMOS总数。高额的质押表明社区信任该验证者，但也意味着该验证者成为黑客攻击的更大目标。随着委托的EVMOS数量增加，验证者预计会变得越来越不受吸引。更大的验证者也会增加网络的集中化程度。

- **佣金率：** 在收入分配给委托人之前，验证者收取的佣金。

- **历史记录：** 委托人可能会查看他们计划委托的验证者的历史记录。这包括资历、对提案的过去投票、历史平均正常运行时间以及节点被攻击的频率。

除了这些标准外，验证者还可以通过信号传递网站地址来完善他们的简历。验证者需要通过某种方式建立声誉以吸引委托人。例如，验证者最好让第三方对其设置进行审计。但请注意，Evmos团队不会批准或进行任何审计。

</details>

## 职责

<details>

<summary><b>验证者需要公开身份吗？</b></summary>

不需要。每个委托人将根据自己的标准评估验证者。验证者可以（并且建议）在自我提名时注册一个网站地址，以便根据他们自己的意愿宣传他们的操作。一些委托人可能更喜欢清楚显示验证者团队和他们的简历的网站，而其他人可能更喜欢具有良好历史记录的匿名验证者。很可能会同时存在已识别和匿名的验证者。

</details>

<details>

<summary><b>验证人的责任是什么？</b></summary>

验证人有三个主要责任：

- **能够始终运行正确版本的软件：** 验证人需要确保他们的服务器始终在线，并且他们的私钥没有被泄露。

- **对社区资金的正确使用提供监督和反馈：** Evmos协议包括一个治理系统，用于提出和促进其货币的采纳。验证人应该对预算执行者进行监督，以确保透明度和有效使用资金。

此外，验证人应该成为社区的积极成员。他们应该始终了解生态系统的当前状态，以便能够轻松适应任何变化。

</details>

<details>

<summary><b>质押意味着什么？</b></summary>

将EVMOS质押可以被视为对验证活动的安全押金。当验证人或委托人想要取回部分或全部押金时，他们发送一个解绑事务。然后，押金经历一个*两周的解绑期*，在此期间，验证人可能因为解绑过程开始之前的潜在不当行为而被处罚。

验证人以及与之关联的委托人会收到区块奖励、区块奖励和手续费奖励。如果验证人行为不端，其总质押的一部分将被削减（惩罚的严重程度取决于不当行为的类型）。这意味着将EVMOS绑定给该验证人的每个用户都会按比例受到惩罚。因此，委托人有动力委托给他们预计会安全运行的验证人。

</details>
<details>

<summary><b>验证人能够带走委托人的EVMOS吗？</b></summary>

通过委托给验证人，用户委托了质押权力。验证人拥有的质押权力越大，其在共识和流程中的权重就越大。这并不意味着验证人拥有其委托人的EVMOS资产。*绝对不可能让验证人带走其委托人的资金*。

尽管委托的资金不会被验证者窃取，但如果验证者行为不端，委托人仍然要承担责任。在这种情况下，每个委托人的股份将按比例被部分削减。

</details>

<details>

<summary><b>验证者被选择提议下一个区块的频率是多少？它是否随着 EVMOS 的质押数量增加而增加？</b></summary>

被选中挖掘下一个区块的验证者被称为**提议者**，也是共识中的“领导者”。每个提议者都是确定性选择的，并且被选择的频率等于验证者的相对总质押（其中总质押=自质押质押 + 委托人质押）的比例。例如，如果所有验证者的总质押为100个 EVMOS，而某个验证者的总质押为10个 EVMOS，则该验证者将有10% 的概率被选择为提议者。

要了解更多关于 Tendermint BFT 共识中提议者选择过程的信息，请阅读他们的[官方文档](https://docs.tendermint.com/master/spec/consensus/proposer-selection.html)。

</details>

## 激励机制

<details>

<summary><b>质押的激励是什么？</b></summary>

验证者的质押池中的每个成员都可以获得不同类型的收益：

- **区块奖励：**由验证者运行的应用程序的原生代币（例如 Evmos 上的 EVMOS）会通胀产生区块奖励。这些奖励存在的目的是激励 EVMOS 持有者进行质押，因为未质押的 EVMOS 会随着时间的推移而稀释。
- **交易费用：**Evmos 维护一个接受作为费用支付的代币的白名单。初始的费用代币是 `evmos`。

这些总收益根据每个验证者的权重在验证者的质押池中进行分配。然后，在每个验证者的质押池中，收益按照每个委托人的质押份额进行分配。验证者在分配之前会对委托人的收益收取佣金。

</details>

<details>

<summary><b>运行验证者的激励是什么？</b></summary>

验证人相比于委托人可以获得更多的收入，这是因为他们可以收取佣金。

验证人在治理中也扮演着重要的角色。如果委托人没有投票，他们将继承其验证人的投票权。这使得验证人在生态系统中承担了重要责任。

</details>

<details>

<summary><b>什么是验证人的佣金？</b></summary>

验证人池获得的收入将在验证人和其委托人之间进行分配。验证人可以对分配给其委托人的收入部分收取佣金。这个佣金以百分比的形式设定。
每个验证人都可以自由设定其初始佣金、每日最大佣金变动率和最大佣金。Evmos会强制执行每个验证人设定的参数。这些参数只能在初始声明候选人时定义，并且在声明后可能会进一步受到限制。

</details>

<details>

<summary><b>区块奖励如何分配？</b></summary>

区块奖励将按照验证人的总质押量（投票权重）与其它验证人的比例进行分配。这意味着即使每个验证人都会获得一定数量的EVMOS作为奖励，所有验证人的权重仍然保持相等。

让我们举个例子，假设有10个具有相等质押能力和1%佣金率的验证人。假设每个区块的奖励是1000个EVMOS，每个验证人自己质押的EVMOS占比为20%。这些代币不会直接给提案者，而是均匀分配给验证人。现在每个验证人的池子里有100个EVMOS。这100个EVMOS将根据每个参与者的质押比例进行分配：

- 佣金：`100*80%*1% = 0.8个EVMOS`
- 验证人获得：`100*20% + 佣金 = 20.8个EVMOS`
- 所有委托人获得：`100*80% - 佣金 = 79.2个EVMOS`

然后，每个委托人可以按照其在验证人质押池中的质押比例来领取其所得的79.2个EVMOS的份额。请注意，验证人的佣金不适用于区块奖励。同时，区块奖励（以EVMOS支付）也是按照相同的机制进行分配的。

</details>

<details>

<summary><b>费用是如何分配的？</b></summary>

费用的分配方式类似，但区块提议者在包含超过所需的最低预提交数时可以获得额外奖励。

当选择一个验证者来提议下一个区块时，它必须至少包含⅔的前一个区块的预提交，以验证者签名的形式。然而，包含超过⅔的预提交会有额外的激励奖励。奖励是线性的：如果提议者包含⅔的预提交（区块的最低要求），奖励为1%；如果提议者包含100%的预提交，奖励为5%。当然，提议者不应等待太久，否则其他验证者可能会超时并转向下一个提议者。因此，验证者需要在等待时间和获取最多签名之间找到平衡，以及在提议下一个区块的风险之间。这个机制旨在激励非空的区块提议，改善验证者之间的网络连接，并减轻审查制度。

让我们通过一个具体的例子来说明上述概念。在这个例子中，有10个具有相等权益的验证者。每个验证者收取1%的佣金，并拥有20%的自质押EVMOS。现在出现了一个成功的区块，收集了总共1005个EVMOS的费用。假设提议者在其区块中包含了100%的签名，因此获得了完整的5%奖励。

我们需要解这个简单的方程来找到每个验证者的奖励$R$：

$$9R ~ + ~ R ~ + ~ 5\%(R) ~ = ~ 1005 ~ \Leftrightarrow ~ R ~ = ~ 1005 ~/ ~10.05 ~ = ~ 100$$

- 对于提议者验证者：

    - 池子获得 $R ~ + ~ 5\%(R)$：105个EVMOS
    - 佣金：$105 ~ *~ 80\% ~* ~ 1\%$ = 0.84个EVMOS
    - 验证者的奖励：$105 ~ * ~ 20\% ~ + ~ 佣金$ = 21.84个EVMOS
    - 委托人的奖励：$105 ~ * ~ 80\% ~ - ~ 佣金$ = 83.16个EVMOS（每个委托人将能够按照他们的权益比例索取他们的奖励部分）

- 对于其他验证者：

    - 池子获得 $R$：100个EVMOS
    - 佣金：$100 ~ *~ 80\% ~* ~ 1\%$ = 0.8个EVMOS
    - 验证者的奖励：$100 ~ * ~ 20\% ~ + ~ 佣金$ = 20.8个EVMOS
    - 委托人的奖励：$100 ~ * ~ 80\% ~ - ~ 佣金$ = 79.2个EVMOS（每个委托人将能够按照他们的权益比例索取他们的奖励部分）

</details>

<details>

<summary><b>什么是惩罚条件？</b></summary>

如果验证人行为不当，其绑定的质押金额以及委托人的质押金额都将被削减。惩罚的严重程度取决于故障类型。有三种主要故障可能导致验证人及其委托人的资金被削减：

- **双签名：** 如果有人在链A上报告一个验证人在链A和链B上在同一高度签署了两个区块，并且链A和链B有一个共同的祖先，那么该验证人将在链A上被削减。双签名的惩罚是总质押金额的10.00%。

- **停机时间：** 如果一个验证人在最近的90,000个区块中错过了超过50%，他们将被削减0.50%。
- **不可用性：** 如果一个验证人的签名在最近的X个区块中没有被包含，验证人将被削减一个与X成比例的微小金额。如果X超过了某个限制Y，那么验证人将被解绑。

请注意，即使验证人没有故意不当行为，如果其节点崩溃、失去连接、遭受DDoS攻击或其私钥被泄露，仍然可能被削减。

以下是一些关于双签名的社区学习链接，值得一看：

- [从 BlockDaemon 中学到的经验](https://blockdaemon.com/documentation/evmos-post-mortem/)

</details>

<details>

<summary><b>有关如何防止双签的最佳实践指南吗？</b></summary>

有一个很棒的指南。Polkachu 是 Evmos 上的验证者，他们编写了[这个页面](https://github.com/polkachu/validator-guide/blob/main/validator_server_migration_best_practice.md)
来提供帮助。

</details>

<details>

<summary><b>验证者需要自我绑定 EVMOS 吗？</b></summary>

不需要。验证者的总质押金额等于其自我绑定质押金额和委托质押金额之和。
这意味着验证者可以通过吸引更多的委托者来弥补自我绑定质押金额较低的情况。这也是为什么验证者的声誉非常重要。

虽然验证者没有义务自我绑定 EVMOS，但委托者应该希望他们的验证者在质押池中有自我绑定的 EVMOS。换句话说，验证者应该有真金白银的投入。

为了让委托人对验证者的质押数量有一定的保证，验证者可以发出信号，表示最低自质押的 EVMOS 数量。如果验证者的自质押低于预设的限制，该验证者及其所有委托人将解除质押。

</details>

<details>

<summary><b>如何防止少数顶级验证者集中控制权益？</b></summary>

目前，我们期望社区能够以聪明和自我保护的方式行事。当比特币的挖矿池拥有过多的挖矿算力时，社区通常会停止对该挖矿池的贡献。Evmos 最初将依赖于相同的效应。在未来，将会部署其他机制，以尽可能平滑这个过程：

- **无惩罚重新委托：** 这允许委托人轻松地从一个验证者切换到另一个验证者，以减少验证者的粘性。
- **用户界面警告：** 钱包可以实现警告功能，如果用户想要委托给已经拥有大量权益的验证者，将显示警告信息。

</details>

## 技术要求

<details>

<summary><b>硬件要求是什么？</b></summary>

验证者应该准备一个或多个数据中心位置，具备冗余电源、网络、防火墙、HSM 和服务器。

我们预计最初需要适度的硬件规格，并且随着网络使用的增加，这些规格可能会提高。参与测试网络是了解更多信息的最佳途径。

</details>

<details>

<summary><b>软件要求是什么？</b></summary>

除了运行 Evmos 节点外，验证者还应开发监控、警报和管理解决方案。

</details>

<details>

<summary><b>带宽要求是什么？</b></summary>

与以太坊或比特币等链相比，Evmos 具有非常高的吞吐量能力。

因此，我们建议数据中心节点仅连接到云端的可信全节点或其他相互了解的验证者。这样可以减轻数据中心节点在抵御拒绝服务攻击方面的负担。

最终，随着网络的更广泛使用，人们可以合理地预期每天的带宽将达到几个千兆字节的数量级。

</details>

<details>

<summary><b>在后勤方面，运行验证者意味着什么？</b></summary>

成功运行验证者将需要多个高技能人员的努力和持续的运营关注。这将比运行比特币矿工要复杂得多。

</details>

<details>

<summary><b>如何处理密钥管理？</b></summary>

验证者应该准备运行支持ed25519密钥的HSM。以下是一些潜在的选择：

- YubiHSM 2
- Ledger Nano S
- Ledger BOLOS SGX enclave
- Thales nShield支持
- [Strangelove Horcrux](https://github.com/strangelove-ventures/horcrux)

Evmos团队不推荐其中任何一种解决方案。鼓励社区加强努力，改进HSM和密钥管理的安全性。

</details>

<details>

<summary><b>验证者在运营方面可以期待什么？</b></summary>

有效的运营是避免意外解除绑定或被削减的关键。这包括能够应对攻击、停机以及在数据中心中保持安全和隔离。

</details>

<details>

<summary><b>维护要求是什么？</b></summary>

验证者应该预期定期进行软件更新以适应升级和错误修复。在引导阶段，网络中不可避免地会出现一些问题，这将需要大量的警惕性。

</details>

<details>

<summary><b>验证者如何保护自己免受拒绝服务攻击？</b></summary>

拒绝服务攻击是指攻击者向某个IP地址发送大量的互联网流量，以阻止该IP地址上的服务器与互联网连接。

攻击者扫描网络，试图了解各个验证者节点的IP地址，并通过向其发送大量流量来阻断它们的通信。

减轻这些风险的一种推荐方法是验证者在网络拓扑中精心构建所谓的哨兵节点架构。

验证节点只应连接到它们信任的全节点，因为它们自己操作这些节点或由其他验证节点运行。验证节点通常在数据中心运行。大多数数据中心提供与主要云服务提供商的网络直接连接。验证节点可以使用这些链接连接到云中的哨兵节点。这将使验证节点的拒绝服务负担转移到其哨兵节点上，并可能需要启动或激活新的哨兵节点以减轻对现有哨兵节点的攻击。

哨兵节点可以快速启动或更改其IP地址。由于与哨兵节点的链接位于私有IP空间中，因此互联网攻击无法直接干扰它们。这将确保验证节点的区块提案和投票始终传递到网络的其他部分。

预计验证节点的良好操作程序将完全减轻这些威胁。

有关哨兵节点架构的更多信息，请参阅[此处](https://forum.cosmos.network/t/sentry-node-architecture-overview/454)。

</details>


---
sidebar_position: 6
---

# Validator FAQ

Check the FAQ for running a validator on Evmos.

## General Concepts

<details>

<summary><b>What is a validator?</b></summary>

Evmos is powered by [Tendermint](https://docs.tendermint.com/v0.34/introduction/what-is-tendermint.html) Core,
which relies on a set of validators to secure the network. Validators run a full node and participate in consensus
by broadcasting votes which contain cryptographic signatures signed by their private key. Validators commit new
blocks in the blockchain and receive revenue in exchange for their work. They also participate in on-protocol
treasury governance by voting on governance proposals. A validator's voting influence is weighted according to
their total stake.

</details>

<details>

<summary><b>What is "staking"?</b></summary>

Evmos is a public Proof-of-Stake (PoS) blockchain, meaning that validator's weight is determined by the amount of
staking tokens (EVMOS) bonded as collateral. These staking tokens can be staked directly by the validator or delegated
to them by EVMOS holders.

Any user in the system can declare its intention to become a validator by sending a `create-validator` transaction.
From there, they become validators.

The weight (i.e. total stake or voting power) of a validator determines wether or not it is an active validator,
and also how frequently this node will have to propose a block and how much revenue it will obtain. Initially, only
the top 150 validators with the most weight will be active validators. If validators double-sign, or are frequently
offline, they risk their staked tokens (including EVMOS delegated by users) being "slashed" by the protocol to
penalize negligence and misbehavior.

</details>

<details>

<summary><b>What is a full node?</b></summary>

A full node is a program that fully validates transactions and blocks of a blockchain. It is distinct from a light
client node that only processes block headers and a small subset of transactions. Running a full node requires more
resources than a light client but is necessary in order to be a validator. In practice, running a full-node only
implies running a non-compromised and up-to-date version of the software with low network latency and without downtime.

Of course, it is possible and encouraged for any user to run full nodes even if they do not plan to be validators.

</details>

<details>

<summary><b>What is a delegator?</b></summary>

Delegators are EVMOS holders who cannot, or do not want to run validator operations themselves. Users can delegate
EVMOS to a validator and obtain a part of its revenue in exchange (for more detail on how revenue is distributed, see
`What is the incentive to stake?` and `What is a validator's commission?` sections below).

Because they share revenue with their validators, delegators also share responsibility. Should a validator misbehave,
each of its delegators will be partially slashed in proportion to their stake. This is why delegators should perform
due-diligence on validators before delegating, as well as diversifying by spreading their stake over multiple validators.

Delegators play a critical role in the system, as they are responsible for choosing validators. Be aware that being a
delegator is not a passive role. Delegators are obligated to remain vigilant and actively monitor the actions of their
validators, switching should they fail to act responsibly.

</details>

## Becoming a Validator

<details>

<summary><b>How to become a validator?</b></summary>

Any participant in the network can signal their intent to become a validator by creating a validator and registering
its validator profile. To do so, the candidate broadcasts a `create-validator` transaction, in which they must
submit the following information:

- **Validator's PubKey**: Validator operators can have different accounts for validating and holding liquid funds.
The PubKey submitted must be associated with the private key with which the validator intends to sign *prevotes*
and *precommits*.
- **Validator's Address**: `evmosvaloper1-` address. This is the address used to identify your validator publicly.
The private key associated with this address is used to bond, unbond, and claim rewards.
- **Validator's name** (also known as the **moniker**)
- **Validator's website** *(optional)*
- **Validator's description** *(optional)*
- **Initial commission rate**: The commission rate on block provisions, block rewards and fees charged to delegators.
- **Maximum commission**: The maximum commission rate which this validator will be allowed to charge.
- **Commission change rate**: The maximum daily increase of the validator commission.
- **Minimum self-bond amount**: Minimum amount of EVMOS the validator needs to have bonded at all times. If the
validator's self-bonded stake falls below this limit, its entire staking pool will be unbonded.
- **Initial self-bond amount**: Initial amount of EVMOS the validator wants to self-bond.

```bash
evmosd tx staking create-validator
--pubkey evmosvalconspub1zcjduepqs5s0vddx5m65h5ntjzwd0x8g3245rgrytpds4ds7vdtlwx06mcesmnkzly
--amount "2aevmos"
--from tmp
--commission-rate="0.20"
--commission-max-rate="1.00"
--commission-max-change-rate="0.01"
--min-self-delegation "1"
--moniker "validator"
--chain-id "evmos_9000-4"
--gas auto
--node tcp://127.0.0.1:26647
```

:::danger
🚨 **DANGER**: <u>Never</u> create your mainnet validator keys using a [`test`](./../../protocol/concepts/keyring#testing)
keying backend. Doing so might result in a loss of funds by making your funds remotely accessible via the
`eth_sendTransaction` JSON-RPC endpoint.

Ref: [Security Advisory: Insecurely configured geth can make funds remotely accessible](https://blog.ethereum.org/2015/08/29/security-alert-insecurely-configured-geth-can-make-funds-remotely-accessible/)
:::

Once a validator is created and registered, EVMOS holders can delegate EVMOS to it, effectively adding stake to
its pool. The total stake of a validator is the sum of the EVMOS self-bonded by the validator's operator and the
EVMOS bonded by external delegators.

**Only the top 150 validators with the most stake are considered the active validators**, becoming
**bonded validators**. If ever a validator's total stake dips below the top 150, the validator loses
its validator privileges (meaning that it won't generate rewards) and no longer serves as part of
the active set (i.e doesn't participate in consensus), entering **unbonding mode** and eventually becomes **unbonded**.

</details>

## Validator keys and states

<details>

<summary><b>What are the different types of keys?</b></summary>

In short, there are two types of keys:

- **Tendermint Key**: This is a unique key used to sign block hashes. It is associated with a public key
`evmosvalconspub`.
    - Generated when the node is created with `evmosd init`.
    - Get this value with `evmosd tendermint show-validator`
e.g. `evmosvalconspub1zcjduc3qcyj09qc03elte23zwshdx92jm6ce88fgc90rtqhjx8v0608qh5ssp0w94c`

- **Application keys**: These keys are created from the application and used to sign transactions. As a validator,
you will probably use one key to sign staking-related transactions, and another key to sign oracle-related
transactions. Application keys are associated with a public key `evmospub-` and an address `evmos-`. Both
are derived from account keys generated by `evmosd keys add`.

:::warning
A validator's operator key is directly tied to an application key, but uses reserved prefixes solely for this
purpose: `evmosvaloper` and `evmosvaloperpub`
:::

</details>

<details>

<summary><b>What are the different states a validator can be in?</b></summary>

After a validator is created with a `create-validator` transaction, it can be in three states:

- `bonded`: Validator is in the active set and participates in consensus. Validator is earning rewards and can be
slashed for misbehaviour.
- `unbonding`: Validator is not in the active set and does not participate in consensus. Validator is not earning
rewards, but can still be slashed for misbehaviour. This is a transition state from `bonded` to `unbonded`. If
validator does not send a `rebond` transaction while in `unbonding` mode, it will take two weeks for the state
transition to complete.
- `unbonded`: Validator is not in the active set, and therefore not signing blocks. Unbonded validators cannot be
slashed, but do not earn any rewards from their operation. It is still possible to delegate EVMOS to this
validator. Un-delegating from an `unbonded` validator is immediate.

Delegators have the same state as their validator.

:::warning
Delegations are not necessarily bonded. EVMOS can be delegated and bonded, delegated and unbonding, delegated and
unbonded, or liquid.
:::

</details>

<details>

<summary><b>What is "self-bond"? How can I increase my "self-bond"?</b></summary>

The validator operator's "self-bond" refers to the amount of EVMOS stake delegated to itself. You can increase your
self-bond by delegating more EVMOS to your validator account.

</details>

<details>

<summary><b>Is there a testnet faucet?</b></summary>

If you want to obtain coins for the testnet, you can do so by using the [faucet](https://faucet.evmos.dev/).

</details>

<details>

<summary><b>Is there a minimum amount of EVMOS that must be staked to be an active (bonded) validator?</b></summary>

There is no minimum. The top 150 validators with the highest total stake (where
`total stake = self-bonded stake + delegators stake`) are the active validators.

</details>

<details>

<summary><b>How will delegators choose their validators?</b></summary>

Delegators are free to choose validators according to their own subjective criteria. That said, criteria anticipated to
be important include:

- **Amount of self-bonded EVMOS:** Number of EVMOS a validator self-bonded to its staking pool. A validator with higher
amount of self-bonded EVMOS has more skin in the game, making it more liable for its actions.

- **Amount of delegated EVMOS:** Total number of EVMOS delegated to a validator. A high stake shows that the community
trusts this validator, but it also means that this validator is a bigger target for hackers. Validators are expected
to become less and less attractive as their amount of delegated EVMOS grows. Bigger validators also increase the
centralization of the network.

- **Commission rate:** Commission applied on revenue by validators before it is distributed to their delegators

- **Track record:** Delegators will likely look at the track record of the validators they plan to delegate to. This
includes seniority, past votes on proposals, historical average uptime and how often the node was compromised.

Apart from these criteria, there will be a possibility for validators to signal a website address to complete their
resume. Validators will need to build reputation one way or another to attract delegators. For example, it would be
a good practice for validators to have their setup audited by third parties. Note though, that the Evmos team will
not approve or conduct any audit itself.

</details>

## Responsibilities

<details>

<summary><b>Do validators need to be publicly identified?</b></summary>

No, they do not. Each delegator will value validators based on their own criteria. Validators will be able(and are
advised) to register a website address when they nominate themselves so that they can advertise their operation as
they see fit. Some delegators may prefer a website that clearly displays the team running the validator and their
resume, while others might prefer anonymous validators with positive track records. Most likely both identified
and anonymous validators will coexist in the validator set.

</details>

<details>

<summary><b>What are the responsibilities of a validator?</b></summary>

Validators have three main responsibilities:

- **Be able to constantly run a correct version of the software:** validators need to make sure that their servers are
always online and their private keys are not compromised.

- **Provide oversight and feedback on correct deployment of community pool funds:** the Evmos protocol includes the a
governance system for proposals to the facilitate adoption of its currencies. Validators are expected to hold budget
executors to account to provide transparency and efficient use of funds.

Additionally, validators are expected to be active members of the community. They should always be up-to-date with the
current state of the ecosystem so that they can easily adapt to any change.

</details>

<details>

<summary><b>What does staking imply?</b></summary>

Staking EVMOS can be thought of as a safety deposit on validation activities. When a validator or a delegator wants to
retrieve part or all of their deposit, they send an unbonding transaction. Then, the deposit undergoes a *two week
unbonding period* during which they are liable to being slashed for potential misbehavior committed by the validator
before the unbonding process started.

Validators, and by association delegators, receive block provisions, block rewards, and fee rewards. If a validator
misbehaves, a certain portion of its total stake is slashed (the severity of the penalty depends on the type
of misbehavior). This means that every user that bonded EVMOS to this validator gets penalized in proportion
to its stake. Delegators are therefore incentivized to delegate to validators that they anticipate will function safely.

</details>
<details>

<summary><b>Can a validator run away with its delegators' EVMOS?</b></summary>

By delegating to a validator, a user delegates staking power. The more staking power a validator has, the more weight
it has in the consensus and processes. This does not mean that the validator has custody of its delegators' EVMOS.
*By no means can a validator run away with its delegator's funds*.

Even though delegated funds cannot be stolen by their validators, delegators are still liable if their validators
misbehave. In such case, each delegators' stake will be partially slashed in proportion to their relative stake.

</details>

<details>

<summary><b>How often will a validator be chosen to propose the next block? Does it go up with the quantity of EVMOS
staked?</b></summary>

The validator that is selected to mine the next block is called the **proposer**, the "leader" in the consensus for the
round. Each proposer is selected deterministically, and the frequency of being chosen is equal to the relative total
stake (where total stake = self-bonded stake + delegators stake) of the validator. For example, if the total
bonded stake across all validators is 100 EVMOS, and a validator's total stake is 10 EVMOS, then this validator
will be chosen 10% of the time as the proposer.

To understand more about the proposer selection process in Tendermint BFT consensus, read more
[in their official docs](https://docs.tendermint.com/master/spec/consensus/proposer-selection.html).

</details>

## Incentives

<details>

<summary><b>What is the incentive to stake?</b></summary>

Each member of a validator's staking pool earns different types of revenue:

- **Block rewards:** Native tokens of applications run by validators (e.g. EVMOS on Evmos) are inflated to produce
block provisions. These provisions exist to incentivize EVMOS holders to bond their stake, as non-bonded EVMOS will
be diluted over time.
- **Transaction fees:** Evmos maintains a whitelist of token that are accepted as fee payment. The initial fee token is
the `evmos`.

This total revenue is divided among validators' staking pools according to each validator's weight. Then, within each
validator's staking pool the revenue is divided among delegators in proportion to each delegator's stake. A commission
on delegators' revenue is applied by the validator before it is distributed.

</details>

<details>

<summary><b>What is the incentive to run a validator?</b></summary>

Validators earn proportionally more revenue than their delegators because of commissions.

Validators also play a major role in governance. If a delegator does not vote, they inherit the vote from their
validator. This gives validators a major responsibility in the ecosystem.

</details>

<details>

<summary><b>What is a validator's commission?</b></summary>

Revenue received by a validator's pool is split between the validator and its delegators. The validator can apply a
commission on the part of the revenue that goes to its delegators. This commission is set as a percentage.
Each validator is free to set its initial commission, maximum daily commission change rate and maximum commission.
Evmos enforces the parameter that each validator sets. These parameters can only be defined when initially declaring
candidacy, and may only be constrained further after being declared.

</details>

<details>

<summary><b>How are block provisions distributed?</b></summary>

Block provisions (rewards) are distributed proportionally to all validators relative to their total stake (voting power).
This means that even though each validator gains EVMOS with each provision, all validators will still maintain equal
weight.

Let us take an example where we have 10 validators with equal staking power and a commission rate of 1%. Let us also
assume that the provision for a block is 1000 EVMOS and that each validator has 20% of self-bonded EVMOS. These tokens
do not go directly to the proposer. Instead, they are evenly spread among validators. So now each validator's pool
has 100 EVMOS. These 100 EVMOS will be distributed according to each participant's stake:

- Commission: `100*80%*1% = 0.8 EVMOS`
- Validator gets: `100\*20% + Commission = 20.8 EVMOS`
- All delegators get: `100\*80% - Commission = 79.2 EVMOS`

Then, each delegator can claim its part of the 79.2 EVMOS in proportion to their stake in the validator's staking pool.
Note that the validator's commission is not applied on block provisions. Note that block rewards (paid in EVMOS) are
distributed according to the same mechanism.

</details>

<details>

<summary><b>How are fees distributed?</b></summary>

Fees are similarly distributed with the exception that the block proposer can get a bonus on the fees of the block it
proposes if it includes more than the strict minimum of required precommits.

When a validator is selected to propose the next block, it must include at least ⅔ precommits for the previous block in
the form of validator signatures. However, there is an incentive to include more than ⅔ precommits in the form of a
bonus. The bonus is linear: it ranges from 1% if the proposer includes ⅔rd precommits (minimum for the block to be
valid) to 5% if the proposer includes 100% precommits. Of course the proposer should not wait too long or other
validators may timeout and move on to the next proposer. As such, validators have to find a balance between
wait-time to get the most signatures and risk of losing out on proposing the next block. This mechanism aims
to incentivize non-empty block proposals, better networking between validators as well as to mitigate censorship.

Let's take a concrete example to illustrate the aforementioned concept. In this example, there are 10 validators with
equal stake. Each of them applies a 1% commission and has 20% of self-bonded EVMOS. Now comes a successful block
that collects a total of 1005 EVMOS in fees. Let's assume that the proposer included 100% of the signatures in its
block. It thus obtains the full bonus of 5%.

We have to solve this simple equation to find the reward $R$ for each validator:

$$9R ~ + ~ R ~ + ~ 5\%(R) ~ = ~ 1005 ~ \Leftrightarrow ~ R ~ = ~ 1005 ~/ ~10.05 ~ = ~ 100$$

- For the proposer validator:

    - The pool obtains $R ~ + ~ 5\%(R)$: 105 EVMOS
    - Commission: $105 ~ *~ 80\% ~* ~ 1\%$ = 0.84 EVMOS
    - Validator's reward: $105 ~ * ~ 20\% ~ + ~ Commission$ = 21.84 EVMOS
    - Delegators' rewards: $105 ~ * ~ 80\% ~ - ~ Commission$ = 83.16 EVMOS \(each delegator will be able to claim its portion
of these rewards in proportion to their stake\)

    - The pool obtains $R$: 100 EVMOS
    - Commission: $100 ~ *~ 80\% ~* ~ 1\%$ = 0.8 EVMOS
    - Validator's reward: $100 ~ * ~ 20\% ~ + ~ Commission$ = 20.8 EVMOS
    - Delegators' rewards: $100 ~ * ~ 80\% ~ - ~ Commission$ = 79.2 EVMOS \(each delegator will be able to claim its portion
of these rewards in proportion to their stake\

</details>

<details>

<summary><b>What are the slashing conditions?</b></summary>

If a validator misbehaves, its bonded stake along with its delegators' stake and will be slashed. The severity of the
punishment depends on the type of fault. There are 3 main faults that can result in slashing of funds for a validator
and its delegators:

- **Double-signing:** If someone reports on chain A that a validator signed two blocks at the same height on chain A and
chain B, and if chain A and chain B share a common ancestor, then this validator will get slashed on chain A. The penalty
for double signing is 10.00% of total stake.

- **Downtime:** If a validator misses more than 50% of the last 90.000 blocks, they will get slashed by 0.50%.
- **Unavailability:** If a validator's signature has not been included in the last X blocks, the validator will get
slashed by a marginal amount proportional to X. If X is above a certain limit Y, then the validator will get unbonded.

Note that even if a validator does not intentionally misbehave, it can still be slashed if its node crashes, looses
connectivity, gets DDoSed, or if its private key is compromised.

Here are some links to community's learning from double signing worth a look:

- [Learnings from BlockDaemon](https://blockdaemon.com/documentation/evmos-post-mortem/)

</details>

<details>

<summary><b>Are there any best practice guides on preventing double-signing?</b></summary>

There is an awesome guide. Polkachu is a validator on Evmos and they have wrote [this page](https://github.com/polkachu/validator-guide/blob/main/validator_server_migration_best_practice.md)
to help out.

</details>

<details>

<summary><b>Do validators need to self-bond EVMOS?</b></summary>

No, they do not. A validators total stake is equal to the sum of its own self-bonded stake and of its delegated stake.
This means that a validator can compensate its low amount of self-bonded stake by attracting more delegators. This is
why reputation is very important for validators.

Even though there is no obligation for validators to self-bond EVMOS, delegators should want their validator to have
self-bonded EVMOS in their staking pool. In other words, validators should have skin-in-the-game.

In order for delegators to have some guarantee about how much skin-in-the-game their validator has, the latter can signal
a minimum amount of self-bonded EVMOS. If a validator's self-bond goes below the limit that it predefined, this
validator and all of its delegators will unbond.

</details>

<details>

<summary><b>How to prevent concentration of stake in the hands of a few top validators?</b></summary>

For now the community is expected to behave in a smart and self-preserving way. When a mining pool in Bitcoin gets too
much mining power the community usually stops contributing to that pool. Evmos will rely on the same effect initially.
In the future, other mechanisms will be deployed to smoothen this process as much as possible:

- **Penalty-free re-delegation:** This is to allow delegators to easily switch from one validator to another, in order
to reduce validator stickiness.
- **UI warning:** Wallets can implement warnings that will be displayed to users if they want to delegate to a validator
that already has a significant amount of staking power.

</details>

## Technical Requirements

<details>

<summary><b>What are hardware requirements?</b></summary>

Validators should expect to provision one or more data center locations with redundant power, networking, firewalls,
HSMs and servers.

We expect that a modest level of hardware specifications will be needed initially and that they might rise as network
use increases. Participating in the testnet is the best way to learn more.

</details>

<details>

<summary><b>What are software requirements?</b></summary>

In addition to running an Evmos node, validators should develop monitoring, alerting and management solutions.

</details>

<details>

<summary><b>What are bandwidth requirements?</b></summary>

Evmos has the capacity for very high throughput compared to chains like Ethereum or Bitcoin.

As such, we recommend that the data center nodes only connect to trusted full nodes in the cloud or other validators
that know each other socially. This relieves the data center node from the burden of mitigating denial-of-service attacks.

Ultimately, as the network becomes more used, one can realistically expect daily bandwidth on the order of several gigabytes.

</details>

<details>

<summary><b>What does running a validator imply in terms of logistics?</b></summary>

A successful validator operation will require the efforts of multiple highly skilled individuals and continuous operational
attention. This will be considerably more involved than running a bitcoin miner for instance.

</details>

<details>

<summary><b>How to handle key management?</b></summary>

Validators should expect to run an HSM that supports ed25519 keys. Here are potential options:

- YubiHSM 2
- Ledger Nano S
- Ledger BOLOS SGX enclave
- Thales nShield support
- [Strangelove Horcrux](https://github.com/strangelove-ventures/horcrux)

The Evmos team does not recommend one solution above the other. The community is encouraged to bolster the effort to
improve HSMs and the security of key management.

</details>

<details>

<summary><b>What can validators expect in terms of operations?</b></summary>

Running effective operation is the key to avoiding unexpectedly unbonding or being slashed. This includes being able to
respond to attacks, outages, as well as to maintain security and isolation in your data center.

</details>

<details>

<summary><b>What are the maintenance requirements?</b></summary>

Validators should expect to perform regular software updates to accommodate upgrades and bug fixes. There will inevitably
be issues with the network early in its bootstrapping phase that will require substantial vigilance.

</details>

<details>

<summary><b>How can validators protect themselves from Denial-of-Service attacks?</b></summary>

Denial-of-service attacks occur when an attacker sends a flood of internet traffic to an IP address to prevent the server
at the IP address from connecting to the internet.

An attacker scans the network, tries to learn the IP address of various validator nodes and disconnect them from
communication by flooding them with traffic.

One recommended way to mitigate these risks is for validators to carefully structure their network topology in a so-called
sentry node architecture.

Validator nodes should only connect to full-nodes they trust because they operate them themselves or are run by other
validators they know socially. A validator node will typically run in a data center. Most data centers provide direct
links the networks of major cloud providers. The validator can use those links to connect to sentry nodes in the cloud.
This shifts the burden of denial-of-service from the validator's node directly to its sentry nodes, and may require new
sentry nodes be spun up or activated to mitigate attacks on existing ones.

Sentry nodes can be quickly spun up or change their IP addresses. Because the links to the sentry nodes are in private
IP space, an internet based attacked cannot disturb them directly. This will ensure validator block proposals and votes
always make it to the rest of the network.

It is expected that good operating procedures on that part of validators will completely mitigate these threats.

For more on sentry node architecture, see [this](https://forum.cosmos.network/t/sentry-node-architecture-overview/454).

</details>
