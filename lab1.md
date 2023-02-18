# lab1

## StandAloneStorage

通过文档中的提示2我们知道，**engine_util对badger进行了封装**，StandAloneStorage就是要**在engine_util的基础上再封装一层**。也就是说其实进行了三层封装：

- 实现好的：engine_util对**badgerDB进行了封装**，engine_util的作用是添加prefix来支持列族(badgerDB本身不支持列族)；
- 要实现的：Storage的`write()/read()`接口对**engine_util做了更进一步的封装**，StandAlongStorage实现了Storage接口，是badgerDB的事务包装，**减少事务调用代码**；
- 要实现的：rawGet/rawPut/rawDelete/rawScan四个对上层暴露的接口对**StandAlongStorage做了更进一步的封装**，用于上层代码与引擎层解耦

```go
type Engines struct {
	// Data, including data which is committed (i.e., committed across other nodes) and un-committed (i.e., only present locally).
	Kv     *badger.DB
	KvPath string
	// Metadata used by Raft.
	Raft     *badger.DB
	RaftPath string
}
```

  那么我们就再封装一层即可，为什么这里要加一个conf？因为我们创建的时候需要用到conf里面的参数，因为不知道后续会不会用到这个config，所以先保存起来。

```go
type StandAloneStorage struct {
	// Your Data Here (1).
	engine *engine_util.Engines
	conf   *config.Config
}

func NewStandAloneStorage(conf *config.Config) *StandAloneStorage {
	// Your Code Here (1).
	dbPath := conf.DBPath
    kvPath := path.Join(dbPath, "kv")
    raftPath := path.Join(dbPath, "raft")
	//func CreateDB(path string, raft bool) *badger.DB
    kvDB := engine_util.CreateDB(kvPath, false)
    raftDB := engine_util.CreateDB(raftPath, true)
    //func NewEngines(kvEngine, raftEngine *badger.DB, kvPath, raftPath string) *Engines
    return &StandAloneStorage{
        engine: engine_util.NewEngines(kvDB, raftDB, kvPath, raftPath),
        conf:   conf,
    }
}
```


StandAloneStorage是一个实例，**实现了Storage接口**。那么我们下面重点关注Write和Reader：

```go
type Storage interface {
	Start() error
	Stop() error
	Write(ctx *kvrpcpb.Context, batch []Modify) error
	Reader(ctx *kvrpcpb.Context) (StorageReader, error)
```

### Write

去看看Modify类型是什么

```go
Write(ctx *kvrpcpb.Context, batch []Modify) error
```

**Modify 本质上代表着Put和Delete两种操作**，通过下面的函数可以看到，它是通过断言来区分Put还是Delete的。一个Modify对应一个kv，所以可以看到上面Write接口中，Modify是一个切片，那么对于Write，我们需要遍历range这个切片进行操作。

```go
// Modify is a single modification to TinyKV's underlying storage.
type Modify struct {
	Data interface{}
}

type Put struct {
	Key   []byte
	Value []byte
	Cf    string
}

type Delete struct {
	Key []byte
	Cf  string
}

func (m *Modify) Key() []byte {
	switch m.Data.(type) {
	case Put:
		return m.Data.(Put).Key
	case Delete:
		return m.Data.(Delete).Key
	}
	return nil
}

```


提示2中说了，engine_util包中提供了方法进行所有的读写操作，可以看到**engine_util提供了两个函数**给我们使用，看一下源码其实也能发现，在真实存储的时候，key被加了前缀：

```go
func PutCF(engine *badger.DB, cf string, key []byte, val []byte) error {
	return engine.Update(func(txn *badger.Txn) error {
		return txn.Set(KeyWithCF(cf, key), val)
	})
}

func DeleteCF(engine *badger.DB, cf string, key []byte) error {
	return engine.Update(func(txn *badger.Txn) error {
		return txn.Delete(KeyWithCF(cf, key))
	})
}
```

在了解了engine_util提供的方法，以及Modify的含义后，Write怎么实现其实就迎刃而解了。

