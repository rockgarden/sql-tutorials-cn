# PG

## Parallel Workers

### 配置实例

#### 资源分析

```bash
lscpu

架构：           x86_64
CPU 运行模式：   32-bit, 64-bit
字节序：         Little Endian
CPU:             8
在线 CPU 列表：  0-7
每个核的线程数： 1
每个座的核数：   1
座：             8
NUMA 节点：      1
厂商 ID：        GenuineIntel
BIOS Vendor ID:  GenuineIntel
CPU 系列：       6
型号：           85
型号名称：       Intel(R) Xeon(R) Gold 5218 CPU @ 2.30GHz
BIOS Model name: Intel(R) Xeon(R) Gold 5218 CPU @ 2.30GHz
步进：           7
CPU MHz：        2299.999
BogoMIPS：       4599.99
超管理器厂商：   VMware
虚拟化类型：     完全
L1d 缓存：       32K
L1i 缓存：       32K
L2 缓存：        1024K
L3 缓存：        22528K
NUMA 节点0 CPU： 0-7
标记：           fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush mmx fxsr sse sse2 ss syscall nx pdpe1gb rdtscp lm constant_tsc arch_perfmon nopl xtopology tsc_reliable nonstop_tsc cpuid pni pclmulqdq ssse3 fma cx16 pcid sse4_1 sse4_2 x2apic movbe popcnt tsc_deadline_timer aes xsave avx f16c rdrand hypervisor lahf_lm abm 3dnowprefetch invpcid_single ssbd ibrs ibpb stibp ibrs_enhanced fsgsbase tsc_adjust bmi1 avx2 smep bmi2 invpcid avx512f avx512dq rdseed adx smap clflushopt clwb avx512cd avx512bw avx512vl xsaveopt xsavec xgetbv1 xsaves arat pku ospke avx512_vnni md_clear flush_l1d arch_capabilities

free -h
              total        used        free      shared  buff/cache   available
Mem:           31Gi       209Mi        30Gi        20Mi       425Mi        30Gi
Swap:         5.0Gi          0B       5.0Gi
```

这是一台虚拟机：运行在 VMware 虚拟化平台上，是非常经典的 “8核 32G” 中小型数据库/应用服务器标准配置。

- CPU配置：分配了 8 个 vCPU（逻辑核），主频 2.3GHz。拓扑结构上，它被识别为 8 个物理座（Sockets），每个座 1 个核心，没有超线程。
  - 宿主机硬件：底层物理服务器使用的是 Intel Xeon Gold 5218（这是英特尔第二代至强可扩展处理器，属于中高端型号）。
    - Gold 5218 是一款 16 核 32 线程的强力 CPU，支持 AVX-512 高级指令集（在最后的“标记”中可以看到 `avx512f` 等）。这意味着如果你的 PG 查询涉及复杂的数学计算或开启了 JIT（即时编译），底层硬件能提供很好的加速。
  - 架构优势：在系统内部表现为 单 NUMA 节点，不需要在操作系统层面配置复杂的 `numactl` 内存绑定策略，这对运行 PostgreSQL 等数据库非常友好，没有跨节点内存访问的性能损耗。
- 内存配置
  - total 31Gi：系统总物理内存。虚拟机分配了 32GB，内核和硬件保留了一小部分，所以显示 31Gi。
  - used 209Mi / available 30Gi：当前实际被进程占用的内存极少（不到 250MB），有 30GB 随时可以分配给新进程（如 PostgreSQL）。
  - buff/cache 425Mi：Linux 会把空闲内存拿来做磁盘缓存以加速系统。因为这台机器刚启动或没跑业务，所以缓存还没建立起来。这是正常现象，当 PG 运行一段时间后，这部分数值会大幅增加。
  - Swap 5.0Gi / used 0B：配置了 5GB 的虚拟内存（Swap），当前完全没有使用。

#### 参数调优

1. 环境优化

    在数据库服务器中，Swap 是性能杀手。
    一旦 PostgreSQL 的内存使用达到峰值，操作系统将 PG 的部分内存交换到磁盘（Swap）上，原本在内存中只需几毫秒的查询，会因为磁盘 I/O 变成几秒甚至几十秒，导致数据库出现严重的“假死”或连接超时。

    ```bash
    # 降低 Swap 使用倾向（推荐，保留 Swap 作为防 OOM 的最后底线）
    # 临时生效（立即执行）
    sysctl vm.swappiness=1

    # 永久生效
    echo "vm.swappiness=1" >> /etc/sysctl.conf
    sysctl -p

    # 彻底关闭 Swap（激进做法，适合有专人监控内存的 DBA）
    
    # 立即关闭 Swap
    swapoff -a

    # 永久关闭（注释掉 /etc/fstab 中的 swap 行）
    sed -i '/swap/s/^/#/' /etc/fstab
    ```

    CPU 超分（Overcommit）风险：因为这是 VMware 虚拟机，如果宿主机上的其他虚拟机也在疯狂消耗 CPU，你的 PG 性能会波动。如果发现 PG 经常卡顿，但 `top` 显示 CPU 没跑满，可能是宿主机超分严重导致的“CPU 就绪时间（CPU Ready Time）”过高，需要找虚拟化管理员排查。

    电源策略：确保 VMware 宿主机和虚拟机的电源策略设置为 “高性能 (High Performance)”，防止 CPU 在低负载时降频，导致 PG 查询出现偶发的延迟毛刺。

    若是PG设置了 `huge_pages = try`，在 Linux 上，仅仅在 PG 里设置是不够的，你必须在操作系统层面预留足够的大页内存（HugePages），否则 PG 启动时会回退到普通内存，失去性能优势。

    - 查看当前系统大页大小：`grep Hugepagesize /proc/meminfo` (通常是 2048 kB)。
    - 计算所需大页数：$8GB / 2MB = 4096$ 页。建议多留一点，设为 4100 页。
    - 修改 `/etc/sysctl.conf`：

        ```bash
        vm.nr_hugepages = 4100
        ```

    - 生效并验证：

        ```bash
        sysctl -p
        grep HugePages_Total /proc/meminfo
        ```

    - *如果 `HugePages_Total` 大于 0，说明配置成功。*

