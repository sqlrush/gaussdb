# GaussDB Dorado 双集群同步等待事件全集

> 把 Dorado 双集群（SS_DORADO_CLUSTER）共享盘 sync 同步架构下，
> 一次 commit 涉及的**全部等待事件**统一梳理成一份完整地图。
> 包含开源 openGauss 与企业版 GaussDB 的差异对照。

---

## 一、Dorado 双集群架构下的事件全景图

```
┌─────────────────────────────────────────────────────────────────────────┐
│                  一次 COMMIT 涉及的所有等待事件                           │
│                                                                         │
│  backend 主线程时间轴                          辅助线程时间轴             │
│  ════════════════════════════                ═══════════════════         │
│                                                                         │
│  ┌─ A: XLogInsert ────────────┐                                         │
│  │  WALInsertLock             │                                         │
│  │  WALBufMappingLock         │                                         │
│  └────────────────────────────┘                                         │
│                                                                         │
│  ┌─ B: XLogWaitFlush ─────────┐    ┌─ walwriter 线程 ────────────────┐   │
│  │  walFlushWaitLock          │ ──►│ E: WALWriteLock                  │   │
│  │  PGSemaphoreLock(黑洞段)   │    │ F: WALWrite (开源)               │   │
│  │  ★ backend 此段无 wait_event│    │   = SharedStorageWalWrite (企业)│   │
│  │                            │    │ G: fsync (无 wait_event)         │   │
│  │                            │    │ H: SharedStorageCtlInfoWriter   │   │
│  │                            │    │   (企业版 only)                 │   │
│  │                            │ ◄──│ 通知 backend                     │   │
│  └────────────────────────────┘    └──────────────────────────────────┘   │
│                                                                         │
│  ┌─ C: SyncRepWaitForLSN ─────┐    ┌─ walsender 线程 ────────────────┐   │
│  │  ★ wait wal sync           │ ◄──│  收备机 reply.apply             │   │
│  │  (STATE_WAIT_WALSYNC)      │    │  → 更新 WalSndCtl->lsn[mode]    │   │
│  │  syncrep.cpp:315→505       │    │  → 唤醒此 backend               │   │
│  └────────────────────────────┘    └──────────────────────────────────┘   │
│                                                                         │
│  ┌─ D: TransactionIdCommitTree┐                                         │
│  │  CLogControlLock           │                                         │
│  │  CSNBufMappingLock         │                                         │
│  └────────────────────────────┘                                         │
│                                                                         │
│                                                                         │
│  另外的旁路（不在 commit 主链路上，但在 Dorado 场景大量出现）             │
│  ════════════════════════════════════════════════════════════════════   │
│                                                                         │
│  事务 B 的 INSERT/SELECT 因事务 A 慢 commit 被殃及:                      │
│                                                                         │
│  ┌─ XactLockTableWait ─────────────────────┐                            │
│  │  transactionid 锁等待                    │  ← pg_thread_wait_status  │
│  │  procarray.cpp:4593/4605                │     locktag=transactionid │
│  └─────────────────────────────────────────┘                            │
│                                                                         │
│  ┌─ SyncLocalXidWait ──────────────────────┐                            │
│  │  ★ wait transaction sync                │                            │
│  │  (STATE_WAIT_XACTSYNC)                  │                            │
│  │  procarray.cpp:4626                     │                            │
│  └─────────────────────────────────────────┘                            │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 二、八大等待事件完整说明

### 事件 1：`WALInsertLock`（段 A）

```
代码位置:  xlog.cpp:1409 ReserveXLogInsertLocation
事件类型:  LWLOCK_EVENT
等待对象:  WAL buffer 8 个分区锁之一
触发场景:  backend 把 redo record 写入 WAL buffer 时
           争夺其中一把 insert lock
           （NUM_XLOGINSERT_LOCKS = 8）

正常时长:  <10μs
异常时长:  超过 100μs 说明 8 把锁不够用，并发太高

Dorado 场景影响:
  - 间接受影响：commit 慢→事务积压→并发更高→更激烈的争用
  - 但和 Dorado 直接关系较小
