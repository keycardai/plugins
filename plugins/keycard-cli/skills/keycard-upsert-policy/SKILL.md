---
name: keycard-upsert-policy
description: |
  Propose, confirm, and apply a Cedar policy change — propose → confirm → write → verify.

  TRIGGER when: user wants to change, add, remove, enable, disable, allow, deny, grant, restrict, block a policy rule, add ITL gating, or route a tool to a credential set.
  DO NOT TRIGGER when: user asks questions about what is allowed or why something was blocked (→ `keycard-query-policy`).
argument-hint: "[policy change request, e.g. 'Allow the Bash tool' or 'Require approval for WebFetch']"
examples:
  - /keycard-upsert-policy Allow the Bash tool
  - /keycard-upsert-policy Require approval for WebFetch
  - /keycard-upsert-policy Block curl commands in Bash
---

# keycard-upsert-policy

You are helping the user modify the Cedar policy enforced by Keycard. Follow the propose → confirm → write → verify pattern: present the full diff first, write only after explicit user confirmation.

See `.agents/reference/cedar-policy.md` for Cedar syntax rules, annotation semantics, and compactness/credential-inference rules. Policy reads must always use `keycard agent policy` — never the Read tool on the file directly.

## Step 1 — Read the policy

Run:
```bash
keycard agent policy
```

If the command fails (no policy file), display the returned message, then stop.

## Step 2 — Accept change request

The user's change request is: `$ARGUMENTS`

If `$ARGUMENTS` is empty, ask: "What policy change would you like to make? (e.g. 'Allow the Bash tool', 'Require approval for WebFetch', 'Block the curl command')"

Wait for the response before continuing.

## Step 3 — Propose change

If the requested change is already fully represented in the current policy, skip Steps 3–5 and respond: "The policy already contains this rule — no change is needed." Then stop.

Generate the **minimal Cedar clause(s)** needed to implement the requested change, following the rules in `.agents/reference/cedar-policy.md`. Key checklist:

- Apply the credential-set inference rule.
- Apply the compactness rule.
- Add `@description(...)` on every new or modified clause.
- Add `@itl("prompt")` only if explicitly requested.
- Add `@credentials("name")` only if explicitly requested.

Compose the **full updated policy** by merging the new clause(s) into the policy from Step 1. Keep the full text internally — you will need it for Step 5.

**Do not ask clarifying questions before presenting the proposal.** Make reasonable assumptions and proceed immediately. In a single response, output all three of the following:

1. A unified diff in a fenced `cedar` code block: unchanged context lines unmodified, removed lines prefixed with `- `, added lines prefixed with `+ `. Use a `cedar` fence (not `diff`) so the content gets Cedar syntax highlighting.
2. A plain-language explanation of what the change does:
   - Which tools become allowed or blocked.
   - Whether any existing rules are removed or superseded.
   - If any affected tool already has or gains a `@itl("prompt")` annotation: note that tool calls will require explicit approval before execution.
   - If any affected `permit` clause already has or gains a `@credentials("name")` annotation: note which credential set tool calls will be routed to.
3. The confirmation prompt: **"Does this look correct? Type 'yes' to apply, or describe any adjustments."**

**Example output** (for "allow the Bash tool with staging credentials and ITL"):

````
```cedar
+ @description("Permit all users to invoke the Bash tool, routed to staging credentials with in-the-loop review.")
+ @itl("prompt")
+ @credentials("staging")
+ permit (
+   principal,
+   action == Action::"Agent::ToolUse",
+   resource == Tool::"bash"
+ );
```

**Policy Updates:**
- `bash` — now permitted with in-the-loop (ITL) and `staging` credentials (inferred from an existing clause in the policy that already uses `staging` credentials).

Does this look correct? Type 'yes' to apply, or describe any adjustments.
````

**Example output** (for "switch Bash to prod credentials"):

````
```cedar
- @description("Permit all users to invoke the Bash tool, routed to staging credentials.")
- @credentials("staging")
- permit (
+ @description("Permit all users to invoke the Bash tool, routed to prod credentials.")
+ @credentials("prod")
+ permit (
    principal,
    action == Action::"Agent::ToolUse",
    resource == Tool::"bash"
  );
```

**Policy Updates:**
- `bash` — credential set changed from `staging` to `prod`; existing `permit` clause replaced.

Does this look correct? Type 'yes' to apply, or describe any adjustments.
````

## Step 4 — Wait for confirmation

Do **not** write anything until the user explicitly confirms (e.g. "yes", "apply", "looks good").

If the user requests adjustments, return to Step 3 with the revised request.

If the user declines, stop without writing.

## Step 5 — Write the policy

Determine the target path. If `$KEYCARD_POLICY_FILE` is set, use it directly (skip the mkdir). Otherwise fall back to the XDG default:
```bash
echo "${KEYCARD_POLICY_FILE:-${XDG_STATE_HOME:-$HOME/.local/state}/keycard/policy.cedar}"
```

If `$KEYCARD_POLICY_FILE` is not set, create the parent directory:
```bash
mkdir -p "${XDG_STATE_HOME:-$HOME/.local/state}/keycard"
```

Write the confirmed policy to the resolved path using the Write tool (overwrite the entire file with the proposed policy text).

## Step 6 — Verify

Re-run:
```bash
keycard agent policy
```

Confirm the output contains the new or modified clause(s) from Step 3.

- **If the clause(s) are present**: confirm success — "Policy updated successfully."
- **If any clause is missing or different**: report the discrepancy clearly:
  - Show the expected clause(s) vs. what was read back.
  - Do **not** claim success.
  - Suggest the user check file permissions or whether another process modified the file.
