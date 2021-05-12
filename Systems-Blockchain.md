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

###### 区块语义验证

语义验证是指需要引用区块头和消息本身之外的信息进行验证。语义验证与构建区块的父tipset和状态有关。

为了进行语义验证，必须从接收到的区块头开始组装完整区块（FullBlock），并检索其Filecoin消息。区块消息CID可以从网络中检索，并被解码为有效的CBOR message或SignedMessage。

在Lotus实现中，区块的语义验证是在Syncer模块中完成的：

```
func (syncer *Syncer) ValidateBlock(ctx context.Context, b *types.FullBlock, useCache bool) (err error) {
	defer func() {
		// b.Cid() could panic for empty blocks that are used in tests.
		if rerr := recover(); rerr != nil {
			err = xerrors.Errorf("validate block panic: %w", rerr)
			return
		}
	}()

	if useCache {
		isValidated, err := syncer.store.IsBlockValidated(ctx, b.Cid())
		if err != nil {
			return xerrors.Errorf("check block validation cache %s: %w", b.Cid(), err)
		}

		if isValidated {
			return nil
		}
	}

	validationStart := build.Clock.Now()
	defer func() {
		stats.Record(ctx, metrics.BlockValidationDurationMilliseconds.M(metrics.SinceInMilliseconds(validationStart)))
		log.Infow("block validation", "took", time.Since(validationStart), "height", b.Header.Height, "age", time.Since(time.Unix(int64(b.Header.Timestamp), 0)))
	}()

	ctx, span := trace.StartSpan(ctx, "validateBlock")
	defer span.End()

	if err := blockSanityChecks(b.Header); err != nil {
		return xerrors.Errorf("incoming header failed basic sanity checks: %w", err)
	}

	h := b.Header

	baseTs, err := syncer.store.LoadTipSet(types.NewTipSetKey(h.Parents...))
	if err != nil {
		return xerrors.Errorf("load parent tipset failed (%s): %w", h.Parents, err)
	}

	lbts, lbst, err := stmgr.GetLookbackTipSetForRound(ctx, syncer.sm, baseTs, h.Height)
	if err != nil {
		return xerrors.Errorf("failed to get lookback tipset for block: %w", err)
	}

	prevBeacon, err := syncer.store.GetLatestBeaconEntry(baseTs)
	if err != nil {
		return xerrors.Errorf("failed to get latest beacon entry: %w", err)
	}

	// fast checks first
	nulls := h.Height - (baseTs.Height() + 1)
	if tgtTs := baseTs.MinTimestamp() + build.BlockDelaySecs*uint64(nulls+1); h.Timestamp != tgtTs {
		return xerrors.Errorf("block has wrong timestamp: %d != %d", h.Timestamp, tgtTs)
	}

	now := uint64(build.Clock.Now().Unix())
	if h.Timestamp > now+build.AllowableClockDriftSecs {
		return xerrors.Errorf("block was from the future (now=%d, blk=%d): %w", now, h.Timestamp, ErrTemporal)
	}
	if h.Timestamp > now {
		log.Warn("Got block from the future, but within threshold", h.Timestamp, build.Clock.Now().Unix())
	}

	msgsCheck := async.Err(func() error {
		if err := syncer.checkBlockMessages(ctx, b, baseTs); err != nil {
			return xerrors.Errorf("block had invalid messages: %w", err)
		}
		return nil
	})

	minerCheck := async.Err(func() error {
		if err := syncer.minerIsValid(ctx, h.Miner, baseTs); err != nil {
			return xerrors.Errorf("minerIsValid failed: %w", err)
		}
		return nil
	})

	baseFeeCheck := async.Err(func() error {
		baseFee, err := syncer.store.ComputeBaseFee(ctx, baseTs)
		if err != nil {
			return xerrors.Errorf("computing base fee: %w", err)
		}
		if types.BigCmp(baseFee, b.Header.ParentBaseFee) != 0 {
			return xerrors.Errorf("base fee doesn't match: %s (header) != %s (computed)",
				b.Header.ParentBaseFee, baseFee)
		}
		return nil
	})
	pweight, err := syncer.store.Weight(ctx, baseTs)
	if err != nil {
		return xerrors.Errorf("getting parent weight: %w", err)
	}

	if types.BigCmp(pweight, b.Header.ParentWeight) != 0 {
		return xerrors.Errorf("parrent weight different: %s (header) != %s (computed)",
			b.Header.ParentWeight, pweight)
	}

	stateRootCheck := async.Err(func() error {
		stateroot, precp, err := syncer.sm.TipSetState(ctx, baseTs)
		if err != nil {
			return xerrors.Errorf("get tipsetstate(%d, %s) failed: %w", h.Height, h.Parents, err)
		}

		if stateroot != h.ParentStateRoot {
			msgs, err := syncer.store.MessagesForTipset(baseTs)
			if err != nil {
				log.Error("failed to load messages for tipset during tipset state mismatch error: ", err)
			} else {
				log.Warn("Messages for tipset with mismatching state:")
				for i, m := range msgs {
					mm := m.VMMessage()
					log.Warnf("Message[%d]: from=%s to=%s method=%d params=%x", i, mm.From, mm.To, mm.Method, mm.Params)
				}
			}

			return xerrors.Errorf("parent state root did not match computed state (%s != %s)", stateroot, h.ParentStateRoot)
		}

		if precp != h.ParentMessageReceipts {
			return xerrors.Errorf("parent receipts root did not match computed value (%s != %s)", precp, h.ParentMessageReceipts)
		}

		return nil
	})

	// Stuff that needs worker address
	waddr, err := stmgr.GetMinerWorkerRaw(ctx, syncer.sm, lbst, h.Miner)
	if err != nil {
		return xerrors.Errorf("GetMinerWorkerRaw failed: %w", err)
	}

	winnerCheck := async.Err(func() error {
		if h.ElectionProof.WinCount < 1 {
			return xerrors.Errorf("block is not claiming to be a winner")
		}

		eligible, err := stmgr.MinerEligibleToMine(ctx, syncer.sm, h.Miner, baseTs, lbts)
		if err != nil {
			return xerrors.Errorf("determining if miner has min power failed: %w", err)
		}

		if !eligible {
			return xerrors.New("block's miner is ineligible to mine")
		}

		rBeacon := *prevBeacon
		if len(h.BeaconEntries) != 0 {
			rBeacon = h.BeaconEntries[len(h.BeaconEntries)-1]
		}
		buf := new(bytes.Buffer)
		if err := h.Miner.MarshalCBOR(buf); err != nil {
			return xerrors.Errorf("failed to marshal miner address to cbor: %w", err)
		}

		vrfBase, err := store.DrawRandomness(rBeacon.Data, crypto.DomainSeparationTag_ElectionProofProduction, h.Height, buf.Bytes())
		if err != nil {
			return xerrors.Errorf("could not draw randomness: %w", err)
		}

		if err := VerifyElectionPoStVRF(ctx, waddr, vrfBase, h.ElectionProof.VRFProof); err != nil {
			return xerrors.Errorf("validating block election proof failed: %w", err)
		}

		slashed, err := stmgr.GetMinerSlashed(ctx, syncer.sm, baseTs, h.Miner)
		if err != nil {
			return xerrors.Errorf("failed to check if block miner was slashed: %w", err)
		}

		if slashed {
			return xerrors.Errorf("received block was from slashed or invalid miner")
		}

		mpow, tpow, _, err := stmgr.GetPowerRaw(ctx, syncer.sm, lbst, h.Miner)
		if err != nil {
			return xerrors.Errorf("failed getting power: %w", err)
		}

		j := h.ElectionProof.ComputeWinCount(mpow.QualityAdjPower, tpow.QualityAdjPower)
		if h.ElectionProof.WinCount != j {
			return xerrors.Errorf("miner claims wrong number of wins: miner: %d, computed: %d", h.ElectionProof.WinCount, j)
		}

		return nil
	})

	blockSigCheck := async.Err(func() error {
		if err := sigs.CheckBlockSignature(ctx, h, waddr); err != nil {
			return xerrors.Errorf("check block signature failed: %w", err)
		}
		return nil
	})

	beaconValuesCheck := async.Err(func() error {
		if os.Getenv("LOTUS_IGNORE_DRAND") == "_yes_" {
			return nil
		}

		if err := beacon.ValidateBlockValues(syncer.beacon, h, baseTs.Height(), *prevBeacon); err != nil {
			return xerrors.Errorf("failed to validate blocks random beacon values: %w", err)
		}
		return nil
	})

	tktsCheck := async.Err(func() error {
		buf := new(bytes.Buffer)
		if err := h.Miner.MarshalCBOR(buf); err != nil {
			return xerrors.Errorf("failed to marshal miner address to cbor: %w", err)
		}

		if h.Height > build.UpgradeSmokeHeight {
			buf.Write(baseTs.MinTicket().VRFProof)
		}

		beaconBase := *prevBeacon
		if len(h.BeaconEntries) != 0 {
			beaconBase = h.BeaconEntries[len(h.BeaconEntries)-1]
		}

		vrfBase, err := store.DrawRandomness(beaconBase.Data, crypto.DomainSeparationTag_TicketProduction, h.Height-build.TicketRandomnessLookback, buf.Bytes())
		if err != nil {
			return xerrors.Errorf("failed to compute vrf base for ticket: %w", err)
		}

		err = VerifyElectionPoStVRF(ctx, waddr, vrfBase, h.Ticket.VRFProof)
		if err != nil {
			return xerrors.Errorf("validating block tickets failed: %w", err)
		}
		return nil
	})

	wproofCheck := async.Err(func() error {
		if err := syncer.VerifyWinningPoStProof(ctx, h, *prevBeacon, lbst, waddr); err != nil {
			return xerrors.Errorf("invalid election post: %w", err)
		}
		return nil
	})

	await := []async.ErrorFuture{
		minerCheck,
		tktsCheck,
		blockSigCheck,
		beaconValuesCheck,
		wproofCheck,
		winnerCheck,
		msgsCheck,
		baseFeeCheck,
		stateRootCheck,
	}

	var merr error
	for _, fut := range await {
		if err := fut.AwaitContext(ctx); err != nil {
			merr = multierror.Append(merr, err)
		}
	}
	if merr != nil {
		mulErr := merr.(*multierror.Error)
		mulErr.ErrorFormat = func(es []error) string {
			if len(es) == 1 {
				return fmt.Sprintf("1 error occurred:\n\t* %+v\n\n", es[0])
			}

			points := make([]string, len(es))
			for i, err := range es {
				points[i] = fmt.Sprintf("* %+v", err)
			}

			return fmt.Sprintf(
				"%d errors occurred:\n\t%s\n\n",
				len(es), strings.Join(points, "\n\t"))
		}
		return mulErr
	}

	if useCache {
		if err := syncer.store.MarkBlockAsValidated(ctx, b.Cid()); err != nil {
			return xerrors.Errorf("caching block validation %s: %w", b.Cid(), err)
		}
	}

	return nil
}
```

