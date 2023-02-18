# lab2文档

lab2主要分为了三个部分，分别为：

- lab2A：实现Raft中各种**消息的接收与回复**和**成员状态变更**。
- lab2B：实现**RawNode**从而对Raft层进行**抽象**，供上层应用调用。
- lab2C：实现**Log GC**以及**SnapShot**的发送。

lab2A测试用例固定且tinyKV提供的抽象非常清洗，因此实现难度适中。但是由于lab2B和lab2C的流程复杂、需要阅读的代码量较大，再加之测试用例完全是随机的，导致这两个部分实现难度较大。下面的文档将分为三个部分，分别从**思路、难点**两个角度记录实验。

## lab2A

### 思路

在Raft层中，每个机器上的Raft首先会从badger底层存储引擎中恢复log entries以及hardState(无法推演得到的状态，如Vote、Term、Commited)。随后我们将各个机器上的Raft变为Follower并初始化计时器、统计信息等状态，等待ElectionTimeout后选举出leader并使用心跳维持集群运转。为了实现心跳超时和选举超时，在tinyKV的Raft中我们使用了**逻辑时钟**对时间进行抽象。在这里**不使用物理时钟**而使用逻辑时钟的原因以我个人的理解有两点：

- **实现简单**：elctionElapsed++/heartbeatElapsed++即可，不需要使用繁琐的时间处理函数。
- **没有时钟漂移**：个人认为这里不需要Spanner中的GPS时钟+原子钟+分布式处理来保证物理时钟绝对精确。

整个逻辑时钟的推进全部在`tick()`进行了抽象。如果**heartbeatElapsed+1 > r.heartbeatTimeout**，则leader要发送MsgHeartbeat。如果**elctionElapsed+1 > r.electionTimeout**，则重新发起选举(leader要保证收到的heartBeat数量peer数量的一半)。

由于Raft中的消息类型各种各样，我们需要了解tinyKV的Raft Layer中各个Msg的种类以及作用：

| 消息类型                              | 作用                                |
| :------------------------------------ | ----------------------------------- |
| pb.MessageType_MsgHup                 | 触发选举流程                        |
| pb.MessageType_MsgPropose             | 触发添加流程，发给leader            |
| pb.MessageType_MsgBeat                | 触发心跳流程，发给leader，          |
| pb.MessageType_MsgRequestVote         | candidate发起选举，要求follower投票 |
| pb.MessageType_MsgRequestVoteResponse | follower投票给candidate             |
| pb.MessageType_MsgHeartbeat           | leader发送心跳以便维持领导状态      |
| pb.MessageType_MsgHeartbeatResponse   | follower响应leader表明alive         |
| pb.MessageType_MsgAppend              | leader发送给follower让其添加日志    |
| pb.MessageType_MsgAppendResponse      | follower响应leader的MsgAppend       |
| pb.MessageType_MsgSnapshot            | leader发送SnapShot给follower        |
| pb.MessageType_MsgTimeoutNow          | 立即超时，用于leader变更            |
| pb.MessageType_MsgTransferLeader      | 进行leader变更                      |

Raft层在整个tinyKV体系中处于下层：上面的应用接收到client的proposal后会先放到Raft层进行主从备份，当备份完成后Raft层的状态会产生变化。若发生变更则反映给RawNode，这个变化会被lab3将要介绍的raft worker使用**for轮询检测**`HasReady()`捕捉。而Raft层一切开始的地方就在于`Step()`这个函数：

