# Kafka

## 消息队列

消息队列解决的核心问题有：

- 解耦：将对多个下游调用的依赖，改为发送方定义消息协议并发送消息，由依赖方自己去订阅消息。
- 异步化：对耗时的处理进行异步化。
- 流量削峰填谷：平滑流量。

消息中间件选型考虑的因素：

- 吞吐量
- 响应时间
- 集群可用性
- 消息可靠性
- 特性支持：优先级队列、延时队列、重试队列、死信队列
- 社区活跃性/维护难度

常见消息队列的对比：

- Kafka：Linkedin 开源的消息队列，已成为 Apache 项目，使用 Scala 开发。特性是高吞吐量，消息会持久化存储在磁盘，客户端使用 pull 模式获取消息。
- RocketMQ：阿里开源的消息队列，已成为 Apache 项目，使用 Java 开发。基本原理：使用 CommitLog（不区分 Topic 顺序写入）+ Consumequeue（记录在 CommitLog 中的偏移量），副本的粒度为 CommitLog。优点是高吞吐（略低于 Kafka）、能够支持更多的 Topic（不会降低效率）、支持更丰富的特性（延时队列、重试队列、死信队列等）。
- RabbitMQ：使用 Erlang 开发，专注于高可靠的消息队列，支持 AMQP 等通用的消息协议，吞吐量比 Kafka 低一到两个数量级。
- Redis：pub-sub 可以当做轻量级的消息队列使用，支持多播模式，消息不进行持久化，不保证可靠性，实时发送给消费者，不理会是否被消费者收到。
- ZeroMQ：本质上不是一个消息队列，而是一套网络库，将 Socket 封装为消息队列的抽象来使用，不存在中间的 Broker 节点。

RabbitMQ 和 Kafka 的对比：

- 定位：在开始时，RabbitMQ 定位高可靠，Kafka 定位高吞吐。随着版本的迭代，两者都在短板方面进行了一定的提升。
- 性能：RabbitMQ 单机在万级别；Kafka 单机在十万级别，甚至百万级别。吞吐量最终会受到硬件限制，比如千兆网卡（1000Mbps）要达到百万吞吐量，每个消息最大不能超过 `1000*2^16 / 8 / (1*10^6) = 134B`。
- 可靠性/可用性：RabbitMQ 通过镜像环形队列（按环形同步消息，回到 master 后再按环形 ack）实现多副本和强一致语义；Kafka 通过 ISR 分区副本同步，并且支持强一致语义（通过 acks 实现）。
- 特性支持：RabbitMQ 支持延迟队列、死信队列，很容易封装实现重试队列；Kafka 不支持延迟队列、重试队列、死信队列。
- 消息获取模式：RabbitMQ 支持 push/pull 两种模式，默认是用 push 模式；Kafka 只支持 pull 模式。Pull 模式更适合有多种不同处理能力的 consumer，由消费者自己决定拉取频率和批量大小；Push 模式消息实时性更高。

## Kafka 核心概念

Kafka 是一个分布式的，基于发布订阅的消息系统。

核心角色：

- Kafka Client：
    - Producer: 消息发送者，生成消息并投递到 Broker 中。
    - Consumer: 消息消费者，从 Broker 中拉取消息并处理。
- Kafka Server:
    - Broker 集群: Kafka 服务实例集群，提供消息处理服务。
    - ZooKeeper 集群：用来存储 Kafka 集群的一些元数据。

核心概念：

- Topic (主题): 消息以主题为单位进行分类，生产者发送消息到某个主题中，消费者消费某个主题的消息。
- Partition (分区): 消息并发的单位，用来提高 Topic 消息的吞吐量。一个 Topic 可以有多个 partition，每个 partition 的发送和消费是有序的。
- Repilica (副本): 分区有多副本机制，用来提高容灾能力。副本具有主从关系，Leader 副本负责处理读写请求，Follower 只负责 Leader 副本的消息同步。当 Leader 出现故障时，从 Follower 中选举新的 Leader。
    - AR (Asiggend Replicas): 分区的所有副本，AR=ISR+OSR
    - ISR (In-Sync Replicas): 与 Leader 分区保持一定程度同步的副本
        - LEO (Log End Offset): 副本日志文件中下一条待写入的消息 offset
        - HW (High Watermark): ISR 中最小的 LEO，即同步最慢的节点的同步位置。消费者只能消费 HW 之前的数据。
    - OSR (Out-of-Sync Replicas): 与 Leader 分区同步滞后过多的副本。如果副本在 `replica.lag.time.max.ms`（默认为10s）内，没有赶上过一次 Leader 的 LEO，则视为滞后过多，将被踢出 ISR。
