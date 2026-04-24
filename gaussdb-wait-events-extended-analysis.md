# GaussDB 等待事件扩展分析（对 wait-wal-sync 分析的补充）

> 在 `gaussdb-wait-wal-sync-analysis.md` 基础上追加的扩展发现，
> 重点覆盖：commit 阻塞 insert 模式、wait wal sync 代码级精确边界、
> SS_DORADO_CLUSTER 下 wait wal sync 的真实语义、以及双集群网络物理架构。
>
> 本文档完整保留对话中讨论的所有图表、表格和源码引用。

---

## 一、pg_thread_wait_status 里"commit 堵 insert"的根因

### 最常见场景：唯一键冲突 + 慢 commit（占 70%+）

```
┌───────────────────────────────────────────────────────────────┐
│ 场景 ①：唯一键/主键冲突（最常见，占 70%+）                     │
├───────────────────────────────────────────────────────────────┤
│                                                               │
│  线程 A (commit)                                              │
│   ├─ INSERT INTO t(id,...) VALUES (1001, ...)                │
│   ├─ 写入成功，id=1001 行在内存中                             │
│   └─ COMMIT  ← 卡在 wait wal sync / wait transaction sync    │
│                                                               │
│  线程 B (insert)                                              │
│   └─ INSERT INTO t(id,...) VALUES (1001, ...)  ← 同一个 id   │
│       ├─ PG 发现 id=1001 有 pending 行                        │
│       ├─ 但不知道 A 会 commit 还是 rollback                    │
│       └─ 等待 A 的 XID 最终化 → transactionid 锁等待          │
│                                                               │
│  在 pg_thread_wait_status 里看:                              │
│    B 的 locktag 类型 = transactionid                         │
│    B 的 locktag_field1 = A 的 xid                            │
│    B 的 wait_event = wait transactionid                      │
│    B 的 blocking_pid = A 的 pid                              │
└───────────────────────────────────────────────────────────────┘
```

### 本质：MVCC 可见性等待，不是 A 主动持锁

这种等待**不是 A 持有某把行锁主动阻塞 B**，而是 PG 的 MVCC 机制：B 必须等到 A 的事务状态（commit/rollback）写入 CLOG，才能决定自己是否违反唯一约束。

- commit A 并没有拿住某把行锁不放
- B 是 PG 的 MVCC 机制强制要等 A 的 XID 状态（CLOG）落定
- A 的 XID 落定取决于 commit 完成

### 其他可能场景

```
② 外键引用（Foreign Key）
   A: INSERT 父表 → commit 中
   B: INSERT 子表，引用 A 刚插入的父行
   → B 需要 SHARE ROW EXCLUSIVE 锁等 A

③ 表级 DDL 被卡
   A: ALTER TABLE ... → commit 中
   B: INSERT 同一张表
   → 表锁冲突（但此时 A 是 DDL 不是 insert）

④ 序列/默认值依赖（少见）
   某些 DEFAULT 引用复杂表达式涉及其他表的锁

⑤ GIN 索引 pending list（非常少见）
   并发 GIN 索引更新的锁

⑥ 分布式 GaussDB 的 GTM/CSN 协调等待
   跨 DN 的两阶段提交协调
```

### 为什么"commit 卡住 → insert 被殃及"在 Dorado sync 下特别明显

```
 正常环境（本地 SSD）:
   commit 耗时 ~50μs → 很少命中唯一键冲突等待
   
 Dorado sync 双集群:
   commit 耗时 1~5ms (wait wal sync 等)
   同一时段并发度高（小事务支付场景）
   → 撞同一个 unique key 的概率飙升
   → pg_thread_wait_status 里大量 "commit 线程堵塞 insert 线程"
   
 本质：
   - 不是 commit 真的拿着锁不放
   - 是 commit 慢到 insert 还没等到它敲定结果
   - 原因是 wait wal sync / wait transaction sync 慢
   - 根因是存储同步 RTT + 单线程 WAL 写
```

### 精确定位诊断 SQL

```sql
-- 1. 看阻塞关系的详细链
SELECT 
  w.pid           AS blocked_pid,
  w.query_id      AS blocked_qid,
  w.lockmode      AS blocked_lockmode,
  w.locktag       AS waiting_on,
  w.wait_status,
  w.wait_event,
  h.pid           AS holder_pid,
  h.query_id      AS holder_qid,
  h.wait_status   AS holder_status,
  h.wait_event    AS holder_event
FROM pg_thread_wait_status w
LEFT JOIN pg_thread_wait_status h
  ON h.pid = w.blocking_pid
WHERE w.wait_status = 'wait transactionid'
   OR w.locktag LIKE 'transactionid%';
```

```sql
-- 2. 看 commit 线程具体在等什么
SELECT pid, usename, application_name,
       wait_status, wait_event, query
FROM pg_stat_activity
WHERE state = 'active'
  AND (query LIKE 'COMMIT%' OR wait_event LIKE '%wal%sync%');
```

```sql
-- 3. 看有多少 insert 在等 commit
SELECT wait_event, COUNT(*)
FROM pg_thread_wait_status
WHERE wait_status = 'wait transactionid'
GROUP BY wait_event;
```

```sql
-- 4. 直接看全局锁等待链
SELECT * FROM pg_locks 
WHERE NOT granted
  AND locktype = 'transactionid';
```

### 关键判别：locktag 类型

`pg_thread_wait_status.locktag` 看第一段即可判别类型：

```
┌─────────────────┬────────────────────────────────┐
│ locktag 开头     │ 含义                           │
├─────────────────┼────────────────────────────────┤
│ transactionid   │ 等某个 xid 的 commit/rollback  │  ← 场景 ①
│                 │ = MVCC 唯一约束等待             │
├─────────────────┼────────────────────────────────┤
│ tuple           │ 等某行的 row lock              │  ← UPDATE/DELETE
│                 │   (insert 不会走这个)          │     才会
├─────────────────┼────────────────────────────────┤
│ relation        │ 表级锁（DDL 冲突）              │  ← 场景 ③
├─────────────────┼────────────────────────────────┤
│ advisory        │ 应用显式 advisory lock         │
├─────────────────┼────────────────────────────────┤
│ virtualxid      │ 等某 backend 事务结束          │  ← VACUUM/REINDEX
└─────────────────┴────────────────────────────────┘

insert 被 commit 堵住 → 99% 是 transactionid 锁类型
```

### 根因判断决策流程

```
 pg_thread_wait_status 显示 commit 堵 insert
            │
            ▼
 看 insert 的 locktag 类型
            │
   ┌────────┴────────┬──────────────┐
   │                 │              │
 transactionid     tuple          relation
   │                 │              │
   │                 │              ▼
   │                 │         DDL 和 DML 冲突
   │                 │         → 查是否有 ALTER/VACUUM FULL
   │                 │
   │                 ▼
   │            commit 拿着 row lock
   │            → 不太可能是"insert 被堵"
   │            → 回查是不是 UPDATE 被标记成 insert
   │
   ▼
 唯一键冲突（最典型）
            │
            ▼
 看 commit 线程在等什么（holder 的 wait_event）
            │
   ┌────────┴────────────────┐
   │                         │
 wait wal sync          WALWriteLock
 wait transaction sync   + IO                
 SharedStorageWalWrite   pg_wait / fsync
   │                         │
   ▼                         ▼
 Dorado sync / 主备复制    本地 IO 瓶颈
 卡顿                      单线程 walwriter
   │                         │
   └─────────┬───────────────┘
             ▼
      根因：commit 延迟 
      表现：级联的 insert 等待
```

### 治标 vs 治本

```
┌────────────────────────┬──────────────────────────────────┐
│ 治标（缓解 insert 被堵） │ 治本（加速 commit）              │
├────────────────────────┼──────────────────────────────────┤
│ 业务侧:                 │ 存储/架构侧:                      │
│ - 拆分热点键            │ - commit_delay 合并小 IO         │
│ - 应用层去重            │ - synchronous_commit=off         │
│ - 拆分事务              │   (弱一致性业务)                  │
│ - 主键加时间戳前缀        │ - 升级 Dorado 阵列性能            │
│                        │ - 减少单事务 WAL 量               │
│ 参数侧:                 │                                  │
│ - lock_timeout          │ 长期:                            │
│   避免 insert 永久挂    │ - 多 walwriter（内核级改造）      │
│                        │ - WAL 写路径优化                  │
└────────────────────────┴──────────────────────────────────┘
```