```

### 事件 2：`WALBufMappingLock`（段 C, 在段 A 内部）

```
代码位置:  xlog.cpp:2426 AdvanceXLInsertBuffer
事件类型:  LWLOCK_EVENT
等待对象:  WAL buffer page 推进锁
触发场景:  WAL buffer 写满后切换到下一页时

正常时长:  <50μs
异常时长:  说明 WAL buffer (wal_buffers) 不够大

Dorado 场景影响:
  - WAL 量大 + walwriter 写慢 → buffer 频繁满 → 此事件升高
  - 是 Dorado 慢的间接症状之一
```

### 事件 3：`walFlushWaitLock` + `PGSemaphoreLock`（段 D，"backend 黑洞段"）

```
代码位置:  xlog.cpp:3475-3516 XLogWaitFlush
事件类型:  LWLOCK_EVENT (walFlushWaitLock)
           + PGSemaphoreLock (无 wait_event 显式名)
等待对象:  walwriter 完成 flush 通知

代码:
  while (flushTo < recptr) {
      LWLockAcquireOrWait(walFlushWaitLock, LW_EXCLUSIVE)  ← LWLock
      PGSemaphoreLock(&walFlushWaitLock->sem, true)        ← 黑洞段
      LWLockRelease
  }

⚠️ 关键观测盲区:
  - PGSemaphoreLock 没有 wait_event 输出
  - backend 在这一段挂起期间，wait_event 显示为空白
  - 实际等的就是 walwriter 完成 SharedStorageWalWrite 的全部时间
  - 这就是分析文档里说的 "backend 统计黑洞段"

如何还原此段时间:
  → 看 walwriter 的 SharedStorageWalWrite 总耗时
  → 即 walwriter 的 IO 累计 ≈ backend 黑洞段累计
```

### 事件 4：`WALWriteLock`（段 E）

```
代码位置:  xlog.cpp:3894 LWLockAcquireOrWait(WALWriteLock, ...)
事件类型:  LWLOCK_EVENT
等待对象:  全局 WAL write lock

触发场景:  walwriter (或 backend 自己) 准备写 WAL 到磁盘前

⚠️ 单线程瓶颈点:
  - 这是 PG/GaussDB 单 walwriter 设计的核心串行化点
  - Oracle 12.2+ 用 Scalable LGWR 把这个锁拆了，PG 系没拆
  - 是农行 1000 IOPS 天花板的根本原因

Dorado 场景影响: 直接相关
  - Dorado 每次 IO 1ms RTT
  - WALWriteLock 持有 1ms 期间，所有想 flush 的 backend 都堵
```

### 事件 5：`WALWrite` / `SharedStorageWalWrite`（段 F）

```
代码位置:  xlog.cpp:2824-2837
事件类型:  IO_EVENT
等待对象:  pwrite() 或 dss_append() 到 DSS / Dorado 完成

★ 开源 openGauss 显示: WALWrite
★ 企业版 GaussDB 显示: SharedStorageWalWrite

【精确覆盖范围】

backend 调用 write 前                                         write 返回
      │                                                             │
      ▼                                                             ▼
┌─────────────────────── F 段 ─────────────────────────────────────┐
│                  SharedStorageWalWrite                            │
│                                                                   │
│    DSS client → DSS server                                        │
│    → Dorado A 控制器                                              │
│    → Dorado A 写本地 cache                                        │
│    → ★ HyperReplication 同步到 Dorado B  ← 双盘存储同步在此段内★ │
│    → Dorado B 写 cache + ACK 回 A                                 │
│    → Dorado A ACK 回 DSS                                          │
│    → DSS 返回 write 成功                                          │
└───────────────────────────────────────────────────────────────────┘

★ F 段结束 = 两个 Dorado 阵列的数据已一致 + DSS 返回成功 ★

这是 Dorado 存储同步的最重要特点：
  存储层双盘一致已在 F 段完成
  跟传统流复制完全不同（流复制是事后异步推数据）

Dorado 场景影响: 100% 直接相关
  - F 段时长 = Dorado HyperMetro 的实际同步延迟
  - F 段慢 → 整个 commit 慢
  - F 段是 Dorado 阵列性能的最直接观测窗口