消息是由Syncer模块来检索。

Syncer模块后续还有两个步骤：

* 1- 使用先前接收到的区块组装出FullTipSet。Block的ParentWeight必须大于最重的tipset(第一个Block)的ParentWeight。
* 2- 从接收到的Block中检索所有tipset。逐个验证这些tipset中的每个区块。验证应该确保：
  * 信标项按轮数排序。
  * Tipset Parents CID与通过BlockSync获取的父Tipset CID匹配。

一个语义上有效的区块必须满足以下所有要求：

###### 父区块相关

* 父区块按它们区块头的ticket的字典顺序排列。
* 区块的ParentStateRoot CID与从父Tipset计算出的状态CID匹配。
* 执行父tipset的消息所产生的状态树，与ParentState匹配。
* parentmessagereceits标识父tipset执行产生的收据列表（来自父tipset的每个唯一消息都有一个收据）。换句话说，Block的parentmessagereceits CID匹配从父tipset计算出的收据CID。
* ParentWeight匹配与父tipset相关的链的权重。

###### 时间相关

* “epoch”大于“父区块”的epoch，并且
  * 不是未来的epoch，根据节点的本地时钟读取当前epoch，
    * 来自未来epoch的区块不应该被拒绝，但是在前面合适的epoch出现之前不应该被计算（验证或包含在tipset中）
* 不是过去更远（比SPC finality定义的软finality更远）的epoch
  * 这条规则只适用于接收到新的gossip区块（从当前区块链链头），而不是节点第一次启动时同步区块链。
* 时间戳（Timestamp），以秒为单位，满足：
  * 不能大于当前时间加上ΑllowableClockDriftSecs
  * 必须小于前一个区块的Timestamp + BlockDelay（含空块）
  * 是由创世区块的时间戳、网络的Βlock时间和Βlock的Epoch所决定的精确值。

###### Miner相关

* 在父tipset的状态中，Miner在存储算力表中处于活动状态。矿工的地址登记Power Actor的Claims HAMT中（TODO：结合代码）
* 对于要验证的每个tipset，应该包含TipSetState。
  * tipset中的每个区块都应该属于不同的矿工。
* 与消息的From地址相关联的Actor存在，是一个帐户Actor，它的Nonce与消息Nonce匹配。
* 包括矿工证明的有效证据（可以访问被挑战扇区的密封版本）。为了达到这个目的：
* 使用WinningPoSt域分离标记绘制当前纪元的随机性。
* 根据随机抽取，获得在这个时代为这个矿工挑战的扇区列表。
* StoragePowerActor中Miner的算力没有被惩罚。

###### 信标或者Ticket相关

* 需要包含有效的BeaconEntries：
  * 检查每个BeaconEntries都是一个消息的签名：previousSignature || round （使用 DRAND的公钥签名）。
  * MaxBeaconRoundForEpoch到prevEntry（前一个的tipset）之间的所有条目都应该包括在内。