```go
func (r *Raft) Step(m pb.Message) error {
	// Your Code Here (2A).
	switch r.State {
	case StateFollower:
		switch m.MsgType {
		case pb.MessageType_MsgHeartbeat:
			r.handleHeartbeat(m)
		case pb.MessageType_MsgAppend:
			r.handleAppendEntries(m)
		case pb.MessageType_MsgHup:
			r.handleHup()
		case pb.MessageType_MsgRequestVote:
			r.handleRequestVote(m) //小于candidateTerm的Vote已经无效，因为是上一轮竞选的投票数据。
		case pb.MessageType_MsgSnapshot:
			r.handleSnapshot(m)
		case pb.MessageType_MsgTimeoutNow:
			r.handleMsgTimeoutNow(m)
		case pb.MessageType_MsgTransferLeader:
			r.handleMsgTransferLeader(m)
		}
	case StateCandidate:
		switch m.MsgType {
		case pb.MessageType_MsgHup:
			r.handleHup()
		case pb.MessageType_MsgHeartbeat:
			r.handleHeartbeat(m)
		case pb.MessageType_MsgAppend:
			r.handleAppendEntries(m)
		case pb.MessageType_MsgRequestVote:
			r.handleRequestVote(m)
		case pb.MessageType_MsgRequestVoteResponse:
			r.handleRequestVoteResponse(m)
		case pb.MessageType_MsgTimeoutNow:
			r.handleMsgTimeoutNow(m)

		}
	case StateLeader:
		switch m.MsgType {
		case pb.MessageType_MsgBeat: //MsgBeat是leader发给leader自己，收到后给peer发送heartBeat
			r.handleMsgBeat()
		case pb.MessageType_MsgPropose: //接受到客户端的请求(proposal)
			r.handlePropose(m)
		case pb.MessageType_MsgTransferLeader:
			r.handleMsgTransferLeader(m)
		case pb.MessageType_MsgHeartbeat:
			r.handleHeartbeat(m)
		case pb.MessageType_MsgHeartbeatResponse:
			r.handleHeartbeatResponse(m)
		case pb.MessageType_MsgAppend: //只有leader才会发送append和heartBeat
			//candidate给自己投一票，不会给同term的candidate投票
			r.handleAppendEntries(m)
		case pb.MessageType_MsgAppendResponse:
			r.handleAppendEntriesResponse(m)
		case pb.MessageType_MsgRequestVote:
			r.handleRequestVote(m)
		}
	}
	return nil
}
```

在`Step()`中，代码对整个Raft状态做了很好的抽象，其中分为了三个状态：**StateFollower**、**StateCandidate**和**StateLeader**。每个Msg都能在`Step()`中找到对应的角色从而实现消息的处理。

同理，tinyKV对角色的变更也在`becomeFollower()`、`becomeCandidate()`和`becomeLeader()`三个函数中进行了抽象。注意状态变更时要更新Term、currentLeader、计时器等信息。在官方文档中提及当变成leader时要提交**no-op(no operation) entry**，但是没有详细解释为什么，在这里我用Raft paper的**Figure6详细解释一下:**

- 可以看到Figure6中1号为leader，因为Index:7,Term:3,X<-5这条last commited log已被提交到majority，因此被commited。但是**最后一条uncommitedlog**(Index;8,Term3,X<-4)只存在于1号机和3号机(可能是leader只复制到了3号机就挂了)。鉴于Raft只能向客户端提供commited log的信息，倘若此时**客户端向1号机请求最后一条log**的信息，则**必须等待**新的proposal发送到leader时**此log随着更新的log一起commited之后才可以访问**(Figure8 Safty Argument)。这就会导致**集群的不可用**，使得集群可用性降低。所以要使得leader中uncommited entry快速提交，我们需要发送一个no-op entry使得uncommited log快速提交保证集群高可用。

下面以三大流程基础介绍一下Raft message流向：

- **选举流程(**MsgHup触发)
  - 首先发起选举的人调用`becomeCandidate()`。如果只有自己则直接升级为leader。
  - 向follower发送MsgRequestVote开启选举。
  - follower收到MsgRequestVote后比较与candidate的Term大小并保证log up-to-date。
  - follower发送MsgRequestVoteResponse指明成功或者失败。
  - leader统计票数，如果成功调用`becomeLeader()`，否则调用`becomeFollower()`重新开始选举。
