---
name: Enrich a person or company record with Leadspace
description: Resolve a partial lead or account against the Leadspace Universal Graph and return firmographics, hierarchy, intent, and scores, handling credits and rate limits correctly.
api: openapi/leadspace-enrichment-openapi.yml
operations:
  - createAuthorizationToken
  - enrichSingleRecord
generated: '2026-07-19'
method: generated
source: https://support.leadspace.com/hc/en-us/articles/23687993264284-Leadspace-Single-Direct-API-v4-Technical-Specifications
---

# Enrich a single record with Leadspace

Use this for real-time, one-record-at-a-time enrichment — form submissions, CRM
record opens, agent lookups. For more than a handful of records at once, use
`leadspace-bulk-enrich-and-poll` instead: single calls are capped at 30 per
second.

## 1. Get a bearer token

Two options. If you were issued a perpetual token, skip to step 2 and use it
directly.

For OAuth 2.0, call `createAuthorizationToken` — `POST /oauth/authorize` with
`Content-Type: application/json`:

```json
{ "user": "<PROGRAM_ID>", "pass": "<AUTH_SECRET>", "audience": "API_GATEWAY" }
```

You get back `token`, `refreshToken`, and `expiration` (epoch seconds). The
token lasts 24 hours. Cache it and refresh with `refreshAuthorizationToken`
(`PUT /oauth/authorize`, body `{ "user", "refreshToken" }`) before expiry —
do not re-authorize on every call.

## 2. Check you have the minimum input

Leadspace returns 400 if the input is too thin. Before calling:

- **Account enrichment** requires `company.name`.
- **Person enrichment** requires `person.first_name`, `person.last_name`, and
  at least one of `company.name`, `person.email`, or `company.website`.

If you hold a `company.lsid` or `company.ugLsid`, send it — Leadspace matches
on the identifier first, which yields a better match than name resolution.

## 3. Call enrichSingleRecord

`POST /enrichment/enrich/single` with `Authorization: Bearer <token>`:

```json
{
  "person": { "first_name": "", "last_name": "", "email": "", "title": "" },
  "company": { "name": "", "website": "", "address": { "country": "" } },
  "external_id": "<your-crm-record-id>"
}
```

Set `external_id` to your own record identifier. Leadspace echoes it verbatim
onto the result so you can reconcile without holding state. It is **not** used
for processing and is **not** an idempotency key.

## 4. Read the result honestly

A 200 does not mean a match. Check, in order:

1. `status` — `success` or `failure`.
2. `data.enrichment_status` — `Not Enriched`, `Company Enriched`,
   `Person Enriched`, or `Person & Company Enriched`.
3. `data.company.matching_confidence.level` — `HIGH`, `MEDIUM`, `LOW`, or
   `VERY LOW`. Do not write `LOW`/`VERY LOW` matches into a system of record
   without review. A person matching confidence level is also returned for
   Universal Graph (v4) customers.
4. `data.person.original_email_verification_status` — `VALID`, `INVALID`,
   `UNKNOWN`, or `CATCH-ALL`. Never send mail to `INVALID`.
5. `data.person.verification_status` — `Verified`, `Moved`, `Not Verified`.
   `Moved` means the person has left the matched company.

Fields gated behind purchased entitlements come back empty when not licensed:
premium mobile phones (`person.person_mobile_phone_1..3`), premium hierarchy
node types (`PARENT`, `HQ`, `SU`), technology signals, personas, and
predictive `total_scores`.

## 5. Handle failures by class

| Status | Meaning | Do |
|---|---|---|
| 400 | Insufficient input or malformed JSON | Fix the payload — see step 2. Do not retry as-is. |
| 401 | Bad or expired credentials | Refresh the token once, then stop. |
| 415 | Bad or missing headers | Send `Content-Type: application/json`. |
| 427 | Insufficient credits or invalid program ID | **Stop.** Terminal. Alert an operator; contact the CSM or support@leadspace.com. |
| 429 | Rate limit exceeded | Back off. The cap is 30 TPS per the tiered SLA — 1,800 records/minute. |
| 500-504 | Server error | Retriable with backoff. Log `tracking_id` from the error body. |

The error envelope is `{ error, tracking_id, request_timestamp, details }` —
not RFC 9457 problem+json. The body may be absent; on 401 the docs say to
ignore it entirely.

**There is no idempotency key.** Every retry re-consumes a credit. Retry only
on 429 and 5xx, with backoff and a bounded attempt count.