* 从父tipset区块头产生一个最小ticket，
  * Ticket.VRFResult由miner Actor的worker帐户公钥有效签名，
* ElectionProof Ticket通过使用区块生产者的密钥检查BLS签名来正确计算。“ElectionProof” ticket应该是一个中奖ticket。

###### 消息和签名相关

* secp256k1消息通过发送Actor的（From）worker帐户密钥正确签名，
* 包含一个BLSAggregate签名，该签名用其发送Actor的密钥对区块引用的所有BLS消息的CID数组进行签名。
* 包含了来自区块的Miner Actor的worker帐户公钥在区块头字段上的有效签名。
* 对于ValidForBlockInclusion()中的每条消息，保持如下：
  * 正确定义了消息字段Version、To、From、Value、GasPrice和GasLimit。
  * 消息GasLimit在消息最小Gas成本(从链高度和消息长度生成)之下。
* 对于ApplyMessage中的每条消息（即被执行之前的消息），以下保持不变:
  * checkMessage()中的基本Gas和值检查:
    * 消息GasLimit大于零。
    * 配置消息GasPrice和Value。
  * 该消息的存储Gas成本低于该消息的GasLimit。
  * Message的Nonce与从Message的from地址检索到的Actor中的Nonce匹配。
  * 消息的最大Gas成本（从它的GasLimit、GasPrice和Value派生出来）在从消息的from地址检索到的Actor的余额之下。
  * 消息转账的金额在从消息的from地址检索到的Actor的余额之下。

除了验证消息的签名之外，对区块中包含的消息没有做语义验证。如果包含在一个块中的所有消息在语法上都是有效的，那么它们可以被执行并产生一个收据。

链同步系统可以分阶段进行语法和语义验证，以减少不必要的资源消耗。

如果以上所有测试都成功，则将区块标记为已验证。最终，无效块区不能进一步传播或作为父节点进行验证。

#### TipSet

预期共识按概率在每个epoch选择多个leader（区块生产者），这意味着Filecoin区块链在每个epoch中可能包含0个或多个区块（每个当选的leader一个）。来自同一epoch的区块被组装到一个tipset中。VM解释器通过执行tipset中的所有消息（在对多个区块中包含的相同消息进行重复删除之后）来修改Filecoin状态树。

(TODO：这里通过一个形象的图来说明)

每个区块都引用父tipset并验证父tipset的状态，同时选择当前epoch应该打包的消息。在将新区块合并到tipset中之前，无法知道新区块的消息所应用之后的最终状态。因此，单独执行来自单个区块的消息是没有意义的：只有在执行该块的tipset中的所有消息时，才会知道最新状态树。

（TODO：这里也是Filecoin设计中的一个特点和难点，对gas费用验证也有影响）

有效tipset包含一个非空的区块集合，这些区块来自不同的矿工，并且所有的矿工都指定相同的下列内容：

* 时代
* 父Tipset
* ParentWeight
* StateRoot
* ReceiptsRoot

Tipset中的区块按照每个区块ticket字节的字典顺序进行规范化排序，这打破了与区块本身CID字节的关系。

由于网络传播延迟，在epoch N+1的矿工可能会忽略其父tipset在epoch N挖出的有效块。这并不会使新生成的区块无效，但是会减少新区块的权重和成为规范链一部分（预期共识的chain Selection函数所定义的协议）的机会（TODO：这里结合代码解释更准确）。

区块生产者应该协调如何选择包含在区块中的消息，以避免重复，从而最大化他们从消息费用中获得的预期收益（见消息池部分）（TODO：确定对生产环境是否有影响）。

Lotus实现中的主要Tipset结构包括以下内容:

```
type TipSet struct {
	cids   []cid.Cid
	blks   []*BlockHeader
	height abi.ChainEpoch
}
```

Tipset的语义验证包括以下检查：

* 一个tipset至少由一个区块组成。(因为每个tipset的区块数量是可变的，这是由随机性决定的，所以我们没有设置上限。)
* 一个tipset中的所有区块都有相同的高度。
* 所有的区块都有相同的父区块（相同数量的父区块和匹配的cid）。

```
func NewTipSet(blks []*BlockHeader) (*TipSet, error) {
	if len(blks) == 0 {
		return nil, xerrors.Errorf("NewTipSet called with zero length array of blocks")
	}

	sort.Slice(blks, tipsetSortFunc(blks))

	var ts TipSet
	ts.cids = []cid.Cid{blks[0].Cid()}
	ts.blks = blks
	for _, b := range blks[1:] {
		if b.Height != blks[0].Height {
			return nil, fmt.Errorf("cannot create tipset with mismatching heights")
		}

		if len(blks[0].Parents) != len(b.Parents) {
			return nil, fmt.Errorf("cannot create tipset with mismatching number of parents")
		}

		for i, cid := range b.Parents {
			if cid != blks[0].Parents[i] {
				return nil, fmt.Errorf("cannot create tipset with mismatching parents")
			}
		}

		ts.cids = append(ts.cids, b.Cid())

	}
	ts.height = blks[0].Height

	return &ts, nil
}
```

#### 链管理器和TipSet管理器

链管理器是区块链系统中的核心组件。它跟踪并更新由给定节点接收到的竞争子链，以选择正确的区块链链头：链管理器感知到的最重链的最新块。

在这个过程中，链管理器处于中心的位置，为Filecoin节点的其他系统提供薄记服务，并公开API供这些系统调用，例如，其他系统能够从区块链中取样随机性，或查看最新打包的区块（TODO：结合代码确定这个API）。

###### 链的扩展和增长：处理收到的区块

对于每个传入的区块，即使没有添加到当前最重的tipset中，链管理器也应该将它添加到正在跟踪的正确子链中，或者独立地跟踪它，直到:

* 通过接收该子链中的另一个区块，使得它可以添加到当前最重的子链中，或者
* 区块在终结之前被挖出，使得它必须被丢弃。

重要的是，要注意，在终结（finality，TODO：确定终结的代码含义）之前，一个给定的子链可能会被放弃，因为在同一轮中挖出更重的子链。为了快速适应这一点，链管理器在终结之前必须维护和更新所有的子链。

链选择是Filecoin区块链如何工作的一个关键组成部分。简而言之，每条链都有一个相关的权重，该权重表示在其上挖掘的区块数量以及它们附加的算力（存储）。链选择如何工作的详细信息可以参考Chain selection部分。

Notes /建议:

* 为了使某些验证检查更简单，区块应该按高度和父区块集合建立索引。通过这种方式，可以快速查询具有给定高度和共同父级的区块集合。
* 在这些集合中计算和缓存区块的最终聚合状态可能也很有用，当一个区块有多个父节点时，这可以节省额外的状态计算。
* 建议区块保存在本地数据存储中，而不管它们是否被理解为目前的最佳技巧——这是为了避免将来不得不重新获取相同的块。

###### Chain Tips Manager

Chain Tips Manager是Filecoin共识的一个子组件，负责跟踪Filecoin区块链的所有实tipset，并跟踪当前“最佳”tipset是什么。

