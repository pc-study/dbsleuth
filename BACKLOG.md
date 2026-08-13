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
- [ ] Render self-contained HTML report using `html/template` ONLY (context auto-escaping). Assert via grep/build step that `text/template` is never imported by any HTML rendering path.
- [ ] Escape ALL fields sourced from the input bundle in HTML output (not just "log content"): `source.path`, `entity_id`, `hostname`, `summary`, `raw`, `codes`, `attributes`, SQL `query_text`.
- [ ] Add source-file and line-range evidence anchors.
- [ ] Implement the **mandatory redaction set** (Tier 1) per [docs/REDACTION_POLICY.md](docs/REDACTION_POLICY.md): Oracle connect strings, TNS PASSWORD, password/secret/token/api-key keyword assignments, AWS keys, private key blocks, JWTs, GitHub tokens, RFC1918 IPs. These are always redacted; not configurable off; only bypassable via a reviewed, committed allow rule.
- [ ] Implement the **candidate set** (Tier 2, opt-in): hostnames, database names, usernames, paths, emails, MAC, non-Oracle connection strings. Detect + preview + user confirm.
- [ ] Add redaction preview and allow/deny rules. Allow rules persisted to a versioned allowlist and recorded in the report's `redaction_summary`.
- [ ] Fail-safe: if a file cannot be scanned (encoding/parse), it MUST NOT be exported in plaintext — drop with `DBSLEUTH_E_REDACT_SCAN_FAILED` or abort.
- [ ] Add a regression test asserting every Tier 1 fixture sample is masked in JSON/Markdown/HTML output (zero raw matches).
- [ ] Create sanitized bundle without changing originals.

## Epic B — Safety and archive handling (revised)

- [ ] Implement configurable safety limits with the defaults from ARCHITECTURE.md: max expanded size 1 GiB, single file 256 MiB, file count 10,000, nesting depth 10, compression ratio 100:1.
- [ ] On any limit hit: stop, emit `PARTIAL` (exit code 2), emit `DBSLEUTH_E_LIMIT_*` on stderr, never silently continue.
- [ ] After extraction, recompute SHA-256 for every extracted file and require exact match with manifest; mismatch → exit code 5 (`INTEGRITY_FAILURE`).
- [ ] Add a regression test asserting no execution path consumes parsed input content (no `os/exec` over parsed data, SQL never handed to `database/sql`).

## Epic F — Quality and release

- [ ] Add golden tests for every accepted fixture.
- [ ] Add fuzzing for parsers and archives.
- [ ] Add Windows/Linux/macOS CI.
- [ ] Benchmark 100 MB and 1 GB bundles.
- [ ] Generate checksums and SBOM for releases.
- [ ] Write bilingual quick start.
- [ ] Publish v0.1.0 release and sample report.

## Post-MVP Epic G — State Intelligence Engine

- [ ] Publish the versioned state dictionary and lifecycle policy.
- [ ] Define state observation, transition, reason, and pattern schemas.
- [ ] Require every state to reference canonical event evidence.
- [ ] Implement `unknown`, inferred-baseline, and data-quality semantics.
- [ ] Add enter/exit hysteresis, duration thresholds, cooldowns, and deduplication.
- [ ] Implement deterministic state replay and idempotent identifiers.
- [ ] Add host, memory, storage, database, and application MVP states.
- [ ] Implement storage-to-database and database-to-application patterns.
- [ ] Preserve supporting evidence, contradictions, topology checks, and limitations.
- [ ] Add golden tests for out-of-order, duplicate, missing, and flapping inputs.

## Post-MVP Epic H — Optional DBSleuth Incident Bundle

- [ ] Publish the sanitized Incident Bundle manifest and compatibility rules.
- [ ] Import versioned timeline, alert, metric-window, topology, and state records.
- [ ] Validate upstream state codes, rule versions, evidence references, and hashes.
- [ ] Preserve conflicts between upstream and locally derived states.
- [ ] Represent SQL evidence with query text, execution status, fields, row count, and result digest.
- [ ] Reject credentials, tokens, private keys, cookies, and application databases.
- [ ] Add import diagnostics for unsupported fields, states, and attachments.
- [ ] Build the storage-to-Oracle illustrated Demo as an executable golden fixture.

## First five implementation issues

1. `spec: define canonical event and evidence schema`
2. `security: implement bounded safe archive inventory`
3. `parser: parse Oracle alert timestamps and multi-line records`
4. `parser: parse Linux syslog/messages records`
5. `report: render JSON and evidence-indexed Markdown`

## Epic I — eBPF kernel observability

- [ ] Implement kernel, BTF, Program/Map/Helper, permission, and fallback capability probing.
- [ ] Build a minimal privileged loader with signed objects, hashes, leases, budgets, and Kill Switch.
- [ ] Define the versioned Kernel Event Header with dual clocks, sequence, identity, and correlation anchors.
- [ ] Report Map usage, Ring Buffer watermarks, event loss, consumer lag, and probe overhead.
- [ ] Implement scheduler, Run Queue, On/Off-CPU, and thread-lifecycle sensors.
- [ ] Implement Block IO queue/service latency, device, process/thread, and error sensors.
- [ ] Implement TCP state, retransmit, reset, Socket Cookie, and process/thread correlation sensors.
- [ ] Implement page-fault, reclaim, allocation-failure, OOM, and OOM Kill sensors.
- [ ] Implement process fork/exec/exit, Build ID, cgroup, and Namespace sensors.
- [ ] Add targeted Futex/lock, syscall latency, and uprobe/USDT probes.
- [ ] Build offline symbol resolution with explicit complete/partial/missing status.
- [ ] Map eBPF events to deterministic states and targeted Snapshot/Dump triggers.
- [ ] Test automatic fallback on unsupported kernels without failing the base Agent.

## Design documents

- [State Intelligence Engine](docs/STATE_ENGINE.md)
- [DBSleuth Incident Bundle](docs/DBSLEUTH_INCIDENT_BUNDLE.md)
- [eBPF kernel observability](docs/EBPF_OBSERVABILITY.md)
- [Storage-to-Oracle case Demo](docs/CASE_DEMO_STORAGE_INCIDENT.md)
