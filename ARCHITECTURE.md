# MVP Architecture

## Recommended stack

- Language: Go
- CLI: Cobra or standard `flag` package
- Templates: Go `html/template` and `text/template`
- Internal interchange: versioned JSON schema
- Packaging: GoReleaser
- CI: GitHub Actions on Windows, Linux, and macOS
- Tests: fixture-driven golden tests plus fuzzing for parsers and archive handling

Go is recommended for single-file distribution, streaming file processing, predictable deployment, and straightforward cross-platform releases.

## Processing pipeline

```text
Input files/archive
  -> safe archive extraction
  -> file inventory and type/encoding detection
  -> source-specific streaming parsers
  -> canonical events
  -> timestamp/timezone normalization
  -> severity/category rules
  -> fingerprinting and duplicate grouping
  -> bounded temporal correlation
  -> redaction candidate scan
  -> evidence index
  -> JSON/Markdown/HTML renderers
```

## Canonical event schema

```json
{
  "schema_version": "1.0",
  "event_id": "sha256:...",
  "source": {
    "path": "alert_ORCL.log",
    "parser": "oracle-alert-text",
    "parser_version": "0.1.0",
    "line_start": 1204,
    "line_end": 1208
  },
  "time": {
    "original": "2026-08-12T01:44:21.123+08:00",
    "normalized_utc": "2026-08-11T17:44:21.123Z",
    "timezone_source": "embedded",
    "confidence": 1.0
  },
  "system": "oracle",
  "component": "rdbms",
  "severity": "high",
  "category": "instance/storage",
  "codes": ["ORA-00240"],
  "summary": "Control file enqueue held for an extended period",
  "raw": "...",
  "fingerprint": "...",
  "attributes": {},
  "rule_ids": ["ORA-00240"],
  "confidence": 0.98
}
```

## Safety requirements

- Reject archive entries with absolute paths or path traversal.
- Enforce configurable limits on expanded size, file count, nesting, and compression ratio.
- Stream large files instead of reading bundles fully into memory.
- Never modify the source bundle.
- Escape all log content in HTML output.
- Do not execute content found in logs.
- Mark guessed encodings and user-supplied timezones in the report.
- Preserve unknown lines around detected events as evidence context.
- Redact exported copies, not original inputs.

## Correlation boundary

MVP correlation is temporal and rule-based:

- show events in a configurable window around a critical event;
- identify ordering and time distance;
- group events sharing codes, normalized templates, identifiers, or components;
- use language such as “preceded”, “followed”, and “co-occurred”.

The MVP must not claim “root cause” unless a deterministic rule has explicit, documented preconditions. Even then, output should say “root-cause candidate” with confidence and evidence.

## Repository layout

```text
cmd/dbsleuth/            CLI entry point
internal/archive/        safe archive inspection
internal/detect/         type and encoding detection
internal/event/          canonical schema
internal/parser/oracle/  Oracle alert parser
internal/parser/linux/   syslog/journal text parser
internal/rules/          deterministic classifications
internal/correlate/      grouping and time-window logic
internal/redact/         candidate detection and transforms
internal/report/         JSON, Markdown, HTML rendering
fixtures/                anonymized regression corpus
schemas/                 published JSON schemas
docs/                    user and contributor documentation
```
