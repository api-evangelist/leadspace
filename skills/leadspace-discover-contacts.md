---
name: Discover net-new contacts inside target accounts with Leadspace
description: Expand a list of target accounts into ranked, persona-matched contacts with the required contact information, then collect the results.
api: openapi/leadspace-discovery-openapi.yml
operations:
  - discoverCompanyContacts
  - getDiscoveryResults
  - discoveryCompleteCallback
generated: '2026-07-19'
method: generated
source: https://support.leadspace.com/hc/en-us/articles/360011926619-Leadspace-Discovery-API
---

# Discover contacts inside target accounts

Enrichment resolves records you already have. Discovery (expansion) finds
people you do not have, inside accounts you care about. Use it to build
outbound lists and to fill named accounts against a persona.

## 1. Authenticate

`Authorization: Bearer <API_KEY>`, same credentials as the enrichment API.

## 2. Build the request

Call `discoverCompanyContacts` — `POST /v2/expansion/company`. Up to **500
accounts** per bulk request.

For each account, **either `account.companyName` or `account.companyWebsite`
is mandatory**. Supply the website where you have it — domain resolution beats
name resolution.

```json
{
  "data": [
    {
      "account": { "companyName": "", "companyWebsite": "" },
      "contact": {
        "persona": ["<up to 5 personas>"],
        "contactCountries": ["United States"]
      },
      "requiredResultsFields": {
        "verifiedEmail": true,
        "directPhone": false,
        "companyPhone": false
      },
      "limits": { "resultPerAccount": 10 },
      "external_id": "account-1"
    }
  ],
  "external_bulk_id": "q3-outbound",
  "callbackUrl": "https://your-host/leadspace/discovery-complete"
}
```

Three levers decide result quality:

- **`contact.persona`** — up to 5 personas, used to *rank* results, not to
  hard-filter them. Read the returned scores rather than assuming every
  contact matches.
- **`requiredResultsFields`** — this *does* hard-filter. Setting
  `verifiedEmail: true` guarantees deliverable addresses but shrinks yield.
  Turn it on for email outbound; leave it off when you want maximum coverage.
- **`limits.resultPerAccount`** — 1 to 100, defaults to 10. Credits are
  consumed per returned lead, so this is your primary cost control.

For `location.country`, omit the property entirely to allow all locations;
send a blank value to get only people with no country; send comma-delimited
values (e.g. `"United States","United Kingdom"`) to restrict.

## 3. Collect the results

A 202 returns `{ "id": "/v1/expansion/results/<bulkId>" }`. Persist the id.

Supply `callbackUrl` to be told when it finishes, or poll
`getDiscoveryResults` — `GET /v1/expansion/results/{bulkId}`:

- `200` — results ready
- `204` — still processing, keep polling with backoff
- `404` — results not found
- `401` — credentials wrong or expired

Reconcile each entry back to your account list with `external_id`.

## 4. Failure classes specific to discovery

| Status | Meaning | Do |
|---|---|---|
| 400 | Basic input validation or JSON failed | Confirm every account has a name or website. |
| 401 | Wrong or expired credentials | Refresh once, then stop. |
| 403 | Program ID is invalid | Terminal. Escalate to Leadspace. |
| 427 | Insufficient credits | Terminal. Do not retry — contact the CSM. |
| 5XX | Server error | Retriable with backoff; log `tracking_id`. |

## 5. Use the results responsibly

Discovery returns personal contact data for people who have not interacted
with you. Handle it under your DPA with Leadspace and your own outbound
compliance rules (CCPA and US state privacy laws, GDPR where applicable).
Leadspace provides opt-out mechanisms and states it does not process special
categories of data; honouring suppression is the customer's obligation.