2. postgresql.conf

    连接数据库后，确认pg版本大于14（`SELECT version();`），定位配置路径（`SHOW config_file;`）。

    postgresql.conf 配置原则：

    - CPU需要在“利用并行加速查询”和“保留资源给并发连接”之间找平衡；
    - 内存需要在“让 PG 尽可能多用内存”和“给操作系统留足余地”之间找到平衡；

    `PostgreSQL 16.14 on x86_64-pc-linux-gnu, compiled by gcc (GCC) 8.5.0 20210514 (Red Hat 8.5.0-28), 64-bit`

    ```ini
    # 总后台进程上限：建议设为 10~16
    # 确保这个值足够大，以支撑并行 Worker 和其他后台进程（如逻辑复制、Autovacuum、walwriter）
    # (8个并行worker)
    max_worker_processes = 16

    # 系统最大并行 worker 数：建议设为 4 到 6
    # (必须留出至少2个核给常规的非并行查询和系统维护，否则系统会卡死)
    # PG16 设为与 CPU 核数相等，让系统充分利用算力
    max_parallel_workers = 8

    # 单次 Gather 最多使用的 worker 数：建议设为 3 或 4
    # (对于单条大查询，用3-4个核加速即可。设太大不仅收益递减，还会挤占其他查询的资源)
    max_parallel_workers_per_gather = 4

    # 核心共享内存：建议 8GB
    # PG 自己管理的内存池，用来缓存数据页，业界经验值是总内存的 25%；不建议设置得更大（如超过 12G），否则会导致操作系统自身缺乏内存来缓存文件系统，反而降低整体 I/O 性能。
    shared_buffers = 8GB

    # 优化器估算内存：建议值`24GB`
    # 这个参数不实际分配内存，只是告诉 PG 的查询优化器：“系统总共有多少内存（包含 PG 的 shared_buffers 和 OS 的 page cache）可以用来缓存数据”。通常设置为总内存的 50% 到 75%。设置得越大，优化器越倾向于使用“索引扫描”而不是“顺序扫描”。
    effective_cache_size = 24GB

    # 排序与哈希内存：建议值 `32MB` 到 `64MB`（建议先设为 `32MB`）
    # 原理解释：用于 `ORDER BY`、`DISTINCT`、哈希连接等操作的内存。
    # 避坑指南：这个内存是按“每个操作”计算的，不是按“每个连接”！ 如果一个查询有 3 个排序节点，且当前有 100 个并发连接，瞬间消耗的内存是 `100 * 3 * work_mem`。如果设得太大（比如 1GB），极易导致内存溢出（OOM）或疯狂使用 Swap。
    # PG 16 的排序算法更高效，可以适当从 32MB 提升到 64MB，如果遇到极个别的复杂报表查询很慢，可以在该会话中临时执行 `SET work_mem = '256MB';` 来加速。
    work_mem = 64MB

    # 维护操作内存：建议值：`1GB` 或 `2GB`
    # 用于 `VACUUM`、`CREATE INDEX`、`ALTER TABLE ADD FOREIGN KEY` 等维护操作。因为这些操作通常不会高并发同时执行，所以可以大胆分配大一点，能显著加快建索引和清理死元组的速度。
    maintenance_work_mem = 2GB

    # WAL 缓冲区：建议值 `64MB`
    # 用于缓存未写入磁盘的 WAL（预写式日志）数据。默认通常是 -1（自动计算为 shared_buffers 的 1/32，即 256MB）。对于 32G 内存的机器，设置 `64MB` 已经足够，足以应对高并发下的日志写入峰值，再大对性能的提升微乎其微，反而浪费了宝贵的共享内存。
    # 只有当你的服务器拥有极大的内存（比如 128GB+），且 shared_buffers 设置得非常大（比如 32GB+），同时你观察到系统在极高并发写入时存在明显的 WAL 写入等待（可以通过监控 pg_stat_bgwriter 中的 buffers_checkpoint 和 buffers_clean 指标判断），才考虑手动将其调大到 64MB 或更高。
    # wal_buffers = 64MB


    # PG 16 强烈建议开启。尝试使用操作系统的 HugePages，能减少 TLB Miss，提升大内存下的性能。
    huge_pages = try

    # 开启 JIT (即时编译)
    # PG 16 的 JIT 编译已经非常成熟，当业务包含大量的复杂计算或长查询时启用。
    jit = on
    # 只有代价超过 10万的查询才启用 JIT，避免小查询的编译开销。
    jit_above_cost = 100000  

    # 增强日志信息量
    # 面向审计和排查：时间和进程 ID、用户、数据库和客户端 IP。
    log_line_prefix = '%m [%p] %u@%d from %h '

    # 开启慢查询日志
    # 建议：开启它来捕获执行超过 5 秒的 SQL，这是性能调优的核心依据。
    log_min_duration_statement = 5000  # 单位毫秒，记录超过1秒的查询

    max_connections = 100
    # 建议：对于 8 核 32G 的机器，100 个连接是合理的。但如果应用使用连接池（如 PgBouncer），可以将此值设得更大（如 200-300），以应对突发流量。如果没有连接池，保持 100 即可，防止过多连接耗尽内存。

    # WAL 上限太小了。这会导致 PostgreSQL 过于频繁地触发 Checkpoint，将 shared_buffers 中的脏页强行刷入磁盘，从而产生不必要的随机 I/O 峰值，影响查询性能。
    # 对于 32GB 内存
    max_wal_size = 4GB
    min_wal_size = 1GB  
    # 建议与 max_wal_size 保持一定比例，避免频繁伸缩
    ```

