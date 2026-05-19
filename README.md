# Keycard Plugins

Agent harness plugins for the [Keycard CLI](https://keycard.ai). Each plugin wires Keycard's policy enforcement, credential injection, and management API into an AI agent runtime.

## Plugins

### `keycard-cli`

Integrates the Keycard CLI with [Claude Code](https://claude.ai/code).

**What it does:**

- **SessionStart hook** — warns at session start if the session was not launched via `keycard run`, so you know policy enforcement and credential injection are inactive.
- **PreToolUse hook** — runs `keycard agent hook --claude` before every tool call to enforce the active Cedar policy. Blocked tools are stopped before execution.
- **UserPromptSubmit hook** — runs `keycard agent hook --claude` on every prompt submission for prompt-level policy checks.

**Skills included:**

| Skill | Purpose |
|---|---|
| `/keycard-credentials` | Show what credentials (tokens, services) are active in the current session |
| `/keycard-discover-entities` | Discover available credential entities and MCP servers via the Management API and register them |
| `/keycard-query-policy` | Read and explain the active Cedar policy; diagnose why a tool was blocked |
| `/keycard-upsert-policy` | Propose, confirm, and apply a Cedar policy change |
| `/keycard-upsert-config` | Set or update a field in `keycard.toml` |
| `/keycard-upsert-mcp-config` | Add or update an MCP server entry in `.mcp.json` |

## Prerequisites

- [Keycard CLI](https://keycard.ai) installed and on your `$PATH`
- Claude Code (CLI or desktop app)

## Installation

Add the Keycard marketplace to Claude Code, then install the plugin:

```bash
claude plugin marketplace add keycardai/plugins
claude plugin install keycard-cli@keycardai
```

## Usage

Start a Claude Code session through Keycard to activate policy enforcement and credential injection:

```bash
keycard run -- claude
```

Without `keycard run`, the hooks are installed but Keycard is not managing the session — the SessionStart hook will remind you.

Once in a session, just ask naturally. For example:

- "What credentials do I have in this session?"
- "What tools am I allowed to use?"
- "Why was that tool blocked?"
- "Allow the Bash tool."
- "Require approval before WebFetch runs."
- "Set my zone to my-zone-id."
- "I need GitHub credentials."

## Repository structure

```
plugins/
  keycard-cli/
    .claude-plugin/
      plugin.json          # hooks and plugin metadata
    reference/
      cedar-policy.md      # Cedar policy syntax reference
      keycard-config-fields.md  # keycard.toml field reference
      keycard-management-api.md # Management API endpoint reference
    skills/
      keycard-credentials/
      keycard-discover-entities/
      keycard-query-policy/
      keycard-upsert-config/
      keycard-upsert-mcp-config/
      keycard-upsert-policy/
.claude-plugin/
  marketplace.json         # marketplace metadata for this repo
```

## License

Apache-2.0 — see [LICENSE](./LICENSE).
