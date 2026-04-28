---
name: keycard-discover-entities
description: |
  Discover and wire credential entities via the Keycard Management API — find available entity URIs and register them in keycard.toml.
  TRIGGER when: user says "I need X credentials", "add a resource", "add an X credential", "what entities are available in the Management API", "configure access to X service", "set up X integration".
  DO NOT TRIGGER when: the user already has credentials and wants to inspect them (use `keycard-credentials`); user wants to edit an existing config field that is not a credential entry (use `keycard-upsert-config`).
argument-hint: "[service or action, e.g. 'I need GitHub credentials' or 'list available entities']"
license: Apache-2.0
metadata: {}
---

# keycard-discover-entities

Discover and register credential entities via the Keycard Management API.

## Step 1 — Check for an existing credential

Run:
```bash
keycard credential info --json
```

If the command fails with "not signed in to management API", tell the user to run `keycard auth signin --org <org-id>` and stop. For any other failure, report the error and stop.

The output is JSONL: one `{"env_var":"VAR_NAME","resource":"https://..."}` object per line.

If any line's `resource` URI matches the target service (e.g. `https://api.github.com` for GitHub, `https://registry.npmjs.org` for npm), the credential is already configured — tell the user and stop.

## Step 2 — Discover available entities and register the credential

The user's request is: `$ARGUMENTS`

If `$ARGUMENTS` does not name a specific service (e.g. "what entities are available?"), walk the endpoint tree to list available entities, display them to the user, and stop — do not proceed to registration.

Parse `$ARGUMENTS` to identify the target service (e.g. "GitHub", "npm").

**Resolve org ID** — use the `ORG` environment variable if set. Otherwise run `keycard agent api /organizations` to list available orgs and ask the user which to use. This value is required before any further API calls.

**Discover available entities** by following the endpoint tree in [`.agents/reference/keycard-management-api.md`](.agents/reference/keycard-management-api.md).

Match the target service against the `resource` URIs in the results — do not walk the full tree interactively unless no match is found. If multiple entries match, display them and ask which to register before proceeding.

Identify the resource URI and appropriate env var name (e.g. `https://api.github.com` → `GITHUB_TOKEN`). Confirm the env var name with the user before writing — it will be injected into subprocesses and must not shadow an existing variable.

**Register the credential** by invoking `/keycard-upsert-config`:
```
/keycard-upsert-config add a credentials entry: env_var = "<ENV_VAR>", resource = "<resource-URI>"
```

**Only write `resource` and `env_var` fields. Do not generate scopes, store secrets, or handle tokens directly.**

## Step 3 — Verify

Run:
```bash
keycard credential info
```

Confirm the new entry appears in the output. If it does not, re-read `keycard.toml` via `/keycard-upsert-config` to check the write succeeded.

## Examples

**Invocation:**
```
/keycard-discover-entities I need GitHub credentials
/keycard-discover-entities what entities are available?
/keycard-discover-entities add an NPM registry credential
/keycard-discover-entities configure access to the GitHub API
```

**Sample output for `/keycard-discover-entities I need GitHub credentials`:**
```
No configured credentials found for GitHub.
Found `https://api.github.com (GitHub API)` resource.

Proposed addition to keycard.toml:

  ```toml
  [[credentials.default]]
  env_var = "GH_TOKEN"
  resource = "https://github.com"
  ```

Confirm? [y/N]

✓ GitHub credential configured, verified via `keycard credential info`.
```