```go
func (s *StandAloneStorage) Write(ctx *kvrpcpb.Context, batch []storage.Modify) error {
	// Your Code Here (1).
	//func PutCF(engine *badger.DB, cf string, key []byte, val []byte)
	var err error
	for _, m := range batch {
		key, val, cf := m.Key(), m.Value(), m.Cf()
		if _, ok := m.Data.(storage.Put); ok {
			err = engine_util.PutCF(s.engine.Kv, cf, key, val)
		} else {
			err = engine_util.DeleteCF(s.engine.Kv, cf, key)
		}
		if err != nil {
			return err
		}
	}
	return nil
}
```



### Reader

Reader函数需要我们返回一个StorageReader接口，那么我们就需要去实现一个StorageReader接口的结构体，来看看实现它需要哪些函数

```go
Reader(ctx *kvrpcpb.Context) (StorageReader, error)

type StorageReader interface {
	// When the key doesn't exist, return nil for the value
	GetCF(cf string, key []byte) ([]byte, error)
	IterCF(cf string) engine_util.DBIterator
	Close()
}
```

看一下engine_util包给我们提供了什么函数，可以看到从txn中读取或者遍历的方法不用自己写，直接使用api即可，那么上面的**GetCF和IterCF就是对下面两个函数的封装**。

```go
// get value
val, err := engine_util.GetCFFromTxn(txn, cf, key)

func GetCFFromTxn(txn *badger.Txn, cf string, key []byte) (val []byte, err error)

// get iterator
iter := engine_util.NewCFIterator(cf, txn)

func NewCFIterator(cf string, txn *badger.Txn) *BadgerIterator
```

可以看到engine_util提供的GetCFFromTxn和NewCFIterator两个函数都需要badger.Txn，那么这个Txn是什么呢？其实是一个事务。获取badger.Txn的函数engine_util并未给出，需要直接调用badger.DB.NewTransaction函数。

```go
txn *badger.Txn

//update为真表示Put/Delete两个写操作，为假表示Get/Scan两个读操作。
//NewTransaction creates a new transaction.
func (db *DB) NewTransaction(update bool) *Txn
```

所以不难发现，想要实现StorageReader接口，那么结构体中就要包含badger.Txn，继而去调用engine_util提供的api。在这里，可以把这个字段放在StandAloneStorage中；不过我还是把它拆到新的结构体里面了。

为什么呢？在上面的Write函数中，我们仅仅传入的是`*badger.DB`，其内部也是有`*badger.Txn`的，但是Write对事务进行了屏蔽。在StorageReader接口的两个函数中，也没有事务的身影，但是我们因为是读，所以需要用到新的读事务，只能去创建一个。

综合来看，这些接口的封装，很明显就是不想让我们去过多的使用事务的，所以干脆把其放在一个新结构体里面return掉好了。

```go
func (s *StandAloneStorage) Reader(ctx *kvrpcpb.Context) (storage.StorageReader, error) {
	// Your Code Here (1).
	//For read-only transactions, set update to false.
	return &StandAloneStorageReader{
		kvTxn: s.kvDB.NewTransaction(false),
	}, nil
}

//实现 type StorageReader interface 接口
type StandAloneStorageReader struct {
	KvTxn *badger.Txn
}

func (s *StandAloneStorageReader) GetCF(cf string, key []byte) ([]byte, error) {
	val, err := engine_util.GetCFFromTxn(s.KvTxn, cf, key)
	if err == badger.ErrKeyNotFound {
		return nil, nil
	}
	return val, err
}

func (s *StandAloneStorageReader) IterCF(cf string) engine_util.DBIterator {
	//func NewCFIterator(cf string, KvTxn *badger.Txn) *BadgerIterator
	return engine_util.NewCFIterator(cf, s.KvTxn)
}

func (s *StandAloneStorageReader) Close() {
	s.KvTxn.Discard()
}
```


## Server

前面以及说过了，原始的 key/value 服务处理程序(Put/Delete/Get/Scan)：其实就是对存储引擎的一层上层封装，有了这层封装，我们可以随意改变底层的存储引擎。使用则无需关注底层细节，只要遵循上层接口使用规范即可。

