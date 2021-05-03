## 系统

####  Filecoin节点类型

* 支持多种节点类型
* Filecoin节点中特别重要的是key store，IPLD store以及libp2p的网络接口

#### 文件和数据

* Filecoin网络的基本数据单元：sector和piece

#### 虚拟机

* 虚拟机的组件：Actor，State Tree

#### 区块链

* 消息
* 区块
* 消息池
* 链同步机制

#### 通证

* 钱包的基本组件

#### 存储挖矿（存储容量证明）

* 存储挖矿的细节
* 存储算力共识
* 存储证明机制

#### 市场

* 存储市场
* 检索市场
* 链下操作



## Filecoin 节点

* 简单来说，节点类型是根据节点所提供的服务来定义的。

* 不同类型的节点应该实现不同的属性和特性。
* lotus实现中的节点类型并不严格遵循其他区块链所定义的标准。
* Filecoin节点中的三大问题：
  * 系统文件的存储问题。这里指的是本地存储仓库，而不是存储证明。
  * 节点互联问题。节点之间如何互相发现和互相连接。
  * 节点时钟问题。

## 节点类型

Filecoin网络中的节点主要是由节点所对外提供服务来标识的。

Filecoin网络中所包含的服务主要有：

* 区块链验证
* 存储市场客户端
* 存储市场提供者
* 检索市场客户端
* 检索市场提供者
* 存储挖矿

参与Filecoin网络的任何节点至少提供区块链验证服务。

节点额外多提供一种服务，就会额外增加一个标签。

需要主要的是，节点的进程实例是和本地repository目录一一对应的，也就是说，一个repo只属于一个节点，这样，一台主机就可以运行多个Filecoin节点，只需要指定不同的repo目录即可。

Filecoin实现需要支持的节点类型（子系统）：

* 区块链验证节点：这是一个Filecoin节点能够参与Filecoin网络需要支持最小功能。验证节点是一种被动节点。在节点第一次加入Filecoin网络时，需要同步区块链，达到当前的共识状态。然后，节点可以持续的获得最新的区块，并验证这些区块，来扩展本地的区块链。
* 客户节点：这种类型的节点构建在区块链验证节点之上，并且由Filecoin网络上的应用来实现。这可以被看作是构建在Filecoin网络上的交易所或去中心化存储应用的基础设施节点（至少就与区块链的交互而言）。客户节点应该实现存储市场服务和检索市场客户服务。客户节点应与存储和检索市场交互，并能够通过数据传输模块进行数据传输。
* 检索矿工节点：该节点类型扩展了区块链验证节点，增加了检索矿工功能，即参与检索市场。因此，该节点类型需要实现检索市场提供者的服务，并能够通过数据传输模块进行数据传输。
* 存储矿工节点：这种类型的节点必须实现验证区块、构建区块和添加区块等区块链所需的所有功能。实现区块链验证、存储挖矿和存储市场提供者服务，并能够通过数据传输模块进行数据传输。



## 节点接口

与上面的介绍对应，lotus的实现定义节点类型的接口如下。

* 本地存储：https://github.com/filecoin-project/lotus/blob/master/node/repo/interface.go
* 节点配置：https://github.com/filecoin-project/lotus/blob/master/node/config/def.go
* 区块链验证节点：https://github.com/filecoin-project/lotus/blob/master/node/impl/full.go
* 客户节点：https://github.com/filecoin-project/lotus/blob/master/node/impl/client/client.go
* 存储矿工节点：https://github.com/filecoin-project/lotus/blob/master/node/impl/storminer.go

```
taoshengshi解读：
1，规范中提到的检索矿工和relay节点，但在代码中没有看到详细的实现。
2，在impl目录中还有remoteworker的节点定义，说明可以单独运行remote worker节点。
3，另外注意node中的module数目录
4，需要解释一下hello协议，也可以放在网络接口部分介绍。
5，介绍中提到的本地存储（keystore和IPLDstore）和用户配置，结合Hello协议，是node中基础设施。
6，另外需要注意的是本地存储中的检索存储。

```



## 节点本地存储

* Filecoin节点运行必须依赖于本地存储
* 本地存储主要存储系统数据和区块链数据
* 本地存储能够被Filecoin的其他子系统访问
* 本地存储与Filestore分离
* 本地存储内容：
  * 节点私钥
  * IPLD状态数据
  * 节点配置数据

#### Keystore

* Filecoin全节点的基础抽象
* 保存私钥对
* 用于miner和worker
* 节点的安全在于如何安全地保存私钥对
  * 私钥与所有子系统分离
  * 子系统单独的keystore
  * 私钥保存在冷存储中
  * 私钥并不用于挖矿
* 存储矿工的私钥设计（**TODO:重点review**）
  * 存储矿工的actor地址：没有自己的地址，绑定在一个actor地址上。
  * owner 私钥对：在矿工注册之前必须存在，公钥绑定在矿工地址上。
  * worker 私钥对：公钥绑定在矿工地址上，可以被矿工选择和修改。用于签名区块和其他一些消息。
  * 多个矿工可以共享一个owner地址，也可以共享一个worker公钥。
  * 改变worker私钥对，2阶段的过程。
