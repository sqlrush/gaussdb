# GaussDB 等待事件扩展分析（对 wait-wal-sync 分析的补充）

> 在 `gaussdb-wait-wal-sync-analysis.md` 基础上追加的扩展发现，
> 重点覆盖：commit 阻塞 insert 模式、wait wal sync 代码级精确边界、
> SS_DORADO_CLUSTER 下 wait wal sync 的真实语义、以及双集群网络物理架构。

---

## 一、pg_thread_wait_status 里"commit 堵 insert"的根因

### 最常见场景：唯一键冲突 + 慢 commit

```
线程 A (commit)
 ├─ INSERT INTO t(id,...) VALUES (1001, ...)
 ├─ 写入成功，id=1001 在内存中
 └─ COMMIT  ← 卡在 wait wal sync / wait transaction sync

线程 B (insert)
 └─ INSERT INTO t(id,...) VALUES (1001, ...)  ← 同一个 id
     ├─ PG 发现 id=1001 有 pending 行
     ├─ 但不知道 A 会 commit 还是 rollback
     └─ 等 A 的 XID 最终化 → transactionid 锁等待

pg_thread_wait_status 表现:
 B 的 locktag 类型 = transactionid
 B 的 locktag_field1 = A 的 xid
 B 的 wait_event = wait transactionid（lock wait）
 B 的 blocking_pid = A 的 pid
```

### 本质：MVCC 可见性等待，不是 A 主动持锁

- commit A 并没有拿住某把行锁不放
- B 是 PG 的 MVCC 机制强制要等 A 的 XID 状态（CLOG）落定
- A 的 XID 落定取决于 commit 完成

### 在 Dorado sync 场景下为何高发

```
正常环境（本地 SSD）:
  commit ~50μs → 几乎不会命中唯一键冲突等待

Dorado sync 双集群:
  commit 1-5ms + 高并发小事务
  → 同一 unique key 并发概率飙升
  → pg_thread_wait_status 大量 "commit 线程堵 insert 线程"

本质:
  - 不是 commit 真的拿锁
  - 是 commit 慢到 insert 还没等到它敲定
  - 根因回到 wait wal sync / Dorado sync 链路
```

### 其他可能根因（非主流）

1. 外键引用（FK 父行 commit 中）
2. 表级 DDL 冲突（ALTER 中）
3. CSN 可见性判定
4. 子事务父事务状态判定
5. 2PC PREPARE 窗口期
6. 分布式版 GTM/CSN 协调

在集中式版 + Dorado sync + 高并发小事务支付场景下，**①（唯一键冲突）压倒性主因，占 90%+**。

### 诊断 SQL

```sql
-- 看阻塞关系
SELECT w.pid AS blocked_pid, w.query_id, w.lockmode, w.locktag,
       w.wait_status, w.wait_event,
       h.pid AS holder_pid, h.query_id, h.wait_status, h.wait_event
FROM pg_thread_wait_status w
LEFT JOIN pg_thread_wait_status h ON h.pid = w.blocking_pid
WHERE w.wait_status = 'wait transactionid'
   OR w.locktag LIKE 'transactionid%';

-- 看 commit 线程在等什么
SELECT pid, usename, wait_status, wait_event, query
FROM pg_stat_activity
WHERE state = 'active'
  AND (query LIKE 'COMMIT%' OR wait_event LIKE '%wal%sync%');
```

### locktag 类型判别

| locktag 开头 | 含义 |
|---|---|
| transactionid | 等 xid 最终化（MVCC 唯一约束等待）← 场景 ① |
| tuple | 行级锁（UPDATE/DELETE 才会） |
| relation | 表级锁（DDL 冲突） |
| advisory | 应用显式锁 |
| virtualxid | 等 backend 事务结束（VACUUM/REINDEX） |

---

## 二、wait transaction sync 是 commit 慢的次生症状

### 完整因果链

