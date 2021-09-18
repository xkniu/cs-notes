# JUC

## AQS (AbstractQueuedSynchronizer)

AQS 核心的设计思想是，如果请求获取共享资源成功，则设为当前工作在线程；如果获取资源失败，则放到一个同步队列中，等待被唤醒。AQS 主要通过 `volatile int state`（代表共享资源）和一个 CLH 队列（FIFO 队列，代表阻塞的线程列表）来实现。

state 指的是要使用的资源，通过方法 `getState/setState/compareAndSetState` 来操作资源。

子类根据使用锁是独占还是共享，需要重写以下方法：

- 独占方式：tryAcquire/tryRelease/isHeldExclusively，表示获取资源/释放资源/当前是否是独占线程。
- 共享方式：tryAcquireShared/tryReleaseShared，表示获取共享资源/释放共享资源。

子类实现通常会调用 AQS 中的 `acquire/acquireShared` 方法来获取资源，这些方法会调用 `tryAcquire/tryAcquireShared` 来判断是否能够获取资源，当获取资源失败时，会**构造一个 Node 节点（独占或者共享节点），通过 CAS 加入到 CLH 队列的尾部**，然后该 Node 节点会进入自旋的状态，自旋中的判断条件为：

- 在同步队列中前驱节点是否是头节点 head。
- 获取资源 `tryAcquire/tryAcquireShared` 能否成功。

当两个条件都成功时，把自己设为头结点并执行；否则继续 `park`，等待前置节点的唤醒。

当节点执行完成时，它会唤醒队列中的下一个节点。如果唤醒的节点是共享节点，且它 `tryAcquireShared` 时返回还有剩余的资源，则它会继续向下传播唤醒后续共享节点。

AQS 还支持多个条件队列，用来支持条件等待和唤醒。AQS 中有一个 ConditionObject 的内部类，可以创建多个该内部类实例。ConditionObject 中维护了一个条件队列。

调用 ConditionObject 的 await 会 **构造一个 Node 节点放到条件队列的尾部**，调用 notify 时将条件队列的头节点移除加入到 AQS 同步队列的尾部。

大部分同步工具可以通过一个匿名静态内部类继承 AQS，然后各方法依赖这个匿名类来实现，来避免必须继承 AQS。

## ReentrantLock

基于 AQS 实现，主要分为公平锁和非公平锁。两种的实现主要区别：

- 公平锁：加锁直接基于调用 `acquire` 实现。
- 非公平锁：先直接尝试 CAS 获取 state 资源，成功则直接执行；失败则调用 `acquire`。

## Semaphore

Semaphore 通常用来控制有限资源的访问。基于 AQS 的共享模式实现。

核心方法：

- `new Semaphore(int permits, boolean fair)`：创建信号量，是否支持公平性。
- `void acquire(int permits)`：申请给定数量的许可。
- `void release(int permits)`：释放给定数量的许可。

## CountDownLatch

CountDownLatch 用来实现一个或多个（通常是一个）等待多个线程执行完成后再执行。基于 AQS 的共享模式实现。

核心方法：

- `new CountDownLatch(int count)`：初始化时执行 Latch 的数量。
- `void await()`：等待 Latch 的计数为0.
- `boolean await(long timeout, TimeUnit unit)`：等待 Latch 计数为0，或者超时。
- `void countDown()`：Latch 的计数减1.

通常生成一个 CountDownLatch 实例传给 N 个任务线程，然后主线程 `await` 所有任务线程的完成。每个任务线程完成后进行 `countDown`，都完成后主线程从等待中返回。

实现原理：

- 基于 AQS 实现。await 调用 `acquireShared`。`tryAcquireShared` 实现为计数为0才能获取资源成功，否则进入同步队列。
- countDown 调用 `releaseShared`，其中 `tryReleaseShared` 实现为计数为0时才返回 true，开始唤醒同步队列中的节点。

## CyclicBarrier

CyclicBarrier 支持一组线程互相等待，直到达到公共屏障点。基于 Lock 和 Condition 实现。

核心方法：

- `new CyclicBarrier(int parties, Runnable barrierAction)`：创建屏障，指定屏障拦截线程数量和屏障触发时执行的单次动作。
- `int await()`：等待屏障触发，并返回到达屏障的序号。

CountDownLatch 和 CyclicBarrier 的主要区别：