- **心跳流程**(MsgBeat触发)
  - 给follower发送MessageType_MsgHeartbeat。
  - follower比较其与leader的Term大小，更新electionElapsed并发送MessageType_MsgHeartbeatResponse。
  - leader处理MsgHeartbeatResponse，发送MsgAppend给恢复成功的follower，至此跳到了添加流程。
- **添加流程**(MsgPropose触发)
  - 从当前的log entries中选取出uncommited entry添加到MsgAppend中发送给follower。
  - **leader的prevLogIndex与follower的lastLogIndex比较：**follower收到RPC后比较与leader的Term大小并确认发送方是当前的leader。随后我们要**判断follower.RaftLog.LastIndex()和MsgAppend中的Index(即prevLogIndex)大小关系。**如果entry.Index <= follower.RaftLog.LastIndex()且两者Term不相等则表明有垃圾日志，则值保留[0,entry.Index)的log并添加新的log，否则直接添加。然后我们更新commitedIndex，最后发送MsgRequestVoteResponse。
  - **失败重试，成功更新：**leader处理MsgAppendResponse。如果**失败则Next-1重发**，否则判断Append成功的数量是否为majority。**如果是则更新**commited index并再次将更新后的commited index放入MsgAppend中发送给follower让他们更新commited index。
  - follower依据Raft paper中的规则更新commited index，返回响应。
  - leader处理MsgAppendResponse(其实这里leader什么也不做)。

其他细节参考Raft paper即可。当然，为了封装，我们不能直接把Raft及内部方法暴露给外界调用，这样有背面向对象的设计思想。因此我们需要使用rawnode对Raft进行抽象上层应用提供响应的方法使用Raft。这里主要要实现的函数为`HasRady()`、`Ready()`、`Advance()`：

- `HasReady()`的作用是判断是否新的ready产生。那么什么时候产生新的ready呢？
  - 当前raft的softState和之前rawnode保存的softState不一致时产生新的ready。
  - 当前raft的hardState和之前rawnode保存的hardState不一致时产生新的ready。
  - pendingSnapshot不为空，也就是说有新的SnapShot产生时产生新的ready。
  - unstableEntries > 0时产生新的ready。
  - nextEnts(commited but not applied) > 0时产生新的ready。
  - msgs > 0时产生新的ready。

- `Ready()`就是返回给调用者当前Raft的新状态。涉及的状态同上，只要现在的某一项状态与之前rawnode保存的oldSoftState/oldHardState状态不一致更新即可。
- `Advance()`主要用于更新Raft中的状态。当log entry被应用后就需要调用`Advance()`推进状态(就是把当前状态变为旧的)。

### 难点

lab2a的难点主要有以下几个：

- MsgAppendResponse/MsgRequestVoteResponse的处理。
- storage.ents和r.raftLog.entries的关系。
- rawnode中ready的初始化和HardState的更新时机。

#### MsgAppendResponse/MsgRequestVoteResponse的处理

MsgAppendResponse处理的难点在于统计follower的match频次。我们假设*match=m*，由于有的follower可能只添加了部分log entries，有的也可能全添加了，这就导致了peer间的match信息可能不一样。因此为了保证commited成功，我们要选取在m尽可能大的情况况下保证*match==m*的peer个数为majority的match。

由于每个follower都会调用一次`handleAppendEntriesResponse()` ，因此我们不能简单粗暴的使用一个局部变量记录每个peer的match信息。在这里我的实现大致如下：

- 首先每次调用`handleAppendEntriesResponse()` 遍历Prs中每个peer的match并统计每种match的出现频次。
- 将match放入数组中并从大到小排序。
- 遍历排序好的match，直到遇到出现Term相同且频次为majority的match，此match就作为commited index。

由于Go的map是无序的，所以对key(match)排序需要单独放到另一个数组里，再按照排序好的key去map中取key对应的频次。

**MsgRequestVoteResponse**同理，我们不能使用局部变量记录投票个数：

