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

Scaffold a Keycard-aware app from a blueprint published in [`keycardai/templates`](https://github.com/keycardai/templates). The skill gathers all requirements upfront — template choice, project name, zone context, and the state of existing Keycard primitives — presents a plan, and only executes after the user confirms.

The catalog is **the templates repo itself**. The skill maintains a local cache clone, refreshes it when stale, and enumerates templates by listing top-level directories that contain a `SPEC.md`. Each template's `SPEC.md` is **authoritative** for the provisioning steps; this skill only prescribes the generic shape.

## Pre-flight rules (read before doing anything)

- **Stay in the user's working directory.** Scaffold into `./<name>` relative to cwd; never `cd` elsewhere first.
- **`SPEC.md` is for the agent, not the user.** Read it; reason from it; never quote it. Paraphrase any jargon-heavy SPEC.md wording into the skill's voice before showing anything to the user.

### Narration style (applies to every step)

This skill is many users' first hands-on contact with Keycard. Treat the run as a guided tour, not a CI job. See [`skill-narration.md`](../../reference/skill-narration.md) for full narration rules and concept glossary.

### Resume protocol — `PROGRESS.md`

See [`skill-resume.md`](../../reference/skill-resume.md) for the full protocol. **File location:** write `PROGRESS.md` at `./<name>/PROGRESS.md` once Step 7 copies the template. Steps 1–6 are cheap to re-derive, so just re-prompt on resume if `./<name>` doesn't exist yet.

---

## Phase A — Gather and Plan

Steps 1–6 collect everything the skill needs and present a plan. No files are written and no API mutations happen until the user confirms.

**Tell the user first:** something like "Pulling the latest starter blueprints from [keycardai/templates](https://github.com/keycardai/templates) — these are pre-wired example apps maintained by Keycard, so we don't start from a blank file."

Clone or refresh [`keycardai/templates`](https://github.com/keycardai/templates) into a cache directory at `${XDG_CACHE_HOME:-$HOME/.cache}/keycard/templates`. Reuse the cache across invocations; tolerate offline by falling back to the cached copy with a warning. The cache is read-only — never commit, push, or modify it.

### Step 1 — Resolve template

**Tell the user first:** if you have to prompt, frame it as "Each template is a working app you can run today — pick the shape that matches what you want to build, and I'll personalize it for you."

Enumerate templates from the cache and present a numbered list with the one-line summary from each `SPEC.md`:

```
Which template?
  1. mcp-server-typescript-express — <one-line summary from SPEC.md>
  2. mcp-brokered-credentials-typescript — <one-line summary from SPEC.md>
  …
```

Use judgment to pick the right template based on the user's prompt, but if in doubt confirm.

### Step 2 — Resolve project name

Ask the user for a project name:

```
What should the project be named? (kebab-case, e.g. <template-default-name>)
```

Read the suggested default from `SPEC.md` if provided, otherwise fall back to `<template-dir>`. Validate against `^[a-z][a-z0-9-]*$`. If `./<name>` exists and is non-empty, ask before overwriting.

### Step 3 — Resolve context

**Tell the user first:** introduce **zone** and **organization** with docs link on first mention. Only say this once.

The skill needs two values: **Zone ID** and **Org ID**. Derive `KEYCARD_URL` as `https://<zone-id>.keycard.cloud`.

`keycard run` does not export these — resolve from local config.

#### Resolve zone ID and org ID

For each value, try in order:

1. Environment variable (`ZONE` / `ORG`)
2. `./keycard.toml` (`zone.id` / `org.id`)
3. If `KEYCARD_RUN=1`, list via `keycard agent api` (`/zones --org <org-id>` / `/organizations`) and let the user pick
4. Otherwise stop — explain the concept, link docs, tell user to run `keycard auth signin`, offer to write the ID into `keycard.toml` directly

Carry zone-id, org-id, and the derived `KEYCARD_URL` forward into all later steps.

#### Active-session check

Required for the Management API calls in Step 5 and Step 9. Without `KEYCARD_RUN`, you may still gather template info (Steps 1–4) and present a plan (Step 6), but warn that Steps 5 and 9 will require `keycard run -- claude`.

### Step 4 — Read the template's SPEC.md from cache

Read `${CACHE}/<template-dir>/SPEC.md` end-to-end — directly from the template cache, **before** copying anything. It is authoritative for data but **not** user-facing copy. If it conflicts with this skill, follow the spec and note the discrepancy in the final summary.

Extract these fields for later steps:

| Field | Where it feeds |
|---|---|
| Required primitives (providers, applications, resources, vault entries, dependencies) | Step 5 (lookup) + Step 9 (provision) |
| Post-copy edits (files to write/patch, manifest fields to set) | Step 8 |
| Default port | Step 8 (port resolution) |
| Build / install command | Step 8 |
| Smoke-test endpoints + ready signal | Step 10 |
| Handoff commands (command, reason, run-where) | Step 11 |

### Step 5 — Look up existing primitives

Query the Management API to determine which primitives the SPEC requires already exist in the zone and which need to be created. This avoids surprises during provisioning and lets the plan show the user exactly what will happen.

Use the zone-scoped listing endpoints from [`keycard-management-api.md`](../../reference/keycard-management-api.md) to fetch providers, applications, resources, and application-credentials (all `GET` requests — no mutations).

For each primitive the SPEC requires, filter the returned `items` array client-side on the `identifier` field (or `{application_id, provider_id, subject}` triplet for application-credentials). Classify each as:

- **exists** — reuse its `id`; carry the ID forward
- **needs creation** — will be created in Step 9

For provider lookups, match on the `type` field (e.g. `keycard-sts`, `keycard-vault`, `oauth`). If the SPEC requires a provider type that doesn't exist in the zone, mark it as a blocker — the plan will surface this.

If the Management API is unavailable (no active session), skip the lookup and mark all primitives as **unknown** — the plan will note that provisioning status couldn't be verified and the user will need `keycard run -- claude`.

### Step 6 — Present plan

Show the user a structured summary of everything that will happen. This is a hard gate — do **not** proceed to Phase B without explicit user confirmation.

Format:

```
Here's what I'm going to set up:

Template:  <template-dir>
           <one-line summary from SPEC.md>
Project:   ./<name>
Zone:      <zone-id> (org: <org-id>)

Config files I'll write:
  - .env (KEYCARD_URL, PORT)
  - keycard.toml (zone/org IDs)
  - <manifest> (project name)

Keycard primitives:
  ✓ exists   Provider "<provider-name>" (<provider-type>)
  → create   Application "<name>"
  → create   Resource "http://localhost:<port>/mcp"

Build:     <build command from SPEC.md>
Verify:    <smoke-test summary from SPEC.md>

Ready to go?
```

Rules for the plan:

- Use `✓ exists` for primitives found in Step 5, `→ create` for those that need creation.
- If Step 5 was skipped (no session), use `? unknown` and note that provisioning status couldn't be checked.
- If the SPEC requires a provider type that doesn't exist, show `✗ missing` and explain what the user needs to create manually before proceeding.
- For templates with many primitives (e.g. `mcp-brokered-credentials-typescript` with Linear provider, Linear app, Linear resource, proxy app, proxy resource, dependency wiring, vault resources), show all of them.
- Use the skill's narration voice and concept glossary, not SPEC.md jargon. Introduce concepts on first mention with docs links per [`skill-narration.md`](../../reference/skill-narration.md).
- If the user wants changes (different name, different template, different zone), loop back to the relevant gather step and re-present the plan.

---

## Phase B — Execute

The user has confirmed. Now copy, configure, provision, verify, and hand off.

### Step 7 — Copy the template into the user's project

**Tell the user first:** "Copying the `<template-dir>` blueprint into `./<name>`."

Print: `✓ Copied <template-dir>@<TEMPLATES_REF> → ./<name>`.

### Step 8 — Fill in the template requirements

**Tell the user first:** "Filling in the blanks — project name, local port, `.env` with your zone URL, and a `keycard.toml` for CLI config."

Apply post-copy edits from `SPEC.md`. Common edits:

- Write `.env` (and committed `.env.example`) with resolved values (`KEYCARD_URL`, `<port>`, etc.)
- Set project name in language-native manifests (`package.json`, `pyproject.toml`, etc.)
- Set `[project].name` in `./<name>/keycard.toml`. Leave `${…}` placeholders and `[credentials]` empty unless the user asks for third-party API access.

#### Resolve a safe port

Before writing `<port>` into `.env`, verify the template's default port is free. Probe with `lsof -iTCP:<candidate> -sTCP:LISTEN -n -P` or equivalent. If busy, scan upward (max 20 attempts, cap at 49151), warn the user, and use the first free port. If the entire window is exhausted, stop with an error — do not silently fall back to a random ephemeral port.

Carry `<port>` forward into `.env`, smoke-test URLs (Step 10), handoff (Step 11), and any Resource identifier that embeds the port (Step 9).

Then run the build/install command from `SPEC.md`. Surface errors verbatim; do not proceed until the build succeeds.

### Step 9 — Provision Keycard primitives (per SPEC.md §1)

**Tell the user first:** explain once what's being registered — an Application, Resource, and Provider — using the concept definitions from the narration section. Mention that re-running updates in place rather than creating duplicates.

For primitives classified as **exists** in Step 5, reuse the carried ID — no API call needed. For primitives classified as **needs creation**, create them now.

After each create call, name the artifact: "✓ Application registered: `<application-id>`" — don't print raw API responses. Use concept introductions from the narration section, never SPEC.md wording.

Use `keycard agent api …` to create or look up each primitive in the order the spec dictates, scoped to `<zone-id>` and `<org-id>`.

General rules:

- **Idempotency.** Use stable identifiers (kebab-case `<name>`). On 409, look the entity up and reuse its ID.
- **Provider selection.** Pick the type SPEC.md specifies. If multiple match, ask. If none, abort with a `https://console.keycard.ai → Providers` pointer. Don't surface internal provider type names to the user.
- **Field accuracy.** Match identifiers to the protected URL exactly — mismatches cause `invalid_target` errors at token-mint time.
- **Failure handoff.** On 403, network error, or missing session: `Could not auto-register. Register manually at https://console.keycard.ai → <Section>.`

Carry every returned ID forward for the final summary.

### Step 10 — Smoke-test (per SPEC.md §3)

**Tell the user first:** "Booting the app locally to confirm everything wired up. I'll shut it down before handing it back."

If `SPEC.md` defines a smoke-test, run it for verification only and stop the process before handoff. Re-verify `<port>` is free before launching; if busy, re-scan per Step 8 and rewrite `.env`.

- Capture the child PID on spawn; kill by PID, not by process name.
- Wait for the documented ready signal with a 30s timeout. On timeout, kill, dump server log, and stop.
- `curl` against `http://127.0.0.1:<port>/…` (not `localhost`) to avoid IPv6/IPv4 false negatives.
- On failure, surface the error and re-check Step 8 values — most failures trace to a missing `.env` value or port/URL mismatch.
- **Always release the port** after verification (pass or fail). The process must not survive this step.

Use the skill's own voice for results — don't quote SPEC.md. If the template has no smoke-test, skip this step.

### Step 11 — Hand off to the user

The final step is **manual**. Print clear, copy-pasteable instructions and stop — do not start long-running processes or block.

If handoff data includes a safe config-write (e.g. `claude mcp add`), run it silently. Anything requiring the user's terminal or a fresh agent session goes into the printed block.

#### What to print — exactly one block

One unified handoff block with three parts:

1. **Recap paragraph.** Plain English summary referencing `<template-dir>`, `<name>`, build command, `<zone-id>`, registered IDs, `<port>`, and verified endpoints.

2. **"What's left for you" numbered list.** Copy-pasteable commands with one-line reasons in the skill's voice. Example:

   > 1. `cd <name> && keycard run -- npm start` — starts your local server with the credentials Keycard manages for it
   > 2. `keycard run -- claude` (new terminal) — restart your agent so it picks up the new server
   > 3. `/mcp` → pick `<name>` → follow the login flow — connects your agent to the server

3. **Console pointer.** "If anything looks off, the registered IDs above are what to check in https://console.keycard.ai."

#### Prohibitions

- Do **not** print a separate "Next steps" or "from SPEC.md" block — one unified block only.
- Do **not** pass through SPEC.md phrasing — translate implementation detail into outcomes.
- Omit empty sections.

## Examples

**Invocation:** `/keycard-template-app` — the skill gathers template, name, and context, presents a plan showing which primitives already exist and which will be created, then executes after confirmation.

**Deploying:** once the scaffold is working locally, run `/keycard-deploy-app` to ship it to a hosting provider (fly.io, etc.).

**Extending the scaffold after handoff:** follow the patterns the template ships (its `README.md`, example modules, etc.). Once handoff is printed, further changes are owned by the user, not this skill.
