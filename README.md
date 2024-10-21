## Proof of Staked Authority(POSA)

POSA:DPOS+POA

BSC here proposes to combine DPoS and PoA for consensus, so that:

1. Blocks are produced by a limited set of validators
2. Validators take turns to produce blocks in a PoA manner, similar to [Ethereum’s Clique](https://eips.ethereum.org/EIPS/eip-225) consensus design
3. Validator set are elected in and out based on a staking based governance

## Validators：

BSC conducts a daily election process post **00:00 UTC** to select the top **45** active validators based on their staking rankings for block production. Among these, the **21** validators with the highest staked amounts are referred to as **Cabinets**, while the remaining **24** validators are known as **Candidates**. The remaining inactive validators must wait for the next round of elections to become active validators before they can participate in block production.

**In the set of 45 active validators, each epoch selects 18 validators from the Cabinets and 3 validators from the Candidates, forming a group of 21 validators as the consensus validators set for the current epoch to produce blocks.** If a validator is elected as the consensus validator but fails to participate in produce blocks, it will face slashing consequences.（18/21+3/24=21/45）

初始从extra字段中获取验证者信息|---Extra Vanity---|---Validators Number（1byte） and Validators Bytes（20byte） (or Empty)---|---Vote Attestation(48byte) (or Empty)---|---Extra Seal---|，按照地址字典顺序排序，每个epoch（源码中设置为200）从智能合约中更新验证者列表。

```go
//Snapshot.go
func parseValidators(){
	n := len(validatorsBytes) / validatorBytesLength
	cnsAddrs := make([]common.Address, n)
	voteAddrs := make([]types.BLSPublicKey, n)
	for i := 0; i < n; i++ {
		cnsAddrs[i] = common.BytesToAddress(validatorsBytes[i*validatorBytesLength : i*validatorBytesLength+common.AddressLength])
		copy(voteAddrs[i][:], validatorsBytes[i*validatorBytesLength+common.AddressLength:(i+1)*validatorBytesLength])
	}
}
```

#### 判断出块节点是否为inturn

##### 若当前区块高度除以验证者个数的余数等于该节点在验证者集中的下标，则该节点为inturn节点

```go
func (s *Snapshot) inturn(validator common.Address) bool {
	validators := s.validators()
	offset := (s.Number + 1) % uint64(len(validators))
	return validators[offset] == validator
}
```

## Fast Finality

**Finality is critical for blockchain security, once the block is finalized, it wouldn’t be reverted anymore.** The fast finality feature is very useful, the users can make sure they get the accurate information from the latest finalized block, then they can decide what to do next instantly. More details of design, please to refer [BEP-126](https://github.com/bnb-chain/BEPs/blob/master/BEPs/BEP126.md)

Before the coming Plato upgrade,to secure as much as BC, BSC users are encouraged to wait until receiving blocks sealed by more than ⅔*N+1 different validators. In that way, the BSC can be trusted at a similar security level to BC and can tolerate less than ⅓*N Byzantine validators.With 21 validators, **if the block time is 3 seconds, the ⅔*N+1 different validator seals will need a time period of (⅔*21+1)*3 = 45 seconds. Any critical applications for BSC may have to wait for ⅔*N+1 to ensure a relatively secure finality.** With above enhancement by slashing mechanism, ½*N+1 or even fewer blocks are enough as confirmation for most transactions.

After the coming Plato upgrade, the feature will be enabled. The chain will be finalized within two blocks if ⅔*N or more validators vote normally, **otherwise the chain has a fixed number of blocks to reach probabilistic finality as before.`Fast Finality`**

It takes several steps to finalize a block:

1. A block is proposed by a validator and propagated to other validators(单个验证者提出区块并广播)

   Validators use their BLS private key to sign for the block as a vote message(投票消息为BLS签名)

2. Gather the votes from validators into a pool(票池收集所有票数，本地会维护一个票池，通过网络同步)

3. Aggregate the BLS signature if its direct parent block has gotten enough votes when proposing a new block（带有BLS私钥的直接父区块在提议新区块时获得了足够的票数，则聚合 BLS 签名）

4. Set the aggregated vote attestation into the extra field of the new block's header(聚合签名成为证明保存至块头)

5. Validators and full nodes who received the new block with the direct parent block's attestation can justify the direct parent block(收到带有直接父区块证明的新区块的验证者和完整节点可以证明直接父区块的正确性)

6. If there are two continuous blocks have been justified, then the previous one is finalized(如果有两个连续的区块已证明合理，则前一个区块最终确定)

### Finality Rules：n+1和n+2两个区块有证明则n被确认

justify:

(1) it is the root, or

(2) there exists attestation for this block in its direct child’s header, we call this block justified（直接子块包含证明则这个块为justified block）

finalize:

(1) it is the root, or

(2) it is justified and its direct child is justified（该块和子块为justified block）

### 投票的结构：一般表示为[SourceNumber->TargetNumber]比如[1->2]

```go
SourceNumber: justifiedBlockNumber,
SourceHash:   justifiedBlockHash,
TargetNumber: parent.Number.Uint64(),
TargetHash:   parent.Hash(),
```

#### 获取Justified block：最新包含证明块的Target指向块

```go
func (p *Parlia) GetJustifiedNumberAndHash(){
	...
	head := headers[len(headers)-1]
	snap= p.snapshot(chain, head.Number.Uint64(), head.Hash(), headers)
	if snap.Attestation == nil {
		return 0, chain.GetHeaderByNumber(0).Hash(), nil
	}
	return snap.Attestation.TargetNumber, snap.Attestation.TargetHash, nil
}
```

#### 获取finalized block：最新包含证明块的Source指向块

```go
func (p *Parlia) GetFinalizedHeader(){
	snap, err := p.snapshot(chain, header.Number.Uint64(), header.Hash(), nil)
	if snap.Attestation == nil {
		return chain.GetHeaderByNumber(0) // keep consistent with GetJustifiedNumberAndHash
	}
	return chain.GetHeader(snap.Attestation.SourceHash, snap.Attestation.SourceNumber)
}
```

#### assembleVoteAttestation

```go
func (p *Parlia) assembleVoteAttestation(){
    if p.VotePool == nil {
		return 
	}
    parent := chain.GetHeaderByHash(header.ParentHash)
	snap, err := p.snapshot(chain, parent.Number.Uint64()-1, parent.ParentHash, nil)
	votes := p.VotePool.FetchVoteByBlockHash(parent.Hash())
    if len(votes) < 2/3 {
		return 
	}
    justifiedBlockNumber, justifiedBlockHash := p.GetJustifiedNumberAndHash(chain, []*types.Header{parent})
	attestation := &types.VoteAttestation{
		Data: &types.VoteData{
			SourceNumber: justifiedBlockNumber,
			SourceHash:   justifiedBlockHash,
			TargetNumber: parent.Number.Uint64(),
			TargetHash:   parent.Hash(),
		},
	}
    // Prepare aggregated vote signature
	voteAddrSet := make(map[types.BLSPublicKey]struct{}, len(votes))
	signatures := make([][]byte, 0, len(votes))
	for _, vote := range votes {
		voteAddrSet[vote.VoteAddress] = struct{}{}
		signatures = append(signatures, vote.Signature[:])
	}
	sigs, err := bls.MultipleSignaturesFromBytes(signatures)
}
```

### Validator Vote Rules

投票时机为验证者在收到新区块时进行投票（即使先收到当前高度的其他验证者的出块也会直接投票，并不是优先自己出的块）

- A validator must not publish two distinct votes for the same height. (Rule 1)（验证者最多必须对任何目标 epoch 进行一次投票）
- A validator must not vote within the span of its other votes . (Rule 2)（不可以同时投出类似[2->3]和[1->4]的票）
- Validators always vote for their canonical chain’s latest block. (Rule 3)

![image-20240919162108084](img/image-20240919162108084.png)

#### 投票保存范围

```go
// votes in the range (currentBlockNum-256,currentBlockNum+11] will be stored
lowerLimitOfVoteBlockNumber = 256
upperLimitOfVoteBlockNumber = 11 // refer to fetcher.maxUncleDist
```

#### core/vote/votemanager.go：投票主函数

```go
func (voteManager *VoteManager) loop() {
	defer voteManager.chainHeadSub.Unsubscribe()
	defer voteManager.syncVoteSub.Unsubscribe()

	events := voteManager.eth.EventMux().Subscribe(downloader.StartEvent{}, downloader.DoneEvent{}, downloader.FailedEvent{})  //订阅与下载器相关的事件（StartEvent、DoneEvent、FailedEvent），以便在这些事件发生时触发相应的逻辑
	defer func() {
		log.Debug("vote manager loop defer func occur")
		if !events.Closed() {
			log.Debug("event not closed, unsubscribed by vote manager loop")
			events.Unsubscribe()
		}
	}()

	dlEventCh := events.Chan()  //获取事件通道 dlEventCh，用于接收下载器相关的事件

	startVote := true
	blockCountSinceMining := 0
	var once sync.Once
	for {
		select {
		case ev := <-dlEventCh:
			if ev == nil {
				log.Debug("dlEvent is nil, continue")
				continue
			}
			switch ev.Data.(type) {
			case downloader.StartEvent:
				log.Warn("downloader is in startEvent mode, will not startVote")
				startVote = false
			case downloader.FailedEvent:
				log.Info("downloader is in FailedEvent mode, set startVote flag as true")
				startVote = true
			case downloader.DoneEvent:
				log.Info("downloader is in DoneEvent mode, set the startVote flag to true")
				startVote = true
			}
		case cHead := <-voteManager.chainHeadCh:
			if !startVote {
				log.Warn("startVote flag is false, continue")
				continue
			}
			if !voteManager.eth.IsMining() {
				blockCountSinceMining = 0
				log.Warn("skip voting because mining is disabled, continue")
				continue
			}
			blockCountSinceMining++
			if blockCountSinceMining <= blocksNumberSinceMining {
				log.Warn("skip voting", "blockCountSinceMining", blockCountSinceMining, "blocksNumberSinceMining", blocksNumberSinceMining)//检查挖矿状态: 启动节点后挖矿数小于设定跳过投票
				continue
			}

			if cHead.Block == nil {
				log.Warn("cHead.Block is nil, continue")
				continue
			}

			curHead := cHead.Block.Header()
			// Check if cur validator is within the validatorSet at curHead
			if !voteManager.engine.IsActiveValidatorAt(voteManager.chain, curHead,
				func(bLSPublicKey *types.BLSPublicKey) bool {
					return bytes.Equal(voteManager.signer.PubKey[:], bLSPublicKey[:])
				}) {
				log.Warn("cur validator is not within the validatorSet at curHead")
				continue
			}

			// Add VoteKey to `miner-info`
			once.Do(func() {
				log.Info("Add VoteKey to 'miner-info'")
				minerInfo := metrics.Get("miner-info")
				if minerInfo != nil {
					minerInfo.(metrics.Label).Value()["VoteKey"] = common.Bytes2Hex(voteManager.signer.PubKey[:])
				}
			})

			// 创建投票数据，并签名。
			log.Info("start voting")
			vote := &types.VoteData{
				TargetNumber: curHead.Number.Uint64(),
				TargetHash:   curHead.Hash(),
			}
			voteMessage := &types.VoteEnvelope{
				Data: vote,
			}

			// 如果签名和写入日志都成功，将投票消息放入投票池。
			if ok, sourceNumber, sourceHash := voteManager.UnderRules(curHead); ok {//检查是否符合投票规则
				log.Info("curHead is underRules for voting")
				if sourceHash == (common.Hash{}) {
					log.Info("sourceHash is empty")
					continue
				}

				voteMessage.Data.SourceNumber = sourceNumber
				voteMessage.Data.SourceHash = sourceHash

				if err := voteManager.signer.SignVote(voteMessage); err != nil {
					log.Error("Failed to sign vote", "err", err, "votedBlockNumber", voteMessage.Data.TargetNumber, "votedBlockHash", voteMessage.Data.TargetHash, "voteMessageHash", voteMessage.Hash())
					votesSigningErrorCounter.Inc(1)
					continue
				}
				if err := voteManager.journal.WriteVote(voteMessage); err != nil {
					log.Error("Failed to write vote into journal", "err", err)
					voteJournalErrorCounter.Inc(1)
					continue
				}

				log.Info("vote manager produced vote", "votedBlockNumber", voteMessage.Data.TargetNumber, "votedBlockHash", voteMessage.Data.TargetHash, "voteMessageHash", voteMessage.Hash())
				voteManager.pool.PutVote(voteMessage)
				votesManagerCounter.Inc(1)
			}
		case event := <-voteManager.syncVoteCh://从 syncVoteCh 通道中获取同步投票事件
			voteMessage := event.Vote
			if voteManager.eth.IsMining() || !bytes.Equal(voteManager.signer.PubKey[:], voteMessage.VoteAddress[:]) {
				continue
			}
			if err := voteManager.journal.WriteVote(voteMessage); err != nil {
				log.Error("Failed to write vote into journal", "err", err)
				voteJournalErrorCounter.Inc(1)
				continue
			}
			log.Info("vote manager synced vote", "votedBlockNumber", voteMessage.Data.TargetNumber, "votedBlockHash", voteMessage.Data.TargetHash, "voteMessageHash", voteMessage.Hash())
			votesManagerCounter.Inc(1)
		case <-voteManager.syncVoteSub.Err():
			log.Debug("voteManager subscribed votes failed")
			return
		case <-voteManager.chainHeadSub.Err():
			log.Debug("voteManager subscribed chainHead failed")
			return
		}
	}
}
```

#### 检查投票规则

```go
// UnderRules checks if the produced header under the following rules:
// A validator must not publish two distinct votes for the same height. (Rule 1)
// A validator must not vote within the span of its other votes . (Rule 2)
// Validators always vote for their canonical chain’s latest block. (Rule 3)
func (voteManager *VoteManager) UnderRules(header *types.Header) (bool, uint64, common.Hash) {
	sourceNumber, sourceHash, err := voteManager.engine.GetJustifiedNumberAndHash(voteManager.chain, []*types.Header{header})
	if err != nil {
		log.Error("failed to get the highest justified number and hash at cur header", "curHeader's BlockNumber", header.Number, "curHeader's BlockHash", header.Hash())
		return false, 0, common.Hash{}
	}

	targetNumber := header.Number.Uint64()

	voteDataBuffer := voteManager.journal.voteDataBuffer
	//Rule 1:  A validator must not publish two distinct votes for the same height.
	if voteDataBuffer.Contains(targetNumber) {
		log.Warn("err: A validator must not publish two distinct votes for the same height.")
		return false, 0, common.Hash{}
	}

	//Rule 2: A validator must not vote within the span of its other votes.
	blockNumber := sourceNumber + 1
	if blockNumber+maliciousVoteSlashScope < targetNumber {
		blockNumber = targetNumber - maliciousVoteSlashScope
	}
	for ; blockNumber < targetNumber; blockNumber++ {
		if voteDataBuffer.Contains(blockNumber) {
			voteData, ok := voteDataBuffer.Get(blockNumber)
			if !ok {
				log.Error("Failed to get voteData info from LRU cache.")
				continue
			}
			if voteData.(*types.VoteData).SourceNumber > sourceNumber {
				log.Error(fmt.Sprintf("error: cur vote %d-->%d is across the span of other votes %d-->%d",
					sourceNumber, targetNumber, voteData.(*types.VoteData).SourceNumber, voteData.(*types.VoteData).TargetNumber))
				return false, 0, common.Hash{}
			}
		}
	}
	for blockNumber := targetNumber + 1; blockNumber <= targetNumber+upperLimitOfVoteBlockNumber; blockNumber++ {
		if voteDataBuffer.Contains(blockNumber) {
			voteData, ok := voteDataBuffer.Get(blockNumber)
			if !ok {
				log.Error("Failed to get voteData info from LRU cache.")
				continue
			}
			if voteData.(*types.VoteData).SourceNumber < sourceNumber {
				log.Error(fmt.Sprintf("error: cur vote %d-->%d is within the span of other votes %d-->%d",
					sourceNumber, targetNumber, voteData.(*types.VoteData).SourceNumber, voteData.(*types.VoteData).TargetNumber))
				return false, 0, common.Hash{}
			}
		}
	}

	// Rule 3: Validators always vote for their canonical chain’s latest block.
	// Since the header subscribed to is the canonical chain, so this rule is satisfied by default.
	log.Info("All three rules check passed")
	return true, sourceNumber, sourceHash
}
```



## Reward Rules:

奖励在每个epoch结束时按权重分配

- Validators whose vote is wrapped into the vote attestation can get one weight for reward(被包装到投票证明中的投票的验证者可以获得一个权重的奖励)
- Validators who assemble vote attestation can get additional weights. The number of weights is equal to the number of extra votes than required（进行投票证明组装的验证者可获得额外权重。权重的数量等于比所需票数多出的票数）
- The total reward is equal to the amount our system reward contract has grown over the last epoch. If the value of the system reward contract hits the upper limit, 1 BNB will be distributed.(总奖励等于系统奖励合约比上一个epoch的增长量。 如果系统奖励合约的价值达到上限，将分配 1 BNB)

## forkchoice:

The new longest chain rule can be described as follows.

1. The fork that includes the higher justified block is considered as the longest chain.
2. When the justified block is the same, fall back to compare the sum of the “Difficulty” field.

#### core/blockchain.go

```go
//writeBlockAndSetHead函数：在插入新块时检测是否需要对链重组
func writeBlockAndSetHead(){
    currentBlock = getCurrentBlock()
	reorg, err = ReorgNeededWithFastFinality(currentBlock, block.Header())//判断justifiedNumber是否相同
    if reorg {
		// Reorganise the chain if the parent is not the head block
		if block.ParentHash() != currentBlock.Hash() {//存在分叉
			reorg(currentBlock, block); err != nil {
		}
}
        
func ReorgNeededWithFastFinality(){
    justifiedNumber = f.chain.GetJustifiedNumber(header)
	curJustifiedNumber = f.chain.GetJustifiedNumber(current)
    if justifiedNumber == curJustifiedNumber {
		return ReorgNeeded(current, header)//若justifiedNumber相同则判断难度值总和
	}
    return justifiedNumber > curJustifiedNumber, nil
}

```

#### bsc/core/forkchoice.go

```go
func (f *ForkChoice) ReorgNeeded(current *types.Header, extern *types.Header) (bool, error) {
	//分别统计新旧链难度值
    var (
		localTD  = f.chain.GetTd(current.Hash(), current.Number.Uint64())
		externTd = f.chain.GetTd(extern.Hash(), extern.Number.Uint64())
	)
	if localTD == nil || externTd == nil {
		return false, errors.New("missing td")
	}
	// Accept the new header as the chain head if the transition
	// is already triggered. We assume all the headers after the
	// transition come from the trusted consensus layer.
	if ttd := f.chain.Config().TerminalTotalDifficulty; ttd != nil && ttd.Cmp(externTd) <= 0 {
		return true, nil
	}

	// If the total difficulty is higher than our known, add it to the canonical chain
	if diff := externTd.Cmp(localTD); diff > 0 {
		return true, nil
	} else if diff < 0 {
		return false, nil
	}
	// Local and external difficulty is identical.
	// Second clause in the if statement reduces the vulnerability to selfish mining.
	// Please refer to http://www.cs.cornell.edu/~ie53/publications/btcProcFC.pdf
	reorg := false
	externNum, localNum := extern.Number.Uint64(), current.Number.Uint64()
	if externNum < localNum {
		reorg = true
	} else if externNum == localNum {//若难度值相同随机选择
		var currentPreserve, externPreserve bool
		if f.preserve != nil {
			currentPreserve, externPreserve = f.preserve(current), f.preserve(extern)
		}
		reorg = !currentPreserve && (externPreserve || f.rand.Float64() < 0.5)
	}
	return reorg, nil
}
```

## Slashing

#### Double Sign：

- Two block headers have the same height and the same parent block hash
- Two block headers are sealed by the same validator
- Two signatures of these two blocks must not be the same
- The time of these two blocks must be within the validity of the evidence, which is 24 hours

If the evidence is valid:

1. **200BNB** would be slashed from the **self-delegated** BNB of the validator
2. The remaining slashed BNB will be allocated to the credit addresses of validators participating in the next distribution
3. Set the validator with a duration of **30 days**, and remove it from the active validator set`jailed`

#### Malicious Vote：

- The target number voted by two votes lags behind the block header of the canonical chain by no more than 256
- The source numbers of the two votes are both smaller than their respective target numbers
- The source hash and target hash of the two votes are both not equal
- The target number of the two votes is the same or the span of one vote includes the span of the other vote
- The two votes are signed by the same voting key, and the verification of signatures are both passed
- The voting key used for signing is in the list sent by the last two breathe blocks

If the evidence is valid:

1. **200BNB** would be slashed from the **self-delegated** BNB of the validator
2. **5BNB** would allocate to the submitter from the system reward contract as a reward if the validator is active when the evidence submitted
3. The remaining slashed BNB will be allocated to the credit addresses of validators participating in the next distribution
4. Set the validator with a duration of **30 days**, and remove it from the active validator set`jailed`

#### Unavailability：

If a validator misses over 50 blocks in 24 hours, they will not receive the block reward; instead, it will be shared among other validators.

If a validator misses more than 150 blocks in 24 hours:

1. **10BNB** would be slashed from the **self-delegated** BNB of the validator
2. The slashed BNB will be allocated to the credit addresses of validators participating in the next distribution
3. Set the validator with a duration of **2 days**, and remove it from the active validator set`jailed`

## Produce block

#### 1: Prepare

- If (height % epoch)==0, 从合约中获取ValidatorSet

#### 2: FinalizeAndAssemble

- If the validator is not the in turn validator, will call liveness slash contract to slash the expected validator and generate a slashing transaction.
- If there is gas-fee in the block, will distribute **1/16** to system reward contract, the rest go to validator contract.

#### 3: Seal

The final step before a validator broadcast the new block.

- 如果轮不到验证者出块，则会等待一个随机时间

## Validate block

#### 1: VerifyHeader

Verify the block header when receiving a new block.

- Verify the signature in the `extraData` of `blockheader`
- Compare the block time of the and the expected block time that the signer suppose to use, will deny a that is smaller than expected. It helps to prevent a selfish validator from rushing to seal a block.`blockHeader`
- The should be the signer and the difficulty should be expected value.`coinbase`

#### 2: Finalize

- If it is an epoch block, a validator node will fetch validatorSet from BSCValidatorSet and compare it with extra_data.
- If the block is not generate by inturn validatorvalidaror, will call slash contract. if there is gas-fee in the block, will distribute 1/16 to system reward contract, the rest go to validator contract.
- The transaction generated by the consensus engine must be the same as the tx in block.

# The parlia protocol of bsc

```go
validators, the set of validator;
vote = <sourceblock,targetblock>;
b, a block:
	parent
	sealer
	number
	difficulty

vote:
	address
	signature
	sourceblock
	targetblock

assembleVoteAttestation(header):
	while votepool do
		p<-header.parent
		votes-<votepool.get(p.hash)
		if len(votes)<2/3 then
            return
        j<-GetJustifiedblock()
        createAttestation(j.number,j.hash,p.number,p.hash)
        for v in votes
            signatures = append(vote.Signature)
        Serialize vote attestation and add to block header extra fields.

seal(header):
    while true do
        n <- header.number
        wait until sign-recently()
        offset <- (n + 1) % (len(validators))
        if validators[offset] = validator then
            header.difficulty = 2
            delay = time.until(header.time,0)
        else 
            header.difficulty = 1
            delay = 200ms + rand(0,(len(validators)/2+1)*500ms)
        assembleVoteAttestation(header)
        sign()
        broadcast()

vote(header):
	while isActiveValidator()
        sourceblock <- GetJustifiedblock()
        vote <- createVoteData(sourceblock,header)
        if isUnderRules(header,vote)
		//Rule 1: A validator must not publish two distinct votes for the same height.
		//Rule 2: A validator must not vote within the span of its other votes. 
		//Rule 3: Validators always vote for their canonical chain’s latest block.
		votepool.putVote()
             

```

#### 区块广播 bsc/eth/handler.go

```go
minedBroadcastLoop():
	for:
    	BroadcastBlock(Block, true)  // First propagate block to peers
        BroadcastBlock(Block, false) // Only then announce to the rest

BroadcastBlock(block, propagate):
	h = block.hash
	peers <- peersWithoutBlock(h)	//peersWithoutBlock retrieves a list of peers that do not have a given block in their set of known hashes, so it might be propagated to them.
	if propagate:
        if directBroadcast:		//似乎是一个可修改配置，原因为主网中区块传播延迟导致有一些空块，添加这个标志来缓解这种情况。
		transfer = peers
	else:
	transfer = peers[int(Sqrt(peers.len))]
	for peer in transfer {
		peer.AsyncSendNewBlock()
	}
	if HasBlock(block):
		for peer in peers {
			peer.AsyncSendNewBlockHash()
		}
```

