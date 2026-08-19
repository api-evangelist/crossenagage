---
name: Maintain the CrossEngage product catalogue
description: Keep the product feed backing CrossEngage personalization current — page through products, upsert and delete by SKU — using the one API CrossEngage publishes as a real Swagger document.
api: openapi/crossenagage-product-feed-openapi.yml
operations: [getProductsPage, getProductBySku, updateOrCreateProduct, deleteProductBySku]
---

# Maintain the CrossEngage product catalogue

The product feed is what CrossEngage message personalization reads from — the product helper
functions in templates ("fetch from product feed") resolve against this catalogue. A stale feed
shows customers stale products.

This is the only CrossEngage API published as a native Swagger 2.0 document rather than an API
Blueprint, so the schemas here are the provider's own.

## Setup

- Base URL: `https://api.crossengage.io/product-feed/v1` (this API carries the version in the
  path **and** still requires the header).
- Headers: `X-XNG-AuthToken: <Master API key>`, `X-XNG-ApiVersion: 1`.

## Operations

- `getProductsPage` — `GET /product`. Paginated with `pageNumber` and `pageSize`; the default is
  page 0 with 10 products. Walk it to reconcile your catalogue against theirs.
- `getProductBySku` — `GET /product/{sku}`. SKU is the identity.
- `updateOrCreateProduct` — `PUT /product/{sku}`. Upsert. Safe to repeat.
- `deleteProductBySku` — `DELETE /product/{sku}`.

## Choosing this API over file upload

CrossEngage offers both a real-time API and (S)FTP/file feeds for products. The provider's own
guidance: use the API when updates need to be frequent or immediate, and files for bulk loads.
For a nightly full catalogue rebuild, use the file path; for price and stock changes during the
day, use `updateOrCreateProduct`.

## Field rules

Custom attributes are supported but must be **string typed** — this is stated in the spec.
Several fields cap at 200 characters (`maxLength: 200` in the schema); check
`openapi/crossenagage-product-feed-openapi.yml` before pushing long descriptions rather than
discovering it as a 400.

## Errors

Same envelope as the rest of the estate: numeric CrossEngage `code` plus `title` and `details`.
Retry 5xx with exponential backoff; bounded retries without backoff otherwise. There is no
idempotency key, but every write here is a PUT or DELETE keyed on SKU, so replay is safe.