- ConsumerGroup（消费组）：每个消费者属于一个消费组。对于某个 Topic 的消息，每条消息只会被发到订阅它的消费者组的一个消费者实例上。可以通过多个消费者组订阅一个 Topic 来实现广播功能。

## 生产者/消费者

生产者核心配置：

- `acks`，多少副本收到消息视为发送成功，在可靠性和吞吐量间平衡。有三种类型的值：
    - 0: 发送消息后不等待回应。如果消息从发送到写入 Kafka 出现了异常，生产者也不知道，消息直接丢失。
    - 1（默认值）: 分区 leader 副本收到消息，就会向生产者响应成功。
    - -1 (all): 所有参与复制的 ISR 都收到消息，生产者才会响应成功。如果需要保证可靠性，还需要配合 broker 的 `min.insync.replicas` 参数。
- 生产者可以设置重试次数（`retries`，默认为 0，不重试）和重试间隔（`retry.backoff.ms`，默认为 100）来进行消息重试。但是重试可能导致 partition 内消息乱序，如果对顺序有强要求，应将 `max.in.flight.requests.per.connection`（收到 Broker 响应前可以发多少消息，默认为5）设为 1。

生产者客户端架构：

- 分为客户端线程和 Sender 线程两个线程。客户端线程负责创建消息，经过拦截器、序列化器、分区器后缓存到消息累加器（RecordAccumulator）中；Sender 线程从消息累加器中获取消息并发送。
- 消息累加器主要用来做批量发送，减少网络传输的资源来提升性能。可以根据 `buffer.memory` 配置，默认为 32M。累加器满了后发送消息会阻塞（或超时异常，默认 60s）。
- 累加器会进行消息合并，通过发送批次消息来减少网络请求次数，提升吞吐量。

消费者概念：

- Kafka 同时支持单播与广播。提供了一个消费组的概念，同一个消费组中的消费者，消息只会投递给其中一个；可以通过创建多个消费组，来支持消息的广播，各个消费组的消费是独立的。
- Kafka 的消费是基于拉模式的，消费者使用 offset 来表示消费到的消息的位置。旧版本的 Kafka 消费位移是存储在 ZooKeeper 中的，新版本存储在 Kafka 的内部 Topic `__consumer_offsets` 中。消费者消费完消息后，需要提交位移。
- 位移的提交默认是自动提交（`enable.auto.commit`，默认为 true），Kafka 客户端会固定时间间隔提交（`auto.commit.interval.ms`，默认为 5s）。因此可能存在重复消费的问题。也可以进行手动提交来进行更加精细的管理。
- 每个分区只能被一个消费组中的一个消费者消费。因此分区固定的情况下，在一个消费组中，如果消费者数量大于分区数量，会有消费者分配不到任何分区。

新版消费 offset 的保存：

- 发送提交到对应的 GroupCoodinator（Broker）上，即 `__consumer_offsets` 对应的分区 Leader。
- offset 保存的时间根据配置 `offsets.retention.minutes`（默认为7天），过期的日志使用压缩的清理策略，消息的 key 为 `Group+Topic+Partition`。
- 如果消费者超过 `offsets.retention.minutes` 没有进行消费（位移提交），原来的 offset 过期，则会根据消费者的 `auto.offset.reset`（earliest；最早的消息；latest: 最新的消息；none：抛出异常） 配置来决定起始消费的位置。

## 日志存储

Kafka 的日志（数据）文件存储在硬盘上。

每个 Topic 下的 Partition 存储为一个单独的文件夹。文件夹命名为 `${topic}-${partition_seq}`，seq 为从 0 开始递增的序列号，即最大为 partition 数量减 1。

Partition 文件夹内部，存储着消息文件。每条消息（Log Entry）在 Partition 内有一个递增的 64 位 offset。一个 Partition 可以分为多个 Segment（逻辑概念），每个 Segment 的命名为固定的20位字符串，使用当前 Segment 中最小的 offset（前置填充 0）命名。

每个 Segment 物理上分为多个文件，数据文件（`.log"`）、日志索引文件（`.index`）、时间索引文件（`.timeindex`）。每个数据文件的大小不必相同，数据文件根据配置切割，可以按照大小（`log.segment.bytes`，默认 1G）、时间（`log.roll.hours`，默认7天）、索引大小（`log.index.size.max.bytes`，默认10M）。

