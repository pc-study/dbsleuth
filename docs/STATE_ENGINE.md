# DBSleuth State Intelligence Engine 设计

> 状态：Post-MVP 设计草案。实现必须建立在经过真实样本验证的统一事件与证据模型之上，不得阻塞当前 Oracle/Linux 离线解析 MVP。

| 属性 | 内容 |
|---|---|
| 版本 | v0.1 Draft |
| 状态 | 架构提案，待实现与样本验证 |
| 适用范围 | DBSleuth CLI、Agent Snapshot 与 Incident Bundle |

## 1. 背景

传统诊断系统长期保存大量 CPU、内存、磁盘、网络、进程、线程和数据库指标。事故发生后，规则或 AI 需要重新理解大量原始数据，导致存储成本高、分析慢、结论难复用。

State Intelligence Engine 在原始证据与 RCA 之间增加一个确定性的状态层：

```text
原始日志 / 指标 / Trace / Dump
  -> 标准事件
  -> 状态识别
  -> 状态观察
  -> 状态转换
  -> 故障模式
  -> Evidence Graph
  -> RCA 候选
```

它把“某一时刻的全部指标值”压缩为“某个实体进入了什么状态、为什么进入、由哪些证据支持”。状态层用于长期留存、模式比较和 AI 上下文压缩，但不取代原始证据。

## 2. 架构原则

### 2.1 状态不是证据

状态是证据经过版本化规则计算得到的投影。每条状态观察必须至少引用一个 canonical `event_id`；每个事件必须继续指向原始文件 SHA-256、解析器版本和准确行号。

### 2.2 未知不等于正常

没有数据、采集失败、时间戳无法解析或规则不覆盖时，必须输出 `unknown` 或数据质量降级。不得因为“没有看到异常”自动生成正常状态。

### 2.3 推断必须显式标记

如果系统首次观察到异常状态，但没有此前的正常证据，可以生成 `NORMAL -> ABNORMAL` 转换用于展示，但必须设置 `inferred_baseline=true`。它表示推断的起点，不表示已经观测到正常状态。

### 2.4 相关性不等于因果性

状态模式只能产生 `root_cause_candidate`。没有满足明确前置条件、拓扑关系和反证检查时，不得输出“已确认根因”。

### 2.5 AI 不能修改确定性事实

AI 可以归纳状态链、解释术语和生成排查建议，但不能新增不存在的状态、改变规则分值、删除反证或绕过事件引用。

## 3. 三层数据保留

| 层级 | 内容 | 默认用途 | 保留策略 |
|---|---|---|---|
| Level 1 | 状态观察、状态转换、模式命中 | 长期趋势、快速检索、AI 上下文 | 长期保存 |
| Level 2 | 关键指标、标准事件、基线与峰值 | 事故分析和复盘 | 保存事故窗口，如 T-30m 至 T+30m |
| Level 3 | 原始日志、Trace、Dump、调用栈和附件 | 证据复核、深度分析 | 原文件不修改，按需归档或脱敏导出 |

状态压缩降低长期存储量，但任何正式报告必须能够从 Level 1 回到 Level 2，再回到 Level 3。

## 4. 状态编码

### 4.1 编码分区

| 编码范围 | 实体域 |
|---:|---|
| `1xxx` | Host 主机 |
| `2xxx` | Process 进程 |
| `3xxx` | Thread 线程 |
| `4xxx` | Memory 内存 |
| `5xxx` | Database 数据库 |
| `6xxx` | Network 网络 |
| `7xxx` | Storage 存储 |
| `8xxx` | Application 应用 |
| `9xxx` | Incident 故障 |

### 4.2 首批状态字典

