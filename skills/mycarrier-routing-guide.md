---
name: mycarrier-routing-guide
description: Manage MyCarrier routing guide rules — the lane-to-carrier policy that decides which carrier a shipment should go to.
api: mycarrier-public-api
base_url: https://api.mycarriertms.com
operations:
  - FetchRoutingGuide
  - MatchRoutingGuide
  - UpsertRoutingGuide
  - DeleteRoutingGuide
  - UploadRoutingGuide
  - DownloadRoutingGuide
  - DownloadRoutingGuideTemplate
generated: '2026-08-26'
method: generated
source: openapi/mycarrier-public-api-openapi.json
---

# Manage the routing guide

The routing guide is MyCarrier's carrier-selection policy: rules that bind an
origin/destination lane to a preferred carrier and service, and exclude others.

## Read the policy

`FetchRoutingGuide` — `GET /api/v1/routing-guide/fetch` — returns all saved rules for the
customer. Each rule carries `id`, `carrierSCAC`, `carrierName`, `carrierService`,
`originState`/`originZip` and `destinationState`/`destinationZip`.

## Ask the policy a question

`MatchRoutingGuide` — `POST /api/v1/routing-guide/match` — filters the guide by your
parameters and returns the details of the **single** carrier that matches. This is the
operation to call before rating when the customer wants policy honoured rather than
lowest price: match first, then pass that SCAC to `GetRates`.

Rules also support a comma-separated list of SCACs to exclude for the lane.

## Change the policy

`UpsertRoutingGuide` — `POST /api/v1/routing-guide/upsert` — creates or updates a single
rule from JSON.

`DeleteRoutingGuide` — `DELETE /api/v1/routing-guide/delete/{id}` — removes one rule by
its integer id. There is no restore, and no window is published; re-create with upsert if
you delete in error.

## Bulk changes go through Excel, not JSON

This part of the API is spreadsheet-shaped and worth knowing before you plan an
integration:

- `DownloadRoutingGuideTemplate` — `GET /api/v1/routing-guide/download/template` —
  returns a standardized Excel template.
- `UploadRoutingGuide` — `POST /api/v1/routing-guide/upload` — uploads many rules at once
  as an Excel file.
- `DownloadRoutingGuide` — `GET /api/v1/routing-guide/download/rules` — exports all saved
  rules in Excel format.

These three return and consume binary Excel, not JSON, and are the only non-JSON
operations in the API. There is no JSON bulk-upsert endpoint, so a fully programmatic
bulk sync means generating a spreadsheet.

## Errors

All routing-guide operations declare 400, 401, 404 and 500, and return the
`{ data, errors[], statusCode }` envelope. The `errors` array is human-readable strings
with no machine-readable code.
