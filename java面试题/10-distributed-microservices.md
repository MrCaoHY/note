# 10 - 分布式理论与微服务架构面试题

> 本章覆盖：CAP 与 BASE 理论、一致性算法（Paxos/Raft）、分布式事务（2PC/TCC/Saga/Seata）、分布式 ID 与雪花算法时钟回拨、微服务组件（Nacos/Gateway/Sentinel）、分布式链路追踪、单点登录 SSO 与 JWT。

---

## 1. 分布式核心理论

### Q1：什么是 CAP 定理？为什么无法同时满足 C、A、P？什么是 BASE 理论？
- **CAP 定理**：一个分布式系统最多只能同时满足以下三项中的两项：
  - **C（Consistency，强一致性）**：所有节点在同一时间看到的数据完全一致。
  - **A（Availability，高可用性）**：非故障节点必须在合理时间内返回正常响应（不能无限等待或报错）。
  - **P（Partition Tolerance，分区容错性）**：网络发生脑裂/丢包分区时，系统仍能继续对外服务。
- **为什么分布式系统只能选 CP 或 AP？**：因为网络分区（P）在分布式环境下是**客观必然存在**的，不可放弃。当发生网络分区时，如果要求强一致（C），则必须阻塞或拒绝写请求（牺牲 A）；如果要求高可用（A），则允许节点各自服务但数据暂时不一致（牺牲 C）。
  - **CP 代表**：ZooKeeper、Etcd、HBase、Nacos（CP 模式）。
  - **AP 代表**：Eureka、Cassandra、Nacos（AP 默认模式）。
- **BASE 理论（对 AP 的延伸）**：**Basically Available（基本可用）**、**Soft State（软状态）**、**Eventually Consistent（最终一致性）**。核心思想是放弃强一致性，通过异步补偿保证数据最终一致，换取高可用与高吞吐。

---

### Q2：简述 Raft 一致性算法的核心原理？（Redis 哨兵 / Nacos / Etcd 均用到）
Raft 将复杂的一致性问题拆解为三个子问题：**Leader 选举**、**日志复制**、**安全性**。
- **三种节点状态**：`Leader`（唯一处理写请求并同步日志）、`Follower`（被动响应请求）、`Candidate`（选举候选人）。
- **选举过程**：每个 Follower 有随机化的选举超时时间（如 150~300ms）。若超时未收到 Leader 心跳，状态变为 Candidate，自增任期号 `Term` 并向所有节点发起拉票；获得**超过半数（N/2 + 1）**选票即当选新 Leader（随机超时机制极大避免了平票活锁）。
- **日志复制（两阶段）**：Leader 收到写请求写入本地日志 → 并行发送 `AppendEntries` 给所有 Follower → 收到超过半数节点确认写入成功 → Leader 提交（Commit）该日志并通知 Follower 提交。

---

## 2. 分布式事务（高频重点）

### Q3：分布式事务有哪些主流解决方案？各有什么优缺点？
| 方案 | 一致性 | 性能与入侵性 | 核心原理与适用场景 |
| :--- | :--- | :--- | :--- |
| **2PC / XA** | 强一致 | 性能差，锁资源时间长 | 协调者分 Prepare（预提交，锁资源）和 Commit 两阶段提交。适合传统金融强一致低并发场景 |
| **Seata AT 模式** | 最终一致 | **零业务侵入**，性能较好 | 一阶段提交本地事务并记录 `UNDO_LOG` 前后镜像 + 持有全局锁；二阶段异步删日志或反向生成 SQL 回滚 |
| **TCC（Try-Confirm-Cancel）** | 最终一致 | **性能极高**，但**代码侵入极大** | 每个服务手写 Try（冻结资源）、Confirm（确认扣减）、Cancel（释放冻结），需自行处理幂等、空回滚、防悬挂 |
| **可靠消息最终一致性** | 最终一致 | 高并发异步解耦 | 基于 **RocketMQ 事务消息** 或 **本地消息表 + 定时任务**，保证“本地事务成功与消息发送成功”原子绑定 |
| **Saga 模式** | 最终一致 | 长事务编排 | 将长事务拆为多个本地短事务 $T_1, T_2 \dots T_n$，任意失败则逆序执行补偿操作 $C_n \dots C_1$ |

