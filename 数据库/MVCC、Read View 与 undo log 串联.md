# MVCC、Read View 与 undo log 串联

## 一句话总览

MVCC（Multi-Version Concurrency Control，多版本并发控制）是 InnoDB 处理普通读写并发的机制。它让查询不必总是等待正在修改同一行的事务：写操作产生新版本，旧版本保留在 undo log 中；读操作根据自己的 `Read View`，从版本链中选择当前可见的版本。

```text
Buffer Pool：保存当前最新的数据页
undo log：保存历史版本
Read View：规定当前事务能看到哪些事务提交的版本
MVCC：把三者组合起来，让普通读读取正确的版本
锁：处理当前读和写操作的并发冲突
```

## MVCC 要解决什么问题？

没有 MVCC 时，一行数据被事务 A 修改但尚未提交，事务 B 要读取这一行，通常只能等待事务 A 结束，或冒险读取未提交数据。

MVCC 的目标是：

```text
普通读不阻塞普通写
普通写不阻塞普通读
读操作只能看到符合隔离级别的已提交版本
```

这能提高读写并发能力，同时避免脏读。

## 一行数据有哪些隐藏信息？

InnoDB 的聚簇索引记录中会维护一些内部信息。理解 MVCC 时，重点关注：

| 字段 | 作用 |
| --- | --- |
| `trx_id` | 最近一次修改这行数据的事务 ID |
| `roll_ptr` | 指向 undo log 中上一个版本的指针 |
| `DB_ROW_ID` | 表没有主键或合适唯一键时，InnoDB 可能使用的内部行 ID |

其中，`trx_id` 说明“最新版本是谁写的”，`roll_ptr` 让 InnoDB 能从当前版本沿着 undo log 找到更早版本。

```text
当前版本：status = published, trx_id = 200, roll_ptr -> undo 版本
                                             ↓
历史版本：status = draft, trx_id = 150
```

这条链称为**版本链**。

## 一次更新如何形成版本链？

假设文章当前状态是：

```text
id = 100
status = draft
trx_id = 150
```

事务 T2 执行：

```sql
UPDATE article
SET status = 'published'
WHERE id = 100;
```

简化后的过程是：

```text
1. 旧值 draft 写入 undo log
2. Buffer Pool 中的数据页改为 published
3. 当前行的 trx_id 改为 T2 的事务 ID，例如 200
4. 当前行的 roll_ptr 指向保存 draft 的 undo 记录
5. redo log 记录页修改，用于崩溃恢复
```

更新后：

```text
Buffer Pool 当前版本：published (trx_id = 200)
                              |
                              | roll_ptr
                              v
undo log 历史版本：draft (trx_id = 150)
```

注意：MVCC 依赖 undo log 保留历史版本；redo log 的职责是崩溃恢复，不负责决定查询该读哪个版本。

## Read View 是什么？

`Read View` 可以理解为一次普通一致性读创建的“可见性快照规则”。它不是直接保存整张表的数据，而是记录当时事务活跃情况，用于判断某个版本对当前事务是否可见。

判断逻辑不需要死记实现细节，但要理解结果：

```text
版本由已提交事务产生，并且提交时间在 Read View 允许范围内
  -> 当前事务可以看到这个版本

版本来自仍未提交的事务，或在快照建立后才提交的事务
  -> 当前事务不能看到这个版本，需要沿 undo 版本链继续找
```

## 用两个事务串起来理解

初始数据：

```text
article(id = 100, status = draft)
```

事务 T1 先开始，并执行普通查询：

```sql
SELECT status FROM article WHERE id = 100;
```

T1 读到：

```text
draft
```

随后事务 T2 更新并提交：

```sql
UPDATE article SET status = 'published' WHERE id = 100;
COMMIT;
```

此时 Buffer Pool 中的最新版本已经是：

```text
published
```

T1 再次执行相同的普通查询时，结果取决于隔离级别。

### 在 RC（Read Committed）下

RC 通常让每次普通一致性读创建新的 Read View。

