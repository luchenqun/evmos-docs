# IBC 中继器

中继器从一个区块链读取数据包，并将其传输到另一个区块链，充当一种分散的邮政服务，允许两个主权区块链相互发送消息。实现这一过程的方法是使用 Inter-Blockchain Communication (IBC) 协议。它允许独立的区块链之间进行通信并交换价值，特别是代币。IBC 中继器是一种软件程序，用于促进支持 IBC 的两个不同区块链网络之间的通信。

![TAO-IBC](https://tutorials.cosmos.network/resized-images/600/academy/3-ibc/images/connectionstate.png)

中继器还可以在链之间打开路径，创建客户端、连接和通道。IBC 协议由两个层次组成：TAO 层和建立在 TAO 之上的 APP 层。TAO 层主要负责 IBC 的功能，允许连接的区块链通过专用通道发送信息数据包，使用智能合约模块，其中包括用于无需信任验证其他区块链发送的状态是否有效的轻客户端。APP 层使任何应用层协议都可以构建在 TAO 之上运行。IBC 已在[54个网络](https://mapofzones.com/)上启用，每月使用 IBC 执行超过 6000 万笔交易。中继器对 IBC 协议的运行至关重要。它们可以同时处理大量的交易，但仍可能发生拥塞事件。中继器基础设施定期进行扩展，但在此期间，用户可以通过将其代币委托给也运行 IBC 中继器的验证者来支持中继器。

大多数中继器还在一个或多个 Cosmos 生态系统区块链上运行验证者节点。这些中继器作为验证者获得的支持越多，他们就越有可能继续在其中继器节点上运行，即使亏损。此外，了解到中继器以这种方式受到重视，应该会鼓励更多的验证者建立自己的中继器基础设施。

Please paste the Markdown content here.


---
sidebar_position: 5
---

# IBC Relayers

Relayers read packets of data from one blockchain and communicate them
to another blockchain, acting as a sort of decentralized postal service that allows two sovereign blockchains to send
messages to each other. The process to enable this utilizes the Inter-Blockchain Communication (IBC) as a protocol.
It allows independent blockchains to communicate with each other
and exchange value, particularly tokens. IBC relayers are software programs that facilitate communication between two
distinct blockchain networks that support IBC.

![TAO-IBC](https://tutorials.cosmos.network/resized-images/600/academy/3-ibc/images/connectionstate.png)

Relayers can also open paths across chains, creating clients, connections, and channels. The
IBC protocol consists of two layers: the TAO layer and the APP layer, built on top of TAO. The TAO layer, which is
primarily responsible for the functionality of IBC, allows connected blockchains to send packets of information via
dedicated channels, using smart contract modules that include a light client for trustlessly verifying that the
state sent by the other blockchain is valid. The APP layer enables any application-layer protocol to be built to
operate on top of TAO. IBC has been enabled on [54 networks](https://mapofzones.com/)
over 60 million transactions currently executed using IBC per month. Relayers are critical to the functioning of
the IBC protocol. They can
handle large numbers of transactions at any given time, but congestion events can still occur. Relayer infrastructure
is being scaled up regularly, but in the meantime, users can support relayers by delegating their tokens to the
validators that also operate IBC relayers.

Most relayers also operate validator nodes on one or more Cosmos
ecosystem blockchains. The more support these relayers receive as validators, the more likely they will be
encouraged to continue operating at a loss on their relayer nodes. Additionally, the knowledge that relayers
are valued in such a way should encourage even more validators to set up relayer infrastructure of their own.