---

## 二、wait transaction sync 是 commit 慢的次生症状

### 完整因果链

```
 根因（物理层）                中间传导              最终症状（等待事件）
 ────────────                  ─────────            ──────────────────
 
 Dorado 存储同步 RTT 1ms
         │
         ▼
 主库 walwriter 单线程
 1000 IOPS 天花板
         │
         ▼
 commit 线程（事务 A）
 卡在 SyncRepWaitForLSN ────────────────────►  ① wait wal sync
         │                                      （commit 线程自身）
         │
         │ A 的 xid 长时间未写入 CLOG
         │
         ▼
 事务 B 的 INSERT 做唯一键检查
         │
         ├─ 发现有 xid=A 的 pending 行
         │
         ├─ 调 XactLockTableWait(A.xid)
         │    → transactionid 锁等待 ───────►  pg_thread_wait_status
         │                                      locktag=transactionid
         │
         └─ 调 SyncLocalXidWait(A.xid) ───────►  ② wait transaction sync
             (procarray.cpp:4614-4626)          （insert 线程被动等）
```

### 对应源码锚点

```
 wait wal sync           = STATE_WAIT_WALSYNC
                          syncrep.cpp:315
                          调用点: xact.cpp:1804 SyncRepWaitForLSN
                          → commit 路径主角

 wait transaction sync   = STATE_WAIT_XACTSYNC  
                          procarray.cpp:4626
                          调用点: SyncLocalXidWait
                          → 任何线程检查"某 xid 是否 commit"时
                          
 pg_thread_wait_status   = transactionid 锁
 的 locktag=xid            XactLockTableWait
                          → insert 做 MVCC 唯一性验证
```

### 为什么 wait transaction sync 次数通常比 wait wal sync 还多

```
 可观测模式（生产常见）:

 wait wal sync  ████████████████████████   3000 次
 wait xact sync ████████████████████████   8000 次   ← 比 wal sync 高
 
 为什么 xact sync 比 wal sync 还高?
 
 因为 1 个慢 commit 可以级联拖住 N 个 insert:
 
 事务 A (commit 慢 2ms)           事务 A 只产生 1 次 wait wal sync
  │
  ├─ 事务 B 撞键 → wait xact sync
  ├─ 事务 C 撞键 → wait xact sync
  ├─ 事务 D 撞键 → wait xact sync   N 次 wait transaction sync
  └─ 事务 E 撞键 → wait xact sync
```

### wait transaction sync 的所有可能根因

虽然在 Dorado sync 场景下 90%+ 的 wait transaction sync 都来自慢 commit，但技术上 `SyncLocalXidWait` 在这些情况下也会被调用：

```
┌─────────────────────────────────────────────────────────┐
│ wait transaction sync 的所有可能根因                     │
├─────────────────────────────────────────────────────────┤
│ ① 慢 commit + 唯一键冲突                                 │
│    (主场景，Dorado sync 下 90%+)                         │
│                                                         │
│ ② 慢 commit + 读行被更新中                               │
│    SELECT ... FOR UPDATE 遇到 pending 行                │
│                                                         │
│ ③ CSN 可见性判定                                          │
│    读路径发现 xid 状态 COMMIT_IN_PROGRESS               │
│    (csnlog 里是 COMMITSEQNO_COMMIT_INPROGRESS)         │
│    必须等最终 CSN 落定                                   │
│                                                         │
│ ④ 子事务父事务状态判定                                    │
│    子事务提交时读 parent xid 状态                        │
│                                                         │
│ ⑤ 应用显式 PREPARE / 两阶段提交                          │
│    2PC 第一阶段后的窗口期                                │
│                                                         │
│ ⑥ 分布式 GaussDB 跨 DN CSN 同步                         │
│    (分布式版，集中式无)                                   │
└─────────────────────────────────────────────────────────┘

 在集中式版 + Dorado sync + 高并发小事务支付场景下:
 ① 是压倒性主因，其他可忽略
```

### 判别矩阵：如何确认就是 ①

```
 诊断清单（在实例上跑一遍）:
 
 ┌──────────────────────────────────────┬────────┐
 │ 症状                                  │ 是否符合 │
 ├──────────────────────────────────────┼────────┤
 │ wait wal sync 和 wait transaction    │        │
 │ sync 同时高                           │ □      │
 ├──────────────────────────────────────┼────────┤
 │ wait transaction sync 次数           │        │
 │ 约为 wait wal sync 的 2~10 倍        │ □      │
 ├──────────────────────────────────────┼────────┤
 │ pg_thread_wait_status 里大量         │        │
 │ locktag=transactionid               │ □      │
 ├──────────────────────────────────────┼────────┤
 │ 被阻塞方（blocked_pid）的 SQL 以      │        │
 │ INSERT/UPSERT 为主                   │ □      │
 ├──────────────────────────────────────┼────────┤
 │ 阻塞方（holder_pid）在 COMMIT 语句    │        │
 │ 或 wait_event=wait wal sync          │ □      │
 ├──────────────────────────────────────┼────────┤
 │ 业务表有 UNIQUE 约束或 PRIMARY KEY    │ □      │
 ├──────────────────────────────────────┼────────┤
 │ SharedStorageWalWrite 不高           │        │
 │ （说明不是本地存储慢）                │ □      │
 ├──────────────────────────────────────┼────────┤
 │ 复制链路 / DMS 响应时间变大           │ □      │
 └──────────────────────────────────────┴────────┘
 
 5 项以上吻合 → 确认就是慢 commit 引发的级联
```

### 回到分析文档的闭环

这个分析和 `gaussdb-wait-wal-sync-analysis.md` 里结论的第三条完全一致：**wait transaction sync 是 commit 慢的次生症状**。

- commit 线程自己看到的是 `wait wal sync`
- 被它拖住的 insert 线程看到的是 `wait transaction sync`
- `pg_thread_wait_status` 里的 `transactionid` 锁等待是它们之间的"桥梁"

治理思路：**治 commit 就治本**（commit_delay / 存储优化 / 弱一致）；**治 insert 是治标**（业务侧拆键 / lock_timeout）。

---

## 三、wait wal sync 的代码级精确边界

### commit 完整代码路径（xact.cpp:1443 RecordTransactionCommit）

精确标注每一段的等待事件类型：

