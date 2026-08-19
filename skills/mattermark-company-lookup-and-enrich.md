---
name: Look up a company and enrich it from Mattermark
description: Resolve a company by name or domain through Mattermark's autocomplete search, then pull its full profile, news and key contacts.
api: openapi/mattermark-rest-api-openapi.yml
operations: [search, get_company, get_companies, get_company_stories, get_company_personnel]
generated: '2026-08-14'
method: generated
source: openapi/mattermark-rest-api-openapi.yml
---

# Look up a company and enrich it from Mattermark

> **Availability check before you use this skill.** As of 2026-08-14
> `https://api.mattermark.com` does not complete a TLS handshake — it returns
> SSL alert 112 (unrecognized_name) because the hostname resolves to
> Mattermark's marketing site IP, which serves no certificate for it. The
> specification these steps are grounded in is real and first-party, but the
> host is not currently callable. Probe the base URL once before running any
> flow and stop if the handshake fails.

## Authentication

Send the account API key as a Bearer token:

```
Authorization: Bearer <api key>
```

The API also accepts the key as a `key` query-string parameter, which every
example in the vendor docs uses. Prefer the header — a key in a query string is
logged by proxies and browser history. Keys come from the Mattermark account's
API settings; there is no self-serve signup path that resolves today, so
provisioning goes through sales@mattermark.com.

## Steps

1. **Resolve the company** — call `search` (`GET /search`) with `term` set to the
   company name or domain, and `object_types=company` to exclude investors. This
   operation is built for autocomplete and is fast, but it *does* consume quota.
   Take `object_id` from the result — that is the join key for every later call.
   Note the response is a bare JSON array, not a paged envelope.

2. **Pull the profile** — call `get_company` (`GET /companies/{id}`) with the
   `object_id`. The `CompanyItem` representation carries 45 fields including
   `mattermark_score`, `stage`, `total_funding`, `last_funding_amount`,
   `last_funding_date`, staged headcount (`employees`, `employees_month_ago`,
   `employees_6_months_ago`), `website_uniques`, `mobile_downloads`, and
   location.

3. **Add news** — call `get_company_stories`
   (`GET /companies/{id}/stories`) for recent articles. Each `CompanyStory` has
   `title`, `url`, `published_at` and `source_title`.

4. **Add people** — call `get_company_personnel`
   (`GET /companies/{id}/people`) for key contacts. `PeopleItem` returns only
   `name`, `title` and `path` — **no email**. If you need an email address, that
   lives on the GraphQL surface as `contactDetails(name:, orgId:)`, whose schema
   description marks it deprecated. Do not expect it from REST.

5. **Or skip search entirely** — if you already hold a domain, call
   `get_companies` (`GET /companies`) with `domain=<domain>`. It also accepts
   `company_name`, `business_models`, `industries`, `keywords`, `stage` and
   `location` for segmentation.

## Conventions you must respect

- **Paging.** List operations return a `meta` envelope with
  `total_record_count`, `total_pages`, `current_page` and `per_page`. Drive it
  with `page` (default 1) and `per_page` (default 10, documented max 50).
- **Trial accounts cannot page at all** and see at most 50 results. If
  `total_record_count` exceeds what you receive and `page=2` returns nothing,
  the account is on trial access, not full API access.
- **Quota, not rate.** Every response carries `X-Quota-Limit`,
  `X-Quota-Remaining` and `X-Quota-Reset`. Read them on every call. Check
  standing for free with `quota` (`GET /ratelimit/usage`) — it does not consume
  quota. There is **no** `Retry-After` header and **no** `RateLimit-*` family.
- **Back off on 429** until the `X-Quota-Reset` Unix timestamp. Do not retry
  immediately.
- **403, not 401.** A bad, expired or unentitled key returns `403`, with no
  `WWW-Authenticate` challenge. Do not treat 403 as a permissions problem with
  the record — check the key first.
- **No idempotency contract.** These are all GETs and safe to retry, but nothing
  in this API de-duplicates writes.
- **No error body.** Failures are bare status codes; there is no
  `application/problem+json` envelope and no error code to branch on.