```
Dorado 存储同步 RTT 1ms
         ↓
walwriter 单线程 1000 IOPS 天花板
         ↓
commit 线程（事务 A）卡在 SyncRepWaitForLSN ──► ① wait wal sync
         │
         │ A 的 xid 长时间未写入 CLOG
         ↓
事务 B 的 INSERT 做唯一键检查
         ├─ 发现有 xid=A 的 pending 行
         ├─ XactLockTableWait(A.xid)        ──► pg_thread_wait_status
         │                                       locktag=transactionid
         └─ SyncLocalXidWait(A.xid)          ──► ② wait transaction sync
             procarray.cpp:4614-4626              （insert 线程被动等）
```

### 代码锚点

```
wait wal sync           = STATE_WAIT_WALSYNC
                          syncrep.cpp:315
                          调用点: xact.cpp:1804 SyncRepWaitForLSN
                          → commit 路径主角

wait transaction sync   = STATE_WAIT_XACTSYNC
                          procarray.cpp:4626
                          调用点: SyncLocalXidWait
                          → 任何线程检查"某 xid 是否 commit"

transactionid 锁        = XactLockTableWait
                          insert 做 MVCC 唯一性验证
```

### 为什么 wait transaction sync 次数通常比 wait wal sync 还多

```
1 个慢 commit 可以级联拖住 N 个 insert:

事务 A (commit 慢 2ms)    → 1 次 wait wal sync
 ├─ 事务 B 撞键 → wait xact sync
 ├─ 事务 C 撞键 → wait xact sync   
 ├─ 事务 D 撞键 → wait xact sync   N 次 wait transaction sync
 └─ 事务 E 撞键 → wait xact sync
```

### 诊断判别

| 观察 | 是否符合"慢 commit 级联" |
|---|---|
| wait wal sync 和 wait transaction sync 同时高 | ✓ |
| wait transaction sync 次数 ≈ wait wal sync 的 2-10 倍 | ✓ |
| pg_thread_wait_status 大量 locktag=transactionid | ✓ |
| 被阻塞方 SQL 以 INSERT/UPSERT 为主 | ✓ |
| 阻塞方在 COMMIT 或 wait_event=wait wal sync | ✓ |
| 业务表有 UNIQUE / PRIMARY KEY 约束 | ✓ |
| 复制链路或 DMS 响应时间增大 | ✓ |

5 项以上吻合 → 确认就是慢 commit 引发的级联。

### 治标 vs 治本

| 治标（缓解 insert 被堵） | 治本（加速 commit） |
|---|---|
| 业务拆热点键 | commit_delay 合并小 IO |
| 应用层去重 | synchronous_commit=off（弱一致业务）|
| 拆小事务 | 升级 Dorado 阵列性能 |
| 主键加时间戳前缀 | 减少单事务 WAL 量 |
| lock_timeout 防永挂 | 长期: 多 walwriter / WAL 写路径优化 |

---

## 三、wait wal sync 的代码级精确边界

### commit 完整代码路径（xact.cpp:1443 RecordTransactionCommit）

```
段 A: 生成 commit xlog record
  xact.cpp:1706/1746
  commitRecLSN = XLogInsert(RM_XACT_ID, XLOG_XACT_COMMIT, true);
  等待事件可能: WALInsertLock (LWLock)
  ✗ 不是 wait wal sync

段 B: 本地 WAL flush 等待
  xact.cpp:1791
  XLogWaitFlush(commitRecLSN);
  内部 (xlog.cpp:3475-3516):
    while (flushTo < recptr) {
      LWLockAcquireOrWait(walFlushWaitLock, LW_EXCLUSIVE)
      PGSemaphoreLock(&walFlushWaitLock->sem, true)
      LWLockRelease
    }
  等待事件: walFlushWaitLock + PGSemaphoreLock（backend 黑洞段）
  实际等的是 walwriter 把 WAL buffer 写到共享存储 + fsync
  在 Dorado 下等的是 SharedStorageWalWrite 完成
  ✗ 不是 wait wal sync

段 C: 远程同步等待 ★ wait wal sync ★
  xact.cpp:1804
  if (synchronous_commit > LOCAL_FLUSH) {
      SyncRepWaitForLSN(commitRecLSN, ...);
  }
  ✅ 唯一计入 wait wal sync 的区段

段 D: CLOG 提交
  xact.cpp:1811
  TransactionIdCommitTree(xid, nchildren, children, GetCommitCsn());
  等待事件: CLogControlLock + CSNBufMappingLock
  ✗ 不是 wait wal sync
```

