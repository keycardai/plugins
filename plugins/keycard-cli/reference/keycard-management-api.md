# Keycard Management API — Endpoint Reference

## Authentication

All `keycard agent api` commands require an active Management API session. If a command fails with "not signed in to management API", run:

```bash
keycard auth signin --org <org-id>
```

Then stop.

## Endpoint tree

`--org` is required on every call. Use `org.id` in `keycard.toml`, the `ORG` environment variable, or `--org` flag. If none is set, call `/organizations` first to find the org ID.

### List organizations

```bash
keycard agent api /organizations --org <org-id>
```

### List zones

```bash
keycard agent api /zones --org <org-id>
```

### Zone-scoped listings

List providers available in a zone:
```bash
keycard agent api /zones/<zone-id>/providers --org <org-id>
```

List resources available in a zone:
```bash
keycard agent api /zones/<zone-id>/resources --org <org-id>
```

List applications available in a zone:
```bash
keycard agent api /zones/<zone-id>/applications --org <org-id>
```