```
┌──────────────────────────────────────────────────────────────────────┐
│ RecordTransactionCommit()                                            │
│                                                                      │
│ ─── 段 A: 生成 commit xlog record ──────────────────────────────────│
│  xact.cpp:1706 / 1746                                               │
│    commitRecLSN = XLogInsert(RM_XACT_ID, XLOG_XACT_COMMIT, true);  │
│                                                                      │
│    内部可能进入:                                                      │
│      ReserveXLogInsertLocation (xlog.cpp:1409)                      │
│      XLogInsertRecord (memcpy 到 WAL buffer)                        │
│                                                                      │
│    可能的等待事件:                                                    │
│      • WALInsertLock (LWLock, 8 把分区锁)                            │
│      • buffer wait (若 WAL buffer 满)                                │
│    ✗ 不是 wait wal sync                                              │
│                                                                      │
│ ─── 段 B: 本地 WAL flush 等待 ───────────────────────────────────────│
│  xact.cpp:1791                                                      │
│    XLogWaitFlush(commitRecLSN);                                     │
│                                                                      │
│    内部 (xlog.cpp:3475-3516):                                        │
│      while (flushTo < recptr) {                                     │
│        LWLockAcquireOrWait(walFlushWaitLock, LW_EXCLUSIVE)  ←第 1   │
│        PGSemaphoreLock(&walFlushWaitLock->sem, true)        ←第 2   │
│        LWLockRelease                                                │
│      }                                                              │
│                                                                      │
│    可能的等待事件:                                                    │
│      • walFlushWaitLock (LWLock)                                    │
│      • PGSemaphoreLock (backend 黑洞段，无 wait_event)               │
│                                                                      │
│    此段实际等的是:                                                    │
│      walwriter 把 WAL buffer 写到共享存储 + fsync                     │
│      在 Dorado 共享盘架构下 = walwriter 完成 SharedStorageWalWrite   │
│                                                                      │
│    ✗ 不是 wait wal sync（是"本地刷盘等待"，不是"远程同步等待")       │
│                                                                      │
│ ─── 段 C: 远程同步等待 ★★★★★ wait wal sync ★★★★★ ───────────────│
│  xact.cpp:1804                                                      │
│    if (synchronous_commit > LOCAL_FLUSH) {                          │
│        SyncRepWaitForLSN(commitRecLSN, ...);                        │
│    }                                                                │
│                                                                      │
│    内部 (syncrep.cpp:230-516):                                      │
│      【详见下一节】                                                   │
│                                                                      │
│    ✅ 这就是 wait wal sync 唯一覆盖的区段                             │
│                                                                      │
│ ─── 段 D: CLOG 提交（写事务最终状态）────────────────────────────────│
│  xact.cpp:1811                                                      │
│    TransactionIdCommitTree(xid, nchildren, children,                │
│                            GetCommitCsn());                         │
│                                                                      │
│    内部:                                                              │
│      TransactionIdSetTreeStatus -> SlruWritePage(CLOG)             │
│      CSNLogSetCommitSeqNo    -> SlruWritePage(CSN Log)             │
│                                                                      │
│    可能的等待事件:                                                    │
│      • CLogControlLock (SLRU)                                       │
│      • CSNBufMappingLock                                            │
│      • CSNLog-related locks                                         │
│    ✗ 不是 wait wal sync（CLOG 写属于 commit 可见性敲定）              │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

### SyncRepWaitForLSN 内部的精确分段（syncrep.cpp:230-516）

**这是 wait wal sync 真正的覆盖范围**。函数内部又分五段，只有第 4 段计入 wait wal sync：

```
┌────────────────────────────────────────────────────────────────────┐
│ SyncRepWaitForLSN(commitRecLSN, ...)                               │
│                                                                    │
│ ═══ 段 1: 快速退出检查（syncrep.cpp:232-276）════════════════════ │
│                                                                    │
│   if (SS_PRIMARY_ENABLE_TARGET_RTO)                                │
│       return SSRealtimeBuildWaitForTime(XactCommitLSN);  [Dorado  │
│                                                            RTO分支]│
│                                                                    │
│   if (ENABLE_DMS && !SS_STREAM_CLUSTER) return NOT_REQUEST;        │
│   if (!SyncRepRequested()) return NOT_REQUEST;                     │
│   if (!SyncStandbysDefined()) return NOT_REQUEST;                  │
│   if (!WalSndCtl->sync_standbys_defined) return ...;               │
│   if (sync_master_standalone && !SS_DORADO_CLUSTER) return ...;    │
│   if (!SynRepWaitCatchup(XactCommitLSN)) return NOT_WAIT_CATCHUP; │
│                                                                    │
│   ✗ wait_event 未改变（此时仍是调用前的状态）                       │
│   耗时：通常 <1μs（纯内存检查）                                       │
│                                                                    │
│ ═══ 段 2: 乐观自旋快路径（syncrep.cpp:282-292）═══════════════════ │
│                                                                    │
│   #define SYNCREPWAIT_TRY_TIMES 10000                              │
│   for (int tryTime = 0; tryTime < 10000; tryTime++) {              │
│       loopLSN = atomic_read(WalSndCtl->lsn[mode]);                 │
│       if (XLByteLE(XactCommitLSN, loopLSN)) {                      │
│           return REPSYNCED;  ← 备机已追上，直接返回                  │
│       }                                                            │
│       pg_usleep(1);   /* 睡 1 μs */                                │
│   }                                                                │
│                                                                    │
│   ⚠️ 极其关键的观测盲区：                                             │
│   这一段最多耗时 10,000 × 1μs = 10 ms                               │
│   但此时 wait_event 还没改成 STATE_WAIT_WALSYNC                     │
│                                                                    │
│   ✗ 如果备机在 10ms 内 ACK，这次 commit 根本不会出现 wait wal sync │
│   → 这就是为什么"快 commit 不显示 wait wal sync"                    │
│                                                                    │
│ ═══ 段 3: ps display 设置（syncrep.cpp:295-313）═════════════════ │
│                                                                    │
│   set_ps_display("... waiting for X/Y");                           │
│   ✗ 不影响 wait_event                                               │
│                                                                    │
│ ═══ 段 4: ★ wait wal sync 正式开始 ★（syncrep.cpp:315-503）════ │
│                                                                    │
│   WaitState oldStatus =                                            │
│       pgstat_report_waitstatus(STATE_WAIT_WALSYNC);  ← ★★★ 起点 │
│                                                                    │
│   while (true) {                                                   │
│       if (XLByteLE(XactCommitLSN, remoteLSN)) {                    │
│           break;  ← 备机 ACK 到位                                    │
│       }                                                            │
│       if (ProcDiePending) { break; }                              │
│       if (QueryCancelPending) { break; }                           │
│       if (!PostmasterIsAlive()) { break; }                         │
│       if (sync_master_standalone) { break; }                       │
│       ...                                                          │
│                                                                    │
│       if (LWLockAcquireOrWait(                                     │
│               walSyncRepWaitLock, LW_EXCLUSIVE)) {                 │
│                                                                    │
│           WaitSndCtl->syncWaitProc = t_thrd.proc;                 │
│           ResetLatch(&procLatch);                                  │
│           WaitLatch(&procLatch,                                    │
│                     WL_LATCH_SET | WL_TIMEOUT                      │
│                     | WL_POSTMASTER_DEATH,                         │
│                     1000L);   /* 1 秒超时 */                        │
│           WaitSndCtl->syncWaitProc = NULL;                         │
│           LWLockRelease(walSyncRepWaitLock);                       │
│       }                                                            │
│       remoteLSN = atomic_read(WalSndCtl->lsn[mode]);              │
│   }                                                                │
│                                                                    │
│ ═══ 段 5: wait wal sync 结束（syncrep.cpp:505）══════════════════│
│                                                                    │
│   (void)pgstat_report_waitstatus(oldStatus);  ← ★★★ 终点          │
│   ps_display 清理                                                   │
│   RESUME_INTERRUPTS();                                             │
│   return waitStopRes;                                              │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

### wait wal sync 的精确定义

```
 代码定义:
   起点 = syncrep.cpp:315
   终点 = syncrep.cpp:505
   
 语义定义:
   "commit 记录已写入本地 WAL buffer 并完成本地 flush 后，
    等待备机（或 DCF Paxos 大多数）确认接收并持久化该 LSN
    的时间段"

 具体是在等谁的 ACK（Dorado 双集群场景）:
   WalSndCtl->lsn[mode]
   ↑
   这个值被 walsender 更新，来源:
   - 本集群 DMS 资源池投票 (SS_DORADO_CLUSTER)
   - 备集群 shared_storage_walreceiver 回传 ACK
   - 由哪方更新取决于部署拓扑
```

### wait wal sync 的三种进入分支

代码里 `SyncRepWaitForLSN` 有一段特殊分流（syncrep.cpp:232-234）：

```
SyncRepWaitForLSN 入口
        │
        ▼
   检查 SS_PRIMARY_ENABLE_TARGET_RTO?
        │
   ┌────┴────┐
   是        否
   │         │
   ▼         │
 SSRealtime  │
 BuildWait   │
 ForTime     │
 (syncrep.cpp│
  :522)      │
             │
             ▼
       正常主循环
       (syncrep.cpp:230-516)

另外还有 Paxos 路径:
       ▼
       如果 enable_dcf:
       SyncPaxosWaitForLSN
       (syncrep.cpp:1794)
       也会设 STATE_WAIT_WALSYNC
```

**三个路径的 wait wal sync 等的对象不同**：