### SyncRepWaitForLSN 内部 5 个子段（syncrep.cpp:230-516）

```
段 1: 快速退出检查 (syncrep.cpp:232-276)
  - SS_PRIMARY_ENABLE_TARGET_RTO → SSRealtimeBuildWaitForTime
  - ENABLE_DMS && !SS_STREAM_CLUSTER → NOT_REQUEST
  - !SyncRepRequested / !SyncStandbysDefined → 早退
  - !SynRepWaitCatchup → NOT_WAIT_CATCHUP
  ✗ wait_event 未改变

段 2: 乐观自旋快路径 (syncrep.cpp:282-292) ⚠️ 观测盲区
  #define SYNCREPWAIT_TRY_TIMES 10000
  for (int tryTime = 0; tryTime < 10000; tryTime++) {
      loopLSN = atomic_read(WalSndCtl->lsn[mode]);
      if (XLByteLE(XactCommitLSN, loopLSN)) return REPSYNCED;
      pg_usleep(1);   // 睡 1μs
  }
  最多 ~10ms，此时 wait_event 还没设成 STATE_WAIT_WALSYNC
  → 快 commit (<10ms 同步) 根本不显示 wait wal sync!

段 3: ps display 设置 (syncrep.cpp:295-313)
  ✗ 不影响 wait_event

段 4: ★ wait wal sync 正式开始 (syncrep.cpp:315-503) ★
  起点: pgstat_report_waitstatus(STATE_WAIT_WALSYNC)
  while (true) {
      if (XLByteLE(XactCommitLSN, remoteLSN)) break;
      ...中断检查...
      if (LWLockAcquireOrWait(walSyncRepWaitLock, LW_EXCLUSIVE)) {
          WaitSndCtl->syncWaitProc = t_thrd.proc;
          ResetLatch(&procLatch);
          WaitLatch(&procLatch, ..., 1000L);  // 1s 超时
          LWLockRelease(walSyncRepWaitLock);
      }
      remoteLSN = atomic_read(WalSndCtl->lsn[mode]);
  }

段 5: wait wal sync 结束 (syncrep.cpp:505)
  终点: pgstat_report_waitstatus(oldStatus);
```

### 精确定义

```
wait wal sync 覆盖范围:
  起点 = syncrep.cpp:315
  终点 = syncrep.cpp:505
  
语义:
  "commit record 已写入本地 WAL buffer 并完成本地 flush 后，
   等待备机（或 DCF Paxos 大多数）确认接收并持久化该 LSN 的时间段"
```

### 三种进入分支

| 函数 | wait wal sync 等的是 |
|---|---|
| SyncRepWaitForLSN (syncrep.cpp:230) | walsender 收到备机 ACK |
| SSRealtimeBuildWaitForTime (syncrep.cpp:522) | Dorado 备集群 redo 追赶 RTO 窗口 |
| SyncPaxosWaitForLSN (syncrep.cpp:1681) | DCF Paxos 过半数确认 |

### 不覆盖的部分（重要）

```
commit 总耗时 = t_A + t_B + t_C + t_D

段A  XLogInsert           → WALInsertLock
段B  XLogWaitFlush         → walFlushWaitLock + PGSemaphoreLock (黑洞段)
                           → 间接反映 SharedStorageWalWrite
段C  SyncRepWaitForLSN     → ★ wait wal sync
段D  TransactionIdCommitTr → CLogControlLock

⚠️ wait wal sync 只能告诉你 t_C，看不到 t_A / t_B / t_D
```

### 观测意义（对 LAS 采样）

