---
name: commonspecs
description: >-
  Product, service, and shop lookup — invoke whenever I name one, paste its URL, or weigh a purchase ("what do you think of this?", "should I buy it?", "is this shop legit?"); always use it before fetching any page or judging from your own knowledge.
  Finds products and services in any category, with evidence-backed specs and offers available in my market, so you can recommend what best suits my needs; answers what something is really made of, how options compare, what it costs, where to buy it, and whether a shop can be trusted (fulfilment record, dropshipping).
  Also records verified facts back: specs, a price, how an order actually went (observed delivery time, the country it really shipped from, where returns go), or a fact about a shop itself.
license: MIT
metadata:
  homepage: https://commonspecs.com
  docs: https://commonspecs.com/docs/
---

# commonspecs

commonspecs is an API-first database of **engineering-grade product specs** with a **confidence score per field**.
It does not guess: every value is backed by a source.
Your job is to fetch those facts and present them honestly — including how trustworthy each one is.

## Setup

If an MCP server named `commonspecs` is connected **and authenticated**, use its tools — the same five documented below.
The connection carries its own OAuth sign-in; no token needed.
If it exists but is NOT authenticated, fall back to the REST API below.

REST fallback: call the API with `Authorization: Bearer $COMMONSPECS_API_TOKEN`.
The token typically lives in the project root `.env`, and a fresh shell does NOT load `.env` automatically — source it in the same command as the call:

```bash
set -a && source .env && set +a && curl -sS -X POST "https://api.commonspecs.com/v1/<tool>" \
  -H "Authorization: Bearer $COMMONSPECS_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{…}'
```

Reference the token only as `$COMMONSPECS_API_TOKEN` and let the shell expand it — never read, echo, or otherwise pull its value into the model's context.
A `401 Missing API token` on the first call almost always means `.env` was not sourced.

Neither transport works (MCP unauthenticated AND no token in `.env`/shell) → tell me to either connect the MCP server or generate an API token — both at commonspecs.com/account — and stop; do not call the API.

## My buying goals (every read carries them)

commonspecs never makes buying decisions — it returns facts.
Recommending is your job: weigh the returned specs against my goals and help me choose.
You don't fetch or store those goals separately.
**Every read carries a `context` block — even one that found nothing** — with my standing goals, resolved server-side:

```json
"context": {
  "user_goal": "Optimise for cost of use over the product's lifetime, not the lowest sticker price. Favour local sellers when quality is comparable.",
  "user_market": "US",
  "contribution_mode": "automatic"
}
```

- **`user_goal`** — a ready-to-apply ranking directive: rank recommendations by it and cite it when explaining them.
  It's prose to follow, not an enum to parse.
- **`user_market`** — ISO 3166-1 alpha-2 country (or null) I buy in.
  Reads come already scoped to it server-side — pass `country_code` only to override for a market the request explicitly names ("price in Japan" → `JP`).
