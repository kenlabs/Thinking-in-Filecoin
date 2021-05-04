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

## 文件和数据

Filecoin的主要目的是存储客户的文件和数据。本节详细介绍与文件、分块(chunking)、编码、图表示、片段(Pieces)、存储抽象等相关的数据结构和工具。

#### 文件

文件是一个可变长度的数据容器。

FileStore是一个可以按路径存储和检索文件的对象。FileStore是一个抽象概念，用于Filecoin将其数据存储到的任何底层系统或设备。它基于Unix文件系统语义，并包含路径的概念。这里的这种抽象是为了确保Filecoin的实现使最终用户能够轻松地使用任何满足他们需求的存储系统来替换底层存储系统。FileStore最简单的版本就是主机操作系统的文件系统。

提出FileStore的抽象是为了应对不同的用户需求。Filecoin用户的需求差异很大，许多用户——尤其是矿工——将在Filecoin底层和周边实现复杂的存储架构。这里的FileStore抽象是为了使这些多样化的需求容易满足。Filecoin协议中的所有文件和扇区的本地数据存储都是根据这个FileStore接口定义的，这使得存储实现可以很容易地替换，最终用户可以替换为他们选择的存储系统。

存储系统案例：

* 主机操作系统的文件系统
* 任何Unix/Posix文件系统
* RAID-backed文件系统
* 分布式文件系统(NFS、HDFS等)
* IPFS
* 数据库
* NAS系统
* 原始串行或块设备
* 原始硬盘驱动器(硬盘扇区等)

#### Filecoin数据片段（Piece）

Filecoin Piece是用户在Filecoin网络上存储数据的主要协商单元。Filecoin Piece不是一个存储单元，它没有特定大小，而是以扇区的大小为上限。一个Filecoin Piece可以是任意大小，但如果一个Piece大于矿工支持的扇区的大小，它必须被拆分为更多的Pieces，以便每个Piece适合一个扇区。

Piece是一个表示文件整体或部分的对象，用于存储客户和存储矿工的交易中。存储客户端雇佣存储矿工来存储Piece。

Piece数据结构设计用于证明存储任意IPLD图和客户端数据。此图显示了Piece及其证明树的详细组成，包括完整的和带宽优化的Piece数据结构。

https://spec.filecoin.io/systems/filecoin_files/piece/pieces.png （TODO：重画此图，或者拿到这个图的原文件）![Minion](https://spec.filecoin.io/systems/filecoin_files/piece/pieces.png )



#### 数据表示

需要强调的是，提交到Filecoin网络的数据需要经过多次转换，才能成为StorageProvider的存储格式。下面是从用户开始准备要存储在Filecoin中的文件，到StorageProvider生成存储在Sector中的所有Pieces标识符的过程。

前三个步骤发生在**客户端**：

1. 当客户端希望在Filecoin网络中存储文件时，他们首先生成文件的IPLD DAG。DAG根节点的散列是ipfs风格的CID，称为有效载荷CID。
2. 为了生成一个Filecoin Piece，IPLD DAG被序列化为一个“内容-可寻址存档”(.car)文件（TODO ： CAR文件格式介绍 https://github.com/ipld/specs/blob/master/block-layer/content-addressable-archives.md#summary ），CAR是原始字节格式。CAR文件是不透明的数据块，它将IPLD节点打包并传输。有效载荷CID在CAR 'ed和非CAR 'ed结构之间是相同的（TODO：？？）。这有助于稍后的数据检索（TODO：当数据在存储客户和存储提供者之间传输时(我们将在后面讨论)）。
3. 生成的.car文件被额外的零位填充，以使该文件形成一个二进制Merkle树。为了实现一个干净的二进制Merkle树，.car文件大小必须是2(^2)大小的幂次。一个填充过程，称为Fr32填充，它将每256位中的254位加2(2)个零位写入到输入文件。下一步，填充过程获取Fr32填充过程的输出，并找到大于它的2的幂值。Fr32填充的结果和2的下一个幂值之间的差距用0填充（TODO：这里描述还不够清晰和准确）。

