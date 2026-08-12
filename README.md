<div align="center">

# DBSleuth · 库迹

### 基于证据的数据库故障分析工具

**库有迹，障可循。**

从 Oracle 与 Linux 日志中重建故障时间线，聚合同类事件，关联跨层线索，\
并让报告中的每项结论都能回到原始文件与准确行号。

[![License](https://img.shields.io/badge/License-Apache--2.0-2563EB?style=flat-square)](LICENSE)
[![Stage](https://img.shields.io/badge/Stage-Feasibility%20Validation-F59E0B?style=flat-square)](ROADMAP.md)
[![Scope](https://img.shields.io/badge/MVP-Oracle%20%2B%20Linux-C2410C?style=flat-square)](#mvp-范围)
[![Privacy](https://img.shields.io/badge/Privacy-Local--first-059669?style=flat-square)](#安全与隐私)
[![Evidence](https://img.shields.io/badge/Analysis-Evidence--backed-7C3AED?style=flat-square)](#核心原则)

[快速了解](#为什么需要-dbsleuth) · [工作方式](#工作方式) · [MVP](#mvp-范围) · [路线图](ROADMAP.md) · [English](README_EN.md)

</div>

---

> [!IMPORTANT]
> DBSleuth 当前处于立项与可行性验证阶段，尚未发布可用版本。本文中的命令展示目标体验，不代表功能已经实现。

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

并稳定回答四个问题：

1. **发生了什么？**
2. **最早的异常是什么？**
3. **数据库故障前后，操作系统发生了什么？**
4. **每项结论由哪些原始证据支持？**

## 目标输出

```text
INCIDENT  2026-08-12 01:42:13 +08:00 — 02:17:56 +08:00

01:42:13  HIGH      Linux    Disk latency increased
01:44:08  HIGH      ASM      Write latency exceeded 900 ms
01:44:21  CRITICAL  Oracle   ORA-00240 detected
01:45:04  CRITICAL  Oracle   Instance terminated
01:45:18  MEDIUM    Network  Listener returned ORA-12514
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
timeline.json            版本化结构事件数据
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
    F --> G["脱敏候选扫描"]
    G --> H["证据索引"]
    H --> I["HTML / Markdown / JSON"]
```

DBSleuth 的分析流水线以确定性解析和规则为基础。未来即使增加本地 AI 解释，它也只能处理已经结构化、带引用的事件，不能替代原始证据。

## 计划中的命令行

```bash
# 检查日志包：识别文件、格式、编码、时间范围与解析覆盖率
dbsleuth inspect incident.zip

# 生成故障时间线和报告
dbsleuth analyze incident.zip --timezone Asia/Shanghai

# 筛选高严重度事件
dbsleuth events incident.zip --severity high

# 预览潜在敏感信息，不修改原文件
dbsleuth redact incident.zip --preview
```

## MVP 范围

### 支持

- Oracle `alert.log`
- Linux syslog、messages 与导出的 `journalctl` 文本
- UTF-8 与 GBK
- `.log`、`.txt`、`.gz` 与 `.zip`
- 时间戳标准化和显式时区处理
- 多行事件重建
- ORA 错误及关键 Linux 系统事件识别
- 确定性严重度分类与重复事件聚合
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
| 无协助完成报告的 DBA | ≥ 10 人 |
| 30 天内重复使用者 | ≥ 5 人 |

## 项目路线

```text
Phase 0  匿名样本库与统一事件模型
Phase 1  Oracle + Linux 流式解析器
Phase 2  聚合、关联、证据报告与脱敏预览
Phase 3  真实故障包验证与跨平台发行
```

完整计划见 [ROADMAP.md](ROADMAP.md)。

## 项目文档

| 文档 | 内容 |
|---|---|
| [PROJECT_CHARTER.md](PROJECT_CHARTER.md) | 产品目标、用户、原则与停止条件 |
| [ARCHITECTURE.md](ARCHITECTURE.md) | MVP 架构、事件模型和安全边界 |
| [ROADMAP.md](ROADMAP.md) | 12 周分阶段路线图 |
| [BACKLOG.md](BACKLOG.md) | 首批 Epic 与 GitHub Issue 建议 |
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
