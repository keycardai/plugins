---
name: keycard-deploy-app
description: |
  Deploy a Keycard-aware app to a hosting provider — wire credentials, configure OIDC, and ship.
  TRIGGER when: user wants to deploy, ship, or push a Keycard-protected app to a hosting platform (fly.io, etc.); says "deploy my app", "ship to fly", "push to production", "deploy to fly.io".
  DO NOT TRIGGER when: user wants to scaffold a new app from a template (use `keycard-template-app`); user wants to inspect existing credentials (use `keycard-credentials`); user wants to modify an existing Cedar policy rule without deploying (use `keycard-upsert-policy`).
license: Apache-2.0
metadata: {}
---

# keycard-deploy-app

Deploy a Keycard-aware app to a hosting provider. The skill is a generic dispatcher — it resolves the provider, reads the provider-specific reference doc, and executes its steps. Per-provider procedures live in `.agents/reference/deploy-<provider>.md` (resolve the link relative to this `SKILL.md`).

Currently supported providers:

| Provider | Reference doc | CLI tool |
|---|---|---|
| `fly` | [`deploy-fly.md`](../../reference/deploy-fly.md) | `flyctl` |

## Pre-flight rules (read before doing anything)

- **Two `keycard.toml` files, two environments.** Provider-CLI credentials (e.g. `FLY_API_TOKEN`) belong in the **session** `keycard.toml` (at the session root, read by `keycard run -- claude`). App-runtime credentials (`KEYCARD_CLIENT_ID/SECRET`, third-party API tokens) belong in `<app-dir>/keycard.toml` (ships with the deployment). Before writing a credential entry, reason about which environment consumes the env var.

- **Stay in the user's working directory.** Deploy from `./<app-dir>` relative to wherever the user invoked it.
- **Never read or echo credential env vars.** The skill must not `cat .env`, `printenv FLY_API_TOKEN`, or grep for secrets. Provider CLIs consume their credentials from the process environment, which Keycard injects automatically.
- **Never substitute a secret value into a shell command.** Always invoke the provider CLI bare and let it consume the env var.
- **All writes go through existing skills.** Credential entries → `/keycard-discover-entities` or `/keycard-upsert-config`. Policy edits → `/keycard-upsert-policy`. `keycard.toml` edits → `/keycard-upsert-config`. This skill owns only Management API reads/creates and provider CLI invocations.

### Narration style (applies to every step)

This skill ships a user's work to production — often their first deploy with Keycard. Treat the run as a guided walkthrough, not a CI log. See [`skill-narration.md`](../../reference/skill-narration.md) for full narration rules and concept glossary.

### Resume protocol — `PROGRESS.md`

See [`skill-resume.md`](../../reference/skill-resume.md) for the full protocol. **File location:** write `PROGRESS.md` at `./<app-dir>/PROGRESS.md` (append below any existing `keycard-template-app` steps).

## Step 1 — Resolve provider

**Tell the user first:** "First, figuring out which hosting platform to deploy to."

Default to `fly`. If the user's prompt names a different provider, match it against the supported table above. If no match, stop with an error listing supported providers.

## Step 2 — Resolve app directory and project context

**Tell the user first:** "Looking at your project to understand what we're deploying."

Infer the app directory from the user's prompt or default to `.`. Read `<app-dir>/keycard.toml` for `[project].name`; fall back to the project manifest (`package.json`, `pyproject.toml`, `Cargo.toml`) if unset.

### Resolve zone and org context

Follow the same resolution logic as `keycard-template-app` Step 3:

1. **Zone ID** — from `keycard.toml` `zone.id`, then `ZONE` env var, then `keycard agent api /zones --org <org-id>` if `KEYCARD_RUN=1`.
2. **Org ID** — from `ORG` env var, then `keycard.toml` `org.id`, then `keycard agent api /organizations` if `KEYCARD_RUN=1`.

If either cannot be resolved, stop with the same friendly error messages as `keycard-template-app` Step 3 (explain what a zone/org is, link docs, tell the user to run `keycard auth signin`).

### Derive zone URL

Derive `KEYCARD_URL` using the env-domain table in `keycard-template-app` Step 3. Carry `<zone-id>`, `<org-id>`, and `<KEYCARD_URL>` forward into all later steps.

## Step 3 — Toolchain pre-flight

**Tell the user first:** "Checking that the hosting provider's CLI tool is installed."

Verify the provider CLI is installed:

```bash
command -v <provider-cli>
```

## Step 4 — Execute provider-specific procedure

**Tell the user first:** Before executing the provider reference doc, give a brief overview of what's about to happen — e.g. "Now I'll walk through the fly.io deploy steps: check credentials, set up the deployment config, tell Keycard to trust your fly org's identity, register the deployed URL, ship, and verify."

Read `.agents/reference/deploy-<provider>.md` end-to-end. It is the **authoritative source** for everything that follows — if anything in this skill conflicts with the reference doc, follow the reference doc.

The reference doc may require provider-specific values (e.g. fly org slug). Prompt the user for any required value not already known. When the reference doc specifies an enumeration command (e.g. `flyctl orgs list`), run it and present the user with their actual choices — never auto-select or invent candidates.

Execute each section of the reference doc in order. When a section wires credentials, apply the two-environment rule: provider-CLI credentials target the **session** `keycard.toml`; app-runtime credentials target `<app-dir>/keycard.toml`. Invoke `/keycard-discover-entities` from the correct working directory — it writes to cwd's `keycard.toml`.

Narrate before each section and surface created artifacts/IDs after. Never dump raw API responses.

## Step 5 — Handoff summary

Print **one unified block** with three parts:

1. **Recap paragraph** — plain-English summary referencing deployed URL, registered IDs, and both Resources (local + deployed).
2. **Status list** — `✓` for completed steps, `→` for outstanding user actions. Build from the provider reference doc's sections.
3. **Console pointer** — "If anything looks off, the registered IDs above are what to check in https://console.keycard.ai."

**Prohibitions:**
- No separate "Next steps" or "from the reference doc" blocks — one unified block only.
- Translate implementation detail into outcomes; do not pass through reference-doc phrasing.
- When a restart is required, include: (a) why, (b) what changes, (c) the exact resume command. Reproduce the rationale from the provider reference doc (e.g. `deploy-fly.md` §2).

## Examples

**Invocation:** `/keycard-deploy-app` — the skill infers the provider (defaults to fly) and app directory, then prompts for any missing values like the fly org slug.
