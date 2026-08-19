---
name: Analyse an investor's portfolio and funding activity with Mattermark
description: Resolve an investor, walk their portfolio companies, and pull funding events — with MSFL for complex segmentation.
api: openapi/mattermark-rest-api-openapi.yml
operations: [search, get_investor, get_investor_portfolio, searchFunding, query_investors, quota]
generated: '2026-08-14'
method: generated
source: openapi/mattermark-rest-api-openapi.yml
---

# Analyse an investor's portfolio and funding activity with Mattermark

> **Availability check before you use this skill.** As of 2026-08-14
> `https://api.mattermark.com` does not complete a TLS handshake (SSL alert 112,
> unrecognized_name). Probe the base URL first and stop if it fails. The
> operations below are verified against Mattermark's own published Swagger 2.0
> definition, but the host is not currently callable.

## Authentication

`Authorization: Bearer <api key>` on every request. See
`authentication/mattermark-authentication.yml`.

## Steps

1. **Resolve the investor** — call `search` (`GET /search`) with `term` set to
   the firm name and `object_types=investor`. Take `object_id` from the row
   whose `object_type` is `investor`.

2. **Pull the investor record** — call `get_investor`
   (`GET /investors/{id}`). `InvestorItem` returns `name`, `mm_slug`, `type`,
   `location`, `portfolio_size`, `number_of_deals`, `three_year_funds_sold`,
   `est_most_recent_fund_date`, `most_recent_funding`, `portfolio_aggregates`,
   `deal_aggregates`, `top_industries` and `most_recent_filings`. It also
   returns `portfolio_path` and `funding_rounds_path` — relative paths you can
   follow rather than constructing URLs yourself.

3. **Walk the portfolio** — call `get_investor_portfolio`
   (`GET /investors/{id}/portfolio`). Page it with `page` / `per_page`; read
   `meta.total_pages` to know when to stop, and compare
   `meta.total_record_count` against the record's `portfolio_size` to detect a
   trial-access truncation.

4. **Pull funding events** — call `searchFunding` (`GET /fundings`). Each
   `FundingItem` carries `company_id`, `company_name`, `investors`,
   `investor_slugs`, `series`, `rounds_funding_date`, `amount`, `currency`,
   `news_url`, `industry` and geography. Join back to a company profile with
   `company_id` via `get_company`.

5. **Segment with MSFL when the flat parameters are not enough** — call
   `query_investors` (`POST /queries`) with a dataset and an MSFL filter. This
   endpoint is documented as **BETA**; treat its shape as unstable and do not
   build unattended pipelines on it. Creating a query returns a query id you
   then read back.

6. **Watch the budget** — call `quota` (`GET /ratelimit/usage`) before and after
   a portfolio walk. It is free and returns `quota_limit`, `quota_remaining`,
   `quota_reset` and `quota_period`. A large portfolio walk is the most
   quota-expensive flow in this API; a 500-company portfolio at `per_page=50` is
   ten calls before you fetch a single company profile.

## When to use GraphQL instead

The REST surface cannot answer some questions at all. Use
`https://eapi.mattermark.com/` (schema in `graphql/mattermark.graphql`) when you
need:

- **`fundingRoundSummaryQuery`** — filters REST does not expose:
  `amountRaisedMin`/`amountRaisedMax`, `cities`, `countries`,
  `fundingDateMin`/`fundingDateMax`, `investorIds`, `investorSlugs`, `series`,
  `states`. Capped at 50 rounds per call.
- **`organizationSummaryQuery(msfl:)`** — MSFL segmentation returning Relay
  connections.
- **`viewer`** — account identity plus quota; REST returns only the counters.

Note the id forms differ across surfaces: REST uses bare numeric ids, GraphQL
uses `"c#<id>"` for companies and `"i#<id>"` for investors. Strip or add the
prefix when crossing between them. Batch `organizations(ids:, domains:)` is
capped at 5 organizations per query.

## Conventions you must respect

- Page with `page` / `per_page` (default 10, documented max 50); read the `meta`
  envelope.
- Read `X-Quota-Limit` / `X-Quota-Remaining` / `X-Quota-Reset` on every
  response; back off to `X-Quota-Reset` on `429`. No `Retry-After` is sent.
- `403` means the key is wrong, expired or unentitled — not that the investor
  record is private.
- No idempotency key exists. `POST /queries` is not de-duplicated; guard
  retries yourself.
