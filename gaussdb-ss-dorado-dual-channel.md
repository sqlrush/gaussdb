# GaussDB SS_DORADO_CLUSTER 双通道架构分析

> 详细分析 Dorado 双集群共享盘 sync 同步模式下，主备之间数据面（Dorado
> 光纤存储同步）和控制面（TCP 复制协议）的分工、以及 walsender/walreceiver
> 在此模式下的角色变化。

---

## 一、核心认知：两条独立通道并存

```
┌────────────────────────────────────────────────────────────────┐
│  通道 A: Dorado 存储通道（光纤/SAN）  "数据面"                   │
│  通道 B: TCP 复制连接（IP/以太网）   "控制面"                    │
│                                                                │
│  两条通道物理分离、协议正交、职责独立                              │
│  都必须存在，缺一不可                                             │
└────────────────────────────────────────────────────────────────┘
```

---

## 二、双通道各自承担什么

### 通道 A：Dorado 存储通道

```
 流什么:
   ✅ WAL record 完整二进制内容
   ✅ WAL segment files（pg_wal/ 下的文件）
   ✅ Control file (pg_control)
   ✅ 数据文件（表/索引/CLOG/CSN Log）
   ✅ 所有需要持久化同步的数据

 谁在用:
   主端: walwriter 的 pwrite → DSS → Dorado A
   备端: shared_storage_walreceiver read → DSS ← Dorado B

 协议:
   SCSI / NVMe-oF + Dorado HyperMetro 私有协议

 特征:
   GaussDB 对这层完全透明
   对 DB 而言像在读/写本地磁盘
```

### 通道 B：TCP 复制连接

```
 流什么:
   ✅ 握手消息（STARTUP / 认证 / IDENTIFY_SYSTEM）
   ✅ START_REPLICATION + 起始 LSN
   ✅ timeline 信息
   ✅ keepalive 心跳
   ✅ ★ reply.write / reply.flush / reply.apply ★
      (备端 LSN 位点反馈)
   ✅ hot standby feedback（xmin horizon）
   ✅ switchover / promote 控制指令

 不流什么:
   ❌ WAL record 实际二进制内容
      (SS_DORADO_CLUSTER 下几乎不发；追赶场景可能例外)

 谁在用:
   主端: walsender 进程
   备端: walreceiver 进程

 协议:
   TCP + libpq (pg-wire 流式复制协议)

 特征:
   流量很小（和普通流复制比），但不可断
```

---

## 三、普通流复制 vs SS_DORADO_CLUSTER 对比

```
┌──────────────────────┬────────────────────────────────────────┐
│ 模式                  │ 两通道承担                              │
├──────────────────────┼────────────────────────────────────────┤
│ 普通流复制             │ 通道 A: 不存在（无共享存储）             │
│ (非共享盘)            │ 通道 B: 运 WAL 全量 + 控制信号           │
│                      │         → TCP 带宽大                    │
├──────────────────────┼────────────────────────────────────────┤
│ SS_DORADO_CLUSTER    │ 通道 A: 运 WAL 全量数据                  │
│                      │ 通道 B: 只运控制信号和位置元数据           │
│                      │         → TCP 带宽极小                    │
└──────────────────────┴────────────────────────────────────────┘

 流量对比（典型 OLTP，WAL 生成速率 100 MB/s）:
 
 普通流复制:
   TCP 带宽 ≈ 100 MB/s + 少量控制
   典型占用 10Gbps 链路 ~8%
 
 SS_DORADO_CLUSTER:
   TCP 带宽 ≈ <1 MB/s
   典型占用 10Gbps 链路 ~0.01%
```

---

## 四、reply.apply 消息格式

备端发给主端的一条典型 reply 消息（约 40 字节）：

```
┌──────────────────────────────────────────────────────────────┐
│ Standby status update message (type 'r')                     │
├──────────────────────────────────────────────────────────────┤
│  writePtr      XLogRecPtr  8 bytes  ← 已写到备端存储的 LSN    │
│  flushPtr      XLogRecPtr  8 bytes  ← 已 fsync 的 LSN        │
│  applyPtr      XLogRecPtr  8 bytes  ★ 已回放的 LSN ★         │
│  sendTime      TimestampTz 8 bytes  ← 发送时间戳              │
│  replyRequested bool       1 byte   ← 是否请求立即回复        │
│  ...                                                         │
└──────────────────────────────────────────────────────────────┘

 发送频率:
   - 默认每 wal_receiver_status_interval 秒（10s）一次
   - 关键事件（回放位点推进到目标 LSN）立即发

 主端处理:
   40 字节就能表达备端全部状态
   → 这就是用户所说的"元数据/位置信息"
```

