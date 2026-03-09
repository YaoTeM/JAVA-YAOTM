按你简历上的「数据库」一条，把 **MySQL（InnoDB）、B+ 树与 SQL 调优、事务隔离与 MVCC、锁、ShardingSphere** 串成一套可背可讲的提纲，并尽量结合 Ragent 项目（事务、表设计、查询场景）；ShardingSphere 本项目未用，可当作「其他项目/学习经验」来答。

---

# 一、MySQL（InnoDB）与 B+ 树索引

## 1. InnoDB 简要

- **存储引擎**：支持事务、行级锁、外键、崩溃恢复；默认引擎。
- **表数据与索引**：聚簇索引（主键 B+ 树）存整行数据；二级索引存「索引列 + 主键」，回表时用主键再查聚簇索引。
- **页**：按页（如 16KB）组织，B+ 树节点即页，便于顺序/范围扫描。

## 2. B+ 树索引

- **结构**：多路平衡树，非叶子节点只存键+指针，叶子节点存键+数据（聚簇）或键+主键（二级），叶子之间链表串起，便于范围查询和排序。
- **为什么用 B+ 树**：相比二叉树减少高度、减少磁盘 I/O；相比 B 树叶子链表使范围扫描更高效；适合等值、范围、ORDER BY。
- **覆盖索引**：查询列都在某二级索引里，不需回表，减少随机 I/O。
- **最左前缀**：联合索引 (a,b,c) 可用于 a、a,b、a,b,c 的等值/范围，不能跳过 a 用 b,c。

## 3. SQL 调优思路（口述级）

- 用 **EXPLAIN** 看 type（const/ref/range/index/all）、possible_keys、key、rows；避免全表扫描（all）和大 rows。
- 为常查条件建索引：WHERE、ORDER BY、GROUP BY 的列；避免在索引列上函数、类型隐式转换。
- 大表分页：深分页用子查询/延迟关联或记上一页最大 id 条件，避免 `LIMIT 大 offset`。
- 项目里：知识库按 `kb_id`、文档按 `doc_id`、状态按 `status` 查，设计索引时会考虑这些查询条件（见下「项目对应」）。

---

# 二、事务隔离、MVCC 与锁

## 1. 事务隔离级别（SQL 标准）

| 级别        | 脏读 | 不可重复读 | 幻读                          |
| ----------- | ---- | ---------- | ----------------------------- |
| 读未提交    | 可能 | 可能       | 可能                          |
| 读已提交 RC | 不会 | 可能       | 可能                          |
| 可重复读 RR | 不会 | 不会       | 可能（InnoDB 通过间隙锁减轻） |
| 串行化      | 不会 | 不会       | 不会                          |

- **MySQL 默认 RR**；很多项目用 RC，减少锁、提高并发。
- **脏读**：读到别的事务未提交的数据；**不可重复读**：同一行两次读结果不同；**幻读**：同一条件两次读行数不同。

## 2. MVCC（多版本并发控制）

- **作用**：读不加锁，通过「版本」读历史快照，提高读并发。
- **实现要点**：每行有隐藏列（如 DB_TRX_ID、DB_ROLL_PTR）；Undo Log 形成版本链；读时根据隔离级别和 Read View 决定能看到哪个版本。
- **RC**：每次读都生成新 Read View，能读到已提交的最新数据。  
- **RR**：第一次读生成 Read View 并复用，所以同一事务内重复读同一行结果一致。

## 3. 锁（简要）

- **行锁**：锁索引记录（主键或二级索引）；**间隙锁**：锁索引间隙，RR 下防幻读；**临键锁**：行锁+间隙锁。
- **共享锁 S**：SELECT ... LOCK IN SHARE MODE；**排他锁 X**：UPDATE/DELETE、SELECT ... FOR UPDATE。
- **意向锁**：表级，表示「表内有行被加锁」，避免逐行检查。

项目里：没有显式 `FOR UPDATE`，但用 **@Transactional** 和 **TransactionTemplate** 做事务边界，配合分布式锁（如文档分块）保证一致性；若面试问「什么时候用行锁」可以说：高并发更新同一行（如库存）会用 `SELECT ... FOR UPDATE` 或乐观锁，我们 Ragent 里是「分布式锁 + 事务」保证文档分块不并发。

---

# 三、ShardingSphere 分库分表（简历可答）

