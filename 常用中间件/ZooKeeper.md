# ZooKeeper

ZooKeeper 是一个开源的分布式协调服务，为分布式应用提供一致性服务的中间件。

## ZooKeeper 集群角色

- Leader：为客户端提供读写服务
- Follower：处理读请求，转发写请求到 Leader；参与写操作的“过半”投票；参与 Leader 选举。
- Observer：ZK 3.3 版本引入，只提供读请求。

## ZooKeeper 数据模型

ZooKeeper 数据模型类似于文件系统的树结构。树上的每一个节点叫做 ZNode，每个节点可以有多个子节点，Znode 的节点标识是它的完整路径。

ZNode 的类型：

- 持久节点（Persistent Node）：节点创建后一直存在，直到被主动删除。
- 临时节点（Ephemeral Node）：与客户端会话绑定，当客户端会话失效后，节点会被自动清理。临时节点下不能再创建子节点。
- 顺序节点（Sequential Node）：每个父节点会维护它的第一级子节点的创建顺序，当指定创建顺序节点时，会给节点添加一个递增的数字后缀（范围 `1 ~ 2^32`）。

其中`顺序节点`可以和`持久节点/临时节点`组合，因此最终会有4种类型的节点。节点的类型在创建时指定，且不能改变。

具体实现如下：

```java
ConcurrentHashMap<String, DataNode> nodes;  // 节点数据信息，key 是 Path，value 是节点数据
ConcurrentHashMap<Long, HashSet<String>> ephemerals; // 临时节点信息，key 是 sessionId，value 是 path set
```

可以向 ZNode 节点写入、修改、删除数据，

ZNode 中的数据内容：

- `cZxid`：创建的事务标识
- `ctime`：创建时间戳
- `mZxid`：修改的事务标识
- `mtime`：修改的时间戳
- `pZxid`：直接子节点最后更新的事务标识
- `cversion`：直接子节点的版本号
- `dataVersion`：节点数据的版本号
- `aclVersion`：节点 ACL 信息的版本号
- `ephemeralOwner`：节点为临时节点时，客户端的 sessionId
- `dataLength`：节点数据存储长度，单位为 byte
- `numChildren`：直接子节点个数

ZNode 中的三个版本号，在对应的内容有变更时会递增，用来在客户端进行操作时作为乐观锁使用。

ZK 中的每一个变化，都会生成唯一的事务 id，即 zxid，是一个 64 位的整数。通过 zxid 可以确定变更的先后顺序。

### Watcher - ZNode 变更通知

ZooKeeper 提供 ZNode 数据变化通知机制，整体流程如下：

1. 客户端注册 Watcher 监听对应节点的变更。
    - 客户端可以设置多种监听点：ZNode 的创建或删除、ZNode 数据变化、ZNode 子节点变化。在调用 exists、getData、getChildren 调用时，可以传入一个自定义 Watcher 或使用默认 Watcher。
2. 服务端对应 Watcher 被触发，通知客户端。
3. 客户端收到通知，执行 Watcher 回调。

ZooKeeper Watcher 通知的特点：

- 单次触发：通知机制是单次触发的操作，如果需要收到多次通知，需要收到通知后重新注册新的 Watcher。
- 轻量级通知：服务端通知的 WatchedEvent，只包含通知状态、节点路径、事件类型，只告诉客户端事件，不告诉客户端具体内容。如果客户端需要获取节点数据，则需要再调接口获取。
- 客户端串行执行：客户端的 Watcher 回调是串行同步的过程。

### Session

Session 核心概念：

- SessionId：会话的 id，用来唯一标识一个会话，每个客户端建立连接时，ZooKeeper 服务端为它分配一个全局唯一的 sessionId，类型为 long，前8位为 ZK server 的机器 id（myid 文件配置），中间40位为当前毫秒数，后面位为递增的数字。
- Timeout：会话超时时间，客户端创建会话时会指定一个 sessionTimeOut，服务端会根据自身的超时时间限制来确定超时时间。
- TickTime：ZK server 会使用分桶进行会话管理，桶即为 tickTime 的整数倍，将会话过期时间向未来取整后放到对应的桶中。

会话的创建：

- 会话创建：Client 创建到 server 的 TCP 连接，server 创建 session 并返回 sessionId。
- 会话刷新：Client 在 `sessionTimeOut/3` 时间内没有读写请求，则会发送一个 PING 请求来刷新 session 过期时间。
- 会话清理：Server 会维护间隔为 tickTime 的桶（map），其中 key 为 tickTime 的整数倍，value 为该时间要过期的 session。收到客户端的请求后移动 session 到新的对应过期时间的桶中。SessionTracker 线程会检测过期的桶，并删除会话的临时节点数据。
- 会话超时：会话超时后，client 会收到 server 的 SESSION_EXPIRED 指令，客户端找任意一台 server 重新创建一个会话。

需要注意会话和连接的概念是不同的，如果客户端在 session 超时时间内重连上了服务器，则 session 还会有效。

## 持久化

ZooKeeper 也有 WAL，对于每个更新操作，先写入 WAL，再对内存数据做更新，最后再返回 client 更新结果。另外 ZooKeeper 还会定期的将内存中的目录树进行 snapshot，存储到磁盘上，这个是为了加快数据恢复的速度。

