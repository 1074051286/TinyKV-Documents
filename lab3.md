## lab3a

### leader禅让

- 发送MsgTransferLeader给当前leader，并检查transfree的资质是否合格(transfree的log必须up-to-date)，否则当前leader会帮助transfree：
  - 如果transfree的日志没有up-to-date，当前的leader首先会停止接受新的命令，并给transfree发送MsgAppend命令补齐日志；
  - 当前leader给transfree发送MsgTimeoutNow，transfree给自己发送MsgHup立即开始选举，由于transfree有更大的term和更新的log，大概率会当选leader；

### 配置变更

- 这里的配置变更并不是raft paper中的共识算法，这里为了方便实现**只能一个一个的删除/添加节点。**由于是一个一个的添加/删除节点，因此ConfEntry必须保证严格有序，只有前一个apply了之后才能继续apply下一个。那么这个有序性是如何保证的呢？
  - 是通过PendingConfIndex保证的。当一个ConfEntry发送到集群中之前时，会将该ConfEntry的Index保存到ConfEntry，也就是说如果PendingConfIndex没有apply的话，PendingConfIndex是为0，如果该ConfEntry被apply就会调用`addNode()`或者`removeNode()`，这两个方法会将PendingConfIndex置0；
  - PendingConfIndex变为新的的前提是applyIndex >= oldPendingConfIndex，这说明旧的ConfEntry已经被apply。前提条件设置目的是防止用户一次向leader发送了多条ConfEntry(比如A，B，C)，PendingConfIndex会先设置为A的Index，如果A还有没有被applied，表明此时applied < r.PendingConfIndex，当遇到B、C是会continue跳过；
  - 上面的处理逻辑实在leader的`handlePropose()`中；
- 所以`addNode()`和`removeNode()`的实现逻辑如下：
  - addNode()：
    - 将PendingConfIndex设置为0，在Prs中注册新加入的peer节点；
  - removeNode()：
    - 将PendingConfIndex设置为0，在数据库中删除移除节点的信息。
    - 由于此时节点数量产生了变动，因此导致majority的标准产生了变化。所以leader要重新判断majority entry，自己在本地提交并发送消息让follower提交；

## lab3b

### leader transfer

- 判断entry是否过期；
- 调用`d.MsgTransferLeader`；
- 返回成功消息；

### change peer

- 开始之前还是需要先判断log是否过期；

- addNode
  - 由于添加了节点，因此peer_msg_handle对应的region的ConfVerson+1;
  - peer_msg_handle对应的region中的Peers信息加入新节点的storeID，在regions中更新该region，注册peerCache(peerID->peer)；
  - 持久化到磁盘；
- removeNode
  - 如果当前peer_msg_handle对应的peer就是要删除的peer，调用`d.destroyPeer()`销毁;
  - 在Peers中找到该Peer删除掉，ConfVerson+1，更新regions，在peerCache中删除该peer；

- 返回成功响应；

### region split
#### woker线程的分类

- woker线程全部都由raftstore的startWorkers开启

  ```go
  func (bs *Raftstore) startWorkers(peers []*peer) {
  	ctx := bs.ctx
  	workers := bs.workers
  	router := bs.router
  	bs.wg.Add(2) // raftWorker, storeWorker
  	rw := newRaftWorker(ctx, router)//start raftWorker
  	go rw.run(bs.closeCh, bs.wg)
  	sw := newStoreWorker(ctx, bs.storeState)//start storeWorker
  	go sw.run(bs.closeCh, bs.wg)
  	router.sendStore(message.Msg{Type: message.MsgTypeStoreStart, Data: ctx.store})
  	for i := 0; i < len(peers); i++ {
  		regionID := peers[i].regionId
  		_ = router.send(regionID, message.Msg{RegionID: regionID, Type: message.MsgTypeStart})
  	}
  	engines := ctx.engine
  	cfg := ctx.cfg
      //四种杂项woker的开启
      //splitCheckWorker/schedulerWorker/raftLogGCWorker/regionWorker
  	workers.splitCheckWorker.Start(runner.NewSplitCheckHandler(engines.Kv, NewRaftstoreRouter(router), cfg))
  	workers.regionWorker.Start(runner.NewRegionTaskHandler(engines, ctx.snapMgr))
  	workers.raftLogGCWorker.Start(runner.NewRaftLogGCTaskHandler())
  	workers.schedulerWorker.Start(runner.NewSchedulerTaskHandler(ctx.store.Id, ctx.schedulerClient, NewRaftstoreRouter(router)))
  	go bs.tickDriver.run()
  }
  ```
