# Initial GitHub Backlog

## Epic A — Specification and fixtures

- [ ] Define canonical event JSON Schema v1.
- [ ] Define severity, category, confidence, and timezone semantics.
- [ ] Publish safe anonymization and fixture contribution guide.
- [ ] Create fixture manifest format.
- [ ] Add synthetic Oracle startup/shutdown fixture.
- [ ] Add synthetic ORA multi-line failure fixture.
- [ ] Add Linux OOM and I/O error fixtures.
- [ ] Add GBK and malformed-byte fixtures.

## Epic B — Input safety

- [ ] Inventory files without modifying source input.
- [ ] Support `.log`, `.txt`, `.gz`, and `.zip`.
- [ ] Reject Zip Slip/path-traversal entries.
- [ ] Add expanded-size, file-count, nesting, and ratio limits.
- [ ] Detect binary and unsupported files explicitly.
- [ ] Implement streaming reads and bounded buffers.

## Epic C — Parsing

- [ ] Detect Oracle alert text format.
- [ ] Parse Oracle timestamp variants across supported releases.
- [ ] Reconstruct Oracle multi-line events.
- [ ] Extract ORA codes without treating SQL text as incidents.
- [ ] Detect Linux syslog/messages format.
- [ ] Parse exported journalctl text.
- [ ] Preserve unknown messages and source spans.
- [ ] Emit parser diagnostics and coverage statistics.

## Epic D — Analysis

- [ ] Normalize timestamps to UTC while preserving originals.
- [ ] Represent missing/ambiguous timezone explicitly.
- [ ] Create deterministic event fingerprints.
- [ ] Group repeated events without losing counts or time range.
- [ ] Identify earliest abnormal event.
- [ ] Build configurable context windows around critical events.
- [ ] Correlate Oracle and Linux events using cautious temporal language.

## Epic E — Reports and redaction

- [ ] Render versioned JSON output.
- [ ] Render Markdown incident report.
- [ ] Render self-contained, escaped HTML report.
- [ ] Add source-file and line-range evidence anchors.
- [ ] Detect candidate IPs, hostnames, database names, usernames, paths, emails, and connection strings.
- [ ] Add redaction preview and allow/deny rules.
- [ ] Create sanitized bundle without changing originals.

## Epic F — Quality and release

- [ ] Add golden tests for every accepted fixture.
- [ ] Add fuzzing for parsers and archives.
- [ ] Add Windows/Linux/macOS CI.
- [ ] Benchmark 100 MB and 1 GB bundles.
- [ ] Generate checksums and SBOM for releases.
- [ ] Write bilingual quick start.
- [ ] Publish v0.1.0 release and sample report.

## First five implementation issues

1. `spec: define canonical event and evidence schema`
2. `security: implement bounded safe archive inventory`
3. `parser: parse Oracle alert timestamps and multi-line records`
4. `parser: parse Linux syslog/messages records`
5. `report: render JSON and evidence-indexed Markdown`