```go
// Server is a TinyKV server, it 'faces outwards', sending and receiving messages from clients such as TinySQL.
type Server struct {
	storage storage.Storage
	//...
}
```


这样做我觉得最大的好处就是，底层的存储引擎无论怎么换，对上层都是无感的。这让我想到了mysql的innodb和myisam，都是无感的。在project1中，我们实现的是单机的存储引擎，所以在后续的project中，我们就可以直接换掉它。

```go
func (server *Server) RawGet(_ context.Context, req *kvrpcpb.RawGetRequest) (*kvrpcpb.RawGetResponse, error)
func (server *Server) RawPut(_ context.Context, req *kvrpcpb.RawPutRequest) (*kvrpcpb.RawPutResponse, error)
func (server *Server) RawDelete(_ context.Context, req *kvrpcpb.RawDeleteRequest) (*kvrpcpb.RawDeleteResponse, error)
func (server *Server) RawScan(_ context.Context, req *kvrpcpb.RawScanRequest) (*kvrpcpb.RawScanResponse, error)
```

这4个接口都是对上面的StandAloneStorage中写的接口的再次封装。

- RawGet： 如果获取不到，返回时要标记 reply.NotFound=true。
- RawPut：把单一的 put 请求用 storage.Modify 包装成一个数组，一次性写入。
- RawDelete：和 RawPut 同理。
- RawScan：通过 reader 获取 iter，从 StartKey 开始，同时注意 limit。
- 需要特别注意的一点是：RawGet： 如果获取不到，返回时要标记 reply.NotFound=true

我所理解的Scan，是线性扫描。那么转入请求中的Limit，就是限制多少个。首先根据Seek定位到第一个key，然后向后读Limit个，同时要注意下一个的合法性，具体见代码。

// The functions below are Server's Raw API. (implements TinyKvServer).
// Some helper methods can be found in sever.go in the current directory

// 上层允许err时返回nil结构体

// RawGet return the corresponding Get response based on RawGetRequest's CF and Key fields
func (server *Server) RawGet(_ context.Context, req *kvrpcpb.RawGetRequest) (*kvrpcpb.RawGetResponse, error) {
	// Your Code Here (1).
	reader, err := server.storage.Reader(req.Context)
	if err != nil {
		return nil, err
	}
	val, err := reader.GetCF(req.Cf, req.Key)
	if err != nil {
		return nil, err
	}
	resp := &kvrpcpb.RawGetResponse{
		Value:    val,
		NotFound: false,
	}
	if val == nil {
		resp.NotFound = true
	}
	return resp, nil
}

// RawPut puts the target data into storage and returns the corresponding response
func (server *Server) RawPut(_ context.Context, req *kvrpcpb.RawPutRequest) (*kvrpcpb.RawPutResponse, error) {
	// Your Code Here (1).
	// Hint: Consider using Storage.Modify to store data to be modified
	put := storage.Put{
		Key:   req.Key,
		Value: req.Value,
		Cf:    req.Cf,
	}
	batch := storage.Modify{Data: put}
	err := server.storage.Write(req.Context, []storage.Modify{batch})
	if err != nil {
		return nil, err

	}
	return &kvrpcpb.RawPutResponse{}, nil
}

// RawDelete delete the target data from storage and returns the corresponding response
func (server *Server) RawDelete(_ context.Context, req *kvrpcpb.RawDeleteRequest) (*kvrpcpb.RawDeleteResponse, error) {
	// Your Code Here (1).
	// Hint: Consider using Storage.Modify to store data to be deleted
	del := storage.Delete{
		Key: req.Key,
		Cf:  req.Cf,
	}
	batch := storage.Modify{Data: del}
	err := server.storage.Write(req.Context, []storage.Modify{batch})
	if err != nil {
		return nil, err
	}
	return &kvrpcpb.RawDeleteResponse{}, nil
}