```
// Returns the ticket that is at round 'r' in the chain behind 'head'
func TicketFromRound(head Tipset, r Round) {}

// Returns the tipset that contains round r (Note: multiple rounds' worth of tickets may exist within a single block due to losing tickets being added to the eventually successfully generated block)
func TipsetFromRound(head Tipset, r Round) {}

// GetBestTipset returns the best known tipset. If the 'best' tipset hasn't changed, then this
// will return the previous best tipset.
func GetBestTipset()

// Adds the losing ticket to the chaintips manager so that blocks can be mined on top of it
func AddLosingTicket(parent Tipset, t Ticket)
```

#### 区块生产者

###### 生产区块的前提条件

如果节点存储算力满足最小阈值要求（TODO：附上代码数字），则向存储算力actor（storage power actor ）注册的区块生产者可以开始生产区块和检查选举赢票。

为了做到这一点，区块生产者必须运行区块链验证功能，并跟踪最近收到的区块。一个矿工的新区块需要基于上一个epoch的父区块。

###### 区块创建

为epoch H生成一个区块需要等待那个epoch的信标项，并使用该信标项来运行GenerateElectionProof。

如果WinCount≥（即，当矿工被选中时），则使用相同的信标项运行WinningPoSt。

借助ElectionProof ticket（GenerateElectionProof的输出）和WinningPoSt证明，区块生产者可以生成一个新的区块。

请参阅VM解释器了解父tipset 校验的详细信息，并参阅Block了解对有效区块头的约束。

要创建一个区块，合格的矿工必须计算几个字段:

* Parents：父tipset所有区块的CID。
* ParentWeight：父链的重量（参见链选择）。
* ParentState：父tipset状态根的CID（参见VM解释器）。
* ParentMessageReceipts：计算ParentState时产生的包含收据的AMT根的CID。
* Epoch：区块的Epoch，由Parents Epoch和生成该区块所需要的Epoch数派生而来。
* Timestamp：在创建区块时生成的Unix时间戳，以秒为单位。
* BeaconEntries：自上一个区块以来生成的一组drand项（参见Beacon 项）。
* Ticket：从上一个epoch的Ticket生成一个新的Ticket（见Ticket生成）。
* Miner：区块生产者的 miner actor 地址。
* Messages：一个TxMeta对象的CID，包含准备打包在新区块中的消息：
  * 从内存池中选择一组消息包含在区块中，满足区块大小和Gas限制
  * 将消息分为BLS签名消息和secpk签名消息
  * TxMeta.BLSMessages：由UnsignedMessages组成的AMT根的CID
  * TxMeta.SECPMessages：由SignedMessages组成的AMT根的CID
* BeaconEntries：用来生成随机数的信标项列表
* BLSAggregate：使用BLS签名的区块中所有消息的聚合签名
* Signature：区块生产者的worker帐户私钥在区块头的序列化表示（空签名）上的签名（也必须匹配ticket签名）。
* ForkSignaling：标识信令分叉的标志。应该在默认情况下设置为0。

请注意，为了生成一个有效的区块，不需要对包含在区块中的消息进行计算。区块链生产者可能希望以预测的方式评估消息，以便将那些成功执行和支付最多Gas的消息打包进来。

当生成一个区块时，区块奖励不会被评估。当区块被包含在下一个时代的tipset中时，区块奖励就会支付。

区块的签名确保了传播后区块的完整性，因为与许多PoW区块链不同，赢票Ticket是独立于区块生成的。

###### 区块广播

一个合格的区块生产者使用GossipSub的/fil/blocks主题将完成的区块传播到网络，假设一切都正确完成，网络将接受该区块，其他区块生产者将在该区块之上继续生产区块，为矿工赢得区块奖励。

区块生产者应该在其有效区块被生产后立即输出，否则其他矿工在EPOCH_CUTOFF之后收到该区块，其他旷工将不会在当前epoch接受该区块。

###### 区块奖励

块奖励由奖励参与者处理。关于区块奖励的详细信息在Filecoin通证部分讨论，关于块奖励抵押品的详细信息在矿工抵押品部分讨论。

#### 消息池

消息池（ Message Pool），也被称为mpool或者是mempool，是Filecoin协议中消息的池化集合。消息池充当Filecoin点对点网络中传播链下消息的接口。消息池被节点用来维护一组他们想要传输到Filecoin VM并添加到链上的消息（例如，添加用于“链上”执行）。

为了让消息上链，首先必须把消息添加到消息池中。实际上，至少在Filecoin的Lotus实现中，并没有中心化的消息池。相反，消息池是一个抽象，具体表现为网络中每个节点保存的消息列表。因此，当一个节点将一条新消息添加到消息池时，该消息将被libp2p的pubsub协议传播到网络的其他节点。节点需要订阅相应的pubsub主题才能接收到传播的消息。

使用GossipSub的消息传播不会立即发生，因此，在不同节点上的消息同步会有一些延迟，也就是说，不同节点上的消息池不会保持为一致的同步状态。在实践中，由于要添加到消息池中的消息是连续的消息流，并且消息传播总有延迟，则消息池永远不会跨网络中的所有节点保持同步。这不是系统的缺陷，因为消息池不需要跨网络进行同步。

消息池应该定义一个最大大小，以避免DoS攻击，在DoS攻击中，节点发送垃圾消息并耗尽内存。建议的消息池大小为5000条消息。

###### 消息传播

消息池必须集成libp2p的GossipSub协议接口。这是因为消息是通过GossipSub相应的/fil/msgs/ 话题传播的。任何节点发布每条消息都是通过/fil/msgs/ 话题宣布。

有两个与消息和区块相关的pubsub 话题：

*  i)携带消息的/fil/msgs/ 话题

* ii)携带区块的/fil/blocks/ 话题。

/fil/msgs/ 话题链接到消息池。流程如下：

1. 当客户端想要在Filecoin网络中发送消息时，他们将消息发布到/fil/msgs/ 话题。
2. 消息使用GossipSub传播到网络中的所有其他节点，并最终到达所有矿工的消息池中。
3. 根据加密经济规则，一些区块生产者最终将从消息池中（与其他消息一起：TODO：这里的其他消息是什么？）挑选消息，并将其打包在一个区块中。
4. 区块生产者在/fil/blocks/ pubsub主题中发布新生产的区块，该区块传播到网络中的所有节点（包括发布该区块打包消息的节点）。

节点必须检查传入的消息是否有效，即它们是否具有有效的签名。如果消息无效，则应删除该消息，且不再进行转发。

GossipSub协议的更新增强版本包括许多攻击缓解策略。例如，当节点收到一个无效的消息时，它会给发送者分配一个负值分数。节点分数不与其他节点共享，而是由每个节点在本地保存与其交互的节点的分数。如果一个对等节点的分数低于一个阈值，它将被排除在对等节点评分列表之外（黑名单）。我们将在GossipSub 小节中讨论这些设置的更多细节。完整的细节可以在GossipSub规范中找到。

注意:

