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

| TOML path | Env var | Default | Notes |
|---|---|---|---|
| `environment` | `KEYCARD_ENV` | `production` | `production`, `staging`, `dev`, `localdev` |
| `zone.id` | `ZONE` / `--zone` | — | Zone ID |
| `zone.issuer.url` | `ISSUER_URL` / `--issuer-url` | — | OIDC issuer override; must use HTTPS — **env/flag only** |
| `org.id` | `ORG` / `--org` | — | Org ID — **env/flag only** |
| `oidc.port` | `OIDC_CALLBACK_PORT` | `0` (random) | Auth callback port — **env/flag only** |
| `home.dir` | `KEYCARD_HOME` | OS default | Home directory override — **env/flag only** |
| `home.state` | `XDG_STATE_HOME` | `~/.local/state/keycard` | State directory override — **env/flag only** |
| `home.cache` | `XDG_CACHE_HOME` | `~/.cache/keycard` | Cache directory override — **env/flag only** |
| `credentials.default[*].env_var` | — | — | Env var to populate from the exchanged token (see [Credentials](#credentials)) |
| `credentials.default[*].resource` | — | — | Resource URI for OIDC token exchange (required) |

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
