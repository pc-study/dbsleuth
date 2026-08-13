# Exit Codes

> Status: normative spec. Implementations MUST follow these codes so callers (humans, CI, cron, wrapper scripts) can branch reliably.

DBSleuth is a CLI that follows the Unix convention "exit non-zero on anything that needs attention". The first principle — *"unknown is not the same as normal, unsupported input must be reported explicitly, never silently mis-parsed"* — drives the distinction between `0`, `2`, and `3`.

## Code table

| Code | Name | Meaning | Typical trigger |
|---|---|---|---|
| **0** | `OK` | Analysis completed; no parse errors and no unknown input encountered. Report produced. | Normal happy path |
| **1** | `GENERIC_ERROR` | Catch-all for unexpected internal errors / panics / unhandled exceptions. Diagnostics to stderr. | Bug, panic, unrecoverable runtime error |
| **2** | `PARTIAL` | Analysis completed but with caveats: some files failed to parse, hit a safety limit (size/count/nesting/ratio), or some input was unrecognized and emitted as `unknown`. Report still produced. | A file failed to parse; zip-bomb limit hit; encoding could not be detected; timezone missing |
| **3** | `UNSUPPORTED_INPUT` | Input format/type is not supported at all (not a partial failure — the tool refused to process it). No report produced. | Unknown archive type, non-Oracle/non-Linux source when no parser matches |
| **4** | `USAGE` | Bad CLI usage, missing required flags, invalid argument values. Equivalent to `argparse`/Cobra conventions. | `--window` not a number, missing input path |
| **5** | `INTEGRITY_FAILURE` | Archive/hash verification failed: tampered manifest, checksum mismatch after extraction, or signature validation failed. Stops processing. | Decompressed file SHA-256 != manifest value |
| **6** | `CONFIG_ERROR` | Config file invalid: schema mismatch, unknown key, unsafe value (e.g. raised a safety limit above its hard ceiling). | YAML parse error, `max_expanded_size: 0` |

## Rules

1. **Partial success is `2`, not `0`.** If even one file parsed as `unknown` or one safety limit fired, the exit code MUST be `2`. This honors the *"unknown ≠ normal"* principle: a caller that only checks `exit 0` will correctly notice something needs review.
2. **Priority order when multiple apply**: `USAGE` (4) > `CONFIG_ERROR` (6) > `INTEGRITY_FAILURE` (5) > `UNSUPPORTED_INPUT` (3) > `GENERIC_ERROR` (1) > `PARTIAL` (2) > `OK` (0). E.g. if usage is wrong AND a file failed, return 4.
3. **Diagnostics go to stderr**, never to stdout. stdout carries only the report path / machine-readable output. A `--verbose` flag increases stderr detail.
4. **Every non-zero exit MUST be accompanied by a human-readable reason on stderr**, plus a stable machine-readable code string (e.g. `DBSLEUTH_E_LIMIT_SIZE`) so CI can match on it without parsing prose.
5. **`PARTIAL` reports are still useful** — the user gets a report plus an explicit "what was skipped and why" section. Do not discard partial work.

## Stable error code strings (machine-readable)

Prefix `DBSLEUTH_E_`, emitted on stderr as `DBSLEUTH_E_<NAME>: <message>`:

- `DBSLEUTH_E_USAGE` — bad flags / missing args
- `DBSLEUTH_E_CONFIG` — config invalid
- `DBSLEUTH_E_UNSUPPORTED` — input type not supported
- `DBSLEUTH_E_INTEGRITY` — checksum / manifest mismatch
- `DBSLEUTH_E_LIMIT_SIZE` / `_LIMIT_COUNT` / `_LIMIT_NESTING` / `_LIMIT_RATIO` — which safety limit fired
- `DBSLEUTH_E_PARSE` — a parser failed on a specific file (collect all, exit 2)
- `DBSLEUTH_E_ENCODING` — encoding could not be detected/guessed
- `DBSLEUTH_E_INTERNAL` — unexpected internal error

This list is the authoritative enumeration; CLI subcommands and exit-code docs must stay in sync.
