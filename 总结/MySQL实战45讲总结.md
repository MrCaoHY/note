# 《MySQL 实战 45 讲》核心知识精炼总结

> 基于《MySQL 实战 45 讲》（林晓斌 / 丁奇）全专栏 45 讲及技术答疑提炼，剔除寒暄、故事铺垫与重复表述，聚焦底层原理、参数机制、锁规则、排查链路与工程最佳实践。

---

## 目录

- [一、基础架构与日志系统（01–02、15）](#一基础架构与日志系统010215)
- [二、事务隔离与 MVCC 机制（03、08）](#二事务隔离与-mvcc-机制0308)
- [三、索引原理、选型与优化（04–05、09–11、18）](#三索引原理选型与优化0405091118)
- [四、锁体系全解与加锁规则（06–07、19–21、30、40）](#四锁体系全解与加锁规则060719213040)
- [五、SQL 执行原理与性能调优（12–14、16–17、19、33–38、44）](#五sql-执行原理与性能调优1214161719333844)
- [六、高可用、主从同步与持久化进阶（22–29）](#六高可用主从同步与持久化进阶2229)
- [七、运维实战、表结构演进与底层边界（31–32、39、41–43、45）](#七运维实战表结构演进与底层边界313239414345)

---

## 一、基础架构与日志系统（01–02、15）

### 1.1 两层架构与查询链路
MySQL 分为 **Server 层**（跨引擎共用）与 **存储引擎层**（插件式，MySQL 5.5.5 起默认 **InnoDB**）：
1. **连接器**：握手认证并**在连接建立时读取权限**（**已建立的长连接不感知后续权限修改**）。空闲状态为 `Sleep`，受 `wait_timeout`（默认 8 小时）控制。长连接临时内存挂在连接对象上，易导致 OOM，可通过定期重连或 MySQL 5.7+ 调用 `mysql_reset_connection` 复位释放。
2. **查询缓存**：Key 为完整 SQL 字符串；**表有任何更新即清空该表全部缓存**；`query_cache_type=DEMAND` 可关闭默认缓存（用 `SQL_CACHE` 显式启用）。**MySQL 8.0 已彻底移除**。
3. **分析器**：词法分析 + 语法分析（报错 `ERROR 1064`）。
4. **优化器**：选择索引、决定多表 `JOIN` 顺序。
5. **执行器**：先校验表级权限（`ERROR 1142`），再调用引擎接口循环取数。慢日志 `rows_examined` 由执行器每调用一次引擎取行加 1，可能小于引擎内部实际扫描行数。

### 1.2 `redo log` vs `binlog`
| 对比维度 | `redo log` | `binlog` |
| :--- | :--- | :--- |
| **所属层级** | InnoDB 引擎层特有 | Server 层（所有引擎共用） |
| **日志类型** | 物理日志（记录“在某个数据页上做了什么修改”） | 逻辑日志（记录语句原始逻辑） |
| **写入方式** | 固定大小循环写（如 4 个 1GB 文件）；`write pos` 追上 `checkpoint` 时写操作阻塞等待刷脏推进 | 追加写；达上限切换新文件，不覆盖旧日志 |
| **核心能力** | WAL（先写日志再写盘）提供 **crash-safe** 能力 | 归档、时间点恢复（PITR）、主从复制 |

### 1.3 更新流程与两阶段提交（2PC）
执行 `update T set c=c+1 where ID=2` 的核心路径：
1. 执行器取 `ID=2` 这一行（内存无则从盘读入 Buffer Pool），计算新值后调用引擎写入接口。
2. InnoDB 更新内存数据页，并将本次修改写入 `redo log`，置为 **`prepare` 状态**。
3. 执行器生成 `binlog` 并写入磁盘。
4. 执行器调用引擎提交接口，InnoDB 将 `redo log` 改为 **`commit` 状态**。

- **为什么必须两阶段提交**：保证 `redo log`（主库崩溃恢复基准）与 `binlog`（备库/恢复库基准）逻辑一致。若不采用 2PC，无论先写谁，在两者间隙崩溃都会造成主库与恢复库/备库数据不一致。
- **崩溃恢复判定（按 `XID` 关联）**：
  - `redo log` 完整（`prepare` + `commit`）→ 直接提交；
  - `redo log` 仅有完整 `prepare` → 检查 `binlog` 中对应事务是否完整（`statement` 以 `COMMIT` 结尾，`row` 以 `XID event` 结尾，5.6.2+ 带 `binlog-checksum` 校验）：**`binlog` 完整则提交，不完整则回滚**。
- **双 1 持久化配置**：`innodb_flush_log_at_trx_commit = 1` + `sync_binlog = 1`（每次提交持久化 `redo log` 与 `binlog`）。

---

## 二、事务隔离与 MVCC 机制（03、08）

### 2.1 四种隔离级别
| 隔离级别 | 解决的问题 | 视图（`read-view`）策略 |
| :--- | :--- | :--- |
| **READ UNCOMMITTED**（读未提交） | — | 无视图，直接读最新值（存在脏读） |
| **READ COMMITTED**（读提交，RC） | 脏读 | **每条 SQL 执行前**创建新视图 |
| **REPEATABLE READ**（可重复读，RR，默认） | 脏读、不可重复读 | **事务启动时**创建一致性视图并复用 |
| **SERIALIZABLE**（串行化） | 脏读、不可重复读、幻读 | 不使用视图，直接加读写锁串行执行 |

### 2.2 MVCC 视图与可见性规则
- **事务启动差异**：`begin`/`start transaction` 并非立即启动，而是执行**第一条操作 InnoDB 表的语句**时才真正启动并生成 RR 视图；`start transaction with consistent snapshot` 则立即启动并创建视图（RC 下二者等价）。
- **多版本实现**：每行更新生成新版本并记录 `row trx_id` 及 `undo log` 指针，旧版本由最新版本沿 `undo log` 链逆向计算得出（不物理存储多份全行数据）。
- **视图组成**：创建时刻系统内所有未提交事务 ID 组成的**活跃事务数组**、**低水位**（数组最小值）、**高水位**（系统已创建最大 `trx_id + 1`）。
- **版本链可见性判定**（除本事务自身更新总是可见外）：
  1. `row trx_id < 低水位` → 视图创建前已提交 → **可见**；
  2. `row trx_id >= 高水位` → 视图创建后才开启 → **不可见**；
  3. 介于低、高水位之间：若在活跃数组中 → 视图创建时未提交 → **不可见**；不在数组中 → 已提交 → **可见**。
  *(等效逻辑：未提交不可见；视图创建后才提交的不可见；视图创建前已提交的可见。)*

### 2.3 一致性读 vs 当前读
- **一致性读（快照读）**：普通 `SELECT`，按上述视图规则读取历史版本。
- **当前读**：所有更新（`INSERT`、`UPDATE`、`DELETE`）以及 `SELECT ... LOCK IN SHARE MODE`（S 锁）、`SELECT ... FOR UPDATE`（X 锁）。**必须读取最新已提交版本**；若最新记录被其他未提交事务加了写锁，需等待锁释放。
- **表结构（DDL）**：无 `row trx_id`，不遵循 RR 快照视图，按当前读可见性生效。

### 2.4 长事务危害与启动规范
- **危害**：长事务持有的老视图会阻止其创建后的所有 `undo log` 回收，导致回滚段膨胀（MySQL 5.5 及以前 `undo` 在 `ibdata` 中永不缩容；5.6+ 建议 `innodb_undo_tablespaces >= 2`）；同时长期占锁。
- **规范**：保持 `set autocommit=1` 并显式开启事务；高频连续短事务可用 `commit work and chain` 减少一次网络往返；通过 `information_schema.innodb_trx` 监控超时长事务，测试环境可用 `SET MAX_EXECUTION_TIME` 或 `pt-kill` 控制。

---

## 三、索引原理、选型与优化（04–05、09–11、18）

### 3.1 索引模型与主键设计
- **数据结构对比**：哈希表仅支持等值，不支持范围；有序数组插入删除 $O(N)$；InnoDB 采用以数据页（16KB）为单位的 **N 叉 B+ 树**（`int` 主键单层扇出 $N \approx 1200$，树高 4 层可存十亿级行，常驻根页下最多 3 次磁盘 IO）。
- **聚簇索引（主键）vs 二级索引**：主键索引叶子节点存**整行数据**；二级索引叶子节点存**主键值**。通过二级索引查询整行需再查一次主键索引，称为**回表**。
- **为何推荐自增主键**（`NOT NULL PRIMARY KEY AUTO_INCREMENT`）：
  1. 递增追加写，避免非递增插入满页引发的**页分裂**（性能下降且页利用率降至 ~50%）；
  2. 主键越小（`int` 4B / `bigint` 8B），所有二级索引叶子节点占用空间越小。
  - **例外**：典型 KV 单索引业务（唯一索引即为业务主键）可直接用业务字段作主键，省去一次回表。
- **联合主键与二级索引冗余**：若主键为 `(a, b)`，二级索引 `(c)` 叶子实际存 `(c, a, b)` 按 `c,a,b` 排序——故 `(c, a)` 与 `(c)` 完全等价冗余，而 `(c, b)` 按 `c,b,a` 排序可用于 `where c=? order by b`。

### 3.2 覆盖索引、最左前缀与索引下推
- **覆盖索引**：索引本身包含查询所需全部字段（如 `select id from T where k between 3 and 5`），无需回表。
- **最左前缀原则**：联合索引 `(a, b)` 可用于查 `(a, b)`、查 `(a)` 或字符串左前缀 `like '张%'`。字段排列原则：能复用则把单列查询列放前面（有 `(a,b)` 无需再建 `(a)`）；必须同时建单列索引时，**联合索引把占空间更大的列放前面**，单独给小列建索引（如 `(name, age)` + `(age)`）。
- **索引下推（ICP，MySQL 5.6+）**：遍历联合索引 `(name, age)` 时，InnoDB 内部直接过滤包含在索引中的条件（如 `age=10`），不满足则跳过回表，显著减少回表次数。

### 3.3 普通索引 vs 唯一索引与 `change buffer`
- **查询差异**：普通索引查到第一条匹配后多向后比对一条直至不匹配；因数据按 16KB 页加载到内存，CPU 指针对比开销微乎其微。
- **更新差异与 `change buffer`**：
  - 当待更新数据页不在 Buffer Pool 时，**普通索引**可将更新操作记入 `change buffer`（位于内存、持久化于 `ibdata1`、写入 `redo log` 保 crash-safe）并直接返回，**省去随机磁盘读 IO**；待后续读该页、后台线程定期或正常 `shutdown` 时再读盘 `merge`。由 `innodb_change_buffer_max_size`（如 `50` 表示占 BP 50%）控制。
  - **唯一索引无法使用 `change buffer`**：每次写入必须判重，必须先读盘加载数据页到内存。
  - **对比 `redo log`**：`redo log` 省去**随机磁盘写 IO**；`change buffer` 省去**随机磁盘读 IO**。
- **选型建议**：需要数据库强制兜底唯一性则用唯一索引；业务可保证唯一且**写多读少**（账单、日志、归档、机械盘）优先用普通索引；若写后立刻读取该记录，触发即时 `merge` 反而更慢，应关闭 `change buffer`。

### 3.4 选错索引原因与处理（10）
- **原因**：优化器综合评估扫描行数 `rows`、临时表、排序及**回表代价**。InnoDB 通过采样 `N` 个页估算基数 `Cardinality`（`innodb_stats_persistent=on` 时 `N=20`，修改超 `1/10` 重算；`off` 时 `N=8, M=16`）。在“删除并重新插入 + 存在未提交长事务”场景下，旧删标记版本与新版本并存导致二级索引统计行数虚高，优化器误判回表代价过大而改走全表/主键扫描。
- **解决**：① `analyze table t;` 重算统计信息；② `force index(idx)` 强制索引（不灵活）；③ 改写 SQL 消除误导条件（如将 `order by b limit 1` 改为 `order by b, a limit 1`）；④ 增删冗余/误导索引。

### 3.5 字符串加索引 4 种方案（11）
1. **前缀索引** `index(email(6))`：通过比较 `count(distinct left(email, N))` 与总去重数确定最小可接受损失长度。**缺点**：即使前缀长度覆盖全长，**仍无法使用覆盖索引**，必然回表；前缀区分度低时扫描行数多。
2. **倒序存储 + 前缀索引**：存 `reverse(str)`，查询 `where col = reverse('...')`（不支持范围查询）。
3. **新增哈希字段 + 索引**：增加 `crc32` 整数列并建索引，查询 `where col_crc = crc32('...') and col = '...'`（必须加原列等值防哈希碰撞；固定 4 字节区分度极高，不支持范围查询）。
4. **定长特征提取转整数**：如学号固定省校前缀，仅截取出年份+序号存 `int`/`bigint`。

### 3.6 索引失效三大典型场景（18）
对索引字段做函数或运算会破坏 B+ 树有序性，导致**放弃树搜索**（退化为全索引扫描 `Using index` 或全表扫描）：
1. **显式函数/表达式计算**：`where month(t_modified)=7`、`where id + 1 = 10000`（优化器不会自动代数移项！）。必须改写为纯索引列范围或等值（`where id = 10000 - 1`）。
2. **隐式类型转换**：MySQL 字符串与数字比较时将**字符串转为数字**（`"10" > 9` 为 `1`）。因此 `varchar` 索引列与数字比较（`where tradeid=110717`）等价于 `CAST(tradeid AS signed int)=110717` 导致**索引失效**；反之 `int` 索引列与字符串比较（`where id="83126"`）仅转换常量，索引有效。
3. **隐式字符集转换**：`utf8` 与 `utf8mb4` 关联时，MySQL 将小字符集 `utf8` 转为超集 `utf8mb4`。若被驱动表索引列为 `utf8`，驱动值为 `utf8mb4`，则对被驱动表列执行 `CONVERT(col USING utf8mb4)` 导致**全表扫描**。解法：统一表字符集为 `utf8mb4`，或显式将驱动侧值转为被驱动表字符集（`CONVERT(l.tradeid USING utf8)`）。

---

## 四、锁体系全解与加锁规则（06–07、19–21、30、40）

### 4.1 全局锁、表锁与 MDL 锁（06）
- **全局锁 FTWRL**（`Flush tables with read lock`）：全库只读，用于不支持事务引擎的逻辑备份；客户端异常断开自动释放（优于 `set global readonly=true`）。全 InnoDB 库应使用 `mysqldump --single-transaction`（基于一致性快照不锁表）。
- **表锁**（`lock tables ... read/write`）：同时限制自身线程操作，InnoDB 表不要使用。
- **元数据锁（MDL，MySQL 5.5+ 自动加）**：DML/查询加 MDL 读锁，DDL 加 MDL 写锁，**事务提交才释放**。
  - **全表堵死链条**：长事务未提交（持 MDL 读锁）→ `ALTER TABLE` 申请 MDL 写锁阻塞 → **后续所有普通查询/DML 全部排队堵在 MDL 写锁之后**，打满连接线程。
  - **安全 DDL**：先查 `information_schema.innodb_trx` 避开/干掉长事务；对热点表使用带超时控制的 `ALTER TABLE ... NOWAIT / WAIT N`（MariaDB / AliSQL 特性）。

### 4.2 两阶段锁协议与死锁处理（07）
- **2PL**：InnoDB 行锁在需要时加上，直到**事务结束（`commit`/`rollback`）才统一释放**。**准则**：把最可能造成锁冲突的更新语句尽量放在事务最后。
- **死锁应对**：
  1. `innodb_lock_wait_timeout`（默认 50s）超时等待；
  2. `innodb_deadlock_detect`（默认 `on`）主动检测环路并回滚代价最小事务。
- **热点行更新 CPU 100% 根因与破局**：每个等锁线程都要遍历等待图做 $O(n)$ 检测，高并发更新同一行触发 $O(n^2)$ 级 CPU 消耗（如千并发百万次检测）。解法：① 确保无死锁时临时关闭死锁检测；② **服务端排队限流**（客户端限流在多机部署下无效）；③ **业务行拆分**（将 1 行余额拆为 10 行随机扣减求和）。

### 4.3 幻读、间隙锁与 `next-key lock`（20）
- **幻读定义**：在可重复读（RR）下，**当前读**（如 `for update`）在同一事务内两次范围查询，后一次看到了其他事务**新插入并提交的行**（修改既有行不叫幻读；快照读也不会产生幻读）。
- **危害**：破坏加锁语义（`where d=5 for update` 锁不住新插的 `d=5`），且在 `statement` binlog 下引发主从数据不一致。
- **间隙锁（Gap Lock）**：锁定索引记录之间的开区间 `(a, b)`，仅与**“往该间隙插入一条记录”**冲突，**间隙锁之间互不冲突**。两个事务先同取间隙锁再各自 `insert` 会立即死锁。
- **`next-key lock`**：行锁 + 间隙锁，前开后闭区间 `(a, b]`。InnoDB 为每个索引加虚拟最大值 `supremum`，尾区间 `(max, supremum]` 同理。
- **规避间隙锁**：改隔离级别为 `read-committed`（仅外键检查保留间隙锁，**语句结束后立即释放未匹配行的行锁**）+ 强制 `binlog_format=row`。

### 4.4 RR 隔离级别加锁 5 条核心规则（21、30）
> 适用于 RR 级别（5.x <= 5.7.24, 8.0 <= 8.0.13）。加锁是执行过程中**动态逐个加上**的，锁加在**索引**上：
1. **原则 1**：加锁的基本单位是 `next-key lock` `(a, b]`。
2. **原则 2**：查找过程中**访问到的对象**才会加锁（覆盖索引仅 `lock in share mode` 时不回表则不锁主键；`for update` 会顺带锁主键）。
3. **优化 1**：索引上的**等值查询**，给**唯一索引**加锁且记录存在时，`next-key lock` 退化为**行锁**。
4. **优化 2**：索引上的**等值查询**，向右遍历时且**最后一个值不满足等值条件**，其 `next-key lock` 退化为**间隙锁**。
5. **一个 Bug**：**唯一索引上的范围查询**会访问到**不满足条件的第一个值为止**（多锁一个区间，如 `id>10 and id<=15` 会锁到 `(15, 20]`）。

- **关键动态推演与优化**：
  - **复合与降序扫描**：如 `id>9 and id<12 order by id desc` 先用等值定位找 `12`（落入 `(10,15)` 触发优化 2 加间隙锁），再向左扫加 `(5,10]`、`(0,5]`；`IN (5,20,10)` 与 `IN (5,20,10) ORDER BY c DESC` 并发执行因加锁顺序相反引发死锁。
  - **删除加 `LIMIT`**：`delete from t where c=10 limit 2` 找到 2 条后即停止，不再向后多锁最后一个间隙。
  - **间隙的动态定义**：“间隙”由**间隙右边的那条记录**定义。若 `delete` 删掉 `id=10`，原本相邻的 `(5,10)` 与 `(10,15)` 会动态合并成 `(5,15)`；`update` 索引列等价于“插入新值 + 删除旧值”，第一步插入受间隙锁阻塞。
  - **`next-key lock` 分两步加锁**：先加间隙锁，再加行锁。线程 B 被行锁阻塞时已持有间隙锁，若线程 A 此时往该间隙插入则触发死锁。
  - **排查命令**：`show engine innodb status` 查看 `LATEST DETECTED DEADLOCK` 与 `TRANSACTIONS` 中的 `lock_mode X locks gap before rec insert intention waiting`。

### 4.5 `INSERT` 语句的锁（40）
1. **`INSERT ... SELECT`**：在 RR + `statement` binlog 下，对源表扫描到的**所有记录和间隙加共享 `next-key lock`**；`INSERT INTO t SELECT ... FROM t`（自插入）还会强制生成内部临时表并对 `t` 全表加间隙锁堵死所有写入（5.6/5.7 应拆为显式 `MEMORY` 临时表过渡）。
2. **唯一键冲突加锁**：不论主键还是普通唯一索引（官方 8.0 文档有误，Bug #93806），插入报唯一键冲突时，RR 下都会在冲突索引上加**共享 `next-key lock`（S 锁）**。多并发同时插同一冲突值并在首事务 `ROLLBACK` 后同时升级 X 锁，极易死锁——**遇唯一键冲突报错应尽快提交或回滚**。
3. **`INSERT INTO ... ON DUPLICATE KEY UPDATE`**：冲突时直接加**排他 `next-key lock`（X 锁）**；多个唯一键同时冲突时按索引定义顺序（主键优先）仅修改第一个冲突行；修改已有行时 `affected rows` 返回 `2`。

---

## 五、SQL 执行原理与性能调优（12–14、16–17、19、33–38、44）

### 5.1 系统“抖”一下：脏页 `flush` 机制（12）
- **4 个刷脏时机**：① `redo log` 写满（**所有更新停止，TPS 跌 0，必须避免**）；② Buffer Pool 满淘汰脏页；③ 系统空闲后台刷；④ 正常关闭。
- **关键参数与速率控制**：
  - `innodb_io_capacity`：设为通过 `fio` 测得的磁盘随机读写 IOPS（设太低如默认/300 在 SSD 上会导致脏页积压卡顿）。
  - `innodb_max_dirty_pages_pct`（默认 `75%`）：通过 `Innodb_buffer_pool_pages_dirty / Innodb_buffer_pool_pages_total` 监控，保持远低于 75%。
  - 刷脏速度 = `innodb_io_capacity * max(F1(脏页比例), F2(redo log 落后进度))`%。
  - `innodb_flush_neighbors`：`1` 顺带刷相邻脏页（机械盘），**SSD 建议设为 `0`**（MySQL 8.0 默认已改为 `0`）。
  - `redo log` 过小（如 100M）会频繁撞到 `checkpoint`，导致高配置机器磁盘占用率低却间歇性卡顿；多 TB 盘建议配 `4 × 1GB`。

### 5.2 表空间收缩与 Online DDL（13）
- **开启独立表空间**：`innodb_file_per_table = ON`（5.6.6+ 默认），`drop table` 立即删 `.ibd` 回收空间。
- **删数据文件不缩小的原因**：`delete` 仅把记录位置或整页标记为“可复用”（形成**空洞**）；随机插入页分裂、更新索引列同样产生空洞。
- **重建表回收空洞**：`alter table t engine=InnoDB;`。
  - **5.6+ Online DDL（`ALGORITHM=inplace`）**：先拿 MDL 写锁并**立即降级为 MDL 读锁** → 拷贝数据至 InnoDB 内部 `tmp_file` → 将并发 DML 记入 `row log` → 重放 `row log` 并替换表文件。**注意**：大表重建仍需足够磁盘剩余空间，生产超大表推荐 `gh-ost`。
  - **命令辨析**：`analyze table` 仅重算索引统计（MDL 读锁）；`optimize table` = `recreate` + `analyze`；`inplace` 不一定总是 Online（如 `FULLTEXT`/`SPATIAL` 索引至 8.0 仍会阻塞 DML）。

### 5.3 `count(*)` 原理与优化（14）
- **MyISAM vs InnoDB**：MyISAM 存总行数（无 `WHERE` 极快）；InnoDB 因 **MVCC 多版本并发可见性不同**，只能逐行扫描计数。优化器会自动挑选**最小的那棵索引树**扫描。`show table status` 的 `TABLE_ROWS` 误差达 40%~50%。
- **业务计数方案**：Redis 计数丢数据且无法保证与 MySQL 快照逻辑一致；**用 MySQL 计数表**在同事务内“**先插数据行，再更新计数行**”可兼顾强一致性与最小化热点行锁时间。
- **性能排序**：`count(普通字段) < count(主键 id) < count(1) ≈ count(*)`。**首选 `count(*)`**（MySQL 专门优化，不取值直接按行计数）。

### 5.4 `order by` 排序与随机消息（16–17）
- **`order by` 三种执行路径**：
  1. **全字段排序**：将查询所需字段全部放入 `sort_buffer`（超 `sort_buffer_size` 则落盘外部归并排序，看 `OPTIMIZER_TRACE` 中 `number_of_tmp_files`），排完直接返回。
  2. **`rowid` 排序**：当返回字段总长超过 `max_length_for_sort_data` 时触发，`sort_buffer` 只存 `<排序键, rowid>`，排完按 `limit` 条数二次回表取全字段。
  3. **联合覆盖索引免排序**：建立 `(city, name, age)` 索引，利用索引天然有序性 + 覆盖索引，既无 `Using filesort` 也无回表。
  - **坑点**：`where city in ('杭州','苏州') order by name` 仍需 `filesort`（跨值不全局有序），应拆分单值查 `limit` 后在应用层归并。
- **无意义更新的优化**：`binlog_format=row`（5.6+ 且 `FULL`/`NOBLOB` 无大字段）时，Server 层对比新旧值未变会直接跳过引擎更新；`statement` 下无法判定旧值仍会加锁执行。
- **随机消息 `order by rand() limit 3`**：
  - 需建 `MEMORY` 临时表（超 `tmp_table_size=16M` 转磁盘表）+ `rowid` 排序；5.6+ 在堆能塞进 `sort_buffer_size` 时触发**优先队列（大顶堆）优化**免除全排序，但仍需全扫基表并写入临时表（万行表扫 `20003` 行）。
  - **推荐做法**：应用层先取总行数 `C`，本地随机算出偏移 `Y1, Y2, Y3`，再发 `select * from t limit Yk, 1`（或主键连续时按 `max/min(id)` 随机取值）。

### 5.5 单行查询变慢排查（19）
1. **被阻塞等待**（`show processlist`）：
   - `Waiting for table metadata lock` / `Waiting for table flush`：查 `sys.schema_table_lock_waits`（需开 `performance_schema`）找出阻塞源并 `KILL`。
   - 等行锁：查 `sys.innodb_lock_waits` 找到持有锁的线程，必须用 **`KILL <id>`** 断连回滚事务（**`KILL QUERY` 仅停语句不放锁！**）。
2. **执行本身慢**：
   - 未建索引扫描大量行（小表测试不易暴露，设为 `long_query_time=0` 查慢日志 `Rows_examined`）。
   - **长 `undo log` 链**：并发长事务压住老视图 + 另一事务高频更新同一行（如 100 万次），普通快照读需沿 `undo` 链逆向计算 100 万次（极慢），而 `lock in share mode` 当前读直接读最新版本（极快）。

### 5.6 大查询内存机制与 Buffer Pool LRU（33）
- **Server 层不会打爆内存（边读边发）**：结果集写入 16KB `net_buffer` → 满即走内核 `socket send buffer` 发送并清空；若网络满则阻塞等待。
  - `Sending to client`：真正卡在等客户端收包（如用了 `--quick` / `mysql_use_result` 且客户端处理极慢，**会拖成隐形长事务**）。
  - `Sending data`：仅表示已进入执行器阶段（包含扫表、等锁等），并非正在发包。
- **InnoDB Buffer Pool 改进型 LRU（5:3 分区）**：
  - 前 5/8 为 `young` 区，后 3/8 为 `old` 区，新读入的数据页插在 **`old` 区头部（`LRU_old`）**。
  - `old` 区页面被再次访问时，只有**在链表停留时间超过 `innodb_old_blocks_time`（默认 `1000` ms）**才晋升至 `young` 区头部。
  - **效果**：大表全扫时同一页内多行访问间隔 `<1s`，页面始终停留在 `old` 区并迅速淘汰，**不会污染 `young` 区热数据**（`Buffer pool hit rate` 维持 `>= 99%`）。

### 5.7 `JOIN` 算法与优化（34–35、44）
- **Index Nested-Loop Join（NLJ）**：被驱动表连接字段**有索引**。驱动表全扫（$N$）+ 每次被驱动表走树搜索（$2 \log_2 M$），总扫描近似 $N + N \cdot 2\log_2 M$。**可用 `JOIN`，且小表作驱动表**（精确定义：**各表经 `WHERE` 过滤后，参与 `JOIN` 的所需字段总数据量更小的表**）。
- **Block Nested-Loop Join（BNL）**：被驱动表**无索引**。驱动表装入 `join_buffer`（默认 256KB），全扫被驱动表在内存无序对比；放不下则分 $K$ 段重扫被驱动表 $K$ 次。
  - **BNL 三大危害**：多次全扫耗尽 IO；$M \times N$ 次内存对比打满 CPU；**持续污染 Buffer Pool**（冷表跨过 1 秒门槛挤进 `young` 区，或反复冲刷 `old` 区导致正常热页无法晋升）。
- **MRR（Multi-Range Read）与 BKA（Batched Key Access）**：
  - **MRR**：二级索引范围查出的 `id` 先放 `read_rnd_buffer` 排序，再**顺序回表读盘**（需 `set optimizer_switch="mrr_cost_based=off";` 强制开启）。
  - **BKA（5.6+）**：用 `join_buffer` 批量攒驱动键值传给被驱动表触发 MRR 顺序回表（`set optimizer_switch='mrr=on,mrr_cost_based=off,batched_key_access=on';`）。
- **BNL 转 BKA 手段**：① 被驱动表加索引；② 低频查询先过滤写入带索引的**临时表**再关联；③ **应用层模拟 Hash Join**（小表拉入内存哈希表，扫大表 O(1) 匹配）。
- **`LEFT JOIN` 坑点（44）**：把右（被驱动）表过滤条件写在 `WHERE` 中会滤掉 `NULL` 补齐行，导致优化器将其等价改写为 `INNER JOIN`（甚至反转驱动表）——**右表条件必须写在 `ON` 中**。

### 5.8 临时表与 Memory 引擎（36–38、44）
- **用户临时表（36）**：`CREATE TEMPORARY TABLE`。仅当前连接可见，会话结束自动销毁。**可与普通表重名**（优先访问临时表）：磁盘 `.frm` 放 `@@tmpdir` 下按 `#sql{pid}_{thread_id}_{seq}` 命名，内存 `table_def_key` 拼接了 `server_id + thread_id`。`RENAME TABLE` 无法改临时表名，须用 `ALTER TABLE ... RENAME`。跨分片汇总排序时可用于归并各分片结果。
- **内部临时表（37）**：
  - `UNION` 建主键去重临时表，`UNION ALL` 直接流式返回无需临时表。
  - 无索引 `GROUP BY` 先用 `MEMORY` 临时表累加再排序（超 `tmp_table_size=16M` 转磁盘 `InnoDB` 临时表）。**优化**：不需要排序加 `ORDER BY NULL`；对分组列/生成列（`GENERATED ALWAYS AS`）建索引彻底消除临时表与排序；数据极大时加 `SQL_BIG_RESULT` 提示直接走 `sort_buffer` 归并聚合。
  - 无索引无聚合函数时，`SELECT DISTINCT a` 与 `SELECT a GROUP BY a ORDER BY NULL` 语义与执行完全一致。
- **Memory 引擎（38）**：堆组织表（按写入顺序存，删除空位复用），**默认索引为 `HASH`（不支持范围查询，需显式 `USING BTREE`）**，不支持 `BLOB`/`TEXT`，`VARCHAR` 按定长 `CHAR` 存。**生产普通表禁用**：仅支持表锁，且重启清空数据（双 M 架构重启自动往 binlog 写 `DELETE FROM tbl` 会把主库数据也清空！）。**唯一适用场景**：行数可控的**用户临时表**（`ENGINE=MEMORY`）。

---

## 六、高可用、主从同步与持久化进阶（22–29）

### 6.1 紧急性能救火与风险（22）
- **连接数打满**：优先 `kill` `innodb_trx` 外**非事务的 `Sleep` 连接**（注意应用重连报错处理）；`--skip-grant-tables` 跳过鉴权（8.0 自动开启 `--skip-networking` 仅限本地）。
- **缺索引/坏 SQL**：紧急时在备库 `set sql_log_bin=off` 加索引再主备切换（会丢 DDL 的 binlog，常规应走 Online DDL 或 `gh-ost`）；MySQL 5.7 可用 `query_rewrite` 插件动态改写 SQL 或注入 `force index`。

### 6.2 日志刷盘与组提交优化（23）
- **`binlog` 路径**：线程独占 `binlog cache`（`binlog_cache_size`，超限写临时盘）→ 提交时 `write`（OS page cache）→ `fsync`（落盘）。`sync_binlog` 控制每 `N` 次提交 `fsync` 一次。
- **`redo log` 路径**：全局共享 `redo log buffer` → `write` → `fsync`。`innodb_flush_log_at_trx_commit`：`0` 留 buffer（进程挂即丢）；`1` **`prepare` 阶段 `fsync`**，`commit` 只 `write`；`2` 提交时仅 `write`（机器断电才丢）。后台每秒线程、buffer 过半、其他 `=1` 事务顺带提交均会触发刷盘。
- **两阶段组提交（Group Commit）顺序**：`redo prepare` 写 buffer → `binlog write` → **`redo fsync`（组提交）** → **`binlog fsync`（组提交）** → `redo commit`（只写 cache）。把 `redo fsync` 延后到 `binlog write` 之后，拉大了 `binlog` 组提交窗口。
- **IO 瓶颈调优顺序**：① 调大 `binlog_group_commit_sync_delay`（微秒）和 `binlog_group_commit_sync_no_delay_count`（或条件，注意 `delay=0` 时 `count` 不生效），增延迟但不丢数据；② `sync_binlog = 100~1000`；③ `innodb_flush_log_at_trx_commit = 2`。

### 6.3 主备一致、`binlog_format` 与高可用切换（24–25）
- **备库 `readonly`**：建议开启（超级权限同步线程不受限），防误写、防切主双写脑裂。
- **`binlog` 三种格式**：
  - `statement`：记原 SQL + 上下文（`SET TIMESTAMP` 等），`delete ... limit 1` 选不同索引易致主从不一致；
  - `row`：记行前后完整镜像（`binlog_row_image=FULL`）与 `Xid`，体积大但**绝对一致且支持行级 Flashback 恢复**；
  - `mixed`：安全用 `statement`，不安全（如 `limit`、`UUID()` 等）自动切 `row`。
- **双 M 防死循环**：备库重放时保留主库 `server_id`，丢弃与自身 `server_id` 相同的事件。三节点或动态改 `server_id` 死循环时用 `CHANGE MASTER TO IGNORE_SERVER_IDS=(...)` 修复。
- **主备延迟（`seconds_behind_master`）**：主要源于备库性能差/读压力大、大事务（大 `delete` / 大 DDL）、或并行度不足。若延迟图像呈 **45° 直线上升**，说明备库完全卡死（如被备库只读大查询堵住 MDL 锁）。
- **切换策略**：**可靠性优先**（等延迟 `<5s` → 主库 `readonly` → 等延迟清零 → 切备库可写，短时不可写但不丢数据）vs **可用性优先**（直接切备库可写，零停机但必丢/乱数据，`row` 模式报错停止、`mixed`/`statement` 静默不一致）。

### 6.4 备库并行复制演进（26）
所有并行复制均须保证：**同一行修改必须进同一 `worker`**，**同一事务不能拆分**。
1. **MySQL 5.6（`DATABASE`）**：按库名哈希，单库热点无效。
2. **MariaDB**：同一次 redo 组提交的 `commit_id` 事务并行，但**组间有屏障**（受组内慢事务拖累）。
3. **MySQL 5.7（`slave-parallel-type=LOGICAL_CLOCK`）**：**只要同时处于 `redo prepare` 阶段（已过锁冲突检查）及 `prepare` 与 `commit` 之间的事务即可并行**，消除组间屏障；增大组提交等待反而提升备库并行度。
4. **MySQL 5.7.22（`binlog-transaction-dependency-tracking=WRITESET` / `WRITESET_SESSION`）**：主库按主键及所有唯一键计算行哈希写入 `binlog`，**只要修改行集合无交集即可并行**（不依赖主库并行度、支持 `statement`、省备库解析开销；**无主键或有外键时退化为单线程**）。`WRITESET_SESSION` 额外保证主库同线程事务在备库顺序执行。

### 6.5 一主多从切换与 GTID（27）
- **基于位点切换**：在备库 A' 补全 relay log 后取故障时刻 `T` 对应的近似位点（宁前勿后），可能遇 `1062`/`1032` 冲突需谨慎跳过（`sql_slave_skip_counter=1`）。
- **GTID（`server_uuid:gno`）切换**：开启 `gtid_mode=on`、`enforce_gtid_consistency=on`，切换只需 `CHANGE MASTER TO ..., master_auto_position=1;`。从库发自身 `set_b` 给新主 A'，A' 自动计算差集发送（若所需日志已删则报错拒绝）。
- **跳过冲突事务标准写法（空事务法）**：`set gtid_next='<gtid>'; begin; commit; set gtid_next=automatic; start slave;`。双 M 加索引也可借备库先执行 + 主库注入空事务实现无锁且保留 binlog。

### 6.6 读写分离与健康检查（28–29）
- **解决读写分离“过期读” 6 方案**：① 强制走主库（按业务分级，最常用）；② `sleep`（不精准）；③ 查延迟为 0 / 位点相等 / GTID 集合相等（仍漏掉主库已提交但未发到从库的窗口）；④ 配合 **semi-sync**（一主一备有效，一主多从或写高峰期失效）；⑤ **等主库位点**：主库取位点，从库执行 `select master_pos_wait(file, pos, timeout)`（推荐）；⑥ **等 GTID（5.7.6+）**：配 `session_track_gtids=OWN_GTID` 从返回包拿 GTID，从库执行 `select wait_for_executed_gtid_set(gtid, timeout)`（推荐，超时降级查主库并配合限流）。
- **实例健康检查演进**：`select 1` 无法发现 InnoDB 并发打满（`innodb_thread_concurrency` 建议 `64~128`，等锁线程会释放名额计数防止死锁）→ 查表无法发现 `binlog` 盘满写阻塞 → **更新 `mysql.health_check`（以 `@@server_id` 作主键防双 M 冲突）+ 配合 `performance_schema` 监控 `innodb_log_file`/`binlog` IO 等待耗时阈值**（如 `MAX_TIMER_WAIT > 200ms` 后 `truncate` 清零统计）。

---

## 七、运维实战、表结构演进与底层边界（31–32、39、41–43、45）

### 7.1 误删数据恢复与预防（31）
- **误删行（`delete`）**：前提开启 `binlog_format=row` + `binlog_row_image=FULL`，用 **Flashback** 反转 binlog（`Write <-> Delete`，`Update` 前后镜像对调，**多事务逆序重放**）；**严禁直连主库执行**，先恢复到临时库校验再导回。事前开启 `sql_safe_updates=on`。
- **误删库/表（`drop` / `truncate`）**：最近全备恢复临时库 + 重放后续 binlog 并跳过误操作（位置截断或空事务注入 GTID）；提速可将临时库挂为备库的从库并用 `change replication filter replicate_do_table` 仅同步目标表利用并行复制。**架构防线**：配置**延迟复制备库**（`MASTER_DELAY=3600`）+ 权限账号分离 + 删表前改后缀观察。

### 7.2 `kill` 不掉的语句与客户端参数（32）
- `kill query` / `kill connection` 只是设线程状态位并唤醒可唤醒等待，**线程必须执行到代码检查埋点才会终止**。
- **`kill` 不掉（`Killed` 状态卡住）两大原因**：① 卡在埋点前（如等进 `innodb_thread_concurrency` 休眠循环不查状态、或等慢磁盘 IO 系统调用）；② 终止回滚本身极慢（超大事务 undo 清理 / 慢 IO 清理临时文件）。**此时强行重启依然要继续回滚，应通过降 IO 压力或临时调大并发参数加速结束**。
- **`Ctrl+C`**：客户端另开新连接发 `kill query`。
- **客户端参数**：`-A` 跳过建连自动补全库表实现秒连；`-q`（`--quick`）改用 `mysql_use_result` 流式逐行读取不缓存内存，但客户端处理慢会反拖慢服务端。

### 7.3 自增主键不连续与锁模式（39）
- **持久化**：5.7 及以前 InnoDB 自增计数器仅存内存，重启重算 `MAX(id)+1`；**8.0+ 写入 `redo log` 持久化**。
- **不连续三大原因**：① 唯一键冲突（自增先于插入，冲突不回退）；② 事务回滚（不回退以防并发插重号）；③ 批量插入（`INSERT ... SELECT`/`LOAD DATA` 按 `1, 2, 4, 8...` 倍增预申请，末批富余丢弃）。
- **`innodb_autoinc_lock_mode`**：`0` 语句结束放锁；`1`（旧默认）已知行数插入立即放锁，批量插入语句结束放锁；`2`（推荐）所有插入立即放锁并发最高，**需搭配 `binlog_format=row`**。

### 7.4 最快复制表 4 种方案（41）
1. `INSERT ... SELECT`：小范围可，大范围易锁源表。
2. `mysqldump --single-transaction --add-locks=0 --no-create-info --set-gtid-purged=OFF` + `source`：支持 `WHERE` 过滤，跨库通用但导入偏慢。
3. `SELECT ... INTO OUTFILE` + `LOAD DATA [LOCAL] INFILE`（单表 CSV，受 `secure_file_priv` 控制；`LOCAL` 通过客户端传文件，binlog 会内嵌文件内容确保从库无需本地目录权限）：导出导入极快。
4. **物理拷贝（InnoDB 5.6+ 传输表空间，全表最快）**：
   ```sql
   CREATE TABLE r LIKE t;
   ALTER TABLE r DISCARD TABLESPACE;
   FLUSH TABLE t FOR EXPORT; -- 源表只读并生成 t.cfg
   -- OS 层 cp t.cfg r.cfg; cp t.ibd r.ibd（注意属主权限）
   UNLOCK TABLES;
   ALTER TABLE r IMPORT TABLESPACE;
   ```

### 7.5 权限刷新时机（42）
- **`GRANT` / `REVOKE` 后不需要执行 `FLUSH PRIVILEGES`**：它们同时修改 `mysql.*` 系统表和内存数组（全局 `acl_users`、库级 `acl_dbs`、表/列 `column_priv_hash`）。全局权限仅对**新连接**生效；库/表/列权限对**现有连接即时生效**（已 `USE db` 缓存的当前库权限除外）。
- **`FLUSH PRIVILEGES` 唯一正确用途**：直接用 `INSERT`/`UPDATE`/`DELETE` 修改了 `mysql.*` 系统权限表导致内存与磁盘不一致时，用于清空并重载内存权限结构。

### 7.6 分区表取舍（43–44）
- **特征**：Server 层视作 1 张表（共用 MDL 锁，操作某分区会阻塞其他分区查询/DDL），Engine 层视作 N 张表（InnoDB 间隙锁隔离在各分区内）。5.7.9+ InnoDB 支持原生分区（解决通用分区首次打开全部分区受 `open_files_limit` 限制的问题）。
- **主键约束**：分区表主键必须包含所有分区键，建议 `PRIMARY KEY (ftime, id), KEY id(id)`（满足自增列必须是某索引首列且便于按时间范围查询）。
- **结论**：最大价值是按时间分区后 `DROP PARTITION` 秒删历史冷数据；控制单分区千万级、避免预建海量空分区即可，无需过度神话。

### 7.7 5 种自增 ID 用尽的后果（45）
| ID 类型 | 上限与重置规则 | 耗尽后果 |
| :--- | :--- | :--- |
| **表自增主键** | 类型上限（`INT UNSIGNED` $2^{32}-1$） | 停滞并报主键冲突（影响可用性；大表提前用 `BIGINT UNSIGNED`） |
| **InnoDB 隐藏 `row_id`** | 6 字节（$2^{48}-1$），无主键表共用 | **回绕归零并静默覆盖旧行数据（丢数据！）**，InnoDB 表必须显式建主键 |
| **`Xid`** | 内存 `global_query_id`（$2^{64}-1$），重启归零并切 binlog | 单 binlog 内唯一，理论不会耗尽 |
| **InnoDB `trx_id`** | 持久 `max_trx_id`（只读事务实际为 0，写/加锁才分配） | 达 $2^{48}-1$ 归零触发 RR 下**永久性脏读 Bug** |
| **`thread_id`** | 4 字节（$2^{32}-1$），回绕时循环避开当前活跃连接 ID | 无冲突，安全复用 |

### 7.8 生产踩坑经验补充（直播回顾）
- **冷启动预热刷脏抢 IO**：刚恢复且无业务流量时只跑全表扫描预热，MySQL 误判系统空闲而全速刷脏打满 IO，使 2 分钟预热拖成 30 分钟；定时发送轻量查询（如每 0.5s）可抑制激进刷脏。
- **SSD/Flashcache 分桶刷脏降级**：Flashcache 按分桶（如 20MB）计 80% 脏页阈值，将顺序写日志（redo/binlog）与随机写数据盘隔离（日志放机械盘，数据放 SSD）可避免单桶频触发强刷导致性能倒退。