* 私钥安全的重要性。

## IPLDStore

* 跨系统和协议的内容寻址数据的互操作性

* 为密码散列原语提供了一种基本的“公共语言”，使数据结构能够在两个独立协议之间被验证地引用和检索。例如，用户可以在以太坊交易或智能合约中引用IPFS目录。

* Filecoin节点的IPLDStore是散列链式数据的本地存储。

* IPFSStore有三层组成：

  * 数据块层，专注于数据块格式和寻址，以及块自描述的编解码器（这里没有使用区块的概念，而是直接使用数据块的概念）
  * 数据模型层，它定义了一组所有实现都必需的类型——下面将更详细地讨论。
  * 模式层，它允许数据模型的扩展，并与更复杂的结构交互，而不需要自定义转换抽象。
  * TODO：如果需要更加详细描述IPLD - https://github.com/ipld/specs

* IPLDStore的数据模型

  * 在其核心，IPLD定义了表示数据的数据模型。数据模型是为跨各种编程语言的具体实现而设计的，同时保持内容寻址和更多工具的可用性。
  * 数据模型包括一系列标准原始类型(或“种类”)，例如布尔值、整数、字符串、空值和字节数组，以及两种递归类型：列表和映射。因为IPLD是为内容寻址设计的，所以它的数据模型中还包含一个“链接”原语。在实践中，链接使用CID规范。IPLD数据被组织成“数据块”，其中数据块由原始的、编码的数据及其内容地址(CID)表示。每个内容可寻址的数据块都可以表示为一个数据块，这些数据块一起可以形成一个连贯的图，即Merkle DAG（https://docs.ipfs.io/concepts/merkle-dag/）。
  * 应用程序通过数据模型与IPLD交互，而IPLD通过一组编解码器处理编码和解码。IPLD编解码器可以支持整个数据模型或部分数据模型。支持完整数据模型的两种编解码器是DAG-CBOR（https://github.com/ipld/specs/blob/master/block-layer/codecs/dag-cbor.md）和DAG-JSON（https://github.com/ipld/specs/blob/master/block-layer/codecs/dag-json.md）。这些编解码器分别基于CBOR和JSON序列化格式，但包括允许它们封装IPLD数据模型(包括其链接类型)的形式化，以及在任何数据集与其各自的内容地址(或散列摘要)之间创建严格映射的附加规则。这些规则包括在编码映射时强制指定键的特定顺序，或者在存储整数类型时指定大小。

* IPLD在Filecoin中的应用

  在Filecoin网络中，IPLD有两种使用方式:

  * 所有系统数据结构都使用DAG-CBOR（一种IPLD编解码器）存储。DAG-CBOR是CBOR的一个更严格子集，具有一个预定义的标记方案，设计用于存储、检索和遍历散列链式数据的DAG。与CBOR相比，DAG-CBOR具有确定性保证。
  * 存储在Filecoin网络上的文件和数据也使用各种IPLD编解码器存储（不一定是DAG-CBOR）。

  IPLD为数据提供了一致和连贯的抽象，允许Filecoin构建复杂的多块数据结构，并与之交互，如HAMT和AMT。Filecoin使用DAG-CBOR编解码器对其数据结构进行序列化和反序列化，并使用IPLD数据模型与该数据交互，在该数据模型上构建了各种工具。IPLD选择器（IPLD Selectors ）还可以用于在链式数据结构中寻址特定节点。

  另外，Filecoin网络主要依赖于两个不同的IPLD GraphStores:

  * 一个存储区块链的ChainStore，包括块头，相关消息等。
  * 一个StateStore，它存储来自给定区块链的有效状态载荷，或者由Filecoin 虚拟机将给定链中的所有区块消息应用到起创世状态而产生的stateTree。

* ChainStore在Chain Sync的引导阶段由节点从其对等节点处下载，之后由节点存储在本地。每接收到一个新的区块，ChainStore就会被更新；或者如果节点同步到一个新的最佳链，ChainStore就会被更新。

* StateStore是通过执行计算ChainStore中的所有区块消息来生成的，之后由节点存储在本地。虚拟机解释器每计算一个新传入的区块时，StateStore就会被更新，并相应地被在区块头的ParentState字段中产生新的状态块（StateStore）引用。

* TODO：这里的解释还不够通俗易懂，ChainStore应该是虚拟机处理之前的数据，StateStore是虚拟机处理之后的数据。

* * 



## 网络接口

* Filecoin节点使用libp2p网络栈的几种协议进行对等发现、对等路由、区块传播和消息传播。
* Libp2p是一个用于对等网络的模块化网络栈。它包括若干协议和机制，以实现高效、安全和弹性的对等通信。
* Libp2p节点打开彼此之间的连接，并在同一连接上挂载不同的协议或流。在最初的握手中，节点交换各自支持的协议，所有与Filecoin相关的协议将被挂载在/fil/…协议标识符。
* Libp2p规范：https://github.com/libp2p/specs （TODO）

#### Libp2p在Filecoin中的应用