> **面试追问**：TCC 中的**空回滚**和**悬挂**问题是什么？怎么解决？  
> - **空回滚**：Try 请求因网络超时未到达，但协调者触发了 Cancel。解法：记录一张事务控制表，Cancel 时检查若未收到 Try，直接返回成功而不做实际扣减。  
> - **防悬挂**：Cancel 先于超时的 Try 到达并执行了空回滚，随后迟到的 Try 才到达导致资源永久冻结。解法：空回滚时在事务控制表插入一条状态为“已回滚”的记录，Try 执行前先检查该记录，若已回滚则直接拒绝。

---

## 3. 分布式 ID

### Q4：分布式 ID 有哪些生成方案？详细说说雪花算法（Snowflake）及如何解决时钟回拨？
- **常见方案对比**：
  1. **UUID**：本地生成无网络开销，但**无序且长（36位）**，作为 MySQL 主键会导致 **B+ 树频繁页分裂**，性能极差。
  2. **Redis `INCR`**：有序高性能，但依赖 Redis 持久化与集群高可用。
  3. **数据库号段模式（如美团 Leaf-segment）**：每次从 DB 批量拉取一个号段（如 `1000~2000`）缓存在应用内存中分发，性能极高且趋势递增。
  4. **雪花算法（Snowflake，64-bit Long）**：
     - `1 bit` 符号位（固定为 0）
     - `41 bit` 毫秒级时间戳（可用约 69 年）
     - `10 bit` 机器 ID（5 bit 机房 + 5 bit 机器，最多 1024 台）
     - `12 bit` 毫秒内序列号（单机每毫秒最多生成 $2^{12} = 4096$ 个 ID）
- **时钟回拨问题（必问追问）**：服务器 NTP 校时导致系统时间倒退，可能生成重复 ID。
  - **解决方案**：
    1. **小幅回拨（如 $\le 5ms$）**：`Thread.sleep` 等待时钟追赶上上次生成时间。
    2. **较大回拨**：直接抛出异常报警，或**切换备用 WorkerID**（预留部分 WorkerID 专门在回拨时漂移使用）。
    3. **不依赖本机时钟（根本解法）**：改用数据库号段模式（Leaf-segment）或百度 UidGenerator（基于自增时间步长）。

---

## 4. 微服务核心组件与鉴权

### Q5：Nacos 和 Eureka / ZooKeeper 的区别？Nacos 配置动态刷新原理？
- **区别**：
  - **Eureka**（已停更）：AP 模型，纯内存注册表，客户端定时拉取与心跳续约，存在自我保护机制。
  - **ZooKeeper**：CP 模型，基于 ZAB 协议，选举期间不可用，不适合大规模微服务注册中心。
  - **Nacos**：**同时支持 AP（默认，基于 Distro 协议）和 CP（基于 Raft 协议，适合持久化实例）**，且集成了**配置中心**，支持长轮询实时推送。
- **Nacos 配置动态刷新原理（长轮询 Long Polling）**：
  - 客户端向 Nacos Server 发起一个带超时时间（默认 30 秒）的 HTTP 请求监听配置 MD5 变化；若配置未变，服务端将请求挂起（AsyncContext）直到配置变更立即返回，或 30 秒超时后客户端再次发起下一轮请求，兼具实时性与低开销。

---