```

### 事件 6：`SharedStorageCtlInfoWriter`（段 H，企业版 only）

```
代码位置:  ss_cluster_replication.cpp:31 WriteSSDoradoCtlInfoFile
           包装在 ss_cluster_replication.cpp:175 UpdateSSDoradoCtlInfoAndSync
事件类型:  IO_EVENT (企业版独有)
等待对象:  写 SS_DORADO_CTRL_FILE 控制信息

【触发时机】
  在 WAL flush 后立即触发（xlog.cpp:2896 调用）
  仅 SS_DORADO_PRIMARY_NODE 才执行

【作用】
  把 LogwrtResult->Write (insertHead) 写到共享的 ctl 文件
  让备集群知道"主端 WAL 写到哪里了"
  备集群启动时能定位起始 LSN

【代码细节】
  void UpdateSSDoradoCtlInfoAndSync()
  {
      if (!SS_DORADO_PRIMARY_NODE) return;
      
      ShareStorageXLogCtl *ctlInfo = g_instance.xlog_cxt.ssReplicationXLogCtl;
      ctlInfo->insertHead = t_thrd.xlog_cxt.LogwrtResult->Write;
      WriteSSDoradoCtlInfoFile();   // 触发 SharedStorageCtlInfoWriter
  }

【和 SharedStorageWalWrite 的关系】
  ┌──────────────────────────────────────────────┐
  │ walwriter 写一组 WAL records:                 │
  │                                              │
  │   1. SharedStorageWalWrite × N               │
  │      （写 N 个 WAL 页）                       │
  │                                              │
  │   2. SharedStorageCtlInfoWriter × 1          │
  │      （更新 ctl 文件让备端知道）              │
  │                                              │
  │  → 两者通常成对出现                           │
  │  → 量级关系：CtlInfoWriter 远少于 WalWrite   │
  │             (一组 WAL 共享一次 ctl 更新)      │
  └──────────────────────────────────────────────┘

【开源 openGauss 没有此事件】
  开源版的 WriteSSDoradoCtlInfoFile 是裸 write()
  没有 wait_event 包装
  → 如果你看的是开源源码，找不到这个事件名
  → 企业版才有
```

### 事件 7：`wait wal sync`（段 I，本文核心）

```
代码位置:  syncrep.cpp:315 (SyncRepWaitForLSN 主路径)
           syncrep.cpp:1794 (SyncPaxosWaitForLSN, DCF 路径)
           syncrep.cpp:522 (SSRealtimeBuildWaitForTime, RTO 路径)
事件类型:  STATUS (WaitState 枚举)
状态枚举:  STATE_WAIT_WALSYNC

【精确边界】
  起点: pgstat_report_waitstatus(STATE_WAIT_WALSYNC)  syncrep.cpp:315
  终点: pgstat_report_waitstatus(oldStatus)            syncrep.cpp:505

【三种进入分支等的对象不同】

┌──────────────────────────────┬──────────────────────────────────┐
│ 函数                          │ 等的是                           │
├──────────────────────────────┼──────────────────────────────────┤
│ SyncRepWaitForLSN            │ walsender 收到备机 reply.apply   │
│ (正常路径)                    │ → WalSndCtl->lsn[mode] 推进     │
│                              │                                  │
│ SSRealtimeBuildWaitForTime   │ Dorado 备集群 redo 追赶 RTO 窗口 │
│ (SS Dorado RTO 路径)          │ 不是 LSN 对比，是时间约束        │
│                              │                                  │
│ SyncPaxosWaitForLSN          │ DCF Paxos 过半数节点确认         │
│ (DCF 路径)                    │ → syncPaxosState = COMPLETE     │
└──────────────────────────────┴──────────────────────────────────┘

【SS_DORADO_CLUSTER 下的真实语义】

关键证据 walsender.cpp:2987:

  if (SS_DORADO_CLUSTER) {
      AdvanceReplicationSlot(reply.apply);   // ★ apply 而非 flush
  } else {
      AdvanceReplicationSlot(reply.flush);
  }

  → 在 SS_DORADO_CLUSTER 下，wait wal sync 等的是
    备集群完成 redo APPLY（而不只是 flush）
  → 因为 Dorado 同步会让主端 WAL 直接覆盖备端 DSS
    "存储已同步" ≠ "数据库已恢复一致"
    必须等回放追上才能保证故障切换不丢

