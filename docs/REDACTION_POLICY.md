# Redaction Policy

> Status: normative spec. This document defines the **mandatory redaction set** (always redacted, even if the user skips preview) and the **candidate set** (opt-in, user confirms), plus detection patterns.

The original design only had an opt-in "candidate scan + user preview" flow. For a tool that ingests Oracle alert logs, TNS descriptors, and connection bundles, that is not enough — alert logs occasionally carry plaintext passwords, TNS entries carry `PASSWORD=...`, and connection strings look like `SCOTT/tiger@orcl`. If a user skips preview or a candidate pattern misses, exported reports leak plaintext credentials.

This policy closes that gap with a two-tier model.

## Tier 1 — Mandatory redaction set (always applied, non-bypassable)

These patterns are redacted in EVERY exported copy (JSON / Markdown / HTML / redacted log bundle), regardless of whether the user runs `--redact-preview` or not. They cannot be turned off by config; the only way to keep them is an explicit per-match `allow` rule reviewed by the user.

| ID | Pattern (illustrative) | Replaces with | Rationale |
|---|---|---|---|
| `PWD_ORACLE_CONNECT` | `(\w+)\/(\S+?)@` (Oracle connect string `user/pass@service`) | `<user>/***@<service>` (keep user+service, mask password) | Classic `SCOTT/tiger@orcl` |
| `PWD_TNS_FIELD` | `PASSWORD\s*=\s*\S+` (TNS descriptor) | `PASSWORD=<redacted>` | TNS plaintext password |
| `PWD_KEYWORD_ASSIGN` | `(?i)(password|passwd|pwd|passphrase)\s*[:=]\s*(\S+)` | `<key>=<redacted>` | Generic `password: secret123` |
| `SECRET_KEYWORD_ASSIGN` | `(?i)(secret|api[_-]?key|access[_-]?key|secret[_-]?key|token|credential|client[_-]?secret)\s*[:=]\s*\S+` | `<key>=<redacted>` | Secrets / API keys / tokens |
| `AWS_AKID` | `AKIA[0-9A-Z]{16}` | `<aws_akid>` | AWS access key id |
| `AWS_SECRET` | (heuristic: 40-char base64 near `aws`/`secret`) | `<aws_secret>` | AWS secret key |
| `PRIVATE_KEY_BLOCK` | `-----BEGIN (.+?) PRIVATE KEY-----` ... `-----END ...` | `<private_key block>` | PEM private keys (RSA/EC/OpenSSH) |
| `JWT` | `eyJ[A-Za-z0-9_-]{8,}\.[A-Za-z0-9_-]{8,}\.[A-Za-z0-9_-]{8,}` | `<jwt>` | JWT tokens |
| `GITHUB_PAT` | `(gh[pousr]_[A-Za-z0-9]{36,})\|(github_pat_[A-Za-z0-9_]{82})` | `<github_token>` | GitHub tokens |
| `PRIVATE_IPV4` | RFC1918 ranges `10./172.16-31./192.168.` | `<ip>` | Internal IPs (configurable, see Tier 2 note) |

Tier 1 patterns MUST be applied as the final step before any renderer writes output. A regression test must assert that a fixture containing each Tier 1 sample produces zero raw matches in every output format.

## Tier 2 — Candidate set (opt-in, user confirms)

These are redacted only when the user runs `--redact-preview` and confirms them, OR sets `--redact-candidates=auto`. Useful but ambiguous — they risk over-redacting (e.g. a hostname that is actually a public doc URL).

| Candidate | Notes |
|---|---|
| hostnames | FQDNs, distinguish `localhost` (always kept) |
| database names / PDB / tablespace names | Often sensitive but sometimes needed for the report to make sense |
| usernames / schemas | `SYS`, `SYSTEM`, `SCOTT` etc. |
| file paths | May contain hostnames or usernames |
| emails | Per RFC |
| MAC addresses | |
| connection strings (non-Oracle) | JDBC URLs, MySQL DSN |

## Allow rules

A user may add an explicit allow rule to keep a Tier 1 match (e.g. a known test password `tiger` in a fixture). Allow rules are:

- Reviewed one-by-one in `--redact-preview` (shown with surrounding context).
- Persisted to a versioned allowlist file that is itself committed (no silent allow).
- Logged in the report's `redaction_summary` so a reviewer knows what was kept and why.

## Fail-safe

If the redaction scanner cannot complete (e.g. a file fails encoding detection and cannot be scanned), that file's content MUST NOT be exported in plaintext — it is either dropped from the export with a `DBSLEUTH_E_REDACT_SCAN_FAILED` note, or the export is aborted. Never export unscanned content on the assumption "probably fine".

## Detection vs masking

- Detection is best-effort; false negatives are the risk. The mandatory set minimizes the common credential patterns.
- When a Tier 1 pattern matches, it is **masked** (replaced with a placeholder), never deleted — surrounding evidence context (line numbers, timestamps) is preserved. This keeps the report useful while removing the secret.
- Masked values are recorded in a per-report `redaction_summary` (counts per pattern id), so the user knows how many of each were found.

## Review checklist before any release

- [ ] Fixture suite covers every Tier 1 pattern, asserted in all 3 output formats.
- [ ] Allowlist mechanism works and is logged.
- [ ] Fail-safe path tested (un-scannable file → not exported).
- [ ] No Tier 1 pattern can be disabled by config (only by explicit reviewed allow).
