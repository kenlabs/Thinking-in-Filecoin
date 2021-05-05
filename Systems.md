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

## 虚拟机

Filecoin区块链中的Actor相当于以太坊虚拟机中的智能合约。

Filecoin虚拟机（VM）是负责执行所有Actor代码的系统组件。Actor在Filecoin VM上的执行（即链上执行）会产生gas成本。

在Filecoin VM上执行的任何操作都会产生状态树形式的输出（将在下面讨论）。最新的状态树（State Tree）是Filecoin区块链中当前的真理来源。状态树由CID标识，CID存储在IPLD存储中。

#### 虚拟机接口

如上所述，Filecoin中的Actor类似于以太坊虚拟机中的智能合约。因此，Actor是Filecoin系统的核心组件。对Filecoin区块链当前状态的任何更改都必须通过Actor方法调用来触发。

本小节描述Actor和Filecoin虚拟机之间的接口。这意味着下面描述的大部分内容严格来说并不属于VM。相反，它是位于VM和Actors之间的接口上的逻辑。

Filecoin网络共有11种内置Actor类型，并不是所有的Actor都与VM交互。有些Actor并不对区块链的StateTree进行更改，因此，这些Actor不需要有调用VM的接口。我们将在后面的系统Actor小节中详细讨论所有系统Actor的细节。

Actor地址是一个稳定的地址，它是通过散列发送方的公钥和创建nonce而生成的（TODO：这里需要check一下代码）。Actor地址应该在链重构的过程中也是稳定的。另一方面，Actor的ID地址是一个自动递增的地址，它很紧凑，但在链重构的情况下可以更改。尽管如此，在创建Actor之后，Actor应该使用Actor地址。

```
package builtin

import (
	addr "github.com/filecoin-project/go-address"
)

// Addresses for singleton system actors.
var (
	// Distinguished AccountActor that is the source of system implicit messages.
	SystemActorAddr           = mustMakeAddress(0)
	InitActorAddr             = mustMakeAddress(1)
	RewardActorAddr           = mustMakeAddress(2)
	CronActorAddr             = mustMakeAddress(3)
	StoragePowerActorAddr     = mustMakeAddress(4)
	StorageMarketActorAddr    = mustMakeAddress(5)
	VerifiedRegistryActorAddr = mustMakeAddress(6)
	// Distinguished AccountActor that is the destination of all burnt funds.
	BurntFundsActorAddr = mustMakeAddress(99)
)

const FirstNonSingletonActorId = 100

func mustMakeAddress(id uint64) addr.Address {
	address, err := addr.NewIDAddress(id)
	if err != nil {
		panic(err)
	}
	return address
}
```

ActorState结构由Actor的余额（就该Actor持有的通证而言）以及一组用于查询、检查链状态并与之交互的状态方法组成。

#### 状态树

状态树是应用于Filecoin区块链上任何操作的执行输出。链上(即VM)状态数据结构是一个映射(以哈希数组映射Trie - HAMT的形式)，这个映射将地址绑定到Actor状态。当前状态树函数由VM在每次actor方法调用时调用。

```
type StateTree struct {
	root        adt.Map
	version     types.StateTreeVersion
	info        cid.Cid
	Store       cbor.IpldStore
	lookupIDFun func(address.Address) (address.Address, error)

	snaps *stateSnaps
}
```

#### 虚拟机消息——Actor方法调用

消息是两个Actor之间的通信单元，因此消息是Actor状态变化的根本原因。

一个消息由以下2部分组成：

* 从发送方转移到接收方的通证数量，以及
* 在接收方调用的方法以及附带的参数（可选的/需要的地方）。

Actor的嵌套调用：在Actor处理接收到的消息时，其自身的代码可以向其他Actor发送额外的消息。消息的处理是同步的，也就是说，Actor在恢复控制之前等待已发送消息的完成。

消息的处理消耗了计算和存储资源，这两种资源都是以Gas为单位计价的。消息的 gas limit为处理该消息所需的计算提供了一个上限。消息的发送方以确定的gas价格为消息执行（包括所有嵌套消息）所消耗的Gas支付费用。区块生产者选择在区块中包含哪些消息，并根据每个消息的Gas价格和资源消耗量获得奖励，从而形成一个市场。

###### 消息语法验证

语法无效的消息不能被传输、保留在消息池中或包含在区块中。如果接收到无效消息，则应删除该消息，并不再进一步传播。

当消息单独传输时（第一次发送到区块链，在包含在区块中之前），消息被打包为`SignedMessage`，不管使用的是什么签名方案。有效的已签名消息的总序列化大小不得大于`message.MessageMaxSize`。

```
type SignedMessage struct {
	Message   Message
	Signature crypto.Signature
}
```

一个语法有效的未签名消息（UnsignedMessage） 需要满足:

* 有一个格式正确的非空的To地址，
* 有一个格式正确的非空的From地址，
* 消息转移的通证数额（Value）不小于零且不大于总通证供应量(2e9 * 1e18)，并且
* GasPrice不能是负值，
* GasLimit至少等于与消息的序列化字节相关联的Gas消耗，
* GasLimit不能大于区块的gas limit网络参数。

```

type Message struct {
	// Version of this message (has to be non-negative)
	Version uint64

	// Address of the receiving actor.
	To   address.Address
	// Address of the sending actor.
	From address.Address

	CallSeqNum uint64

	// Value to transfer from sender's to receiver's balance.
	Value BigInt

	// GasPrice is a Gas-to-FIL cost
	GasPrice BigInt
	// Maximum Gas to be spent on the processing of this message
	GasLimit int64

	// Optional method to invoke on receiver, zero for a plain value transfer.
	Method abi.MethodNum
	//Serialized parameters to the method.
	Params []byte
}
```

这里有几个方法（函数）能够从Message结构中提取信息，例如发送方和接收方地址、要传递的通证数量（值）、执行消息所需的通证和消息的CID。

