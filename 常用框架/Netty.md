# Netty

Netty 是一个基于 NIO 的 Client/Server 网络编程框架。

优点：

- 提供了简单而强大的线程模型。
- 提供了常用的协议栈处理能力。
- 相比于 JDK 原生的 NIO，更加容易使用。
- 提供了很多高性能优化，更低的资源消耗和更少的内存复制。
- 社区活跃，成熟稳定。很多开源项目都使用了 Netty，如 Dubbo、RocketMQ 等。

Netty 的核心组件：

- Channel：对网络操作类的抽象，用来处理网络 IO，是对 Socket 类的封装，提供更方便的 API。
- ChannelHandler：Channel 上的处理器
- ChannelPipeline：责任链模式，将多个 ChannelHandler 组成的链式处理。每个 Channel 有它专属的 ChannelPipeline，用来处理 Channel 产生的具体事件。
- EventLoop：EventLoop 是一个 Reactor 处理器，对应一个线程，内部会维护一个 selector 和 taskQueue 来处理 IO 和内部事件。一个 EventLoop 可以处理多个 Channel。
- EventLoopGroup：EventLoopGroup 包含了多个 EventGroup，可以认为是一个 Reactor 的线程池。

Netty 服务端启动时需要指定两个 EventLoopGroup：

- bossGroup：用来接受客户端连接请求。
- workerGroup：用来处理某个连接的具体 IO 请求。

创建 EventLoopGroup 时，可以指定其中的 EventLoop 数量，即 Reactor 线程的数量，默认为 `CPU核心数*2` 个。

Netty 的线程模型：

- 单线程模型：bossGroup 和 workerGroup 使用同一个，且大小为 1。这样一个 Reactor 线程负责所有请求的接入和处理，Redis 就是这种线程模型。
- 多线程模型：bossGroup 大小为 1，workerGroup 大小为 n。相当于 bossGroup 中唯一的 Acceptor 接收到连接后，把连接的请求注册到 workerGroup 中的一个 Reactor 线程处理。
- 主从多线程模型：bossGroup 大小为 n，workerGroup 大小为 n。BossGroup 的一个 Acceptor 线程负责接收请求，之后随机在 bossGroup 选一个 Reactor 线程来处理具体的连接创建（认证、握手等），连接创建成功后，把连接的请求注册到 workerGroup 中的一个 Reactor 线程处理。

通常情况下，使用多线程模型可以满足性能的要求。当客户端连接过多，或者创建连接需要进行安全认证之类时，一个 Acceptor 线程处理请求的创建可能性能不足，这时可以使用主从多线程模型。

Netty 的一些核心优化：

- 无锁化串行设计：让消息的处理尽量在一个线程内完成，期间不进行线程切换，避免了多线程竞争和同步锁。通过调整 NIO 线程池大小，可以通过启动多个串行化线程的并行，这种局部无锁设计比多线程+单个队列的模式性能更优。
- 零拷贝：通过 `FileChannel.tranferTo` 直接将文件拷贝到 Socket 缓冲区，来实现零拷贝。通过 ByteBuf 使用堆外内存，避免在网络 IO 时堆内到堆外的内存拷贝。
- 规避 NIO Bug：JDK Selector 空轮询 bug（JDK 8 该问题也没有被完全修复，只是发生概率变低），当没有就绪事件时，selector 可能也会返回，导致 CPU 使用率 100%。Netty 通过统计空轮询次数，超过一定次数后，使用新的 selector 替换原 selector。
