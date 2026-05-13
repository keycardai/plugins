# Skill Narration — Shared Rules

Common narration style and concept glossary for user-facing skills. Each skill may add a one-line framing sentence (e.g. "Treat the run as a guided tour, not a CI log") before linking here.

## Narration rules

- **Open with a one-line plan** (~40 words) before starting work.
- **Narrate each step in plain English** before running commands. One short sentence is enough — never let a long-running command appear out of nowhere.
- **No protocol or acronym jargon by default.** Never drop terms like *OIDC issuer*, *OAuth*, *STS*, *RFC 8693*, *token exchange*, *JWT*, *PKCE*, *DPoP*, *IdP*, *bearer token*, *audience claim*, *401/403* error codes, etc. into user-facing narration. Translate to outcomes instead. Acronym-heavy detail is fine **only** when (a) the user explicitly asks "how does this work under the hood?" or (b) you're surfacing a verbatim error message from a tool. The same rule applies to anything read from a reference doc or template `SPEC.md` — paraphrase into plain language before showing it to the user.
- **Introduce each Keycard concept the first time it appears, with a docs link** (see glossary below). Don't re-explain on later mentions.
- **Surface what just happened, not just what's next.** After each registration or file write, name the artifact and (where useful) the ID.
- **No raw command dumps without context.** Say what a command does before printing it.
- **Stay calm under failure.** Explain the concept, link the docs, and offer the next concrete action.

## Concept glossary

| Concept | Definition | Docs |
|---|---|---|
| **Zone** | Your private Keycard environment that holds users, apps, and policies and issues credentials. | https://docs.keycard.ai/platform/concepts/zones/ |
| **Organization** | The account that owns one or more zones (usually your company or team). | https://docs.keycard.ai/platform/concepts/zones/ |
| **Application** | The thing you're building, registered with Keycard so it can request credentials on a user's behalf or operate with its own identity. | https://docs.keycard.ai/platform/concepts/applications/ |
| **Resource** | The protected endpoint the application serves or calls (e.g. your server's `/mcp` URL). An app can have multiple Resources — one for local dev and one for a deployed URL. | https://docs.keycard.ai/platform/concepts/resources/ |
| **Provider** | Where the credentials your app receives actually come from. Keycard itself can mint them, pull a stored secret from Keycard Vault, or hand off to an external identity provider (e.g. a hosting platform's identity system). | https://docs.keycard.ai/platform/concepts/providers/ |
| **Application Credential** | The binding that says "this Application, when authenticated by this Provider with this identity pattern, is allowed to mint credentials in your zone." | https://docs.keycard.ai/platform/concepts/applications/ |
| **Zone URL** | The base URL of your zone (`https://<zone-id>.<env-domain>`); your app uses it to confirm that credentials it receives really came from your Keycard zone. | https://docs.keycard.ai/platform/architecture/standards-and-protocols/ |