（TODO：设计原因，内容和Piece CID一一对应，内容一旦确定，CID就确定）为了理解这些步骤背后的原因，了解StorageClient和StorageProvider之间的整体协商过程是很重要的。CID或CommP是客户与存储提供商协商并同意的交易内容。当协议达成后，客户端将文件发送给提供者(使用GraphSync)。提供者必须从接收到的文件中构造CAR文件，并派生出他们一方的Piece CID。为了避免客户端向约定的一方发送不同的文件，提供者生成的Piece CID必须与之前协商的交易中包含的相同。

以下步骤发生在**StorageProvider端**(步骤4也可以发生在客户端)。

4. 一旦StorageProvider从客户端接收到文件，它们就从Piece(填充的.car文件)的散列中计算Merkle根。干净二叉Merkle树的结果根是Piece CID。这也被称为CommP或Piece Commitment，如前所述，必须与交易中包含的承诺相同。
5. Piece与来自其他交易的数据一起包含在一个Sector中。然后StorageProvider计算扇区内所有Pieces的Merkle根。该树的根是CommD(又名committed of Data或UnsealedSectorCID)。
6. 然后StorageProvider密封扇区，产生的Merkle根是CommRLast。
7. 复制证明(PoRep)，特别是SDR，会生成另一个名为CommC的Merkle根哈希，作为对CommD承诺的数据的复制已经正确执行的验证。
8. 最后，CommR(或Commitment of Replication)是CommC || CommRLast的哈希值。

TODO：这里一段需要结合实际文件存储过程来解释清楚。

重要提示:

* Fr32是字段元素的32位表示(在我们的例子中，它是BLS12-381的算术字段)。为了格式良好，类型Fr32的值必须实际适合该字段，但类型系统并不强制这样做。它是一个不变量，必须通过正确的使用来保持。在所谓的Fr32填充的情况下，在一个最多需要254位来表示的数字后面插入两个零位。这保证了结果将是Fr32，而不管最初的254位的值是多少。这是一种“保守的”技术，因为对于一些初始值，实际上只需要一个位的零填充。
* 上面的第2步和第3步特定于Lotus实现。同样的结果可以通过不同的方式实现，例如，不使用Fr32位填充。然而，任何实现都必须确保初始IPLD DAG被序列化和填充，以便它给出一个干净的二叉树，因此，从结果数据blob计算Merkle根给出相同的Piece CID。只要是这种情况，实现就可以偏离上面的前三个步骤。
* 最后，添加一个与Payload CID(在上面的前两个步骤中讨论)和数据检索过程相关的注释是很重要的。检索协议是在有效载荷CID的基础上协商的。当检索协议达成后，检索矿机开始向客户机发送未密封的“非car’ed”文件。传输从IPLD Merkle树的根节点开始，通过这种方式，客户端可以从传输开始验证Payload CID，并验证他们正在接收的文件是他们在交易中协商的文件，而不是随机比特。

#### PieceStore
PieceStore模块允许从一些本地存储中存储和检索Pieces。PieceStore的主要目标是帮助存储和检索市场模块找到密封数据在各个Sector的位置。存储市场写数据，检索市场读数据，然后发送给检索客户端。

PieceStore模块的实现可以在这里找到：https://github.com/filecoin-project/go-fil-markets/tree/master/piecestore

## Filecoin的数据传输

数据传输协议是一种协议，用于在交易完成时在网络上传输全部或部分Piece数据。数据传输模块的总体目标是使其成为底层传输介质的抽象，使得数据可以在Filecoin网络的不同参与方之间传输。目前，

实际用于进行数据传输的底层媒介或协议是GraphSync。因此，数据传输协议也可以被认为是一个协商协议。

数据传输协议主要用于存储和检索交易。在这两种情况下，数据传输请求都是由客户端发起。这样做的主要原因是，客户端通常会在NAT网络的后面，因此，从客户端发起开始任何数据传输都更方便。

* 在Storage Deals的场景下，数据传输请求作为推送（push request）请求发起，以将数据发送到存储提供者。
* 在Retrieval Deals的场景下，数据传输请求被存储提供者作为检索数据的拉请求（pull request）发起。

发起数据传输的请求内容包括一个凭单（voucher）或通证（不要与支付通道凭单混淆），凭单是指向双方之前同意的特定交易。这样存储提供者就可以识别并将请求链接到它已经同意的交易，而不是忽略该请求。如下面所述，检索交易的情况可能略有不同，其中交易提议和数据传输请求可以同时发送。

