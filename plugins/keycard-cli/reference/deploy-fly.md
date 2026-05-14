# Deploy to fly.io — Provider Reference

Provider-specific procedure for `keycard-deploy-app`. Authoritative for the fly provider.

| Field | Value |
|---|---|
| CLI tool | `flyctl` (aliased as `fly` — same binary, either name works) |
| Install | `curl -L https://fly.io/install.sh \| sh` |
| Resource URI | `https://api.machines.dev` |
| Credential env var | `FLY_API_TOKEN` |
| OIDC issuer template | `https://oidc.fly.io/<fly-org-slug>` |

> **`fly` vs `flyctl`** — fly.io's installer creates two binaries that point at the same executable: `flyctl` (the original name) and `fly` (the newer short alias). Every command in this doc is written as `flyctl <subcommand>` for consistency, but `fly <subcommand>` is equivalent and you may see it in fly.io's own documentation. Use whichever is on `PATH`; do not treat the absence of one name as a missing tool if the other is present.

## Provider arguments

| Position | Name | Required | Notes |
|---|---|---|---|
| 1 | `<fly-org-slug>` | yes | Fly organization slug. |

When `<fly-org-slug>` is not supplied as a positional arg, enumerate the user's orgs with `flyctl orgs list` (read-only) and prompt:

- **Zero orgs** — stop and instruct the user to create or join a fly.io organization at <https://fly.io/dashboard>.
- **One org** — present that single slug and ask the user to confirm before continuing.
- **Multiple orgs** — list every slug from the command output and ask the user to choose.

Never select an org silently, and never seed the candidate set from documentation examples — only from `flyctl orgs list` output.

## §1 — Credential pre-flight

**Tell the user:** "Checking that the fly.io CLI can talk to your account — this confirms the deploy credential is already wired into your session."

Probe with `flyctl auth whoami`.

- Exit 0 → credential is wired; skip to §3.
- Non-zero → no credential is wired; continue to §2.

Never read or echo `FLY_API_TOKEN`. `flyctl auth whoami` is the only probe.

## §2 — Configure fly.io credential

The fly.io deploy credential is not yet available. Present the user with two options:

**Tell the user:**

> The fly.io CLI needs a credential to deploy. There are two ways to set this up:
>
> 1. **Keycard Vault (recommended)** — I'll create a vault resource in Keycard and give you a link to paste your fly.io token. The token is encrypted at rest, never touches your machine, and is brokered per-session.
> 2. **Manual login** — Run `flyctl auth login` in a separate terminal to authenticate the fly CLI directly. This stores a credential on your machine outside of Keycard.
>
> Which would you prefer?

Wait for the user to choose before proceeding.

### Option A — Vaulted credential (recommended)

Fly.io doesn't support federated token exchange, so Keycard can't mint per-request tokens the way it does for services like Linear or GitHub. Instead, the fly.io deploy token is stored in Keycard as a **vaulted static credential** — Keycard brokers access to the token for the duration of the agent session, and no persistent credentials are stored on the user's machine.

**Tell the user:** "Setting up the vault resource in Keycard now — you'll just need to paste your fly.io token into the Keycard Console once."

`FLY_API_TOKEN` is consumed by `flyctl`, which the **agent** runs — it belongs in the **session** `keycard.toml` (the one at the session root that `keycard run -- claude` reads at startup). Do **not** add it to `<app-dir>/keycard.toml` — that file ships with the deployed app, which has no use for a deploy-time token.

#### Resolve org and zone identifiers for console URL

The skill needs both the `id` and `identifier` (slug) fields for orgs and zones. The `id` is used for API calls; the `identifier` is used to construct the console URL.

1. List organizations: `GET /organizations --org <org-id>`. From the response, find the org matching `<org-id>` and extract its `identifier` as `<org-identifier>`.
2. List zones: `GET /zones --org <org-id>`. From the response, find the zone matching `<zone-id>` and extract its `identifier` as `<zone-identifier>`.

