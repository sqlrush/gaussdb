# WAL Writer 并行写架构跨数据库对比

> 对比 Oracle / MySQL / PostgreSQL / GaussDB 在 WAL（redo log）写入线程模型、
> fsync 处理、SCN/LSN 顺序保证等方面的设计差异。
> 背景是农行 Dorado 双集群 sync 同步下单线程 WAL 写 1000 IOPS 天花板问题的横向调研。

## 一、总览对比

| 维度 | Oracle 12.2+ | MySQL 8.0+ | PostgreSQL 17 | GaussDB |
|---|---|---|---|---|
| WAL 写入线程模型 | LGWR + N worker (1 + N 并行) | 4 专职线程（流水线） | 单 walwriter | 单 walwriter（继承 PG） |
| 日志缓冲插入 | M 条 strand 并行 | lock-free ring buffer | 8 个 insert lock | 8 个 insert lock |
| 物理 IO 并发 | ✅ 多 worker | ⚠️ 单 writer（流水线隐藏延迟） | ❌ 单线程 | ❌ 单线程 |
| fsync 并发 | ✅ 多路 | ⚠️ 单 flusher | ❌ 单路 | ❌ 单路 |
| IO 模型 | ASM 裸设备 / O_DSYNC | buffered + fsync | buffered + fsync | buffered + fsync |
| 核心瓶颈 | 存储 IO 上限 | 单盘 IO | 单线程串行 IO | 单线程串行 IO |

## 二、演进时间线

```
Oracle:
  8i/9i/10g/11g   单 LGWR
  12.1            单 LGWR + adaptive log file sync
  12.2 (2016) ⭐   Scalable LGWR: LGWR coordinator + N LGnn worker
  19c             默认启用，成熟
  21c / 23c       进一步优化

MySQL:
  5.7 及以前       log_sys 全局 mutex（极度争用）
  8.0 (2018) ⭐   WL#10310: lock-free log buffer + 4 专职线程
  8.0+           innodb_log_writer_threads 参数

PostgreSQL:
  任何版本         单 walwriter + 8 insert locks
  社区多次讨论      2019/2022 都有 POC 但未进主线
  PG 18 (开发中)   io_method=aio（异步 IO，仍单 writer）
  → 可预见未来保持单线程
```

## 三、Oracle Scalable LGWR 架构细节

### 进程模型

```
 Session1  Session2  ...  SessionN
     │         │              │
     ▼         ▼              ▼
  Strand1   Strand2    ...  StrandM
  (log buffer 分 M 条并行区段)
     │         │              │
     └─────────┼──────────────┘
               ▼
      ┌────────────────────┐
      │ LGWR (coordinator) │
      │ 分派任务 + 序号     │
      └──┬──────┬──────┬───┘
         │      │      │
         ▼      ▼      ▼
       LG00   LG01   LG02  ...  LGnn
         │      │      │
         ▼      ▼      ▼
        ASM / redo log files
```

### 关键参数

```
_use_single_log_writer      隐含参数，12.2+ 默认 FALSE（启用多 LGWR）
log_writer_processes        19c+ 显式参数，1-36，默认按 CPU_COUNT
_log_parallelism_max        log buffer strand 数，默认 ≈ CPU/16
```

### SCN 顺序保证机制

核心原理：**SCN 全局原子分配 → 物理写入只是把有序 SCN 落到不同 strand**

1. **SCN 生成**：全局原子计数器（SGA），session 在生成 redo 时先取号
2. **strand 插入**：按 hash(session_id) 分配到某个 strand
3. **物理布局**：redo log file 内部分多 strand region，不同 LGnn 写不同 offset
4. **commit 可见性**：LGWR coordinator 维护 `durable_watermark = min(所有 strand 已 flush RBA)`，等最慢 worker 完成才唤醒 session
5. **Recovery**：读所有 strand 按 SCN 全局归并排序，apply 在逻辑 SCN 顺序

### Redo block 自描述结构

```
┌──────────────────────────────┐
│ Block Header                 │
│   signature                  │
│   block_number               │
│   sequence_number            │
│   offset                     │
│   checksum       ← 自校验     │
├──────────────────────────────┤
│ Redo Record 1:               │
│   SCN | thread# | subSCN ... │
│   change vectors ...         │
├──────────────────────────────┤
│ Redo Record 2:               │
│   SCN | ...                  │
└──────────────────────────────┘
```

### 数据完整性自保

| 风险 | 解法 |
|---|---|
| 写错位 | block_number 自描述 + recovery 校验 |
| 写乱（partial write）| O_DSYNC 保证整块原子 + checksum |
| 数据腐败 | DB_BLOCK_CHECKSUM = FULL |
| 单盘故障 | redo log 多 member 镜像 + ASM 多副本 |
| Worker 失败 | coordinator 不更新 watermark，commit 卡住 |
| 物理写顺序乱 | SCN 顺序在生成阶段已定，recovery 重排 |
| 内存→设备路径 | HARD（Hardware Assisted Resilient Data） |

