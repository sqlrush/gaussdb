# GaussDB 技术分析

基于 openGauss 源码（`~/opengauss/source`）整理的 GaussDB 内核技术分析笔记，
重点覆盖等待事件源码解析、WAL 并行写架构对比、Dorado 双集群 sync 同步机制，
以及可观测性方法论。

## 文档索引

### 1. [GaussDB wait wal sync 源码级分析](./gaussdb-wait-wal-sync-analysis.md)

Dorado 双集群共享盘 sync 架构下 `wait wal sync` / `wait transaction sync` /
`SharedStorageWalWrite` 三个等待事件的源码级分析。

覆盖：
- commit 时间轴 A~I 各段代码映射（含 file:line）
- F 段（SharedStorageWalWrite）和 I 段（wait wal sync）精确覆盖范围
- wait transaction sync 与 commit 链路的旁路关系
- 开源 openGauss 与企业版 GaussDB 的 SharedStorage* 事件差异
- Dorado 双集群架构下 I 段等待对象的三种可能
- 排查 SQL 与决策树

### 2. [GaussDB 等待事件扩展分析](./gaussdb-wait-events-extended-analysis.md)

在 wait-wal-sync 基础分析上的扩展发现（1480 行完整版）。

覆盖：
- `pg_thread_wait_status` 里"commit 线程堵 insert 线程"根因（慢 commit + MVCC 唯一键可见性）
- `wait transaction sync` 是 `wait wal sync` 的级联次生症状（因果链 + 代码锚点）
- `wait wal sync` 代码级精确边界（`syncrep.cpp:315→505`）+ 5 个子段完整代码
- 10ms 自旋快路径是观测盲区
- SS_DORADO_CLUSTER 下 `wait wal sync` 真实语义（等备集群 redo apply，不是等 Dorado 同步）
- 关键代码证据 `walsender.cpp:2987`（apply vs flush 语义差异）
- 三种 `wait wal sync` 高的情况及判别矩阵
- 双集群 4 条物理链路（3 条 SAN + 1 条 IP）
- 完整数据流 + 跨机房双光纤的细节

### 3. [GaussDB SS_DORADO_CLUSTER 双通道架构](./gaussdb-ss-dorado-dual-channel.md)

Dorado 双集群模式下数据面（Dorado 光纤同步）和控制面（TCP 复制协议）的分工分析。

覆盖：
- 双通道各自承担的信息类型
- 普通流复制 vs SS_DORADO_CLUSTER 的流量差异
- `reply.apply` 消息格式（40 字节控制面元数据）
- 长连接建立时机与生命周期（不是"突然"建立）
- `walsender` 主循环 + `ProcessStandbyReplyMessage` 源码路径
- `shared_storage_walreceiver` 特殊实现与普通 `walreceiver` 对比
- 配置映射（`synchronous_standby_names` × `application_name`）

### 4. [WAL Writer 并行写架构跨数据库对比](./wal-writer-architecture-comparison.md)

Oracle / MySQL / PostgreSQL / GaussDB 在 WAL 写入线程模型、fsync 处理、
SCN/LSN 顺序保证等方面的设计差异。

覆盖：
- Oracle 12.2+ Scalable LGWR（LGWR + N worker 并行写）
- Oracle SCN 顺序保证机制（strand + coordinator + durable_watermark + recovery 归并）
- Oracle 为何能并行——根本不用 fsync（ASM 裸设备 / O_DIRECT + O_DSYNC）
- MySQL 8.0 Scalable Redo Log（lock-free buffer + 4 专职线程流水线）
- PostgreSQL / GaussDB 现状（单 walwriter + 8 WAL insert lock）
- GaussDB 学 Oracle 的分层改造路径（O_DSYNC → DSS 裸卷 → 多 walwriter → strand 化 buffer）

### 5. [GaussDB Dorado 双集群同步等待事件全集](./gaussdb-dorado-wait-events-complete.md) ⭐ 推荐先看

把 Dorado 双集群（SS_DORADO_CLUSTER）共享盘 sync 同步架构下，
一次 commit 涉及的**全部 8 类等待事件**统一梳理成一份完整地图。

覆盖：
- 8 大等待事件逐个详解（A~I 段 + 旁路）：WALInsertLock / WALBufMappingLock / walFlushWaitLock / WALWriteLock / WALWrite (SharedStorageWalWrite) / fsync / SharedStorageCtlInfoWriter / wait wal sync / wait transaction sync
- 开源 openGauss vs 企业版 GaussDB 事件对照表
- 事件之间的"配对关系"（F vs I / F vs H / wait wal sync vs wait transaction sync）
- 完整判别矩阵（3 事件组合 + 2 事件组合）
- Dorado 双集群完整 commit 时间轴（带所有事件标注）
- 关键源码锚点完整索引（主链路 + 旁路 + walsender + 宏定义 + 事件名映射）
- 农行场景的特征模式确诊

### 6. [GaussDB 可观测性与诊断方法论](./gaussdb-observability-methodology.md)

GaussDB 等待事件采样、慢 SQL 诊断、短暂性能 spike 捕获、日志查看的方法论
与工具，含与 Oracle 同类机制的横向对比。

覆盖：
- LAS（local_active_session）采样误差分析（4 个场景图示）
- Oracle ASH vs GaussDB LAS 全方位对比（7 个维度 + 差距可视化）
- GaussDB 类似 v$sqlstat 的视图：`dbe_perf.statement` / `statement_history`
- 短暂性能 spike 捕获（0.3→0.5ms 持续 2 秒的 5 种方案 + 生产架构推荐）
- GaussDB 监听日志查看（`$GAUSSLOG/pg_log/`）
- GaussDB 时间戳处理函数（`date_trunc` / `TRUNC` 等）

## 源码基础

分析基于本地 openGauss 源码树：`~/opengauss/source`

关键源码路径：
- `src/gausskernel/storage/access/transam/xlog.cpp`
- `src/gausskernel/storage/access/transam/xact.cpp`
- `src/gausskernel/storage/replication/syncrep.cpp`
- `src/gausskernel/storage/replication/walsender.cpp`
- `src/gausskernel/storage/replication/walreceiver.cpp`
- `src/gausskernel/storage/replication/shared_storage_walreceiver.cpp`
- `src/gausskernel/storage/replication/ss_cluster_replication.cpp`
- `src/gausskernel/storage/ipc/procarray.cpp`
- `src/gausskernel/storage/access/transam/clog.cpp`
- `src/gausskernel/storage/access/transam/csnlog.cpp`
- `src/include/replication/ss_disaster_cluster.h`

## 背景场景

文档分析起点是**中国农业银行同城双集群 Dorado sync 同步架构**下的性能瓶颈：

- 共享盘每次 IO 延迟 ~1ms
- 主库 WAL 写线程单线程，无法并发
- 单线程 IOPS 上限约 1000
- 小事务支付场景下 TPS 受限
- 伴随大量 `wait wal sync` / `wait transaction sync` 等待事件

## 阅读建议

1. 新手先看 [SS_DORADO 双通道架构](./gaussdb-ss-dorado-dual-channel.md) 建立整体架构认知
2. 再看 [wait wal sync 源码分析](./gaussdb-wait-wal-sync-analysis.md) 理解 commit 时间轴
3. 然后看 [等待事件扩展分析](./gaussdb-wait-events-extended-analysis.md) 掌握诊断模式（重点）
4. 接着看 [可观测性方法论](./gaussdb-observability-methodology.md) 学采样/诊断技巧
5. 最后看 [WAL Writer 跨数据库对比](./wal-writer-architecture-comparison.md) 了解优化空间
