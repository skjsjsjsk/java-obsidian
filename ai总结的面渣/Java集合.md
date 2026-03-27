# 一、集合体系与整体认知
## 1. Java 里常见的集合类有哪些？你用过哪些？
### 15 秒简答
Java 集合主要分两大派系：Collection 单列集合和 Map 双列集合。Collection 下有 List（有序可重复，如 ArrayList、LinkedList）、Set（唯一，如 HashSet、TreeSet）、Queue（队列，如 LinkedList、ArrayDeque、PriorityQueue）。Map 下最常用 HashMap、LinkedHashMap、TreeMap，并发场景常用 ConcurrentHashMap。日常开发中 ArrayList 和 HashMap 用得最多。
### 3 分钟详答
Java 集合框架从 JDK 1.2 开始引入，核心是两棵继承树：  
- `Collection` 接口：单列集合，又分为：
  - `List`：有序、可重复，按索引访问。典型实现：
    - `ArrayList`：基于动态数组，查询快，增删慢。
    - `LinkedList`：基于双向链表，增删快，查询慢。
  - `Set`：元素唯一，不重复。典型实现：
    - `HashSet`：基于哈希表，无序。
    - `TreeSet`：基于红黑树，有序。
  - `Queue`/`Deque`：队列/双端队列，FIFO 或栈式使用。
- `Map` 接口：键值对映射，key 唯一。
  - `HashMap`：最常用的哈希表实现，非线程安全。
  - `LinkedHashMap`：保留插入顺序或访问顺序。
  - `TreeMap`：基于红黑树的有序 Map。
  - `ConcurrentHashMap`：高并发场景下的线程安全 Map。
牛客面经里，面试官经常问“你熟悉的集合类以及原理”，一般就是从这些类里挑几个追问底层结构和线程安全问题。
---
## 2. List、Set、Map 三者的区别是什么？
### 15 秒简答
List：有序、可重复，按索引访问。  
Set：无序、不可重复。  
Map：键值对映射，key 不可重复，value 可重复。
### 3 分钟详答
从三个维度区分：  
- 是否有序：  
  - List 有序，按插入顺序保存，可以通过下标直接访问。  
  - Set 一般无序（HashSet），TreeSet 按排序规则“有序”。  
  - Map 的 key 无序（HashMap），TreeMap 按排序有序。  
- 是否允许重复：  
  - List 允许重复。  
  - Set 不允许重复元素。  
  - Map 的 key 不允许重复，value 可以重复。  
- 典型实现：  
  - List：ArrayList、LinkedList。  
  - Set：HashSet、TreeSet。  
  - Map：HashMap、TreeMap、ConcurrentHashMap。
牛客有赞面经里，会先把 List/Set/Map 的实现类一股脑列出来，再让你逐个解释。
---
## 3. Java 集合框架中的主要接口有哪些？
### 15 秒简答
顶层是 `Collection` 和 `Map`。Collection 下有 `List`、`Set`、`Queue`/`Deque`；Map 就是 `Map` 接口。另外还有 `SortedSet`、`SortedMap`、`NavigableSet`/`NavigableMap` 等子接口。
### 3 分钟详答
核心接口层级大致是：  
- `Collection`：单列集合根接口。
  - `List`：有序列表。
  - `Set`：唯一元素集合。
  - `Queue`：队列；`Deque`：双端队列。
- `Map`：键值对根接口。
  - `SortedMap`：有序 Map，`TreeMap` 实现该接口。
  - `NavigableMap`：扩展了搜索方法，如 `lowerEntry()`、`floorEntry()` 等。
- 辅助接口：
  - `RandomAccess`：标记接口，表示支持随机访问（ArrayList 实现了此接口）。
  - `Cloneable`、`Serializable`：支持克隆与序列化。
