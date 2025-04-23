# Java 并发编程

## JMM

JMM (Java Memory Model) 是 Java 对内存模型的一种简化抽象，屏蔽了 CPU 多级缓存、寄存器、指令重排等细节，将内存模型抽象为线程本地缓存和主存两层模型，其中线程本地缓存的更新只对该线程可见，主存的更新对所有线程可见。

顺序一致性模型与 JMM 模型：

- 顺序一致性模型：所有线程的操作是顺序执行的，并且立刻对其他线程可见。
- JMM 模型：会进行各种指令进行重排，只提供最小的安全性，即线程读取到的值，要么是之前其他线程写入的，要么是默认值，不会无中生有。

指令重排序可能来自于：

- 编译器重排序
- 指令级并行的重排序：现代 CPU 可以多指令并行执行或进行分支预测。
- 内存系统的重排：处理器的读写缓冲区让加载和存储看起来像乱序执行。

重排序的基本原则：`as-if-serial` 语义，不论怎么重排，单线程的程序执行结果不能变。因此单个线程内逻辑上有依赖的操作不能进行重排。

JMM 为了提供简化的模型，并减少对编译器和处理器的约束。提供了和 `as-if-serial` 语义一样的 happens-before 规则，即前面操作的结果对后面操作可见。主要规则如下：

- 程序顺序规则：一个线程中的每个操作 happens-before 它的后续操作。
- 监视器锁规则：对于锁的解锁 happens-before 之后对这个锁的加锁。
- volatile 变量规则：volatile 写 happens-before 之后对这个 volatile 的读。
- 传递性规则：如果 A happens-before B，B happens-before C，则 A happens-before C。
- start 规则：A 启动的 B 线程，则 A 的 start 操作 happens-before B 的所有操作。
- join 规则：如果 A 执行 `B.join()`，则 B 的所有操作 happens-before A 的 `B.join()`。

happens-before 并非禁止重排序，如果重排序不影响 happens-before 语义，则 JMM 运行这样的重排序。

## 线程安全

线程安全指的是多个线程并发的访问某个类，不使用额外的同步，这个类能保持正确性。正确性指的是类的行为和它的规约一致。

实现线程安全的常用方式：

- 不可变：不可变的类是线程安全的，如 String、各种无状态的工具类。
- 互斥同步：使用 synchronized 或者 Lock 进行正确的同步。
- 非阻塞同步：使用 CAS 等无锁机制进行同步，如 Atomic 类。
- 线程封闭：Ad-hoc 线程封闭（由实现者自己控制的封闭）、栈封闭、ThreadLoad 封闭。

## volatile

volatile 主要特性：

- 单个变量读写的原子性
- 内存可见性
- 禁止部分指令重排序

volatile 从 JSR-133（JDK 5）开始支持 happens-before 规则，提供了一种比锁更轻量级的线程通信机制。

在 JMM 下，volatile 变量的读写语义可以理解为：

- 读 volatile 变量后，将线程本地缓存置为无效，从主存读取。
- 写 volatile 变量前，将线程本地缓存刷回主存。

volatile 的实现原理：

- volatile 写不能和前面的所有操作重排序；volatile 读不能和后面的所有操作重排序；volatile 写不能和后面的 volatile 读重排序。
- 插入对应的内存屏障以禁止屏障前后对应的操作重排序。主要的四种屏障为 StoreStore、StoreLoad、LoadStore、LoadLoad。通常会组合使用屏障，两种常用的组合屏障为：
  - Write Release Barrier = StoreStore + LoadStore，即该写操作前的所有指令都不能和该写操作乱序。
  - Read Acquire Barrier = LoadLoad + LoadStore，即该读操作后的所有指令都不能和该读操作乱序。
- volatile 写前面插入 Write Release Barrier，后面插入 StoreLoad Barrier；volatile 读后面插入 Read Acquire Barrier。
- 不同处理器对重排序的准许情况是不同的，JMM 只插入必要的屏障即可。比如 x86 限制比较严格，只允许 StoreLoad 重排；PowerPC 限制比较宽松，允许四种重排。因此在 x86 体系下只需要在 volatile 写后面插入 StoreLoad Barrier 即可，其他的几个屏障可以省去。

CPU 级别的内存屏障的原理，MESI 协议、Store Buffer、Invalidate Queue。

- volatile 写会将 Store Buffer 的数据刷回 L1 缓存；volatile 读会让 CPU 先处理完成 Invalidate Queue 里的所有缓存失效请求。数据一旦写入缓存，则由 MESI 协议保证数据的一致性。

