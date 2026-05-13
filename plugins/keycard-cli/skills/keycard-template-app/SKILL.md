---
name: keycard-template-app
description: |
  Scaffold a Keycard-aware app from a blueprint in `keycardai/templates` and execute the template's `SPEC.md` to provision Keycard primitives, configure the project, smoke-test, and hand off. 
  TRIGGER when: user wants to scaffold, template, or bootstrap a new Keycard-aware app from a blueprint.
  DO NOT TRIGGER when: user wants to register a credential for an existing service (use `keycard-discover-entities`); user wants to inspect what is already attached to the session (use `keycard-credentials`); user wants to deploy or modify an existing app (out of scope).
license: Apache-2.0
metadata: {}
---

# keycard-template-app

Scaffold a Keycard-aware app from a blueprint published in [`keycardai/templates`](https://github.com/keycardai/templates). The skill picks a template directory, copies it, fills in the project-name + env placeholders, and runs the Keycard wiring (Application/Resource registration, smoke-test, handoff).

The catalog is **the templates repo itself**. The skill maintains a local cache clone, refreshes it when stale, and enumerates templates by listing top-level directories that contain a `SPEC.md`. Each template's `SPEC.md` is **authoritative** for the provisioning steps; this skill only prescribes the generic shape.

## Pre-flight rules (read before doing anything)

- **Stay in the user's working directory.** Scaffold into `./<name>` relative to cwd; never `cd` elsewhere first.
- **`SPEC.md` is for the agent, not the user.** Read it; reason from it; never quote it. Paraphrase any jargon-heavy SPEC.md wording into the skill's voice before showing anything to the user.

### Narration style (applies to every step)

This skill is many users' first hands-on contact with Keycard. Treat the run as a guided tour, not a CI job. See [`skill-narration.md`](../../reference/skill-narration.md) for full narration rules and concept glossary.

### Resume protocol — `PROGRESS.md`

See [`skill-resume.md`](../../reference/skill-resume.md) for the full protocol. **File location:** write `PROGRESS.md` at `./<name>/PROGRESS.md` once Step 4 copies the template. Steps 1–3 are cheap to re-derive, so just re-prompt on resume if `./<name>` doesn't exist yet.

**Tell the user first:** something like "Pulling the latest starter blueprints from [keycardai/templates](https://github.com/keycardai/templates) — these are pre-wired example apps maintained by Keycard, so we don't start from a blank file."

Clone or refresh [`keycardai/templates`](https://github.com/keycardai/templates) into a cache directory at `${XDG_CACHE_HOME:-$HOME/.cache}/keycard/templates`. Reuse the cache across invocations; tolerate offline by falling back to the cached copy with a warning. The cache is read-only — never commit, push, or modify it.

## Step 1 — Resolve template

**Tell the user first:** if you have to prompt, frame it as "Each template is a working app you can run today — pick the shape that matches what you want to build, and I'll personalize it for you."

Enumerate templates from the cache and present a numbered list with the one-line summary from each `SPEC.md`:

```
Which template?
  1. mcp-server-typescript-express — <one-line summary from SPEC.md>
  2. mcp-brokered-credentials-typescript — <one-line summary from SPEC.md>
  …
```

Use judgment to pick the right template based on the user's prompt, but if in doubt confirm.

## Step 2 — Resolve project name

Ask the user for a project name:

```
What should the project be named? (kebab-case, e.g. <template-default-name>)
```

Read the suggested default from `SPEC.md` if provided, otherwise fall back to `<template-dir>`. Validate against `^[a-z][a-z0-9-]*$`. If `./<name>` exists and is non-empty, ask before overwriting.

## Step 3 — Resolve context

**Tell the user first:** introduce **zone** and **organization** with docs link on first mention. Only say this once.

The skill needs two values: **Zone ID** and **Org ID**. Derive `KEYCARD_URL` as `https://<zone-id>.keycard.cloud`.

`keycard run` does not export these — resolve from local config.

### Resolve zone ID and org ID

For each value, try in order:

1. Environment variable (`ZONE` / `ORG`)
2. `./keycard.toml` (`zone.id` / `org.id`)
3. If `KEYCARD_RUN=1`, list via `keycard agent api` (`/zones --org <org-id>` / `/organizations`) and let the user pick
4. Otherwise stop — explain the concept, link docs, tell user to run `keycard auth signin`, offer to write the ID into `keycard.toml` directly

Carry zone-id, org-id, and the derived `KEYCARD_URL` forward into all later steps.

### Active-session check

Only required for Management API calls (Step 7). Without `KEYCARD_RUN`, you may still scaffold and build (Steps 4–6) — warn that Step 7 will require `keycard run -- claude`.

## Step 4 — Copy the template into the user's project

**Tell the user first:** "Copying the `<template-dir>` blueprint into `./<name>`."

Print: `✓ Copied <template-dir>@<TEMPLATES_REF> → ./<name>`.

## Step 5 — Read the template's SPEC.md

Read `./<name>/SPEC.md` end-to-end. It is authoritative for data but **not** user-facing copy. If it conflicts with this skill, follow the spec and note the discrepancy in the final summary.

Extract these fields for later steps:

| Field | Where it feeds |
|---|---|
| Required primitives (providers, applications, resources, vault entries, dependencies) | Step 7 |
| Post-copy edits (files to write/patch, manifest fields to set) | Step 6 |
| Default port | Step 6 (port resolution) |
| Build / install command | Step 6 |
| Smoke-test endpoints + ready signal | Step 8 |
| Handoff commands (command, reason, run-where) | Step 9 |

## Step 6 — Fill in the template requirements

**Tell the user first:** "Filling in the blanks — project name, local port, `.env` with your zone URL, and a `keycard.toml` for CLI config."

Apply post-copy edits from `SPEC.md`. Common edits:

- Write `.env` (and committed `.env.example`) with resolved values (`KEYCARD_URL`, `<port>`, etc.)
- Set project name in language-native manifests (`package.json`, `pyproject.toml`, etc.)
- Set `[project].name` in `./<name>/keycard.toml`. Leave `${…}` placeholders and `[credentials]` empty unless the user asks for third-party API access.

### Resolve a safe port

Before writing `<port>` into `.env`, verify the template's default port is free. Probe with `lsof -iTCP:<candidate> -sTCP:LISTEN -n -P` or equivalent. If busy, scan upward (max 20 attempts, cap at 49151), warn the user, and use the first free port. If the entire window is exhausted, stop with an error — do not silently fall back to a random ephemeral port.

Carry `<port>` forward into `.env`, smoke-test URLs (Step 8), handoff (Step 9), and any Resource identifier that embeds the port (Step 7).

Then run the build/install command from `SPEC.md`. Surface errors verbatim; do not proceed until the build succeeds.

## Step 7 — Provision Keycard primitives (per SPEC.md §1)

**Tell the user first:** explain once what's being registered — an Application, Resource, and Provider — using the concept definitions from the narration section. Mention that re-running updates in place rather than creating duplicates.

After each create call, name the artifact: "✓ Application registered: `<application-id>`" — don't print raw API responses. Use concept introductions from the narration section, never SPEC.md wording.

Use `keycard agent api …` to create or look up each primitive in the order the spec dictates, scoped to `<zone-id>` and `<org-id>`.

General rules:

- **Idempotency.** Use stable identifiers (kebab-case `<name>`). On 409, look the entity up and reuse its ID.
- **Provider selection.** Pick the type SPEC.md specifies. If multiple match, ask. If none, abort with a `https://console.keycard.ai → Providers` pointer. Don't surface internal provider type names to the user.
- **Field accuracy.** Match identifiers to the protected URL exactly — mismatches cause `invalid_target` errors at token-mint time.
- **Failure handoff.** On 403, network error, or missing session: `Could not auto-register. Register manually at https://console.keycard.ai → <Section>.`

Carry every returned ID forward for the final summary.

## Step 8 — Smoke-test (per SPEC.md §3)

**Tell the user first:** "Booting the app locally to confirm everything wired up. I'll shut it down before handing it back."

If `SPEC.md` defines a smoke-test, run it for verification only and stop the process before handoff. Re-verify `<port>` is free before launching; if busy, re-scan per Step 6 and rewrite `.env`.

- Capture the child PID on spawn; kill by PID, not by process name.
- Wait for the documented ready signal with a 30s timeout. On timeout, kill, dump server log, and stop.
- `curl` against `http://127.0.0.1:<port>/…` (not `localhost`) to avoid IPv6/IPv4 false negatives.
- On failure, surface the error and re-check Step 6 values — most failures trace to a missing `.env` value or port/URL mismatch.
- **Always release the port** after verification (pass or fail). The process must not survive this step.

Use the skill's own voice for results — don't quote SPEC.md. If the template has no smoke-test, skip this step.

## Step 9 — Hand off to the user

The final step is **manual**. Print clear, copy-pasteable instructions and stop — do not start long-running processes or block.

If handoff data includes a safe config-write (e.g. `claude mcp add`), run it silently. Anything requiring the user's terminal or a fresh agent session goes into the printed block.

### What to print — exactly one block

One unified handoff block with three parts:

1. **Recap paragraph.** Plain English summary referencing `<template-dir>`, `<name>`, build command, `<zone-id>`, registered IDs, `<port>`, and verified endpoints.

2. **"What's left for you" numbered list.** Copy-pasteable commands with one-line reasons in the skill's voice. Example:

   > 1. `cd <name> && keycard run -- npm start` — starts your local server with the credentials Keycard manages for it
   > 2. `keycard run -- claude` (new terminal) — restart your agent so it picks up the new server
   > 3. `/mcp` → pick `<name>` → follow the login flow — connects your agent to the server

3. **Console pointer.** "If anything looks off, the registered IDs above are what to check in https://console.keycard.ai."

### Prohibitions

- Do **not** print a separate "Next steps" or "from SPEC.md" block — one unified block only.
- Do **not** pass through SPEC.md phrasing — translate implementation detail into outcomes.
- Omit empty sections.

## Examples

**Invocation:** `/keycard-template-app` — the skill prompts for template and project name.

**Deploying:** once the scaffold is working locally, run `/keycard-deploy-app` to ship it to a hosting provider (fly.io, etc.).

**Extending the scaffold after handoff:** follow the patterns the template ships (its `README.md`, example modules, etc.). Once handoff is printed, further changes are owned by the user, not this skill.
