---
name: keycard-credentials
description: Show what credentials are configured in this Keycard session — which services are available and what they provide access to
---

# keycard-credentials

You are helping the user understand what credentials are available in their current Keycard session.

## Step 1 — Read credential info

Run:
```bash
keycard credential info --json
```

If the command fails (no template found), display the returned message, then stop.

## Step 2 — Present the session context

The command outputs one JSON object per line (JSONL): `{"env":"VAR_NAME","resource":"https://..."}`.

Display the results in a way that helps the user and agent understand what services are accessible:

- For each entry, show the env var name and the resource it provides access to (e.g. `GITHUB_TOKEN → https://api.github.com` → GitHub API)
- If the output is empty (no lines), say so clearly and suggest running `keycard run -- <command>` to start a hydrated session
- Do not display or request credential values — identifiers only
