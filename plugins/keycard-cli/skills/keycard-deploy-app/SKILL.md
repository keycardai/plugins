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

Deploy a Keycard-aware app to a hosting provider. The skill gathers all requirements upfront — provider, app directory, zone context, toolchain status, and the state of existing Keycard primitives — presents a plan, and only executes after the user confirms.

The skill is a generic dispatcher — it resolves the provider, reads the provider-specific reference doc, and executes its steps. Per-provider procedures live in `reference/deploy-<provider>.md` (resolve the link relative to this `SKILL.md`).

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

---

## Phase A — Gather and Plan

Steps 1–5 collect everything the skill needs and present a plan. No files are written and no API mutations happen until the user confirms.

### Step 1 — Resolve provider

**Tell the user first:** "First, figuring out which hosting platform to deploy to."

Default to `fly`. If the user's prompt names a different provider, match it against the supported table above. If no match, stop with an error listing supported providers.

### Step 2 — Resolve app directory and project context

**Tell the user first:** "Looking at your project to understand what we're deploying."

Infer the app directory from the user's prompt or default to `.`. Read `<app-dir>/keycard.toml` for `[project].name`; fall back to the project manifest (`package.json`, `pyproject.toml`, `Cargo.toml`) if unset.

#### Resolve zone and org context

Follow the same resolution logic as `keycard-template-app` Step 3:

1. **Zone ID** — from `keycard.toml` `zone.id`, then `ZONE` env var, then `keycard agent api /zones --org <org-id>` if `KEYCARD_RUN=1`.
2. **Org ID** — from `ORG` env var, then `keycard.toml` `org.id`, then `keycard agent api /organizations` if `KEYCARD_RUN=1`.

If either cannot be resolved, stop with the same friendly error messages as `keycard-template-app` Step 3 (explain what a zone/org is, link docs, tell the user to run `keycard auth signin`).

#### Derive zone URL

Derive `KEYCARD_URL` as `https://<zone-id>.keycard.cloud`. Carry `<zone-id>`, `<org-id>`, and `<KEYCARD_URL>` forward into all later steps.

### Step 3 — Toolchain pre-flight

**Tell the user first:** "Checking that the hosting provider's CLI tool is installed."

Verify the provider CLI is installed:

```bash
command -v <provider-cli>
```

If the CLI is missing, surface the install command from the provider reference doc and stop.

### Step 4 — Read provider reference doc and look up existing primitives

**Tell the user first:** "Checking what's already registered in Keycard so I know what needs to be created."

Read `reference/deploy-<provider>.md` end-to-end from the skill's reference directory. It is the **authoritative source** for the provider-specific procedure. Extract:

| Field | Where it feeds |
|---|---|
| Required primitives (providers, applications, resources, application-credentials) | Step 4 (lookup) + Phase B (provision) |
| Provider arguments (e.g. fly org slug) | Step 4 (prompt) + Phase B (execution) |
| Credential env var and probe command | Step 4 (status check) + Phase B (wiring) |
| Deploy command | Phase B (deploy) |
| Verification endpoints | Phase B (verify) |

#### Resolve provider-specific arguments

If the reference doc requires additional arguments (e.g. `<fly-org-slug>`), resolve them now. When the reference doc specifies an enumeration command (e.g. `flyctl orgs list`), run it and present the user with their actual choices — never auto-select or invent candidates.

#### Check credential status

Probe whether the deploy credential is already wired using the reference doc's probe command (e.g. `flyctl auth whoami`). Classify as:

- **wired** — credential is available; no session restart will be needed
- **not wired** — credential needs to be configured; the user will choose between Keycard Vault (automated setup + console link + session restart) or manual login (`flyctl auth login`, no restart needed)

#### Look up existing Keycard primitives

Query the Management API to determine which primitives the reference doc requires already exist in the zone and which need to be created. This avoids surprises during execution and lets the plan show the user exactly what will happen.

Use the zone-scoped listing endpoints from [`keycard-management-api.md`](../../reference/keycard-management-api.md) to fetch providers, applications, resources, and application-credentials (all `GET` requests — no mutations).

For each primitive the reference doc requires, classify as:

- **exists** — reuse its `id`; carry the ID forward
- **needs creation** — will be created in Phase B

For provider lookups, match on the `protocols.oauth2.issuer` field (e.g. `https://oidc.fly.io/<fly-org-slug>`). If the reference doc requires a provider type that doesn't exist in the zone, mark it as a blocker.

If the Management API is unavailable (no active session), skip the lookup and mark all primitives as **unknown** — the plan will note that provisioning status couldn't be verified.

### Step 5 — Present plan

Show the user a structured summary of everything that will happen. This is a hard gate — do **not** proceed to Phase B without explicit user confirmation.

Format:

```
Here's the deploy plan:

Provider:     fly.io
App:          ./<app-dir> (<project-name>)
Fly org:      <fly-org-slug>
Zone:         <zone-id> (org: <org-id>)

Credential:
  ✓ wired     FLY_API_TOKEN (flyctl auth whoami succeeded)
  — or —
  → configure FLY_API_TOKEN (choose: Keycard Vault or manual login)

Deploy artifacts:
  - fly.toml (KEYCARD_URL, internal_port=<port>)
  - Dockerfile (if needed)

Keycard primitives:
  ✓ exists    Provider "fly-<fly-org-slug>" (OIDC)
  → create    Application "<project-name>"
  → create    Resource "https://<project-name>.fly.dev/<path>"
  → create    Application Credential (<fly-org-slug>:<project-name>:*)

Ship:         flyctl deploy --remote-only
Verify:       flyctl status + well-known endpoint checks

Ready to go?
```

