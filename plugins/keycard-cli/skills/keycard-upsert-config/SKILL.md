---
name: keycard-upsert-config
description: |
  Set or change a field in keycard.toml — reads the current value and writes a targeted update.
  TRIGGER when: user wants to set or change a field in keycard.toml; wants to add or update a credentials entry.
  DO NOT TRIGGER when: user asks about Cedar policy rules (use `keycard-query-policy`); user asks what credentials are active in the current session (use `keycard-credentials`); user is asking a field question only with no intent to write (read `../../reference/keycard-config-fields.md` directly).
argument-hint: "[what to change, e.g. 'set my zone to dev-123' or 'add a GitHub credential entry for GITHUB_TOKEN']"
license: Apache-2.0
---

# keycard-upsert-config

You are helping the user set or change a field in the Keycard CLI configuration file.

For field definitions, precedence rules, and the credentials section format, see [`../../reference/keycard-config-fields.md`](../../reference/keycard-config-fields.md).

## How to edit the config

1. Read `keycard.toml`. If the file does not exist, start with an empty document — do not error.
2. Make a targeted change to only the relevant field. Do not write fields marked **env/flag only** — they are silently ignored in TOML. If the user requests a change to such a field, tell them: "The `<field>` field is env/flag only and is silently ignored in `keycard.toml`. Set it with `export <ENV_VAR>=<value>` instead." Then stop.
3. Write back the updated file.
4. Re-read the updated `keycard.toml` and show the user the modified section to confirm the change was applied.

### Example: set the zone

```toml
environment = "production"

[zone]
id = "my-zone-id"
```

### Example: env/flag only fields (do not write to TOML)

```toml
# WRONG — org.id is env/flag only; this line will be silently ignored
[org]
id = "my-org"
```

Set these via environment variable instead:

```bash
export ORG=my-org
```

## Invocation examples

```
/keycard-upsert-config set my zone to dev-zone-123
/keycard-upsert-config switch to staging
/keycard-upsert-config add a credentials entry: env_var = "GITHUB_TOKEN", resource = "https://api.github.com"
```