* Graphsync：Graphsync是一个跨对等点进行图同步的协议。它用于在Filecoin节点之间引用、寻址、请求和传输区块链数据和用户数据。GraphSync规范草案（TODO：https://github.com/ipld/specs/blob/master/block-layer/graphsync/graphsync.md）提供了关于GraphSync使用的概念、接口和网络消息的更多细节。filecoin对协议id没有特定的修改。
* Gossipsub：使用基于gossip的pubsub协议(简称Gossipsub)。Filecoin网络中的区块头和消息通过Gossipsub协议传播。与传统pubsub协议一样，节点订阅主题并接收关于这些主题发布的消息。当节点从它们订阅的主题接收消息时，它们运行一个验证过程：
  * i)将消息传递给应用
  * ii)将消息进一步转发给节点所知道的订阅相同主题的节点。
* 此外，
  * Filecoin中使用的GossipSub v1.1版本增强了安全机制，使协议具有抵御安全攻击的弹性。
  * GossipSub规范（TODO:https://github.com/libp2p/specs/tree/master/pubsub/gossipsub）提供了与它的设计和实现有关的所有协议细节，以及协议参数的特定设置。
  * filecoin对协议id没有特定的修改。但是主题标识符必须是fil/blocks/<network-name>和fil/msgs/<network-name>的形式。
* Kademlia DHT：Kademlia DHT是一个分布式哈希表，对一个特定节点的最大查找次数以对数为界。在Filecoin网络中，Kademlia DHT主要用于对等节点发现和对等节点路由。特别是，当一个节点想要在Filecoin网络中存储数据时，它们会得到一个矿工列表及其节点信息。该节点信息包括矿工的PeerID(以及其他信息)。为了连接到矿机并交换数据，想要在网络中存储数据的节点必须找到矿机的Multiaddress，这是通过查询DHT完成的。libp2p Kad DHT规范（TODO：https://github.com/libp2p/go-libp2p-kad-dht）提供了DHT结构的实现细节。对于Filecoin网络，协议id必须是fil/<network-name>/kad/1.0.0格式。
* 启动引导列表：这是一个新节点在加入网络时试图连接到的节点列表。引导节点列表及其地址由用户(即应用程序)定义。
* 对等节点交换：该协议实现了上述在Kademlia DHT上讨论的对等节点发现过程。通过与DHT进行交互，并为它们想要连接的对等节点创建和发出查询，它使对等节点能够找到网络中其他对等节点的信息和地址。

## 时钟

Filecoin假设系统中的参与者之间存在弱时钟同步。也就是说，系统依赖于参与者必须能够访问全局同步的时钟（允许一些有限的时钟偏移）。

Filecoin依赖于这个系统时钟，以确保共识安全。具体来说，时钟对于支持验证规则是必要的，这些规则可以防止区块生产者使用未来的时间戳来挖出区块，并违反协议规定更频繁地进行leader选举。

#### 使用Filecoin系统时钟

以下几种场景使用了Filecoin系统时钟：

* 链同步节点根据时间戳来验证传入的区块是否在适当时间区间内挖出（参考区块验证：语法验证）。这是可以做到的，因为系统时钟将所有的时间映射到一个唯一的纪元号，这个纪元号完全由创世块的开始时间决定（TODO:这里需要结合代码解释的更清楚）。
* 同步节点删除来自未来的区块。
* 挖矿节点维持协议的活性：如果在当前一轮中没有节点产生区块，则允许矿工进行下一轮的leader选举。(参见存储算力共识)。

为了保证矿工完成上述工作，系统时钟必须:

* 和其他节点保持足够低的时钟偏移量，这样就不会被其他节点认为是来自未来的区块（根据验证规则，这些未来的区块会被拒绝：语义验证）。
* 节点初始化时，将epoch号设置为：Floor[(current_time - genesis_time) / epoch_time]。注：Floor为go语言中向下取整的函数。

其他子系统可以向时钟子系统注册NewRound()事件。

## 时钟需求

作为Filecoin协议一部分的时钟应保持同步，偏移量应小于1秒，以便启用适当的验证。

计算机级晶体的偏差可以达到1ppm（即每秒1微秒，或每周0.6秒）。因此，为了尊重上述要求:

* 节点应该运行一个NTP守护进程(例如timesyncd, ntpd, chronyd)来保持它们的时钟同步到一个或多个可靠的外部引用。我们推荐以下来源:

  * pool.ntp.org(https://www.ntppool.org/en/use.html)

  * time.cloudflare.com: 1234(https://www.cloudflare.com/time/)
  * time.google.com(https://developers.google.com/time)
  * time.nist.gov(https://tf.nist.gov/tf-cgi/servers.cgi)

* 较大的挖矿运维可以考虑使用本地NTP/PTP服务器，并且附带GPS参考或频率稳定的外部时钟，以保持时间同步。

挖矿运维有强烈的动机来防止他们的时钟向前飘逸超过一个epoch，从而导致产生的区块被拒绝。同样，挖矿运维也有动机防止时钟偏离到多个epoch，以避免将自己从网络中的同步节点中分离出来。

TODO：这里需要更加详细的介绍一下。

