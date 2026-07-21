# 内容采集与检索平台数据库面试题库 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 创建一份基于假设内容采集与检索平台的数据库项目面试题库。

**Architecture:** 新文档以场景说明开篇，再按架构设计、事务并发、索引查询、缓存扩展、日志恢复和方案取舍组织问答。每个问题使用“推荐回答 + 追问边界”格式，确保技术结论和场景假设分离。

**Tech Stack:** Markdown、MySQL InnoDB、Redis、对象存储、Milvus。

---

### Task 1: 创建项目场景和数据流

**Files:**
- Create: `数据库/内容采集与检索平台-数据库面试项目题库.md`

- [ ] **Step 1: 写入场景声明**

在文档开头写明以下内容：该平台和其中的规模、故障均是面试演练假设，不应表述为真实项目经历；平台包含采集、清洗、去重、分块、向量化、检索六个阶段。

- [ ] **Step 2: 写入数据职责划分**

明确 MySQL 保存业务元数据和任务状态，Redis 保存短期缓存与幂等标记，对象存储保存原始正文与冷日志，Milvus 保存向量和向量检索所需的标量字段。

- [ ] **Step 3: 校验边界**

检查场景中没有编造具体的 QPS、文章数、成本、事故损失或线上指标。

### Task 2: 编写核心项目问答

**Files:**
- Modify: `数据库/内容采集与检索平台-数据库面试项目题库.md`

- [ ] **Step 1: 编写架构和表设计问答**

覆盖数据为何分层保存、文章去重键、文章状态建模、主键选择、物理外键的使用边界。

- [ ] **Step 2: 编写事务与并发问答**

覆盖重复投递、幂等写入、任务状态迁移、乐观锁与悲观锁的选择、死锁排查和隔离级别选择。

- [ ] **Step 3: 编写索引与查询问答**

覆盖联合索引设计、深分页、慢 SQL 排查、覆盖索引和索引失效的判断方法。

- [ ] **Step 4: 编写缓存与扩展问答**

覆盖 Cache Aside、缓存失效后的并发回源、读写分离条件、分区与分库分表的选择边界。

- [ ] **Step 5: 编写日志、恢复与检索问答**

覆盖业务数据与日志分离、MySQL redo/undo/binlog 的职责、归档策略、MySQL 与 Milvus 数据一致性和失败补偿。

### Task 3: 完成技术和结构校验

**Files:**
- Modify: `数据库/内容采集与检索平台-数据库面试项目题库.md`

- [ ] **Step 1: 核对 MySQL 机制表述**

确保文档区分 MVCC 快照读与当前读；不将乐观锁描述成可重复读方案；不将固定行数或流量写成拆库阈值。

- [ ] **Step 2: 核对问答可复述性**

每个回答先给结论，再说明原因和工程代价；每个问题保留一个能展开的追问边界。

- [ ] **Step 3: 执行文本检查**

Run: `rg -n '大厂.*禁止|绝对|一定|千万级|TODO|TBD' '数据库/内容采集与检索平台-数据库面试项目题库.md'`

Expected: 不出现未经限定的绝对化工程结论，也不出现占位文本。
