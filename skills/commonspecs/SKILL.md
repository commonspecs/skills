---
name: commonspecs
description: >-
  Find the best thing to buy in any category — a product or a service — by its
  objective specs, with price offers and availability in your region. Use when
  the user asks what to buy, what something is actually made of or includes, how
  the options compare and which is better, or what it costs and where to buy it.
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

The skill needs one environment variable — your API token (shape `cs_live_…`, sent as
`Authorization: Bearer`) — read from the project root `.env` (or your shell):

```
COMMONSPECS_API_TOKEN=cs_live_…
```

If `COMMONSPECS_API_TOKEN` is unset, tell the user to set it and stop — do not call the API.

The token lives **only** in the environment: reference it as `$COMMONSPECS_API_TOKEN` and let the
shell expand it when calling the API — never read, echo, or otherwise pull the token value into the
model's context.

## The user's buying goals (every read carries them)

The pick — "which of these should I buy?" — is YOUR judgement: the facts the API returns, weighed
against what the user is optimising for. You don't fetch or store that separately. **Every read
that returns a product or offers carries a `context` block** with the user's standing goals,
resolved server-side:

```json
"context": {
  "user_goal": "Optimise for cost of use over the product's lifetime, not the lowest sticker price. Favour local sellers when quality is comparable.",
  "user_market": "PL",
  "contribution_mode": "automatic"
}
```

- **`user_goal`** — a ready-to-apply directive: rank and explain recommendations by it. It's prose
  meant to be applied, not a setting to decode.
- **`user_market`** — ISO 3166-1 alpha-2 country (or null) the user buys in; scope prices and
  availability to it.
