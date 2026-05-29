# Keycard Management API — Endpoint Reference

## Authentication

All `keycard agent api` commands require an active Management API session. If a command fails with "not signed in to management API", run:

```bash
keycard auth signin --org <org-id>
```

Then stop.

## Endpoint tree

`--org` is an optional override. The CLI resolves org automatically from `org.id` in `keycard.toml` or the `ORG` environment variable; pass `--org` only when you need to override the configured value.

Zone is resolved automatically from `zone.id` in `keycard.toml` or the `ZONE` environment variable. Use `{zone}` in the path — it is interpolated before the request fires.

### List organizations

```bash
keycard agent api /organizations [--org <org-id>]
```

### List zones

```bash
keycard agent api /zones [--org <org-id>]
```

### Zone-scoped listings

List providers available in a zone:
```bash
keycard agent api /zones/{zone}/providers
```

List resources available in a zone:
```bash
keycard agent api /zones/{zone}/resources
```

List applications available in a zone:
```bash
keycard agent api /zones/{zone}/applications
```