- 我们要在Raft结构体中建立一个voteMap用来记录是否投票。
- 每次调用`handleRequestVoteResponse()`时都要遍历voteMap统计票数。
- 如果拒绝投票的个数为majority则调用`becomeFollower()`否则调用`becomeLeader()`。
- 这几个下标的关系为：first <= applied <= committed <= stabled <= last

#### Storage.ents和raftLog.entries的关系

这个问题包括我在内很多人都晕了好久。原因就是在于这两者的关系涉及到了Compact/SnapShot导致没有了解过的同学产生迷惑。两者的区别如下：

- 如果没有SnapShot的情况下，Storage.ents[0]是一个dummy head，原因就是Storage.ents存的是可能被压缩后的log，计算下标是ents[0].Index+i，为了统一下标计算代码所以引入dummyHead；

- Storage.ents是从底层的badger加载而来的，不会丢失；而raftLog.entries为了速度只保存在内存中。

最后不要忘记实现log.go中其他的工具函数~

#### rawnode中ready的初始化和HardState的更新位置

首先ready中的SoftState和HardState不能无脑的初始化为当前Raft的状态，因为还不能判断状态是否进行了变更。ready只把变更的数据提供给badger进行持久化。

此外由于HardState是必须要在badger中持久化的状态，因此HardState的更新与其他状态不同，他需要等所有State持久化到DB后再在`Advance()`中更新到rawnode中的oldHardState，保证Raft下一次状态刷新时与上一个版本状态不一致。



## lab2B

命令处理的总体流程：

- 调用raftSrotage的write()，命令传递给router
- router会包含在raftWorker中，raftWorker首先调用HandleMsg处理命令
  - 通过proposal记录每个命令对应的回调函数
  - 提交到raft层
- raftWoker调用HasReady检测raft层是否有更新，若有则执行命令：
  - 通过proposal记录的Index和Term判断proposal对应命令是否过期
  - 如果过期：调用proposal记录的回调函数给客户端失败响应
  - 若成功：执行命令，调用proposal记录的回调函数给客户端成功响应

### 思路

- Store：一个真正的服务器实例，包含此服务器的id(StoreID)，服务器状态(上线/下线)，通信地址；
- RaftStorage：会被传入`NewServer()`在集群中创建一个新的**集群节点**，注意这个集群是指的物理上服务器组成的集群。主要依靠raftStore干活生成，相当于**raftStore的包装**；
- RaftStore：用来生成集群中的一个节点，一个raftStore包含不同region的peer；
- peer：一个region里的一个**raft节点**。一台服务器里面会有属于不同region的peer。
- **区分集群节点和raft节点！！！**

一条客户端的请求到tinyKV的流程如下：

