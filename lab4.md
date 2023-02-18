# lab4文档

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

    