假定Messages最终应该包含在区块中并添加到区块链，则应该根据消息的发送方和接收方来检查消息的有效性，以及转移的通证数量（值，应该是非负的，并且总是小于总循环供应），Gas价格（同样应该是非负的）和BlockGasLimit不应该大于整个区块的gas limit（TODO：这里需要解释的更清楚准确）。



###### 消息语义验证

语义验证是指对消息本身之外的信息进行验证。

语义上有效的SignedMessage必须携带一个签名，该签名验证消息是否已经用From地址所标识的帐户Actor的公钥签名。注意，当From地址是ID-address时，必须从区块标识的父状态发送帐户Actor的状态中查找公钥（TODO：这里需要解释更清楚，可以画一个图来解释）。

注意：发送Actor必须存在于打包消息的区块所标识的父状态中。这意味着单个区块不能包含创建新帐户Actor的消息和来自该同一Actor的普通消息。来自该Actor的第一个消息必须等待到下一个epoch。消息池可能会排除尚未存在于链状态中的Actor的消息。

对于可能导致包含该消息的区块无效的消息，没有做进一步的语义验证。每个语法上有效且正确签名的消息都可以包含在一个区块中，并在执行时产生一个收据。

`MessageReceipt sturct`包括以下内容：

```
type MessageReceipt struct {
	ExitCode exitcode.ExitCode
	Return   []byte
	GasUsed  int64
}
```

然而，消息可能无法执行到完成，在这种情况下，它将不会触发状态更改。

这种“没有做进一步语义验证”策略的原因是，作为tipset的一部分，在消息执行之前，无法知道消息执行之后的最终状态。一个区块生产者不知道在tipset中是否有另一个区块在它之前，从而改变了该区块的消息本来将要修改的状态。

```
package types

import (
	"bytes"
	"encoding/json"
	"fmt"

	"github.com/filecoin-project/go-state-types/abi"
	"github.com/filecoin-project/go-state-types/big"
	"github.com/filecoin-project/lotus/build"
	block "github.com/ipfs/go-block-format"
	"github.com/ipfs/go-cid"
	xerrors "golang.org/x/xerrors"

	"github.com/filecoin-project/go-address"
)

const MessageVersion = 0

type ChainMsg interface {
	Cid() cid.Cid
	VMMessage() *Message
	ToStorageBlock() (block.Block, error)
	// FIXME: This is the *message* length, this name is misleading.
	ChainLength() int
}

type Message struct {
	Version uint64

	To   address.Address
	From address.Address

	Nonce uint64

	Value abi.TokenAmount

	GasLimit   int64
	GasFeeCap  abi.TokenAmount
	GasPremium abi.TokenAmount

	Method abi.MethodNum
	Params []byte
}

func (m *Message) Caller() address.Address {
	return m.From
}

func (m *Message) Receiver() address.Address {
	return m.To
}

func (m *Message) ValueReceived() abi.TokenAmount {
	return m.Value
}

func DecodeMessage(b []byte) (*Message, error) {
	var msg Message
	if err := msg.UnmarshalCBOR(bytes.NewReader(b)); err != nil {
		return nil, err
	}

	if msg.Version != MessageVersion {
		return nil, fmt.Errorf("decoded message had incorrect version (%d)", msg.Version)
	}

	return &msg, nil
}

func (m *Message) Serialize() ([]byte, error) {
	buf := new(bytes.Buffer)
	if err := m.MarshalCBOR(buf); err != nil {
		return nil, err
	}
	return buf.Bytes(), nil
}

func (m *Message) ChainLength() int {
	ser, err := m.Serialize()
	if err != nil {
		panic(err)
	}
	return len(ser)
}

func (m *Message) ToStorageBlock() (block.Block, error) {
	data, err := m.Serialize()
	if err != nil {
		return nil, err
	}

	c, err := abi.CidBuilder.Sum(data)
	if err != nil {
		return nil, err
	}

	return block.NewBlockWithCid(data, c)
}

func (m *Message) Cid() cid.Cid {
	b, err := m.ToStorageBlock()
	if err != nil {
		panic(fmt.Sprintf("failed to marshal message: %s", err)) // I think this is maybe sketchy, what happens if we try to serialize a message with an undefined address in it?
	}

	return b.Cid()
}

type mCid struct {
	*RawMessage
	CID cid.Cid
}

type RawMessage Message

func (m *Message) MarshalJSON() ([]byte, error) {
	return json.Marshal(&mCid{
		RawMessage: (*RawMessage)(m),
		CID:        m.Cid(),
	})
}

func (m *Message) RequiredFunds() BigInt {
	return BigMul(m.GasFeeCap, NewInt(uint64(m.GasLimit)))
}

func (m *Message) VMMessage() *Message {
	return m
}

func (m *Message) Equals(o *Message) bool {
	return m.Cid() == o.Cid()
}

func (m *Message) EqualCall(o *Message) bool {
	m1 := *m
	m2 := *o

	m1.GasLimit, m2.GasLimit = 0, 0
	m1.GasFeeCap, m2.GasFeeCap = big.Zero(), big.Zero()
	m1.GasPremium, m2.GasPremium = big.Zero(), big.Zero()

	return (&m1).Equals(&m2)
}

func (m *Message) ValidForBlockInclusion(minGas int64) error {
	if m.Version != 0 {
		return xerrors.New("'Version' unsupported")
	}

	if m.To == address.Undef {
		return xerrors.New("'To' address cannot be empty")
	}

	if m.From == address.Undef {
		return xerrors.New("'From' address cannot be empty")
	}

	if m.Value.Int == nil {
		return xerrors.New("'Value' cannot be nil")
	}

	if m.Value.LessThan(big.Zero()) {
		return xerrors.New("'Value' field cannot be negative")
	}

	if m.Value.GreaterThan(TotalFilecoinInt) {
		return xerrors.New("'Value' field cannot be greater than total filecoin supply")
	}

	if m.GasFeeCap.Int == nil {
		return xerrors.New("'GasFeeCap' cannot be nil")
	}

	if m.GasFeeCap.LessThan(big.Zero()) {
		return xerrors.New("'GasFeeCap' field cannot be negative")
	}

	if m.GasPremium.Int == nil {
		return xerrors.New("'GasPremium' cannot be nil")
	}

	if m.GasPremium.LessThan(big.Zero()) {
		return xerrors.New("'GasPremium' field cannot be negative")
	}

	if m.GasPremium.GreaterThan(m.GasFeeCap) {
		return xerrors.New("'GasFeeCap' less than 'GasPremium'")
	}

	if m.GasLimit > build.BlockGasLimit {
		return xerrors.New("'GasLimit' field cannot be greater than a block's gas limit")
	}

	// since prices might vary with time, this is technically semantic validation
	if m.GasLimit < minGas {
		return xerrors.Errorf("'GasLimit' field cannot be less than the cost of storing a message on chain %d < %d", m.GasLimit, minGas)
	}

	return nil
}

const TestGasLimit = 100e6
```

