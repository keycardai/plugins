# Keycard CLI Plugin — Agent Instructions

## What This Repo Is

A Claude Code plugin that integrates the Keycard CLI for zero-trust tool access
control. It enforces Cedar policies at runtime (via hooks) and provides skills
for managing those policies and credentials.

**Published as:** `keycardai@keycard`
**Current version:** 0.0.2

## Architecture

Two layers:

1. **Runtime enforcement (hooks)** — `PreToolUse` and `UserPromptSubmit` hooks
   call `keycard agent hook --claude` before every tool invocation and prompt
   submission. A `SessionStart` hook warns when Keycard isn't active. These
   are defined in `plugins/keycard-cli/.claude-plugin/plugin.json`.

2. **Management UX (skills)** — Six `/keycard-*` skills let users scaffold
   config, discover credentials, query policies, and edit both config and
   policies. Skills live under `plugins/keycard-cli/skills/`.

```
                      ┌─────────────────────┐
                      │   Claude Code User   │
                      └──────────┬──────────┘
                                 │
                    ┌────────────┴────────────┐
                    │                         │
              hooks (auto)              skills (manual)
                    │                         │
          ┌─────────┴─────────┐    ┌──────────┴──────────┐
          │ PreToolUse        │    │ /keycard-init        │
          │ UserPromptSubmit  │    │ /keycard-credentials │
          │ SessionStart      │    │ /keycard-discover-*  │
          └─────────┬─────────┘    │ /keycard-query-*    │
                    │              │ /keycard-upsert-*   │
                    ▼              └──────────┬──────────┘
          ┌─────────────────┐                │
          │  keycard CLI     │◄───────────────┘
          │  (Cedar engine)  │
          └─────────────────┘
```

## Key Files

| Path | Purpose |
|---|---|
| `.claude-plugin/marketplace.json` | Marketplace metadata |
| `plugins/keycard-cli/.claude-plugin/plugin.json` | Hook definitions + version |
| `plugins/keycard-cli/skills/*/SKILL.md` | Skill implementations |
| `plugins/keycard-cli/reference/cedar-policy.md` | Cedar syntax reference |
| `plugins/keycard-cli/reference/keycard-config-fields.md` | `keycard.toml` field reference |
| `plugins/keycard-cli/reference/keycard-management-api.md` | Management API endpoints |

## Working in This Repo

- Skills are plain Markdown files (`SKILL.md`) with frontmatter
- Hooks are shell commands defined in `plugin.json`
- Reference docs are Markdown consumed by skills at runtime
- No build step — everything is interpreted by Claude Code directly
- Test changes by installing the plugin locally: `claude plugin add ./plugins/keycard-cli`

## Conventions

- Tool names in Cedar are always lowercase: `Tool::"bash"`, not `Tool::"Bash"`
- Skills never read or echo actual credential values — only env var names
- Policy changes always require explicit user confirmation before writing
- `forbid` overrides `permit` in Cedar — this is by design, not a bug
- Bash policy rules must target specific subcommands (`"git *"`), never broad wildcards (`"bash *"`)

## Skill Responsibilities

| Skill | Reads | Writes | Side effects |
|---|---|---|---|
| `keycard-init` | git remote, manifests, workflow files | `keycard.toml`, `policy.cedar` | None |
| `keycard-credentials` | `keycard credential info` output | Nothing | None |
| `keycard-discover-entities` | Management API, `keycard.toml` | `keycard.toml` | None |
| `keycard-query-policy` | Active Cedar policy | Nothing | None |
| `keycard-upsert-config` | `keycard.toml` | `keycard.toml` | None |
| `keycard-upsert-policy` | Active Cedar policy | `policy.cedar` | None |