```
┌──────────────────────────────┬──────────────────────────────────┐
│ 函数                          │ wait wal sync 等的是什么          │
├──────────────────────────────┼──────────────────────────────────┤
│ SyncRepWaitForLSN            │ walsender 收到备机 ACK            │
│ (syncrep.cpp:230, 正常路径)   │ → WalSndCtl->lsn[mode] >= target │
│                              │                                  │
│ SSRealtimeBuildWaitForTime   │ Dorado 备集群 redo 追赶 RTO 窗口  │
│ (syncrep.cpp:522, SS Dorado) │ 不是 LSN 对比，是时间约束         │
│                              │                                  │
│ SyncPaxosWaitForLSN          │ DCF Paxos 过半数节点确认          │
│ (syncrep.cpp:1681, DCF)      │ → syncPaxosState = COMPLETE      │
└──────────────────────────────┴──────────────────────────────────┘
```

### wait wal sync 不覆盖的部分（重要）

```
 commit 总耗时 = t_A + t_B + t_C + t_D
                ↓     ↓     ↓     ↓
                段A   段B   段C   段D
                
 ┌────────┬────────────────────┬──────────────────┐
 │ 段      │ 事件                │ wait_event       │
 ├────────┼────────────────────┼──────────────────┤
 │ A       │ XLogInsert          │ WALInsertLock    │
 │        │ 生成 commit record  │ (可能，短)        │
 ├────────┼────────────────────┼──────────────────┤
 │ B       │ XLogWaitFlush       │ walFlushWaitLock │
 │        │ 本地刷盘等待         │ + PGSemaphoreLock│
 │        │                    │ (backend 黑洞段)  │
 ├────────┼────────────────────┼──────────────────┤
 │ C       │ SyncRepWaitForLSN   │ ★ wait wal sync │
 │        │ 远程同步等待         │                  │
 ├────────┼────────────────────┼──────────────────┤
 │ D       │ TransactionIdCommit │ CLogControlLock  │
 │        │ Tree (CLOG 更新)    │ 等 SLRU 相关      │
 │        │                    │ (可能，短)        │
 └────────┴────────────────────┴──────────────────┘

 ⚠️ wait wal sync 只能告诉你 t_C 有多长，
    无法反映 t_A / t_B / t_D。
    
 常见陷阱：
   commit 从 50μs 变到 5ms，你看到 wait wal sync 次数上升
   → 你可能以为慢化在远程同步
   → 但实际可能是 t_B 的 walFlushWaitLock 争用变大
   → 需要同时看 SharedStorageWalWrite 等事件
```

### wait wal sync 的触发前提（三个 AND 条件）

代码里逐层过滤，三个都满足才会到段 4：

```
条件 1: 调用方传入了需要同步的 synchronous_commit
  xact.cpp:1798:
    if (wrote_xlog && synchronous_commit > SYNCHRONOUS_COMMIT_LOCAL_FLUSH)
        SyncRepWaitForLSN(...);
  
  也就是说 synchronous_commit = off / local 时根本不进此函数

条件 2: 通过 SyncRepWaitForLSN 的所有前置检查
  syncrep.cpp:245-276:
    - SyncRepRequested() ：有请求同步复制
    - SyncStandbysDefined() ：配置了 sync standby
    - WalSndCtl->sync_standbys_defined ：sync standby 已就绪
    - 非 sync_master_standalone 状态
    - 通过 SynRepWaitCatchup 检查

条件 3: 10000 次乐观自旋未命中
  syncrep.cpp:282-292:
    如果备机在 ~10ms 内 ACK，直接返回，不记 wait_event
  
  → 只有"同步等待耗时 > ~10ms" 的 commit 才会留下 wait wal sync 痕迹
```

### 观测层的含义（给 LAS 采样使用）

结合 LAS 采样问题，这个精确边界有重要实践价值：

```
 场景分析:
 
 假设 commit 总耗时 20ms：
   t_A: XLogInsert         = 0.1 ms
   t_B: XLogWaitFlush      = 5.0 ms   (本地刷盘，Dorado 写)
   t_C: SyncRepWaitForLSN  = 14.5 ms  (远程同步)
     ├─ 前 10ms 自旋不计 wait_event
     └─ 后 4.5ms 计为 wait wal sync
   t_D: TransactionIdCommit = 0.4 ms
 
 LAS 1Hz 采样在这 20ms commit 里:
   - 命中 t_C 后半段（4.5ms）的概率 ≈ 4.5/1000 = 0.45%
   - 即便命中，wait_event 显示 wait wal sync
   - 但它只代表 4.5ms 的等待，不代表整个 20ms commit
 
 如果你按"wait wal sync 命中次数 × 1s"估算慢 commit 总耗时：
   - 会严重低估（因为前 10ms 的自旋阶段不显示）
   - 再加上 t_A/B/D 都不在此事件里
   - 需要结合 SharedStorageWalWrite (覆盖 t_B 的 walwriter 写盘段) 联合观测
```

### 一张图总结

```
         commit 总时间线（单事务）
  ├──────────────────────────────────────────────────┤
  │                                                  │
  XLogInsert  XLogWaitFlush  SyncRepWaitForLSN    CLOG 
  ~0.1ms       ~5ms           ~15ms              ~0.3ms
  │           │               │                    │
  │           │               ├─ 10ms 自旋 (无事件) │
  │           │               └─ 剩余主循环          │
  │           │                    ↑                │
  │           │                    └── wait wal sync
  │           │
  │           ├── walFlushWaitLock (lock 等待)
  │           ├── PGSemaphoreLock (黑洞段)
  │           └── 间接反映 walwriter 的 SharedStorageWalWrite
  │
  └── WALInsertLock (可能)

  wait wal sync 覆盖 ↑
  = 仅"远程 ACK 主循环"一段
  = syncrep.cpp:315 → 505
  ≠ 整个 commit
  ≠ 本地 flush
  ≠ CLOG 更新
```

### 核心要点

1. **代码边界**：严格 `syncrep.cpp:315` 到 `syncrep.cpp:505`，共约 190 行代码路径
2. **语义边界**：只覆盖"远程同步主循环"，不含自旋快路径、不含本地 flush、不含 CLOG
3. **盲区**：10000 次自旋共 ~10ms 是 wait wal sync 的"零计数区"——快 commit 在这里返回不会留痕迹
4. **三条路径**：正常 SyncRepWaitForLSN / SSRealtimeBuildWaitForTime / SyncPaxosWaitForLSN 都用同一个 STATE_WAIT_WALSYNC，但等待对象不同
5. **观测协同**：诊断 commit 慢化要 wait wal sync + SharedStorageWalWrite + walFlushWaitLock 三件套联看，单看 wait wal sync 会误判

---

## 四、SS_DORADO_CLUSTER 下 wait wal sync 的真实语义

### 关键认知：架构里有两个独立的 sync 边界

这是理解的关键。这个架构下存在**两层异步/同步复制**，容易被混为一谈：

```
 ┌────────────────────────────────────────────────────────────────┐
 │ 主集群（多节点 + DMS）                                          │
 │                                                                │
 │   Primary node      Standby node 1    Standby node 2           │
 │   (backend)          (DMS 参与)        (DMS 参与)              │
 │        │                 │                  │                  │
 │        └─────────┬───────┴────────┬─────────┘                  │
 │                  │ DMS 投票/资源池│                            │
 │                  ▼                ▼                            │
 │              ┌────────────────────────┐                         │
 │              │  DSS (Distributed      │                         │
 │              │      Shared Storage)   │  ← 主集群的共享盘       │
 │              └──────────┬─────────────┘                         │
 │                         │ pwrite                                │
 └─────────────────────────┼───────────────────────────────────────┘
                           │
                           ▼
 ┌────────────────────────────────────────────────────────────────┐
 │                    Dorado 阵列 A（主端）                        │
 │                                                                │
 │   ★ 边界 1：Dorado HyperMetro / HyperReplication ★             │
 │                                                                │
 │   Dorado A ══════ sync 复制 ══════► Dorado B                   │
 │   (1ms RTT 物理下限)                                            │
 │                                                                │
 │   这一层完全在存储侧，GaussDB 不参与协议                         │
 │   → 主集群 pwrite 返回 = 两边都已落盘                           │
 │   → 这段等待在 walwriter 里，事件名 SharedStorageWalWrite       │
 └────────────────────────────────────────────────────────────────┘
                           │
                           ▼ Dorado B 的数据
 ┌────────────────────────────────────────────────────────────────┐
 │ 备集群（多节点 + DMS）                                          │
 │                                                                │
 │   Primary node      Standby node 1    Standby node 2           │
 │   (回放主力)          (协同参与)       (协同参与)              │
 │        │                                                       │
 │        │ shared_storage_walreceiver 读 WAL                     │
 │        │ 然后 startupProcess 做 redo 回放                       │
 │        │                                                       │
 │        │ ★ 边界 2：GaussDB 级复制确认 ★                         │
 │        │                                                       │
 │        │ 备集群回放进度（reply.apply）→                          │
 │        │ 通过 walsender/walreceiver 协议回传给主集群             │
 │        └─────────────────────────────┐                         │
 └──────────────────────────────────────┼─────────────────────────┘
                                        │
                                        │ 回 ACK 给主集群
                                        ▼
                        主集群的 walsender 更新
                        WalSndCtl->lsn[mode]
                                        │
                                        ▼
                        唤醒正在 wait wal sync 的 backend
```