---
## 4. 你是怎么选用集合的？
### 15 秒简答
需要键值对用 Map；要唯一性用 Set；需要有序可重复用 List。并发环境用并发集合（如 ConcurrentHashMap），需要排序用 TreeMap/TreeSet。
### 3 分钟详答
选型时主要考虑：  
- 存储结构：单列还是键值对？  
- 是否需要唯一性：需要唯一就用 Set 或 Map 的 key。  
- 是否需要顺序：需要插入顺序用 `LinkedHashMap`/`LinkedHashSet`；需要排序用 `TreeMap`/`TreeSet`。  
- 性能：频繁查询用 ArrayList；频繁增删用 LinkedList；大量查找用 HashSet/HashMap。  
- 线程安全：单线程用 HashMap/ArrayList；多线程用 ConcurrentHashMap、CopyOnWriteArrayList 等。
---
# 二、List 常见问题
## 5. ArrayList 的底层实现是什么？扩容机制是怎样的？
### 15 秒简答
ArrayList 底层是 `Object[] elementData` 动态数组。默认初始容量 10，当容量不足时，按 1.5 倍扩容（newCapacity = oldCapacity + oldCapacity >> 1），再通过 `Arrays.copyOf()` 复制数据。
### 3 分钟详答
关键点：  
- 底层结构：`transient Object[] elementData`，实际元素个数由 `size` 记录。  
- 构造：
  - 无参构造：初始化为空数组，第一次 add 时才真正分配默认容量 10。
  - 有参构造：直接创建指定长度的数组。  
- 扩容流程：
  1. `add()` 时先调用 `ensureCapacityInternal(size + 1)`。
  2. 如果需要的最小容量 > 当前数组长度，调用 `grow()`。
  3. `grow()` 中：`int newCapacity = oldCapacity + (oldCapacity >> 1)`，即约 1.5 倍。
  4. 若新容量仍不够，直接使用需要的最小容量。
  5. 最后用 `Arrays.copyOf(elementData, newCapacity)` 复制到新数组。
牛客面经中经常问“ArrayList 既然是数组，怎么动态扩容”，核心就是 grow + Arrays.copyOf。
---
## 6. ArrayList 和 LinkedList 的区别？各自的适用场景？
### 15 秒简答
ArrayList：基于动态数组，随机访问 O(1)，增删要移动元素，O(n)；内存占用少。  
LinkedList：基于双向链表，随机访问 O(n)，增删只需改指针，O(1)；内存占用多（存前后指针）。  
查询多、增删少用 ArrayList；频繁增删、尤其是头尾增删用 LinkedList。
### 3 分钟详答
从底层结构、时间复杂度、内存和使用场景对比：  
- 底层结构：
  - ArrayList：`Object[]` 数组，内存连续。
  - LinkedList：双向链表，节点存 `prev`、`next` 和 `item`。  
- 随机访问：
  - ArrayList：`get(index)` 直接用数组下标访问，O(1)。
  - LinkedList：需要从 head 或 tail 遍历，O(n)。  
- 插入/删除：
  - ArrayList：中间插入要移动后面所有元素，O(n)；尾部插入接近 O(1)。
  - LinkedList：在已知位置插入/删除，只需修改前后指针，O(1)。  
- 内存：
  - ArrayList：仅存对象引用和容量。
  - LinkedList：每个节点额外存两个引用，开销更大。  
- 适用场景：
  - 频查、少增删：ArrayList。
  - 频繁在中间或两端增删：LinkedList 或 ArrayDeque。
---
## 7. ArrayList 和 Vector 的区别？
### 15 秒简答
Vector 是线程安全的古老类，方法大多用 `synchronized` 修饰；ArrayList 非线程安全。扩容时 Vector 默认翻倍，ArrayList 扩容 1.5 倍。实际开发中多用 ArrayList，需要线程安全时推荐使用并发集合或 `Collections.synchronizedList`。
### 3 分钟详答
- 线程安全：Vector 是同步的，ArrayList 不是。  
- 性能：Vector 由于方法级同步，性能较差。  
- 扩容：
  - ArrayList：1.5 倍。
  - Vector：如果未指定容量增量，默认 2 倍。  
- 历史：Vector 属于早期集合，现在一般被认为是“遗留类”，在 JUC 包下有更好的替代方案。
---
## 8. 如何实现数组和 List 之间的转换？
### 15 秒简答
数组转 List：`Arrays.asList(array)`，注意返回的是固定大小的 List，不能增删。  
List 转数组：`list.toArray(new T[0])` 或 `list.toArray(new T[list.size()])`。
### 3 分钟详答
- 数组 → List：
  - `Arrays.asList(T... a)`：返回的是 `Arrays.ArrayList`，是 Arrays 的内部类，不是 `java.util.ArrayList`，不支持 `add`/`remove`，会抛 `UnsupportedOperationException`。
  - 如需可增删的 List：`new ArrayList<>(Arrays.asList(array))`。  
