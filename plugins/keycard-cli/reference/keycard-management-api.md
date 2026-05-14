# Keycard Management API — Endpoint Reference

## Authentication

All `keycard agent api` commands require an active Management API session. If a command fails with "not signed in to management API", run:

```bash
keycard auth signin --org <org-id>
```

Then stop.

## Endpoint tree

`--org` is required on every call. Use the `ORG` environment variable or pass `--org` explicitly; if neither is available, call `/organizations` first to find the org ID.

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

List application-credentials available in a zone:
```bash
keycard agent api /zones/<zone-id>/application-credentials --org <org-id>
```

## Environment variable caveat

When calling `keycard agent api` from inside a `keycard run` session (i.e. from an agent subprocess), the session env vars `KEYCARD_RUN`, `KEYCARD_SOCKET`, and `KEYCARD_RUN_SESSION_ID` cause the CLI to route through the session socket instead of hitting the Management API directly. Unset them so the command uses the normal API path:

```bash
env -u KEYCARD_RUN -u KEYCARD_SOCKET -u KEYCARD_RUN_SESSION_ID \
  keycard agent api /zones/<zone-id>/... --org <org-id>
```

All `keycard agent api` examples in this document assume these variables are unset. If commands fail unexpectedly inside a `keycard run` session, this is the most likely cause.

## Create endpoints

All create endpoints are zone-scoped `POST` requests. Every endpoint returns the created entity (including its `id`) on success and 409 on identifier collision.

### Create a provider

```bash
keycard agent api /zones/<zone-id>/providers \
  -X POST \
  -d '{"name":"fly-myorg","identifier":"fly-myorg","protocols":{"oauth2":{"issuer":"https://oidc.fly.io/myorg"}}}' \
  --org <org-id>
```

Required fields: `name`, `identifier`.
Optional: `protocols.oauth2.issuer` (OIDC issuer URL for discovery and token validation), `client_id`, `client_secret`, `description`, `metadata`.

### Create an application

```bash
keycard agent api /zones/<zone-id>/applications \
  -X POST \
  -d '{"name":"my-app","identifier":"my-app"}' \
  --org <org-id>
```

Required fields: `name`, `identifier`.
Optional: `description`, `dependencies` (array of `{"resource_id":"..."}` items), `metadata`, `protocols`.

### Create a resource

```bash
keycard agent api /zones/<zone-id>/resources \
  -X POST \
  -d '{"name":"my-app-prod","identifier":"https://my-app.fly.dev/mcp"}' \
  --org <org-id>
```

Required fields: `name`, `identifier` (typically the protected URL).
Optional: `description`, `scopes` (array of strings), `credential_provider_id`, `application_id`, `application_type` (`native` or `web`), `metadata`.

### Create an application credential

Binds an Application to a Provider with a subject constraint — tells Keycard "this application, authenticated by this provider with this subject pattern, is allowed to mint credentials in the zone."

```bash
keycard agent api /zones/<zone-id>/application-credentials \
  -X POST \
  -d '{"application_id":"<application-id>","provider_id":"<provider-id>","type":"token","subject":"<fly-org-slug>:<project-name>:*"}' \
  --org <org-id>
```

Required fields: `application_id`, `provider_id`, `type`, `subject`.
The `subject` field must match the `sub` claim of the token the provider issues. Do not omit it — the API falls back to `*` (accept any token from this provider), which is almost never desired.

## Mutation endpoints

### Add a resource dependency to an application

Binds a Resource to an Application so the application can request credentials scoped to that resource.

```bash
keycard agent api /zones/<zone-id>/applications/<application-id>/dependencies/<resource-id> \
  -X PUT \
  --org <org-id>
```

No request body. Returns the updated dependency list on success.

### Delete an application credential

```bash
keycard agent api /zones/<zone-id>/application-credentials/<credential-id> \
  -X DELETE \
  --org <org-id>
```

Returns 204 on success.
