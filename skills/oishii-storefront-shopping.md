---
name: Shop the Oishii storefront via MCP
description: Search Oishii's catalog, inspect a product variant, build a cart, and hand the buyer a checkout URL — using the anonymous storefront MCP server at https://oishii.com/api/mcp, without ever completing a payment autonomously.
api: mcp/oishii-mcp.yml
server: https://oishii.com/api/mcp
transport: JSON-RPC 2.0 over HTTP POST (MCP protocol 2025-06-18)
operations:
  - search_catalog
  - get_product_details
  - update_cart
  - get_cart
  - search_shop_policies_and_faqs
generated: '2026-08-04'
method: generated
source: mcp/oishii-storefront-mcp-tools.json
---

# Shop the Oishii storefront via MCP

Oishii sells indoor-grown Japanese strawberries (the Omakase Berry, the Koyo Berry) and strawberry
preserves direct to consumers. The store is agent-addressable: an anonymous MCP server at
`https://oishii.com/api/mcp` exposes catalog, product, cart and policy tools with real JSON Schema input
contracts. Every tool name and parameter below was read from a live `tools/list` response and is saved
verbatim in `mcp/oishii-storefront-mcp-tools.json`.

## Before you start

- **No credentials are needed.** `tools/list` and all five tools answer anonymously.
- **Set `Accept: application/json, text/event-stream`** and `Content-Type: application/json`.
- **Never complete a payment.** Both `/agents.md` and `/robots.txt` state that checkout, payment and order
  placement must not be finalized without an explicit, contemporaneous human approval step. This skill
  stops at the checkout URL. If you are a buy-for-me agent, route the purchase through the UCP endpoint
  (`https://oishii.com/api/ucp/mcp`, which requires a UCP agent profile URI) or the Shop skill at
  `https://shop.app/SKILL.md`, both of which enforce buyer approval.
- **Back off on 429.** The endpoint is rate limited per IP.

## Steps

### 1. Find candidate products — `search_catalog`

Pass at least one of `catalog.query` or `catalog.filters`. Always send buyer context so pricing and
availability are correct:

```json
{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"search_catalog","arguments":{
  "catalog":{
    "query":"omakase strawberries",
    "context":{"address_country":"US","currency":"USD","language":"en"},
    "filters":{"price":{"max":10000}},
    "pagination":{"limit":10}
  }}}}
```

- Prices in `filters.price` are **ISO 4217 minor units** — `10000` is $100.00.
- Results are cursor paginated: send `catalog.pagination.cursor` from the previous response to page.
  Default page size is 10, maximum 250, and the server may clamp lower.
- The response conforms to the UCP catalog-search capability (`dev.ucp.shopping.catalog.search`).

### 2. Confirm the exact variant — `get_product_details`

`product_id` is required and is a Shopify GID (`gid://shopify/Product/123`). Pass `options` to select a
variant; without it you get the first available variant. `country` and `language` localize the result.

```json
{"name":"get_product_details","arguments":{"product_id":"gid://shopify/Product/123","country":"US"}}
```

Read the variant `id` from the response — that is the `product_variant_id` the cart needs.

### 3. Build the cart — `update_cart`

Omit `cart_id` to create a new cart; `add_items` is required when creating one.

```json
{"name":"update_cart","arguments":{"add_items":[{"product_variant_id":"<variant gid>","quantity":1}]}}
```

To change the cart later, pass the returned `cart_id` with:
- `update_items` — `[{"id":"<line id>","quantity":2}]`; **quantity `0` removes the line**
- `remove_line_ids` — explicit line-item removal
- `buyer_identity`, delivery details, discount codes and notes as the schema allows

There is **no idempotency key** on this surface (see `conventions/oishii-conventions.yml`). Do not blindly
retry `add_items` on a network error — re-read the cart with `get_cart` first and reconcile quantities.

### 4. Show the buyer the cart — `get_cart`

```json
{"name":"get_cart","arguments":{"cart_id":"gid://shopify/Cart/c1-...?key=..."}}
```

Returns line items, shipping options, discount info and the **checkout URL**. Hand that URL to the human.
Stop here.

### 5. Answer policy questions — `search_shop_policies_and_faqs`

Natural-language retrieval over shipping, refunds, privacy, terms and FAQ content. `query` is required;
`context` carries any extra situational detail.

```json
{"name":"search_shop_policies_and_faqs","arguments":{"query":"how is the fruit shipped and is it refundable?"}}
```

Use this before promising a delivery window — Oishii ships fresh produce and the shipping policy is the
authority, not the product page.

## Fallback: no MCP client

The same catalog is readable as plain JSON with no MCP client and no credentials:

- `GET https://oishii.com/products.json`
- `GET https://oishii.com/products/{handle}.json`
- `GET https://oishii.com/collections/{handle}/products.json`

## Errors

Errors come back as JSON-RPC 2.0 error objects, not `application/problem+json`. The one gate you will hit
is on the **other** endpoint: `POST https://oishii.com/api/ucp/mcp` without a UCP agent profile returns
HTTP 422 with `-32001 invalid_profile_url`. See `errors/oishii-problem-types.yml`.