// RawScan scan the data starting from the start key up to limit. and return the corresponding result
func (server *Server) RawScan(_ context.Context, req *kvrpcpb.RawScanRequest) (*kvrpcpb.RawScanResponse, error) {
	// Your Code Here (1).
	// Hint: Consider using reader.IterCF
	reader, err := server.storage.Reader(req.Context)
	if err != nil {
		return nil, err
	}
	var Kvs []*kvrpcpb.KvPair
	limit := req.Limit

	iter := reader.IterCF(req.Cf)
	defer iter.Close()


	for iter.Seek(req.StartKey); iter.Valid(); iter.Next() {
		item := iter.Item()
		key := item.Key()
		val, _ := item.Value()
	
		Kvs = append(Kvs, &kvrpcpb.KvPair{
			Key:   key,
			Value: val,
		})
		limit--
		if limit == 0 {
			break
		}
	}
	
	resp := &kvrpcpb.RawScanResponse{
		Kvs: Kvs,
	}
	return resp, nil
}



## LSM-Tree基本结构

memtable+sstable(sorted string table，itself is a file)，支持高效写操作。

- sstable为什么要排序？

sstable在合并的是后可以快速合并(与归并排序的归并思想一致)



### 合并策略

#### Size-Tiered Compaction(STCS)

##### 优点

- 有效控制了sstable的数量增长，但是单个sstable的大小是指数级别增长；

- 相对于LCS来说，**减少了写放大**，减少了在压缩过程中相同数据的复制次数；

##### 缺点

###### 空间放大

- **合并时的临时突然放大：**比如我们规定当4个`smaller sstable`放满时合并成为1个bigger sstable，即`4*smaller sstable size = 1*bigger sstable size`。当`bigger sstable`产生时旧的4个`smaller sstable`并没有被删除，所以此时的磁盘就会突然多了`4*smaller sstable`的占用。也就是说正常的情况下只有1个bigger sstable的空间占用，但此时的空间占用变成了`2*bigger sstable`。如果磁盘容量是10G，合并后的bigger sstable为8G，由于空间放大的原因就会导致放大后的空间16G>10G产生磁盘瓶颈(此时磁盘还有剩余空间没有被使用，造成浪费)。空间放大**最直观的后果**就是必须保证任何时间磁盘都必须有一半的空闲空间；
- **Hot Key导致的空间放大**：如果频繁对同一个key进行大量的写操作(改/删)，只有当4个smaller sstable放满时才能进行合并，而这个key可能在相同level中的4个sstable table中各存在一份，相当于放大了4倍；再加上合并时产生的一份，相当于放大了5倍。此外这个key也可能存在于更低级的level中，最终的结果相当于**放大了5倍以上**。

###### 

#### Leveled Compaction (LCS)

##### 优点

- **适合只读场景**：90%数据在最高层的sstable中，大部分读取在最高层进行，**减少了读放大和空间放大**；
- **Hot Key修改**：由于lsm-tree在合并时会将同一个key的操作进行合并，因此对于**修改操作**来说的话**key的数量永远是1**，即只保存key当前最新的value。因此合并操作基本**只会在L0和L1两层执行**，这种场景也适合使用LCS；

##### 缺点

###### 写放大

- 由于每一层的sstable个数是按照一个增长因子指数α(比如10)进行增长且每一层的key range是相同的，因此当某一层的sstable个数超过规定个数时需要将该层中的一个sstable与高一层的α个sstable合并，因为单个sstable表示的范围与高一层的α个sstable表示的范围相同。所以，在对α(10)+1个sstable归并完写入后相当于写操作放大了α+1，即11倍；

###### 写放大爆IO带宽

- **写操作放大后**很容易**写IO爆带宽**：即使**读多**(90%)**写少**(10%)的场景也会出现问题，由于更多的磁盘带宽专用于写入，读取请求减慢，这使得90%的读取请求需要从最高层sstable读取的优势变得毫无意义;

- L0 sstable积压：如果磁盘带宽**被写请求爆掉**的话，合并压缩就无法进行，会导致大量的sstable积压在L0。这样会**爆内存**并产生**读放大**。

## WiscKey