#### 虚拟机环境

* 收据
  * `MessageReceipt`包含顶层消息执行的结果。每个语法上有效且正确签名的消息都可以包含在一个区块中，并在执行时产生一个收据。
  * 一个语法上有效的收据有:
    * 一个非负ExitCode,
    * 仅当退出码为零时，返回值为非空
    * 一个非负GasUsed。

```
type MessageReceipt struct {
	ExitCode exitcode.ExitCode
	Return   []byte
	GasUsed  int64
}
```

* 虚拟机/运行时 Actor接口
  Actors接口的实现可以在这里找到： https://github.com/filecoin-project/specs-actors/blob/master/actors/runtime/runtime.go
* 虚拟机/运行时实现
  可以在这里找到Filecoin虚拟机运行时的Lotus实现：https://github.com/filecoin-project/lotus/blob/master/chain/vm/runtime.go
* 退出代码
  有一些公共运行时退出代码是由不同的Actor共享的。它们的定义可以在这里找到：https://github.com/filecoin-project/go-state-types/blob/master/exitcode/common.go

#### Gas 费用

与许多传统区块链的情况一样，Gas是一个度量单位，用来衡量一个链上消息执行消耗了多少存储或计算资源。在较高的层次上，Gas费用的工作机制如下：消息发送者指定他们愿意支付的最大Gas费用，以便他们的消息被执行并包含在一个区块中。这是根据Gas的总单位数（GasLimit，通常预计会高于实际使用的Gas数量：GasUsed）和每单位气体的价格（或费用）来规定的（GasFeeCap）。

传统上，GasUsed * GasFeeCap会作为奖励给区块生产者。该乘积的结果被视为消息被打包的优先级费用，也就是说，消息按乘积降序排列，而GasUsed * GasFeeCap最高的消息将会被优先打包，因为它们会给矿工带来更多利润。

然而，据观察，这种按照GasUsed * GasFee支付的策略对区块生产者来说是有问题的，原因有几个。首先，一个区块生产者可能会免费打包一个非常昂贵的消息（昂贵是根据所需的区块链资源而言），在这种情况下区块链本身需要承担成本。其次，消息发送者可以为低成本的消息设置任意高的价格（同样，昂贵是根据所需的区块链资源而言)，从而导致DoS攻击的可能。

为了克服这种情况，Filecoin区块链定义了一个BaseFee，每个消息都会燃烧BaseFee。其基本原理是，鉴于Gas是链上资源消耗的一种衡量标准，与其奖励矿工，不如燃烧Gas费。这样就可以避免矿工像上述描述那样操纵Gas费用。BaseFee是动态的，根据网络拥塞状况而自动调整。这一事实使网络对垃圾消息攻击具有弹性。考虑到垃圾消息攻击会导致网络负载增加，从而导致BaseFee的增加，攻击者因此不可能在长时间内维持全区块的垃圾消息。

最后，GasPremium是发送者设置的优先打包费用，用于激励矿工选择最赚钱的消息。换句话说，如果消息发送者希望它的消息被更快地包含，他们可以设置一个更高的GasPremium。



###### 参数介绍

* GasUsed度量执行消息而消耗的资源数量(或Gas单位)。GasUsed的每一个单位是用attoFIL来测量的，因此，GasUsed是一个数字，表示能源消耗的单位。GasUsed与消息是否正确执行无关。
* BaseFee是每次执行消息时每单位燃烧（发送到一个不可恢复的地址）的Gas设定价格（以attoFIL/Gas单位计量）。BaseFee是动态的，会根据当前的网络拥塞状况进行调整。例如，当网络使用超过5B的Gas限制使用时，BaseFee会增加，而网络使用低于5BGas限制时则相反。应用于每个区块的BaseFee应该包含在区块本身中。应该可以从区块链的链首获取当前BaseFee的值。BaseFee应用于每单位GasUsed，因此，消息燃烧的Gas总量是BaseFee * GasUsed。注意，BaseFee是针对所有消息的，但是它的值对于同一区块中的所有消息都是相同的。
* GasLimit是Gas单位的度量，并有由消息发送者设置。它对一个消息的执行在链上可以消耗的Gas的数量（即Gas的单位数）施加了一个硬性限制。消息对其触发的每个基本操作都消耗Gas，Gas耗尽的消息将会执行失败。当消息失败时，由于该消息执行而发生的对状态的所有修改都将恢复到之前的状态。不管消息执行是否成功，区块生产者都将获得执行消息所消耗资源的奖励（参见下面的GasPremium）。
* GasFeeCap是消息发送者愿意支付的每单位Gas（以attoFIL/Gas单位计量）的最高价格。与GasLimit一起，GasFeeCap设置发送者将为一条消息支付的最大FIL金额：发送者保证一条消息将永远不会超过GasLimit * GasFeeCap attoFIL（并不包括任何消息包含的收件人的Premium)。
* GasPremium是每单位Gas的价格（以attoFIL/Gas单位计量），消息发送者愿意支付高于BaseFee 的价格，并“提示”区块生产者将该消息打包在块中。一个消息通常赚取它的矿工GasLimit * GasPremium attoFIL，其中有效的GasPremium = GasFeeCap - BaseFee（TODO：这一句里需要解释的更加清楚）。请注意，GasPremium应用于GasLimit，而不是GasUsed，目的是让矿工的消息选择更加直接。