- 写请求在raft中广播：写请求发送到raftStorage(raft_server.go)中，raftStorage解析请求并转换成RaftCommmand和请求对应的回调函数，合并发送到router中进而发送到请求中指定的region所在的peer中(一个Store/raftStore对应一台服务器，里面会有属于不同region的peer)。

  ```go
  //raft_server.go
  func (rs *RaftStorage) Write(ctx *kvrpcpb.Context, batch []storage.Modify) error {
  	//code...
  	cb := message.NewCallback()
      
  	//发送到regionID指定的peer中去
  	if err := rs.raftRouter.SendRaftCommand(request, cb); err != nil {
  		return err
  	}
  
  	return rs.checkResponse(cb.WaitResp(), len(reqs))
  }
  ```

  ```go
  //router.go
  func (r *RaftstoreRouter) SendRaftCommand(req *raft_cmdpb.RaftCmdRequest, cb *message.Callback) error {
  	cmd := &message.MsgRaftCmd{
  		Request:  req,
  		Callback: cb,
  	}
  	regionID := req.Header.RegionId
      //注意这里！！
  	return r.router.send(regionID, message.NewPeerMsg(message.MsgTypeRaftCmd, regionID, cmd))
  }
  
  func (pr *router) send(regionID uint64, msg message.Msg) error {
  	msg.RegionID = regionID
      p := pr.get(regionID) //找到这台机器上regionID对应的peer。该<regionID,peer>映射关系(router.go中的peers)在raftStore.go的初始化被加载
  	if p == nil || atomic.LoadUint32(&p.closed) == 1 {
  		return errPeerNotFound
  	}
      //看到了吗，这里就是raftCommand的传入端！！
  	pr.peerSender <- msg
  	return nil
  }
  ```

  raftStorage中`start()`函数会调用`CreateRaftstore()`函数创建raftStore和router：

  ```go
  //raftStore.go
  func CreateRaftstore(cfg *config.Config) (*RaftstoreRouter, *Raftstore) {
  	storeSender, storeState := newStoreState(cfg)
      
  	router := newRouter(storeSender)
  	
      raftstore := &Raftstore{
  		router:     router,
  		storeState: storeState,
  		tickDriver: newTickDriver(cfg.RaftBaseTickInterval, router, storeState.ticker),
  		closeCh:    make(chan struct{}),
  		wg:         new(sync.WaitGroup),
  	}
  	return NewRaftstoreRouter(router), raftstore
  }
  ```

  raftStore在初始化raftWorker的时候**又会把router传递给raftWorker**，由于**peerSender存储RaftCommmand**，这样raftWorker就可以拿到RaftCommmand：

  ```go
  //raftStore初始化worker
  func newRaftWorker(ctx *GlobalContext, pm *router) *raftWorker {
  	return &raftWorker{
  		raftCh: pm.peerSender, //  raftCh在下面就是接收端!!!
  		ctx:    ctx,
  		pr:     pm,
  	}
  }
  ```

  而创建的router和raftStore会返回给raftStorage，并赋值给raftStorage.raftRouter和raftStorage.raftSystem。此后raftStorage会把raftStore传入`node.go`中的`NewNode()`生成一个新的集群节点：

  ```go
  //raft_server.go
  func (rs *RaftStorage) Start() error {
  	cfg := rs.config
  	schedulerClient, err := scheduler_client.NewClient(strings.Split(cfg.SchedulerAddr, ","), "")
  	if err != nil {
  		return err
  	}
      //创建raftStore和router
  	rs.raftRouter, rs.raftSystem = raftstore.CreateRaftstore(cfg)
  
  	rs.resolveWorker = worker.NewWorker("resolver", &rs.wg)
  	resolveSender := rs.resolveWorker.Sender()
  	resolveRunner := newResolverRunner(schedulerClient)
  	rs.resolveWorker.Start(resolveRunner)
  
  	rs.snapManager = snap.NewSnapManager(filepath.Join(cfg.DBPath, "snap"))
  	rs.snapWorker = worker.NewWorker("snap-worker", &rs.wg)
  	snapSender := rs.snapWorker.Sender()
  	snapRunner := newSnapRunner(rs.snapManager, rs.config, rs.raftRouter)
  	rs.snapWorker.Start(snapRunner)
  
  	raftClient := newRaftClient(cfg)
  	trans := NewServerTransport(raftClient, snapSender, rs.raftRouter, resolveSender)
  	//生成并启动该服务器(新的集群节点),集群信息从badgerDB读取，raftStore是包名
  	rs.node = raftstore.NewNode(rs.raftSystem, rs.config, schedulerClient)
      //node会调用raftSystem.Start()，即raftStore的Start()函数。Start()又会去调用raftStore的startWorkers()函数，进而启动raftStore里面的众多worker接受处理命令:
  	err = rs.node.Start(context.TODO(), rs.engines, trans, rs.snapManager)
  	if err != nil {
  		return err
  	}
  	return nil
  }
  ```

