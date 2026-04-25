# GaussDB 可观测性与诊断方法论

> 整理 GaussDB 在等待事件采样、慢 SQL 诊断、短暂性能 spike 捕获、
> 日志查看等方面的方法论与工具，含与 Oracle 同类机制的横向对比。

---

## 一、LAS（local_active_session）采样误差分析

### 背景：LAS 只记次数不记时长

`local_active_session` 在 GaussDB 里只在固定时刻（通常 1 秒）拍快照，
拍到什么就记什么，**没有 time_waited 字段**。
拿"出现次数"估算"等待时长"在某些场景下严重失真。

### 前置约定

```
─── 运行中（on CPU / 非等待）
███ 在等待事件 X
 ↓  LAS 采样点（每 1 秒）
 √  采样命中：本次采样记录了 X
```

### 场景 A：真·持续等待 10 秒

```
 时间   0    1    2    3    4    5    6    7    8    9    10 s
        ↓    ↓    ↓    ↓    ↓    ↓    ↓    ↓    ↓    ↓    ↓
 会话  █████████████████████████████████████████████████████  ← 一直等 X
 命中  √    √    √    √    √    √    √    √    √    √   √

 LAS 统计：event X 命中 10 次
 实际等待：~10 秒
 比值：    1 次 ≈ 1 秒 ✅ 统计与真实吻合
```

### 场景 B：间歇等待，每次很短，恰好撞上采样点

```
 时间   0    1    2    3    4    5    6    7    8    9    10 s
        ↓    ↓    ↓    ↓    ↓    ↓    ↓    ↓    ↓    ↓    ↓
 会话  █──────█──────█──────█──────█──────█──────█──────█──
       ↑      ↑      ↑      ↑      ↑      ↑      ↑      ↑
       X 只持续 10ms，其余时间在跑，但恰好发生在采样瞬间
 命中  √    √    √    √    √    √    √    √    √    √   √

 LAS 统计：event X 命中 10 次（和场景 A 一样！）
 实际等待：10 × 10ms = 100 ms
 比值：    1 次 ≈ 10 ms  ⚠️ 比场景 A 真实值少 100 倍
```

### 场景 C：高频短等待，大部分被遗漏

```
 时间   0    1    2    3    4    5    6    7    8    9    10 s
        ↓    ↓    ↓    ↓    ↓    ↓    ↓    ↓    ↓    ↓    ↓
 会话  █─█─█─█─█─█─█─█─█─█─█─█─█─█─█─█─█─█─█─█─█─█─█─█─█─
       ↑每 400ms 等一次 X，每次 50ms；累计等了 1.25 秒
 命中       √         √         √                  √
       （大部分采样点落在运行段，没记录到 X）

 LAS 统计：event X 命中 4 次
 实际等待：1,250 ms
 比值：    1 次 ≈ 312 ms  ⚠️ 被严重低估
```

### 场景 D：单次长等待 + 单次短等待，统计完全模糊

```
 时间   0    1    2    3    4    5    6    7    8    9    10 s
        ↓    ↓    ↓    ↓    ↓    ↓    ↓    ↓    ↓    ↓    ↓
 会话  ────────█████████████████────────────█─────────────
                真·等 X 5 秒              只等 20ms
 命中                 √    √    √    √              √

 LAS 统计：event X 命中 5 次
 实际等待：5,020 ms
 两个业务含义完全不同的事件，被合并成一个计数
```

### 四场景对比总结

| 场景 | LAS 命中次数 | 实际等待时间 | 真实"每次命中"含义 |
|---|---|---|---|
| A 持续等 | 10 | 10,000 ms | 1000 ms/次 ✅ |
| B 间歇但命中 | 10 | 100 ms | **10 ms/次 ❌** |
| C 高频短等 | 4 | 1,250 ms | **312 ms/次 ❌** |
| D 混合 | 5 | 5,020 ms | **模糊不可分** |

**同一个 LAS 命中数，真实等待时间能差 2 个数量级**——这就是为什么"事件次数 × 采样间隔 ≈ 等待时长"在高频短等待场景下彻底失灵。

### 工程含义

1. **只能信趋势，不能信绝对值**：命中数从 100 涨到 1000 可以说"更严重了"，但不能说"等了 10 倍时间"
2. **区分不了"一直卡"和"频繁短卡"**：前者可能是锁死/网络超时，后者可能是抖动，处置方案完全不同
3. **真正需要的指标**：`total_wait_time`（累计等待毫秒）、`wait_count`（发生次数）—— Oracle ASH / pg_wait_sampling 都有
4. **临时弥补**：
   - 提高采样频率（LAS 1Hz → 10Hz，敏感度涨一个量级）
   - 对可疑等待点开 perf/ebpf trace
   - 用 pg_stat_activity 前后对比 state_change 时间粗估

---

## 二、Oracle ASH vs GaussDB LAS 全方位对比

Oracle ASH 的 **TIME_WAITED + Fixup 机制**是它最核心的设计，GaussDB LAS 至今没对齐。
下面从 7 个维度对比。

### 维度 1：核心差距 TIME_WAITED 字段 + Fixup 机制

Oracle ASH 的杀手锏不只是有 `TIME_WAITED` 字段，而是**"回填修正（fixup）"机制**：
采样时如果会话正在等待，先记一行；等该等待事件真正结束后，ASH 会**回头更新之前那行的
TIME_WAITED 字段**填入真实时长。

