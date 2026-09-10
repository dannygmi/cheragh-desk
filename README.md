# Cheragh Desk

**https://dannygmi.github.io/cheragh-desk/**

One page for orders, costs and margins. It reads and writes the Supabase project
`CHERAGH` (`tbmtznnpgibcwibxdvdq`) and has no server of its own, so it does not
care whether the Mac is awake. That is the whole point: the ten desks on ports
8787 to 8796 all died when the laptop slept.

Source of the page: `dash/index.html` here, mirrored to the public repo
[`dannygmi/cheragh-desk`](https://github.com/dannygmi/cheragh-desk), which is what
GitHub Pages serves.

---

## Setup: all four done, 10 September 2026

| | |
|---|---|
| Login | Danny's Supabase auth user exists. Nobody else has the password, including me. |
| Ads feed | Script `Dashboard` on 323-939-6319, authorised, **Hourly**, enabled. |
| Claude Code | `.mcp.json` at the repo root, project scope, read only. |
| ChatGPT | Custom plugin `Cheragh Desk`, developer mode, OAuth connected, `execute_sql` discovered. |

### If any of it has to be rebuilt

**Login.** Supabase dashboard &rarr; Authentication &rarr; Users &rarr; Add user &rarr;
*Create new user*, tick **Auto Confirm User**.

**Ads feed.** Google Ads &rarr; Tools &rarr; Bulk actions &rarr; Scripts &rarr; **+** &rarr;
paste `dash/ads-script.js` &rarr; **Authorise** &rarr; **Run**, then set Frequency to
**Hourly**. Google hosts the scheduler, so it keeps reporting while the Mac sleeps.
The key inside it can call exactly one database function and read nothing at all,
so it is safe sitting in the Ads UI in plaintext. **Never put the service key there.**

Hourly rather than daily because the query window includes today. GAQL's
`LAST_14_DAYS` ends *yesterday*, which left the desk's Today tab permanently
blank while the campaign was spending fifty pounds a day. Each run rewrites the
same thirty-odd rows, so twenty-four runs cost nothing.

Note: Google Ads developer tokens were sunset on 9 September 2026. This route
needs no token, no OAuth client and no access-level application.

**Claude Code.**

    claude mcp add --scope project --transport http supabase \
      "https://mcp.supabase.com/mcp?project_ref=tbmtznnpgibcwibxdvdq&read_only=true&features=database,docs"

**ChatGPT.** Settings &rarr; Security and login &rarr; turn on **Developer mode**
(it is off by default and marked elevated risk). Then Plugins &rarr; **+** &rarr;
name it, paste the same URL by hand, tick the risk box, Create, and sign in to
Supabase when it asks.

**Do not install the Supabase entry from the plugin directory.** It carries no
parameters and declares read *and* write across the whole account. The
hand-entered URL is scoped to this project and read only.

Two things to know rather than assume:

- **Developer-mode connectors are web only.** No mobile surface is documented, so
  a question from the phone cannot reach the database through ChatGPT. There is
  no chat box in this page either: an earlier version of this file claimed there
  was, and that was wrong. Danny chose on 2026-09-10 not to build one rather
  than pay for an API key, so the phone shows the desk's numbers and the asking
  happens on a laptop.
- **`read_only` is not a wall around the data.** It downgrades the connection to a
  read-only Postgres user; it does not confine it to a schema. ChatGPT can read
  `orders.email` and `shipping_address`. That is deliberate, because the point is
  for it to check an order and draft a reply, which needs exactly that.

---

## What is in the database

| | |
|---|---|
| `order_financials` | Revenue, real fees, real cost of goods, profit as a generated column |
| `order_transaction_fees` | Every fee Shopify actually charged, with its rate name |
| `supplier_orders` | What you paid the factory, in the currency you paid it |
| `expenses` / `expense_entries` | The recurring run rate, and dated cash that left |
| `expense_categories` | Yours to add to |
| `variant_economics` | 2,890 rows, one per offer, with the air-freight flag |
| `ad_days` | One row per day per campaign, written hourly by the Ads Script, with status, type, bid strategy and budget |
| `shop_days` | Storefront sessions per day from Shopify, the denominator for conversion |
| `feed_runs` | One row per feed, stamped every run. Freshness reads this |
| `agent_notes` | The shared log. Claude, ChatGPT and you all write here |
| `store_settings` | VAT position, lead times, refund deadlines, measured fee rates |

Views: `order_pnl`, `cost_summary`, `cost_ledger`, `ad_campaigns`,
`expense_totals`, `expense_by_category`, `expense_run_rate`, `data_freshness`,
`storefront_summary`, `product_margins`, plus an `ai` schema of read surfaces
for the models.

`cost_summary` is one row and it is the whole cost of the business, drawn from
the three places cost actually lives so nothing is retyped: overheads from
`expense_entries`, advertising from `ad_days`, per-order cost from `order_pnl`.
`ad_campaigns` is Google's own campaign table plus the two figures Google will
never compute for this business: the break-even conversion rate at each
campaign's own cost per click, and the number of clicks before zero conversions
starts to mean something.

### Talking to it in plain English

`ai.brief` is the thing that makes that work. It holds the operating facts that
change how a number should be read, and every table carries a comment, because
the Supabase MCP surfaces comments when it lists tables. That is the only place
a warning reaches the model *before* it writes the SQL.

Verified 2026-09-10: "How's the shop doing?" with no schema named returned one
customer order, GBP 89.12 before ads, GBP 261.38 spent, 0.26% conversion against
sessions rather than clicks, and flagged both the unresolved VAT and the fact
that Google is claiming zero conversions. Every one of those is a trap written
down in `ai.brief`.

If you add a table, comment it. An uncommented table is one the models will
guess about.

### Two rules the schema enforces rather than trusts

**Profit is labelled, never guessed.** `order_financials.cogs_source` says whether
the cost of goods is the pricing model's estimate or a real number you entered.
The page shows that label next to every profit figure. Order #1005 modelled at
&pound;99.90 and settled at &pound;89.13, so an unlabelled figure is a rumour.

**Spent and burn are never summed.** `expense_entries` is cash that left on a date.
`expenses` is a run rate. Adding them gives a number that is true of nothing.

**Not every cost is Cheragh's, and funding a credit is not spending it.** Added
2026-09-10 after ChatGPT read the inbox and produced a verified register.
`allocation` is the share that belongs to the store, 0 to 1; Canva, the Xolo
autonomo fee and one Apple receipt are 0 and are reported as unallocated rather
than charged in full or quietly dropped. `recognized` is false for the
&pound;472.68 of OpenAI and Anthropic balance that has been funded but not proven
consumed: cash out today, expense on the day it is used. Charging that to the
profit and loss would have made the store look &pound;472.68 worse than it is.

**A per-order cost is not an overhead.** `scope` separates them. The &pound;75.34
paid to Zhi Yi for #1005 was in `expense_entries` *and* in `supplier_orders`,
which is where `order_pnl` gets it, so the Costs tab and the Orders tab were
double-counting the same payment.

---

## Keeping it fed

    python3 processor/orders_sync.py      # Shopify orders -> Supabase, every 5 min via LaunchAgent
    python3 processor/money_sync.py --all # fees, financials, fulfilment reconcile, variant economics
    python3 processor/shop_sync.py        # Shopify sessions and orders per day
    python3 processor/cost_import.py      # the verified cost audit workbook -> expense_entries + expenses

`cost_import.py` is idempotent: every row carries `source = 'import'` and a
stable `external_ref` under a unique index, so a second run updates rather than
duplicates. It deliberately does **not** import three rows the workbook
contains, because the desk already reads them from a live feed and importing
them would double the count: Google Ads spend (`ad_days`), the #1005 supplier
payment (`supplier_orders`) and the #1005 payment fees
(`order_transaction_fees`). FX is back-computed from each invoice's own GBP
total rather than a spot rate, so the figures reproduce the audit to the penny:
POKY billed USD 3.54 and the invoice says GBP 2.62, where a spot rate would have
said 2.80 and been wrong about a number somebody can check.

`money_sync.py --orders` reads the real processing and FX fees off the Shopify
transactions API and reconciles fulfilment state against Shopify, which
`orders_sync.py` never did: it seeded one row per order and nothing advanced it,
so every row read `new` including one Shopify already had in progress.

`money_sync.py --products` republishes all 2,890 variant rows after an
`engine.py` run. It is a wholesale replace, never a merge.

---

## Security, and how it was checked

The page ships the project URL and the **publishable** key, as every static front
end must. Neither grants anything. Row level security decides access.

Verified 2026-09-10 by calling the API with only that key and no session:

- Every base table returns empty or refuses.
- Every view refuses. Views were the hole: a Postgres view runs with its owner's
  privileges, so `order_pnl` and `fulfilment_queue` handed back customer names and
  email addresses until `0007_view_security.sql` set `security_invoker` on them.
  `fulfilment_queue` had been open since 22 August.
- The ads key can call one function and read nothing, including the table it writes.

If you add a view, set `security_invoker = on` on it in the same migration. It is
not the default and the failure is silent.

---

## Deploying a change

    cp dash/index.html <the cheragh-desk clone>/index.html
    cd <the cheragh-desk clone> && git commit -am "..." && git push

Pages rebuilds in about a minute. The page sets `Cache-Control` off its own
`<meta>` and GitHub serves it fresh, but hard-refresh if in doubt.

Local, including from the phone on the same wifi:

    python3 dash/serve.py
