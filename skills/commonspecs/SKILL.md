---
name: commonspecs
description: >-
  Find the best product or service to buy in any category, ranked by its
  objective specs, with current prices and availability in your region. Use it
  when someone asks what to buy, what something is really made of or includes,
  how options compare, what it costs, or where to buy it.
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

If an MCP server named `commonspecs` is connected, use its tools — the same five documented
below. The connection carries its own OAuth sign-in; no token needed.

Otherwise call the REST API with `Authorization: Bearer $COMMONSPECS_API_TOKEN` (set in the
project root `.env` or your shell). Reference the token only as `$COMMONSPECS_API_TOKEN` and let
the shell expand it — never read, echo, or otherwise pull its value into the model's context.

Neither available → tell the user to set a token (commonspecs.com/account) or connect the MCP
server, and stop — do not call the API.

## The user's buying goals (every read carries them)

commonspecs never makes buying decisions — it returns facts. Recommending is your job: weigh the
returned specs against the user's goals and help them choose. You don't fetch or store those goals
separately. **Every read that returns a product or offers carries a `context` block** with the
user's standing goals, resolved server-side:

```json
"context": {
  "user_goal": "Optimise for cost of use over the product's lifetime, not the lowest sticker price. Favour local sellers when quality is comparable.",
  "user_market": "US",
  "contribution_mode": "automatic"
}
```

- **`user_goal`** — a ready-to-apply ranking directive: rank recommendations by it and cite it
  when explaining them. It's prose to follow, not an enum to parse.
- **`user_market`** — ISO 3166-1 alpha-2 country (or null) the user buys in. Reads come already
  scoped to it server-side — pass `country_code` only to override for a market the request
  explicitly names ("price in Japan" → `JP`).