- List → 数组：
  - 无参 `toArray()` 返回 `Object[]`，容易丢失类型。
  - 推荐带泛型参数：`list.toArray(new String[0])` 或 `new String[list.size()]`，JDK 会根据传入数组长度自动选择复用或新建。
---
## 9. ArrayList 的遍历和 LinkedList 的遍历性能有什么区别？
### 15 秒简答
遍历性能 ArrayList 通常更好。ArrayList 内存连续，CPU 缓存友好；LinkedList 链式存储，遍历时要跳转指针，缓存命中率低，且每个节点对象更多，导致遍历更慢。
### 3 分钟详答
原因：  
- 内存连续性：ArrayList 底层数组在内存中是连续块，现代 CPU 会预读连续内存到缓存，遍历时缓存命中率高。  
- LinkedList：每个节点在堆中分散存储，遍历时需要不断根据 next 引用跳转，缓存命中率低。  
- 对象开销：LinkedList 每个节点是一个独立对象，有额外开销，遍历时访问的对象更多。  
因此，即使是“只遍历不修改”，大多数场景下 ArrayList 仍然更快。
---
# 三、Set 与 Queue 基础
## 10. HashSet 的底层实现是什么？如何保证元素不重复？
### 15 秒简答
HashSet 内部使用 `HashMap` 来保存元素，元素存为 key，value 是一个固定对象 `new Object()`。通过 key 的 `hashCode()` 和 `equals()` 判断是否重复。
### 3 分钟详答
核心点：  
- `HashSet<E>` 里有一个 `HashMap<E, Object> map`，还有一个 `Object PRESENT = new Object()` 作为虚拟值。  
- `add(e)` 实际上是 `map.put(e, PRESENT)`，如果 key 已存在，就会返回旧 value，从而判断是否重复。  
- 判断重复的流程：
  1. 先计算 key 的 `hashCode()`，得到哈希值。
  2. 在哈希表中找到对应桶。
  3. 桶内链表/红黑树中用 `equals()` 比较 key 是否存在。  
因此，存入 HashSet 的对象必须正确重写 `hashCode()` 和 `equals()`。
---
## 11. HashSet 和 TreeSet 的区别？
### 15 秒简答
HashSet：基于哈希表，无序，查询速度快。  
TreeSet：基于红黑树，元素有序（自然顺序或定制比较器），查询稍慢，但可以范围查找。
### 3 分钟详答
- 底层结构：
  - HashSet：内部是 HashMap。
  - TreeSet：内部是 TreeMap，元素作为 key。  
- 排序：
  - HashSet：无序。
  - TreeSet：按自然顺序或 `Comparator` 排序。  
- 性能：
  - HashSet 的 `add/remove/contains` 平均 O(1)。
  - TreeSet 的操作是 O(logn)。  
- 使用场景：
  - 只需去重、查找：HashSet。
  - 需要排序、范围查找：TreeSet。
---
## 12. Queue 中 poll() 和 remove() 有什么区别？
### 15 秒简答
当队列为空时：  
- `poll()` 返回 `null`。  
- `remove()` 抛出 `NoSuchElementException`。  
都是取出队首元素。
### 3 分钟详答
- `poll()`：队列非空时取出并返回队首；为空则返回 `null`。  
- `remove()`：队列非空时取出队首；为空则抛异常。  
类似地，`peek()` 与 `element()` 的区别也是：`peek()` 在空队列返回 `null`，`element()` 抛异常。
---
## 13. 你用过哪些队列？ArrayDeque 和 LinkedList 有什么区别？
### 15 秒简答
常用 `LinkedList`、`ArrayDeque` 作为队列或栈；还有 `PriorityQueue`（优先队列）。ArrayDeque 基于可扩容数组，LinkedList 基于双向链表。ArrayDeque 一般作栈/队列更快，LinkedList 更多作为 List 使用。
### 3 分钟详答
- `PriorityQueue`：基于二叉堆（数组），按优先级出队，而不是 FIFO。  
- `ArrayDeque`：
  - 基于可扩容数组，两头都可以插入删除，适合做栈或双端队列。
  - 通常比 LinkedList 更快，因为数组连续、缓存友好，且没有节点指针开销。  
