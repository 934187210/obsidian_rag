# InnoDB 索引、MVCC、锁与日志：一次订单更新全链路

## 先建立一条主线

理解 InnoDB，重点不是分别背诵“索引、锁、MVCC、redo log”，而是回答同一个问题：**一笔订单被修改时，数据如何被定位、并发事务各自能看见什么、宕机后又如何保证结果正确？**

本文用一次订单状态更新串起这条链路。已经理解某个局部概念时，可继续阅读：[[MVCC、Read View 与 undo log 串联]] 和 [[InnoDB Buffer Pool、redo log、undo log 与 binlog 串联]]。

假设订单表如下：

```sql
CREATE TABLE orders (
  id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  user_id BIGINT UNSIGNED NOT NULL,
  status TINYINT NOT NULL,
  amount DECIMAL(10, 2) NOT NULL,
  created_at DATETIME NOT NULL,
  PRIMARY KEY (id),
  KEY idx_user_status_created (user_id, status, created_at)
) ENGINE = InnoDB;
```

一条更新的简化路径：

```text
SQL
  -> 通过索引定位记录
  -> 在 Buffer Pool 中修改页、保留 undo 旧版本
  -> 记录 redo，并在提交时协调 binlog
  -> 返回提交成功
  -> 后台 checkpoint 刷脏页，purge 清理无用旧版本
```

## 1. 起点：数据实际上放在哪棵树里？

InnoDB 的主键索引通常是**聚簇索引**。它的叶子节点保存完整的订单行，因此按主键查询能直接得到 `user_id`、`status`、`amount` 等字段。

二级索引 `idx_user_status_created` 则可以概括为保存：

```text
(user_id, status, created_at, id)
```

其中 `id` 是主键值。InnoDB 会把主键列附加到二级索引记录中；若主键是联合主键，就会附加所有主键列。

这就是主键设计会影响所有二级索引的原因：主键越长，每个二级索引记录越大，同样大小的 Buffer Pool 能缓存的索引页就越少；写入时也需要维护更多字节。

`AUTO_INCREMENT BIGINT` 常被用于内部主键，不是因为它“绝对最好”，而是它通常短、稳定，并且新记录倾向持续写入 B+ 树右侧附近，局部性较好。完全随机且较长的 UUID 会让插入更分散，更容易产生页分裂与碎片。是否选 UUID 仍要结合分布式生成、对外暴露 ID 和写入压力决定。

## 2. 查询：什么时候会回表？

```sql
SELECT id, status, created_at
FROM orders
WHERE user_id = 10 AND status = 1;
```

这三个字段可从 `idx_user_status_created` 获得：索引已经覆盖查询所需内容，不必再找聚簇索引。

但下面的 SQL 需要 `amount`：

```sql
SELECT amount
FROM orders
WHERE user_id = 10 AND status = 1;
```

执行过程是：

```text
二级索引找到匹配项和主键 id
  -> 使用 id 到聚簇索引取完整行
  -> 读取 amount
```

第二步就是常说的**回表**。少量结果的回表很正常；如果范围查询返回大量记录，反复回表会放大随机页访问和 Buffer Pool 压力。

优化不等于给每条查询都添加覆盖索引。可以把 `amount` 放进更宽的联合索引以避免回表，但这个索引会占更多空间，并使每次 `INSERT`、`UPDATE`、`DELETE` 的维护成本更高。应先以真实 SQL、数据分布和 `EXPLAIN` 验证收益。

## 3. 事务 T1 更新时，先发生什么？

事务 T1 执行：

```sql
START TRANSACTION;

UPDATE orders
SET status = 2
WHERE id = 1001;
```

`id` 是主键，InnoDB 可以精准定位聚簇索引中的记录。若对应索引页或数据页不在 Buffer Pool，先从磁盘读入；随后在内存中修改，数据页成为脏页。

由于 `status` 同时属于二级索引 `idx_user_status_created`，这次更新还要维护该二级索引。概念上，原来的二级索引项会失效，新状态对应的新索引项会出现；这也是索引越多，写入越贵的直接原因。

更新并不是“只锁业务代码里的一行对象”，而是会锁住执行路径上遇到的**索引记录**。通过主键等值更新时，范围通常很小；若改成没有合适索引的条件：

```sql
UPDATE orders
SET status = 2
WHERE amount > 1000;
```

InnoDB 可能需要扫描和锁定更多记录，并发能力会明显下降。范围当前读在默认可重复读隔离级别下，还可能涉及间隙锁或临键锁，避免其他事务在范围中插入会影响当前读结果的新记录。

## 4. MVCC：另一个事务为什么不一定要等待？

T1 改完 `status`，但还没提交。此时事务 T2 做普通查询：

```sql
SELECT status
FROM orders
WHERE id = 1001;
```

为了让 T2 不必读取 T1 的未提交值，也不必一律等待，InnoDB 会保留旧版本信息：

```text
聚簇索引当前版本：status = 2，最近修改者是 T1
                       |
                       | roll_ptr
                       v
undo 中的旧版本：status = 1
```

T2 的普通 `SELECT` 是一致性读。它根据自己的 **Read View** 判断当前版本是否可见：

- T1 尚未提交：T1 的版本不可见，T2 沿 undo 版本链读到 `status = 1`。
- T1 已提交、但提交发生在 T2 的快照之后：在默认 `REPEATABLE READ` 下，T2 的同一事务通常仍读到旧版本。
- 若 T2 使用 `READ COMMITTED`：每次普通一致性读通常建立新的 Read View，下一次读取可能看到 T1 已提交的新值。

MVCC 的作用可以精炼为：**undo 提供历史版本，Read View 决定哪个版本可见，普通读因此可与写操作并发。**