快照数据：

- 生成的快照是 fuzzy 快照，因为生成过程中数据还在变化，因此要求 WAL 操作能幂等的应用到快照中。ZK 重新启动后，先加载快照，在将快照生成之后的 WAL 全部应用一遍。
- 快照生成的时机：基于配置的事务操作的阀值 `snapCount`，同时引入一定的随机性，在事务操作数量在 `(snapCount/2, snapCount)` 间随机开始生成。避免所有 server 节点一起生产快照。
- 生成快照时同时使用新的 WAL 文件，之前的日志已经含在快照中了。文件命名会使用 ZXID 来命名。

事务日志：

- 由于 WAL 日志需要频繁的 flush 到磁盘，消耗大量 IO，因此使用空间预分配策略，当剩余空间小于4k时，扩容64m，减少磁盘 seek。

## ZooKeeper 工作原理

ZooKeeper 通过 ZAB（ZooKeeper Atomic Broadcast）协议来保证各个 Server 之间的一致，这是一个 leader-based 的分布式一致性协议。

集群中每个节点有三种状态：LOOKING、LEADING、FOLLOWING。

ZAB 协议主要分为两个阶段：

- 恢复模式（选主）：集群 leader 宕机后，节点会进入 LOOKING 状态，并递增 epoch 和广播自己的投票，发送自身的 `<epoch, zxid, sid>` 作为选票，收到别人的选票后，如果别人的选票比自己的大，会改变自己的选票后重新发送。各节点收集是否有过半机器投票自己，获得过半选票的机器成为 Leader，将自身的状态改为 LEADING 状态。之后开始进入恢复模式，leader 发送 NewLeader 命令给其他节点，将自身的数据同步到各 follower。恢复状态下是阻塞的，不会处理集群请求。
- 广播模式（复制）：Leader 会处理所有的事务请求，对于事务请求，leader 写入本地日志后，发送 proposal 同步给所有的 follower，收到半数的回复后，回复 client 成功，并发送 commit 消息给 follower。Client 收到返回成功时，日志已经复制到了半数以上节点，但是可能还没有 commit。

Raft 和 ZAB 对比：

- 都是 leader-based 协议，都主要分为选主和日志复制两个阶段。
- Leader 选举差异：Raft 具有**最大已经 commit 编号**的机器就可以成为 leader；ZAB 中会尽量让有最大 zxid（包含未提交的）的机器成为 leader。比如一个日志只复制到了机器的半数以下，那么 raft 和 ZAB 中每台机器都可能当选 leader，但是 ZAB 中日志更多的当选的概率大一点（可以变票，并且最大优先）。
- 选举效率差异：Raft 协议，每个节点在每个 term 中只投一次票，如果平票则等待 term 超时后；ZAB 中，每个节点每个 electionEpoch 轮次内，可以投多次票，只要遇到更大的票就更新自己的选票重新投票，因此一轮选举一定可以选出 leader，但是需要多次交互，效率较低。
- Leader 处理未提交消息的区别：Raft 的 leader 选举后，会通过 NO-OP LogEntry 来提交之前 term 的未提交消息，因此对于未提交消息是否会被提交，跟选举出来的 leader 有关；ZAB 选举出来 leader 后，会进入恢复模式，将数据同步给 follower，该阶段阻塞不处理请求。
- 读请求处理：Raft 默认只有 leader 处理所有读写请求，因此是线性化一致读，不会出现 stale read；ZAB 中 follower 节点可以处理读请求，会出现 stale read，但是提供了 sync 接口来进行线性化一致读。

## ZooKeeper 典型应用场景

常用的应用场景有：

- 配置中心
- Leader 选举
- 分布式锁
- 命名服务/发号器：为节点分配唯一的名称或编号，如使用 snowflake 作为分布式 id 时，机器号可以通过 ZK 来获取。

排它锁实现：

- 可以在通过在对应的资源节点下创建固定名称的临时节点来实现，如 `/${lockName}/lock`。创建成功的节点即为抢到锁，其他节点注册锁节点删除的 watcher。抢到锁的客户端处理完成或者 session 失效后删除节点。这种实现的缺点是锁释放时，所有监听者都被唤醒但是只有一个能获取到锁，都是无效的通知，这叫做羊群效应。
- 为了避免羊群效应，可以通过在对应的资源节点下创建临时顺序节点来表示等待抢锁的队列。每个客户端只对比自己小的上一个临时节点注册 watcher。

共享锁实现：

- 共享锁也可以通过在对应的资源节点下创建临时顺序节点来表示等待抢锁的队列，节点名称可以通过 RW 来区分读锁和写锁，读请求对比自己小的上一个写请求节点注册 watcher，写请求对比自己小的上一个节点注册 watcher。

## ZooKeeper 客户端

ZooKeeper 常用客户端：

- ZK 原生客户端：提供底层的 API，对节点的操作和监听。
- ZkClient：开源的客户端，包装了原生的客户端，提供了 session 的超时重连、watcher 的反复注册等特性。
- Curator：Netflix 开源的 ZK 客户端，使用最广泛的客户端之一。提供了更高抽象的工具，如分布式锁、Master 选举、分布式计数器等。