- `LinkedList`：
  - 既实现了 List，又实现了 Deque，可以作为队列、栈、双端队列。
  - 增删快，但随机访问慢。  
日常开发中，如果只需要栈/双端队列，优先使用 ArrayDeque。
---
# 四、Map 高频问题
## 14. HashMap 的底层实现原理是什么？
### 15 秒简答
JDK 1.8 以后，HashMap 是数组 + 链表 + 红黑树。key 的哈希值决定数组下标，冲突时形成链表；当链表长度 ≥ 8 且数组长度 ≥ 64 时，链表转成红黑树。默认初始容量 16，负载因子 0.75，超过阈值就扩容为原来的 2 倍。
### 3 分钟详答
核心结构：  
- `Node<K,V>[] table`：哈希桶数组。
- 每个 `Node` 包含 `hash`、`key`、`value`、`next`（链表下一个节点）。  
关键流程：  
1. `put(key, value)`：
   - 计算 `hash(key)`：`(h = key.hashCode()) ^ (h >>> 16)`，扰动函数让高位参与运算，减少冲突。
   - 计算桶下标：`(n - 1) & hash`（n 是数组长度，必须是 2 的幂）。
   - 如果该桶为空，直接插入新节点。
   - 否则遍历链表/红黑树：
     - 如果 key 已存在，覆盖 value。
     - 否则插入链表尾部或红黑树中。
   - 插入后判断是否需要树化（链表长度 ≥ 8，数组长度 ≥ 64）。
   - 最后检查 `size > threshold`（容量 × 负载因子），触发扩容。  
2. 扩容机制：
   - 创建新数组，容量翻倍。
   - 将旧数组元素重新计算下标（JDK1.8 优化：要么原位置，要么原位置 + oldCap）。
   - 扩容期间可能触发链表 → 红黑树或反向退化。
牛客有赞、猫眼面经里，HashMap 原理是必问项。
---
## 15. HashMap 的扩容机制是怎样的？为什么容量是 2 的幂？
### 15 秒简答
当元素数量 > 容量 × 负载因子（默认 0.75）时扩容，容量翻倍。新下标计算时，用 `hash & (newCap - 1)`，容量必须是 2 的幂才能保证这种位运算等价于取模，且分布更均匀。
### 3 分钟详答
- 扩容触发条件：`size > threshold = capacity * loadFactor`。  
- 扩容步骤：
  1. 新建一个 2 倍大小的数组。
  2. 遍历旧数组每个桶，将链表/红黑树节点重新计算下标，迁移到新数组。
  3. JDK1.8 优化：迁移时保持链表顺序，避免逆序导致的死循环问题。  
- 容量为何是 2 的幂：
  - 计算下标：`index = (n - 1) & hash`。
  - 当 `n` 为 2 的幂时，`n - 1` 的二进制形式全是低位 1，`hash & (n - 1)` 等价于 `hash % n`，但位运算更快。
  - 同时能保证哈希值在数组范围内分布更均匀，减少冲突。
---
## 16. HashMap 在 JDK 1.7 和 1.8 中有什么区别？
### 15 秒简答
JDK1.7：数组 + 链表，头插法，多线程扩容时可能形成死循环。  
JDK1.8：数组 + 链表 + 红黑树，尾插法，优化了扩容和树化，解决死循环问题，但仍有线程安全问题。
### 3 分钟详答
- 数据结构：
  - 1.7：数组 + 链表。
  - 1.8：数组 + 链表 + 红黑树。  
- 插入方式：
  - 1.7：头插法，新节点插入链表头部，并发扩容时容易形成环形链表。
  - 1.8：尾插法，保持链表顺序，避免逆序。  
- 树化：
  - 1.8 当链表长度 ≥ 8 且数组长度 ≥ 64 时转为红黑树，查询从 O(n) 提升到 O(logn)。  
- 线程安全：
  - 两个版本都不是线程安全的，但 1.7 在并发扩容时更易死循环，1.8 改进了扩容流程。