- raftStore里面会包含各种各样的worker负责接受命令和监听状态变更。在调用raftWorker的`run()`时，我们会创建peer_msg_handler并将RaftCommand传递给peer_msg_handler：

  ```Go
  func (rw *raftWorker) run(closeCh <-chan struct{}, wg *sync.WaitGroup) {
  	defer wg.Done()
  	var msgs []message.Msg
  	for {
  		msgs = msgs[:0]
  		select {
  		case <-closeCh:
  			return
  		case msg := <-rw.raftCh: //raftCh是接收者
  			msgs = append(msgs, msg)
  		}
  		pending := len(rw.raftCh)
  		for i := 0; i < pending; i++ {
  			msgs = append(msgs, <-rw.raftCh)
  		}
  		peerStateMap := make(map[uint64]*peerState)
  		for _, msg := range msgs {
  			peerState := rw.getPeerState(peerStateMap, msg.RegionID)
  			if peerState == nil {
  				continue
  			}
              
              //看这里！！
  			newPeerMsgHandler(peerState.peer, rw.ctx).HandleMsg(msg)
  		}
  		for _, peerState := range peerStateMap {
              //轮询判断是否有新的Ready
  			newPeerMsgHandler(peerState.peer, rw.ctx).HandleRaftReady()
  		}
  	}
  }
  ```

- peer_msg_handler的`HandleMsg()`接收到命令并发送到`proposeRaftCommand()`中处理。`proposeRaftCommand()`：

  - 从请求中取出对应的key检查是否在region的key range中
  - 使用proposals数组对回调函数进行存储，方便apply时使用
  - 调用`d.RaftGroup.Propose()`将这条请求放入Raft层进行备份
  - 循环直到请求处理完成

  注意这里还并没有apply command，只是放入Raft层和记录回调函数，真正的应用在`HandleRaftReady()`中进行。

- 等待Raft中的状态产生变更时，就会被raftWorker轮询的`HandleRaftReady()`发现，此时就可以开始apply命令了。尽管apply的流程官方文档给出了源码，但是在这里还是更详细的说一下：

  - 调用`d.RaftGroup.HasReady()`判断是否有新的状态，若有则取出新的ready。
  - 调用`d.RaftGroup.saveReadyState()`存储HardState和unstableEntries这些需要持久化的状态。注意调用`ps.Append()`添加unstableEntries到writeBatch中；HardState持久化之前需要调用`raft.IsEmptyHardState()`判断是否为空。最后我们把writeBatch的内容一起flush到badger中。`ps.Append()`的实现思路如下：
    - 如果要添加的entry为空，返回。
    - 如果entries[len(entries)-1].Index < ps.fistIndex，返回。
    - 如果ps.firstIndex > entries[0].Index且entries[len(entries)-1].Index > ps.firstIndex ，处理entries截取ps中还没有entry。
    - 如果entries[len(entries)-1].Index > ps.raftState.LastIndex，截取entry.Index > ps.raftState.LastIndex的entry。
    - 更新ps.raftState.LastIndex和ps.raftState.LastTerm最后一条entry的Index和Term。
    - 将entries写入wrtiteBatch。
  - 遍历CommittedEntries，apply每一个entry。apply的流程如下：
    - 根据命令，对相应的数据操作添加writeBatch。
    - 遍历proposals，比较proposals[0].index和entry.index的大小关系。如果小于entry.index说明**porposals[0]对应的entry没有被apply，是一个过期的无用的entry**。如果index相同但是Term不同也是同理。对于合格的entry我们根据命令生成对应的response通过传入proposals记录的回调函数调用返回。
    - 更新d.peerStorage.applyState.AppliedIndex为entry.index并将d.peerStorage.applyState写入writeBatch。
  - 将writeBatch写入badger。
  - 调用`Advance()`推进系统状态。

### 难点

lab2b的难点主要有以下几个：

- 消息发送流程的传递，已在思路中介绍，不再重复。
- raftWriteBatch和kvWriteBatch要做好区分。
- staleEntry的判断和处理。有两种情况缺一不可，任何一条不满足就代表着proposal对应的cmd已经过时了：
  - proposals[0].index < entry.index
  - proposals[0].index == entry.index && proposals[0].Term== entry.Term

## lab2C

### 思路

在2C开始前，要明确两件事：