Carry `<org-identifier>` and `<zone-identifier>` forward.

#### Look up the zone Vault provider

1. List providers: `GET /zones/<zone-id>/providers --org <org-id>`.
2. Filter for `type == "keycard-vault"`.
3. Carry the matching provider's `id` forward as `<vault-provider-id>`.
4. If no `keycard-vault` provider exists in the zone, stop and instruct the user to verify their zone configuration at `https://console.keycard.ai`.

#### Idempotent lookup-then-create the fly.io resource

Follow the [idempotency pattern](keycard-management-api.md#idempotency):

1. List resources: `GET /zones/<zone-id>/resources --org <org-id>`.
2. Filter for `identifier == "https://api.machines.dev"`.
3. If found, reuse its `id` as `<fly-resource-id>`. Surface: "Found existing fly.io resource (`<fly-resource-id>`) — already registered."
4. If not found, create:

```bash
keycard agent api /zones/<zone-id>/resources \
  -X POST \
  -d '{"name":"https://api.machines.dev","identifier":"https://api.machines.dev","credential_provider_id":"<vault-provider-id>"}' \
  --org <org-id>
```

5. On 409, re-list and reuse.
6. Extract the returned `id` as `<fly-resource-id>`.
7. Surface: "Created fly.io deploy resource (`<fly-resource-id>`) backed by your zone's Vault provider."

#### Wire credential in session keycard.toml

From the **session-root working directory** (not from inside `<app-dir>`), delegate to `/keycard-upsert-config` to add the credentials entry:

```
/keycard-upsert-config add a credentials entry: env_var = "FLY_API_TOKEN", resource = "https://api.machines.dev"
```

Then read the active policy with `keycard agent policy`. If it would block `flyctl` invocations through `Tool::"bash"`, delegate to `/keycard-upsert-policy` to allow them.

#### Direct user to add the token

Construct the console credentials URL:

```
https://console.keycard.ai/orgs/<org-identifier>/zones/<zone-identifier>/resources/<fly-resource-id>/credentials
```

**Tell the user** (verbatim, substituting values):

> The vault resource is set up. Now you need to store your fly.io deploy token — this is a one-time step you do in the browser, so the token never passes through me.
>
> 1. Create a deploy token scoped to your fly.io org at https://fly.io/docs/security/tokens/
> 2. Open <constructed-console-url> and paste it as the API token under **Credentials**
>
> ⚠ Never paste credentials into an agent conversation. The Keycard Console keeps secrets off the LLM path entirely.

Wait for the user to confirm they've added the token before proceeding.

#### Session restart

After wiring the credential, the agent session must be restarted. Print the following rationale block verbatim (substituting `<session-root-dir>`, `<app-dir>`, and `<fly-org-slug>`), then create `PROGRESS.md` with the same content:

```
The new FLY_API_TOKEN credential entry was added to your session keycard.toml,
but the running agent session was started before that entry existed.

Why a restart is needed:
  `keycard run -- claude` reads keycard.toml ONCE at startup and injects the
  declared credentials as env vars into the agent process. New entries added
  mid-session are not injected until you start a fresh session.

What changes after restart:
  - FLY_API_TOKEN will be present in this agent's environment.
  - `flyctl` (which the deploy skill invokes) will pick it up automatically
    from the process env — no manual export needed.
  - `keycard credential info` will list FLY_API_TOKEN alongside the existing
    entries, confirming the injection.

To resume:
  1. Exit this session (Ctrl-D or /exit).
  2. From <session-root-dir>, run:
       keycard run -- claude
  3. Re-invoke the deploy skill:
       /keycard-deploy-app <app-dir> <fly-org-slug>
     The skill is idempotent — it will detect FLY_API_TOKEN is already
     wired and skip to §3 (Deploy artifacts).
```

Stop here — do not proceed to §3 until the session has been restarted and `flyctl auth whoami` succeeds in the new session.

### Option B — Manual login

**Tell the user:** "You can log in to fly.io directly from your terminal. This stores the credential on your machine rather than in Keycard."

Instruct the user to run `flyctl auth login` in a **separate terminal** (not inside the agent session). This opens a browser-based OAuth flow.

After the user confirms they've logged in, re-probe with `flyctl auth whoami`:

- Exit 0 → credential is now available; skip to §3. No session restart is needed since `flyctl` reads its own config file, not a Keycard-injected env var.
- Non-zero → login did not succeed. Ask the user to retry or switch to Option A.

Read the active policy with `keycard agent policy`. If it would block `flyctl` invocations through `Tool::"bash"`, delegate to `/keycard-upsert-policy` to allow them.

## §3 — Deploy artifacts

**Tell the user:** "Setting up the deployment configuration — this tells fly.io how to build and run your app."

### Port alignment

Before generating or validating deploy artifacts, determine the app's expected port:

1. Read `PORT` from `<app-dir>/.env` if it exists.
2. Otherwise read `<app-dir>/keycard.toml` for a port field.
3. Otherwise check `<app-dir>/package.json` scripts or source for a default port constant.
4. Fallback: `8080`.

Carry this value forward as `<port>`.

### Generate or validate fly.toml

If `./<app-dir>/fly.toml` exists, treat it as authoritative. Validate that:
- `[http_service].internal_port` matches `<port>`. If it doesn't, patch it and surface: `⚠ Aligned fly.toml internal_port=<port> with the server's PORT`.
- `[env].KEYCARD_URL` is set (see below). If missing, add it.

If `fly.toml` does not exist, run `flyctl launch --no-deploy --copy-config --ha=false` with the project name and `<fly-org-slug>`. After generation:
- Patch `[http_service].internal_port` to `<port>` if the generated value differs.
- Add `[env].KEYCARD_URL` (see below).

Surface generated files (`fly.toml`, `Dockerfile`, `.dockerignore`) and require user confirmation before continuing. Never overwrite an existing `Dockerfile` without explicit confirmation.

If `fly.toml` declares an `org` that disagrees with `<fly-org-slug>`, stop and ask the user which is correct.

### Single-machine default

By default, deploy to a **single Fly machine** (no high-availability redundancy). This keeps costs minimal and is appropriate for most Keycard demo and dev deployments.

Ensure `fly.toml` contains:

```toml
[[vm]]
  size = 'shared-cpu-1x'
  memory = 256
```

If `fly.toml` already has a `[[vm]]` section, leave it as-is. If it does not, add the block above.

**Override:** only deploy with multiple machines / HA if the user's prompt explicitly requests it (e.g. "deploy with HA", "deploy two machines", "enable high availability"). In that case, omit the `--ha=false` flag from `flyctl launch` and `flyctl deploy`, and do not add the `[[vm]]` section.

### Pre-write KEYCARD_URL

The deployed app needs `KEYCARD_URL` to know which Keycard zone to trust. Derive it using the same env-domain table from `keycard-deploy-app` Step 2:

| `KEYCARD_ENV` | Domain |
|---|---|
| `production` (default) | `keycard.cloud` |
| `staging` | `keycard-stage.cloud` |
| `dev` | `keycard-dev.cloud` |
| `localdev` | `localdev.keycard.sh` |

Write `KEYCARD_URL = 'https://<zone-id>.<domain>'` into the `[env]` section of `fly.toml`. If `[env]` doesn't exist, create it before `[http_service]`.

**Tell the user:** "Added KEYCARD_URL to fly.toml — this tells your deployed app which Keycard zone to trust, so it only accepts credentials issued by your zone."

## §4 — Register fly OIDC provider in Keycard

**Tell the user:** "Now telling Keycard to trust identities from your fly.io org — this is what lets your deployed app prove its identity to Keycard without any API keys baked in."

The OIDC issuer is `https://oidc.fly.io/<fly-org-slug>`.

### Idempotent lookup-then-create

Follow the [idempotency pattern](keycard-management-api.md#idempotency):

1. List existing providers: `GET /zones/<zone-id>/providers --org <org-id>`.
2. Filter for any provider whose `protocols.oauth2.issuer` matches `https://oidc.fly.io/<fly-org-slug>`.
3. If found, reuse its `id` as `<provider-id>`. Surface: "Found existing Provider `fly-<fly-org-slug>` (`<provider-id>`) — reusing it."
4. If not found, create:

```bash
keycard agent api /zones/<zone-id>/providers \
  -X POST \
  -d '{"name":"fly-<fly-org-slug>","identifier":"fly-<fly-org-slug>","protocols":{"oauth2":{"issuer":"https://oidc.fly.io/<fly-org-slug>"}}}' \
  --org <org-id>
```

5. On 409 (race condition), re-list and reuse.
6. Surface: "Registered Provider `fly-<fly-org-slug>` (`<provider-id>`) — Keycard now trusts identities from your fly.io org."

Carry `<provider-id>` forward.

## §5 — Look up or create the Application

**Tell the user:** "Looking up your app in Keycard — it was registered when you scaffolded, so I'll use the existing one."

### Idempotent lookup-then-create

1. List applications: `GET /zones/<zone-id>/applications --org <org-id>`.
2. Filter for `identifier == <project-name>`.
3. If found, reuse its `id` as `<application-id>`. Surface: "Found Application `<project-name>` (`<application-id>`)."
4. If not found, create:

```bash
keycard agent api /zones/<zone-id>/applications \
  -X POST \
  -d '{"name":"<project-name>","identifier":"<project-name>"}' \
  --org <org-id>
```

5. On 409, re-list and reuse.
6. Surface: "Created Application `<project-name>` (`<application-id>`)."

Carry `<application-id>` forward.

## §5b — Resource reconciliation (deployed URL)

**Tell the user:** "Registering the deployed URL as a Resource in Keycard — your app will have two Resources side-by-side: one for local dev and one for production. Both share the same Application identity."

The deployed app needs its own Resource so Keycard can scope credentials to the production URL. The local-dev Resource (e.g. `http://localhost:<port>/mcp`) created by `keycard-template-app` stays in place — it's still valid for local development.

### Determine the deployed resource identifier

The deployed identifier is `https://<project-name>.fly.dev/<path>` where `<path>` comes from:
1. The template's `SPEC.md` resource path (e.g. `/mcp`) if `<app-dir>/SPEC.md` exists and declares one.
2. Otherwise `/` as the default.

### Look up the zone STS provider

The deployed resource needs a `credential_provider_id` so Keycard's STS can mint OAuth tokens for it. Without it, credential requests fail with "missing a credential provider".

1. List providers: `GET /zones/<zone-id>/providers --org <org-id>`.
2. Filter for `type == "keycard-sts"`.
3. Carry the matching provider's `id` forward as `<sts-provider-id>`.
4. If no `keycard-sts` provider exists in the zone, stop and instruct the user to verify their zone configuration at `https://console.keycard.ai`.

### Idempotent lookup-then-create

1. List resources provided by the application: `GET /zones/<zone-id>/applications/<application-id>/resources --org <org-id>`.
2. Check if any returned resource has `identifier` matching the deployed URL.
3. If found, reuse its `id` as `<deployed-resource-id>`. Surface: "Found existing deployed Resource (`<deployed-resource-id>`) — already registered."
4. If not found, create the resource:

```bash
keycard agent api /zones/<zone-id>/resources \
  -X POST \
  -d '{"name":"<project-name>-prod","identifier":"https://<project-name>.fly.dev/<path>","application_id":"<application-id>","credential_provider_id":"<sts-provider-id>","scopes":["mcp:tools"]}' \
  --org <org-id>
```

5. On 409, list zone resources, filter by identifier, and reuse.
6. If the newly created resource is not yet bound to the application (i.e. it was created without `application_id` or the binding is missing), bind it:

```bash
keycard agent api /zones/<zone-id>/applications/<application-id>/dependencies/<deployed-resource-id> \
  -X PUT \
  --org <org-id>
```

7. Surface what exists now: "Your app now has two Resources — `http://localhost:<port>/<path>` for local dev and `https://<project-name>.fly.dev/<path>` for production. Both share Application `<project-name>`."

Carry `<deployed-resource-id>` forward.

## §6 — Configure the Application Credential

**Tell the user:** "Setting up the trust binding — this tells Keycard that when your fly app identifies itself with your org's identity, it's allowed to mint credentials."

### Idempotent lookup-then-create

1. List application-credentials: `GET /zones/<zone-id>/application-credentials --org <org-id>`.
2. Filter for the triplet: `application_id == <application-id>` AND `provider_id == <provider-id>` AND `subject == <fly-org-slug>:<project-name>:*`.
3. If found, reuse. Surface: "Application Credential already exists (`<credential-id>`) — reusing."
4. If not found, create:

```bash
keycard agent api /zones/<zone-id>/application-credentials \
  -X POST \
  -d '{"application_id":"<application-id>","provider_id":"<provider-id>","type":"token","subject":"<fly-org-slug>:<project-name>:*"}' \
  --org <org-id>
```

5. On 409, re-list and reuse.
6. Surface: "Configured Application Credential (`<credential-id>`) — your fly app can now mint credentials in your zone."

The `subject` field matches the `sub` claim of the fly-issued identity token. Do **not** omit it — the API falls back to `*` (accept any token from this provider), which is almost never desired.

Carry `<credential-id>` forward.

## §7 — Deploy

**Tell the user:** "Shipping to fly.io — this builds a container image remotely and deploys it to your fly org."

Run `flyctl deploy --remote-only --ha=false` from `<app-dir>`.

If the user explicitly requested HA / multi-machine deployment, omit `--ha=false`.

On failure: surface the verbatim flyctl output, run `flyctl logs` for triage, and stop. Do **not** read `.env` or echo secret values.

## §8 — Verify

**Tell the user:** "Verifying the deploy — checking the app is running and Keycard trust is configured correctly."

### App status

- `flyctl status` — confirm the app is running.

### Trust chain verification

Health-check the well-known endpoints against `https://<project-name>.fly.dev`:

1. `curl -sf https://<project-name>.fly.dev/.well-known/oauth-protected-resource` — assert it returns JSON with a `resource` field that equals the deployed Resource identifier (`https://<project-name>.fly.dev/<path>`). If it returns the localhost URL instead, the app's `KEYCARD_URL` is misconfigured or the resource identifier in Keycard doesn't match the deployed URL. Surface:
   > ⚠ The deployed app is advertising `<actual-resource>` as its resource identifier, but we registered `https://<project-name>.fly.dev/<path>`. Check that `KEYCARD_URL` in fly.toml matches your zone URL and that the resource identifier registered in Keycard matches the deployed URL.

2. `curl -sf https://<project-name>.fly.dev/.well-known/oauth-authorization-server` — confirm it responds (exact shape depends on the template).

### Additional template-specific checks

If the template's `SPEC.md` defines additional verification endpoints, check them here.

### On failure

If verification fails, suggest checking:
- Deployed `KEYCARD_URL` in fly.toml `[env]` — must match the zone URL.
- Provider issuer URL vs the fly org slug — must be `https://oidc.fly.io/<fly-org-slug>`.
- Resource identifier in Keycard vs the actual deployed URL.

Print a final summary of: deployed URL, Provider name + ID, Application + ID, deployed Resource identifier + ID, Application Credential subject + ID.
