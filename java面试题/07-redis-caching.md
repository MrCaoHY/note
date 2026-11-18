# 07 - Redis 原理与高并发缓存架构面试题

> 本章覆盖：Redis 5 大基础 + 4 大扩展数据类型与底层编码（SDS / intset / ziplist / listpack / quicklist / dict 渐进式 Rehash / SkipList）、单线程与 IO 多路复用、Redis 6.0/7.0 多线程 IO、RESP 协议、过期键删除与 8 种淘汰策略（近似 LRU / LFU）、RDB（fork COW）/ AOF / Multi-Part AOF / 混合持久化、主从复制（PSYNC / 复制风暴）、哨兵（Raft 选主 / 脑裂）、Redis Cluster（16384 槽 / Gossip / MOVED & ASK / 扩缩容）、缓存穿透/击穿/雪崩、Cache-Aside 与 Canal+Binlog+MQ 最终一致性、手写分布式锁与 Redisson WatchDog / Redlock 争议、BigKey & HotKey 治理、Pipeline / Lua / Redis 事务不回滚机制、延迟队列与限流器实战。

---

## 目录

- [1. 数据类型与底层编码（Q1 - Q9）](#1-数据类型与底层编码q1---q9)
- [2. 高性能模型、协议与内存管理（Q10 - Q17）](#2-高性能模型协议与内存管理q10---q17)
- [3. 持久化与高可用集群（Q18 - Q25）](#3-持久化与高可用集群q18---q25)
- [4. 缓存三大难题与双写一致性（Q26 - Q30）](#4-缓存三大难题与双写一致性q26---q30)
- [5. 分布式锁、事务、Lua 与生产治理（Q31 - Q40）](#5-分布式锁事务lua-与生产治理q31---q40)

---

## 1. 数据类型与底层编码（Q1 - Q9）

### Q1：Redis 有哪些数据类型？各自底层数据结构与典型业务场景是什么？

**直接结论**  
Redis 常见数据类型分为 **5 种基础类型**（`String`、`List`、`Hash`、`Set`、`ZSet`）和 **4 种常用扩展类型**（`Bitmap`、`HyperLogLog`、`GEO`、`Stream`）。需要注意的是：**对外暴露的是“数据类型”，底层真正存储使用的是多种“内部编码（Encoding）”**，Redis 会根据数据规模自动进行编码转换（如 `listpack` → `hashtable`）。

| 数据类型 | 底层内部编码（Redis 7.0+ 为主） | 核心特性 | 典型业务场景 |
| :--- | :--- | :--- | :--- |
| **String** | `int` / `embstr` / `raw`（基于 SDS） | 二进制安全、最大 512MB、支持原子增减 | 缓存对象 JSON、分布式锁、全局计数/限流、Session |
| **List** | `listpack`（小列表） / `quicklist`（底层节点为 `listpack`） | 按插入顺序排序、两端 O(1) 读写、允许重复 | 简单消息队列、最新文章/动态列表（`LPUSH`+`LTRIM`） |
| **Hash** | `listpack`（Redis 6- 为 `ziplist`） / `hashtable`（Dict） | 字段级读写、节省 Key 空间 | 购物车、用户信息对象、分布式可重入锁结构 |
| **Set** | `intset` / `hashtable` | 无序、自动去重、支持交并差集 | 点赞/收藏去重、共同好友（`SINTER`）、抽奖（`SRANDMEMBER`/`SPOP`） |
| **ZSet** | `listpack`（Redis 6- 为 `ziplist`） / `skiplist` + `hashtable` | 按 `score` 排序、成员唯一 | 实时排行榜、滑动窗口限流、延迟队列 |
| **Bitmap** | 基于 `String`（最大 512MB 即 $2^{32}$ bit） | 按位操作、极省空间 | 每日签到、活跃用户统计、布隆过滤器底层存储 |
| **HyperLogLog** | 基于 `String`（固定占用 **12KB**） | 概率基数统计、标准误差 **0.81%** | 亿级网页 UV 去重统计（`PFADD` / `PFCOUNT`） |
| **GEO** | 底层就是 **`ZSet`**（`score` 为 52 位 GeoHash 整数） | 经纬度距离计算与半径检索 | 附近的人、附近门店、外卖骑手位置追踪 |
| **Stream** | 底层为 **`listpack` + 基数树（Radix Tree `rax`）** | 持久化、消费者组、ACK、历史消息回溯 | 轻量级可靠消息队列（替代 `List` 和 `Pub/Sub`） |

- **面试官连环追问**：
  1. **怎么查看一个 Key 当前的底层编码？**  
     使用命令 `OBJECT ENCODING key`（例如返回 `embstr`、`listpack`、`skiplist`）。
  2. **Redis 的 `String` 为什么最大是 512MB？**  
     Redis 底层源码 `sds.h` 与协议限制单次请求/字符串最大长度默认 `512 * 1024 * 1024` 字节（通过 `proto-max-bulk-len` 限制，防止单 Key 过大拖垮单线程）。
- **实战避坑**：不要把整个百万字大对象或巨型 List 塞进 Redis，单 Value 超过 10KB 就属于大 Key 风险区；同时 `Hash` 只有小对象时才用紧凑的 `listpack`，一旦字段数或值长度超标就会升级为 `hashtable`（不可逆降级），内存占用会陡增。

---

### Q2：String 底层 SDS 原理是什么？为什么不用 C 原生字符串？`int`/`embstr`/`raw` 如何转换？

**直接结论**  
Redis 的 `String` 底层是 **SDS（Simple Dynamic String，简单动态字符串）**。相比 C 语言以 `\0` 结尾的字符数组，SDS 具备 **$O(1)$ 获取长度、二进制安全、杜绝缓冲区溢出、减少内存重分配次数** 四大优势，且按长度分为 `sdshdr5/8/16/32/64` 五种头部结构以节省内存。

**深度原理剖析**
1. **SDS 结构（Redis 3.2+）**：
   ```c
   struct __attribute__ ((__packed__)) sdshdr8 {
       uint8_t len;         // 已使用字节长度（O(1) 获取 STRLEN）
       uint8_t alloc;       // 分配的总容量（不包含头和末尾 '\0'）
       unsigned char flags; // 低 3 位存类型（sdshdr5/8/16/32/64），高 5 位保留
       char buf[];          // 柔性数组，实际存储字节数据，末尾仍保留 '\0' 兼容 C 函数
   };
   ```
   - **二进制安全**：通过 `len` 判断结束而不是 `\0`，因此可以安全存储图片、Protobuf/Kryo 序列化字节流。
   - **空间预分配与惰性释放**：当拼接导致空间不足时，若新长度 `< 1MB` 则按 2 倍扩容（`alloc = 2 * newlen`）；若 `>= 1MB` 则每次多分配 `1MB` 冗余空间。缩短字符串时只修改 `len` 不立即 `free`，供后续追加复用。
2. **String 三种内部编码及转换条件**：
   - **`int`**：保存的是 64 位有符号长整型（`long`）整数值（且在共享整数池 `0~9999` 内直接复用共享对象，前提是未设置 `maxmemory` LRU/LFU 淘汰策略）。
   - **`embstr`**：保存长度 **$\le 44$ 字节** 的字符串。`redisObject`（16B）与 `sdshdr8`（3B + 1B `\0`）+ 数据（44B）= **64 字节**，恰好填满 jemalloc 一个 64B 内存块，**只需一次 `malloc` 内存分配**，且缓存行局部性极佳。
   - **`raw`**：长度 **$> 44$ 字节** 的字符串。`redisObject` 与 `sds` 分两次 `malloc` 分配。
   - **编码转换（单向不可逆）**：`int` 执行 `APPEND` 或转为浮点数/非整数字符串 → 变成 `raw`（即使原本不到 44 字节也直接变 `raw`，因为 `embstr` 是只读设计，修改会触发转成 `raw`）；`embstr` 执行 `APPEND` 修改 → 直接升级为 `raw`。

- **面试官连环追问**：
  1. **为什么 `embstr` 的分界线在 Redis 3.2 之前是 39 字节，3.2 之后变成 44 字节？**  
     Redis 3.2 之前 `sdshdr` 固定占 8 字节（`int len` 4B + `int free` 4B + `\0` 1B + `redisObject` 16B = 25B，64 - 25 = 39B）；Redis 3.2 引入 `sdshdr8`（`uint8_t len` 1B + `uint8_t alloc` 1B + `flags` 1B = 3B，16 + 3 + 1(`\0`) = 20B，64 - 20 = 44B）。
  2. **为什么用 `__attribute__ ((__packed__))`？**  
     取消编译器默认的字节对齐填充（Padding），紧凑排列结构体字段，极致压缩短字符串内存。
- **实战避坑**：尽量让高频短文本 Key/Value 控制在 44 字节以内（利用 `embstr` 减少内存分配碎片），且注意 `embstr` 只要做一次 `APPEND` 就会立刻变成 `raw`，能用 `SET` 覆盖写就不要频繁 `APPEND`。

---

### Q3：List 底层数据结构是如何演进的？`ziplist` 的级联更新问题是什么？`listpack` 如何解决？

**直接结论**  
Redis 的 `List` 经历了 `ziplist + linkedlist`（Redis 3.2 前）→ **`quicklist`（双向链表 + `ziplist` 节点，Redis 3.2~6.x）** → **`quicklist`（内部节点改为 `listpack`，Redis 7.0+）** 的演进。核心驱动力是：**既要省内存（避免大量链表前后指针开销），又要彻底消除 `ziplist` 的级联更新抖动**。

**深度原理剖析**
1. **为什么不用纯 `linkedlist` 或纯 `ziplist`？**
   - 纯双向链表每个节点需要 `prev` 和 `next` 指针（64 位机占 16B）+ 单独 `malloc`，内存碎片大、CPU 缓存命中率低。
   - 纯 `ziplist` 是一整块连续内存，元素多时插入/删除需要 `realloc` 并整片搬移内存，且存在致命的**级联更新（Cascade Update）**。
2. **什么是 `ziplist` 的级联更新？**
   - `ziplist` 每个 `entry` 头部记录了**前一个元素的长度 `prevlen`**：如果前一个元素长度 `< 254` 字节，`prevlen` 占 **1 字节**；如果 `>= 254` 字节，`prevlen` 占 **5 字节**。
   - **极端场景**：若连续有多个长度在 `250~253` 字节的 `entry` 相邻，此时在头部插入一个 `>= 254` 字节的元素，导致后一个元素的 `prevlen` 从 1B 膨胀为 5B，总长度突破 254B，从而引发再下一个元素的 `prevlen` 也膨胀……最坏触发 $O(N^2)$ 级别的连锁内存重分配！
3. **`listpack` 如何彻底解决级联更新？**
   - `listpack` 每个 `entry` 只记录**自身长度**（`encoding-type` + `element-data` + **`element-tot-len`**）：
     ```text
     | lp-bytes (4B) | lp-num-elements (2B) | entry1 | entry2 | ... | lp-end (1B: 0xFF) |
     其中每个 entry 结构: [ encoding | data | backlen(当前 entry 自身总长度, 变长 1~5B) ]
     ```
   - 因为每个 `entry` 不再依赖前一个元素的长度，插入或修改一个元素只会影响它自己的 `backlen`，**绝对不会波及后续任何元素**！同时，反向遍历时通过读前一个元素末尾的 `backlen`（最高位为 0/1 标识结束）依然可以 $O(1)$ 定位前驱起点。
4. **`quicklist`（Redis 7.0+）**：
   - 将长列表切分为多个 `quicklistNode`，每个节点持有一个 `listpack`（大小受 `list-max-listpack-size` 控制，默认 `-2` 表示单节点上限 8KB），并支持对中间冷节点开启 LZF 压缩（`list-compress-depth`）。

- **面试官连环追问**：
  1. **`list-max-listpack-size` 配置正数和负数分别代表什么含义？**  
     正数表示每个 `quicklistNode` 最多包含的元素个数；负数表示字节大小限制：`-1` (4KB)、`-2` (8KB，默认)、`-3` (16KB)、`-4` (32KB)、`-5` (64KB)。
- **实战避坑**：把 `List` 当作消息队列使用时，必须警惕 `LRANGE key 0 -1` 扫全表阻塞主线程；如果需要分页或弹出，尽量只操作头尾（`LPUSH`/`RPOP`/`BLPOP`），或者直接升级为 `Stream`。

---

### Q4：Hash 和 Set 的底层实现是什么？Dict 渐进式 Rehash 过程是怎样的？

**直接结论**  
- **`Hash`**：数据量小时使用 **`listpack`**（Redis 6.x 及以前为 `ziplist`），满足“元素个数 $\le 512$ 且所有 key/value 长度 $\le 64$ 字节”（`hash-max-listpack-entries` / `hash-max-listpack-value`）时启用；超出则转为 **`hashtable`（`dict`）**。
- **`Set`**：当所有元素都是整数且数量 $\le 512$（`set-max-intset-entries`）时用 **`intset`**（有序整数数组，二分查找，支持 `int16` → `int32` → `int64` 升级）；否则使用 **`hashtable`**（Redis 7.2 还新增了小字符串集合优化 `listpack`）。

**深度原理剖析：Dict 渐进式 Rehash**
Redis 的 `dict` 内部维护了两个哈希表 `ht[0]` 和 `ht[1]`（`dictht`），通过链地址法（头插法）解决哈希冲突，并使用 **SipHash** 算法防哈希碰撞 DoS 攻击：
```c
typedef struct dict {
    dictType *type;
    dictht ht[2];       // ht[0] 平时使用，ht[1] 仅在 rehash 时使用
    long rehashidx;     // -1 表示未在 rehash；>=0 表示当前正在迁移 ht[0] 的桶索引
    int16_t pauserehash;
} dict;
```
1. **触发扩容条件**（负载因子 `load_factor = ht[0].used / ht[0].size`）：
   - 没有在执行 `bgsave` / `bgrewriteaof` 子进程时：`load_factor >= 1` 触发扩容。
   - 正在执行 `bgsave` / `bgrewriteaof` 时：`load_factor >= 5` 才扩容（避免父进程大量修改页表触发操作系统写时复制 COW 导致内存暴涨）。
   - **缩容**：`load_factor < 0.1` 时触发缩容。
   - 扩容目标大小：第一个大于等于 `ht[0].used * 2` 的 $2^n$。
2. **渐进式 Rehash 四步流程**（为什么不一次性迁移？因为百万 Key 一次性迁移会卡死单线程）：
   - ① 为 `ht[1]` 分配空间，设置 `rehashidx = 0`。
   - ② **读写删查联动**：每次对字典执行增、删、改、查时，除完成本职操作外，顺带将 `ht[0].table[rehashidx]` 整个桶链表迁移到 `ht[1]`，然后 `rehashidx++`。
   - ③ **定时任务兜底**：即使没有请求访问，Redis 主线程的 `serverCron`（默认每 10ms）也会在 1ms 时间片内主动推进搬迁。
   - ④ **迁移期间读写规则**：**新增**一律直接写入 `ht[1]`；**查询/修改/删除**会先查 `ht[0]`，没找到再去查 `ht[1]`。
   - ⑤ 全部桶迁移完毕后，释放 `ht[0]`，将 `ht[1]` 赋给 `ht[0]`，重置 `rehashidx = -1`。

- **面试官连环追问**：
  1. **为什么 `bgsave` 期间要把扩容阈值从 1 提高到 5？**  
     因为 `fork` 子进程采用 Copy-On-Write（写时复制），如果父进程此时大面积 Rehash 修改哈希表指针，会导致大量物理内存页被复制，引发内存 OOM 风险。
  2. **`intset` 升级后可以降级吗？**  
     不支持降级。一旦插入一个 `int64` 整数引发全数组升级，即使之后删掉这个大数，依然保持 `int64` 存储。
- **实战避坑**：Hash 存放大量对象属性时，如果某一个字段值不小心超过 64 字节，会导致整个 Hash 从紧凑的 `listpack` 退化为 `hashtable`，内存可能翻 3~5 倍；大 Hash 建议拆桶（如 `user:info:{uid/1000}` 做二级 Hash）控制每桶在 500 个字段以内，既省内存又避免 BigKey。

---

### Q5：ZSet 底层为什么用跳表（SkipList）+ 哈希表？为什么不用红黑树或 B+ 树？

**直接结论**  
ZSet 在元素较多时同时使用 **`zskiplist`（跳表）+ `dict`（哈希表）**：哈希表实现 $O(1)$ 时间复杂度根据 `member` 查 `score`（`ZSCORE`）；跳表按 `score`（相同则按 `member` 字典序）排序，支持 $O(\log N)$ 增删改查和高效范围查询（`ZRANGE` / `ZRANGEBYSCORE`）。选择跳表而非红黑树的核心原因是：**范围查询更高效、实现简单无旋转、内存可调且局部改动小**。

**深度原理剖析**
1. **跳表节点结构（`zskiplistNode`）**：
   - 每个节点包含 `ele`（SDS 成员）、`score`（double 分值）、`backward`（后退指针，支持从尾到头 `ZREVRANGE` 遍历）以及 `level[]` 柔性数组。
   - 每一层包含 `forward` 前进指针和 **`span` 跨度**（记录当前节点到下一节点跨越了多少个底层节点，使得 `ZRANK` 计算排名能在 $O(\log N)$ 内直接累加 `span` 得出，而无需从头计数！）。
   - **随机层高生成**：新节点层高从 1 开始，以概率 `p = 0.25`（25%）逐层递增，最大层高限制为 `ZSKIPLIST_MAXLEVEL`（Redis 5- 为 64，Redis 6+ 为 **32**）。期望平均每个节点指针数为 $1 / (1 - 0.25) = 1.33$ 个，比红黑树每个节点固定 2 个指针（左右孩子）+ 父指针（共 3 个）还省内存！
2. **为什么不用红黑树 / AVL 树 / B+ 树？**
   - **vs 红黑树（RB-Tree）**：
     - **范围查询**：跳表定位到区间起点后，只需沿着第 0 层 `forward` 链表顺序往后扫即可，缓存局部性好；红黑树做范围查询需要复杂的中序遍历回溯父节点。
     - **维护成本**：跳表插入/删除只需在各层链表做局部指针修改；红黑树需要左旋、右旋和变色来维持全局平衡，代码复杂且涉及整条路径。
     - **可调优性**：通过修改概率 `p` 可以灵活权衡空间与查询时间复杂度，红黑树结构死板。
   - **vs B+ 树**：
     - B+ 树的核心优势是单节点（页）存储多个索引键以**减少磁盘 I/O 次数**；而 Redis 是**纯内存数据库**，没有磁盘寻道痛点，B+ 树节点内二分/数组搬移反而带来额外 CPU 写拷贝开销。

- **面试官连环追问**：
  1. **ZSet 的 `score` 相同时如何排序？**  
     在跳表比较逻辑中，如果 `score` 相等，会进一步按 `ele`（成员字符串）的**字典序（memcmp）**升序排列。
  2. **ZSet 什么情况下会从 `listpack` 转成 `skiplist`？**  
     当元素个数 $> 128$（`zset-max-listpack-entries`）或任意成员长度 $> 64$ 字节（`zset-max-listpack-value`）时触发转换。
- **实战避坑**：ZSet `score` 是 IEEE 754 双精度浮点数（`double`），只有 **53 位有效整数精度**（超过 $2^{53} \approx 9 \times 10^{15}$ 会丢失精度！），所以如果把“雪花算法 64 位 ID”或“复合时间戳+自增序号”直接拼进 `score` 做同分排序，超过 53 位后排名会出现错乱。

---

### Q6：Bitmap 和 HyperLogLog 的底层原理与误差控制？如何实现百万用户连续签到统计？

**直接结论**  
- **Bitmap** 本质是 `String`（按字节数组的每一位 bit 存储 0/1），1 亿用户的单日签到仅需 $10^8 / 8 \approx 12\text{MB}$ 内存。
- **HyperLogLog（HLL）** 是基于**伯努利试验 + 分桶调和平均**的概率基数估计算法，固定只占 **12KB** 内存即可统计最多 $2^{64}$ 个不同元素的基数，标准误差仅 **0.81%**。

**深度原理剖析**
1. **HyperLogLog 核心原理**：
   - 将输入元素经过 64 位哈希，取低 **14 位** 作为桶下标（共 $2^{14} = 16384$ 个桶），高 **50 位** 从低位起统计连续 `0` 的个数（记为首次出现 `1` 的位置 $k$），每个桶只用 **6 bit** 记录最大前导零位数 $k_{\max}$（$16384 \times 6\text{ bit} = 12\text{KB}$）。
   - 根据各桶的 $k_{\max}$，通过**调和平均数**（而非算术平均，防止极端哈希碰撞拉偏结果）估算总体基数，并在数据量少时使用稀疏矩阵存储 + 线性计数修正小基数误差。
2. **百万用户连续签到统计实战（Bitmap + `BITFIELD`）**：
   - **Key 设计**：`sign:{userId}:{yyyyMM}`（按月隔离，每月最多 31 bit，即 4 字节，防止单 Key 随天数无限膨胀成 BigKey，且便于按月过期归档）。
   - **签到命令**：`SETBIT sign:1001:202611 17 1`（11月18号，偏移量从0算起为17）。
   - **计算本月总签到次数**：`BITCOUNT sign:1001:202611`。
   - **计算截至今天的连续签到天数（核心高频题）**：利用 `BITFIELD sign:1001:202611 GET u18 0` 一次取出从 1 号到 18 号的无符号 18 位整数，在 Java 内存中通过位运算 `(val & 1) == 1`、`val >>>= 1` 从低位向右循环统计连续 `1` 的个数即可，**仅 1 次网络 RTT**！

- **面试官连环追问**：
  1. **用 Bitmap 记录用户 ID（如雪花算法 64 位整数）签到有什么致命隐患？**  
     如果直接用几十位长的雪花 ID 作为 `SETBIT` 的 offset，比如 offset = `1000000000`，Redis 会立刻分配 $10^9 / 8 \approx 125\text{MB}$ 的全零字符串，**瞬间制造超大 BigKey 导致阻塞或 OOM**！必须采用紧凑自增整数 UID，或者给每人独立按月建 Key（`sign:{uid}:{yyyyMM}`，offset 为几号 `0~30`）。
- **实战避坑**：HLL 只能统计总数（`PFCOUNT`）和合并多个 HLL（`PFMERGE`），**无法判断某个具体元素是否在集合内**，也无法反推出元素明文；需要判断是否存在必须用 `Set` 或 `Bitmap/布隆过滤器`。

---

### Q7：Redis GEO 的底层原理是什么？GeoHash 编码与边界问题如何处理？

**直接结论**  
Redis GEO 并不是一种独立的底层结构，它完全基于 **`ZSet`** 实现：通过 **GeoHash 算法** 将二维的 `(经度, 纬度)` 交叉编码成一个 **52 位无符号整数** 作为 ZSet 的 `score`，从而支持高效的半径检索（`GEOSEARCH`，Redis 6.2+ 推荐替代已废弃的 `GEORADIUS`）。

**深度原理剖析**
1. **GeoHash 编码原理（二分逼近 + 交叉位编码）**：
   - 纬度范围 `[-85.05112878, 85.05112878]`（Web Mercator 投影限制）、经度范围 `[-180, 180]`，分别做 26 次二分逼近（落在左/下区间记 `0`，右/上区间记 `1`），得到 26 位纬度编码和 26 位经度编码。
   - **Z 阶曲线（Morton Code）交叉合并**：偶数位放经度、奇数位放纬度，拼成 52 位整数（正好不超出 IEEE 754 `double` 的 53 位精确整数上限！）。
   - **本质**：地理位置相近的点，其 GeoHash 二进制**前缀往往相同**，转化为整数区间后在 ZSet 中位置相邻，从而把二维空间搜索降维成一维 ZSet 范围查询！
2. **GeoHash 的边界突变问题与 Redis 解法**：
   - 两个物理距离极近的点，如果恰好分居在主格子分割线两侧，它们的 GeoHash 前缀可能完全不同。
   - **解法**：Redis 在执行 `GEOSEARCH` 时，不仅查中心点所在的网格，还会同时计算并查询**周围相邻的 8 个网格**（共 9 宫格），将这 9 个区间对应的 ZSet 候选点全部拉出来后，再用 Haversine 球面距离公式做精确过滤与排序。

- **面试官连环追问**：
  1. **海量城市门店（如全国千万级 POI）适合全部塞进一个 Redis GEO Key 吗？**  
     绝对不行！千万元素在一个 ZSet 里就是巨型 BigKey，不仅单分片内存撑爆，Cluster 迁移也会卡死。应该按**城市/区县编码拆分 Key**（如 `geo:shop:{cityCode}`），且如果只需单次范围查可用 `GEOSEARCH ... COUNT 50 ASC` 限制返回数量避免阻塞。
- **实战避坑**：Redis 6.2 起 `GEORADIUS` 和 `GEORADIUSBYMEMBER` 已标记为过时（Deprecated），新项目统一使用 **`GEOSEARCH`** 和 **`GEOSEARCHSTORE`**；同时尽量不要在同一个 GEO Key 中存放超过 10 万条点位。

---

### Q8：Redis Stream 的核心结构与命令流是什么？它与 List、Pub/Sub、Kafka 相比有什么优劣？

**直接结论**  
Redis 5.0 引入的 **Stream** 是专为可靠消息队列设计的持久化数据结构，底层结合了 **`rax`（基数树）+ `listpack`**。它彻底补齐了 `List` 不支持多播消费组、`Pub/Sub` 消息不持久化且断线必丢的短板，适合**轻量级、低延迟、不允许丢消息的分布式消息场景**。

**深度原理剖析**
1. **Stream 核心模型与命令流**：
   - **消息 ID**：格式 `<毫秒时间戳>-<序号>`（如 `1731920000000-0`），全局单调递增。
   - **生产消息**：`XADD mystream MAXLEN ~ 100000 * orderId 1001`（`MAXLEN ~` 近似裁剪，防止队列无限膨胀）。
   - **消费者组（Consumer Group）**：
     - 创建组：`XGROUP CREATE mystream groupA 0`（`0` 从头消费，`$` 从最新消息开始消费）。
     - 组内拉取：`XREADGROUP GROUP groupA consumer1 COUNT 10 BLOCK 2000 STREAMS mystream >`（`>` 表示只读取未投递给组内任何消费者的新消息）。
     - **PEL（Pending Entries List，未确认队列）**：消息发给 `consumer1` 后会暂存在它的 PEL 中；消费成功后必须显式执行 **`XACK mystream groupA <msgId>`** 才会从 PEL 移除。若消费者宕机，其他消费者可通过 `XAUTOCLAIM` 接管超时未 ACK 的消息。
2. **四种消息队列方案横评**：

| 对比维度 | List (`LPUSH`/`BRPOP`) | Pub/Sub | Redis Stream | Kafka / RocketMQ |
| :--- | :--- | :--- | :--- | :--- |
| **持久化** | 支持（RDB/AOF） | **不支持**（纯内存转发） | 支持（RDB/AOF） | 磁盘顺序写，极可靠 |
| **消费组多播** | 不支持（一条消息仅一人消费） | 支持广播，不支持负载均衡 | **完美支持** | 完美支持 |
| **ACK 与消息回溯** | 无 ACK（弹出即删） | 无 ACK、无回溯 | **支持 PEL + XACK + 重读** | 完善 Offset + 重试死信 |
| **堆积能力** | 受限于内存 | 无堆积 | 受限于内存（需 `MAXLEN`） | TB/PB 级海量堆积 |

- **面试官连环追问**：
  1. **Redis Stream 能完全替代 RocketMQ / Kafka 吗？**  
     不能。Redis 依然可能因为**异步主从复制丢数据**（主挂了从没同步完）、**内存容量上限**（消息大量堆积直接打满内存导致 OOM）、以及**缺少原生事务消息/死信队列/延迟消息**而无法胜任金融订单核心链路。
- **实战避坑**：使用 `XADD` 务必带上 `MAXLEN ~ 阈值` 限制队列长度，且消费端必须定期用 `XAUTOCLAIM` 处理 PEL 中卡死的僵尸消息，否则 PEL 越积越大也会造成内存泄漏。

---

### Q9：Redis 内存对象模型（`redisObject`）包含哪些字段？如何影响编码转换与内存占用？

**直接结论**  
Redis 中每一个 Value（以及 Hash/Set 等里的对象）在 C 源码中都被封装为一个 **16 字节的 `redisObject`（`robj`）结构体**，它是实现“对外统一数据类型 + 对内多态编码切换 + LRU/LFU 淘汰”的总枢纽。

**深度原理剖析**
```c
typedef struct redisObject {
    unsigned type:4;        // 对外逻辑类型：OBJ_STRING(0)/LIST(1)/HASH(2)/SET(3)/ZSET(4)/STREAM(6)
    unsigned encoding:4;    // 内部物理编码：raw/int/embstr/hashtable/intset/skiplist/quicklist/listpack
    unsigned lru:24;        // LRU 模式存最后访问秒级时间戳(24bit); LFU 模式高16位存分钟时间戳, 低8位存对数访问频次
    int refcount;           // 引用计数（为 0 时释放，0~9999 整数共享对象池设为 OBJ_SHARED_REFCOUNT）
    void *ptr;              // 指向底层真实数据结构（如 SDS / dict / skiplist）的指针（8字节）
} robj;
```
- **总大小计算**：`4bit + 4bit + 24bit` = 4 字节 + `int refcount` (4 字节) + `void *ptr` (8 字节) = **16 字节**。
- **常见面试洞察**：一个只存了 `SET k1 1` 的键值对，实际占用远不止几个字节！除了 `redisObject`(16B)，还有 `dictEntry`(24B + 分配器对齐到 32B)、Key 的 SDS、以及 `ptr`（如果是 `int` 则直接把整数值强转存在 `ptr` 指针位置上，省去额外分配；如果是 `embstr` 则与 `redisObject` 合并一次 `malloc`）。所以**尽量避免海量碎片化的单 String 小 Key**，将其聚合到一个 `Hash` 的 `listpack` 中可以省去海量的 `dictEntry` 和 `redisObject` 头部开销（内存可省 80%+）。

- **面试官连环追问**：
  1. **为什么开启 `maxmemory` 淘汰策略后，`0~9999` 共享整数对象池会失效？**  
     因为 `lru` 字段记录在 `redisObject` 上，如果多个 Key 共享同一个 `redisObject`，就无法区分到底是哪个 Key 最近被访问过，会导致 LRU/LFU 淘汰统计完全错乱。

---

## 2. 高性能模型、协议与内存管理（Q10 - Q17）

### Q10：Redis 为什么那么快？IO 多路复用（Reactor 模型）是如何工作的？

**直接结论**  
Redis 单机轻松达到 **10 万+ QPS** 的四大基石是：**① 纯内存操作**；**② 高效底层数据结构（SDS/跳表/listpack）**；**③ 单线程执行命令消除上下文切换与锁竞争**；**④ 基于 `epoll`/`kqueue` 的单线程 Reactor IO 多路复用模型**。

**深度原理剖析：Redis Reactor 事件循环**
```text
             ┌──────────────────────────────────────┐
             │  AE 事件循环 (aeMain)                 │
Clients ───► │  epoll_wait (监听上万 Socket 就绪事件) │
             └───────────────┬──────────────────────┘
                             │ 分发就绪事件
       ┌─────────────────────┼─────────────────────┐
       ▼                     ▼                     ▼
  连接应答处理器         命令请求处理器         命令回复处理器
 (acceptTcpHandler)  (readQueryFromClient) (sendReplyToClient)
  监听新连接建立       读 Socket + 解析命令    将结果缓冲写回 Socket
                            │
                            ▼
                  单线程串行执行内存命令
```
- **核心机制**：操作系统提供的 `epoll`（Linux）/ `kqueue`（macOS）允许一个线程同时监听成千上万个客户端连接（FD）。只有当某个 Socket 真正有数据可读/可写时，`epoll_wait` 才返回就绪事件列表，**绝不会在无数据的连接上阻塞等待**。
- **为什么命令执行坚持单线程？**  
  因为内存操作耗时极短（纳秒~微秒级），单核 CPU 根本不是瓶颈，真正的瓶颈往往是**网络带宽和内存大小**。如果做成多线程并发执行命令，反而会因为**线程上下文切换**和**复杂数据结构加锁竞争**（如跳表、字典加锁）导致吞吐下降。

- **面试官连环追问**：
  1. **为什么说 `epoll` 比 `select` / `poll` 强？**  
     `select` 有 1024 FD 上限且每次都需要将全量 FD 集合从用户态拷贝到内核态并 $O(N)$ 轮询；`epoll` 通过红黑树管理 FD、就绪链表回调通知，只返回就绪的 FD，时间复杂度接近 $O(1)$。

---

### Q11：Redis 6.0 / 7.0 引入多线程是怎么回事？核心命令执行还是单线程吗？

**直接结论**  
**核心命令的内存执行依然是严格单线程的（绝对线程安全）！** Redis 6.0 引入的多线程仅负责 **“网络 IO 数据的读取解析”** 与 **“响应数据的写回 Socket”**，因为高并发下大量时间消耗在 Socket 读写和字节流解析上。

**深度原理剖析**
```text
1. 主线程接收新连接，放入全局读队列 clients_pending_read
2. 主线程通过轮询 (Round-Robin) 将可读连接分配给 IO 线程池 (io-threads) 和主线程自己
3. [多线程并行]: 读取 Socket 字节流并解析成 Redis 命令参数 (但不执行！)
4. 主线程自旋等待所有 IO 线程解析完毕
5. [主线程严格单线程串行]: 顺序执行所有已解析的命令，将响应写入客户端 output buffer
6. [多线程并行]: 主线程再将待写回队列分配给 IO 线程池并行 flush 到 Socket
```
- **如何开启与调优（`redis.conf`）**：
  - 默认 `io-threads` 是关闭的（值为 1），且**默认只开写多线程**。
  - 若开启读写多线程：
    ```ini
    io-threads 4           # 建议设为 CPU 核数的一半（如 8 核设 4，不要超过 8）
    io-threads-do-reads yes # 开启读+解析多线程
    ```
  - **Redis 7.0 补充**：优化了多线程唤醒机制和共享内存/协议解析细节，进一步降低了高并发小包场景下的延迟抖动。

- **实战避坑**：不要盲目把 `io-threads` 设得很大（如 32），超过 8 个后线程同步与自旋开销反而会拖慢性能；如果单机 QPS 在几万以下，单线程完全够用，甚至无需开启 `io-threads`。

---

### Q12：什么是 RESP 协议（RESP2 vs RESP3）？为什么 Redis 选择文本型 RESP 协议？

**直接结论**  
**RESP（REdis Serialization Protocol）** 是 Redis 客户端与服务端通信的序列化协议。它是一种**面向行、自描述、带长度前缀的二进制安全文本协议**（基于 TCP），在**可读性**、**解析速度**与**扩展性**之间取得了极佳平衡。

**深度原理剖析**
1. **RESP2 五大数据类型前缀（每行以 `\r\n` 结尾）**：
   - `+`：简单字符串（如 `+OK\r\n`）
   - `-`：错误响应（如 `-ERR unknown command\r\n`）
   - `:`：整数（如 `:1000\r\n`）
   - `$`：批量字符串（带字节长度，**二进制安全**！如 `$5\r\nhello\r\n`，空值为 `$-1\r\n`）
   - `*`：数组（如命令 `SET k1 v1` 编码为 `*3\r\n$3\r\nSET\r\n$2\r\nk1\r\n$2\r\nv1\r\n`）
2. **RESP3（Redis 6.0+ 新增，需显式发送 `HELLO 3` 切换）**：
   - 新增了原生的 **Map（`%`）**、**Set（`~`）**、**Boolean（`#t`/`#f`）**、**Double（`,`）**、**Null（`_`）**、**Push 推送帧（`>`）** 等类型，使得客户端无需在一条连接上把 Hash/ZSet 扁平数组再二次反序列化，同时让同一连接支持客户端缓存失效推送（Client-side Caching Tracking）。
3. **为什么不直接用 HTTP/JSON 或 Protobuf？**  
   HTTP/JSON 冗余大且 JSON 遇到二进制字符需转义、解析慢；Protobuf 需要预先定义 `.proto` Schema，不够灵活；而 RESP **无需预先约定 Schema、直接按长度偏移切片读取（无需扫描 `\0`）、人肉可读可用 `nc`/`telnet` 直接调试**。

---

### Q13：Redis 的过期键删除策略是什么？主从库如何处理过期键？

**直接结论**  
Redis 采用 **“惰性删除（Lazy Expiration） + 定期删除（Active Expiration）”** 相结合的过期策略，并在内存达到上限时由 **内存淘汰策略（Eviction）** 兜底。

**深度原理剖析**
1. **过期字典 `expires`**：每个 `redisDb` 除了保存数据的 `dict`，还维护一个 `expires` 字典，Key 是指针指向真实 Key 对象，Value 是 `long long` 毫秒级绝对过期时间戳。
2. **惰性删除（省 CPU、费内存）**：
   - 只有当客户端真正读写某个 Key 时，`expireIfNeeded()` 才检查它是否过期，若过期则删除并返回 `nil`。缺点是“冷过期 Key”没人访问就会一直赖在内存里。
3. **定期删除（折中平衡）**：
   - Redis 默认每秒执行 10 次（`hz 10`）`activeExpireCycle()`，每次从各个 DB 的 `expires` 字典中**随机抽取 20 个 Key** 检查并删除过期 Key；如果本轮过期比例 **$> 25\%$**，则继续重复抽查；同时设置了**单次最大执行时间上限**（慢模式默认最多占用 25% CPU 时间，即 25ms），避免卡死主线程！
4. **主从复制与持久化对过期键的处理（高频细节）**：
   - **RDB**：生成快照时忽略已过期键；主库加载 RDB 忽略过期键；**从库加载 RDB 会保留过期键**（等主库发 `DEL` 同步过来删，保证主从数据一致性）。
   - **AOF**：写入 AOF 时不记录过期判断，只有当某 Key 真正被惰性/定期删除时，主库才追加一条 **`DEL key`** 到 AOF 并同步给从库。
   - **从库读取过期键**：Redis 3.2 之前从库读到过期键还会误返回旧值；**Redis 3.2+ 从库读到已过期键会直接返回 `nil`**（但自己不主动删，仍等主库发 `DEL`）。

- **面试官连环追问**：
  1. **如果大量 Key 在同一秒过期，定期删除会直接把 Redis 卡死吗？**  
     正因为有“随机抽样 + 最大时间片上限（默认不超过 25ms）”，定期删除不会一次性删光导致卡死，但它会导致这批未及时清理的 Key 占满内存（配合缓存雪崩），且每次触发循环都在消耗 25% CPU，延迟会明显上升。

---

### Q14：Redis 的 8 种内存淘汰策略有哪些？近似 LRU 和 LFU（Morris 计数器）如何实现？

**直接结论**  
当 Redis 已用内存超过 `maxmemory` 时，触发**内存淘汰（Eviction）**（与 Key 是否设置 TTL 无关，而是为了腾空间）。共有 **8 种策略**，生产缓存场景最推荐使用 **`allkeys-lfu`** 或 **`allkeys-lru`**。

**深度原理剖析**
1. **8 种淘汰策略全景**：
   - **不淘汰**：`noeviction`（**默认策略**！写请求直接报错 OOM，只读不写，适合把 Redis 当 DB 用但不适合纯缓存）。
   - **仅对设置了过期时间的 Key（`volatile-*`）**：
     - `volatile-lru`：淘汰最久未使用的。
     - `volatile-lfu`：淘汰访问频率最低的。
     - `volatile-ttl`：淘汰剩余寿命（TTL）最短的。
     - `volatile-random`：随机淘汰。
   - **对所有 Key（`allkeys-*`）**：
     - `allkeys-lru`：全局淘汰最久未使用的。
     - `allkeys-lfu`：全局淘汰访问频率最低的。
     - `allkeys-random`：随机淘汰。
2. **为什么 Redis 用“近似 LRU”而不是标准 LRU？**  
   标准 LRU 需要一个覆盖所有 Key 的双向链表，不仅额外消耗每 Key 16 字节指针，还会引发并发/内存移动开销。Redis 采用**随机采样池（`EvictionPool`）**：每次随机抽取 `maxmemory-samples`（默认 **5** 个）Key，放入大小为 16 的有序淘汰池中，淘汰其中最老的那个。当样本数调到 10 时，效果几乎与真实 LRU 一致！
3. **LFU 原理（解决 LRU 扫描污染问题）**：
   - **LRU 的缺陷**：如果某个冷数据在几分钟前被全表扫描偶然摸了一下，LRU 会以为它是热数据而误删真正的高频热点 Key。
   - **LFU 实现**：复用 `redisObject.lru` 的 24 bit：**高 16 位**存上次访问时间（分钟级 `ldt`），**低 8 位**存访问频次 **`logc`（0~255）**。
     - **概率递增（Morris Counter）**：并非每次访问都 +1，而是以 $1 / (\text{logc} \times \text{lfu-log-factor} + 1)$ 的概率递增（`logc` 越大越难涨，255 可表达百万级访问）。
     - **时间衰减**：距上次访问每经过 `lfu-decay-time` 分钟（默认 1 分钟），`logc` 减 1，防止曾经爆红、现在过气的旧热点长期霸占内存。

- **实战避坑**：如果把 Redis 用作**纯缓存**，一定要把 `maxmemory-policy` 从默认的 `noeviction` 改成 **`allkeys-lfu`** 或 **`allkeys-lru`**，否则缓存满后业务写直接大面积抛异常；但如果 Redis 里同时混存了**分布式锁 Key** 和**普通缓存 Key**，千万别用 `allkeys-*`，否则锁被淘汰会导致超卖，应使用 `volatile-*` 且确保锁不设或单独隔离实例！

---

### Q15：什么是 Redis 内存碎片？`mem_fragmentation_ratio` 过高或过低说明什么？

**直接结论**  
执行 `INFO memory` 可以看到 **`mem_fragmentation_ratio = used_memory_rss / used_memory`**（操作系统实际分配给 Redis 进程的物理内存 / Redis 自己申请保存数据的逻辑内存）。**健康范围为 `1.0 ~ 1.5`**。

**深度原理剖析**
1. **为什么会有内存碎片？**
   - **分配器按固定步长分配**：Redis 默认使用 **jemalloc**（按 8B/16B/32B/64B... 固定档位分配），如果存 13B 数据会分配 16B，多出 3B 内部碎片。
   - **频繁修改与删除**：大量删除小 Key 或 Value 长度频变后，释放出的零散不连续内存块无法被 OS 回收，也难以满足后续大对象的连续分配需求。
2. **指标异常诊断与治理**：
   - **`> 1.5`（碎片严重）**：存在大量空洞内存浪费。
     - **解法**：Redis 4.0+ 支持**在线自动碎片整理**（`CONFIG SET activedefrag yes`），后台会在不影响主线程延迟的前提下（受 `active-defrag-cycle-min/max` CPU 占比限制）把数据重新拷贝整合，释放连续空页给 OS。若版本太老或极端严重，只能平滑主从切换重启节点。
   - **`< 1.0`（极度危险！）**：说明部分 Redis 内存已经被操作系统 **Swap（换出到磁盘虚拟内存）** 了！一旦访问被 Swap 的页就会发生磁盘 I/O，Redis 延迟会瞬间从微秒飙升到百毫秒级！必须立即扩容或清理数据，并关闭/降低 OS `vm.swappiness`。

---

### Q16：`KEYS *` 为什么禁止在生产使用？`SCAN` 游标底层原理是什么？为什么会重复或遗漏？

**直接结论**  
`KEYS *` 会一次性遍历全局哈希表所有槽位，时间复杂度 $O(N)$，在千万级 Key 下会**直接阻塞主线程数秒甚至数十秒**，引发雪崩和哨兵误判宕机！生产模糊查询必须使用渐进式 **`SCAN cursor [MATCH pattern] [COUNT count]`**。

**深度原理剖析**
1. **`SCAN` 游标核心原理：高位进位加法（逆向二进制迭代）**
   - `SCAN` 的 `cursor` 并不是从 `0 -> 1 -> 2 -> ...` 递增，而是**按二进制高位向低位进位**遍历桶索引！
   - 例如容量为 8 的哈希表，游标访问顺序为：`0(000) -> 4(100) -> 2(010) -> 6(110) -> 1(001) -> 5(101) -> 3(011) -> 7(111)`。
   - **为什么这么设计？** 当哈希表发生 **2 倍扩容**（如 8 → 16）或 **缩容**（8 → 4）时，原 3 号桶元素会精确分裂到 `3(0011)` 和 `11(1011)`，而高位迭代序下，这两个新桶恰好都排在后续未遍历区间里；从而**既能保证在扩容/缩容期间不遗漏未扫描的 Key，又最大程度减少重复扫已扫过的桶**！
2. **为什么 `SCAN` 仍可能返回重复 Key、或空列表，甚至漏掉新 Key？**
   - **可能重复**：在 `rehash` 迁移过程中或缩容合并桶时，旧桶已被扫描的元素可能落入后续未扫的新桶。因此**业务侧拿到 `SCAN` 结果后必须自行去重**。
   - **可能返回空列表**：`COUNT 100` 并不是返回 100 个匹配元素，而是**扫描 100 个哈希桶**，如果这些桶里没有命中 `MATCH` 正则或全是空桶，就会返回空数组但游标不为 0，**必须循环直到返回的游标为 `"0"` 才算结束**！
   - **迭代期间新增/删除**：如果在游标后面的桶删了或前面桶新增了，确实可能扫不到。

- **实战避坑**：同样道理，`HGETALL`、`SMEMBERS`、`ZRANGE 0 -1` 在大集合上也会阻塞主线程，必须用 **`HSCAN` / `SSCAN` / `ZSCAN`** 代替！

---

### Q17：Redis 为什么会突然变慢？线上慢查询完整排查思路是什么？

**直接结论**  
Redis 变慢通常不是因为并发太高，而是**单线程被某个操作“卡住”了**，或者**底层 OS 发生了磁盘/内存抖动**。

**系统化排查 Checklist（按优先级）**：
1. **看慢日志（`SLOWLOG GET 20`）**：检查是否有 `KEYS *`、大集合 `HGETALL`/`SMEMBERS`、`SORT`、复杂 Lua 死循环。注意：`slowlog` 只统计**命令本身在内存执行的时间**（默认 `> 10ms` 记录），不含网络传输和排队时间！
2. **查 BigKey 与阻塞删除**：是否有大 Key 写入/读取打满网卡，或用了 `DEL` 同步释放百万元素（应改用 `UNLINK` + 开启 `lazyfree-lazy-expire yes` 等 4 个异步释放参数）。
3. **查集中过期（定时删除卡顿）**：是否大量 Key 设置了同一秒过期，导致主线程 `activeExpireCycle` 占满 25% CPU 时间片循环删 Key。
4. **查持久化 `fork` 阻塞与 AOF 刷盘阻塞**：
   - `bgsave` / `bgrewriteaof` 执行 `fork()` 时，若实例内存达 20GB+，拷贝页表可能导致主线程阻塞几十到几百毫秒（检查 `INFO stats` 中的 `latest_fork_usec`）。
   - AOF 配置为 `everysec` 时，如果磁盘 I/O 极其繁忙导致后台 `fsync` 线程阻塞超过 2 秒，**主线程为了保数据安全会主动阻塞等待磁盘刷盘完成**！
5. **查系统层隐患（Swap 与 THP）**：
   - 操作系统开启了 **THP（Transparent Huge Pages，透明大页）**：Linux 默认 4KB 内存页，THP 会合并为 2MB 大页；当 `fork()` 发生写时复制（COW）时，哪怕只改 10 字节也会复制整块 2MB 内存，延迟与内存双双爆炸！**生产务必关闭 THP**。
   - 内存发生 **Swap**：用 `redis-cli --intrinsic-latency 100` 测基准延迟，若很高则排查 Swap 或云主机 CPU 争抢。

---

## 3. 持久化与高可用集群（Q18 - Q25）

### Q18：RDB 持久化原理是什么？`save` 与 `bgsave` 区别？`fork()` 写时复制（COW）如何导致内存飙升？

**直接结论**  
**RDB（Redis Database）** 是将某一时刻的内存全量数据生成紧凑的二进制快照文件（`dump.rdb`）。生产严禁使用阻塞主线程的 `save`，必须使用异步的 **`bgsave`**（依赖 OS 的 `fork()` + **写时复制 Copy-On-Write**）。

**深度原理剖析**
1. **`save` vs `bgsave`**：
   - `save`：由主线程直接写 RDB 文件，期间**完全阻塞所有客户端请求**。
   - `bgsave`：主线程 `fork` 一个子进程，子进程与主线程**共享同一份物理内存页**（只读），仅拷贝页表；随后主线程继续处理请求，子进程遍历内存写入临时 RDB 文件并原子替换旧文件。
2. **什么是写时复制（COW）以及内存飙升陷阱？**
   - `fork` 瞬间父子进程共享所有物理内存页（标为 Read-Only）。
   - 当父进程收到写请求修改某个 Key 所在内存页时，CPU 触发**缺页异常（Page Fault）**，OS 内核会将该 4KB 内存页**复制一份副本**给父进程去修改，子进程仍保留原页继续安心写 RDB。
   - **生产内存陷阱**：如果 `bgsave` 期间正好有大量写请求，最坏情况几乎所有内存页都被改写一遍，**Redis 内存占用会直接翻倍（2x）**！因此生产上部署 Redis 服务器，**物理内存一定要预留至少 30%~50% 冗余**，防止 `bgsave` 期间触发 OOM Killer！

- **面试官连环追问**：
  1. **RDB 自动触发条件 `save 900 1`、`save 300 10` 是怎么工作的？**  
     只要在 900 秒内发生至少 1 次写操作，**或** 300 秒内至少 10 次写操作，满足任一即自动触发 `bgsave`。注意：`FLUSHALL` 和主从全量同步也会触发生成 RDB。

---

### Q19：AOF 持久化原理与三种刷盘策略是什么？AOF 重写及 Redis 7.0 Multi-Part AOF 原理？

**直接结论**  
**AOF（Append Only File）** 以文本协议（RESP）追加记录每一条**写命令**，先写入 `aof_buf` 内存缓冲区，再根据 `appendfsync` 策略同步到磁盘，具备极高的数据安全性。

**深度原理剖析**
1. **三种刷盘策略（`appendfsync`）**：
   - **`always`**：每执行一条写命令就立刻调用 `fsync` 刷盘。**不丢数据，但磁盘 IO 压力巨大，吞吐量暴跌**。
   - **`everysec`（默认，生产推荐）**：写命令先写进系统 PageCache，后台独立线程每秒调用一次 `fsync`。性能极高，极端宕机**最多丢失 1 秒数据**（若磁盘卡顿超 2s 会阻塞主线程）。
   - **`no`**：何时刷盘完全交由操作系统 PageCache 调度（通常 30 秒左右）。性能最好但宕机丢数据不可控。
2. **AOF 重写（`bgrewriteaof`）原理**：
   - 为什么需要重写？因为连续执行 `INCR count` 100 次，AOF 会记录 100 条命令，而重写后只需一条 `SET count 100`！
   - **重写流程（Redis 6-）**：`fork` 子进程直接扫描当前内存数据生成精简命令（**不读旧 AOF 文件**）写入新 AOF；重写期间主线程收到的新写命令同时写入 `aof_buf` 和 **`aof_rewrite_buf`（重写缓冲区）**；子进程写完后通过管道通知主线程把 `aof_rewrite_buf` 追加到新 AOF 并原子 `rename` 覆盖。
3. **Redis 7.0 革命性升级：Multi-Part AOF**：
   - **痛点**：Redis 6- 重写期间需要双倍写缓冲区 `aof_rewrite_buf` 且主线程最后追加大量增量时可能阻塞，同时混合结构全塞在一个大文件里不好管理。
   - **Redis 7.0 方案**：将 AOF 拆分为三类文件并由一个 **`manifest` 清单文件**统一管理：
     - `base` 文件（RDB 或 AOF 格式的基线快照）
     - `incr` 文件（增量的写命令日志，重写时直接开一个新的 `incr` 文件，**彻底去掉了 `aof_rewrite_buf` 内存浪费与拷贝阻塞！**）
     - `history` 文件（已合并的旧文件，异步安全清理）。

---

### Q20：什么是混合持久化？生产中 RDB 和 AOF 应该如何选型配置？

**直接结论**  
Redis 4.0 引入、5.0+ 默认开启的**混合持久化（`aof-use-rdb-preamble yes`）**，是在 AOF 重写时将当前全量内存以 **RDB 二进制格式**写入 AOF 文件头部，把重写期间的增量命令以 **AOF 文本格式**追加在尾部。**兼顾了 RDB 的秒级重启恢复速度与 AOF 的秒级防丢能力！**

**生产选型建议**：
- **纯缓存场景（丢了能从 DB 恢复）**：可关闭持久化，或仅在从库开启 RDB 定期冷备，追求极致性能。
- **高可靠数据/热点核心业务**：**开启混合持久化（RDB + AOF `everysec` + `aof-use-rdb-preamble yes`）**，并开启 `no-appendfsync-on-rewrite yes`（重写子进程疯狂写盘时，主库暂停 `fsync` 避免磁盘 I/O 互相抢占卡死主线程，代价是重写期间若宕机可能多丢几秒数据）。
- **容灾恢复优先级**：Redis 重启时如果同时存在 RDB 和 AOF，**优先加载 AOF 文件**（因为 AOF 数据通常更完整）。

---

### Q21：主从复制全量同步与增量同步（PSYNC）原理？什么是 `replid`、`offset`、`repl_backlog_buffer`？

**直接结论**  
Redis 2.8+ 使用 **`PSYNC <replid> <offset>`** 实现主从同步，能够智能判断是走耗时耗带宽的**全量同步**，还是走轻量级的**增量同步**。

**深度原理剖析**
1. **三大核心概念**：
   - **`replid`（Replication ID，40 位随机十六进制串）**：主节点的身份标识（Redis 4.0 起从 `runid` 细化为 `replid` + `replid2` 支持故障切换后新主继承旧 ID 避免全量同步）。
   - **`offset`（复制偏移量）**：主从各自维护一个累计发送/接收字节数的单调递增整数。通过对比主从 `offset` 即可判断复制延迟进度。
   - **`repl_backlog_buffer`（复制积压缓冲区）**：主节点内存中的一个**固定大小的环形缓冲区（默认仅 1MB！）**，记录最近发送出去的写命令和对应的 `offset`。
2. **全量同步 vs 增量同步完整流程**：
   ```text
   从节点发送: PSYNC <master_replid> <offset>
        │
        ├──► 情况 A: 首次连接 (? -1) 或 replid 不匹配 或 offset 已不在环形缓冲区内
        │           └──► 主节点回复 +FULLRESYNC <replid> <offset> (全量同步)
        │                1. 主节点执行 bgsave 生成 RDB 发给从节点 (从节点清空旧数据加载 RDB)
        │                2. 生成与传输期间的新写命令缓存在 client-output-buffer-limit slave 中
        │                3. RDB 加载完毕后，主节点将缓存期间增量命令发给从节点
        │
        └──► 情况 B: replid 匹配 且 offset 依然在 repl_backlog_buffer 环形缓冲区中
                    └──► 主节点回复 +CONTINUE (增量同步)
                         直接从环形缓冲区提取 [offset+1, master_offset] 区间命令补发即可！
   ```

- **实战避坑**：默认 `repl-backlog-size` 仅 **1MB**！在高写入流量下（比如每秒 5MB 写），从节点只要网络闪断 1 秒就会被环形缓冲区覆盖，重连立即触发毁灭性的**全量同步**！**生产务必调大 `repl-backlog-size` 至 128MB~512MB**（计算公式：`断线容忍秒数 × 每秒写入流量 × 2`）。

---

### Q22：主从复制风暴与缓冲区溢出问题如何解决？无盘复制适用于什么场景？

**直接结论**  
当**一个主节点挂了过多从节点**，或多个从节点同时重启重连主节点时，主节点同时执行多个 RDB 传输和缓冲，导致**CPU、磁盘 IO、网络带宽全部打满**，这就是**主从复制风暴**。

**深度原理剖析与解法**
1. **树状主从拓扑（级联复制）**：不要让 1 个 Master 直连 10 个 Slave，而是 `Master -> Slave1 -> (Slave2, Slave3)`，让部分从节点充当中间代理分发复制流，彻底卸载 Master 带宽压力。
2. **输出缓冲区溢出死循环（`client-output-buffer-limit replica`）**：
   - 全量同步期间主节点会把增量写命令存在从节点专属输出缓冲区（默认软限制 64MB 持续 60s / 硬限制 256MB）。如果从节点磁盘太慢加载 RDB 超时导致主节点缓冲区打满，主节点会**直接断开从节点** → 从节点重连 → 再次全量同步 → **无限死循环风暴**！生产高写入场景需适当调大该硬限制（如 `512mb` / `256mb 120`）。
3. **无盘复制（`repl-diskless-sync yes`）**：
   - 传统 `bgsave` 需把 RDB 先写进主库磁盘再读出走网卡。当服务器**磁盘 I/O 很慢但网络带宽很充裕（万兆网卡）**时，开启无盘复制后，子进程直接通过 Socket 把 RDB 字节流推送给从节点，不落盘，大幅降低磁盘压力。

---

### Q23：哨兵（Sentinel）高可用原理：主观下线 vs 客观下线、Raft 选举、选主规则？

**直接结论**  
Redis Sentinel 本质是一个运行在特殊模式下的独立 Redis 集群，负责**监控（Monitoring）、通知（Notification）与自动故障转移（Automatic Failover）**，建议至少部署 **3 个奇数节点**。

**深度原理剖析**
1. **三大内部定时任务**：
   - 每 10 秒向 Master/Slave 发 `INFO` 发现整个拓扑（自动感知新加入的从库）。
   - 每 2 秒通过 `__sentinel__:hello` 频道向其他哨兵交换自身认知与 Master 状态。
   - 每 1 秒向所有节点发送 `PING` 做心跳检测。
2. **主观下线（SDOWN）与客观下线（ODOWN）**：
   - **主观下线**：单个哨兵向某节点发 `PING` 超过 `down-after-milliseconds` 未收到有效回复，则**单方面认为**它主观下线。
   - **客观下线**：只有 Master 才会被判定为客观下线。该哨兵向其他哨兵发送 `SENTINEL is-master-down-by-addr` 询问，当确认下线的哨兵票数达到 **`quorum`（法定人数，通常设为 `N/2 + 1`）** 时，正式标记为客观下线，准备故障转移。
3. **Raft 选举 Leader 哨兵（由谁来执行切换）**：
   - 判定 ODOWN 的哨兵向其他哨兵拉票，遵循“先到先得”原则，最终获得 **超过半数（`> N/2`）且 `>= quorum`** 票数的哨兵成为本次故障转移的 Leader。
4. **选拔新 Master 的 4 级严苛规则**：
   - ① 过滤掉已断线、或最近 5 秒未响应 `INFO`、或与旧 Master 断开连接超过 `down-after-milliseconds * 10` 的不健康从节点。
   - ② 比 **`replica-priority`**：优先级数值越小越优先（`0` 表示永不参选）。
   - ③ 比 **`offset`（复制偏移量）**：偏移量越大说明数据越完整越新，优先当选。
   - ④ 比 **`runid`**：前两者全同，选字典序最小的。
5. **切换收尾**：Leader 给选中从库发 `SLAVEOF NO ONE` 晋升为主库，向其余从库发 `SLAVEOF <new_master>`，通过 `__sentinel__:switch-master` 频道通知客户端更新地址，并把旧主库降级为新主的从库。

---

### Q24：哨兵模式下的“脑裂（Split-Brain）”问题是如何发生的？如何彻底兜底？

**直接结论**  
**脑裂**是指：原 Master 因网络分区（或长 GC/假死）与哨兵及从节点失联，但**依然与部分客户端保持连通**；此时哨兵误以为旧主挂了而选出新 Master，导致集群同时存在**两个可写的 Master**！一旦网络恢复，旧 Master 被强行降级并执行全量同步清空自身数据，**导致分区期间写入旧 Master 的数据全部永久丢失**！

**深度原理剖析与兜底配置**
在 `redis.conf` 中配置如下两个黄金参数（配合使用）：
```ini
# 1. 要求主节点至少连接着 1 个健康的从节点才能接收写请求（三节点一主两从设为 1；多从可设半数）
min-replicas-to-write 1

# 2. 从节点与主节点的心跳延迟不能超过 10 秒，否则视为不健康
min-replicas-max-lag 10
```
- **原理**：发生网络分区时，旧 Master 失去了与从节点的连接（健康从节点数 `< 1`），于是旧 Master **主动拒绝所有客户端写请求**，直接返回报错，从而杜绝脑裂期间出现“双写分叉丢数据”。
- **权衡代价**：此配置降低了极端情况下的可用性（CAP 中偏向 C 牺牲一点 A），如果主节点只有一个从节点且该从节点自身宕机了，主节点也会无法写入！因此生产上一主两从配合 `min-replicas-to-write 1` 是最稳妥的折中方案。

---

### Q25：Redis Cluster 切片集群原理：为什么是 16384 个槽？Gossip 协议、`MOVED` vs `ASK`？

**直接结论**  
Redis Cluster 采用**去中心化、无代理（P2P）**的切片集群架构，固定划分 **`16384`（16K）个哈希槽（Hash Slot）**，通过 `CRC16(key) & 16383` 将 Key 映射到具体槽位并分布到各主节点，实现**线性水平扩展（容量 + 写并发）**。

**深度原理剖析**
1. **为什么哈希槽是 16384 个，而不是 65536 个？（经典高频题）**
   - ① **心跳包大小**：每个节点每秒通过 Gossip 协议互发 PING/PONG 心跳，包头必须携带一个位图（Bitmap）来宣告自己负责哪些槽。`16384` 位只需 **`2KB`**（$16384 / 8$）；若用 `65536` 则需 **`8KB`**，在集群几十个节点高频心跳下会严重浪费网络带宽。
   - ② **集群规模上限**：Redis Cluster 设计规模建议不超过 **1000 个节点**（Gossip 广播风暴限制），16384 个槽分摊到 1000 个节点仍有每节点 16 个槽，完全足够均匀；且槽位越多，路由表压缩与解压消耗的 CPU 越多。
2. **Hash Tag（路由绑定技巧）**：
   - 集群下 `MGET`/`MSET` 或 Lua 脚本如果操作多个 Key 且落在不同节点会报错 `CROSSSLOT Keys in request don't hash to the same slot`！
   - **解法**：使用大括号 `{...}` 包裹共同部分，如 `order:{1001}:info` 和 `order:{1001}:items`，Redis 只对 `{1001}` 计算 CRC16，强制将它们分配到同一个槽！
3. **`MOVED` 重定向 vs `ASK` 重定向**：
   - **`MOVED`（永久路由更新）**：槽位明确归属于另一个节点。客户端收到 `-MOVED 3999 127.0.0.1:7002` 后，**会更新本地路由缓存表**并重定向请求。
   - **`ASK`（临时迁移重定向）**：发生在**在线扩缩容槽迁移（Resharding）过程中**！某个槽一部分 Key 已迁到新节点，另一部分还在原节点：客户端先问原节点，若找不到则返回 `-ASK 3999 127.0.0.1:7002`。客户端收到 `ASK` 后，**不更新本地槽缓存**，仅对本次请求先向目标节点发 `ASKING` 命令再发真实命令。

- **面试官连环追问**：
  1. **集群中某个主节点挂了且没有可用从节点，整个集群还能用吗？**  
     取决于配置 `cluster-require-full-coverage`：默认为 **`yes`**，只要有一段槽没人负责，**整个集群都会拒绝服务**；生产上如果允许局部业务降级可用，建议设为 **`no`**，让剩余健康槽位继续提供服务！
- **实战避坑**：不要过度滥用相同的 Hash Tag（如所有 Key 都加 `{common}`），否则全部数据会倾斜打爆单一分片！

---

## 4. 缓存三大难题与双写一致性（Q26 - Q30）

### Q26：什么是缓存穿透？布隆过滤器原理、实现与删除缺陷怎么解？

**直接结论**  
**缓存穿透**是指大量请求查询**缓存和数据库中都不存在的数据**（例如恶意攻击伪造 `id = -99999` 或随机不存在的订单号），请求每次都穿透 Redis 直接轰击 MySQL 导致数据库被打挂。

**深度解决方案**
1. **方案一：接口层合法性校验 + 缓存空对象（Null Caching）**
   - 对明显非法的 ID（负数、不符合雪花/UUID 格式）直接在网关/Controller 拦截。
   - DB 查不到时，向 Redis 写入一个空占位符（如 `""` 或 `"NULL"`）并设置较短的过期时间（如 `2~5` 分钟，加上随机抖动），拦截短期内的重复穿透。
   - **缺点**：如果攻击者每次随机变换上千万个不存在的 Key，缓存空对象会撑爆 Redis 内存！
2. **方案二：布隆过滤器（Bloom Filter）前置拦截（千万级穿透终极杀器）**
   - **底层原理**：一个长度为 $m$ 的二进制 Bit 数组 + $k$ 个独立的哈希函数。
     - **添加元素**：经过 $k$ 个哈希计算得到 $k$ 个数组下标，全部置为 `1`。
     - **查询元素**：计算这 $k$ 个下标——**只要有一个为 `0`，该元素 100% 绝对不存在！** 如果全为 `1`，则**可能存在**（因为其他元素的哈希冲突也可能凑齐这几个 `1`，即存在可控的**误判率**）。
   - **Java 实战（Redisson `RBloomFilter`）**：
     ```java
     RBloomFilter<Long> bloomFilter = redissonClient.getBloomFilter("product:bloom");
     // 预期数据量 100万，误判率 1% (0.01) -> 自动算出约需 958万 bit (≈1.14MB) 与 7 个哈希函数
     bloomFilter.tryInit(1_000_000L, 0.01);
     bloomFilter.add(10086L);
     if (!bloomFilter.contains(requestId)) {
         return null; // 绝对不存在，直接挡回，零 DB/缓存开销！
     }
     ```
   - **布隆过滤器的缺陷与解法**：
     - **不支持删除**：把某位清零会影响其他共享该位的元素。解法：① 定期全量重建；② 改用 **Counting Bloom Filter（计数布隆，每位改用 4bit 计数器，内存扩大 4 倍）** 或 **Cuckoo Filter（布谷鸟过滤器，支持删除且空间效率高）**。
     - **不可动态扩容**：初始化预估量要留足 2~3 倍冗余。

---

### Q27：什么是缓存击穿？互斥锁与逻辑过期代码如何手写？

**直接结论**  
**缓存击穿**是指某一个**超高并发的热点 Key（如爆款秒杀商品）在缓存过期的那一瞬间**，成千上万个并发请求同时发现缓存未命中，瞬间全部直接穿透打向数据库。

**两种核心解法与生产级代码**
1. **方案一：分布式互斥锁 + DCL 双重检查（强一致性，会短暂阻塞）**
   ```java
   public Product getProductWithMutex(Long id) {
       String cacheKey = "product:" + id;
       String json = stringRedisTemplate.opsForValue().get(cacheKey);
       if (StringUtils.hasText(json)) {
           return JSON.parseObject(json, Product.class);
       }
       String lockKey = "lock:product:" + id;
       RLock lock = redissonClient.getLock(lockKey);
       try {
           if (lock.tryLock(3, 10, TimeUnit.SECONDS)) {
               try {
                   // DCL (Double-Check Locking) 二次查缓存，防止前一个线程刚重建完自己又去查 DB！
                   json = stringRedisTemplate.opsForValue().get(cacheKey);
                   if (StringUtils.hasText(json)) {
                       return JSON.parseObject(json, Product.class);
                   }
                   Product dbVal = productMapper.selectById(id);
                   // 即使 DB 为 null 也写入空值防穿透
                   stringRedisTemplate.opsForValue().set(cacheKey,
                       dbVal == null ? "" : JSON.toJSONString(dbVal),
                       Duration.ofMinutes(dbVal == null ? 2 : 30));
                   return dbVal;
               } finally {
                   if (lock.isHeldByCurrentThread()) lock.unlock();
               }
           }
           // 未抢到锁可短睡眠后重试或降级返回兜底数据
           Thread.sleep(50);
           return getProductWithMutex(id);
       } catch (InterruptedException e) {
           Thread.currentThread().interrupt();
           throw new RuntimeException(e);
       }
   }
   ```
2. **方案二：逻辑过期 + 异步重建（高可用、零等待，允许短暂返回旧数据）**
   - Redis Key 本身**不设置物理 TTL（永不过期）**，而是在 Value JSON 中封装一个 `expireTime` 字段。
   - 线程读取时发现 `expireTime < now`，先尝试抢分布式互斥锁；**抢到锁的线程丢给独立线程池去查 DB 异步刷新缓存，然后本线程和所有未抢到锁的并发线程全部直接先返回旧数据！**（绝不阻塞前端吞吐）。

---

### Q28：什么是缓存雪崩？如何设计全链路防御体系？

**直接结论**  
**缓存雪崩**分为两类：**① 大量 Key 在同一时间集体过期**（如凌晨统一批量预热全设为 24 小时过期）；**② Redis 整个集群突然宕机**，导致所有缓存请求如雪崩般压垮 MySQL，引发连锁服务雪崩。

**事前、事中、事后三层全链路防御**：
1. **事前防集体过期 + 高可用**：
   - **随机 TTL 打散**：基础过期时间 + 随机抖动（如 `30min + Random(0~10min)`）。
   - **Redis 高可用集群**：哨兵主从或 Redis Cluster 跨机房容灾；重要活动提前**缓存预热**。
2. **事中防打死 DB**：
   - **本地多级缓存（Caffeine）**：即使 Redis 整体宕机，应用内存的本地缓存依然能兜住 80%+ 热点读流量。
   - **限流与熔断降级（Sentinel / Resilience4j）**：对回源 DB 的接口做 QPS 限流和慢调用比例熔断，宁可部分请求快速降级返回兜底页，也绝不把数据库打死。
3. **事后快速恢复**：
   - 依赖 Redis RDB + AOF 混合持久化快速拉起恢复集群，配合异步预热脚本重新灌入热点数据。

---

### Q29：缓存与数据库双写一致性：为什么是“先更新 DB 再删缓存”？为什么删缓存而不是更新缓存？

**直接结论**  
工业界最基础且通用的标准模式是 **Cache-Aside Pattern（旁路缓存模式）**：**读时先查缓存，未命中读 DB 再回填；写时先更新 DB，再删除缓存**。

**深度原理拆解（两个核心灵魂拷问）**
1. **为什么是“删除缓存”而不是“更新缓存”？**
   - **并发覆盖脏数据**：线程 A 先更新 DB 为 10，线程 B 后更新 DB 为 20，但因为网络抖动，线程 B 先更新缓存为 20，线程 A 后更新缓存为 10 → **缓存永久错成旧值 10**！
   - **无效计算浪费（懒加载思想）**：如果某条数据写多读少（比如 1 分钟修改 100 次才读 1 次），每次写都重新序列化拼装复杂大对象去更新缓存纯属浪费，删掉让下一次读按需加载才最合理。
2. **为什么是“先更新 DB，再删缓存”，而不是“先删缓存，再更新 DB”？**
   - **反例（先删缓存再更新 DB）的极高概率并发 Bug**：
     线程 A（写）先删掉缓存 → 还没来得及更新 DB → 线程 B（读）恰好进来发现缓存没了，去 DB 读到**旧值 10** → 线程 A 更新 DB 为 20 → 线程 B 把刚才读到的**旧值 10 回填进缓存**！由于写 DB 耗时（几毫秒）远高于读缓存/删缓存（微秒级），这种场景**极易大概率发生**！
   - **正例（先更新 DB 再删缓存）为什么安全得多？**
     它唯一出错的极端路径是：缓存恰好自己失效 → 线程 B（读）去 DB 读到旧值 10 → **在线程 B 还没把 10 写回缓存的间隙**，线程 A（写）完成了“更新 DB 为 20 + 删缓存”的全套动作 → 最后线程 B 把旧值 10 写回缓存。
     **因为数据库写操作耗时远大于一次内存读操作**，线程 A 几乎不可能比线程 B 慢，因此该冲突在实际生产中概率极低（且可再给缓存设置兜底 TTL）。

---

### Q30：高并发下如何实现更强的最终一致性与强一致性？（延迟双删 vs Canal+Binlog+MQ）

**直接结论**  
单纯的“先更新 DB 再删缓存”仍有两个潜在缺口：**① 第二步“删缓存”本身可能因网络抖动失败**；**② 极低概率的读写并发旧值回填**。生产上有以下进阶解法：

1. **延迟双删（早期妥协方案，有缺陷）**：
   - 流程：`删缓存 -> 更新 DB -> Thread.sleep(200~500ms) -> 再删缓存`。
   - **缺陷**：`sleep` 多久全靠猜！写请求阻塞吞吐量下降，且如果第二次删缓存失败依然不一致；如果改用异步 MQ 延迟消息发第二次删，复杂度高。
2. **Canal + Binlog + MQ 异步删缓存（工业界最推荐主流架构）**：
   ```text
   业务服务 ──► 更新 MySQL (只需写 DB，完全解耦 Redis!)
                    │ 产生 Binlog
                    ▼
   Canal (伪装成 MySQL 从库订阅 Binlog) ──► 投递到 RocketMQ / Kafka
                                                │ 消费删缓存 (失败则 MQ 自动重试!)
                                                ▼
                                          DEL Redis Key
   ```
   - **核心优势**：① **彻底解耦**业务代码与缓存删除逻辑；② **保证可靠性**（依托 MQ 重试机制，删缓存哪怕当时 Redis 抖动失败也能在几秒内重试成功，达成可靠最终一致性）；③ **无业务侵入与主链路零耗时**。
3. **如果业务要求“绝对强一致性”（如金融账户余额，绝不容忍毫秒级读旧值）怎么办？**
   - **方案 A**：强一致核心数据**直接读主库 DB，不走缓存**（最彻底）。
   - **方案 B**：使用分布式读写锁（Redisson `RReadWriteLock`），读加共享锁、写加排他锁（牺牲并发换强一致）。
   - **方案 C**：单线程写后自己只更新本地会话或路由到**主库/内存队列串行化**。

---

## 5. 分布式锁、事务、Lua 与生产治理（Q31 - Q40）

### Q31：如何手写一个生产级 Redis 分布式锁？有哪些潜在陷阱？

**直接结论**  
手写最简可靠的 Redis 分布式锁必须同时满足 **“原子加锁 + 唯一防误删标识 + TTL 防死锁 + Lua 原子释放”** 四要素，缺一不可。

**核心指令与代码**：
```lua
-- 1. 加锁：必须一条命令完成 NX 与 PX，绝不能 SETNX 后再 EXPIRE（中间宕机会死锁！）
-- uuid_threadId 作为唯一值，防止 A 线程业务超时锁过期后，把 B 线程刚加的锁误删！
SET lock:order:1001 "uuid-thread-1" NX PX 30000

-- 2. 解锁 Lua 脚本：先校验归属权，再删除（保证判断与删除两步在 Redis 内原子执行）
if redis.call('get', KEYS[1]) == ARGV[1] then
    return redis.call('del', KEYS[1])
else
    return 0
end
```
- **手写锁的三大不足（为什么生产推荐 Redisson）**：
  1. **不可重入**：同一线程嵌套调用带同一把锁的方法会自己把自己锁死。
  2. **锁时间写死无法自动续期**：30 秒业务没跑完锁提前释放，并发安全直接击穿。
  3. **抢锁失败只能盲目 `sleep` 自旋轮询**：空耗 Redis CPU 与连接数。

---

### Q32：Redisson 分布式锁源码级原理：可重入、WatchDog 自动续期、信号量唤醒？

**直接结论**  
Redisson 底层通过 **Redis Hash 结构 + 精细 Lua 脚本 + Netty `HashedWheelTimer` 时间轮（WatchDog）+ Redis Pub/Sub 订阅唤醒**，完美解决了手写锁的所有缺陷。

**深度原理剖析**
1. **可重入原理（Hash 结构存储）**：
   - Key：`lock_name`
   - Field：`UUID:threadId`（当前客户端实例 + 线程 ID）
   - Value：`重入次数`（每次重入 `HINCRBY ... 1`，每次解锁 `HINCRBY ... -1`，减到 `0` 时执行 `DEL` 并向 Pub/Sub 频道广播解锁消息）。
2. **WatchDog（看门狗）自动续期机制**：
   - **触发前提**：调用 `lock.lock()` 或 `lock.tryLock(waitTime, -1, unit)`，即**不显式指定 `leaseTime`（传入 `-1`）**！如果显式传了如 `leaseTime = 10s`，则到点强制过期，**绝对不会启动 WatchDog**！
   - **续期流程**：默认锁过期时间 `internalLockLeaseTime = 30s`。加锁成功后，Redisson 启动一个时间轮后台定时任务，每隔 **`30 / 3 = 10` 秒**检查当前线程是否还持有锁（通过 Lua 判断 `HEXISTS`）；若还在持有，就执行 `PEXPIRE` 把过期时间重新重置为 30 秒，直到业务主动 `unlock()` 或当前 JVM 进程宕机（JVM 挂了心跳停了，30 秒后锁自动安全释放）。
3. **失败等待不空转（Pub/Sub 信号量机制）**：
   - 未抢到锁的线程不会傻傻 `while(true)` 轮询 Redis，而是订阅 `redisson_lock__channel:{lock_name}`，并借助 JDK `Semaphore.tryAcquire(ttl)` 挂起等待；一旦持锁线程释放锁发布广播消息，等待线程立刻被唤醒重新争抢（避免无意义网络风暴）。

- **实战避坑**：使用 `tryLock(10, 30, TimeUnit.SECONDS)` 传了第二个参数 `30` 就失去了看门狗续期！若既要等待超时又要看门狗保护，应该用 `tryLock(10, -1, TimeUnit.SECONDS)` 或无参 `lock()`；另外 `unlock()` 前务必判断 `if (lock.isLocked() && lock.isHeldByCurrentThread())`，否则锁超时自动释放后调用 `unlock()` 会抛 `IllegalMonitorStateException`！

---

### Q33：什么是 Redlock（红锁）算法？业界核心争议是什么？Redis 锁 vs ZooKeeper 锁如何选型？

**直接结论**  
在主从模式下，如果客户端 A 在 Master 加锁成功，还没来得及异步同步给 Slave，Master 就宕机了，Slave 晋升为新 Master 后**根本不知道这把锁的存在**，客户端 B 也能加锁成功，导致双锁并发！为此 Redis 作者 Antirez 提出了 **Redlock**，但在分布式理论界引发了巨大争议。

**深度原理与争议**
1. **Redlock 核心做法**：
   - 部署 $N$ 台（如 **5 台**）完全独立的 Redis Master（**不要主从**），客户端记录开始时间戳，依次向 5 台实例发相同的 `SET NX PX` 请求（单台设极短超时如 50ms）；如果至少在 **半数以上（$\ge 3$ 台）** 加锁成功，且总耗时小于锁有效时间 `TTL`，才算加锁成功；否则向所有节点发解锁脚本回滚。
2. **Martin Kleppmann 对 Redlock 的致命质疑（经典面试必杀答法）**：
   - **GC STW（Stop-The-World）问题**：客户端 1 拿到锁后发生长达几十秒的 Full GC 停顿，期间锁早已过期被客户端 2 拿走；GC 恢复后客户端 1 以为自己还拿着锁，照样写入共享资源（需引入 ZooKeeper/DB 的单调递增 **Fencing Token** 才能彻底防住，而普通锁做不到）。
   - **系统时钟跳跃问题**：Redlock 严重依赖各节点本地物理时钟流逝，如果某节点发生 NTP 时钟回拨/跳跃，锁会提前过期。
3. **Redis 锁 vs ZooKeeper / Etcd 锁如何选型？**
   - **追求高并发吞吐（10w+ TPS）、允许极罕见主从切换掉锁兜底（如防重复提交、库存扣减配合 DB 乐观锁）** → 选 **Redis（Redisson）**。
   - **强一致性绝对不能掉锁（如分布式调度选主、金融级核心排他资源）** → 选 **ZooKeeper（临时顺序节点 + Watch）** 或 **Etcd（Lease + Revision Fencing Token）**。

---

### Q34：Redis 事务（`MULTI`/`EXEC`/`WATCH`）如何实现？为什么不支持回滚？

**直接结论**  
Redis 事务通过 **`MULTI`（开启）→ 命令入队 → `EXEC`（批量原子执行）** 实现，能保证队列里的命令按顺序**串行、不被其他客户端打断**地执行，**但 Redis 事务不具备传统数据库 ACID 的原子性（发生运行时错误不会回滚！）**。

**深度原理剖析**
1. **两种错误的截然不同表现**：
   - **入队期语法错误**（如命令拼错 `SETT`）：整个事务队列在 `EXEC` 时直接报错放弃，**全都不执行**（等价于回滚）。
   - **执行期运行时错误**（如对 String 类型的 Key 执行 `INCR`）：错误的那一条命令报错跳过，**其余正确的命令依然全部正常执行提交，绝不会回滚**！
2. **为什么 Redis 官方坚决不支持回滚？**
   - 作者认为运行时错误只可能是程序员代码 Bug，在生产环境中不该出现；支持回滚机制（记录 Undo Log）会**极大拖累 Redis 简洁高速的设计哲学**。
3. **`WATCH` 实现乐观锁（CAS）**：
   - 执行 `MULTI` 之前先执行 `WATCH key1`。如果从 `WATCH` 到 `EXEC` 之间，有其他客户端修改了 `key1`，那么当前客户端执行 `EXEC` 时会直接返回 `nil`（事务被打断失败），业务侧捕获后重试即可。

---

### Q35：Pipeline（管道）与 Lua 脚本、Redis 7.0 Functions 的区别是什么？

**直接结论**  
三者都能**将多次命令合并为一次网络往返（RTT）**大幅提升吞吐，但在**原子性**与**逻辑判断能力**上有本质区别：

| 特性维度 | Pipeline（管道） | Lua 脚本（`EVAL` / `EVALSHA`） | Redis 7.0 Functions |
| :--- | :--- | :--- | :--- |
| **执行原子性** | **不保证原子性**（多条命令在服务端可能穿插其他客户端命令） | **绝对原子性**（单线程一口气跑完整个脚本） | **绝对原子性**（同 Lua，但是服务端持久化函数库） |
| **依赖前一步结果** | **不支持**（一次发过去，无法用第1步返回值做 if 判断） | **支持**（如检查库存够了再扣减） | **支持** |
| **存储与主从复用** | 无状态客户端打包 | 客户端每次传脚本或靠 `SCRIPT LOAD` 缓存易丢失（重启/切换需重传） | **原生持久化在 RDB/AOF 并同步到从库**，像存储过程一样调用 `FCALL` |

- **Lua 生产禁忌**：① 脚本里绝不允许写死循环或耗时操作（会彻底卡死整个 Redis，默认超 5 秒触发 `lua-time-limit` 阻断）；② 务必提前提取好所有操作 Key 放进 `KEYS[]` 数组传参（不要在脚本内拼接 Key 名，否则 Cluster 集群无法校验槽位归属）。

---

### Q36：什么是大 Key（BigKey）？如何发现与安全清理？

**直接结论**  
**大 Key 并非指 Key 名称长，而是指对应的 Value 占用空间过大或元素过多**。业界通常判定标准：String 类型 Value **$> 10\text{KB}$**；Hash/List/Set/ZSet 元素个数 **$> 5000$** 或总内存 **$> 10\text{MB}$**。

**危害与治理方案**：
1. **核心危害**：读取/序列化打满网卡带宽；阻塞单线程造成其他请求超时；Cluster 模式下各分片内存严重倾斜；过期淘汰或 `DEL` 时瞬间阻塞主线程数秒。
2. **发现手段**：
   - **`redis-cli --bigkeys`**：基于 `SCAN` 扫全库输出各类型 Top1 大 Key（低侵入，但建议在**从节点**跑）。
   - **`MEMORY USAGE key`**：精确查看某 Key 真实字节占用。
   - **离线分析 RDB 工具（如 `rdbtools`）**：生产最安全，零线上影响。
3. **安全删除与治理**：
   - **绝对禁止直接 `DEL` 大 Key！**
   - **对于 String 大 Key**：直接用 **`UNLINK key`**（异步放到后台线程释放内存）。
   - **对于巨型集合（百万元素 Hash/ZSet）**：即使是 `UNLINK` 在百万节点上释放也要先分批：`HSCAN` / `SSCAN` / `ZREMRANGEBYRANK` 每次删 500 条，循环删完后再 `UNLINK`。
   - **配置异步淘汰防抖**：在 `redis.conf` 开启 `lazyfree-lazy-eviction yes`、`lazyfree-lazy-expire yes`、`lazyfree-lazy-server-del yes`、`replica-lazyflush yes`。
   - **业务拆分**：大 Hash 按 `hash(field) % 100` 拆成 100 个小 Hash；大 String 启用 GZIP/LZ4 压缩后再存。

---

### Q37：什么是热 Key（HotKey）？如何实时发现？多级缓存与热 Key 分片如何落地？

**直接结论**  
**热 Key** 是指在极短时间内（如突发微博热搜、头部主播带货秒杀），某少数几个 Key 的访问 QPS 暴涨到数万甚至数十万，直接**打爆所在单个 Redis 分片的 CPU 或网卡带宽**（即使 Cluster 集群有 100 个节点也救不了单个热点分片）。

**发现与治理方案**：
1. **如何发现？**
   - **`redis-cli --hotkeys`**：仅在淘汰策略为 `LFU` 时可用，扫描全库看 `logc` 最高者（偏静态历史统计）。
   - **客户端/SDK 本地滑动窗口采集（推荐，如京东 `hotkey` 框架）**：在客户端/代理层异步统计高频 Key，超阈值主动上报并推下发到本地内存。
   - **Redis `MONITOR` 短时采样**（慎用，严重损耗性能，只开几百毫秒立即关闭）。
2. **两大终极治理武器**：
   - **武器一：本地多级缓存（Caffeine + Redis）**：一旦识别为热 Key，直接推送到业务应用节点的 **JVM 本地 Caffeine 缓存**中，拦截 99.9% 的重复读请求，Redis 瞬间回归平静。
   - **武器二：热 Key 分片复制（读扩散）**：把热 Key 冗余写 $N$ 份分散到 Cluster 不同槽位（如 `hot:item:1001_0` ~ `hot:item:1001_9`），读取时客户端随机拼接 `_0 ~ _9` 后缀，将单分片压力均摊到 10 个节点（缺点是更新时需同步更新这 10 份）。

---

### Q38：如何用 Redis 实现延迟队列？（ZSet vs Redisson `RDelayedQueue` vs 过期通知的坑）

**直接结论**  
生产中**绝对禁止**使用 Redis Keyspace Notifications（`__keyevent@0__:expired` 键空间过期通知）做可靠延迟队列！最稳妥的 Redis 延迟队列是 **ZSet + Lua** 或直接使用 **Redisson `RDelayedQueue`**。

**三种方案深度对比**：
1. **为什么严禁使用“Key 过期事件通知”？**
   - ① **时间极不准**：Redis 是惰性 + 定期删除，Key 到期后可能延迟几分钟才真正被清理并触发通知。
   - ② **极易丢消息**：底层基于 `Pub/Sub`，客户端断线重启期间的过期事件**永久丢失**，且多实例消费会重复触发。
2. **ZSet 方案原理**：
   - 将任务内容作为 `member`，**到期时间戳**作为 `score` 执行 `ZADD delay_queue <triggerTime> <taskJson>`。
   - 后台线程周期性通过 **Lua 脚本**执行 `ZRANGEBYSCORE delay_queue 0 <now> LIMIT 0 10`，拿到后在同一个 Lua 脚本里立刻调用 `ZREM` 删掉并返回（**保证分布式多消费者下每条任务只被抢到一次，绝不重复消费**）。
3. **Redisson `RDelayedQueue` 源码原理**：
   - 底层本质就是用了一个 **ZSet 存到期时间** + 一个 **List 存到期后的待消费任务** + **Redisson Pub/Sub 精准通知唤醒**（把到期时间通过 `ZRANGE` 查首元素算出最近倒计时，让客户端时间轮定时触发 Lua 把到期元素从 ZSet 搬运到目标阻塞队列 List 中），既省去高频无效空轮询，又保证多实例不重不漏！

---

### Q39：如何用 Redis + Lua 实现分布式限流？（滑动窗口与令牌桶完整 Lua 脚本）

**直接结论**  
分布式环境下单机 Guava `RateLimiter` 无法全局生效，通常采用 **Redis + Lua 脚本** 实现**滑动窗口限流**（基于 ZSet）或**令牌桶限流**（基于 Hash，可应对突发脉冲流量）。

**核心实战代码（Lua 脚本）**：
1. **滑动窗口限流（精确统计最近 $W$ 毫秒内最多允许 $N$ 次请求）**：
   ```lua
   -- KEYS[1]: 限流 Key (如 rate:api:login:uid100)
   -- ARGV[1]: 当前毫秒时间戳, ARGV[2]: 窗口大小(ms), ARGV[3]: 最大阈值, ARGV[4]: 本次唯一请求ID
   local key = KEYS[1]
   local now = tonumber(ARGV[1])
   local window = tonumber(ARGV[2])
   local limit = tonumber(ARGV[3])
   local reqId = ARGV[4]

   -- 1. 清理窗口期之前的过期记录
   redis.call('ZREMRANGEBYSCORE', key, 0, now - window)
   -- 2. 统计当前窗口内请求总数
   local count = redis.call('ZCARD', key)
   if count >= limit then
       return 0 -- 触发限流
   end
   -- 3. 放行：写入当前请求并续期 Key 防止冷数据残留
   redis.call('ZADD', key, now, reqId)
   redis.call('PEXPIRE', key, window)
   return 1
   ```
   - **注意**：`ZSet` 滑动窗口会记录窗口内每一次请求成员，QPS 几十万的超大接口会占大量内存！超大规模限流应改用 **令牌桶算法**。
2. **令牌桶限流（只存 2 个字段，空间复杂度 $O(1)$！）**：
   ```lua
   -- KEYS[1]: 令牌桶 Key
   -- ARGV[1]: 桶容量 capacity, ARGV[2]: 每秒生成速率 rate, ARGV[3]: 当前时间戳(秒,支持小数), ARGV[4]: 申请令牌数
   local bucket = KEYS[1]
   local capacity = tonumber(ARGV[1])
   local rate = tonumber(ARGV[2])
   local now = tonumber(ARGV[3])
   local requested = tonumber(ARGV[4])

   local info = redis.call('HMGET', bucket, 'tokens', 'last_time')
   local tokens = tonumber(info[1])
   local last_time = tonumber(info[2])
   if tokens == nil then
       tokens = capacity
       last_time = now
   else
       local delta = math.max(0, now - last_time)
       tokens = math.min(capacity, tokens + delta * rate)
   end

   if tokens < requested then
       return 0 -- 令牌不足，拒绝
   end
   tokens = tokens - requested
   redis.call('HMSET', bucket, 'tokens', tokens, 'last_time', now)
   redis.call('EXPIRE', bucket, math.ceil(capacity / rate) * 2)
   return 1
   ```

---

### Q40：多级缓存架构（Caffeine + Redis）如何保持本地缓存一致性？客户端选型 Jedis vs Lettuce？

**直接结论**  
在百万级 QPS 架构中，常采用 **L1（JVM 本地 Caffeine）+ L2（分布式 Redis）** 多级缓存。解决 L1 本地缓存多节点数据不一致的核心思路是：**短 TTL 兜底 + 基于 Redis Pub/Sub 或 RocketMQ 广播模式主动失效**。

**深度解析与选型**
1. **多级缓存同步链路**：
   - 当某条核心数据更新 DB → 删除 Redis 后，向 **RocketMQ 广播主题（或 Redis Pub/Sub）** 发送一条 `{"key": "product:1001"}` 通知。
   - 集群内所有微服务实例都会收到该广播消息，立刻调用 `caffeineCache.invalidate("product:1001")` 剔除各自内存中的旧值！
   - **兜底**：即使某个节点网络抖动漏掉通知，Caffeine 配置了 `expireAfterWrite(30, TimeUnit.SECONDS)` 也能在 30 秒后自动过期。
   - **为什么推荐 Caffeine 而不是 Guava Cache？** Caffeine 采用更先进的 **W-TinyLFU** 淘汰算法（结合了 LRU 和 LFU 优点）以及异步 RingBuffer 缓冲区，命中率和并发吞吐量全面碾压 Guava Cache。
2. **Java 客户端选型：Jedis vs Lettuce vs Redisson**：
   - **Jedis**：基于传统阻塞式 BIO，**连接非线程安全**，必须依赖 Apache Commons Pool 连接池（每线程拿一个连接），高并发下连接池耗尽风险高，但 API 直观、轻量。
   - **Lettuce（Spring Boot 2.x/3.x 默认）**：基于 **Netty NIO 异步非阻塞通信**，**单个物理连接即可被多个业务线程安全共享**（底层请求靠单 TCP 复用流水线排队），极大节省 Redis 服务端连接数；但存在坑：高并发下不要在共享连接上做阻塞操作（如 `BLPOP` / 事务）否则会阻塞整个连接上的其他线程，此类操作需单独开启连接池或独立连接。
   - **Redisson**：在 Netty 基础上重点强化了**分布式对象、分布式锁、布隆过滤器、延迟队列、限流器**等高级并发构件。日常 CRUD 用 `StringRedisTemplate`（Lettuce），分布式高级同步构件必用 `Redisson`。

- **实战避坑**：使用 Lettuce 时若出现 `RedisCommandTimeoutException`，高概率是因为开启了集群拓扑自动刷新前未启用 `ClusterTopologyRefreshOptions`（主从切换后客户端未及时感知最新槽位分布导致一直打旧节点），生产务必配置自适应 + 周期性拓扑刷新！
