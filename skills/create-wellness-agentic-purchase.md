---
name: create-wellness-agentic-purchase
description: >-
  Search the Create Wellness catalog, build a cart, and take a checkout right up
  to — but never through — payment on the store's UCP commerce MCP endpoint.
  Every tool named here was read from a live tools/list response, not invented.
api: Create Wellness UCP Commerce MCP
endpoint: https://trycreate.co/api/ucp/mcp
protocol: UCP 2026-04-08 over MCP
generated: '2026-08-11'
method: generated
source: mcp/create-wellness-mcp-tools.json
operations:
  - search_catalog
  - lookup_catalog
  - get_product
  - create_cart
  - get_cart
  - update_cart
  - cancel_cart
  - create_checkout
  - get_checkout
  - update_checkout
  - complete_checkout
  - cancel_checkout
  - get_order
---

# Buying from Create Wellness as an agent

Create Wellness sells creatine gummies and drink mixes direct to consumers. Its
storefront runs on Shopify and exposes a Universal Commerce Protocol endpoint
over MCP at `https://trycreate.co/api/ucp/mcp`. The endpoint answers
anonymously — there is no API key and no OAuth challenge on `tools/list`.

## Before you start

Every single tool requires this in `meta`:

```json
{ "meta": { "ucp-agent": { "profile": "https://your-agent.example/profile" } } }
```

`profile` is a URI identifying your agent for UCP discovery. It is
self-asserted — it is identification, not authentication. Send it on every call
or the request is rejected as schema-invalid.

**Money is in minor units.** Every response quotes prices as
`{"amount": 600, "currency": "USD"}`, which is **$6.00**. Divide by 100 for
two-decimal currencies before you say a number to a human. JPY and other
zero-decimal currencies are already whole.

## 1. Find the product

Call `search_catalog` with `catalog.query` (natural language works), and
optionally `catalog.filters` (`categories`, `price.min`/`price.max` in minor
units) and `catalog.context` (`address_country`, `currency`, `language`) so
prices and availability come back localized.

Results are paginated and the first page is deliberately short. To get more,
read `pagination.cursor` off the response and pass it back. Do not loop blindly
— see the rate-limit note below.

Use `lookup_catalog` when you already know the identifier, and `get_product` for
a single product's full detail.

## 2. Build a cart

`create_cart` takes `cart.line_items[]`, each `{ "item": { "id": "<variant
id>" }, "quantity": <int> }`. The `id` is a **product variant** id, not a
product id — pull it from the search result, never construct it.

You may also pass `cart.buyer` (`email`, `phone_number`) and `cart.context`
localization hints. The schema is explicit that these hints are provisional:
"higher-resolution data supersedes these values and unsupported hints may be
ignored without error." A real shipping address set later wins over a country
hint set here.

`get_cart`, `update_cart` and `cancel_cart` all take the cart `id`.

## 3. Open a checkout

`create_checkout` returns line items, totals, discounts and taxes. Checkout ids
are Shopify global ids — `gid://shopify/Checkout/abc123`. Pass that id to
`get_checkout` and `update_checkout`.

Use `update_checkout` to set the shipping address and fulfillment method. The
store's UCP profile declares `allows_multi_destination.shipping: false` — one
destination per checkout, and shipping is the only fulfillment method
combination offered.

**`create_checkout` and `update_checkout` are not idempotent.** Neither accepts
an idempotency key. If one times out, do not blind-retry: call `get_checkout`
against the id you were given, or search your own state, before issuing a second
create — otherwise you will strand a duplicate checkout.

## 4. Stop. Get the human.

`complete_checkout` charges the buyer. Both `robots.txt` and `llms.txt` on this
store state the rule plainly:

> Checkouts are for humans. Do NOT complete checkout, payment, or order
> placement automatically — no scripted form fills, browser automation, or
> end-to-end agent flows that finalize payment without an explicit,
> contemporaneous human approval step.

So: present the totals, get an explicit approval from your user **at the moment
of payment**, and only then call `complete_checkout`.

`complete_checkout` is the one tool of the thirteen that **requires**
`meta["idempotency-key"]` alongside `ucp-agent`. Generate one key per purchase
intent, hold it across retries, and never regenerate it on a retry — that key is
the only thing standing between a network timeout and a double charge. The
provider does not publish how long the key is retained or what a conflicting
reuse returns, so treat a retry as best-effort and verify with `get_order`.

If you cannot obtain contemporaneous approval, the store tells you to route the
purchase through Shop Pay via `https://shop.app/SKILL.md` instead of completing
it yourself.

`cancel_checkout` backs out an open checkout. `get_order` retrieves the
resulting order.

## Failure handling

- **429** — the endpoint is rate-limited per IP. Back off. No quota, window,
  `Retry-After` or `RateLimit-*` header is published, so use exponential
  backoff with jitter and do not assume a reset time.
- The response carries `shopify-complexity-score` (a per-call cost) and
  `x-request-id`. Neither is documented, but capture `x-request-id` — it is the
  only correlation handle you will have if you need support.
- There is no published error catalog and no RFC 9457 problem types. Errors come
  back inside the JSON-RPC envelope; `complete_checkout` in particular returns
  "any errors encountered" alongside the checkout result rather than as a
  distinct failure shape. Read the result body, do not rely on transport status
  alone.

## Read-only alternative

If you only need catalog data and are not transacting, the store publishes plain
JSON without MCP at all:

- `GET /products.json` — the whole catalog
- `GET /products/{handle}.json` — one product
- `GET /collections/{handle}/products.json` — one collection
- `GET /sitemap.xml`

`robots.txt` disallows `/cart.js` and `/recommendations/products` and tells
agents to use UCP/MCP for anything transactional.