【和 Dorado 的关系】
  直接关系: ❌ 不调任何 Dorado API
  间接关系: ✅ Dorado 把 WAL 搬到备端 DSS 是回放的前提

【观测盲区】
  syncrep.cpp:282-292 有 10000 次自旋快路径（共 ~10ms）
  此期间 wait_event 还没设成 STATE_WAIT_WALSYNC
  → 10ms 内的快 commit 不会留下 wait wal sync 痕迹
```

### 事件 8：`wait transaction sync`（旁路）

```
代码位置:  procarray.cpp:4626 SyncLocalXidWait
事件类型:  STATUS (WaitState 枚举)
状态枚举:  STATE_WAIT_XACTSYNC

【触发路径】
  XactLockTableWait (procarray.cpp:4593/4605)
    或
  DMS 跨节点 xid 回调 (ss_dms_callback.cpp:192/194)
    或
  Ustore 可见性判断 (knl_uvisibility.cpp:1413)
       │
       ▼
  SyncLocalXidWait (procarray.cpp:4614)
       │
       ▼
  pgstat_report_waitstatus_xid(STATE_WAIT_XACTSYNC)  procarray.cpp:4626

【真实语义】
  线程 B 检查另一个线程 A 的 xid 状态
  发现 A 还在 commit 中（xid 未在 CLOG 落定）
  → 必须等 A 完成（commit 或 rollback）才能继续

【和 commit 链路的关系】
  ⚠️ 不是 commit 主链路
  ⚠️ 是 commit 慢的"次生症状"
  
  事务 A commit 慢（卡 wait wal sync）
       ↓
  事务 A 的 xid 长时间未落定
       ↓
  事务 B/C/D... 的 INSERT/SELECT 撞到 A 的 pending 行
       ↓
  事务 B/C/D... 进入 wait transaction sync (各发生 1 次)
       ↓
  ★ 一个慢 commit 可以级联拖住 N 个其他事务

【可观测模式】
  生产场景常见数量关系:
    wait wal sync × 1
    wait transaction sync × N（N = 撞键的并发事务数）
    
  典型比例: 1 : 2~10
```

---

## 三、所有事件的统一速查表

```
┌─────┬──────────────────────────────┬──────────┬────────────────┬─────────────────────┐
│ 段  │ 事件名                        │ 类型      │ 线程            │ 与 Dorado 关系       │
├─────┼──────────────────────────────┼──────────┼────────────────┼─────────────────────┤
│ A   │ WALInsertLock                │ LWLock   │ backend        │ 间接（高并发）       │
│ A'  │ WALBufMappingLock            │ LWLock   │ backend        │ 间接（buffer 满）    │
│ D   │ walFlushWaitLock             │ LWLock   │ backend        │ 间接                │
│ D'  │ PGSemaphoreLock (黑洞段)     │ -        │ backend        │ 间接（等 walwriter）│
│ E   │ WALWriteLock                 │ LWLock   │ walwriter      │ 直接（单线程瓶颈）   │
│ F   │ WALWrite (开源)              │ IO       │ walwriter      │ 直接（含双盘同步）   │
│ F   │ SharedStorageWalWrite (企业) │ IO       │ walwriter      │ ★ 直接              │
│ G   │ fsync (无事件)                │ -        │ walwriter      │ 直接                │
│ H   │ SharedStorageCtlInfoWriter   │ IO       │ walwriter      │ ★ 直接（企业版 only)│
│     │ (企业版)                      │          │                │                     │
│ I   │ wait wal sync (主路径)       │ STATUS   │ backend        │ 间接（等备集群回放） │
│ I'  │ wait wal sync (DCF Paxos)    │ STATUS   │ backend        │ 不相关              │
│ I'' │ wait wal sync (SS RTO 路径)  │ STATUS   │ backend        │ 间接（RTO 时间约束）│
│ -   │ wait transaction sync        │ STATUS   │ backend (其他) │ 间接（次生症状）    │
└─────┴──────────────────────────────┴──────────┴────────────────┴─────────────────────┘
```

---

## 四、开源 openGauss vs 企业版 GaussDB 事件对照

```
┌──────────────────────────────┬──────────────────┬──────────────────┐
│ 事件名                        │ 开源 openGauss   │ 企业版 GaussDB   │
├──────────────────────────────┼──────────────────┼──────────────────┤
│ WALInsertLock                │ ✅ 有             │ ✅ 有             │
│ WALBufMappingLock            │ ✅ 有             │ ✅ 有             │
│ walFlushWaitLock             │ ✅ 有             │ ✅ 有             │
│ WALWriteLock                 │ ✅ 有             │ ✅ 有             │
│ WALWrite                     │ ✅ 有             │ ❌ 改名           │
│ SharedStorageWalWrite        │ ❌ 没有           │ ✅ 有 (新增)      │
│ SharedStorageCtlInfoWriter   │ ❌ 没有           │ ✅ 有 (新增)      │
│ wait wal sync                │ ✅ 有             │ ✅ 有             │
│ wait transaction sync        │ ✅ 有             │ ✅ 有             │
└──────────────────────────────┴──────────────────┴──────────────────┘