Rules for the plan:

- Use `✓ exists` / `✓ wired` for items found in Step 4, `→ create` / `→ configure` for those that need work.
- If Step 4 lookups were skipped (no session), use `? unknown` and note that provisioning status couldn't be checked.
- If the reference doc requires a provider type that doesn't exist, show `✗ missing` and explain what the user needs to create manually.
- If the credential is not wired, prominently note that a **session restart** will be required partway through execution (after wiring the credential, before deploying).
- Use the skill's narration voice and concept glossary, not reference-doc jargon. Introduce concepts on first mention with docs links per [`skill-narration.md`](../../reference/skill-narration.md).
- If the user wants changes (different provider, different app dir, different fly org), loop back to the relevant gather step and re-present the plan.

---

## Phase B — Execute

The user has confirmed. Now wire credentials, configure deploy artifacts, provision primitives, ship, and verify.

Execute each section of the provider reference doc in order. Narrate before each section and surface created artifacts/IDs after. Never dump raw API responses.

### Step 6 — Credential pre-flight (per reference doc §1)

If the credential was classified as **wired** in Step 4, skip ahead to Step 8.

If the credential was classified as **not wired**, proceed to Step 7.

### Step 7 — Configure credential (per reference doc §2)

Present the user with the two options from the reference doc:

- **Option A — Keycard Vault (recommended):** Create a vault resource via the Management API, wire the credential entry in the **session-root** `keycard.toml` via `/keycard-upsert-config` (invoke from the session-root working directory, not from inside `<app-dir>`), construct the console credentials URL and present it so the user can paste their deploy token, then trigger a session restart.
- **Option B — Manual login:** Instruct the user to run `flyctl auth login` in a separate terminal, then re-probe with `flyctl auth whoami`. No session restart needed.

Follow the reference doc's detailed procedure for whichever option the user picks. Read the active policy with `keycard agent policy` in either case — if it would block provider CLI invocations, delegate to `/keycard-upsert-policy`.

#### Session restart (Option A only)

After wiring the credential, the agent session must be restarted. Print the rationale block from the reference doc (substituting actual values), create `PROGRESS.md`, and stop. Do not proceed until the session has been restarted and the credential probe succeeds in the new session.

### Step 8 — Deploy artifacts (per reference doc §3)

**Tell the user first:** "Setting up the deployment configuration — this tells the hosting provider how to build and run your app."

Follow the reference doc's deploy artifact procedure:
- Port alignment between the app and the deploy config.
- Generate or validate provider-specific config files (e.g. `fly.toml`).
- Write `KEYCARD_URL` into the deploy config.
- Surface generated files and require user confirmation before continuing.

### Step 9 — Provision Keycard primitives (per reference doc §4–§6)

**Tell the user first:** explain once what's being registered — Providers, Applications, Resources, and Application Credentials — using the concept definitions from the narration section. Mention that re-running updates in place rather than creating duplicates.

For primitives classified as **exists** in Step 4, reuse the carried ID — no API call needed. For primitives classified as **needs creation**, create them now following the reference doc's procedure.

After each create call, name the artifact: "✓ Provider registered: `fly-<fly-org-slug>`" — don't print raw API responses.

General rules:

- **Idempotency.** Use stable identifiers. On 409, look the entity up and reuse its ID.
- **Provider selection.** Pick the type the reference doc specifies. If multiple match, ask. If none, abort with a `https://console.keycard.ai → Providers` pointer.
- **Field accuracy.** Match identifiers to the deployed URL exactly — mismatches cause `invalid_target` errors at token-mint time.
- **Failure handoff.** On 403, network error, or missing session: `Could not auto-register. Register manually at https://console.keycard.ai → <Section>.`

Carry every returned ID forward for the final summary.

### Step 10 — Deploy (per reference doc §7)

**Tell the user first:** "Shipping to the hosting provider — this builds and deploys your app."

Run the deploy command from the reference doc (e.g. `flyctl deploy --remote-only` from `<app-dir>`).

On failure: surface the verbatim provider CLI output, run provider-specific log commands for triage, and stop. Do **not** read `.env` or echo secret values.

### Step 11 — Verify (per reference doc §8)

**Tell the user first:** "Verifying the deploy — checking the app is running and Keycard trust is configured correctly."

Follow the reference doc's verification procedure (app status, well-known endpoint checks, trust chain verification). Surface any mismatches with actionable fix instructions.

### Step 12 — Handoff summary

Print **one unified block** with three parts:

1. **Recap paragraph** — plain-English summary referencing deployed URL, registered IDs, and both Resources (local + deployed).
2. **Status list** — `✓` for completed steps, `→` for outstanding user actions. Build from the provider reference doc's sections.
3. **Console pointer** — "If anything looks off, the registered IDs above are what to check in https://console.keycard.ai."

**Prohibitions:**
- No separate "Next steps" or "from the reference doc" blocks — one unified block only.
- Translate implementation detail into outcomes; do not pass through reference-doc phrasing.
- When a restart is required, include: (a) why, (b) what changes, (c) the exact resume command. Reproduce the rationale from the provider reference doc.

## Examples

**Invocation:** `/keycard-deploy-app` — the skill gathers provider, app directory, zone context, and fly org, checks which primitives exist, presents a plan showing credential status and what will be created, then executes after confirmation.

**From a fresh scaffold:** after `/keycard-template-app` sets up the local app, run `/keycard-deploy-app` to ship it to production. The plan will show the Application as already existing (created during scaffolding) and flag only the deploy-specific primitives (OIDC provider, deployed Resource, Application Credential) for creation.