```
假设 commit 总耗时 20ms:
  t_A: XLogInsert         = 0.1 ms
  t_B: XLogWaitFlush      = 5.0 ms
  t_C: SyncRepWaitForLSN  = 14.5 ms
    ├─ 前 10ms 自旋不计 wait_event
    └─ 后 4.5ms 计为 wait wal sync
  t_D: TransactionIdCommit = 0.4 ms

LAS 1Hz 采样:
  命中 wait wal sync 的概率 ≈ 4.5/1000 = 0.45%
  按"命中 × 1s"估算严重低估（前 10ms 自旋阶段不显示）
  需结合 SharedStorageWalWrite 联合观测
```

---

## 四、SS_DORADO_CLUSTER 下 wait wal sync 的真实语义

### 关键认知：架构里有两个独立的 sync 边界

```
边界 1：Dorado 存储级同步（HyperMetro/HyperReplication）
  位置: walwriter 线程，pwrite → DSS → Dorado A → Dorado B
  事件: SharedStorageWalWrite
  对 GaussDB 透明，在 t_B (XLogWaitFlush) 内

边界 2：GaussDB 复制协议确认
  位置: backend 线程，SyncRepWaitForLSN
  事件: ★ wait wal sync ★
  等备集群 GaussDB 回放并通过 TCP 回传 reply.apply
```

### 关键源码证据（walsender.cpp:2987）

```cpp
/*
 * 1. When starting ss dorado replication, we need to know replayPtr that
 *    standby has already replayed, because primary xlog will cover standby
 *    xlog by Dorado synchronous replication.
 */
if (SS_DORADO_CLUSTER) {
    AdvanceReplicationSlot(reply.apply);   // ★ 用 apply（回放位点）
} else {
    AdvanceReplicationSlot(reply.flush);   // 普通流复制用 flush（落盘位点）
}
```

**SS_DORADO_CLUSTER 特殊语义**：
- 普通流复制：备机 ACK 的是 flush position（收到落盘即可）
- SS_DORADO_CLUSTER：备机 ACK 的是 apply position（必须回放完才算）
- 原因：Dorado 同步会让主端 WAL 直接覆盖备端 DSS，"存储已同步"≠"数据库已恢复一致"，必须等回放追上才能保证故障切换不丢

### wait wal sync 在 SS_DORADO_CLUSTER 下具体等什么

```
主集群 backend 的 wait wal sync 等的是：

备集群 Primary 节点完成以下步骤:
  ① shared_storage_walreceiver 从 DSS B 读到新 WAL
  ② 转交 startupProcess
  ③ redo apply 回放
  ④ 更新自身 apply LSN
  ⑤ 经 walsender→walreceiver 通道通过 TCP 回传主集群
  ⑥ 主集群 walsender 更新 WalSndCtl->lsn[mode]
  ⑦ 唤醒主 backend 的 WaitLatch
```

### wait wal sync 和 Dorado 的关系：间接

```
直接关系: ❌
  wait wal sync 代码不调任何 Dorado API
  唤醒条件是 WalSndCtl->lsn[mode] 被更新
  更新来源是备集群 walreceiver 的 TCP ACK
  Dorado 阵列本身不参与这个协议

间接关系: ✅
  wait wal sync 结束依赖"备集群看到 WAL"
  备集群通过共享存储看 WAL
  共享存储同步 = Dorado HyperMetro 的职责
  Dorado 慢 → 备端 DSS 看不到新 WAL
         → walreceiver 读不到
         → 回放不动
         → apply LSN 不推进
         → wait wal sync 被动拉长
```

### 三种可能的瓶颈 + 诊断信号

