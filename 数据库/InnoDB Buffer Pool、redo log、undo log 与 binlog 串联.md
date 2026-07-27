# InnoDB Buffer Pool、redo log、undo log 与 binlog 串联

## 一句话总览

一次 InnoDB 更新并不是“直接修改磁盘上的一行数据”。数据页先在 `Buffer Pool` 中被读取和修改；`undo log` 保存旧版本，用于回滚和 MVCC；`redo log` 保证已提交但尚未刷入数据文件的修改在崩溃后可以恢复；`binlog` 记录服务层的逻辑变更，用于复制和按时间点恢复。

```text
Buffer Pool：数据页和索引页在内存中如何被读取、修改和缓存
undo log：修改前的版本如何回滚，以及其他事务如何读取历史版本
redo log：脏页尚未刷盘时，崩溃后如何恢复已提交修改
binlog：主从复制、逻辑恢复和变更消费如何获得这次变更
```

## 先区分四个角色

| 组件 | 所属层 | 主要作用 | 解决的问题 |
| --- | --- | --- | --- |
| Buffer Pool | InnoDB 内存 | 缓存数据页、索引页，承载日常读写 | 避免每次查询或修改都访问磁盘 |
| undo log | InnoDB | 保存旧版本 | 事务回滚、MVCC 一致性读 |
| redo log | InnoDB | 记录可重做的页修改信息 | 崩溃恢复、持久性 |
| binlog | MySQL Server | 记录逻辑变更事件 | 复制、按时间点恢复、CDC |

`redo log buffer` 和 `Buffer Pool` 不是同一个东西：前者暂存 redo 记录，后者缓存表数据与索引页。

## 一次 UPDATE 的完整链路

以这条 SQL 为例：

```sql
UPDATE article
SET status = 'published'
WHERE id = 100;
```

### 1. 定位并读取数据页

InnoDB 先通过聚簇索引定位 `id = 100` 所在的数据页。

- 如果索引页和数据页已经在 Buffer Pool，直接在内存中访问。
- 如果不在 Buffer Pool，才从磁盘把对应页读进内存。

因此，索引和热点数据越紧凑，越容易留在 Buffer Pool，查询越少发生磁盘 I/O。这也是短主键、避免无效二级索引的重要原因之一。

### 2. 生成 undo log，保留旧版本

真正修改前，InnoDB 会保留足以撤销该修改的旧版本信息。例如旧值是 `status = 'draft'`，undo 记录会支持把它恢复回来。

undo log 有两个用途：

1. 当前事务执行 `ROLLBACK` 时，根据 undo 撤销修改。
2. 并发事务做普通一致性读时，结合 `Read View` 沿 undo 版本链找到自己可见的历史版本。

所以 MVCC 不是“复制整张表”，而是通过行的隐藏字段、版本链和可见性规则读取合适的版本。

### 3. 修改 Buffer Pool 中的数据页

InnoDB 在 Buffer Pool 内把该行状态改为 `published`。此时内存页已经和磁盘上的数据文件不一致，这个内存页称为**脏页**。

脏页不会因为一条 SQL 提交就立刻写回数据文件。若每次更新都同步刷新数据页，随机 I/O 成本会非常高。

### 4. 生成 redo log，满足 WAL

修改数据页的同时，InnoDB 会生成对应的 redo 记录，先进入 redo log buffer，再在提交或其他合适时机写入 redo log 文件。

核心原则是 WAL（Write-Ahead Logging）：**在脏页被刷入数据文件之前，对应的 redo 必须已经持久化**。这样即使数据文件还没来得及更新，崩溃后也能根据 redo 重做已提交修改。

### 5. 写入 binlog，并协调提交

binlog 位于 MySQL Server 层，记录的是这次 SQL 产生的逻辑变更事件。它主要服务于：

- 主从复制；
- 基于时间点的恢复；
- CDC、审计或下游数据同步。

当 binlog 开启时，MySQL 会通过内部 XA 两阶段提交协调 redo 和 binlog，简化后的顺序是：

```text
1. InnoDB 将 redo 置为 prepare 状态
2. MySQL Server 写入并按配置刷盘 binlog
3. InnoDB 将 redo 提交为 commit 状态
4. 返回客户端提交成功
```

