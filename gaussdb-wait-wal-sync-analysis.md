# GaussDB wait wal sync / wait transaction sync 等待事件分析

> 讨论日期：2026-04-21 ~ 2026-04-22
> 基于 openGauss 开源源码 `~/opengauss/source` 的实际代码验证

## 一、问题背景

生产环境现象：
- `wait wal sync` 等待事件大量出现
- `wait transaction sync` 等待事件大量出现
- 对应的 `SharedStorageWalWrite` / `SharedStorageCtlInfoWriter` 等待事件**没有出现**

历史对比：之前两套事件是成对出现的。

**架构信息**：Dorado 双集群共享盘 sync 同步架构（`SS_DORADO_CLUSTER` 模式）
- 两个 GaussDB 集群（主集群 + 备集群）
- 每个集群内部是 DMS + DSS 资源池化
- 主集群挂 Dorado A 共享阵列，备集群挂 Dorado B
- Dorado A ↔ Dorado B 通过 HyperReplication 做存储层同步复制

---

## 二、源码级关键发现

### 2.1 两个事件语义不同，不是"同族"

| 事件 | 状态枚举 | 源码位置 | 真实语义 |
|------|---------|---------|---------|
| `wait wal sync` | `STATE_WAIT_WALSYNC` | `syncrep.cpp:315` (`SyncRepWaitForLSN`)<br>`syncrep.cpp:1794` (`SyncPaxosWaitForLSN`) | commit 等同步备机/Paxos 回 LSN ACK |
| `wait transaction sync` | `STATE_WAIT_XACTSYNC` | `procarray.cpp:4626` (`SyncLocalXidWait`) | 等**另一个本地事务**完成（锁可见性、DMS xid 协调） |

**关键订正**：`wait transaction sync` **不是**"commit 同步等待"，它跟 commit 链路**没有直接父子关系**，是独立旁路。

### 2.2 SharedStorageWalWrite / CtlInfoWriter 不在 openGauss 开源源码中

- 全文搜索确认：没有该事件名的枚举、没有字符串
- DSS 写 WAL 走的是 `xlog.cpp:2824-2837` 的 `WAIT_EVENT_WAL_WRITE`（显示名 `WALWrite`），包 `dss_append()` 调用
- Dorado 控制信息写入 `WriteSSDoradoCtlInfoFile`（`ss_cluster_replication.cpp:31`）是裸 `write()`，社区版**没有 emit 任何 wait event**

→ 推断：`SharedStorageWalWrite` / `SharedStorageCtlInfoWriter` 是 **GaussDB 商业版（企业版）** 在开源基础上新增的事件，用于包装这些代码段。

### 2.3 Dorado 集群宏定义（`ss_disaster_cluster.h:76`）

```cpp
#define SS_DORADO_CLUSTER \
    (ENABLE_DMS && ENABLE_DSS && g_instance.attr.attr_storage.ss_disaster_mode == SS_DISASTER_DORADO)
```

### 2.4 Commit 路径主干（`xact.cpp:1639+` → `RecordTransactionCommit`）

```
xact.cpp:1746  XLogInsert(RM_XACT_ID, XLOG_XACT_COMMIT)
                ↓ 生成 commit 记录
xact.cpp:1791  XLogWaitFlush(commitRecLSN)
                ↓ 等本地 WAL 刷盘（通过 walwriter 或自己）
xact.cpp:1804  SyncRepWaitForLSN(commitRecLSN)
                ↓ 等远端 LSN ACK（设置 STATE_WAIT_WALSYNC）
xact.cpp:1811  TransactionIdCommitTree → 更新 CLOG → 返回客户端
```

---

## 三、完整的 commit 时间轴（源码级映射）

### 3.1 各段事件对应关系