### 

split_check的woker线程会定期检查某个region是否满足spilt的条件(设置的固定大小)，如果满足就去找出split的key并发送出去：

```go
func (r *splitCheckHandler) Handle(t worker.Task) {
	spCheckTask, ok := t.(*SplitCheckTask)
	if !ok {
		log.Errorf("unsupported worker.Task: %+v", t)
		return
	}
	region := spCheckTask.Region
	regionId := region.Id
	log.Debugf("executing split check worker.Task: [regionId: %d, startKey: %s, endKey: %s]", regionId,
		hex.EncodeToString(region.StartKey), hex.EncodeToString(region.EndKey))
	key := r.splitCheck(regionId, region.StartKey, region.EndKey)
	if key != nil {
		_, userKey, err := codec.DecodeBytes(key)
		if err == nil {
			// It's not a raw key.
			// To make sure the keys of same user key locate in one Region, decode and then encode to truncate the timestamp
			key = codec.EncodeBytes(userKey)
		}
		msg := message.Msg{
			Type:     message.MsgTypeSplitRegion,
			RegionID: regionId,
			Data: &message.MsgSplitRegion{
				RegionEpoch: region.GetRegionEpoch(),
				SplitKey:    key,
			},
		}
		err = r.router.Send(regionId, msg)
		if err != nil {
			log.Warnf("failed to send check result: [regionId: %d, err: %v]", regionId, err)
		}
	} else {
		log.Debugf("no need to send, split key not found: [regionId: %v]", regionId)
	}
}

```

scheduler_task的worker线程会去scheduler中拿到新region的id，并发送admin命令执行split region：

```go
func (r *SchedulerTaskHandler) Handle(t worker.Task) {
	switch t.(type) {
	case *SchedulerAskSplitTask:
		r.onAskSplit(t.(*SchedulerAskSplitTask))
	case *SchedulerRegionHeartbeatTask:
		r.onHeartbeat(t.(*SchedulerRegionHeartbeatTask))
	case *SchedulerStoreHeartbeatTask:
		r.onStoreHeartbeat(t.(*SchedulerStoreHeartbeatTask))
	default:
		log.Errorf("unsupported worker.Task: %+v", t)
	}
}
//发送split命令到raft集群中
func (r *SchedulerTaskHandler) onAskSplit(t *SchedulerAskSplitTask) {
	resp, err := r.SchedulerClient.AskSplit(context.TODO(), t.Region)
	if err != nil {
		log.Error(err)
		return
	}

	aq := &raft_cmdpb.AdminRequest{
		CmdType: raft_cmdpb.AdminCmdType_Split,
		Split: &raft_cmdpb.SplitRequest{
			SplitKey:    t.SplitKey,
			NewRegionId: resp.NewRegionId,
			NewPeerIds:  resp.NewPeerIds,
		},
	}
	r.sendAdminRequest(t.Region.GetId(), t.Region.GetRegionEpoch(), t.Peer, aq, t.Callback)
}
```

提交AdminCmdType_Split命令后我们就需要执行AdminCmdType_Split命令：

- 三个检查，失败返回失败响应：

  - 检查该entry是否过期；

  - 检查传入的region和负责执行命令的peer_msg_handler绑定的peer对应的region是否相同；
  - 检查传入的splitKey是否在该region中存在；
  - 检查当前regionVersion是否过期；
- 创建new_region，设置old/new region的元信息(regionID，起始/结束键，peers列表)，old region的version+1。**版本号就是为了判断孤立节点的配置是否过期，**防止孤立节点成为leader后使用过期的配置：
  - ConfVer：当remove或者add成员是配置版本号+1;
  - Version：当region发生merge/split时版本号+1；