#### 数据传输模块

这张图显示了数据传输及其模块如何与存储市场和检索市场相交互。特别要注意，来自市场的数据传输请求验证器是如何插入到数据传输模块的，但是它们的代码属于市场系统（TODO：这里需要进一步的阐述清楚，并重画此图）。

![data_transfer](https://spec.filecoin.io/systems/filecoin_files/data_transfer/data-transfer-modules.png)

#### 数据传输中的术语

* 发送请求（Push Request）：请求向另一方发送数据——通常由客户端发起，主要是在存储交易的场景下。
* 拉取请求（Pull Request）：请求另一方发送数据给自己——通常由客户端发起，主要是在检索交易的场景下。
* 请求者（Requestor）：发起数据传输请求的一方(Push或Pull)——通常是客户端，至少目前在Filecoin中实现中是这样，以克服NAT穿越问题。
* 响应者（Responder）：接收数据传输请求的一方——通常是存储提供者。
* 数据传输凭证或令牌（Data Transfer Voucher or Token）：存储数据或检索数据的包装器，可以识别并验证向对方的传输请求。
* 请求验证器（Request Validator）：只有当响应者可以验证请求是直接关联到已有的存储交易或检索交易时，数据传输模块才会启动传输。数据传输模块本身不执行验证。相反，请求验证器检查数据传输凭证，以决定是响应请求还是拒绝请求。
* 数据传输器（Transporter）:一旦完成请求协商和请求验证，实际的传输将由双方的数据传输器（Transporter）管理。传输器是数据传输模块的一部分，但与协商过程隔离。它可以访问底层的可验证传输协议，并使用它发送数据和跟踪进度。
* 订阅者（Subscriber）：通过订阅数据传输事件（如进行中或已完成）来监视数据传输进度的外部组件。
* GraphSync：数据传输器（Transporter）使用的默认底层传输协议。完整的图形同步规范可以在这里找到：https://github.com/ipld/specs/blob/master/block-layer/graphsync/graphsync.md

#### 请求阶段

任何数据传输都有两个基本阶段:

* 协商：请求者和响应者通过验证数据传输凭证来达成传输交易。
* 传输：一旦协商阶段完成，数据就开始传输。用于数据传输的默认协议是Graphsync。

注意，协商阶段和传输阶段可能发生在单独的消息往返过程中，也可能发生在相同的消息往返过程中。在相同往返的情况下，请求方通过发送请求隐式地包含同意交易，而响应方可以同意交易并立即发送或接收数据。这个过程是发生在单次消息往返还是多次消息往返，部分取决于请求是push请求（存储交易）还是pull请求（检索处理），以及数据传输协商过程是否能够顺表使用底层传输机制。在GraphSync作为传输机制的情况下，数据传输请求可以使用GraphSync内置的可扩展性作为GraphSync协议的扩展。因此，Pull Requests只需要一次往返。然而，因为Graphsync是一个请求/响应协议，对push类型的请求没有直接支持，在push类型的请求中，协商过程发生在数据传输模块的libp2p协议`/fil/ datattransfer /1.0.0`的单独请求中。未来其他的传输机制可能同时处理Push和Pull，或者两者都不作为一个单一的往返行程。

在接收到数据传输请求后，数据传输模块对凭证进行解码，并将其交付给请求验证器。在存储交易中，请求验证器检查包含的交易是否是接收方以前同意的交易。对于检索处理，请求提议包含了检索交易本身。只要请求验证器接受交易提议，所有的事情都是一次性完成的。

值得注意的是，在检索交易的场景下，数据提供者可以接受交易请求和数据传输请求，然后暂停检索服务本身，以执行解封过程。存储提供者必须在启动真正的数据传输之前解封所有被请求的数据。此外，存储提供者还可以选择在解封过程开始之前暂停检索数据流，以便请求解封支付请求。存储提供者可以选择请求支付这笔费用，以支付解封计算成本，并避免成为恶意检索行为攻击的受害者（TODO：评估现在网络中发生这种共攻击的风险）。

#### 数据流示例

###### 推送数据流

1. 当请求者想要向另一方发送数据时，它会发起Push传输。
2. 请求者的数据传输模块将向响应者发送一个推送请求以及数据传输凭证。
3. 响应者的数据传输模块通过其依赖的验证器（Validator）验证数据传输请求。
4. 响应者的数据传输模块通过发出一个GraphSync请求来启动传输。
5. 请求者接收GraphSync请求，验证它是否识别了数据传输，并开始发送数据。
6. 响应者接收数据并能产生进度指示。
7. 响应者完成接收数据，并通知任何订阅者。

push流是存储交易的理想选择，在这种情况下，一旦提供者表示他们愿意接受并发布客户的交易提议，客户端就会直接发起数据传输。

![push](https://spec.filecoin.io/_gen/diagrams/systems/filecoin_files/data_transfer/push-flow.svg)

###### 拉取数据流

1. 当请求者想从另一方拉取数据时，它会发起Pull传输。
2. 请求者的数据传输模块通过向响应者发送一个嵌入GraphSync请求中的pull请求来发起传输。该请求包括数据传输凭证。
3. 响应者接收GraphSync请求，并将数据传输请求转发给数据传输模块。
4. 响应者的数据传输模块通过其依赖的PullValidator来验证数据传输请求。
5. 响应者接受GraphSync请求，并将接受响应与数据传输响应一起发送。
6. 请求者接收数据并产生进度指示。这个步骤的时机是在存储提供者完成对数据的解封之后。
7. 请求者完成接收数据，并通知任何监听者。

![pull](https://spec.filecoin.io/_gen/diagrams/systems/filecoin_files/data_transfer/alternate-pull-flow.svg)

#### 协议

数据传输可以通过数据传输协议(一种libp2p协议类型)在网络上进行协商。

使用数据传输协议作为一个独立的libp2p通信机制并不是一个硬性要求——只要双方都有一个可以相互通信的数据传输子系统的实现，任何传输机制(包括脱机机制)都是可以的。

#### 数据结构

###### 数据传输类型

```
package datatransfer

import (
	"fmt"

	"github.com/ipfs/go-cid"
	"github.com/ipld/go-ipld-prime"
	"github.com/libp2p/go-libp2p-core/peer"

	"github.com/filecoin-project/go-data-transfer/encoding"
)

//go:generate cbor-gen-for ChannelID

// TypeIdentifier is a unique string identifier for a type of encodable object in a
// registry
type TypeIdentifier string

// EmptyTypeIdentifier means there is no voucher present
const EmptyTypeIdentifier = TypeIdentifier("")

// Registerable is a type of object in a registry. It must be encodable and must
// have a single method that uniquely identifies its type
type Registerable interface {
	encoding.Encodable
	// Type is a unique string identifier for this voucher type
	Type() TypeIdentifier
}

// Voucher is used to validate
// a data transfer request against the underlying storage or retrieval deal
// that precipitated it. The only requirement is a voucher can read and write
// from bytes, and has a string identifier type
type Voucher Registerable

// VoucherResult is used to provide option additional information about a
// voucher being rejected or accepted
type VoucherResult Registerable

// TransferID is an identifier for a data transfer, shared between
// request/responder and unique to the requester
type TransferID uint64

// ChannelID is a unique identifier for a channel, distinct by both the other
// party's peer ID + the transfer ID
type ChannelID struct {
	Initiator peer.ID
	Responder peer.ID
	ID        TransferID
}

func (c ChannelID) String() string {
	return fmt.Sprintf("%s-%s-%d", c.Initiator, c.Responder, c.ID)
}

// OtherParty returns the peer on the other side of the request, depending
// on whether this peer is the initiator or responder
func (c ChannelID) OtherParty(thisPeer peer.ID) peer.ID {
	if thisPeer == c.Initiator {
		return c.Responder
	}
	return c.Initiator
}

// Channel represents all the parameters for a single data transfer
type Channel interface {
	// TransferID returns the transfer id for this channel
	TransferID() TransferID

	// BaseCID returns the CID that is at the root of this data transfer
	BaseCID() cid.Cid

	// Selector returns the IPLD selector for this data transfer (represented as
	// an IPLD node)
	Selector() ipld.Node

	// Voucher returns the voucher for this data transfer
	Voucher() Voucher

	// Sender returns the peer id for the node that is sending data
	Sender() peer.ID

	// Recipient returns the peer id for the node that is receiving data
	Recipient() peer.ID

	// TotalSize returns the total size for the data being transferred
	TotalSize() uint64

	// IsPull returns whether this is a pull request
	IsPull() bool

	// ChannelID returns the ChannelID for this request
	ChannelID() ChannelID

	// OtherPeer returns the counter party peer for this channel
	OtherPeer() peer.ID
}

// ChannelState is channel parameters plus it's current state
type ChannelState interface {
	Channel

	// SelfPeer returns the peer this channel belongs to
	SelfPeer() peer.ID

	// Status is the current status of this channel
	Status() Status

	// Sent returns the number of bytes sent
	Sent() uint64

	// Received returns the number of bytes received
	Received() uint64

	// Message offers additional information about the current status
	Message() string

	// Vouchers returns all vouchers sent on this channel
	Vouchers() []Voucher

	// VoucherResults are results of vouchers sent on the channel
	VoucherResults() []VoucherResult

	// LastVoucher returns the last voucher sent on the channel
	LastVoucher() Voucher

	// LastVoucherResult returns the last voucher result sent on the channel
	LastVoucherResult() VoucherResult

	// ReceivedCids returns the cids received so far on the channel
	ReceivedCids() []cid.Cid

	// Queued returns the number of bytes read from the node and queued for sending
	Queued() uint64
}
```

TODO：分析解读以上代码



###### 数据传输状态

```
package datatransfer

// Status is the status of transfer for a given channel
type Status uint64

const (
	// Requested means a data transfer was requested by has not yet been approved
	Requested Status = iota

	// Ongoing means the data transfer is in progress
	Ongoing

	// TransferFinished indicates the initiator is done sending/receiving
	// data but is awaiting confirmation from the responder
	TransferFinished

	// ResponderCompleted indicates the initiator received a message from the
	// responder that it's completed
	ResponderCompleted

	// Finalizing means the responder is awaiting a final message from the initator to
	// consider the transfer done
	Finalizing

	// Completing just means we have some final cleanup for a completed request
	Completing

	// Completed means the data transfer is completed successfully
	Completed

	// Failing just means we have some final cleanup for a failed request
	Failing

	// Failed means the data transfer failed
	Failed

	// Cancelling just means we have some final cleanup for a cancelled request
	Cancelling

	// Cancelled means the data transfer ended prematurely
	Cancelled

	// InitiatorPaused means the data sender has paused the channel (only the sender can unpause this)
	InitiatorPaused

	// ResponderPaused means the data receiver has paused the channel (only the receiver can unpause this)
	ResponderPaused

	// BothPaused means both sender and receiver have paused the channel seperately (both must unpause)
	BothPaused

	// ResponderFinalizing is a unique state where the responder is awaiting a final voucher
	ResponderFinalizing

	// ResponderFinalizingTransferFinished is a unique state where the responder is awaiting a final voucher
	// and we have received all data
	ResponderFinalizingTransferFinished

	// ChannelNotFoundError means the searched for data transfer does not exist
	ChannelNotFoundError
)

// Statuses are human readable names for data transfer states
var Statuses = map[Status]string{
	// Requested means a data transfer was requested by has not yet been approved
	Requested:                           "Requested",
	Ongoing:                             "Ongoing",
	TransferFinished:                    "TransferFinished",
	ResponderCompleted:                  "ResponderCompleted",
	Finalizing:                          "Finalizing",
	Completing:                          "Completing",
	Completed:                           "Completed",
	Failing:                             "Failing",
	Failed:                              "Failed",
	Cancelling:                          "Cancelling",
	Cancelled:                           "Cancelled",
	InitiatorPaused:                     "InitiatorPaused",
	ResponderPaused:                     "ResponderPaused",
	BothPaused:                          "BothPaused",
	ResponderFinalizing:                 "ResponderFinalizing",
	ResponderFinalizingTransferFinished: "ResponderFinalizingTransferFinished",
	ChannelNotFoundError:                "ChannelNotFoundError",
}
```

TODO：分析解读以上代码



###### 数据传输管理器

Manager是所有数据传输子系统的核心接口。

```
type Manager interface {

	// Start initializes data transfer processing
	Start(ctx context.Context) error

	// OnReady registers a listener for when the data transfer comes on line
	OnReady(ReadyFunc)

	// Stop terminates all data transfers and ends processing
	Stop(ctx context.Context) error

	// RegisterVoucherType registers a validator for the given voucher type
	// will error if voucher type does not implement voucher
	// or if there is a voucher type registered with an identical identifier
	RegisterVoucherType(voucherType Voucher, validator RequestValidator) error

	// RegisterRevalidator registers a revalidator for the given voucher type
	// Note: this is the voucher type used to revalidate. It can share a name
	// with the initial validator type and CAN be the same type, or a different type.
	// The revalidator can simply be the sampe as the original request validator,
	// or a different validator that satisfies the revalidator interface.
	RegisterRevalidator(voucherType Voucher, revalidator Revalidator) error

	// RegisterVoucherResultType allows deserialization of a voucher result,
	// so that a listener can read the metadata
	RegisterVoucherResultType(resultType VoucherResult) error

	// RegisterTransportConfigurer registers the given transport configurer to be run on requests with the given voucher
	// type
	RegisterTransportConfigurer(voucherType Voucher, configurer TransportConfigurer) error

	// open a data transfer that will send data to the recipient peer and
	// transfer parts of the piece that match the selector
	OpenPushDataChannel(ctx context.Context, to peer.ID, voucher Voucher, baseCid cid.Cid, selector ipld.Node) (ChannelID, error)

	// open a data transfer that will request data from the sending peer and
	// transfer parts of the piece that match the selector
	OpenPullDataChannel(ctx context.Context, to peer.ID, voucher Voucher, baseCid cid.Cid, selector ipld.Node) (ChannelID, error)

	// send an intermediate voucher as needed when the receiver sends a request for revalidation
	SendVoucher(ctx context.Context, chid ChannelID, voucher Voucher) error

	// close an open channel (effectively a cancel)
	CloseDataTransferChannel(ctx context.Context, chid ChannelID) error

	// pause a data transfer channel (only allowed if transport supports it)
	PauseDataTransferChannel(ctx context.Context, chid ChannelID) error

	// resume a data transfer channel (only allowed if transport supports it)
	ResumeDataTransferChannel(ctx context.Context, chid ChannelID) error

	// get status of a transfer
	TransferChannelStatus(ctx context.Context, x ChannelID) Status

	// get notified when certain types of events happen
	SubscribeToEvents(subscriber Subscriber) Unsubscribe

	// get all in progress transfers
	InProgressChannels(ctx context.Context) (map[ChannelID]ChannelState, error)

	// RestartDataTransferChannel restarts an existing data transfer channel
	RestartDataTransferChannel(ctx context.Context, chid ChannelID) error
}
```

TODO：分析解读以上代码

## 数据格式和序列化

Filecoin力求尽可能少地使用数据格式，通过规范良好的序列化规则，通过简单性和互操作性（支持Filecoin协议实现之间）来提高协议安全性。

阅读更多关于cbor使用和Filecoin中int类型的设计注意事项：

https://github.com/filecoin-project/specs/issues/621

https://github.com/filecoin-project/specs/issues/615

#### 数据格式
Filecoin内存中的数据类型通常很简单。实现应该支持两种整型：Int（表示本机64位整型）和BigInt（表示任意长度），并避免处理浮点数，以最大限度地减少跨编程语言和实现的互操作性问题。

您还可以阅读更多关于Filecoin协议中随机生成的数据格式的内容。

https://spec.filecoin.io/#section-algorithms.crypto.randomness

#### 序列化
Filecoin中的数据序列化确保了在传输中和存储中内存数据序列化的一致格式。序列化对于协议安全性和跨Filecoin协议实现的互操作性至关重要，它允许Filecoin进行一致的跨节点状态更新。

Filecoin中的所有数据结构都是cbor元组编码的。也就是说，Filecoin系统中使用的任何数据结构（本规范中的结构）都应该序列化为cbor数组，其中包含与数据结构字段对应的项，这些项按照声明的顺序排列。

您可以在这里找到cbo中主要数据类型的编码结构：https://tools.ietf.org/html/rfc7049#section-2.1

举例来说，内存中的映射将表示为按预先确定的顺序列出的键和值的cbor数组。短期对序列化格式的更新将涉及适当地标记字段，以确保随着协议的发展进行适当的序列化/反序列化。