---

## 五、"reply.apply 不是突然"——长连接早已存在

### 常见误解

```
错的画面:
  主集群 commit → 写 Dorado → Dorado 同步到 B
  突然 TCP 数据包从天而降
  主集群："这谁？发什么？"
  
错在哪:
  这种理解忽略了连接早已建立的事实
```

### 真实的连接生命周期

#### 阶段 ①：建连（备端主动发起，在任何 commit 之前）

```
 备集群侧（walreceiver）:
   T=0  启动 walreceiver 线程
   T=1  读配置: primary_conninfo = 'host=... port=5432 ...'
   T=2  libpq connect() 到主集群
   T=3  STARTUP 消息（带 application_name）
   T=4  认证（pg_hba.conf + replication 权限）
   T=5  IDENTIFY_SYSTEM 查主库 ID
   T=6  START_REPLICATION <LSN>
   T=7  接收主库确认
   T=8  ★ 进入 streaming 状态（长期保持）★

 主集群侧（walsender）:
   postmaster 接受连接 → fork 一个 walsender 进程
   一对一匹配此备机
   握手完成后进入 WalSndLoop
```

#### 阶段 ②：稳态（持续几小时/天/周，和 commit 无关）

```
 主 walsender ───────────────► 备 walreceiver
   发 keepalive（心跳）
   (SS_DORADO_CLUSTER 下不发 WAL 数据)

 备 walreceiver ─────────────► 主 walsender
   reply.write / reply.flush / reply.apply
   keepalive 响应

 默认间隔: wal_receiver_status_interval = 10s
```

#### 阶段 ③：commit 到达时（完全预期）

```
 T10000  主 backend 执行 commit
         ├─ 写 commit record
         ├─ XLogWaitFlush 完成
         ├─ backend 进入 SyncRepWaitForLSN 等待
         └─ 加入 SyncRepQueue（按 LSN 排序）

 T10001  备集群:
         ├─ shared_storage_walreceiver 从 Dorado B 读到新 WAL
         ├─ startup 回放
         ├─ apply LSN 推进
         └─ walreceiver 通过已有 TCP 连接发 reply.apply

 T10002  主 walsender 的 epoll 被唤醒
         ├─ 读已存在 TCP 连接的新数据
         ├─ 解析出 reply.apply
         ├─ SyncRepReleaseWaiters():
         │   ├─ 更新 WalSndCtl->lsn[mode]
         │   └─ 唤醒 SyncRepQueue 中 LSN <= apply 的 backend
         └─ backend 从 wait wal sync 返回
```

---

## 六、walsender 主循环（一直在等）

```c
// walsender.cpp::WalSndLoop() 骨架

for (;;) {
    // 非阻塞读备端消息
    ProcessRepliesIfAny();   // ← reply.apply 在这里被处理

    // 处理定时器任务
    ...

    // 检查是否需要发送新 WAL（SS_DORADO 下多数时候不用）
    ...

    // 阻塞在 WaitLatchOrSocket
    WaitLatchOrSocket(
        &MyProc->procLatch,
        WL_LATCH_SET | WL_SOCKET_READABLE | ...,
        sock_fd,
        timeout_ms
    );
}

// ProcessRepliesIfAny() 内部
read(sock_fd, buffer, ...);
switch (buffer[0]) {
    case 'r':  // Standby status update
        ProcessStandbyReplyMessage();
        break;
    case 'h':  // Hot standby feedback
        ProcessStandbyHSFeedbackMessage();
        break;
}

// ProcessStandbyReplyMessage() 内部（walsender.cpp:2978+）
reply.write, reply.flush, reply.apply = parse(...);
if (!AM_WAL_STANDBY_SENDER) {
    SyncRepReleaseWaiters();    // ← 唤醒 wait wal sync
}
if (SS_DORADO_CLUSTER) {
    AdvanceReplicationSlot(reply.apply);   // walsender.cpp:2987
}
```

---

## 七、代码证据：shared_storage_walreceiver 特殊实现

```
src/gausskernel/storage/replication/
├── walreceiver.cpp                    ← 普通流复制的 walreceiver
├── archive_walreceiver.cpp            ← 归档日志接收
├── shared_storage_walreceiver.cpp ★   ← SS_DORADO 专用
└── libpqwalreceiver.cpp               ← libpq 底层连接
```

### 核心区别