| 段 | 代码位置 | 内容 | 对应 wait event |
|----|---------|------|----------------|
| A | `xlog.cpp:1409` ReserveXLogInsertLocation | 竞争 WALInsertLock slot（N 个） | `WALInsertLock` (LWLOCK_EVENT) |
| B | 内存 memcpy 到 shared WAL buffer | CPU 操作 | 无事件 |
| C | `xlog.cpp:2426` AdvanceXLInsertBuffer | 推进 WAL 页边界时 | `WALBufMappingLock` (LWLOCK_EVENT) |
| D | `xlog.cpp:3475` XLogWaitFlush | backend 挂 `PGSemaphore` 等 walwriter | **无显式事件名** |
| E | XLogWrite 入口 | 持有 WALWriteLock | `WALWriteLock` (LWLOCK_EVENT) |
| F | `xlog.cpp:2824-2837` | `pwrite()` 或 `dss_append()` 到 DSS | 社区版：`WALWrite` (IO_EVENT)<br>企业版：**`SharedStorageWalWrite`** |
| G | `xlog.cpp:2883` issue_xlog_fsync | fsync | **无 wait event**（仅 instr_time 内部） |
| H | `ss_cluster_replication.cpp:31` WriteSSDoradoCtlInfoFile | 写 SS_DORADO_CTRL_FILE | 社区版：无<br>企业版：**`SharedStorageCtlInfoWriter`** |
| I | `syncrep.cpp:315` SyncRepWaitForLSN | WaitLatch 等 walsender 更新 WalSndCtl->lsn | **`wait wal sync`** (STATUS) |

### 3.2 旁路：wait transaction sync 独立触发路径

```
XactLockTableWait              →  procarray.cpp:4593/4605
DMS 跨节点 xid 回调            →  ss_dms_callback.cpp:192,194
Ustore 可见性判断              →  knl_uvisibility.cpp:1413
          ↓ 任一触发
SyncLocalXidWait               →  procarray.cpp:4614
  pgstat_report_waitstatus_xid(STATE_WAIT_XACTSYNC)  [procarray.cpp:4626]
  ⏱  显示为 "wait transaction sync"
```

---

## 四、F 段（SharedStorageWalWrite）精确覆盖范围

在 Dorado 双集群共享盘架构下，F 段的计时窗口：

```
backend 调用 write 前                                         write 返回
      │                                                             │
      ▼                                                             ▼
┌─────────────────────── F 段 ─────────────────────────────────────┐
│                  SharedStorageWalWrite                            │
│                                                                   │
│    DSS client → DSS server                                        │
│    → Dorado A 控制器                                              │
│    → Dorado A 写本地 cache                                        │
│    → HyperReplication 同步到 Dorado B  ← 双盘存储同步在此段内    │
│    → Dorado B 写 cache + ACK 回 A                                 │
│    → Dorado A ACK 回 DSS                                          │
│    → DSS 返回 write 成功                                          │
└───────────────────────────────────────────────────────────────────┘
```

**F 段结束 = 两个 Dorado 阵列的数据已一致 + DSS 返回成功**。

这是 Dorado 存储同步的特点：存储层双盘一致已在 F 段完成，跟传统数据库流复制不同。

---

## 五、I 段（wait wal sync）等待的对象

F 段结束后，commit 还没返回客户端的原因——**backend 挂在 `SyncRepWaitForLSN` 等某个"确认"**。

这个"某个确认"可能是下面三者之一或多者，取决于配置：

| 等待对象 | 触发条件 | 慢的原因 |
|---------|---------|---------|
| ① 备集群 `shared_storage_walreceiver` 回 ACK | 跨集群同步 RPO=0 | 备集群读 Dorado B 慢 / 备集群 redo 慢 / 跨集群控制面延迟 |
| ② 主集群内其他 DB 节点的 DMS LSN 推进 | 主集群是多节点资源池化 | DMS 消息堆积 / 节点间网络 |
| ③ DCF/Paxos 多数派共识 | `enable_dcf=on` | Paxos 组内短板节点 |

**Dorado 双集群架构下 ① 最常见。**

---

## 六、wait event 边界原则（最容易绕晕的点）

**Wait event 只记录"自己这个进程在这段代码里花了多少时间"，不跨进程、不跨节点、不跨集群。**

具体到 F 段 vs I 段：

- **`SharedStorageWalWrite`** 是主集群主节点 **backend 进程**内部的计时，严格截止到 `write()` / `dss_append()` 返回那一刻。
- 备集群的一切（walreceiver 读 Dorado B / redo / 回 ACK）都发生在这次 write 返回**之后**，由**另一个进程**在**另一台机器**上完成。
- 主库 backend 对这段时间的全部感知 = **一个事件 `wait wal sync`**——"我在等，不知道对方在干什么"。

