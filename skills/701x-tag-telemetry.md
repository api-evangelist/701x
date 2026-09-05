---
name: 701x-tag-telemetry
description: >-
  Read GPS, activity, battery and water telemetry from 701x smart ear tags, subscribe to the
  server-sent change feed, and upload raw tag data files.
api: 701x API V1
base_url: https://api.701x.com
operations:
- GET /TagData/GetHerdTrackingDataByFilter
- GET /TagData/GetAllTrackingTagDataByFilter
- GET /TagData/TagDataStepCountByFilter
- GET /TagData/GetLastBatteryDataByFilter
- GET /TagData/GetLastWaterDataByFilter
- GET /TagData/GetLastActivityDataByFilter
- GET /AnimalLog/LatestAlertSummary
- GET /AnimalLog/GetAllSSE
- POST /TagDataFile/upload
- POST /TagDataFile/UploadTagEventsFile
generated: '2026-09-05'
method: generated
source: openapi/701x-api-v1-openapi.json + conventions/701x-conventions.yml
---

# Read 701x tag telemetry

## 1. Authenticate

OAuth 2.0 authorization code against `https://login.701x.com`, scope **`API701x`**.

## 2. Choose the read shape

All of these take the standard filter set — `top`, `orderBy`, `offset`, `fetch`, `CompanyId`,
`FarmId`, `AnimalId`, `TagId`, `FarmVirtualFenceId`, `UploadedByTagId`, plus the time window
parameters `GreaterThenCreatedByDate` and `LessThenCreatedByDate` (note the spelling — it is
`Then`, not `Than`, in the published contract).

- Herd positions over a window: `GET /TagData/GetHerdTrackingDataByFilter`
- Every tracking point: `GET /TagData/GetAllTrackingTagDataByFilter` (and `...Raw`)
- Activity: `GET /TagData/TagDataStepCountByFilter`, `GET /TagData/GetLastActivityDataByFilter`
- Device health: `GET /TagData/GetLastBatteryDataByFilter`, `GET /TagData/GetAllRSSITagDataByFilter`
- Water sensors: `GET /TagData/GetLastWaterDataByFilter`
- Heat map: `GET|POST /TagData/GetHeatMapDataByFilter`

Always send a bounded time window. Telemetry collections have no total count and no cursor, and an
unbounded pull over a herd is the one call on this API most likely to be very large.

## 3. Alerts and the change feed

- `GET /AnimalLog/LatestAlertSummary` takes `CompanyId`, `LogTypeIds`, `DateAfter`, `DateBefore`.
- `GET /AnimalLog/GetAllSSE` takes `companyId` and is a server-sent-event stream. The contract
  declares it as a plain `200` with **no media type and no event schema**, so the payload shape is
  undocumented — read defensively and do not assume a stable event envelope. The same applies to
  `/Company/GetAllSSE`, `/Company/GetAllChangesSSE`, `/Company/FullPullSSE` and
  `/LogMaster/GetAllSSE`.

701x publishes **no AsyncAPI document and no outbound webhook catalog**. The single `/webhook`
path in the contract is an inbound Stripe receiver (`tag: StripeWebHook`), not a subscription
endpoint for integrators.

## 4. Uploading raw tag data

`POST /TagDataFile/upload` and its siblings (`UploadTagEventsFile`, `UploadFileWithoutHeader`,
`UploadFileHighSpeedWithoutHeader`, `UploadFileDiagnostics`, `UploadFileOnBehalf`) are the only
operations in the whole API that carry an `operationId`, and the only ones that declare a `400`
("Bad Request", body is a bare JSON string) and a `503` ("Server Error").

On `503`, retry with backoff — but note there is no idempotency key, so a retried upload can be
ingested twice. Reconcile with a bounded `GET /TagData/TagDataByFilter` read-back rather than
retrying blind.