日志文件是顺序写，写入时使用 mmap，直接将消息写入内核缓存中，避免用户态到内核态复制的开销，同时由操作系统决定什么时候写回磁盘文件。对于 mmap 写入可能不可靠问题，可以通过设置参数来同步或异步刷回磁盘，也可以依靠 replica 机制来保证可靠性。

Kafka 的索引均为稀疏索引，在效率和存储空间中进行折中，由于占用较小，可以加载到内存中，提高查询效率。根据 `log.index.interval.bytes` 配置（默认为 4K），写入多少大小的消息后生成一条新的索引，可以根据这个配置调整索引的稀疏密度。

日志索引文件的存储格式为：相对偏移量（相对于文件名 offset，4B）和对应的数据文件的物理偏移地址（4B），单条数据大小为 4+4=8B，使用相对偏移量是为了节约存储空间。索引的查询过程为，先根据跳表查询到对应的 Segment，然后在 Segment 的 index 文件中做二分查找，找到物理偏移地址后去数据文件查询消息。

时间索引文件的存储格式为：时间戳（8B）和相对偏移量（4B），单条数据大小为 8+4=12B。时间戳可以根据配置 `log.message.timestamp.type` 来使用 LogAppendTime（Broker 时间） 或者 CreateTime（Producer 指定），时间戳索引保持单调递增。时间索引的查询过程为，先使用顺序查找找到对应的 Segment，然后在分段的 timeindex 中做二分查找，找到相对偏移量后去 index 文件中进行二分查找，找到物理偏移地址后去数据文件查询消息。

消息数据的格式主要演进了三个版本，v0、v1 版本区别不大，主要是支持了时间戳。v2 版本改动较大。具体格式如下：

- v0/v1: 消息对象为 MessageSet，主要包含消息头（offset、messageSize）和消息体（CRC、版本号、属性、key 长度、key、value 长度、value）。对于消息压缩，外层 MessageSet 的值为多个 MessageSet，内层 MessageSet 的消息头都存相对的位移。v1 版本的消息体比 v0 增加了一个 timestamp offset。
- v2: 使用了 Record Batch 和 Record。Record Batch 用来做压缩消息的外层消息，包含了 attributes、offset 和 timestamp，和用来支持幂等和事务的字段（producer epoch、producer id），以及内层消息最大/最小的 offset、timestamp offset、消息数量等；Record 存了相对于外层的 offset delta 和 timestamp delta，以节省空间。引入了 Varints（变长整型）和 ZigZag 编码，压缩了小数字和小负数的存储空间。支持 Message Header。

日志文件的清理。支持日志的清除（Rentition）或压缩（Compaction）。

- 日志的清除指的是，根据配置 `log.retention.ms`（默认7天）来清除过旧的日志，或者根据配置 `log.retention.bytes`（默认-1，无穷大）在日志量大小超过阀值后删除旧的日志。也可以手动根据日志的偏移量进行清除。
- 日志的压缩指的是，对于相同 Key 的日志，只保留最新的一条数据，生成新的 segment 文件。Kafka 中的用来保留消费者位移的主题，就是用的日志压缩策略。

## 消息投递语义

消息投递主要有三种语义：

- At most once：消息可能会丢失，但绝不会重复投递
- At least once：消息绝不会丢失，但可能会重复投递
- Exactly once：消息肯定会被传递一次且仅传递一次，既不会丢失消息，也不会重复投递。

Kafka 并没有提供一个选项来控制整体的消息投递语义，而是需要通过合理的设置 producer/broker/consumer 的配置，来达到某种投递语义效果。

在实际业务中，很少会使用 "At most once" 消息语义。通常使用 "At least once + 支持消费幂等" 来作为消息的处理方案。

如果要支持 At least once 语义，需要进行以下配置：

- Producer：
    - acks（默认值为 1）：需要配置为 -1（all），来确保所有的 ISR 节点复制完才返回成功。
- Broker：
    - min.insync.replicas（默认值为 1）：需要设置为大于 1，例如 2。当生产者设置了 acks 为 all 时，指定了 ISR 集合中的最小副本数，如果小于副本数，则抛出异常。
    - unclean.leader.election.enable（默认值为 false）：需要设置为 false，不允许非 ISR 节点选举成为 leader，否则可能发生数据丢失。
