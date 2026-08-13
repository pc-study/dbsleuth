<div align="center">

# DBSleuth · 库迹

### 本地优先、基于证据的数据库故障分析 CLI

**库有迹，障可循。**

导入 Oracle 与 Linux 日志，重建故障时间线、聚合同类事件、关联跨层线索，\
并让报告中的每项结论都能回到原始文件与准确行号。

[![License](https://img.shields.io/badge/License-Apache--2.0-2563EB?style=flat-square)](LICENSE)
[![Stage](https://img.shields.io/badge/Stage-Feasibility%20Validation-F59E0B?style=flat-square)](#当前状态)
[![Scope](https://img.shields.io/badge/MVP-Oracle%20%2B%20Linux-C2410C?style=flat-square)](#当前-mvp)
[![Privacy](https://img.shields.io/badge/Privacy-Local--first-059669?style=flat-square)](#安全与隐私)
[![Evidence](https://img.shields.io/badge/Analysis-Evidence--backed-7C3AED?style=flat-square)](#证据原则)

[项目目标](#为什么需要-dbsleuth) · [目标体验](#目标体验) · [当前 MVP](#当前-mvp) · [长期愿景](#长期愿景) · [路线图](ROADMAP.md) · [English](README_EN.md)

</div>

---

> [!IMPORTANT]
> DBSleuth 当前处于立项与可行性验证阶段，尚未发布可用版本。本文中的命令与输出用于说明目标体验，不代表对应功能已经实现。

## 为什么需要 DBSleuth

一次数据库故障往往分散在 Oracle Alert Log、Linux syslog、`journalctl`、内核消息和其他组件日志中。它们使用不同的时间格式、时区、编码和消息结构。

DBA 通常需要手工完成：

```text
搜索错误 → 对齐时间 → 去除重复 → 拼接上下文 → 判断先后关系 → 复制证据 → 编写报告
```

DBSleuth 希望把这个过程收敛为一个离线、可审计的工作流，并稳定回答：

1. **发生了什么？**
2. **最早出现的异常是什么？**
3. **数据库故障前后，操作系统发生了什么？**
4. **每项结论由哪些原始证据支持？**

## 目标体验

```bash
dbsleuth inspect incident.zip
dbsleuth analyze incident.zip --timezone Asia/Shanghai
```

目标输出：

```text
INCIDENT  2026-08-12 01:42:13 +08:00 — 02:17:56 +08:00

01:42:13  HIGH      Linux    Disk latency increased
01:44:21  CRITICAL  Oracle   ORA-00240 detected
01:45:04  CRITICAL  Oracle   Instance terminated
01:45:18  MEDIUM    Network  TNS-12514 detected
02:02:16  INFO      Oracle   Database opened successfully

OBSERVATION
  Storage-related events preceded the instance termination by 56 seconds.

EVIDENCE
  alert_ORCL.log:18231-18236
  messages:9218-9224

CONFIDENCE  0.93 · temporal correlation, not a root-cause claim
```

计划生成自包含 HTML、Markdown、版本化事件 JSON、证据索引以及用户确认后的脱敏日志包。

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

| 能力 | 处理方式 |
|---|---|
| 多源日志 | 自动识别明确支持的数据库与系统日志 |
| 时间对齐 | 保留原始时间，同时标准化到统一时区 |
| 多行事件 | 重建堆栈、ORA 错误及跨行上下文 |
| 重复噪声 | 按确定性指纹聚合，保留次数和时间范围 |
| 跨层关联 | 展示数据库事件前后的系统事件与时间距离 |
| 证据追踪 | 记录来源文件、行号、解析器和规则版本 |
| 脱敏分享 | 先预览候选项，再生成脱敏副本 |

## 当前 MVP

### 计划支持

- Oracle `alert.log`
- Linux syslog、messages 与导出的 `journalctl` 文本
- UTF-8 与 GBK
- `.log`、`.txt`、`.gz` 与 `.zip`
- 时间戳标准化、多行事件重建与确定性事件分类
- 重复事件聚合、证据索引与本地报告
- 本地脱敏候选预览

### 明确不在 MVP 中

- 常驻 Agent、实时采集、eBPF 或动态探针
- 数据库凭据管理或直接连接生产数据库
- Trace、Dump、内存与线程快照采集
- 长期存储、告警、通知或自动修复
- 多租户、控制平面、企业高可用与灾备
- 无证据引用的 AI 根因判断
- 其他数据库、Windows、ASM、CRS 与 Listener 日志

这些边界用于保证首个版本足够小、可验证，并且不会以错误结论换取“看起来智能”。

## 证据原则

> **Evidence before explanation.**

- 确定性解析优先于概率推断。
- 不编造缺失的时间、字段、事件或因果关系。
- 时间邻近不直接证明因果。
- 无法可靠解析时明确标记为未知。
- 每项规则必须配套匿名化样本和回归测试。
- 每份报告必须记录解析器版本和证据位置。

## 安全与隐私

- 默认完全本地运行，初始版本不包含遥测。
- 不连接数据库，不接收或保存数据库密码。
- 不修改原始日志和故障包。
- 限制压缩包展开体积、文件数量、嵌套深度和压缩比。
- 拒绝绝对路径和目录穿越条目。
- HTML 报告严格转义日志内容。
- 脱敏只作用于导出副本，并要求用户预览确认。

## 长期愿景

以下方向与项目主题相关，但均属于 **Post-MVP 研究或规划**，不构成当前产品承诺：

- **State Intelligence Engine**：将带证据的事件投影为可重放状态、转换和故障模式；详见 [设计草案](docs/STATE_ENGINE.md)。
- **eBPF 内核动态观测层**：以 Linux eBPF 捕获调度、进程、Block IO、TCP/Socket、内存和锁的短时变化，定位目标后触发增量 Snapshot；不支持的内核必须自动降级，事件丢失必须作为证据缺口。详见 [生产设计](docs/EBPF_OBSERVABILITY.md)。
- **证据图与受约束 AI**：仅解释已结构化、带引用的事实，不创建确定性事实。
- **更多数据库与操作系统**：根据真实 Issue 需求逐步扩展。
- **可选生态适配器**：版本化的 [DBSleuth Incident Bundle](docs/DBSLEUTH_INCIDENT_BUNDLE.md) 属于 Post-MVP 提案，不是当前运行依赖。
- **在线采集与企业部署**：只在离线 CLI 获得真实验证后评估，并可能拆分成独立项目。

eBPF 不是 Dump 的替代品：它负责发现异常发生的过程和目标对象，Snapshot 与 Dump 负责保存现场，状态机负责压缩演化，证据图负责连接跨层关系。所有高强度探针都必须使用签名白名单、目标范围、资源预算、TTL、Ring Buffer 丢失统计和 Kill Switch。

[存储异常到 Oracle 故障案例](docs/CASE_DEMO_STORAGE_INCIDENT.md) 是验证事件、证据和状态模型的**合成案例**，不是已实现的产品演示。

## 当前状态

| 能力 | 状态 |
|---|---|
| 项目章程与 MVP 边界 | 已定义 |
| 统一事件与证据模型 | 设计中 |
| Oracle/Linux 解析器 | 未实现 |
| 报告生成 | 未实现 |
| 状态智能引擎 | Post-MVP 设计草案 |
| eBPF 内核动态观测 | Post-MVP 生产设计 |
| DBSleuth Incident Bundle | Post-MVP 可选提案 |
| AI 调查助手 | 长期研究方向 |

## 质量门槛

| 指标 | 门槛 |
|---|---:|
| 支持格式的时间戳提取率 | ≥ 95% |
| 严重/高风险事件分类准确率 | ≥ 90% |
| 虚构事件或错误证据位置 | 0 |
| 事件到原始文本的可追溯率 | 100% |
| 无协助完成报告的 DBA | ≥ 10 人 |
| 30 天内重复使用者 | ≥ 5 人 |

eBPF 进入 Post-MVP 生产实现前，还必须满足：不支持的内核自动降级、基础 Agent 不受加载失败影响、所有探针带预算/TTL/Kill Switch、事件序列与 Ring Buffer 丢失 100% 可观测、只加载签名白名单对象。

## 项目文档

| 文档 | 内容 |
|---|---|
| [PROJECT_CHARTER.md](PROJECT_CHARTER.md) | 产品目标、用户、原则与停止条件 |
| [ARCHITECTURE.md](ARCHITECTURE.md) | MVP 架构、事件模型和安全边界 |
| [ROADMAP.md](ROADMAP.md) | 分阶段实现路线 |
| [BACKLOG.md](BACKLOG.md) | 首批 Epic 与工程任务 |
| [docs/STATE_ENGINE.md](docs/STATE_ENGINE.md) | Post-MVP 状态智能设计草案 |
| [docs/EBPF_OBSERVABILITY.md](docs/EBPF_OBSERVABILITY.md) | Post-MVP eBPF 传感器、安全、降级和 Snapshot 联动设计 |
| [docs/DBSLEUTH_INCIDENT_BUNDLE.md](docs/DBSLEUTH_INCIDENT_BUNDLE.md) | Post-MVP 事故数据契约提案 |
| [docs/CASE_DEMO_STORAGE_INCIDENT.md](docs/CASE_DEMO_STORAGE_INCIDENT.md) | 合成的存储到 Oracle 故障案例 |
| [docs/SEVERITY.md](docs/SEVERITY.md) | **权威** 严重度枚举定义（critical/high/medium/low/info/unknown） |
| [docs/EXIT_CODES.md](docs/EXIT_CODES.md) | CLI 退出码规范（0/1/2/3/4/5/6）与机器可读错误码 |
| [docs/REDACTION_POLICY.md](docs/REDACTION_POLICY.md) | 脱敏策略：强制集（Tier 1）+ 候选集（Tier 2）+ fail-safe |
| [README_EN.md](README_EN.md) | English overview |

## 参与贡献

项目当前最需要匿名化 Oracle/Linux 日志样本、时间戳与编码边界案例、可复现分类规则，以及解析器、测试和文档贡献。

> [!CAUTION]
> 不要在公开 Issue 中上传未经人工复核的生产日志、Dump 或线程快照。提交前必须移除密码、Token、连接串、真实 IP、主机名、数据库名、用户名、路径及业务数据。

## 许可证

DBSleuth 使用 [Apache License 2.0](LICENSE) 开源。

---

<div align="center">

**DBSleuth · 库有迹，障可循**

</div>