- **KV分离**

  由于key相对于value来说会很小且相对固定，因此LSM-Tree中只存储<key,val_addr>，value存在单独的vLog中。这样就解决了两个问题：

  - 减少了写放大：同样放大10倍的情况下，由于wisckey只存储key，因此需要写入数据的大小要远小于kv一起存在LSM-Tree的情况。
  - 减少了读放大：由于单个元素只存储key，LSM-Tree每一层可以存储的元素更多，**层数会减少，响应查询key需要遍历的层数也会减少**。同时**内存也可以缓存更多的元素**，提高读速度。

- **利用SSD的并行IO**

  KV的分离导致每一次获取val到需要进行一次random IO。如果是单次查询这个开销可以被上面提到的优化摊销掉，但是在range query中会非常浪费时间。作者发现**当请求的数据大小达到一定规模后，SSD的并行IO速度和顺序IO速度相差很小**。因此当系统当发现用户使用**Next()/Prev()等有range query意图的API后**，系统会**pre-fetching**后面一定数量的key，并把这些key放入队列中使用多个线程一起获取这些key对应的value。

- **vLog GC**

  vLogGC的过程类似于插入排序。vLog内容的末端被head指向，用于插入valid value。tail不断地向后遍历vLog entry，通过entry里的key在LSM-Tree中搜索是否是有效的。有效的话就在head插入并移动head和tail。最终tail和head夹住的内容才是有效的。

  由于vLog里面已经记录了value对应的key，因此WAL Log已经不需要了，crash后系统可以根据vLog中的key恢复LSM-Tree。

  

# lab2

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

# lab3



# lab4

## 2PC

### 流程

由于Percolater是一个2PC协议，首先需要了解什么是2PC：

在分布式系统中，一个事务要处理散落在不同机器上的数据。比如x可能放在A上，y放在B上，也就是说要执行的事务分到了两台机器上。此时的事务在满足单机事务的要求上还要遵循**原子提交协议**：

- server1和server2必须都同时提交才事务成功，只要有一台机器提交失败(比如y-1前y=0，账户没有负的)则事务abort。

在开始之前，需要知道几个概念：

**参与者(participant)：**真正执行事务代码的server。这里就是存放x/y的A和B。

**事务ID(tid)：**一个分布式系统中会有很多事务，需要区分事务。比如A/B会维护一个Lock表

**TC协调者(transaction coordinator)：**分配tid和分发请求。A/B中的Lock表使用tid表示哪个锁当前被哪个事务占用，防止重复获取锁；处理协调来自A/B的消息保证事务的完成。

<img src="C:\Users\hp\Desktop\求职\设计模式\pics\2pc.png" alt="2pc" style="zoom:67%;" />

### 2PC故障

#### 机器故障

- **如果B发生故障**

  - **B在发送YES/NO之前发生故障**

    由于TC永远收不到B的YES/NO，所以TC也永远不会向参与者发送commit/abort。参与者的状态因此不会落盘，根据原子提交协议，B有权中断这个事务的执行。

  - **B在参与者发送完YES/NO到收到TC的commit/abort之前发生故障**

    由于TC收到了所有参与者的YES会向所有参与者发送commit。如果A收到了commit那么A会将消息落盘。因此假设B此时故障了不能单方面终止事务，因为A已经落盘，终止事务会破坏原子提交协议破坏数据一致性。因此要抓紧恢复B的运行并且让B重新接收commit完成未完成的事务。

    所以，参与者在发送完YES/NO后要将自身状态持久化到磁盘LOG文件中，以便故障恢复后重新接收commit完成未完成的事务。

- **如果TC发生故障** 

  - **TC在发送PREPARE前故障**

    事务相当于没有开始，无所谓。

  - **TC在收到YES/NO但是还没发送commit时故障**

    由于TC已经向参与者发送了PREPARE，因此事务已经开始。但是参与者们并没有收到commit/abort，因此**TC可以单方面终止事务**，不会引起一致性问题。 由于事务会占用需要的公共资源因此会阻塞其他事务。

  - **TC在发送一条或多条commit后故障(阻塞)**

    如果发送了部分commit/abort说明很有可能部分参与者已经提交/终止事务了，因此**TC需要恢复重新发送**这一组commit/abort保证一致性。**TC在收到YES/NO后会将其对应的tid记录下来保存到磁盘LOG中方便恢复重发commit/abort**。因此参与者也需要去掉重复的commit/abort请求。