### 两个边界对应的代码和等待事件是不同的

```
┌──────────────────────┬────────────────────────┬──────────────────────┐
│ 边界                  │ 代码位置                │ 等待事件              │
├──────────────────────┼────────────────────────┼──────────────────────┤
│ 边界 1                │ walwriter 线程的        │ SharedStorageWalWrite│
│ Dorado 存储复制        │ pwrite 到 DSS           │                      │
│ (HyperMetro/sync)    │                        │                      │
│                      │ xlog.cpp:2824 附近      │ 阻塞在 DSS API 调用  │
│                      │ 包含 dss_append /       │ 直到 Dorado 两边落盘 │
│                      │ SharedStorageIoSemSet    │                      │
├──────────────────────┼────────────────────────┼──────────────────────┤
│ 边界 2                │ backend 线程的          │ ★ wait wal sync       │
│ GaussDB 复制协议       │ SyncRepWaitForLSN       │                      │
│                      │                        │                      │
│                      │ syncrep.cpp:230-516     │ 阻塞在主集群           │
│                      │                        │ WalSndCtl->lsn[mode] │
│                      │                        │ 被 walsender 更新      │
└──────────────────────┴────────────────────────┴──────────────────────┘
```

### wait wal sync 在 SS_DORADO_CLUSTER 下具体等什么

关键证据在 `walsender.cpp:2987`：

```cpp
/*
 * 1. When starting ss dorado replication, we need to know replayPtr that
 *    standby has already replayed, because primary xlog will cover standby
 *    xlog by Dorado synchronous replication.
 * 2. Otherwise, we only need to confirm that standby xlog has been flushed
 *    successfully.
 */
if (SS_DORADO_CLUSTER) {
    AdvanceReplicationSlot(reply.apply);   // ★ 用 apply（回放位点）
} else {
    AdvanceReplicationSlot(reply.flush);   // 普通流复制用 flush（落盘位点）
}
```

**这行代码揭示了 SS_DORADO_CLUSTER 的特殊语义**：

```
 普通流复制 (非 Dorado):
   备机 ACK 的是 flush position ─── 只要备机收到并落盘就算数
   
 SS_DORADO_CLUSTER:
   备机 ACK 的是 apply position ─── 必须等备机回放完才算数
                        ↑
                        这个要求更严格！
                        
 为什么要更严格？
 注释说："primary xlog will cover standby xlog by Dorado synchronous
        replication"
 
 意思是：Dorado 存储同步会把主端 WAL 直接写到备端 DSS（覆盖备端 WAL）
        → 备端的"存储已同步"不等于"数据库已恢复一致"
        → 必须等备端回放追上，才能保证故障切换不丢数据
```

### 完整的时间轴（带 Dorado 关联点）

```
 主集群 backend 执行 COMMIT 的时间轴:

 t0 ───── t1 ─────── t2 ─────────── t3 ─────────── t4
 │         │          │              │              │
 │         │          │              │              │
 XLogInsert  XLogWaitFlush    SyncRepWaitForLSN     CLOG
 (段A)       (段B)            (段C = wait wal sync)  (段D)
 │           │                │                      │
 │           │                │                      │
 │           │                │                      └─ TransactionId
 │           │                │                         CommitTree
 │           │                │
 │           │                └─────────────────────────┐
 │           │                                          │
 │           │                                          │
 │           ├── walwriter 被唤醒做 pwrite (DSS API)    │
 │           │    ↓                                     │
 │           │   DSS → Dorado A 写入                    │
 │           │    ↓                                     │
 │           │   ★ Dorado HyperMetro ★                  │
 │           │   sync 复制到 Dorado B                    │
 │           │   (这里物理下限 ~1ms RTT)                  │
 │           │    ↓                                     │
 │           │   两边都 ACK，pwrite 返回                  │
 │           │    ↓                                     │
 │           │   walwriter 的 SharedStorageWalWrite 结束 │
 │           │    ↓                                     │
 │           │   主 backend 的 XLogWaitFlush 返回        │
 │           │   (段 B 结束)                            │
 │           │                                          │
 │           └──────────────────────────────────────────┘
 │                                                      │
 │                                                      │
 │                        此时：                         │
 │                        - WAL 已在 Dorado B 上（存储已同步）│
 │                        - 但备集群 GaussDB 可能还没读/回放  │
 │                                                      │
 │                        段 C (wait wal sync) 要等:     │
 │                                                      │
 │                        备集群 Primary 节点:           │
 │                        ① shared_storage_walreceiver    │
 │                           读 DSS B 的新 WAL             │
 │                        ② 转交 startupProcess             │
 │                        ③ 回放（redo apply）              │
 │                        ④ 更新自身 apply LSN              │
 │                        ⑤ 经 walsender→walreceiver 通道  │
 │                           回传给主集群                    │
 │                        ⑥ 主集群 walsender 更新          │
 │                           WalSndCtl->lsn[mode]         │
 │                        ⑦ 唤醒主 backend 的 WaitLatch    │
 │                                                      │
 │                                                      │
 └───────────────────── 段 C 结束 ──────────────────────┘
```

### wait wal sync 和 Dorado 的关系：间接的

```
┌──────────────────────────────────────────────────────────┐
│ 问题：wait wal sync 和 Dorado 有没有关系？                  │
├──────────────────────────────────────────────────────────┤
│                                                          │
│ 直接关系: ❌ 没有                                          │
│   wait wal sync 的代码路径不调任何 Dorado API            │
│   它的唤醒条件是 WalSndCtl->lsn[mode] 被更新              │
│   更新来源是"备集群 walreceiver 回传的 ACK 消息"           │
│   Dorado 阵列本身不参与这个协议                            │
│                                                          │
│ 间接关系: ✅ 存在                                          │
│   wait wal sync 的结束依赖"备集群看到 WAL"                │
│   备集群通过共享存储看 WAL                                  │
│   共享存储内容的同步 = Dorado HyperMetro 的职责            │
│   所以：                                                    │
│     - Dorado 阵列慢 → 备集群 DSS 看不到新 WAL             │
│     - 备集群 walreceiver 读不到数据                        │
│     - 备集群回放不动                                        │
│     - apply LSN 不推进                                    │
│     - 主集群 wait wal sync 被动拉长                       │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

### 三种可能的瓶颈及其诊断信号

```
 在 Dorado 双集群 sync 场景下，wait wal sync 变长，
 实际可能是以下三种情况之一：
 