```
场景：采样到一次跨采样周期的等待（实际等了 2.3 秒）

时间   0s    1s    2s    3s    4s    5s
       ↓     ↓     ↓     ↓     ↓     ↓
会话   ──────███████████████───────────
                  ↑         ↑
                 等待开始   等待结束（持续 2.3s）

───────────────────── Oracle ASH ─────────────────────
采样 1s: 记录一行 {EVENT=X, TIME_WAITED=0}         ← in-flight，先占位
采样 2s: 记录一行 {EVENT=X, TIME_WAITED=0}         ← 还在等
等待结束时（3.1s）: FIXUP 触发
   → 回填 1s 那行：{EVENT=X, TIME_WAITED=2300000μs} ✅
   → 回填 2s 那行：{EVENT=X, TIME_WAITED=1300000μs} ✅

结果：两行都能还原出真实等待时长

───────────────────── GaussDB LAS ─────────────────────
采样 1s: 记录 {event=X}  ← 只知道"在等 X"，不知等多久
采样 2s: 记录 {event=X}  ← 还是只知道"在等 X"
等待结束：无任何回填逻辑

结果：只能数次数估时长（见上节误差问题）
```

**差距**：ASH 的一行数据自带"这次等了多久"，LAS 的一行数据只能告诉你"此刻在等"。

### 维度 2：采样记录字段丰富度

```
 ┌────────────────────────────────────────────────────────────┐
 │              Oracle ASH 一行记录（简化版 ~30 字段）         │
 ├────────────────────────────────────────────────────────────┤
 │ SAMPLE_TIME        2024-03-15 10:32:45.123                 │
 │ SESSION_ID         892                                     │
 │ SQL_ID             g3fstb4dm5k4m        ← 精确 SQL 定位    │
 │ SQL_EXEC_ID        16777221                                │
 │ SQL_PLAN_HASH      2935830462           ← 执行计划指纹      │
 │ EVENT              db file sequential read                 │
 │ WAIT_CLASS         User I/O                                │
 │ TIME_WAITED        2,347 μs             ★ 等了多久         │
 │ P1/P2/P3           file#=4, block#=8921, cnt=1             │
 │ BLOCKING_SESSION   145                  ★ 谁在阻塞我        │
 │ BLOCKING_SESSION_STATUS VALID                              │
 │ CURRENT_OBJ#       74523                ← 当前操作的对象    │
 │ CURRENT_FILE#      4                                       │
 │ CURRENT_BLOCK#     8921                                    │
 │ IN_PARSE           N                                       │
 │ IN_HARD_PARSE      N                    ← SQL 执行阶段      │
 │ IN_SQL_EXECUTION   Y                                       │
 │ IN_PLSQL_EXECUTION N                                       │
 │ DELTA_TIME         1,002,345 μs         ← 两次采样间 CPU    │
 │ DELTA_READ_IO_REQUESTS 42                                  │
 │ MODULE / ACTION    "SQL*Plus" / "UPDATE_BATCH"             │
 │ CLIENT_ID          "user_42"                               │
 │ ... (共 30+ 字段)                                          │
 └────────────────────────────────────────────────────────────┘

 ┌────────────────────────────────────────────────────────────┐
 │              GaussDB LAS 一行记录（~15 字段）               │
 ├────────────────────────────────────────────────────────────┤
 │ sample_time        2024-03-15 10:32:45                     │
 │ sessionid          140382763438848                         │
 │ pid                12345                                   │
 │ usesysid           10                                      │
 │ application_name   gsql                                    │
 │ query_id           72340172838076673   ← 仅可查当前 SQL 文本│
 │ event              WalWriteLock                            │
 │ wait_status        wait event                              │
 │ xact_start_time    2024-03-15 10:32:40                     │
 │ query_start        2024-03-15 10:32:43                     │
 │ state              active                                  │
 │ ... (无 TIME_WAITED)                                        │
 │ ... (无 BLOCKING_SESSION)                                  │
 │ ... (无 CURRENT_OBJ/FILE/BLOCK)                            │
 │ ... (无 IN_PARSE/IN_EXEC 阶段)                             │
 │ ... (无 DELTA_IO/CPU 指标)                                 │
 └────────────────────────────────────────────────────────────┘
```

### 维度 3：阻塞链（Blocking Chain）追溯能力

```
生产事故：会话 A 的 UPDATE 卡住不动，要找根因

────────── Oracle ASH 做法（一条 SQL 搞定）──────────

SELECT session_id, blocking_session, event, time_waited
FROM v$active_session_history
WHERE sample_time > SYSDATE - 5/1440
CONNECT BY PRIOR blocking_session = session_id
START WITH session_id = 892;

结果（阻塞链一目了然）：
  892  → blocked by 145  (enq: TX row lock, 5.2s)
  145  → blocked by 603  (enq: TX row lock, 8.7s)
  603  → blocked by NULL (idle, holding lock 12s) ← 根凶

────────── GaussDB LAS 做法 ──────────

LAS 没有 blocking_session 字段，只能：
  1. 从 pg_locks 现查当前锁关系（但这是"此刻"，不是历史回溯）
  2. 事故已过去 → pg_locks 已清空 → 无法追溯
  3. 靠运气：如果事故还在进行，可以手工 JOIN pg_locks 还原
  
结果：事后分析几乎做不了，事中靠手工关联。
```

