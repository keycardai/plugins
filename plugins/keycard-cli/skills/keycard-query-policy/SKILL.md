---
name: keycard-query-policy
description: |
  Answer questions about the active Cedar policy and diagnose tool blocks — read-only; does not modify the policy.

  TRIGGER when: user asks what tools are allowed, whether a specific tool is permitted (e.g. "Can I use X?", "Am I allowed to use X?", "What's my policy?"), why a tool was blocked, or reports "a tool was just blocked."
  DO NOT TRIGGER when: user wants to change, add, or remove a policy rule (→ `keycard-upsert-policy`); user asks general questions about Cedar concepts without reference to their active policy.
argument-hint: "[policy question or blocked tool, e.g. 'May I use the Bash tool?' or 'Why was Read blocked?']"
license: Apache-2.0
---

# keycard-query-policy

You are helping the user understand their active Cedar policy. This skill is read-only — to modify the policy, tell the user to describe the change they'd like to make.

See `../../reference/cedar-policy.md` for Cedar syntax reference. Policy reads must always use `keycard agent policy` — never the Read tool on the file directly.

## Step 1 — Accept question

The user's input is: `$ARGUMENTS`

If `$ARGUMENTS` is empty, ask: "What would you like to know about the policy? (e.g. 'May I use Bash?', 'What tools are allowed?', 'Why was X blocked?')"

Wait for the response, then continue to Step 2.

## Step 2 — Read the policy

Run:
```bash
keycard agent policy
```

If the command fails (no policy file or CLI error), display the returned message and stop.

## Step 3 — Answer

Handle the question or diagnostic request using the policy from Step 2:

### Q&A intents

- **"May I use X?"** — state allow or deny, citing the specific Cedar clause. If a `forbid` matches, note it takes precedence over any `permit`.
- **"What can I do?"** / **"What tools are allowed?"** — list all tools covered by `permit` clauses. Note any with `@itl` (require approval). Note any `forbid` overrides.
- **Active-policy questions** — answer questions about the *contents of the loaded policy* (e.g. "do I have any `@itl` rules?") using the policy text from Step 2. For abstract Cedar concepts not tied to the active policy, say: "That's a general Cedar question — see `../../reference/cedar-policy.md` for syntax details. To change your policy, describe the change you'd like to make."

### Block-diagnosis intents

When the user reports a tool was blocked ("a tool was just blocked", "Why was X blocked?"):

1. Identify the tool name from `$ARGUMENTS` or ask: "Which tool was blocked? (e.g. `Bash`, `Read`, `mcp__<server>__<function>` — see `.agents/reference/cedar-policy.md` MCP tools)"
2. Search the policy for clauses matching the tool name (case-insensitive, e.g. `Tool::"bash"` matches `Bash`). If a matching clause has `when`/`unless` conditions, note that it may only apply conditionally — quote the conditions alongside the clause.
3. Output exactly three parts for whichever case applies:

**Case A — One or more matching `forbid` clauses found:**

1. **Diagnosis**: "A `forbid` clause is blocking this tool — adding a `permit` will have no effect while it remains."
2. **Blocking clause(s)**: Quote all matching `forbid` clauses exactly as they appear in the policy.
3. **Suggested fix**: "Remove or narrow the `forbid` clause by telling Claude you'd like to update your policy (e.g. 'remove the forbid on the Read tool')."

**Case B — No matching `permit` found (and no active `forbid`):**

1. **Diagnosis**: "No `permit` clause was found for this tool."
2. **Blocking clause**: `"no matching permit found"`
3. **Suggested fix**: "Add a permit clause by telling Claude you'd like to allow the <tool name> tool."

**Case C — A `permit` exists but no `forbid` is present:**

1. **Diagnosis**: "A `permit` clause exists, but it may carry unsatisfied `when`/`unless` conditions, or the tool requires in-the-loop (`@itl`) approval not given at call time."
2. **Matching clause(s)**: Quote all matching `permit` clause(s) exactly, including any `when`/`unless` conditions.
3. **Suggested fix**: For `@itl`: "Approve the tool call when prompted." For conditional permits: "Check whether the runtime context matched the `when` conditions; tell Claude you'd like to loosen or remove the condition."

Keep answers concise. Quote relevant Cedar clause(s) to justify the answer. Do not propose or write any policy changes.

## Examples

**"May I use X?" query:**

Input: `May I use the Bash tool?`

Output:
> **Allowed.** The following clause permits it:
> ```cedar
> permit (principal, action == Action::"Agent::ToolUse", resource == Tool::"bash");
> ```
> No active `forbid` overrides this.

---

**"What tools are allowed?" query:**

Input: `What tools are allowed?`

Output:
> **Permitted tools:**
> - `bash` — unrestricted
> - `read` — requires in-the-loop approval (`@itl`)
>
> No `forbid` clauses are active.

---

**Block-diagnosis — Case A (`forbid` found):**

Input: `Why was Read blocked?`

Output:
> **Diagnosis**: A `forbid` clause is blocking this tool — adding a `permit` will have no effect while it remains.
>
> **Blocking clause:**
> ```cedar
> forbid (principal, action == Action::"Agent::ToolUse", resource == Tool::"read");
> ```
>
> **Suggested fix**: Remove or narrow the `forbid` clause by telling Claude you'd like to update your policy (e.g. "remove the forbid on the Read tool").
