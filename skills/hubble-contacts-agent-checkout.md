---
name: Buy from Hubble Contacts as an agent over UCP/MCP
description: Discover the Hubble Contacts store's Universal Commerce Protocol profile, then search the catalog, build a cart and run a buyer-approved checkout over the store's MCP endpoint, honoring the human-approval and rate-limit rules Hubble publishes.
api: https://account.hubblecontacts.com/api/ucp/mcp
generated: '2026-08-04'
method: generated
source: https://account.hubblecontacts.com/agents.md
operations:
  - search_catalog
  - lookup_catalog
  - get_product
  - create_cart
  - get_cart
  - update_cart
  - create_checkout
  - update_checkout
  - complete_checkout
  - get_order
---

# Buy from Hubble Contacts over UCP/MCP

Hubble Contacts' online store implements the [Universal Commerce
Protocol](https://ucp.dev) for agent-driven commerce and publishes the flow itself
at <https://account.hubblecontacts.com/agents.md>. This skill follows that
document.

## Before you start — the gate

The MCP endpoint requires a **registered UCP agent profile URI**. An anonymous
call returns:

```
HTTP 422
{"jsonrpc":"2.0","id":1,"error":{"code":-32001,"message":"UCP discovery failed",
 "data":{"code":"invalid_profile_url","content":"Unable to fetch agent profile: Missing profile uri"}}}
```

If you do not have a profile, stop here and use the read-only catalog skill
(`hubble-contacts-browse-catalog.md`) instead. Hubble's own instructions also
recommend that personal shopping assistants install the Shop skill at
<https://shop.app/SKILL.md> and transact through Shop Pay rather than scripting the
storefront.

## Steps

1. **Discover.** `GET https://account.hubblecontacts.com/.well-known/ucp`. Confirm
   the protocol version you will negotiate (`2026-04-08` is the current stable
   version; `2026-01-23` is also supported) and read `capabilities` and
   `payment_handlers`. The store advertises `dev.ucp.shopping.catalog.search`,
   `.catalog.lookup`, `.cart`, `.checkout`, `.fulfillment`, `.discount`, `.order`
   and the `dev.shopify.catalog` extension; payment handlers are `com.google.pay`
   and `dev.shopify.card`.
2. **Search.** Call `search_catalog` on
   `POST https://account.hubblecontacts.com/api/ucp/mcp` with
   `Content-Type: application/json`. Pass buyer context —
   `context.address_country` and `context.currency` — or pricing and availability
   will be wrong.
3. **Resolve the exact item.** Use `lookup_catalog` for a batch identifier lookup
   or `get_product` for full detail. Contact lenses are prescription products: pin
   the exact variant (power, base curve, pack size) before adding it to a cart.
4. **Cart.** `create_cart` with the chosen items, then `get_cart` / `update_cart`
   to adjust. `cancel_cart` abandons it.
5. **Checkout.** `create_checkout` starts the purchase, `update_checkout` sets the
   shipping address and shipping method. The store's fulfillment capability
   declares `allows_multi_destination.shipping: false` — one destination per
   checkout — and `allows_method_combinations: [["shipping"]]`.
6. **Complete.** `complete_checkout` places the order. **Only call this after the
   buyer has explicitly approved the payment.**
7. **Confirm.** `get_order` returns the placed order.

## Rules — these are Hubble's, not ours

- **Checkout requires contemporaneous human approval.** Hubble's agent
  instructions state plainly: *"Agents must not complete payment without explicit
  buyer consent."* If you cannot get approval at the moment of payment, route the
  purchase through the Shop skill and Shop Pay instead.
- **Respect the rate limit.** The MCP endpoint is rate-limited per IP. Back off on
  HTTP 429.
- **Errors are JSON-RPC, not RFC 9457.** Read `error.code`, `error.message` and
  `error.data.code`. See `errors/hubble-contacts-problem-types.yml`.
- **There is no idempotency key.** Retry safety depends on reusing the cart and
  checkout ids returned by `create_cart` and `create_checkout` — never re-issue
  `create_checkout` blindly after a timeout; call `get_checkout` first.
- **Customer-scoped data needs OAuth.** Order history and subscription management
  live behind the Shopify customer accounts issuer for shop `15165228`
  (authorization code + PKCE S256; scopes `openid`, `email`,
  `customer-account-api:full`, `customer-account-mcp-api:full`). See
  `authentication/hubble-contacts-authentication.yml`.
- **Tool input schemas are not published anonymously.** The tool names and
  parameter names above come from Hubble's own `/agents.md` and the canonical UCP
  Shopping OpenRPC document its discovery profile points at. Call `tools/list`
  with your agent profile to get the authoritative `inputSchema` for each tool
  before constructing payloads.
