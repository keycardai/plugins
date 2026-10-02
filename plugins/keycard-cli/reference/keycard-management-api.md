# Keycard Management API — Endpoint Reference

## Authentication

All `keycard agent api` commands require an active Management API session. If a command fails with "not signed in to management API", run:

```bash
keycard auth signin --org <org-id>
```

Then stop.

## Endpoint tree

`--org` is an optional override. The CLI resolves org automatically from `org.id` in `keycard.toml` or the `ORG` environment variable; pass `--org` only when you need to override the configured value.

Zone is resolved automatically from `zone.id` in `keycard.toml` or the `ZONE` environment variable. Use `{zone}` in the path — it is interpolated before the request fires. Similarly, use `{org}` in the path to interpolate the resolved org ID. **If no `keycard.toml` is present in the working directory and `ZONE` is not set, zone-scoped commands will fail** with `zone is required: set via --zone flag, ZONE environment variable, or zone.id in keycard.toml`. In that case pass `--zone <zone-id>` explicitly, or copy a `keycard.toml` into the working directory.

### List organizations

```bash
keycard agent api /organizations [--org <org-id>]
```

### List zones

The `filter[organization_id]` query parameter is **required** — the API returns a 400 without it. `{org}` is interpolated in query parameters just like in paths, so if `org.id` is set in `keycard.toml` or `$ORG` is set, this just works:

```bash
keycard agent api '/zones?filter[organization_id]={org}'
```

If org is not configured, resolve it first via `/organizations` and pass it explicitly:

```bash
keycard agent api '/zones?filter[organization_id]=<org-id>'
```

### Mint an org token

```bash
keycard agent api -X POST /organizations/{org}/token
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

### Create a resource

```bash
keycard agent api -X POST -d '{"name":"...","identifier":"...","credential_provider_id":"...","application_type":"web"}' /zones/{zone}/resources
```
