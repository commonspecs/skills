---
name: commonspecs
description: >-
  Look up engineering-grade product specifications with per-field confidence
  (e.g. raw denim weight & weave, shoe construction, country of origin) plus
  dated price offers, and contribute specs and prices you have verified. Use
  when the user asks what a product is actually made of, how two products
  compare on hard specs, what it costs / where to buy it, or to record specs or
  a price read off a page or physical label.
license: MIT
metadata:
  homepage: https://commonspecs.com
  docs: https://commonspecs.com/docs/
---

# commonspecs

commonspecs is an API-first database of **engineering-grade product specs** with a
**confidence score per field**. It does not guess: every value is backed by evidence and
carries a calibrated confidence. Your job is to fetch those facts and present them
honestly — including how trustworthy each one is.

## Setup

Configuration comes from environment variables (or `~/.commonspecs/config.json`):

| Variable | Required | Default | Meaning |
| --- | --- | --- | --- |
| `COMMONSPECS_API_TOKEN` | yes | — | API token, shape `cs_live_…`. Sent as `Authorization: Bearer`. |
| `COMMONSPECS_API_URL` | no | `https://api.commonspecs.com` | API base URL. |
| `COMMONSPECS_DEFAULT_COUNTRY` | no | — | ISO 3166-1 alpha-2 (e.g. `PL`) for country-scoped reads. |

If `COMMONSPECS_API_TOKEN` is unset, tell the user to set it and stop — do not call the API.

## First run — capture the user's buying goals

The pick ("which of these should I buy?") is YOUR judgement, made from the facts the API returns
weighed against what the user actually cares about. Capture that once, up front — the API does not.

Before your first `lookup`/`search`/`rankings` call, read `~/.commonspecs/config.json` and check
for `quality_strategy` and `locality_strategy`. If either is missing, ask the user (one short
message, not a form), then persist their answers to `~/.commonspecs/config.json`:

- **quality_strategy** — how to trade quality against price:
  - `quality_first` — maximise quality; price is secondary.
  - `cost_over_time` — prefer the best cost of use over time, not the lowest sticker price.
- **locality_strategy** — whether to favour local sellers:
  - `local_bonus` — favour local products/shops, but never over clearly better quality.
  - `global_best` — no locality bonus; the best offer worldwide wins.
  - `local_only` — restrict to locally available products/shops, even at the cost of choice.

While you are at it, if `default_country` is unset, ask for it (ISO 3166-1 alpha-2, e.g. `PL`) —
it scopes prices and availability. Optionally capture `contribution_mode`
(`automatic` default | `confirm` | `disabled`): whether you may submit specs and offers you
extract back to the shared base silently, with a one-tap confirm, or not at all.

Persist the answers, for example:

```json
{ "api_token": "cs_live_…", "default_country": "PL",
  "quality_strategy": "cost_over_time", "locality_strategy": "local_bonus",
  "contribution_mode": "automatic" }
```

On later runs, read these as the DEFAULT context for ranking and for explaining recommendations;
a single request may override them ("ignore locality this time"). This works even when the base is
empty: the user can hand you candidates (specs + price + shop) and you rank them against these
goals directly, then contribute the durable facts back (see `submit_contribution`).

**Choosing `country_code` per call.** The `PL` in the examples below is illustrative — never
hardcode it. Resolve the market for each read in this order: (1) the market the user's request is
about ("price in Germany" → `DE`); (2) otherwise `default_country` / `COMMONSPECS_DEFAULT_COUNTRY`.
If neither is known, ask once, or omit `country_code` — but note the cost of omitting: `lookup`
returns `top_offer: null` and `get_offers` returns offers across all markets mixed together.
Submit an `offer` with the country the price was actually observed in, which is not necessarily the
user's market (a `DE` shop shipping to `PL` is a `PL`-destination offer).

## Tools

All calls send `Authorization: Bearer $COMMONSPECS_API_TOKEN`. Examples assume
`API="$COMMONSPECS_API_URL"` (fall back to the default above).

### lookup_product — fetch one product's specs

Resolve a product by **exactly one** of: `url`, `ean`, or `brand` + `model`.

