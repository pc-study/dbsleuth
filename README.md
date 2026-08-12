<div align="center">

# DBSleuth · 库迹

### 基于证据与状态智能的数据库故障分析工具

**库有迹，障可循。**

从 Oracle 与 Linux 日志中重建故障时间线，把离散事件归纳为系统状态与状态转换，\
关联跨层故障链，并让报告中的每项结论都能回到原始文件与准确行号。

[![License](https://img.shields.io/badge/License-Apache--2.0-2563EB?style=flat-square)](LICENSE)
[![Stage](https://img.shields.io/badge/Stage-Feasibility%20Validation-F59E0B?style=flat-square)](ROADMAP.md)
[![Scope](https://img.shields.io/badge/MVP-Oracle%20%2B%20Linux-C2410C?style=flat-square)](#mvp-范围)
[![Privacy](https://img.shields.io/badge/Privacy-Local--first-059669?style=flat-square)](#安全与隐私)
[![Evidence](https://img.shields.io/badge/Analysis-Evidence--backed-7C3AED?style=flat-square)](#核心原则)
[![State Engine](https://img.shields.io/badge/State%20Engine-Design%20Draft-0EA5E9?style=flat-square)](docs/STATE_ENGINE.md)

[快速了解](#为什么需要-dbsleuth) · [状态智能](#state-intelligence-engine) · [TraceMind 融合](#与-tracemind-融合) · [案例 Demo](docs/CASE_DEMO_STORAGE_INCIDENT.md) · [MVP](#mvp-范围) · [English](README_EN.md)

</div>

---

> [!IMPORTANT]
> DBSleuth 当前处于立项与可行性验证阶段，尚未发布可用版本。本文中的命令展示目标体验，不代表功能已经实现。

## 本次新增设计

| 设计 | 解决的问题 | 文档 |
|---|---|---|
| State Intelligence Engine | 把指标和事件转换为可重放、可追溯的状态时间线 | [状态智能引擎](docs/STATE_ENGINE.md) |
| TraceMind Incident Bundle | 接收脱敏后的指标、拓扑、AWR、APM、告警和证据 | [TraceMind 集成设计](docs/TRACEMIND_INTEGRATION.md) |
| 故障链案例 Demo | 展示存储异常如何演化为 Oracle 故障和 MES 超时 | [端到端图文案例](docs/CASE_DEMO_STORAGE_INCIDENT.md) |

新增设计仍然遵守 DBSleuth 最重要的边界：**状态不是证据，AI 解释也不是证据。** 状态、转换、模式和结论都必须能够回到标准事件，再回到原始文件与准确位置。

## 为什么需要 DBSleuth

一次数据库故障往往分散在多个来源中：Oracle Alert Log、Linux syslog、`journalctl`、内核消息以及其他组件日志。它们使用不同的时间格式、时区、编码和消息结构。

DBA 通常需要手工完成：

```text
搜索错误 → 对齐时间 → 去除重复 → 拼接上下文 → 判断先后关系 → 复制证据 → 编写报告
```

DBSleuth 希望把它变成：

```bash
dbsleuth analyze incident.zip --timezone Asia/Shanghai
```

并稳定回答六个问题：

1. **发生了什么？**
2. **最早的异常是什么？**
3. **数据库故障前后，操作系统发生了什么？**
4. **每项结论由哪些原始证据支持？**
5. **系统状态如何从健康演化到异常？**
6. **哪些证据支持或反驳当前根因候选？**

## 目标输出

```text
INCIDENT  2026-08-12 01:42:13 +08:00 — 02:17:56 +08:00

01:42:13  HIGH      Linux    Disk latency increased
01:44:08  HIGH      ASM      Write latency exceeded 900 ms
01:44:21  CRITICAL  Oracle   ORA-00240 detected
01:45:04  CRITICAL  Oracle   Instance terminated
01:45:18  MEDIUM    Network  Listener returned TNS-12514
02:02:16  INFO      Oracle   Database opened successfully

OBSERVATION
  Storage-related events preceded the instance termination by 56 seconds.

EVIDENCE
  alert_ORCL.log:18231-18236
  messages:9218-9224
  asm_alert.log:7712-7718

CONFIDENCE  0.93 · temporal correlation, not a root-cause claim
```

最终计划生成：

```text
incident-report.html     可交付的自包含故障报告
incident-report.json     机器可读的分析结论与限制
timeline.json            版本化结构事件数据
state-timeline.json      状态观察与状态转换
state-transitions.json   状态变化、原因与规则版本
evidence-index.json      结论与原始证据映射
redacted-logs.zip        用户确认后生成的脱敏日志包
```

## 核心能力

| 能力 | DBSleuth 的处理方式 |
|---|---|
| 多源日志 | 自动识别支持的数据库与系统日志 |
| 时间对齐 | 保留原始时间，同时标准化到统一时区 |
| 多行事件 | 重建堆栈、ORA 错误及跨行上下文 |
| 重复噪声 | 按指纹聚合，同时保留次数和时间范围 |
| 跨层关联 | 展示数据库事件前后的系统事件与时间距离 |
| 状态智能 | 将带证据的事件归纳为状态、转换和故障模式 |
| 证据追踪 | 每个事件记录来源文件、行号和解析器版本 |
| 脱敏分享 | 先预览候选项，再生成脱敏副本 |
| 离线交付 | 输出 HTML、Markdown 与 JSON，不依赖云服务 |

## 工作方式

```mermaid
flowchart LR
    A["日志文件或压缩包"] --> B["安全清点与格式识别"]
    B --> C["来源专用解析器"]
    C --> D["统一事件模型"]
    D --> E["时间与时区标准化"]
    E --> F["分类、聚合与时间关联"]
    F --> G["状态识别与状态转换"]
    G --> H["故障模式与反证检查"]
    H --> I["证据索引"]
    I --> J["脱敏候选扫描"]
    J --> K["HTML / Markdown / JSON"]
```

DBSleuth 的分析流水线以确定性解析和规则为基础。计划中的状态智能层用于压缩和复用系统运行状态，但每个状态仍必须引用标准事件，每个标准事件仍必须回到原始证据。未来即使增加本地 AI 解释，它也只能处理已经结构化、带引用的事实，不能替代原始证据。

## State Intelligence Engine

> 状态：设计草案已形成，待实现与匿名化样本验证。

传统监控系统保存大量原始指标；事故发生后，规则或 AI 还需要重新理解整段数据。DBSleuth 计划增加状态智能层，把“某时刻有哪些值”转换成“哪个实体进入了什么状态、为什么进入、由哪些证据支持”。

```mermaid
flowchart TB
    A["日志 / 指标 / Trace / Dump"] --> B["Canonical Events<br/>统一事件"]
    B --> C["State Recognition<br/>状态识别"]
    C --> D["State Observations<br/>状态观察"]
    D --> E["State Transitions<br/>状态转换"]
    E --> F["Failure Patterns<br/>故障模式"]
    F --> G["Evidence Graph<br/>证据图"]
    G --> H["Root-cause Candidate<br/>根因候选"]
    G --> I["Constrained AI<br/>受约束解释"]
    H --> J["可审计事故报告"]
    I --> J
```

### 状态编码

| 编码 | 实体域 | 示例 |
|---:|---|---|
| `1xxx` | 主机 | CPU、内存、内核故障 |
| `2xxx` | 进程 | 高 CPU、线程泄漏、崩溃 |
| `3xxx` | 线程 | 锁、IO、网络等待与死锁 |
| `4xxx` | 内存 | 压力、Swap、泄漏、OOM 风险 |
| `5xxx` | 数据库 | IO、日志、锁、内存、挂起 |
| `6xxx` | 网络 | 链路异常与连接失败 |
| `7xxx` | 存储 | 延迟、路径与 IO 故障 |
| `8xxx` | 应用 | 超时与业务失败 |
| `9xxx` | 事故 | 重大故障与影响范围 |

状态字典带独立版本。已公开编码不能复用或改变语义；没有数据、采集失败或规则无法覆盖时输出 `unknown`，不能把未知解释为健康。

### 故障演化示例

```text
7030  STORAGE_IO_FAILURE
  ↓ 115 秒，同一存储依赖链
5010  DB_IO_PRESSURE
  ↓ 13 秒
5099  DB_HANG
  ↓ 61 秒
8040  APPLICATION_TIMEOUT

根因候选：存储路径异常
置信度：0.93
限制：缺少 SAN 交换机与阵列日志，尚不能最终确认
```

每个箭头都保存时间距离、拓扑关系、规则版本、支持证据与反证。完整计算过程见 [存储异常到 Oracle 故障案例 Demo](docs/CASE_DEMO_STORAGE_INCIDENT.md)。

### 三层数据保留

| 层级 | 保存内容 | 用途 |
|---|---|---|
| Level 1 | 状态、转换、模式命中 | 长期时间线、快速检索、AI 上下文 |
| Level 2 | 标准事件、关键指标、基线与峰值 | 事故窗口分析与规则重放 |
| Level 3 | 原始日志、Trace、Dump 和附件 | 证据复核与深度诊断 |

状态机降低的是长期存储和理解成本，不是证据标准。正式报告必须能从 Level 1 回到 Level 2，再回到未修改的 Level 3 原始证据。

### 状态引擎约束

- 状态必须引用至少一个标准事件；
- 标准事件必须记录文件、行号、解析器和规则版本；
- 进入与退出阈值分离，并设置持续时间和冷却窗口，避免状态抖动；
- 时间邻近只表示相关性，不直接证明因果；
- 根因自动输出始终是候选，人工确认后才能成为已确认根因；
- AI 可以解释事实和提出建议，不能新增状态、修改分值或删除反证。

完整状态字典、Schema、转换、防抖和准确度门槛见 [docs/STATE_ENGINE.md](docs/STATE_ENGINE.md)。

## 与 TraceMind 融合

TraceMind 负责在线采集和事故现场冻结，DBSleuth 负责本地、离线、可复核的证据分析。两者通过版本化、脱敏后的 Incident Bundle 连接，不共享平台数据库或连接凭据。

```mermaid
flowchart LR
    A["TraceMind<br/>日志、指标、AWR、APM、拓扑"] --> B["脱敏预览<br/>用户确认"]
    B --> C["Incident Bundle<br/>版本、哈希、时间窗口"]
    C --> D["DBSleuth<br/>安全清点与解析"]
    D --> E["事件 + 状态 + 证据图"]
    E --> F["HTML / Markdown / JSON"]
```

| 可以进入 DBSleuth | 默认禁止进入 |
|---|---|
| 已脱敏的标准事件与状态 | `.env` 和数据库连接配置 |
| 指标窗口、阈值和实际值 | 密码、Token、Cookie、私钥与证书 |
| AWR、SQL、APM 和告警证据 | TraceMind 平台数据库文件 |
| 匿名化资产 ID 与拓扑关系 | 未经复核的生产日志和业务数据 |
| 用户明确选择的脱敏附件 | 用户、授权和许可证数据库 |

DBSleuth 不会回连 Incident Bundle 中的主机或数据库地址，也不会执行附件中的 SQL、脚本或命令。连接或解析失败必须显示失败或降级，不能继续生成健康结论。接口细节见 [docs/TRACEMIND_INTEGRATION.md](docs/TRACEMIND_INTEGRATION.md)。

## 计划中的命令行

```bash
# 检查日志包：识别文件、格式、编码、时间范围与解析覆盖率
dbsleuth inspect incident.zip

# 生成故障时间线和报告
dbsleuth analyze incident.zip --timezone Asia/Shanghai

# 筛选高严重度事件
dbsleuth events incident.zip --severity high

# 生成状态时间线和状态转换（规划中）
dbsleuth states incident.zip --timezone Asia/Shanghai

# 预览潜在敏感信息，不修改原文件
dbsleuth redact incident.zip --preview
```

## MVP 范围

### 目标支持

- Oracle `alert.log`
- Linux syslog、messages 与导出的 `journalctl` 文本
- UTF-8 与 GBK
- `.log`、`.txt`、`.gz` 与 `.zip`
- 时间戳标准化和显式时区处理
- 多行事件重建
- ORA 错误及关键 Linux 系统事件识别
- 确定性严重度分类与重复事件聚合
- 受版本控制、可回溯的状态识别与状态时间线
- 来源文件与行号范围证据索引
- HTML、Markdown 与 JSON 报告
- 本地脱敏候选预览

### 暂不支持

- 实时采集、Agent 或长期日志存储
- 数据库凭据管理或直接连接生产数据库
- 告警、通知或自动修复
- Elasticsearch、Loki、Splunk 或 lnav 的通用替代能力
- 无证据引用的 AI 根因判断
- MySQL、PostgreSQL、Windows、ASM、CRS 与 Listener 日志

这些边界用于保证首个版本足够小、可验证，并且不会以错误结论换取“看起来智能”。

## 核心原则

> **Evidence before explanation.**

- 证据优先于解释。
- 确定性解析优先于概率推断。
- 不编造缺失的时间、字段、事件或因果关系。
- 相关性只描述为“先于”“晚于”或“同时出现”，不冒充因果关系。
- 无法可靠解析时明确标记为未知。
- 每项规则都必须配套匿名化样本和回归测试。
- 每份报告必须记录解析器版本和证据位置。

## 安全与隐私

- 默认完全本地运行，初始版本不包含遥测。
- 不连接数据库，不接收或保存数据库密码。
- 不修改原始日志和故障包。
- 压缩包读取限制展开体积、文件数量、嵌套深度和压缩比。
- 拒绝绝对路径和目录穿越条目。
- HTML 报告严格转义日志内容，不执行其中的任何代码。
- 脱敏只作用于导出副本，并要求用户预览确认。

## 质量门槛

MVP 将使用至少 30 份匿名化真实日志包进行盲测。进入下一阶段必须满足：

| 指标 | 门槛 |
|---|---:|
| 支持格式的时间戳提取率 | ≥ 95% |
| 严重/高风险事件分类准确率 | ≥ 90% |
| 虚构事件或错误证据位置 | 0 |
| 事件到原始文本的可追溯率 | 100% |
| 状态到标准事件及原始证据的可追溯率 | 100% |
| 关键状态识别准确率 | ≥ 90% |
| 无协助完成报告的 DBA | ≥ 10 人 |
| 30 天内重复使用者 | ≥ 5 人 |

## 项目路线

```text
Phase 0  匿名样本库与统一事件、状态模型
Phase 1  Oracle + Linux 流式解析器
Phase 2  聚合、状态转换、故障模式、证据报告与脱敏预览
Phase 3  TraceMind Incident Bundle、真实故障包验证与跨平台发行
```

完整计划见 [ROADMAP.md](ROADMAP.md)。

## 项目文档

| 文档 | 内容 |
|---|---|
| [PROJECT_CHARTER.md](PROJECT_CHARTER.md) | 产品目标、用户、原则与停止条件 |
| [ARCHITECTURE.md](ARCHITECTURE.md) | MVP 架构、事件模型和安全边界 |
| [ROADMAP.md](ROADMAP.md) | 12 周分阶段路线图 |
| [BACKLOG.md](BACKLOG.md) | 首批 Epic 与 GitHub Issue 建议 |
| [docs/STATE_ENGINE.md](docs/STATE_ENGINE.md) | 状态编码、状态转换、故障模式与质量约束 |
| [docs/TRACEMIND_INTEGRATION.md](docs/TRACEMIND_INTEGRATION.md) | TraceMind Incident Bundle 集成边界 |
| [docs/CASE_DEMO_STORAGE_INCIDENT.md](docs/CASE_DEMO_STORAGE_INCIDENT.md) | 存储异常到 Oracle 故障的端到端图文 Demo |
| [README_EN.md](README_EN.md) | English overview |

## 参与贡献

项目当前最需要：

- 经确认完成脱敏的 Oracle/Linux 日志样本；
- 时间戳、编码、多行消息等边界案例；
- 可复现的错误分类规则；
- 解析器、测试和文档贡献。

> [!CAUTION]
> 提交样本前，请移除密码、Token、连接串、真实 IP、主机名、数据库名、用户名、路径及业务数据。不要在公开 Issue 中上传未经人工复核的生产日志。

## 许可证

DBSleuth 使用 [Apache License 2.0](LICENSE) 开源。

---

<div align="center">

**DBSleuth · 库有迹，障可循**

</div>
