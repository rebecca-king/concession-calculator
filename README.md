# Concession Calculator — Gorgias Sidebar Widget

A CX agent tool that calculates, generates, and logs customer concessions directly inside the Gorgias support interface. Built as a working prototype with simulated data modeled after real e-commerce operations workflows.

---

## What It Does

When a customer contacts support about a poor experience, CX agents typically have to:

1. Look up the customer's order history manually
2. Decide what concession (if any) is appropriate
3. Generate a discount code in Shopify
4. Copy it into a reply macro
5. Log the concession somewhere for future reference

This tool collapses all five steps into a single sidebar widget. The agent loads a customer by email, sees their full context, gets a policy-driven recommendation, generates a code in one click, reviews the pre-filled macro, and approves — everything is logged automatically.

---

## The Business Problem

Inconsistent concessions erode margin and create fairness issues. Without a structured system:

- Different agents issue different discount levels for identical situations
- High-claim customers receive repeated concessions with no visibility
- There's no audit trail connecting a concession to a ticket, order, or agent
- New agents have no guidance on what's appropriate

This tool enforces a single policy, surfaces risk signals, and creates a complete record on every concession issued.

---

## Concession Tier Logic

Tiers are calculated from two inputs: the **order value** of the ticket's associated order, and the customer's **eligible poor experience count**.

### Base tier (from order value)

| Order Value | Base Tier |
|---|---|
| Under $20 | No concession |
| $20 – $75 | Free shipping code |
| $75 – $250 | $5 discount code |
| Over $250 | $10 discount code |

### Bumping

Each **additional eligible instance beyond the first** bumps the tier up one level.

The tier ladder is: `No concession → Free shipping → $5 off → $10 off`

**Example:** A customer with 2 eligible instances and a $60 order gets free shipping (base) bumped to $5 off (one additional instance).

---

## Poor Experience Tracking

Customer poor experiences are stored in a Shopify metafield (`poor_experience_summary`) with a count breakdown by type. Not all types are created equal:

| Type | Triggers a concession? | Rule |
|---|---|---|
| `canceled_order` | Yes | Each occurrence = 1 eligible instance |
| `wrong_item` | Yes, after threshold | Every **3 occurrences** = 1 eligible instance |
| `poor_store_experience` | Never directly | Increments the counter, but cannot trigger a concession on its own |

This prevents single low-severity complaints from generating automatic discounts while still building a record over time.

### Metafield structure

```yaml
poor_experience_summary:
  total: 4
  last_occurrence: 2026-02-03
  breakdown:
    canceled_order:
      count: 1
      last_occurrence: 2026-01-15
    wrong_item:
      count: 3
      last_occurrence: 2026-02-03
    poor_store_experience:
      count: 0
      last_occurrence: null
```

---

## Gorgias Widget Workflow

The widget runs as a Gorgias sidebar app. It auto-loads the customer associated with the open ticket.

```
1. Load       →  Customer profile, LTV, order history, and poor experience
                 summary pulled from Shopify metafields.

2. Recommend  →  Tier calculated automatically from order value + eligible
                 instance count. Recent concession flag shown if a code was
                 issued within the last 30 days (warning only — does not block).

3. Generate   →  Agent clicks "Generate Discount Code." A unique code is
                 created via the Shopify Discounts API and inserted into the
                 appropriate macro template as [DISCOUNT CODE].

4. Approve    →  Agent reviews the pre-filled macro and order note preview.
                 Optional free-text note field for context. Agent clicks
                 "Approve & Log."

5. Log        →  Three writes happen simultaneously:
                   - Customer metafield updated (concession_issued, code, tier, ticket ID)
                   - Order note appended (never overwritten)
                   - recent_concession flag set (suppresses auto-suggestions for 30 days)
                   - Macro inserted into the Gorgias reply editor
```

### Order note format

```
[CONCESSION ISSUED 2026-04-09] Type: canceled_order | Tier: $5 discount | Code: SAVE5SARA3K2X | Order: #4821 ($134.00)
```

### Macro mapping

| Tier | Macro |
|---|---|
| No concession | No macro generated |
| Free shipping | `free_ship_apology` |
| $5 discount | `discount_5_apology` |
| $10 discount | `discount_10_apology` |

---

## Demo Profiles

The prototype includes 5 simulated customer profiles that cover the key scenarios a CX team encounters:

| Profile | Scenario | Outcome |
|---|---|---|
| Sarah Chen | VIP, first canceled order, $134 order | $5 discount |
| Marcus Powell | 3× wrong_item (threshold hit) + 1 canceled_order, $58 order | Free shipping → bumped to $5 |
| Ava Morrison | New customer, first canceled order, $54 order | Free shipping |
| Diana Reyes | VIP, 2 canceled orders, $290 order, concession issued 7 days ago | $10 discount + recent concession warning |
| Kevin Tran | 3× poor store experience + 1× wrong_item (below threshold) | No concession triggered |

---

## Running It

This is a single-file prototype — no build step, no dependencies.

```bash
open concession-widget.html
```

Or just double-click the file. Use the demo chips at the top of the widget to load each profile, or type any of the emails from the table above into the lookup bar.

---

## Production Architecture

This prototype uses simulated in-memory data. In production, the widget would:

- **Load customer data** via the [Shopify Customer API](https://shopify.dev/docs/api/admin-rest/2024-01/resources/customer) and a custom metafield namespace (`cx.poor_experience_summary`)
- **Generate discount codes** via the [Shopify Discount Codes API](https://shopify.dev/docs/api/admin-rest/2024-01/resources/discountcode) with expiry and usage limits set automatically
- **Append order notes** via the [Shopify Orders API](https://shopify.dev/docs/api/admin-rest/2024-01/resources/order) (append-only, preserving history)
- **Insert macros** via the [Gorgias HTTP Integration](https://developers.gorgias.com/) or Gorgias Widget SDK
- **Authenticate** via OAuth tokens stored in a lightweight backend proxy (keeping Shopify secrets off the client)
- **Track concession_recently_issued** as a separate metafield with a TTL-equivalent check on load

The widget UI and business logic in this prototype are designed to match that production shape as closely as possible, so the primary integration work is replacing the simulated data layer with live API calls.

---

## Tech Stack

- Vanilla HTML, CSS, JavaScript — no framework, no build tooling
- Designed to embed as a Gorgias sidebar app (340px fixed width)
- All state is in-memory; persistence is simulated via console-style session log

---

*Prototype built with simulated data. Not connected to live Shopify or Gorgias environments.*