┌────────────────────────────────────────────────────────────┐
│ 情况 ①：备集群 GaussDB 回放慢（最常见）                    │
├────────────────────────────────────────────────────────────┤
│                                                            │
│ 表现:                                                       │
│   - wait wal sync 高                                        │
│   - SharedStorageWalWrite 正常                              │
│   - 备集群 redo worker CPU 高                               │
│   - pg_stat_get_wal_receiver().apply_location 推进慢        │
│                                                            │
│ 原因:                                                       │
│   - 备集群硬件规格低                                         │
│   - recovery_parse_workers 不够                            │
│   - 大事务 / DDL 阻塞回放                                   │
│                                                            │
│ 和 Dorado 的关系: 无                                        │
│                                                            │
└────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────┐
│ 情况 ②：备集群 walreceiver 读取慢（次常见）                 │
├────────────────────────────────────────────────────────────┤
│                                                            │
│ 表现:                                                       │
│   - wait wal sync 高                                        │
│   - SharedStorageWalWrite 正常                              │
│   - 备集群 walreceiver 读 DSS 慢                            │
│   - 备集群对 Dorado B 的读 IO 延迟高                        │
│                                                            │
│ 原因:                                                       │
│   - 备集群 Dorado B 读性能差（缓存 cold 等）                 │
│   - 备集群 DSS 路径有问题                                    │
│   - 备端网络异常                                              │
│                                                            │
│ 和 Dorado 的关系: 部分相关（备端 Dorado 读性能）            │
│                                                            │
└────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────┐
│ 情况 ③：Dorado HyperMetro 本身慢（大家最担心的）            │
├────────────────────────────────────────────────────────────┤
│                                                            │
│ 表现:                                                       │
│   - wait wal sync 高                                        │
│   - ★ SharedStorageWalWrite 同时高 ★  ← 关键特征            │
│   - SharedStorageCtlInfoWriter 可能高                       │
│   - Dorado 阵列监控显示 HyperMetro 同步延迟高                │
│                                                            │
│ 原因:                                                       │
│   - 双 Dorado 链路带宽不足                                   │
│   - 双 Dorado 之间的 RTT 延迟增大（光模块、网卡）            │
│   - Dorado 阵列硬件故障（降级模式）                          │
│                                                            │
│ 和 Dorado 的关系: 直接相关                                   │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

### 判别公式

```
 根据三个事件的组合判别根因:
 
 ┌─────────────────┬─────────────────────┬─────────────────────┐
 │ wait wal sync   │ SharedStorageWalWrite│ 根因                │
 ├─────────────────┼─────────────────────┼─────────────────────┤
 │ 高               │ 正常                 │ ① 备集群回放/读慢    │
 │ 高               │ 高                   │ ③ Dorado 本身慢      │
 │ 正常             │ 高                   │ 主端存储/walwriter慢 │
 │ 低               │ 低                   │ commit 流不是瓶颈    │
 └─────────────────┴─────────────────────┴─────────────────────┘

 生产观察模式:
   "SharedStorageCtlInfoWriter / SharedStorageWalWrite 这个事件之前是
    有的，会和 wait wal sync 一起出现，这次没有"
   
   → 这就对应第一种模式：
      SharedStorageWalWrite 正常 + wait wal sync 高
      → Dorado 没问题，问题在备集群 GaussDB 侧
      → 排查方向：备集群回放能力、网络往返、walreceiver 状态
```

### SS_DORADO_CLUSTER 下的 wait wal sync 一张图

```
                    SS_DORADO_CLUSTER 下 commit 等待拆解
 ────────────────────────────────────────────────────────────────────
                                                                    
 ┌── 主集群 backend ──────────────────────────────────────────┐      
 │                                                            │      
 │ XLogInsert ─► XLogWaitFlush ─► SyncRepWaitForLSN ─► CLOG │      
 │               │                 │                          │      
 │               │ 唤醒 walwriter  │ 等 standby apply ACK     │      
 │               ▼                 │                          │      
 │         ┌──────────┐            │                          │      
 │         │walwriter │            │                          │      
 │         │ 写 DSS   │            │                          │      
 │         │ ⬇        │            │                          │      
 │         │SharedStor│ ← F 段      │                          │      
 │         │ WalWrite │            │                          │      
 │         └──┬───────┘            │                          │      
 └────────────┼────────────────────┼──────────────────────────┘      
              │                    │                                 
              ▼                    │                                 
     ┌────────────────┐            │                                 
     │  Dorado A      │            │                                 
     │     ║          │            │                                 
     │     ║ sync     │            │                                 
     │     ║ (1ms)    │            │                                 
     │     ▼          │            │                                 
     │  Dorado B      │            │                                 
     └────────────────┘            │                                 
              │                    │                                 
              ▼ WAL 可见            │                                 
 ┌── 备集群 ─────────────────┐      │                                 
 │ shared_storage_           │      │                                 
 │ walreceiver 读 DSS        │      │                                 
 │  ⬇                        │      │                                 
 │ startupProcess 回放        │      │                                 
 │  ⬇                        │      │                                 
 │ apply LSN 推进             │      │                                 
 │  ⬇                        │      │                                 
 │ reply.apply 回传主集群     │ ──────┘                                
 └───────────────────────────┘                                        
                                    │                                 
                                    ▼                                 
                       主集群 walsender 更新                           
                       WalSndCtl->lsn[mode]                          
                                    │                                 
                                    ▼                                 
                     唤醒 backend 的 wait wal sync (I 段)              
                                                                      
 ─────────────────────────────────────────────────────────────────── 
  F 段 = Dorado 存储同步等待（walwriter 中）                           
  I 段 = 备集群 GaussDB 回放确认等待（backend 中）                      
  两段串行发生但在不同线程，属于 commit 链路的两个独立延迟源             
```

### 精确回答

**wait wal sync 在 SS_DORADO_CLUSTER 下等的是**：
> 备集群（另一个 Dorado 阵列上的 GaussDB 集群）完成该 commit LSN 的 **redo 回放** 并把 `apply` 位点通过 walsender/walreceiver 通道回传给主集群的时间。

**和 Dorado 的关系**：
- **直接等待对象：不是 Dorado**（等的是备 GaussDB 的 apply）
- **间接依赖：是 Dorado**（Dorado 把主端 WAL 搬到备端 DSS 是前提）
- **Dorado 自己的同步延迟被归到 SharedStorageWalWrite 事件**（在 walwriter 里），不在 wait wal sync 里

调优策略：看到 wait wal sync 高，**不要先责怪 Dorado 阵列**。先看 SharedStorageWalWrite 是不是也高。如果 Shared* 正常、wait wal sync 单独高，问题在**备集群的 GaussDB 回放链路**。只有 Shared* 和 wait wal sync 一起涨，才是 Dorado 真慢。

---

## 五、双集群网络物理架构

### 两张完全独立的网络

```
┌──────────────────────────────────────────────────────────────────┐
│                                                                  │
│  ┌──────────────────────────────────────────────────┐           │
│  │            主集群 GaussDB (多节点)                 │           │
│  └──────────────┬────────────────┬───────────────────┘           │
│                 │                │                               │
│                 │ HBA            │ NIC                           │
│                 │ (FC/iSCSI)     │ (以太网)                      │
│                 │                │                               │
│   ═══════════════════════        ═════════════════               │
│   │                     │        │                │              │
│   │  SAN (存储网络)     │        │  IP 网络       │              │
│   │  独立光纤交换机      │        │  (数据/管理)   │              │
│   │                     │        │                │              │
│   ═══════════════════════        ═════════════════               │
│                 │                │                               │
│                 ▼                │                               │
│          ┌──────────┐            │                               │
│          │ Dorado A │            │                               │
│          │          │            │                               │
│          │  ═══════════╗          │                               │
│          │  HyperMetro ║  ← ★ Dorado 专用复制链路               │
│          │  同步 FC/IP ║         (独立光纤，GaussDB 看不到)        │
│          │  SAN 链路   ║                                          │
│          │  ═══════════╝          │                               │
│          │     ↕                  │                               │
│          │ Dorado B │            │                               │
│          └──────────┘            │                               │
│                 │                │                               │
│                 │ HBA            │ NIC                           │
│                 │                │                               │
│  ┌──────────────┴────────────────┴───────────────────┐           │
│  │            备集群 GaussDB (多节点)                 │           │
│  └──────────────────────────────────────────────────┘           │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘

                 存储网络             IP 网络
                 ─────                ──────
 用途:          块级 IO 读写          应用层复制协议
                Dorado 同步            walsender/walreceiver
 
 协议:          SCSI/NVMe-oF          TCP + libpq
                Dorado HyperMetro      pg-wire protocol
 
 可见性:        GaussDB 透明           GaussDB 直接使用
                (OS 层 multipath)     (pg_hba, replconninfo)
 
 管理:          存储管理员              DBA
                Dorado 控制台          postgresql.conf
```

