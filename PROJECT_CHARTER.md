# DBSleuth（库迹）Project Charter

## 1. Problem

Database incidents commonly involve several independent logs with different timestamps, formats, encodings, and levels of detail. DBAs spend significant time searching, aligning, deduplicating, and copying fragments into incident reports. Generic log viewers can merge and search logs, but they do not provide an Oracle-aware, evidence-backed incident narrative suitable for handoff or review.

## 2. Target users

Primary users:

- Oracle DBAs handling incidents and postmortems;
- database support engineers receiving customer log bundles;
- infrastructure engineers correlating Linux and Oracle events;
- consultants preparing diagnostic evidence for customers.

Secondary users:

- application support teams;
- training and lab instructors;
- vendors receiving sanitized support bundles.

## 3. Value proposition

DBSleuth turns a folder or archive of Oracle and Linux logs into a chronological, deduplicated incident report. It runs locally, requires no credentials, and never presents a conclusion without traceable source evidence.

## 4. Principles

1. Evidence before explanation.
2. Deterministic parsing before probabilistic inference.
3. Never invent missing timestamps, fields, or causal relationships.
4. Preserve original logs unchanged.
5. Local-only by default; no telemetry in the initial releases.
6. Redaction must be previewable and reversible against preserved originals.
7. Unsupported input is reported explicitly, not silently misparsed.
8. A correlation is not presented as causation.

## 5. Core user journey

```text
Collect/export logs
    -> create incident folder or ZIP
-> dbsleuth inspect
    -> correct timezone/source hints if required
-> dbsleuth analyze
    -> review evidence and redaction candidates
    -> export/share report and sanitized bundle
```

## 6. MVP deliverables

- Cross-platform CLI executable.
- Oracle alert-log parser.
- Linux syslog/messages/journalctl-text parser.
- Canonical event schema.
- Timestamp/timezone normalization.
- Multi-line reconstruction.
- Severity and category rule engine.
- Duplicate-event grouping.
- Cross-source temporal context windows.
- Evidence-backed Markdown, HTML, and JSON reports.
- Redaction candidate detector with preview.
- An anonymized fixture corpus and regression tests.
- English and Chinese documentation.

## 7. Non-goals

DBSleuth is not a replacement for Loki, Elasticsearch, Splunk, lnav, OEM, or database monitoring. It does not collect live telemetry, execute SQL, manage credentials, or repair production systems.

## 8. Proposed license and governance

- License: Apache-2.0.
- Public roadmap and issue tracker.
- Conventional commits and semantic versioning.
- Parser/rule changes require fixtures and expected-event tests.
- Security reports use a private disclosure channel.
- Real logs are accepted only after contributor-confirmed anonymization.

## 9. Key metrics

- Supported lines parsed vs. preserved as unknown.
- Timestamp extraction accuracy.
- Event classification precision and recall.
- Evidence-link correctness.
- Report completion time for a 100 MB bundle.
- Peak memory consumption.
- First successful report rate.
- 30-day repeat usage.
- External fixtures, rules, issues, and pull requests.

## 10. Stop/pivot conditions

Pause or narrow the project if:

- real logs cannot be obtained safely for validation;
- supported formats produce less than 90% critical-event precision after two parser iterations;
- useful reports require database access or proprietary collectors;
- users mainly request generic search/viewing already served by lnav;
- fewer than 5 of the first 20 DBA testers would use it again.