ComputeGasOverestimationBurn计算需要支付的Gas总量和以及需要燃烧的Gas总量（支付，燃烧)

```
func ComputeGasOverestimationBurn(gasUsed, gasLimit int64) (int64, int64) {
	if gasUsed == 0 {
		return 0, gasLimit
	}

	// over = gasLimit/gasUsed - 1 - 0.1
	// over = min(over, 1)
	// gasToBurn = (gasLimit - gasUsed) * over

	// so to factor out division from `over`
	// over*gasUsed = min(gasLimit - (11*gasUsed)/10, gasUsed)
	// gasToBurn = ((gasLimit - gasUsed)*over*gasUsed) / gasUsed
	over := gasLimit - (gasOveruseNum*gasUsed)/gasOveruseDenom
	if over < 0 {
		return gasLimit - gasUsed, 0
	}

	// if we want sharper scaling it goes here:
	// over *= 2

	if over > gasUsed {
		over = gasUsed
	}

	// needs bigint, as it overflows in pathological case gasLimit > 2^32 gasUsed = gasLimit / 2
	gasToBurn := big.NewInt(gasLimit - gasUsed)
	gasToBurn = big.Mul(gasToBurn, big.NewInt(over))
	gasToBurn = big.Div(gasToBurn, big.NewInt(gasUsed))

	return gasLimit - gasUsed - gasToBurn.Int64(), gasToBurn.Int64()
}
```

ComputeNextBaseFee

```
func ComputeNextBaseFee(baseFee types.BigInt, gasLimitUsed int64, noOfBlocks int, epoch abi.ChainEpoch) types.BigInt {
	// deta := gasLimitUsed/noOfBlocks - build.BlockGasTarget
	// change := baseFee * deta / BlockGasTarget
	// nextBaseFee = baseFee + change
	// nextBaseFee = max(nextBaseFee, build.MinimumBaseFee)

	var delta int64
	if epoch > build.UpgradeSmokeHeight {
		delta = gasLimitUsed / int64(noOfBlocks)
		delta -= build.BlockGasTarget
	} else {
		delta = build.PackingEfficiencyDenom * gasLimitUsed / (int64(noOfBlocks) * build.PackingEfficiencyNum)
		delta -= build.BlockGasTarget
	}

	// cap change at 12.5% (BaseFeeMaxChangeDenom) by capping delta
	if delta > build.BlockGasTarget {
		delta = build.BlockGasTarget
	}
	if delta < -build.BlockGasTarget {
		delta = -build.BlockGasTarget
	}

	change := big.Mul(baseFee, big.NewInt(delta))
	change = big.Div(change, big.NewInt(build.BlockGasTarget))
	change = big.Div(change, big.NewInt(build.BaseFeeMaxChangeDenom))

	nextBaseFee := big.Add(baseFee, change)
	if big.Cmp(nextBaseFee, big.NewInt(build.MinimumBaseFee)) < 0 {
		nextBaseFee = big.NewInt(build.MinimumBaseFee)
	}
	return nextBaseFee
}
```

###### 注释

* GasFeeCap应该始终高于网络的BaseFee。如果消息的GasFeeCap低于BaseFee，那么差距的部分将来自矿工（作为惩罚）。这个惩罚应用于矿工，因为他们选择的消息支付低于网络BaseFee（即，没有覆盖网络成本）。但是，如果同一个发送者在消息池中有另一条消息，其GasFeeCap比BaseFee大得多，则矿工可能希望选择GasFeeCap比BaseFee小得多的消息。回想一下，如果存在多个消息，那么区块生产者应该从消息池中选择发送者的所有消息。其理由是，增加的费用将弥补第一个消息的损失（TODO：这里需要更加详细的解释）。
* 如果BaseFee + GasPremium > GasFeeCap，那么矿工可能不会获得整个GasLimit * GasPremium作为奖励（TODO：这里需要更加详细的解释）。
* 一个信息被严格限制在支出不超过GasFeeCap * GasLimit。从这个金额开始，首先支付（烧毁）网络BaseFee。扣除BaseFee部分之后，剩余的金额作为奖励给予矿工，奖励的上限为GasLimit * GasPremium（TODO：这里需要更加详细的解释）。
* 一条消息因为耗尽Gas而失败，并带有“耗尽Gas”退出代码。GasUsed * BaseFee仍将被燃烧（在这种情况下GasUsed = GasLimit），矿工仍将获得GasLimit * GasPremium奖励。这里假设GasFeeCap > BaseFee + GasPremium（TODO：这里需要更加详细的解释）。
* GasFeeCap的低值可能会导致消息被卡在消息池中，因为就利润而言，它没有足够的吸引力让任何矿工选择它们，并将它们包含在区块中。当这种情况发生时，会有一个更新GasFeeCap的过程，从而使消息对矿工更有吸引力。发送方可以推动一个新的消息到消息池（默认情况下，将传播到其他矿工的消息池），其中（TODO：这里需要更加详细的解释）：
  * i)新旧信息的标识符是相同的（例如，相同的Nonce）
  * ii) GasPremium更新，基于前一个值至少增加25%。



#### 系统Actor

总共有11个内置的系统Actor，但不是所有的Actor都与VM交互。每个Actor都由一个代码ID (CID)标识。

VM处理需要两个系统Actor:

* 初始化新Actor并记录网络名称的InitActor，以及
* CronActor，一个在每个epoch运行关键函数的调度器actor。