```text
T1 第一次查询 -> draft
T2 提交
T1 第二次查询 -> published
```

因此 RC 避免脏读，但同一事务中两次读取可能不同，这就是不可重复读。

### 在 RR（Repeatable Read）下

RR 下，同一事务中的普通一致性读通常复用第一次一致性读建立的 Read View。

```text
T1 第一次查询 -> draft
T2 提交
T1 第二次查询 -> 仍然是 draft
```

此时 T1 虽然能在 Buffer Pool 中找到最新的 `published`，但这个版本对它不可见，于是会沿 `roll_ptr` 到 undo log 中找到可见的 `draft`。

## 快照读和当前读一定要区分

MVCC 主要服务于**普通一致性读**，也就是常见的普通 `SELECT`。

```sql
SELECT * FROM article WHERE id = 100;
```

这类读取通常称为快照读，读取的是符合 Read View 规则的版本。

而以下操作属于当前读或写操作：

```sql
SELECT * FROM article WHERE id = 100 FOR UPDATE;
SELECT * FROM article WHERE id = 100 FOR SHARE;
UPDATE article SET status = 'published' WHERE id = 100;
DELETE FROM article WHERE id = 100;
```

当前读要读取最新可见版本，并通过锁处理并发修改。它不能只依赖 MVCC，否则会出现丢失更新或范围并发冲突。

```text
普通 SELECT
  -> 快照读
  -> MVCC + Read View

SELECT ... FOR UPDATE / UPDATE / DELETE
  -> 当前读或写
  -> 最新版本 + 锁机制
```

## MVCC 和幻读的关系

不要简单说“MVCC 解决所有幻读”。更准确的表达是：

- RR 下的普通快照读复用 Read View，结果集通常保持稳定。
- 对范围 `UPDATE`、`DELETE`、`SELECT ... FOR UPDATE` 等当前读，InnoDB 还需要根据索引和查询条件使用记录锁、间隙锁或临键锁，防止其他事务插入符合范围的新记录。

所以，MVCC 与锁机制是配合关系，不是替代关系。

## 长事务为什么会带来问题？

只要有旧事务仍可能需要历史版本，对应 undo 记录就不能马上清理。长事务会让历史版本堆积，增加 undo 空间、purge 压力和版本链查找成本。

```text
长事务长期不提交
  -> 旧 Read View 一直存在
  -> 新版本无法完全清理旧 undo
  -> 历史链变长、资源压力变大
```

因此，应避免在事务中等待网络调用、用户输入或长时间计算；批处理也要控制单次事务大小和持续时间。

## MVCC 与其他组件的边界

| 机制 | 主要解决的问题 | 不负责什么 |
| --- | --- | --- |
| MVCC | 普通读写并发和版本可见性 | 崩溃恢复、复制 |
| undo log | 回滚、历史版本 | 已提交数据页的崩溃重做 |
| redo log | 崩溃恢复和持久性 | 事务读取哪个版本 |
| binlog | 复制、逻辑恢复、CDC | InnoDB 页恢复 |
| Buffer Pool | 缓存数据页和索引页 | 业务级缓存、版本可见性规则 |
| 锁 | 当前读和写操作的冲突控制 | 普通快照读的版本选择 |

## 面试推荐回答

> MVCC 是 InnoDB 的多版本并发控制机制，主要用于让普通读和写尽量不互相阻塞。更新数据时，最新版本会写入 Buffer Pool，旧版本通过 undo log 保留，并由 `roll_ptr` 串成版本链；普通查询创建或复用 Read View，再判断当前行的哪个版本可见。RC 下每次一致性读通常看到新的已提交版本，RR 下同一事务的普通一致性读通常复用快照，因此可以避免不可重复读。MVCC 不等于没有锁，`SELECT ... FOR UPDATE`、`UPDATE`、`DELETE` 等当前读和写操作仍需要锁机制保证并发正确性。

## 记忆口诀

```text
新值留在 Buffer Pool，
旧值沿 undo 找回头；
Read View 决定谁可见，
当前读写仍要上锁。
```