- log GC和发送SnapShot是两码事；
- log GC是为了防止OOM和log过多导致的恢复缓慢；
- **log GC日志被截断之后**，落后的节点/新加入的节点在leader中找不到旧的log entry，就会触发leader向其发送SnapShot。

#### log GC

- `onTick()`中推进时钟，`isOnTick()`检测到达设定时间阈值时就调用`d.onRaftGCLogTick()`开始GC流程:

  ```go
  func (d *peerMsgHandler) onTick() {
  	if d.stopped { return }
  	d.ticker.tickClock()
  	if d.ticker.isOnTick(PeerTickRaft) { d.onRaftBaseTick() }
  	if d.ticker.isOnTick(PeerTickRaftLogGC) { d.onRaftGCLogTick() }
  	//code...
  	d.ctx.tickDriverSender <- d.regionId
  }
  ```

- 调用`d.onRaftGCLogTick()`。注意`d.onRaftGCLogTick()`只能leader调用，也就是说log compact由leader发起并先进性，但是真正的gc过程是集群中每一个节点自己独立完成的。log compact截断的index就是applied index，因为log被applied之后也就没有用了：

  ```go
  func (d *peerMsgHandler) onRaftGCLogTick() {
  	d.ticker.schedule(PeerTickRaftLogGC)
      //leader only !!!!!
  	if !d.IsLeader() { return }
  
  	appliedIdx := d.peerStorage.AppliedIndex()
  	firstIdx, _ := d.peerStorage.FirstIndex()
  	var compactIdx uint64
      
  	if appliedIdx > firstIdx && appliedIdx-firstIdx >= d.ctx.cfg.RaftLogGcCountLimit { //out of threshold
  		compactIdx = appliedIdx //compactIdx就是applied index
  	} else { return }
  
  	y.Assert(compactIdx > 0)
  	compactIdx -= 1
      // In case compact_idx == first_idx before subtraction.
  	if compactIdx < firstIdx { return }
  
  	term, err := d.RaftGroup.Raft.RaftLog.Term(compactIdx)
  	if err != nil { panic(err) }
  
  	// Create a compact log request and notify directly.
  	regionID := d.regionId
  	request := newCompactLogRequest(regionID, d.Meta, compactIdx, term)
  	d.proposeRaftCommand(request, nil)
  }
  ```

- 调用`d.proposeRaftCommand()`将**压缩日志请求**放入Raft层发送给follower。

- 集群中的每个节点中的raftWorker会检测到Raft层状态变更(**注意不是raftLogGCWorker**)，开始执行日志压缩请求。

- 应用压缩请求：首先要判断compactIdx >= applyState.TruncatedState.Index。如果是就更新applyState.TruncatedState.Index并放入writeBatch进行持久化。最后我们要调用`ScheduleCompactLog()`向raftLogGCTaskSender发送raftLogGCTask开始异步GC日志：

  ```go
  //raftlog_gc.go
  func (d *peerMsgHandler) ScheduleCompactLog(truncatedIndex uint64) {
  	raftLogGCTask := &runner.RaftLogGCTask{
  		RaftEngine: d.ctx.engine.Raft,
  		RegionID:   d.regionId,
          //上一次GC的ps.applyState.TruncatedState.Index
  		StartIdx:   d.LastCompactedIdx,
          //本次GC的ps.applyState.TruncatedState.Index
  		EndIdx:     truncatedIndex + 1,
  	}
  	d.LastCompactedIdx = raftLogGCTask.EndIdx
  	d.ctx.raftLogGCTaskSender <- raftLogGCTask
  }
  ```