* 资金检查：需要注意的是，消息池逻辑不是检查消息发布者的帐户中是否有足够的资金。而是检查打包区块的区块生产者是否有足够的资金。
* 消息排序：消息到达区块生产者的消息池时，根据区块生产者遵循的加密经济规则在消息池中进行排序，以便区块生产者组成下一个区块。

###### 消息存储

如前所述，不存在中心化的消息池。相反，每个节点都必须为传入消息分配内存。

#### 链同步

区块链同步(“sync”)是区块链系统的关键组成部分。它处理区块和消息的检索和传播，因此主要负责区块链系统的分布式状态复制。因此，这个过程是安全关键的——状态复制的问题可能会对区块链的操作造成严重的后果。

当节点第一次加入网络时，它首先发现对等节点（通过上面讨论的对等节点发现），并订阅/fil/blocks和/fil/msgs GossipSub主题。新加入节点监听由其他节点传播的新区块。然后选择一个区块作为BestTargetHead（最佳目标区块链链头，具有一个高度），并开始从TrustedCheckpoint同步区块链到这个高度，TrustedCheckpoint默认为GenesisBlock或GenesisCheckpoint。为了挑选BestTargetHead（最佳目标区块链链头），加点需要比较高度和重量的组合——这些值越高，所选择的区块是主链的机会就越高。如果有两个区块高度相同，则节点应选择重量较大的区块。一旦节点选择了BestTargetHead，它就会使用BlockSync协议来获取区块并达到当前的高度。达到当前高度之后，节点处于CHAIN_FOLLOW模式。在这种模式下，节点使用GossipSub接收新的区块。如果节点获悉一个区块，却没有通过GossipSub收到，则使用Bitswap协议来获取该区块。

TODO：CHAIN_FOLLOW列举一些代码中的状态。



###### 概述

ChainSync是Filecoin用来同步区块链的协议。它是特定于Filecoin网络在状态表征和共识规则方面的选择，但也足够普遍和通用，可以服务于其他区块链。ChainSync是一组较小的协议，这些协议分别处理同步过程的不同部分。

一般来说，链同步适用于以下场景：

* 当节点第一次加入网络，首先需要达到当前的最新状态，之后才能验证或扩展区块链。
* 当节点没有和区块链保持同步，例如，由于短暂的网络连接断开。
* 当节点正常运行期间，同步最新的消息和区块。

支持上述三种场景的区块同步协议：

* GossipSub是libp2p的pubsub协议，用于传播消息和区块。它主要用于上面的第三个过程：当节点需要与最新生成的区块保持同步时。
* BlockSync用于同步区块链的特定部分，即从一个特定的高度和到一个特定的高度。
* Hello协议，主要用于两个对等节点第一次“见面”（即第一次彼此连接）时。根据Hello协议规则，对等节点在第一次“见面”时会交换区块链链首。

此外，当一个节点已经同步好（处于“紧跟”模式时），但GossipSub传播区块失败的情况下，系统使用Bitswap协议来请求和接收区块。总之，作为用来获取区块链的一部分的协议，GraphSync是比Bitswap更有效的版本。

Filecoin节点也是libp2p节点，因此也可以运行其他多种协议。与Filecoin中的任何其他部分一样，节点可以选择使用其他协议来实现结果。也就是说，节点必须实现本规范中描述的ChainSync版本，才能被视为Filecoin的实现。



###### 术语和概念

* LastCheckpoint是ChainSync所知道的最后一个刚性全局共识检查点。这个共识检查点定义了最小最终结果，以及最小历史。ChainSync信任LastCheckpoint，并以LastCheckpoint为基础，一旦确定，就不再改变（TODO：结合代码阐述的更清楚）。
* TargetHeads表示最新生产区块的blockcid列表。这些都是ChainSync所知道的最新且最好的区块。他们是“目标”链首，因为ChainSync会尝试同步到这些链首。这个列表是按照“成为最佳链的可能性”排序的。这种可能性上只是通过ChainWeight实现的。
* BestTargetHead单最佳链首BlockCID，尝试同步到该链首。BestTargetHead是TargetHeads的第一个元素



###### 链同步状态机

在高层次上，ChainSync做了以下工作:

* 第1部分：验证内部状态（下面的INIT状态）
  * 必须验证数据结构和验证本地区块链
  * 资源昂贵的验证可能会被跳过，而节点自己承担风险
* 第2部分：引导网络（Bootstrap）
  * 步骤1，引导网络，并获得一组“足够安全”的对等节点（下面有更多细节）
  * 步骤2，启动GossipSub频道
* 第3部分：同步可信检查点状态（SYNC_CHECKPOINT）
  * 步骤1，从TrustedCheckpoint（默认为GenesisCheckpoint）开始，TrustedCheckpoint不应该在软件中进行验证，它应该由运维人员进行验证。
  * 步骤2，获取TrustedCheckpoint所指向的区块，以及该区块的父区块
  * 步骤3，获取StateTree
* 第4部分：紧跟最新区块链（CHAIN_CATCHUP）
  * 步骤1，维护一组TargetHeads（blockcid），并从中选择BestTargetHead
  * 步骤2，同步到最近观察到的链首，验证链首后面的区块（请求中间区块）
  * 步骤3，随着验证的进行，TargetHeads和BestTargetHead可能会发生变化，因为最新生成的区块将会到来，而之前TargetHeads和BestTargetHead指向的一些链首或路径可能已无法验证。
  * 步骤4，当节点“赶上”BestTargetHead（检索所有状态，链接到本地区块链，验证所有区块，等等）时，链同步完成。
* 第5部分：保持区块同步并参与区块传播（CHAIN_FOLLOW）
  * 步骤1，如果安全条件发生变化，则返回到第4部分（CHAIN_CATCHUP）
  * 步骤2，接收、验证并传播接收到的区块
  * 步骤3，现在对前进的链状态有更大的确定性：有最好的链，最后的Tipsets等。



ChainSync使用以下概念状态机。由于这是一个概念性的状态机，实现可能会偏离这些状态的精确表达或严格划分。实现可能会模糊状态之间的界限。如果是，不同的实现必须确保更改的协议的安全性。

（TODO：重画此图，维护中文版）

![chain sync](https://spec.filecoin.io/_gen/diagrams/systems/filecoin_blockchain/chainsync/chainsync_fsm.svg?1605115265)



###### 对等节点的发现
对等节点发现是整个体系结构的关键部分。对等节点发现的出错可能会对协议的运行产生严重的后果。一个新节点在加入网络时最初连接到的一组节点可能会完全控制该节点对其他节点的感知，从而控制该节点所拥有的网络状态的视图（eclipse攻击，taoshengshi注释）。

对等节点发现可以由任意的外部方式驱动，并被放置在ChainSync涉及的协议（GossipSub, Bitswap, BlockSync）核心功能之外。这允许正交的、应用驱动的开发，并且协议实现没有外部依赖。尽管如此，流言子协议支持:

* i)对等交换，
* ii)显式对等协议。

**对等交换**

