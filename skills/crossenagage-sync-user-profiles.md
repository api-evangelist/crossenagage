---
name: Sync customer profiles into CrossEngage
description: Create, update, look up and delete CrossEngage user profiles through the asynchronous User Management v2 API, including 1,000-user batches and tracking-id status polling.
api: openapi/crossenagage-user-management-v2-openapi.yml
operations: [createUpdateASingleUser, createUpdateDeleteMultipleUsers, getASingleUserWithId, getASingleUserWithEmailAndBu, deleteASingleUser, retrieveStatus]
---

# Sync customer profiles into CrossEngage

CrossEngage is a customer data platform. This skill covers the primary write path: getting
first-party customer profiles into the platform so they can be segmented and messaged.

## Before you call anything

- Base URL is `https://api.crossengage.io`.
- Every request needs **two** headers, not one:
  - `X-XNG-AuthToken: <Master API key>` — issued in the app under Settings -> System setup -> API keys.
  - `X-XNG-ApiVersion: 2` — CrossEngage versions by header, not by URL path. Omitting it does not
    default to v2.
- `Content-Type: application/json`. All dates are ISO 8601 UTC with the `Z` designator.

## Single profile

1. `createUpdateASingleUser` — `PUT /users`. Upsert by the external id you own. Because it is a
   PUT keyed on your identifier, it is safe to repeat with the same body.
2. Read back with `getASingleUserWithId` (`GET /users/{id}`) or, when you only hold an email,
   `getASingleUserWithEmailAndBu` (`GET /users/email/{email}/bu/{bu}`) — `bu` is the business unit
   configured on the account, and it is required, not optional.
3. `deleteASingleUser` — `DELETE /users/{id}`.

## Batches

`createUpdateDeleteMultipleUsers` — `POST /users/batch`. The body carries `updated[]` and
`deleted[]` arrays.

- **Hard cap: 1,000 users per call.** Exceeding it returns CrossEngage error code `9`,
  "Max batch request size violation", with the violated bound quoted in `details`.
- The call is **asynchronous**. A 202 means accepted, not applied.

## Confirming asynchronous work

v2 is asynchronous by design — this is the stated difference from v1. Do not treat a 2xx as
"the profile is live".

1. Keep the `trackingId` returned by the write.
2. Poll `retrieveStatus` — `GET /users/track/{trackingId}` — until it reports a terminal state.

## Errors and retries

Errors come back as a CrossEngage JSON envelope with a numeric `code`, a `title` and a `details`
string — **not** RFC 9457 problem+json. The full registry is in
`errors/crossenagage-error-codes.yml`. The ones you will actually hit:

| code | meaning | what to do |
|---|---|---|
| 2 | Invalid field value or missing required field | fix the payload; do not retry unchanged |
| 6 | Unknown Attribute | create it with `createAttribute` (v1) or ask support to provision it |
| 7 | Identifier Mismatch | the path id and the body id disagree |
| 8 | Invalid Business Unit Format | use a `bu` configured on the account |
| 9 | Max batch request size violation | split the batch below 1,000 |

Retry `500`, `502`, `503`, `504` with exponential backoff. Retry other classes a bounded number
of times (the provider suggests ten or fewer) and **without** backoff.

## Idempotency warning

**CrossEngage documents no idempotency key.** `PUT /users` is safe to repeat because it is keyed
on the resource. `POST /users/batch` is **not** — a retried batch after a timeout can apply twice.
Before retrying a batch, poll the tracking id from the original attempt; only resend if it is
genuinely absent.

## No rate limits are published

There are no documented request-rate limits, no `RateLimit-*` headers and no `429` on any
operation. Pace conservatively and back off on 5xx. Every response carries `X-Request-ID` —
log it; it is what support will ask for.
