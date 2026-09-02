---
name: amerco-facility-inventory-sync
description: >-
  Pull current unit inventory, rates and availability for one or more U-Haul Self-Storage
  Affiliate facilities from the WebSelfStorage Affiliate API, so a website or listing feed can
  show real prices and real vacancy instead of a stale copy.
api: WebSelfStorage Affiliate API (AMERCO / eMove, Inc.)
base_url: https://api.webselfstorage.com
version: v4
generated: '2026-09-02'
method: generated
source: openapi/amerco-webselfstorage-affiliate-api-v4-openapi.yml
operations:
  - GET /v4/locations
  - GET /v4/location/{entity}
  - GET /v4/movein/{entity}
  - GET /v4/location/{entity}/images
  - GET /v4/location/{entity}/reviews
consequence: read-only
---

# Sync facility inventory from WebSelfStorage

The contract publishes **no operationIds**, so every operation below is named by its HTTP
method and path exactly as it appears in the spec. Do not invent a method name.

## Before you start

- You need an affiliate access token. Send it as `Authorization: Bearer <token>` on every
  request. There is no self-service issuance; keys come with the U-Haul Self-Storage Affiliate
  Network relationship.
- Smoke-test the credential with `GET /v4/test` ("Simulates a success"). A `401` returns
  `{"Value":{"Success":false,"ErrorMessage":"Unauthorized. Specify your API key in the
  Authorization header."},...}` — note the PascalCase; the in-contract envelope is camelCase.

## Steps

1. **Discover the facilities you may address.**
   `GET /v4/locations` → `LocationsResponse`. Read `entities[]` (integers). Everything else in
   the API requires one of these as the `{entity}` path segment; there is no organization-level
   resource, so a multi-facility operator loops.

2. **Read the facility profile.**
   `GET /v4/location/{entity}` → `LocationResponse.location` (`LocationInformation`): `name`,
   `address`, `phone`, `hoursOfOperation`, `facilityFeatures[]`, `locationFeatures[]`,
   `locationServices[]`, `coupons[]`, `allowsReservations`, `reservationDates[]`, and
   `units[]` (`UnitType`) with `unitSize`, `monthly`, `length`/`width`/`height`,
   `squareFootage`, `cubicFootage`, `totalUnits`, `vacantUnits`, `insuranceOptions[]`,
   `serviceCharges[]` and `roomSizeReservationRules`.

3. **Read live move-in availability.**
   `GET /v4/movein/{entity}` → `AvailableUnitResponse` carrying `SelfMoveInUnit` entries:
   `unitSize`, `monthly`, `vacantUnits`, `totalUnits`, `rentableObjectId`, `insuranceOptions[]`
   and the concrete `units[]` (`unitId` + `unitNumber`). Use this — not step 2 — as the source
   of truth for "can someone rent this right now".

4. **Optionally enrich.**
   `GET /v4/location/{entity}/images` and `GET /v4/location/{entity}/reviews`.

## Rules

- **Check `success` on every response, including 200s.** Every schema in this API carries
  `success` (boolean) and `errorMessage` (string), and a 400 returns the *same* schema with the
  payload empty rather than a distinct error body.
- **There is no pagination.** No limit/offset/page/cursor parameter exists anywhere. A large
  facility returns everything in one array; budget for it.
- **There is no rate-limit signal.** No `RateLimit-*` headers, no `Retry-After`, no declared
  429. Pace yourself conservatively and back off on 500s, which are declared on
  `/rentroll`, `/waitinglist`, `/movein` and `/movein/cost`.
- **Cache on your side.** Rates and vacancy change; the API publishes no ETag, no
  `Last-Modified` and no webhooks, so polling is the only option.

## Errors you will actually see

| Status | Meaning |
|---|---|
| 401 | No/!valid token. Gateway envelope, PascalCase, served as `application/problem+json` but not RFC 9457. |
| 400 on `/location/{entity}` | "No locations found or an error occurred for the requested entity" — the entity is not one of yours. |
| 400 on `/movein/{entity}` | "Invalid request". |
| 500 | "An unknown error occurred on the server". No `Retry-After`; back off yourself. |

See `errors/amerco-problem-types.yml` and `conventions/amerco-conventions.yml`.
