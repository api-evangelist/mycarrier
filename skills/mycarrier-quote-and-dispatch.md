---
name: mycarrier-quote-and-dispatch
description: Rate an LTL shipment across MyCarrier's carrier partners and dispatch the chosen quote, using the MyCarrier Public API.
api: mycarrier-public-api
base_url: https://api.mycarriertms.com
operations:
  - GetRates
  - DispatchQuote
  - ShipmentDetails
generated: '2026-08-26'
method: generated
source: openapi/mycarrier-public-api-openapi.json
---

# Quote and dispatch an LTL shipment

This is MyCarrier's core flow: price a shipment across carriers, then commit one.
The two steps are deliberately separate operations, which means you can always
rehearse the price before you commit freight.

## Before you start

- Authentication is HTTP Basic. Username is the account admin email, password is the
  Order API Key from Customer Settings in the MyCarrier app. HTTPS is required.
- Work against the sandbox host first (`https://order-public-api.preprod.mycarrier.dev`
  for orders). MyCarrier provisions sandbox access and the key on request.
- Optionally send a `Traceparent` header (W3C Trace Context) so the call can be
  correlated on MyCarrier's side.

## Step 1 — rate the shipment

`GetRates` — `POST /api/v1/quote/rate`

Supply origin and destination, handling units with weight and dimensions, and the
commodities on each unit. Classify freight with its NMFC code (`commodityNMFC`,
`commodityNMFCSubCode`) rather than free text. If any commodity is hazardous, set
`commodityHazMat` and supply `hazmatHazardClass` (FMCSA), `hazmatIDNumber` (UN/NA
format), `hazmatPackingGroup` and `hazmatProperShippingName` — these are regulated
fields, not optional metadata.

Leave the carrier SCAC list null to request all carriers, or pass a comma-separated
SCAC list to narrow it.

The response is a `RatingResponse` with a `rates[]` array. Each rate item carries the
carrier SCAC, service level, lane type, a `priceDetails` breakdown of charge items, and
appointment/transit estimates.

**Read `statusInfo` before you read the rates.** Each entry in `statusInfo.notes[]`
carries a `code` (e.g. `VALIDATION_001`), a `message`, a `status`
(`Success` | `Info` | `Warning` | `Error`) and — critically — a `source`
(`Carrier` | `MyCarrier` | `ThirdParty` | `Validation`). That `source` field is the only
way to tell a carrier declining to quote from your own payload failing validation.
Treat them differently: a `Validation` error means fix the request; a `Carrier` error
means try another carrier.

To keep the quote for later, use the `RATE_AND_SAVE` action, which returns a `quoteId`
and `quoteKey`.

## Step 2 — dispatch the chosen quote

`DispatchQuote` — `POST /api/v1/quote/dispatch`

Handles dispatch, manual dispatch and edit-dispatch. Pass the shipment id only when
editing an existing dispatch.

The response returns `bolNumber`, `proNumber`, `proNumberCheckDigit`, `pickupNumber`
and a `securityKey` used to authorize BOL and label download links.

**This step is not reversible through the API.** Dispatch tenders freight to a real
carrier. MyCarrier supports cancellation — it emits a `shipment.canceled` webhook and
the shipment model has an `isCanceled` flag — but no public API operation performs the
cancellation; it is done in the MyCarrier UI. Confirm intent before calling this.
There is also no idempotency key on dispatch, so a retry after an ambiguous timeout can
double-tender. Verify with `ShipmentDetails` before retrying.

## Step 3 — confirm

`ShipmentDetails` — `GET /api/v1/shipments/{id}`

Accepts any shipment identifier — shipment id, BOL number, PRO number or your own
customer reference number. Use this to confirm a dispatch landed rather than blindly
retrying.

## Errors

400 and 401 are declared on both operations; 404 on `GetRates`; 500 on all. Rate limits
exist but are unpublished — on a 429, back off with a progressively increasing delay.
429 is not declared in the contract, so a generated client will not expect it.