#### 网络故障

- **TC给B发送PREPARE，但是没收到B的YES/NO回复**

  TC可以间隔重发，若一直不回复则可以单方面终止事务防止占用资源。

- **B发送了YES/NO但始终未收到TC的commit/abort(阻塞)**

  由于B发送了YES给TC，因此TC可能已经给部分参与者发送了commit，也就是说部分参与者提交了事务，但是B没有。因此为了一致性B**不能单方面终止事务**。**因此B要一直阻塞到TC发送commit/abort。**

### 2PC缺点

- 慢。因为有很多数据要磁盘IO，此外还要发送大量消息。此外2PC是强一致。
- 有阻塞问题存在。

## Percolator

### 基本结构

Percolator主要有以下三种column family，Percolator的事务就是借助这三个column family实现的。此外Percolator基于行事务，即单行操作满足事务，但是行与行之间不满足事务(这也是我们要实现的)：

| CF                | Key                        | Value     |      |
| ----------------- | -------------------------- | --------- | ---- |
| default           | (userKey，startTimeStamp)  | userValue |      |
| lock              | userkey                    | Lock      |      |
| write(代表已提交) | (userKey，commitTimeStamp) | Write     |      |

Lock的结构如下：

```go
type Lock struct {
	Primary []byte
	Ts      uint64 //txn start timestamp
	Ttl     uint64
	Kind    WriteKind
}
```

Write的结构如下：

```go
type Write struct {
	StartTS uint64
	Kind    WriteKind
}
```

### 要点

- lock不为空表明当前的修改还没有commited。
- lock为空但write不为空表明修改被commited。因此如果我们要获取value应该满足(lock==nil)&&(write!=nil)。
- 一个事务肯定会涉及到多个key，我们选取其中的一个**key作为primary作为判断事务是否commit的标准。**如果primary提交了就说明这个事务提交了，没有提交的row会**异步提交**或在**下次其他事务执行时提交。**如果primary没有提交就需要回滚write不为空的行。
- 由于tinyVK中不支持行事务，为了满足行事务我们要对default，lock，write这三个cf加锁且**加锁/释放锁**都要三个列族**一次完成**，这样就能保证(default，lock，write)这一行满足行事务的要求。一次完成上锁/释放锁是通过waitGroup实现的：
  - 加锁就调用`wg.Add(1)`对latchGroup中key对应的waitGroup+1，如果waitGroup != 0的话别的线程就会调用`wg.Wait()`轮询等待。
  - 释放锁时就调用`wg.Done()`对latchGroup中key对应的waitGroup-1(此时waitGroup == 0，不阻塞)并在latchGroup删除这个key的映射。
  - 其他线程拿锁重复上述过程。
  - latchGroup修改的线程安全通过Mutex保证。
- 注意遍历CFValue和CFWrite时要注意复合key的存储顺序：首先是按userKey升序排列，对于相同的userKey再按timestamp降序排列。

### 流程

#### 写事务

##### Prewrite阶段

1. 随机取一个写操作作为primary，其他的写操作则自动成为secondary。Percolator总是先操作primary。
2. 冲突检测：
3. 如果在start_ts之后，发现write列有数据，则说明有其他事务在当前事务开始之后提交了。这相当于2个事务的并发写冲突，所以需要将当前事务abort。
4. 如果在任何timestamp上发现lock列有数据，则有可能有其他事务正在修改数据，也将当前事务abort。当然也有可能另一个事务已经因为崩溃而失败了，但lock列还在，对于这一问题后面的故障恢复会讨论到。
5. 锁定和写入：对于每一行每一列要写入的数据，先将它锁定（通过标记lock列，secondary的lock列会指向primary），然后将数据写入它的data列（timestamp就是start_ts）。此时，因为该数据的write列还没有被写入，所以其他事务看不到这次修改。

