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