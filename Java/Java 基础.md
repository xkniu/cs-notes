# Java 基础

常用集合接口：

- `Collection <- Set <- SortedSet <- NavigableSet`
- `Collection <- List`
- `Collection <- Queue <- Deque`
- `Map <- SortedMap <- NavagableMap`

集合装饰：

- `Collections.unmodifiableXxx`: 返回容器的一个不可变视图，使用 decorator 设计模式。
- `Collections.synchronizedXxx`：将容器包装成一个同步的容器，使用 decorator 设计模式，默认使用 this 作为同步对象，可以自己传入用于同步控制的对象。

## List 常用实现

### ArrayList

使用预分配数组实现，空间不够时扩容到 1.5 倍。适合尾部插入删除和随机访问。

时间复杂度：

- 尾部插入 `O(1)`；其他位置插入需要移动元素，平均 `O(n)`
- 实现了 RandomAccess 接口，通过 index 访问 `O(1)`

### LinkedList

双向链表实现，实现了 `Deque` 接口，可以做双端队列使用，适合头尾插入删除。

时间复杂度：

- 根据 index 访问复杂度 `O(n)`，因此随机插入/删除复杂度 `O(n)`。
- 头尾插入复杂度 `O(1)`。

## Map 常用实现

### HashMap

哈希表实现，当出现 hash 冲突时，使用链表解决冲突。在 JDK 1.8 中，当链表过长时，优化为红黑树。

- capacity：容量，即 hash 桶的数量，初始为 16。
- loadFactory：加载因子，默认为 0.75。
- threshold：扩容阀值，`threshold = capacity * loadFactory`，当元素数量超过扩容阀值时，容量扩大到原来的2倍。

优化细节：

- 容量为2的幂次，对容量取余其实就是取二进制的低位。因此不会直接使用 key 的 hashCode 来直接分配，而是将 hashCode 的高16位和低16位异或（扰动函数）后的结果来取余。
- rehash 的时候桶数量增加一倍，相当于取余的二进制位多判断了一位。因此原来某个桶中的元素会重新分散到两个桶中。

JDK 1.8 优化：

- 当桶的链表过长时，会优化为红黑树。
    - `TREEIFY_THRESHOLD`：树化阀值，默认为8，链表长度超过8时转换为红黑树。
    - `UNTREEIFY_THRESHOLD`：链化阀值，默认值6，rehash 时桶内数据元素小于6，转换为链表。
    - `MIN_TREEIFY_CAPACITY`：最小树化容量，默认为64，在容量小于这之前，优先扩容，而不是进行树化。
- JDK 1.7 及以前 rehash 时使用头插法，会让同一个桶中的数据逆序，在并发时可能形成环导致死循环；1.8 后改成尾插法，rehash 后桶中的元素顺序不变，不会出现死循环。当然 HashMap 都不支持并发，不应该在并发中使用。

### LinkedHashMap

在 HashMap 的基础上，又维护了一个双向链表用来记录数据插入（或访问）的顺序。

核心参数：

- `accessOrder`：默认为 false，表示链表使用插入顺序；为 true 时，使用访问顺序。

通过使用访问顺序和删除旧的元素，能够很容易的实现一个 LRU Cache，代码如下：

```java
public class LRUCache<K, V> extends LinkedHashMap<K, V> {
    
    private int cacheSize;

    public LRUCache(int cacheSize) {
        super(16, 0.75f, true);
        this.cacheSize = cacheSize;
    }
    
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        return size() >= cacheSize;
    }
}
```

### TreeMap

有序的 Map，基于 Key 排序，使用红黑树实现。实现了 `NavigableMap` 接口。

时间复杂度：

- 查找时间复杂度为 `O(log n)`
- 插入删除的时间复杂度为 `O(log n)`

常用场景：

- 需要排序统计之类的功能。比如实现一致性哈希，查找大于某个值的下一个节点。

### EnumMap

底层使用数组实现，使用 Enum 的 ordinal 作为数组的 index 来存取。

## Set 常用实现

### HashSet & LinkedHashSet & TreeSet

通过对应的 Map 来实现的，使用占位对象作为 value。

### EnumSet

思想是使用 bitSet 来节约存储空间。当 Enum 的值个数小于等于 64 个时，使用 long 来存储；当大于 64 个时，使用 long 数组来存储。
