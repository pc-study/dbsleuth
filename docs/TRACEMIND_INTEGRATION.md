# TraceMind 与 DBSleuth 集成设计

| 属性 | 内容 |
|---|---|
| 版本 | v0.1 Draft |
| 状态 | 接口提案，待实现 |

## 1. 定位

DBSleuth 是本地优先、基于证据的故障分析引擎；TraceMind 是包含资产、巡检、监控、AWR、APM、快照、诊断和处置闭环的平台。

两者不互相替代：

```text
TraceMind
  -> 采集现场、指标、SQL/AWR、告警、拓扑和调用链
  -> 导出标准化 Incident Bundle

DBSleuth
  -> 离线解析日志和 Incident Bundle
  -> 生成事件、状态、证据索引和可交付事故报告
```

集成目标是复用 TraceMind 已验证的时间线、证据链和状态智能思想，不把平台账号、数据库凭据、业务数据库或私有配置复制到开源项目。

## 2. 能力映射

| TraceMind 能力 | DBSleuth 接收内容 | DBSleuth 输出 |
|---|---|---|
| Incident Snapshot | 冻结窗口内的标准事件与附件清单 | 离线时间线与证据索引 |
| Zabbix / Prometheus | 已触发告警、指标名、实际值、阈值 | 监控状态和跨层时间关联 |
| AWR / SQL 分析 | 等待事件、SQL 指纹、采集 SQL 和结果摘要 | 数据库状态与可审计证据 |
| APM | Trace/Span、服务、端点、SQL 指纹关联 | 应用到数据库影响链 |
| Host Profile | CPU、内存、磁盘、网卡和虚拟化事实 | 主机状态上下文 |
| CMDB / 资产中心 | 脱敏后的稳定资产 ID 和关系 | 拓扑匹配与置信度提升 |
| Diagnosis | 规则候选、支持证据和反证 | 可复核 RCA 候选报告 |

## 3. 边界

### 3.1 可以进入 DBSleuth

- 已脱敏的标准事件；
- 指标窗口的汇总值和必要样本；
- 状态编号、转换原因和规则版本；
- 告警、AWR、APM 和日志证据引用；
- 匿名化的资产关系；
- 用户明确选择导出的日志和附件。

### 3.2 不允许进入 DBSleuth 仓库或默认导出包

- `.env`、数据库连接配置和密码；
- Access Token、Cookie、私钥和客户端证书；
- TraceMind 平台数据库文件；
- 未经人工复核的生产日志；
- 完整进程环境变量；
- 包含业务数据的 SQL 结果；
- 用户、授权和许可证数据库。

## 4. Incident Bundle

建议结构：

```text
incident-bundle/
  manifest.json
  events/
    timeline.jsonl
    alerts.jsonl
    database.jsonl
    application.jsonl
  metrics/
    key-series.jsonl.gz
  states/
    observations.jsonl
    transitions.jsonl
  evidence/
    evidence-index.json
  attachments/
    selected-redacted-logs/
```

`manifest.json` 记录 Schema 版本、时间范围、时区、时钟质量、文件 SHA-256、脱敏状态和来源产品版本。

## 5. 最小事件记录

```json
{
  "event_time": "2026-08-12T01:45:10+08:00",
  "system": "application",
  "component": "apm",
  "severity": "high",
  "category": "availability",
  "title": "MES database calls timed out",
  "message": "Requests exceeded the configured deadline",
  "entity_id": "asset:application:mes-api",
  "state_code": 8040,
  "reason_code": "APM_TIMEOUT",
  "source_ref": {
    "kind": "trace_span",
    "trace_id": "sanitized-trace-id",
    "span_id": "sanitized-span-id"
  }
}
```

支持 JSON 文档和 JSON Lines。集合字段建议使用 `events`、`records`、`timeline`、`evidence` 或 `data`。

## 6. SQL 与查询结果

为了避免“只有结论，没有执行依据”，数据库证据必须区分：

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
- 结果过大时保留摘要、截断标记和原始附件引用；
- 包含业务数据时默认不导出明细，只导出结构和摘要。

## 7. 状态集成

TraceMind 可以发送已经由平台确定性规则识别的 `state_code`。DBSleuth 必须：

1. 校验状态码是否存在于版本化字典；
2. 保留上游规则版本和证据引用；
3. 不接受只有 AI 文本、没有证据的状态；
4. 对同一实体的状态按时间排序并执行幂等去重；
5. 将上游状态和本地日志识别状态放入同一时间线；
6. 出现冲突时保留两侧证据并降低置信度，不静默覆盖。

## 8. 兼容与版本

- canonical event、state、bundle manifest 分别版本化；
- 主版本变化表示不兼容变更；
- 未识别字段保存在 `attributes`，不能丢弃；
- 未识别状态码标记为 unknown，不猜测含义；
- 导入报告记录 TraceMind 版本、DBSleuth 版本和解析器版本；
- 同一输入 SHA-256、规则版本和参数必须得到相同确定性结果。

## 9. 安全流程

```text
TraceMind 选择事故
  -> 生成导出预览
  -> 扫描 IP / 主机名 / 用户 / 路径 / 连接串 / Token
  -> 用户确认脱敏规则
  -> 生成只读 Incident Bundle
  -> DBSleuth 安全清点和完整性校验
  -> 本地分析
```

DBSleuth 不回连 TraceMind，不使用导出包中的地址尝试连接数据库，也不执行日志或附件中的命令。

## 10. 验收

- TraceMind 导出包不包含凭据、配置数据库或私钥；
- DBSleuth 能识别 TraceMind 事件、状态和上游证据引用；
- 所有状态和结论都能回到事件或附件；
- SQL 证据明确区分已执行、跳过和失败；
- 重复导入结果幂等；
- 不支持的字段、状态和附件明确报告；
- Oracle/Linux 原始日志分析不依赖 TraceMind，也能独立运行。

## 11. 配套案例

[从存储异常到 Oracle 实例故障的图文案例 Demo](CASE_DEMO_STORAGE_INCIDENT.md) 展示 TraceMind 导出 Incident Bundle 后，DBSleuth 如何完成安全清点、事件标准化、状态识别、故障模式匹配、证据图和受约束 AI 解释。