差异原因:
  - 企业版增加了对共享存储 IO 的精细化监控
  - WALWrite 在共享存储模式下被替换为 SharedStorageWalWrite
  - SharedStorageCtlInfoWriter 是企业版专为 SS_DORADO_CLUSTER 新增

诊断时要注意:
  - 看的是开源版还是企业版（影响事件名映射）
  - 企业版生产环境通常监控 SharedStorageWalWrite
  - 开源版只能看 WALWrite (兼具这两种语义)
```

---

## 五、事件之间的"配对关系"

### 配对 1：F 段 vs I 段（最重要）

```
┌─────────────────────────────────────────────────┐
│ F 段 = SharedStorageWalWrite                    │
│   walwriter 写 DSS → Dorado A → Dorado B 同步   │
│   完成时刻 = 两个 Dorado 阵列已一致              │
│                                                 │
│ I 段 = wait wal sync                            │
│   backend 等备集群 GaussDB 把 WAL 回放到 LSN    │
│   完成时刻 = 备集群 apply LSN 推进 + ACK 回主端 │
└─────────────────────────────────────────────────┘

时间关系: F 段在前，I 段在后（同一次 commit 内串行）
线程关系: F 段在 walwriter，I 段在 backend（不同线程）
观测关系: 
  - 单 commit 看：F 段先完，I 段才开始等
  - 聚合看：N 个 commit 的 F 段和 N 个 backend 的 I 段重叠
  
判别意义:
  - F 高 + I 高 → Dorado 阵列慢
  - F 正常 + I 高 → 备集群回放慢
  - F 高 + I 正常 → 主端 walwriter 串行瓶颈
  - F 低 + I 低 → 不是 commit 链路问题
```

### 配对 2：F 段 vs H 段（企业版双指标）

```
┌─────────────────────────────────────────────────┐
│ F 段 = SharedStorageWalWrite                    │
│   写 WAL 数据本身                               │
│   每个 WAL 页都触发                             │
│                                                 │
│ H 段 = SharedStorageCtlInfoWriter               │
│   写 ctl 文件元数据                             │
│   每组 WAL 共享一次（频率远低于 F 段）          │
└─────────────────────────────────────────────────┘

正常关系: H 数量 << F 数量（量级差异 1:10 ~ 1:100）

异常模式:
  - H 涨 + F 不涨 → ctl 文件 IO 异常（小概率）
  - F 涨 + H 不涨 → WAL 写入异常但元数据写正常
  - 两者都涨 → Dorado 整体慢

如果两者都没出现:
  → 看的是开源版（不发这两个事件）
  → 或者主集群不是 SS_DORADO_PRIMARY_NODE