### 维度 4：SQL 执行阶段分解

```
一条慢 SQL（总耗时 10s），时间到底花在哪？

时间线：  ──parse──|──hard_parse──|──execute──|──wait I/O──|──wait lock──
            0.1s       0.3s          5.0s         3.0s         1.6s

────────── Oracle ASH ──────────
IN_PARSE           = 1 次采样  → 1s parse
IN_HARD_PARSE      = 1 次采样  → 1s hard parse
IN_SQL_EXECUTION   = 8 次采样  → 8s 执行（含等待）
  └─ EVENT=CPU              3 次  → 3s CPU
  └─ EVENT=db file seq read 3 次  → 3s I/O
  └─ EVENT=enq: TX          2 次  → 2s 锁等待

能清晰分离：解析 vs 执行；CPU vs I/O vs 锁。

────────── GaussDB LAS ──────────
event 字段只有 1 个维度，每行只能说"此刻在做 X"：
  WalWriteLock × 5
  IO × 3
  CPU × 2

不知道是 parse 阶段的 CPU 还是 execute 阶段的 CPU。
不知道是否跨越了多个执行阶段。
```

### 维度 5：存储架构 — 内存 + 磁盘双层

```
───────── Oracle ASH 两层存储 ─────────

    每 1 秒采样                    每 10 秒 flush
        ↓                              ↓
  ┌─────────────┐             ┌──────────────────┐
  │ V$ACTIVE_   │             │ DBA_HIST_ACTIVE_ │
  │ SESSION_    │──1/10 采样─→│ SESS_HISTORY     │
  │ HISTORY     │             │ (AWR, 磁盘)      │
  │ (内存环形)  │             │                  │
  │ 保留 ~1 小时│             │ 保留 8–30 天     │
  └─────────────┘             └──────────────────┘
        ↑                              ↑
   在线故障诊断                 历史趋势分析

  加上 ADDM、AWR Report、SQL Monitor、SQL Tuning Advisor
  整套工具链都基于这两层数据构建

───────── GaussDB LAS ─────────

  ┌─────────────┐
  │ GS_ASP /    │     部分版本有 ASP（Active Session Profile）
  │ LOCAL_      │     做了持久化，但采样粒度、字段数都比 ASH 少
  │ ACTIVE_     │
  │ SESSIONS    │     内存环形保留有限（几千到几万行）
  └─────────────┘     溢出后老数据直接丢

  历史分析困难：1 小时前的 spike，大概率已被覆盖。
```

### 维度 6：增量指标（Delta Metrics）

Oracle ASH 每次采样会算"距上次采样期间这个会话干了什么"：

```
                t=10s           t=11s
                  ↓               ↓
 会话活动： ─────(做了各种事)──────
                  
 采样 t=11s 时 ASH 记录的 delta 字段：
   DELTA_TIME              = 1,000,000 μs   ← 这 1 秒挂着什么状态
   DELTA_READ_IO_REQUESTS  = 42             ← 读了 42 次 I/O
   DELTA_READ_IO_BYTES     = 344 KB
   DELTA_WRITE_IO_REQUESTS = 0
   DELTA_INTERCONNECT_IO_BYTES = 0           ← RAC 跨节点传输
   PGA_ALLOCATED           = 12 MB
   TEMP_SPACE_ALLOCATED    = 0
   
 有了这些，能算出：
   - 每个 SQL 的 I/O 画像 
   - 谁在吃 PGA 内存
   - RAC 节点间流量热点

 GaussDB LAS：无 delta 字段，全靠 pg_stat_database/tables 间接推算。
```

### 维度 7：生态工具链

```
┌────────────────────┬─────────────────────┬────────────────────┐
│ 分析能力            │ Oracle ASH          │ GaussDB LAS        │
├────────────────────┼─────────────────────┼────────────────────┤
│ 实时 session 监控   │ v$active_session_    │ pg_stat_activity   │
│                    │ history + EM        │ + 自研脚本          │
├────────────────────┼─────────────────────┼────────────────────┤
│ SQL 执行追踪        │ SQL Monitor (实时    │ 无对等工具          │
│                    │ 执行计划 + 等待)      │                    │
├────────────────────┼─────────────────────┼────────────────────┤
│ 历史报告            │ AWR Report          │ WDR Report（弱化版）│
│                    │ (60+ 维度分析)      │ (10+ 维度)         │
├────────────────────┼─────────────────────┼────────────────────┤
│ 自动诊断            │ ADDM                │ 无                 │
├────────────────────┼─────────────────────┼────────────────────┤
│ SQL 调优建议        │ SQL Tuning Advisor  │ 无                 │
├────────────────────┼─────────────────────┼────────────────────┤
│ 阻塞链 + 会话画像   │ 原生               │ 需手工拼 pg_locks  │
├────────────────────┼─────────────────────┼────────────────────┤
│ ASH Analytics UI   │ Oracle EM / OEM     │ 无官方 UI          │
└────────────────────┴─────────────────────┴────────────────────┘
```

### 差距可视化总览