```cpp
// 普通 walreceiver (walreceiver.cpp):
//   从 TCP socket 读 WAL bytes
while (walrcv_receive(..., &buf)) {
    XLogWalRcvReceive(buf, len);   // ← TCP 来的数据写到 pg_wal/
    XLogWalRcvFlush(...);
}

// shared_storage_walreceiver:
//   从 DSS 读 WAL bytes
while (...) {
    // 直接从共享存储读，不经 TCP
    start_lsn = walRcvCtlBlock->receivePtr;
    XLogPageRead(start_lsn, ...);   // ← 读共享存储页面
    // 推进 receivePtr/writePtr
    // 但 status reply 仍通过 TCP 发给主端
}
```

---

## 八、为什么这样设计

```
 SS_DORADO_CLUSTER 设计哲学:

 1. 存储同步已经保证"数据一致"
    → 没必要再通过 TCP 把 10MB/s WAL 搬一遍
    → 节省网络带宽

 2. 但数据库语义必须依靠 DB 协议
    → "回放进度"是 DB 内部概念（LSN/SCN/XID）
    → Dorado 不懂这些语义
    → 必须用 pg-wire 协议表达

 3. 解耦两个维度
    → 带宽敏感的数据面: 专用 SAN + 硬件级同步
    → 延迟敏感的控制面: 标准 TCP + 软件协议
    → 独立扩展、独立故障、独立监控

 4. 最大化复用已有协议
    → 只替换 walreceiver 的 WAL 来源（TCP → DSS）
    → 其他控制面协议代码一行不用改
    → 降低实现复杂度
```

---

## 九、连接不存在会怎样

```
 场景 A: 备集群没启动（walreceiver 没连上来）
   synchronous_standby_names 配置的备机"未就绪"
   commit 行为:
     ├─ synchronous_commit=local：直接返回，不等同步
     ├─ synchronous_commit=on + sync_master_standalone：降级为 standalone
     ├─ synchronous_commit=on + 严格模式：hang 住等备机回来

 场景 B: 连接中途断开（网络抖动）
   备 walreceiver 自动重连（libpq 断线重连）
   主 walsender 进程退出，新连接来时重新 fork
   期间所有同步 commit 全部 hang
   → 所以 IP 复制网要高可用

 场景 C: 连接存在但备端已挂
   TCP keepalive 检测不到立即失败
   主机 commit hang 到 TCP 超时（默认几分钟）
   → 必须配 wal_sender_timeout 或 wal_receiver_timeout
```

---

## 十、配置映射：主集群知道等谁

```
 主集群 postgresql.conf:
   synchronous_standby_names = 'FIRST 1 (standby_cluster_node1)'
                                         ↑
                                         关键：这个名字

 备集群 recovery.conf / postgresql.conf:
   primary_conninfo = 'host=... application_name=standby_cluster_node1'
                                                  ↑
                                                  必须匹配

 walsender 启动时:
   ├─ 读客户端 application_name
   ├─ 和 synchronous_standby_names 匹配
   ├─ 匹配成功 → 标记此 walsender 为 sync
   └─ 这个 walsender 的 reply 才会触发 SyncRepReleaseWaiters

 匹配失败的 walsender 发来的 reply 不会唤醒 sync commit
 → 防止"任何连接都能告诉主机 commit 完成了"
```

---

## 十一、用比喻理解

```
 类比：外卖配送

 错的画面（直觉）:
   你点外卖 → 外卖员在路上 → 朋友突然打来电话说"你的外卖到了"
   你："你怎么知道我点了？你怎么有我电话？"

 实际画面:
   你和朋友提前加了微信好友
   点外卖前微信早就加好
   外卖送到朋友那后，朋友在已有微信发"xxx 外卖到了"
   你收到消息毫不意外

 对应关系:
   微信好友关系      = walsender↔walreceiver 长连接
   加好友过程        = 备集群启动时的握手
   外卖配送          = Dorado 同步
   朋友的微信消息    = reply.apply

 不突然，前提是连接早就建好了。
```

---

## 十二、一句话结论

**SS_DORADO_CLUSTER 下，walsender 和 walreceiver 之间的 TCP 通道不传递 WAL 的实际内容（那个走 Dorado 光纤同步），只传递控制面信号和位置元数据**——握手、认证、timeline、心跳，以及最关键的 **reply.write/flush/apply 三个 LSN 位点**。整个架构是"**数据面走 SAN，控制面走 IP**"的双通道设计，物理隔离、协议正交、职责分明。`wait wal sync` 等的恰恰是这 40 字节左右的控制面消息，**不是 WAL 数据本身的传输**。这也解释了为什么 SS_DORADO_CLUSTER 仍然必须保持主备 IP 网络可达——光有 Dorado 光纤是不够的。