- Consumer：
    - enable.auto.commit（默认值为 true）：设置为 false，并且手动管理位移的提交，当消费失败的时候不提交消息。然后再进行重复消息，并且在业务实现上支持幂等的处理。在某些框架里也可以配置为自动提交，且当有错误的时候不进行提交。

如果要支持 At most once 语义，需要进行以下配置：

- Producer：
    - acks：配置为 0，发送消息后不等待回复，即无视是否发送成功。
- Consumer：
    - 消费者先提交 offset 再处理消息。

如果要支持 Exactly Once 语义，需要在 "At least once" 配置的基础上，再进行以下配置：

- Producer：
    - enable.idempotence：设置支持发送的幂等。
- Consumer：
    - 要求消费的处理逻辑和 offset 的提交是原子性的。即消费成功则 offset 一定能提交，消费失败 offset 需要跟着回滚。可以使用分布式事务，或者将 offset 和业务数据存储在同一个支持事务的地方。

## Kafka 的事务

幂等和事务特性是支持 EOS（Exact only semantics）的核心。

生产者通过设置 `enable.idempotence=true` 来开启幂等。Broker 会为生产者分配一个 PID，然后为 `<PID, 分区>` 维持一个序列号，收到消息的时候，要求序列号是连续递增的，对旧序列号消息忽略，而不连续的消息报错（说明发生了消息丢失）。幂等是针对每个 `<PID, 分区>` 来说的，即只保证单个生产者中单分区的幂等，如果把同一个消息发送到两个分区，则不会保证幂等。

事务可以支持多个分区写入的原子性，事务开启后会默认开启幂等。每个 Producer 需要自己设定一个唯一的 TransactionId，如果使用同一个 TransactionId 启动两个生产者会报错。不直接用由 Broker 生成的 PID，而是增加一个 TransactionId 概念，是为了生产者失败重启后可以继续处理前面的事务，并且 Broker 能忽略之前 Epoch 的消息。

1. Producer 向 Broker 发送请求确定 TransactionCoodinator 节点。Producer 对应的 TransactionCoodinator 为内部 Topic `__transaction_state` 的对应 Partition（`hash(txnId)%transactionStatePartitionCount`）所在 Leader 的 Broker。
2. 向 TransactionCoodinator 获取 PID，TransactionCoodinator 记录 `<TransactionId, PID>` 的映射。
3. Producer 开启事务，发送消息，除了写入到对应的分区外，还会向 TransactionCoodinator 写入事务信息。
4. Producer 提交事务，TransactionCoodinator 向事务中消息的各个分区发送 commit 控制消息；Producer 终止事务或者事务超时，TransactionCoodinator 向事务中消息的各个分区发送 abort 控制消息。控制消息也作为一条正常日志信息存储。

消息的事务是针对消息发送的，它并不能保证一个事务的消息都被消费到。例如消费者可以从任意位置开始消费；事务中可能部分日志已经因为过期被清理等。

消费者支持两种隔离级别，`read_uncommitted`（默认）和 `read_committed`。

- read_uncommitted：能读到 HW 前的所有消息。
- read_committed：只能读到 LSO（Last Stable Offset），即该分区未完成事务的第一条消息的位置。即只能读到已提交的数据。Consumer 实际消费的时候会缓存之后的数据，但是在收到控制消息 commit 之前，不会把这些数据推给用户，如果收到控制消息 abort，则会丢弃这些消息。

对流式应用（典型的场景为 consume-transfrom-produce），从 Kafka 消费数据，并且生产的结果也发往 Kafka。可以通过事务特性，来保证操作的原子性，来达到了消费的 Exactly Once 语义。

## 集群高可用

Kafka 集群使用 replica 来保证集群的高可用，避免某台机器宕机导致整个集群不可用。Replica 是 Partition 维度的，会有一个 Leader 和若干个 Follower，由 Leader 提供消息读写服务；Follower 进行数据复制，不提供消息读写服务。

Kafka 既不是同步复制，也不是异步复制。使用 ISR+HW 的机制，在可靠性和性能之间进行权衡。ISR 和 Majority Vote 复制相比，Kafka 需要的冗余度较低，当要容忍f个副本失败，两者 commit 前都需要同步 f+1 个副本，ISR 需要的总的副本数为 f+1，而 Majority Vote 需要的副本数为 2f+1。因此 ISR 需要的总副本数量较少，复制带来的集群开销更低。

Replica 如何在集群中分配？整体目标，分区副本在 Broker 中尽量均衡分配：