## 单独重启任意一台从库（Slave/Standby）

在 PostgreSQL 的一主多从（流复制）架构中，单独重启任意一台从库（Slave/Standby）是标准且安全的日常维护操作，不会对主库和其他从库造成任何影响。

为什么可以单独重启？

- 进程独立：主库和每个从库在操作系统层面都是完全独立的 PostgreSQL 进程。重启从库只是杀死了该从库的进程并重新启动，主库的进程和其他从库的进程完全不受干扰。
- 流复制自动恢复：从库重启后，它的后台进程（Startup process 和 WalReceiver）会自动启动。它会重新读取 `recovery.conf`（PG 12+ 中为 `postgresql.conf` 和 `standby.signal`）中的配置，自动重新连接到主库，并请求从断开点之后的 WAL 日志，继续追赶数据。这个过程是全自动的。

虽然底层数据库集群是安全的，但重启从库会对业务请求产生短暂影响，具体取决于你的应用架构：

- 如果有负载均衡/读写分离中间件（如 Pgpool-II, HAProxy, Patroni, 或应用层的读写分离）：
  中间件会检测到该从库宕机，自动将发往该从库的读请求平滑地切换到另外一台健康的从库上。业务完全无感知（零停机）。
- 如果应用直连了这台从库的 IP（没有负载均衡）：
  在从库重启的几十秒内，发往该 IP 的读请求会报错或超时。写请求不受影响（因为写请求是发给主库的）。

在需要重启的那台从库服务器上执行重启从库：

```bash
# 确认是由 systemd 管理
systemctl list-units --type=service | grep postgres
  postgresql-16.service                                 loaded active running PostgreSQL 16 database server

sudo systemctl restart postgresql-16  # 请替换为你的实际服务名

```

从库启动后，登录到该从库的 `psql`，执行以下 SQL 确认它已经重新连上主库并在同步数据：

```sql
-- 1. 检查 WalReceiver 进程是否正在运行（Status 应为 streaming）
SELECT status, receive_start_lsn, written_lsn, flushed_lsn, replayed_lsn 
FROM pg_stat_wal_receiver;

-- 2. 检查主从延迟（如果返回的字节数很小或为0，说明已经追上主库）
SELECT pg_wal_lsn_diff(pg_last_wal_receive_lsn(), pg_last_wal_replay_lsn()) AS replay_lag_bytes;
```

查看从库的 PostgreSQL 日志文件（通常在 `log_directory` 下），确保没有报错，并且能看到类似以下的成功连接日志：

```text
LOG:  started streaming WAL from primary at X/X on timeline Y
```

利用此特性实现“零停机”修改配置

如果你修改了 `postgresql.conf` 中的某些参数（比如 `shared_buffers` 或 `max_connections`），这些参数必须重启数据库才能生效。

在一主两从的架构下，你可以利用“单独重启从库”的特性，实现整个集群的滚动重启（Rolling Restart），从而做到业务零停机：

1. 修改配置：将修改好的 `postgresql.conf` 同步到主库和两台从库。
2. 重启从库 A：单独重启从库 A。此时业务流量由主库和从库 B 承担。
3. 重启从库 B：等从库 A 追上主库后，单独重启从库 B。此时业务流量由主库和从库 A 承担。
4. 主备切换 (Switchover)：将主库降级为从库，将刚才重启过的从库 A（或 B）提升为新主库。
5. 重启原主库：此时原来的主库变成了从库，单独重启它。

通过这种“滚动”的方式，你可以在不中断任何业务的情况下，完成整个集群需要重启才能生效的参数更新。
