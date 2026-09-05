---
name: 701x-herd-inventory
description: >-
  Authenticate against the 701x identity server and page a company's cattle inventory out of the
  701x API V1, including group, farm and tag filters.
api: 701x API V1
base_url: https://api.701x.com
operations:
- GET /Animal/AnimalDataByFilter
- GET /Animal/{id}
- POST /Animal/GetByIds
- GET /AnimalGroup/AnimalGroupDataByFilter
- GET /Farm/FarmMasterDataByFilter
generated: '2026-09-05'
method: generated
source: openapi/701x-api-v1-openapi.json + conventions/701x-conventions.yml
---

# Read a 701x herd inventory

The 701x contract publishes no operationIds for these paths (7 of 1,391 operations have one), so
every step below names the HTTP method and path exactly as they appear in
`openapi/701x-api-v1-openapi.json`.

## 1. Get a token

All of these operations declare `security: [{oauth2: [API701x]}]`.

- Authorization: `https://login.701x.com/connect/authorize`
- Token: `https://login.701x.com/connect/token`
- Scope to request: **`API701x`** — not `api1`. The contract's `securitySchemes.oauth2` documents
  `api1` ("Demo API - full access") and the issuer's `scopes_supported` lists `api1`, but no
  operation requires it. Requesting `api1` alone will not authorize these calls.
- PKCE is supported (`S256`); prefer it over `plain`.
- Send `Authorization: Bearer <access_token>`.

## 2. Find the tenant

Every collection filter is keyed on `CompanyId` (a UUID). `Company_Master` is the account root of
the whole data model — see `data-model/701x-data-model.yml`. Narrow further with `FarmId` for a
single ranch unit.

## 3. Page the inventory

`GET /Animal/AnimalDataByFilter` accepts offset/limit paging plus resource filters:

- Paging: `top` (page size), `offset`, `fetch`, `orderBy`
- Scope: `CompanyId`, `FarmId`, `GroupId`, `TagId`
- Animal filters: `Status`, `Sex`, `Age`, `Breed`, `CattleNumber`, `MaleAnimalId`, `FemaleAnimalId`

**No total count is returned.** Collection responses are bare arrays with no envelope, no
`total`, no `next` link and no cursor. Page until a short page comes back; do not expect to know
how many pages remain.

## 4. Fetch specific animals

- One record: `GET /Animal/{id}`
- A batch: `POST /Animal/GetByIds` with a JSON array of UUIDs in the body and `CompanyId` as a
  query parameter.

## 5. Related lookups

- Groups: `GET /AnimalGroup/AnimalGroupDataByFilter`
- Farms and pastures: `GET /Farm/FarmMasterDataByFilter`

## Error handling

The contract declares only `401` and `403` on these operations, with **no response body of any
kind** (see `errors/701x-problem-types.yml`).

- `401` — token missing, expired, or issued without `API701x`. Refresh and retry.
- `403` — token is valid but the principal lacks rights on that `CompanyId`. Many resources have
  a `*ControllerAdmin` twin; calling the admin variant as a non-admin is the common cause.
- `404` is **not declared anywhere**, so do not branch on it — treat an empty array as "not found"
  for filter operations.
- No `429` is declared and no rate-limit headers are published
  (`rate-limits/701x-rate-limits.yml`). Rate-limit behaviour is unknown; back off conservatively.
