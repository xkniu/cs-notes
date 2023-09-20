# HBase

## 基础信息

支持海量数据读写（尤其是写），单表百亿行百万列，主从架构，底层依赖 HDFS，ZK 作为分布式协同服务。不支持事务，不支持二级索引。

适用的场景：

- 不需要复杂查询的应用，只支持 RowKey 索引，其他需要全表扫描
- 写密集应用，HBase 是一个写快读慢的应用（慢是相对的）
- 对事务要求不高，只支持基于 RowKey 的事务
- 对吞吐量、可靠性要求高的应用，HBase 不存在单点问题，可用性高
- 数据量级特别大的应用

存储模型：

- 不同于关系数据库的行存储模式，HBase 是基于列存储的，每个列族都由几个文件保存，不同的列族文件是分离的。
- RowKey 为一行数据，每一行有若干个 column family（列族），物理存储以列族来区分，每个列族包含若干列，列是动态的。
- Hbase 是一个稀疏的、多维度、排序的映射表，索引为 (RowKey, Column Family, Column, Version）。

核心概念：

- RowKey：字典序的，基于 RowKey 实现索引
- Column Family：一个表被分为多个列族，基本的访问控制单元
- Column：列族里的数据通过列来定位
- Version：多版本数据，每一列都可以有多个版本的数据，获取指定版本的数据（默认返回最新版本）
- 稀疏矩阵：行与行之前的列数可以不同，实际列才会占用空间

## Hbase 集群

### 集群结构

- HMaster 集群：管理和维护 HBase 表的分区信息，维护 Region 服务器列表，分配 Region 负载均衡。通过 ZK 选主
- ZK 集群：用于存储 meta 信息
- Region 服务器：存储 Region 数据

Client 请求的流程是，访问 ZK 获取 hbase:meta，然后去 meta region 中获取 region 的分布信息，最后再去 region server 上获取数据。因此 master 的负载很小，client 不和 master 通信。

客户端获取 hbase:meta 集群的地址后，会缓存该地址，减少对 ZK 的压力。同时访问 meta 服务器拿到对应的 region server 后，也会缓存该地址，减少调用次数。

### HBase 强一致性

每个 Region 只有一台 Region Server 负责，所以读没有一致性问题。

HBase 底层依赖 HDFS，文件写入的话，会 sync 至少三份。所以对于 HFile 写入完成（Memstore 达到阀值，刷盘生成），会同步到另外两个 DataNode 上，数据是一模一样的。

对于 Memstore 中的增量数据，是根据写的 WAL 日志 HLog，HLog 也是用 HDFS 存储的，所以也是会有3本副本，当机器故障的时候是可以恢复的。

HDFS 的写 sync，默认是管道写，即同步到第一台服务器，第一台再同步到第二台，然后才返回客户端。也可以多路写，写同时发给3台，都成功了，客户端才能继续。管道写时间比较慢，延时高，但是能更好的利用带宽；多路写延时低，只需要等最慢的 DataNode 确认完成，但是写入需要共享发送服务器的网络带宽，可能会成为瓶颈。

Region 挂了的时候，region failover 到其他 server，需要根据 WAL log 来 redo，这时候 region 是不可用的，牺牲了可用性，保证了强一致。

参考文档：

- <https://www.cnblogs.com/captainlucky/p/4720986.html>

### Region

Region 是 HBase 分布式存储的基本单元。分为元数据 region 和用户 region 两类。

Meta Region 记录每个 User Region 的路由信息。因此读写 region 流程为，寻找 meta region 地址，再从 meta region 找到 user region 地址。

Column Family 是 Region 的物理存储单元，同一个 Region 下有多个 Column Family，存在不同的路径下面。

一个 Region 由多个 Store 组成，每个 Store 对应一个 Column Family。Store 的结构为一个 Memstore，和满了后刷新的一个个 HFile。另外 Region Server 共用一个 HLog 来写 WAL 日志，用来在故障时恢复还没写回 HFile 的 Memstore 数据。

写入的时候，先写 Memstore 和 HLog，写完 HLog 才给客户端返回成功。读取的时候，先读 Memstore，没有再去读 HFile。HFile 越来越多的时候，读取性能越来越差，可以出发 compact 来提高性能。

单 region 过大会自动分裂，现在的机器配置，大小建议设为 1~2G 左右。每个 Region 服务器存储 10~1000 个 Region。

参考文档：

- <https://cshihong.github.io/2018/05/17/HBase%E6%8A%80%E6%9C%AF%E5%8E%9F%E7%90%86/>

HFile Compact，分为两种：

- Minor Compact：选取一些小的，相邻的 StoreFile 做合并，合成更大的 StoreFIle，不处理删除或失效。
- Major Compact：将所有的 StoreFile 合并成一个文件，并且处理删除和超过 TTL 和版本的数据。

## HBase 性能优化

Row Key 设计，由于按字典序存储，因此设计的时候，把经常访问的数据放在一块，又要避免都落在一台服务器上。如把访问主体 hash 后+时间戳来代表 row key，如果经常需要获取最新的数据，可以考虑 Long.MAX_VALUE - timestamp 来代替时间戳。

创建表的时候，设置是否需要多版本，不需要的话可以 setMaxVersion 为 1.

创建表的时候，设置数据的生命周期，超过生命周期的数据将被删除。

Compact 的时候，会影响读写性能，所以尽量在业务低峰期。

Region Split 的时候，也会阻塞读写，但是其实并没有真正分裂，而是通过文件引用来快速分裂，对效率影响不大。在 compact 的时候才真正的分裂。

Scan 读取的时候，setCache 批量读的行数，setBatch 单次返回的列数（最多不超过一行请求的所有列）。也可以用 setMaxResultSize（来限制返回的字节数，另外两个设为最大值）。

## LSM 原理

Log Structured-Merge Tree，HBase、LevelDB 等使用的文件组织方式。不同于传统数据库的组织方式，提供更好的写吞吐量。

利用了硬盘顺序写效率高的特点。传统的数据库如 B+ 树等，修改都是随机写，会有比较差的写性能。

内存中维护一个 memtable，每次写入变更操作，到一定大小后，就整个作为一个文件 sstable 写入磁盘（顺序写）。每次读的时候，先读 memtable，没找到则依次的读 sstable 文件，最差的情况下需要读完 sstable。

为了优化读性能，sstable 过多后，需要 compact 到一个更大的文件中，然后每次读还是一次的读所有的 sstable，最差情况下所有 sstable 都需要读。优化 memtable 的判定，可以使用 bloom filter。

LevelDB 对此的处理方案是，不进行 Simple Compact，而是进行 Level Compact。即 sstable 过多后，把最后一个 Compact 到更高的一层文件中，而更高的一层文件是按数据分段的，即一个数据只可能出现在一个文件中。每层文件又可以继续合并到更高层文件中，这样就可以每层只读一个文件。这样会有比较好的读性能，但是 IO 次数变得更多，写场景复杂化。在 Level Compact 下，只有第0层，是无序的，需要读取所有文件，下面的每一层都是有序的，只需要读一个文件就可以了。

参考文档：

- LSM 原理：<https://www.open-open.com/lib/view/open1424916275249.html>

## 简单谈谈 OceanBase 原理

OB 按照分区 partition 来组织数据，它可以是普通表的一个分区，一个表的分区如果太大可能会放在多个 server 上，OB 会自动获取多台数据做 merge。对于有关系的表，可以设置有关系，来放在同一个 partition group 里，同一个分区组的数据，会放在同一个机器上。

通过 ObProxy 来选择访问哪个 ObServer，每份数据按分区组织，每个分区都有三个副本，三个副本在一个 paxos group 中，保证强一致性（半数节点通过）。某个副本挂了后，会通过 paxos 协议选出新的主，每个 partition 都有一个 leader，选举大约需要 14~20秒。应用会通过 ObProxy 感知 leader 的切换，不需要做任何修改。

底层的存储引擎本质是 LSM，但是做了很多优化。为了减少 compact 的影响，在 compact 的时候，OB 做了轮转合并。主副本提供服务的时候，其他副本会做 compact，其他副本完成后，主副本开始合并，流量切到其他的副本。把正常服务时间和合并时间错开。
