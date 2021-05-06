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



## 区块链

Filecoin区块链是一个分布式虚拟机，在Filecoin协议中实现共识、处理消息、存储账户和维护安全。它是在Filecoin系统中链接各种Actor的主要接口。

Filecoin区块链系统包括:

* 一个消息池子系统，节点使用它来跟踪和传播矿工声明要打包在区块链中的消息。
* 一个虚拟机子系统，用于解释和执行消息以更新系统状态。
* 一个状态树子系统，状态树管理状态树（系统状态，由给定子链的虚拟机确定性地生成）的创建和维护。
* 一个链同步（ChainSync）的susbystem，它跟踪和传播已验证的消息和区块，维护一组候选区块链，矿工可以在这些候选区块链上生产区块并对传入的区块进行语法验证。
* 一个存储算力共识子系统，它跟踪给定链的存储状态（即存储子系统），并帮助区块链系统选择要扩展的子链和要打包在区块链中的区块。

区块链系统还包括：

* 一个链管理器，它维护给定区块链的状态，为其他区块链子系统提供基础设施（这些子系统将查询最新链的状态以保持正常运行），并确保传入的区块在被打包到区块链之前进行语义验证。
* 一个区块生产者，在leader选举成功的情况下被调用，目的是生产一个新的区块，扩展当前的最重链，然后转发到区块同步器进行传播。

从高层次上说，Filecoin区块链通过连续几轮的领导者选举以增长区块链。在领导者选举中，一些矿工被选出来以生成一个区块，被打包到区块链中的区块将为生产者赢得区块奖励。Filecoin的区块链依靠存储算力运行。也就是说，基于Filecoin的共识算法，区块生产者是根据该子链的存储量来确定是否对该子链进行扩展（TODO：结合代码再解释这一句）。存储算力共识子系统维护一个算力表，跟踪storage miner actors通过扇区承诺和时空证明向网络贡献的存储量。



#### 区块

区块（Block）是Filecoin区块链的基本单元，这也是其他大多数区块链的基本单元。区块消息直接与tipset链接，tipset是一组Block消息。tipset将在本节后面部分详细介绍。下面我们将讨论区块（Block）消息的主要结构以及Filecoin区块链验证区块（Block）消息的过程。

区块
区块（Block）是Filecoin区块链的基本单元。

Filecoin区块链中的区块（Block）结构包括：

* i)块头，

* ii)块内的消息列表，

* 以及iii)已签名的消息。

这在FullBlock抽象数据结构中表示。其中的消息是指需要应用的更改集，以使得区块链达到确定状态。区块的Lotus实现具有以下结构:

```
type FullBlock struct {
	Header        *BlockHeader
	BlsMessages   []*Message
	SecpkMessages []*SignedMessage
}
```

请注意：
在Filecoin协议中，区块在功能上与区块头相同。但是，区块头通过Merkle根链接包含完整的系统状态、消息和消息收据，区块是该系统状态、消息和消息收据的完整集合（不仅仅是Merkle根链接，而是状态树、消息树、收据树等的完整数据）。因为一个完整的区块很大，所以Filecoin区块链由区块头组成，而不是完整的区块。我们经常交替使用区块（block）和区块头（block header）这两个术语。

区块头（BlockHeader）是区块的规范化（正则化，规约化）表示。区块头（BlockHeader）在矿工节点之间传播。从区块头（BlockHeader）消息中，区块生产者获得了应用相关完整区块（FullBlock）状态和更新区块链所需的所有信息。为了能够做到这一点，区块头（BlockHeader）需要包含的最少信息项如下所示，包括：区块生产者地址，ticket，时空证明，父区块CID（IPLD DAG），以及消息的CID。

区块头（BlockHeader）的Lotus实现有以下结构：

BlockHeader

```
type BlockHeader struct {
	Miner address.Address // 0

	Ticket *Ticket // 1

	ElectionProof *ElectionProof // 2

	BeaconEntries []BeaconEntry // 3

	WinPoStProof []proof2.PoStProof // 4

	Parents []cid.Cid // 5

	ParentWeight BigInt // 6

	Height abi.ChainEpoch // 7

	ParentStateRoot cid.Cid // 8

	ParentMessageReceipts cid.Cid // 8

	Messages cid.Cid // 10

	BLSAggregate *crypto.Signature // 11

	Timestamp uint64 // 12

	BlockSig *crypto.Signature // 13

	ForkSignaling uint64 // 14

	// ParentBaseFee is the base fee after executing parent tipset
	ParentBaseFee abi.TokenAmount // 15

	// internal
	validated bool // true if the signature has been validated
}
```

Ticket

```
type Ticket struct {
	VRFProof []byte
}
```

ElectionProof

```
type ElectionProof struct {
	WinCount int64
	VRFProof []byte
}
```

BeaconEntry

```
type BeaconEntry struct {
	Round uint64
	Data  []byte
}
```



区块头（BlockHeader）结构必须引用当前一轮的TicketWinner，以确保正确的获胜者被传递给ChainSync（TODO：结合代码准确表示这里）。

```
func IsTicketWinner(vrfTicket []byte, mypow BigInt, totpow BigInt) bool
```



Message结构必须包括源地址（From）和目的地址（to），一个Nonce和GasPrice。

消息对象的Lotus实现具有以下结构:

```
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
```

消息在传递到链同步逻辑之前也会被验证：

```
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
```

###### 区块语法验证

语法验证是指在区块及其消息上执行的验证。语法验证不需要引用类似父状态树等外部信息，因此这种类型的验证有时称为静态验证。

一个无效区块不能用来传播或者作为父区块来引用。

一个语法上有效的区块头（BlockHeader）必须解码成匹配下面定义的字段，必须是一个有效的CBOR PubSub BlockMsg消息，并且必须包含:

* 要么没有父区块（Epoch等于0），要么父区块的CID个数在1到5*ec.ExpectedLeaders之间（Epoch大于0）。
* 一个非负ParentWeight,
* 消息数量小于或等于BlockMessageLimit，
* 聚合消息CID，封装在MsgMeta结构中，序列化到区块头中的消息 CID，
* 一个区块生产者地址，这是一个ID地址。区块头（BlockHeader）中的区块生产者地址必须存在，并对应于当前链状态中的公钥地址。
* 区块签名（BlockSig），并且属于区块生产者的公钥地址
* 一个非负的epoch,
* 一个正的时间戳（Timestamp）
* 一个非空的Ticket，包含VRFResult
* ElectionPoStOutput包含:
  * 数量在1和EC.ExpectedLeaders（包含）之间的候选数组。
  * 非空的PoStRandomness字段，
  * 一个非空的证明（Proof）域，
* 非空的ForkSignal字段。

一个语法上有效的完整区块必须具有：

* 所有引用的消息语法有效，
* 所有引用的父收据语法有效，
* 区块头和包含的消息的序列化大小的总和不大于block.BlockMaxSize。
* 所有显式消息的gas limit的总和不大于block.BlockGasLimit。

注意，区块签名的验证需要从父tipset状态访问区块生产者worker地址和公钥，因此签名验证是语义验证的一部分。类似地，消息签名验证需要查找与区块父状态中每个消息的From帐户Actor相关联的公钥。