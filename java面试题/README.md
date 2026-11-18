# Java & 后端开发面试完全指南（从最基础到项目架构）

> 本仓库汇总了全网（GitHub 高赞开源项目、一线大厂高频真题、牛客/掘金面经）的 **Java 与后端开发核心面试题**，覆盖 **Java 基础 → 集合源码 → 并发编程 → JVM 调优 → 计算机基础与设计模式 → MySQL → Redis → Spring/Spring Boot/MyBatis → 消息队列/ES/Netty/RPC → 分布式与微服务 → 高并发项目架构与系统设计**，可直接用于日常复习、面试冲刺与 GitHub 知识库沉淀。

---

## 📚 目录导航

| 序号 | 模块 | 核心内容 | 文件链接 |
| :--- | :--- | :--- | :--- |
| 01 | **Java 核心基础** | OOP、JDK 8~21 新特性、String 常量池、反射与动态代理、泛型、异常、BIO/NIO/AIO | [01-java-basics.md](./01-java-basics.md) |
| 02 | **Java 集合框架** | `ArrayList`/`LinkedList`、`HashMap` 1.7 vs 1.8 源码、`ConcurrentHashMap`、`ThreadLocal`、`BlockingQueue` | [02-java-collections.md](./02-java-collections.md) |
| 03 | **Java 并发编程 (JUC)** | JMM、`volatile`、`synchronized` 锁升级、CAS 与 ABA、AQS、`ReentrantLock`、线程池、JDK 21 虚拟线程 | [03-java-concurrency.md](./03-java-concurrency.md) |
| 04 | **JVM 底层与调优** | 运行时数据区、对象创建与内存布局、类加载与双亲委派、GC 算法与 CMS/G1/ZGC、OOM 排查与 Arthas 实战 | [04-jvm-internals-tuning.md](./04-jvm-internals-tuning.md) |
| 05 | **计算机基础与设计模式** | TCP/UDP、HTTP/HTTPS、IO 多路复用、进程/线程/协程、Linux 常用排查命令、23 种设计模式及源码应用 | [05-cs-fundamentals.md](./05-cs-fundamentals.md) |
| 06 | **MySQL 深度解析** | InnoDB vs MyISAM、B+树索引、回表与索引下推、ACID 与三大日志、MVCC、行锁/间隙锁/临键锁、慢 SQL 与分库分表 | [06-mysql-database.md](./06-mysql-database.md) |
| 07 | **Redis 原理与缓存** | 五大结构底层编码、IO 多路复用、RDB/AOF/混合持久化、主从/哨兵/Cluster、穿透/击穿/雪崩、双写一致性、分布式锁 | [07-redis-caching.md](./07-redis-caching.md) |
| 08 | **Spring 与主流框架** | IoC 与 Bean 生命周期、循环依赖与三级缓存、AOP 与 `@Transactional` 失效、Spring MVC、Spring Boot 3 自动装配、MyBatis | [08-spring-framework.md](./08-spring-framework.md) |
| 09 | **MQ / 搜索 / 网络 / RPC** | Kafka vs RocketMQ、消息不丢失/幂等/顺序/积压、Elasticsearch 倒排索引与调优、Netty 零拷贝、Dubbo SPI 与负载均衡 | [09-mq-search-rpc.md](./09-mq-search-rpc.md) |
| 10 | **分布式与微服务架构** | CAP/BASE、Paxos/Raft、分布式事务 (2PC/TCC/Seata)、雪花算法与时钟回拨、Nacos、Gateway、Sentinel、SSO/JWT | [10-distributed-microservices.md](./10-distributed-microservices.md) |
| 11 | **项目架构与系统设计** | 秒杀系统、短链系统、抢红包、订单支付状态机、海量数据去重/TopK、Docker/K8s、Spring AI/RAG 架构、项目难点话术 | [11-system-design-architecture.md](./11-system-design-architecture.md) |

---

## 🗺️ 面试复习路线图

```text
第一阶段：语言与底层内功（夯实基础）
  01 Java基础 ──▶ 02 集合源码 ──▶ 03 并发编程(JUC) ──▶ 04 JVM调优 ──▶ 05 计网/OS/设计模式

第二阶段：存储与核心框架（工程主力）
  06 MySQL ──▶ 07 Redis ──▶ 08 Spring / Spring Boot / MyBatis

第三阶段：中间件与分布式（中高级分水岭）
  09 Kafka / RocketMQ / ES / Netty / Dubbo ──▶ 10 分布式事务 / 限流熔断 / 微服务治理

第四阶段：系统设计与项目落地（冲击高薪 / 架构师）
  11 秒杀 / 短链 / 分库分表 / 云原生 / AI工程化 / 项目STAR法则复盘
```

---

## 🎯 面试答题通用方法论（三段式高分模板）

面对面试官的“连环炮”追问，切忌只背干瘪的概念，建议采用 **“结论先行 → 底层原理 → 工程实践/避坑”** 的三段式回答法：

1. **结论先行（10秒内给出核心答案）**：先直接回答“是什么 / 选哪个 / 能不能”，让面试官知道你抓住了重点。
2. **底层原理（展示技术深度）**：结合源码、数据结构、内存模型或网络协议讲清楚“为什么”。
3. **工程实践与权衡（展示项目经验）**：结合真实业务场景说明 Trade-off（取舍）、踩过的坑、监控指标与线上排查手段。

---

## 🌟 GitHub 经典开源参考仓库推荐

本资料在整理过程中参考并融合了以下 GitHub 开源社区公认的高质量 Java 面试/进阶仓库的知识体系：

- **[Snailclimb/JavaGuide](https://github.com/Snailclimb/JavaGuide)** — Java 学习与面试指南（涵盖 Java、JVM、并发、MySQL、Redis、系统设计等）。
- **[doocs/advanced-java](https://github.com/doocs/advanced-java)** — 互联网 Java 工程师进阶知识完全扫盲（高并发、分布式、高可用、微服务）。
- **[CyC2018/CS-Notes](https://github.com/CyC2018/CS-Notes)** — 技术面试必备基础知识（算法、操作系统、网络、面向对象、数据库、Java）。
- **[xbox1994/Java-Interview](https://github.com/xbox1994/Java-Interview)** — 高级 Java 后台开发面试指南（偏大厂实战与架构基础）。
- **[OUYANGSIHAI/JavaInterview](https://github.com/OUYANGSIHAI/JavaInterview)** — 互联网后端面试指南（含源码分析与系统设计）。

---

## 💡 2025-2026 面试新趋势提示

1. **技术底座集体换挡**：主流大厂已全面迁移至 **JDK 17 / JDK 21+** 与 **Spring Boot 3.x**，面试高频新增 **虚拟线程 (Virtual Threads) 与 Pinning 避坑**、`AutoConfiguration.imports` 新装配机制、**GraalVM Native Image (AOT 编译)** 等考点。
2. **重场景轻死记（八股占比降至 ~30%）**：纯概念背诵减少，面试官更关注“线上 CPU 100% / OOM / 虚拟线程卡顿怎么用 JFR 和 Arthas 排查？”、“Redis 挂了或 MQ 积压百万消息怎么止血？”。
3. **云原生可观测性与 AI 工程化加分项**：具备 **OpenTelemetry (OTel) 全链路追踪与异步上下文传播**、**Docker/K8s 健康检查与内存调优**、以及 **Spring AI + RAG 向量检索 + MCP (Model Context Protocol) 协议落地经验** 的候选人在社招和校招中极具竞争力。