还有另外两个Actor与VM交互：

* 负责用户帐户（非单例）的AccountActor，以及
* RewardActor用于区块奖励和通证归属（单例）。

其余7个不直接与VM交互的内置系统Actor如下:

* StorageMarketActor：负责管理存储和检索交易：https://github.com/filecoin-project/specs-actors/blob/master/actors/builtin/market/market_actor.go
* StorageMinerActor：负责处理存储挖矿操作和收集证明的Actor ：https://github.com/filecoin-project/specs-actors/blob/master/actors/builtin/miner/miner_actor.go
* MultisigActor(或Multi-Signature Wallet Actor)：负责处理涉及Filecoin钱包的操作：https://github.com/filecoin-project/specs-actors/blob/master/actors/builtin/multisig/multisig_actor.go
* PaymentChannelActor:负责支付通道相关资金的建立和结算：https://github.com/filecoin-project/specs-actors/blob/master/actors/builtin/paych/paych_actor.go
* StoragePowerActor：负责跟踪每个存储矿机分配的存储算力：https://github.com/filecoin-project/specs-actors/blob/master/actors/builtin/power/power_actor.go
* VerifiedRegistryActor：负责管理已验证的客户：https://github.com/filecoin-project/specs-actors/blob/master/actors/builtin/verifreg/verified_registry_actor.go
* SystemActor：一般系统Actor ：https://github.com/filecoin-project/specs-actors/blob/master/actors/builtin/system/system_actor.go



###### CronActor

CronActor内置在创世状态中，它的分派表调用StoragePowerActor和StorageMarketActor来维护内部状态和处理延迟事件。原则上，它可以在网络升级后调用其他参与者。

```
package cron

import (
	"github.com/filecoin-project/go-state-types/abi"
	"github.com/filecoin-project/go-state-types/cbor"
	cron0 "github.com/filecoin-project/specs-actors/actors/builtin/cron"
	"github.com/ipfs/go-cid"

	"github.com/filecoin-project/specs-actors/v2/actors/builtin"
	"github.com/filecoin-project/specs-actors/v2/actors/runtime"
)

// The cron actor is a built-in singleton that sends messages to other registered actors at the end of each epoch.
type Actor struct{}

func (a Actor) Exports() []interface{} {
	return []interface{}{
		builtin.MethodConstructor: a.Constructor,
		2:                         a.EpochTick,
	}
}

func (a Actor) Code() cid.Cid {
	return builtin.CronActorCodeID
}

func (a Actor) IsSingleton() bool {
	return true
}

func (a Actor) State() cbor.Er {
	return new(State)
}

var _ runtime.VMActor = Actor{}

//type ConstructorParams struct {
//	Entries []Entry
//}
type ConstructorParams = cron0.ConstructorParams

type EntryParam = cron0.Entry

func (a Actor) Constructor(rt runtime.Runtime, params *ConstructorParams) *abi.EmptyValue {
	rt.ValidateImmediateCallerIs(builtin.SystemActorAddr)
	entries := make([]Entry, len(params.Entries))
	for i, e := range params.Entries {
		entries[i] = Entry(e) // Identical
	}
	rt.StateCreate(ConstructState(entries))
	return nil
}

// Invoked by the system after all other messages in the epoch have been processed.
func (a Actor) EpochTick(rt runtime.Runtime, _ *abi.EmptyValue) *abi.EmptyValue {
	rt.ValidateImmediateCallerIs(builtin.SystemActorAddr)

	var st State
	rt.StateReadonly(&st)
	for _, entry := range st.Entries {
		_ = rt.Send(entry.Receiver, entry.MethodNum, nil, abi.NewTokenAmount(0), &builtin.Discard{})
		// Any error and return value are ignored.
	}

	return nil
}
```



###### InitActor

InitActor有权创建新的Actor，例如，那些进入系统的Actor。它维护一个表，将公钥和临时Actor地址解析为它们的规范ID地址。无效的cid不应该被提交到状态树。

请注意，在链重构的情况下，规范ID地址不会持久存在。Actor地址或公钥在链重构后仍然存在。

（TODO：这里需要结合代码，讲述的更加准确和详细）