```
t=0      t=3ms                            t=55ms
 │────────│────────── ... ─────────────────│
 │        │                                │
 │←F 段→ │                                │
 │        │←─────── I 段 ─────────────→│
 │                                         │
 │  备集群在这段时间内部:                   │
 │  读 Dorado B → redo → 回 ACK            │
 │  (这一切发生在 I 段内, 不影响            │
 │   主库的任何 wait event 计时)            │
```

**备集群慢 100% 体现在 `wait wal sync`，一毫秒也不会漏到 `SharedStorageWalWrite`。**

---

## 七、对本次现象的最终解读

### 7.1 各事件高/低的含义

| 观察 | 含义 |
|------|------|
| `SharedStorageWalWrite` / `CtlInfoWriter` **低** | 主库写 Dorado A + 双阵列存储同步 **不慢** |
| `wait wal sync` **高** | 阻塞发生在"WAL 已物理落两个阵列之后"——数据库集群层的确认机制慢 |
| `wait transaction sync` **高** | **是 `wait wal sync` 的次生症状**，其他会话读未完成 commit 的行时卡在 `SyncLocalXidWait` |

### 7.2 因果链

```
F 段 (SharedStorageWalWrite) 正常
  ↓ 说明
主库→Dorado A、Dorado A→Dorado B 存储写入链路没瓶颈
  ↓ 所以
WAL 数据已经物理存在于两个阵列上
  ↓ 但 commit 还没返回, 因为
I 段 (wait wal sync) 慢
  ↓ 说明
数据库集群层的"确认机制"慢:
  - 备集群的 walreceiver 还没从 Dorado B 把 WAL 读进来
  - 或读了但 redo 没跟上
  - 或 redo 了但 ACK 回主集群的控制面链路有延迟
  - 或主集群内 DMS 跨节点协调慢
  ↓ 连锁反应
其他会话读"已写 WAL 但未 commit"事务的行 → wait transaction sync 飙升
```

### 7.3 "wait wal sync 和 SharedStorageWalWrite 不重合" 的结论

**在同一个 commit 的时间轴上，两个事件严格顺序、零交叠**：

```
┌── F 段 ──┐ ┌────── I 段 ──────┐
│ SharedStg │ │  wait wal sync   │
│ WalWrite  │ │                  │
└───────────┘ └──────────────────┘
       │              ↑
       │   F 段结束那一刻 = I 段开始前一刻
       │
       ▼
  write() 返回
  = WAL 已物理写入两个 Dorado 阵列
```

**注意**：在累计视图（`dbe_perf.wait_events`）里看到两者 `total_wait_time` 都高，是因为系统里**有大量并发 commit 在流水线上**——任一时间点，有的 backend 在 F、有的在 I，分别累加。**"都高" ≠ "时间重叠"**。

---

## 八、下一步排查建议

### 8.1 确认架构与模式

```sql
SHOW ss_disaster_mode;              -- 预期 dorado
SHOW ss_enable_dms;                 -- on
SHOW ss_enable_dss;                 -- on
SHOW enable_dcf;                    -- 判断是否走 Paxos 路径
SHOW ss_enable_ondemand_realtime_build;  -- RTO 模式会走 SSRealtimeBuildWaitForTime 旁路
```

### 8.2 拉累计等待事件（而非快照）

```sql
SELECT event, wait AS count, total_wait_time,
       total_wait_time / NULLIF(wait, 0) AS avg_us
FROM dbe_perf.wait_events
WHERE event IN ('SharedStorageWalWrite',
                'SharedStorageCtlInfoWriter',
                'wait wal sync',
                'wait transaction sync',
                'WALWrite', 'WALWriteLock', 'WALInsertLock')
ORDER BY total_wait_time DESC;
```

### 8.3 DMS 跨节点协调延迟

```sql
SELECT event, wait, total_wait_time,
       total_wait_time / NULLIF(wait,0) AS avg_us
FROM dbe_perf.wait_events
WHERE type = 'DMS_EVENT'
ORDER BY total_wait_time DESC LIMIT 20;
-- 关注: DCS_TRANSFER_PAGE, TXN_REQ_INFO, LATCH_X_REMOTE, BROADCAST_* 等
```