```

### 配对 3：wait wal sync vs wait transaction sync

```
┌─────────────────────────────────────────────────┐
│ wait wal sync = 主因                            │
│   commit 线程等远端 ACK                         │
│   每个慢 commit 产生 1 次                       │
│                                                 │
│ wait transaction sync = 次生                    │
│   被堵的其他线程等慢 commit 完成                 │
│   一个慢 commit 可级联产生 N 次                  │
└─────────────────────────────────────────────────┘

数量关系: wait transaction sync 通常是 wait wal sync 的 2~10 倍

判别:
  两者比例 1:1   → 没有热点，commit 慢但单独发生
  两者比例 1:5+  → 有热点 unique key，事务级联堵塞
  只有 wait txn sync → 不是 commit 链路慢，可能是 DMS 跨节点协调
```

---

## 六、在生产里看到不同组合的根因判别

### 完整判别矩阵

```
┌──────────────────┬──────────────────┬───────────────┬───────────────────────────┐
│ wait wal sync    │ SharedStorage    │ SharedStorage │ 根因判定                  │
│                  │ WalWrite         │ CtlInfoWriter │                           │
├──────────────────┼──────────────────┼───────────────┼───────────────────────────┤
│ 高               │ 高                │ 高             │ Dorado 阵列慢/链路慢      │
│ 高               │ 正常              │ 正常           │ 备集群回放慢 ⭐ 农行场景  │
│ 高               │ 高                │ 正常           │ Dorado WAL 慢但元数据 OK  │
│ 正常             │ 高                │ 高             │ 主端写慢但备端追上了      │
│ 正常             │ 正常              │ 正常           │ commit 链路非瓶颈         │
│ 高               │ -                 │ -             │ 看的是开源版（找不到 SS*）│
└──────────────────┴──────────────────┴───────────────┴───────────────────────────┘

农行生产观察:
  "wait wal sync 大量出现，但 SharedStorageWalWrite / 
   SharedStorageCtlInfoWriter 没出现"
  → 对应第二行：备集群回放/读 DSS 慢
  → 排查方向：备集群 recovery_parse_workers / Dorado B 读性能 / 备端网络
