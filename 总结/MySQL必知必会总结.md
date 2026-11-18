# 《MySQL 必知必会》核心知识精炼总结

> 基于《MySQL 必知必会》（朱晓峰）全专栏 36 篇内容提炼，剔除故事背景、寒暄与重复表述，保留完整语法、规则、参数、坑点与工程实践。

---

## 目录

- [一、架构、环境与核心概念（01、环境准备）](#一架构环境与核心概念01环境准备)
- [二、数据类型与表约束（02–03）](#二数据类型与表约束0203)
- [三、数据操作与基础查询（04–08）](#三数据操作与基础查询0408)
- [四、内置函数：时间、数学、字符串与条件（09–10）](#四内置函数时间数学字符串与条件0910)
- [五、索引、事务与临时表（11–13）](#五索引事务与临时表1113)
- [六、视图、存储过程、游标与触发器（14–17）](#六视图存储过程游标与触发器1417)
- [七、权限管理、日志体系与备份恢复（18–21）](#七权限管理日志体系与备份恢复1821)
- [八、数据库设计与性能优化（22–26）](#八数据库设计与性能优化2226)
- [九、综合实战：连锁超市系统设计要点（27–28）](#九综合实战连锁超市系统设计要点2728)
- [十、MySQL 8 新特性、空间定位与高频面试题（特别放送）](#十mysql-8-新特性空间定位与高频面试题特别放送)

---

## 一、架构、环境与核心概念（01、环境准备）

### 1.1 系统架构与存储流程
- **层次结构**：数据库服务器 → 数据库（最大存储单元）→ 数据表 → 行（记录）与列（字段）。
- **数据存储 4 步**：创建数据库 → 确认字段类型与约束 → 创建数据表 → 插入数据。
- **自带系统库**：
  - `information_schema`：存储库名、表名、字段名、权限、数据目录等元数据。
  - `performance_schema`：监控 MySQL 各类底层性能指标。
  - `sys`：以易读视图展示性能监控指标。
  - `mysql`：存储运行时系统信息（数据目录、字符集、用户权限、约束检查等）。

### 1.2 核心库表操作与规范
```sql
SHOW DATABASES;
CREATE DATABASE demo;
USE demo;
DROP DATABASE demo;

SHOW TABLES;
CREATE TABLE demo.test (
  id INT PRIMARY KEY AUTO_INCREMENT,
  barcode VARCHAR(20),
  price DECIMAL(10, 2)
);
DESCRIBE demo.test; -- 返回 Field, Type, Null, Key, Default, Extra
```
- **书写规范**：建议显式带库名（`db.table`）；建表最后一个字段后不能加逗号；表名与字段名使用英文；每条 SQL 以分号 `;` 结尾。
- **`NULL` vs 空字符串 `''`**：`NULL` 不等于 `''`。`''` 长度为 `0`，`NULL` 长度为 `NULL`，且 **`NULL` 在 MySQL 中占用存储空间**。

### 1.3 连接、认证与导入编码
- **连接方式与端口**：`TCP/IP` 支持网络访问，默认端口 **`3306`**，X Protocol 端口 **`33060`**；`Named Pipe` / `Shared Memory` 仅限本机，速度更快。
- **认证插件**：MySQL 8 默认 `caching_sha2_password`（基于 SHA-256 + 随机盐，相同密码生成不同哈希）。
- **内存配置模式**：`Development Computer`（最小内存）、`Server Computer`（中等内存，多服务共存）、`Dedicated Computer`（独占整机内存）。
- **CSV 导入**：通过 Workbench 导入 CSV 前需将文件转码保存为 **UTF-8**。

---

## 二、数据类型与表约束（02–03）

### 2.1 数据类型选型
| 类别 | 类型 | 核心特性与选型原则 |
| :--- | :--- | :--- |
| **整数** | `TINYINT`, `SMALLINT`, `MEDIUMINT`, `INT`, `BIGINT` | 优先确保不溢出（如无符号 `SMALLINT` 上限 `65535` 易越界时选 `INT`），再考虑省空间 |
| **浮点数** | `FLOAT` (4B), `DOUBLE` (8B), `REAL` (默认同 `DOUBLE`) | 二进制近似存储，**有精度误差**（如 `0.47+0.44+0.19` 无法精确等于 `1.1`，等值过滤失败）；`SET sql_mode="REAL_AS_FLOAT"` 可改 `REAL` 为单精度；无论有无符号均存符号位，无符号范围等于有符号 `>= 0` 部分 |
| **定点数** | `DECIMAL(M, D)` | 整数与小数拆分转十六进制存储，**无精度损失**；`M <= 65`（总位数），`D < M`（小数位）；金额等场景必用 |
| **定长/变长文本** | `CHAR(M)`, `VARCHAR(M)` | `CHAR` 定长（过长浪费、过短越界）；`VARCHAR` 变长按实际存 |
| **大文本** | `TINYTEXT`(255), `TEXT`(65535), `MEDIUMTEXT`(16M), `LONGTEXT`(4G) | 无需预设长度，按实际存；**`TEXT` 不能作主键**，性能低于 `CHAR`/`VARCHAR` |
| **枚举/集合** | `ENUM`, `SET` | `ENUM` 取预设集合中 1 个值；`SET` 取 0 个或多个 |
| **日期时间** | `YEAR`, `TIME`, `DATE`, `TIMESTAMP`, `DATETIME` | `TIME` 可表示超过 24 小时时间间隔；**优先用 `DATETIME`**（范围最大且包含完整年月日时分秒） |

- **通用防错选型**：整数用 `INT`、精确小数用 `DECIMAL`、非主键文本用 `TEXT`、日期时间用 `DATETIME`。

### 2.2 6 种表约束
1. **`DEFAULT 值`**：插入未指定字段时自动填充默认值。
2. **`PRIMARY KEY`**：每表最多 1 个；必须唯一且非空（自动附带 `NOT NULL`）。
3. **`FOREIGN KEY`**：定义在从表上，引用主表主键保证引用完整性。
4. **`NOT NULL`**：插入时必须赋值（或存在默认值），否则报错。
5. **`UNIQUE`**：值不可重复，每表可设多个；**允许为 `NULL`（且最多 1 条记录为 `NULL`）**。
6. **`AUTO_INCREMENT`**：**仅限整数类型**；新增记录自动 +1；若手动插入更大的值（如 `100`），自增基数重置，下次默认从 `101` 开始。

### 2.3 表结构修改常用语法
```sql
CREATE TABLE 新表名 LIKE 原表名; -- 复制空表结构
ALTER TABLE 表名 ADD [COLUMN] 字段名 类型 [FIRST | AFTER 字段名];
ALTER TABLE 表名 CHANGE 旧字段名 新字段名 新类型;
ALTER TABLE 表名 MODIFY 字段名 新类型 [FIRST | AFTER 字段名];
ALTER TABLE 表名 ENGINE=INNODB;
```
- **其他扩展属性**：`AUTO_EXTENDSIZE` 指定表空间自增步长；`INVISIBLE`（MySQL 8.0.23+）隐藏列但保留功能。
- **避坑**：字段名避开保留字（如不用 `value`，改用 `importvalue`）；`MODIFY ... NOT NULL` 前必须先清理或更新现有 `NULL` 行，否则报错。

---

## 三、数据操作与基础查询（04–08）

### 3.1 增删改查（CRUD）
```sql
-- 插入与查询导入
INSERT INTO 表名 (字段1, 字段2) VALUES (值1, 值2);
INSERT INTO 表名 (字段列表) SELECT 字段列表 FROM 源表 WHERE 条件;

-- 主键/唯一键冲突时自动更新
INSERT INTO demo.goodsmaster
SELECT * FROM demo.goodsmaster1 AS a
ON DUPLICATE KEY UPDATE barcode = a.barcode, goodsname = a.goodsname;

-- 修改与删除（不要修改主键值；Workbench 安全模式下 WHERE 须含主键或唯一键）
UPDATE 表名 SET 字段 = 值 WHERE 条件;
DELETE FROM 表名 WHERE 条件;

-- 查询语法完整顺序
SELECT [DISTINCT] 字段列表
FROM 数据源
WHERE 行过滤条件
GROUP BY 分组字段
HAVING 分组过滤条件
ORDER BY 排序字段 [ASC | DESC]
LIMIT 起始偏移, 行数; -- 偏移从 0 开始
```
- **派生表规则**：`FROM` 后接子查询时，MySQL **必须指定表别名**（`AS 别名`）。

### 3.2 主键设计（05）
| 方案 | 适用场景 | 缺陷与注意事项 |
| :--- | :--- | :--- |
| **业务字段作主键**（卡号/手机号/身份证） | **尽量避免** | 业务字段可能回收重用（导致历史流水归属错乱）、隐私字段可能为空，后期改主键代价极高 |
| **自增字段作主键**（`AUTO_INCREMENT`） | **单机单库** | 多门店/多节点各自独立写入再向总部合并时，必然产生主键冲突 |
| **手动维护全局编号** | **多节点写入合并** | 取消自增，在统一管理表中维护最大编号，写入前取最大值 +1 并更新参数表 |

```sql
ALTER TABLE demo.membermaster DROP PRIMARY KEY; -- 删主键约束但保留字段
ALTER TABLE demo.membermaster ADD id INT PRIMARY KEY AUTO_INCREMENT;
-- 不删表迁移外键引用
UPDATE demo.trans AS a, demo.membermaster AS b
SET a.memberid = b.id WHERE a.transactionno > 0 AND a.cardno = b.cardno;
```

### 3.3 外键与连接查询（06）
- **外键约束规则**：定义在**从表**上。插入须**先主表后从表**；删除须**先从表后主表**（否则报错阻止）。
  ```sql
  ALTER TABLE 从表 ADD CONSTRAINT fk_name FOREIGN KEY (外键列) REFERENCES 主表 (主键列);
  -- 查询外键元数据
  SELECT constraint_name, table_name, column_name, referenced_table_name, referenced_column_name
  FROM information_schema.KEY_COLUMN_USAGE WHERE constraint_name = 'fk_name';
  ```
- **连接类型**：
  - `JOIN` / `INNER JOIN` / `CROSS JOIN` 在 MySQL 中等价，仅返回满足 `ON` 条件的交集记录。
  - `LEFT JOIN`：返回左表全部记录 + 右表匹配记录（不匹配填 `NULL`）。
  - `RIGHT JOIN`：返回右表全部记录 + 左表匹配记录（不匹配填 `NULL`）。
- **核心误区**：**`JOIN` 不依赖外键约束**；高并发场景外键有额外资源开销，若放弃数据库外键，**必须在应用层实现等价的一致性校验**。

### 3.4 `WHERE` vs `HAVING`（07）与聚合函数（08）
| 对比项 | `WHERE` | `HAVING` |
| :--- | :--- | :--- |
| **执行时机** | **先过滤后连接/分组**（缩小参与 `JOIN` 的数据集，性能高） | **先连接、分组后过滤**（资源消耗大） |
| **聚合函数** | **不可使用** | **可使用**（如 `HAVING COUNT(*) >= 2`、`HAVING SUM(x) > 100`），必须配合 `GROUP BY` |

- **最佳实践**：普通字段过滤全放 `WHERE`，聚合结果过滤放 `HAVING`，二者可在同一 SQL 中联用。
- **5 个聚合函数**：`SUM()`、`AVG()`、`MAX()`、`MIN()`、`COUNT(*)`（含 `NULL` 行）vs `COUNT(字段)`（**忽略 `NULL` 值**）。
- **易错点**：
  - 无 `GROUP BY` 时整表视作一个分组。
  - 同一查询中 `MAX(字段1)` 与 `MAX(字段2)` **各自独立计算**，结果不一定来自同一行。
  - 对 `DATETIME` 按天分组统计可用 `LEFT(transdate, 10)` 提取 `YYYY-MM-DD`。
  - 分页先用 `COUNT(*)` 求总行数再分批 `LIMIT` 拉取，避免内存溢出。

---

## 四、内置函数：时间、数学、字符串与条件（09–10）

### 4.1 时间日期函数（09）
- **提取**：`EXTRACT(YEAR|MONTH|DAY|HOUR FROM date)`；独立函数 `YEAR(d)`、`MONTH(d)`、`DAY(d)`、`HOUR(d)`、`MINUTE(d)`、`SECOND(d)`。
- **计算**：
  - `DATE_ADD(date, INTERVAL 表达式 type)` / `ADDDATE()`（支持负数向前推）；`DATE_SUB()` / `SUBDATE()` 方向相反。
  - `LAST_DAY(date)`：所在月最后一天。
  - `DATEDIFF(date1, date2)`：两日期相差天数。
  - **历史整月边界计算**：起始日用 `DATE_ADD(LAST_DAY(上上月日期), INTERVAL 1 DAY)`（不可直接用上月最后一天，否则多算 1 天）；截止日用次月 1 日。
- **当前与格式化**：
  - `CURDATE()` 返回 `'YYYY-MM-DD'`。
  - `DAYOFWEEK(date)`：**`1`=周日，`2`=周一，…，`7`=周六**（可配合 `CASE WHEN` 转为周一=1～周日=7）。
  - `DATE_FORMAT(date, format)`：`"%T"` 为 24 小时制（`13:25:50`），`"%r"` 为 12 小时制（`01:25:50 PM`）。
- **系统时间坑**：MySQL 取服务器本地系统时间，多机部署需开启 OS 网络时间同步或统一从高可靠主库获取。

### 4.2 数学、字符串与条件判断（10）
- **数学函数**：`CEIL(X)` / `CEILING(X)`（向上取整）、`FLOOR(X)`（向下取整）、`ROUND(X, D)`（四舍五入保留 `D` 位，**注意负数如 `ROUND(-1.5)` 返回 `-2`**）、`ABS(X)`、`MOD(X, Y)`。
- **字符串函数**：`CONCAT(s1, s2, ...)`、`CAST(expr AS CHAR)`、`CHAR_LENGTH(str)`、`SPACE(n)`（生成 `n` 个空格）、`SUBSTR()`、`MID()`、`TRIM()`、`LTRIM()`、`RTRIM()`。
- **条件判断**：`IFNULL(V1, V2)`、`IF(expr, V1, V2)`、`ISNULL(expr)`、`CASE 表达式 WHEN 值1 THEN ... ELSE ... END`。
- **核心坑点**：
  - **`NULL` 传播**：`CONCAT()` 任一参数为 `NULL`，结果必为 `NULL`，必须用 `IFNULL()` 兜底。
  - **显示宽度**：`CHAR_LENGTH()` 中 1 个汉字计为 `1`，但等宽排版时 1 个汉字通常占 2 个英文字符宽度。

---

## 五、索引、事务与临时表（11–13）

### 5.1 索引（11）
- **自动创建**：定义 `PRIMARY KEY` 或 `UNIQUE` 时自动生成对应索引。
- **语法与限制**：
  ```sql
  CREATE INDEX idx_name ON 表名 (字段1, 字段2); -- 组合索引最多支持 16 个字段
  CREATE INDEX idx_date ON demo.trans (transdate(10)); -- 前缀索引
  ALTER TABLE 表名 ADD { INDEX | KEY } idx_name (字段列表);
  DROP INDEX idx_name ON 表名;
  ALTER TABLE 表名 DROP PRIMARY KEY;
  ```
- **优化器选路规则**：
  - **多单字段索引互斥**：`WHERE` 中出现多个独立索引列时，优化器仅选取预估扫描行数（`rows`）最少的**一个最优单字段索引**。
  - **组合索引最左前缀原则**：条件必须从组合索引最左列开始向右连续匹配；缺失最左列或中间列中断，后续列无法使用该组合索引。
  - **范围查询中断**：前缀列使用范围条件（如 `>`）后，其右侧列无法继续利用组合索引。
- **成本**：① 额外磁盘存储开销；② 每次 `INSERT`/`UPDATE`/`DELETE` 涉及索引列时均需同步维护索引，降低写入速度。

### 5.2 事务（12）
- **基本语法**：`START TRANSACTION` / `BEGIN` → `COMMIT`（提交）/ `ROLLBACK`（回滚）。
- **ACID**：原子性（不可分割）、一致性（完整性不被破坏）、隔离性（并发互不干扰）、持久性（提交后永久生效）。
- **关键坑点**：
  - **MySQL 事务不会自动回滚报错 SQL！** 某条语句失败若不显式 `ROLLBACK` 并继续 `COMMIT`，其余成功语句仍会生效。
  - 用 `SELECT ROW_COUNT();` 检测失败（返回 `-1` 表示失败，否则为受影响行数），或在存储过程中声明 `DECLARE EXIT HANDLER FOR SQLEXCEPTION ROLLBACK;`。
  - **DDL 不可回滚**：`CREATE`/`DROP` 库表等 DDL 无法通过事务回滚。
- **4 种隔离级别**（通过锁实现，越高开销越大）：
  1. `READ UNCOMMITTED`（读未提交）
  2. `READ COMMITTED`（读已提交）
  3. `REPEATABLE READ`（**MySQL 默认**：事务内多次读取结果与首次一致）
  4. `SERIALIZABLE`（串行化：操作期间加锁禁止其他事务任何操作；仅在日结/核心结算时临时开启，用完务必恢复默认）

### 5.3 临时表（13）
```sql
CREATE TEMPORARY TABLE tmp_sales SELECT ...; -- 默认存储引擎 InnoDB（落盘）
CREATE TEMPORARY TABLE tmp_mem (...) ENGINE = MEMORY; -- 内存临时表
```
- **三大特征**：必须带 `TEMPORARY`；**连接隔离**（仅当前连接可见，不同连接可同名，天然支持并发）；**连接断开自动销毁**。
- **内存 vs 磁盘**：`MEMORY` 极快（实测 `<10ms` vs 磁盘 `210ms`），但断电丢失且挤占内存；默认 `InnoDB` 占磁盘 I/O。
- **多临时表关联坑点**：多个局部聚合临时表各自缺行时直接 `JOIN` 会丢数据，正确做法是以全集基表（如商品表）放左侧做 `LEFT JOIN`，并用 `IFNULL(col, 0)` 补 0。

---

## 六、视图、存储过程、游标与触发器（14–17）

### 6.1 视图（14）
- **本质**：仅保存 `SELECT` 定义的虚拟表，不存实际数据。
- **语法**：`CREATE [OR REPLACE] VIEW 视图名 [(字段)] AS 查询;`、`ALTER VIEW`、`DESCRIBE`、`DROP VIEW`。
- **更新限制**：穿透执行到基表；仅当视图字段与基表完全一致时可 `INSERT`；含 `GROUP BY`、聚合函数、`UNION`、`DISTINCT` 的视图**不可更新**。
- **实践建议**：**仅用于查询**（简化复杂 SQL、屏蔽敏感列、基表删列时可用 `'' AS col` 在视图中占位保证兼容）；**避免嵌套视图**（可读性差、维护成本极高）。

### 6.2 存储过程与存储函数（15–16）
```sql
DELIMITER //
CREATE PROCEDURE proc_name (IN p1 INT, OUT p2 DECIMAL(10,2), INOUT p3 INT)
BEGIN
    DECLARE done INT DEFAULT FALSE;
    DECLARE v_val DECIMAL(10,2);
    DECLARE cur CURSOR FOR SELECT col FROM tbl;
    DECLARE CONTINUE HANDLER FOR NOT FOUND SET done = TRUE;

    OPEN cur;
    REPEAT
        FETCH cur INTO v_val;
        -- 业务逻辑...
    UNTIL done END REPEAT;
    CLOSE cur;
END //
DELIMITER ;
```
- **参数类型**：`IN`（默认，仅输入）、`OUT`（仅输出）、`INOUT`（输入输出）。
- **常用管理**：`CALL proc_name(...);`、`SHOW CREATE PROCEDURE name \G`、`DROP PROCEDURE name;`（不支持 SQL 直接修改过程体，需删后重建）。
- **存储函数 vs 存储过程**：
  - 存储函数（`CREATE FUNCTION name(...) RETURNS type`）必须返回值/表，**可直接嵌入 `SELECT` 中**，不可 `CALL`，**不支持建删表与事务**。
  - 存储过程可不返回值，用 `CALL` 调用，支持事务与建删表，不可嵌入 `SELECT`。
- **游标与流程控制**（仅限存储过程/函数内部使用）：
  - **游标特点**：只读、单向向前、不可跳跃。4 步：`DECLARE ... CURSOR` → `OPEN` → `FETCH ... INTO`（列数必须与变量数一致）→ **`CLOSE`（务必显式关闭释放资源）**。
  - **流程控制**：`IF ... ELSEIF ... ELSE ... END IF`、`CASE ... END CASE`、`LOOP ... END LOOP`（需配合 `LEAVE` 跳出 / `ITERATE` 继续）、`WHILE ... DO ... END WHILE`（先判断）、`REPEAT ... UNTIL ... END REPEAT`（先执行）。
- **优缺点**：减少网络往返、不暴露源码；但**无原生集成调试器**（可用 `SELECT 变量` 打印中间值），注意防范除零等边界错误。

### 6.3 触发器（17）
```sql
CREATE TRIGGER trig_name {BEFORE | AFTER} {INSERT | UPDATE | DELETE}
ON 表名 FOR EACH ROW
BEGIN
    -- 可使用 OLD.字段（变更前）与 NEW.字段（变更后）
END;
```
- **管理**：`SHOW TRIGGERS\G;`、`DROP TRIGGER trig_name;`。
- **原子性与错误传播**：**触发器内操作失败时，原触发 DML 语句一并失败回滚**（应用端可用 `SELECT ROW_COUNT();` 返回 `-1` 捕获）。
- **优缺点**：适合强一致性校验、自动写历史/审计日志；但**隐蔽性强、可读性差**，关联表改动易引发难排查的报错，使用必须配齐设计文档。

---

## 七、权限管理、日志体系与备份恢复（18–21）

### 7.1 权限与角色管理（18，MySQL 8.0+）
- **原则**：按水平（库表范围）与垂直（`SELECT`/`INSERT`/`UPDATE`/`DELETE`）最小授权，**严禁应用直连 `root`**。
- **核心操作**：
  ```sql
  CREATE ROLE 'stocker'; -- 省略主机名默认为 '%'，初始仅 USAGE 登录权限
  GRANT SELECT, INSERT, UPDATE, DELETE ON demo.invcount TO 'stocker';
  CREATE USER 'zhangsan' IDENTIFIED BY 'mysql';
  GRANT 'stocker' TO 'zhangsan';
  SET GLOBAL activate_all_roles_on_login = ON; -- 必须开启！否则用户登录后角色默认未激活
  SHOW GRANTS FOR 'zhangsan';
  DROP ROLE 'stocker'; -- 自动回收关联用户的对应权限
  DROP USER 'zhangsan';
  ```

### 7.2 六大日志体系（19–20）
| 日志 | 默认状态与配置 | 核心作用与运维操作 |
| :--- | :--- | :--- |
| **通用查询日志** | 默认 `OFF` | 明文记录所有连接与 SQL，开销大。`SET GLOBAL general_log='ON';`；轮转需先 `OFF` → 移走旧文件 → 再 `ON` |
| **慢查询日志** | `my.ini` 配置并重启 | 记录 `examined_rows >= min_examined_row_limit AND query_time > long_query_time`（默认 10s）的语句，含 `Query_time`、`Lock_time`、`Rows_examined` 等 |
| **错误日志** | 默认开启 (`log-error`) | 记录启停时间及运行时错误/警告/诊断信息 |
| **二进制日志 (Binlog)** | 启停/刷新/超 `max_binlog_size` 自动切换 | 记录 DDL/DML 变更及耗时，用于**无损时间点恢复**与**主从复制**。命令：`SHOW MASTER STATUS;`、`SHOW BINARY LOGS;`、`SHOW BINLOG EVENTS IN '...'`、`FLUSH BINARY LOGS;`、`PURGE MASTER LOGS TO '...'`、`RESET MASTER;`（清空并从 `.000001` 重计） |
| **中继日志 (Relay Log)** | 仅从库存在 | 缓存主库传来的 binlog；命名 `<主机名>-relay-bin.<序号>`。**坑点：重装系统改主机名会导致从库无法识别原有中继日志，需改回原主机名** |
| **回滚日志 (Undo Log)** | `innodb_undo_tablespaces=2` 等 | 存变更前数据映像，支撑回滚与 MVCC；开启 `innodb_undo_log_truncate=ON`（配合独立 undo 表空间）可自动回收空间（默认单文件上限 1GB） |
| **重做日志 (Redo Log)** | `innodb_log_files_in_group` (1~4) | 保证持久性，宕机后重放内存中尚未刷盘的已提交事务 |

### 7.3 数据备份、导出与恢复（20–21）
- **`mysqldump` 逻辑备份**（不写密码则在 `-p` 后交互输入）：
  ```bash
  # 1. 备份指定表（恢复时需先指定目标库）
  mysqldump -h localhost -u root -p demo goodsmaster > tbl.sql
  # 2. 备份指定库（含 CREATE DATABASE 与 USE）
  mysqldump -u root -p --databases demo inventory > dbs.sql
  # 3. 备份全实例（含系统库）
  mysqldump -u root -p --all-databases > all.sql
  ```
- **全量 + 增量无损恢复流程**：
  1. 全量备份后立即执行 `FLUSH BINARY LOGS;` 切分新 binlog。
  2. 故障恢复前**再次执行 `FLUSH BINARY LOGS;`**，避免恢复语句污染待回放的增量 binlog。
  3. 恢复全量：`mysql -u root -p demo < tbl.sql`（或在客户端 `USE demo; SOURCE H:\\tbl.sql;`）。
  4. 用 `SHOW BINLOG EVENTS IN '...'` 定位起点/终点 pos，回放增量：
     ```bash
     mysqlbinlog --start-position=306 --stop-position=xxx "路径\bin.000012" | mysql -u root -p
     ```
- **高速导出与导入**（远快于 Workbench 图形向导）：
  ```sql
  SELECT * INTO OUTFILE 'C:/ProgramData/MySQL/MySQL Server 8.0/Uploads/data.txt'
  FIELDS TERMINATED BY ',' LINES TERMINATED BY '\n' FROM demo.trans;

  LOAD DATA INFILE 'C:/ProgramData/MySQL/MySQL Server 8.0/Uploads/data.txt'
  INTO TABLE demo.trans FIELDS TERMINATED BY ',' LINES TERMINATED BY '\n';
  ```
  - 若报 `ERROR 1290 (HY000)`，受 `my.ini` 中 `secure-file-priv` 目录限制，须放到允许路径下。

---

## 八、数据库设计与性能优化（22–26）

### 8.1 范式与业务优先原则（22）
- **1NF**：列不可再分（原子性，如将 `property` 拆为 `specification` 和 `unit`）。
- **2NF**：满足 1NF + 有主键唯一标识 + 所有非主键列完全依赖**整个**主键（消除部分依赖，如用代理主键 `itemnumber` 替代可能重复的 `barcode`）。
- **3NF**：满足 2NF + 消除非主键列之间的传递依赖（如将依赖 `supplierid` 的 `suppliername` 拆入 `supplier` 表）。MySQL 设计一般满足 3NF 即可。
- **业务优先原则（适度反范式）**：当严格范式引发业务精度或计算复杂度问题时保留冗余。例：`importdetails` 同时存 `quantity`、`importprice`、`importvalue`，避免促销只给总价时因单价四舍五入反算总价产生累计误差（如 `25.5/34=0.74` → `0.74*34=25.16`，差 `0.34`）。

### 8.2 ER 模型与建表转换（23）
- **ER 三要素**：**实体**（矩形；强实体可独立存在，弱实体强依赖父实体用粗线矩形）、**属性**（椭圆形；不可再分子项）、**联系**（菱形；`1:1`、`1:N`、`M:N`）。
- **ER → 关系表 4 条规则**：
  1. **1 实体 → 1 表**：弱实体表中必须包含父强实体主键作 `NOT NULL` 外键。
  2. **`M:N` 联系 → 独立中间表**：包含各方主键及联系自身属性，按 3NF 拆单头/单明细。
  3. **`1:1` / `1:N` 联系 → 外键约束**：不单独建表，直接通过外键关联。
  4. **属性 → 表字段**：赋予数据类型与约束。

### 8.3 查询优化与 `EXPLAIN`（24）
- **执行计划**：`EXPLAIN 查询语句;`
  - `select_type`：`SIMPLE`、`PRIMARY`、`UNION`、`DEPENDENT UNION`、`DEPENDENT SUBQUERY`、`UNION RESULT`。
  - `type`（由劣到优关注）：`ALL`（全表扫描）→ `index_merge` → `range`（范围索引）→ `ref` → `eq_ref`。
  - `possible_keys`（候选索引）、`key`（实际命中索引）、`rows`（预估扫描行数）、`Extra`（`Using where`、`Using index condition`、`Using temporary`、`Using sort_union(...)`、`Using MRR`）。
- **SQL 改写铁律**：
  1. **`LIKE` 通配符**：`LIKE '%aa'` 或 `LIKE '%aa%'` 导致索引失效（`type=ALL`）；**`LIKE 'aa%'` 可走索引（`type=range`）**。
  2. **`OR` 条件**：`WHERE expr1 OR expr2` **只有当所有 `OR` 分支涉及列均有索引时**才触发 `index_merge`；任一分支无索引即退化为全表扫描。
  3. **子查询改写**：`IN (SELECT ...)` 易物化生成临时表，改写为 **`JOIN`** 效率更高。

### 8.4 大表设计优化（25）
MySQL 按固定大小数据页读写，单行越小，单页容纳记录越多，I/O 效率越高：
1. **缩小数据类型**：选用覆盖业务范围的最小整数类型（如 `MEDIUMINT` 3B 替代 `INT` 4B 省 25%）；能用整数存的数字编号不用文本（18 位数字 `BIGINT` 8B vs `TEXT` 20B 省 60%）。
2. **受控冗余字段**：对极高频查询且极少变更的关联字段（如流水表冗余 `goodsname`）消除大表 `JOIN`，需保证同步更新。
3. **垂直拆表**：按冷热访问频率拆分宽表（如主表与支付明细表共享主键）。
4. **尽量加 `NOT NULL`**：省去每列 1 bit 标记位与空值判断开销，且更利于索引。

### 8.5 系统参数调优与 `Performance Schema`（26）
- **修改 `my.ini` 注意**：Windows 下必须以 **ANSI 编码**保存（不可用 UTF-8），修改后重启 MySQL 生效。
- **核心 InnoDB 参数**：
  - `innodb_flush_log_at_trx_commit`：
    - `1`（默认）：每次提交写日志并刷盘，最安全但 I/O 与 CPU 压力最大。
    - `0`：每秒写日志并刷盘一次，最快但最易丢数据。
    - `2`：每次提交写日志、每秒刷盘一次，高并发下兼顾安全与吞吐。
  - `innodb_buffer_pool_size`：缓存索引与数据（如 128G 专用库可设 `64G`）。
  - `innodb_buffer_pool_instances`：缓冲池分区实例数（如 `64`），提升并发并行度。
- **慢 SQL 定位（`Performance Schema`）**：
  ```sql
  UPDATE performance_schema.setup_instruments SET ENABLED = 'YES', TIMED = 'YES';
  UPDATE performance_schema.setup_consumers SET ENABLED = 'YES';
  -- TIMER_WAIT 单位为皮秒（10^-12 秒），TRUNCATE 直接截断不四舍五入
  SELECT TRUNCATE(TIMER_WAIT / 1000000000000, 6) AS duration, sql_text, EVENT_ID
  FROM performance_schema.events_statements_history_long
  WHERE TRUNCATE(TIMER_WAIT / 1000000000000, 6) <> 0 AND sql_text IS NOT NULL
  ORDER BY duration DESC LIMIT 1, 2;
  ```
  - 3 张历史表：`events_statements_current`（当前）、`events_statements_history`（每线程最近 10 条）、`events_statements_history_long`（全局最近 10000 条）。

---

## 九、综合实战：连锁超市系统设计要点（27–28）

- **SaaS 两层隔离**：商户编号 `groupnumber`（租户隔离）+ 门店编号 `branchid`（总店/直营/加盟/配送中心）。
- **分库分表 4 维策略**（设计期预留，小项目勿过度拆分）：
  - **垂直分表**：高频列与低频列拆分（如 `goods_o` 存名称/条码/售价，`goods_f` 存规格/单位），通过双向 `BEFORE DELETE` 触发器保持删除一致。
  - **垂直分库**：按业务域拆分为 `sales`、`inventory`、`operation`、`member` 库，分散流量并隔离故障。
  - **水平分表（冷热归档 + 租户分段）**：已确认盘点单通过事务+游标+存储过程更新库存后，`INSERT ... SELECT` 归档至只读历史表 `invcountheadhist` / `invcountdetailshist` 并从热表删除；大表按 `groupnumber` 区间（如 500–1000）拆分。
  - **水平分库**：按商户数阈值（如每 500 商户一套库、每 5000 商户一台服务器）动态路由。
- **高可用与备份配合**：
  - **主从复制**：主库配 `log-bin`、`binlog-do-db`、`server-id=1`；从库配 `replicate_do_db`、`server-id=2`；从库执行 `CHANGE MASTER TO ...; START SLAVE;` 同步。
  - **必须叠加定期备份**：主从复制无法防范误删表或区域故障，仍需每日 `mysqldump` 按星期 7 天滚动备份。

---

## 十、MySQL 8 新特性、空间定位与高频面试题（特别放送）

### 10.1 MySQL 8 核心新特性
- **窗口函数**：`func() OVER ([PARTITION BY 分组列] [ORDER BY 排序列])` 或复用 `WINDOW w AS (...)`。**不折叠行**，每行均可直接附带分区或全局聚合值，避免临时表与自连接。
  - 排名函数对比：
    - `RANK()`：并列同名次，**留空位**（`1, 1, 3`）
    - `DENSE_RANK()`：并列同名次，**不留空位**（`1, 1, 2`）
    - `ROW_NUMBER()`：严格递增，忽略并列（`1, 2, 3`）
- **公用表表达式（CTE）**：
  - 普通 CTE：`WITH cte AS (子查询) SELECT ...`（可多次引用或串联）。
  - **递归 CTE**：`WITH RECURSIVE cte AS (初始查询 UNION [ALL] 递归查询)`，初始查询执行 1 次，递归查询不断引用 `cte` 直至无新行产生，专治无限层级树形/组织结构查询。
- **原子 DDL**：`CREATE`、`ALTER`、`DROP`（含 `CREATE TABLE ... SELECT`）具备原子性，失败自动回滚。

### 10.2 空间定位与 GIS（特别放送四）
- **数据类型**：单值 `GEOMETRY`、`POINT`、`LINESTRING`、`POLYGON`；集合 `MULTIPOINT`、`MULTILINESTRING`、`MULTIPOLYGON`、`GEOMETRYCOLLECTION`。
- **WKT 与 SRID**：`ST_GeomFromText(WKT, SRID)`；`SRID 0` 为平面坐标，`SRID 4326` 为 WGS 84 地心坐标系。
  - **格式坑**：`SRID 4326` 下 `POINT(纬度 经度)` **纬度在前、经度在后、空格分隔（无逗号）**；`POLYGON((...))` 首尾点必须闭合一致。
- **核心函数**：`ST_Distance_Sphere(g1, g2)` 返回两点最短球面距离（单位：**米**）；`MBRContains(g1, g2)` / `MBRWithin(g1, g2)` 判断最小外接矩形包含关系。
- **空间索引（R-Tree）**：InnoDB/MyISAM 均支持；字段**必须 `NOT NULL` 且显式指定 `SRID 4326`**；`WHERE` 使用 `MBRContains`/`MBRWithin` 即可触发索引（`EXPLAIN` 显示 `type: range`）。

### 10.3 面试高频核心结论速查
1. **取第 N 高去重值**：`SELECT DISTINCT score FROM tbl ORDER BY score DESC LIMIT N-1, 1;`
2. **分组取 Top-K（相关子查询通用写法）**：`WHERE K > (SELECT COUNT(DISTINCT b.points) FROM student b WHERE b.points > a.points AND b.classid = a.classid)`
3. **WAL（预写日志）三原则**：先写顺序日志再刷数据页；按执行顺序写日志；日志落盘成功事务才返回提交成功。
4. **Checkpoints（检查点）**：定期将内存脏页批量刷盘并记录位置，崩溃恢复仅需从检查点重放，无需扫描全量日志。
5. **三类恢复相关日志分工**：`Undo log` 回滚未提交事务；`Redo log` 重放已提交但未刷盘数据；`Binlog` 用于主从复制与时间点增量恢复。
6. **存储引擎与事务**：支持与否取决于存储引擎（`SHOW CREATE TABLE` 查看）；`InnoDB` 支持，`MyISAM` 忽略事务控制。