```bash
curl -sS -X POST "$API/v1/lookup" \
  -H "Authorization: Bearer $COMMONSPECS_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"brand":"Nudie","model":"Gritty Jackson"}'
```

`{"url":"https://…"}` or `{"ean":"7340028912345"}` are the other two forms. Optional:
`exclude_low_confidence: true` to drop the low-confidence bucket, and `country_code`
(ISO 3166-1 alpha-2) to get a `top_offer` for that market in the response.

If the user pastes a bare 8–14 digit number (EAN-8, UPC-A, EAN-13, or GTIN-14), treat it
as an `ean` and look it up that way before trying to parse it as a model.

Response `status`:
- `hit` — one product. Body: `product`, `quality_score` (0–100 or null), `top_offer`
  (the recency-best price for `country_code`, or null), `fields`, `low_confidence_fields`,
  `enrichment_opportunities` (see below).
- `candidates` — several variants matched (`candidates: [...]`); ask the user which, or look up by `url`/`ean`.
- `miss` — nothing found. Offer to contribute specs (`submit_contribution`).

### search_products — find products, best-scored first

```bash
curl -sS -X POST "$API/v1/search" \
  -H "Authorization: Bearer $COMMONSPECS_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"query":"raw denim","category":"jeans-denim"}'
```

Matches on brand name and model. Optional `category` (slug) filter and `limit` (≤20).
Returns `results` ranked by `quality_score` descending (best specs first; products with
thin data sort last), each with the same `quality_score` field as a lookup. Use this when
the user asks "what should I buy in <category>" rather than naming one product.

### compare_products — side-by-side on hard specs

Use when the user names **two or more** specific products to choose between.

```bash
curl -sS -X POST "$API/v1/compare" \
  -H "Authorization: Bearer $COMMONSPECS_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"product_ids":["<id1>","<id2>"]}'
```

Up to 5 ids. Returns `products` (each with `quality_score`), a `comparison` matrix
(`[{field, cells: {<product_id>: {value, confidence} | null}}]`), and `overall_winner`
(the product_id with the highest `total_score`). Present the facts side by side, name the
`overall_winner`, then explain the trade-off yourself from the facts + scores — the API
deliberately does not return per-dimension winners or a rationale; the value-vs-cost and
fitness-for-purpose judgement is yours to make from the data.

### get_rankings — top products in a category

Use when the user asks "what are the best X" without naming a product.

```bash
curl -sS "$API/v1/categories/jeans-denim/rankings?country=PL&limit=20" \
  -H "Authorization: Bearer $COMMONSPECS_API_TOKEN"
```

Returns `results` ranked by `quality_score` (each with a `rank`). `ranking_scope` is
`global` for now — country-localized rankings are deferred, so a `country_code` is echoed
back but the ranking is the global score order. To bring price into a category browse, read
offers per candidate with `get_offers` and weigh them yourself. Prefer this over many
individual lookups when the user is browsing a category.

### get_product — fetch by id

```bash
curl -sS "$API/v1/products/$PRODUCT_ID" \
  -H "Authorization: Bearer $COMMONSPECS_API_TOKEN"
```

### get_quality_score — overall quality (0–100)

```bash
curl -sS "$API/v1/products/$PRODUCT_ID/quality-score" \
  -H "Authorization: Bearer $COMMONSPECS_API_TOKEN"
```

