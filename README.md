# Keycard Plugins

Agent harness plugins for the [Keycard CLI](https://keycard.ai). Each plugin wires Keycard's credential injection and tool-use auditing into an AI agent runtime.

## Plugins

### `keycard-cli`

Integrates the Keycard CLI with [Claude Code](https://claude.ai/code).

**What it does:**

- **SessionStart hook** — when the session was not started with `keycard run claude`, adds a note to Claude's context saying so, and that the credentials declared in `keycard.toml` are not in the session's environment. Such a session is not connected to your Zone's MCP gateways either.
- **PreToolUse hook** — runs `keycard agent hook claude` before every tool call. It records the call to your Zone's audit log and, for shell commands, injects the session's credentials by running the command through `keycard agent exec`.
- **PostToolUse hook** — runs `keycard agent hook claude` after every tool call. In a session started with `keycard run claude`, it redacts the session's credential values from the tool's output: where a value appeared, Claude sees a placeholder that names the resource, such as `<redacted+https://api.github.com>`. Outside `keycard run claude` the hook does not know the values that `keycard agent exec` mints for shell commands, so any that appear in a command's output reach Claude unredacted.
- **UserPromptSubmit hook** — runs `keycard agent hook claude` on every prompt submission to record the prompt to your Zone's audit log.

**Skills included:**

| Skill | Purpose |
|---|---|
| `/keycard-credentials` | Show what credentials (tokens, services) are active in the current session |
| `/keycard-upsert-config` | Set or update a field in `keycard.toml` |
| `/keycard-upsert-mcp-config` | Add or update an MCP server entry in `.mcp.json` |

## Prerequisites

- [Keycard CLI](https://keycard.ai) v0.0.35 or later, installed and on your `$PATH`. Earlier releases answer the PostToolUse hook in a shape Claude Code rejects, so it reports a hook error on every tool call and redacts nothing.
- Claude Code (CLI or desktop app)

## Installation

Add the Keycard marketplace to Claude Code, then install the plugin at project scope, so its hooks run only in projects that use Keycard:

```bash
claude plugin marketplace add keycardai/plugins
claude plugin install keycard-cli@keycardai -s project
```

`keycard run claude` checks for the plugin before launching: it offers to install it when it is missing and to update it when it is older than the version the CLI requires. To update by hand:

```bash
claude plugin marketplace update keycardai && claude plugin update keycard-cli@keycardai -s project
```

The marketplace update is what makes a new release visible; `claude plugin update` alone compares against the copy of the marketplace you already have.

## Usage

Start a Claude Code session through Keycard to put the credentials declared in `keycard.toml` in its environment and connect it to your Zone's MCP gateways:

```bash
keycard run claude
```

Without `keycard run claude`, the hooks still run, but Claude's environment has no Keycard credentials and no MCP gateways. The SessionStart hook tells Claude the session was not started with `keycard run claude`.

Once in a session, just ask naturally. For example:

- "What credentials do I have in this session?"
- "Set my zone to my-zone-id."
- "Add an MCP server called my-server using npx with args -y @my/mcp-server."

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