- CountDownLatch 只能单次使用；CyclicBarrier 可以重置后重新使用，比较适合处理更复杂的业务场景，如出错后重试。
- CountDownLatch 主要用于1等N的场景；CyclicBarrier 主要用于N个互相等待的场景。

## BlockingQueue

阻塞队列接口，提供可阻塞的入队/出队等队列操作。大部分实现都是基于 Lock 和 Condition 实现，通常创建两个 Condition 实例。如创建 notEmpty 和 notFull 分别代表队列非空和队列不满，当入队失败时，等待在 notFull 队列上，当任务出队时，通知 notFull 队列上的节点。

## AtomicLong

AtomicLong 提供高并发下的原子技术操作。基于 Unsafe 的循环 CAS 来实现。

## LongAdder

LongAdder 提供比 AtomicLong 更好的并发计数性能，基于分段锁的思想，分段为多个 Cell，每个 Cell 维护一个 long 值，通过 CAS 来计数。当冲突严重时，扩容 Cell 数组的大小。

## ThreadLocal

线程封闭原理，每个线程内部的变量副本，读写的时候不需要锁，效率很高。

Thread 类本身有一个 ThreadLocalMap 成员变量，存储的 Entry 为：ThreadLocal 为 key（使用 WeakReference 引用），存储的对象副本为 value。当发生 hash 冲突时，通过开发定址法来解决。

ThreadLocal 的 get 方法，是在当前线程的 ThreadLocalMap 中以 ThreadLocal 为 key 来获取本线程中存储的对象副本。

优点：

- 每个 map 是 Thread 私有的，操作时无需加锁，效率很高，用线程封闭的原理来解决并发问题。
- map 在 Thread 对象中，线程终止时自然会被清除。

使用时的注意事项：

- ThreadLocal 中的 Entry 继承了 `WeakReference<ThreadLocal>`，所以 ThreadLocal 是一个弱引用，但是值不是弱引用；当 ThreadLocal 被引用回收后，对应的 value 在下一次 ThreadLocalMap 调用 `set`、`get`、`remove` 时会被清除。但如果之后这些方法没被调用过，还是可能内存泄露。所以最佳实践是当不再使用 ThreadLocal 时，主动调用 `remove` 方法。
- 项目中使用了线程池，则每次应自己清理 ThreadLocal。否则之后线程可能拿到之前线程存放的脏数据。

使用场景：

- 存储方便全局使用的信息，比如用户信息。
- 存储构造代价高，又线程不安全的对象，例如 `SimpleDateFormat`。

## ConcurrentHashMap

### JDK 7 ConcurrentHashMap

并发的 HashMap，使用**锁分段**的思想，减少锁的粒度来降低并发冲突。

核心原理：Segment 数组 + HashEntry 数组 + volatile 读写（Unsafe 数组读写）

整体结构是一个 Segment 数组，每个 Segment 相当于一个哈希表，里面有一个 HashEntry 的数组。HashEntry 节点中的属性都是 volatile 和 final 的来保证可见性（key 是 final，value 和 next 都是 volatile）。

Segment 继承自 ReentrantLock，因此每个 Segment 就是一把锁。Segment 数组的大小是创建时决定的，初始化后不会发生改变，代表了 map 的最高并发度。Segment 中是独立的，随着不断的写操作，每个 Segment 中的 HashEntry 数组会独立扩容，互不相关。

put 操作时，先通过 hash 的高位定位到对应的 Segment，然后获取**对应 Segment 的锁**，然后再通过 hash 的低位定位到 Segment 中对应的 HashEntry 进行操作。获取不到锁的时候，优先自旋等待，超过自旋次数则阻塞等待。

get 不需要加锁，是通过 volatile 的 happens-before 语义保证的，写使用 volatile 写（数组元素使用 `Unsafe.putOrderedObject`），读使用 volatile 读（数组元素使用 `unsafe.getObjectVolatile`）来保证写操作对读的可见性，从而避免加锁。

get 的时候只通过 volatile 来保证可见性，那么在写操作时就必须维护原结构的正常。比如在 put 和 remove 时，对链的影响的操作一定只是一个原子的 volatile 写（先构造好节点的其他信息）；在 rehash 时，要先构造好 rehash 后的 HashEntry 数组，再原子替换，不过 rehash 可以优化为尽量复用之前可以复用的节点（如原链条可以直接搬过来的部分）。