```
情况 ①: 备集群 GaussDB 回放慢（最常见）
  wait wal sync 高 + SharedStorageWalWrite 正常
  原因: 备集群硬件低 / recovery_parse_workers 不够 / 大事务
  和 Dorado 关系: 无

情况 ②: 备集群 walreceiver 读慢
  wait wal sync 高 + SharedStorageWalWrite 正常
  备端对 Dorado B 读 IO 延迟高
  和 Dorado 关系: 部分相关（备端 Dorado 读性能）

情况 ③: Dorado HyperMetro 慢
  wait wal sync 高 + ★ SharedStorageWalWrite 同时高 ★
  + SharedStorageCtlInfoWriter 可能高
  原因: 双 Dorado 链路带宽不足 / RTT 增大 / 硬件降级
  和 Dorado 关系: 直接相关
```

### 判别矩阵

| wait wal sync | SharedStorageWalWrite | 根因 |
|---|---|---|
| 高 | 正常 | ① 备集群回放/读慢 |
| 高 | 高 | ③ Dorado 本身慢 |
| 正常 | 高 | 主端存储/walwriter慢 |
| 低 | 低 | commit 链路非瓶颈 |

---

## 五、双集群网络物理架构

### 两张完全独立的网络

```
┌─────────── 主机房 ───────────┐  ┌────── 备机房 ──────┐
│                              │  │                    │
│  主 GaussDB                   │  │  备 GaussDB         │
│  ┌──────────┐                 │  │ ┌──────────┐       │
│  │ backend  │                 │  │ │walreceive│       │
│  │ walwriter│                 │  │ │startup   │       │
│  │ walsender│◄──④ IP/TCP─────┼──┼─│  回放     │       │
│  └────┬─────┘                │  │ └─────┬────┘        │
│       │ ①                    │  │       │ ③          │
│       │ FC SAN               │  │       │ FC SAN     │
│       │ (本机房)              │  │       │ (本机房)    │
│       ▼                      │  │       ▼            │
│  ┌──────────┐                │  │ ┌──────────┐       │
│  │ Dorado A │ ② HyperMetro   │  │ │ Dorado B │       │
│  │          │═══════════════════│═│          │       │
│  │          │  专用 FC / iSCSI  │ │          │       │
│  │          │   跨机房光纤      │  │ │          │       │
│  └──────────┘                 │  │ └──────────┘       │
└──────────────────────────────┘  └────────────────────┘

SAN 网络 ①②③ = 块级 IO（Dorado 同步走这里）
IP 网络 ④ = 应用层协议（reply.apply 走这里）
```

### 四条链路的对应

| 段 | 描述 | 代码对应 | 物理链路 |
|---|---|---|---|
| ① 主 GaussDB → 主 Dorado | 主端 walwriter pwrite | xlog.cpp:2824 附近 | 主机房 FC SAN |
| ② 主 Dorado → 备 Dorado | HyperMetro sync | Dorado 硬件协议 | 专用跨机房 FC/iSCSI |
| ③ 备 GaussDB 读备 Dorado | shared_storage_walreceiver | 备集群 DSS 读 | 备机房 FC SAN |
| ④ reply.apply 回传 | walreceiver→walsender | TCP + pg-wire | 跨机房 IP 网络 |

### 一次 commit 的延迟贡献

```
主 backend 生成 commit record
   ↓
① 主机房 FC SAN: ~50-200 μs
   ↓ walwriter pwrite
② 跨机房 Dorado FC: ~500μs-1ms  ← 物理光速下限
   ↓ Dorado HyperMetro sync
Dorado B ACK，pwrite 完成
   ↓
(主 backend 开始等 wait wal sync)
   ↓
③ 备机房 FC SAN: ~50-200 μs
   ↓ walreceiver 读 DSS
备端回放（CPU 密集）
   ↓
apply LSN 推进
   ↓
④ 跨机房 IP 网络: ~200-500 μs  ← 可能比 Dorado FC 高
   ↓ TCP reply.apply
主端 walsender 更新 lsn[mode]
   ↓
主 backend 的 wait wal sync 唤醒
```

### 两张网络的协议/用途对比

