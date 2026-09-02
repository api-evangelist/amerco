---
name: amerco-move-in-quote-and-commit
description: >-
  Quote and then commit a self-storage move-in at a U-Haul Self-Storage Affiliate facility
  through the WebSelfStorage Affiliate API. This flow moves money and CANNOT be reversed
  through the API — read the guard rails before you call it.
api: WebSelfStorage Affiliate API (AMERCO / eMove, Inc.)
base_url: https://api.webselfstorage.com
version: v4
generated: '2026-09-02'
method: generated
source: openapi/amerco-webselfstorage-affiliate-api-v4-openapi.yml
operations:
  - GET /v4/movein/{entity}
  - GET /v4/movein/{entity}/cost
  - GET /v4/paymentPortalUrl/{entity}
  - POST /v4/movein/{entity}
  - POST /v4/reservation/{entity}
consequence: irreversible-write
---

# Quote and commit a move-in

## Stop and read this first

`POST /v4/movein/{entity}` creates a tenancy contract and charges the card in `paymentInfo`.
`POST /v4/reservation/{entity}` reserves units against a card.

- **There is no reversal.** The contract has no cancel, void, refund, reverse, `DELETE`, `PUT`
  or `PATCH` operation. AMERCO publishes no reversal window for either write. If you commit
  the wrong move-in, the fix is a human in the WebSelfStorage application or the affiliate's
  merchant services — not an API call.
- **There is no idempotency key.** No `Idempotency-Key` parameter or header exists. If a write
  times out you cannot safely retry it; a duplicate charge is the realistic outcome. Treat a
  timeout or a 500 on these two operations as *indeterminate* and reconcile out of band before
  retrying.
- **You are in PCI DSS scope** the moment you populate `PaymentInfo.creditCard` /
  `expirationMMYY` / `csc`. `GET /v4/paymentPortalUrl/{entity}` returns a hosted payment portal
  URL and is the scope-reducing alternative wherever the flow allows it.
- **Require explicit human confirmation** before either POST. Do not chain them off a model
  decision.

## Steps

1. **List what is available.**
   `GET /v4/movein/{entity}` → `SelfMoveInUnit[]`. Pick a `unitId` from `units[]` (or the
   `rentableObjectId` for the type) and, if the tenant wants cover, an `insuranceId` from
   `insuranceOptions[]` (`InsuranceOption` carries `monthlyRate`, `due`, `tax`, `total`,
   `insuranceCoverageType`).

2. **Rehearse the money — this is the only dry run the API offers.**
   `GET /v4/movein/{entity}/cost` with query parameters `UnitId`, `InsuranceId`,
   `ExpectedMoveInDate`, `IsTaxExempt`, `TaxExemptNumber`, `OneMonthFreeDiscount`.
   → `MoveInCostResponse` with `totalCost` and a `costBreakDown`. Show this to the human and
   get confirmation of the exact number.

3. **Commit.**
   `POST /v4/movein/{entity}` with `MoveInRequest`: `unitID`, `insuranceId`, `moveInDate`,
   `payAmount` (use the confirmed `totalCost`), `gatePassword`, `isTaxExempt`/`taxNumber`,
   `oneMonthFreeDiscount`, `saveCardForFutureUse`, `enrollIntoAutopay`, `paymentInfo`,
   `alternatePhoneNumber` and `alternateContact` (`Customer`).
   → `MoveInResponse`: `contractNumber`, `unitNumbers[]`, `gatePassword`, `moveInDirections`,
   `rentalAgreementHtml`.
   **Persist `contractNumber` immediately.** It is the only handle you get back, and there is
   no lookup operation to recover it later.

4. **Reservation instead of move-in.**
   `POST /v4/reservation/{entity}` with `ReservationRequest`: `reservationDay`, `units[]`
   (`UnitReservation`), `paymentInfo`. → `ReservationResponse`: `reservationNumber`, `entity`,
   `moveInDate`, `totalCost`. Same guard rails apply. Check
   `LocationInformation.allowsReservations` and `reservationDates[]` before you offer this.

## Error handling

- `400` on `POST /v4/reservation/{entity}` returns `InvalidParameterResult` — the *only* error
  in the contract that names the offending field (`parameter`, `reason`). Surface both.
- `400` on `POST /v4/movein/{entity}` returns a `MoveInResponse` with `success: false` and
  `errorMessage` — same schema as success. Always branch on `success`, never on the presence of
  a body.
- `500` on either POST: **do not auto-retry.** See the indeterminate-write rule above.
- `401`: the gateway envelope, PascalCase, `application/problem+json` but not RFC 9457.