```
维度                  Oracle ASH   GaussDB LAS   GAP
                      ─────────    ──────────    
TIME_WAITED 字段       ██████████   ░░░░░░░░░░   最大差距 ★★★★★
Fixup 回填机制         ██████████   ░░░░░░░░░░   最大差距 ★★★★★
字段丰富度（30 vs 15）  ██████████   █████░░░░░   显著      ★★★★
阻塞链追溯            ██████████   █░░░░░░░░░   严重      ★★★★★
SQL 阶段分解          ██████████   ██░░░░░░░░   显著      ★★★★
历史持久化            ██████████   ████░░░░░░   中等      ★★★
Delta 指标           ██████████   █░░░░░░░░░   严重      ★★★★
自动诊断工具          ██████████   ░░░░░░░░░░   无对等    ★★★★★
实时执行监控          ██████████   ██░░░░░░░░   显著      ★★★★
```

### 设计哲学差异

```
Oracle ASH:
  "每一次采样都是一次高维事件快照，未完成的等待会被 fixup 回填。
   把采样 + 补偿机制拼起来，得到接近 trace 精度的观测数据。"
     ↓
   结果：低开销（<1% CPU）+ 高精度（μs 级等待时长）

GaussDB LAS:
  "每一次采样就是 pg_stat_activity 的一行快照。
   谁在等什么 = 问题的全貌。"
     ↓
   结果：低开销 + 低精度（只有"在不在等"）
```

### 实际工程含义

| 场景 | Oracle ASH 做得到 | GaussDB LAS 当前局限 |
|---|---|---|
| 昨晚 02:15 的业务 spike 诊断 | 回查 AWR，1 分钟出结论 | 内存可能已覆盖，只能从日志猜 |
| 慢 SQL 时间分布（解析/执行/等待） | 一条 SQL 直接出饼图 | 需要额外开 explain analyze + 抓包 |
| 谁阻塞了我的事务（历史） | blocking_session 链查询 | 事故过后无法追溯 |
| 定位具体文件/块的热点 I/O | current_obj/file/block 直接聚合 | 要配 pg_stat_io_queries 等扩展 |
| 判断一次长等待 vs 100 次短等 | TIME_WAITED 直接区分 | 无法区分 |

**一句话结论**：Oracle ASH 是**"采样 + 时长"**（quantitative）观测，GaussDB LAS 是**"采样 + 标签"**（qualitative）观测，差一代工程投入。要追上 ASH，LAS 最关键的两个缺口是 `TIME_WAITED` 字段和 `BLOCKING_SESSION` 字段。

---

## 三、GaussDB 类似 v$sqlstat 的视图：dbe_perf.statement

### 对应关系总览

```
┌────────────────────────────────┬────────────────────────────────────┐
│ Oracle                          │ GaussDB                            │
├────────────────────────────────┼────────────────────────────────────┤
│ v$sqlstats（累计）              │ dbe_perf.statement                 │
│ DBA_HIST_SQLSTAT（历史快照）    │ WDR 里的 statement snapshot        │
│ v$sql（单次游标执行）           │ dbe_perf.statement_history         │
│ 10046 trace                    │ statement_history.details (部分)   │
│ v$session / ASH                │ pg_stat_activity /                 │
│                                │ dbe_perf.local_active_session      │
└────────────────────────────────┴────────────────────────────────────┘

前置开关：
  enable_resource_track = on
  instr_unique_sql_count = 10000   -- 追踪多少条 unique SQL
  track_stmt_stat_level = 'L1,L1'  -- 普通 SQL / 全量 SQL 追踪级别
  log_min_duration_statement = 500 -- 超过多少 ms 进 statement_history
```

### 核心视图：`dbe_perf.statement`

按 `unique_sql_id` 聚合（等价于 Oracle 的 SQL_ID），字段已经做了时间分解：

```
┌─────────────────────────────┬────────────────────────────────────┐
│ 字段                         │ 对应 Oracle 字段 / 含义             │
├─────────────────────────────┼────────────────────────────────────┤
│ unique_sql_id               │ SQL_ID                             │
│ query                       │ SQL 文本（归一化）                  │
│ n_calls                     │ EXECUTIONS                         │
│                             │                                    │
│ ─── 时间分解（纳秒/微秒级）─── │                                    │
│ db_time                     │ ≈ ELAPSED_TIME                     │
│ cpu_time                    │ CPU_TIME                           │
│ execution_time              │ 纯执行时间（不含 parse/plan）       │
│ parse_time                  │ 解析时间                           │
│ plan_time                   │ 生成计划时间                        │
│ rewrite_time                │ 重写时间                           │
│ pl_execution_time           │ PL 执行                            │
│ pl_compilation_time         │ PL 编译                            │
│ data_io_time                │ USER_IO_WAIT_TIME                  │
│ net_send_info (含 time)     │ 客户端网络传输                      │
│ net_recv_info               │                                    │
│ net_stream_send_info        │ 分布式节点间流（DWS/TDSQL）         │
│ net_stream_recv_info        │                                    │
│                             │                                    │
│ ─── 执行动作 ───             │                                    │
│ n_returned_rows             │ ROWS_PROCESSED                     │
│ n_tuples_fetched/returned   │ 行级细分                           │
│ n_tuples_inserted/updated   │                                    │
│ n_tuples_deleted            │                                    │
│ n_blocks_fetched            │ BUFFER_GETS + DISK_READS           │
│ n_blocks_hit                │ ≈ BUFFER_GETS（cache 命中）         │
│ n_soft_parse                │ soft parse 次数                    │
│ n_hard_parse                │ HARD_PARSES                        │
│                             │                                    │
│ ─── 算子级细节（Oracle 没有）── │                                    │
│ sort_count, sort_time       │ 排序次数/耗时                      │
│ sort_mem_used               │ 排序内存                           │
│ sort_spill_count/size       │ 下盘次数/量                        │
│ hash_count, hash_time       │ Hash 算子                          │
│ hash_mem_used               │                                    │
│ hash_spill_count/size       │                                    │
│                             │                                    │
│ ─── 锁相关（Oracle 得查其他视图）─ │                                  │
│ lock_count, lock_time       │ 锁获取次数 + 持有时间               │
│ lock_wait_count             │ 锁等待次数                         │
│ lock_wait_time              │ ≈ APPLICATION_WAIT_TIME            │
│ lock_max_count              │ 最大并发锁数                        │
│                             │                                    │
│ last_updated                │                                    │
└─────────────────────────────┴────────────────────────────────────┘
```

