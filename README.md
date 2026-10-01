# Keycard Plugins

Agent harness plugins for the [Keycard CLI](https://keycard.ai). Each plugin wires Keycard's credential injection, tool-use auditing, and management API into an AI agent runtime.

## Plugins

### `keycard-cli`

Integrates the Keycard CLI with [Claude Code](https://claude.ai/code).

**What it does:**

- **SessionStart hook** — reminds you when the session was not started with `keycard run claude`, so the credentials declared in `keycard.toml` are not in its environment.
- **PreToolUse hook** — runs `keycard agent hook claude` before every tool call. It records the call to your Zone's audit log and, for shell commands, injects the session's credentials by running the command through `keycard agent exec`.
- **PostToolUse hook** — runs `keycard agent hook claude` after every tool call to redact injected credential values from the tool's output.
- **UserPromptSubmit hook** — runs `keycard agent hook claude` on every prompt submission to record the prompt to your Zone's audit log.

**Skills included:**

| Skill | Purpose |
|---|---|
| `/keycard-credentials` | Show what credentials (tokens, services) are active in the current session |
| `/keycard-upsert-config` | Set or update a field in `keycard.toml` |
| `/keycard-upsert-mcp-config` | Add or update an MCP server entry in `.mcp.json` |

## Prerequisites

- [Keycard CLI](https://keycard.ai) installed and on your `$PATH`
- Claude Code (CLI or desktop app)

## Installation

Add the Keycard marketplace to Claude Code, then install the plugin at project scope, so its hooks run only in projects that use Keycard:

```bash
claude plugin marketplace add keycardai/plugins
claude plugin install keycard-cli@keycardai -s project
```

`keycard run claude` offers the same two commands when the plugin is missing or out of date.

## Usage

Start a Claude Code session through Keycard to put the credentials declared in `keycard.toml` in its environment and connect it to your Zone's MCP gateways:

```bash
keycard run claude
```

Without `keycard run claude`, the hooks are installed but the session's credentials are not — the SessionStart hook will remind you.

Once in a session, just ask naturally. For example:

- "What credentials do I have in this session?"
- "Set my zone to my-zone-id."
- "I need GitHub credentials."

## Repository structure

```
plugins/
  keycard-cli/
    .claude-plugin/
      plugin.json          # hooks and plugin metadata
    reference/
      keycard-config-fields.md  # keycard.toml field reference
      keycard-management-api.md # Management API endpoint reference
    skills/
      keycard-credentials/
      keycard-upsert-config/
      keycard-upsert-mcp-config/
.claude-plugin/
  marketplace.json         # marketplace metadata for this repo
```

## License

Apache-2.0 — see [LICENSE](./LICENSE).