### 8.4 备集群 walreceiver 延迟（在备集群执行）

```sql
SELECT * FROM pg_stat_get_wal_receiver();
-- 或看 walreceiver 进程的 pg_stat_activity.wait_event
```

### 8.5 主备 LSN 追赶

```sql
SELECT application_name, sync_state,
       pg_wal_lsn_diff(sent_lsn, flush_lsn) AS flush_lag_bytes,
       pg_wal_lsn_diff(sent_lsn, replay_lsn) AS replay_lag_bytes
FROM pg_stat_replication;
```

### 8.6 定位路径决策树

```
wait wal sync / wait transaction sync 升高
          │
          ├─ SharedStorageWalWrite 累计时长也升高
          │   → 存储/HyperMetro 链路问题 (本次不是)
          │
          └─ SharedStorageWalWrite 基本不变/不出现
                    → DB/集群间问题:
                    │
                    ├─ DMS/IPC 类等待飙升
                    │   → 跨节点协调 (主集群内)
                    │
                    ├─ 备集群 walreceiver/redo 慢
                    │   → 跨集群控制链路
                    │
                    ├─ WAL*Lock 飙升
                    │   → WAL 锁争抢 (WAL 生成速率过高)
                    │
                    └─ 都不明显
                        → 看 commit_delay、大事务、日志量激增
```

---

## 九、常见误解订正

### 9.1 误解一："wait wal sync 是 commit 全周期等待，包含 SharedStorageWalWrite"

❌ **错误**：把 `wait wal sync` 当成伞状事件，假设它覆盖从 WAL 生成到 commit 返回的所有时间

✅ **正确**：在 openGauss/GaussDB 里等待事件是**平铺**的，不像 Oracle 有父子层级

- `wait wal sync` 只覆盖 `SyncRepWaitForLSN`（`syncrep.cpp:230-516`）这一段，也就是"WAL 已落盘，等对端 LSN ACK"
- 它**不包含**前面的 WAL 生成、WAL 写入、fsync 等任何阶段
- 同一进程上的等待事件不能嵌套——`pgstat_report_waitevent(X)` 会覆盖之前的值

### 9.2 误解二："等待事件完全互斥，不能并存"

❌ **错误**：认为一个 backend 在同一时刻只能有一个等待

✅ **正确**：GaussDB 的 backend 状态结构体 `PgBackendStatus` 有**三个独立字段**同时记录：

```cpp
pgstat.h:1667    WaitState      st_waitstatus;        // 等待状态 (coarse)
pgstat.h:1672    uint32         st_waitevent;         // 等待事件 (fine)
pgstat.h:1675    WaitStatePhase st_waitstatus_phase;  // 状态子阶段
```

| 维度 | 字段 | 示例 | 维度内互斥？ | 跨维度可否共存？ |
|------|------|------|------------|---------------|
| WaitState | `st_waitstatus` | `wait wal sync`, `wait transaction sync` | ✅ 是 | — |
| WaitEvent | `st_waitevent` | `WALWrite`, `WALInsertLock`, `SharedStorageWalWrite`（企业版）| ✅ 是 | — |
| WaitStatePhase | `st_waitstatus_phase` | 仅用于 "wait node" 的分 phase | ✅ 是 | — |
| 三者之间 | — | — | — | ✅ **可以共存** |

**判别方法**：
- 名字形如 `"wait xxx"`（小写+空格+含 wait 前缀）→ `WaitState`，从 `STATE_WAIT_*` 枚举
- 名字形如 `"WALWrite"`, `"SharedStorageWalWrite"`（驼峰）→ `WaitEvent`，从 `WAIT_EVENT_*` 枚举

### 9.3 误解三："两个事件时间不重合"——需要分层理解

❌ **过度简化**："F 段和 I 段时间上不重合"

✅ **更准确的三段式表述**：