- **`contribution_mode`** — `automatic` (submit extracted specs/offers silently) · `ask_user`
  (one-tap confirm first) · `never` (don't submit). Respect it before any `submit_contribution`.

The user sets these on the site (commonspecs.com/account); you only **read** them — never ask for
them, never set them. A single request may still override them in conversation ("ignore locality
this time", "price in Germany") without changing anything stored.

**The market is server-side — you don't pass it.** Reads are scoped to the user's saved market
(`user_market`) by the server: `search` returns products available where the user buys, `get_offers`
prices for that market, `lookup` prices its `top_offer` there. Pass `country_code` **only to
override** for a different market the request explicitly names ("price in Japan" → `JP`). Submit an
`offer` with the country the price was actually observed in, not necessarily the user's market (a
`DE` shop shipping to `PL` is a `PL`-destination offer).

## Tools

All calls send `Authorization: Bearer $COMMONSPECS_API_TOKEN`.

### lookup_product — fetch one product's specs

Resolve a product by **exactly one** of: `url`, `ean`, or `brand` + `model`.

```bash
curl -sS -X POST "https://api.commonspecs.com/v1/lookup" \
  -H "Authorization: Bearer $COMMONSPECS_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"brand":"Nudie","model":"Gritty Jackson"}'
```

`{"url":"https://…"}` or `{"ean":"7340028912345"}` are the other two forms. Optional:
`exclude_low_confidence: true` to drop the low-confidence bucket. `top_offer` is priced in the
user's saved market automatically; pass `country_code` only to override it for a different market.

If the user pastes a bare 8–14 digit number (EAN-8, UPC-A, EAN-13, or GTIN-14), treat it
as an `ean` and look it up that way before trying to parse it as a model.

Response `status`:
- `hit` — one product. Body: `product`, `quality_score` (0–100 or null), `top_offer`
  (the recency-best price for `country_code`, or null), `fields`, `low_confidence_fields`,
  `enrichment_opportunities` (see below).
- `candidates` — several variants matched (`candidates: [...]`); ask the user which, or look up by `url`/`ean`.
- `miss` — nothing found. Offer to contribute specs (`submit_contribution`).

When `top_offer` is **null** (no offer on record in the user's market), report the specs but say
that **availability in the user's country still needs checking** — don't assert it's unavailable.
If you then research a local price or availability, act on `contribution_mode`: `automatic` → submit
it with `submit_contribution`; `ask_user` → ask the user first; `never` → don't submit.

### search_products — find products, best-scored first

```bash
curl -sS -X POST "https://api.commonspecs.com/v1/search" \
  -H "Authorization: Bearer $COMMONSPECS_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"query":"raw denim","category":"jeans-denim"}'
```

Matches on brand name and model. **Results are scoped to the user's market** — only products
available where they buy (every locality setting; the locality choice steers which seller/origin you
prefer, not availability). Optional `category` (slug) filter, `limit` (≤20), and `country_code` to
override the market. Returns `results` ranked by
`quality_score` descending (best specs first; products with thin data sort last), each with the same
`quality_score` field as a lookup. Use this when the user asks "what should I buy in <category>"
rather than naming one product.

### compare_products — side-by-side on hard specs

Use when the user names **two or more** specific products to choose between.

```bash
curl -sS -X POST "https://api.commonspecs.com/v1/compare" \
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
curl -sS "https://api.commonspecs.com/v1/categories/jeans-denim/rankings?country=PL&limit=20" \
  -H "Authorization: Bearer $COMMONSPECS_API_TOKEN"
```

Returns `results` ranked by `quality_score` (each with a `rank`). `ranking_scope` is
`global` for now — country-localized rankings are deferred, so a `country_code` is echoed
back but the ranking is the global score order. To bring price into a category browse, read
offers per candidate with `get_offers` and weigh them yourself. Prefer this over many
individual lookups when the user is browsing a category.

### get_product — fetch by id

```bash
curl -sS "https://api.commonspecs.com/v1/products/$PRODUCT_ID" \
  -H "Authorization: Bearer $COMMONSPECS_API_TOKEN"
```

### get_quality_score — overall quality (0–100)

```bash
curl -sS "https://api.commonspecs.com/v1/products/$PRODUCT_ID/quality-score" \
  -H "Authorization: Bearer $COMMONSPECS_API_TOKEN"
```

Returns `total_score` (0–100, or `null` if the category isn't scored yet) and
`missing_fields` — the spec fields whose absence is holding the score back. A lower score
often means *thin data*, not a bad product; use `missing_fields` to decide what to
contribute. The score is a single number on purpose — the per-field weighting behind it is
not exposed; reason about trade-offs from the facts themselves.

### get_offers — prices for a product

```bash
curl -sS "https://api.commonspecs.com/v1/products/$PRODUCT_ID/offers" \
  -H "Authorization: Bearer $COMMONSPECS_API_TOKEN"
```

Returns `offers`: dated price observations, recency-sorted (best price today first) and
**never hidden by age** — a week-old price still shows, just lower in the order. Each offer
carries `merchant`, `merchant_country` (where the **shop** is based), `country_code` (where the
offer **ships to**), `channel` (`online`/`in_store`), `price`, `currency`, `shipping_cost`,
`landed_price` (price + shipping), `availability_status`, and `observed_at`. `merchant_country` ≠
`country_code` — a `DE` shop shipping to `PL` is `merchant_country: "DE"`, `country_code: "PL"`.
Use `merchant_country` to honour the user's locality goal: `local_only` already returns only domestic
shops; for `local_bonus`, prefer offers whose `merchant_country` matches the user's market (it also
decides which tax regime applies). Many prices across shops/countries are all true at once — they are
observations, not a single truth, and never disputed against each other. Scoped to the user's saved
market by default; pass `?country_code=` only to override it for a different market.

Price is deliberately **not** in `quality_score`: the score measures the thing as a thing.
Value-per-money is yours to compute — weigh `landed_price` against the spec quality and the
user's `user_goal` from the response `context`.

### submit_contribution — add specs and/or a price you have verified

One submission carries a `fields` array **and/or** an `offer` (a price observation grabbed
from the same page you read the specs off — the cheapest moment to capture it). Identify the
product by exactly one of `url` / `ean` / `brand`+`model` (brand+model creates the product if
new). Each field should carry the `source_url` and a verbatim `snippet` you read the value
from — that evidence is what earns confidence. `source` is `web` (default, a web page) or
`label` (a physical label — see below).

```bash
curl -sS -X POST "https://api.commonspecs.com/v1/contributions" \
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
curl -sS -X POST "https://api.commonspecs.com/v1/contributions" \
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
fabricate calls to it — say the capability isn't live. (Country-localized rankings are also
deferred — see `get_rankings`.)
