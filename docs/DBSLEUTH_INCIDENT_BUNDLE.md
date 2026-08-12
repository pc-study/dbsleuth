# DBSleuth Incident Bundle 设计

| 属性 | 内容 |
|---|---|
| 版本 | v0.1 Draft |
| 状态 | 数据契约提案，待实现 |

## 1. 定位

DBSleuth Incident Bundle 是 DBSleuth Agent、Collector、离线导入工具与分析引擎之间的标准事故数据包。它将在线采集和离线分析解耦，使分析过程不依赖生产连接，也不需要携带数据库凭据。

```text
DBSleuth Agent / Collector
  -> 冻结日志、指标、线程、内存、数据库、拓扑和变更证据
  -> 生成版本化、可校验的 Incident Bundle

DBSleuth Analysis Engine
  -> 安全清点、解析和验证 Incident Bundle
  -> 生成事件、状态、证据图、根因候选与事故报告
```

同一格式也用于人工收集的日志目录和第三方系统导出数据。来源不同不会改变证据标准：每项事实都必须有来源、时间、实体、完整性和数据质量信息。

## 2. 能力映射

| DBSleuth 数据源 | Bundle 内容 | 分析输出 |
|---|---|---|
| Incident Snapshot | 冻结窗口、任务状态与 Artifact 清单 | 现场完整度与事故时间线 |
| Metrics Collector | 指标样本、聚合、阈值和基线 | 主机、进程与数据库状态 |
| Database Collector | 等待、会话、SQL 指纹、执行计划和结果摘要 | 数据库证据与阻塞关系 |
| Application Probe | Trace/Span、线程、端点和 SQL 指纹 | 应用到数据库影响链 |
| Host Profiler | CPU、内存、磁盘、网卡和虚拟化事实 | 主机资源上下文 |
| Asset Inventory | 稳定资产 ID、拓扑和依赖关系 | 实体解析与关联置信度 |
| Change Collector | 发布、配置、脚本和人工操作 | 变更时间线与候选诱因 |

## 3. 安全边界

### 3.1 可以进入 Bundle

- 已脱敏的标准事件和必要原始证据；
- 指标窗口的汇总值和必要样本；
- 状态编号、转换原因和规则版本；
- AWR、SQL、Trace、线程和日志证据引用；
- 匿名化或授权保留的资产关系；
- 用户明确批准导出的附件和受限快照。

### 3.2 默认禁止进入 Bundle

- `.env`、数据库连接配置和密码；
- Access Token、Cookie、私钥和客户端证书；
- DBSleuth 控制面或元数据数据库文件；
- 未经授权的完整生产日志、Dump 和业务数据；
- 完整进程环境变量；
- 未脱敏的 SQL 参数和查询结果；
- 用户、授权、审批和许可证数据库。

## 4. 目录结构

```text
incident-bundle/
  manifest.json
  timeline/
    canonical-events.jsonl
    alerts.jsonl
  metrics/
    key-series.jsonl.gz
  states/
    observations.jsonl
    transitions.jsonl
  host/
  process/
  threads/
  memory/
  database/
  probes/
  changes/
  evidence/
    evidence-index.json
    graph.json
  attachments/
    selected-redacted-logs/
  checksums.sha256
```

`manifest.json` 记录 Schema 版本、事故 ID、资产、采集窗口、时区、时钟质量、Artifact 状态、数据等级、文件大小、SHA-256、脱敏状态、采集器版本和缺失原因。

## 5. 最小事件记录

```json
{
  "schema_version": "1.0",
  "event_time": "2026-08-12T01:45:10+08:00",
  "system": "application",
  "component": "apm",
  "severity": "high",
  "category": "availability",
  "title": "Database calls timed out",
  "message": "Requests exceeded the configured deadline",
  "entity_id": "asset:application:mes-api",
  "state_code": 8040,
  "reason_code": "APM_TIMEOUT",
  "source_ref": {
    "artifact_id": "artifact-01",
    "kind": "trace_span",
    "trace_id": "sanitized-trace-id",
    "span_id": "sanitized-span-id"
  }
}
```

支持 JSON 文档和 JSON Lines。集合字段使用 `events`、`records`、`timeline`、`evidence` 或 `data`，扩展字段放入 `attributes`。

## 6. SQL 与查询结果

为了避免“只有结论，没有执行依据”，数据库证据必须区分查询文本、执行状态和返回结果：

```json
{
  "evidence_type": "database_query",
  "query_id": "sha256:...",
  "query_text": "SELECT ...",
  "executed": true,
  "started_at": "2026-08-12T01:44:00+08:00",
  "duration_ms": 124,
  "row_count": 3,
  "columns": ["event", "waits"],
  "result_digest": "sha256:...",
  "result_preview": [],
  "truncated": false,
  "error": null
}
```

规则：

- 未执行的 SQL 不能显示为执行成功；
- 连接失败时任务状态必须为失败，不能生成健康结论；
- 报告必须展示采集 SQL、执行状态、返回行数、字段和结果摘要；
- 结果过大时保留摘要、截断标记和原始 Artifact 引用；
- 包含业务数据时默认不导出明细，只导出结构和摘要。

## 7. 状态记录

Agent 或 Collector 可以提交由确定性规则识别的 `state_code`。分析引擎必须：

1. 校验状态码是否存在于版本化字典；
2. 保留采集端规则版本和证据引用；
3. 不接受只有 AI 文本、没有证据的状态；
4. 对同一实体的状态按时间排序并执行幂等去重；
5. 将采集端状态与本地解析得到的状态放入同一时间线；
6. 出现冲突时保留两侧证据并降低置信度，不静默覆盖。

## 8. 兼容与确定性

- canonical event、state 和 bundle manifest 分别版本化；
- 主版本变化表示不兼容变更；
- 未识别字段保存在 `attributes`，不能丢弃；
- 未识别状态码标记为 `unknown`，不猜测含义；
- 导入报告记录 Agent、Collector、DBSleuth 和解析器版本；
- 同一输入哈希、规则版本和参数必须得到相同确定性结果；
- 重复上传或导入使用稳定幂等键，不能制造重复事件。

## 9. 安全流程

```text
事故触发或人工选择时间窗
  -> Agent / Collector 冻结现场
  -> 扫描 IP、主机名、用户、路径、连接串和 Token
  -> 根据策略脱敏并生成只读 Bundle
  -> 计算 Manifest 与 SHA-256
  -> DBSleuth 安全清点和完整性校验
  -> 本地或服务端分析
```

分析引擎默认不使用 Bundle 中的地址回连主机或数据库，也不执行日志、SQL、脚本或附件中的命令。需要补采时必须生成独立、可审批、限时的采集任务。

## 10. 验收

- Bundle 不包含凭据、私钥或控制面数据库；
- DBSleuth 能识别事件、状态、Artifact 和证据引用；
- 所有状态和结论都能回到事件或附件；
- SQL 证据明确区分已执行、跳过和失败；
- 重复导入结果幂等；
- 不支持的字段、状态和附件明确报告；
- 部分 Snapshot 不生成“现场完整”的结论；
- 原始日志目录和压缩包无需 Agent 也能独立分析。

## 11. 配套案例

[从存储异常到 Oracle 实例故障的图文案例 Demo](CASE_DEMO_STORAGE_INCIDENT.md) 展示 DBSleuth 如何冻结或导入 Incident Bundle，完成安全清点、事件标准化、状态识别、故障模式匹配、证据图和受约束 AI 解释。
