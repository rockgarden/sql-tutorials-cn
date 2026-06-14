# 特征

## Parallel Query（并行查询）

是 PG 10 引入的一项重大性能增强特性，并在后续版本（PG 11~16）中不断扩展和完善。它的核心目标是：利用单台服务器的多个 CPU 核心，协同完成单个复杂查询的计算任务，从而大幅降低查询延迟。

### 工作原理

PG 的并行查询采用 Leader-Worker（领导者-工作者） 模型，基于 动态共享内存（Dynamic Shared Memory, DSM） 进行进程间通信。

1. Leader 进程：执行查询的后端进程（即客户端连接的那个进程）。它负责解析 SQL、生成执行计划，并在需要时启动 Worker 进程。Leader 自己也会参与部分计算工作。
 is the backend process that handles the client connection.
2. Worker 进程：由 Leader 动态派生出的后台进程（`parallel worker`）。它们从共享内存中读取任务，执行部分计算（如扫描部分数据块），并将中间结果写回共享内存。
3. Gather / Gather Merge 节点：执行计划中的关键节点。Leader 进程通过 `Gather` 节点收集所有 Worker 产生的中间结果，并进行最终的合并或汇总，然后返回给客户端。

> 本质：它是单机多核并行（SMP 并行），不是多机分布式并行。

### 支持并行的操作类型

| 操作类型         | 支持的并行方式                                                      | 引入版本 / 说明                                                               |
| :--------------- | :------------------------------------------------------------------ | :---------------------------------------------------------------------------- |
| 顺序扫描         | `Parallel Seq Scan`                                                 | PG 10+。最基础的并行，将表的数据块（Blocks）划分给多个 Worker 扫描。          |
| 索引扫描         | `Parallel Index Scan`、`Parallel Index Only Scan`                   | PG 11+。多个 Worker 协同扫描同一个 B-Tree 索引的不同部分。                    |
| 位图扫描         | `Parallel Bitmap Heap Scan`                                         | PG 10+。Worker 协同构建位图，然后协同扫描堆表。                               |
| 表连接 (Join)    | `Parallel Hash Join`、`Parallel Merge Join`、`Parallel Nested Loop` | PG 11 极大增强了 Parallel Hash Join，现在是并行查询性能提升最明显的场景之一。 |
| 聚合 (Aggregate) | `Partial Aggregate` → `Finalize Aggregate`                          | PG 10+。Worker 先做局部聚合（如局部 COUNT/SUM），Leader 再做最终聚合。        |
| 排序 (Sort)      | `Parallel Sort` (通过 Gather Merge)                                 | PG 10+。Worker 各自排序，Leader 使用多路归并（Merge）合并结果。               |
| 其他             | `Parallel Append` (分区表)、`Parallel Create Table As`              | PG 10+ / PG 12+。并行扫描多个分区，或并行写入新表。                           |

### 配置参数

PG 通过代价模型（Cost-based Optimizer）自动决定是否使用并行。以下参数控制并行的“开关”和“规模”：

```sql
-- 1. 规模控制（最关键）
SHOW max_worker_processes;             -- 系统允许的最大后台进程总数 (默认 8)
SHOW max_parallel_workers;             -- 系统允许的最大并行 worker 总数 (默认 8)
SHOW max_parallel_workers_per_gather;  -- 单个 Gather 节点最多使用的 worker 数 (默认 2) ⭐

-- 2. 触发门槛（表必须足够大才考虑并行）
SHOW min_parallel_table_scan_size;     -- 触发并行顺序扫描的最小表大小 (默认 8MB)
SHOW min_parallel_index_scan_size;     -- 触发并行索引扫描的最小索引大小 (默认 512kB)

-- 3. 代价模型（影响优化器的决策）
SHOW parallel_setup_cost;              -- 启动并行进程的代价 (默认 1000)
SHOW parallel_tuple_cost;              -- Worker 向 Leader 传递一行数据的代价 (默认 0.1)
```

调优建议：如果你的服务器是 32 核，不要简单地把 `max_parallel_workers_per_gather` 设为 32。这会导致单个查询耗尽所有 CPU，阻塞其他并发查询。推荐设置为 CPU 总核心数的 1/4 到 1/2（例如 32 核设为 8 或 16）。

### 常见问题

为什么配置了却不触发并行？（）

即使参数配置正确，PG 优化器在以下情况会主动放弃并行查询：

1. 表太小：表大小低于 `min_parallel_table_scan_size`，优化器认为串行更快（启动 Worker 的开销 > 收益）。
2. 事务中已有写操作：如果在当前事务中已经执行了 `INSERT/UPDATE/DELETE`，PG 会禁用该事务后续查询的并行（为了保证数据一致性视图）。
3. 使用了游标 (Cursor)：`DECLARE cursor FOR ...` 不支持并行。
4. 调用了“非并行安全”的函数：如果查询中调用了标记为 `PARALLEL UNSAFE` 的自定义函数或某些系统函数（如 `currval()`, `setval()`），并行会被禁用。
5. 隔离级别为 SERIALIZABLE：可串行化隔离级别下不支持并行查询。
6. 使用了某些特定的执行计划节点：如 `LIMIT` 没有配合 `ORDER BY`，或者使用了 `Locking clause` (`FOR UPDATE`)。
7. 统计信息过期：如果表没有 `ANALYZE`，优化器可能误判表很小，从而不选择并行。

### 最佳实践与调优指南

1. 保持统计信息最新：
   定期执行 `ANALYZE your_table;` 或配置 autovacuum。优化器严重依赖 `pg_class.reltuples` 和 `relpages` 来决定是否并行。
2. 善用 `EXPLAIN (ANALYZE, BUFFERS)`：
   不要只看 `EXPLAIN`，一定要加 `ANALYZE` 看实际执行时间，加 `BUFFERS` 看是否真的减少了 I/O 等待。关注 `Workers Launched` 是否大于 0。
3. 针对特定查询强制并行（调试用）：
   如果优化器“犯傻”没选并行，可以临时在当前会话强制开启，验证并行是否能带来收益：

   ```sql
   SET force_parallel_mode = on; 
   -- 或 SET force_parallel_mode = regress; (用于测试并行代码路径)
   ```

4. 避免在 OLTP 核心链路滥用：
   并行查询会消耗大量 CPU 和共享内存。它最适合 OLAP 场景（如夜间报表、复杂分析、大表全表扫描）。对于高并发的简单点查（OLTP），并行反而会因为进程切换和 DSM 通信导致性能下降。
5. 注意 `work_mem` 的放大效应：
   在并行查询中，每个 Worker 都会独立分配 `work_mem`。如果一个查询启动了 4 个 Worker，且都涉及 Hash Join 或 Sort，那么该查询消耗的内存将是 `work_mem * 4`。需确保系统总内存充足，避免 OOM。