对等交换允许应用从一组已知的对等节点启动，而无需外部对等节点发现机制。这个过程可以通过引导节点或其他正常节点来实现。引导节点必须由系统运维人员维护，并且必须正确配置。它们必须是稳定的，并且独立于协议结构运行，例如GossipSub网格结构，也就是说，引导节点不维护到网格的连接。

有关对等交换的更多细节，请参阅GossipSub规范。

**显式对等协议**
使用显式对等协议，系统运维人员必须指定一个对等节点列表，当节点加入时应该连接到这些对等节点。协议必须有可供指定的选项。对于每个显式对等节点，路由器必须建立并维护一个双向（互惠）连接。

###### 步进区块验证
* 为了减少资源支出，区块可以分阶段进行验证。

* 验证计算量是很大的，也是一个严重的DOS攻击向量。
* 安全实现必须小心地安排验证，可以通过修建区块来实现不完全验证，从而最小化验证工作。
* ChainSync应该维护一个未验证的区块缓存（理想情况下，按照属于链的可能性排序)，当未验证的区块被FinalityTipset传递时，或者当ChainSync处于巨大的资源负载下时，删除区块缓存。

这些阶段可以部分地跨候选链中的许多区块使用，以便在实际执行昂贵的验证工作之前，清除明显的坏区块。

区块验证的步进阶段：

* BV0 - 语法：序列化，类型，值范围。
* BV1 - 可疑的共识：可疑的矿工，权重，和epoch值（例如从链状态在b.ChainEpoch - Consensus . lookbackparameter）。
* BV2 - 区块签名
* BV3 - 信标表项：有效的随机信标表项已经插入到区块中（参见信标表项验证）。
* BV4 - ElectionProof： 一个有效的选举证明已经生成。
* BV5 - WinningPoSt： 一个正确的PoSt已经生成。
* BV6 - 链的起始和终结：在终结之前，验证区块是否被链接回可信链。
* BV7 -消息签名:
* BV8 -状态树：父tipset消息执行产生的状态树树根和收据。

#### 存储算力共识

存储算力共识（Storage Power Consensus，SPC）子系统是使得Filecoin节点就系统状态达成一致的主要接口。存储算力共识在其算力表中描述了给定区块链中单个存储矿工（区块生产者）相对于共识的有效算力。它还运行预期共识（Filecoin使用的底层共识算法)，使存储矿工（区块生产者）能够运行领导者选举并生成新区块来更新Filecoin系统的状态。

简单地说，SPC子系统提供以下服务：

* 每个子链对算力表的访问，包括单个存储矿工（区块生产者）的算力和链上总算力。

* 访问单个存储矿工（区块生产者）的预期共识，使:
  * 访问由drand提供的可验证随机性ticket，drand为协议的其余部分服务。
  * 运行领导者选举产生新的区块。
  * 使用预期共识（EC）的加权函数对多个子链的进行选择。
  * 标识最近完成的tipset，供所有协议参与者使用。

###### 区分存储矿工和区块矿工

有两种方法在Filecoin网络中获得Filecoin通证:

* 以存储提供者的身份参与存储市场，并由客户为文件存储交易付费。
* 通过生产新的区块，扩展区块链，保障Filecoin共识安全，并作为存储矿工运行智能合约执行状态更新。

有两种类型的“矿工（区块生产者）”（存储矿工和区块矿工）需要区分。Filecoin中的领袖选举是基于矿工的存储能力。因此，虽然所有的区块矿工都将是存储矿工，但存储矿工不一定是区块矿工。

然而，由于Filecoin的“有用工作量证明”是通过文件存储（PoRep和PoSt）实现的，存储矿工参与领导者选举的开销很小。这样的存储矿工Actor只需要向存储算力Actor注册，就可以参与预期共识和区块生产。



###### 关于算力
质量调整算力作为扇区质量的静态函数分配给各个扇区，包括：

* i)扇区的时空，这是扇区大小承诺存储时间的乘积
*  ii)  交易权重，将交易占据的时空转换成共识
* iii)交易质量乘数取决于扇区上交易的类型（例如,CC常规交易或验证客户端交易),
* 后,iv)扇区质量乘数，这是交易质量乘数的平均值，权重为扇区中每种类型的交易所占据的时空数量。

扇区质量是一种衡量方法，它将一个扇区在其生命周期内的规模、持续时间和活跃交易类型，映射到其对算力和奖励分配的影响。

一个扇区的质量取决于该扇区内部的数据承载的交易。通常有三种类型的交易：

* 承诺容量（Committed Capacity ，CC），实际上是空交易，区块生产者存储任意数据在扇区内部
* 常规交易（*Regular Deals*），区块生产者（存储矿工）和客户在市场上就价格达成一致。
* 验证客户交易（*Verified Client* deals），赋予扇区更多的算力。关于扇区类型和扇区质量的详细信息，请参阅扇区和扇区质量部分；关于验证客户；请参阅验证客户部分；关于交易权重和质量乘数的具体参数值，请参阅加密经济学部分（TODO）。

质量调整算力（Quality-Adjusted Power）是一个矿工在秘密领导者选举中拥有的选票数量，并被定义为随着一个矿工承诺给网络的有用存储线性增加。

更准确地说，我们有以下定义:

原始字节算力：扇区的大小(以字节为单位)。
质量调整算力：在网络上存储的数据所具备的共识算力，等于原始字节算力乘以扇区质量乘数。



###### 信标项

Filecoin协议使用drand信标产生的随机性来作为在区块链中使用的无偏随机性种子（参见随机性）。

反过来，这些随机的种子被用于:

* sector_sealer：作为SealSeeds将扇区承诺绑定到给定的子链。
* post_generator：作为PoStChallenges来证明扇区在给定区块（epoch）仍然处于有效的提交状态。
* 在Secret Leader Election中，存储算力子系统作为随机性来决定一个矿工（区块生产者）被选择产生一个新区块的频率。

这种随机性可能来自Filecoin区块链不同的epoch，由各自的协议根据其安全需求使用它们。

需要注意的是，给定的Filecoin网络和给定的drand网络不需要相同的轮次时间，也就是说，Filecoin生成的区块可能比drand生成的随机性更快或更慢。例如，如果drand信标产生随机性的速度是Filecoin产生区块的两倍，我们可能会期望在一个Filecoin epoch中产生两个随机值，相反，如果Filecoin网络的速度是drand的两倍，我们可能会期望每隔一个Filecoin epoch就产生一个随机值。相应地，根据两个网络的配置，某些Filecoin区块可能包含多个drand项，也可能不包含drand项。此外，在网络中断期间对drand网络的任何新的随机条目的调用都必须被阻塞，如下面的drand. public()调用所示。在所有情况下，Filecoin区块必须包含自最后一个epoch以来在区块头的BeaconEntries字段中生成的所有drand信标输出。任何对给定Filecoin epoch的随机性的使用都应该使用Filecoin区块中包含的最后一个有效drand项。（TODO：这一段，需要根据代码重新梳理一下）。如下所示。