| 视角 | 是否重合 | 原因 |
|------|---------|------|
| **同一个 commit 的因果链** | 不重合 | F 严格在 I 之前，因果依赖 |
| **同一个 commit 的物理执行** | 不重合 | F 在 walwriter 进程、I 在 backend 进程，状态字段不互相干扰 |
| **系统聚合视角（`dbe_perf.wait_events` / 挂钟时间）** | **重合** | 不同 commit 的 F 和 I 在流水线上并行，累计时长当然会共同存在 |

所以 "`SharedStorageWalWrite` 和 `wait wal sync` 在累计视图里都高" **不代表** 单次 commit 同时有这两个事件——只是系统里有很多 commit 在不同阶段。

### 9.4 Backend 的"统计黑洞"——等 walwriter 那段时间没有 wait event

这是单看 backend 视角时最容易漏掉的一块，**直接影响用 wait event 做总耗时分解时的账目对不上**。

**源码证据**（`xlog.cpp:3505-3513` 的 `XLogWaitFlush`）：

```cpp
while (flushTo < recptr) {
    if (!isWalWriterUp)
        XLogSelfFlush();
    else if (LWLockAcquireOrWait(walFlushWaitLock.l.lock, LW_EXCLUSIVE)) {
        PGSemaphoreLock(&walFlushWaitLock.l.sem, true);   // 【关键】裸 PGSemaphoreLock
        LWLockRelease(walFlushWaitLock.l.lock);
    }
}
```

注意 `PGSemaphoreLock(&walFlushWaitLock.l.sem, ...)` **前后没有任何 `pgstat_report_waitevent` 包装**。

**Backend 在等 walwriter 期间的 wait_event 状态演化：**

| 阶段 | 占时比例 | `st_waitevent` | `st_waitstatus` |
|------|---------|---------------|----------------|
| ①  LWLockAcquireOrWait 争 walFlushWaitLock | ~微秒级 | `WALFlushWait` (LWLOCK_EVENT) | 不变 |
| ②  PGSemaphoreLock 睡等 walwriter 唤醒 | **占 99% 时长** | **`WAIT_EVENT_END`（空）** | 不变（通常 RUNNING）|

其中：
- 步骤 ① 的 `WALFlushWait` 来自 `LWLockAcquireOrWait` 内部 `pgstat_report_waitevent(PG_WAIT_LWLOCK \| lock->tranche)`（`lwlock.cpp:1746`），tranche 显示名是 `WALFlushWait`（`lwlock.cpp:196`）
- 步骤 ① 结束时调用 `pgstat_report_waitevent(WAIT_EVENT_END)` 清空事件（`lwlock.cpp:1797`）
- 步骤 ② 的 `PGSemaphoreLock` 是裸调用，**不会 emit 任何事件**

**对照 walwriter 和 backend 的精确时间轴：**

```
挂钟时间 →

walwriter:   ...idle...|═════ F段 (SharedStorageWalWrite) ═════|...idle...
                        ↑ pgstat_report_waitevent(WAIT_EVENT_WAL_WRITE)
                        │     [xlog.cpp:2824]                 │
                        │                                     │
backend                 │                                     │
 (同一 commit):          │                                     │
步骤①                   ├|短暂 WALFlushWait|                    │
                        │  (LWLock 争抢)                        │
                        │                                      │
步骤②                   │              ├════ 无 wait_event ════|
                        │              │  st_waitevent=END      │
                        │              │  st_waitstatus=RUNNING │
                        │              │  PGSemaphoreLock 在睡  │
                        │                                      │
                        │                                 walwriter 唤醒
                        │                                      │
步骤③                                                          ├─| I 段 wait wal sync
```

**现象：`pg_stat_activity` 看到 backend 在 commit 但 `wait_event` 为 NULL**

这是因为：
- `state = 'active'`（有一个 commit 在执行）
- `wait_event_type` / `wait_event` = NULL（上面步骤 ②）
- 实际状态是"在 PGSemaphoreLock 上睡等 walwriter 完成 WAL 写入"

**这是 GaussDB 的刻意设计：不重复记账**

- walwriter 进程记录 WAL 写入耗时（`WAIT_EVENT_WAL_WRITE` / 企业版 `SharedStorageWalWrite`）
- Backend 进程不记"等 WAL 写完"的时间（避免同一段时间被两个进程重复计入）
- 要得到完整 commit 时间分布，**必须同时看 walwriter 和 backend 两侧的事件**

