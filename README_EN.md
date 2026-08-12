# DBSleuth

[![License](https://img.shields.io/badge/license-Apache--2.0-blue.svg)](LICENSE)
[![Status](https://img.shields.io/badge/status-project%20initiation-orange.svg)](ROADMAP.md)

[简体中文](README.md) | [English](README_EN.md)

> Evidence-driven database incident analysis.

DBSleuth imports database and operating-system logs, normalizes timestamps, groups repeated messages, correlates related events, and produces an incident report in which every conclusion links back to original evidence.

The planned State Intelligence Engine projects cited events into versioned state observations, transitions, and failure patterns. States compress the incident timeline but never replace the original evidence.

The planned Linux eBPF observation layer captures bounded scheduler, process, Block IO, TCP/socket, memory-pressure, and lock events. It identifies the affected thread, process, cgroup, device, or connection and triggers targeted snapshots instead of indiscriminate full-host dumps. Unsupported kernels fall back safely, and event loss is reported as incomplete evidence rather than interpreted as health.

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
dbsleuth states incident.zip --timezone Asia/Shanghai
dbsleuth redact incident.zip --preview
```

> The project is currently in its initiation and feasibility-validation phase. These commands have not been released yet.

## Success gate

Proceed beyond MVP only if a blinded evaluation over at least 30 anonymized real-world bundles achieves:

- at least 95% timestamp extraction on supported log lines;
- at least 90% precision for critical/high event classification;
- no fabricated events or evidence locations;
- every reported event links to an exact source span;
- at least 10 DBA users complete a report without maintainer assistance;
- at least 5 users reuse the tool within 30 days.

## Design documents

- [Project charter](PROJECT_CHARTER.md)
- [Architecture](ARCHITECTURE.md)
- [Roadmap](ROADMAP.md)
- [Backlog](BACKLOG.md)
- [State Intelligence Engine](docs/STATE_ENGINE.md)
- [DBSleuth Incident Bundle](docs/DBSLEUTH_INCIDENT_BUNDLE.md)
- [eBPF kernel observability](docs/EBPF_OBSERVABILITY.md)
- [Illustrated storage-to-Oracle case Demo](docs/CASE_DEMO_STORAGE_INCIDENT.md)

## License

DBSleuth is released under the [Apache License 2.0](LICENSE).
