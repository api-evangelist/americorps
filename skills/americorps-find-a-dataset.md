---
name: americorps-find-a-dataset
description: >-
  Find the right AmeriCorps open-data dataset and resolve its four-by-four identifier, which
  every other AmeriCorps call needs. Use this before any query.
api: AmeriCorps Catalog API
base_url: https://data.americorps.gov
operations:
  - listViews
generated: '2026-09-02'
method: generated
source: >-
  Grounded in the operationIds declared in openapi/americorps-catalog-api-openapi.yml and in
  responses captured live from data.americorps.gov on 2026-09-02.
---

# Find an AmeriCorps dataset

AmeriCorps publishes 534 public datasets. Every other operation is addressed by a Socrata
four-by-four identifier (`^[a-z0-9]{4}-[a-z0-9]{4}$`, e.g. `fzpw-9z8s`), so this is always
step one. **Never hard-code a four-by-four** — they are reissued when a dataset is
republished, and a retired one starts returning `404 dataset.missing` with no advance signal.

No credential is needed for any step.

## Step 1 — prefer the DCAT catalog for searching

```
GET https://data.americorps.gov/data.json
```

This is not an OpenAPI operation, but it is the better search surface: a DCAT-US 1.1
`dcat:Catalog` where every `dcat:Dataset` carries `title`, `description`, `keyword[]`,
`issued`, `modified`, `accessLevel` and a `contactPoint`. Match on `title`/`description`/
`keyword`, then take the four-by-four from the tail of `identifier`
(`https://data.americorps.gov/api/views/23sc-gdgq` → `23sc-gdgq`).

## Step 2 — or list assets through the API

```
GET https://data.americorps.gov/api/views?limit=200&page=1
```

`operationId: listViews`. Returns a **bare JSON array** — no envelope, no total, no cursor.
Page with `limit` (max 200) and `page` until a short page comes back.

Fields worth reading: `id` (the four-by-four), `name`, `description`, `category`, `tags`,
`assetType` (filter to `dataset` — the catalog also contains charts, filtered views and
other assets), `rowsUpdatedAt` (unix **seconds**, not ISO-8601).

## Step 3 — confirm before you commit

```
GET https://data.americorps.gov/api/views/{dataset_id}.json
```

`operationId: getViewMetadata`. Check the column list and that `rowsUpdatedAt` is recent
enough for your purpose. Then hand the id to `americorps-query-a-dataset`.

## Failure modes

- `404` with `{"code":"dataset.missing","error":true,"message":"Not found","data":{"id":"..."}}`
  — the four-by-four does not exist on this domain. Re-resolve it; do not retry.
- `429` — you are throttled. Register a free Socrata application token and send it as
  `X-App-Token`. No rate-limit headers are returned, so you cannot see this coming.
- `500` — retry with backoff, then check https://status.socrata.com ("High Compliance -
  United States" region).

## Rules

- Read-only. Nothing in this skill changes any state.
- Cache the catalog. It changes on the order of days, not seconds, and `Crawl-delay: 1` in
  the portal's robots.txt is the only pacing figure AmeriCorps publishes.