### 用 dbe_perf.statement 诊断 0.3ms → 0.5ms 慢化

```sql
-- 计算每个 unique SQL 的平均执行画像
SELECT
  unique_sql_id,
  LEFT(query, 60)                               AS sql_,
  n_calls,
  ROUND(db_time * 1.0      / n_calls / 1000, 3) AS avg_db_ms,
  ROUND(cpu_time * 1.0     / n_calls, 1)        AS avg_cpu_us,
  ROUND(data_io_time * 1.0 / n_calls, 1)        AS avg_io_us,
  ROUND(lock_wait_time * 1.0 / n_calls, 1)      AS avg_lock_us,
  ROUND(parse_time * 1.0   / n_calls, 1)        AS avg_parse_us,
  ROUND(plan_time * 1.0    / n_calls, 1)        AS avg_plan_us,
  ROUND(sort_time * 1.0    / n_calls, 1)        AS avg_sort_us,
  ROUND(hash_time * 1.0    / n_calls, 1)        AS avg_hash_us,
  ROUND(n_blocks_fetched * 1.0 / n_calls, 2)    AS lio_per_exec,
  ROUND((n_blocks_fetched - n_blocks_hit) * 1.0 / n_calls, 4) AS pio_per_exec,
  ROUND(n_hard_parse * 1.0 / n_calls, 4)        AS hparse_ratio
FROM dbe_perf.statement
WHERE unique_sql_id = 3295726549
ORDER BY n_calls DESC;
```

### 单次执行级：`dbe_perf.statement_history`

慢 SQL（超过 `log_min_duration_statement` 阈值）会在 `statement_history` 留下单次执行的详细记录。
**这比 Oracle v$sqlstats 更进一步——直接给出单次执行时的每个等待事件明细**。

```
┌─────────────────────────────┬─────────────────────────────────────┐
│ 字段                         │ 说明                                 │
├─────────────────────────────┼─────────────────────────────────────┤
│ debug_query_id              │ 本次执行唯一 ID（类似 SQL_EXEC_ID）   │
│ unique_query_id             │ unique_sql_id                       │
│ query                       │ 完整 SQL                             │
│ start_time, finish_time     │ 开始/结束时间                        │
│ slow_sql_threshold          │ 触发阈值                             │
│                             │                                     │
│ db_time / cpu_time / ...    │ 所有时间字段（同 statement 视图）     │
│ lock_count / lock_wait_time │                                     │
│ n_tuples_*                  │                                     │
│                             │                                     │
│ ★ details                  │ 等待事件明细链（"mini 10046 trace"）│
│ query_plan                  │ 执行计划文本                         │
└─────────────────────────────┴─────────────────────────────────────┘
```

`details` 字段展开后类似 mini 10046 trace（Oracle 没直接对应的）：

```
[
  { time: 2024-04-22 10:32:45.123, event: 'LWLock:ProcArrayLock', wait: 1230us },
  { time: 2024-04-22 10:32:45.125, event: 'IO:DataFileRead',       wait:  450us },
  { time: 2024-04-22 10:32:45.126, event: 'LWLock:WALWriteLock',   wait: 2340us },
  { time: 2024-04-22 10:32:45.128, event: 'WaitEvent:wait wal sync',wait: 8970us },
  ...
]
```

### Oracle v$sqlstats vs GaussDB dbe_perf.statement 对比

```
┌────────────────────────────────┬──────────┬──────────┐
│ 维度                            │ Oracle   │ GaussDB  │
├────────────────────────────────┼──────────┼──────────┤
│ 累计时间分解（CPU/IO/解析等）    │ ████████ │ ████████ │ 打平
│ 锁等待单独字段                  │ ░░░░░░░░ │ ███████  │ GaussDB 胜
│ 排序/Hash 算子级细节             │ ░░░░░░░░ │ ███████  │ GaussDB 胜
│ 并发争用细分（latch/mutex）      │ ████████ │ ███░░░░░ │ Oracle 胜
│ 集群跨节点等待                   │ ████████ │ ██████░░ │ 基本打平
│ 历史快照 + 时间序列对比           │ ████████ │ ██████░░ │ WDR 弱于 AWR  
│ 单次执行等待事件序列             │ ██░░░░░░ │ ███████  │ GaussDB 胜
│ 执行计划每步耗时                 │ ███████░ │ ██░░░░░░ │ Oracle 胜
│ 绑定变量捕获                    │ ██████░░ │ ░░░░░░░░ │ Oracle 胜
│ 自动诊断（ADDM/SQL Advisor）     │ ████████ │ ░░░░░░░░ │ Oracle 胜
│ unique_sql 容量管理             │ N/A      │ ⚠️ LRU 溢出 │ GaussDB 隐患
└────────────────────────────────┴──────────┴──────────┘
```