```
package init

import (
	addr "github.com/filecoin-project/go-address"
	"github.com/filecoin-project/go-state-types/abi"
	"github.com/filecoin-project/go-state-types/cbor"
	"github.com/filecoin-project/go-state-types/exitcode"
	init0 "github.com/filecoin-project/specs-actors/actors/builtin/init"
	cid "github.com/ipfs/go-cid"

	"github.com/filecoin-project/specs-actors/v2/actors/builtin"
	"github.com/filecoin-project/specs-actors/v2/actors/runtime"
	autil "github.com/filecoin-project/specs-actors/v2/actors/util"
	"github.com/filecoin-project/specs-actors/v2/actors/util/adt"
)

// The init actor uniquely has the power to create new actors.
// It maintains a table resolving pubkey and temporary actor addresses to the canonical ID-addresses.
type Actor struct{}

func (a Actor) Exports() []interface{} {
	return []interface{}{
		builtin.MethodConstructor: a.Constructor,
		2:                         a.Exec,
	}
}

func (a Actor) Code() cid.Cid {
	return builtin.InitActorCodeID
}

func (a Actor) IsSingleton() bool {
	return true
}

func (a Actor) State() cbor.Er { return new(State) }

var _ runtime.VMActor = Actor{}

//type ConstructorParams struct {
//	NetworkName string
//}
type ConstructorParams = init0.ConstructorParams

func (a Actor) Constructor(rt runtime.Runtime, params *ConstructorParams) *abi.EmptyValue {
	rt.ValidateImmediateCallerIs(builtin.SystemActorAddr)
	emptyMap, err := adt.MakeEmptyMap(adt.AsStore(rt)).Root()
	builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to construct state")

	st := ConstructState(emptyMap, params.NetworkName)
	rt.StateCreate(st)
	return nil
}

//type ExecParams struct {
//	CodeCID           cid.Cid `checked:"true"` // invalid CIDs won't get committed to the state tree
//	ConstructorParams []byte
//}
type ExecParams = init0.ExecParams

//type ExecReturn struct {
//	IDAddress     addr.Address // The canonical ID-based address for the actor.
//	RobustAddress addr.Address // A more expensive but re-org-safe address for the newly created actor.
//}
type ExecReturn = init0.ExecReturn

func (a Actor) Exec(rt runtime.Runtime, params *ExecParams) *ExecReturn {
	rt.ValidateImmediateCallerAcceptAny()
	callerCodeCID, ok := rt.GetActorCodeCID(rt.Caller())
	autil.AssertMsg(ok, "no code for actor at %s", rt.Caller())
	if !canExec(callerCodeCID, params.CodeCID) {
		rt.Abortf(exitcode.ErrForbidden, "caller type %v cannot exec actor type %v", callerCodeCID, params.CodeCID)
	}

	// Compute a re-org-stable address.
	// This address exists for use by messages coming from outside the system, in order to
	// stably address the newly created actor even if a chain re-org causes it to end up with
	// a different ID.
	uniqueAddress := rt.NewActorAddress()

	// Allocate an ID for this actor.
	// Store mapping of pubkey or actor address to actor ID
	var st State
	var idAddr addr.Address
	rt.StateTransaction(&st, func() {
		var err error
		idAddr, err = st.MapAddressToNewID(adt.AsStore(rt), uniqueAddress)
		builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to allocate ID address")
	})

	// Create an empty actor.
	rt.CreateActor(params.CodeCID, idAddr)

	// Invoke constructor.
	code := rt.Send(idAddr, builtin.MethodConstructor, builtin.CBORBytes(params.ConstructorParams), rt.ValueReceived(), &builtin.Discard{})
	builtin.RequireSuccess(rt, code, "constructor failed")

	return &ExecReturn{IDAddress: idAddr, RobustAddress: uniqueAddress}
}

func canExec(callerCodeID cid.Cid, execCodeID cid.Cid) bool {
	switch execCodeID {
	case builtin.StorageMinerActorCodeID:
		if callerCodeID == builtin.StoragePowerActorCodeID {
			return true
		}
		return false
	case builtin.PaymentChannelActorCodeID, builtin.MultisigActorCodeID:
		return true
	default:
		return false
	}
}
```



###### RewardActor

RewardActor是保存未铸造的Filecoin通证的地方。Actor将奖励直接分配给miner Actor，在miner Actor中通证被锁定以进行归属。当前epoch使用的奖励值在epoch结束时通过cron Actor周期更新。

（TODO：这里需要结合代码，讲述的更加准确和详细）