## 四、Oracle 为什么能并行 fsync——关键在"根本不用 fsync"

### PG/GaussDB 路径（buffered IO + fsync）

```
write() → 获取 inode i_rwsem 共享 → 进 page cache → 释放锁 → 返回
fsync() → 获取 inode i_rwsem 独占 🚫 → flush 所有 dirty page → 释放锁
         ↑
         全局串行化点
```

### Oracle 路径（O_DSYNC + O_DIRECT 或 ASM）

```
Oracle 打开 redo log:
  open(redo.log, O_RDWR | O_DIRECT | O_DSYNC, 0600)
                               ↑         ↑
                               │         └─ 每次 write 等设备 ACK
                               │            不需要 fsync
                               └─ 绕过 page cache

pwrite() → 短暂持 i_rwsem（读） → 构造 BIO 提交 → 释放锁
        → 等设备完成（NVMe IRQ/polling） → 返回

→ 多个 pwrite 可几乎完全并发，瓶颈下沉到设备队列深度
```

### Oracle 三层绕过 fsync

1. **ASM（大多数生产）**：裸块设备，无文件系统，无 inode，无 fsync
2. **O_DSYNC + O_DIRECT（文件系统）**：绕过 page cache，每次 write 即 durable
3. **多 redo log member**：镜像到不同存储，文件级并行

### 设备级真实并行度

| 设备 | 队列深度 |
|---|---|
| SATA SSD | 32 |
| SAS SSD | 128 |
| NVMe 消费级 | 32K per queue |
| NVMe 企业级 | 64K × 64 queues |
| Dorado 阵列 | 数千 |

## 五、MySQL 8.0 Scalable Redo Log 架构

### 旧架构（5.7 及以前）

```
Session1  Session2  Session3
   │         │         │
   └────┬────┴─────────┘
        │
        ▼
  ┌────────────────────┐
  │ log_sys global mutex│ ← 所有 session 串行争锁
  └────────┬───────────┘
           ▼
     log buffer 写入
           │
           ▼
        LGWR 线程（write + fsync 一条龙）
```

### 新架构（8.0+）

```
SessionN
   │ (无 mutex 争用，原子 reserve LSN)
   ▼
┌────────────────────────────┐
│   Lock-free log buffer      │  ← "link buf" 数据结构
│   (并行 memcpy)             │
└────────┬───────────────────┘
         │
         ▼
┌────────────────┐  通知 ┌──────────────────────┐
│ log_writer     │──────►│ log_write_notifier  │
│ (只 pwrite)    │       │ (唤醒等 write 的)    │
└────────┬───────┘       └─────────────────────┘
         │
         ▼
┌────────────────┐  通知 ┌──────────────────────┐
│ log_flusher    │──────►│ log_flush_notifier  │
│ (只 fsync)     │       │ (唤醒等 durable 的)  │
└────────────────┘       └─────────────────────┘
```

### 关键改进

| 改进点 | 说明 |
|---|---|
| 移除 log_sys 全局锁 | 换成 link buf（lock-free），session 插入不串行 |
| 4 个专职线程 | writer / flusher / write_notifier / flush_notifier |
| 流水线重叠 | writer 在 pwrite 下一批时 flusher 在 fsync 上一批 |
| 参数控制 | `innodb_log_writer_threads = ON` |

### 和 Oracle 的本质区别

```
Oracle Scalable LGWR:  多 worker 并行 pwrite 同一组 redo → 并行
MySQL 8.0 Redo Threads: 单 writer + 单 flusher，但不同阶段流水线 → 流水线

Oracle 吞吐来源 = N 倍 IO 并发能力
MySQL 吞吐来源 = 消除 mutex 争用 + 阶段重叠带来的延迟隐藏
```

### 性能数据（8.0 vs 5.7 sysbench）

| 并发数 | 5.7 TPS | 8.0 TPS | 提升 |
|---|---|---|---|
| 32 | 45,000 | 52,000 | +15% |
| 128 | 68,000 | 115,000 | +69% |
| 512 | 72,000 | 160,000 | +122% |
| 1024 | 71,000 | 180,000 | +153% |

## 六、PostgreSQL 现状（和 GaussDB 同）

```
Backend1  Backend2  ...  BackendN
   │         │              │
   ▼         ▼              ▼
 ┌──────────────────────────────┐
 │ WAL Insertion Locks (8 把)    │  ← 部分并行 hash(pid)
 └──────────────────────────────┘
   │
   ▼
 WAL buffers
   │
   ▼
 ┌──────────────────────┐
 │  WALWriteLock         │  ← 单把锁，写盘串行
 └──────┬───────────────┘
        ▼
  walwriter (单进程，200ms 唤醒)
        │
        ▼
     pg_wal/
```

### 哪层并行哪层不并行