**为VM获取drand随机性**
对于PoRep创建、证明验证或任何需要Filecoin VM随机性的操作，应该有一个方法可以正确地从链中提取drand项。请注意，如果drand较慢，round可能跨越多个filecoin epoch；最低epoch的区块将包含请求的信标项。类似地，如果在信标应该被插入的地方存在空轮（空块？），我们需要在链上迭代以找到条目被插入的位置（迭代，这里的意思是向后面推迟）。具体来说，下一个非空区块必须包含根据定义请求的drand项（下一个非空区块需要包含drand项）。（TODO：这里需要根据代码确定的更准确）

**从drand网络获取随机性**
当挖矿时，矿工可以从drand网络获取随机项，并将随机项打包在新区块中。

DrandBeacon将Lotus与drand网络连接起来，以一种与Filecoin轮数/epoch一致的方式为系统提供随机性。

我们通过DrandBeacon 对等节点的公共HTTP端点连接到drand对等节点。对等节点存储在drandServers的枚举变量中。

Drand链的根信任是从build.DrandChain配置的。

```
ype DrandBeacon struct {
	client dclient.Client

	pubkey kyber.Point

	// seconds
	interval time.Duration

	drandGenTime uint64
	filGenTime   uint64
	filRoundTime uint64

	cacheLk    sync.Mutex
	localCache map[uint64]types.BeaconEntry
}
```

区块的信标项：

```
func BeaconEntriesForBlock(ctx context.Context, bSchedule Schedule, epoch abi.ChainEpoch, parentEpoch abi.ChainEpoch, prev types.BeaconEntry) ([]types.BeaconEntry, error) {
	{
		parentBeacon := bSchedule.BeaconForEpoch(parentEpoch)
		currBeacon := bSchedule.BeaconForEpoch(epoch)
		if parentBeacon != currBeacon {
			// Fork logic
			round := currBeacon.MaxBeaconRoundForEpoch(epoch)
			out := make([]types.BeaconEntry, 2)
			rch := currBeacon.Entry(ctx, round-1)
			res := <-rch
			if res.Err != nil {
				return nil, xerrors.Errorf("getting entry %d returned error: %w", round-1, res.Err)
			}
			out[0] = res.Entry
			rch = currBeacon.Entry(ctx, round)
			res = <-rch
			if res.Err != nil {
				return nil, xerrors.Errorf("getting entry %d returned error: %w", round, res.Err)
			}
			out[1] = res.Entry
			return out, nil
		}
	}

	beacon := bSchedule.BeaconForEpoch(epoch)

	start := build.Clock.Now()

	maxRound := beacon.MaxBeaconRoundForEpoch(epoch)
	if maxRound == prev.Round {
		return nil, nil
	}

	// TODO: this is a sketchy way to handle the genesis block not having a beacon entry
	if prev.Round == 0 {
		prev.Round = maxRound - 1
	}

	cur := maxRound
	var out []types.BeaconEntry
	for cur > prev.Round {
		rch := beacon.Entry(ctx, cur)
		select {
		case resp := <-rch:
			if resp.Err != nil {
				return nil, xerrors.Errorf("beacon entry request returned error: %w", resp.Err)
			}

			out = append(out, resp.Entry)
			cur = resp.Entry.Round - 1
		case <-ctx.Done():
			return nil, xerrors.Errorf("context timed out waiting on beacon entry to come back for epoch %d: %w", epoch, ctx.Err())
		}
	}

	log.Debugw("fetching beacon entries", "took", build.Clock.Since(start), "numEntries", len(out))
	reverse(out)
	return out, nil
}
```

epoch的最大信标轮次

```
func (db *DrandBeacon) MaxBeaconRoundForEpoch(filEpoch abi.ChainEpoch) uint64 {
	// TODO: sometimes the genesis time for filecoin is zero and this goes negative
	latestTs := ((uint64(filEpoch) * db.filRoundTime) + db.filGenTime) - db.filRoundTime
	dround := (latestTs - db.drandGenTime) / uint64(db.interval.Seconds())
	return dround
}
```

**验证接收区块上的信标项**
Filecoin区块链将包含从Filecoin创世区块到当前区块的信标链输出的全部内容。

考虑到信标项在领导者选举和Filecoin其他关键协议中所扮演的角色，一个区块的信标项必须对每个区块进行验证。详情见drand部分。通过确保每个信标项都是区块链中前一个信标项的有效签名，这可以rand 's Verify端点来实现这一点（校验区块值）:

```
func ValidateBlockValues(bSchedule Schedule, h *types.BlockHeader, parentEpoch abi.ChainEpoch,
	prevEntry types.BeaconEntry) error {
	{
		parentBeacon := bSchedule.BeaconForEpoch(parentEpoch)
		currBeacon := bSchedule.BeaconForEpoch(h.Height)
		if parentBeacon != currBeacon {
			if len(h.BeaconEntries) != 2 {
				return xerrors.Errorf("expected two beacon entries at beacon fork, got %d", len(h.BeaconEntries))
			}
			err := currBeacon.VerifyEntry(h.BeaconEntries[1], h.BeaconEntries[0])
			if err != nil {
				return xerrors.Errorf("beacon at fork point invalid: (%v, %v): %w",
					h.BeaconEntries[1], h.BeaconEntries[0], err)
			}
			return nil
		}
	}

	// TODO: fork logic
	b := bSchedule.BeaconForEpoch(h.Height)
	maxRound := b.MaxBeaconRoundForEpoch(h.Height)
	if maxRound == prevEntry.Round {
		if len(h.BeaconEntries) != 0 {
			return xerrors.Errorf("expected not to have any beacon entries in this block, got %d", len(h.BeaconEntries))
		}
		return nil
	}

	if len(h.BeaconEntries) == 0 {
		return xerrors.Errorf("expected to have beacon entries in this block, but didn't find any")
	}

	last := h.BeaconEntries[len(h.BeaconEntries)-1]
	if last.Round != maxRound {
		return xerrors.Errorf("expected final beacon entry in block to be at round %d, got %d", maxRound, last.Round)
	}

	for i, e := range h.BeaconEntries {
		if err := b.VerifyEntry(e, prevEntry); err != nil {
			return xerrors.Errorf("beacon entry %d (%d - %x (%d)) was invalid: %w", i, e.Round, e.Data, len(e.Data), err)
		}
		prevEntry = e
	}

	return nil
}
```

###### Ticket
Filecoin区块头还包含一个由其epoch的信标项生成的“ticket”。在分叉选择规则中，ticket被用来打破分叉重量的相等的场景。

每当比较Filecoin中的Ticket时，比较的是ticket的VRF摘要字节。

* 随机性Ticket生成

在Filecoin epoch n处，使用合适的信标项为epoch n生成一个新Ticket。

矿工（区块生产者）通过可验证随机函数（Verifiable Random Function, VRF）运行信标项，以获得新的唯一Ticket。信标项以Ticket域分离标记为前缀，并与矿工（区块生产者）的actor地址连接（以确保使用相同worker密钥的矿工获得不同的ticket）。

为给定的epoch n生成一个Ticket：

