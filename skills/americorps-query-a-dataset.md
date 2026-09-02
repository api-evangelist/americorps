---
name: americorps-query-a-dataset
description: >-
  Query rows from an AmeriCorps dataset with SoQL — filter, project, aggregate and page —
  and export in bulk as CSV. Requires a four-by-four identifier from americorps-find-a-dataset.
api: AmeriCorps Datasets API
base_url: https://data.americorps.gov
operations:
  - getDatasetJson
  - getDatasetCsv
  - getViewMetadata
generated: '2026-09-02'
method: generated
source: >-
  Grounded in the operationIds declared in openapi/americorps-datasets-api-openapi.yml and
  openapi/americorps-metadata-api-openapi.yml, and in responses captured live from
  data.americorps.gov on 2026-09-02.
---

# Query an AmeriCorps dataset

No credential required. Every operation is a GET; nothing here can change state.

## Step 1 — learn the shape in one call

```
GET https://data.americorps.gov/resource/{dataset_id}.json?$limit=1
```

`operationId: getDatasetJson`. Read the response **headers**, not just the body:

- `X-SODA2-Fields` → `["code","all","asn","nccc","vista"]`
- `X-SODA2-Types` → `["text","number","number","number","number"]`

That is the whole schema, for free. (`getViewMetadata` returns the same information in the
body if you prefer it.) **The row schema is not in the OpenAPI** — it is per-dataset and
must be discovered this way.

## Step 2 — query with SoQL

```
GET https://data.americorps.gov/resource/{dataset_id}.json
      ?$select=code,all
      &$where=all > 0.8
      &$order=all DESC
      &$limit=1000
      &$offset=0
```

- `$select` — projection. Fetch only the columns you need.
- `$where` — filter. Types must match `X-SODA2-Types` or you get a
  `soql.analyzer.typechecker.type-mismatch`.
- `$order` — **effectively required whenever you use `$offset`.** Without a stable sort,
  paging a mutating dataset can repeat or skip rows.
- `$group` — server-side aggregation. Do it here rather than pulling rows and aggregating
  locally.
- `$q` — full-text search.
- `$limit` — max 50000. `$offset` — page cursor.

## Step 3 — handle the type trap

`/resource` returns numeric columns as JSON **strings**: `{"all":"0.9035"}`, even though
`X-SODA2-Types` says `number`. Coerce them. If you would rather have real numbers, the same
rows are available typed correctly at `https://data.americorps.gov/api/odata/v4/{dataset_id}`
(OData v4), which also adds a row identifier `__id`.

## Step 4 — bulk export

```
GET https://data.americorps.gov/resource/{dataset_id}.csv?$limit=50000&$offset=0
```

`operationId: getDatasetCsv`. Use this rather than paging JSON when you want the whole
dataset.

## Step 5 — do not re-fetch what has not changed

`/resource` returns `ETag` and `Last-Modified`. Send `If-None-Match` or
`If-Modified-Since` on every repeat call. Most AmeriCorps datasets change a few times a
year, so this is the single largest saving available. `X-SODA2-Truth-Last-Modified` and
`X-SODA2-Data-Out-Of-Date` give you the same signal in-band.

## Errors — there are two envelopes, not one

A bad SoQL clause (`400`):

```json
{"message":"Query coordinator error: query.soql.no-such-column; No such column: nosuchcolumn",
 "errorCode":"query.soql.no-such-column",
 "data":{"column":"nosuchcolumn","dataset":"juliett.212871","position":{"row":1,"column":8}}}
```

A missing dataset (`404`):

```json
{"code":"dataset.missing","error":true,"message":"Not found","data":{"id":"zzzz-zzzz"}}
```

Note `errorCode` versus `code`. A parser keyed on one silently reads `null` from the other.
Neither is RFC 9457 `application/problem+json`.

On `400`, fix the query — do not retry it. On `429`, back off exponentially and register a
free Socrata app token (`X-App-Token`); no rate-limit headers are returned, so budget
blind. On `202`, retry the same request — the query is still materializing.

## Rules

- Read-only, so there is nothing to make idempotent, rehearse, or reverse. `$limit=1` is the
  closest thing to a dry run.
- Never invent a four-by-four. Resolve it with `americorps-find-a-dataset`.
- Quote `X-Socrata-RequestId` from the response when reporting a problem.
