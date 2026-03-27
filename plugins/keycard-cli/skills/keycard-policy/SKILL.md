---
name: keycard-policy
description: Inspect Keycard's Cedar policy and answer questions about what tools are allowed like "May I use the Bash tool?"
argument-hint: "[policy question, e.g. 'What tools are allowed?']"
---

# keycard-policy

You are helping the user understand what tools are permitted by their active Keycard Cedar policy.

## Step 1 — Read the policy

Run:
```bash
keycard agent policy
```

If the command fails (no policy file), display the returned message, then stop.

## Step 2 — Answer the question

The user's tool use question is: `$ARGUMENTS`

If `$ARGUMENTS` is empty, ask: "What would you like to know? For example: 'What tools are allowed?' or 'Is the Bash tool permitted?'"

Use the policy to clearly answer the question:
- For "May I use X?" questions: state allow or deny, citing the specific policy clause.
- For "What can I do?" questions: list permitted tools/actions with brief explanations of each permit clause.
- For "Why was X blocked?" questions: identify the forbid or absence of permit clause which caused the denial.

Keep the answer concise. Quote relevant Cedar clause(s) to justify the answer.