### 各自承担的数据流

```
┌───────────────────────────────────┬─────────────────────────────┐
│  走 Dorado 存储光纤的              │  走 IP 网络的                │
├───────────────────────────────────┼─────────────────────────────┤
│                                   │                             │
│ ✅ WAL 文件内容                    │ ✅ walsender/walreceiver 连接│
│    (主端 pwrite 后 Dorado 自动     │    握手 / 心跳 / keepalive  │
│     同步到 B)                     │                             │
│                                   │ ✅ 主备流复制状态报文          │
│ ✅ 数据文件                        │    (standby status)         │
│    (ctl / pg_data 等)             │                             │
│                                   │ ✅ ★ reply.apply / reply.   │
│ ✅ 备集群 shared_storage_          │      flush / reply.write    │
│    walreceiver 读 WAL              │    (备端回放位点回传)        │
│    (从 Dorado B 的 DSS 读)         │                             │
│                                   │ ✅ 控制面消息                 │
│ ✅ checkpoint 等 DB 文件变更       │    (主备切换、重连)          │
│                                   │                             │
│ ❌ 不走复制协议消息                │ ✅ CM 集群管理心跳           │
│                                   │                             │
│                                   │ ❌ 不走 WAL 数据本身          │
│                                   │    (在 SS_DORADO_CLUSTER 下) │
│                                   │                             │
└───────────────────────────────────┴─────────────────────────────┘
```

### 为什么 SS_DORADO_CLUSTER 还要有 IP 复制网络

架构上看似"既然 Dorado 已经同步过去了，备集群直接读就行，为啥还要 IP 网络连主集群？"——答案是**控制面和数据面的分离**：

```
┌─────────────────────────────────────────────────────────────┐
│ 原因 ①：Dorado 是"单向数据搬运"，无法回传状态               │
│                                                             │
│ Dorado HyperMetro 只保证:                                   │
│   主端写的块 → 备端块一致                                    │
│                                                             │
│ 但无法做到:                                                  │
│   备端告诉主端"我已经回放到哪里"                              │
│   备端告诉主端"我的负载状态"                                 │
│   备端告诉主端"我现在能不能接管"                              │
│                                                             │
│ 这些需要 GaussDB 自己的协议                                  │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ 原因 ②：需要握手 + 心跳                                      │
│                                                             │
│ walreceiver 启动时要对主库做:                                │
│   - 身份认证（pg_hba）                                       │
│   - 协商协议版本                                              │
│   - 请求起始 LSN                                              │
│                                                             │
│ 运行期要保持心跳:                                             │
│   - 每 wal_receiver_status_interval 秒发状态                 │
│   - 主库用这个判断 standby 是否存活                           │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ 原因 ③：sync 复制的 commit 确认需要端到端信号                │
│                                                             │
│ 主 backend 的 wait wal sync 等的是 WalSndCtl->lsn[mode]    │
│ 这个值必须被主端的 walsender 主动更新                         │
│ walsender 的更新触发条件 = 收到 standby 的 reply 消息        │
│ reply 消息 = TCP 网络过来                                    │
│                                                             │
│ 如果只有 Dorado 同步，主端永远不知道备端是否回放完            │
│ → wait wal sync 永远无法结束                                 │
└─────────────────────────────────────────────────────────────┘
```

### 典型部署：两套网络的物理配置

```
 生产环境一般是这样部署:
 
 ┌─────────────────────────────┐         ┌─────────────────────────────┐
 │     主集群机房                │         │     备集群机房                │
 │                             │         │                             │
 │  GaussDB 节点                 │         │  GaussDB 节点                 │
 │  ┌──────┐                    │         │  ┌──────┐                    │
 │  │ NIC  │────────────────────┼─────────┼──│ NIC  │                    │
 │  └──────┘   专用复制 VLAN     │         │  └──────┘                    │
 │  ┌──────┐   (万兆以太网)      │         │  ┌──────┐                    │
 │  │ HBA  │                    │         │  │ HBA  │                    │
 │  └───┬──┘                    │         │  └───┬──┘                    │
 │      │                       │         │      │                       │
 │  ┌───▼──────┐                │         │  ┌───▼──────┐                │
 │  │ Dorado A │════════════════┼═════════┼══│ Dorado B │                │
 │  └──────────┘  专用 FC SAN   │         │  └──────────┘                │
 │               (16/32Gbps     │         │                             │
 │                光纤)          │         │                             │
 │                             │         │                             │
 └─────────────────────────────┘         └─────────────────────────────┘
                                                                        
     两条网络:                                                           
     1. IP 复制 VLAN  ← walsender/walreceiver ACK 走这里                
     2. Dorado FC SAN ← 存储块同步走这里                                 
```

### 两套网络各自的延迟贡献

```
 一次 commit 在 sync 模式下的总延迟 = f(两套网络 + 两边 DB)
 
 ┌────────────────────────┬─────────────────┬─────────────────┐
 │ 段                      │ 延迟来源         │ 走哪条网络        │
 ├────────────────────────┼─────────────────┼─────────────────┤
 │ 主端 walwriter pwrite   │ 本机 DSS API    │ 主端 HBA→Dorado A│
 │                        │                 │ (本机内)          │
 ├────────────────────────┼─────────────────┼─────────────────┤
 │ ★ Dorado A → B 同步    │ ~0.5-1ms RTT    │ Dorado FC SAN    │
 │                        │ (光纤物理下限)   │                  │
 ├────────────────────────┼─────────────────┼─────────────────┤
 │ 备端 walreceiver 读    │ 备端 DSS API    │ 备端 HBA→Dorado B│
 │ WAL                    │                 │ (本机内)          │
 ├────────────────────────┼─────────────────┼─────────────────┤
 │ 备端回放                │ CPU 密集        │ -                │
 ├────────────────────────┼─────────────────┼─────────────────┤
 │ ★ reply.apply 回传     │ TCP RTT         │ IP 复制网         │
 │                        │ ~0.2-0.5ms      │                  │
 ├────────────────────────┼─────────────────┼─────────────────┤
 │ 主端唤醒 backend        │ 本机调度        │ -                │
 └────────────────────────┴─────────────────┴─────────────────┘
 
 "两条网络都要稳":
   - Dorado FC 链路抖动 → SharedStorageWalWrite 慢
   - IP 复制链路抖动  → wait wal sync 慢（ACK 回不来）
   - 备端回放慢       → wait wal sync 慢（apply 推进慢）
 
 三者都会让 commit 变慢，但暴露的等待事件完全不同。
```

### 实际部署的几个"坑"

```
┌──────────────────────────────────────────────────────────┐
│ 坑 ①：IP 复制网络和业务网络混用                           │
│                                                          │
│ 场景：为了省端口，把 walsender 的 TCP 和业务流量放同一网卡 │
│ 问题：业务高峰期 TCP 延迟抖动 → reply.apply 回传慢         │
│ 表现：wait wal sync 随业务 QPS 波动                        │
│ 解决：专用 VLAN 或物理网卡给复制流量                       │
└──────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────┐
│ 坑 ②：Dorado FC 链路和 IP 复制链路延迟不匹配              │
│                                                          │
│ 场景：Dorado 走同城光纤 1ms，IP 网络走 SDN 可能 5ms        │
│ 问题：Dorado 已同步，但 ACK 还在路上                       │
│ 表现：wait wal sync > SharedStorageWalWrite（反直觉）     │
│ 解决：两套网络用同一条物理路径，或至少延迟 SLA 对齐          │
└──────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────┐
│ 坑 ③：IP 复制网络断掉，Dorado 还在复制                    │
│                                                          │
│ 场景：IP 链路故障，但 SAN 链路正常                         │
│ 问题：                                                    │
│   - Dorado 还在把数据搬到 B                               │
│   - 但主端 walsender 断连了                                │
│   - wait wal sync 永远等不到 ACK                           │
│   - 主集群 commit 全部阻塞                                  │
│ 解决：                                                    │
│   - 配置 synchronous_standby_names 包含多个备             │
│   - 配置 sync_master_standalone 自动降级                  │
│   - CM 监控 IP 链路主动切换                                │
└──────────────────────────────────────────────────────────┘
```

