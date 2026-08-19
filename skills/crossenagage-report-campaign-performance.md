---
name: Report on CrossEngage campaign performance
description: Pull campaign, story and message statistics out of CrossEngage — discover the available KPIs, then request detailed or overall figures for up to 30 entities at a time.
api: openapi/crossenagage-statistics-v1-openapi.yml
operations: [getAllKPIs, getACertainKPI, getDetailedStatistics, getOverallStatistics]
---

# Report on CrossEngage campaign performance

The Statistics API is the read path for campaign, story and message performance. Unlike User
Management v2 and the Raw Export API, it is **synchronous** — you get numbers back on the call.

## Setup

- Base URL: `https://api.crossengage.io/statistics`.
- Headers: `X-XNG-AuthToken: <Public API key>`, `X-XNG-ApiVersion: 2`.
- Note the key: Statistics and File Attachments use the **Public** API key, while User
  Management, Product Feed and Raw Export use the **Master** key. Sending the wrong one is a 401,
  not a 403.

## Discover the KPIs before you ask for them

Do not hardcode metric names. KPI availability depends on the channels configured on the account.

1. `getAllKPIs` — `GET /kpi`. Returns the KPI catalogue.
2. `getACertainKPI` — `GET /kpi/{id}` for the definition of one.

Webhook-channel campaigns surface `Webhook Delivered`, `Webhook Bounced` and error counts here —
these are the same events described in `asyncapi/crossenagage-webhooks.yml`, which is how you
reconcile webhook delivery against campaign reporting.

## Pull the numbers

- `getDetailedStatistics` — `POST /detailed`. Per-entity breakdown.
- `getOverallStatistics` — `POST /overall`. Roll-up totals.

Both take an `entities[]` array where an entity is a campaign, story or message.

**Cap: 30 entities per request.** This is stated in the request schema. Chunk larger reporting
runs into groups of 30 and merge client-side; there is no server-side paging over entities.

## Errors and pacing

Standard CrossEngage envelope with a numeric `code`. Retry 5xx with exponential backoff, bounded
retries otherwise. No rate limits are published and no `429` is declared, so when you are
chunking a large reporting job, throttle yourself — 30 entities per call against an unbounded API
is an easy way to find an undocumented limit the hard way. Log the `X-Request-ID` from each
response.