### 9.5 完整的 commit 时间账户（backend 视角）

单看 backend 的事件累计，**无法 100% 凑齐 commit 总耗时**。账目关系如下：

```
commit 总耗时（从客户端发 commit 到收到 ACK）
│
├─ [backend 视角可见]
│   ├─ WALInsertLock 等待              (LWLOCK_EVENT)
│   ├─ WALBufMappingLock 等待           (LWLOCK_EVENT, 偶发)
│   ├─ WALFlushWait 短暂锁争抢           (LWLOCK_EVENT, 微秒级)
│   ├─ wait wal sync                   (STATUS)  ← I 段
│   └─ 其他零碎 (CLOG 更新等)
│
└─ [backend 视角【不可见】——统计黑洞]
    └─ PGSemaphoreLock 等 walwriter     (无事件)
         ↕ 真实耗时映射
    walwriter 进程的:
         ├─ WALWriteLock                (LWLOCK_EVENT)
         ├─ WALWrite / SharedStorageWalWrite   (IO_EVENT)
         ├─ issue_xlog_fsync             (无事件)
         └─ SharedStorageCtlInfoWriter   (IO_EVENT, 企业版)
```

**实用建议**：做 commit 耗时分解时，把 `SharedStorageWalWrite`（来自 walwriter 进程累计）直接加到 backend 侧的账目里——那就是 backend 里"PGSemaphoreLock 黑洞段"的实际耗时近似值。

---

## 十、关键源码位置索引

| 主题 | 文件:行号 |
|------|----------|
| RecordTransactionCommit 主干 | `src/gausskernel/storage/access/transam/xact.cpp:1639+` |
| commit 记录生成 | `xact.cpp:1746` |
| 本地 flush 等待 | `xact.cpp:1791` → `xlog.cpp:3475` (XLogWaitFlush) |
| 远端 LSN 同步等待 | `xact.cpp:1804` → `syncrep.cpp:230` (SyncRepWaitForLSN) |
| STATE_WAIT_WALSYNC 设置点 | `syncrep.cpp:315`, `syncrep.cpp:1794` |
| STATE_WAIT_XACTSYNC 设置点 | `procarray.cpp:4626` (SyncLocalXidWait) |
| Dorado RTO 特殊路径 | `syncrep.cpp:232-234` → `syncrep.cpp:522` (SSRealtimeBuildWaitForTime) |
| WAL buffer 预留位置 | `xlog.cpp:1409` (ReserveXLogInsertLocation) |
| WAL buffer 前进 | `xlog.cpp:2426` (WALBufMappingLock 获取) |
| 实际 WAL 写入 | `xlog.cpp:2824-2837` (WAIT_EVENT_WAL_WRITE 包 pwrite/dss_append) |
| fsync 调用 | `xlog.cpp:2883` (issue_xlog_fsync) |
| Dorado 控制信息同步 | `xlog.cpp:2896` → `ss_cluster_replication.cpp:175` (UpdateSSDoradoCtlInfoAndSync) |
| Dorado 控制文件写 | `ss_cluster_replication.cpp:31` (WriteSSDoradoCtlInfoFile) |
| SS_DORADO_CLUSTER 宏定义 | `src/include/replication/ss_disaster_cluster.h:76` |
| wait event 显示名映射 | `src/gausskernel/cbb/instruments/ash/wait_event_info.cpp` |
| 状态名映射 | `src/common/backend/utils/adt/pgstatfuncs.cpp:355-362` |
| WaitEventIO 枚举 | `src/include/pgstat.h:1320-1362` |
| WaitEventDMS 枚举 | `src/include/pgstat.h:1364-1412` |
| XLogWaitFlush 实现 | `xlog.cpp:3475-3516` |
| LWLockAcquireOrWait 包事件 | `lwlock.cpp:1743-1797` |
| WALFlushWait tranche 名 | `lwlock.cpp:196` (`LWTRANCHE_WAL_FLUSH_WAIT`) |
| PgBackendStatus 三字段 | `src/include/pgstat.h:1667, 1672, 1675` |