### Q6：Sentinel 的限流算法有哪些？熔断降级的状态机是怎样的？
- **4 种经典限流算法**：
  1. **固定窗口计数器**：实现简单，但存在**临界突变问题**（窗口交界处 2 倍流量突刺）。
  2. **滑动时间窗口（Sentinel 默认统计基石）**：将大窗口切分为多个小格子，平滑统计最近 1 秒请求数，解决临界问题。
  3. **漏桶算法（Leaky Bucket）**：以**恒定速率**流出请求，平滑突发流量（Sentinel 排队等待模式）。
  4. **令牌桶算法（Token Bucket，Guava RateLimiter / Gateway 默认）**：以恒定速率向桶中放令牌，允许桶内积攒一定量令牌以**应对合理的突发流量**。
- **熔断器三种状态（Circuit Breaker）**：
  - **`Closed`（关闭，正常）** → 达到异常比例/慢调用比例阈值 → **`Open`（打开，直接快速失败走降级逻辑）** → 经过熔断时长 `timeWindow` → **`Half-Open`（半开，放行 1 个试探请求）** → 成功则回到 `Closed`，失败则重新回到 `Open`。

---

### Q7：分布式系统中的单点登录（SSO）和 JWT 鉴权怎么实现？JWT 有什么缺点？
- **JWT（JSON Web Token）结构**：`Header（算法与类型）.Payload（用户信息与过期时间，Base64 编码未加密，勿放密码）.Signature（密钥签名防篡改）`。
- **JWT 的缺点与解决**：
  1. **无法主动注销/踢人下线**：Token 一旦签发在过期前一直有效。**解法**：结合 Redis 维护黑名单（注销时将 Token 存入 Redis 黑名单至过期时间），或使用“短有效期 AccessToken + 长有效期 RefreshToken”双 Token 机制。
  2. **体积比 SessionId 大**，每次请求携带占用带宽。
- **单点登录（SSO）核心流程（CAS / OAuth2 思想）**：
  - 用户访问系统 A（未登录） → 重定向到统一认证中心（SSO Server） → 用户输入账号密码登录，SSO Server 种下全局 Cookie 并生成一次性授权码 `Ticket/Code` 重定向回系统 A → 系统 A 后端拿着 `Code` 向 SSO Server 校验换取用户信息/Token，完成局部登录；访问系统 B 时同理自动免密完成校验。

---

## 5. 可观测性与 OpenTelemetry（2026 必考）

### Q8：什么是可观测性的三大支柱？OpenTelemetry (OTel) 如何解决异步场景下的 TraceId 丢失？
- **三大支柱**：
  1. **Traces（链路追踪）**：记录一次请求跨微服务的完整调用链（TraceId + SpanId），定位耗时瓶颈。
  2. **Metrics（指标监控）**：聚合数值型指标（QPS、P99 延迟、JVM GC、CPU），配合 **Prometheus + Grafana** 告警。
  3. **Logs（日志）**：离散事件记录，通过注入 TraceId 实现日志与链路联动检索。
- **OpenTelemetry 行业收敛**：OTel 已成为云原生 CNCF 事实标准，统一了 Trace/Metrics/Logs 的 API 与采集协议（OTLP），Spring Boot 3 原生通过 **Micrometer Tracing** 无缝对接 OTel。
- **异步线程池 / 虚拟线程中 TraceId 丢失怎么办？**
  - **原因**：传统追踪依赖 `ThreadLocal`（MDC）存储 TraceContext，切换线程时上下文不会自动传递。
  - **解法**：
    1. **线程池包装**：使用阿里 `TransmittableThreadLocal (TTL)` 或 OTel `Context.taskWrapping(executor)` / Micrometer `ContextSnapshot` 包装 `ExecutorService` 和 `CompletableFuture`。
    2. **消息队列 / RPC 透传**：将 `traceparent` 写入 HTTP Header、Dubbo Attachment 或 MQ Message Header，消费端提取后恢复 Context。
    3. **虚拟线程注意**：避免在百万虚拟线程中滥用重量级 `ThreadLocal`，优先结合 OTel Java Agent 无侵入字节码增强或 JDK 21 `ScopedValue`。