猫眼面经中直接问“HashMap、ConcurrentHashMap1.7 和 1.8 的区别”。
---
## 17. HashMap 为什么线程不安全？会有什么问题？
### 15 秒简答
HashMap 的方法没有同步，多线程并发 put、扩容时，可能导致数据覆盖、丢失，JDK1.7 还可能形成环形链表导致死循环。多线程环境下应使用 ConcurrentHashMap。
### 3 分钟详答
典型问题：  
- JDK1.7：扩容时使用头插法，并发 rehash 时可能将链表顺序倒置，形成环形链表，导致 get 时死循环。  
- JDK1.8：改用尾插法，避免了环形链表，但并发 put 仍可能：
  - 多个线程同时判断同一位置为空，各自插入节点，导致后插入的节点覆盖前一个。
  - 计数不准确，size 计数非原子。  
- 解决方案：
  - 使用 `ConcurrentHashMap`（推荐）。
  - 使用 `Collections.synchronizedMap(new HashMap<>())`，但性能较差。
---
## 18. HashMap 和 Hashtable 有什么区别？
### 15 秒简答
Hashtable 是线程安全的古老类，方法大多用 `synchronized`，不支持 null 键/值。HashMap 非线程安全，支持 null 键/值。一般不推荐再使用 Hashtable。
### 3 分钟详答
- 线程安全：Hashtable 方法同步，HashMap 不是。  
- null 支持：
  - HashMap：允许一个 null 键和多个 null 值。
  - Hashtable：不允许 null 键/值，否则抛 NPE。  
- 继承体系：
  - Hashtable 继承 `Dictionary`，比较老。
  - HashMap 继承 `AbstractMap`。  
- 性能：Hashtable 全表锁，性能较差；HashMap 在单线程下更快。  
- 替代：如果需要线程安全的 Map，使用 ConcurrentHashMap。
---
## 19. HashMap 和 HashSet 有什么关系？
### 15 秒简答
HashSet 内部使用一个 HashMap 来存储元素，元素作为 key，value 是一个固定的 Object。HashSet 的很多方法都是委托给 HashMap 实现的。
### 3 分钟详答
- HashSet 里有一个 `HashMap<E, Object> map`，以及 `Object PRESENT`。  
- `add(e)` → `map.put(e, PRESENT)`。  
- `contains(e)` → `map.containsKey(e)`。  
- `remove(e)` → `map.remove(e)`。  
因此，HashSet 本质上是“只关心 key”的 HashMap 封装，去重逻辑完全依赖 HashMap 的 key 机制。
---
## 20. ConcurrentHashMap 如何实现线程安全？JDK 1.7 和 1.8 有什么区别？
### 15 秒简答
JDK 1.7：使用分段锁（Segment），每个 Segment 继承 ReentrantLock，锁粒度较大。  
JDK 1.8：弃用 Segment，直接使用 CAS + synchronized 锁单个桶，锁粒度更细，并发度更高，并引入红黑树优化查询。
### 3 分钟详答
- JDK 1.7：
  - 数据结构：`Segment[]` + `HashEntry[]`，每个 Segment 相当于一个小 HashMap。
  - 写操作锁住整个 Segment，读操作大多无锁（依赖 volatile）。
  - 默认并发度 16，最多 16 个线程同时写不同的 Segment。  
- JDK 1.8：
  - 数据结构：Node 数组 + 链表/红黑树，与 HashMap 类似。
  - 并发控制：
    - 读操作无锁，直接访问 volatile 的节点。
    - 写操作：先用 CAS 尝试更新，失败再用 `synchronized` 锁住当前桶的头节点。
  - 优点：锁粒度更小，并发度更高；同时继承 HashMap 的树化优化。  
猫眼面经中直接问“HashMap、ConcurrentHashMap 1.7 和 1.8 的区别”。
---
## 21. ConcurrentHashMap 和 Hashtable 有什么区别？
### 15 秒简答
ConcurrentHashMap 使用分段锁或 CAS + synchronized，锁粒度小，并发度高；Hashtable 对所有方法加 `synchronized`，锁粒度大，性能差。因此并发环境下推荐使用 ConcurrentHashMap。
### 3 分钟详答
- 锁粒度：
  - Hashtable：整个表锁，一次只能一个线程操作。
  - ConcurrentHashMap：JDK1.7 分段锁，JDK1.8 锁单个桶。  