```

### 加上 wait transaction sync 的扩展矩阵

```
┌──────────────────┬─────────────────┬─────────────────┐
│ wait wal sync    │ wait txn sync    │ 业务影响        │
├──────────────────┼─────────────────┼─────────────────┤
│ 高 (1x)          │ 极高 (10x+)      │ 热点 commit +   │
│                  │                  │ 大量 insert 堵  │
│                  │                  │ ⭐ 农行小事务   │
├──────────────────┼─────────────────┼─────────────────┤
│ 高 (1x)          │ 中 (2-5x)        │ commit 慢但分散 │
│                  │                  │ 撞键概率适中    │
├──────────────────┼─────────────────┼─────────────────┤
│ 高 (1x)          │ 低 (1x)          │ commit 慢但无   │
│                  │                  │ 热点冲突        │
├──────────────────┼─────────────────┼─────────────────┤
│ 低               │ 高               │ DMS 跨节点协调  │
│                  │                  │ 慢，非 commit   │
└──────────────────┴─────────────────┴─────────────────┘
```

---

## 七、Dorado 双集群完整 commit 时间轴（带所有事件）

```
 主集群 backend                                                    备集群
 ════════════════════════════════════════════════════════════════════════════
 
 t=0    XLogInsert (段 A: WALInsertLock)
        │
 t=10μs ↓
        XLogWaitFlush 入口
        ├─ LWLockAcquireOrWait(walFlushWaitLock)  ← wait_event = walFlushWaitLock
        ├─ PGSemaphoreLock                         ← ★ wait_event = 空 (黑洞段)
        │  │
        │  └──► 唤醒 walwriter 线程:
        │      │
        │      ▼
        │      ┌─ walwriter 拿 WALWriteLock (段 E)
        │      │
        │      ▼
        │      段 F: pwrite to DSS
        │      ┌──────────────────────────────────┐
        │      │ wait_event = SharedStorageWalWrite│
        │      │ (开源: WALWrite)                 │
        │      │                                  │
        │      │ DSS → Dorado A → HyperMetro →    │
        │      │ Dorado B → ACK → Dorado A → ACK  │
        │      │                                  │
        │      │ 持续 ~1ms (Dorado RTT 物理下限)  │
        │      └──────────────────────────────────┘
        │      │
        │      ▼
        │      段 G: fsync (无 wait_event)
        │      │
        │      ▼
        │      段 H: SharedStorageCtlInfoWriter (企业版 only)
        │      ┌──────────────────────────────────┐
        │      │ UpdateSSDoradoCtlInfoAndSync()   │
        │      │ 写 SS_DORADO_CTRL_FILE          │
        │      │ 让备集群知道 WAL 写到哪          │
        │      └──────────────────────────────────┘
        │      │
        │      ▼
        │      walwriter 通知 backend (PGSemaphoreUnlock)
        │
 t=2ms  ▼
        backend 从 PGSemaphoreLock 醒来
        XLogWaitFlush 返回
        │
        ▼
        SyncRepWaitForLSN 入口 (段 C/I)
        ├─ 段 1-3: 快速检查 + 自旋 (~10ms 内可能直接返回)
        │
        ├─ 段 4: 设 wait_event = STATE_WAIT_WALSYNC
        │  ┌──────────────────────────────────────┐
        │  │ ★ wait wal sync ★                   │
        │  │                                      │
        │  │ 等待 WalSndCtl->lsn[mode] 推进       │
        │  │                                      │
        │  │ 此期间:                               │
        │  │   备集群 shared_storage_walreceiver   │ ─── (这边在动)
        │  │   从 Dorado B 读新 WAL                │   备集群线程
        │  │   ↓                                   │   shared_storage_
        │  │   startupProcess 回放                 │   walreceiver
        │  │   ↓                                   │   读 DSS
        │  │   apply LSN 推进                      │   ↓
        │  │   ↓                                   │   startup 回放
        │  │   walreceiver 通过 TCP 发             │   ↓
        │  │   reply.apply                         │   walreceiver 发
        │  │   ↓                                   │   reply.apply
        │  │   主集群 walsender 收到               │   (TCP)
        │  │   更新 WalSndCtl->lsn[mode]           │
        │  │   ↓                                   │
        │  │   唤醒此 backend                      │
        │  └──────────────────────────────────────┘
        │  │
        ├─ 段 5: 清除 wait_event
        │
 t=5ms  ▼
        TransactionIdCommitTree (段 D)
        ├─ CLogControlLock
        └─ CSNBufMappingLock
        │
 t=5.5ms▼
        commit 完成，返回客户端

 ─────────────────────────────────────────────────────────────────────────
 
 同时在另外的线程里发生:
 
 事务 B（撞 unique key）:
   t=2ms  INSERT 同 key
   t=2ms  XactLockTableWait(A.xid)  ← locktag=transactionid
   t=2ms  SyncLocalXidWait(A.xid)   ← wait_event = wait transaction sync
   t=5.5ms 事务 A 完成 → 唤醒 B
   t=5.5ms B 决定：要么报唯一键冲突，要么继续
