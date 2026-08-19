---
name: Export raw user and event data out of CrossEngage
description: Run the asynchronous Raw Export API end to end — pick a destination, request an export, poll it to terminal state or receive the completion callback, and cancel a running job.
api: openapi/crossenagage-raw-export-v1-openapi.yml
operations: [getAllDestinationsAndTheirDetails, getACertainDestinationAndItsDetails, getAllEventClasses, requestAnExportUsers, getAllExportsAndTheirDetails, getACertainExportSDetails, cancelAnExport]
---

# Export raw user and event data out of CrossEngage

This is the data-out path: bulk user and event data leaving the platform for a warehouse.

## Setup

- Base URL: `https://api.crossengage.io`.
- Headers: `X-XNG-AuthToken: <Master API key>`, `X-XNG-ApiVersion: 2`.

## The destination comes first

You cannot export to an arbitrary URL. Exports write to a destination that already exists on the
account.

1. `getAllDestinationsAndTheirDetails` — `GET /destination`. Filter by `type`.
   - `EXPORT` — an (S)FTP integration. **Required.** If none exists, a human must add it in the
     app under Settings -> Integrations -> Add a new integration -> CrossEngage (S)FTP Upload.
     There is no API to create one.
   - `CALLBACK` — optional webhook fired when the export finishes.
2. `getACertainDestinationAndItsDetails` — `GET /destination/{id}` for one of them.
3. `getAllEventClasses` — `GET /event-class` to see which event classes you may export.

Passing an invalid destination id to an export request is a validation error, not a runtime
failure — check first.

## Running an export

1. `requestAnExportUsers` — `POST /export`, carrying the export destination id and, optionally, a
   callback destination id.
2. `getAllExportsAndTheirDetails` — `GET /export`, filterable by `type` and `status`, paginated
   with `pageNumber` and `pageSize`.
3. `getACertainExportSDetails` — `GET /export/{exportId}` to poll one job.
4. `cancelAnExport` — `DELETE /export/{exportId}` to stop a running job.

## Know the data-freshness rule before you trust a window

CrossEngage deliberately holds back recent data: **no event with a timestamp inside the last
3 hours before export start is included.** This exists so late or out-of-order events do not get
skipped. If you schedule hourly exports and stitch them, you will see an apparent 3-hour lag —
that is the contract, not a bug. Plan windows accordingly.

## Getting told it finished, instead of polling

Register a `CALLBACK` destination and CrossEngage POSTs on completion. The endpoint must be
`POST`, accept JSON, and use Basic authentication, or it will not appear in the destination list.

```json
{
  "exportId": "5d90695a-c61f-453a-bd2d-7c4f0b8536cf",
  "exportDestinationId": "c88c0ddf-d3e3-4190-b1f8-df3dfbc7238f",
  "callbackDestinationId": "9f86d4c8-d902-497d-8d78-53ce184ac3ce",
  "status": "FINISHED",
  "finishedAt": "2019-08-07T12:24:06.772Z"
}
```

`status` is `FINISHED` or `FAILED`. No signature or HMAC is documented on this callback, so
authenticate it yourself: the Basic credentials you configured are the only proof of origin.
Treat the payload as a trigger to go read `getACertainExportSDetails`, not as trusted state.

## Errors

Numeric CrossEngage error codes; see `errors/crossenagage-error-codes.yml`. Retry 5xx with
exponential backoff. `POST /export` has no idempotency key — a blind retry can start a second
export, so list exports first and match on destination and time before resending.