- 性能：在高并发下 ConcurrentHashMap 明显优于 Hashtable。  
- 迭代器：
  - Hashtable 的迭代器是 fail-fast。
  - ConcurrentHashMap 的迭代器是弱一致性，允许在迭代时并发修改，不会抛 `ConcurrentModificationException`。
---
## 22. LinkedHashMap 的底层实现和应用场景？
### 15 秒简答
LinkedHashMap 继承 HashMap，在 Node 基础上增加 `before`/`after` 双向链表，维护插入顺序或访问顺序。常用于实现 LRU 缓存。
### 3 分钟详答
- 结构：
  - 继承 HashMap，有 `head`、`tail` 两个指针。
  - 每个 Entry 新增 `before`、`after` 形成双向链表。  
- 两种顺序：
  - 插入顺序（默认）。
  - 访问顺序（构造函数中 `accessOrder=true`），每次 `get`/`put` 会把节点移到链表尾部。  
- 应用：
  - 简单 LRU 缓存：结合 `removeEldestEntry(Map.Entry eldest)`，在插入后判断是否删除最久未使用的节点。  
 LinkedHashMap 在“需要保持顺序的缓存”场景很常见。
---
## 23. TreeMap 的底层结构是什么？什么时候用 TreeMap？
### 15 秒简答
TreeMap 基于红黑树，键按自然顺序或 `Comparator` 排序。适合需要按 key 顺序遍历或范围查找的场景，如排行榜、区间查询。
### 3 分钟详答
- 底层结构：红黑树，保证大致平衡，操作复杂度 O(logn)。  
- 排序方式：
  - 键实现 `Comparable` 接口（自然顺序）。
  - 或在构造 TreeMap 时传入 `Comparator`。  
- 典型场景：
  - 需要按 key 从小到大遍历。
  - 需要查找某个范围的 key：`subMap()`、`tailMap()` 等。  
- 与 HashMap 相比：
  - HashMap 查询/插入平均 O(1)，无序。
  - TreeMap 查询/插入 O(logn)，有序，适合“顺序敏感”的场景。
---
## 24. Map 的 key 应该如何设计？需要注意什么？
### 15 秒简答
Map 的 key 最好是不可变对象，并且正确重写 `hashCode()` 和 `equals()`，保证插入后哈希值不变，否则无法正常查找。
### 3 分钟详答
关键点：  
- 不可变：如果 key 的哈希值在放入 Map 后被改变，就再也无法正确定位到原来的桶，导致内存泄漏。  
- hashCode/equals 协定：
  - 如果两个对象 `equals` 返回 true，`hashCode` 必须相同。
  - 重写 `equals` 必须重写 `hashCode`，否则 HashMap/HashSet 会表现异常。  
- 典型不可变 key：String、Integer 等包装类。  
- 如果用可变对象作 key：
  - 必须保证影响 `hashCode()` 的字段在放入 Map 后不再改变。
  - 或使用不依赖对象字段的哈希策略（不推荐）。
---
# 五、迭代器与并发集合
## 25. Iterator 是什么？怎么用？
### 15 秒简答
Iterator 是集合的迭代器接口，用于遍历集合，支持在遍历时安全删除元素。常用方式：`while (iterator.hasNext()) { E e = iterator.next(); ... iterator.remove(); }`。
### 3 分钟详答
- 作用：提供统一遍历 Collection 的方式，不依赖具体集合实现。  
- 核心方法：
  - `boolean hasNext()`：是否还有下一个元素。
  - `E next()`：返回下一个元素，没有则抛 `NoSuchElementException`。
  - `default void remove()`：删除上次 `next()` 返回的元素，只能调用一次。  
- 与 for-each 的关系：for-each 底层依赖 Iterator，但在遍历时不能安全修改集合，否则可能抛 `ConcurrentModificationException`。
---
## 26. Iterator 和 ListIterator 有什么区别？
### 15 秒简答
Iterator 可以遍历 List、Set、Queue，只能向前遍历，支持 `remove()`。  
ListIterator 只能用于 List，可以双向遍历，支持 `add()`、`set()` 等更多操作。
### 3 分钟详答
- Iterator：
  - 通用迭代器，适用于所有 Collection。
  - 只能向后遍历：`hasNext()`/`next()`。
  - 只支持 `remove()`。  