### GaussDB 特有的坑

```
⚠️ instr_unique_sql_count 限制
   默认值较小（如 1000），超出后 LRU 淘汰
   → 诊断时发现某个 SQL "查不到了"，可能不是没执行过，是被踢出了
   → 生产环境建议调到 10000+，占一点内存换观测精度

⚠️ statement_history 的持久化
   默认只保留有限行（几万到几十万），写到本地文件
   老记录会被清理，不像 AWR 能按天做保留策略
   → 重要事故需要及时导出

⚠️ track_stmt_stat_level 分级
   'OFF/L0/L1/L2' 四级精度不同：
     OFF = 不记录
     L0  = 只记执行次数
     L1  = 记时间分解但不记 details
     L2  = 全量，含 details 等待序列
   L2 开销约 5-8% CPU，生产默认 L1

⚠️ 分布式版（DWS/TDSQL）每个节点各自统计
   要看全局画像需要跨节点聚合
```

---

## 四、短暂性能 spike 的捕获方法（0.3 → 0.5ms 持续 2 秒）

### 为什么 2 秒 spike 是观测难题

```
假设：高频 SQL 10 万次/秒，0.3 → 0.5ms（多 200μs/次），持续 2s

 多出来的总 DB 时间：
   100,000 × 2 × 200μs = 40,000,000 μs = 40 秒 DB time
   
 在不同时间窗里看信号强度：

 观测窗    异常 DB time 占比    能看到吗？
 ──────    ────────────────    ────────
  1 小时      40s / 3600s = 1.1%    ❌ 埋进均值里
 10 分钟     40s /  600s = 6.7%    ⚠️ 需要盯着看
  1 分钟     40s /   60s = 67%     ✅ 非常明显  
 10 秒       40s /   10s = 400%    ✅ 爆炸显著
  1 秒       40s /    2s = 2000%   ✅ 触目惊心

 → 观测窗 ≤ 10 秒才能清晰捕获 2 秒 spike
 → WDR 默认 1 小时、LAS 默认 1Hz 都基本看不见
```

### 5 种捕获方案

#### 方案 1：高频 delta 轮询（最实用）

外挂一个采集进程，**每秒**查 `dbe_perf.statement`，算 delta，发现异常立即 dump 全景。

```python
prev = {}
baseline = {}  # 历史基线

while True:
    rows = db.query("""
      SELECT unique_sql_id, n_calls, db_time, lock_wait_time, data_io_time
      FROM dbe_perf.statement
      WHERE n_calls > 100000
    """)
    
    for r in rows:
        sid = r.unique_sql_id
        if sid in prev:
            dcalls = r.n_calls - prev[sid].n_calls
            dtime  = r.db_time - prev[sid].db_time
            if dcalls > 0:
                avg_us = dtime / dcalls
                if sid in baseline and avg_us > baseline[sid] * 1.5:
                    dump_full_snapshot(sid, avg_us)
                    alert(f"SQL {sid} slowdown: {baseline[sid]:.1f}→{avg_us:.1f}μs")
        prev[sid] = r
    
    time.sleep(1)
```

**优点**：不侵入内核，开销可控，秒级捕获
**缺点**：基线要自己维护

#### 方案 2：降低 slow SQL 阈值 + 打开 L2 trace

```
日常模式（低开销）:
  log_min_duration_statement = 500
  track_stmt_stat_level = 'L1,L0'

嫌疑时段"开窗期模式":
  log_min_duration_statement = 0
  track_stmt_stat_level = 'L2,L2'

代价：5-8% CPU 开销 + 写入量暴增
适用：已知大概时段，开窗抓
```

#### 方案 3：客户端侧计时（最可靠的外部信号）

```
应用层 JDBC/连接池拦截器每次 SQL 都打点：
  long t0 = System.nanoTime();
  execute(sql);
  long elapsed = System.nanoTime() - t0;
  if (elapsed > threshold_us) {
      log(sql_id, elapsed, bind_values, session_id, txn_id);
  }
       ↓
  实时写 Kafka / Log
       ↓
  流处理 (Flink/Spark) 滑动窗口 5 秒
       ↓
  P99 > 500μs 告警
       ↓
  告警触发 → 调用 DB 侧 dump
```

**优势**：100% 捕获率，不是采样；自带业务上下文；可做 trace ID 关联

#### 方案 4：LAS 加速采样 + 专用存储

```
默认: asp_sample_interval = 1s
诊断期: asp_sample_interval = 100ms (10Hz)

数学：10Hz × 2 秒 = 20 个样本足够统计
代价：CPU 开销从 <0.1% 涨到 ~1%
```