对于同一个写操作来说，data、lock、write列的修改由BigTable单行事务保证事务性。

由冲突检测的b可以推测：如果有多个并发的大事务，并且操作的数据有重合，则可能会频繁abort事务，这会是一个问题。在TiDB的改进中会谈到它们怎么解决这一问题。

##### Commit阶段

1. 从TSO处获取一个timestamp作为事务的提交时间（后称为commit_ts）。
2. 提交primary, 如果失败，则abort事务：
3. 检查primary上的lock是否还存在，如果不存在，则abort。（其他事务有可能会认为当前事务已经失败，从而清理掉当前事务的lock）
4. 以commit_ts为timestamp, 写入write列，value为start_ts。清理lock列的数据。注意，此时为Commit Point。“写write列”和“清理lock列”由BigTable的单行事务保证ACID。
5. 一旦primary提交成功，则整个事务成功。此时已经可以给客户端返回成功了。secondary的数据可以异步的写入，即便发生了故障，此时也可以通过primary中数据的状态来判断出secondary的结果。具体分析可见故障恢复。

### 2PC优化

- Percolator使用client作为TC，去掉了传统2PC中的TC，**减少了传统2PC中的单机TC故障**；
- **Percolator的事务只有提交/未提交的状态。**传统2PC的Commit Point在写本地磁盘的那一刻，Percolator 2PC的Commit Point在完成primary提交的一刻。原因就是传统2PC在收到prepare确认时会落盘事务tid用于TC崩溃时的恢复重发，但是Percolator是client作为TC，崩溃大概率无法恢复；
- Primary提交成功后Secondary可以异步顺序写入，因为事务已经是提交状态。通过异步可以提高系统的事务处理能力；
- Percolator的rollback是lazy+lock和write状态判断；

### 写偏

- 我们先举个例子:

  比如说有四个棋子分别为黑、黑、白、白。

  事务甲：把所有白色棋子变成黑色。
  事务乙：把所有黑色棋子变成白色。

  每个事务要做的事情都是：第一步，查找所有白（黑）色棋子；第二步，把找到的棋子改成黑（白）色；

如果两个事务都是在对方做第二步之前就做了自己的第一步查询棋子颜色(两个事务查询到的颜色一致)，事务甲会把那两个原先黑的改成白的，但是乙进行修改的前提是依据甲修改之前的颜色/最初查询到的颜色。事务乙根据旧状态把那两个原先白的改成黑的，最后变成了白、白、黑、黑。

**写偏**就是如果事务A先对数据库查询，根据查询的结果决定之后怎么写，但是事务B在事务A查询完后立马修改了A查询的值的状态，导致A的第二步写的前提条件遭到了破坏。

## lab4a

#### 思路

我们按照上表中的定义实现即可，注意涉及到修改的操作要使用`storage.Modify{}`添加到`txn.writes`中。主要说一下

`GetValue()`、`CurrentWrite()`、`MostRecentWrite()`这三个比较有难度的函数实现思路：

- `GetValue()`：注意官方注释的这句话：

  > valid at the start timestamp of this transaction

  既然要对于当前timestamp有效的value，我们肯定首先要去write中搜索找到key==userKey的write记录。通过write记录中的StartTS和key组合成default key再去default中得到有效的value。

  此外要注意`write.Kind`的判断，我们**不能**使用WriteKindDelete/WriteKindRollback的记录!

- `CurrentWrite()`：遍历CFWrite，选出userKey == key的write记录。解析write得到write对应事务的开始时间write.startTs，如果write.startTs和当前事务startTs相同就返回。

- `MostRecentWrite()`：`CurrentWrite()`去掉startTs的版本，返回指定的key最新的一条write信息。

## lab4b

4b对应的三个函数就是Percolator上的三个函数，论文给出了伪码，可以参考伪码实现。这里说一下我的实现思路：