```
randSeed = GetRandomnessFromBeacon(n)
newTicketRandomness = VRF_miner(H(TicketProdDST || index || Serialization(randSeed, minerActorAddress)))
```

生成Ticket使用的是（Verifiable Random Functions）。

* Ticket验证

每个Ticket都应该从VRF链中的前一个票证中生成，并进行相应的验证。



###### 最小的矿工大小
为了保证存储算力共识的安全性，Filecoin系统定义了参与共识所需的最小算力大小。

具体来说，矿工（区块生产者）必须至少有一个MIN_MINER_SIZE_STOR的算力（即存储交易中目前使用的存储算力），才能参与领导者选举。如果没有矿工（区块生产者）具有MIN_MINER_SIZE_STOR或更大算力，则最大的MIN_MINER_SIZE_TARG（按存储功率排序）个矿工（区块生产者）将能够参加领导者选举。用简单的英语来说，以MIN_MINER_SIZE_TARG = 3为例，这意味着拥有至少是第三大矿工（区块生产者）才有资格参与共识。

小于这个值的矿工不能生产区块，也不能网络中获得区块奖励。他们的算力仍将计入总网络（原始或声称的）存储算力，即使他们的算力不会被计入领导者选举的选票。然而，重要的是要注意，这样的矿工仍然可以有他们的算力故障和相应的惩罚。

因此，要启动网络，创世区块必须包括初始化好的矿工，可能只是CommittedCapacity扇区，以启动网络。

只要任何一个矿工（区块生产者）拥有超过MIN_MINER_SIZE_STOR算力，MIN_MINER_SIZE_TARG条件将不会在网络中使用。尽管如此，它仍被定义为确保小型网络（例如接近创世或在大量算力下降之后）的活性。



###### Storage Power Actor

StoragePowerActorState实现

```
type State struct {
	TotalRawBytePower abi.StoragePower
	// TotalBytesCommitted includes claims from miners below min power threshold
	TotalBytesCommitted  abi.StoragePower
	TotalQualityAdjPower abi.StoragePower
	// TotalQABytesCommitted includes claims from miners below min power threshold
	TotalQABytesCommitted abi.StoragePower
	TotalPledgeCollateral abi.TokenAmount

	// These fields are set once per epoch in the previous cron tick and used
	// for consistent values across a single epoch's state transition.
	ThisEpochRawBytePower     abi.StoragePower
	ThisEpochQualityAdjPower  abi.StoragePower
	ThisEpochPledgeCollateral abi.TokenAmount
	ThisEpochQAPowerSmoothed  smoothing.FilterEstimate

	MinerCount int64
	// Number of miners having proven the minimum consensus power.
	MinerAboveMinPowerCount int64

	// A queue of events to be triggered by cron, indexed by epoch.
	CronEventQueue cid.Cid // Multimap, (HAMT[ChainEpoch]AMT[CronEvent])

	// First epoch in which a cron task may be stored.
	// Cron will iterate every epoch between this and the current epoch inclusively to find tasks to execute.
	FirstCronEpoch abi.ChainEpoch

	// Claimed power for each miner.
	Claims cid.Cid // Map, HAMT[address]Claim

	ProofValidationBatch *cid.Cid // Multimap, (HAMT[Address]AMT[SealVerifyInfo])
}
```

StoragePowerActor实现

```
func (a Actor) Exports() []interface{} {
	return []interface{}{
		builtin.MethodConstructor: a.Constructor,
		2:                         a.CreateMiner,
		3:                         a.UpdateClaimedPower,
		4:                         a.EnrollCronEvent,
		5:                         a.OnEpochTickEnd,
		6:                         a.UpdatePledgeTotal,
		7:                         nil, // deprecated
		8:                         a.SubmitPoRepForBulkVerify,
		9:                         a.CurrentTotalPower,
	}
}
```

MinerConstructorParams

存储矿工（？？）actor的构造函数参数在这里定义，以便power actor可以将它们发送给init actor来实例化矿机。从v0以来已经有改变，增加了ControlAddrs：

```
type MinerConstructorParams struct {
	OwnerAddr     addr.Address
	WorkerAddr    addr.Address
	ControlAddrs  []addr.Address
	SealProofType abi.RegisteredSealProof
	PeerId        abi.PeerID
	Multiaddrs    []abi.Multiaddrs
}
```



**算力表（PoS的关键组成部分）**

一个给定的矿工（？？）通过EC中的领导者选举产生的区块的部分（所以他们所获得的区块奖励）与他们的质量调整后的算力比例（Quality-Adjusted Power Fraction）成正比。也就是说，如果一个矿工的质量调整算力代表了网络上总质量调整算力的1%，那么它应该按照预期生产1%的区块。

SPC提供了一个算力表抽象，它可以随着时间的推移跟踪矿工（？？）的算力（即矿工存储与网络存储总量的关系)。算力表将针对新的扇区承诺（增加矿机算力）、失败的post（减少矿工算力）或其他存储和共识故障进行更新。

Sector ProveCommit是第一次提交算力证明给网络，因此算力是第一次添加在成功的扇区ProveCommit。当一个扇区被宣布恢复时，算力也会被增加。矿工（？？）需要提交所有对算力有贡献的扇区的证明。

当一个扇区过期，或者当一个扇区被声明或检测到故障，或者当它通过矿工（？？）调用被终止时，算力就会下降。矿工还可以通过ExtendSectorExpiration延长一个扇区的生命周期。

power表中的Miner生命周期大致如下:

* MinerRegistration：存储挖矿子系统将一个关联worker公钥和地址的新miner，以及它们关联的扇区大小（每个worker只有一个扇区大小）注册到power表中。
* UpdatePower：这些算力的增加和减少由不同的存储Actor调用（因此必须由网络上的每个完整节点进行验证）。具体地说:
  * 权力在SectorProveCommit增加
  * 在错过WindowPoSt (DetectedFault)之后，分区所对应的算力立即下降（TODO：第二天才下降？）。
  * 当通过“声明错误Declared Faults ”或“跳过错误Skipped Faults”进入故障状态时，特定扇区的算力会下降。
  * 某一特定扇区被宣布恢复并被PoSt证实后，算力回归。
  * 当某个特定扇区过期或者被矿工（？）宣布终止时，该扇区的算力将被取消。

总而言之，只有处于Active状态的扇区才能控制算力。在ProveCommit上添加扇区时，扇区变为激活状态。当它进入故障状态时，算力立即减少。当宣布的恢复得到证实时，算力就会恢复。当一个扇区的算力过期或通过矿工（？？）调用终止时，它将被删除。

**质押惩罚**
对于任何影响存储算力共识的故障，抵押品被大幅削减，包括:

* 特别是预期共识的错误（见共识错误），这将由一个slasher报告给StoragePowerActor以换取奖励。
* 影响算力共识更普遍的故障，特别是未提交算力故障（即存储故障），这将由CronActor自动报告或当矿工早于其承诺的持续时间终止一个扇区。

有关质押的更详细的讨论，请参阅矿工质押部分。