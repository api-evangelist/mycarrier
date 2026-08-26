---
name: mycarrier-order-integration
description: Push orders from an ERP or WMS into MyCarrier, keep them in sync by reference ID, and remove them — using the MyCarrier Order API.
api: mycarrier-public-api
base_url: https://api.mycarriertms.com
operations:
  - UploadOrder
  - Order
  - DeleteOrder
  - GetShippingLocations
  - GetShippingLocationByLocationId
generated: '2026-08-26'
method: generated
source: openapi/mycarrier-public-api-openapi.json
---

# Integrate orders from an ERP or WMS

Use this when MyCarrier is not the system of record. Your ERP or WMS owns the order;
MyCarrier receives it, rates it and moves the freight.

## The reference ID is the contract

`UploadOrder` — `POST /api/v1/orders` — is documented as "Create a new order or update
existing by Reference ID". The reference ID is **yours**: it is the key your system
supplies, and every other order operation addresses the resource through it
(`GET /api/v1/orders/referenceId/{referenceId}`,
`DELETE /api/v1/orders/referenceId/{referenceId}`).

This makes order upload **idempotent by natural key**. Re-sending the same payload with
the same reference ID updates the existing order rather than creating a duplicate.
MyCarrier has no `Idempotency-Key` header, so this upsert behaviour is the safety
mechanism you have — always send your own stable reference ID, never let MyCarrier
allocate one.

The order carries a `changeVersion` counter incremented on every change, so you can
detect that MyCarrier-side edits happened since your last read.

## Attach your own record id

Two optional connector headers exist specifically for integrations:

- `X-Mc-Meta-Customerinternalrecordid` — your internal record id for this request
- `X-Mc-Meta-Customerdevicename` — the connector device name

Send the first one on every write. It is how you reconcile a MyCarrier record back to a
row in your own system when something goes wrong.

## Resolve addresses before you send

`GetShippingLocations` — `GET /api/v1/address/shipping-locations` — lists the customer's
saved locations. It is the only paginated collection in the API: use `skip` and `take`,
and watch the `isTruncated` boolean in the response, because there is no total count.
`GetShippingLocationByLocationId` — `GET /api/v1/address/shipping-locations/{locationId}` —
returns one location with its default accessorials.

Prefer a saved `locationId` over a raw address; it carries accessorials and contact
details that affect rating.

## Reading back and removing

`Order` — `GET /api/v1/orders/referenceId/{referenceId}` — returns the order with its
`status` and `eventHistory` of status logs.

`DeleteOrder` — `DELETE /api/v1/orders/referenceId/{referenceId}` — removes it. MyCarrier
publishes no window for how long deletion remains possible, and there is no restore
operation. Deleting an order does not recall a shipment that was already dispatched from
it.

## Errors

Order operations return a `Result` envelope — `{ isSuccess, errorMessages[] }` — on 400.
This is a different shape from the `{ data, errors[], statusCode }` envelope the address,
routing-guide and shipment operations use, and different again from the `statusInfo`/
`notes[]` shape rating returns. Handle all three if you touch all three surfaces.