目的是避免“事务在主库已经提交，却没有对应 binlog”或“binlog 已经对外可见，但 InnoDB 事务没有提交”的不一致。实际实现还会通过组提交合并多事务的刷盘操作，提高吞吐。

### 6. checkpoint 后台刷脏页

事务提交后，Buffer Pool 中的脏页仍可继续留在内存。后台线程会在合适时机执行 checkpoint，把部分脏页刷回数据文件，并推进可复用的 redo 空间。

```text
事务提交成功
  != 数据页已经立刻写入磁盘

事务提交成功
  = redo 和 binlog 已达到配置要求的持久化状态
```

## 发生 ROLLBACK 时会怎样？

如果事务尚未提交并执行 `ROLLBACK`：

```text
undo log
  -> 找到修改前版本
  -> 在 Buffer Pool 中恢复旧值
  -> 相关页仍可能需要 redo 记录来保证回滚过程本身的崩溃安全
```

因此，undo 负责“把业务修改撤回去”，redo 负责“即使在回滚或提交过程中崩溃，也能让数据页恢复到正确状态”。

## 宕机后如何恢复？

| 宕机场景 | 主要依赖 | 恢复结果 |
| --- | --- | --- |
| 已提交，但脏页尚未刷入数据文件 | redo log | 重做已提交修改 |
| 事务未提交或处于恢复中的中间状态 | undo log | 回滚未完成事务 |
| 需要把主库变更同步到从库 | binlog | 从库重放逻辑事件 |
| 误操作后要恢复到某一时刻 | 全量备份 + binlog | 恢复到指定时间点 |

对于处在两阶段提交中间状态的事务，恢复过程会结合 redo 的状态和 binlog 是否存在，判断该事务应提交还是回滚。

## Buffer Pool 为什么和索引设计强相关？

Buffer Pool 缓存的是页，不是单条记录，也不是 Redis 那样的业务结果缓存。

```text
索引更小、热点数据更集中
  -> 同一块内存能容纳更多索引页和数据页
  -> Buffer Pool 命中率通常更高
  -> 磁盘读取更少

索引更多、主键更长、访问路径更分散
  -> 需要缓存的页更多
  -> 更容易发生页替换和磁盘 I/O
  -> 查询与写入成本上升
```

但索引不是越小越好。索引首先要正确服务于查询路径；在查询满足需求的前提下，再控制主键长度、冗余索引和无效索引。

## 常见误区

### 1. “redo log 就是记录 SQL”

不准确。binlog 才更接近逻辑 SQL/行变更事件；redo 面向 InnoDB 页修改和崩溃恢复，两者职责不同。

### 2. “undo log 只用于回滚”

不完整。undo 还为 MVCC 提供历史版本链，是普通一致性读的重要基础。

### 3. “提交成功后，数据一定已经刷入数据文件”

不准确。提交成功主要依赖 redo 和 binlog 的持久化策略；数据页可以稍后由 checkpoint 刷盘。

### 4. “MVCC 让所有读都不加锁”

不准确。普通一致性读通常读取快照；`SELECT ... FOR UPDATE`、`UPDATE`、`DELETE` 等当前读和写操作仍要使用锁。

### 5. “Buffer Pool 就是 Redis”

不准确。Buffer Pool 是 InnoDB 自动管理的页缓存，服务于数据库内部读写；Redis 是应用可直接使用的独立内存数据库或缓存系统。

## 面试推荐回答

> 一次更新时，InnoDB 会先把相关数据页读入 Buffer Pool。修改前通过 undo log 保留旧版本，用于回滚和 MVCC；随后在 Buffer Pool 中修改数据页，页面变为脏页。与此同时生成 redo log，遵循 WAL，保证脏页刷入数据文件前 redo 已经持久化，因此宕机后能够恢复已提交修改。MySQL Server 层还会写 binlog，用于主从复制和按时间点恢复。提交后，后台 checkpoint 再逐步把 Buffer Pool 的脏页刷入数据文件。简单说，Buffer Pool 管日常读写，undo 管回滚和历史版本，redo 管崩溃恢复，binlog 管复制和逻辑恢复。

## 记忆口诀

```text
数据先在 Buffer Pool 改，
旧值交给 undo 来管；
已提交怕宕机靠 redo，
复制恢复再看 binlog。
```
