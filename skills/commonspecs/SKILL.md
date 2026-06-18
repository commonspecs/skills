---
name: commonspecs
description: >-
  Look up engineering-grade product specifications with per-field confidence
  (e.g. raw denim weight & weave, shoe construction, country of origin), and
  contribute specs you have verified. Use when the user asks what a product is
  actually made of, how two products compare on hard specs, or to record specs
  read off a page or physical label.
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
`exclude_low_confidence: true` to drop the low-confidence bucket.

If the user pastes a bare 8–14 digit number (EAN-8, UPC-A, EAN-13, or GTIN-14), treat it
as an `ean` and look it up that way before trying to parse it as a model.

Response `status`:
- `hit` — one product. Body: `product`, `quality_score` (0–100 or null), `fields`,
  `low_confidence_fields`, `enrichment_opportunities` (see below).
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
`global` for now — country-localized rankings need pricing/availability that isn't live,
so a `country_code` is echoed back but the ranking is the global score order. Prefer this
over many individual lookups when the user is browsing a category.

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

### submit_contribution — add specs you have verified

One submission carries a `fields` array. Identify the product by exactly one of
`url` / `ean` / `brand`+`model` (brand+model creates the product if new). Each field
should carry the `source_url` and a verbatim `snippet` you read the value from — that
evidence is what earns confidence. `source` is `web` (default, a web page) or `label` (a
physical label — see below).

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

Physical world: if the user shows a photo of a label/product, **read the values and the
EAN yourself and send only the extracted values** — the photo never leaves the user's
machine. Look the product up by `ean` first, then contribute the missing `fields` with
`"source": "label"`.

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

`offers`/pricing and `flag_stale` are not exposed by the API yet. Don't fabricate calls to
them — say the capability isn't live.