| Code | Key | 含义 | 默认级别 |
|---:|---|---|---|
| 1000 | `HOST_NORMAL` | 主机健康 | info |
| 1010 | `HOST_CPU_PRESSURE` | 主机 CPU 压力 | high |
| 1020 | `HOST_MEMORY_PRESSURE` | 主机内存压力 | high |
| 1030 | `HOST_IO_PRESSURE` | 主机 IO 压力 | high |
| 1040 | `HOST_KERNEL_ERROR` | 内核异常 | high |
| 1099 | `HOST_FAILURE` | 主机故障 | critical |
| 2000 | `PROCESS_RUNNING` | 进程正常运行 | info |
| 2110 | `PROCESS_HIGH_CPU` | 进程 CPU 异常 | high |
| 2120 | `PROCESS_MEMORY_GROWING` | 进程内存持续增长 | medium |
| 2130 | `PROCESS_THREAD_LEAK` | 进程线程泄漏 | high |
| 2140 | `PROCESS_BLOCKED` | 进程阻塞 | high |
| 2199 | `PROCESS_CRASH` | 进程崩溃或被终止 | critical |
| 3000 | `THREAD_RUNNING` | 线程运行 | info |
| 3010 | `THREAD_CPU_BUSY` | 线程 CPU 忙 | high |
| 3020 | `THREAD_WAIT_LOCK` | 线程等待锁 | high |
| 3030 | `THREAD_WAIT_IO` | 线程等待 IO | high |
| 3040 | `THREAD_WAIT_NETWORK` | 线程等待网络 | high |
| 3050 | `THREAD_DEADLOCK` | 线程死锁 | critical |
| 3060 | `THREAD_ZOMBIE` | 僵尸线程 | high |
| 4000 | `MEM_NORMAL` | 内存健康 | info |
| 4010 | `MEM_PRESSURE` | 内存压力 | high |
| 4020 | `MEM_SWAP` | 明显换页 | high |
| 4030 | `MEM_LEAK` | 内存泄漏嫌疑 | high |
| 4099 | `MEM_OOM_RISK` | 内存耗尽或 OOM | critical |
| 5000 | `DB_NORMAL` | 数据库健康 | info |
| 5010 | `DB_IO_PRESSURE` | 数据库 IO 压力 | high |
| 5020 | `DB_LOG_WAIT` | 数据库日志等待 | high |
| 5030 | `DB_LOCK_CONTENTION` | 数据库锁竞争 | high |
| 5040 | `DB_MEMORY_PRESSURE` | 数据库内存压力 | high |
| 5060 | `DB_LISTENER_FAILURE` | 监听或连接异常 | high |
| 5080 | `DB_DEGRADED` | 数据库降级 | medium |
| 5099 | `DB_HANG` | 数据库挂起或实例终止 | critical |
| 6000 | `NETWORK_NORMAL` | 网络健康 | info |
| 6030 | `NETWORK_FAILURE` | 网络链路异常 | high |
| 7000 | `STORAGE_NORMAL` | 存储健康 | info |
| 7030 | `STORAGE_IO_FAILURE` | 磁盘或存储 IO 异常 | high |
| 8000 | `APPLICATION_NORMAL` | 应用健康 | info |
| 8040 | `APPLICATION_TIMEOUT` | 应用调用超时或失败 | high |
| 9001 | `MAJOR_INCIDENT` | 重大故障 | critical |

状态字典必须带版本。编码一旦公开，不复用、不改变既有语义；废弃状态使用 `deprecated` 标记并提供替代编码。

## 5. 核心模型

### 5.1 状态观察

```json
{
  "observation_id": "sha256:...",
  "entity_type": "database",
  "entity_id": "orcl1",
  "state_code": 5010,
  "state_key": "DB_IO_PRESSURE",
  "start_time": "2026-08-12T01:44:21+08:00",
  "end_time": "2026-08-12T01:44:59+08:00",
  "occurrence_count": 4,
  "supporting_event_ids": ["sha256:..."],
  "rule_ids": ["ORACLE_STORAGE_FAILURE"],
  "confidence": 0.95,
  "inferred": false
}
```

同一实体连续出现相同状态时进行聚合，保留首次、末次、次数和全部证据 ID。状态中不复制大段原始日志。

### 5.2 状态转换

```json
{
  "transition_id": "sha256:...",
  "entity_type": "database",
  "entity_id": "orcl1",
  "old_state_code": 5000,
  "new_state_code": 5010,
  "transition_time": "2026-08-12T01:44:21+08:00",
  "reason_code": "ORACLE_STORAGE_FAILURE",
  "reason": "数据库证据表明 IO 路径存在压力",
  "supporting_event_ids": ["sha256:..."],
  "confidence": 0.95,
  "inferred_baseline": true
}
```

### 5.3 故障模式

```json
{
  "pattern_key": "storage_to_database",
  "state_codes": [7030, 5010, 8040],
  "supporting_event_ids": ["sha256:...", "sha256:...", "sha256:..."],
  "root_cause_candidate": "存储路径异常",
  "confidence": 0.86,
  "limitations": "时间相关性不能单独证明因果"
}
```

## 6. 状态识别

### 6.1 输入来源

- Oracle、Linux 和后续数据库/中间件日志中的确定性事件；
- DBSleuth Incident Snapshot 的标准事件；
- 指标窗口计算得到的阈值、持续时长、变化率和基线偏离；
- AWR、等待事件、Zabbix、Prometheus、APM 和操作系统证据；
- 已验证的上游 `state_code`。

### 6.2 识别规则

指标规则必须同时定义进入和退出条件，避免界面和状态时间线在阈值附近反复抖动：

```yaml
key: host_cpu_pressure
entity_type: host
enter:
  all:
    - metric: host.cpu.usage_pct
      op: ">="
      value: 80
      for: 120s
exit:
  all:
    - metric: host.cpu.usage_pct
      op: "<"
      value: 65
      for: 180s
cooldown: 300s
state_code: 1010
```

