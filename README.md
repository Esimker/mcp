# esimker MCP server

[esimker](https://esimker.com) sells prepaid travel **data** eSIMs for 38
destinations - countries and a few regions - without accounts: an order lives
behind a secret link, the eSIM arrives as a QR code / LPA activation code
within about a minute of payment. Prices in USD from $1.00. Data only: no
phone numbers, no SMS, no voice. Payment today is crypto (BTC, ETH, TRX,
USDT); cards are not live yet.

This repository holds what an AI agent needs to use the store: the
[skill](skills/esimker/SKILL.md) (`npx skills add Esimker/mcp`)
and the listing metadata of the MCP server. The server itself runs at
esimker.com, over the same API the storefront uses.

## The MCP server

- URL: `https://esimker.com/api/mcp` - Streamable HTTP, stateless, no OAuth.
- Credential: none for the catalogue, the checkout flow and minting a wallet;
  the four wallet tools read the wallet code from `Authorization: Bearer <code>`.
- Guide: https://esimker.com/agents · `llms.txt`: https://esimker.com/llms.txt ·
  OpenAPI: https://esimker.com/api/openapi.json (Swagger: https://esimker.com/api/docs)

### Connect

```
# Claude Code
claude mcp add --transport http esimker https://esimker.com/api/mcp
claude mcp add --transport http esimker https://esimker.com/api/mcp --header "Authorization: Bearer <wallet code>"

# Cursor, VS Code, Windsurf and other JSON configs
{ "mcpServers": { "esimker": { "url": "https://esimker.com/api/mcp",
                               "headers": { "Authorization": "Bearer <wallet code>" } } } }

# Claude Desktop and claude.ai: Settings → Connectors → Add custom connector → the URL above
```

A custom connector of Claude.ai / Claude Desktop sends no headers: the
catalogue, the checkout flow and `create_wallet` work there; the wallet tools
need a client that can send the `Authorization` header (Claude Code, Cursor,
VS Code, agents built on the SDKs).

### Install the skill

```
npx skills add Esimker/mcp
```

The skill tells the agent how to connect, which path to take (a person pays
at a checkout link, or the agent buys from a prepaid wallet) and the rules:
confirm the price with the person, keep the wallet code and the order token
private, reuse `client_ref` on a retry.

## Tools

Without a credential:

| Tool | What it does |
| --- | --- |
| `search_destinations(query)` | destinations by name in any of 17 languages, the slug or the ISO code |
| `get_plans(slug)` | the plans of one destination, with the `plan_id` |
| `get_payment_methods()` | which of crypto, card and wallet are open right now |
| `create_checkout(plan_id, …)` | an order for a person to pay: `checkout_url` and `order_url` |
| `get_order(token)` | the status and, when ready, the activation code |
| `list_topups(iccid)` | top-ups that fit an eSIM esimker issued |
| `get_usage(iccid)` | remaining data |
| `create_wallet(accept_terms)` | mint a prepaid wallet: the code and the 12-word phrase, shown once |

With the wallet code (`Authorization: Bearer <code>`):

| Tool | What it does |
| --- | --- |
| `get_wallet()` | the balance, the deposit rules, the ledger and the order links |
| `create_deposit(amount_usd, currency)` | a top-up: the coin address, the exact amount, the address's expiry |
| `get_deposit(token)` | pending → paid; poll it after sending |
| `purchase_esim(plan_id, client_ref, …)` | charge the wallet and issue at once; idempotent by `client_ref` |

## Two ways to buy

**For a person who pays** - no credential:
`search_destinations` → `get_plans` → `create_checkout` → hand `checkout_url`
to the person → `get_order(token)` until `status` is `ready` → show the person
`order_url`.

**Autonomously, from the prepaid wallet:**

1. `create_wallet(accept_terms=true)` - returns the secret once: `code`
   (32 hex characters) and the same secret as a 12-word `phrase`. Save the
   code; there is no recovery and no identity to prove.
2. Send the code as `Authorization: Bearer <code>` on every wallet call.
3. `create_deposit(amount_usd, currency)` - `pay` in the answer is the coin
   address, the exact amount (network fee included) and the expiry, about 15
   minutes. Send exactly that amount; `checkout_url` is the hosted page for a
   person.
4. `get_deposit(token)` until `paid`; the balance moves when the network
   confirms.
5. `purchase_esim(plan_id, client_ref)` - the order comes back paid; a retry
   with the same `client_ref` returns the same order without a second charge.
6. `get_order(token)` until `ready`; `esim.activation_code` is the LPA string.

Money rules (read the live values from `get_wallet`): the first deposit from
$10, later ones from $1, up to $1000 per deposit; crypto deposits earn +2%
from $25, +3% from $50, +5% from $100 (capped at $25); every purchase returns
3% cashback. A wallet balance is spent on eSIMs and never paid back out.

## Rules and limits

- Confirm the destination, the plan and the price with the person before
  `create_checkout` or `purchase_esim`. An eSIM that has not been installed
  can be refunded in full; an installed or used one is consumed.
- The wallet code, the order token and the activation code are secrets: keep
  the code where you keep credentials, show `order_url` to the buyer only,
  never log them. A lost wallet code cannot be recovered.
- `daily_unlimited` plans are priced per day: pass `period_num` (1-365,
  default 7), the amount is price × days. Regional plans list the countries
  they cover in `location_codes`.
- Limits: 60 requests a minute per IP on the endpoint (429 with
  `Retry-After`); 10 orders a minute per IP; 5 wallet creations or deposits a
  minute per IP; one wallet pays for at most 50 orders in 24 hours. A 403
  means the caller's jurisdiction is embargoed - tell the person, do not
  retry. A 402 is a short balance.
- Acceptable use: buying through the API, the MCP server or a wallet for your
  own use, or for the person you act for, is allowed; automated bulk
  purchasing for resale needs the reseller API
  (https://esimker.com/agents#wallet).

## Support

info@esimker.com - send the order token or the wallet's `ref` (a public
handle like `W-3F9A2C1D`), never the wallet code. The fine print:
[terms](https://esimker.com/terms), [refund policy](https://esimker.com/refund),
[acceptable use](https://esimker.com/aup).

## License

MIT - for the contents of this repository (the skill and the metadata). The
esimker service itself is governed by its terms.
