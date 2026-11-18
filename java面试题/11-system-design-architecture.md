# 11 - 高并发项目架构与系统设计实战面试题

> 本章覆盖：系统设计通用方法论与容量估算、12 大高频端到端系统设计实战（秒杀、短链、抢红包、订单与支付、Feed 信息流、IM 即时通讯、分布式任务调度、统一限流与灰度发布、海量数据 TopK/去重、多级缓存与 Canal Binlog 同步）、云原生 Docker/K8s 部署与可观测性、以及 2025-2026 前沿 AI 工程化架构（Spring AI / RAG / MCP）。

---

## 目录导航

- [0. 系统设计通用答题框架与容量估算（面试前必背）](#0-系统设计通用答题框架与容量估算面试前必背)
- [1. 高并发秒杀系统全链路架构](#1-高并发秒杀系统全链路架构)
- [2. 高并发短链生成系统（Short URL）](#2-高并发短链生成系统short-url)
- [3. 高并发微信抢红包系统](#3-高并发微信抢红包系统)
- [4. 电商订单与支付状态机（防重、超时关单、对账补偿）](#4-电商订单与支付状态机防重超时关单对账补偿)
- [5. 信息流 / 朋友圈 Feed 流系统](#5-信息流--朋友圈-feed-流系统)
- [6. 高并发即时通讯（IM）系统](#6-高并发即时通讯im系统)
- [7. 分布式任务调度系统（XXL-JOB vs ElasticJob）](#7-分布式任务调度系统xxl-job-vs-elasticjob)
- [8. 统一限流与全链路灰度发布系统](#8-统一限流与全链路灰度发布系统)
- [9. 海量数据处理（Bitmap / 布隆过滤器 / TopK / HyperLogLog）](#9-海量数据处理bitmap--布隆过滤器--topk--hyperloglog)
- [10. 多级缓存 + Canal Binlog 异步一致性架构](#10-多级缓存--canal-binlog-异步一致性架构)
- [11. 云原生部署实战（Docker / K8s / 优雅上下线 / 可观测性）](#11-云原生部署实战docker--k8s--优雅上下线--可观测性)
- [12. 2025-2026 前沿趋势：Java 后端 AI 工程化（Spring AI / RAG / MCP）](#12-2025-2026-前沿趋势java-后端-ai-工程化spring-ai--rag--mcp)

---

## 0. 系统设计通用答题框架与容量估算（面试前必背）

面对开放式系统设计题（如“设计一个秒杀 / 短链 / 微博系统”），切忌一上来就堆砌组件名词，推荐采用 **4 步拆解法（45 分钟标准节奏）**：

1. **明确需求与边界（5 min）**：区分功能性需求（核心主流程）与非功能性需求（QPS、可用性 99.99%、一致性强一致还是最终一致、P99 延迟 < 100ms）。
2. **容量估算（5 min）**：用 **二八定律** 计算平均 QPS 与峰值 QPS，估算存储空间、内存与网络带宽。
   - **常用换算基准**：
     - 1 天 = $86400 \text{ s} \approx 10^5 \text{ s}$；100 万 DAU，人均每天 20 次请求 $\Rightarrow$ 平均 QPS $\approx \frac{2 \times 10^7}{86400} \approx 230$；峰值（按 5~10 倍或二八定律集中在峰值时段）约 $1500 \sim 2500$ QPS。
     - **单机极限经验值（工程估算用）**：
       - **Nginx / API Gateway**：单机 3 万 ~ 10 万 QPS
       - **Spring Boot Tomcat（常规 CRUD）**：单机 1000 ~ 3000 QPS
       - **Redis（单分片）**：读 8 万 ~ 10 万 QPS，写 5 万 ~ 8 万 QPS
       - **MySQL（InnoDB，16C32G SSD）**：单机读 5000 ~ 1 万 QPS，写（含事务/索引）1000 ~ 3000 TPS
       - **Kafka / RocketMQ**：单 Broker 写 5 万 ~ 10 万+ TPS
       - **带宽换算**：10 万 QPS $\times$ 单包 2KB = $200\text{ MB/s} = 1.6\text{ Gbps}$（注意千兆网卡极限仅 1Gbps $\approx$ 125MB/s，需升级万兆网卡或横向扩容接入层）。
3. **核心架构与数据模型设计（20 min）**：画出端到端架构图（客户端 $\rightarrow$ 网关 $\rightarrow$ 业务服务 $\rightarrow$ 缓存/MQ $\rightarrow$ DB），给出核心表结构与索引设计、关键交互时序。
4. **瓶颈突破与高可用演进（15 min）**：主动分析**热点倾斜、缓存雪崩、数据一致性、分库分表、限流降级、多活容灾**等 Trade-off（架构取舍）。

---

## 1. 高并发秒杀系统全链路架构

### Q1：如何从零设计一个支撑百万 QPS 的高并发秒杀系统？

**核心设计哲学**：**读多写少、动静分离、流量漏斗（越上游拦截越多无效流量）、热点隔离、异步削峰、防超卖**。

#### 1. 容量估算与全链路流量漏斗架构

- **假设规模**：100 万用户同时抢购 1000 件茅台，瞬时峰值 QPS 达到 100 万，单机 MySQL 绝对会瞬间崩溃。通过“流量漏斗”层层削减，最终打到 MySQL 的有效写请求仅 1000 笔。

```text
[用户请求 100万 QPS]
       │
       ▼ 1. 客户端 & CDN 层（拦截 ~60% 流量）
  ├── 静态资源（商品图/CSS/JS/活动页HTML）全量推 CDN 边缘节点，不回源站
  ├── 前端按钮点击后置灰 + 倒计时防连点 + 局部随机延迟(0~500ms)打散瞬时尖刺
  └── 答题/滑块验证码（防机器脚本 + 将瞬时脉冲拉平到 2~3 秒）
       │  (剩余 ~40万 QPS)
       ▼ 2. 接入层 & 网关层（拦截 ~30% 恶意/超限流量）
  ├── LVS / Nginx 四七层负载均衡 + OpenResty(Lua) IP/UID 黑名单与单机限流
  └── API Gateway：令牌桶全局限流、动态隐藏秒杀 URL（开抢前动态下发 path salt）
       │  (剩余 ~10万 QPS)
       ▼ 3. 应用服务层 + 本地缓存（拦截无效/已售罄流量）
  ├── JVM 本地缓存（Caffeine）：缓存商品静态详情 + “已售罄（soldOut）”内存标记
  └── 一旦 Redis 扣减返回售罄，通过 Redis Pub/Sub 广播所有节点置 soldOut=true，直接快速失败
       │  (剩余 ~数万 QPS)
       ▼ 4. Redis 集群（原子预扣减核心，放行极少数有效名额）
  └── 执行 Lua 脚本：校验一人一单(Set/Bitmap) + 检查库存 + DECRBY 预扣库存
       │  (仅放行 = 真实库存量 的请求，如 1000 个)
       ▼ 5. MQ 异步削峰（RocketMQ / Kafka）
  └── 预扣成功即返回“排队中/抢购成功”，发送可靠消息至 MQ，消费者按 DB 承受力匀速消费
       │
       ▼ 6. MySQL 落库（最终兜底，~1000 TPS 匀速写入）
  └── 事务内：创建订单（user_id + activity_id 唯一索引） + 乐观锁扣减库存（stock >= buy_num）
```

#### 2. 核心表结构设计

```sql
-- 秒杀库存表（独立秒杀库，与主站库物理隔离）
CREATE TABLE t_seckill_stock (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    activity_id BIGINT NOT NULL COMMENT '活动ID',
    sku_id BIGINT NOT NULL COMMENT '商品SKU ID',
    total_stock INT NOT NULL COMMENT '总库存',
    available_stock INT NOT NULL COMMENT '可用库存',
    version INT NOT NULL DEFAULT 0,
    UNIQUE KEY uk_act_sku (activity_id, sku_id)
) ENGINE=InnoDB;

-- 秒杀订单表（防一人多单最终防线）
CREATE TABLE t_seckill_order (
    id BIGINT PRIMARY KEY COMMENT '雪花算法订单ID',
    user_id BIGINT NOT NULL,
    activity_id BIGINT NOT NULL,
    sku_id BIGINT NOT NULL,
    status TINYINT NOT NULL DEFAULT 0 COMMENT '0-待支付 1-已支付 2-超时取消',
    create_time DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    UNIQUE KEY uk_user_act (user_id, activity_id)
) ENGINE=InnoDB;
```

#### 3. 关键技术落地细节与代码

1. **热点库存分片（防止单 Redis 分片被打爆）**：
   - 假设某爆款 SKU 库存 10000 件，可拆分为 10 个子 Key：`seckill:stock:{1001}:0` ~ `seckill:stock:{1001}:9`（注意 `{1001}` 为 Redis Cluster HashTag，如果想要分散到不同物理 Redis 节点提升吞吐，则**故意不用 HashTag**，让 `seckill:stock:1001:0` ~ `:9` 均匀散落到集群不同 Master 分片上！）。
   - 请求按 `userId % 10` 路由到对应分片；若当前分片库存扣完，代码层轮询尝试其他剩余分片（需同步维护一个本地“已售罄分片位图”，避免无意义重试）。
2. **Redis Lua 脚本原子校验（防超卖 + 防一人多单）**：

```lua
-- KEYS[1]: 库存 Key, 如 seckill:stock:{1001}
-- KEYS[2]: 已购用户集合 Key, 如 seckill:users:{1001}
-- ARGV[1]: userId
-- ARGV[2]: 购买数量 quantity (通常为 1)
-- 返回值: 1=抢购成功, 0=库存不足, -1=重复下单, -2=活动未预热

local stockKey = KEYS[1]
local userKey  = KEYS[2]
local userId   = ARGV[1]
local qty      = tonumber(ARGV[2])

if redis.call('SISMEMBER', userKey, userId) == 1 then
    return -1
end

local stockStr = redis.call('GET', stockKey)
if not stockStr then
    return -2
end
local stock = tonumber(stockStr)
if stock < qty then
    return 0
end

redis.call('DECRBY', stockKey, qty)
redis.call('SADD', userKey, userId)
return 1
```

3. **本地“售罄标记”快速失败 + Redis Pub/Sub 广播同步**：
   - 当 Lua 返回 `0`（库存不足）时，服务节点将本地 `ConcurrentHashMap<Long, Boolean> SOLD_OUT_MAP` 设为 `true`，并通过 Redis `PUBLISH seckill:soldout skuId` 广播给集群内所有其他节点，后续请求连 Redis 都不用查，直接 1ms 内快速失败！
   - 若发生超时未支付回补库存，再发送 `PUBLISH seckill:restock skuId` 清除 `SOLD_OUT_MAP` 标记。

4. **动态秒杀接口 URL & 异常回滚兜底**：
   - **防脚本提前抢**：前端在活动开始前几秒才向服务端请求一个带加密盐值的 `path`（存入 Redis，有效期 60s），真实下单接口为 `/seckill/{path}/order`，未到开抢时间或 `path` 校验失败直接拒绝。
   - **Redis 扣了但 MQ/MySQL 失败怎么回滚？**：
     - 发 MQ 失败：立即执行补偿 Lua 脚本（`INCRBY` 库存 + `SREM` 用户集合）。
     - 消费端落库失败：RocketMQ 重试 3 次进入死信队列（DLQ），人工/自动补偿脚本回补 Redis；**注意**：若是因为 MySQL 唯一索引 `uk_user_act` 报错（说明用户已下单），则直接确认消费成功（幂等丢弃），绝不能回补库存！

#### 4. 面试官连环追问与避坑
- **追问 1：秒杀服务和普通交易服务为什么要做隔离？**
  - 秒杀是极高并发脉冲流量，若与主站下单共用 Tomcat 线程池、Redis 实例和 MySQL 数据库，一旦秒杀被打垮会拖垮全站正常交易。必须做到**域名隔离、微服务集群隔离、Redis/MQ 集群隔离、独立秒杀库**。
- **追问 2：如果 Redis 集群使用了 `SADD` 记录用户，1000 万用户参与抢购，Set 会不会太大？**
  - 若单活动参与人数达千万级，大 Key `Set` 会占用数百 MB 内存且阻塞主线程。若 `userId` 为自增整型，可改用 **Redis `Bitmap`（`SETBIT` / `GETBIT`）**，1000 万用户仅需约 `1.2MB` 内存；若 `userId` 为雪花长整型，可按 `userId % 64` 分桶拆分成 64 个小 Set。
- **追问 3：秒杀系统前端显示“排队中”，结果是怎么通知给用户的？**
  - 方案 A（推荐，简单稳定）：前端拿到“排队中”凭证后，使用 **短轮询（每隔 1~2 秒查一次订单结果接口，最多轮询 5 次）**，后端查询 Redis 订单结果标记，查到即跳转支付页。
  - 方案 B：WebSocket / SSE 主动推送（适合长连接基础设施完善的大型 App）。

---

## 2. 高并发短链生成系统（Short URL）

### Q2：如何设计一个支持高并发读写、高可用的短链生成与跳转系统（如 `t.cn/A1b2C3`）？

**业务特征**：典型的**读多写少**（生成短链 QPS 几百~几千，跳转点击 QPS 可达数万~十万+），要求跳转延迟极低（P99 < 20ms）、短链极短且全局唯一、支持过期与多维点击统计。

#### 1. 容量估算
- 假设每天新增 1000 万条短链，平均写 QPS $\approx 115$，峰值 1000 QPS；每条记录约 500 字节（长链+短码+元信息），一年存储 $\approx 3.65 \times 10^9 \times 500\text{B} \approx 1.8\text{ TB}$。
- 假设读写比 100:1，则跳转读峰值可达 **10 万 QPS**，需依赖多层缓存抗住读流量。

#### 2. 端到端架构设计

```text
写链路（生成短链）：
[业务方/用户] ──▶ [API Gateway] ──▶ [ShortUrl Service]
                                        ├── 1. 校验长链格式与安全黑名单（防钓鱼/色情）
                                        ├── 2. 分布式发号器 (Leaf 号段模式) 获取全局自增 ID
                                        ├── 3. 混淆后转 62 进制 (Base62) 生成 short_code
                                        ├── 4. 写入 MySQL (按 short_code Hash 分库分表) + 布隆过滤器 + 预热 Redis
                                        └── 5. 返回短链 https://t.cn/A1b2C3

读链路（短链跳转）：
[点击短链] ──▶ [Nginx / OpenResty] ──▶ [ShortUrl Service]
                                          ├── 1. 布隆过滤器判断 short_code 是否存在（防穿透）
                                          ├── 2. 查 L1 Caffeine ──▶ 查 L2 Redis ──▶ 查 MySQL（Singleflight防击穿）
                                          ├── 3. 异步发 Kafka 记录访问日志（IP/UA/Referer/时间）
                                          └── 4. 返回 HTTP 302 Location: long_url
```

#### 3. 核心表结构与分库分表策略

```sql
CREATE TABLE t_short_url_00 (
    id BIGINT PRIMARY KEY COMMENT 'Leaf生成的全局唯一ID',
    short_code VARCHAR(10) NOT NULL COMMENT '短码',
    long_url VARCHAR(2048) NOT NULL COMMENT '原始长链',
    url_md5 CHAR(32) NOT NULL COMMENT '长链MD5，用于去重/索引',
    expire_time DATETIME NULL COMMENT '过期时间，NULL表示永久',
    status TINYINT NOT NULL DEFAULT 1 COMMENT '1-有效 0-禁用',
    create_time DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    UNIQUE KEY uk_short_code (short_code),
    KEY idx_url_md5 (url_md5)
) ENGINE=InnoDB;
```
- **分片键选择**：高并发跳转都是通过 `short_code` 查询 `long_url`，因此**必须以 `short_code` 作为分库分表 Sharding Key**（避免广播查询全部分片）！

#### 4. 四大核心技术选型与代码实现

1. **短链生成算法对比（发号器 Base62 vs MurmurHash3）**：
   - **方案一：分布式发号器 + 62 进制转换（生产首选，无碰撞）**
     - 字符集：`0-9`（10）+ `a-z`（26）+ `A-Z`（26）= 62 个字符。6 位可表示 $62^6 \approx 568$ 亿条短链，7 位可达 $3.5$ 万亿条。
     - 发号器采用 **美团 Leaf 号段模式**（DB 批量拉取号段到内存双 Buffer，单机 10w+ TPS）。
     - **安全性优化（防遍历爬取）**：纯自增 ID 转 62 进制容易被竞品按顺序遍历推算每日新增单量。工程上可采用**乱序步长**或**Feistel 密码置换 / 与固定密钥异或混淆**后再转 Base62（如在 32 位/48 位空间内做可逆双射置换，既保证绝对唯一，又让短码看起来完全随机）。
   - **方案二：MurmurHash3 哈希 + 冲突重试**
     - 对 `longUrl` 计算 32 位无符号 MurmurHash3（非加密哈希，速度极快），转为 62 进制。
     - **解决冲突**：直接 `INSERT` 依靠 `uk_short_code` 唯一索引判重（或先过布隆过滤器）；若捕获 `DuplicateKeyException`，在 `longUrl` 尾部拼接固定盐值 `[DUPLICATE]` 重新哈希重试，直到成功。

```java
public class Base62Encoder {
    // 可打乱字符顺序进一步增加防猜测性
    private static final String BASE62 = "0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ";
    private static final int RADIX = 62;

    public static String encode(long num) {
        if (num == 0) {
            return "0";
        }
        StringBuilder sb = new StringBuilder();
        while (num > 0) {
            sb.append(BASE62.charAt((int) (num % RADIX)));
            num /= RADIX;
        }
        return sb.reverse().toString();
    }
}
```

2. **301 永久重定向 vs 302 临时重定向？**
   - **必须选 `302`（或 `307`）临时重定向**！
   - 原因：`301` 会被客户端浏览器强缓存，用户第二次点击短链时浏览器直接跳转目标长链，请求不再经过短链服务器。这会导致**无法统计真实的点击次数（PV/UV）、用户来源地域、设备终端（UA）、也无法在运营后台随时修改或封禁恶意目标链接**。

3. **防穿透与爆款热点保护**：
   - **防缓存穿透**：接入 **Redisson / Guava 布隆过滤器**拦截不存在的 `short_code`（若短链支持删除/过期，可对已过期/不存在的短码在 Redis 缓存空值 `"NONE"` 设置 60s 短 TTL 兜底）。
   - **防热点击穿**：爆款营销短信里的短链瞬间几十万 QPS，采用 **L1 Caffeine（5 分钟）+ L2 Redis + Singleflight（本地并发查库合并）** 保护 MySQL。

4. **海量点击统计架构**：
   - 跳转接口**绝不能同步写 MySQL**！异步发送日志到 **Kafka** $\rightarrow$ **Flink** 实时解析 IP 归属地与 UA 设备类型 $\rightarrow$ 写入 **ClickHouse**（做多维聚合报表）+ **Redis HyperLogLog**（极低内存计算去重 UV）。

#### 5. 面试官连环追问
- **追问 1：同一长链多次请求生成短链，应该返回同一个还是不同的？**
  - 视业务而定：如果营销渠道需要区分（短信渠道 vs 微信渠道独立统计点击转化），每次生成不同短链（发号器天然支持）；如果为了节省存储要求严格复用，则对长链计算 `url_md5` 查库复用（注意并发生成需加分布式锁或允许少量冗余）。
- **追问 2：如果布隆过滤器内存装不下千亿级短码，或者短链过期了怎么从布隆过滤器删除？**
  - 定期归档冷数据（超过 2 年无人点击的短链归档至冷库并释放短码）；布隆过滤器本身不支持删除，所以布隆过滤器只作为前置初筛，对于过期失效短链，在 Redis 缓存空值标记即可；或使用支持删除的 **Cuckoo Filter（布谷鸟过滤器）**。

---

## 3. 高并发微信抢红包系统

### Q3：如何设计高并发微信抢红包系统（保证金额公平、不超发、低延迟）？

#### 1. 核心难点
- **写竞争极高**：一个 500 人大群发一个红包，瞬间几百人同时点击抢，同一红包存在极高并发写冲突。
- **资金强一致**：总金额绝对不能多发（超抢）也不能少发，必须精确到“分”，严禁出现浮点精度丢失。
- **低延迟体验**：拆红包瞬间必须立刻看到抢到的金额（< 50ms）。

#### 2. 核心算法：二倍均值法 vs 线段切割法

- **二倍均值法（生产最常用，分布相对均匀）**：
  - 全部金额以**分（`long` 整型）**为单位计算，彻底避免 `double` 精度误差。
  - 每次抢到的金额区间为：$[1, \min(\text{剩余金额} - (\text{剩余人数} - 1),\ \lfloor \frac{\text{剩余金额}}{\text{剩余人数}} \times 2 \rfloor)]$，保证后面每人至少有 1 分钱，最后一人拿走全部余款。
- **线段切割法**：
  - 把总金额 $M$（分）看作长度 $M$ 的线段，在 $[1, M-1]$ 之间随机生成 $N-1$ 个不重复的切割点，排序后相邻两点之差即为每个红包金额。特点：可能出现极大包与极小包，方差更大。

```java
// 生产级二倍均值法预拆分红包（单位：分）
public static List<Long> splitRedPacket(long totalAmountCents, int count) {
    if (totalAmountCents < count || count <= 0) {
        throw new IllegalArgumentException("总金额不能少于每人1分钱");
    }
    List<Long> result = new ArrayList<>(count);
    long restAmount = totalAmountCents;
    int restCount = count;
    ThreadLocalRandom random = ThreadLocalRandom.current();

    for (int i = 0; i < count - 1; i++) {
        long maxAllowed = restAmount - (restCount - 1);
        long doubleMean = (restAmount / restCount) * 2;
        long max = Math.min(maxAllowed, Math.max(1L, doubleMean));
        long amount = random.nextLong(1, max + 1);
        result.add(amount);
        restAmount -= amount;
        restCount--;
    }
    result.add(restAmount);
    return result;
}
```

#### 3. 预拆分 + Redis List + Lua 原子抢包架构

为什么不在抢红包时现场计算？因为现场计算需要“读剩余金额与个数 $\rightarrow$ 计算 $\rightarrow$ 写回”，高并发下必须加分布式锁，串行化导致吞吐暴跌！

```text
发红包阶段：
1. 扣除发包人账户余额（分布式事务/本地事务表冻结资金），MySQL 写入红包主表 t_red_packet。
2. 调用二倍均值法在内存中预拆分成 N 份金额列表 [230, 150, 420, ...]。
3. 一次性 RPUSH 到 Redis List: `rp:pool:{packetId}`（设置 25h 过期，略大于 24h 退款周期）。

抢红包阶段（无锁化超高并发）：
1. 客户端点击“拆红包”，请求路由到抢红包服务。
2. 执行 Redis Lua 脚本（保证单线程原子性）：
   - 判断 Hash `rp:grabbed:{packetId}` 中是否已有 `userId`（防同一个人重复抢）
   - 若无，对 `rp:pool:{packetId}` 执行 `LPOP` 弹出一个金额
   - 若弹出为空，返回 -1（手慢了，红包已抢完）
   - 若弹出成功，写入 `HSET rp:grabbed:{packetId} userId amount` 返回该金额
3. 抢到后立即返回前端金额展示，同时异步发送 MQ 消息入账。

异步入账与 24 小时过期退款：
1. MQ 消费者消费抢包明细，写入 MySQL 明细表（uk_packet_user 唯一索引幂等），并增加收包人余额。
2. 发送 24 小时延迟消息：检查红包主表剩余金额，清理 Redis 池并原路退回未领金额。
```

```lua
-- 抢红包原子 Lua 脚本
-- KEYS[1]: 红包池 List, 如 rp:pool:{10086}
-- KEYS[2]: 已抢用户记录 Hash, 如 rp:grabbed:{10086}
-- ARGV[1]: userId
-- 返回值: >0 表示抢到的金额(分); -1=已抢过; 0=已被抢光

if redis.call('HEXISTS', KEYS[2], ARGV[1]) == 1 then
    return -1
end

local amount = redis.call('LPOP', KEYS[1])
if not amount then
    return 0
end

redis.call('HSET', KEYS[2], ARGV[1], amount)
return tonumber(amount)
```

#### 4. 面试官连环追问
- **追问 1：500 人的大群里，490 个人都没抢到，大家同时点开看手气排行，怎么避免查爆 DB？**
  - 抢完后“手气明细列表”直接从 Redis `HGETALL rp:grabbed:{packetId}` 读取并缓存到内存/CDN，只有超过 24h 后的历史详情才回源查 MySQL 从库。
- **追问 2：如果发完红包写入了 DB，但在 `RPUSH` 到 Redis 前服务崩溃了怎么办？**
  - 发红包采用“先落 DB 红包主表（状态=`INIT`）+ 本地消息表 / MQ 异步初始化 Redis 池，成功后更新状态为 `ACTIVE`”；若 Redis 池初始化失败，定时补偿任务重试或直接原路退款。

---

## 4. 电商订单与支付状态机（防重、超时关单、对账补偿）

### Q4：电商订单与支付系统如何设计状态机、提交防重、超时关单及对账补偿？

#### 1. 订单状态机设计（拒绝脆弱的 `if-else`）

订单流转必须受严格的状态机约束，防止并发请求（如用户刚发起取消，支付回调同时到达）导致状态错乱：

```text
  [待支付 UNPAID] ──(支付成功)──▶ [已支付/待发货 PAID] ──(商家发货)──▶ [已发货 SHIPPED] ──▶ [已完成 COMPLETED]
         │                                │
         ├──(用户取消/超时关单)              ├──(申请退款)
         ▼                                ▼
  [已取消 CANCELLED]               [退款中 REFUNDING] ──▶ [已退款 REFUNDED]
```

- **工程实现方式（状态转移表 + 数据库乐观锁原子推进）**：

```java
public enum OrderStatus { UNPAID, PAID, SHIPPED, COMPLETED, CANCELLED, REFUNDING, REFUNDED }
public enum OrderEvent  { PAY_SUCCESS, USER_CANCEL, TIMEOUT_CLOSE, DELIVER, CONFIRM_RECEIPT, APPLY_REFUND, REFUND_SUCCESS }

public class OrderStateMachine {
    private static final Map<OrderStatus, Map<OrderEvent, OrderStatus>> TRANSITIONS = new EnumMap<>(OrderStatus.class);

    static {
        TRANSITIONS.put(OrderStatus.UNPAID, Map.of(
            OrderEvent.PAY_SUCCESS, OrderStatus.PAID,
            OrderEvent.USER_CANCEL, OrderStatus.CANCELLED,
            OrderEvent.TIMEOUT_CLOSE, OrderStatus.CANCELLED
        ));
        TRANSITIONS.put(OrderStatus.PAID, Map.of(
            OrderEvent.DELIVER, OrderStatus.SHIPPED,
            OrderEvent.APPLY_REFUND, OrderStatus.REFUNDING
        ));
        // 其余合法状态流转依次注册...
    }

    public static OrderStatus nextStatus(OrderStatus current, OrderEvent event) {
        Map<OrderEvent, OrderStatus> eventMap = TRANSITIONS.get(current);
        if (eventMap == null || !eventMap.containsKey(event)) {
            throw new IllegalStateException("非法状态流转: " + current + " -> " + event);
        }
        return eventMap.get(event);
    }
}
```
- **数据库乐观锁条件更新（最后一道硬防线）**：
  ```sql
  UPDATE t_order
  SET status = 'PAID', pay_time = NOW(), version = version + 1
  WHERE order_no = 'ORD20260001' AND status = 'UNPAID' AND version = 1;
  ```
  必须校验 `affectedRows == 1` 才算推进成功，否则抛出状态冲突异常或触发补偿退款。

#### 2. 下单接口防重与幂等（Idempotency Token）

用户网络卡顿连点两次“提交订单”，绝不能生成两笔订单、扣两次优惠券！
- **Token 令牌机制**：
  1. 进入确认订单页（结算页）时，服务端生成一个全局唯一 `orderToken` 存入 Redis（TTL 15 分钟），并返回给前端。
  2. 提交订单时前端回传 `orderToken`，服务端通过 **Redis Lua 脚本原子执行 `GET` 校验 + `DEL` 删除**：
     ```lua
     if redis.call('GET', KEYS[1]) == ARGV[1] then
         return redis.call('DEL', KEYS[1])
     else
         return 0
     end
     ```
  3. 同步在 DB 订单表对业务幂等键（如 `idempotent_key`）建**唯一索引**，彻底防止分布式并发重入。

#### 3. 订单超时未支付自动关闭（延迟方案对比 & 边界竞态处理）

| 方案 | 原理 | 优点 | 缺点与适用场景 |
| :--- | :--- | :--- | :--- |
| **RocketMQ 延迟消息** | 下单时发送 Delay Level=16（30分钟）或 RocketMQ 5.x 任意时间定时消息 | 高吞吐、持久化、支持分布式水平扩展、天然削峰 | 需维护 MQ 集群（**大厂电商主流首选**） |
| **RabbitMQ 死信队列 / 延迟插件** | TTL + DLX 或 `x-delayed-message` | 接入简单 | 单队列大量不同 TTL 会产生队头阻塞；插件高并发下性能一般 |
| **Redisson `RDelayedQueue` / 时间轮** | Redis ZSet 轮询或 Netty `HashedWheelTimer` | 毫秒级精度，不依赖重型 MQ | 单机时间轮重启丢任务；Redis ZSet 需注意大 Key 拆分 |
| **定时任务扫表（如 XXL-JOB）** | 每分钟扫描 `status='UNPAID' AND create_time < NOW()-30m` | 实现最简单，绝对不丢任务 | 对 DB 压力大，精度分钟级；**通常作为 MQ 方案的兜底补偿** |

- **🔥 高频并发竞态坑（支付回调与超时关单同时撞车）**：
  - **场景**：用户在第 29 分 59 秒完成支付，第 30 分 00 秒关单消费者抢先将订单改为 `CANCELLED` 并释放了库存，100ms 后支付宝回调成功！
  - **解决**：
    1. 关单前先加**订单级分布式锁**，并向第三方支付渠道发起一次**主动查单**（确认确实未支付才关单）。
    2. 支付回调若发现订单已处于 `CANCELLED`（说明库存已被释放），**切勿强行把状态改回 `PAID`（会导致超卖！）**，应记录异常单并自动触发**原路退款**流程，同时短信提示用户重新下单。

#### 4. 支付回调幂等与 T+1 资金对账补偿
- **回调三重防重**：`分布式锁(lock:pay:orderNo)` $\rightarrow$ `支付流水表 pay_trade_no 唯一索引` $\rightarrow$ `订单表 WHERE status = 'UNPAID'` 条件更新。
- **异步通知丢失兜底（阶梯主动查单）**：发起支付后发送 15s / 1m / 5m 阶梯延迟消息主动向微信/支付宝拉取支付结果，确保不因公网回调丢包导致卡单。
- **T+1 对账系统**：每日凌晨拉取微信/支付宝昨日账单，与本地支付流水全量比对，自动识别并处理“长款（渠道有、本地无 $\rightarrow$ 补单或退款）”、“短款（本地有、渠道无 $\rightarrow$ 异常核查）”、“金额不一致”等资金差错。

#### 5. 核心表结构与面试官连环追问

```sql
CREATE TABLE t_order (
    id BIGINT PRIMARY KEY,
    order_no VARCHAR(32) NOT NULL COMMENT '订单号',
    user_id BIGINT NOT NULL,
    total_amount BIGINT NOT NULL COMMENT '总金额(分)',
    status VARCHAR(20) NOT NULL COMMENT 'UNPAID/PAID/SHIPPED/COMPLETED/CANCELLED',
    idempotent_key VARCHAR(64) NOT NULL COMMENT '下单幂等键',
    version INT NOT NULL DEFAULT 0 COMMENT '乐观锁版本号',
    expire_time DATETIME NOT NULL COMMENT '支付超时时间',
    UNIQUE KEY uk_order_no (order_no),
    UNIQUE KEY uk_idempotent (user_id, idempotent_key),
    KEY idx_user_create (user_id, create_time)
) ENGINE=InnoDB;

CREATE TABLE t_payment_record (
    id BIGINT PRIMARY KEY,
    order_no VARCHAR(32) NOT NULL,
    pay_trade_no VARCHAR(64) NOT NULL COMMENT '第三方渠道支付流水号',
    channel VARCHAR(16) NOT NULL COMMENT 'ALIPAY/WECHAT',
    pay_amount BIGINT NOT NULL COMMENT '实付金额(分)',
    pay_status VARCHAR(16) NOT NULL COMMENT 'SUCCESS/REFUNDED',
    UNIQUE KEY uk_trade_no (channel, pay_trade_no),
    KEY idx_order_no (order_no)
) ENGINE=InnoDB;
```

- **追问 1：支付回调时为什么一定要校验 `pay_amount` 是否等于订单应付金额？**
  - 防范“金额篡改 / 1 分钱薅羊毛”漏洞：必须核对回调报文中的实付金额、商户号、AppId 与本地订单金额完全一致（以分为单位比较），且校验渠道 RSA2 签名，防止伪造回调请求。
- **追问 2：海量历史订单表（几亿行）怎么分库分表？**
  - 按 `user_id` 哈希分片（解决 C 端用户查询“我的订单”不跨库）；同时通过 **Canal + Elasticsearch** 同步订单宽表，解决商家后台按时间、商品名、状态等多条件检索和运营统计需求；超过 6 个月的冷订单归档至历史库或 TiDB/ClickHouse。
- **追问 3：订单取消或退款时，用了优惠券/积分怎么退？**
  - 下单锁定优惠券（`USED`），关单/全额退款时通过事务消息异步通知营销服务回滚优惠券（注意校验券是否已过有效期、部分退款时按比例分摊退回实付金额）。

---

## 5. 信息流 / 朋友圈 Feed 流系统

### Q5：如何设计微博 / 朋友圈等高并发 Feed 信息流系统？

**核心挑战**：用户关注关系网高度不均衡（普通用户几百粉，大 V 数千万粉），如何在发帖写入与刷首页读取之间实现低延迟与低成本平衡？

#### 1. 三种扩散模型深度对比

| 模式 | 核心机制 | 优点 | 缺点 | 适用场景 |
| :--- | :--- | :--- | :--- | :--- |
| **推模式（写扩散 Fan-out on Write）** | 发帖时遍历作者所有粉丝，将 `postId` 写入每个粉丝的收件箱（Inbox，Redis ZSet） | **读极其简单快速**：用户刷首页只需查自己 Inbox（$O(1)$），天然有序分页 | **大 V 写放大灾难**：1 千万粉丝大 V 发帖要写 1 千万次，MQ/Redis 被打爆且僵尸粉浪费内存 | 微信朋友圈（好友上限 5000，纯写扩散完全可控） |
| **拉模式（读扩散 Fan-out on Read）** | 发帖只写作者自己的发件箱（Outbox）；刷首页时拉取所有关注对象的 Outbox 在内存多路归并排序 | **无写放大**，发帖极快，存储成本极低 | **读延迟爆炸**：关注 1000 人就要拉取 1000 个列表并归并，性能极差 | 极低频阅读场景或早期原型 |
| **推拉结合（混合模式 Hybrid）** | **普通用户发帖走推模式**；**大 V（如粉丝 > 10万）发帖只写自己的 Outbox**。刷首页时查自己 Inbox + 拉取所关注的少数大 V 的 Outbox 归并 | 兼顾读写性能，既无大 V 写扩散风暴，又避免拉取海量普通人的读开销 | 架构复杂度高（需维护大 V 识别、冷热粉丝分层） | **微博 / X / 小红书关注流（工业界标准）** |

#### 2. 推拉结合存储设计与读取流程

- **Inbox / Outbox 只存 `postId`（轻量索引），不存帖子正文**：
  - Redis ZSet `feed:inbox:{userId}`：`member = postId`，`score = publishTimestamp`。每个用户只保留最近 1000 条（超限通过 `ZREMRANGEBYRANK` 裁剪），历史数据回落 MySQL/HBase。
  - 拿到 `postId` 列表后，再批量 `MGET` 从文章缓存组装详情（Hydration，避免一条热点动态被冗余拷贝几百万份）。
- **活跃粉丝分层优化**：即使大 V 或腰部博主有 50 万粉丝，但近 7 天登录的活跃粉丝只有 8 万。写扩散时只推给**活跃粉丝**的 Inbox；沉睡用户重新上线时，异步触发一次拉取重建其 Inbox。

```text
用户刷关注流 GET /feed/timeline?cursor=1763450000000&size=20
  │
  ├── 1. ZREVRANGEBYSCORE feed:inbox:{uid} (cursor -inf LIMIT 0 20) ──▶ 获取普通关注者动态 ID
  ├── 2. 查关注列表中的大 V 集合（通常每个人关注的大 V 仅 5~20 个），
  │      并行 ZREVRANGEBYSCORE feed:outbox:{bigVId} (cursor -inf LIMIT 0 20)
  ├── 3. 内存最小堆 / 归并排序合并两路结果，取 Top 20 postIds，更新 nextCursor
  ├── 4. 过滤已拉黑/已删除帖子，批量 MGET post:{id} + 批量查点赞/评论计数
  └── 5. 返回渲染列表与 nextCursor
```

```java
// 推拉结合读取核心伪代码
public FeedPageResponse pullTimeline(long userId, long cursorTs, int size) {
    // 1. 拉取自己 Inbox 中的普通关注者帖子 ID（带 score 时间戳）
    Set<ZSetOperations.TypedTuple<String>> inboxItems =
        redisTemplate.opsForZSet().reverseRangeByScoreWithScores(
            "feed:inbox:" + userId, Double.NEGATIVE_INFINITY, cursorTs - 1, 0, size);

    // 2. 获取当前用户关注的大 V 列表（缓存 Set），并发拉取大 V 的 Outbox
    List<Long> bigVIds = followService.getFollowedBigVs(userId);
    List<CompletableFuture<Set<ZSetOperations.TypedTuple<String>>>> futures = bigVIds.stream()
        .map(bigVId -> CompletableFuture.supplyAsync(() ->
            redisTemplate.opsForZSet().reverseRangeByScoreWithScores(
                "feed:outbox:" + bigVId, Double.NEGATIVE_INFINITY, cursorTs - 1, 0, size),
            feedExecutor))
        .toList();

    // 3. 合并去重并按 score 降序取前 size 条
    List<PostItem> merged = mergeAndSortTopN(inboxItems, futures, size);
    // 4. 批量 Hydration 填充正文、作者头像与计数（过滤已删/拉黑）
    List<PostDTO> posts = hydratePosts(merged);
    long nextCursor = merged.isEmpty() ? 0L : merged.get(merged.size() - 1).timestamp();
    return new FeedPageResponse(posts, nextCursor);
}
```

#### 3. 深度分页与未读红点设计
- **禁止 `LIMIT offset, size`**：Feed 流不断有新内容插入，传统页码翻页会出现重复或漏刷。必须使用 **游标分页（`cursor = lastTimestamp` 或 `lastPostId`）**，稳定 $O(\log N + M)$。
- **未读计数与红点**：若只需要“有新动态”红点提示，只需记录用户 `last_read_time`，对比所有关注大 V 最新发帖时间 `max(latest_post_time)` 是否大于 `last_read_time` 及 Inbox 顶部 score 即可 $O(1)$ 判断。

#### 4. 面试官连环追问
- **追问 1：博主删帖或者拉黑了某个粉丝，怎么保证 Inbox 里不再显示？**
  - 不要去全量遍历几百万粉丝的 Inbox 做 `ZREM`！只需把原帖在 DB/缓存标记 `deleted=1`（或将拉黑关系写入布隆过滤器/黑名单 Set），在读链路第 4 步 Hydration 时过滤丢弃即可（惰性清理）。
- **追问 2：如果某个大 V 刚涨粉突破 10 万阈值，从“推”变成“拉”，新老数据怎么过渡？**
  - 晋升为大 V 前已推入粉丝 Inbox 的老帖子不动，晋升后的新帖子只写 Outbox；粉丝读取时将 Inbox 与 Outbox 按 `postId` 去重归并，过渡期平滑无感知。

---

## 6. 高并发即时通讯（IM）系统

### Q6：如何设计一个高并发、消息不丢不重不乱序的即时通讯（IM）系统？

#### 1. 核心架构分层

```text
[App / Web 客户端] ◀── WebSocket / TCP 自定义协议(Protobuf) ──▶ [Netty 长连接网关集群]
                                                                       │ (Dubbo / gRPC)
[Redis 连接路由表: userId -> gateway_ip:port] ◀────────────────────────┤
                                                                       ▼
[APNs / 厂商离线推送] ◀── [Push 服务] ◀── [Kafka 消息总线] ◀── [IM Logic 核心逻辑服务]
                                                                       │
                                                                       ▼
                                                        [MySQL / HBase / TiDB 消息存储]
```

#### 2. Netty 长连接网关核心设计
1. **自定义二进制协议帧设计（解决粘包/拆包）**：
   - `Magic(2B)` + `Version(1B)` + `SerializeType(1B: Protobuf)` + `Command(1B: 心跳/单聊/群聊/ACK)` + `BodyLength(4B)` + `Body(N bytes)`。
   - 配合 Netty `LengthFieldBasedFrameDecoder` 精准拆包，基于 Linux `epoll` 单机可稳定维持 20万~50万 长连接。
2. **心跳保活与僵尸连接清理**：
   - 使用 `IdleStateHandler(60, 0, 0)` 检测读空闲，超过 60 秒未收到客户端 Ping 心跳帧则主动关闭 `Channel` 并清理 Redis 路由，防止大量半开死连接占满文件描述符（`ulimit -n` 需调大至 100 万）。

```java
// Netty IM 网关 Pipeline 核心配置
pipeline.addLast(new IdleStateHandler(60, 0, 0, TimeUnit.SECONDS));
pipeline.addLast(new LengthFieldBasedFrameDecoder(65536, 5, 4, 0, 0));
pipeline.addLast(new ImPacketCodec()); // 编解码 Protobuf 帧
pipeline.addLast(new SimpleChannelInboundHandler<ImPacket>() {
    @Override
    public void userEventTriggered(ChannelHandlerContext ctx, Object evt) {
        if (evt instanceof IdleStateEvent e && e.state() == IdleState.READER_IDLE) {
            // 超过 60s 无心跳/数据，清理用户路由并关闭连接
            routeService.unbindChannel(ctx.channel());
            ctx.close();
        }
    }
    @Override
    protected void channelRead0(ChannelHandlerContext ctx, ImPacket packet) {
        // 业务处理务必投递到独立业务线程池，避免阻塞 Netty NioEventLoop IO 线程！
        bizExecutor.execute(() -> imDispatcher.dispatch(ctx, packet));
    }
});
```

3. **精准消息路由**：
   - 用户 A 上线连接 `Gateway-1`，网关在 Redis 注册 `im:route:{userA} = Gateway-1:9000`（支持多端登录可存 Hash：`deviceType -> gatewayAddr`）。B 给 A 发消息时，Logic 服务查 Redis 路由并通过 RPC 投递给 `Gateway-1` 下发；若离线，转入离线存储并触发 APNs / 华为 / 小米厂商 Push。

#### 3. 如何保证消息“不丢、不重、有序”？（核心必考）

```text
[发送方 A]                 [IM Server]                [接收方 B]
    │                           │                          │
    │──1. SendMsg(clientMsgId)─▶│                          │
    │                           │──2. 分配seqId & 落库     │
    │◀─3. Server ACK (seqId)────│                          │
    │                           │──4. PushMsg(seqId)──────▶│
    │                           │                          │──5. 本地SQLite去重落库
    │                           │◀─6. Deliver ACK(seqId)───│
```

- **不丢（双重 ACK + 推拉结合）**：
  - 发送方没收到 `Server ACK` 超时自动重传（提示转圈圈/红色感叹号）。
  - 接收方没回 `Deliver ACK` 或断线重连后，客户端携带本地最大 `local_max_seq` 向服务端拉取增量漫游消息（`seq > local_max_seq`），彻底解决弱网推丢问题。
- **不重（全链路幂等去重）**：
  - 发送方生成唯一 `clientMsgId`（雪花 ID / UUID）。服务端借助 Redis（短期滑动去重窗口）+ DB 唯一索引去重；接收方本地 SQLite 也以 `msg_id` 唯一索引去重，重传消息直接丢弃并补回 ACK。
- **有序（会话级单调递增 `seqId` + 空洞补齐）**：
  - **绝不能用服务器或客户端时间戳排序**（时钟偏差会导致乱序）！每个会话（`convId`）通过 Redis `INCR seq:{convId}` 或独立发号段分配严格递增的 `seqId`。
  - 如果接收端本地最新 `seq` 是 100，突然收到 102，说明 101 丢包/乱序迟到，客户端先暂存 102 并主动发起**空洞拉取请求**补齐 101 后再渲染。

#### 4. 群聊扩散与已读未读水线
- **群聊消息扩散（写消息只存一份，推信令扩散）**：
  - 500 人以下普通群：**群消息正文只写一份**到群消息表（分配群 `seqId`），然后异步遍历在线群成员下发轻量化推送通知。
  - 万人超大群/直播间：严禁全群写扩散通知，采用**读扩散 / 分批广播**。
- **已读/未读数（Read Receipt Waterline）**：
  - 切勿对每条消息记录“谁已读”，存储会爆炸！只需在 `t_conversation_member` 维护一个**已读水位线 `last_read_seq`**。用户打开聊天框滑到底部上报 `ACK_READ(convId, seq=105)`，更新 `last_read_seq = 105`，未读数 = `max_seq - last_read_seq`。

#### 5. 面试官连环追问
- **追问 1：IM 历史聊天记录非常庞大，怎么分库分表存储？**
  - 按 `conversation_id`（会话 ID）哈希分库分表（或写入 HBase / TiDB 等宽表/分布式库），查询某个会话的历史消息只打到单一分片；冷热分离，超过 6 个月的消息归档到对象存储/冷库，仅保留近半年热数据在 SSD。
- **追问 2：如何实现微信“发送 2 分钟内撤回消息”？**
  - 撤回本质是发送一条类型为 `RECALL(targetMsgId)` 的新控制信令消息（同样分配递增 `seqId`）：服务端校验原消息发送时间是否 $\le 2$ 分钟，若合法则将原消息标记 `status=RECALLED`，并向接收方推送这条撤回信令；接收方收到后将本地 SQLite 对应消息替换为灰色提示“对方撤回了一条消息”。
- **追问 3：多端同时在线（手机 + PC）时，消息怎么同步和保证已读一致？**
  - Redis 路由表使用 Hash 结构存储多端映射（`MOBILE -> Gateway-1`, `PC -> Gateway-2`），收到发给该用户的消息时遍历 Hash 向所有在线端推送；手机端上报 `ACK_READ` 更新水位线时，顺带给 PC 端推送一条“已读同步信令”，消除 PC 端小红点。

---

## 7. 分布式任务调度系统（XXL-JOB vs ElasticJob）

### Q7：分布式定时任务调度系统如何设计？XXL-JOB 和 ElasticJob 有什么区别？

#### 1. 为什么不能用单机 `@Scheduled` 或原生 Quartz 集群？
- **Spring `@Scheduled`**：多实例集群部署会**重复跑多次**；无统一管理界面、无法动态改 Cron、无失败告警与分片并行能力。
- **原生 Quartz 集群**：基于数据库表行锁（`QRTZ_LOCKS`）抢占调度权，任务数一多 DB 锁竞争严重，且调度与执行耦合在业务进程内，缺乏路由策略与可视化实时日志。

#### 2. XXL-JOB vs ElasticJob 核心架构对比

| 维度 | XXL-JOB（中心化调度架构） | ElasticJob-Lite（去中心化分片架构） |
| :--- | :--- | :--- |
| **架构模式** | **调度中心（Admin） + 执行器（Executor）** 分离。Admin 通过 DB 行锁防重调度，通过 Netty HTTP RPC 触发业务执行器 | **无中心调度节点**，基于 **ZooKeeper** 选举与协调，各节点内置 Quartz 各自触发 |
| **分片能力** | 支持“分片广播”路由策略（所有执行器各自收到 `shardIndex` 和 `shardTotal`） | **原生强分片**：将分片项与 ZK 节点动态绑定，节点上下线**自动弹性 Rebalance** |
| **动态运维** | 极其完善：Web 控制台支持动态启停、改 Cron、手动触发、失败重试、实时滚屏查看远程执行日志 | 偏重静态/ZK 配置，动态运维与日志界面较弱 |
| **适用场景** | **90% 企业级常规业务首选**（对账、报表、数据清理、超时补偿） | 超大规模纯数据并行清洗/计算型任务 |

#### 3. 核心原理与生产避坑
1. **XXL-JOB 调度中心集群如何防止重复调度？**
   - 调度线程通过 **`SELECT ... FOR UPDATE` 悲观行锁**锁定 MySQL `xxl_job_lock` 表中的 `schedule_lock` 记录，同一秒内只有一台 Admin 节点能扫表取出到期任务并存入时间轮/线程池异步触发执行器。
2. **分片广播处理千万级数据实战**：
   ```java
   @XxlJob("sendCouponShardingJob")
   public void sendCouponJob() {
       int shardIndex = XxlJobHelper.getShardIndex(); // 当前分片序号 0 ~ N-1
       int shardTotal = XxlJobHelper.getShardTotal(); // 总分片数(执行器实例数)
       // 按用户 ID 取模分片并行拉取处理（注意使用游标 id > lastId 替代 OFFSET 深分页！）
       long lastId = 0L;
       while (true) {
           List<User> users = userMapper.selectShardUsers(shardIndex, shardTotal, lastId, 500);
           if (users.isEmpty()) break;
           processBatch(users);
           lastId = users.get(users.size() - 1).getId();
       }
   }
   ```
3. **关键策略配置（面试高频）**：
   - **Misfire（调度过期）**：调度中心宕机或忙不过来错过触发时间（默认超 5 秒），可选 `DO_NOTHING`（忽略等下一次）或 `FIRE_ONCE_NOW`（立即补偿一次）。
   - **Failover（故障转移）**：Admin 下发前逐台发 `/beat` 心跳检测，跳过失联执行器，转移到下一台健康节点。
   - **阻塞处理策略**：当上一轮任务没跑完下一轮又触发时，可选 `单机串行`、`丢弃后续调度`（防止任务堆积雪崩）、`覆盖之前调度`。
   - **幂等性要求**：调度系统只能保证 **At-Least-Once（至少一次）**，业务任务代码内部必须自行保证**幂等性**！

---

## 8. 统一限流与全链路灰度发布系统

### Q8：如何设计企业级统一限流体系与微服务全链路灰度发布（Canary Release）？

#### 1. 四大限流算法原理与选型

| 算法 | 原理 | 优点 | 缺点 |
| :--- | :--- | :--- | :--- |
| **固定窗口（计数器）** | 统计固定时间片（如 1s）内请求数，超限拒绝 | 实现极简 | **临界突刺问题**：0.9s~1.0s 与 1.0s~1.1s 各放行 100 次，0.2s 内承受 200 次（2 倍）流量冲击 |
| **滑动窗口** | 将 1 秒切分为多个子窗口，或用 Redis ZSet 记录每次请求时间戳，窗口随时间滑动 | 彻底消除临界突刺（Sentinel 默认采用） | 需记录子窗口数组或请求时间戳，内存开销略高 |
| **漏桶（Leaky Bucket）** | 请求先入缓冲队列，以恒定速率匀速流出，桶满溢出拒绝 | **绝对平滑整形**下游流量，保护脆弱 DB | 无法应对合理的短时突发流量 |
| **令牌桶（Token Bucket）** | 以恒定速率往桶内生成令牌（上限为容量 $C$），请求拿到令牌放行，否则拒绝 | **既限制平均速率，又允许一定程度的突发流量** | 实现略复杂（Guava `RateLimiter` / Spring Cloud Gateway 默认） |

#### 2. 多层限流架构 & Redis + Lua 脚本实战

- **三层防线**：`Nginx 层（limit_req_zone 防单 IP 洪水）` $\rightarrow$ `Gateway 集群限流（Redis 令牌桶/滑动窗口）` $\rightarrow$ `应用内单机限流熔断（Sentinel 保护核心线程池与慢依赖）`。

```lua
-- Redis + Lua 精确滑动窗口限流脚本
-- KEYS[1]: rate:limit:api:/order/create:user123
-- ARGV[1]: 窗口大小(ms, 如 1000), ARGV[2]: 阈值(如 10), ARGV[3]: 当前时间戳(ms), ARGV[4]: 唯一请求ID

local key = KEYS[1]
local window = tonumber(ARGV[1])
local limit = tonumber(ARGV[2])
local now = tonumber(ARGV[3])
local member = ARGV[4]

redis.call('ZREMRANGEBYSCORE', key, 0, now - window)
local current = redis.call('ZCARD', key)
if current >= limit then
    return 0 -- 触发限流
end

redis.call('ZADD', key, now, member)
redis.call('PEXPIRE', key, window)
return 1 -- 放行
```
> **实战避坑**：ZSet 的 `member` 严禁直接传时间戳 `now`！高并发同一毫秒内多个请求会导致 `member` 冲突覆盖，使计数偏少，必须传 `now + ":" + uuid`。若 QPS 极高怕 ZSet 太大，可改用 **Redis 令牌桶 Lua 脚本**（只需存 `tokens` 和 `last_refill_time` 两个 Hash 字段，$O(1)$ 空间与时间！）。

```lua
-- Redis + Lua 生产级令牌桶限流脚本 (O(1) 内存与时间复杂度)
-- KEYS[1]: rate:bucket:api:/seckill
-- ARGV[1]: 桶容量 capacity (如 100), ARGV[2]: 填充速率 rate(令牌/秒, 如 50)
-- ARGV[3]: 当前时间戳(秒, 浮点或毫秒/1000), ARGV[4]: 本次申请令牌数(通常为 1)

local key = KEYS[1]
local capacity = tonumber(ARGV[1])
local rate = tonumber(ARGV[2])
local now = tonumber(ARGV[3])
local requested = tonumber(ARGV[4])

local data = redis.call('HMGET', key, 'tokens', 'last_ts')
local tokens = tonumber(data[1])
local last_ts = tonumber(data[2])

if not tokens then
    tokens = capacity
    last_ts = now
else
    local elapsed = math.max(0, now - last_ts)
    tokens = math.min(capacity, tokens + elapsed * rate)
    last_ts = now
end

if tokens < requested then
    redis.call('HMSET', key, 'tokens', tokens, 'last_ts', last_ts)
    redis.call('EXPIRE', key, math.ceil(capacity / rate) * 2)
    return 0 -- 限流拒绝
end

tokens = tokens - requested
redis.call('HMSET', key, 'tokens', tokens, 'last_ts', last_ts)
redis.call('EXPIRE', key, math.ceil(capacity / rate) * 2)
return 1 -- 放行
```

#### 3. 微服务全链路灰度发布（Canary）如何实现？

**目标**：新版本（`version=gray`）只对内部员工或 5% 特定 UserId 生效，且请求经过 `Gateway -> 订单 -> 库存 -> RocketMQ -> 积分` 全程走在灰度节点；若下游某服务本次未发布灰度版，则自动兜底路由到基线版（`base`）。

- **全链路实现核心 4 步**：
  1. **流量染色（Gateway 层）**：网关匹配 Header（`X-Env: gray`）、UID 白名单或百分比规则，给请求打上灰度标签（`tag=gray`）。
  2. **跨线程与跨进程上下文透传**：
     - 进程内异步线程池：使用阿里 **`TransmittableThreadLocal (TTL)`** 透传上下文，防止线程池复用导致灰度标签丢失或串号！
     - 跨 RPC 调用（OpenFeign / Dubbo）：通过 `RequestInterceptor` / Dubbo `Filter` 将灰度标签写入 HTTP Header 或 RPC `Attachment`。
     - 跨 MQ（RocketMQ/Kafka）：生产者拦截器将灰度标签写入消息 `UserProperty`，灰度消费者与基线消费者通过 Tag / SQL92 属性过滤各自消费对应流量。
  3. **自定义负载均衡路由器**：扩展 Spring Cloud LoadBalancer 的 `ServiceInstanceListSupplier`，读取 Nacos 实例元数据 `metadata.version`：
     - 优先筛选 `metadata.version == gray` 的实例；
     - 若下游服务无 `gray` 实例，**自动降级选择 `base` 基线实例**，保证调用链路不断。
  4. **蓝绿发布 vs 灰度（金丝雀）发布**：蓝绿需两套全量对等机器（成本翻倍、回滚秒级）；灰度仅需少量灰度节点逐步切流（1% $\rightarrow$ 10% $\rightarrow$ 50% $\rightarrow$ 100%），爆炸半径小、资源利用率高。

```java
// 灰度上下文透传 + Spring Cloud LoadBalancer 兜底路由核心骨架
public class GrayContextHolder {
    // 使用阿里 TransmittableThreadLocal 解决父子线程与线程池复用透传
    private static final TransmittableThreadLocal<String> GRAY_TAG = new TransmittableThreadLocal<>();
    public static void setTag(String tag) { GRAY_TAG.set(tag); }
    public static String getTag() { return GRAY_TAG.get(); }
    public static void clear() { GRAY_TAG.remove(); }
}

// 自定义实例过滤逻辑（集成进 ReactorServiceInstanceLoadBalancer）
public List<ServiceInstance> filterInstances(List<ServiceInstance> allInstances, String targetTag) {
    if ("gray".equals(targetTag)) {
        List<ServiceInstance> grayNodes = allInstances.stream()
            .filter(i -> "gray".equals(i.getMetadata().get("version")))
            .toList();
        if (!grayNodes.isEmpty()) {
            return grayNodes; // 优先走灰度实例
        }
    }
    // 兜底走基线版实例（避免灰度流量打到其他未知实验版本，或无灰度节点时保底可用）
    List<ServiceInstance> baseNodes = allInstances.stream()
        .filter(i -> !"gray".equals(i.getMetadata().get("version")))
        .toList();
    return baseNodes.isEmpty() ? allInstances : baseNodes;
}
```

#### 4. 面试官连环追问
- **追问 1：分布式限流每次请求都要查一次 Redis，会不会把 Redis 压垮或增加接口 RT？**
  - **优化方案（本地批量预取令牌 / 限流降级）**：应用服务每次向 Redis 批量申请一批令牌（如一次拿 100 个，本地内存消耗完再去 Redis 拿），将 Redis 请求量降低 100 倍（类似 Sentinel 集群流控 Token Server 思想）；同时设置极短的 Redis 超时（如 20ms），Redis 故障时自动降级为单机本地限流，绝不让限流组件拖垮业务。
- **追问 2：灰度发布期间发现新版本有 Bug，如何秒级回滚？**
  - 网关动态修改 Nacos/Apollo 配置中心里的灰度规则（把灰度比例调为 0% 或移除 Header 规则），全链路新流量 1 秒内全部切回 `base` 基线节点，无需等 K8s 重新打包镜像重启！

---

## 9. 海量数据处理（Bitmap / 布隆过滤器 / TopK / HyperLogLog）

### Q9：内存受限（如 1GB）下，如何处理 10 亿级海量数据的去重、排序与 TopK？

#### 1. 场景一：40 亿个不重复非负整数，如何判重与排序？
- **内存算账**：直接用 `HashSet<Integer>` 或 `int[]` 需 $4 \times 10^9 \times 4\text{B} \approx 16\text{GB}$ 内存，直接 OOM。
- **解法：Bitmap（位图）**：
  - 1 个 `bit` 代表 1 个整数是否存在（`byteIndex = num >>> 3`, `bitOffset = num & 7`）。
  - $40\text{ 亿 bit} \approx 500\text{ MB}$！轻松装入 1GB 内存，查询与插入均为 $O(1)$，从低位到高位扫描即可 $O(N)$ 完成排序。
  - **稀疏数据优化**：若数据范围大但实际元素稀疏（如仅几百万个），用 **`RoaringBitmap`**（将 32 位整型高 16 位作桶，低 16 位动态采用有序数组、Bitmap 或游程编码），可节省 90% 以上空间（ES、ClickHouse、Spark 底层均内置）。

#### 2. 场景二：10 亿个 URL 黑名单，如何极省内存地判断 URL 是否存在？
- **解法：布隆过滤器（Bloom Filter）**：
  - 长度为 $m$ 的位数组 + $k$ 个独立哈希函数。判断“不存在”时 **100% 准确**；判断“存在”时**有一定误判率**；原生不支持删除（若需删除可用 Counting Bloom Filter 或 Cuckoo Filter）。
  - **经典参数估算**：数据量 $n = 10^9$，期望误判率 $p = 0.01$（1%）：
    - 最优 bit 数 $m = -\frac{n \ln p}{(\ln 2)^2} \approx 9.58 \times n \approx 1.2\text{ GB}$；最优哈希函数个数 $k = \frac{m}{n} \ln 2 \approx 7$ 个。

#### 3. 场景三：100GB 日志文件存满 URL，内存只有 1GB，如何找出频率最高的 Top 100？
- **经典“分治 Hash + 小顶堆”四步法**：
  1. **Hash 取模分片**：逐行读取 100GB 大文件，计算 `(url.hashCode() & Integer.MAX_VALUE) % 1000`（生产推荐 MurmurHash3 分布更均匀），写入 1000 个小文件 `part_0` ~ `part_999`。**核心不变式：相同 URL 必进同一小文件**，每个文件约 100MB。
  2. **单文件频次统计**：依次将小文件读入 `HashMap<String, Integer>` 统计词频。
  3. **小顶堆筛选 Top 100**：维护容量为 100 的小顶堆 `PriorityQueue`，堆顶是当前 Top 100 中频次最小的。若某 URL 频次 > 堆顶，弹出堆顶并入堆；处理完清空 `HashMap`，只保留 100 条结果。
  4. **全局归并**：将 1000 个文件产出的 $1000 \times 100 = 10$ 万条候选结果，再用容量为 100 的小顶堆归并，得出全局精确 Top 100。

```java
// 单文件统计 TopK 核心代码模板
public List<Map.Entry<String, Integer>> topKInChunk(Map<String, Integer> freqMap, int k) {
    PriorityQueue<Map.Entry<String, Integer>> minHeap =
        new PriorityQueue<>(k, Comparator.comparingInt(Map.Entry::getValue));

    for (Map.Entry<String, Integer> entry : freqMap.entrySet()) {
        if (minHeap.size() < k) {
            minHeap.offer(entry);
        } else if (entry.getValue() > minHeap.peek().getValue()) {
            minHeap.poll();
            minHeap.offer(entry);
        }
    }
    List<Map.Entry<String, Integer>> res = new ArrayList<>(minHeap);
    res.sort((a, b) -> b.getValue() - a.getValue());
    return res;
}
```
- **🔥 面试官追问：如果某个超级热点 URL（如百度首页）出现几百亿次，导致某单个分片文件高达 50GB，依然装不下 1GB 内存怎么办？**
  - **解决热点倾斜**：先用 **Count-Min Sketch**（极小内存概率草图）扫一遍识别出超频热 Key 单独直接计数；或当单个分片超过 500MB 时，换用另一种哈希函数对该分片进行**二次哈希拆分子文件**（或者对分片文件做外部归并排序后再顺序计数）。

#### 4. 场景四：海量日活（UV）去重统计，如何用 12KB 内存统计上亿 UV？
- **解法：Redis `HyperLogLog`**：
  - 基于伯努利试验极值概率分布，无论统计 1 万还是 1 亿个不重复元素，**每个 Key 固定只占 12KB 内存**，标准误差仅 **0.81%**。
  - 命令：`PFADD uv:20261118 user_123`、`PFCOUNT uv:20261118`、`PFMERGE uv:week uv:day1 uv:day2 ...`（支持多天合并求周活而不重复计数）。

---

## 10. 多级缓存 + Canal Binlog 异步一致性架构

### Q10：高并发场景下如何设计多级缓存，并通过 Canal Binlog 保障缓存与 DB 的最终一致性？

#### 1. 架构全景：L1 Caffeine + L2 Redis + Canal CDC + MQ 广播

```text
读请求路径：
[Client] ──▶ [App Pod]
               ├── 1. 查 L1 Caffeine 本地缓存（命中 < 0.1ms，无网络开销）
               ├── 2. 查 L2 Redis 集群（命中回填 L1）
               └── 3. 查 L3 MySQL（带分布式锁/Singleflight 防缓存击穿）──▶ 回填 L2 & L1

写请求与异步一致性路径：
[App] ──▶ 1. 写 MySQL 主库（业务代码无需耦合多处删缓存逻辑）
               │ (Binlog ROW 格式)
               ▼
          [Canal Server]（伪装成 MySQL Slave 实时订阅 Binlog）
               │ (按表名+主键 ID Hash 投递保证同一行变更顺序消费)
               ▼
          [RocketMQ / Kafka]
               │
               ▼
          [缓存同步消费者 Sync Service]
               ├── 1. 删除/更新 L2 Redis 对应 Key（失败自动重试 + 死信告警）
               └── 2. 发送 Redis Pub/Sub（或 RocketMQ 广播模式）通知所有 App Pod
                        │
                        ▼
               [所有 App Pod] 收到消息，主动失效各自内存中的 L1 Caffeine Key
```

#### 2. 关键技术细节与避坑

1. **为什么选 Canal 监听 Binlog，而不是“先写 DB 再删缓存”或“延时双删”？**
   - 业务代码双写痛点：多处业务改同一张表容易漏删缓存；删缓存失败缺乏自动重试机制；延时双删需要 `Thread.sleep(500)` 阻塞工作线程或引入额外延迟任务，且时间不好拍脑袋确定。
   - **Canal 优势**：彻底解耦业务与缓存；支持 MQ 失败重试保障最终一致性；保证同一主键多次连续修改在 MQ 分区中**严格顺序执行**。
2. **Canal 异步延迟窗口（50ms~200ms）怎么处理？**
   - 绝大部分读场景完全可接受毫秒级最终一致；对于“用户修改完资料立刻刷新看到最新值”的强体验场景，可在写接口**同步删一次 Redis（或写入短期“自己写优先读主”Cookie 标记）**，再由 Canal 兜底二次异步清理。
3. **L1 本地缓存（Caffeine）必踩的 3 个坑**：
   - **坑 1（无上限导致 OOM）**：必须配置 `maximumSize(10_000)` 和 `expireAfterWrite(60, TimeUnit.SECONDS)`，使用 W-TinyLFU 淘汰，严禁无限增长。
   - **坑 2（多节点脏读）**：单机本地缓存无法被其他 Pod 感知失效，必须通过 **Redis Pub/Sub / MQ 广播主动逐出 + 兜底短 TTL**。
   - **坑 3（对象引用污染）**：Caffeine 返回的是堆内同一个 Java 对象引用！**绝对禁止调用方直接修改 Caffeine 返回对象的属性**，否则会直接污染缓存里的共享对象（应返回不可变对象 / Record 或深拷贝）。
4. **突发超级热 Key 处理（如明星官宣导致 Redis 单节点被打满）**：
   - 接入 **JD HotKey** 等热 Key 探测框架：客户端收集访问频次上报探测集群，一旦某 Key QPS 超阈值，主动下发推送到所有 App Pod 晋升入 L1 本地缓存，拦截 99% 打向 Redis 的流量。

#### 3. 生产级 L1 + L2 多级缓存读取代码模板（含防击穿）

```java
@Service
public class ProductCacheService {
    // L1: Caffeine 本地缓存（必须限制容量和 TTL）
    private final Cache<Long, ProductDTO> localCache = Caffeine.newBuilder()
            .maximumSize(10_000)
            .expireAfterWrite(60, TimeUnit.SECONDS)
            .recordStats()
            .build();

    private final RedissonClient redissonClient;
    private final StringRedisTemplate redisTemplate;
    private final ProductMapper productMapper;

    public ProductDTO getProduct(Long skuId) {
        // 1. 查 L1 本地缓存
        ProductDTO cached = localCache.getIfPresent(skuId);
        if (cached != null) {
            return cached == ProductDTO.EMPTY ? null : cached;
        }

        // 2. 查 L2 Redis
        String redisKey = "cache:product:" + skuId;
        String json = redisTemplate.opsForValue().get(redisKey);
        if (json != null) {
            ProductDTO dto = "NULL".equals(json) ? ProductDTO.EMPTY : JsonUtils.parse(json, ProductDTO.class);
            localCache.put(skuId, dto);
            return dto == ProductDTO.EMPTY ? null : dto;
        }

        // 3. L1/L2 均未命中：分布式互斥锁（或 Singleflight）防缓存击穿回源 DB
        RLock lock = redissonClient.getLock("lock:cache:product:" + skuId);
        lock.lock(5, TimeUnit.SECONDS);
        try {
            // Double Check Redis（防止排队线程重复回源 DB）
            String recheck = redisTemplate.opsForValue().get(redisKey);
            if (recheck != null) {
                ProductDTO dto = "NULL".equals(recheck) ? ProductDTO.EMPTY : JsonUtils.parse(recheck, ProductDTO.class);
                localCache.put(skuId, dto);
                return dto == ProductDTO.EMPTY ? null : dto;
            }

            ProductDO entity = productMapper.selectById(skuId);
            if (entity == null) {
                // 空值短 TTL 缓存防穿透
                redisTemplate.opsForValue().set(redisKey, "NULL", 60, TimeUnit.SECONDS);
                localCache.put(skuId, ProductDTO.EMPTY);
                return null;
            }
            ProductDTO dto = convert(entity);
            // 加随机过期时间防雪崩
            long ttl = 3600 + ThreadLocalRandom.current().nextInt(300);
            redisTemplate.opsForValue().set(redisKey, JsonUtils.toJson(dto), ttl, TimeUnit.SECONDS);
            localCache.put(skuId, dto);
            return dto;
        } finally {
            if (lock.isHeldByCurrentThread()) {
                lock.unlock();
            }
        }
    }
}
```

#### 4. 面试官连环追问
- **追问 1：Canal 监听 Binlog 时，为什么 MySQL 必须开启 `binlog_format = ROW`？**
  - `STATEMENT` 模式只记录 SQL 语句（如 `UPDATE t SET update_time = NOW()`），主从同步和 Canal 解析时无法拿到变更前后的精确字段值且容易因函数产生歧义；`ROW` 模式精确记录每一行变更前（Before）和变更后（After）的数据镜像，Canal 才能准确提取主键 ID 与变更字段。
- **追问 2：如果一次 `UPDATE` 批量更新了 100 万行数据，Canal 会不会瞬间产生 100 万条缓存失效消息把 MQ 和 Redis 打垮？**
  - 会！这是大厂常见线上事故（“批量扫表更新引发缓存雪崩”）。应对策略：1）严禁在线上大表执行无 `LIMIT` 的全表批量 `UPDATE`；2）Canal 消费者端对同一 Key 的短时间连续变更做**聚合去重（Debounce）**，并控制消费者并发限流，避免打爆 Redis。
- **追问 3：除了缓存失效删 Key，Canal 同步还常用于哪些架构场景？**
  - 1）**MySQL $\rightarrow$ Elasticsearch 实时异构索引同步**（解决多条件复杂搜索）；2）**MySQL $\rightarrow$ ClickHouse/Doris 实时数仓同步**（实现业务库与 OLAP 分析隔离）。

---

## 11. 云原生部署实战（Docker / K8s / 优雅上下线 / 可观测性）

### Q11：Spring Boot 微服务在 Docker & Kubernetes (K8s) 中如何做到高性能部署、优雅上下线与全链路可观测？

#### 1. 生产级多阶段精简 Dockerfile（Spring Boot 分层缓存 + 非 Root 安全基线）

```dockerfile
# 阶段一：利用 Spring Boot 分层特性解压 JAR，最大化利用 Docker Build Cache
FROM eclipse-temurin:21-jre-alpine AS builder
WORKDIR /app
ARG JAR_FILE=target/*.jar
COPY ${JAR_FILE} app.jar
RUN java -Djarmode=layertools -jar app.jar extract

# 阶段二：最终精简运行时镜像
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

# 按变更频率从低到高依次 COPY（业务代码改了，依赖层依然命中缓存）
COPY --from=builder /app/dependencies/ ./
COPY --from=builder /app/spring-boot-loader/ ./
COPY --from=builder /app/snapshot-dependencies/ ./
COPY --from=builder /app/application/ ./

RUN chown -R appuser:appgroup /app
USER appuser

ENTRYPOINT ["sh", "-c", "java ${JAVA_OPTS} org.springframework.boot.loader.launch.JarLauncher"]
```

#### 2. 容器化 JVM 内存模型与启动参数调优

- **为什么容器内不能写死 `-Xmx` 或忽视容器内存限制？**
  - 现代 JDK（8u191+ / 11 / 17 / 21）默认开启 `-XX:+UseContainerSupport` 自动读取 cgroup 内存上限。
  - **容器总内存组成** = `Heap` + `Metaspace` + `DirectMemory(Netty堆外)` + `Thread Stacks(线程数×1MB)` + `CodeCache` + `OS PageCache`。
  - 如果把 Pod `limits.memory` 设为 4Gi，却把 `-Xmx` 设为 3.6Gi，高并发下线程栈或 Netty 堆外内存一涨就会直接触发 Linux Kernel **OOMKilled（Exit Code 137）**！
- **推荐线上参数（4C 4Gi Pod 为例）**：
  ```bash
  -XX:+UseContainerSupport
  -XX:InitialRAMPercentage=70.0
  -XX:MaxRAMPercentage=75.0
  -XX:MetaspaceSize=256m
  -XX:MaxMetaspaceSize=512m
  -XX:+UseG1GC
  -XX:MaxGCPauseMillis=200
  -XX:+HeapDumpOnOutOfMemoryError
  -XX:HeapDumpPath=/var/log/heapdump/heapdump.hprof
  ```
  > 提示：K8s 中建议将 `resources.requests.memory` 与 `limits.memory` 设为**相等（Guaranteed QoS 级别）**，防止节点内存紧张时业务 Pod 被优先驱逐或 OOMKilled。

#### 3. K8s 部署 YAML（三大探针 + 优雅停机 + 零报错发布）

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  replicas: 4
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 25%
  template:
    spec:
      terminationGracePeriodSeconds: 60  # 必须大于 preStop + Spring优雅停机耗时
      containers:
        - name: order-service
          image: registry.example.com/order-service:1.2.0
          resources:
            requests: { cpu: "2", memory: "4Gi" }
            limits:   { cpu: "4", memory: "4Gi" }
          startupProbe:
            httpGet: { path: /actuator/health/liveness, port: 8080 }
            failureThreshold: 30   # 最多给 150 秒冷启动缓冲
            periodSeconds: 5
          readinessProbe:
            httpGet: { path: /actuator/health/readiness, port: 8080 }
            periodSeconds: 5
            failureThreshold: 3    # 连续失败 3 次从 Service Endpoints 摘除流量
          livenessProbe:
            httpGet: { path: /actuator/health/liveness, port: 8080 }
            periodSeconds: 10
            failureThreshold: 3    # 连续失败 3 次 K8s 重启 Pod
          lifecycle:
            preStop:
              exec:
                # 1. 等待 kube-proxy/Ingress 摘除 Endpoint 规则生效
                # 2. 若用 Nacos/Dubbo，可在此先调用服务下线接口
                command: ["sh", "-c", "sleep 10"]
```

- **🔥 生产致命天坑**：**严禁在 `livenessProbe` 中检查 MySQL / Redis / 下游 RPC 的健康状态！**
  - 一旦 MySQL 短暂抖动 15 秒，K8s 会误判全集群几十个业务 Pod 全部死锁并集体杀掉重启，引发雪崩！`liveness` 只检查 JVM 进程自身存活，`readiness` 才检查依赖连通性（断连时只摘流不重启）。
- **Spring Boot 侧优雅停机配置**：
  ```yaml
  server:
    shutdown: graceful
  spring:
    lifecycle:
      timeout-per-shutdown-phase: 30s  # 等待在途 HTTP 请求与线程池任务执行完毕
  ```

#### 4. 云原生可观测性三支柱（Metrics / Logging / Tracing）

| 支柱 | 核心组件栈 | 关键监控指标与实战排查作用 |
| :--- | :--- | :--- |
| **Metrics（指标）** | `Micrometer` + `Prometheus` + `Grafana` + `Alertmanager` | 重点监控：`jvm_gc_pause_seconds`（STW 耗时）、`hikaricp_connections_active`（DB 连接池打满）、`tomcat_threads_busy_threads`、接口 QPS 与 P99 延迟 |
| **Logging（日志）** | `Logback JSON` + `Filebeat/Promtail` + `ELK / Grafana Loki` | 输出单行 JSON 含 `traceId`/`spanId`，避免多行异常堆栈被拆碎；低成本高压缩推荐 Loki |
| **Tracing（链路）** | `OpenTelemetry (OTel)` / `SkyWalking` + MDC | 字节码无侵入埋点，贯穿 HTTP/Feign/Dubbo/Redis/MySQL/MQ，实现“监控告警 $\rightarrow$ Trace 定位慢 SQL $\rightarrow$ 关联 TraceId 查错误日志”闭环 |

#### 5. 面试官连环追问
- **追问 1：Pod 频繁发生 `OOMKilled (Exit Code 137)`，但 Heap Dump 里堆内存才用了 50%，怎么排查？**
  - 说明是**堆外内存（Native Memory）溢出**！排查步骤：启动时加上 `-XX:NativeMemoryTracking=detail`，线上用 `jcmd <pid> VM.native_memory detail` 对比 `baseline` 增长项，重点排查：1）Netty `DirectByteBuffer` 泄漏（`PooledByteBuf` 未 `release()`）；2）线程池创建过多线程导致 Thread Stack 膨胀；3）JNI / GZIP / 第三方 C 库内存未释放或 glibc `arena` 内存碎片。
- **追问 2：K8s 如何配合业务高峰实现自动弹性扩缩容（HPA）？**
  - 配置 `HorizontalPodAutoscaler` 基于 CPU 利用率（如 70%）或 Prometheus 自定义业务指标（如 QPS / MQ 积压量）触发扩容；**核心避坑**：Java 应用冷启动较慢（15~60s），必须配置 `readinessProbe` 防止未启动完就被打入流量，并设置 HPA 缩容冷却窗口（`stabilizationWindowSeconds: 300`）防止流量抖动导致 Pod 频繁创建销毁（抖动震荡），同时配合 `PodDisruptionBudget (PDB)` 保障节点维护时至少保留一定比例的可用副本。

---

## 12. 2025-2026 前沿趋势：Java 后端 AI 工程化（Spring AI / RAG / MCP）

### Q12：Java 后端如何设计高可用企业级 AI 应用（Spring AI + RAG + Function Calling + MCP）？

越来越多的 Java 后端面试考察如何将大模型稳定接入现有微服务体系，核心考点集中在 **Spring AI 架构、SSE 流式输出与断连控制、企业知识库混合检索 RAG、MCP 标准协议、以及成本与幻觉护栏**。

#### 1. Spring AI 核心架构与 `Advisor` 拦截链

- **模型无关抽象**：通过 `ChatClient`、`EmbeddingModel`、`VectorStore` 标准接口，仅改配置即可在 **OpenAI、DeepSeek、通义千问（DashScope）、本地 Ollama** 之间无缝切换。
- **`Advisor` 切面责任链（类似 Spring MVC Interceptor）**：
  - `MessageChatMemoryAdvisor`：自动持久化多轮对话上下文到 Redis/JDBC（配合滑动窗口截断防超 Token 上限）。
  - `QuestionAnswerAdvisor`：自动触发向量检索并注入 RAG 上下文。
  - `SafeGuardAdvisor`：前置敏感词/越狱 Prompt 拦截与后置合规校验。
- **结构化输出（`BeanOutputConverter`）**：自动在 Prompt 注入 JSON Schema 约束，并将 LLM 文本输出强类型反序列化为 Java `Record` / POJO，直接驱动业务工作流。

#### 2. 大模型流式输出（SSE）与背压/断连省钱控制

大模型首字延迟（TTFT）约 0.5~2s，完整输出耗时 10~30s，必须使用 **SSE（Server-Sent Events）** 流式推送：

```java
@GetMapping(value = "/ai/chat/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<ServerSentEvent<String>> streamChat(@RequestParam String sessionId,
                                                @RequestParam String query) {
    return chatClient.prompt()
            .user(query)
            .advisors(a -> a.param(ChatMemory.CONVERSATION_ID, sessionId))
            .stream()
            .content()
            .map(token -> ServerSentEvent.builder(token).event("delta").build())
            .concatWithValues(ServerSentEvent.builder("[DONE]").event("done").build())
            .doOnCancel(() -> log.info("客户端主动取消连接, sessionId={}", sessionId))
            .onErrorResume(ex -> Flux.just(
                ServerSentEvent.builder("AI 服务繁忙，请稍后重试").event("error").build()
            ));
}
```
- **🔥 生产落地 3 大细节**：
  1. **Nginx 必须关闭缓冲**：配置 `proxy_buffering off;`、`proxy_cache off;`、`chunked_transfer_encoding on;`，否则 Nginx 会攒满缓冲区才吐给前端，导致界面卡死十几秒突然一次性弹出整段文字。
  2. **客户端取消及时中断推理**：当用户点击“停止生成”或关闭页面时，WebFlux 会触发 `doOnCancel` 并取消下游 HTTP 请求，避免模型继续空跑浪费高额 Token 费用。
  3. **线程模型选型**：若使用传统 Spring MVC `SseEmitter` 配合同步阻塞调用，务必开启 **JDK 21 虚拟线程**（`spring.threads.virtual.enabled=true`）避免耗尽 Tomcat 工作线程；或直接使用响应式 **Spring WebFlux**。

#### 3. 企业知识库 RAG（混合检索 + 重排）标准流水线

```text
【离线文档入库 Pipeline】
多源文档(PDF/Word/MD/Wiki) ──▶ Apache Tika / PDFBox 解析清洗
  ──▶ 文本切片 (RecursiveCharacterTextSplitter: chunkSize=500, overlap=80)
  ──▶ Embedding 向量化 (如 bge-m3 / text-embedding-v3)
  ──▶ 存入向量数据库 (Milvus HNSW索引 / PGVector / Elasticsearch) + 保留倒排索引与权限标签

【在线检索与生成 Pipeline】
用户提问 ──▶ 1. Query 改写/指代消解（结合历史对话补全主语）
         ──▶ 2. 混合检索（Hybrid Search）：向量语义检索 Top-20 + BM25 关键词检索 Top-20
         ──▶ 3. RRF 融合与精排：RRF(d) = Σ 1 / (60 + rank_i(d)) 合并后送入 Rerank 模型取 Top-5
         ──▶ 4. 权限 Metadata 过滤（仅召回当前员工有权查看的部门文档，严防越权泄密！）
         ──▶ 5. 拼接 Grounding Prompt（强制引用来源编号，无依据时拒答）──▶ LLM 流式输出
```
- **为什么不能只用纯向量检索？**
  - 向量检索擅长语义同义匹配（“怎么报销差旅费”匹配“出差费用结算规范”），但对**精确工单号、错误码（如 `ERR_5023`）、专有产品型号**检索效果极差！工业界标配必须是 **Vector 语义 + BM25 关键词混合检索 + Cross-Encoder Rerank 重排**。

#### 4. Function Calling 与 MCP（Model Context Protocol）协议

- **MCP 解决了什么痛点？**：过去 $M$ 个 AI 应用对接 $N$ 个企业内部数据源（MySQL、GitLab、订单系统），需要写 $M \times N$ 套胶水集成代码。**MCP 是 AI 时代的“USB-C 标准协议”**，将业务系统封装为标准 **MCP Server**，统一对外暴露：
  - **Tools**（可执行工具，如 `queryOrderStatus`、`createTicket`）
  - **Resources**（只读上下文资源，如 DB Schema、系统日志）
  - **Prompts**（标准化提示词模板）
- **Java 落地代码骨架（Spring AI MCP Server）**：

```java
@Service
public class OrderMcpTools {
    @Tool(description = "根据订单号查询当前订单状态与物流单号")
    public String queryOrderStatus(
            @ToolParam(description = "电商订单号，如 ORD20260001") String orderNo) {
        // 必须校验当前登录用户是否有权查看该订单，严防越权！
        OrderDTO order = orderService.getOrderByNo(orderNo);
        return order == null ? "未找到该订单" : order.toSummaryText();
    }
}

@Configuration
public class McpServerConfig {
    @Bean
    public ToolCallbackProvider orderToolProvider(OrderMcpTools orderMcpTools) {
        return MethodToolCallbackProvider.builder().toolObjects(orderMcpTools).build();
    }
}
```
- **安全警告**：AI 自主调用 Tool 必须做**RBAC 权限校验 + 高危操作人工二次确认（Human-in-the-loop）**，防范 Prompt 注入攻击诱使模型调用删库或退款接口！

#### 5. 生产级护栏：超时熔断、Token 成本控制与幻觉治理
1. **超时与多模型容灾**：设置合理的首字超时（5s）与总超时（60s），结合 Resilience4j / Sentinel 实现主备模型自动切换（如 DeepSeek 限流时自动降级到通义千问或预设 FAQ）。
2. **Token 成本优化**：
   - **语义缓存（Semantic Cache）**：将高频提问向量化存入 Redis Stack，新提问相似度 $> 0.95$ 直接命中缓存返回，**0 Token 消耗、毫秒级响应**。
   - **大小模型路由**：简单意图分类、槽位提取走低成本小模型，复杂推理才调度旗舰大模型。
3. **幻觉与合规治理**：
   - **输入侧**：正则/NLP 脱敏手机号、身份证、银行卡等 PII 数据后再发送给外部公有云大模型；拦截常见越狱提示词。
   - **输出侧**：若 RAG 召回最高相关性分数低于阈值（如 `< 0.6`），直接触发兜底拒答并引导转人工，绝不让模型自由发挥编造虚假企业制度或价格信息。

#### 6. 面试官连环追问
- **追问 1：RAG 系统上线后，怎么科学评估回答质量好不好（而不是靠人工盲测）？**
  - 引入自动化评估框架（如 **RAGAS** / TruLens），从 4 个量化维度评估：1）**Context Recall / Precision**（检索召回率与精确率——排查是切片或向量检索出了问题）；2）**Faithfulness（忠实度）**（答案是否严格忠于检索上下文，有无幻觉）；3）**Answer Relevance（回答相关性）**（是否答非所问）。
- **追问 2：企业文档切片（Chunking）大小怎么选？遇到表格、图片怎么办？**
  - 切片太小会丢失上下文语义，太大则引入过多噪声稀释向量精度。常规文本推荐 **300~500 Tokens + 10%~20% Overlap（重叠窗口）**，且优先按 Markdown 标题/段落边界切分（Semantic/Recursive Splitting）；遇到 PDF 复杂跨页表格，不要直接拆碎成纯文本，应借助多模态模型或版面分析工具（如 MinerU / Unstructured）转成 **Markdown 表格或 HTML 整体作为一个 Chunk**，或者采用“**父子文档索引（Parent-Child）**”（用小块精准检索，召回后把所属完整父段落喂给 LLM）。