size 操作，连续统计两次每个 Segment 的 size 和 modCount，如果这两次的 modCount 一致（统计过程中 map 没有发生变化），则返回汇总的 size 值。如果连续3次统计都发生了变化，则对所有 Segment 加锁后统计。

ConcurrentHashMap 为什么是弱一致性的？

- get 是弱一致的。因为读不加锁，那么读就可以与写操作并行，读的可见性是通过 volatile 读实现的，那么写操作中 volatile 写后对读接口就已经可见了，这时写方法还没执行完。
- 迭代器是弱一致性的。迭代过程中容器会发生的改变，对于遍历过了对应的 Segment 和 HashEntry，则不会再反映出来。
- clear 操作是弱一致性。clear 操作，每个 Segment 分别进行 clear，没有全局加锁。

ConcurrentHashMap 的弱一致性主要是为了提高效率，在性能和一致性之间平衡。

### JDK 8 ConcurrentHashMap

JDK 8 中不再使用 Segment 分段锁机制，选择了与 HashMap 类型的数组+链表/红黑树的方式来实现，并发控制则通过 synchronized+CAS 来实现。

ConcurrentHashMap 采用 Node 作为基础的存储单元，其中主要的子类型有：

- Node：普通节点类型，表示链表的头结点。
- TreeBin：红黑树的代理节点，用来放在桶内作为红黑树的头结点。
- TreeNode：红黑树结构中，红黑树的节点（链表长度超过8时转换为红黑树）。
- ForwardingNode：扩容节点，在扩容阶段，表示该桶内的数据正在复制。

put 方法，首先根据 hash（扰动函数：hash 高16位与低16位异或）找到对应的索引位置，然后通过对数组的 volatile 读（`Unsafe.getObjectVolatile`）来获取元素，之后开始自旋的进行操作：

- 如果节点是 null，CAS 插入当前元素。
- 如果节点是 ForwardingNode 节点，说明其他线程正在扩容，则帮助一起尽快完成 map 扩容。
- 如果节点是其他类型节点，在节点上加 synchronized 锁后，找到对应的位置插入节点，为了保证都对同一个元素加锁，所以对于链表，采用尾插法；对于红黑树，使用一个代理节点。

put 完成后会 CAS 统计节点数量，如果超过阀值（容量*0.75）则会进行扩容，扩容的时候会新建一个长度为原来两倍数组的 nextTable，然后一个一个节点的渐进式转移，转移的时候会加锁并复制原来的节点，然后完成一个节点的转移后，使用一个 ForwardingNode 替换原节点。扩容完成后用 nextTable 替换 table。

get 方法，首先根据 hash 找到对应的索引位置，然后对数组进行 volatile 读获取节点元素，之后根据节点来具体操作：

- 头节点就是需要的节点，则返回值。
- 头结点是 ForwardingNode，则该桶正在 rehash，调用 `ForwardingNode.find` 在新的 nextTable 里找到对应元素。
- 头结点是 TreeBin，则调用 `TreeBin.find` 找到对应的元素，由于红黑树可能正在旋转调整，需要加读写锁。
- 头结点是链表节点，则顺序遍历链表找对应的键。

使用无锁算法实现 HashMap get 的关键是要维持遍历结构的有效性，对于链表可以做到修改的时候不影响原链条（已经遍历到的某个节点位置）的遍历；对于红黑树做不到的，就使用读写锁来避免读写并发。扩容的过程，渐进式的构造新的 table，过程中要根据节点类型，可能需要读两个 table。

size 方法，没有在每个 HashEntry 中保存 size，而是和 HashEntry 独立开，默认对全局的 baseCount 进行 CAS 来记录大小，当发送冲突时，使用类似于 LongAdder 的思想，创建一个数组（最大扩容到机器核心数）counterCells 来进行分段，一个线程 CAS 对应分段的 counterCell 中的 volatile count 即可。

相比于 JDK 7 的 Segment 分段锁实现，JDK 8 进一步降低了锁的粒度（HashEntry 的首节点）。使用 synchronized 替换了 Lock，synchronized 随着 Java 的不断优化，效率已经比较高了，自带了轻量级锁自旋能力，不需要自己使用锁来控制自旋还是阻塞。