- raftLogGCWorker检测到Task生成，调用`Handle()`开始GC：

  ```go
  //worker.go
  func (w *Worker) Start(handler TaskHandler) {
  	w.wg.Add(1)
  	go func() {
  		defer w.wg.Done()
  		if s, ok := handler.(Starter); ok { s.Start() }
  		for {
  			Task := <-w.receiver
  			if _, ok := Task.(TaskStop); ok {
  				return
  			}
  			handler.Handle(Task)
  		}}()}
  ```

  ```go
  //raftlog_gc.go
  func (r *raftLogGCTaskHandler) Handle(t worker.Task) {
  	logGcTask, ok := t.(*RaftLogGCTask)
  	//code...
  	collected, err := r.gcRaftLog(logGcTask.RaftEngine, logGcTask.RegionID, logGcTask.StartIdx, logGcTask.EndIdx)
  	//code...
  	r.reportCollected(collected)
  }
  ```

  firstIdx和endIdx是在创建RaftLogGCTask时定义好的，分别为**上一次的**ps.applyState.TruncatedState.Index和**这一次的**ps.applyState.TruncatedState.Index。`gcRaftLog()`真正的把log截断：

  ```go
  //raftlog_gc.go
  func (r *raftLogGCTaskHandler) gcRaftLog(raftDb *badger.DB, regionId, startIdx, endIdx uint64) (uint64, error) {
      //...
      for idx := firstIdx; idx < endIdx; idx += 1 {                                             //GC就是把[firstIdx,endIdx]的log全部删除          
          key := meta.RaftLogKey(regionId, idx)
          raftWb.DeleteMeta(key)
      } 
      //...
  }
  ```

#### SnapShot

SnapShot的流程首先需要添加raft.go的代码，思路如下：

- 修改`sendAppend()`，当要发送的对象的Next-1在当前leader找不到对应的log entry时表明这个Next太旧以至于对应的log entry已被GC。这时候就要发送snapshot。
- 调用`r.RaftLog.storage.Snapshot()`获得快照并发送给follower。
- follower调用`handleSnapshot()`处理snapshot：
  - 更新applied、commmted、stable三个index并清空entries，换为snapshot。
  - 重新设定每个peer的Next为m.Snapshot.GetMetadata().GetIndex() + 1，
  - 将r.RaftLog.pendingSnapshot(没有处理的snapshot)设置为当前的snapshot。
- 调用`SaveReadyState()` 更新并持久化snapshot中的状态。函数内部会调用`ps.ApplySnapshot()`处理，这也是我们需要实现的函数：
  - 判断peerStorage是否初始化。如果是，则要处理两种情况：
    - 由于应用snapshot意味着之前所有的数据都不在有意义，因此需要`ps.clearMeta()`清除当前peer数据库中过时的数据。
    - 如果一个leaderGC了key range在1-100的数据，而1-100包含了多个个region。那么就需要调用`ps.clearExtraData()`去除掉不在当前peer所对应的region中。
  - 更新ps.applyState、ps.raftState并写入writeBatch持久化。
  - 发送RegionTaskApply给regionWorker使用。
  - regionWoker的工作流程参考log GC的raftLogGCWorker。
  - 把writeBatch刷入badger持久化。
  - 调用`Advance()`将**pendingSnapshot设为nil**。

### 难点

- raftWoker和raftLogGCWorker/regionWorker要区分清楚。snapshot的流程比较复杂，开始的时候我把这几个worker混为一滩。raftWoker主要负责轮询Ready状态，其余两个woker主要负责处理异步Task请求。
- pendingSnapshot官方文档并没有解释作用。其实就是一个Snapshot没有处理完毕之前别的Snapshot不能处理，为了表明有一个还没有/正在处理的snapshot，需要把pendingSnapshot设为可以处理的Snapshot。
- apply Snapshot时对于follower的log处理我直接采用了全部删除的方法。因为Next就是有效的下一条log存放的位置，Next-1找不到说明follower该有的最后一条log没有，那么现在follower的所有log肯定都是过期的，采取全部删除的方法。
- `handleSnapshot()`中不要忘记修改Next。
- 状态的更变特别是hardState、raftState一定不要忘记持久化。

主要难度还是在于理解代码上。这几个woker协作流程复杂，需要好好阅读源码才能理解。另外一方面就是官方文档并不是特别全面，只能该问就问，该参考就参考。