- 调用`createPeer()`创建一个new_peer。注意一个peer对应一个peer_msg_handler，这样每个机器上的peer_msg_handler都在new_region中创建了一个new_peer;
- 修改`storeMeta.regionRanges`中region的key范围，并把new_peer放入new_region中；
- region元信息落盘，开启new_peer计时器，响应成功；

## lab3c

### region balance

由于Scheduler需要定期知到每个region的信息才能知道负载均衡时该如何调度peer的位置，因此scheduler需要定期接收region心跳来获得region的负载信息。region的状态可以进行如下的切分：

- region没有split，但是因为传入region脑裂的原因导致传入region的两个version全部过期；
- region发生了split，这是就需要根据old region的key range来获取分裂后的两个新region；
- 注意传入的region中是最新的版本信息，raftCluster中是旧的版本信息，正常情况下region中的version是一 定 >= raftCluster中的version的

```go
// processRegionHeartbeat updates the region information.
func (c *RaftCluster) processRegionHeartbeat(region *core.RegionInfo) error {
	//Scheuler视角下的状态,region的信息最新的,也就说region的两个Version都要比			  RaftCluster本地的新;
	localRegion, _ := c.GetRegionByID(region.GetID())
    //没有split,传入的region过期
	if localRegion != nil {
		if !c.isRegionValid(region, localRegion) {
			staleRegion := &metapb.Region{
				Id:          region.GetID(),
				StartKey:    region.GetStartKey(),
				EndKey:      region.GetEndKey(),
				RegionEpoch: region.GetRegionEpoch(),
				Peers:       region.GetPeers(),
			}
			return ErrRegionIsStale(staleRegion, localRegion)
		}
	} else {
		//region发生split的情况，原本一个region的数据被shard成为了两个region
		validRegions := c.ScanRegions(region.GetStartKey(), region.GetEndKey(), -1)
		for _, validRegionInfo := range validRegions {
			validRegion, _ := c.GetRegionByID(validRegionInfo.GetID())
            ////没有split,传入的region过期
			if !c.isRegionValid(region, validRegion) {
				staleRegion := &metapb.Region{
					Id:          region.GetID(),
					StartKey:    region.GetStartKey(),
					EndKey:      region.GetEndKey(),
					RegionEpoch: region.GetRegionEpoch(),
					Peers:       region.GetPeers(),
				}
				return ErrRegionIsStale(staleRegion, localRegion)
			}
		}
	}
	c.putRegion(region)
	//一个server对应一个store，因此一个region会对应多个store，这些store都需要更新
	for storeId := range region.GetStoreIds() {
		c.updateStoreStatusLocked(storeId)
	}
	return nil
}
```

调度器会按照 `GetMinInterval` 设置的时间调用`schedule()`方法来进行负载均衡，这也是我们需要实现的方法。调度策略如下：

- 首先需要正序排序一遍选取region占用最大的store来move out，注意region的心跳rpc需要通过DownTime来判断是否过期/下线；
- 我们需要选取移动的region，**如果storeID和regionID都选出来的话也就唯一确定了要移动的peer**(peer = storeID+regionID)。region的选取遵循以下原则：
  - 首先根据store id在本机随机选取一个有pending peer在本机上的region，有pending peer的出现说明磁盘压力很大；
  - 否则再根据store id在本机随机选取一个leader在本机上的region，由于leader承担写操作，如果leader所在store压力过大的话会影响写入；
  - 否则根据store id在follower中随机选取一个；
- 最后我们要选取剩余空间最大的store进行放入。所以我们会对store中的region大小进行逆序排序，选择store中region最小的store来move in；
- 根据选出来的两个store判断此次的移动是否是有价值的，如果两个storeID的region差距过小，scheduler就会在两者之间来回移动，差距大话的就开始移动，否则循环找下一个store。如果都没有就不需要移动：
  - 两个store的差距大小必须大于二倍的region大小；
  - 还需要保证移动之后目标的store中的region大小仍然小于原store中的region大小；
