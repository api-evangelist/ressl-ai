---
name: ressl
description: >
  Create and call hosted mock SaaS APIs (any granted provider) via the Ressl
  platform API. Use when an agent or eval needs a live HTTPS mock of a SaaS
  product without a real tenant.
license: MIT
compatibility: Requires an org API key (rsk_…) from https://simulation.ressl.ai Settings.
metadata:
  author: ressl
  version: "1.0"
  control_plane: "https://simulation.ressl.ai"
  mock_host_pattern: "https://{snapshotId}.{slug}.mock.ressl.cc"
---

# Ressl mock SaaS snapshots

## When to use

- You need a **hosted mock** of a SaaS HTTP API for an agent, harness, or eval.
- The org already has access to that provider on Ressl.
- You want a normal HTTPS base URL that expires after a TTL.

## Do not confuse hosts

| Purpose | Base |
|---------|------|
| Control API (list, create) | `https://simulation.ressl.ai` |
| Mock traffic | `https://{snapshotId}.{slug}.mock.ressl.cc` |

Never invent mock hosts under `ressl.ai`. Mocks live on `*.mock.ressl.cc`.

## Auth

1. Human creates an API key in the console: Settings → API keys (`rsk_…`).
2. Send on every control-plane request:

```http
Authorization: Bearer rsk_...
```

Also accepted: `X-API-Key: rsk_...`.

Keys are org-scoped. Invalid/revoked → `401`.

## Workflow

### 1. List providers

```http
GET https://simulation.ressl.ai/providers/list
Authorization: Bearer rsk_...
```

Response:

```json
{ "providers": ["jira", "salesforce", "slack"] }
```

Use only slugs from this list.

### 2. Create a snapshot

```http
POST https://simulation.ressl.ai/providers/{slug}/create-snapshot
Authorization: Bearer rsk_...
Content-Type: application/json
```

Body (all optional):

```json
{
  "ttl": "1h",
  "seed": { "<slug>": { } },
  "config": {}
}
```

- `ttl`: `15m` / `1h` / `1d` style; default `1h`; max `7d`.
- `seed`: initial data; prefer `{ "<slug>": { … } }`. Bare objects are auto-wrapped. Empty/default is `{ "<slug>": {} }`.
- `config`: provider overrides. For **salesforce**, describe JSON that **fully replaces** baked metadata; omit to keep baked. Other slugs reject `config` today. Details: docs site `/providers/salesforce`.

Provisioning may take 1–2 minutes.

Success includes:

- `snapshotId`, `slug`, `url`, `expiresAt`, `ttl`

Errors: `400` bad input, `403` no grant, `502` provision failure.

### 3. Call the mock

Use `url` as the HTTP base. Paths match that provider’s real API. Snapshot URLs are public but unguessable until `expiresAt`.

## Constraints

- One provider per snapshot (`EMULATOR_TOOLS` = single slug).
- Org must be granted the slug (`403` otherwise).
- No list/delete snapshot HTTP APIs in v1 — TTL tears the sandbox down.
- Salesforce `config` replaces describes for that snapshot only; see `/providers/salesforce` on the docs site for seed/config templates.

## Agent tips

- Always call `/providers/list` before create if the allowed set is unknown.
- Persist `url` + `expiresAt` for the run; do not hardcode hosts.
- On `502`, retry create once after a short wait; surface `detail` if present.