MESI 协议是一种广泛使用的支持缓存写回策略的缓存一致性协议。在 MESI 协议下，CPU 可以根据自身缓存中数据的状态，保证读取到一致的值。

- MESI 协议属于 snooping（窥探）协议，所有内存传输都在一条共享的总线上，所有的处理器都能看到这条总线。窥探协议的思想是，缓存不仅仅在做内存传输的时候才和总线打交道，而是不停地在窥探总线上发生的数据交换，跟踪其他缓存在做什么，来控制自身缓存的状态。
- MESI 是四种缓存行状态的缩写：
  - I（Invalid）：缓存已失效，内容已过时，下次需要从内存读取。
  - S（Shared）：共享状态，是内存数据的一份拷贝，只能被读取，不能被写入。它可以在多个 CPU 的缓存中。
  - E（Exclusive）：独占状态，是内存数据的一份拷贝，但是只存在一个 CPU 缓存中，其他处理器原先持有的会失效。
  - M（Modified）：已修改，被 CPU 修改了还没有刷回到内存，一定先获取 E 状态后才能修改，转为 M 状态。

CPU 支持 MESI 协议时，为了提高处理效率，引入了两个额外组件：

- Store Buffer：CPU 写入数据时候，等其他 CPU 答复缓存失效状态时，先写入 Store Buffer。等到其他 CPU ACK 缓存失效成功后，再刷入 L1 缓存。
- Invalidate Queue：由于 Store Buffer 大小有限，满了后会阻塞 CPU 的数据写入，因此每个 CPU 引入 Invalidate Queue 来快速 ACK，即先把消息入队，而不是真的去失效缓存。

两者的引入提高了处理的效率，但是破坏了缓存的一致性语义，因此提供了 Memory Barrier 指令，可以让开发者进行抉择，在一致性和效率之间平衡。

## synchronized 监视器锁

在不同位置使用 synchronized，锁定的对象：

- 普通方法：锁在当前实例对象上。
- 静态方法：锁在当前类 class 对象上。
- 同步代码块：锁在 synchronized 指定的对象上。很多类库实现锁定在类的一个私有的成员变量上，优点是调用方不能干涉库的同步规则。

JVM 对于 synchronized 的实现：

- 方法级同步使用 ACC_SYNCHRONIZED 标记。
- 同步代码块使用 monitorenter/monitorexit 指令。

升级为重量级锁时，这些指令底层通过 C++ 的 ObjectMonitor 类实现，每个实例有一个 BLOCKING 队列和 WAITING 队列。

锁的优化：

- 锁升级：无锁->偏向锁->轻量级锁->重量级锁。
- 锁粗化：多个连续的加锁合并为同一个锁，因为获取和释放锁也有性能损失。
- 锁消除：JVM 在 JIT 编译器逃逸分析后，对于不可能存在竞争的锁，直接去掉。
- 自适应自旋：轻量级锁自旋等待的次数，JVM 会根据之前的自旋时间来动态调整自旋的时间。

JDK 1.6 版本优化了 synchronized 的性能，增加了偏向锁和轻量级锁。锁升级的具体流程：

- 偏向锁：为了减少无竞争时锁定的开销，很多时候一个锁大部分时间是由一个线程持有的。线程把自己的 Thread Id 写入到锁的对象头 MarkWord 中，之后访问时比较 MarkWord 是自己的 id 则直接执行，轻量锁不会主动释放。当其他线程想要竞争锁，则会在时间停止时（safe point），JVM 确认线程是否还持有锁，如果没有则撤销偏向锁，否则升级为该线程的轻量级锁。
- 轻量级锁：当锁竞争不多，且持有时间较短时，与其阻塞进入内核态，不如自旋的获取锁，等待锁释放。将自己的 DisplaceMarkWord CAS 到锁的对象头 MarkWord（比较 MarkWord 是无锁状态则 set）中，自旋 CAS 超过一定的次数，则升级到重量级锁。
- 重量级锁：进入 BLOCKED 状态，并加入锁的 BLOCKING 队列，等待唤醒。

synchronized 和 ReentrantLock 对比：

- synchronized 是 JVM 支持的；ReentrantLock 是 JDK 实现的。
- JDK 1.6 之前，synchronized 性能较差，1.6 优化后性能与 ReentrantLock 相当。
- ReentrantLock 提供了更丰富的策略和灵活 API，如公平性支持、可中断、超时，可以不在一个方法里解锁，支持多个 Condition，从而支持多个等待队列。

