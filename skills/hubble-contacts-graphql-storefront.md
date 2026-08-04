---
name: Query the Hubble Contacts storefront over GraphQL
description: Read the full Hubble Contacts catalog — products, prescription variants, subscription selling plans, collections, pricing and localization — and build a cart, over the anonymous Shopify Storefront GraphQL API served on Hubble's own host. This is the richest surface on the property and needs no credentials.
api: https://account.hubblecontacts.com/api/2026-01/graphql.json
generated: '2026-08-04'
method: generated
source: graphql/hubble-contacts-storefront.graphql (SDL captured by live anonymous introspection 2026-08-04)
operations:
  - query products
  - query product
  - query productByHandle
  - query collection
  - query collections
  - query search
  - query predictiveSearch
  - query productRecommendations
  - query shop
  - query localization
  - query paymentSettings
  - query cart
  - query publicApiVersions
  - mutation cartCreate
  - mutation cartLinesAdd
  - mutation cartLinesUpdate
  - mutation cartLinesRemove
  - mutation cartBuyerIdentityUpdate
  - mutation cartDeliveryAddressesAdd
  - mutation cartSelectedDeliveryOptionsUpdate
  - mutation cartPrepareForCompletion
  - mutation cartSubmitForCompletion
---

# Query the Hubble Contacts storefront over GraphQL

Use this when you need **more than the storefront JSON gives you** — subscription
plans, price ranges, availability per prescription power, collection facets, market
and currency context — or when you want to build a cart **without** registering a
UCP agent profile.

## Endpoint

```
POST https://account.hubblecontacts.com/api/2026-01/graphql.json
Content-Type: application/json
```

No `X-Shopify-Storefront-Access-Token` was required on any call made while writing
this skill. Introspection is open, so you can read the whole schema yourself; the
captured SDL sits at `graphql/hubble-contacts-storefront.graphql` in this repo.

Ask the server which versions it still supports before pinning one:

```graphql
{ publicApiVersions { handle displayName supported } }
```

At capture time `2025-10`, `2026-01`, `2026-04` and `2026-07` were supported,
`2026-07` was latest, and `2026-10` was a release candidate.

## Read the catalog

```graphql
query Catalog($n: Int!, $cursor: String) {
  products(first: $n, after: $cursor) {
    pageInfo { hasNextPage endCursor }
    edges {
      node {
        id
        handle
        title
        vendor
        productType
        tags
        availableForSale
        totalInventory
        requiresSellingPlan
        priceRange { minVariantPrice { amount currencyCode } }
        options { name optionValues { name } }
        sellingPlanGroups(first: 5) {
          edges { node { name sellingPlans(first: 10) { edges { node { id name } } } } }
        }
        variants(first: 100) {
          edges { node { id title sku availableForSale price { amount currencyCode } } }
        }
      }
    }
  }
}
```

`first` maxes out at **250**. Page with `pageInfo.endCursor` — this is a Relay
cursor connection, not page numbers.

Prescription powers are **variant options**, not separate products: one lens product
carried 54 variants at probe time. Read `options` first, then match a variant with
`variantBySelectedOptions`. Use `options { optionValues { name } }` — the older
`options { values }` field is one of 55 fields this schema marks `@deprecated`.

Subscription is Hubble's whole business model and it only appears on this surface —
`requiresSellingPlan` plus `sellingPlanGroups`. Neither the storefront JSON nor the
documented UCP tool list exposes it.

## Search

```graphql
{ search(query: "daily lenses", first: 10, types: PRODUCT) {
    edges { node { ... on Product { handle title } } } } }
```

Use `predictiveSearch` for type-ahead, `productRecommendations(productId:)` for
related items, and `productTags` / `productTypes` to enumerate the taxonomy.

## Price the buyer correctly

```graphql
{ shop { name }
  paymentSettings { currencyCode countryCode acceptedCardBrands }
  localization { country { isoCode currency { isoCode } } availableCountries { isoCode } } }
```

Probed store context was `Hubble Contacts`, `USD`, `US`. Pass buyer country/currency
the same way Hubble's `/agents.md` asks you to on the UCP surface.

## Build a cart

```graphql
mutation Build($lines: [CartLineInput!]!) {
  cartCreate(input: { lines: $lines }) {
    cart { id checkoutUrl totalQuantity cost { totalAmount { amount currencyCode } } }
    userErrors { field message }
  }
}
```

Then `cartLinesAdd` / `cartLinesUpdate` / `cartLinesRemove`,
`cartBuyerIdentityUpdate`, `cartDeliveryAddressesAdd`,
`cartSelectedDeliveryOptionsUpdate`, and finally `cartPrepareForCompletion` →
`cartSubmitForCompletion`.

**The buyer-approval rule still applies.** Hubble's `/agents.md` states agents must
not complete payment without explicit, contemporaneous buyer consent. Reaching the
same outcome over GraphQL instead of UCP does not make it consented. Hand the buyer
`cart.checkoutUrl`, or route through the Shop skill Hubble points at, and stop.

## Errors

GraphQL returns `HTTP 200` even when the call failed. Branch on the body:

- schema/validation problems land in `errors[]` with a typed
  `extensions.code` (`undefinedField` was observed).
- business failures land in the mutation's own `userErrors[]` (`CartUserError`,
  `CustomerUserError`) while `data` stays 200.

## Cost and back-off

Every response carries `extensions.cost.requestedQueryCost` and the headers
`shopify-complexity-score` / `shopify-complexity-score-v2`. `{shop{name}}` cost 1;
`products(first:250)` cost 23. Budget against those rather than firing blind and
waiting for a 429.

## What this surface will NOT do

- No orders without a `customerAccessToken` (and Shopify steers order history to the
  Customer Account API, reachable through the OAuth/OIDC issuer in
  `authentication/`).
- No checkout object — this schema version promotes the cart instead.
- No cart cancellation.

See `mcp/hubble-contacts-tool-crosswalk.yml` for the full surface-by-surface map.