```

---

## 八、关键源码锚点完整索引

### 主链路代码

| 段 | 功能 | 文件:行号 |
|---|---|---|
| - | RecordTransactionCommit 主干 | `xact.cpp:1639` 起 |
| A | XLogInsert 调用 | `xact.cpp:1706/1746` |
| A | ReserveXLogInsertLocation | `xlog.cpp:1409` |
| C | AdvanceXLInsertBuffer | `xlog.cpp:2426` |
| B | XLogWaitFlush 调用 | `xact.cpp:1791` |
| B/D | XLogWaitFlush 实现 | `xlog.cpp:3475-3516` |
| D | walFlushWaitLock 持锁 | `xlog.cpp:3508` |
| D | PGSemaphoreLock | `xlog.cpp:3509` |
| E | WALWriteLock | `xlog.cpp:3894` |
| F | pwrite/dss_append | `xlog.cpp:2824-2837` |
| G | issue_xlog_fsync | `xlog.cpp:2883` |
| H | UpdateSSDoradoCtlInfoAndSync | `xlog.cpp:2896` 调 / `ss_cluster_replication.cpp:175` 实现 |
| H | WriteSSDoradoCtlInfoFile | `ss_cluster_replication.cpp:31` |
| C/I | SyncRepWaitForLSN 调用 | `xact.cpp:1804` |
| I | SyncRepWaitForLSN 入口 | `syncrep.cpp:230` |
| I | 10000 次自旋快路径 | `syncrep.cpp:282-292` |
| I | wait wal sync 设置 | `syncrep.cpp:315` |
| I | wait wal sync 清除 | `syncrep.cpp:505` |
| I | SSRealtimeBuildWaitForTime | `syncrep.cpp:522` |
| I | SyncPaxosWaitForLSN | `syncrep.cpp:1681-1794` |
| D | TransactionIdCommitTree | `xact.cpp:1811` |

### 旁路代码

| 功能 | 文件:行号 |
|---|---|
| XactLockTableWait | `procarray.cpp:4593/4605` |
| DMS xid 跨节点回调 | `ss_dms_callback.cpp:192/194` |
| Ustore 可见性判断 | `knl_uvisibility.cpp:1413` |
| SyncLocalXidWait | `procarray.cpp:4614` |
| wait transaction sync 设置 | `procarray.cpp:4626` |

### walsender / 复制代码

| 功能 | 文件:行号 |
|---|---|
| WalSndLoop | `walsender.cpp` |
| ProcessStandbyReplyMessage | `walsender.cpp` 内 |
| SyncRepReleaseWaiters 调用 | `walsender.cpp:2978` |
| SS_DORADO_CLUSTER apply 语义 | `walsender.cpp:2987` |
| shared_storage_walreceiver | `shared_storage_walreceiver.cpp` |

### 宏定义

| 宏 | 文件:行号 |
|---|---|
| SS_DORADO_CLUSTER | `ss_disaster_cluster.h:76` |
| SS_DORADO_PRIMARY_CLUSTER | `ss_disaster_cluster.h:80-81` |
| SS_DORADO_STANDBY_CLUSTER | `ss_disaster_cluster.h:84-85` |
| SS_DORADO_PRIMARY_NODE | `ss_disaster_cluster.h:88-89` |

### 事件名映射

| 功能 | 文件 |
|---|---|
| Wait event 显示名映射 | `src/gausskernel/cbb/instruments/ash/wait_event_info.cpp` |
| 状态名映射 | `src/common/backend/utils/adt/pgstatfuncs.cpp:355-362` |
| WaitEventIO 枚举 | `src/include/pgstat.h:1320-1362` |
| WaitEventDMS 枚举 | `src/include/pgstat.h:1364-1412` |
| LWLock tranche 名 | `lwlock.cpp:196` |
| PgBackendStatus 三字段 | `src/include/pgstat.h:1667/1672/1675` |

---

## 九、核心结论

1. **Dorado 双集群同步 commit 涉及 8 类等待事件**：4 个 LWLock + 2 个 IO + 2 个 Status + 1 个旁路
2. **企业版多 2 个事件**：`SharedStorageWalWrite`（替代开源 `WALWrite`）和 `SharedStorageCtlInfoWriter`（开源没有）
3. **F 段（SharedStorageWalWrite）覆盖完整的 Dorado 双盘同步**：DSS → Dorado A → HyperMetro → Dorado B → ACK 全程
4. **I 段（wait wal sync）等的不是 Dorado 同步**：等的是备集群 GaussDB 的 redo apply（注：SS_DORADO 用 reply.apply 不是 reply.flush，证据 walsender.cpp:2987）
5. **F 段和 I 段串行不重叠**：F 段在 walwriter 内，I 段在 backend 内，同一次 commit 是先 F 后 I
6. **backend 黑洞段**：D 段的 PGSemaphoreLock 没有显式 wait_event，时长要从 walwriter 的 SharedStorageWalWrite 反推
7. **wait transaction sync 是次生症状**：commit 慢级联拖累其他事务，不是独立瓶颈
8. **判别 Dorado 是否慢的金标准**：看 SharedStorageWalWrite 是否高，单看 wait wal sync 不够
9. **农行场景的特征模式**：wait wal sync 高 + SharedStorageWalWrite 正常 = 备集群回放慢，不是 Dorado 阵列问题
