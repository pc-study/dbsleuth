# Roadmap

## Phase 0 — Feasibility and corpus (Weeks 1–2)

Objective: prove that representative logs can be parsed safely and consistently.

- Define event schema and evidence model.
- Freeze the initial state dictionary, observation/transition schema, and evidence invariants.
- Define the sanitized DBSleuth Incident Bundle contract without control-plane or database credentials.
- Collect at least 30 anonymized samples across Oracle 11g, 12c, 19c, and 21c where available.
- Include Linux syslog/messages and journalctl exports.
- Document anonymization procedure and prohibited data.
- Build a fixture manifest with encoding, timezone, source type, and expected key events.
- Implement a parser spike that emits JSON Lines.

Exit gate: exact source spans and timestamps can be recovered for the major events in at least 80% of the initial samples, and every proposed state maps to explicit event evidence.

## Phase 1 — v0.1 parser MVP (Weeks 3–6)

- Safe input/archive inventory.
- Encoding and format detection.
- Oracle alert text parser.
- Linux syslog/messages parser.
- Exported journalctl text parser.
- Multi-line reconstruction.
- ORA code and Linux critical-event extraction.
- Canonical JSON output.
- Deterministic state recognition for the first supported Oracle and Linux conditions.
- Fixture-driven tests and parser fuzz tests.

Exit gate: no fabricated events or states; critical/high classification and key-state precision are at least 90% on the held-out fixture set.

## Phase 2 — v0.2 useful report (Weeks 7–9)

- Duplicate grouping and normalized fingerprints.
- Critical-event context windows.
- First-abnormal-event detection.
- State observation aggregation, transitions, anti-flapping, and replay.
- Initial storage-to-database and memory-to-failure patterns with contradiction checks.
- Markdown and self-contained HTML reports.
- Evidence anchors linking every event to source spans.
- Basic redaction candidate preview.
- Windows, Linux, and macOS binaries.

Exit gate: 10 DBA testers can produce and understand a report without maintainer assistance.

## Phase 3 — v0.3 field validation (Weeks 10–12)

- Analyze at least 20 real incident bundles.
- Improve timezone and restart-boundary handling.
- Add performance benchmarks for 100 MB and 1 GB inputs.
- Publish example bundles and reports.
- Validate the illustrated storage-to-Oracle Demo against golden fixtures.
- Validate sanitized DBSleuth Incident Bundle import and version diagnostics.
- Add contribution guide for parsers, rules, and fixtures.
- Package via GitHub Releases, Homebrew, Scoop, and container image where useful.

Exit gate: at least 5 testers reuse the tool and at least one external fixture/rule contribution is accepted.

## Post-MVP candidates

Prioritize only from real issue demand:

1. Oracle Listener log.
2. ASM alert log.
3. CRS alert log.
4. Data Guard broker and redo transport.
5. MySQL error log.
6. PostgreSQL log.
7. Windows Event Log export.
8. Optional lnav format contributions/integration.
9. Optional local LLM explanation over already-structured, cited events.
10. Additional host, process, thread, network, storage, database, and application state families.
