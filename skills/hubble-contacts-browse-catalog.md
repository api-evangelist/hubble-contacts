---
name: Browse the Hubble Contacts catalog without authenticating
description: Read Hubble Contacts product, variant and collection data over the unauthenticated storefront JSON endpoints the store documents in its own /agents.md, for price checks, availability and prescription-variant lookups.
api: https://account.hubblecontacts.com
generated: '2026-08-04'
method: generated
source: https://account.hubblecontacts.com/agents.md
operations:
  - GET /products.json
  - GET /products/{handle}.json
  - GET /collections/{handle}/products.json
  - GET /collections/all
  - GET /search?q={query}&type=product
  - GET /sitemap.xml
---

# Browse the Hubble Contacts catalog (read-only)

Use this when you only need to **read** store data — you are answering a question
about what Hubble sells, what a lens costs, or whether a prescription power is
available. Do not use the UCP/MCP surface for this; it is gated and unnecessary.

Hubble documents this surface itself at
<https://account.hubblecontacts.com/agents.md> under "Read-Only Browsing (No
Authentication Required)".

## Base

All paths are relative to `https://account.hubblecontacts.com`. No credentials,
no API key, no bearer token.

## Steps

1. **List the catalog.** `GET /products.json` returns a `products[]` envelope. At
   the time this skill was written it returned 30 products. Each product carries
   `id`, `title`, `handle`, `vendor`, `product_type`, `tags`, `options[]`,
   `images[]` and `variants[]`.
2. **Resolve a single product.** `GET /products/{handle}.json` where `handle` is
   the URL slug from step 1 (for example `classic-2-0`). This is much cheaper than
   pulling the full list when you already know the product.
3. **Search by keyword.** `GET /search?q={query}&type=product` when the user gave
   you a product name rather than a handle.
4. **Read a collection.** `GET /collections/{handle}/products.json` returns the
   same `products[]` envelope scoped to one collection. `GET /collections/all` is
   the human page for the whole catalog.
5. **Pick the right variant.** Prescription products fan out into many variants —
   the `Hubble 30 box` product carried 54 at probe time. Match on
   `option1`/`option2`/`option3` and check `available` before quoting a price.
   Prices are on the **variant**, not the product: read `price` and
   `compare_at_price`.
6. **Discover URLs.** `GET /sitemap.xml` is a sitemap index if you need to walk
   the store's page structure.

## Rules

- **Vendor is not brand-neutral.** Hubble resells third-party lenses. Check
  `vendor` — values observed include `Hubble Contacts` and `Bausch & Lomb` — before
  telling a user a lens is a Hubble house brand.
- **404s are HTML, not JSON.** Branch on the HTTP status code. Do not try to parse
  the body of an error response; the storefront returns an HTML error page.
- **There is no documented idempotency key and no rate-limit header** on this
  surface. Be conservative with request volume anyway.
- **Do not use this surface to transact.** Purchasing runs through the UCP/MCP
  surface — see `hubble-contacts-agent-checkout.md`.
- **Correlate with `x-request-id`** from the response headers if you need to report
  a problem to Hubble.