| 维度 | SAN（存储网络） | IP 网络 |
|---|---|---|
| 用途 | 块级 IO 读写 + Dorado 同步 | walsender/walreceiver 协议 |
| 协议 | SCSI / NVMe-oF / HyperMetro | TCP + libpq + pg-wire |
| 可见性 | GaussDB 透明（OS multipath） | GaussDB 直接使用 |
| 管理 | 存储管理员，Dorado 控制台 | DBA，pg_hba/replconninfo |
| 承载数据 | WAL 文件内容、数据文件 | 握手/心跳/reply.apply/控制面 |

### 为什么必须有两张网（SS_DORADO_CLUSTER 仍需 IP）

1. **Dorado 是单向数据搬运**：无法回传"我已回放到哪里"这种数据库语义
2. **需要握手和心跳**：walreceiver 启动要认证、协商、保活
3. **sync commit 需要端到端信号**：wait wal sync 必须等 TCP 上的 reply.apply

### 常见部署的坑

```
坑 ①: IP 复制和业务网络混用
  业务高峰 → TCP 延迟抖动 → reply.apply 慢 → wait wal sync 随 QPS 波动
  解法: 专用 VLAN / 独立物理网卡

坑 ②: SAN 和 IP 链路延迟不匹配
  Dorado 同城 FC 1ms，IP 走 SDN 可能 5ms
  → wait wal sync > SharedStorageWalWrite（反直觉）
  解法: 两套网络 SLA 对齐

坑 ③: IP 断开但 SAN 还通
  Dorado 继续同步，但 walsender 断连
  → wait wal sync 永远等不到 ACK → 主集群全部阻塞
  解法: synchronous_standby_names 多备 + sync_master_standalone + CM 监控
```

---

## 六、关键代码锚点速查表

| 功能 | 文件 | 行号 |
|---|---|---|
| wait wal sync 设置 | syncrep.cpp | 315 |
| wait wal sync 清除 | syncrep.cpp | 505 |
| SyncRepWaitForLSN 入口 | syncrep.cpp | 230 |
| 10000 次自旋快路径 | syncrep.cpp | 282-292 |
| SSRealtimeBuildWaitForTime | syncrep.cpp | 522 |
| SyncPaxosWaitForLSN | syncrep.cpp | 1681-1794 |
| SS_DORADO_CLUSTER apply 语义 | walsender.cpp | 2987 |
| SyncLocalXidWait | procarray.cpp | 4614 |
| wait transaction sync 设置 | procarray.cpp | 4626 |
| XLogInsert (commit record) | xact.cpp | 1706/1746 |
| XLogWaitFlush | xact.cpp | 1791 |
| SyncRepWaitForLSN 调用 | xact.cpp | 1804 |
| TransactionIdCommitTree | xact.cpp | 1811 |
| XLogWaitFlush 实现 | xlog.cpp | 3475-3516 |
| SS_DORADO_CLUSTER 宏 | ss_disaster_cluster.h | 76 |
| UpdateSSDoradoCtlInfoAndSync | ss_cluster_replication.cpp | 175 |

---

## 七、核心结论汇总

1. **commit 堵 insert 的 pg_thread_wait_status 模式** = 慢 commit + 唯一键 MVCC 可见性等待，不是 commit 主动持锁
2. **wait transaction sync 是 wait wal sync 的级联次生症状**：同一个延迟事件在不同线程/视图的两个切面
3. **wait wal sync 代码边界严格**：只覆盖 `syncrep.cpp:315→505` 的主等待循环，不含 10ms 自旋快路径、不含本地 flush、不含 CLOG
4. **SS_DORADO_CLUSTER 下 wait wal sync 等的是备集群 redo apply 进度**（不是 Dorado 存储同步本身）
5. **Dorado 存储同步的延迟归到 SharedStorageWalWrite**（在 walwriter 里），不在 wait wal sync 里
6. **双集群架构有 4 条物理链路**：①②③ 走 SAN 光纤（FC 块 IO），④ 走 IP 以太网（TCP 控制面）
7. **reply.apply 明确走 IP 网络**，不走 Dorado FC 光纤——这是常被混淆的一点
8. **诊断 commit 慢化要三件套联看**：wait wal sync + SharedStorageWalWrite + walFlushWaitLock