- 对于某个 Topic，随机它的首个 Partition 的副本在 Broker 中的起始位置（避免都从第一个 Broker 开始分，导致负载不均衡），其他 Partition 的首个副本相对于首个 Partition 依次向后移动一个 Broker。对于每个 Partition，剩余的第一个副本相对于首个副本根据随机数 `secondReplicaShift` 来进行一个偏移，偏移后再依次向后（循环）分配其他副本到 Broker 中。

Kafka 的日志复制过程：

1. 分区 Leader 收到消息A后，写入消息日志，更新自己的 LEO。
2. 分区 Follower 向 Leader 轮询 Fetch 最新的日志数据；Fetch 时，会将自身的 LEO 作为参数。
3. Leader 收到 FetchRequest 后，取最小的 Follower 的 LEO 作为分区的 HW，将最新的日志数据和 HW 信息返回给 Follower。
4. Follower 将收到的日志写入本地磁盘，更新 HW 和 LEO 信息。
5. Follower 再次向 Leader Fetch 时，带上新的 LEO 信息（已经包含了消息A），Leader 就能更新 HW 到消息 A 的下一个位置，即消息A在集群中已经保证写入成功。

根据上面的日志复制过程，当 Producer 设置了 acks 为 all 的时候，调用消息发送后，消息写入 Leader 后，需要 Follower 进行两轮 fetch，Leader 才能返回成功给 Producer。因此生产者使用高可用的发送语义，会增加端到端延时。

当分区 Leader 宕机后，如果设置参数 `unclean.leader.election.enable` 为 true，则从 ISR 副本中选择一个作为新的 Leader。由于 Follower 和 Leader 的数据同步需要一次 Fetch 的延时，可能存在不一致（如 Leader 宕机前已经更新了自身 HW，但是还没有返回给 Follower）。因此分区产生新的 Leader 后，会保留自身的所有数据，并递增 Leader Epoch。其他 Follower（可能有原 Leader 重启成功）不能根据自身的 HW 来决定是否进行日志截断，而是传给 Leader 自身的 Epoch，Leader 告诉它之前 Epoch 的最大日志 Offset，Follower 将之后的日志进行截断。

根据以上的分析，在全链路使用适当的配置，支持了 "At least once" 消息投递语义下，集群能够通过副本机制来保证消息的高可靠。

## 负载均衡

Partition 是 Kafka 中的最小并行操作单元。吞吐量与分区数量的关系为，吞吐量先随着分区的增加而提高，达到一个临界值后随着分区的增加而下降。因此分区数量的选择要合适，过多和过少都有一定的缺点：

- 分区过多：客户端/服务器需要更多的内存；打开更多的文件句柄；增加端到端延时（副本复制延时变高）；宕机 Leader 选举时间变长，影响可用性。
- 分区过少：限制了消息发送和消息消费的并发度，影响系统吞吐量。

Partition 数量的设置建议：

- 根据预期的吞吐量估算：假设 t 为预期吞吐量，p 为生产者单个分区吞吐量，c 为消费者单个分区吞吐量。则至少要设置为 max(t/p, t/c) 个分区才能达到吞吐量预期。
- 是否需要全局顺序性消费：如果消息需要全局顺序性，则分区只能使用一个，避免并发。

生产者的分区策略（默认为 `org.apache.kafka.clients.producer.internals.DefaultPartitioner`），当发送消息时，按照以下优先级依次确定分区：

1. 如果指定了分区，则直接使用。
2. 如果指定了 Key，则根据 Key 的 Hash 值选择一个分区，保证相同 Key 的数据是有序的。
3. 使用黏性分区策略，优先使用一个分区，直到分区的 Batch 塞满或者状态完成，之后再随机选择一个分区。这种策略能充分的利用生产者的批量发送特性，减少请求次数，把时间维度拉长，消息也能够均匀的分布到各个分区。在 2.4 版本之前使用轮询分区策略，不能充分地利用批量发送特性。

消费者的分区分配，主要由 GroupCoordinator 和 ConsumerCoodinator 负责：

- GroupCoordinator: 根据 `hash(groupId)%groupMetadataTopicPartitionCount` 找到对应的 `__consumer_offsets` 的Partition，找到该分区的 Leader，该 Broker 即为这个消费组的组协调器。GroupCoodinator 会负责存储该消费组的位移和进行消费组的重分配触发。消费者最开始向负载最低的节点请求获取 GroupCoordinator 信息。
- ConsumerCoodinator：消费组分区分配的计算是由 Consumer 完成的，而不是由 GroupCoordinator 决定的。策略实现在 Client 端，可以灵活地进行改变，并降低 Broker 负担。

