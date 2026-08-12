# DBSleuth

[简体中文](README.md) | [English](README_EN.md)

> Local-first, evidence-driven database incident analysis CLI.

DBSleuth aims to import Oracle and Linux logs, reconstruct incident timelines, group repeated events, correlate cross-layer signals, and link every observation to exact source evidence.

> **Status:** initiation and feasibility validation. No usable release exists yet. Commands shown in the documentation describe the intended experience.

## Current MVP

Planned scope:

- Oracle `alert.log`;
- Linux syslog/messages and exported `journalctl` text;
- UTF-8 and GBK input;
- `.log`, `.txt`, `.gz`, and `.zip`;
- timestamp normalization and multi-line event reconstruction;
- deterministic classification and duplicate grouping;
- exact source-span evidence links;
- local HTML, Markdown, and JSON reports;
- redaction candidate preview.

Explicitly outside the MVP:

- resident agents, live collection, and dynamic probes;
- credentials or direct production database connections;
- trace, dump, memory, or thread-snapshot collection;
- long-term storage, alerting, notification, or remediation;
- multi-tenancy, control planes, HA, and disaster recovery;
- unsupported AI root-cause claims;
- additional databases and operating systems.

## Intended CLI

```bash
dbsleuth inspect incident.zip
dbsleuth analyze incident.zip --timezone Asia/Shanghai
dbsleuth events incident.zip --severity high
dbsleuth redact incident.zip --preview
```

## Evidence contract

- deterministic parsing precedes probabilistic inference;
- missing facts and causal links are never fabricated;
- temporal proximity is correlation, not causation;
- unsupported input is reported as unknown;
- rules require anonymized fixtures and regression tests;
- events retain source spans, parser versions, and rule versions.

## Long-term vision

Post-MVP research topics—not current product commitments—include an evidence-backed State Intelligence Engine, evidence graphs, constrained AI explanations, additional database parsers, optional Incident Bundle adapters, and possible online or enterprise components.

TraceMind is only an [optional Post-MVP adapter proposal](docs/TRACEMIND_INTEGRATION.md); DBSleuth must remain independently useful. The [storage-to-Oracle walkthrough](docs/CASE_DEMO_STORAGE_INCIDENT.md) is a synthetic design case, not an implemented demo.

## Documentation

- [Project charter](PROJECT_CHARTER.md)
- [Architecture](ARCHITECTURE.md)
- [Roadmap](ROADMAP.md)
- [Initial backlog](BACKLOG.md)
- [Post-MVP state engine draft](docs/STATE_ENGINE.md)

## License

DBSleuth is released under the [Apache License 2.0](LICENSE).