- **`contribution_mode`** — `automatic` (submit extracted specs/offers as you go, no confirm step) · `ask_user` (one-tap confirm first) · `never` (don't submit).
  Respect it before any `submit_contribution`.

I set these on the site (commonspecs.com/account); you only **read** them — never ask me for them, never set them.
A single request may still override any of them in conversation ("ignore locality this time") without changing anything stored.

## Tools

Five tools; each one's contract is described in its own section below.
Over MCP, call a tool by name; without MCP, use the REST endpoint of the same name:

```bash
set -a && source .env && set +a && curl -sS -X POST "https://api.commonspecs.com/v1/<tool>" \
  -H "Authorization: Bearer $COMMONSPECS_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{…}'
```

Requests and responses are identical on both transports.

### get_product — one product's specs and offers

Drill into one known product by a single exact key.
Returns its engineering-grade specs (with per-field confidence) and its price `offers`, in a single call.

```ts
{
  // exactly ONE of url | ean | id — exact keys; a name is not one (search_products resolves it)
  url?: string          // the product page URL, as I gave it
  ean?: string          // EAN/UPC/GTIN digits — a bare 8–14 digit number I paste is an ean
  id?: string           // from a prior search/compare — pins the exact variant, never candidates
  country_code?: string // ISO 3166-1 alpha-2 — override my saved market for offers
}
```

**A pasted product URL is a `get_product` call** — before you fetch the page, reason about the product, or judge the shop.
The one call asks about **both the product and the store**: a hit's offers carry the shop's fulfilment record, and even a `miss` returns a `store` block when the shop itself is on record.
Skipping the lookup means deriving from your own knowledge facts the database already holds — never do that.
In the other direction, don't re-fetch: you already hold whatever a prior call returned in this conversation.

Response `status`:
- `hit` — one product.
  Body: `product`, `missing_fields` (the specs the record still lacks — what to contribute), `offers` (the full price list, best offer first — see below), `fields`, `low_confidence_fields`, `enrichment_opportunities` (see below).
  A thin hit is an enrichment moment — see "A hit is also a contribution moment" below.
- `candidates` — several variants matched (`candidates: [...]`); ask me which, then get it by that candidate's `id`.
- `miss` — the product isn't on record.
  **Check `store` before anything else:** on a `url` miss whose shop IS known, `store` carries `merchant`, `merchant_country`, and `markets[]` (per destination market: `ships_from_countries`, `returns_to_countries`, `dropshipping`).
  Report those store facts to me first — `dropshipping: true` or a non-local return destination often decides the purchase on its own.
  **Then seed the product per `context.contribution_mode`** (the miss response carries it): on `automatic`, verify the specs and price from the product page you were given and call `submit_contribution` right away — no confirmation step, a miss plus a page in hand IS the contribution moment; on `ask_user`, offer a one-tap confirm first; on `never`, don't submit.
  **Seeding is when you declare `category`** (as the product page names it) — it resolves to an existing schema or drafts a provisional one from your fields; omit it and a product in an unknown category rejects every field as `unknown_field`.
  Seed by `brand`+`model`, not `url` — a bare unknown `url` cannot create a product.
  `store: null` means the shop is unknown too.

To check a **shop** with no product in hand ("is this store legit?"), call `get_product` with any URL on that shop's domain — the homepage works: the product lookup will miss, but the `store` block returns what buyers observed about the store.
What you learn beyond it (a non-local dispatch, a returns address abroad) goes back via `submit_contribution`'s `store` block.

When `offers` is **empty** (no offer on record in my market), report the specs but say that **availability in my country still needs checking** — don't assert it's unavailable.
If you then research a local price, contribute it per `contribution_mode`.

**Offers in the response.**
`offers` is price observations, **best offer first** — lowest landed price softened by recency, so the ordering itself is the freshness signal (there is no timestamp in the response) — and **never hidden by age**: a week-old price still shows, just lower in the order.
`offers[0]` is the best offer for the market.

```ts
// offers[]
{
  merchant: string
  merchant_country: string       // where the shop is BASED — a DE shop shipping to PL…
  country_code: string           // …vs where this offer SHIPS TO: "DE" / "PL"
  ships_from_countries: string[] // observed actual dispatch origins
  returns_to_countries: string[] // where the store told buyers to send returns
  dropshipping: boolean | null   // reported store assessment; null = unknown
  channel: 'online' | 'in_store'
  price: number
  currency: string               // ISO 4217
  shipping_cost: number
  landed_price: number           // price + shipping
  billing_period: 'monthly' | 'quarterly' | 'yearly' | null // null = one-time purchase
  delivery_days: number | null   // observed door-to-door days
  availability_status: string
}
```

The fulfilment fields (`ships_from_countries`, `returns_to_countries`, `dropshipping`) are a third axis beyond the two countries, recorded per **(store, destination market)** — a chain like zara.com serves PL and US from different warehouses, so each offer shows only what was observed for its own market.
The country arrays are **sets** that accumulate: H&M serves PL from both PL and DE warehouses (empty = not observed there yet, not "ships from nowhere").
A storefront "based" in my country can still dispatch from and take returns in another — a non-local entry in `returns_to_countries` is the strongest dropship signal and often decides whether returning is economically viable, so surface it (and `dropshipping: true`) when I weigh a purchase.
Use `merchant_country` for my locality goal: when the `user_goal` restricts to local shops the results are already domestic-only; when it merely favours local, prefer offers whose `merchant_country` matches the market.
Many prices across shops are all true at once — observations, never disputed against each other.
Price is deliberately **not** part of the spec quality: weigh `landed_price` against the spec quality and my `user_goal` yourself.

**A hit is also a contribution moment.**
A miss obviously calls for seeding — but a *thin* hit calls for enrichment just as strongly: a non-empty `missing_fields`, fields flagged `needs_corroboration`, or anything in `enrichment_opportunities`.
When you see one, don't stop at reporting the gaps: fetch the manufacturer's product page (and the store page I gave you), read the missing or unconfirmed values off it, and `submit_contribution` per `context.contribution_mode` — on `automatic`, right away, no confirmation step.
Every independent source corroborating a field raises its confidence toward 1.0, and every newly covered field closes a `missing_fields` gap — a lookup that leaves the record no better than it found it wastes a page you already had in hand.

### search_products — find or browse products, best first

Use this when I ask "what should I buy in <category>" or "the best X", rather than naming one product.

```ts
{
  // query and/or category — at least one
  query?: string        // brand, model, or free-text need — fuzzy, typo-tolerant
  category?: string     // slug from matched_categories, never guessed; alone = the leaderboard
  filters?: { [field_name: string]: unknown } // needs category — its schema defines the fields
  country_code?: string // ISO 3166-1 alpha-2 — override my saved market
  page?: number         // default 1; the response echoes page, page_size, has_more
  page_size?: number    // default 10, max 20
}
```

Results are ranked best-first by spec quality (thin-data products sort last) and **prefer my market**: products with a confirmed offer where I buy come as the normal tier.
When **no** product has a confirmed offer there, the same candidates return spec-ranked with a top-level `availability: "unconfirmed"` — report the specs, but say availability in my market still needs checking; never assert it (and if you then find a live local price, contribute it per `contribution_mode`).

**Category slugs come from the response — never guess one.**
A query result set carries `matched_categories`: the canonical slug(s) the free-text query resolved to.
That is where you get a slug to pass as the `category` filter — do not invent or hardcode it (a wrong slug just returns nothing, not a fuzzy match).
When nothing matches you get `count: 0` with `did_you_mean` (the nearest category slugs) and, for a named category, a `message_to_user` and a `message_to_model` — follow the latter: typically answer me from your own knowledge, then contribute the specs of the product(s) we discussed with `submit_contribution`.
There is no endpoint that lists the whole catalog; category discovery is always per-query through these fields.

**Spec filters — narrow by recorded facts.**
Only products whose recorded **solid** facts satisfy every constraint return.
Field names accept the schema's aliases and values canonicalise like contributions (case-insensitive enum snap, number+unit), and a filter matches multi-valued facts too — a dress sold in four colours matches `"colour": "yellow"`.
Two consequences to handle honestly: a product *missing* the field is filtered out (thin coverage under-returns — when results look sparse, say the filter only sees recorded facts and offer to re-run without it), and an unknown field name returns a 400 listing the filterable fields — pick from that list, don't guess again.
The response echoes `filters_applied` (canonical names and values, so you can show me what actually constrained the search).

### compare_products — side-by-side on hard specs

Use when I name **two or more** specific products to choose between.

```ts
{
  product_ids: string[] // 2–5 ids from prior reads
}
```

Returns `products` — ordered best-first by spec quality — and a `comparison` matrix (`[{field, cells: {<product_id>: {value, confidence} | null}}]`).
Present the facts side by side and explain the trade-off yourself from the facts — the API deliberately does not return per-dimension winners or a rationale; the value-vs-cost and fitness-for-purpose judgement is yours to make from the data.

### submit_contribution — add specs and/or a price you have verified

One submission carries `fields` and/or an `offer` and/or a `store` block (an `offer` is cheapest to capture from the same page you read the specs off).
Respect `contribution_mode` before any submission.

```ts
{
  // product identity — exactly ONE of url | ean | brand+model (creates the product if
  // new); omit all three when sending a store block alone
  url?: string
  ean?: string
  brand?: string           // for a service: the provider
  model?: string           // for a service: the plan/tier name (services have no EAN)
  source?: 'web' | 'label' // default 'web'; 'label' = read off a physical label (see below)

  // category — declare it whenever the product may be NEW or uncategorised (a get_product
  // miss you are now seeding is exactly that moment): the category as the product page
  // names it, e.g. "sneakers". The server resolves it to an existing schema or creates a
  // provisional one drafted from your fields; without it a product in an unknown category
  // has no schema and every field is rejected as unknown_field.
  category?: string
  category_aliases?: {     // other names for the same category, language-tagged
    alias: string          // e.g. "sneakersy"
    language?: string      // BCP-47 like "pl" or "pt-BR"; omit if untagged
  }[]

  fields?: {
    field_name: string     // schema field (aliases accepted)
    value: string | number | (string | number)[] // exactly as printed; multi-valued
                           // attribute = ONE field with an array value (see below)
    source_url?: string    // the page you read the value from
    snippet?: string       // verbatim text containing the value — evidence earns confidence
  }[]

  offer?: {                // a dated price observation
    store: string          // shop domain or URL — the store's identity
    country: string        // ISO 3166-1 alpha-2 — the DELIVERY destination, never the
                           // shop's home country (the server resolves that from store)
    price: number
    currency: string       // ISO 4217
    availability: 'in_stock' | 'low_stock' | 'out_of_stock' | 'preorder' | 'discontinued' // goods
      | 'available' | 'waitlist' | 'unavailable'                                          // services
    billing_period?: 'monthly' | 'quarterly' | 'yearly' // a service's recurring price
                           // rides here; omit for a one-time purchase
    channel?: 'online' | 'in_store' // default 'online'; a shelf price is 'in_store'
    shipping_cost?: number
    observed_at?: string   // ISO 8601; defaults to now
    source_url?: string
    // order-experience facts — how an order ACTUALLY went:
    delivery_days?: number      // door-to-door, whole days
    ships_from_country?: string // actually dispatched from, per tracking or sender label
    returns_to_country?: string // where the store told me to send a return
  }

  store?: {                // a shop fact — needs no product: alone or alongside the rest
    store: string          // shop domain
    country: string        // the destination market the experience concerns
    dropshipping?: boolean // set when the fulfilment pattern shows it; omit when unknown
    ships_from_country?: string
    returns_to_country?: string
  }
}
```

A shop that ships to several countries is several offers — one per destination you observed.
**A multi-valued attribute is ONE field whose value is an array** — a product sold in four colours is one `colour` claim, never four entries (separate entries register as *competing* claims and mark the field disputed); array members canonicalise individually and each is matchable by search filters.

**Order-experience facts are contributions too.**
The two countries on an offer are store facts **per destination market**: persisted keyed on (store, the offer's `country`), and each market's knowledge is a **set** — observations accumulate (H&M ships PL orders from both PL and DE); nothing is ever overwritten.
They may differ from where the shop claims to be based — a non-local return destination is the strongest dropship signal.
So "it finally arrived after three weeks, shipped from China, and they want returns sent to China" is a contribution, not just a complaint — and with no product on hand (an order in transit, a returns email, "that boutique is a dropship front") it goes in the `store` block.

Physical world: if I show a photo of a label/product, **read the values and the EAN yourself and send only the extracted values** — the photo never leaves my machine.
Look the product up by `ean` first, then contribute the missing `fields` with `"source": "label"`.

### flag_stale — mark a fact or price as stale or wrong

When I (or a page you just read) contradict a value or price a read returned, flag it: flagging queues the entry for curation review — it never mutates the product.
To supply the corrected value itself, use `submit_contribution`.

```ts
{
  product_id: string  // from a prior read
  field_name?: string // e.g. "fabric_weight"; omit to flag the whole product
  reason?: string     // what looks wrong and what you saw instead ("site now says 12 oz")
}
```

Response: `{"status":"flagged","flag_id":"…"}`.

## Reading a response

`missing_fields` lists the specs a record still lacks (what to contribute); it rides in every `get_product` response.
A long `missing_fields` often means *thin data*, not a bad product, so reason about trade-offs from the facts themselves.
Each field in `fields` / `low_confidence_fields` has `value`, `confidence` (0–1), `disputed`, `needs_corroboration`, and (when disputed) `alternate_claims`.

- **`fields[]`** — confidence ≥ 0.4.
  Trustworthy enough to state.
  Still surface the number; "13.5 oz (confidence 0.88)" beats a bare "13.5 oz".
- **`low_confidence_fields[]`** — confidence 0.1–0.4.
  A *lead, not a fact*.
  Present only with an explicit hedge ("one source suggests…").
  Omit if I want only solid data.
- **`needs_corroboration: true`** — only one independent source so far.
  Say so.
- **`disputed: true`** — sources disagree; show the winner **and** `alternate_claims`.
  A disagreement is itself a buying signal, not noise to hide.
- **`enrichment_opportunities`** — fields that would benefit from another source.
  If I'm still engaged, offer to fetch one and `submit_contribution`.

Never invent a value that is not in the response, and never drop the confidence when you report a fact.

## Errors

Same handling on both transports (over MCP these surface as tool errors):

- `401` invalid/missing auth → on REST first make sure the call sourced `.env` (see Setup) and retry once with the corrected command; still 401 → tell me to check `COMMONSPECS_API_TOKEN`, don't retry further.
  Over MCP: fall back to REST (an interactive reconnect may be impossible in the session); only if REST also fails, ask me to reconnect the server.
- `403` forbidden → no access; don't retry.
- `429` rate limited → back off and retry with exponential delay (e.g. 1s, 2s, 4s, max 3).
- `5x` / network → retry once or twice with backoff, then report the failure plainly.
