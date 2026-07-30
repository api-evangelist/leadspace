---
name: Refresh buyer intent for accounts with Leadspace
description: Score a list of companies for surging intent topics and intent-model levels, and interpret the scores before routing accounts to sales.
api: openapi/leadspace-intent-openapi.yml
operations:
  - scoreCompanyIntent
  - getIntentResults
generated: '2026-07-19'
method: generated
source: https://support.leadspace.com/hc/en-us/articles/360012583400-Leadspace-Intent-Only-API
---

# Refresh buyer intent for accounts

Use this when you only need intent — a weekly or monthly refresh of intent
values on accounts you have already enriched. It is cheaper and narrower than
a full enrichment call, which also returns `company.intent_modeling_v4`.

## 1. Authenticate

`Authorization: Bearer <API_KEY>`. Same credentials as enrichment.

## 2. Submit companies

Call `scoreCompanyIntent` — `POST /v1/intent/score/`. Note the **trailing
slash**; it is part of the documented path.

`company.name` is required. Send `company.website` too — intent is resolved
against a domain, and the response tells you which one it used via
`data.domain_used`.

```json
{
  "data": [
    {
      "company": {
        "name": "",
        "website": "",
        "address": { "country": "", "state": "", "city": "" }
      },
      "external_id": "sfdc-account-id"
    }
  ],
  "external_bulk_id": "intent-week-29"
}
```

Use `external_id` to carry your CRM account id through; it is echoed back.

## 3. Collect

A 202 returns `{ "id": "/v1/intent/results/<jobId>" }`. Persist it, then call
`getIntentResults` — `GET /v1/intent/results/{jobId}`.

## 4. Interpret the scores before acting

The response `data` carries:

- **`domain_intent`** — the account's top surging intent topics. When there is
  nothing surging, this reads `"No Surging Topics"`. Treat that string as an
  empty result, not as a topic.
- **`domain_used`** — the domain intent was resolved against. If this is not
  the domain you expected, the match is wrong and the score is about a
  different company.
- **`intent_modeling_scores.project_name`** — your configured intent modeling
  project.
- **`intent_modeling_scores.project_score`** — by default: 0-1 No Intent,
  1-75 Low, 75-95 Medium, 95-100 High. **These bands are configurable per
  project**, so read your project's definition rather than hard-coding the
  defaults.
- **`intent_modeling_scores.surging_topics`** — the drivers behind the score.
  Route these to sales alongside the score; a High with no legible drivers is
  not actionable.

From full enrichment, `company.intent_modeling_v4` adds `source` (LS Intent or
Bombora), `cadence` (always weekly), `new_high_intent` (whether there are new
drivers this week), `industries`, `top_metros`, and `domain_origins`.

## 5. Cadence and freshness

Intent models update **weekly**. Polling more often than that spends credits
for values that have not moved. Schedule refreshes on a weekly cycle and use
`new_high_intent` to decide which accounts are worth alerting on.

## 6. Failures

`400` insufficient input or bad JSON; `401` bad or expired credentials;
`427` insufficient credits or invalid program ID (terminal — contact the CSM);
`5XX` retriable with backoff, log `tracking_id`. Record-level failures inside
a successful job use code 1100 (Internal Error, retriable) or 1200 (Invalid
Input, not retriable as submitted).