```
✅ WAL insert 并行: 8 个 WALInsertLock 分区
❌ WAL write 不并行: 单 WALWriteLock，pwrite 串行
❌ fsync 不并行: 和 write 同一把锁保护
```

### 社区不做的理由

1. group commit 已经成熟（commit_delay + commit_siblings）
2. fsync 本身是串行化点
3. 多 writer 的协调复杂度（LSN 顺序、checkpoint、recovery 幂等）
4. 对典型负载收益不明显

### 关键代码位置

```
src/backend/access/transam/xlog.c
  - XLogWrite() ← 单线程串行写
  - WALWriteLock ← 全局 LWLock

src/backend/postmaster/walwriter.c
  - 单进程，200ms 唤醒
  - 只做后台 async flush

src/include/access/xlog.h
  - NUM_XLOGINSERT_LOCKS = 8（唯一并行维度）
```

## 七、对 GaussDB 的启示（农行场景）

解决 "1ms IO × 单线程 = 1000 IOPS 上限" 的能力对比：

| 数据库 | 能力 |
|---|---|
| Oracle 12.2+ | ✅ Scalable LGWR 直接横向扩展 |
| MySQL 8.0+ | ⚠️ 流水线 + lock-free buffer，对 1ms IO 天花板缓解有限 |
| PostgreSQL | ❌ 单 walwriter，只能靠 group commit 合并 IO |
| GaussDB | ❌ 单 walwriter（继承自 PG），Dorado sync 下暴露瓶颈 |

### 三种解法的层次（从简到繁）

```
层次 1: commit_delay / group commit
  成本: 几十行内核改动
  收益: 2-10 倍（小事务场景）
  结论: 最务实的短期方案（农行方案即此）

层次 2: 流水线 + lock-free buffer（MySQL 8.0 路线）
  成本: 千行级重构
  收益: 2-5 倍（主要解决锁争用）
  瓶颈: 仍然是单盘 IO 上限

层次 3: 真多 writer 并行（Oracle Scalable LGWR 路线）
  成本: 万行级重构 + 回归测试
  收益: N 倍（直接提高 IO 并发度）
  瓶颈: 存储设备本身 IOPS 上限
  社区共识: PG 系不认，只能自研
```

### 学 Oracle 的完整路径（从易到难）

```
① 切到 O_DSYNC 打开 WAL 文件
   成本：参数调整 (wal_sync_method = open_datasync)
   收益：消除 fsync 延迟（5-10%）

② 改用 DSS 裸卷（类似 ASM，GaussDB DMS+DSS 已有）
   成本：部署改造
   收益：绕过文件系统 inode 锁

③ 引入多 walwriter worker
   成本：大（内核级改造）
   收益：N 倍 IO 并发，真正突破单线程天花板

④ strand 化 WAL buffer
   成本：很大（和 Oracle 12.2 改造规模相当）
   收益：消除 WAL insert 锁争用
```

## 八、并发写入能力可视化

```
PG / GaussDB (fsync + 单 walwriter):
  ──W──●fsync──W──●fsync──W──●fsync──
  设备利用率 ~5%，有效并发 = 1

MySQL 8.0 (lock-free + 单 writer):
  ──W──W──W──W──●fsync──W──W──W──●fsync
  设备利用率 ~20%，有效并发 = 1（写）+ 1（flush 流水线）

Oracle (ASM + O_DSYNC + Scalable LGWR):
  ──W1──W1──W1── (worker 1)
  ──W2──W2──W2── (worker 2)
  ──W3──W3──W3── (worker 3)
  ──W4──W4──W4── (worker 4)
  设备利用率 ~80%，有效并发 = N（通常 4-8）
```

## 九、一句话结论

- **Oracle 能并行的根本原因**：从文件系统层就跑开了 fsync 这个串行化点（ASM 或 O_DIRECT + O_DSYNC），设备 queue depth 提供物理并发
- **MySQL 8.0 的改进**：消除 log_sys 全局 mutex，通过流水线隐藏延迟，但仍单 writer
- **PG / GaussDB 现状**：单 walwriter + buffered IO + fsync 的传统架构，社区无意改造
- **GaussDB 要追 Oracle 能力**：需要 O_DSYNC + DSS 裸卷 + 多 walwriter + strand 化 buffer 的组合改造，代价很大
- **短期最优解**：commit_delay 合并 IO（ROI 最高的层次 1 方案）

## 十、参考代码路径

```
Oracle:
  （闭源，参考 Oracle 12.2 Scalable LGWR 白皮书和 WL 文档）

MySQL:
  storage/innobase/log/log0write.cc
  storage/innobase/log/log0buf.cc
  storage/innobase/log/log0log.cc
  WL#10310: Redo log optimization: dedicated threads and lock-free buffer

PostgreSQL / GaussDB:
  src/backend/access/transam/xlog.c
  src/backend/postmaster/walwriter.c
  src/gausskernel/storage/access/transam/xlog.cpp (GaussDB)
  src/gausskernel/storage/access/transam/xact.cpp (GaussDB)
```
