# Skill Resume Protocol — `PROGRESS.md`

Shared resume protocol for multi-step skills. Each skill specifies its own file location; everything else is common.

## Rules

- The skill must be **resumable** — when re-invoked it picks up where it left off rather than creating duplicates.
- **File shape.** Plain Markdown with: metadata (skill-specific context like project path, zone, org, timestamps), a step checklist (`[x]`/`[ ]`) with sub-items for each registered ID or failure reason, and a free-form Notes section.
- **Update cadence.** Update after every meaningful action (registered ID, deploy result, smoke-test, handoff). Always bump `Last updated`.
- **On resume.** When a `PROGRESS.md` exists, summarize it to the user and ask whether to **resume**, **start over** (rename to `.bak-<timestamp>`; never delete user code), or **abandon**. Never silently continue.
- **Sanity-check before reuse.** Verify claimed IDs via `keycard agent api` before reusing — if gone, warn and re-create.
- **On clean success.** Mark every step `[x]` and add `Completed: <timestamp>`.