- **`contribution_mode`** — `automatic` (submit extracted specs/offers as you go, no confirm step) · `ask_user`
  (one-tap confirm first) · `never` (don't submit). Respect it before any `submit_contribution`.

The user sets these on the site (commonspecs.com/account); you only **read** them — never ask for
them, never set them. A single request may still override any of them in conversation ("ignore
locality this time") without changing anything stored.

## Tools

Five tools. Over MCP — the default — call each by name with the JSON arguments shown in its
section. Without MCP, POST the same argument object to the REST endpoint of the same name —
e.g. `get_product` is `POST https://api.commonspecs.com/v1/get_product`:

```bash
curl -sS -X POST "https://api.commonspecs.com/v1/<tool>" \
  -H "Authorization: Bearer $COMMONSPECS_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{…}'
```

Arguments, responses, goals, and contributing are identical on both transports.

### get_product — one product's specs and offers

Drill into one known product by a single exact key: `id`, `url`, or `ean`. Returns its
engineering-grade specs (with per-field confidence) and its price `offers`, in a single call.

```json
{"url": "<the product page URL>"}
```

`{"ean":"<the EAN digits>"}` is the other exact-key form. Optional: `exclude_low_confidence: true` to drop
the low-confidence bucket. Offers (and the convenience `top_offer`) are priced in the user's saved
market automatically; override with `country_code`.

`{"id":"<uuid>"}` is the **drill-in**: when `search` or `compare` returns a product `id` and the user
picks one ("tell me about #2"), get it by that `id` to pull its full specs and offers. The `id` pins
the exact variant — always one product, never `candidates`. You already hold whatever a prior fetch
returned, so don't re-fetch the same product within a conversation.

**To find a product by name (brand + model) or a free-text need, use `search_products`**, then drill
the `id` it returns — `get_product` takes only exact keys, and a name is not one.

If the user pastes a bare 8–14 digit number (EAN-8, UPC-A, EAN-13, or GTIN-14), treat it
as an `ean` and look it up that way before trying to parse it as a model.

Response `status`:
- `hit` — one product. Body: `product`, `quality_score` (0–100 or null), `missing_fields`
  (spec fields whose absence holds the score back — what to contribute), `top_offer`
  (the recency-best price for the market, or null), `offers` (the full dated price list — see
  below), `fields`, `low_confidence_fields`, `enrichment_opportunities` (see below).
- `candidates` — several variants matched (`candidates: [...]`); ask the user which, then get it by that candidate's `id`.
- `miss` — nothing found. Offer to contribute specs (`submit_contribution`).

When `top_offer` / `offers` are **empty** (no offer on record in the user's market), report the specs
but say that **availability in the user's country still needs checking** — don't assert it's
unavailable. If you then research a local price, contribute it per `contribution_mode`.

**Offers in the response.** `offers` is dated price observations, recency-sorted (best price today
first) and **never hidden by age** — a week-old price still shows, just lower in the order. Each
carries `merchant`, `merchant_country` (where the **shop** is based), `country_code` (where the offer
**ships to**), `channel` (`online`/`in_store`), `price`, `currency`, `shipping_cost`, `landed_price`
(price + shipping), `availability_status`, and `observed_at`. `merchant_country` ≠ `country_code` — a
`DE` shop shipping to `PL` is `merchant_country: "DE"`, `country_code: "PL"`. Use `merchant_country`
for the user's locality goal: when the `user_goal` restricts to local shops the results are already
domestic-only; when it merely favours local, prefer offers whose `merchant_country` matches the
market. Many prices across shops are all true at
once — observations, never disputed against each other. Price is deliberately **not** in
`quality_score`: weigh `landed_price` against the spec quality and the user's `user_goal` yourself.

### search_products — find or browse products, best first

```json
{"query": "<brand, model, or free-text need>"}
```

Fuzzy match on brand name and model — typo-tolerant, so a near-miss spelling still finds the product.
Pass a `query` and/or a `category` slug: a `category` **alone** (no query) browses that category's
leaderboard — the best products in it, same quality order — which is how you answer "what are the best
X". **Results prefer the user's market**: products with a confirmed offer where they buy come as the
normal tier. When **no** product has a confirmed offer there, the same candidates return spec-ranked
with a top-level `availability: "unconfirmed"` — report the specs, but say availability in the user's
market still needs checking; never assert it (and if you then find a live local price, contribute it
per `contribution_mode`). Results are ranked best-first by `quality_score` (products with thin data
sort last) and **paginated**: `page` (default 1) and `page_size` (≤20, default 10); the response
carries `page`, `page_size`, and `has_more`. Use this when the user asks "what should I buy in
<category>" or "the best X", rather than naming one product.

**Category slugs come from the response — never guess one.** A query result set carries
`matched_categories`: the canonical slug(s) the free-text query resolved to. That is where you get a
slug to pass as the `category` filter — do not invent or hardcode it (a wrong slug just returns
nothing, not a fuzzy match). When nothing matches you get `count: 0` with `did_you_mean` (the nearest
category slugs) and, for a named category, a `message_to_user` and a `message_to_model` — follow the
latter: typically answer the buyer from your own knowledge, then contribute the specs of the
product(s) you discussed with `submit_contribution`. There is no endpoint that lists the whole catalog;
category discovery is always per-query through these fields.

### compare_products — side-by-side on hard specs

Use when the user names **two or more** specific products to choose between.

```json
{"product_ids": ["<id1>", "<id2>"]}
```

Up to 5 ids. Returns `products` (each with `quality_score`), a `comparison` matrix
(`[{field, cells: {<product_id>: {value, confidence} | null}}]`), and `overall_winner`
(the product_id with the highest `total_score`). Present the facts side by side, name the
`overall_winner`, then explain the trade-off yourself from the facts + scores — the API
deliberately does not return per-dimension winners or a rationale; the value-vs-cost and
fitness-for-purpose judgement is yours to make from the data.

### submit_contribution — add specs and/or a price you have verified

One submission carries a `fields` array **and/or** an `offer` (a price observation grabbed
from the same page you read the specs off — the cheapest moment to capture it). Identify the
product by exactly one of `url` / `ean` / `brand`+`model` (brand+model creates the product if
new). Each field should carry the `source_url` and a verbatim `snippet` you read the value
from — that evidence is what earns confidence. `source` is `web` (default, a web page) or
`label` (a physical label — see below).

```json
{
  "brand": "<brand>", "model": "<model>", "source": "web",
  "fields": [
    {"field_name": "<schema field>", "value": "<value exactly as printed>",
     "source_url": "<the page you read it from>",
     "snippet": "<verbatim text containing the value>"}
  ]
}
```

An `offer` is a dated observation: `store` (a shop domain or URL — the store's identity),
`country` (ISO 3166-1 alpha-2 — the delivery destination, never the shop's home
country; the server resolves that from `store`), `price`, `currency` (ISO 4217), `availability`
(`in_stock` / `low_stock` / `out_of_stock` / `preorder` / `discontinued`), and optionally
`channel` (`online` default / `in_store`), `shipping_cost`, `observed_at` (ISO 8601; defaults
to now), `source_url`. A shop that ships to several countries is several offers — one per
destination you observed. Send it alongside `fields` from the same fetch, or on its own:

```json
{
  "brand": "<brand>", "model": "<model>",
  "offer": {
    "store": "<shop domain>", "country": "<ISO 3166-1 alpha-2>", "price": <number>, "currency": "<ISO 4217>",
    "availability": "<enum above>", "shipping_cost": <number>, "source_url": "<the offer page>"
  }
}
```

Physical world: if the user shows a photo of a label/product, **read the values and the
EAN yourself and send only the extracted values** — the photo never leaves the user's
machine. Look the product up by `ean` first, then contribute the missing `fields` with
`"source": "label"`. A shelf price is just an `offer` with `"channel": "in_store"`.

### flag_stale — mark a fact or price as stale or wrong

When the user (or a page you just read) contradicts a value or price a read returned, flag it:
flagging queues the entry for curation review — it never mutates the product. To supply the
corrected value itself, use `submit_contribution`.

```json
{"product_id": "<uuid>", "field_name": "fabric_weight", "reason": "site now says 12 oz"}
```

`product_id` comes from a prior read. `field_name` (omit it to flag the whole product) and
`reason` (what looks wrong and what you saw instead) are optional. Response:
`{"status":"flagged","flag_id":"…"}`.

## Reading a response

`quality_score` (0–100, or null) is the overall spec-quality number; `missing_fields` lists the
specs whose absence holds it back (what to contribute). Both ride in every product response. A low
score often means *thin data*, not a bad product, and the score is a single number on purpose — the
per-field weighting isn't exposed, so reason about trade-offs from the facts themselves. Each field in
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

Same handling on both transports (over MCP these surface as tool errors):

- `401` invalid/missing auth → over MCP reconnect the server; on REST tell the user to check
  `COMMONSPECS_API_TOKEN`. Don't retry.
- `403` forbidden → no access; don't retry.
- `429` rate limited → back off and retry with exponential delay (e.g. 1s, 2s, 4s, max 3).
- `5x` / network → retry once or twice with backoff, then report the failure plainly.