```
package reward

import (
	"github.com/filecoin-project/go-state-types/abi"
	"github.com/filecoin-project/go-state-types/big"
	"github.com/filecoin-project/go-state-types/cbor"
	"github.com/filecoin-project/go-state-types/exitcode"
	rtt "github.com/filecoin-project/go-state-types/rt"
	reward0 "github.com/filecoin-project/specs-actors/actors/builtin/reward"
	"github.com/ipfs/go-cid"

	"github.com/filecoin-project/specs-actors/v2/actors/builtin"
	"github.com/filecoin-project/specs-actors/v2/actors/runtime"
	. "github.com/filecoin-project/specs-actors/v2/actors/util"
	"github.com/filecoin-project/specs-actors/v2/actors/util/smoothing"
)

// PenaltyMultiplier is the factor miner penaltys are scaled up by
const PenaltyMultiplier = 3

type Actor struct{}

func (a Actor) Exports() []interface{} {
	return []interface{}{
		builtin.MethodConstructor: a.Constructor,
		2:                         a.AwardBlockReward,
		3:                         a.ThisEpochReward,
		4:                         a.UpdateNetworkKPI,
	}
}

func (a Actor) Code() cid.Cid {
	return builtin.RewardActorCodeID
}

func (a Actor) IsSingleton() bool {
	return true
}

func (a Actor) State() cbor.Er {
	return new(State)
}

var _ runtime.VMActor = Actor{}

func (a Actor) Constructor(rt runtime.Runtime, currRealizedPower *abi.StoragePower) *abi.EmptyValue {
	rt.ValidateImmediateCallerIs(builtin.SystemActorAddr)

	if currRealizedPower == nil {
		rt.Abortf(exitcode.ErrIllegalArgument, "argument should not be nil")
		return nil // linter does not understand abort exiting
	}
	st := ConstructState(*currRealizedPower)
	rt.StateCreate(st)
	return nil
}

//type AwardBlockRewardParams struct {
//	Miner     address.Address
//	Penalty   abi.TokenAmount // penalty for including bad messages in a block, >= 0
//	GasReward abi.TokenAmount // gas reward from all gas fees in a block, >= 0
//	WinCount  int64           // number of reward units won, > 0
//}
type AwardBlockRewardParams = reward0.AwardBlockRewardParams

// Awards a reward to a block producer.
// This method is called only by the system actor, implicitly, as the last message in the evaluation of a block.
// The system actor thus computes the parameters and attached value.
//
// The reward includes two components:
// - the epoch block reward, computed and paid from the reward actor's balance,
// - the block gas reward, expected to be transferred to the reward actor with this invocation.
//
// The reward is reduced before the residual is credited to the block producer, by:
// - a penalty amount, provided as a parameter, which is burnt,
func (a Actor) AwardBlockReward(rt runtime.Runtime, params *AwardBlockRewardParams) *abi.EmptyValue {
	rt.ValidateImmediateCallerIs(builtin.SystemActorAddr)
	priorBalance := rt.CurrentBalance()
	if params.Penalty.LessThan(big.Zero()) {
		rt.Abortf(exitcode.ErrIllegalArgument, "negative penalty %v", params.Penalty)
	}
	if params.GasReward.LessThan(big.Zero()) {
		rt.Abortf(exitcode.ErrIllegalArgument, "negative gas reward %v", params.GasReward)
	}
	if priorBalance.LessThan(params.GasReward) {
		rt.Abortf(exitcode.ErrIllegalState, "actor current balance %v insufficient to pay gas reward %v",
			priorBalance, params.GasReward)
	}
	if params.WinCount <= 0 {
		rt.Abortf(exitcode.ErrIllegalArgument, "invalid win count %d", params.WinCount)
	}

	minerAddr, ok := rt.ResolveAddress(params.Miner)
	if !ok {
		rt.Abortf(exitcode.ErrNotFound, "failed to resolve given owner address")
	}
	// The miner penalty is scaled up by a factor of PenaltyMultiplier
	penalty := big.Mul(big.NewInt(PenaltyMultiplier), params.Penalty)
	totalReward := big.Zero()
	var st State
	rt.StateTransaction(&st, func() {
		blockReward := big.Mul(st.ThisEpochReward, big.NewInt(params.WinCount))
		blockReward = big.Div(blockReward, big.NewInt(builtin.ExpectedLeadersPerEpoch))
		totalReward = big.Add(blockReward, params.GasReward)
		currBalance := rt.CurrentBalance()
		if totalReward.GreaterThan(currBalance) {
			rt.Log(rtt.WARN, "reward actor balance %d below totalReward expected %d, paying out rest of balance", currBalance, totalReward)
			totalReward = currBalance

			blockReward = big.Sub(totalReward, params.GasReward)
			// Since we have already asserted the balance is greater than gas reward blockReward is >= 0
			AssertMsg(blockReward.GreaterThanEqual(big.Zero()), "programming error, block reward is %v below zero", blockReward)
		}
		st.TotalStoragePowerReward = big.Add(st.TotalStoragePowerReward, blockReward)
	})

	AssertMsg(totalReward.LessThanEqual(priorBalance), "reward %v exceeds balance %v", totalReward, priorBalance)

	// if this fails, we can assume the miner is responsible and avoid failing here.
	rewardParams := builtin.ApplyRewardParams{
		Reward:  totalReward,
		Penalty: penalty,
	}
	code := rt.Send(minerAddr, builtin.MethodsMiner.ApplyRewards, &rewardParams, totalReward, &builtin.Discard{})
	if !code.IsSuccess() {
		rt.Log(rtt.ERROR, "failed to send ApplyRewards call to the miner actor with funds: %v, code: %v", totalReward, code)
		code := rt.Send(builtin.BurntFundsActorAddr, builtin.MethodSend, nil, totalReward, &builtin.Discard{})
		if !code.IsSuccess() {
			rt.Log(rtt.ERROR, "failed to send unsent reward to the burnt funds actor, code: %v", code)
		}
	}

	return nil
}

// Changed since v0:
// - removed ThisEpochReward (unsmoothed)
type ThisEpochRewardReturn struct {
	ThisEpochRewardSmoothed smoothing.FilterEstimate
	ThisEpochBaselinePower  abi.StoragePower
}

// The award value used for the current epoch, updated at the end of an epoch
// through cron tick.  In the case previous epochs were null blocks this
// is the reward value as calculated at the last non-null epoch.
func (a Actor) ThisEpochReward(rt runtime.Runtime, _ *abi.EmptyValue) *ThisEpochRewardReturn {
	rt.ValidateImmediateCallerAcceptAny()

	var st State
	rt.StateReadonly(&st)
	return &ThisEpochRewardReturn{
		ThisEpochRewardSmoothed: st.ThisEpochRewardSmoothed,
		ThisEpochBaselinePower:  st.ThisEpochBaselinePower,
	}
}

// Called at the end of each epoch by the power actor (in turn by its cron hook).
// This is only invoked for non-empty tipsets, but catches up any number of null
// epochs to compute the next epoch reward.
func (a Actor) UpdateNetworkKPI(rt runtime.Runtime, currRealizedPower *abi.StoragePower) *abi.EmptyValue {
	rt.ValidateImmediateCallerIs(builtin.StoragePowerActorAddr)
	if currRealizedPower == nil {
		rt.Abortf(exitcode.ErrIllegalArgument, "arugment should not be nil")
	}

	var st State
	rt.StateTransaction(&st, func() {
		prev := st.Epoch
		// if there were null runs catch up the computation until
		// st.Epoch == rt.CurrEpoch()
		for st.Epoch < rt.CurrEpoch() {
			// Update to next epoch to process null rounds
			st.updateToNextEpoch(*currRealizedPower)
		}

		st.updateToNextEpochWithReward(*currRealizedPower)
		// only update smoothed estimates after updating reward and epoch
		st.updateSmoothedEstimates(st.Epoch - prev)
	})
	return nil
}
```



###### AccountActor

AccountActor负责用户帐户。AccountActor不是由InitActor创建的，但是它们的构造函数是由系统调用的。AccountActor是通过将消息发送到一个公钥样式的地址来创建的。地址必须是BLS或SECP，否则应该有退出错误。AccountActor正在用新的Actor地址更新状态树。

（TODO：这里需要结合InitActor，讲述的更加准确和详细）

