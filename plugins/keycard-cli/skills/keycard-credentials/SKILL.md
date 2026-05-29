---
name: keycard-credentials
description: |
  Shows what credentials are configured in this Keycard session — which services are available and what they provide access to.

  TRIGGER when: user asks what credentials, tokens, or services are available in the current Keycard session ("what tokens do I have", "what services are available", "am I authenticated", "list my credentials", "do I have access to X", "am I signed in to X", "is my X token loaded").
  DO NOT TRIGGER when: user wants to add, remove, or rotate credentials (use `keycard credential` commands directly).
license: Apache-2.0
metadata: {}
---

# keycard-credentials

Report what credentials are available in the current Keycard session.

## Step 1 — Read credential info

Run:
```bash
keycard credential info --json
```

If the command fails, display the returned error message and stop.

## Step 2 — Present the results

The command outputs one JSON object per line (JSONL): `{"env_var":"VAR_NAME","resource":"https://..."}`.

Present results under a `## Credentials` heading as a Markdown list:

- For each entry: `` `ENV_VAR` → `resource-URL` (service label) `` where `ENV_VAR` is the `env_var` field value.
- Derive the service label from the resource hostname (e.g. `api.github.com` → GitHub API). Omit the label if the hostname does not map to a recognizable service.
- If the output is empty, say so and suggest `keycard run -- <command>` to start a hydrated session.
- Never display or request credential values — identifiers only.

## Examples

**User asks:** "What credentials do I have?" / "Am I authenticated?" / "What services are available?"

**Sample output (credentials present):**
```
## Credentials
- `GITHUB_TOKEN` → `https://api.github.com` (GitHub API)
- `NPM_TOKEN` → `https://registry.npmjs.org` (npm Registry)
```

**Sample output (no credentials):**
```
No credentials found. Start a secure session with: `keycard run -- <your-command>`.
```

**Command failure (no configuration file):**

User asks "is my GitHub token loaded?" but no keycard.toml is found — sample output:
```
No keycard.toml found. Ensure keycard.toml exists and contains at least one [[credentials.default]] entry, then start a secure session with: `keycard run -- <your-command>`.
```