整体流程为，所有成员向 GroupCoordinator 发送请求加入组，GroupCoordinator 选出 ConsumerCoodinator，并将信息发送给它。ConsumerCoodinator 收到成员信息后，计算出具体的分配结果返回给 GroupCoordinator，再由 GroupCoordinator 将方案发给各个 Consumer。

消费者默认支持的分配策略，有以下三种：

- RangeAssignor（默认策略）：每个 Topic 的分区独立分配。对于某个 Topic，将消费者按序号排序，将所有的分区排序，然后将分区按顺序（循环）分给消费者，即前面的几个消费者对于每个 Topic 有可能多分一个分区。如果集群中消费者数量远大于大部分 Topic 的分区数量，则前面（序号）的消费者负担很重，后面的消费者闲置。
- RoundRobinAssignor：将消费者组内的所有消费者和订阅的所有主题的所有分区按照字典排序，然后通过轮询的方式将分区依次分给消费者。即把一个消费者组订阅的所有 Topic 分区组成 t0p0 的形式，分给这些消费者。这样在消费者数量大于每个 Topic 分区数时，不会把消息都分给前面的消费者，保证消费负载的均衡。
- StickyAssignor：优先保证分区尽可能均匀；其次在消费者 rebalance 时，尽可能少的改变现有分配。

为什么只有 Leader 处理消息读写，而不使用读写分离来做负载均衡？

Kafka 不是特别符合读多写少的场景，如果使用读写分离，主节点负担可能还是很重，并没有做到特别的均衡，而且读写分离还会引入一致性和延时问题。Kafka 通过 Partition 的均衡分配来实现每个 Broker 节点总体流量（含读写）的负载均衡。

## Kafka 的选举

Kafka 里有多个地方会进行 Leader 选举，主要有：

- 控制器选举：集群中会有一台 Broker 被选举为 Controller，它负责管理集群中所有分区状态和副本。比如选取新的分区 Leader、通知其他 Broker 元信息的变化。选举的方法很简单，在 ZK 中创建 `/controller` 这个临时节点，创建成功的就是 Leader，其他的节点都会监听该节点。使用控制器而不是每个节点分别监听 ZK 是为了避免所有的节点都需要监听大量的 ZK 元数据节点，造成 ZK 过载。
- 分区 Leader 选举：当分区 Leader 宕机后，由控制器进行选举。在 AR 集合中的副本按顺序查找第一个在 ISR 中的副本。使用 AR 集合进行查找的原因是，AR 的集合顺序是稳定的，且第一个为优先副本，而 ISR 随着运行顺序会发生变化。
- ConsumerCoodinator 选举：第一个加入消费组的消费者就是 Leader，后续如果 Leader 退出，重新选举，则从消费组成员的 HashMap 取出第一个，就相当于随机选取。

## Kafka 性能关键技术

Producer：

- 批量发送：Nagle 策略，利用缓冲区和超时时间，合并消息进行批量发送。减少了网络 IO 的次数，提高了有效 payload 占比，提高了传输效率。
- 数据压缩：数据压缩后发送，Broker 存储压缩的消息，消费者进行解压，提高了网络传输效率。

Broker：

- 通过 Partition 来提供并行处理的能力。
- 通过 ISR 来在可用性和一致性之间进行平衡。
- 消息日志文件顺序写磁盘，使用 mmap 来提高消息写入效率。可以使用 replica 来保证 Page Cache 异步刷回时候的可靠性。
- 文件分段+稀疏索引。
- 使用 sendFile 系统调用进行 Zero Copy，减少了上下文切换次数和数据拷贝次数（避免了 CPU 拷贝，全部由 DMA 拷贝）。日志文件通过 DMA 从磁盘文件拷贝到内核 Read Buffer，然后再通过 DMA 直接拷贝到网卡中。

Consumer：

- 批量拉取，消费后定时提交 offset，减少网络交互次数。可能有重复消费问题，需要幂等设计。

## 参考文档

- 《深入理解 Kafka：核心设计与实践原理》：<https://book.douban.com/subject/30437872/>
- 消息中间件选型分析：<https://www.infoq.cn/article/kafka-vs-rabbitmq>
- Kafka 高性能关键技术解析：<https://www.infoq.cn/article/kafka-analysis-part-6>
- Zero Copy 原理：<https://www.cnblogs.com/xiaolincoding/p/13719610.html>
