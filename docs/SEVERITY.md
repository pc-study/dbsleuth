# Severity Enumeration (authoritative)

> Status: normative spec. This is the SINGLE source of truth for event severity. All other documents (README, ARCHITECTURE, STATE_ENGINE, schemas, code) MUST reference this file instead of re-defining severity inline.

## Enum values

| Value | Meaning | Example |
|---|---|---|
| `critical` | Imminent or in-progress failure; data loss / downtime risk; requires immediate action | ORA-00600 fatal, instance abort, disk full |
| `high` | Serious condition likely to cause failure if unaddressed; proactive action needed | Control file enqueue held long, storage near threshold |
| `medium` | Notable condition worth attention; may degrade performance/reliability | Recurring ORA-01555, lock wait spikes |
| `low` | Minor / informational / benign but worth logging | Session disconnected, auto-task retry |
| `info` | Pure informational / context (not a fault) | Startup banner, parameter change note |
| `unknown` | Could not be classified reliably. **Must NOT be treated as normal.** | Parser saw an unrecognized ORA code or unparseable line |

## Rules

1. `unknown` is its own severity — it is NOT a synonym for `low` or `info`. Per the project first principle, unknown input is reported explicitly and must never be silently reclassified as normal.
2. Severity is assigned by deterministic rules (`internal/rules/`). If no rule matches, the event keeps `unknown` — it is never guessed.
3. Severity ordering for filtering/sorting: `critical > high > medium > low > info`, with `unknown` sorted to the top of any "needs review" view (so it gets attention, not buried).
4. This enum is versioned alongside `schema_version`. Adding a value requires bumping the schema major version and migrating fixtures.

## Where this is referenced

- `ARCHITECTURE.md` canonical event schema → `severity` field values come from this enum.
- `docs/STATE_ENGINE.md` state codes carry a severity projection derived from this enum.
- `schemas/` JSON Schema files use `"enum": ["critical","high","medium","low","info","unknown"]`.
- CLI `--min-severity` flag accepts exactly these values.
