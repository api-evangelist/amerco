---
name: amerco-rent-roll-reporting
description: >-
  Pull the rent roll and waiting list for a U-Haul Self-Storage Affiliate facility from the
  WebSelfStorage Affiliate API for occupancy, delinquency and demand reporting. Read-only, but
  both payloads are full of tenant PII.
api: WebSelfStorage Affiliate API (AMERCO / eMove, Inc.)
base_url: https://api.webselfstorage.com
version: v4
generated: '2026-09-02'
method: generated
source: openapi/amerco-webselfstorage-affiliate-api-v4-openapi.yml
operations:
  - GET /v4/locations
  - GET /v4/location/{entity}/rentroll
  - GET /v4/location/{entity}/waitinglist
  - GET /v4/location/{entity}
consequence: read-only-sensitive
---

# Rent roll and waiting list reporting

## Handling rule that comes before everything else

Both endpoints return **tenant personal data**, and the API gives you no way to request a
redacted view.

- `RentRoll`: `customerName`, `customerPhoneNumber`, `address1`, `address2`, `apartment`,
  `city`, `stateName`, `zip`, plus `balance`, `streetRate`, `insurance`, `dateMovedIn`,
  `paidThru`, `daysOccupied`, `roomNumber`, `contractType`, `contractUnitId`.
- `WaitingListItemViewModel`: `personFirstName`, `personLastName`, `emailAddress`, `homePhone`,
  `mobilePhone`, `businessPhone` (+ extension), `primaryAddress1`, `primaryApartment`,
  `primaryCity`, `primaryState`, `primaryZip`, plus `desiredSizeCode`, `desiredSizeCodeRate`,
  `dateNeeded`, `duration`, `notes[]`, `waitingListID`, `active`, `addedBy`, `createDate`.

Aggregate before you store. Do not put either payload into a prompt, a log, or a third-party
analytics tool without the affiliate's explicit instruction.

## Steps

1. `GET /v4/locations` → `entities[]`.
2. `GET /v4/location/{entity}/rentroll` → `RentRollResponse.` Derive occupancy from the row
   count against `LocationInformation.units[].totalUnits` (step 4), delinquency from `balance`
   against `paidThru`, and rate spread from `streetRate` against the published `monthly` on the
   matching `UnitType`.
3. `GET /v4/location/{entity}/waitinglist` → `WaitingListResponse`. Group by
   `desiredSizeCode` / `desiredSizeCodeDescription` to see which sizes are demand-constrained;
   cross-check `active`.
4. `GET /v4/location/{entity}` for the denominator: `units[]` with `totalUnits` and
   `vacantUnits` per `UnitType`.

## Rules

- **No pagination, no filtering, no date range.** Every call returns the whole set. There is no
  "changed since" parameter, so an incremental pipeline has to diff locally.
- **`contractUnitId` does not resolve.** It is a bare uuid with no lookup operation; to name the
  unit you must join against `units[]` from step 4 yourself.
- Both operations declare `400` (request failed / nothing found) and `500` (unknown server
  error) returning `BaseResponse`. Check `success` on every response.
- No `Retry-After` and no declared 429 — back off on your own schedule.
- Status of the underlying platform is published at https://status.uhaul.com/ (the
  "WebSelfStorage" component); `https://status.uhaul.com/api/v2/summary.json` is machine
  readable if you want to gate a nightly job on it.