#### 方案 5：eBPF / perf 挂探针（终极精度）

```bash
bpftrace -e '
  uprobe:/opt/gaussdb/bin/gaussdb:exec_simple_query {
      @start[tid] = nsecs;
  }
  uretprobe:/opt/gaussdb/bin/gaussdb:exec_simple_query 
  /@start[tid]/ {
      $dur = nsecs - @start[tid];
      if ($dur > 400000) {
          printf("%d %s\n", $dur, str(arg0));
      }
      delete(@start[tid]);
  }'
```

纳秒级计时，0 漏掉；需要 root + perf_event

### 5 种方案对比

```
┌───────────────────────────┬────────┬────────┬────────┬─────────┬────────┐
│ 方案                       │ 捕获率  │ 延迟   │ 开销    │ 部署难度 │ 适用    │
├───────────────────────────┼────────┼────────┼────────┼─────────┼────────┤
│ ① 高频 delta 轮询           │ 95%    │ 1-2s   │ 极低    │ 低      │ 日常    │
│ ② slow 阈值=0 + L2 trace   │ 100%   │ 秒级    │ 5-8%   │ 中      │ 开窗抓  │
│ ③ 客户端计时               │ 100%   │ 实时    │ 微乎   │ 高      │ 生产默认│
│ ④ LAS 10Hz 采样             │ 80%    │ 亚秒    │ 1%     │ 中      │ 诊断期  │
│ ⑤ eBPF uprobe              │ 100%   │ 实时    │ 1-3%   │ 高      │ 终极攻坚│
└───────────────────────────┴────────┴────────┴────────┴─────────┴────────┘
```

### 生产级架构推荐

```
第 1 层（必做）：客户端侧计时 + 流处理
  ├─ 所有 SQL 执行打点（<1μs 开销）
  ├─ 每个 app 节点本地聚合 1 秒桶
  ├─ 实时流（Kafka + Flink）做全局 P99
  └─ P99 异常 → 立即触发第 2 层

第 2 层（必做）：DB 侧高频 delta 采集
  ├─ dbe_perf.statement 每秒 delta
  ├─ local_active_session 每秒导出
  ├─ 写时序库（Prometheus / VictoriaMetrics）
  └─ 异常触发 → 调用第 3 层

第 3 层（触发式）：全景快照
  ├─ 被第 1/2 层告警触发
  ├─ 瞬间抓取 20+ 视图 / 系统指标
  ├─ 存档到对象存储（按事件 ID 归档）
  └─ 生成事件报告发工单

第 4 层（按需开启）：L2 trace / eBPF
  ├─ 重大事故不复现时开启
  ├─ 抓 24-48 小时
  └─ 必出一次 → 关闭
```

### 捕获后的根因定位

```
时刻 T (检测到 spike)
  │
  ├── DB 内部
  │   ├─ local_active_session（T-1s 到 T+2s 所有采样）
  │   ├─ dbe_perf.statement_history
  │   ├─ dbe_perf.wait_events（实例级累计等待）
  │   ├─ dbe_perf.global_xact_status
  │   ├─ pg_stat_bgwriter / checkpointer
  │   └─ pg_stat_replication
  │
  ├── OS 层
  │   ├─ /proc/diskstats 每秒采样
  │   ├─ /proc/net/dev
  │   ├─ /proc/pressure/{cpu,io,memory} (PSI)
  │   └─ perf / off-CPU flamegraph
  │
  └── 存储层（Dorado）
      ├─ 存储侧 SCSI timeout / latency spike
      └─ 双集群 HyperMetro 同步 RTT
```

---

## 五、GaussDB 监听日志查看

GaussDB **没有像 Oracle 那样独立的 LSNR listener 进程**，连接/监听事件都记录在主服务器日志里。

### 日志位置

```bash
# 默认目录（集中式版）
$GAUSSLOG/pg_log/<node_name>/postgresql-YYYY-MM-DD_HHMMSS.log

# 典型完整路径
/var/log/gaussdb/omm/pg_log/dn_6001/postgresql-*.log

# 分布式版各节点日志
$GAUSSLOG/gs_log/<instance>/pg_log/

# 查环境变量
echo $GAUSSLOG
gs_om -t view
```

### 开启连接监听日志

```conf
# postgresql.conf
log_connections = on           # 记录连接建立
log_disconnections = on        # 记录连接断开
log_hostname = on              # 记录客户端主机名（会慢，按需）
log_line_prefix = '%m [%p] %u@%d %h '
```

```sql
-- 不用重启
ALTER SYSTEM SET log_connections = on;
ALTER SYSTEM SET log_disconnections = on;
SELECT pg_reload_conf();
```

```bash
gs_guc reload -Z coordinator -N all -I all -c "log_connections=on"
gs_guc reload -Z datanode    -N all -I all -c "log_connections=on"
```

### 查看日志

```bash
# 实时看最新
tail -f $GAUSSLOG/pg_log/*/postgresql-*.log

# 按时间定位
grep "2026-04-24 10:3" $GAUSSLOG/pg_log/*/postgresql-*.log

# 只看连接事件
grep -E "connection (received|authorized|authenticated|closed)" \
  $GAUSSLOG/pg_log/*/postgresql-*.log

# 看失败的登录
grep -E "FATAL|authentication failed|no pg_hba.conf entry" \
  $GAUSSLOG/pg_log/*/postgresql-*.log
```

