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

