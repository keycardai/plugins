---
name: keycard-template-app
description: |
  Scaffold a Keycard-aware app from a blueprint in `keycardai/templates` and execute the template's `SPEC.md` to provision Keycard primitives, configure the project, smoke-test, and hand off. 
  TRIGGER when: user wants to scaffold, template, or bootstrap a new Keycard-aware app from a blueprint.
  DO NOT TRIGGER when: user wants to register a credential for an existing service (use `keycard-discover-entities`); user wants to inspect what is already attached to the session (use `keycard-credentials`); user wants to deploy or modify an existing app (out of scope).
argument-hint: "[template] [project-name]  (e.g. 'mcp-server-typescript-express my-mcp-server' — either may be omitted; the skill will prompt)"
license: Apache-2.0
metadata: {}
---

# keycard-template-app

Scaffold a Keycard-aware app from a blueprint published in [`keycardai/templates`](https://github.com/keycardai/templates). The skill is a generic interpreter — it picks a template directory, copies it, fills in the project-name + env placeholders, and runs the Keycard wiring (Application/Resource registration, smoke-test, handoff).

The catalog of available templates is **the templates repo itself**. The skill maintains a local cache clone, refreshes it when stale, and enumerates templates by listing top-level directories that contain a `SPEC.md`. Each template's own `SPEC.md` is **authoritative** for the provisioning steps the agent must run; this skill only prescribes the generic shape.

## Pre-flight rules (read before doing anything)

- **Parse `$ARGUMENTS` first.** The skill takes up to two positional args: `[template] [project-name]`. Resolve template (Step 1) and project name (Step 2) from `$ARGUMENTS` before doing anything else; prompt the user for whichever is missing. Do not silently fall back to defaults.
- **Stay in the user's working directory.** The skill scaffolds into `./<name>` relative to wherever the user invoked it; never `cd` somewhere else first.
- **`SPEC.md` is for the agent, not the user.** Every template ships a `SPEC.md` describing primitives, edits, ports, smoke tests, and handoff steps. Read it; reason from it; never quote it. The skill is the single renderer of all user-facing narration. If a template's wording is jargon-heavy, terse, or implementation-focused, paraphrase it into the skill's voice (per the narration-style rules below) before showing anything to the user. The user should never be able to tell whether a sentence came from the skill or from a particular template's SPEC.

### Narration style (applies to every step)

This skill is many users' first hands-on contact with Keycard. Treat the run as a guided tour, not a CI job:

- **Open with a one-line plan.** Before Step 0, say what you're about to do end-to-end: "I'm going to grab a starter blueprint, copy it into a new folder for you, register it with your Keycard zone so it can issue real credentials, smoke-test it locally, and then hand it back for you to run." Keep it under ~40 words.
- **Narrate each step in plain English** before you run the commands. One short sentence is enough — "Copying the blueprint into `./<name>` now." — but never let a long-running command appear out of nowhere.
- **No protocol or acronym jargon by default.** Don't drop terms like *OIDC issuer*, *OAuth*, *STS*, *RFC 8693*, *token exchange*, *MCP proxy*, *JWT*, *PKCE*, *DPoP*, *IdP*, *bearer token*, *audience claim*, *401/403* error codes, etc. into user-facing narration. Translate to outcomes instead — "so your app trusts credentials from your zone and nowhere else", "so Keycard can hand your app a short-lived credential for Linear without you pasting an API key", "Keycard is calling Linear on your behalf". The Keycard concepts in the bullet list below (zone, org, application, resource, provider) are the only domain terms you should introduce, and only the first time they appear. Acronym-heavy detail is fine **only** when (a) the user explicitly asks "how does this work under the hood?" / "what protocol does it use?" / similar, or (b) you're surfacing a verbatim error message from a tool. The same rule applies to anything you read out of a template's `SPEC.md` — paraphrase its description into plain language before showing it to the user; don't pass through phrasing like "MCP proxy that calls Linear's MCP server on your behalf, with Keycard brokering a Linear-scoped OAuth token via RFC 8693 token exchange".
- **Introduce each Keycard concept the first time it appears, with a docs link.** Use a single sentence + a link the user can open in a side tab while you keep working. Don't re-explain a concept on later mentions. The concepts that show up in this skill, with their canonical docs:
  - **Zone** — your private Keycard environment that holds users, apps, and policies and issues credentials. https://docs.keycard.ai/platform/concepts/zones/
  - **Organization** — the account that owns one or more zones (usually your company or team). https://docs.keycard.ai/platform/concepts/zones/
  - **Application** — the thing you're building, registered with Keycard so it can request credentials on a user's behalf or operate with its own identity. https://docs.keycard.ai/platform/concepts/applications/
  - **Resource** — the protected endpoint the application calls (e.g. your MCP server's `/mcp` URL). https://docs.keycard.ai/platform/concepts/resources/
  - **Provider** — where the credentials your app receives actually come from (Keycard itself can mint them, or pull a stored secret from Keycard Vault, or hand off to an external identity provider). https://docs.keycard.ai/platform/concepts/providers/
  - **Zone URL** — the base URL of your zone (`https://<zone-id>.<env-domain>`); your app uses it to confirm that credentials it receives really came from your Keycard zone. https://docs.keycard.ai/platform/architecture/standards-and-protocols/
- **Surface what just happened, not just what's next.** After each registration or file write, name the artifact and (where useful) the ID. "Registered your app as Application `app_…` in zone `<zone-id>`."
- **No raw command dumps without context.** If you have to print a command the user could run later, say what it does first.
- **Stay calm under failure.** When something fails, fall back to the same friendly framing as the error paths in Step 3 — explain the concept, link the docs, and offer the next concrete action.

### Resume protocol — `PROGRESS.md`

Things will fail mid-run: a build error, a 4xx from the Management API, a port collision the second time, a `keycard auth signin` the user has to step away to do. The skill must be **resumable** — when re-invoked it should pick up where it left off rather than start over and create duplicates.

**File location.** Once Step 4 has copied the template, write a `PROGRESS.md` at `./<name>/PROGRESS.md`. This is the single source of truth for resume state. Steps 1–3 (template choice, project name, zone/org resolution) are cheap to re-derive, so they don't need their own progress file before the project directory exists; just re-prompt on resume if `./<name>` doesn't exist yet.

**File shape.** Plain Markdown so a human can read or edit it. Include, in this order:

```markdown
# Keycard template-app progress

- Skill version / TEMPLATES_REF / cached SHA: <values>
- Template: <template-dir>
- Project: <name> (at <absolute path>)
- Zone: <zone-id>
- Org: <org-id>
- KEYCARD_URL: <derived URL>
- Port: <port>
- Started: <ISO-8601 timestamp>
- Last updated: <ISO-8601 timestamp>

## Steps

- [x] Step 4 — Copied template
- [x] Step 5 — Read SPEC.md
- [x] Step 6 — Filled in <files>; built with <command>
- [ ] Step 7 — Provision Keycard primitives
  - [x] Provider lookup → <provider-id>
  - [x] Application created → <application-id>
  - [ ] Resource create — FAILED at <ISO-8601>: <verbatim error>
- [ ] Step 8 — Smoke-test
- [ ] Step 9 — Handoff

## Notes

<free-form: anything the agent or user wants to remember between runs>
```

**Update cadence.** Append/replace `PROGRESS.md` after every meaningful action: each registered ID (Step 7), the smoke-test result (Step 8), and the handoff (Step 9). Always bump the `Last updated` timestamp. Never let the file disagree with reality on disk.

**On resume — always verify with the user before continuing.** When the skill starts and finds a `PROGRESS.md` at `./<arg>/PROGRESS.md` (or `./<name>/PROGRESS.md` derived from arguments), do **not** silently continue. The file may have been edited, copied from another project, or left over from a different template. Read it, summarize it back to the user in plain English, and ask:

> I found a `PROGRESS.md` in `./<name>` from a previous run of this skill (started <when>, last updated <when>). It says:
>
> - Template: `<template-dir>` from `<TEMPLATES_REF>`
> - Zone `<zone-id>` / org `<org-id>` / port `<port>`
> - Done: <human summary of completed steps and IDs>
> - Stopped at: <step + reason from the file>
>
> Want me to **resume from where it stopped**, **start over from scratch** (I'll move the old `PROGRESS.md` aside as `PROGRESS.md.bak-<timestamp>` and keep the rest of the directory), or **abandon** (do nothing)?

Only proceed in resume mode after explicit confirmation. If the user says start over, never delete the user's code — only rename the progress file.

**Sanity-check the file before trusting it.** If `PROGRESS.md` claims an Application ID exists, look it up via `keycard agent api …` to confirm it's real before reusing it. If it's gone (zone changed, org changed, manually deleted in the console), warn the user, drop that line from the in-memory state, and re-create it. The same goes for paths and ports — re-probe `<port>` per Step 6's rules before relaunching the smoke-test.

**On clean success.** After Step 9's handoff prints, leave `PROGRESS.md` in place but mark every step `[x]` and add a final `Completed: <timestamp>` line. The user can delete it; the skill will re-prompt and re-verify on any future invocation.

**Tell the user first:** something like "Pulling the latest starter blueprints from [keycardai/templates](https://github.com/keycardai/templates) — these are pre-wired example apps maintained by Keycard, so we don't start from a blank file."

Maintain a single shared clone under the user cache directory and reuse it across invocations:

```bash
TEMPLATES_REPO="https://github.com/keycardai/templates"
TEMPLATES_REF="kamil/initial-template"   # tracks the published default until templates lands on main
CACHE_DIR="${XDG_CACHE_HOME:-$HOME/.cache}/keycard/templates"

if [ -d "$CACHE_DIR/.git" ]; then
  git -C "$CACHE_DIR" fetch --depth=1 origin "$TEMPLATES_REF" \
    && git -C "$CACHE_DIR" checkout -q "FETCH_HEAD"
else
  mkdir -p "$(dirname "$CACHE_DIR")"
  git clone --depth=1 --branch "$TEMPLATES_REF" "$TEMPLATES_REPO" "$CACHE_DIR"
fi
```

Rules:

- **Reuse the cache.** Do not re-clone on every invocation; a fetch + checkout is sufficient.
- **Tolerate offline.** If the fetch fails (network, auth) but `$CACHE_DIR/.git` exists, continue with the cached working tree and warn the user: `⚠ Could not refresh templates cache; using cached copy from <last-fetch-date>.`
- **Fail loud only when there is nothing.** If the cache is missing AND the clone fails, abort with the verbatim git error and: `Error: could not clone keycardai/templates. Confirm network access and that the ref "<TEMPLATES_REF>" exists.`
- **Do not commit, push, or modify** the cache working tree. It is a read-only source for copying.

Carry `$CACHE_DIR` and the resolved `<TEMPLATES_REF>` (or the cached commit SHA, if you want a stable receipt) forward into later steps.

## Step 1 — Resolve template

**Tell the user first:** if you have to prompt, frame it as "Each template is a working app you can run today — pick the shape that matches what you want to build, and I'll personalize it for you." Do not ask the user to pick by directory name alone; always show the one-line summary.

The template directory is the **first positional argument** in `$ARGUMENTS`. Treat it as a known template if `$CACHE_DIR/<arg>/SPEC.md` exists. If so, use it directly as `<template-dir>`.

Otherwise (argument missing, or no matching directory — in which case it is almost certainly the project name, see Step 2), enumerate the templates from the cache and present them to the user:

```bash
ls -1 "$CACHE_DIR" | while read -r d; do
  [ -f "$CACHE_DIR/$d/SPEC.md" ] || continue
  printf '%s\n' "$d"
done
```

For each candidate, read the first heading / one-liner from `$CACHE_DIR/<d>/SPEC.md` (or `README.md` if `SPEC.md` has no summary line) and present a numbered list:

```
Which template?
  1. mcp-server-typescript-express — <one-line summary from SPEC.md>
  2. mcp-brokered-credentials-typescript — <one-line summary from SPEC.md>
  …
```

Wait for the user to pick one. Do **not** silently default — even when there's only one entry, confirm it explicitly so the user knows what they're getting. Carry the chosen directory forward as `<template-dir>`.

## Step 2 — Resolve project name

The project name is the **second positional argument** in `$ARGUMENTS` (or the first, if the first turned out to be a project name rather than a template directory in Step 1). Kebab-case it and use it as `<name>`. Otherwise, ask the user:

```
What should the project be named? (kebab-case, e.g. <template-default-name>)
```

Read the suggested default (`<template-default-name>`) from the chosen template's `SPEC.md` if it provides one, otherwise fall back to `<template-dir>`.

Validate against `^[a-z][a-z0-9-]*$`. If the resulting `./<name>` directory already exists and is non-empty, ask the user whether to overwrite before continuing. Carry the chosen name forward as `<name>`.

## Step 3 — Resolve context

**Tell the user first:** something like "Now I need to figure out which **Keycard zone** and **organization** this app should live in — that's what lets Keycard issue real credentials for it. (Zone = your private Keycard environment; org = the account that owns it. More: https://docs.keycard.ai/platform/concepts/zones/)" Only say this once, on the first mention; later steps can refer to "your zone" without re-explaining.

The skill needs three values before scaffolding:

1. **Zone ID** — from `keycard.toml`, env, or API
2. **Org ID** — from env, `keycard.toml`, or API
3. **Zone URL** (`KEYCARD_URL`) — derived from zone ID + environment domain (the generated server uses this to verify that credentials it receives came from your zone)

`keycard run -- claude` only injects `KEYCARD_RUN=1`, `KEYCARD_RUN_SESSION_ID`, `KEYCARD_ENV`, `KEYCARD_SOCKET`, and the configured credential env vars into the subprocess. **It does not export `KEYCARD_URL` or `ORG`** — those must be resolved from local config. Do not abort just because `KEYCARD_URL` is empty.

Resolve these silently — read env vars by name (do NOT shell out to `env | grep …`) and read `./keycard.toml` directly with the file-read tool (do NOT shell out to `cat`/`ls`).

### Resolve zone ID (in priority order)

1. `ZONE` environment variable
2. `zone.id` field in `./keycard.toml` (or the path passed via `--config`).
3. If still missing and `KEYCARD_RUN=1` is set, run `keycard agent api /zones --org <org-id>` and ask the user which zone to use — frame it as "Which Keycard environment should this app live in?" rather than asking for a raw zone ID.
4. Otherwise stop and show the friendly explanation below — do NOT print a bare "cannot resolve zone ID" error.

When you have to stop, say something like:

> I need to know which **Keycard zone** to set this app up in before I can continue.
>
> A zone is your own private Keycard environment — it's where your users, apps, and access policies live, and it's what issues the credentials your app will use. Most people have one zone per project or per team. You can read more here: https://docs.keycard.ai/platform/concepts/zones/
>
> I couldn't find one yet. I'd run sign-in for you, but `keycard auth signin` opens a browser window and asks *you* to log in and pick a zone — that has to be a human action, I can't impersonate you against your identity provider. So please run `keycard auth signin` in your terminal; it'll save the zone to `keycard.toml` and I'll pick up automatically from there. (If you already know the zone ID and just want to skip sign-in, tell me and I'll write it into `keycard.toml` myself.)

### Resolve org ID (in priority order)

1. `ORG` environment variable
2. `org.id` field in `./keycard.toml`.
3. If still missing and `KEYCARD_RUN=1` is set, run `keycard agent api /organizations` and ask the user which to use — phrase it as "Which of your Keycard organizations should own this app?" and show the human-readable names from the API, not just IDs.
4. Otherwise stop with the friendly explanation below — do NOT print a bare "cannot resolve org ID" error.

When you have to stop, say something like:

> I also need to know which **Keycard organization** this app belongs to.
>
> An organization is the account your zone lives under — usually your company or team. The same org can own several zones (for example, one for staging and one for production). See https://docs.keycard.ai/platform/concepts/zones/ for how zones and orgs fit together.
>
> Same handoff reason as above — `keycard auth signin` is an interactive browser login that has to happen as you, not as me. Run it in your terminal and it'll save the org alongside the zone; I'll resume from there. (Or just tell me which organization to use and I'll write it into `keycard.toml` directly.)

### Derive zone URL

Map the active environment to the matching base domain, then build `https://<zone-id>.<domain>`:

| `KEYCARD_ENV` | Domain | Scheme |
|---|---|---|
| `production` (default if unset) | `keycard.cloud` | `https` |
| `staging` | `keycard-stage.cloud` | `https` |
| `dev` | `keycard-dev.cloud` | `https` |
| `localdev` | `localdev.keycard.sh` | `http` |

Example: zone ID `o36mbsre94s2vlt8x5jq6nbxs0` in production → `https://o36mbsre94s2vlt8x5jq6nbxs0.keycard.cloud`.

Carry zone-id, org-id, and zone-url forward into all later steps; they are referenced as `<zone-id>`, `<org-id>`, and `<KEYCARD_URL>` from here on.

### Active-session check

Only required for steps that talk to the Management API (step 7). If `KEYCARD_RUN` is unset and no API call has been attempted yet, you may still scaffold and build (steps 4–6) — just warn the user that registration in step 7 will require `keycard run -- claude` (or a manual `keycard auth signin`).

## Step 4 — Copy the template into the user's project

**Tell the user first:** "Copying the `<template-dir>` blueprint into `./<name>` — this is now your project; the cached copy stays untouched so re-running this skill is always safe."

Copy from the refreshed cache (Step 0) into the user's working directory under the project name. Do not modify the cache.

```bash
cp -R "$CACHE_DIR/<template-dir>" "./<name>"
```

Print a one-line confirmation: `✓ Copied <template-dir>@<TEMPLATES_REF> → ./<name>` (include the cached commit SHA if you captured it in Step 0).

## Step 5 — Read the template's SPEC.md

Each template ships a `SPEC.md` at `./<name>/SPEC.md` describing the Keycard primitives that **must** exist for it to work, plus any template-specific quirks (port, env vars, post-copy edits, smoke-test, handoff). It is the authoritative source for the *data* the skill needs from here on — but it is **not** user-facing copy. Never quote or echo SPEC.md prose to the user.

Read `./<name>/SPEC.md` end-to-end before continuing. If anything in this skill conflicts with the template's spec, follow the spec and surface the discrepancy in the final summary so it can be reconciled.

After reading, mentally extract these fields — later steps consume them directly and do not re-read or re-quote SPEC.md:

| Field | Where it feeds |
|---|---|
| Required primitives (providers, applications, resources, vault entries, dependencies) | Step 7 |
| Post-copy edits (files to write/patch, manifest fields to set) | Step 6 |
| Default port | Step 6 (port resolution) |
| Build / install command | Step 6 |
| Smoke-test endpoints + ready signal | Step 8 |
| Handoff commands (each: command, one-line reason, run-where: "their terminal" / "this session" / "new agent session") | Step 9 |

The remaining steps describe the **shape** of the work; substitute the extracted values at every step.

## Step 6 — Fill in the template requirements

**Tell the user first:** "Filling in the blanks the template left for us — your project name, a local port, and an `.env` with your zone URL so the app only trusts credentials issued by your zone. I'll also drop a `keycard.toml` next to it; that's the per-project config the Keycard CLI reads (which zone, which credentials, etc.)."

Apply the post-copy edits called for in `SPEC.md`. Common edits include:

- Writing an `.env` (and a committed `.env.example`) with resolved values such as `KEYCARD_URL`, `<port>`, etc. — `.env` is typically gitignored by the template.
- Setting the project name in language-native manifests (e.g. `package.json` `"name"`, `pyproject.toml` `[project].name`, `Cargo.toml` `[package].name`).
- Setting `[project].name = "<name>"` in `./<name>/keycard.toml`. Leave any `${…}` placeholders that interpolate from `.env` at runtime untouched. Leave `[credentials]` empty unless the user asks for a tool that calls a third-party API.

### Resolve a safe port

**Tell the user first (only if you have to substitute):** "The template's default port is busy on your machine, so I'm picking the next free one — your `.env` and the smoke-test will use that instead."

Before writing `<port>` into `.env`, check that the template's default port is actually free on the user's machine. Never assume — port collisions are the #1 reason smoke-tests fail.

1. Take the default port from the template's `SPEC.md` as `<candidate>`.
2. Probe whether anything is listening on `127.0.0.1:<candidate>` using a non-destructive check. Prefer, in order: a Go/Node/Python `net.Listen`-style probe if the toolchain is already in use, otherwise `lsof -iTCP:<candidate> -sTCP:LISTEN -n -P 2>/dev/null` (macOS/Linux) or `ss -ltn "sport = :<candidate>"` as a fallback. Do **not** `curl` or `nc` against an unknown port — that can wake up an unrelated service.
3. If the port is free, use it.
4. If the port is in use, scan upward from `<candidate>+1` for the first free port in a bounded window (cap at 20 attempts; do not wander into ephemeral-port territory above 49151). Surface the substitution to the user before continuing:

   ```
   ⚠ Port <candidate> is in use (PID <pid>, <command>). Using <chosen> instead.
   ```

5. If the entire window is exhausted, stop with: `Error: no free port in <candidate>..<candidate+20>. Free a port or pass --port explicitly.` — do **not** silently fall back to a random ephemeral port; the value is written to `.env` and referenced by the user later, so it must be predictable.
6. Carry the chosen value forward as `<port>` and use it in `.env`, the smoke-test URLs (Step 8), and the handoff summary (Step 9). If the template registers a Resource whose identifier embeds the port (e.g. `http://localhost:<port>/mcp`), use the chosen port there too — Step 7 must read `<port>` from this step, not from the template default.

Then run the build/install command listed in the template's `SPEC.md` (e.g. `npm install && npm run build`, `uv sync`, `cargo build`). Surface errors verbatim and do not proceed until the build succeeds.

## Step 7 — Provision Keycard primitives (per SPEC.md §1)

**Tell the user first:** explain — once, on the first registration — what's about to be created in their zone. Use roughly:

> Now I'm registering this project with your Keycard zone. That means creating a few records in Keycard so it knows your app exists and can mint credentials for it:
>
> - an **Application** — the identity for the thing you're building, so Keycard can recognize it. https://docs.keycard.ai/platform/concepts/applications/
> - a **Resource** — the protected URL the application calls (your local server's `/mcp` endpoint, in this case), so Keycard knows what scope the credentials grant access to. https://docs.keycard.ai/platform/concepts/resources/
> - and we'll point them at a **Provider** — where the credentials your app receives actually come from (Keycard itself can mint them, or hand back a stored secret from Keycard Vault). https://docs.keycard.ai/platform/concepts/providers/
>
> All of these go into your zone, and re-running this skill updates them in place rather than creating duplicates.

After each create call returns, name what you got back: "✓ Application registered: `<application-id>`" — don't print the raw API response. When explaining *why* a primitive is being created, use the concept introductions from the narration-style bullet list above — never echo SPEC.md's wording for it.

`SPEC.md` lists the primitives the agent must create in the active zone — typically some combination of credential-provider lookup, Application, and Resource. Use `keycard agent api …` to create or look up each primitive in the order the spec dictates, scoped to `<zone-id>` and `<org-id>` resolved in Step 3.

General rules that apply regardless of template:

- **Idempotency.** Use stable identifiers (e.g. the kebab-case `<name>`) so re-running the skill updates rather than duplicates. On 409 ("already exists"), look the entity up and reuse its ID.
- **Provider selection.** When `SPEC.md` requires a credential provider, pick the type it specifies (e.g. `keycard-sts` when Keycard mints the credential itself, `keycard-vault` when it returns a stored downstream secret). If multiple match, ask the user. If none, abort with a `https://console.keycard.ai → Providers` pointer. (These provider type names are internal — don't surface them to the user; just say "Keycard will mint the credential" or "Keycard will return your stored secret".)
- **Field accuracy.** Match identifiers to the protected URL exactly when the spec says so — mismatches surface later as `invalid_target` or `Requested authorization for unknown resource ...` at token-mint time.
- **Failure handoff.** On 403, network error, or missing session, print: `Could not auto-register. Register manually at https://console.keycard.ai → <Section>.` and stop.

Carry every returned ID forward (`<application-id>`, `<resource-id>`, `<provider-id>`, …) — later steps and the final summary reference them.

## Step 8 — Smoke-test (per SPEC.md §3)

**Tell the user first:** "Booting the app locally for a few seconds just to confirm everything wired up — the app trusts your zone, the URL we registered matches what the server actually serves, and the endpoints respond. I'll shut it down again before handing it back to you so the port stays free."

If `SPEC.md` defines a smoke-test, run it from the current session **for verification only** and stop the process before the handoff. Typically this means starting the server in the background, waiting for its ready log line, and `curl`-ing the endpoints the spec lists.

### Port-safety pre-flight

Even though Step 6 picked a free `<port>`, re-verify it is still free immediately before launching — something else could have grabbed it between steps. Use the same probe as Step 6 (`lsof -iTCP:<port> -sTCP:LISTEN -n -P` etc.). If it is now busy, repeat the upward scan from Step 6 to pick a new `<port>`, rewrite `.env` with the new value, and warn the user — do **not** kill the existing listener.

### Launch and verify

- Capture the child's PID the moment you spawn it (e.g. `$!` in bash) so it can be killed deterministically. Do **not** rely on `pkill -f` against the binary name — that can take down unrelated user processes.
- Wait for the template's documented ready signal (log line, health endpoint) with a bounded timeout (default 30s). If the timeout elapses, kill the captured PID, dump the last ~50 lines of the server log verbatim, and stop.
- `curl` against `http://127.0.0.1:<port>/…` (not `localhost`) to avoid IPv6/IPv4 ambiguity surfacing as a false negative. Each endpoint must return the response shape `SPEC.md` specifies.
- If verification fails, surface the error verbatim and re-check the values written in Step 6 — most failures trace back to a missing or placeholder `.env` value (e.g. an unresolved `KEYCARD_URL`) or a mismatch between the port in `.env` and the URL you registered as the Resource.

### Always release the port

Once verification passes (or fails), terminate the smoke-test process by the captured PID, then re-probe `<port>` to confirm it is free again before printing the handoff. Trapping `EXIT`/`ERR` in the launch script is the safest pattern — the process must not survive a partial failure of this step.

Do **not** leave it running: it can hold the port and downstream agent harnesses (e.g. Claude Code) will not pick up servers added mid-session anyway.

When framing smoke-test results for the user, use the skill's own voice — do not quote SPEC.md's smoke-test section. Verbatim tool/server output in error messages is fine (per the existing "verbatim error" exception), but any surrounding explanation must be paraphrased.

If the template has no smoke-test, skip this step.

## Step 9 — Hand off to the user

The final step is **manual**. Most templates produce something the user must run in a separate terminal (a server, a worker) and may require restarting the agent harness so it discovers newly registered MCP servers or credentials. The skill must print clear, copy-pasteable instructions and stop — do not start long-running processes, do not attach to the live agent session, and do not block waiting for the user to come back.

If the extracted handoff data (Step 5) includes a config-write step that is safe from the current session (e.g. `claude mcp add`, writing to `~/.claude.json`), run it silently — these mutate config only and do not affect the live process. Anything that requires the user's own terminal or a fresh agent session goes into the printed block below.

### What to print — exactly one block

The skill renders **one** unified handoff block. Do not print a separate preliminary block, a "from SPEC.md" section, or any other pre-/post- handoff text. The block has three parts:

1. **Recap paragraph.** A couple of sentences in plain English summarizing the work. Reference the carried values (`<template-dir>`, `<name>`, build command, `<zone-id>`, registered IDs, `<port>`, verified endpoints) inline so the user can verify what was registered. Example opening:

   > **You're set up.** I started from the `<template-dir>` blueprint, dropped it into `./<name>`, and got it building cleanly with `<build command>`. On the Keycard side, your zone (`<zone-id>`) now has an Application (`<application-id>`) pointing at a Resource (`<resource-id>`) for `http://localhost:<port>/<path>`, and the smoke-test came back clean.

2. **"What's left for you" numbered list.** Each item is a verbatim, copy-pasteable command followed by a one-line reason **written in the skill's own voice** (no-jargon rule applies). Commands come from the extracted handoff data (Step 5); reasons are paraphrased by the skill, never pasted from SPEC.md. Example:

   > 1. `cd <name> && keycard run -- npm start` — starts your local server with the credentials Keycard manages for it
   > 2. `keycard run -- claude` (new terminal) — restart your agent so it picks up the new server
   > 3. `/mcp` → pick `<name>` → follow the login flow — connects your agent to the server (you'll see a consent screen the first time because this app depends on another service)

3. **Console pointer.** A one-liner: "If anything looks off, the registered IDs above are what to check in https://console.keycard.ai."

### Prohibitions

- Do **not** print a separate "Next steps" or "from SPEC.md" block before or after the unified block — that is the source of the duplicate-messaging bug this rule exists to prevent.
- Do **not** pass through SPEC.md phrasing. Translate implementation detail into outcomes: "the first time you'll see a consent screen because this app depends on Linear" instead of "transitive consent kicking in"; "Keycard hands the proxy a short-lived credential" instead of "RFC 8693 token exchange"; "starts your server with the credentials Keycard manages" instead of "brokers KEYCARD_CLIENT_ID and KEYCARD_CLIENT_SECRET from the zone vault".
- If a section has nothing to report (e.g. no smoke-test, no in-session config write), leave it out — don't print empty placeholders.

The goal is for the user to close the loop feeling like a teammate just walked them through the work, not like they're reading a CI log.

## Examples

**Invocation:**
```
/keycard-template-app                                              # prompts for both template and project name
/keycard-template-app mcp-server-typescript-express                # prompts for project name
/keycard-template-app mcp-server-typescript-express my-mcp-server  # fully specified, no prompts
/keycard-template-app my-mcp-server                                # first arg isn't a known template — treated as project name; prompts for template
```

**Deploying:** once the scaffold is working locally, run `/keycard-deploy-app` to ship it to a hosting provider (fly.io, etc.).

**Extending the scaffold after handoff:** follow the patterns the template ships (its `README.md`, example modules, etc.). Once handoff is printed, further changes are owned by the user, not this skill.