- `KvGet()`：
  - 获取reader和latch。
  - 获取请求中key对应的lock。如果**lock.Ts <= 当前事务的startTS**，失败(有未提交的事务，不能保证数据安全)。
  - 调用`MostRecentWrite()`取出key对应的最新的write信息。
  - 将write记录中的StartTS和key组合成default key再去default中得到有效的value并返回。

- `KvPrewrite()`：

  - 获取reader和latch。
  - 遍历request中的每个mutation，并调用`MostRecentWrite()`获得对应mutation中的key最新的write。

  - 如果**此write的commitedTs > 当前事务startTime**，失败(这个mutation已经过期了)。
  - 如果mutation的key在数据库中**任意时间**有lock记录的话，失败(别的事务可能还没提交)。
  - 根据mutation不同的操作类型将default和lock写入。
  - 调用`server.storage.Write()`将`txn.writes`刷入磁盘。

- `KvCommit()`：清除lock中的记录并写入write。

  - 获取reader和latch。
  - 遍历request中的每个要提交的key，获取key对应的lock。如果(lock == null)&&(write != null)，又分为两种**失败**情况(两种情况的response处理要区分开)：
    - prewrite时间太长(超过一个ttl)被其他事物rollback导致此事务对应的锁记录消失，write记录为writeRollBack(这条操作会在`KvCheckTxnStatus()`中进行)。
    - 重复提交一个commit请求，write已经为WriteKindPut/WriteKindPutDelete
  - 如果lock存在但是lock.Ts != txn.startTs，失败。
  - 删除key对应的lock记录，添加write记录。
  - 调用`server.storage.Write()`将`txn.writes`刷入磁盘。

## lab4c

- `KvCheckTxnStatus()`：(只)根据主键检查处理事务的状态。

  - 获取reader和latch。
  - 获取主键对应的write和lock。
  - 如果lock为空：
    - 若write !=null 且write.kind != WriteKindRollback表明提交成功(不需要判断CFValue因为行事务)。
    - 若write == null表明提交失败，需要写入一个write.kind == WriteKindRollback的write进行回滚。
  - 如果lock不为空：
    - 如果事务的**Current Physical Time > Start Physical Time + lock.ttl**，说明超时，需要写入一个write(write.kind == WriteKindRollback)进行rollback。
    - 否则成功
  - 调用`server.storage.Write()`将`txn.writes`刷入磁盘。

- `KvBatchRollback()`：一次批量回滚多个key。

  - 获取reader和latch。
  - 遍历要回滚的key并获取对应的write。
  - 若write != null且write.Kind != mvcc.WriteKindRollback说明已经提交了，立即Abort。
  - 若write == null获取lock。如果lock被其他事务占用（lock.Ts != req.StartVersion）则立即Abort。
  - 删除key对应的value和lock，并写入一个write(write.kind == WriteKindRollback)进行rollback。
  - 调用`server.storage.Write()`将`txn.writes`刷入磁盘。

- `KvResolveLock()`

  `KvResolveLock()`是对非主键的key（secondary key）进行调用的。我们首先要判断主键是否commited，并根据主键的提交状态通过此函数一次性将secondary key批量commit/rollback。

  - 获取reader和latch。
  - 获取CFLock的迭代器，遍历CFLock。
  - 将lock.Ts == req.StartVersion的lock的key存入validKeys。
  - 如果req.CommitVersion == 0表明要对validKeys全部rollback，调用`KvBatchRollback()`执行。
  - 否则表明要对validKeys全部提交，调用`KvCommitk()`执行。

- `KvScan()`

  - 由于get的value必须有效，因此我们必须从CFWrite中遍历获取。

  - 实现Scanner：

    - 初始化startKey、txn并获取CFWriter的迭代器。

    - 若果userKey != scan.startKey，说明找到了新的元素，更新scan.startKey并使用`scan.txn.GetValue(key)`拿到对应的value值返回。
    - 如果相等，使用迭代器遍历CFWrite，直到找到一个userKey != scan.startKey的元素，重复上面步骤。

  - 获取Scanner并遍历。

  - 设置计数器，每取一个元素就+1，当计数器达到req.limit时返回获取的元素。

    