```
package account

import (
	addr "github.com/filecoin-project/go-address"
	"github.com/filecoin-project/go-state-types/abi"
	"github.com/filecoin-project/go-state-types/cbor"
	"github.com/filecoin-project/go-state-types/exitcode"
	"github.com/ipfs/go-cid"

	"github.com/filecoin-project/specs-actors/v2/actors/builtin"
	"github.com/filecoin-project/specs-actors/v2/actors/runtime"
)

type Actor struct{}

func (a Actor) Exports() []interface{} {
	return []interface{}{
		1: a.Constructor,
		2: a.PubkeyAddress,
	}
}

func (a Actor) Code() cid.Cid {
	return builtin.AccountActorCodeID
}

func (a Actor) State() cbor.Er {
	return new(State)
}

var _ runtime.VMActor = Actor{}

type State struct {
	Address addr.Address
}

func (a Actor) Constructor(rt runtime.Runtime, address *addr.Address) *abi.EmptyValue {
	// Account actors are created implicitly by sending a message to a pubkey-style address.
	// This constructor is not invoked by the InitActor, but by the system.
	rt.ValidateImmediateCallerIs(builtin.SystemActorAddr)
	switch address.Protocol() {
	case addr.SECP256K1:
	case addr.BLS:
		break // ok
	default:
		rt.Abortf(exitcode.ErrIllegalArgument, "address must use BLS or SECP protocol, got %v", address.Protocol())
	}
	st := State{Address: *address}
	rt.StateCreate(&st)
	return nil
}

// Fetches the pubkey-type address from this actor.
func (a Actor) PubkeyAddress(rt runtime.Runtime, _ *abi.EmptyValue) *addr.Address {
	rt.ValidateImmediateCallerAcceptAny()
	var st State
	rt.StateReadonly(&st)
	return &st.Address
}
```



#### 虚拟机解释器

###### 显式消息调用（VM外部）
VM解释器编排tipset中消息的执行，并且，消息执行是基于当前tipset的父状态，消息执行的结果产生一个新状态和一系列消息收据。

虚拟机对Tipset链的维护：这个新状态和收据的集合的CID被包含在下一个epoch的区块中。为了形成一个新的tipset，下一个epoch的这些区块必须对这些cid达成一致。（TODO：结合代码，把这里解释的更加清楚）

每个状态更改都是由消息的执行驱动的。

虚拟机对Tipset中消息的执行顺序：为了产生下一个状态，必须执行来自tipset中所有区块的消息。来自同一个tipset的第一个区块的所有消息都在第二个和后续区块的消息之前执行。对于每个区块，首先执行bls聚合的消息，然后执行SECP签名的消息。

###### 隐式消息调用（VM内部）

除了显式地打包在每个区块中的消息之外，隐式消息还在每个epoch中进行一些状态更改。隐式消息不会在节点之间传递，而是由解释器在求值时构造。

对于tipset中的每个区块，有一条隐式消息:

* 调用区块生产者的miner actor来处理(已经验证过的)选举PoSt提交，作为区块中的第一个消息;
* 调用奖励参与者将块奖励支付给矿工的所有者帐户，作为区块中的最终消息;

对于每个tipset，有一条隐式消息:

* 调用cron actor来处理自动的结账和支付，作为tipset中的最后一条消息。

所有隐式消息都是用From地址作为区分的系统account actor构造的。它们指定Gas价格为零，但必须包括在计算中。它们必须成功(退出码为零)，以便计算新的状态。隐式消息的收据不包括在收据列表中，只有显式消息有显式收据。

###### Gas支付
在大多数情况下，消息的发送者为其执行向打包该消息的区块生产者支付Gas费用。

在消息执行后，为每条消息执行所支付的Gas费用立即支付给区块生产者的owner帐户。区块奖励或Gas费都不存在任何障碍：两者都可以立即消费。

###### 重复的信息

由于不同的矿工在同一epoch产生区块，因此单个tipset中的多个区块可能包含相同的消息（由相同的CID标识）。在这种情况下，消息只在以tipset的规范顺序第一次遇到时才被处理。该消息的后续实例将被忽略，并且不会导致任何状态冲突，产生收据，或向区块生产者支付费用。

因此，一个tipset的执行序列总结如下（TODO：根据代码来确定更加准确的内容）:

* 第一个区块的支付奖励
* 第一个区块的选举Post
* 第一个区块的信息（BLS在SECP之前）
* 第二个区块支付奖励
* 第一个区块的选举Post
* 第二个块的消息(BLS在SECP之前，跳过任何已经遇到的)
* […之后的区块…]
* cron tick

###### 消息有效性和消息失败
一个有效区块中的每条消息都可以被处理并产生一个收据（注意，区块有效性意味着所有消息在语法上都是有效的——请参阅消息语法——并正确签名）。但是，消息执行可能成功也可能失败，这取决于消息应用的状态。如果消息执行失败，相应的收据将携带一个非零退出码。

如果消息失败，其原因可以合理归因于区块生产者打包了消息，但永远无法应用到父状态；或者，因为发送方缺乏基金来支付最大消息成本，那么矿工通过燃烧Gas费用而支付罚款（而不是发送方向区块生产者支付费用）（TODO：这结合代码再确定一下）。

由消息失败导致的唯一状态更改是:

* 增加发送Actor的CallSeqNum，并从发送者向打包消息的区块生产者的owner支付Gas费用；或者区块生产者支付与失败消息的gas费用相等的惩罚，从区块生产者那里烧毁（发送者的CallSeqNum不变）。

在以下情况下，消息执行将失败:

* From Actor在状态中不存在（区块生产者被处罚），
* From Actor不是帐户Actor（区块生产者被处罚），
* 消息的CallSeqNum与From actor的CallSeqNum（区块生产者被处罚），
* From Actor没有足够的余额来支付消息转账额度加上最大的Gas成本（GasLimit * GasPrice ，区块生产者被处罚)的总和，
* To Actor不存在于状态中，To地址也不是公钥风格的地址，
* To Actor色存在（或隐式创建为一个帐户），但没有对应于非零MethodNum的方法 （TODO:??），
* 反序列化的Params不是一个长度匹配To Actor的MethodNum方法的长度的数组，
* 反序列化的参数对于由To Actor的MethodNum方法指定的类型无效，
* 被调用的方法消耗的气体比GasLimit允许的要多，
* 被调用的方法以非零代码退出（通过Runtime.Abort()），或
* 接收方发送的任何内部消息都因上述原因而失败。
  

注意，如果To Actor的状态不存在，并且地址是有效的H(pubkey)地址，那么它将被创建为一个Account Actor。