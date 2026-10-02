# keycard.toml Field Reference

## File location

Keycard reads `keycard.toml` from the project root. Override the path with the `--config <path>` flag or the `CLI_CONFIG` environment variable.

## Precedence

When the same value is set in multiple places, the highest-priority source wins:

1. **Flags** — e.g. `--zone`, `--org`
2. **Environment variables** — e.g. `ZONE`, `ORG`
3. **TOML file** — `keycard.toml`
4. **Built-in defaults**

## Field reference

Only the fields below are recognised in `keycard.toml`. Fields marked **env/flag only** are silently ignored if written to TOML — use the corresponding environment variable or flag instead.

- **`environment`**
  - **Env var**: `KEYCARD_ENV`
  - **Default**: `production`
  - **Notes**: `production`, `staging`, `dev`, `localdev`
- **`zone.id`**
  - **Env var / flag**: `ZONE` / `--zone`
  - **Notes**: Zone ID
- **`zone.issuer.url`**
  - **Env var / flag**: `ISSUER_URL` / `--issuer-url`
  - **Notes**: OIDC issuer override; must use HTTPS. **Env/flag only**.
- **`org.id`**
  - **Env var / flag**: `ORG` / `--org`
  - **Notes**: Org ID
- **`oidc.port`**
  - **Env var**: `OIDC_CALLBACK_PORT`
  - **Default**: `0` (random)
  - **Notes**: Auth callback port. **Env/flag only**.
- **`auth.device`**
  - **Env var / flag**: `KEYCARD_DEVICE` / `--device`
  - **Default**: `false`
  - **Notes**: Use the RFC 8628 device authorization flow for `keycard auth signin`. Not written back when the CLI rewrites `keycard.toml`.
- **`home.dir`**
  - **Env var**: `KEYCARD_HOME`
  - **Default**: OS default
  - **Notes**: Home directory override
- **`home.state`**
  - **Env var**: `XDG_STATE_HOME`
  - **Default**: `~/.local/state/keycard`
  - **Notes**: State directory override
- **`home.cache`**
  - **Env var**: `XDG_CACHE_HOME`
  - **Default**: `~/.cache/keycard`
  - **Notes**: Cache directory override
- **`credentials.default[*].env_var`**
  - **Notes**: Env var to populate from the exchanged token. See [Credentials](#credentials).
- **`credentials.default[*].resource`**
  - **Notes**: Resource URI for OIDC token exchange. Required.

> If a TOML edit has no effect, the field is probably env/flag only. Use the corresponding environment variable instead.

> Fields tagged `internal` in `config.go` are excluded from this table — they are implementation details not intended for direct configuration.

## Credentials

`credentials` is the only complex section writable to TOML. It is a list of credential entries under `[[credentials.default]]`.

Each `[[credentials.default]]` entry requires two fields:
- `resource` — OIDC resource URI to exchange a token for
- `env_var` — environment variable to populate with the exchanged token

```toml
[[credentials.default]]
env_var = "GH_TOKEN"
resource = "https://github.com"

[[credentials.default]]
env_var = "MY_API_TOKEN"
resource = "https://api.example.com"
```