但 MVCC 不是“所有读取都不加锁”。如果 T2 要基于最新数据继续修改：

```sql
SELECT status
FROM orders
WHERE id = 1001
FOR UPDATE;
```

这属于当前读。它必须获取最新可用版本并锁住相应索引记录，所以会等待 T1 提交或回滚。`UPDATE`、`DELETE` 也属于这一类需要锁协调的操作。

## 5. undo、redo、binlog：同一条更新的三种不同记录

T1 修改订单时，三个名字相近的日志服务于完全不同的目标：

| 机制 | 主要职责 | 在这次更新中的作用 |
| --- | --- | --- |
| undo log | 回滚、MVCC 历史版本 | 保留 `status = 1`，用于回滚或给旧快照读取 |
| redo log | 崩溃恢复、持久性 | 记录可重放的 InnoDB 修改，保护尚未刷盘的脏页 |
| binlog | 复制、逻辑恢复、CDC | 对外提供这次逻辑变更事件 |

完整过程可以这样理解：

1. 生成 undo，确保需要时可撤销修改，并让其他事务可构造旧版本。
2. 在 Buffer Pool 修改聚簇索引页及受影响的二级索引页。
3. 生成 redo。它遵循 WAL 原则：数据页刷入数据文件前，对应 redo 必须已持久化。
4. 提交时，MySQL Server 层写 binlog，并与 InnoDB 的 redo 提交协调，避免数据提交状态与复制日志状态不一致。
5. 是否在提交时真正同步刷盘，仍受 `innodb_flush_log_at_trx_commit` 和 `sync_binlog` 等持久化配置影响。

因此不能说“事务提交成功等于数据页已经写进表空间文件”。更准确的说法是：提交成功后，redo 与 binlog 达到当前配置所要求的持久化状态；数据页可以继续作为 Buffer Pool 的脏页，稍后再刷盘。

## 6. checkpoint 与 purge：提交之后的后台收尾

`COMMIT` 返回成功，不表示所有后续工作都已完成。

**checkpoint** 会逐步把 Buffer Pool 的脏页写回数据文件，并推进恢复起点。这样，较早的 redo 空间可以复用；发生崩溃时，也不必从无限久远的位置开始恢复。

**purge** 则负责清理不再被任何 Read View 需要的历史版本，以及可以清理的删除标记记录。它解释了长事务的危险：

```text
长事务一直不结束
  -> 旧 Read View 一直存在
  -> 旧 undo 版本不能及时清理
  -> 历史链、undo 空间和 purge 压力不断增长
```

所以事务应尽量只包住必要的数据库操作；不要把网络请求、用户等待、文件下载或大规模计算放在长事务中。

## 7. 如果机器突然宕机？

假设 T1 已经收到提交成功，但对应脏页还在 Buffer Pool，没有写回数据文件。

重启后，InnoDB 从 checkpoint 之后扫描 redo，重放需要恢复的修改；对于崩溃时未提交的事务，则根据 undo 回滚。最终数据回到一致状态。

```text
已提交、数据页尚未落盘
  -> redo 重做

未提交或需要撤销
  -> undo 回滚

需要复制到从库、按时间点恢复或供 CDC 消费
  -> binlog 重放逻辑变更
```

这就是三者不能互相替代的原因：redo 面向 InnoDB 崩溃恢复，undo 面向回滚与版本，binlog 面向复制和逻辑恢复。

## 8. 用这条链路指导日常设计

1. **先按访问路径设计索引。** 查询条件、排序和返回字段决定索引；不要仅因为一个字段“经常出现”就建索引。
2. **让主键短、稳定，并匹配写入模式。** 主键会出现在二级索引中；有序写入通常更友好，但不是业务唯一约束的替代品。
3. **控制事务边界。** 索引决定锁的定位范围，事务时长决定锁保持多久。
4. **区分普通读和当前读。** 需要稳定视图时用一致性读；需要读取最新值并修改时，用锁定读或条件更新保证正确性。
5. **把长事务当作容量与并发风险。** 它既可能增加锁等待，也会阻碍 undo purge。
6. **不要只看查询耗时。** 索引优化还要观察写放大、Buffer Pool 命中、锁等待、磁盘空间和备份恢复窗口。

## 面试中的 60 秒回答

> InnoDB 以主键聚簇索引保存完整行，二级索引保存二级键和主键值，所以非覆盖查询需要通过主键回表。一次更新会先通过索引定位并锁定相关记录，在 Buffer Pool 中修改数据页；undo 保留旧版本，供回滚和 MVCC 一致性读使用，Read View 决定普通查询能看到哪个版本。redo 保证已提交但尚未刷入数据文件的修改可以在宕机后恢复，binlog 负责复制和逻辑恢复。提交后 checkpoint 逐步刷脏页并推进 redo 复用，purge 在没有旧事务需要历史版本时清理 undo。索引是否命中不仅影响查询速度，也决定锁范围、内存命中率和写入成本。

## 官方参考

- [InnoDB 索引扩展与二级索引中的主键列](https://dev.mysql.com/doc/refman/8.0/en/index-extensions.html)
- [InnoDB 一致性读与 Read View](https://dev.mysql.com/doc/refman/8.0/en/innodb-consistent-read.html)
- [InnoDB 锁与临键锁](https://dev.mysql.com/doc/refman/8.0/en/innodb-locking.html)
- [锁定读](https://dev.mysql.com/doc/refman/8.0/en/innodb-locking-reads.html)
- [InnoDB 崩溃恢复](https://dev.mysql.com/doc/refman/8.0/en/innodb-recovery.html)
