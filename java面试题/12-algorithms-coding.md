# 12 - Java 并发手撕与高频算法面试通关指南（32 题精讲）

> 本章覆盖大厂后端笔试与现场白板/共享屏幕“手撕代码”最高频考题：**Part A 为 6 大 Java 并发编程手写必考题**（交替打印、阻塞队列、单例模式、生产者消费者、手写死锁与排查修复、Java 限流器）；**Part B 为 26 道数据结构与 Hot 100 算法必刷题**（含完整可运行 Java 代码、核心思路、时空复杂度分析、边界用例与面试追问）。

---

## 目录导航

- [Part A：Java 并发编程手撕代码（6 大必考）](#part-ajava-并发编程手撕代码6-大必考)
  - [A1. 多线程交替打印 ABC（3 种经典写法 + 奇偶数打印）](#a1-多线程交替打印-abc3-种经典写法--奇偶数打印)
  - [A2. 手写有界阻塞队列（ReentrantLock + 双 Condition 环形数组）](#a2-手写有界阻塞队列reentrantlock--双-condition-环形数组)
  - [A3. 线程安全单例模式（DCL / 静态内部类 / 枚举 + 防破坏）](#a3-线程安全单例模式dcl--静态内部类--枚举--防破坏)
  - [A4. 经典生产者-消费者模型（BlockingQueue & wait/notifyAll + 优雅停机）](#a4-经典生产者-消费者模型blockingqueue--waitnotifyall--优雅停机)
  - [A5. 手写死锁 + 线上排查步骤 + 3 种修复方案](#a5-手写死锁--线上排查步骤--3-种修复方案)
  - [A6. 手写 Java 限流器（令牌桶 Token Bucket & 滑动窗口 Sliding Window）](#a6-手写-java-限流器令牌桶-token-bucket--滑动窗口-sliding-window)
- [Part B：数据结构与 Hot 100 高频算法（26 题精讲）](#part-b数据结构与-hot-100-高频算法26-题精讲)
  - [B1. LRU 缓存机制（LeetCode 146 — 双向链表 + HashMap）](#b1-lru-缓存机制leetcode-146--双向链表--hashmap)
  - [B2. LFU 缓存机制（LeetCode 460 — O(1) 双哈希 + 频次链表）](#b2-lfu-缓存机制leetcode-460--o1-双哈希--频次链表)
  - [B3. 反转链表 & 区间反转链表（LeetCode 206 / 92）](#b3-反转链表--区间反转链表leetcode-206--92)
  - [B4. K 个一组翻转链表（LeetCode 25）](#b4-k-个一组翻转链表leetcode-25)
  - [B5. 环形链表 II — 找入环第一个节点（LeetCode 142）](#b5-环形链表-ii--找入环第一个节点leetcode-142)
  - [B6. 合并 K 个升序链表（LeetCode 23 — 最小堆 & 分治归并）](#b6-合并-k-个升序链表leetcode-23--最小堆--分治归并)
  - [B7. 两数之和 & 三数之和（LeetCode 1 / 15）](#b7-两数之和--三数之和leetcode-1--15)
  - [B8. 无重复字符的最长子串（LeetCode 3 — 滑动窗口）](#b8-无重复字符的最长子串leetcode-3--滑动窗口)
  - [B9. 接雨水（LeetCode 42 — 双指针 O(1) 空间 & 单调栈）](#b9-接雨水leetcode-42--双指针-o1-空间--单调栈)
  - [B10. 二叉树的层序遍历 & 锯齿形层序遍历（LeetCode 102 / 103）](#b10-二叉树的层序遍历--锯齿形层序遍历leetcode-102--103)
  - [B11. 二叉树的最近公共祖先 LCA（LeetCode 236）](#b11-二叉树的最近公共祖先-lcaleetcode-236)
  - [B12. 验证二叉搜索树（LeetCode 98）](#b12-验证二叉搜索树leetcode-98)
  - [B13. 手写快速排序 & 数组中第 K 个最大元素（LeetCode 215）](#b13-手写快速排序--数组中第-k-个最大元素leetcode-215)
  - [B14. 手写归并排序（稳定排序 & 逆序对计数）](#b14-手写归并排序稳定排序--逆序对计数)
  - [B15. 手写堆排序（自底向上 O(N) 建堆 & 下沉调整）](#b15-手写堆排序自底向上-on-建堆--下沉调整)
  - [B16. 二分查找全套变体（标准 / 左右边界 / 旋转数组 LeetCode 704 / 34 / 33）](#b16-二分查找全套变体标准--左右边界--旋转数组-leetcode-704--34--33)
  - [B17. 岛屿数量（LeetCode 200 — DFS 沉岛 & BFS 队列）](#b17-岛屿数量leetcode-200--dfs-沉岛--bfs-队列)
  - [B18. 全排列 & 子集 — 回溯万能模板（LeetCode 46 / 78）](#b18-全排列--子集--回溯万能模板leetcode-46--78)
  - [B19. 爬楼梯 & 打家劫舍（LeetCode 70 / 198）](#b19-爬楼梯--打家劫舍leetcode-70--198)
  - [B20. 零钱兑换 & 零钱兑换 II（LeetCode 322 / 518 — 完全背包）](#b20-零钱兑换--零钱兑换-iileetcode-322--518--完全背包)
  - [B21. 0-1 背包标准模板 & 分割等和子集（LeetCode 416）](#b21-0-1-背包标准模板--分割等和子集leetcode-416)
  - [B22. 最长递增子序列 LIS（LeetCode 300 — O(N²) & O(N log N)）](#b22-最长递增子序列-lisleetcode-300--on²--on-log-n)
  - [B23. 编辑距离（LeetCode 72）](#b23-编辑距离leetcode-72)
  - [B24. 最长回文子串（LeetCode 5 — 中心扩展法）](#b24-最长回文子串leetcode-5--中心扩展法)
  - [B25. 每日温度（LeetCode 739 — 单调栈模板）](#b25-每日温度leetcode-739--单调栈模板)
  - [B26. 实现 Trie 前缀树（LeetCode 208）](#b26-实现-trie-前缀树leetcode-208)

---

## Part A：Java 并发编程手撕代码（6 大必考）

### A1. 多线程交替打印 ABC（3 种经典写法 + 奇偶数打印）

**题目描述**：启动三个线程 A、B、C，要求按顺序交替打印 `ABCABC...` 共 10 轮。

**核心思路**：考察线程间通信与精准唤醒。
1. **`ReentrantLock + 多 Condition`（最优，精准唤醒下一个线程，无惊群效应）**。
2. **`Semaphore`（代码最简洁，信号量接力传递许可）**。
3. **`synchronized + wait/notifyAll`（最基础，需注意 `while` 防虚假唤醒，用 `notifyAll` 防止 `notify` 随机唤醒导致死锁）**。

```java
import java.util.concurrent.Semaphore;
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.ReentrantLock;

public class AlternatePrintABC {

    // ==================== 写法一：ReentrantLock + 多 Condition（推荐） ====================
    static class LockConditionPrinter {
        private final ReentrantLock lock = new ReentrantLock();
        private final Condition condA = lock.newCondition();
        private final Condition condB = lock.newCondition();
        private final Condition condC = lock.newCondition();
        private int state = 0; // 0->A, 1->B, 2->C
        private final int rounds;

        LockConditionPrinter(int rounds) { this.rounds = rounds; }

        public void print(String name, int targetState, Condition current, Condition next) {
            for (int i = 0; i < rounds; i++) {
                lock.lock();
                try {
                    while (state % 3 != targetState) {
                        current.await();
                    }
                    System.out.print(name);
                    state++;
                    next.signal(); // 精准唤醒下一个线程
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                    return;
                } finally {
                    lock.unlock();
                }
            }
        }
    }

    // ==================== 写法二：Semaphore 信号量接力（最简） ====================
    static class SemaphorePrinter {
        private final Semaphore semA = new Semaphore(1); // A 先拿许可
        private final Semaphore semB = new Semaphore(0);
        private final Semaphore semC = new Semaphore(0);
        private final int rounds;

        SemaphorePrinter(int rounds) { this.rounds = rounds; }

        public void print(String name, Semaphore current, Semaphore next) {
            for (int i = 0; i < rounds; i++) {
                try {
                    current.acquire();
                    System.out.print(name);
                    next.release();
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                    return;
                }
            }
        }
    }

    // ==================== 写法三：synchronized + wait / notifyAll ====================
    static class SyncPrinter {
        private int state = 0;
        private final int rounds;

        SyncPrinter(int rounds) { this.rounds = rounds; }

        public synchronized void print(String name, int targetState) {
            for (int i = 0; i < rounds; i++) {
                while (state % 3 != targetState) { // 必须用 while 防止虚假唤醒
                    try {
                        wait();
                    } catch (InterruptedException e) {
                        Thread.currentThread().interrupt();
                        return;
                    }
                }
                System.out.print(name);
                state++;
                notifyAll(); // 不能用 notify()，否则可能唤醒错线程导致全部阻塞死锁
            }
        }
    }

    public static void main(String[] args) {
        LockConditionPrinter p1 = new LockConditionPrinter(10);
        new Thread(() -> p1.print("A", 0, p1.condA, p1.condB)).start();
        new Thread(() -> p1.print("B", 1, p1.condB, p1.condC)).start();
        new Thread(() -> p1.print("C", 2, p1.condC, p1.condA)).start();
    }
}
```
- **边界与追问**：如果改为两线程交替打印 1~100 奇偶数，只需把状态判断改为 `(num & 1) == targetParity` 且 `while (num <= 100)`，注意最后一次打印完 `100` 后要确保另一个等待线程退出循环不卡死（`while (num <= 100 && (num & 1) != parity) wait(); if (num <= 100) ...; notifyAll();`）。

---

### A2. 手写有界阻塞队列（ReentrantLock + 双 Condition 环形数组）

**题目描述**：仿照 `ArrayBlockingQueue` 手写一个线程安全的有界阻塞队列，支持阻塞 `put` / `take` 和超时 `offer`。

**核心思路**：环形数组 + `head`/`tail`/`count` + 单把 `ReentrantLock` 配合两个条件队列 `notFull` 和 `notEmpty`。
- **关键不变式**：判断队列满/空**必须用 `while` 循环**（不能用 `if`），防止多消费者/多生产者环境下被唤醒时队列状态已被其他线程抢先改变或发生虚假唤醒。

```java
import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.ReentrantLock;

public class MyBoundedBlockingQueue<E> {
    private final Object[] items;
    private int head = 0;
    private int tail = 0;
    private int count = 0;

    private final ReentrantLock lock = new ReentrantLock();
    private final Condition notEmpty = lock.newCondition();
    private final Condition notFull = lock.newCondition();

    public MyBoundedBlockingQueue(int capacity) {
        if (capacity <= 0) {
            throw new IllegalArgumentException("Capacity must be positive");
        }
        this.items = new Object[capacity];
    }

    public void put(E e) throws InterruptedException {
        if (e == null) throw new NullPointerException();
        lock.lockInterruptibly();
        try {
            while (count == items.length) {
                notFull.await();
            }
            enqueue(e);
        } finally {
            lock.unlock();
        }
    }

    public boolean offer(E e, long timeout, TimeUnit unit) throws InterruptedException {
        if (e == null) throw new NullPointerException();
        long nanos = unit.toNanos(timeout);
        lock.lockInterruptibly();
        try {
            while (count == items.length) {
                if (nanos <= 0L) {
                    return false;
                }
                nanos = notFull.awaitNanos(nanos);
            }
            enqueue(e);
            return true;
        } finally {
            lock.unlock();
        }
    }

    @SuppressWarnings("unchecked")
    public E take() throws InterruptedException {
        lock.lockInterruptibly();
        try {
            while (count == 0) {
                notEmpty.await();
            }
            return dequeue();
        } finally {
            lock.unlock();
        }
    }

    private void enqueue(E e) {
        items[tail] = e;
        if (++tail == items.length) tail = 0;
        count++;
        notEmpty.signal();
    }

    @SuppressWarnings("unchecked")
    private E dequeue() {
        E x = (E) items[head];
        items[head] = null; // Help GC，防止内存泄漏！
        if (++head == items.length) head = 0;
        count--;
        notFull.signal();
        return x;
    }

    public int size() {
        lock.lock();
        try {
            return count;
        } finally {
            lock.unlock();
        }
    }
}
```
- **面试追问**：为什么 `LinkedBlockingQueue` 用两把锁（`putLock` + `takeLock`）而 `ArrayBlockingQueue` 用一把锁？因为数组内存连续且固定，读写共用索引/数组状态竞争较少且不需要额外 Node 对象分配；而链表需要独立维护 `head` 和 `last` 节点，双锁配合 `AtomicInteger count` 可实现读写真正并行。

---

### A3. 线程安全单例模式（DCL / 静态内部类 / 枚举 + 防破坏）

**题目描述**：手写线程安全的懒汉单例，解释 DCL 为何必须加 `volatile`，以及如何防御反射与反序列化破坏。

```java
import java.io.Serializable;

public class SingletonPatterns {

    // 1. 双重检查锁 (DCL) — 最常考源码原理
    public static final class DclSingleton implements Serializable {
        // 必须加 volatile：禁止 new DclSingleton() 的指令重排序！
        private static volatile DclSingleton instance;

        private DclSingleton() {
            // 防反射破坏：若已实例化则拒绝再次调用构造器
            if (instance != null) {
                throw new IllegalStateException("Singleton already initialized");
            }
        }

        public static DclSingleton getInstance() {
            if (instance == null) {                     // 第一次检查：无锁快速路径
                synchronized (DclSingleton.class) {
                    if (instance == null) {             // 第二次检查：防并发重复创建
                        instance = new DclSingleton();  // 1.分配内存 2.初始化对象 3.引用赋值
                    }
                }
            }
            return instance;
        }

        // 防反序列化生成新实例
        private Object readResolve() {
            return getInstance();
        }
    }

    // 2. 静态内部类 (Initialization-on-demand holder) — 懒加载 + 零锁开销
    public static final class HolderSingleton {
        private HolderSingleton() {}

        private static class Holder {
            // JVM 保证类的 <clinit>() 初始化过程天生加锁且只执行一次
            private static final HolderSingleton INSTANCE = new HolderSingleton();
        }

        public static HolderSingleton getInstance() {
            return Holder.INSTANCE;
        }
    }

    // 3. 枚举单例 — 《Effective Java》最推荐，天然防反射 & 防反序列化
    public enum EnumSingleton {
        INSTANCE;

        public void doWork() {
            System.out.println("Singleton working...");
        }
    }
}
```
- **追问必答**：`new DclSingleton()` 在字节码层面分 3 步：① 分配内存；② 调用构造器初始化字段；③ 将 `instance` 引用指向内存地址。如果不加 `volatile`，JIT 编译器/CPU 可能将 ② 和 ③ 重排，导致线程 B 在外层 `if (instance == null)` 看到非空引用，直接拿走一个**字段尚未初始化的半成品对象**！

---

### A4. 经典生产者-消费者模型（BlockingQueue & wait/notifyAll + 优雅停机）

**题目描述**：手写多生产者多消费者模型，并说明如何在主线程通知关闭时让生产者和消费者优雅退出（不丢队列残留数据）。

```java
import java.util.LinkedList;
import java.util.Queue;
import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.BlockingQueue;
import java.util.concurrent.atomic.AtomicInteger;

public class ProducerConsumerDemo {

    // 写法一：基于 wait/notifyAll 手写缓冲池
    static class WaitNotifyBuffer<T> {
        private final Queue<T> queue = new LinkedList<>();
        private final int capacity;

        WaitNotifyBuffer(int capacity) { this.capacity = capacity; }

        public synchronized void produce(T item) throws InterruptedException {
            while (queue.size() == capacity) {
                wait();
            }
            queue.offer(item);
            notifyAll();
        }

        public synchronized T consume() throws InterruptedException {
            while (queue.isEmpty()) {
                wait();
            }
            T item = queue.poll();
            notifyAll();
            return item;
        }
    }

    // 写法二：生产级 BlockingQueue + 毒丸 (Poison Pill) 优雅停机
    private static final String POISON_PILL = "__SHUTDOWN__";

    public static void runGracefulDemo() throws InterruptedException {
        BlockingQueue<String> queue = new ArrayBlockingQueue<>(16);
        AtomicInteger idGen = new AtomicInteger(1);

        Thread producer = new Thread(() -> {
            try {
                for (int i = 0; i < 10; i++) {
                    queue.put("Task-" + idGen.getAndIncrement());
                }
                queue.put(POISON_PILL); // 投递毒丸，通知消费者消费完剩余任务后退出
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }, "Producer");

        Thread consumer = new Thread(() -> {
            try {
                while (true) {
                    String task = queue.take();
                    if (POISON_PILL.equals(task)) {
                        break; // 若有多个消费者，可再放回一个毒丸或按消费者数量投递
                    }
                    System.out.println("Consumed: " + task);
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }, "Consumer");

        producer.start();
        consumer.start();
        producer.join();
        consumer.join();
    }
}
```

---

### A5. 手写死锁 + 线上排查步骤 + 3 种修复方案

**题目描述**：手写一个必然发生死锁的 Java 程序，并说明线上遇到死锁如何排查以及从代码层如何修复。

```java
import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.ReentrantLock;

public class DeadlockAndFix {
    private static final Object LOCK_A = new Object();
    private static final Object LOCK_B = new Object();

    // 1. 必现死锁示例（交叉持锁，破坏“循环等待”的反面教材）
    public static void triggerDeadlock() {
        new Thread(() -> {
            synchronized (LOCK_A) {
                sleepQuietly(100);
                synchronized (LOCK_B) {
                    System.out.println("Thread-1 acquired both");
                }
            }
        }, "Deadlock-T1").start();

        new Thread(() -> {
            synchronized (LOCK_B) {
                sleepQuietly(100);
                synchronized (LOCK_A) {
                    System.out.println("Thread-2 acquired both");
                }
            }
        }, "Deadlock-T2").start();
    }

    // 2. 修复方案一：按全局一致的顺序加锁（比如按对象 hashCode / 业务主键 ID 升序）
    public static void transferSafeOrder(Object lock1, Object lock2) {
        int h1 = System.identityHashCode(lock1);
        int h2 = System.identityHashCode(lock2);
        Object first = h1 <= h2 ? lock1 : lock2;
        Object second = h1 <= h2 ? lock2 : lock1;

        synchronized (first) {
            synchronized (second) {
                // 当 h1 == h2 极小概率哈希冲突时，可再加一把全局 tieLock 兜底
                System.out.println("Safe ordered execution");
            }
        }
    }

    // 3. 修复方案二：使用 ReentrantLock.tryLock(timeout) 破坏“不可剥夺”条件
    public static boolean tryAcquireBoth(ReentrantLock r1, ReentrantLock r2) throws InterruptedException {
        while (true) {
            if (r1.tryLock(50, TimeUnit.MILLISECONDS)) {
                try {
                    if (r2.tryLock(50, TimeUnit.MILLISECONDS)) {
                        try {
                            return true; // 业务操作
                        } finally {
                            r2.unlock();
                        }
                    }
                } finally {
                    r1.unlock(); // 拿不到第二把锁则主动释放第一把锁并随机退避
                }
            }
            TimeUnit.MILLISECONDS.sleep(10);
        }
    }

    private static void sleepQuietly(long ms) {
        try { Thread.sleep(ms); } catch (InterruptedException ignored) { Thread.currentThread().interrupt(); }
    }
}
```
- **线上死锁排查命令**：
  1. `jps -l` 找到 Java 进程 PID。
  2. `jstack -l <pid>` 导出线程栈，末尾会明确打印 `Found one Java-level deadlock:` 以及相互等待的锁地址和代码行号。
  3. 或使用阿里 **Arthas**：`thread -b` 一键找出当前阻塞其他线程的罪魁祸首线程，或 `thread --state BLOCKED`。

---

### A6. 手写 Java 限流器（令牌桶 Token Bucket & 滑动窗口 Sliding Window）

**题目描述**：不依赖 Guava/Sentinel，手写线程安全的单机令牌桶限流器与滑动窗口限流器。

**核心思路**：
- **令牌桶（惰性补充，无需额外后台定时线程）**：记录上次补充时间 `lastRefillNanos` 和当前余量 `tokens`，每次 `tryAcquire()` 时先根据经过时间补充令牌 `elapsed * rate`，不超过桶容量 `capacity`，够扣减则返回 `true`。
- **滑动窗口**：维护一个存储请求时间戳的队列 `ArrayDeque<Long>`，每次请求先清理窗口左侧过期的时间戳，再判断队列大小是否达到阈值。

```java
import java.util.ArrayDeque;
import java.util.Deque;

public class RateLimiters {

    // 1. 线程安全令牌桶限流器（惰性补充，O(1) 时间与空间）
    public static class TokenBucketRateLimiter {
        private final double capacity;
        private final double refillRatePerSec;
        private double tokens;
        private long lastRefillNanos;

        public TokenBucketRateLimiter(double capacity, double refillRatePerSec) {
            this.capacity = capacity;
            this.refillRatePerSec = refillRatePerSec;
            this.tokens = capacity;
            this.lastRefillNanos = System.nanoTime();
        }

        public synchronized boolean tryAcquire(int permits) {
            long now = System.nanoTime();
            double elapsedSec = (now - lastRefillNanos) / 1_000_000_000.0;
            if (elapsedSec > 0) {
                tokens = Math.min(capacity, tokens + elapsedSec * refillRatePerSec);
                lastRefillNanos = now;
            }
            if (tokens >= permits) {
                tokens -= permits;
                return true;
            }
            return false;
        }
    }

    // 2. 线程安全滑动时间窗口限流器
    public static class SlidingWindowRateLimiter {
        private final int maxRequests;
        private final long windowMillis;
        private final Deque<Long> timestamps = new ArrayDeque<>();

        public SlidingWindowRateLimiter(int maxRequests, long windowMillis) {
            this.maxRequests = maxRequests;
            this.windowMillis = windowMillis;
        }

        public synchronized boolean tryAcquire() {
            long now = System.currentTimeMillis();
            long boundary = now - windowMillis;
            while (!timestamps.isEmpty() && timestamps.peekFirst() <= boundary) {
                timestamps.pollFirst();
            }
            if (timestamps.size() < maxRequests) {
                timestamps.offerLast(now);
                return true;
            }
            return false;
        }
    }
}
```
- **追问**：为什么单机高频限流下，如果 `maxRequests` 非常大（如每秒 10 万次），用 `Deque<Long>` 存每次请求时间戳内存会炸？怎么优化？
  - 将 1 秒切分成 10 个 100ms 的**环形计数桶数组 `int[10]`**，每个桶只累加计数，内存恒定 $O(\text{桶数})$！

---

## Part B：数据结构与 Hot 100 高频算法（26 题精讲）

### B1. LRU 缓存机制（LeetCode 146 — 双向链表 + HashMap）

**题目描述**：设计并实现满足 LRU（最近最少使用）缓存约束的数据结构，`get(key)` 和 `put(key, value)` 均要求 **$O(1)$ 时间复杂度**。
**核心思路**：**面试时绝对不要只写 `extends LinkedHashMap` 交差**，必须手写**哈希表 + 带虚拟头尾哨兵节点的双向链表**！双向链表维护访问时序（头部最新，尾部最久未用），哈希表实现 $O(1)$ 查找定位节点。

```java
import java.util.HashMap;
import java.util.Map;

public class LRUCache {
    private static class Node {
        int key, val;
        Node prev, next;
        Node(int key, int val) { this.key = key; this.val = val; }
    }

    private final int capacity;
    private final Map<Integer, Node> map = new HashMap<>();
    private final Node head = new Node(0, 0); // 哨兵头（最新）
    private final Node tail = new Node(0, 0); // 哨兵尾（最久未用）

    public LRUCache(int capacity) {
        this.capacity = capacity;
        head.next = tail;
        tail.prev = head;
    }

    public int get(int key) {
        Node node = map.get(key);
        if (node == null) return -1;
        moveToHead(node);
        return node.val;
    }

    public void put(int key, int value) {
        Node node = map.get(key);
        if (node != null) {
            node.val = value;
            moveToHead(node);
            return;
        }
        if (map.size() == capacity) {
            Node lru = tail.prev;
            removeNode(lru);
            map.remove(lru.key);
        }
        Node newNode = new Node(key, value);
        map.put(key, newNode);
        addFirst(newNode);
    }

    private void moveToHead(Node node) {
        removeNode(node);
        addFirst(node);
    }

    private void addFirst(Node node) {
        node.prev = head;
        node.next = head.next;
        head.next.prev = node;
        head.next = node;
    }

    private void removeNode(Node node) {
        node.prev.next = node.next;
        node.next.prev = node.prev;
    }
}
```
- **复杂度**：`get` / `put` 均摊时间 $O(1)$；空间 $O(\text{capacity})$。
- **边界与追问**：1）为什么节点里也要存 `key`？因为淘汰尾部节点 `lru` 时需要从 `map.remove(lru.key)` 删除；2）高并发下如何改造成线程安全？（全局 `ReentrantReadWriteLock` 不行，因为 `get` 也要改链表结构；应采用**分段锁 `ConcurrentHashMap` 思想分 16 个 LRU 分片**，或 Caffeine 的读缓冲异步重放思想）。

---

### B2. LFU 缓存机制（LeetCode 460 — O(1) 双哈希 + 频次链表）

**题目描述**：实现最不经常使用（LFU）缓存；当存在多个相同最低访问频次时，淘汰其中**最久未使用**的键。`get` 与 `put` 均要求 $O(1)$。
**核心思路**：
- `keyMap`: `key -> Node(key, val, freq)`。
- `freqMap`: `freq -> DoublyLinkedList`（同频次内按时间排序，新插入放头部，淘汰时删尾部）。
- 维护全局 `minFreq` 变量：当某节点频次从 `f` 升为 `f+1` 且原 `f` 链表变空且 `f == minFreq` 时，`minFreq++`；插入全新节点时 `minFreq = 1`。

```java
import java.util.HashMap;
import java.util.Map;

public class LFUCache {
    private static class Node {
        int key, val, freq = 1;
        Node prev, next;
        Node(int key, int val) { this.key = key; this.val = val; }
    }

    private static class DList {
        final Node head = new Node(0, 0);
        final Node tail = new Node(0, 0);
        int size = 0;
        DList() { head.next = tail; tail.prev = head; }

        void addFirst(Node n) {
            n.prev = head; n.next = head.next;
            head.next.prev = n; head.next = n;
            size++;
        }
        void remove(Node n) {
            n.prev.next = n.next; n.next.prev = n.prev;
            size--;
        }
        Node removeLast() {
            if (size == 0) return null;
            Node last = tail.prev;
            remove(last);
            return last;
        }
    }

    private final int capacity;
    private int minFreq = 0;
    private final Map<Integer, Node> keyMap = new HashMap<>();
    private final Map<Integer, DList> freqMap = new HashMap<>();

    public LFUCache(int capacity) { this.capacity = capacity; }

    public int get(int key) {
        Node node = keyMap.get(key);
        if (node == null) return -1;
        increaseFreq(node);
        return node.val;
    }

    public void put(int key, int value) {
        if (capacity <= 0) return;
        Node node = keyMap.get(key);
        if (node != null) {
            node.val = value;
            increaseFreq(node);
            return;
        }
        if (keyMap.size() == capacity) {
            DList minList = freqMap.get(minFreq);
            Node evicted = minList.removeLast();
            keyMap.remove(evicted.key);
        }
        Node newNode = new Node(key, value);
        keyMap.put(key, newNode);
        freqMap.computeIfAbsent(1, k -> new DList()).addFirst(newNode);
        minFreq = 1;
    }

    private void increaseFreq(Node node) {
        int oldFreq = node.freq;
        DList oldList = freqMap.get(oldFreq);
        oldList.remove(node);
        if (oldFreq == minFreq && oldList.size == 0) {
            minFreq++;
        }
        node.freq++;
        freqMap.computeIfAbsent(node.freq, k -> new DList()).addFirst(node);
    }
}
```
- **复杂度**：时间 $O(1)$，空间 $O(\text{capacity})$。注意 `capacity == 0` 的边界处理！

---

### B3. 反转链表 & 区间反转链表（LeetCode 206 / 92）

**核心思路**：三指针迭代反转（`prev`, `curr`, `next`）；区间反转 `[left, right]` 使用**哑节点 `dummy` + 头插法**一次遍历即可完成。

```java
public class ReverseLinkedListSolution {
    public static class ListNode {
        int val; ListNode next;
        ListNode(int val) { this.val = val; }
    }

    // LeetCode 206: 迭代反转全链表
    public ListNode reverseList(ListNode head) {
        ListNode prev = null, curr = head;
        while (curr != null) {
            ListNode next = curr.next;
            curr.next = prev;
            prev = curr;
            curr = next;
        }
        return prev;
    }

    // LeetCode 92: 反转位置 left 到 right 之间的节点（一次遍历头插法）
    public ListNode reverseBetween(ListNode head, int left, int right) {
        ListNode dummy = new ListNode(0);
        dummy.next = head;
        ListNode pre = dummy;
        for (int i = 1; i < left; i++) {
            pre = pre.next;
        }
        ListNode curr = pre.next;
        for (int i = 0; i < right - left; i++) {
            ListNode next = curr.next;
            curr.next = next.next;
            next.next = pre.next;
            pre.next = next;
        }
        return dummy.next;
    }
}
```
- **复杂度**：时间 $O(N)$，空间 $O(1)$。

---

### B4. K 个一组翻转链表（LeetCode 25）

**题目描述**：每 `k` 个节点一组进行翻转，如果节点总数不是 `k` 的整数倍，最后剩余节点保持原有顺序。
**核心思路**：使用 `dummy` 节点，每次先用 `end` 指针往前探测 `k` 步；若不足 `k` 步直接 `break`；否则记录 `nextGroup = end.next`，断开 `end.next = null` 局部翻转后再接回前后指针。

```java
public class ReverseKGroupSolution {
    public ListNode reverseKGroup(ListNode head, int k) {
        ListNode dummy = new ListNode(0);
        dummy.next = head;
        ListNode pre = dummy, end = dummy;

        while (end.next != null) {
            for (int i = 0; i < k && end != null; i++) {
                end = end.next;
            }
            if (end == null) break; // 不足 k 个保持原样

            ListNode start = pre.next;
            ListNode nextGroup = end.next;
            end.next = null; // 断链进行局部反转

            pre.next = reverse(start);
            start.next = nextGroup; // 原先的 start 反转后变成了本组尾巴

            pre = start;
            end = start;
        }
        return dummy.next;
    }

    private ListNode reverse(ListNode head) {
        ListNode prev = null, curr = head;
        while (curr != null) {
            ListNode next = curr.next;
            curr.next = prev;
            prev = curr;
            curr = next;
        }
        return prev;
    }
}
```
- **复杂度**：时间 $O(N)$，空间 $O(1)$。

---

### B5. 环形链表 II — 找入环第一个节点（LeetCode 142）

**核心数学推导**：设起点到入环点距离为 $a$，入环点到相遇点为 $b$，相遇点再走 $c$ 回到入环点（环长 $L = b + c$）。
快指针走 $2(a+b) = a + b + k(b+c) \implies a = (k-1)(b+c) + c$。
因此相遇后，将一个指针放回 `head`，两指针同时每次走 1 步，必然在**入环口**相遇！

```java
public ListNode detectCycle(ListNode head) {
    ListNode slow = head, fast = head;
    while (fast != null && fast.next != null) {
        slow = slow.next;
        fast = fast.next.next;
        if (slow == fast) {
            ListNode ptr = head;
            while (ptr != slow) {
                ptr = ptr.next;
                slow = slow.next;
            }
            return ptr;
        }
    }
    return null;
}
```
- **复杂度**：时间 $O(N)$，空间 $O(1)$。

---

### B6. 合并 K 个升序链表（LeetCode 23 — 最小堆 & 分治归并）

**核心思路**：最小堆维护 K 个链表的当前头节点（时间 $O(N \log K)$，空间 $O(K)$）；或者两两分治归并（时间 $O(N \log K)$，递归栈空间 $O(\log K)$）。

```java
import java.util.PriorityQueue;

public class MergeKListsSolution {
    public ListNode mergeKLists(ListNode[] lists) {
        if (lists == null || lists.length == 0) return null;
        PriorityQueue<ListNode> pq = new PriorityQueue<>((a, b) -> Integer.compare(a.val, b.val));
        for (ListNode node : lists) {
            if (node != null) pq.offer(node);
        }
        ListNode dummy = new ListNode(0), tail = dummy;
        while (!pq.isEmpty()) {
            ListNode min = pq.poll();
            tail.next = min;
            tail = min;
            if (min.next != null) {
                pq.offer(min.next);
            }
        }
        return dummy.next;
    }
}
```
- **边界**：注意 `lists` 为空数组或包含 `null` 元素的链表！比较器用 `Integer.compare` 防大整数相减溢出。

---

### B7. 两数之和 & 三数之和（LeetCode 1 / 15）

**核心思路**：
- **两数之和**：`HashMap` 一遍扫描记录 `值 -> 下标`。
- **三数之和**：**排序 + 固定第一位 + 双指针夹逼**。面试官重点看 **去重细节**（`i > 0 && nums[i] == nums[i-1]` 跳过；找到一组解后 `left`/`right` 也要跳过重复值）。

```java
import java.util.*;

public class TwoAndThreeSum {
    public int[] twoSum(int[] nums, int target) {
        Map<Integer, Integer> map = new HashMap<>();
        for (int i = 0; i < nums.length; i++) {
            int need = target - nums[i];
            if (map.containsKey(need)) {
                return new int[]{map.get(need), i};
            }
            map.put(nums[i], i);
        }
        return new int[0];
    }

    public List<List<Integer>> threeSum(int[] nums) {
        List<List<Integer>> res = new ArrayList<>();
        if (nums == null || nums.length < 3) return res;
        Arrays.sort(nums);
        for (int i = 0; i < nums.length - 2; i++) {
            if (nums[i] > 0) break; // 排序后最小值 > 0，不可能凑出 0
            if (i > 0 && nums[i] == nums[i - 1]) continue; // 外层去重
            int l = i + 1, r = nums.length - 1;
            while (l < r) {
                int sum = nums[i] + nums[l] + nums[r];
                if (sum == 0) {
                    res.add(Arrays.asList(nums[i], nums[l], nums[r]));
                    while (l < r && nums[l] == nums[l + 1]) l++; // 内层左去重
                    while (l < r && nums[r] == nums[r - 1]) r--; // 内层右去重
                    l++;
                    r--;
                } else if (sum < 0) {
                    l++;
                } else {
                    r--;
                }
            }
        }
        return res;
    }
}
```
- **复杂度**：`twoSum` 时间 $O(N)$、空间 $O(N)$；`threeSum` 时间 $O(N^2)$、排序空间 $O(\log N)$。

---

### B8. 无重复字符的最长子串（LeetCode 3 — 滑动窗口）

**核心思路**：滑动窗口 `[left, right]`，用数组/HashMap 记录每个字符上一次出现的下标；遇到重复字符时，`left` 直接跳跃到 `Math.max(left, lastIndex + 1)`（注意取 `max` 防止 `left` 倒退！）。

```java
import java.util.Arrays;

public int lengthOfLongestSubstring(String s) {
    int[] lastIdx = new int[128]; // ASCII 字符集直接用数组替代 HashMap，性能翻倍
    Arrays.fill(lastIdx, -1);
    int left = 0, maxLen = 0;
    for (int right = 0; right < s.length(); right++) {
        char c = s.charAt(right);
        if (lastIdx[c] >= left) {
            left = lastIdx[c] + 1;
        }
        lastIdx[c] = right;
        maxLen = Math.max(maxLen, right - left + 1);
    }
    return maxLen;
}
```
- **复杂度**：时间 $O(N)$，空间 $O(1)$（固定大小 128；若包含 Unicode 中文可改用 `HashMap<Character, Integer>`）。

---

### B9. 接雨水（LeetCode 42 — 双指针 O(1) 空间 & 单调栈）

**核心思路**：每个柱子能接的雨水 = `min(左侧最高, 右侧最高) - height[i]`。双指针维护 `leftMax` 和 `rightMax`：谁小就先结算哪一侧，因为另一侧至少有一个更高的挡板在！

```java
public int trap(int[] height) {
    if (height == null || height.length < 3) return 0;
    int left = 0, right = height.length - 1;
    int leftMax = 0, rightMax = 0, ans = 0;
    while (left < right) {
        leftMax = Math.max(leftMax, height[left]);
        rightMax = Math.max(rightMax, height[right]);
        if (leftMax < rightMax) {
            ans += leftMax - height[left];
            left++;
        } else {
            ans += rightMax - height[right];
            right--;
        }
    }
    return ans;
}
```
- **复杂度**：时间 $O(N)$，空间 $O(1)$（优于前后缀数组 $O(N)$ 空间）。

---

### B10. 二叉树的层序遍历 & 锯齿形层序遍历（LeetCode 102 / 103）

**核心思路**：BFS 队列，外层循环每次读取 `size = queue.size()` 控制当前层节点数；锯齿形遍历只需用 `LinkedList` 在奇数层 `addFirst`、偶数层 `addLast`，无需对结果 `Collections.reverse()`。

```java
import java.util.*;

public class TreeLevelOrder {
    public static class TreeNode {
        int val; TreeNode left, right;
        TreeNode(int val) { this.val = val; }
    }

    public List<List<Integer>> zigzagLevelOrder(TreeNode root) {
        List<List<Integer>> res = new ArrayList<>();
        if (root == null) return res;
        Queue<TreeNode> queue = new ArrayDeque<>();
        queue.offer(root);
        boolean leftToRight = true;

        while (!queue.isEmpty()) {
            int size = queue.size();
            LinkedList<Integer> level = new LinkedList<>();
            for (int i = 0; i < size; i++) {
                TreeNode node = queue.poll();
                if (leftToRight) level.addLast(node.val);
                else level.addFirst(node.val);

                if (node.left != null) queue.offer(node.left);
                if (node.right != null) queue.offer(node.right);
            }
            res.add(level);
            leftToRight = !leftToRight;
        }
        return res;
    }
}
```
- **复杂度**：时间 $O(N)$，空间 $O(N)$。

---

### B11. 二叉树的最近公共祖先 LCA（LeetCode 236）

**核心思路**：后序递归。若 `root == null || root == p || root == q`，直接返回 `root`。分别向左右子树查找：若左右都非空，说明 `p` 和 `q` 分别在左右两侧，当前 `root` 即为最近公共祖先；否则返回非空的那一侧。

```java
public TreeNode lowestCommonAncestor(TreeNode root, TreeNode p, TreeNode q) {
    if (root == null || root == p || root == q) {
        return root;
    }
    TreeNode left = lowestCommonAncestor(root.left, p, q);
    TreeNode right = lowestCommonAncestor(root.right, p, q);
    if (left != null && right != null) {
        return root;
    }
    return left != null ? left : right;
}
```
- **追问变体**：如果是**二叉搜索树（BST，LeetCode 235）**，利用大小关系只需 $O(H)$ 时间 $O(1)$ 空间迭代即可（若 `p.val < root.val && q.val < root.val` 往左走；都大于往右走；分叉或相等即为答案）。

---

### B12. 验证二叉搜索树（LeetCode 98）

**核心陷阱**：不能只判断 `left.val < root.val < right.val`！必须保证**左子树所有节点都小于 `root.val`，右子树所有节点都大于 `root.val`**。边界值要用 `long`（`Long.MIN_VALUE` / `Long.MAX_VALUE`），防止节点值刚好等于 `Integer.MIN_VALUE` 或 `Integer.MAX_VALUE`！

```java
public boolean isValidBST(TreeNode root) {
    return validate(root, Long.MIN_VALUE, Long.MAX_VALUE);
}

private boolean validate(TreeNode node, long minVal, long maxVal) {
    if (node == null) return true;
    if (node.val <= minVal || node.val >= maxVal) return false;
    return validate(node.left, minVal, node.val)
        && validate(node.right, node.val, maxVal);
}
```
- **复杂度**：时间 $O(N)$，空间 $O(H)$。也可用中序遍历严格递增验证。

---

### B13. 手写快速排序 & 数组中第 K 个最大元素（LeetCode 215）

**核心考点**：手写快排**必须随机选基准（Random Pivot）**，避免有序数组退化为 $O(N^2)$！第 K 大元素采用 QuickSelect，均摊 $O(N)$ 时间复杂度（且遇到大量重复元素时推荐**三路划分**或双指针 Hoare 分区防止退化）。

```java
import java.util.concurrent.ThreadLocalRandom;

public class QuickSortAndSelect {
    public void quickSort(int[] nums, int left, int right) {
        if (left >= right) return;
        int pivotIdx = partition(nums, left, right);
        quickSort(nums, left, pivotIdx - 1);
        quickSort(nums, pivotIdx + 1, right);
    }

    // LeetCode 215: 数组中的第 K 个最大元素 -> 即升序排列后的下标 nums.length - k
    public int findKthLargest(int[] nums, int k) {
        int target = nums.length - k;
        int left = 0, right = nums.length - 1;
        while (left <= right) {
            int pivotIdx = partition(nums, left, right);
            if (pivotIdx == target) return nums[pivotIdx];
            if (pivotIdx < target) left = pivotIdx + 1;
            else right = pivotIdx - 1;
        }
        return -1;
    }

    // 随机基准 + 双路划分（有效缓解大量重复元素退化）
    private int partition(int[] nums, int left, int right) {
        int randIdx = ThreadLocalRandom.current().nextInt(left, right + 1);
        swap(nums, left, randIdx);
        int pivot = nums[left];
        int i = left + 1, j = right;
        while (true) {
            while (i <= j && nums[i] < pivot) i++;
            while (i <= j && nums[j] > pivot) j--;
            if (i >= j) break;
            swap(nums, i++, j--);
        }
        swap(nums, left, j);
        return j;
    }

    private void swap(int[] nums, int i, int j) {
        int t = nums[i]; nums[i] = nums[j]; nums[j] = t;
    }
}
```
- **复杂度**：快排平均时间 $O(N \log N)$，空间 $O(\log N)$；QuickSelect 平均时间 $O(N)$，空间 $O(1)$（迭代版）。

---

### B14. 手写归并排序（稳定排序 & 逆序对计数）

**核心思路**：分治拆成两半各自排序再有序合并；归并过程中当 `tmp[i] > tmp[j]` 时，左边剩余 `mid - i + 1` 个元素都与 `tmp[j]` 构成逆序对。

```java
public class MergeSortSolution {
    public void mergeSort(int[] nums) {
        if (nums == null || nums.length < 2) return;
        int[] temp = new int[nums.length]; // 提前分配辅助数组，避免递归内频繁 new
        sort(nums, 0, nums.length - 1, temp);
    }

    private void sort(int[] nums, int left, int right, int[] temp) {
        if (left >= right) return;
        int mid = left + ((right - left) >>> 1);
        sort(nums, left, mid, temp);
        sort(nums, mid + 1, right, temp);
        if (nums[mid] <= nums[mid + 1]) return; // 已有序剪枝
        merge(nums, left, mid, right, temp);
    }

    private void merge(int[] nums, int left, int mid, int right, int[] temp) {
        System.arraycopy(nums, left, temp, left, right - left + 1);
        int i = left, j = mid + 1;
        for (int k = left; k <= right; k++) {
            if (i > mid) nums[k] = temp[j++];
            else if (j > right) nums[k] = temp[i++];
            else if (temp[i] <= temp[j]) nums[k] = temp[i++]; // <= 保证排序稳定性！
            else nums[k] = temp[j++];
        }
    }
}
```
- **复杂度**：时间稳定 $O(N \log N)$，空间 $O(N)$，稳定排序。扩展：链表排序（LeetCode 148）最优解就是归并排序（$O(N \log N)$）。

---

### B15. 手写堆排序（自底向上 O(N) 建堆 & 下沉调整）

**核心思路**：升序排序需要构建**大顶堆**：1）从最后一个非叶子节点 `n / 2 - 1` 自底向上做 `siftDown` 建堆（$O(N)$）；2）反复把堆顶最大值交换到数组末尾并缩减堆大小再 `siftDown`。

```java
public class HeapSortSolution {
    public void heapSort(int[] nums) {
        if (nums == null || nums.length < 2) return;
        int n = nums.length;
        // 1. 自底向上建大顶堆: O(N)
        for (int i = n / 2 - 1; i >= 0; i--) {
            siftDown(nums, i, n);
        }
        // 2. 交换堆顶到末尾并重新下沉: O(N log N)
        for (int end = n - 1; end > 0; end--) {
            swap(nums, 0, end);
            siftDown(nums, 0, end);
        }
    }

    private void siftDown(int[] nums, int i, int heapSize) {
        while (2 * i + 1 < heapSize) {
            int largest = 2 * i + 1; // 左孩子
            int right = largest + 1;
            if (right < heapSize && nums[right] > nums[largest]) {
                largest = right;
            }
            if (nums[i] >= nums[largest]) break;
            swap(nums, i, largest);
            i = largest;
        }
    }

    private void swap(int[] nums, int i, int j) {
        int t = nums[i]; nums[i] = nums[j]; nums[j] = t;
    }
}
```
- **复杂度**：时间 $O(N \log N)$，空间 $O(1)$，不稳定排序。

---

### B16. 二分查找全套变体（标准 / 左右边界 / 旋转数组 LeetCode 704 / 34 / 33）

**核心原则**：统一使用左闭右闭 `[left, right]`，中点 `mid = left + ((right - left) >>> 1)` 防溢出。

```java
public class BinarySearchVariants {
    // 1. 查找目标值第一个位置 (Lower Bound: >= target 的第一个下标)
    public int lowerBound(int[] nums, int target) {
        int left = 0, right = nums.length - 1, ans = nums.length;
        while (left <= right) {
            int mid = left + ((right - left) >>> 1);
            if (nums[mid] >= target) {
                ans = mid;
                right = mid - 1;
            } else {
                left = mid + 1;
            }
        }
        return ans;
    }

    // 2. LeetCode 34: 在排序数组中查找元素的第一个和最后一个位置
    public int[] searchRange(int[] nums, int target) {
        int first = lowerBound(nums, target);
        if (first == nums.length || nums[first] != target) {
            return new int[]{-1, -1};
        }
        int last = lowerBound(nums, target + 1) - 1;
        return new int[]{first, last};
    }

    // 3. LeetCode 33: 搜索旋转排序数组（无重复元素）
    public int searchRotated(int[] nums, int target) {
        int left = 0, right = nums.length - 1;
        while (left <= right) {
            int mid = left + ((right - left) >>> 1);
            if (nums[mid] == target) return mid;
            if (nums[left] <= nums[mid]) { // 左半边有序
                if (nums[left] <= target && target < nums[mid]) right = mid - 1;
                else left = mid + 1;
            } else { // 右半边有序
                if (nums[mid] < target && target <= nums[right]) left = mid + 1;
                else right = mid - 1;
            }
        }
        return -1;
    }
}
```
- **复杂度**：时间均为 $O(\log N)$，空间 $O(1)$。

---

### B17. 岛屿数量（LeetCode 200 — DFS 沉岛 & BFS 队列）

**核心思路**：遇到 `'1'` 计数加 1，随即通过 DFS/BFS 将相邻陆地全部“沉没”为 `'0'`。注意大矩阵下递归 DFS 可能触发 `StackOverflowError`，面试官追问时要能立刻切换为 **BFS 迭代队列**！

```java
import java.util.ArrayDeque;
import java.util.Queue;

public class NumIslandsSolution {
    private static final int[][] DIRS = {{1, 0}, {-1, 0}, {0, 1}, {0, -1}};

    public int numIslands(char[][] grid) {
        if (grid == null || grid.length == 0) return 0;
        int m = grid.length, n = grid[0].length, count = 0;
        for (int i = 0; i < m; i++) {
            for (int j = 0; j < n; j++) {
                if (grid[i][j] == '1') {
                    count++;
                    bfsSink(grid, i, j, m, n);
                }
            }
        }
        return count;
    }

    private void bfsSink(char[][] grid, int r, int c, int m, int n) {
        Queue<int[]> q = new ArrayDeque<>();
        q.offer(new int[]{r, c});
        grid[r][c] = '0'; // 入队即标记沉岛，防止重复入队 OOM/TLE！
        while (!q.isEmpty()) {
            int[] cur = q.poll();
            for (int[] d : DIRS) {
                int nr = cur[0] + d[0], nc = cur[1] + d[1];
                if (nr >= 0 && nr < m && nc >= 0 && nc < n && grid[nr][nc] == '1') {
                    grid[nr][nc] = '0';
                    q.offer(new int[]{nr, nc});
                }
            }
        }
    }
}
```
- **复杂度**：时间 $O(M \times N)$，空间 $O(\min(M, N))$（BFS 最坏队列尺寸）。

---

### B18. 全排列 & 子集 — 回溯万能模板（LeetCode 46 / 78）

**核心区分**：**排列问题**关心顺序，每次从 `0` 开始配合 `boolean[] used`；**子集/组合问题**不关心顺序，用 `startIndex` 控制只往后选避免重复。若有重复元素（LeetCode 47 / 90），先排序再加同层剪枝 `if (i > start && nums[i] == nums[i - 1]) continue;`。

```java
import java.util.*;

public class BacktrackingTemplates {
    // LeetCode 46: 全排列
    public List<List<Integer>> permute(int[] nums) {
        List<List<Integer>> res = new ArrayList<>();
        backtrackPermute(nums, new boolean[nums.length], new ArrayList<>(), res);
        return res;
    }

    private void backtrackPermute(int[] nums, boolean[] used, List<Integer> path, List<List<Integer>> res) {
        if (path.size() == nums.length) {
            res.add(new ArrayList<>(path)); // 必须深拷贝！
            return;
        }
        for (int i = 0; i < nums.length; i++) {
            if (used[i]) continue;
            used[i] = true;
            path.add(nums[i]);
            backtrackPermute(nums, used, path, res);
            path.remove(path.size() - 1);
            used[i] = false;
        }
    }

    // LeetCode 78: 子集
    public List<List<Integer>> subsets(int[] nums) {
        List<List<Integer>> res = new ArrayList<>();
        backtrackSubsets(nums, 0, new ArrayList<>(), res);
        return res;
    }

    private void backtrackSubsets(int[] nums, int start, List<Integer> path, List<List<Integer>> res) {
        res.add(new ArrayList<>(path));
        for (int i = start; i < nums.length; i++) {
            path.add(nums[i]);
            backtrackSubsets(nums, i + 1, path, res);
            path.remove(path.size() - 1);
        }
    }
}
```
- **复杂度**：全排列时间 $O(N \cdot N!)$；子集时间 $O(N \cdot 2^N)$。

---

### B19. 爬楼梯 & 打家劫舍（LeetCode 70 / 198）

**核心思路**：入门状态压缩 DP，`dp[i]` 仅依赖前两项，直接滚动两个变量实现 $O(1)$ 空间。

```java
public class BasicDP {
    // LeetCode 70: 爬楼梯
    public int climbStairs(int n) {
        if (n <= 2) return n;
        int prev2 = 1, prev1 = 2;
        for (int i = 3; i <= n; i++) {
            int cur = prev1 + prev2;
            prev2 = prev1;
            prev1 = cur;
        }
        return prev1;
    }

    // LeetCode 198: 打家劫舍（dp[i] = max(dp[i-1], dp[i-2] + nums[i])）
    public int rob(int[] nums) {
        if (nums == null || nums.length == 0) return 0;
        int prev2 = 0, prev1 = 0;
        for (int num : nums) {
            int cur = Math.max(prev1, prev2 + num);
            prev2 = prev1;
            prev1 = cur;
        }
        return prev1;
    }
}
```
- **复杂度**：时间 $O(N)$，空间 $O(1)$。

---

### B20. 零钱兑换 & 零钱兑换 II（LeetCode 322 / 518 — 完全背包）

**核心考点（必考区分）**：
- **完全背包**内层容量正序遍历（每个硬币可重复选）。
- 求**组合数**（LeetCode 518，不计顺序）：**外层遍历物品，内层遍历容量**！若内外层颠倒则求的是排列数（LeetCode 377）。
- 求**最少硬币数**（LeetCode 322）：初始化为 `amount + 1`（哨兵极大值，切勿用 `Integer.MAX_VALUE` 防 `+ 1` 溢出！）。

```java
import java.util.Arrays;

public class CoinChangeSolutions {
    // LeetCode 322: 最少硬币个数
    public int coinChange(int[] coins, int amount) {
        int max = amount + 1;
        int[] dp = new int[amount + 1];
        Arrays.fill(dp, max);
        dp[0] = 0;
        for (int coin : coins) {
            for (int j = coin; j <= amount; j++) {
                dp[j] = Math.min(dp[j], dp[j - coin] + 1);
            }
        }
        return dp[amount] > amount ? -1 : dp[amount];
    }

    // LeetCode 518: 凑成总金额的组合数
    public int changeCombinations(int amount, int[] coins) {
        int[] dp = new int[amount + 1];
        dp[0] = 1;
        for (int coin : coins) { // 物品在外层 -> 组合数
            for (int j = coin; j <= amount; j++) {
                dp[j] += dp[j - coin];
            }
        }
        return dp[amount];
    }
}
```
- **复杂度**：时间 $O(N \cdot \text{amount})$，空间 $O(\text{amount})$。

---

### B21. 0-1 背包标准模板 & 分割等和子集（LeetCode 416）

**核心思路**：每个物品只能选一次，一维滚动数组**内层容量必须从大到小逆序遍历**，防止同一件物品在当轮被重复叠加使用！分割等和子集等价于：能否用数组中元素恰好装满容量为 `sum / 2` 的背包。

```java
public class Knapsack01 {
    // 标准 0-1 背包最大价值模板
    public int knapsack01(int[] weights, int[] values, int capacity) {
        int[] dp = new int[capacity + 1];
        for (int i = 0; i < weights.length; i++) {
            for (int j = capacity; j >= weights[i]; j--) { // 逆序遍历！
                dp[j] = Math.max(dp[j], dp[j - weights[i]] + values[i]);
            }
        }
        return dp[capacity];
    }

    // LeetCode 416: 分割等和子集
    public boolean canPartition(int[] nums) {
        int sum = 0;
        for (int num : nums) sum += num;
        if ((sum & 1) == 1) return false;
        int target = sum >>> 1;
        boolean[] dp = new boolean[target + 1];
        dp[0] = true;
        for (int num : nums) {
            for (int j = target; j >= num; j--) {
                dp[j] = dp[j] || dp[j - num];
            }
        }
        return dp[target];
    }
}
```
- **复杂度**：时间 $O(N \cdot \text{target})$，空间 $O(\text{target})$。

---

### B22. 最长递增子序列 LIS（LeetCode 300 — O(N²) & O(N log N)）

**核心思路**：
- $O(N^2)$ DP：`dp[i]` 表示以 `nums[i]` 结尾的最长严格递增子序列长度。
- **$O(N \log N)$ 贪心 + 二分（面试高频加分项）**：维护 `tails[k]` 表示长度为 `k+1` 的递增子序列结尾元素的最小值；`tails` 严格递增，遍历每个数时二分查找第一个 $\ge \text{num}$ 的位置替换（若都小于则追加延长）。

```java
public class LISSolution {
    public int lengthOfLIS(int[] nums) {
        if (nums == null || nums.length == 0) return 0;
        int[] tails = new int[nums.length];
        int len = 0;
        for (int num : nums) {
            int l = 0, r = len;
            while (l < r) {
                int mid = l + ((r - l) >>> 1);
                if (tails[mid] < num) l = mid + 1;
                else r = mid;
            }
            tails[l] = num;
            if (l == len) len++;
        }
        return len;
    }
}
```
- **注意**：`tails` 数组本身不一定是最终的合法子序列，但它的**长度 `len`** 一定等于严格最长递增子序列的长度！

---

### B23. 编辑距离（LeetCode 72）

**核心思路**：`dp[i][j]` 表示 `word1[0..i-1]` 变成 `word2[0..j-1]` 的最少操作数。若末尾字符相等则 `dp[i][j] = dp[i-1][j-1]`；否则取 `1 + min(插入 dp[i][j-1], 删除 dp[i-1][j], 替换 dp[i-1][j-1])`。

```java
public int minDistance(String word1, String word2) {
    int m = word1.length(), n = word2.length();
    int[][] dp = new int[m + 1][n + 1];
    for (int i = 0; i <= m; i++) dp[i][0] = i;
    for (int j = 0; j <= n; j++) dp[0][j] = j;

    for (int i = 1; i <= m; i++) {
        for (int j = 1; j <= n; j++) {
            if (word1.charAt(i - 1) == word2.charAt(j - 1)) {
                dp[i][j] = dp[i - 1][j - 1];
            } else {
                dp[i][j] = 1 + Math.min(dp[i - 1][j - 1],
                               Math.min(dp[i - 1][j], dp[i][j - 1]));
            }
        }
    }
    return dp[m][n];
}
```
- **复杂度**：时间 $O(M \cdot N)$，空间 $O(M \cdot N)$（可滚动数组优化到 $O(N)$）。

---

### B24. 最长回文子串（LeetCode 5 — 中心扩展法）

**核心思路**：相比二维 DP 需要 $O(N^2)$ 空间，中心扩展法只需 $O(1)$ 空间！枚举每个字符（奇数长度中心 `i, i`）和每对相邻间隙（偶数长度中心 `i, i+1`）向两侧扩展。

```java
public class LongestPalindromeSolution {
    public String longestPalindrome(String s) {
        if (s == null || s.length() < 2) return s;
        int start = 0, maxLen = 1;
        for (int i = 0; i < s.length(); i++) {
            int len1 = expand(s, i, i);
            int len2 = expand(s, i, i + 1);
            int len = Math.max(len1, len2);
            if (len > maxLen) {
                maxLen = len;
                start = i - (len - 1) / 2;
            }
        }
        return s.substring(start, start + maxLen);
    }

    private int expand(String s, int l, int r) {
        while (l >= 0 && r < s.length() && s.charAt(l) == s.charAt(r)) {
            l--;
            r++;
        }
        return r - l - 1;
    }
}
```
- **复杂度**：时间 $O(N^2)$，空间 $O(1)$。

---

### B25. 每日温度（LeetCode 739 — 单调栈模板）

**核心思路**：求“下一个更大元素”的标准模板！维护一个**单调递减栈（存下标）**；新元素比栈顶温度高时，说明找到了栈顶那一天的升温日，弹出并计算下标差 `i - idx`。

```java
import java.util.ArrayDeque;
import java.util.Deque;

public int[] dailyTemperatures(int[] temperatures) {
    int n = temperatures.length;
    int[] ans = new int[n];
    Deque<Integer> stack = new ArrayDeque<>(); // 存下标，用 ArrayDeque 替代过时的 Stack 类
    for (int i = 0; i < n; i++) {
        while (!stack.isEmpty() && temperatures[i] > temperatures[stack.peek()]) {
            int prevIdx = stack.pop();
            ans[prevIdx] = i - prevIdx;
        }
        stack.push(i);
    }
    return ans;
}
```
- **复杂度**：每个元素最多入栈出栈各 1 次，时间 $O(N)$，空间 $O(N)$。

---

### B26. 实现 Trie 前缀树（LeetCode 208）

**核心思路**：多叉树节点含 `TrieNode[26] children` 和 `boolean isEnd`。广泛用于搜索自动补全、敏感词过滤与 IP 路由最长前缀匹配。

```java
public class Trie {
    private static class TrieNode {
        TrieNode[] children = new TrieNode[26];
        boolean isEnd = false;
    }

    private final TrieNode root = new TrieNode();

    public void insert(String word) {
        TrieNode node = root;
        for (int i = 0; i < word.length(); i++) {
            int idx = word.charAt(i) - 'a';
            if (node.children[idx] == null) {
                node.children[idx] = new TrieNode();
            }
            node = node.children[idx];
        }
        node.isEnd = true;
    }

    public boolean search(String word) {
        TrieNode node = searchPrefix(word);
        return node != null && node.isEnd;
    }

    public boolean startsWith(String prefix) {
        return searchPrefix(prefix) != null;
    }

    private TrieNode searchPrefix(String prefix) {
        TrieNode node = root;
        for (int i = 0; i < prefix.length(); i++) {
            int idx = prefix.charAt(i) - 'a';
            if (node.children[idx] == null) return null;
            node = node.children[idx];
        }
        return node;
    }
}
```
- **复杂度**：插入与查找时间均为 $O(L)$（$L$ 为单词长度），空间 $O(\sum L \times 26)$。若字符集包含中文或特殊符号，可将 `children` 数组改为 `HashMap<Character, TrieNode>`。