Returns `total_score` (0–100, or `null` if the category isn't scored yet) and
`missing_fields` — the spec fields whose absence is holding the score back. A lower score
often means *thin data*, not a bad product; use `missing_fields` to decide what to
contribute. The score is a single number on purpose — the per-field weighting behind it is
not exposed; reason about trade-offs from the facts themselves.

### get_offers — prices for a product

```bash
curl -sS "$API/v1/products/$PRODUCT_ID/offers?country_code=PL" \
  -H "Authorization: Bearer $COMMONSPECS_API_TOKEN"
```

Returns `offers`: dated price observations, recency-sorted (best price today first) and
**never hidden by age** — a week-old price still shows, just lower in the order. Each offer
carries `merchant`, `country_code`, `channel` (`online`/`in_store`), `price`, `currency`,
`shipping_cost`, `landed_price` (price + shipping), `availability_status`, and `observed_at`.
Many prices across shops/countries are all true at once — they are observations, not a single
truth, and never disputed against each other. Optional `country_code` restricts to one market.

Price is deliberately **not** in `quality_score`: the score measures the thing as a thing.
Value-per-money is yours to compute — weigh `landed_price` against the spec quality and the
user's `quality_strategy`/`locality_strategy`.

### submit_contribution — add specs and/or a price you have verified

One submission carries a `fields` array **and/or** an `offer` (a price observation grabbed
from the same page you read the specs off — the cheapest moment to capture it). Identify the
product by exactly one of `url` / `ean` / `brand`+`model` (brand+model creates the product if
new). Each field should carry the `source_url` and a verbatim `snippet` you read the value
from — that evidence is what earns confidence. `source` is `web` (default, a web page) or
`label` (a physical label — see below).

```bash
curl -sS -X POST "$API/v1/contributions" \
  -H "Authorization: Bearer $COMMONSPECS_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "brand": "Nudie Jeans", "model": "Gritty Jackson", "source": "web",
    "fields": [
      {"field_name": "fabric_weight", "value": "13.5 oz",
       "source_url": "https://www.nudiejeans.com/product/gritty-jackson",
       "snippet": "13.5 oz organic dry denim"}
    ]
  }'
```

An `offer` is a dated observation: `store` (a shop domain or URL — the store's identity),
`country` (ISO 3166-1 alpha-2), `price`, `currency` (ISO 4217), `availability`
(`in_stock` / `low_stock` / `out_of_stock` / `preorder` / `discontinued`), and optionally
`channel` (`online` default / `in_store`), `shipping_cost`, `observed_at` (ISO 8601; defaults
to now), `source_url`. Send it alongside `fields` from the same fetch, or on its own:

```bash
curl -sS -X POST "$API/v1/contributions" \
  -H "Authorization: Bearer $COMMONSPECS_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "brand": "Nudie Jeans", "model": "Gritty Jackson",
    "offer": {
      "store": "nudiejeans.com", "country": "PL", "price": 690.00, "currency": "PLN",
      "availability": "in_stock", "shipping_cost": 0, "source_url": "https://www.nudiejeans.com/product/gritty-jackson"
    }
  }'
```

Physical world: if the user shows a photo of a label/product, **read the values and the
EAN yourself and send only the extracted values** — the photo never leaves the user's
machine. Look the product up by `ean` first, then contribute the missing `fields` with
`"source": "label"`. A shelf price is just an `offer` with `"channel": "in_store"`.

## Reading a response

`quality_score` (0–100, or null) is the product's overall spec quality; pair it with
`missing_fields` from `get_quality_score` when the user asks "is this good?". Each field in
`fields` / `low_confidence_fields` has `value`, `confidence` (0–1), `disputed`,
`needs_corroboration`, and (when disputed) `alternate_claims`.

- **`fields[]`** — confidence ≥ 0.4. Trustworthy enough to state. Still surface the
  number; "13.5 oz (confidence 0.88)" beats a bare "13.5 oz".
- **`low_confidence_fields[]`** — confidence 0.1–0.4. A *lead, not a fact*. Present only
  with an explicit hedge ("one source suggests…"). Omit if the user wants only solid data.
- **`needs_corroboration: true`** — only one independent source so far. Say so.
- **`disputed: true`** — sources disagree; show the winner **and** `alternate_claims`.
  A disagreement is itself a buying signal, not noise to hide.
- **`enrichment_opportunities`** — fields that would benefit from another source. If the
  user is engaged, offer to fetch one and `submit_contribution`.

Never invent a value that is not in the response, and never drop the confidence when you
report a fact.

## Errors

- `401` invalid/missing token → tell the user to check `COMMONSPECS_API_TOKEN`; don't retry.
- `403` forbidden → token lacks access; don't retry.
- `429` rate limited → back off and retry with exponential delay (e.g. 1s, 2s, 4s, max 3).
- `5x` / network → retry once or twice with backoff, then report the failure plainly.

## Not yet available

`flag_stale` (marking a price or fact out of date) is not exposed by the API yet. Don't
fabricate calls to it — say the capability isn't live. Country-localized rankings are also
deferred; rankings stay global (read `get_offers` per candidate to factor in price).