在 JDK 1.6 后，synchronized 和 ReentrantLock 性能已经没有明显差距。通常可以使用 synchronized，因为语义方便清晰，对各种 JVM 调优工具友好。ReentrantLock 提供了更加丰富的特性，当我们需要这些特性时，选择 ReentrantLock。

## 线程

线程的状态：

- NEW：新建了一个线程对象，还没调用 `start()` 方法。
- RUNNABLE：Java 将 Ready 和 Running 两个状态都成为 Runnable。Ready 还没有获取到时间片，等待 CPU 执行。
- BLOCKED：阻塞在 synchronized 代码块上。
- WAITING：不带超时时间的等待。
- TIMED_WAITING：带超时时间的等待。
- TERMINATED：线程已执行完毕。

线程的状态图：![线程的状态图](https://segmentfault.com/img/remote/1460000023194699)

线程的开销：

- 创建和销毁的开销：可以通过线程池减少这部分的开销。
- 上下文切换的开销：时间片有最小值，即线程最短执行时间，避免过于频繁的切换。线程在阻塞或者调用 `yield` 时，会让出时间片。

最大线程数估算：

- XSS：每个线程的堆栈大小，默认值为 1M。
- `最大线程数=(系统可用内存-JVM分配堆内存)/XSS`

活跃性风险：死锁；饥饿（公平性）；活锁，可以重试时增加随机性。

## 线程协作

常用的协作范式：

- 等待通知范式：两个（多个）线程协作，一个线程循环检测等待条件，满足则执行操作，不满足则等待；某线程修改触发条件后，通知等待的线程。
- 等待超时范式：一个线程等待结果或者到某时间点超时。

```java
// 等待通知范式-等待方
sychronized (obj) {
    while (条件不满足) {
        obj.wait();
    }
    doSth();
}

// 等待超时范式
sychronized (obj) {
    long future = System.currentTimeMillis()+mills;
    long wait = mills;
    while (条件不满足 && wait>0) {
        obj.wait(wait);
        wait = future - System.currentTimeMillis();
    }
    if (wait > 0) {
        doSth();
    }
}
```

`wait/park` 都可能被意外的唤醒。因此范式中要处理意外唤醒，通常都是循环检测条件，不满足则继续等待。

常用的等待通知工具：

- Object：`wait/notify/notifyAll`，wait 可以带超时时间。
- Lock Condition：`await/signal/signalAll`，await 可以带超时时间。
- LockSupport：`park/parkNanos/unpark`

wait 的执行过程：

- 先获取对象的锁，调用 wait，线程进入 `WAITING/TIMED_WATING` 状态。
- 当被唤醒后，线程变成 `BLOCKED` 状态，重新获取锁后才会从 wait 调用中返回。

`wait/notify` 和 LockSupport 的 `park/unpark` 的对比：

- park/unpark 可以以任意顺序调用，类似单值信号量；notify 必须在 wait 后执行，否则信号会丢失。
- park 不需要提前获取锁；wait 必须要先获取到对象的锁才能调用，调用后会释放当前对象的锁.

`wait/notify` 和 Condition 的 `await/signal` 对比：

- 两者用法相似，使用前都需要获取到对应的锁，wait 需要先获取到监视器锁，await 需要先获取到 Lock。
- 一个 Lock 可以有多个 Condition 实例，有多个等待队列；一个 Object 只有一个等待队列。

## 线程池

线程池是为了复用线程处理任务，降低创建/销毁线程带来的消耗。

线程池 ThreadPoolExecutor 核心参数有：

- corePoolSize：核心线程池大小。核心的线程会随着任务的提交被逐渐创建，或使用 `prestartAllCoreThreads` 在开始时就创建核心线程数个线程。
- maximumPoolSize：线程池最大大小。阻塞队列满了后，开始继续增加线程直到线程池最大大小，达到后对后续的任务触发拒绝策略。
- keepAliveTime：线程池中超过 corePoolSize 数量的 worker，在超过该时间没有从阻塞队列中获取到任务后，结束线程。
- workerQueue：阻塞队列。
- rejectHandler：达到 maximumPoolSize 后的拒绝策略。
- threadFactory：创建线程的工厂，可以用来给线程命名编号。

整体工作流程为：

- 开始提交任务，如果没有达到 corePoolSize，则创建新的线程处理，如果已经达到 corePoolSize，则放入阻塞队列。阻塞队列满了后，继续创建线程处理，直到 maximumPoolSize，增长到 maximumPoolSize 之后执行拒绝策略。
- 每个 worker 执行完成初始任务后，使用 keepAliveTime 从阻塞队列中获取任务，如果获取到任务则执行，并继续循环超时获取；如果超时没有获取到任务，当前线程数大于 corePoolSize，则不断减少 worker 数量，直到 corePoolSize。

提交任务：

- 无需返回结果的任务：`execute(Runnable)`
- 需要返回结果的任务：`Future submit(Callable)`，会返回一个 FutureTask 实例。

常用的阻塞队列有：

- ArrayBlockingQueue：基于数组实现的有界阻塞队列。
- LinkedBlockingQueue：基于链表实现的阻塞队列，支持有界队列和无界队列，无参的构造器创建无界队列。吞吐量比 ArrayBlockingQueue 高。
- SynchronousQueue：不存储元素的阻塞队列，hand off，吞吐量最高。
- PriorityBlockingQueue：优先级的无界队列。

常用的拒绝策略有：

- AbortPolicy：抛出异常。
- CallerRunsPolicy：由提交任务的线程自己执行，即不再使用线程异步执行。
- DiscardOldestPolicy：丢弃最早的一个任务，执行当前任务。
- DiscardPolicy：不处理，丢掉当前任务。

线程池的关闭：

- shutdown：将线程池置为 SHUTDOWN 状态，阻止新任务的提交，中断没有执行任务的线程，已提交了的任务会继续执行。
- awaitTermination：等待线程池达到 TERMINATED 状态（所有任务执行完毕），或者超时后返回。常和 shutdown 搭配使用。
- shutdownNow：将线程池设为 STOP 状态，阻止新任务的提交，尝试停止所有正在执行的线程，返回等待执行的任务列表。由于是通过中断停止的，可能线程并不会响应中断，因此不代表线程池能立刻退出。

JDK 1.7 新增了 ForkJoinPool，原理为：

- 创建一个固定大小的 worker 线程池，每个 worker 有两个 WorkQueue 双端队列用来保存任务。
- 提交一个任务 ForkJoinTask 后，某 worker 会获取到该任务，它可以 fork 出来一些新的子 ForkJoinTask，放入自己的 WorkQueue 中。奇数的 WorkQueue 用来放外部提交的任务，偶数的 WorkQueue 用来放自己提交的任务。
- worker 对自己的任务是在队头进行操作，相当于 LIFO。另外当一个 worker 空闲时，它可以 steal 其他 worker 的队列任务，从队尾进行窃取，相当于 FIFO。取任务是用 CAS 实现的，取自己队列任务和偷取别人队列任务使用队列的不同端，能够减少并发冲突。

ForkJoinPool 比较适合计算密集型任务，worker 数量不需要太多，通常和 CPU 核数保持一致，思想是把大任务拆分为小任务并发执行，然后合并结果，通过 work stealing 机制避免 worker 空闲。

使用 Executors 工具类创建的常用的线程池：

- newSingleThreadExecutor：`core=1, max=1, queue=LinkedBlockingQueue`，只有一个线程来处理任务，可以保证任务有序执行。
- newFixedThreadPool：`core=n, max=n, queue=LinkedBlockingQueue`，线程池大小固定，多余的任务全部入队。
- newCachedThreadPool：`core=0, max=Integer.MAX, queue=SynchronousQueue`，没有核心线程池，来多少任务创建多少线程，空闲的线程慢慢终止。
- newScheduledThreadPool：指定核心线程数，使用无界队列 `DelayQueue`。
- newSingleThreadScheduledExecutor：核心线程数为1的 ScheduledThreadPool。
- newWorkStealingPool：JDK 1.7 后提供，使用 ForkJoinPool 实现，工作窃取队列，适用于计算密集型任务。

线程池大小估算：

- 计算密集型：`线程池大小 = CPU核心数 + 1`，多一个用来在发生缺页中断时，能够充分的利用 CPU。
- IO 密集型：`线程池大小 = CPU核心数 * 预期的CPU使用率 * (1 + W/C)`，其中 `W/C` 为 IO 等待时间和 CPU 计算时间的比。

## 并发编程优化

开发高并发程序时，可以做哪些优化？

- 使用合适参数的线程池。
- 减小锁的粒度，来降低冲突的可能性，提高并发能力。
- 使用不可变类来避免线程安全问题。
- 对于创建代价高，且线程不安全的类，使用 ThreadLocal 保存避免重复创建。
- 多用并发容器，少用同步容器。
- 多用更高层的同步工具，如 Lock/Condition、CountDownLatch、CyclicBarrier 等；少用最底层的同步工具，如 wait/notify。

## 参考文档

- 缓存一致性（Cache Coherency）入门：<https://www.infoq.cn/article/cache-coherency-primer>