- ListIterator：
  - 继承 Iterator，仅 List 通过 `list.listIterator()` 获得。
  - 支持向前遍历：`hasPrevious()`/`previous()`。
  - 支持 `add(E e)`：在当前位置插入元素。
  - 支持 `set(E e)`：替换最后一次 `next()`/`previous()` 返回的元素。  
ListIterator 功能更强大，但仅限于 List。
---
## 27. 什么是 fail-fast 迭代器？HashMap 的迭代器是 fail-fast 吗？
### 15 秒简答
fail-fast 迭代器在迭代过程中，如果发现集合结构被其他线程或本线程修改（除迭代器自身的 `remove()` 外），会立即抛出 `ConcurrentModificationException`。HashMap 的迭代器是 fail-fast 的。
### 3 分钟详答
- 实现原理：
  - 迭代器里记录一个 `expectedModCount`，创建时等于集合的 `modCount`。
  - 每次迭代 `next()` 时，检查两者是否相等，不等则抛异常。  
- HashMap：
  - 迭代器是 fail-fast，如果在迭代时直接调用 `map.put()`/`remove()`，会修改 `modCount`，导致并发修改异常。
  - 只能通过迭代器自身的 `remove()` 删除元素。  
- 注意：fail-fast 行为不能完全保证，在不同 JVM 实现中可能不同，只能作为调试辅助手段。
---
## 28. Java 中有哪些线程安全的集合？
### 15 秒简答
常见线程安全集合：`Vector`、`Hashtable`、`Collections.synchronizedXxx` 包装类，以及 JUC 包下的 `ConcurrentHashMap`、`CopyOnWriteArrayList`、`CopyOnWriteArraySet`、`ConcurrentLinkedQueue` 等。
### 3 分钟详答
- 传统同步集合：
  - `Vector`、`Hashtable`：方法级 `synchronized`，性能较差。
  - `Collections.synchronizedList/Set/Map()`：包装普通集合，对所有方法加同步块。  
- JUC 并发集合：
  - `ConcurrentHashMap`：高并发读写，性能好。
  - `CopyOnWriteArrayList`/`CopyOnWriteArraySet`：写时复制，适合读多写少。
  - `ConcurrentLinkedQueue`/`ConcurrentLinkedDeque`：非阻塞并发队列。
  - `ArrayBlockingQueue`/`LinkedBlockingQueue`：阻塞队列，适合生产者-消费者模型。
  - `ConcurrentSkipListSet`/`ConcurrentSkipListMap`：基于跳表的并发有序集合。
牛客面经中常问“你刚才说的都是不安全的集合，那有哪些是线程安全的”。
---
## 29. CopyOnWriteArrayList 的原理和适用场景？
### 15 秒简答
CopyOnWriteArrayList 在写操作时会复制整个底层数组，读操作无锁。适合读多写少、遍历远多于修改的场景，如监听器列表、配置列表。
### 3 分钟详答
- 原理：
  - 内部维护一个 `volatile` 的数组引用。
  - 写操作：`add()`/`remove()` 时，先复制一个新数组，修改后再将引用指向新数组。
  - 读操作：直接读取原数组，不需要加锁。  
- 特点：
  - 线程安全，且读操作完全无锁，性能很高。
  - 写操作开销大，需要复制数组。  
- 适用场景：
  - 读多写少，如系统配置、监听器列表、事件监听队列。
  - 需要避免 `ConcurrentModificationException` 的场景。  
- 注意：数据一致性是弱一致性的，迭代时可能看到旧数据。
---
## 30. ConcurrentHashMap 的 size() 方法是如何实现的？
### 15 秒简答
ConcurrentHashMap 没有全局计数器，而是通过分散的计数单元（`CounterCell` 数组 + `baseCount`）来近似统计 size，避免全局竞争。
### 3 分钟详答
- JDK 1.8 实现：
  - 使用 `baseCount` 作为基础计数。
  - 当更新竞争激烈时，使用 `CounterCell[]` 数组，每个线程在自己的单元上计数。
  - `size()` 通过 `sumCount()` 将所有 CounterCell 值与 baseCount 累加。  
- 特点：
  - 不追求强一致，只保证最终一致性。
  - 性能优于全局锁计数，适合高并发场景。  
- 如果需要精确计数，可以在业务层使用 `AtomicLong` 或外部计数器。
---
