---
name: esimker
description: Buy prepaid travel data eSIMs from esimker (esimker.com) through its MCP server or plain HTTPS API - search destinations, compare plans, create a checkout link for a person to pay in crypto, or buy autonomously from a prepaid wallet funded in crypto, then read the activation code. Use when someone needs mobile data abroad (a trip, "internet in Turkey", an eSIM for a country or region), asks to top up or check an eSIM bought at esimker, or an agent has to purchase connectivity with nobody at the checkout.
license: MIT
metadata:
  author: esimker
  homepage: https://esimker.com/agents
  docs: https://esimker.com/llms.txt
  repository: https://github.com/Esimker/mcp
  version: "1.0.0"
---

# esimker

esimker sells prepaid travel **data** eSIMs: data only, no phone numbers, no
SMS, no voice. 38 destinations (countries and a few regions), prices in USD
from $1.00, no accounts and no KYC. An order is addressed by a secret token,
a wallet by a secret code; the eSIM comes back as an LPA activation code to
render as a QR. Payment today is crypto (BTC, ETH, TRX, USDT); cards are not
live yet.

## Connect

Prefer the MCP server; fall back to the HTTPS API when the host cannot speak
MCP. Both go through the same rules, limits and prices.

```
# Claude Code
claude mcp add --transport http esimker https://esimker.com/api/mcp
claude mcp add --transport http esimker https://esimker.com/api/mcp --header "Authorization: Bearer <wallet code>"

# Cursor, VS Code, Windsurf and other JSON configs
{ "mcpServers": { "esimker": { "url": "https://esimker.com/api/mcp",
                               "headers": { "Authorization": "Bearer <wallet code>" } } } }
```

The catalogue, the checkout flow and `create_wallet` need no credential. The
four wallet tools read the wallet code from `Authorization: Bearer <code>`.
The custom connectors of Claude.ai and Claude Desktop cannot send that header:
use them for the checkout flow, and a client that sends headers (Claude Code,
Cursor, VS Code, an agent built on an SDK) for the wallet.

HTTPS API: `https://esimker.com/api/openapi.json` (Swagger at `/api/docs`);
the wallet code travels as the `X-Wallet-Code` header there.

## Pick a path

**A person is present and pays themselves** (no credential needed):
`search_destinations` → `get_plans` → confirm the plan and the price with the
person → `create_checkout` → hand `checkout_url` to the person → poll
`get_order(token)` every ~10 s until `status` is `ready` → show the person
`order_url` (the order page with the QR code and the install steps).

**Nobody is at the checkout** (the prepaid wallet):
`create_wallet(accept_terms=true)` once and save its `code` where credentials
live → send it as `Authorization: Bearer <code>` on every wallet call →
`create_deposit(amount_usd, currency)` and pay exactly `pay.amount` to
`pay.address` on `pay.network` within about 15 minutes → poll
`get_deposit(token)` every ~30 s until `paid` → `purchase_esim(plan_id,
client_ref)` → `get_order(token)` until `ready`.

The first deposit is from $10, later ones from $1, at most $1000 per deposit;
crypto deposits earn a bonus from $25 and every purchase returns 3% cashback.
The exact numbers are in `get_wallet` and in the `create_deposit` description
- read them from there, not from memory.

## Rules

- Confirm the destination, the plan and the price with the person before
  `create_checkout` or `purchase_esim`. An installed eSIM cannot be refunded;
  a wallet balance is spent on eSIMs and never paid back out.
- The wallet code, the order token and the activation code are secrets. Keep
  the code where you keep credentials, show `order_url` to the buyer only,
  never log or repeat them elsewhere, never put them in a URL you share. A
  lost wallet code cannot be recovered; there is no identity to prove.
- Reuse `client_ref` when retrying `purchase_esim` - the same value returns
  the same order without a second charge. Never create a second checkout
  while one is open.
- `daily_unlimited` plans are priced per day: pass `period_num` (1-365,
  default 7); the amount is price × days.
- Regional plans list the countries they cover in `location_codes`; check it
  when the person visits several countries.
- A 403 means the caller's jurisdiction is embargoed: tell the person, do not
  retry. A 402 on a wallet purchase is a short balance: `create_deposit`
  first. A 429 is a limit: 60 requests a minute per IP on the endpoint, 10
  orders and 5 wallet operations a minute per IP, and one wallet pays for at
  most 50 orders in 24 hours - wait for `Retry-After`, do not hammer.

## Tools

Without a credential:

- `search_destinations(query)` - destinations by name in any of 17 languages, the slug or the ISO code
- `get_plans(slug)` - the plans of one destination, with the `plan_id`
- `get_payment_methods()` - which of crypto, card and wallet are open
- `create_checkout(plan_id, …)` - an order for a person to pay: `checkout_url` and `order_url`
- `get_order(token)` - the status and, when ready, the activation code
- `list_topups(iccid)` - top-ups that fit an eSIM esimker issued
- `get_usage(iccid)` - remaining data
- `create_wallet(accept_terms)` - mint a wallet: the code and the phrase, shown once

With the wallet code:

- `get_wallet()` - the balance, the rules, the ledger and the order links
- `create_deposit(amount_usd, currency)` - a top-up with the coin address and the exact amount
- `get_deposit(token)` - pending → paid, poll it after sending
- `purchase_esim(plan_id, client_ref, …)` - charge the wallet and issue, idempotent by `client_ref`

## Source of truth

Prices, coverage and the money rules change; the API is the source of truth.
Read them at run time - `search_destinations`, `get_plans`, `get_wallet` -
and never quote a price you have not just fetched. Human-readable guide:
https://esimker.com/agents. Support: info@esimker.com (send the order token
or the wallet's `ref`, never the wallet code).
