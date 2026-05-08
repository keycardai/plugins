# Cedar Policy Reference

Keycard enforces a Cedar policy that controls which tools agents may invoke. This document is the canonical reference for Cedar syntax and Keycard-specific conventions.

## Policy file

The active policy is read via:
```bash
keycard agent policy
```

Never read `~/.local/state/keycard/policy.cedar` directly — use the CLI as the authoritative source. To write a policy, write to `~/.local/state/keycard/policy.cedar` (or `$XDG_STATE_HOME/keycard/policy.cedar` if `XDG_STATE_HOME` is set), then verify with `keycard agent policy`.

## Permit/forbid syntax

```cedar
@description("Allow all users to use the Read tool.")
permit (
  principal,
  action == Action::"Agent::ToolUse",
  resource == Tool::"read"
);

@description("Block the Bash tool for all users.")
forbid (
  principal,
  action == Action::"Agent::ToolUse",
  resource == Tool::"bash"
);
```

**`forbid` always overrides `permit`.** A `forbid` clause blocks access even when a matching `permit` exists — no exception.

## Principal

- `User::"<email>"` — targets a specific user (e.g. `User::"alice@example.com"`).
- Omitting the principal condition (`principal,`) applies the clause to all users.

## Action

Always `Action::"Agent::ToolUse"` for tool-use policies.

## Resource

`Tool::"<tool_name>"` — the tool name as it appears in the Keycard runtime.

**Lowercase convention**: use lowercase tool names (e.g. `Tool::"bash"`, `Tool::"read"`) for cross-harness compatibility. Some harnesses (e.g. pi-mono) use lowercase names; matching existing conventions in the policy avoids split-coverage bugs. Case-insensitive matching is used at diagnosis time — `Tool::"bash"` matches a block on `Bash`.

## Annotations

Annotations are placed immediately before the `permit` or `forbid` keyword, in this order:

1. `@description("...")` — **always required** on any new or modified clause. One sentence: who it affects, what it allows/denies, and why (if inferable). Do not retroactively annotate existing clauses that were not part of a requested change.
2. `@itl("prompt")` — adds in-the-loop (ITL) gating: the user must explicitly approve the tool call before it executes. Add only when explicitly requested. `@itl` on `forbid` has no effect — annotations are only evaluated on `permit`-matched clauses.

```cedar
@description("Permit all users to invoke the Bash tool with in-the-loop review.")
@itl("prompt")
permit (
  principal,
  action == Action::"Agent::ToolUse",
  resource == Tool::"bash"
);
```

## Compactness and carve-outs

Prefer modifying or splitting an existing clause over appending a new overlapping one.

**Carve-out rule**: a carve-out is needed when a new `permit` clause is narrower than an existing `permit` clause — add an `unless` condition to the broader clause so the two clauses have disjoint match sets. Without a carve-out, both clauses match the narrower principal/resource, creating annotation ambiguity. A carve-out is not needed for `permit`+`forbid` pairs (`forbid` always wins) or when no existing `permit` clause covers the same scope.

**Example — permit+permit narrowing (broken → fixed):**

```cedar
// BROKEN: alice matches both clauses; annotation is ambiguous.
@description("Permit all users to invoke Bash.")
permit (
  principal,
  action == Action::"Agent::ToolUse",
  resource == Tool::"bash"
);

@description("Permit alice to invoke Bash with ITL review.")
@itl("prompt")
permit (
  principal == User::"alice@example.com",
  action == Action::"Agent::ToolUse",
  resource == Tool::"bash"
);

// FIXED: carved out — disjoint match sets.
@description("Permit all users except alice to invoke Bash.")
permit (
  principal,
  action == Action::"Agent::ToolUse",
  resource == Tool::"bash"
) unless {
  principal == User::"alice@example.com"
};

@description("Permit alice to invoke Bash with ITL review.")
@itl("prompt")
permit (
  principal == User::"alice@example.com",
  action == Action::"Agent::ToolUse",
  resource == Tool::"bash"
);
```

## Forbid-diagnosis rule

If a `forbid` clause matches a blocked tool, adding a `permit` will have **no effect** while the `forbid` remains. Fix: remove or narrow the `forbid` clause, then add a `permit` if needed.

## Bash command restrictions

For `Tool::"bash"` (or `Tool::"Bash"`), use `context.command like` conditions to restrict which commands are allowed. Prefer specific subcommands over broad wildcards:

```cedar
permit (
  principal,
  action == Action::"Agent::ToolUse",
  resource == Tool::"bash"
) when {
  context has command && (
    context.command like "go test *" ||
    context.command like "go build *" ||
    context.command like "git *" ||
    context.command like "gh *"
  )
};
```

Never permit unrestricted shell execution via wildcards such as `"sh *"`, `"bash *"`, or `"curl *"` — they bypass policy intent. Use `forbid` clauses sparingly; omission of a `permit` is sufficient to deny.
