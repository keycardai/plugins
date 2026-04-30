# Keycard CLI Plugin for Claude Code

Zero-trust tool access control and credential injection for Claude Code,
powered by [Keycard](https://keycard.ai) and [Cedar](https://www.cedarpolicy.com) policies.

## What It Does

This plugin hooks into Claude Code's execution pipeline to enforce security
policies before any tool is invoked. It gates tool use through Cedar policies
(permit, deny, require approval) and injects credentials at runtime — so
secrets never live in config files or environment dotfiles.

**Three hooks run automatically:**

| Hook | When | What |
|---|---|---|
| `SessionStart` | Claude boots | Warns if `keycard run` wasn't used to launch the session |
| `PreToolUse` | Before every tool call | Enforces Cedar policy; blocks disallowed tools; injects credentials |
| `UserPromptSubmit` | After user sends a prompt | Policy enforcement checkpoint on prompt processing |

**Six skills for managing policies and credentials:**

| Skill | Purpose |
|---|---|
| `/keycard-init` | Scaffold `keycard.toml` and initial Cedar policy for a project |
| `/keycard-credentials` | Show what credentials are active in the current session |
| `/keycard-discover-entities` | Find available credential entities via the Management API and register them |
| `/keycard-query-policy` | Explain the active Cedar policy; diagnose why a tool was blocked (read-only) |
| `/keycard-upsert-config` | Set or change fields in `keycard.toml` |
| `/keycard-upsert-policy` | Propose, confirm, and apply Cedar policy changes |

**Reference docs** are bundled under `plugins/keycard-cli/reference/` for Cedar
syntax, `keycard.toml` field definitions, and the Management API endpoint tree.

## Repository Layout

```
.claude-plugin/
  marketplace.json          # Plugin marketplace metadata (keycardai)

plugins/keycard-cli/
  .claude-plugin/
    plugin.json             # Plugin manifest — hooks + version
  skills/
    keycard-init/           # Project scaffolding
    keycard-credentials/    # Active credential reporting
    keycard-discover-entities/  # Credential entity discovery
    keycard-query-policy/   # Policy Q&A and block diagnosis
    keycard-upsert-config/  # keycard.toml editor
    keycard-upsert-policy/  # Cedar policy editor
  reference/
    cedar-policy.md         # Cedar syntax and Keycard conventions
    keycard-config-fields.md    # keycard.toml field reference
    keycard-management-api.md   # Management API endpoint tree
```

## Prerequisites

- [Keycard CLI](https://keycard.ai) installed and on your `$PATH`
- A Keycard account with a configured zone
- Claude Code

## Installation

Add the marketplace and install the plugin:

```
/plugin marketplace add keycardai/plugins
/plugin install keycard-cli@keycardai
```

Or for local development, point Claude Code at the plugin directory:

```bash
git clone https://github.com/keycardai/plugins.git
claude --plugin-dir ./plugins/keycard-cli
```

## Getting Started

1. **Install the plugin** (see above).

2. **Launch Claude through Keycard** so hooks are active:

   ```bash
   keycard run -- claude
   ```

   Without this, the `SessionStart` hook will warn you that policy enforcement
   and credential injection are inactive.

3. **Initialize your project** (if you haven't already):

   ```
   /keycard-init
   ```

   This detects your project's toolchain, scaffolds `keycard.toml`, and
   proposes an initial Cedar policy.

4. **Discover and register credentials:**

   ```
   /keycard-discover-entities
   ```

5. **Check what's allowed:**

   ```
   /keycard-query-policy
   ```

6. **Adjust policies as needed:**

   ```
   /keycard-upsert-policy Allow the Bash tool with ITL approval
   ```

## How Cedar Policies Work

Cedar is a human-readable policy language. Keycard uses it to control which
tools Claude can invoke. Key concepts:

- **`permit`** allows a tool. **`forbid`** blocks it — and forbid always wins.
- **`@itl("prompt")`** requires explicit user approval before execution.
- **`@credentials("name")`** routes the tool call through a named credential set.
- Omitting a permit clause for a tool is sufficient to deny it.

Example policy:

```cedar
@description("Allow Git operations with approval required.")
@itl("prompt")
permit (
  principal,
  action == Action::"Agent::ToolUse",
  resource == Tool::"bash"
)
when { context.command like "git *" };
```

See `plugins/keycard-cli/reference/cedar-policy.md` for the full reference.

## Security Model

- **Zero-trust**: every tool invocation is checked against Cedar policy
- **No secrets in config**: credentials are injected at runtime by `keycard run`
- **Skills never read credential values** — only env var names
- **Policy changes require explicit confirmation** before being written
- **Forbid overrides permit** — conservative by design

## License

Proprietary. Copyright Keycard Labs, Inc.
