# 02 - Java 集合框架源码级面试题（深度精讲版）

> **本章涵盖**：集合全景体系、List/Queue/Set 源码与陷阱、HashMap（JDK 1.7 vs 1.8 全链路源码剖析）、LinkedHashMap 与手写 LRU、TreeMap/WeakHashMap、ConcurrentHashMap 演进与源码、ThreadLocal 全家桶与内存泄漏、JUC 阻塞队列（40 道高频源码级面试题）。

---

## 目录

- [1. 集合全景体系与 List / Set / Queue（Q1 ~ Q11）](#1-集合全景体系与-list--set--queueq1--q11)
- [2. HashMap 源码级深度剖析（Q12 ~ Q21）](#2-hashmap-源码级深度剖析q12--q21)
- [3. LinkedHashMap、TreeMap、WeakHashMap 与同步 Map（Q22 ~ Q27）](#3-linkedhashmaptreemapweakhashmap-与同步-mapq22--q27)
- [4. ConcurrentHashMap 源码级深度解析（Q28 ~ Q32）](#4-concurrenthashmap-源码级深度解析q28--q32)
- [5. ThreadLocal 全家桶与内存泄漏（Q33 ~ Q36）](#5-threadlocal-全家桶与内存泄漏q33--q36)
- [6. JUC 阻塞队列（BlockingQueue）深度解析（Q37 ~ Q40）](#6-juc-阻塞队列blockingqueue深度解析q37--q40)

---

## 1. 集合全景体系与 List / Set / Queue（Q1 ~ Q11）

### Q1：Java 集合框架的整体继承体系是怎样的？日常如何选型？

- **核心结论**：Java 集合框架（`java.util`）分为两大顶层体系——**`Collection`（单列集合）** 与 **`Map`（双列键值对集合）**，两者完全独立（`Map` 不继承 `Collection`）。
- **体系结构全景图**：

```text
Collection (接口)
 ├── List (有序、可重复)
 │    ├── ArrayList       (动态数组，随机访问 O(1))
 │    ├── LinkedList      (双向链表，同时实现 Deque 接口)
 │    └── Vector / Stack  (古老线程安全动态数组，已淘汰)
 ├── Set (无序/有序、不可重复)
 │    ├── HashSet         (基于 HashMap)
 │    ├── LinkedHashSet   (基于 LinkedHashMap，维护插入顺序)
 │    └── TreeSet         (基于 TreeMap 红黑树，按规则排序)
 └── Queue / Deque (队列 / 双端队列)
      ├── PriorityQueue   (基于二叉小顶堆，优先级出队)
      ├── ArrayDeque      (基于循环数组，栈/队列首选)
      └── BlockingQueue   (JUC 阻塞队列：Array/Linked/Synchronous/Delay)

Map (接口，Key 唯一)
 ├── HashMap              (数组 + 链表 + 红黑树，线程不安全)
 ├── LinkedHashMap        (HashMap + 双向链表，有序/LRU)
 ├── TreeMap              (红黑树，Key 有序)
 ├── WeakHashMap          (弱引用 Key，GC 自动回收条目)
 ├── Hashtable            (全表 synchronized，已淘汰)
 └── ConcurrentHashMap    (JUC 高并发 Map：CAS + synchronized)
```

- **选型口诀**：随机读多用 `ArrayList`，去重无序用 `HashSet`，去重保序用 `LinkedHashSet`，去重排序用 `TreeSet`；普通键值对用 `HashMap`，并发场景用 `ConcurrentHashMap`，队列/栈首选 `ArrayDeque`（非阻塞）或 `LinkedBlockingQueue`（阻塞）。

---

### Q2：`ArrayList`、`LinkedList` 和 `Vector` 有什么区别？为什么很少用 `LinkedList`？

- **核心对比**：

| 对比维度 | `ArrayList` | `LinkedList` | `Vector` |
| :--- | :--- | :--- | :--- |
| **底层数据结构** | 动态数组（`Object[]`） | 双向链表（`Node<E>`） | 动态数组（`Object[]`） |
| **随机访问** | 支持 `RandomAccess`，**$O(1)$** | 不支持，需折半遍历，**$O(n)$** | 支持 `RandomAccess`，**$O(1)$** |
| **头尾插删** | 尾插均摊 $O(1)$，头插需移动数组 $O(n)$ | **头尾插删 $O(1)$** | 同 `ArrayList` |
| **中间插删** | 拷贝移动数组（`System.arraycopy` 连续内存拷贝极快） | **先 $O(n)$ 寻址**，再 $O(1)$ 改指针 | 同 `ArrayList` |
| **扩容倍数** | **1.5 倍** | 无需扩容，按节点动态分配 | **2 倍**（可指定 `capacityIncrement`） |
| **内存与缓存** | 内存连续，CPU 缓存行（Cache Line）局部性极佳 | 节点含前驱/后继双指针，内存开销大且不连续 | 同 `ArrayList` |
| **线程安全** | 不安全 | 不安全 | 方法加 **`synchronized`**（性能差，已淘汰） |

- **为什么实际开发几乎不用 `LinkedList`？（包括作者 Josh Bloch 都后悔写过它）**
  1. **随机访问极慢**，即使插入删除也要先从头/尾遍历定位节点（$O(n)$）。
  2. **内存占用巨大**：每个元素都要创建一个 `Node` 对象（对象头 12B + `item` 4B + `next` 4B + `prev` 4B = 24 字节额外包装）。
  3. **CPU 缓存不友好**：链表节点散落在堆内存各处，无法享受 CPU L1/L2 缓存行预取加速，数据量上万时遍历速度远不如 `ArrayList`；而在中部插入场景，`ArrayList` 的 `System.arraycopy` 是底层 C/汇编级 `memmove` 块拷贝，实际耗时往往比 `LinkedList` 寻址还快！
- **实战避坑**：遍历 `List` 时可以通过 `list instanceof RandomAccess` 选择 `for (int i = 0; ...)` 还是迭代器（`LinkedList` 用普通下标 `for` 循环遍历会导致 **$O(n^2)$ 级灾难性耗时**！）。

---

### Q3：`ArrayList` 源码深度剖析：懒加载与 `grow()` 扩容机制是怎样的？

- **构造与懒加载（JDK 8+）**：
  - `new ArrayList<>()`：底层赋值为空数组 `DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {}`（**初始容量为 0，懒加载**，节省空集合内存）。
  - **第一次调用 `add()`**：发现是默认空数组时，直接分配容量为 **10**（`DEFAULT_CAPACITY`）。
  - 注意：`new ArrayList<>(0)` 赋值为 `EMPTY_ELEMENTDATA = {}`，第一次 `add()` 时直接按最小所需容量 1 扩容，后续从 1 扩到 2、3、4……（而非直接跳到 10）。
- **`grow(int minCapacity)` 核心源码逻辑**：
  1. `int oldCapacity = elementData.length;`
  2. `int newCapacity = oldCapacity + (oldCapacity >> 1);` —— **扩容为旧容量的 1.5 倍**。
  3. 如果 `newCapacity - minCapacity < 0`（如 `addAll` 一次塞入大量元素，1.5 倍仍不够），则直接取 **`newCapacity = minCapacity`**。
  4. 如果 `newCapacity - MAX_ARRAY_SIZE > 0`（`MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8`，减 8 是因为部分 JVM 会在数组对象头额外保留存储长度的元数据），调用 `hugeCapacity()` 最高扩至 `Integer.MAX_VALUE`。
  5. 最后调用 `Arrays.copyOf(elementData, newCapacity)`（底层再调 `System.arraycopy`）完成新数组分配与旧数据迁移。
- **面试官连环追问**：
  - **追问**：`System.arraycopy` 和 `Arrays.copyOf` 有什么区别？
    - **回答**：`System.arraycopy` 是 `native` 方法，需要调用方提前创建好目标数组，将源数组指定范围拷贝到目标数组指定位置；`Arrays.copyOf` 底层也是调用 `System.arraycopy`，但会自动根据指定的新长度先创建好新数组再拷贝返回。
- **实战避坑**：若能预估集合大小（例如查出 1000 条 DB 记录转 DTO），务必显式指定容量 `new ArrayList<>(list.size())`，消除多次扩容拷贝及旧数组产生的堆垃圾。

---

### Q4：`ArrayList.subList()` 有哪些致命陷阱？

- **核心结论**：`list.subList(fromIndex, toIndex)` **返回的不是一个新的 `ArrayList`**，而是 `ArrayList` 的静态/非静态内部类 **`SubList` 视图**，它和原列表共用同一份底层数组 `elementData`！
- **五大致命陷阱**：
  1. **不能强转为 `ArrayList`**：`SubList` 继承自 `AbstractList`，强转 `(ArrayList<T>) list.subList(0, 2)` 直接抛出 **`ClassCastException`**！
  2. **修改子列表直接污染原列表**：对 `subList` 执行 `set()`、`add()`、`remove()`，原 `ArrayList` 的对应数据也会同步被修改。
  3. **修改原列表结构触发 `ConcurrentModificationException`**：一旦对原 `ArrayList` 增删元素（导致 `modCount` 变化），再操作之前切出来的 `subList` 任何方法（包括 `get`/`size`），都会立即抛出 **`ConcurrentModificationException`**！
  4. **造成内存泄漏（OOM）**：即使原列表是一个包含 100 万条大对象的集合，如果你切出一个只含 1 个元素的 `subList` 并长期持有该 `subList` 引用，由于 `SubList` 隐式持有外部整个原 `ArrayList` 对象（`parent` 字段），导致 **100 万条数据全部无法被 GC 回收**！
  5. **不支持序列化**：`SubList` 没有实现 `Serializable` 接口，若将 `subList` 返回给 RPC 接口直接报错。
- **实战避坑**：如果只是需要一个独立的子集合，切完后务必立刻拷贝构造新集合：`new ArrayList<>(list.subList(from, to))`。

---

### Q5：`Arrays.asList()` 和 JDK 9 `List.of()` 有哪些坑？

- **核心结论**：两者返回的都是**定长或不可变集合**，绝不能直接当普通 `ArrayList` 进行增删！
- **核心区别与 5 大坑点**：

| 对比维度 | `Arrays.asList(T... a)` | `List.of(E... elements)`（JDK 9+） |
| :--- | :--- | :--- |
| **返回类类型** | `Arrays$ArrayList`（`Arrays` 内部类） | `ImmutableCollections$ListN`（真正不可变集合） |
| **能否 `add`/`remove`** | 不能，抛 `UnsupportedOperationException` | 不能，抛 `UnsupportedOperationException` |
| **能否 `set` 修改元素** | **可以**（且同步修改原数组） | **不能**，抛 `UnsupportedOperationException` |
| **是否支持 `null` 元素** | **允许 `null`** | **禁止 `null`**（传入直接抛 `NullPointerException`） |
| **基本类型数组陷阱** | **`int[] arr = {1,2}; Arrays.asList(arr)` 的 `size` 是 1**！因为基本类型数组不被拆箱，整个 `int[]` 被当成唯一泛型对象 `T` | 同理，需传包装类数组 `Integer[]` |

- **实战避坑**：快速创建**可增删**的列表，推荐 `new ArrayList<>(Arrays.asList("a", "b"))` 或 `new ArrayList<>(List.of("a", "b"))`。

---

### Q6：什么是 `fail-fast`（快速失败）和 `fail-safe`（安全失败）？遍历删除元素的正确姿势？

- **核心原理**：
  - **`fail-fast`（快速失败）**：代表如 `ArrayList`、`HashMap`。集合内部维护修改次数 **`modCount`**；创建迭代器时记录快照 **`expectedModCount = modCount`**。在每次 `next()` / `remove()` 时检查 `if (modCount != expectedModCount)`，只要发现不一致（无论是其他线程修改，还是当前线程在 `foreach` 中直接调用 `list.remove()`），立即抛出 **`ConcurrentModificationException`（CME）**，防止在不一致状态继续出错。
  - **`fail-safe`（安全失败）**：代表如 `CopyOnWriteArrayList`、`ConcurrentHashMap`。遍历基于写时复制的旧数组快照或弱一致性节点视图，**不抛 CME**，但遍历过程中可能**读不到最新修改**（弱一致性）。
- **遍历并删除元素的 4 种正确姿势**：
  1. **使用 `Iterator.remove()`（通用推荐）**：`iterator.remove()` 在删除后会同步更新 `expectedModCount = modCount`，不会触发 CME。
  2. **使用 `Collection.removeIf()`（JDK 8+ 最简洁）**：`list.removeIf(item -> item.equals("x"))`，内部基于迭代器高效安全删除。
  3. **倒序 `for` 循环遍历删除**（仅限支持随机访问的 `ArrayList`）：`for (int i = list.size() - 1; i >= 0; i--)`，倒序删除不会影响前面尚未遍历元素的下标（正序删除会漏删紧邻元素！）。
  4. **Stream `filter` 过滤生成新集合**：不修改原集合，函数式无副作用。
- **实战避坑**：`foreach` 循环编译期会展开为 `Iterator` 遍历。**特例坑**：如果在 `ArrayList` 的 `foreach` 中删除**倒数第二个元素**，可能**恰好不会抛 CME**！原因在于删除后 `cursor == size`，下一次 `hasNext()` 返回 `false` 直接退出循环，并未调用 `next()` 检查 `modCount`！千万别以为测试偶尔没报错就认为 `foreach` 里调用 `list.remove()` 是安全的！

---

### Q7：`CopyOnWriteArrayList` 源码原理、适用场景与缺陷是什么？

- **核心原理（写时复制 Copy-On-Write）**：
  - 底层数组使用 `private transient volatile Object[] array;` 修饰。
  - **读操作（`get`/`iterator`）**：**完全不加锁**，直接读取当前 `volatile` 数组引用，实现极高并发读性能。
  - **写操作（`add`/`set`/`remove`）**：先加互斥锁（JDK 8 用 `ReentrantLock`，JDK 11+ 内部改为 `synchronized (lock)`），调用 `Arrays.copyOf()` 复制一份新数组（长度 +1），在新数组完成修改后，调用 `setArray(newElements)` 更新 `volatile` 数组引用（对其他读线程立即可见），最后释放锁。
- **适用场景**：**读多写少、数据量不大、允许短暂数据不一致**的场景（如黑白名单校验、路由规则表、系统监听器列表）。
- **四大缺陷**：
  1. **内存占用翻倍**：每次写操作都会复制整份数组，大集合下极易引发频繁 Young GC 甚至 Full GC / OOM。
  2. **写性能极差**：多写并发时串行加锁且反复拷贝全量数组（若需批量添加，务必使用 `addAll()` 只拷贝一次，绝不在循环里调 `add()`）。
  3. **弱一致性（无法保证实时读取）**：写线程修改完成赋值新引用之前，读线程依然读到旧数组快照。
  4. **迭代器不支持修改**：`COWIterator` 的 `remove()`/`set()`/`add()` 直接抛 `UnsupportedOperationException`。

---

### Q8：`HashSet`、`LinkedHashSet`、`TreeSet` 的底层实现与去重原理？

- **核心结论**：JDK 中所有 `Set` 底层**全部复用了对应的 `Map`**（将 Set 的元素作为 Map 的 **`Key`**，Value 统一填充为一个静态虚设空对象 `private static final Object PRESENT = new Object()`）：
  - **`HashSet`**：底层就是 **`HashMap`**。去重逻辑完全依赖元素对象的 **`hashCode()` 与 `equals()`**：先算 `hashCode` 定位桶，再遍历比对 `hash` 相同且 `(k == key || key.equals(k))`，存在则保留旧 Key 不插入。
  - **`LinkedHashSet`**：继承自 `HashSet`，调用父类包级私有构造器初始化底层为 **`LinkedHashMap`**，因此既去重又能严格按**插入顺序**遍历。
  - **`TreeSet`**：底层是 **`TreeMap`**（红黑树）。**去重逻辑不再依赖 `hashCode`/`equals`**，而是依赖 **`Comparable.compareTo()` 或 `Comparator.compare()` 返回 `0`**！
- **实战避坑**：使用 `TreeSet` 存储自定义对象时，如果对象的 `compareTo()` 返回 `0`（例如仅按年龄排序且年龄相同），`TreeSet` 会认为这两个对象“相等”而直接**去重丢弃**第二个对象！若不想被去重，比较器必须在主属性相等时补充次要唯一字段（如 `id`）进行二次比较。

---

### Q9：`PriorityQueue`（优先队列）底层原理是什么？Top-K 怎么实现？

- **核心结论**：`PriorityQueue` 是基于**数组实现的二叉小顶堆（默认自然序最小元素在堆顶）**的无界非线程安全优先队列（不支持 `null` 元素）。
- **二叉堆的数组下标映射公式**（根节点在下标 `0`）：
  - 父节点下标：`(i - 1) >>> 1`
  - 左子节点下标：`(i << 1) + 1`
  - 右子节点下标：`(i << 1) + 2`
- **核心入队与出队过程**：
  - **入队（`offer` -> `siftUp` 上浮）**：将新元素放到数组末尾 `size` 处，然后不断与父节点比较；若比父节点小（小顶堆），则交换位置上浮，直到满足堆性质或到达堆顶（$O(\log n)$）。
  - **出队（`poll` -> `siftDown` 下沉）**：取出堆顶 `queue[0]` 返回，将数组最后一个元素挪到 `queue[0]` 处，然后不断与左右子节点中较小的那个比较并向下交换下沉，直到恢复堆结构（$O(\log n)$）。
- **经典面试手撕：海量数据中求最大的前 K 个数（Top-K 问题）**
  - **解题关键**：求**最大的 K 个**反而要维护一个大小为 K 的**小顶堆**（堆顶是这 K 个里最小的门槛），新元素只有大于堆顶才踢掉堆顶入堆，时间复杂度 $O(n \log K)$，空间复杂度 $O(K)$！

```java
public List<Integer> topKMax(int[] nums, int k) {
    PriorityQueue<Integer> minHeap = new PriorityQueue<>(k); // 默认即小顶堆
    for (int num : nums) {
        if (minHeap.size() < k) {
            minHeap.offer(num);
        } else if (num > minHeap.peek()) {
            minHeap.poll();
            minHeap.offer(num);
        }
    }
    return new ArrayList<>(minHeap);
}
```

---

### Q10：`ArrayDeque` 与 `LinkedList` 实现栈和队列有何差异？为什么淘汰了 `Stack`？

- **为什么彻底淘汰 `java.util.Stack`？**
  - `Stack` 是 JDK 1.0 遗留类，继承自古老的 `Vector`，所有方法都加了粗粒度的 `synchronized` 同步锁，单线程下性能极差；且因为继承 `Vector` 破坏了栈的封装性（竟允许通过下标在中间插入删除元素）。
- **`ArrayDeque`（官方强烈推荐的栈/队列实现）**：
  - **底层结构**：**循环数组（Circular Array）** + `head` / `tail` 双指针（容量始终保持为 **2 的幂**，利用位运算 `(head - 1) & (elements.length - 1)` 实现指针回绕取模）。
  - **对比 `LinkedList`**：`ArrayDeque` 内存连续、无 `Node` 对象封装开销与 GC 压力、CPU 缓存命中率极高，无论是做栈（`push`/`pop`）还是队列（`offer`/`poll`），**实测性能均全面碾压 `LinkedList`**（仅注意 `ArrayDeque` **不允许放入 `null`**）。

---

### Q11：`Collections.sort()` 和 `Arrays.sort()` 底层用的是什么排序算法？

- **基本数据类型数组（`Arrays.sort(int[])`）**：
  - 采用 **`DualPivotQuicksort`（双轴快速排序）**。小数组（长度 $< 47$）退化为插入排序；中大型数组选取 5 个采样点中的 2 个作为双轴划分三段区间，比经典单轴快排减少了约 20% 的数据移动与比较次数；若递归过深或元素极少有优化兜底（计数排序/归并检查）。
- **对象数组与集合（`Arrays.sort(T[])` / `Collections.sort(List<T>)`）**：
  - JDK 7+ 统一采用 **`TimSort`**（融合了**归并排序**与**二分插入排序**的稳定排序算法，时间复杂度稳定在 $O(n \log n)$，面对现实业务中大量“局部已有序”的序列可达到接近 $O(n)$）。
- **实战避坑（线上高频异常）**：
  - JDK 7 起 `TimSort` 严格校验比较器的**自反性、反对称性、传递性（严格弱序）**！如果自定义 `Comparator` 写了诸如 `return a.getScore() > b.getScore() ? 1 : -1;`（漏掉了相等返回 `0`！），运行期会直接抛出 **`IllegalArgumentException: Comparison method violates its general contract`**！必须使用 `Integer.compare(a, b)` 或 `Comparator.comparingInt(...)`！

---

## 2. HashMap 源码级深度剖析（Q12 ~ Q21）

### Q12：`HashMap` 在 JDK 1.7 和 JDK 1.8 中的底层实现及核心区别？

- **核心对比**：

| 对比维度 | JDK 1.7 | JDK 1.8 |
| :--- | :--- | :--- |
| **底层数据结构** | **数组 + 单向链表** | **数组 + 单向链表 + 红黑树** |
| **节点类名称** | `Entry<K, V>` | `Node<K, V>` / `TreeNode<K, V>` |
| **链表插入方式** | **头插法**（多线程扩容易成环死循环） | **尾插法**（避免死循环，并发仍会数据覆盖） |
| **Hash 扰动计算** | 9 次位运算（4 次右移 + 5 次异或） | **2 次运算**：`(h = key.hashCode()) ^ (h >>> 16)` |
| **初始化时机** | 构造器中按参数立即初始化数组（懒加载是 `inflateTable`） | 构造器仅设负载因子与阈值，**首次 `put` 才分配数组** |
| **扩容与插入顺序** | 先判断是否需要扩容，再插入新节点 | **先插入节点，再判断 `++size > threshold` 触发扩容** |
| **扩容位置重算** | 重新调用 `indexFor(e.hash, newCap)` 全部重算 | 利用高位掩码 `e.hash & oldCap == 0` 拆分为原位或 `原位 + oldCap` |
| **最坏查询复杂度** | 链表过长时退化为 **$O(n)$** | 链表转红黑树后退化为 **$O(\log n)$** |

---

### Q13：详细描述 `HashMap` JDK 1.8 `putVal()` 的完整执行流程

- **核心流程图与步骤解析**：

```text
put(key, value)
  │
  ▼
计算 hash(key) = (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16)
  │
  ▼
1. table 是否为空或长度为 0？ ──(是)──▶ resize() 初始化容量（默认 16）
  │ (否)
  ▼
2. 计算桶索引 i = (n - 1) & hash；若 tab[i] == null ──(是)──▶ 直接新建 Node 放入 tab[i]
  │ (否，发生哈希冲突)
  ▼
3. 判断桶头节点 p 的 key 是否与插入 key 相等？ ──(是)──▶ 记录 e = p，准备覆盖 value
  │ (否)
  ▼
4. 判断 p 是否为 TreeNode（红黑树）？ ──(是)──▶ 调用 putTreeVal() 插入或命中树节点
  │ (否，普通链表)
  ▼
5. 遍历链表（尾插法）：
   - 若遍历过程中遇到相同 key，记录 e 并 break；
   - 若到达链表尾部（p.next == null），尾插新 Node；
   - 若此时链表节点数达到 TREEIFY_THRESHOLD - 1（即新插入后长度 >= 8，binCount >= 7），调用 treeifyBin()（内部先检查 table.length >= 64，不足则扩容，>= 64 则转红黑树）。
  │
  ▼
6. 若 e != null（key 已存在）：且 !onlyIfAbsent，覆盖旧 value 并返回旧值（不触发扩容）。
  │
  ▼
7. 否则 ++modCount，若 ++size > threshold ──▶ resize() 扩容为 2 倍！
```

---

### Q14：为什么 `HashMap` 的数组容量必须是 2 的幂次方？`tableSizeFor` 如何工作？

- **必须是 2 的幂次方的三大核心原因**：
  1. **位运算替代取模，性能极高**：当容量 $n = 2^k$ 时，`hash % n` 完全等价于 **`(n - 1) & hash`**，CPU 位运算速度远超除法取模指令。
  2. **均匀散列，减少碰撞**：`n` 为 2 的幂时，`n - 1` 的二进制低位全为 `1`（如 `16 - 1 = 15 -> 0000 1111`），按位与后能够覆盖 `[0, n-1]` 的每一个槽位；如果 `n` 不是 2 的幂（如 `15`，`n - 1 = 14 -> 1110`），最低位永远是 `0`，所有奇数索引桶（`0001`, `0011`...）将永远无法命中，碰撞率翻倍！
  3. **扩容高效迁移**：容量翻倍扩容时，新掩码仅仅在最高有效位多了一个 `1`，因此元素的新下标只需看 `hash` 对应那一位是 `0` 还是 `1`（即 `hash & oldCap == 0`），直接把原链表拆分成两条分别放入原索引或 `原索引 + oldCap`，无需重新计算哈希下标。
- **`tableSizeFor(int cap)` 原理**：
  - 当你传入非 2 的幂（如 `new HashMap<>(17)`）时，构造函数通过 `tableSizeFor(17)` 将其向上取整到**大于等于该值的最小 2 的幂（即 `32`）**。
  - 实现原理：先 `n = cap - 1`（防止原本就是 2 的幂时翻倍），然后连续 5 次无符号右移并按位或（`n |= n >>> 1; n |= n >>> 2; n |= n >>> 4; n |= n >>> 8; n |= n >>> 16;`），把最高位 `1` 后面的所有低位全变成 `1`，最后 `n + 1` 即为最小的 2 的幂！

---

### Q15：`HashMap` 的扰动函数 `(h = key.hashCode()) ^ (h >>> 16)` 有什么用？

- **核心原因**：`HashMap` 默认初始容量只有 `16`（`n - 1 = 15`，二进制低 4 位有效）。
- 如果直接用 `key.hashCode()` 和 `n - 1` 做与运算，**`hashCode()` 的高 16 位特征完全被屏蔽丢弃了**！如果有大量对象的 `hashCode()` 仅在高位有差异、低 4 位相同，就会全部撞在同一个桶里。
- **扰动机制**：将 `hashCode` 无符号右移 16 位，让**高 16 位与低 16 位做异或（XOR）混合**（选择异或是因为与运算偏向 0、或运算偏向 1，只有异或能保留 0/1 概率 50% 的均衡性），用极小的位运算成本让高位信息也参与桶下标路由，显著降低碰撞概率。

---

### Q16：为什么默认负载因子 `loadFactor` 是 `0.75`？初始化 `HashMap` 容量怎么算最优？

- **为什么默认是 `0.75`？（时间与空间的折中平衡）**
  - **太大（如 `1.0`）**：虽然数组空间利用率高，但元素装满才扩容，导致哈希冲突剧增、链表变长甚至频繁树化，查询性能急剧下降。
  - **太小（如 `0.5`）**：空间利用率极低，刚存一半就触发昂贵的 `resize()` 扩容拷贝，浪费大量内存。
  - **统计学最优解**：根据源码注释中的牛顿二项式与泊松分布推算，`0.75` 是时间复杂度与空间利用率的黄金平衡点（此时扩容阈值 `threshold = capacity * 0.75` 也是整数，避免浮点数精度问题）。
- **实战避坑（初始化容量公式）**：
  - 已知需要存放 `expectedSize` 个元素（如 1000 个）时，**不要直接写 `new HashMap<>(1000)`**！因为 `1000` 会向上对齐到 `1024`，但触发扩容的阈值是 `1024 * 0.75 = 768`，放到第 769 个元素时依然会触发一次无谓的 `resize` 扩容到 `2048`！
  - **正确计算公式**：`(int) (expectedSize / 0.75f + 1.0f)`（例如存 1000 个元素应传 `1335`，对齐到 `2048`，阈值为 `1536 > 1000`，零扩容）；JDK 19+ 可直接调用 **`HashMap.newHashMap(expectedSize)`**，Guava 可用 `Maps.newHashMapWithExpectedSize(expectedSize)`。

---

### Q17：为什么红黑树树化阈值是 `8`，退化阈值是 `6`，且要求数组容量 $\ge 64$？

- **核心原理解析（结合源码注释中的泊松分布）**：
  1. **为什么树化阈值是 `8`？**
     - 红黑树节点 **`TreeNode` 占用的内存大小约为普通链表节点 `Node` 的 2 倍**（多出了 `parent`、`left`、`right`、`prev`、`red` 等字段），不到万不得已不轻易树化。
     - JDK 源码作者通过统计计算：在理想随机哈希且负载因子为 `0.75` 时，单桶链表长度分布遵循参数 $\lambda = 0.5$ 的**泊松分布**——长度为 8 的概率仅为 **$0.00000006$（千万分之六）**！
     - 因此正常情况下几乎不可能达到 8；一旦长度超过 8，说明哈希冲突已极度恶化（如用户重写了极差的 `hashCode()` 或遭受 Hash 碰撞 DoS 攻击），此时转红黑树将查找从 $O(n)$ 降至 $O(\log n)$ 性价比极高。
  2. **为什么退化（反树化 `untreeify`）阈值是 `6` 而不是 `7`？**
     - 为了留出一个 **缓冲滞后区间（Hysteresis）**，防止节点数量恰好在 `8` 附近频繁插入、删除，导致数据结构在“链表 $\leftrightarrow$ 红黑树”之间**反复震荡转换**浪费性能。
  3. **为什么要求数组长度 $\ge 64$（`MIN_TREEIFY_CAPACITY`）才树化？**
     - 当数组长度还很小（如 16 或 32）时，单桶碰撞多往往只是因为“桶太少”拥挤所致。此时优先通过 **`resize()` 翻倍扩容把冲突散开**（不仅解决当前桶，还能顺便分散其他桶），远比原地维护复杂的红黑树划算；只有当数组长度达到 64 以上仍发生 8 次碰撞，才真正执行树化。

---

### Q18：`HashMap` JDK 1.8 `resize()` 扩容源码细节：如何不用重新计算 Hash 确定新桶位？

- **核心原理（高位拆分，JDK 1.8 经典优化）**：
  - 假设旧容量 `oldCap = 16`（二进制 `0001 0000`），新容量 `newCap = 32`（`0010 0000`）。
  - 旧掩码 `oldCap - 1 = 0000 1111`（看低 4 位）；新掩码 `newCap - 1 = 0001 1111`（看低 5 位）。
  - 对比发现，新索引只比旧索引多考虑了 **`hash` 的倒数第 5 位**！而这一位正好可以通过 **`(e.hash & oldCap) == 0`** 直接测出来：
    - 若结果为 **`0`**（该位是 0）：元素扩容后的新下标 **保持不变（仍在原索引 `j`）**，串入低位链表 `loHead / loTail`。
    - 若结果 **`!= 0`**（该位是 1）：元素扩容后的新下标为 **`j + oldCap`**，串入高位链表 `hiHead / hiTail`。
  - 遍历完原桶链表后，直接把 `loHead` 挂到 `newTab[j]`，把 `hiHead` 挂到 `newTab[j + oldCap]`，不仅**省去了重新计算哈希下标的时间，还利用两条尾插链表保持了节点原有的相对顺序**！
  - （注：若当前桶是红黑树 `TreeNode`，扩容拆分后如果低位/高位子树节点数 $\le 6$，会直接调用 `untreeify()` 退化回普通链表！）

---

### Q19：JDK 1.7 `HashMap` 多线程扩容为什么会死循环？JDK 1.8 还会线程不安全吗？

- **JDK 1.7 环形链表死循环原理（头插法 + 并发 `transfer`）**：
  - JDK 1.7 扩容迁移链表采用**头插法**，每遍历一个旧节点就插到新桶头部，会导致原链表节点顺序**完全反转**。
  - **并发场景**：假设线程 A 和线程 B 同时触发扩容，原桶 0 有链表 `A -> B -> null`。
    - 线程 A 执行到 `Entry<K,V> next = e.next;`（此时局部指针 `e = A`，`next = B`）时被 CPU 挂起。
    - 线程 B 一气呵成完成扩容，由于头插法反转，新表里变成了 `B -> A -> null`（此时堆中对象 B 的 `next` 已指向 A，A 的 `next` 指向 `null`）。
    - 线程 A 恢复执行，继续用挂起前的局部引用操作：先把 `A` 头插到自己的新表；接着处理 `next` 即 `B`，把 `B` 头插到 `A` 前面（`B -> A`）；但紧接着读取 `B.next` 时，读到的正是线程 B 修改后的 **`A`**！于是又把 `A` 插到 `B` 前面，最终形成 **`A.next = B` 且 `B.next = A` 的环形链表**！
    - 后续只要有线程调用 `get()` 查不到 Key 且刚好落到这个桶，就会在链表中无限循环导致 **CPU 100%**！
- **JDK 1.8 改用尾插法后就线程安全了吗？**
  - **依然线程不安全！** 虽然尾插法消除了环形链表死循环，但在多线程并发下存在两大严重问题：
    1. **数据覆盖（丢失）**：在 `putVal()` 中，线程 A 和线程 B 同时判断 `tab[i] == null`（或同时拿到同一个链表尾节点），线程 A 写入后，线程 B 紧接着直接写入覆盖了线程 A 的节点！
    2. **`size` 计数不准**：`++size` 非原子操作，多线程并发写入后 `size()` 小于实际元素数。
    3. **并发读写触发 `modCount` 异常或读到未初始化完的扩容表**。

---

### Q20：为什么 `HashMap` 允许 `null` Key，而 `ConcurrentHashMap` / `Hashtable` 不允许？

- **核心结论（二义性原则，ConcurrentHashMap 作者 Doug Lea 解释）**：
  - **`HashMap`**：允许 **1 个 `null` Key**（`hash(null)` 固定返回 `0`，存在下标 `0` 的桶中），允许多个 `null` Value。因为 `HashMap` 用于单线程环境，当 `map.get(k)` 返回 `null` 时，可以通过紧接着调用 `map.containsKey(k)` 明确区分“Key 不存在”还是“Key 对应的 Value 本身就是 `null`”。
  - **`Hashtable` & `ConcurrentHashMap`**：**Key 和 Value 都严禁为 `null`**（直接抛 `NullPointerException`）。因为在**多线程并发环境**下，如果你先调用 `map.containsKey(k)` 返回 `true`，但在你紧接着调用 `map.get(k)` 的间隙，另一个线程可能已经删掉了这个 Key，导致无法消除“未命中 vs 值本身为空”的**二义性（Ambiguity）**。
- **实战避坑（可变对象作为 Key 的致命后果）**：
  - 如果把一个重写了 `hashCode()`/`equals()` 的可变对象作为 Key 放入 `HashMap` 后，再修改这个对象的属性字段，会导致该对象的 `hashCode()` 发生变化！此时再调用 `map.get(key)` 或 `map.remove(key)` 会算到另一个空桶里去，导致**数据“查不到也删不掉”引发永久内存泄漏**！**`HashMap` 的 Key 必须使用不可变对象**（如 `String`、`Integer`）。

---

### Q21：`HashMap` 中的红黑树节点 `TreeNode` 是如何比较大小进行插入查找的？

- **核心原理解析（`putTreeVal` / `find`）**：
  - 很多人困惑：放入 `HashMap` 的 Key 未必实现了 `Comparable` 接口，红黑树是二叉排序树，它怎么比较两个 Key 的“大小”？
  - **四级比较规则（源码顺序）**：
    1. **优先比较 `hash` 值**：`ph = p.hash`，若 `ph > h` 走左子树，`ph < h` 走右子树。
    2. **若 `hash` 相同，先比 `equals`**：如果 `pk == k || (k != null && k.equals(pk))`，直接命中返回！
    3. **如果 `hash` 相同且不 `equals`，检查是否可比较**：只有当两个 Key 属于**同一个类且实现了 `Comparable` 接口**（`comparableClassFor(k)`），才调用 `compareComparables()` 比较大小（注意：如果实现 `Comparable` 但 `compareTo` 返回 `0` 且 `equals` 为 `false`，继续走第 4 步）。
    4. **终极决胜法 `tieBreakOrder(Object a, Object b)`**：先比较两个类名的字典序，若类名相同（同一类），则调用 **`System.identityHashCode(a)` vs `System.identityHashCode(b)`**（基于原始对象标识哈希码比较，决不返回 `0`）强制决定左右子树方向！

---

## 3. LinkedHashMap、TreeMap、WeakHashMap 与同步 Map（Q22 ~ Q27）

### Q22：`LinkedHashMap` 是如何实现有序的？两种有序模式有什么区别？

- **核心原理**：`LinkedHashMap` 继承自 `HashMap`，节点 `LinkedHashMap.Entry` 继承自 `HashMap.Node`，额外增加了 **`before` 和 `after` 两个指针**。从而在哈希桶数组+链表/红黑树的结构之上，**串联出一条贯穿所有节点的双向链表**（维护 `head` 和 `tail`）。
- **构造参数 `boolean accessOrder` 控制两种顺序**：
  - **`accessOrder = false`（默认）：插入顺序**。新元素始终挂在双向链表尾部 `tail`；修改已存在 Key 的 Value 不改变顺序。
  - **`accessOrder = true`：访问顺序（LRU 基石）**。每当调用 **`get(k)` 或 `put(k, v)` 命中已存在节点**时，钩子方法 **`afterNodeAccess(Node<K,V> e)`** 会把该节点从链表当前位置摘除并移动到双向链表**尾部 `tail`**（最近访问过的都在尾部，最久未访问的自动沉淀在头部 `head`！）。

---

### Q23：如何手写一个生产/面试级 LRU 缓存？（双版本代码）

- **版本一：继承 `LinkedHashMap`（极简版 + 加锁线程安全）**
  - 重写 **`removeEldestEntry(Map.Entry<K, V> eldest)`**，该方法在 `HashMap.putVal` 结束时的回调钩子 `afterNodeInsertion` 中被调用。

```java
public class SimpleLruCache<K, V> extends LinkedHashMap<K, V> {
    private final int capacity;

    public SimpleLruCache(int capacity) {
        // 负载因子 0.75，开启 accessOrder = true
        super((int) (capacity / 0.75f) + 1, 0.75f, true);
        this.capacity = capacity;
    }

    @Override
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        return size() > capacity; // 超过容量自动删除双向链表头节点（最久未使用）
    }
}
```

- **版本二：纯手写 `HashMap + 双向链表`（LeetCode 146 高频大厂手撕标准答案，时间复杂度 $O(1)$）**

```java
import java.util.HashMap;
import java.util.Map;

public class LRUCache<K, V> {
    private static class Node<K, V> {
        K key;
        V val;
        Node<K, V> prev, next;
        Node() {}
        Node(K key, V val) { this.key = key; this.val = val; }
    }

    private final int capacity;
    private final Map<K, Node<K, V>> map = new HashMap<>();
    private final Node<K, V> head = new Node<>(); // 哨兵头节点（最久未使用侧）
    private final Node<K, V> tail = new Node<>(); // 哨兵尾节点（最新访问侧）

    public LRUCache(int capacity) {
        this.capacity = capacity;
        head.next = tail;
        tail.prev = head;
    }

    public synchronized V get(K key) {
        Node<K, V> node = map.get(key);
        if (node == null) return null;
        moveToTail(node);
        return node.val;
    }

    public synchronized void put(K key, V value) {
        Node<K, V> node = map.get(key);
        if (node != null) {
            node.val = value;
            moveToTail(node);
            return;
        }
        if (map.size() >= capacity) {
            Node<K, V> eldest = head.next; // 淘汰哨兵头后面的第一个真实节点
            removeNode(eldest);
            map.remove(eldest.key);
        }
        Node<K, V> newNode = new Node<>(key, value);
        map.put(key, newNode);
        addToTail(newNode);
    }

    private void moveToTail(Node<K, V> node) {
        removeNode(node);
        addToTail(node);
    }

    private void removeNode(Node<K, V> node) {
        node.prev.next = node.next;
        node.next.prev = node.prev;
    }

    private void addToTail(Node<K, V> node) {
        node.prev = tail.prev;
        node.next = tail;
        tail.prev.next = node;
        tail.prev = node;
    }
}
```

---

### Q24：`TreeMap` 底层原理是什么？`Comparable` 与 `Comparator` 有什么区别？

- **底层原理**：`TreeMap` 基于**红黑树（自平衡二叉查找树）**实现，增删查时间复杂度稳定在 **$O(\log n)$**，支持按 Key 自然排序或自定义比较器排序，且实现了 `NavigableMap` 接口（支持 `floorKey`、`ceilingKey`、`subMap` 等范围查找，非常适合实现一致性哈希环！）。
- **红黑树的 5 条核心性质**（保证最长路径不超过最短路径的 2 倍，弱平衡减少旋转）：
  1. 每个节点非红即黑；
  2. 根节点为黑；
  3. 所有叶子节点（NIL 空节点）为黑；
  4. 红色节点的两个子节点必为黑（不能有连续红节点）；
  5. 从任一节点到其每个叶子的所有路径都包含相同数目的黑节点。
- **`Comparable` vs `Comparator`**：
  - **`Comparable`（内部比较器）**：定义在 `java.lang` 包，由数据类自身实现 `compareTo(T o)` 方法（如 `String`、`Integer`），强耦合但可作为默认自然序。
  - **`Comparator`（外部比较器）**：定义在 `java.util` 包，作为策略模式独立传入 `compare(T o1, T o2)`，符合**开闭原则**（无需修改原类即可灵活定制多维度排序规则，如先按年龄降序再按姓名升序）。

---

### Q25：`WeakHashMap` 底层原理是什么？有哪些经典生产应用？

- **核心结论**：`WeakHashMap` 的 **`Entry<K, V>` 继承自 `WeakReference<Object>`**，即它的 **Key 是弱引用**（Value 仍是强引用）！只要外部不再持有某个 Key 的强引用，**下一次 GC 无论内存是否充足都会回收该 Key**。
- **自动清理过期 Entry 的源码机制**：
  - `WeakHashMap` 内部维护了一个引用队列 `ReferenceQueue<Object> queue`。
  - 当 Key 被 GC 回收时，JVM 会把对应的 `Entry` 对象放入 `queue` 中。
  - 在后续每次调用 `WeakHashMap` 的 `get()`、`put()`、`size()` 等方法时，都会触发 **`expungeStaleEntries()`**：从 `queue` 中循环取出已失效的 `Entry`，将其从哈希桶链表中摘除，并将 `e.value = null` 断掉强引用，防止 Value 内存泄漏！
- **经典应用**：
  - **高速缓存（对内存敏感且 Key 生命周期与外部绑定的元数据缓存）**。例如 **Tomcat** 中的 `ConcurrentCache`（`eden` 区用 `ConcurrentHashMap`，满后降级将冷数据放入 `longterm` 区的 `WeakHashMap` 避免 OOM）；以及各类反射元数据/类加载器缓存（防止 ClassLoader 卸载时被静态 Map 死死拽住无法回收导致 Metaspace OOM）。
- **实战避坑**：绝对不要用字符串字面量（如 `"abc"`）或缓存范围内的 `Integer`（如 `100`）作为 `WeakHashMap` 的 Key！因为它们存在于常量池/缓存池中始终有强引用，永远不会被 GC 回收。

---

### Q26：`IdentityHashMap` 和 `EnumMap` 有什么特殊用途？

- **`IdentityHashMap`**：
  - **核心区别**：普通 `HashMap` 使用 `equals()` 和 `hashCode()` 判等；而 `IdentityHashMap` 严格使用 **`==`（内存地址相等）** 判等，哈希值使用 **`System.identityHashCode(k)`**！
  - **底层结构**：采用**开放地址法（线性探测）**，直接在一个 `Object[] table` 数组中交替存放 `key` 和 `value`（`table[i]` 存 Key，`table[i+1]` 存 Value）。
  - **应用场景**：对象图深度拓扑遍历防环（判断是不是同一个对象实例）、JVM/Spring 底层保留不同实例的相同内容 Key。
- **`EnumMap`**：
  - **核心原理**：Key 必须是同一个枚举类型。底层直接用**定长数组 `Object[] vals`**（长度等于枚举常量个数 `keyUniverse.length`），通过 `key.ordinal()` 下标直接定位！
  - **优势**：**无哈希计算、无哈希冲突、空间极其紧凑、遍历严格按枚举声明顺序**，在状态机/策略路由中性能远超 `HashMap`。

---

### Q27：`Collections.synchronizedMap()`、`Hashtable` 与 `ConcurrentHashMap` 的区别？

- **核心对比**：
  - **`Hashtable`**：古老遗留类，所有方法直接用 **`synchronized`** 锁整个对象实例，多线程读也会互斥阻塞；不允许 `null` 键值。
  - **`Collections.synchronizedMap(Map)`**：装饰器模式包装普通 `Map`，内部维护一个 `final Object mutex` 互斥锁对象，所有方法用 `synchronized (mutex)` 同步（本质和 `Hashtable` 一样都是**全表串行锁**，并发度仅为 1）。
  - **`ConcurrentHashMap`**：读操作无锁（`volatile`），写操作锁粒度细化到单桶头节点（JDK 1.8）或段（JDK 1.7），支持多线程并发读写与协助扩容。
- **实战避坑（复合操作的原子性陷阱）**：
  - 即使是 `Collections.synchronizedMap` 甚至 `ConcurrentHashMap`，**单方法原子性不代表复合操作原子性**！
  - 错误写法：`if (!map.containsKey(k)) { map.put(k, v); }`（check-then-act 非原子，多线程下会重复覆盖写入）。
  - 正确写法：使用 `ConcurrentHashMap` 原生原子方法 **`putIfAbsent`**、**`computeIfAbsent`**、**`merge`**！

---

## 4. ConcurrentHashMap 源码级深度解析（Q28 ~ Q32）

### Q28：`ConcurrentHashMap` 在 JDK 1.7 中的 `Segment` 分段锁架构是怎样的？

- **核心架构**：由 **`Segment[] segments` 数组 + `HashEntry[] table` 数组 + 单向链表** 构成（两次哈希定位）。
- **核心原理**：
  - 每个 `Segment` 本身继承自 **`ReentrantLock`**，相当于一把独立的小锁，默认有 **16 个 `Segment`**（即默认并发度 `concurrencyLevel = 16`，初始化后不可扩容）。
  - 写入数据时，先对 `key` 做一次哈希定位到具体的 `Segment[i]`，然后只对该 `Segment` 加锁（不影响其他 15 个段），再做第二次哈希定位到段内部的 `HashEntry[]` 桶完成插入；局部扩容也只发生在单个 `Segment` 内部。
  - 读操作 `get()` **不加锁**：因为 `HashEntry` 的 `value` 和 `next` 指针都由 **`volatile`** 修饰，保证了内存可见性（遵循 Happens-Before）。

---

### Q29：`ConcurrentHashMap` JDK 1.8 的底层结构与 `putVal()` 核心流程？

- **底层结构**：与 `HashMap` 1.8 一致，抛弃了 `Segment`，采用 **`Node<K, V>[] table` 数组 + 链表 + 红黑树（`TreeNode`）**，通过 **`CAS + synchronized`** 保证并发安全。
- **`putVal()` 核心源码流程**：
  1. **空校验**：`key == null || value == null` 直接抛 `NullPointerException`。
  2. **计算分散 Hash**：`spread(key.hashCode())` -> `(h ^ (h >>> 16)) & HASH_BITS`（`HASH_BITS = 0x7fffffff`，确保最高位符号位为 `0`，即普通节点 hash 恒为非负数！**负数保留给特殊节点**：`MOVED = -1` 表示正在扩容的 `ForwardingNode`，`TREEBIN = -2` 表示红黑树代理根节点 `TreeBin`）。
  3. **自旋死循环 `for (Node<K,V>[] tab = table;;)`**：
     - **情况 1（表未初始化）**：调用 `initTable()` 初始化默认容量 16 的数组。
     - **情况 2（目标桶为空 `f == null`）**：**无锁 CAS** 调用 `casTabAt(tab, i, null, new Node<>(...))` 尝试直接写入头节点；成功则退出循环，失败说明有并发竞争则自旋重试。
     - **情况 3（桶正在扩容 `fh == MOVED (-1)`）**：说明当前桶头节点是 `ForwardingNode`，当前线程**不干等，而是主动调用 `helpTransfer(tab, f)` 协助扩容迁移**！
     - **情况 4（普通哈希冲突）**：使用 **`synchronized (f)`** 锁住当前桶的头节点 `f`！锁内双重检查 `tabAt(tab, i) == f` 后，若是链表（`fh >= 0`）则尾插/更新；若是红黑树代理节点 `TreeBin`（`fh < 0`）则调用 `putTreeVal`。
  4. 释放锁后，若链表长度达到 `8`，调用 `treeifyBin()`（同样要求数组长度 $\ge 64$ 才真正转红黑树，否则优先 `tryPresize` 扩容）。
  5. 调用 `addCount(1L, binCount)` 更新全局元素计数（类似 `LongAdder` 分段计数）并在必要时触发扩容。

---

### Q30：为什么 JDK 1.8 `ConcurrentHashMap` 放弃 `ReentrantLock` 改用 `synchronized`？

- **四大核心原因**：
  1. **大幅降低锁粒度与内存开销**：JDK 1.7 `Segment` 锁住一整段多个桶；JDK 1.8 直接锁住**单个桶的头节点**（理论上数组有多大，并发度就有多大）。如果改用 `ReentrantLock` 去锁每个桶，就必须让每个 `Node` 继承或持有一个 `ReentrantLock` 对象，内存膨胀不可接受；而 `synchronized` 直接利用每个对象头（Mark Word）即可加锁，**零额外内存成本**。
  2. **JVM 对 `synchronized` 的深度优化**：JDK 6 后引入了锁升级（偏向锁、轻量级锁/自旋、重量级锁）及锁消除/粗化，在大多数哈希碰撞概率较低的桶竞争场景下，无竞争/轻竞争走 CAS 轻量级锁性能极高。
  3. **异常自动释放与语法简洁**：JVM 底层字节码 `monitorenter/monitorexit` 保证即使发生异常也会自动释放锁。
  4. **并发扩容机制支撑**：引入了 `ForwardingNode` 与多线程 `helpTransfer()` 协同迁移，整体吞吐量远胜 1.7 分段锁。

---

### Q31：`ConcurrentHashMap` 中 `sizeCtl` 变量各种取值的含义是什么？

- **`sizeCtl`（`private transient volatile int sizeCtl;`）** 是 `ConcurrentHashMap` 最核心的并发状态控制变量，不同取值含义如下：
  - **`sizeCtl == 0`**：默认值，表示数组尚未初始化且未指定初始容量（首次初始化将使用默认容量 `16`）。
  - **`sizeCtl > 0`**：
    - 数组**未初始化**时：表示**指定的初始容量大小**（注意：1.8 如果构造传 `initialCapacity`，`tableSizeFor` 会算出 2 的幂存于 `sizeCtl`）。
    - 数组**已初始化**时：表示**下一次触发扩容的阈值**（即 `0.75 * capacity`）。
  - **`sizeCtl == -1`**：表示数组**正在初始化中**（`initTable()` 中通过 CAS 将 `sizeCtl` 设为 `-1` 抢占初始化权限，其他线程看到 `-1` 后调用 `Thread.yield()` 让出 CPU 自旋等待）。
  - **`sizeCtl < -1`**（如 `- (1 + n)`）：表示**当前正在进行扩容**，高 16 位记录当前容量对应的**扩容戳（Resize Stamp）**，低 16 位表示 **`1 + 正在参与协助扩容的线程数`**（例如 `rs + 2` 代表有 1 个线程正在扩容，`rs + 3` 代表有 2 个线程正在协同扩容；注意高并发下判断高 16 位戳避免 ABA 问题）。

---

### Q32：`ConcurrentHashMap` 的多线程协助扩容（`helpTransfer`）和 `size()` 计数原理？

- **多线程协助扩容（`transfer`）机制**：
  - 为了避免单线程全量迁移大数组造成长时间阻塞，1.8 将整个数组划分为多个连续的迁移任务块，每块大小称为步长 **`stride`**（根据 CPU 核数计算，最小为 `16` 桶）。
  - 线程通过 CAS 竞争更新 `transferIndex` 领取属于自己的 `[bound, i]` 区间桶块，逐桶锁住头节点将节点拆分为高/低位两条链放入 `nextTable`，迁移完一个空桶或已处理桶后，向旧表该槽位放入 **`ForwardingNode`**（`hash = MOVED = -1`，内部持有 `nextTable` 引用）。
  - 当其他并发线程在 `put` 时遇到 `ForwardingNode`，调用 `helpTransfer()` 领取下一段 `stride` 区间帮忙迁移；若 `get()` 读操作遇到 `ForwardingNode`，则直接通过 `ForwardingNode.find()` **路由到 `nextTable` 新表去查找**，完全不阻塞读！
- **`size()` / `mappingCount()` 高并发计数原理（借鉴 `LongAdder`）**：
  - 放弃全局单一 `AtomicLong`（高并发下 CAS 热点自旋会严重拖垮性能），采用 **`baseCount` + `CounterCell[] counterCells`** 分段计数设计。
  - 无并发竞争时直接 CAS 更新 `baseCount`；一旦 CAS 失败发生竞争，线程通过探针哈希将增量写入各自独立的 **`CounterCell`** 槽位（`CounterCell` 使用 `@Contended` 注解消除 CPU 伪共享）；`size()` / `mappingCount()`（官方推荐后者返回 `long` 防 `int` 溢出）只需将 `baseCount` 与所有 `CounterCell.value` 求和即可（弱一致性实时近似值）。

---

## 5. ThreadLocal 全家桶与内存泄漏（Q33 ~ Q36）

### Q33：`ThreadLocal` 底层架构是什么？`ThreadLocalMap` 如何解决哈希冲突？

- **三者引用关系架构**：
  - 误区纠正：不是 `ThreadLocal` 内部维护了一个 `Map<Thread, Value>`！恰恰相反，**每个 `Thread` 线程对象内部自己维护了一个成员变量 `ThreadLocal.ThreadLocalMap threadLocals`**！
  - `ThreadLocal` 本身不存储数据，它只是一把访问当前线程 `ThreadLocalMap` 的**钥匙（Key）**，实现真正的数据线程隔离。
- **`ThreadLocalMap` 解决哈希冲突（开放地址法 / 线性探测）**：
  - 区别于 `HashMap` 的“拉链法”，`ThreadLocalMap` 底层只有一张 **`Entry[] table`** 数组（初始容量 `16`，负载因子 `2/3`）。
  - `ThreadLocal` 通过内部魔数 **`0x61c88647`（黄金分割数斐波那契散列增量）** 分配极具均匀性的 `threadLocalHashCode`，计算桶位 `i = key.threadLocalHashCode & (len - 1)`。
  - 若 `table[i]` 已被其他 `ThreadLocal` 占用，则向后**线性探测 `nextIndex(i, len)`**（即 `(i + 1) % len`），直到找到空槽位或过期槽位放入（因为一个线程通常不会绑定太多 `ThreadLocal`，开放地址法更节省指针内存且局部性好）。

---

### Q34：`ThreadLocal` 为什么会导致内存泄漏？为什么 Key 用弱引用？

- **内存泄漏完整链路分析**：
  - `ThreadLocalMap.Entry` 继承自 **`WeakReference<ThreadLocal<?>>`**：
    - **Key**（`ThreadLocal` 实例）是**弱引用**（`super(k)`）。
    - **Value**（用户存储的对象）是**强引用**（`this.value = v`）。
  - **泄漏场景**：当业务方法结束，外部对 `ThreadLocal` 对象的强引用消失后，由于 `Entry` 的 Key 是弱引用，下一次 GC 就会把 `ThreadLocal` 实例回收，导致这个 `Entry` 变成 **`Key == null`，但 `Value != null`** 的“脏 Entry”。
  - 如果当前线程是**线程池的核心线程**（长期存活不销毁），存在一条永久的强引用链：`ThreadRef -> Thread -> ThreadLocalMap -> Entry[] -> Entry -> Value`！这个无法再被访问到的 `Value` 对象将永远滞留堆内存中无法回收，最终积累导致 **OOM**！
- **面试官连环追问**：
  - **追问 1**：既然 Key 用弱引用还是会漏 Value，那为什么不干脆把 Key 也设为强引用呢？
    - **回答**：如果 Key 是强引用，一旦外部把 `tl = null` 置空，只要线程不死，不仅 Value 漏了，**连 `ThreadLocal` 对象本身也永远无法被回收**！设为弱引用是 JDK 的**兜底防线**：让 Key 能被自动回收并标记为 `stale`（过期），这样在后续调用 `ThreadLocalMap` 的 `set()`、`get()`、`remove()` 时，内部就能顺便执行**启发式清理（`cleanSomeSlots`）**与**探测式清理（`expungeStaleEntry`）**将脏 Entry 的 Value 置空释放！
  - **追问 2**：既然内部有顺带清理机制，为什么还强调必须手动 `remove()`？
    - **回答**：因为如果线程放入大对象后**再也不调用**任何 `ThreadLocal` 的 `get`/`set`/`remove`，清理机制就永远没有机会触发！**最佳实践**：务必在 `try { ... } finally { threadLocal.remove(); }` 中显式清理（也能顺便防止线程池复用时读到上一个请求的“串号”脏数据！）。

---

### Q35：父子线程和线程池之间如何传递 `ThreadLocal` 上下文？

- **三大演进方案对比**：
  1. **`InheritableThreadLocal`（JDK 原生，父子线程传递）**：
     - **原理**：每个 `Thread` 内部除了 `threadLocals`，还有一个 `inheritableThreadLocals`。在 `new Thread()` 创建子线程调用 `init()` 时，会将父线程当前的 `inheritableThreadLocals` 浅拷贝一份给子线程。
     - **致命缺陷（线程池失效）**：线程池里的工作线程是**提前创建好并长期复用的**，不会在提交任务时每次都 `new Thread()`！因此使用 `InheritableThreadLocal` 在线程池中传递用户/TraceId 上下文会彻底失效甚至串数据！
  2. **阿里开源 `TransmittableThreadLocal`（TTL，线程池上下文传递标准方案）**：
     - **原理**：通过 **`TtlRunnable` / `TtlExecutors`** 装饰线程池提交的任务（或者借助 Java Agent 字节码无侵入增强）。在**提交任务瞬间**，在主线程捕获当前所有 TTL 值保存为快照（`capture`）；等线程池线程真正执行 `run()` 前，先备份该工作线程原有上下文并注入快照值（`replay`）；任务执行完毕后在 `finally` 中恢复该工作线程原有的上下文（`restore`），完美解决复用与串数据问题！
  3. **手动上下文透传**：在提交 `Runnable`/`Callable` 前捕获 MDC / 上下文对象传入包装类。

---

### Q36：JDK 21 `ScopedValue`（作用域值）相比 `ThreadLocal` 有哪些碾压性优势？

- **为什么虚拟线程时代不建议用 `ThreadLocal`？**
  - 虚拟线程数量可达百万级，且用完即毁；若每个虚拟线程都绑定一份可变的 `ThreadLocal` 缓存大对象，以及大量继承拷贝 `InheritableThreadLocal` 的开销，会导致灾难性内存膨胀。
- **`ScopedValue`（JDK 20 孵化 / JDK 21+ 预览转正）四大核心优势**：
  1. **不可变性（Immutable）**：绑定后在作用域内只读不可随意 `set` 篡改，天然线程安全，避免状态混乱。
  2. **明确的生命周期（零内存泄漏）**：通过 `ScopedValue.where(KEY, val).run(() -> { ... })` 绑定，**代码块退出作用域后自动销毁解绑**，再也无需担心忘记 `remove()` 导致的内存泄漏！
  3. **子线程零拷贝共享（$O(1)$）**：配合结构化并发 `StructuredTaskScope`，父虚拟线程派生海量子虚拟线程时，由于值不可变，子线程**直接共享读取父作用域的引用，无需像 `InheritableThreadLocal` 那样拷贝整个 Map**！
  4. **高性能读写**：底层利用线程栈帧上下文和缓存优化，读取开销远低于 `ThreadLocalMap` 哈希查找。

---

## 6. JUC 阻塞队列（BlockingQueue）深度解析（Q37 ~ Q40）

### Q37：`BlockingQueue` 的 4 组核心操作方法有什么区别？

- **4 组 API 行为对比（必背面试表）**：

| 操作类型 | 抛出异常 | 返回特殊值（布尔/null） | **一直阻塞** | **超时等待** |
| :--- | :--- | :--- | :--- | :--- |
| **入队（插入）** | `add(e)`（满抛 `IllegalStateException`） | `offer(e)`（满返 `false`） | **`put(e)`** | `offer(e, time, unit)` |
| **出队（移除）** | `remove()`（空抛 `NoSuchElementException`） | `poll()`（空返 `null`） | **`take()`** | `poll(time, unit)` |
| **检查队头** | `element()`（空抛异常） | `peek()`（空返 `null`） | 不支持 | 不支持 |

- **核心约束**：所有 `BlockingQueue` 实现都**严禁插入 `null` 元素**（直接抛 `NullPointerException`），因为 `poll()` 需要用返回 `null` 来表示队列为空。

---

### Q38：`ArrayBlockingQueue` 和 `LinkedBlockingQueue` 的源码级深度对比？

- **核心对比**：

| 对比维度 | `ArrayBlockingQueue` | `LinkedBlockingQueue` |
| :--- | :--- | :--- |
| **底层结构** | 定长循环数组（`Object[] items`）+ `takeIndex`/`putIndex` | 单向链表（`Node`）+ `head`/`last` 哨兵节点 |
| **有界性** | **必须显式指定容量（有界）** | 无参构造默认容量 **`Integer.MAX_VALUE`（近似无界，易 OOM！）** |
| **锁机制** | **单锁双条件**：1 把 `ReentrantLock` + `notEmpty` / `notFull` 两个 `Condition` | **读写分离双锁**：`putLock`（配 `notFull`）和 `takeLock`（配 `notEmpty`），`count` 用 `AtomicInteger` 维护 |
| **并发吞吐** | 同一时刻只能进或出（读写互斥） | **生产者和消费者可同时并发运行**，高并发下吞吐量通常更高 |
| **内存与 GC** | 初始化一次性分配好数组，**入队出队零对象创建（GC 友好）**；支持构造参数开启**公平锁** | 每次 `put` 都 `new Node`、每次 `take` 都脱链 GC 回收，大数据高吞吐下有一定 GC 压力 |

- **实战避坑**：`Executors.newFixedThreadPool()` 和 `newSingleThreadExecutor()` 底层默认使用的是**无界 `LinkedBlockingQueue`**，一旦下游处理变慢导致任务无限堆积，就会直接把堆内存打爆引发 **OOM**！生产线程池必须手动指定有界队列容量。

---

### Q39：`SynchronousQueue` 的底层原理是什么？和 `LinkedTransferQueue` 有何联系？

- **核心原理**：`SynchronousQueue` 是一个**容量为 0（不存储任何元素）**的阻塞队列。每一个 `put`（生产者）必须阻塞等待一个 `take`（消费者）把元素直接交接走，反之亦然（“一手交钱，一手交货”的直接移交机制）。
- **底层实现（JDK 6+ 无锁双数据结构）**：
  - 内部通过抽象类 `Transferer` 实现，没有存储集合，而是**直接把阻塞的线程本身封装成节点**排队等待交接：
    1. **非公平模式（默认，`TransferStack` 栈，LIFO）**：后来的请求直接与栈顶等待线程匹配交接，无需遍历，CPU 缓存命中率高，吞吐量极大。
    2. **公平模式（`TransferQueue` 队列，FIFO）**：严格按照先来后到顺序匹配交接，避免线程饥饿。
- **典型应用**：**`Executors.newCachedThreadPool()`** 底层队列就是 `SynchronousQueue`（来一个任务直接交接给空闲线程，没有空闲线程立刻新建线程，适合执行大量短耗时异步任务）。
- **扩展：`LinkedTransferQueue`**：结合了 `ConcurrentLinkedQueue`、`SynchronousQueue`（公平模式）和 `LinkedBlockingQueue` 的优点，提供 `transfer(e)`（阻塞等消费者接走）和 `tryTransfer(e)`（有现成等待的消费者才直接移交，否则不阻塞直接返回 `false`）。

---

### Q40：`DelayQueue` 和 `PriorityBlockingQueue` 的底层原理？`DelayQueue` 如何避免 CPU 空转和无效唤醒？

- **`PriorityBlockingQueue`**：
  - 基于**可扩容数组实现的二叉堆** + 单把 **`ReentrantLock`**（仅出队阻塞 `notEmpty`，入队不会阻塞因为数组自动 1.5 倍扩容直到 OOM）。
- **`DelayQueue` 底层原理（组合模式）**：
  - 内部持有一把 `ReentrantLock`、一个普通的非线程安全 **`PriorityQueue`（按到期时间 `getDelay` 排序的小顶堆）** 和一个 `Condition available`。元素必须实现 `Delayed` 接口（`Comparable` + `getDelay(TimeUnit)`）。
  - **核心设计：Leader-Follower（领导者-跟随者）模式**！
    - 当多个消费者线程同时调用 `take()`，且堆顶最快到期的元素还差 `delay > 0` 毫秒才到期时：
    - **不会让所有线程都傻傻调用 `awaitNanos(delay)` 然后到点后一起惊群唤醒抢同一个元素**！
    - 而是通过一个成员变量 `Thread leader`：第一个发现未到期的线程将自己设为 **`leader`**，调用 `available.awaitNanos(delay)` 精确等待剩余时间；**其他随后进来的消费者线程看到 `leader != null`，直接调用无期限的 `available.await()` 挂起当跟随者**！
    - 等 `leader` 到点醒来取走堆顶元素后，在 `finally` 中调用 `available.signal()` 唤醒下一个跟随者线程成为新的 `leader` 去盯新的堆顶，**极大减少了多线程同时超时唤醒造成的无谓上下文切换与锁竞争**！
- **实战避坑**：`DelayQueue` 属于**单机内存级延迟队列**，服务重启会丢失所有未到期任务，且海量延迟任务会撑爆堆内存；生产分布式系统（如订单 30 分钟未支付自动关单）应优先选用 **RocketMQ 延迟消息**、**RabbitMQ 死信/延迟插件** 或 **Redis ZSet + 时间轮**。