---

## 六、完整数据流（按物理路径梳理）

### 四段物理链路及协议对应

```
┌──────────────────────────────────┬────────────────────────┬──────────────────┐
│ 描述                              │ 代码层对应              │ 物理链路          │
├──────────────────────────────────┼────────────────────────┼──────────────────┤
│ ① 主集群往主 Dorado 写数据         │ walwriter.pwrite()    │ 本机房 FC SAN    │
│   通过光纤网络                     │ → DSS → Dorado A       │                  │
├──────────────────────────────────┼────────────────────────┼──────────────────┤
│ ② 主 Dorado 往备 Dorado 写数据    │ Dorado HyperMetro     │ 专用跨机房 FC     │
│   用光纤网络                      │ (对 GaussDB 透明)       │ 或 iSCSI 链路    │
│                                  │                        │                  │
│   精确：不是"A 写完再同步 B"，     │                        │                  │
│   而是 sync 模式下 A 的写 = 等到   │                        │                  │
│   B 落盘才返回                    │                        │                  │
├──────────────────────────────────┼────────────────────────┼──────────────────┤
│ ③ 备集群通过光纤网络读取            │ shared_storage_        │ 备机房 FC SAN    │
│   备 Dorado 日志                   │ walreceiver 从 DSS 读  │                  │
│                                  │                        │                  │
│   注意：是"在备机房内部"走光纤     │                        │                  │
│   不是跨机房的那根                 │                        │                  │
├──────────────────────────────────┼────────────────────────┼──────────────────┤
│ 在备集群完成日志应用                │ startupProcess         │ -                │
│                                  │ 回放 redo              │ (纯本机 CPU)     │
├──────────────────────────────────┼────────────────────────┼──────────────────┤
│ ④ 应用后通过 NIC 网络告知主集群     │ walreceiver→walsender │ 跨机房 IP 网络    │
│                                  │ TCP 上的 pg-wire 协议  │ (以太网)         │
│                                  │ reply.apply/flush      │                  │
│                                  │                        │                  │
│   精确：不是"应用后立刻发"，       │                        │                  │
│   而是 wal_receiver_status_       │                        │                  │
│   interval 秒发一次 + 关键事件    │                        │                  │
│   触发时立刻发                    │                        │                  │
└──────────────────────────────────┴────────────────────────┴──────────────────┘
```

### 一张简图总结

```
 主机房                                            备机房
 ┌─────────────────┐                              ┌─────────────────┐
 │  主 GaussDB      │                              │ 备 GaussDB       │
 │  ┌─────────┐    │                              │  ┌──────────┐   │
 │  │ backend │    │ ④ IP/TCP (reply.apply)       │  │walreceive│   │
 │  │ walwriter│   │◄─────────────────────────────│  │startup   │   │
 │  │ walsender│   │    NIC 跨机房以太网            │  └─────┬────┘   │
 │  └────┬────┘    │                              │        │        │
 │       │         │                              │        │        │
 │       │ ①       │                              │        │ ③      │
 │       │ FC      │                              │        │ FC     │
 │       ▼         │                              │        ▼        │
 │  ┌─────────┐    │                              │   ┌─────────┐   │
 │  │ Dorado A │   │  ② FC/iSCSI (HyperMetro)     │   │Dorado B │   │
 │  │         │───────────────────────────────────│──►│         │   │
 │  │         │   │    专用跨机房光纤               │   │         │   │
 │  └─────────┘   │                              │   └─────────┘   │
 └─────────────────┘                              └─────────────────┘
 
     ①②③ 走 FC SAN (块级 IO)
      ④  走 IP 网 (应用层协议)
```

### Dorado A→B 同步的时机（精确澄清）

"主 Dorado 往备 Dorado 写数据"其实应该理解为：

```
 sync 模式（同步复制）:
   主端发起一个 write 到 Dorado A
     │
     ▼
   Dorado A 内部:
     1. 写到 A 的 cache
     2. 同时转发给 B
     3. 等 B 确认写到 cache
     4. 才给主端返回 ACK
     │
     ▼
   主端 write() 系统调用返回
 
 → 主端感知的"写延迟" = 包含了到 B 的往返 RTT
 → 所以 Dorado 同步的延迟其实是算在 ① + ② 一起的
 → 对 GaussDB 就是 SharedStorageWalWrite 事件的耗时
```

### 跨机房的"光纤"可能有两根不同的

```
 很多部署混用术语"光纤"，但实际上跨机房往往有两条独立的光纤:
 
 光纤 1: FC SAN 专用（Dorado HyperMetro）
   - 用 FC 协议（8/16/32 Gbps）
   - 只走块 IO
   - 延迟低、抖动小
   - 独立的 FC 交换机
 
 光纤 2: IP 以太网（TCP 复制）
   - 用以太网协议（10/25/100 Gbps）
   - 走各种业务流量 + 复制
   - 延迟抖动相对大
   - 通常共享 SDN/核心路由
 
 两条光纤可能:
   - 走同一个光缆（物理上同路径，故障一起断）
   - 走不同光缆（双路由保护，成本高）
 
 运维上要关注:
   - 两条链路的独立性（防单点）
   - 两条链路的延迟 SLA（匹配度）
   - 抖动时哪条先出问题（影响 wait 事件的分布）
```

---

## 七、关键代码锚点速查表

| 功能 | 文件 | 行号 |
|---|---|---|
| wait wal sync 设置 | syncrep.cpp | 315 |
| wait wal sync 清除 | syncrep.cpp | 505 |
| SyncRepWaitForLSN 入口 | syncrep.cpp | 230 |
| 10000 次自旋快路径 | syncrep.cpp | 282-292 |
| SSRealtimeBuildWaitForTime | syncrep.cpp | 522 |
| SyncPaxosWaitForLSN | syncrep.cpp | 1681-1794 |
| SS_DORADO_CLUSTER apply 语义 | walsender.cpp | 2987 |
| SyncRepReleaseWaiters 调用 | walsender.cpp | 2978 |
| SyncLocalXidWait | procarray.cpp | 4614 |
| wait transaction sync 设置 | procarray.cpp | 4626 |
| XLogInsert (commit record) | xact.cpp | 1706/1746 |
| XLogWaitFlush | xact.cpp | 1791 |
| SyncRepWaitForLSN 调用 | xact.cpp | 1804 |
| TransactionIdCommitTree | xact.cpp | 1811 |
| XLogWaitFlush 实现 | xlog.cpp | 3475-3516 |
| WALFlushWaitLock 等待 | xlog.cpp | 3508-3511 |
| SS_DORADO_CLUSTER 宏 | ss_disaster_cluster.h | 76 |
| SS_DORADO_PRIMARY_CLUSTER | ss_disaster_cluster.h | 80-81 |
| SS_DORADO_STANDBY_CLUSTER | ss_disaster_cluster.h | 84-85 |
| UpdateSSDoradoCtlInfoAndSync | ss_cluster_replication.cpp | 175 |

---

## 八、核心结论汇总

1. **commit 堵 insert 的 pg_thread_wait_status 模式** = 慢 commit + 唯一键 MVCC 可见性等待，不是 commit 主动持锁
2. **wait transaction sync 是 wait wal sync 的级联次生症状**：同一个延迟事件在不同线程/视图的两个切面
3. **wait wal sync 代码边界严格**：只覆盖 `syncrep.cpp:315→505` 的主等待循环，不含 10ms 自旋快路径、不含本地 flush、不含 CLOG
4. **SS_DORADO_CLUSTER 下 wait wal sync 等的是备集群 redo apply 进度**（不是 Dorado 存储同步本身）
5. **Dorado 存储同步的延迟归到 SharedStorageWalWrite**（在 walwriter 里），不在 wait wal sync 里
6. **双集群架构有 4 条物理链路**：①②③ 走 SAN 光纤（FC 块 IO），④ 走 IP 以太网（TCP 控制面）
7. **reply.apply 明确走 IP 网络**，不走 Dorado FC 光纤——这是常被混淆的一点
8. **诊断 commit 慢化要三件套联看**：wait wal sync + SharedStorageWalWrite + walFlushWaitLock
