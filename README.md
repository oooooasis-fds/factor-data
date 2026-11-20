# Factor Data — ooooOasis

Point-in-time factor data for North American equities, served as a REST API and as an MCP server. The universe covers 583 tickers (US and Canadian listings); 180 factors are registered, 83 of them with computed history — 724,421 factor values as of the latest snapshot, plus SEC company facts (2.66M rows), an SEC filings index, earnings calendars and option-chain snapshots. Every factor value is point-in-time: it carries both the date it describes and the moment it became available (`available_at`), so backtests never see information from the future.

Product page: https://www.oooooasis.com/fct · Pricing: https://api.oooooasis.com/pricing

## Get an API key

Keys are per service — a Factor Data key (prefix `ffds_live_`) works only on this service.

1. Register at https://api.oooooasis.com/register — choose the Factor Data service and enter your email.
2. Click the verification link we email you (valid 24 hours). Your key is shown **once** on that page — store it immediately.
3. Lost your key? Re-issue it at https://api.oooooasis.com/reissue — after email verification the old key is revoked and a new one is shown once.

The free tier is live: **100 calls/day**, no card required. Free keys expire 90 days after issue (register again to renew). Paid tiers are **not on sale yet** — see "Limits & errors" for exactly what a free key can call, and join the waitlist by emailing fct-data@oooooasis.com.

Send your key on every call in the `Authorization` header:

```
Authorization: Bearer <your key>
```

Keys in URLs (`?api_key=...`) are rejected with a 400 error.

## REST API

Base URL: `https://api.oooooasis.com/fct` — read-only, GET only (other methods answer 405).

Data responses share one shape: `{"query": ..., "rows": N, "values": [...]}`. Factor rows include `date`, `available_at`, and a read-time quality `state` (`fresh` / `stale`) evaluated at `state_as_of`. Missing data is missing — nothing is zero-filled or backfilled.

| Method | Path | Access | Key parameters | Returns |
|---|---|---|---|---|
| GET | `/health` | Public | — | Service status (no key, not metered) |
| GET | `/meta` | Public | — | Table row counts, latest data dates, universe coverage (no key, not metered) |
| GET | `/asof` | **Free** | `ticker`*, `factor`*, `as_of`*, `date` | Point-in-time value(s) of one factor for one ticker: rows with `available_at ≤ as_of`; add `date` for a single date's then-visible value |
| GET | `/history` | Paid | `ticker`*, `factor`*, `start`, `end`, `as_of` | Time series for one ticker × factor |
| GET | `/cross_section` | Paid | `date`*, `factor`, `as_of` | One factor (or all factors) across all tickers on one date |
| GET | `/registry` | Paid | `factor` | Factor registry metadata (definition, status, source); omit `factor` for the whole registry (≤1,000 entries per call) |
| GET | `/px` | Paid | `symbol`*, `start`*, `end`*, `as_of` | Price history rows for one symbol |
| GET | `/bars` | Paid | `symbol`*, `start`*, `end`*, `as_of` | Daily bars for one symbol |
| GET | `/calendar` | Paid | `ticker`*, `as_of`*, `date` | Earnings / ex-dividend calendar for one ticker, as known at `as_of` |
| GET | `/calendar/crosscheck` | Paid | `ticker`*, `as_of`* | Earnings-calendar cross-check rows for one ticker |
| GET | `/regime` | Paid | `date` or `start`+`end`, `as_of` | Market-regime series |
| GET | `/markets` | Paid | `date` or `start`+`end`, `as_of` | Seven-markets index series |
| GET | `/universe` | Paid | `date` or `ticker`, `as_of` | Universe membership rows |
| GET | `/chain` | Paid | `symbol`*, `date`*, `expiry`, `right` (`call`\|`put`), `as_of` | Option-chain snapshot rows |
| GET | `/iv` | Paid | `ticker`*, `start`*, `end`*, `as_of` | Implied-volatility history for one ticker |
| GET | `/iv/term` | Paid | `ticker`*, `date`*, `expiry`, `as_of` | IV term-structure rows |
| GET | `/iv/census` | Paid | `ticker`*, `as_of` | IV census rows for one ticker |
| GET | `/iv/overlap` | Paid | `date`*, `as_of` | IV overlap-check rows for one date |
| GET | `/sec/facts` | Paid | `ticker`*, `concept`, `start`, `end`, `fy`, `fp`, `as_of` | SEC company-facts (XBRL) values; `fy`/`fp` optional, default latest filed |
| GET | `/sec/filings` | Paid | `ticker` or `accession`, `form`, `start`, `end`, `as_of` | SEC filings index rows |

\* = required. **Free** = callable on the free tier (scope below). **Paid** = requires a paid tier; paid tiers are not on sale yet, so free keys receive `403 tier_not_entitled` on these today.

### Free-tier scope

A free key can call `/asof` only (plus the public `/health` and `/meta`), within this scope:

- Tickers: the S&P 500 Top 100 by market cap (monthly list).
- Factors: the 37 recognized free-tier factors.
- One ticker × one factor per call.
- Known exception: 252-day IV Rank (`iv_rank`) is **not available yet** — the IV sequence started 2026-11-04 and needs 252 trading days (earliest ~2027-11). Requests for it answer `403 factor_not_available` with that reason. The realized-volatility percentile (`rv_pctile_6mo`) is a different measure and is never substituted for it.

### Examples

Service metadata — public, no key:

```bash
curl "https://api.oooooasis.com/fct/meta"
```

Current factor value — the free-tier call:

```bash
curl -H "Authorization: Bearer $KEY" \
  "https://api.oooooasis.com/fct/asof?ticker=AAPL&factor=rsi14&as_of=2026-11-20"
```

Factor history (paid tiers, not yet on sale):

```bash
curl -H "Authorization: Bearer $KEY" \
  "https://api.oooooasis.com/fct/history?ticker=AAPL&factor=rsi14&start=2026-10-01&end=2026-11-19"
```

## MCP server

URL: `https://mcp.oooooasis.com/fct` — Model Context Protocol (protocol version 2024-11-05), Streamable HTTP (`POST /mcp`) with an SSE transport (`GET /sse`, `POST /mcp/messages`) also available. Every request needs your Factor Data key in the `Authorization: Bearer` header; the key selects this service. The same quota, tier scope and coverage metering as REST apply to tool calls.

Generic client configuration:

```json
{
  "mcpServers": {
    "factor-data": {
      "url": "https://mcp.oooooasis.com/fct",
      "headers": { "Authorization": "Bearer <your factor key>" }
    }
  }
}
```

`tools/list` only shows the tools your key can currently call. 14 tools are registered; while paid tiers are not on sale, only the two marked **Available now** are listed and callable — the rest appear when paid access opens.

| Tool | Available | Parameters | Purpose |
|---|---|---|---|
| `factor_asof` | **Available now** (free) | `ticker`*, `factor`*, `as_of`* | Current value of one factor for one ticker (free-tier scope: Top-100 ticker × one of the 37 recognized factors) |
| `factor_meta` | **Available now** (not metered) | — | Service metadata: snapshot time, row counts |
| `factor_history` | Paid | `ticker`, `factor`, `from`, `to` | Factor history for one ticker × factor |
| `factor_cross_section` | Paid | `factor`, `as_of`, `limit` | Cross-section of one factor on one date |
| `factor_registry` | Paid | `factor`, `limit` | Factor registry lookup |
| `factor_px` | Paid | `ticker`, `from`, `to` | Market prices for one ticker |
| `factor_bars` | Paid | `ticker`, `from`, `to` | Market bars for one ticker |
| `factor_calendar` | Paid | `from`, `to` | Trading calendar |
| `factor_regime` | Paid | `ticker`, `from`, `to` | Market regime for one ticker |
| `factor_markets` | Paid | — | Market list |
| `factor_universe` | Paid | `as_of`, `limit` | Universe list |
| `factor_chain` | Paid | `ticker` | Option-chain lookup for one ticker |
| `factor_iv` | Paid | `ticker` | Implied volatility for one ticker |
| `factor_sec` | Paid | `ticker`, `form`, `from`, `to` | SEC filings for one ticker |

## Limits & errors

- **Daily quota (free):** 100 calls/day across the entitled endpoints. Responses carry `X-RateLimit-Limit`, `X-RateLimit-Remaining` and `X-RateLimit-Reset` (epoch seconds; quota resets at midnight US/Eastern).
- **Coverage metering:** distinct (ticker × factor) pairs you touch are counted over a rolling 30-day window; the free-tier cap is 300 pairs. At 80% you get an `X-Coverage-Warning: 80%` header; over the cap, calls answer `429 coverage_exceeded` (pairs roll out of the window daily — there is no fixed reset day).
- **Response caps:** `/history` and `/cross_section` return at most 10,000 rows per call; `/chain` 10,000 rows; `/sec/facts` and `/sec/filings` 1,000 rows; `/px`, `/bars`, `/regime`, `/markets` date ranges are limited to 366 days. Over a cap the API answers 400 — it never silently truncates.
- **Errors** are always JSON: `{"error": {"code": ..., "message": ...}}`. Codes you may see:
  - `400 key_in_url_rejected` — key sent in the URL; use the `Authorization` header.
  - `400 upstream_error` — bad or missing parameters, or an over-cap request (the message says which).
  - `401 unauthorized` — missing, invalid, revoked or expired key.
  - `403 tier_not_entitled` — your tier may not call that endpoint group.
  - `403 factor_not_available` — the factor is known-unavailable (e.g. 252-day IV Rank, above).
  - `404 not_found` — unknown path.
  - `429 quota_exceeded` — daily quota used up; the body carries `limit`, `used` and `resets_at`, and the response carries a `Retry-After` header.
  - `429 coverage_exceeded` — rolling 30-day coverage cap reached; the body carries `limit` and `used_30d`.

## Data notes

- **Point-in-time by construction.** Factor values are stored as revisions with `available_at` timestamps; as-of queries return the highest revision that was available at the requested moment, and every response is reverse-checked so no row with `available_at > as_of` can leak through. Restatements appear as new revisions — history is never silently rewritten.
- **Freshness.** The factor history is updated on trading days; `/fct/meta` always reports the current latest date per table (factor history latest: 2026-11-19 at the time of writing). Price history (`px_history_daily`) can lag the factor panel by a few days (latest: 2026-11-16 at the time of writing) — check `/meta` for live values rather than assuming.
- **Coverage.** 583 tickers have factor history (83 factors); the registry lists 180 factors, including registered-but-not-yet-computed ones — a registry entry is a definition, not a promise of data. Free-tier scope (Top 100 × 37 factors) is narrower than the full panel by design.

## Links & contact

- Product page: https://www.oooooasis.com/fct
- Pricing & paid-tier waitlist: https://api.oooooasis.com/pricing
- Sister service — TSX Data (Canadian market data): https://github.com/ooooOasis-fds/tsx-data · https://www.oooooasis.com/tsx
- Contact: fct-data@oooooasis.com