- **场景**：单表数据量大、写/读压力大，通过分库分表做水平拆分。
- **分片键**：按用户 ID、订单 ID、时间等取模或范围分到不同库/表；查询尽量带分片键，避免全库扫描。
- **ShardingSphere**：透明分片、SQL 解析与改写、分布式主键、读写分离等；可结合「在别的项目里对订单/日志表按 user_id 或 time 分表，配合 ShardingSphere 配置分片规则」来答。
- Ragent 当前未用 ShardingSphere，面试可说：「当前项目数据量还没到要分库分表，在 XX 项目里用过 ShardingSphere 做分表」。

---

# 四、项目里的数据库用法（Ragent）

## 1. 表与主键

- **MyBatis Plus**：`@TableName("t_knowledge_document")`、`@TableId(type = IdType.ASSIGN_ID)`（雪花等分布式 ID）、`@TableLogic` 逻辑删除。
- **自动填充**：`MyMetaObjectHandler` 在 insert 时填 `createTime`、`updateTime`、`deleted`，update 时填 `updateTime`，统一审计字段。

## 2. 事务

- **声明式**：`@Transactional(rollbackFor = Exception.class)` 用在文档/知识库/分块/管道/会话等 Service 方法上，异常回滚。
- **编程式**：文档分块前在**分布式锁内**用 **TransactionTemplate.executeWithoutResult**：查文档、校验状态、删历史分块、写 schedule、改状态、再提交异步分块任务，保证「查-改-写」在同一事务里；后续保存 chunk 结果也用 TransactionTemplate 做批量写+状态更新，保证一致性。

## 3. 查询与索引思路

- 文档按 `kb_id`、`doc_id`、`status` 查；定时任务按 `next_run_time`、`lock_until` 扫待执行；分块按 `doc_id` 查。设计索引时会考虑这些条件（如 kb_id+status、doc_id、next_run_time 等），避免全表扫描。

---

# 五、面试 Q&A

**Q1：InnoDB 的 B+ 树索引结构？为什么用 B+ 树？**  
答：InnoDB 用 B+ 树做索引，聚簇索引叶子存整行，二级索引叶子存索引列+主键。B+ 树多路平衡、高度低、I/O 少，叶子成链表便于范围查询和排序，适合数据库的等值和范围查询。我们项目里知识库、文档、分块等表的主键用雪花 ID，查询常带 kb_id、doc_id、status，设计时会为这些查询条件考虑索引。

---

**Q2：事务隔离级别有哪些？MySQL 默认是哪个？MVCC 解决什么问题？**  
答：读未提交、读已提交、可重复读、串行化；MySQL 默认可重复读。MVCC 通过多版本和 Read View 让读不加锁也能看到一致快照，提高读并发；RR 下同一事务内重复读同一行结果一致。我们业务里用 @Transactional 和 TransactionTemplate 控制事务边界，配合分布式锁（如文档分块）保证不并发写同一资源。

---

**Q3：你们项目里事务是怎么用的？为什么有的地方用 TransactionTemplate？**  
答：大部分写操作用 @Transactional(rollbackFor = Exception.class)。文档分块时要在分布式锁内做「查文档、删历史分块、写 schedule、改状态、再提交异步任务」，这一段需要在一个事务里完成，但又要先拿锁再进事务，所以用 TransactionTemplate.executeWithoutResult 在锁内手动开事务，保证要么全部成功要么全部回滚；保存分块结果时同样用 TransactionTemplate 做批量插入和状态更新。

---

**Q4：SQL 调优一般怎么做？**  
答：先用 EXPLAIN 看执行计划，看是否走索引、type 和 rows；避免索引列上函数、隐式类型转换导致索引失效；常查的 WHERE、ORDER BY 考虑联合索引和最左前缀；大表分页避免大 offset，用延迟关联或上次最大 id。我们表上有 kb_id、doc_id、status、next_run_time 等查询，设计索引时会围绕这些条件。

---

**Q5：有分库分表经验吗？**  
答：当前 Ragent 还没做分库分表。在别的项目里用过 ShardingSphere，对订单/日志类表按 user_id 或时间分表，配置分片规则和分布式主键，查询尽量带分片键避免全库扫描。若 Ragent 以后知识库或对话数据量上来，也可以考虑对部分表做分表。

---

**Q6：MyBatis Plus 你们用了哪些特性？**  
答：用 @TableName、@TableId(ASSIGN_ID)、@TableLogic；MetaObjectHandler 统一填充 createTime、updateTime、deleted；LambdaQueryWrapper 写条件；事务用 Spring 的 @Transactional 和 TransactionTemplate。主键用雪花算法生成，兼容分布式和迁移。

---

按上面顺序过一遍「InnoDB/B+ 树/调优 → 隔离/MVCC/锁 → ShardingSphere → 项目事务与表设计」，再练这几道 Q&A，数据库这块就能和简历、项目对上，面试有话说。