每次规则命中保存实际值、基线、阈值、时间窗和规则版本，不能只保存 `true/false`。

### 6.3 防抖与去重

- `enter_threshold` 和 `exit_threshold` 分离，形成滞回区间；
- 必须满足 `for` 持续时间后才转换；
- 相同实体、状态和冷却窗口内合并；
- 严重度升级可以立即追加转换；
- 乱序数据先按可信时间排序，时间质量差时降低置信度；
- 规则重跑必须幂等，相同输入和规则版本生成相同 ID。

## 7. 实体身份与拓扑

状态不能只绑定显示名称。实体最小身份为：

```text
entity_type + canonical_entity_id + observation_scope
```

优先使用 DBSleuth 资产目录的稳定 ID；离线日志没有稳定 ID 时，使用经脱敏的主机名、实例名或来源文件作为临时实体，并设置 `identity_confidence`。

跨层故障模式需要尽量验证拓扑：

```text
host -> process -> thread
host -> database instance -> cluster
application -> endpoint -> trace -> SQL -> database instance
database -> storage mount / ASM diskgroup
```

没有拓扑匹配时，模式仍可显示时间邻近，但必须降低置信度并注明“未验证同一依赖链”。

## 8. 首批故障模式

| Pattern | 状态链 | 时间窗 | 输出 |
|---|---|---:|---|
| `storage_to_database` | 7030 -> 5010/5099 | 10 分钟 | 存储路径异常候选 |
| `memory_to_failure` | 4099 -> 2199/5099 | 10 分钟 | 主机内存耗尽候选 |
| `network_to_listener` | 6030 -> 5060 | 5 分钟 | 网络链路异常候选 |
| `database_to_application` | 5010/5030 -> 8040 | 5 分钟 | 数据库瓶颈影响应用候选 |

模式规则必须定义：

- 状态顺序和允许的中间状态；
- 最大时间距离；
- 必须匹配的实体或拓扑关系；
- 支持证据与反证；
- 最低数据质量；
- 置信度计算和降级条件；
- 可解释的结论模板。

## 9. 置信度

建议的模式置信度：

```text
confidence =
  0.35 * direct_evidence
  + 0.20 * temporal_order
  + 0.20 * topology_match
  + 0.15 * rule_specificity
  + 0.10 * data_quality
  - contradiction_penalty
```

- `< 0.60`：只显示“证据不足”或低可信观察；
- `0.60 - 0.84`：显示“可能原因”；
- `>= 0.85`：显示“高可信根因候选”；
- 不论分值多高，自动输出仍是候选，人工确认后才能成为已确认根因。

## 10. 输出与命令

计划增加：

```bash
dbsleuth states incident.zip --timezone Asia/Shanghai
dbsleuth analyze incident.zip --timezone Asia/Shanghai
```

分析包输出：

```text
state-dictionary.json
state-timeline.json
state-transitions.json
incident-report.json
incident-report.md
incident-report.html
timeline.json
evidence-index.json
redaction-candidates.json
```

报告中的状态节点可展开查看：规则、实际值、转换原因、事件 ID、原始文件和行号。

## 11. 测试与验收

### 11.1 正确性

- 每个状态有至少一个有效事件引用；
- 每个事件能够回到原始文件和准确行号；
- 相同输入和规则版本生成相同状态、转换和模式 ID；
- 缺失数据不会生成正常状态；
- 时间相关性不会被描述为已确认因果；
- 规则升级可以重放，但旧结果保留规则版本。

### 11.2 稳定性

- 阈值附近波动不会产生高频状态抖动；
- 重复日志被聚合但不丢失次数和时间范围；
- 乱序、重复、跨时区和时钟偏移有回归测试；
- 100 MB 和 1 GB 输入采用流式处理并满足内存预算。

### 11.3 准确度门槛

- 关键状态识别精确率不低于 90%；
- 已知故障链模式精确率不低于 85%；
- 虚构状态或错误证据位置为 0；
- 状态到原始证据的可追溯率为 100%。

## 12. 实施阶段

1. 冻结状态字典、Schema 和实体身份规则。
2. 基于 Oracle Alert 与 Linux 日志实现确定性状态识别。
3. 接入 DBSleuth Snapshot 标准事件和显式状态码。
4. 实现状态去重、转换、防抖和时间质量处理。
5. 实现首批四类故障模式和反证模型。
6. 接入报告、证据图谱与 AI 受约束解释。
7. 使用匿名化真实事故包盲测，达到质量门槛后再扩展数据库和中间件状态。

## 13. 配套文档

- [DBSleuth Incident Bundle 设计](DBSLEUTH_INCIDENT_BUNDLE.md)
- [从存储异常到 Oracle 实例故障的图文案例 Demo](CASE_DEMO_STORAGE_INCIDENT.md)