### 典型日志条目含义

```
LOG: connection received: host=10.0.0.5 port=54321
  ← 监听进程接到 TCP 连接

LOG: connection authorized: user=omm database=postgres
  ← 认证通过

FATAL: no pg_hba.conf entry for host "10.0.0.5"
  ← 白名单未配置（最常见的"连不上"原因）

FATAL: password authentication failed for user "omm"
  ← 密码错

LOG: disconnection: session time: 0:00:03.123 user=omm database=postgres
  ← 连接断开 + 会话时长
```

### 其他相关日志

```
$GAUSSLOG/cm/cm_agent/          # CM 集群管理（主备切换、节点状态）
$GAUSSLOG/cm/cm_server/
$GAUSSLOG/gs_profile/           # 性能采样
$GAUSSLOG/bin/gs_ctl/           # 启停操作
$GAUSSLOG/bin/gs_om/            # 运维工具
$GAUSSLOG/asp_data/             # LAS/ASP 持久化
```

### 实时查询当前连接

```sql
-- 当前所有连接
SELECT pid, usename, application_name, client_addr, client_port,
       backend_start, state, query
FROM pg_stat_activity;

-- 监听端口确认
SHOW port;
SHOW listen_addresses;
```

```bash
# OS 层
netstat -tlnp | grep gaussdb
ss -tlnp | grep 5432
ps -ef | grep gaussdb
```

### 常见排查命令（"连不上 GaussDB"三步走）

```bash
# 1. 进程是否在
ps -ef | grep gaussdb

# 2. 端口是否监听
ss -tlnp | grep -E "5432|15400"

# 3. 最近有没有拒绝连接
tail -200 $GAUSSLOG/pg_log/*/postgresql-*.log | grep -E "FATAL|ERROR"
```

---

## 六、GaussDB 时间戳处理函数

### 把时间戳截取到秒

```sql
-- 方式 1: date_trunc（PG 兼容语法，所有 GaussDB 版本都支持）
SELECT date_trunc('second', your_timestamp);

-- 方式 2: TRUNC（Oracle 兼容语法，需在 A 兼容数据库下）
SELECT TRUNC(your_timestamp, 'SS');
```

| 写法 | 适用 | 返回类型 |
|---|---|---|
| `date_trunc('second', ts)` | 任何兼容模式 | timestamp（保留毫秒字段但置 0） |
| `TRUNC(ts, 'SS')` | 仅 `DBCOMPATIBILITY = 'A'` 库 | timestamp |

```sql
-- 示例
SELECT date_trunc('second', TIMESTAMP '2026-04-24 10:32:45.678');
-- 输出: 2026-04-24 10:32:45

SELECT TRUNC(TIMESTAMP '2026-04-24 10:32:45.678', 'SS');
-- 输出: 2026-04-24 10:32:45
```

注意 `TRUNC(ts)` 不带格式参数时默认截到天（00:00:00），跟 Oracle 行为一致。要截到秒必须显式写 `'SS'`。

### 其他相关函数

| 函数 | 兼容模式 | 说明 |
|---|---|---|
| `date_trunc('second', ts)` | 所有模式 | 首选，PG 原生 |
| `TRUNC(ts, 'SS')` | 仅 A 兼容 | Oracle 风格 |
| `to_char(ts, 'YYYY-MM-DD HH24:MI:SS')` | 所有模式 | 返回字符串 |
| `ts::timestamp(0)` | 所有模式 | 按精度截断（注意是四舍五入不是截断） |

### 时间戳相减按 100ms 处理

```sql
-- 100ms 四舍五入
SELECT ROUND(EXTRACT(EPOCH FROM (t2 - t1)) * 10) / 10 AS rounded_sec;

-- 100ms 截断
SELECT TRUNC(EXTRACT(EPOCH FROM (t2 - t1)) * 10) / 10 AS truncated_sec;

-- 字符串截取前 N 字节
SELECT SUBSTRB(your_string, 1, N);
```

---

## 七、核心结论汇总

1. **LAS 采样误差**：4 种场景下"命中次数 × 采样间隔"估算等待时长可能有 100 倍误差
2. **Oracle ASH vs GaussDB LAS**：ASH 是"采样 + 时长"（quantitative），LAS 是"采样 + 标签"（qualitative），核心差距是 TIME_WAITED + Fixup
3. **GaussDB v$sqlstats 对应**：`dbe_perf.statement`（累计）+ `dbe_perf.statement_history`（单次执行）+ WDR（历史快照）
4. **dbe_perf 优势**：锁等待、排序/Hash 算子、单次执行 details 比 Oracle 更细
5. **dbe_perf 劣势**：无 ADDM 自动诊断、unique_sql 容量有上限
6. **2 秒 spike 捕获**：默认工具看不见，需客户端计时 + DB 侧 1Hz delta 轮询 + 触发式全景快照三层架构
7. **GaussDB 监听日志**：在 `$GAUSSLOG/pg_log/postgresql-*.log`，开 `log_connections` 记录连接事件
8. **时间戳截秒**：`date_trunc('second', ts)` 是首选，`TRUNC(ts, 'SS')` 仅 A 兼容模式
