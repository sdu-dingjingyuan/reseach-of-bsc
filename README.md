# reseach-of-bsc
#### 出块人选择规则:按照地址字典顺序排序

```go
func (s *Snapshot) validators() []common.Address {
	validators := make([]common.Address, 0, len(s.Validators))
	for v := range s.Validators {
		validators = append(validators, v)
	}
	sort.Sort(validatorsAscending(validators))
	return validators
}
```

#### 分叉选择:难度值比较

bsc/core/forkchoice.go

```go
func (f *ForkChoice) ReorgNeeded(current *types.Header, extern *types.Header) (bool, error) {
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
	} else if externNum == localNum {
		var currentPreserve, externPreserve bool
		if f.preserve != nil {
			currentPreserve, externPreserve = f.preserve(current), f.preserve(extern)
		}
		reorg = !currentPreserve && (externPreserve || f.rand.Float64() < 0.5)
	}
	return reorg, nil
}
```

##### 获取Justified block

```go
func (p *Parlia) GetJustifiedNumberAndHash(chain consensus.ChainHeaderReader, headers []*types.Header) (uint64, common.Hash, error) {
	...
	head := headers[len(headers)-1]
	snap, err := p.snapshot(chain, head.Number.Uint64(), head.Hash(), headers)
	if snap.Attestation == nil {
		return 0, chain.GetHeaderByNumber(0).Hash(), nil
	}
	return snap.Attestation.TargetNumber, snap.Attestation.TargetHash, nil
}
```

##### 获取finalized block

```go
func (p *Parlia) GetFinalizedHeader(){
	snap, err := p.snapshot(chain, header.Number.Uint64(), header.Hash(), nil)
	if snap.Attestation == nil {
		return chain.GetHeaderByNumber(0) // keep consistent with GetJustifiedNumberAndHash
	}
	return chain.GetHeader(snap.Attestation.SourceHash, snap.Attestation.SourceNumber)
}
```

##### assembleVoteAttestation

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
}
```

##### vote

```go
func (voteManager *VoteManager) loop() {
	log.Info("vote manager routine loop started")
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
				log.Warn("skip voting", "blockCountSinceMining", blockCountSinceMining, "blocksNumberSinceMining", blocksNumberSinceMining)//检查挖矿状态: 如果当前不在挖矿中，跳过投票并重置 blockCountSinceMining
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
			if ok, sourceNumber, sourceHash := voteManager.UnderRules(curHead); ok {
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

