# DBSleuth（库迹）

[![License](https://img.shields.io/badge/license-Apache--2.0-blue.svg)](LICENSE)
[![Status](https://img.shields.io/badge/status-project%20initiation-orange.svg)](ROADMAP.md)

> Evidence-driven database incident analysis.

**库有迹，障可循。**

DBSleuth imports database and operating-system logs, normalizes timestamps, groups repeated messages, correlates related events, and produces an incident report in which every conclusion links back to original evidence.

## Product promise

Given an incident bundle, answer four questions:

1. What important events occurred?
2. What was the earliest abnormal event?
3. What happened at the operating-system layer around the database failure?
4. Which source file and line support every conclusion?

## MVP scope

- Oracle `alert.log`
- Linux syslog/messages and exported `journalctl` text
- UTF-8 and GBK input
- `.log`, `.txt`, `.gz`, and `.zip`
- Timestamp normalization and explicit timezone handling
- Multi-line event reconstruction
- Deterministic event classification and duplicate grouping
- Evidence index with source file and line range
- HTML, Markdown, and JSON output
- Local-only redaction preview

## Explicitly out of scope for v0.1

- Live collection or agents
- Database credentials or direct database connections
- Long-term log storage
- Alerting and notification
- Automatic remediation
- Generic observability platform features
- Untraceable AI root-cause claims
- MySQL, PostgreSQL, Windows, ASM, CRS, and Listener logs

## Proposed CLI

```bash
dbsleuth inspect incident.zip
dbsleuth analyze incident.zip --timezone Asia/Shanghai
dbsleuth events incident.zip --severity high
dbsleuth redact incident.zip --preview
```

## Success gate

Proceed beyond MVP only if a blinded evaluation over at least 30 anonymized real-world bundles achieves:

- at least 95% timestamp extraction on supported log lines;
- at least 90% precision for critical/high event classification;
- no fabricated events or evidence locations;
- every reported event links to an exact source span;
- at least 10 DBA users complete a report without maintainer assistance;
- at least 5 users reuse the tool within 30 days.

See [PROJECT_CHARTER.md](PROJECT_CHARTER.md), [ARCHITECTURE.md](ARCHITECTURE.md), [ROADMAP.md](ROADMAP.md), and [BACKLOG.md](BACKLOG.md).
