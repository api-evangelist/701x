---
name: 701x-record-calving
description: >-
  Record and amend calving events in the 701x digital calving book, including the bulk paths, with
  the retry hazards that follow from 701x publishing no idempotency mechanism.
api: 701x API V1
base_url: https://api.701x.com
operations:
- GET /AnimalCalving/AnimalCalvingDataByFilter
- POST /AnimalCalving
- GET /AnimalCalving/{id}
- PUT /AnimalCalving/{id}
- DELETE /AnimalCalving/{id}
- POST /AnimalCalving/MassInsert
- POST /AnimalCalving/MassUpdate
- POST /AnimalCalving/MassDelete
generated: '2026-09-05'
method: generated
source: openapi/701x-api-v1-openapi.json + conventions/701x-conventions.yml
---

# Record a calving event in 701x

## Before you write anything

**There is no idempotency mechanism on this API.** No `Idempotency-Key` header exists on any of
the 899 mutating operations, and the contract declares no request headers at all
(`conventions/701x-conventions.yml`, `idempotency.coverage: none`). A `POST /AnimalCalving` that
times out may or may not have been applied, and re-sending it will create a second record.

**There is also no reversal path for a calving record beyond delete.** No restore, undo or
unarchive operation is published, and no retention window is stated anywhere, so a
`DELETE /AnimalCalving/{id}` should be treated as permanent.

Therefore: before retrying any write, **read back first** with
`GET /AnimalCalving/AnimalCalvingDataByFilter` filtered on the same `CompanyId` plus
`DamAnimalId` / `CalfAnimalId` / `SeasonId`, and only retry if the record is genuinely absent.

## 1. Authenticate

OAuth 2.0 authorization code at `https://login.701x.com/connect/authorize`, token at
`/connect/token`, scope **`API701x`**. See `skills/701x-herd-inventory.md` step 1.

## 2. Create the record

`POST /AnimalCalving` with an `Animal_Calving` body (`components.schemas.Animal_Calving`). The
schema is entirely optional fields — nothing is marked required — and carries, among ~60
properties:

- Tenancy: `companyId`, `farmId`, `farmVirtualFenceId`, `seasonId`
- Parentage: `sireAnimalId`, `damAnimalId`, `fosterAnimalId`, `recipientAnimalId`, each paired
  with its `*CattleNumber`, `*AssociationCode` and `*RegistrationNumber`
- Season window: `calvingStartDate`, `calvingEndDate`

Because nothing is required, the API will accept an under-populated record. Validate on your side;
no `400` or `422` is declared on this operation, so malformed input has no documented signal.

## 3. Amend or remove

- `GET /AnimalCalving/{id}` — read one record
- `PUT /AnimalCalving/{id}` — replace it
- `DELETE /AnimalCalving/{id}` — remove it (no restore path exists)

## 4. Bulk paths

`POST /AnimalCalving/MassInsert`, `/MassUpdate` and `/MassDelete` take arrays of the same schema.
They carry the same no-idempotency exposure, multiplied: a partially applied bulk insert cannot be
identified from the response, which declares no body. Prefer batches small enough that a read-back
reconciliation is cheap.

## 5. Admin variants

`/AnimalCalvingControllerAdmin/*` mirrors every operation above. Use the plain `/AnimalCalving`
paths unless the principal is a company administrator; the admin twin returns `403` otherwise.
