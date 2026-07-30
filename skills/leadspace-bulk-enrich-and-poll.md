---
name: Bulk enrich records with Leadspace and collect the results
description: Submit up to 500 records per batch for asynchronous enrichment, then collect results by polling or by receiving the completion callback, without double-spending credits.
api: openapi/leadspace-enrichment-openapi.yml
operations:
  - createAuthorizationToken
  - enrichBulkRecords
  - getBulkEnrichmentResults
  - bulkCompleteCallback
generated: '2026-07-19'
method: generated
source: https://support.leadspace.com/hc/en-us/articles/23687991628572-Leadspace-Bulk-API-v4-Technical-Specifications
---

# Bulk enrich with Leadspace

Use this for list refreshes, CRM backfills, and any job over a few dozen
records. Bulk is one API transaction per 500 records, so it clears far more
volume than `enrichSingleRecord` under the same 30 TPS ceiling.

## 1. Authenticate

Same as the single-record skill: either the perpetual bearer token, or
`createAuthorizationToken` (`POST /oauth/authorize`) with `user`, `pass`, and
`audience: API_GATEWAY`. Cache the 24-hour token.

## 2. Chunk to 500

`enrichBulkRecords` accepts a maximum of **500 records per POST** in `data[]`.
Chunk your input and submit sequentially. Leadspace holds up to 100,000 leads
in its processing backlog, so you can queue several batches, but respect the
30 TPS submission ceiling.

Give every record its own `external_id` and every batch an
`external_bulk_id`. Both are echoed back and are your only reconciliation
handle — Leadspace does not preserve input order guarantees for you.

## 3. Submit

`POST /enrichment/enrich/bulk`:

```json
{
  "data": [
    { "person": { "first_name": "", "last_name": "", "email": "" },
      "company": { "name": "" },
      "external_id": "crm-1" }
  ],
  "callbackUrl": "https://your-host/leadspace/complete",
  "external_bulk_id": "refresh-2026-07-19"
}
```

A 202 returns `{ "id": "/v3/enrichment/results/<bulkId>" }`. **Persist that
bulk id immediately, before doing anything else.** It is the only way to
collect results you have already paid for. Losing it means resubmitting and
re-spending credits — there is no idempotency key to protect you.

## 4. Collect results — two paths

**Callback (preferred).** If you supplied `callbackUrl`, Leadspace POSTs to it
on completion:

```json
{
  "bulkId": "...",
  "callbackMethod": { "callbackUrl": "...", "pollingUrl": "..." },
  "bulkStatus": "COMPLETED",
  "successRecords": 480,
  "personEnriched": 431,
  "companyEnriched": 476
}
```

Branch on `bulkStatus`:

- `COMPLETED` — GET the `pollingUrl`.
- `INSUFFICIENT_CREDITS` — stop and alert an operator. Do not resubmit.
- `INTERNAL_ERROR` — resubmit the batch.

The callback is unauthenticated and unsigned. Use an unguessable callback URL
and treat the payload as a trigger to fetch, never as trusted data.

**Polling (fallback).** Call `getBulkEnrichmentResults` —
`GET /v3/enrichment/results/{bulkId}`:

- `200` — results are ready, read `results[]`.
- `204` — still processing. Wait and poll again with backoff; **204 is not an
  error and not an empty result.**
- `404` — results not found. Check the bulk id.
- `401` — refresh the token.

## 5. Process records individually

Each entry in `results[]` is a full enrichment result. A batch that returns 200
still contains individual failures — a valid bulk with a bad record does not
fail the HTTP call. Check per record:

- `status` and `data.enrichment_status`
- `data.company.matching_confidence.level` — gate `LOW`/`VERY LOW` writes
- record-level error codes: **1100** Internal Error (retriable — resubmit just
  that record) and **1200** Invalid Input (not retriable as submitted — the
  record lacked required fields such as first and last name)

Reconcile back to your system using `external_id`, not position.

## 6. Never blind-retry a batch

Every resubmission consumes credits again for every record. Before resubmitting
anything, poll the original `bulkId` — if it returns 204 the job is still
running, and if it returns 200 you already have the results.
