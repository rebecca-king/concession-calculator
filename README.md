# Concession Calculator — Gorgias Sidebar Widget

A CX operator tool that calculates, generates, and logs customer concessions directly inside the Gorgias support interface. Built as a working prototype with simulated data modeled after real e-commerce operations workflows.

---

## What It Does

When a customer contacts support about a poor experience, CX operators typically have to:

1. Review the customer's order history manually
2. Decide what concession (if any) is appropriate
3. Generate a discount code in Shopify
4. Copy it into a reply macro
5. Log the concession somewhere for future reference (if it happens at all, it's inconsistent and hard to track across systems)

This tool collapses all five steps into a single sidebar widget. The operator loads a customer by email, sees their full context, gets a policy-driven recommendation, generates a code in one click, reviews the pre-filled macro, and approves — everything is logged automatically.

---

## The Business Problem

Inconsistent concessions erode margin and create fairness issues. Without a structured system:

- Different operators issue different discount levels for identical situations
- High-claim customers receive repeated concessions with no visibility
- There's no audit trail connecting a concession to a ticket, order, or operator
- New operators have no guidance on what's appropriate

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

Customer poor experiences are stored in a Shopify metafield (`poor_experience_summary`) with a count breakdown by type. Not all types are created equal.

> **Note:** Refunds are handled separately through the standard order management process where applicable. The concessions tracked here — discount codes, free shipping — are issued *in addition to* any refund, as a makegood for the overall experience.

| Type | Triggers a concession? | Rule |
|---|---|---|
| `canceled_order` | Yes | Each occurrence = 1 eligible instance |
| `canceled_item` | Yes | Each occurrence = 1 eligible instance |
| `missing_order` | Yes, manual only | Each cleared occurrence = 1 eligible instance — requires fraud review first |
| `missing_item` | Yes, manual only | Each cleared occurrence = 1 eligible instance — requires fraud review first |
| `wrong_item` | Yes, after threshold | Every **3 occurrences** = 1 eligible instance |
| `poor_store_experience` | Never directly | Increments the counter, but cannot trigger a concession on its own |

This prevents single low-severity complaints from generating automatic discounts while still building a record over time.

### Missing order / missing item — fraud review gate

`missing_order` and `missing_item` follow a different process from other types. Because these cases carry fraud risk, the widget **blocks the concession flow** until an operator manually confirms that fraud review has been completed for the ticket.

Once confirmed, the operator clicks "Mark fraud review cleared & log" — this increments the count in the breakdown, recalculates the recommended tier, and unlocks the normal generate → approve flow. The clearing action is logged in the session log.

### Metafield structure

```yaml
poor_experience_summary:
  total: 4
  last_occurrence: 2026-02-03
  breakdown:
    canceled_order:
      count: 1
      last_occurrence: 2026-01-15
    canceled_item:
      count: 0
      last_occurrence: null
    missing_order:
      count: 0
      last_occurrence: null
    missing_item:
      count: 0
      last_occurrence: null
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

3. Generate   →  Operator clicks "Generate Discount Code." A unique code is
                 created via the Shopify Discounts API and inserted into the
                 appropriate macro template as [DISCOUNT CODE].

4. Approve    →  Operator reviews the pre-filled macro and order note preview.
                 Optional free-text note field for context. Operator clicks
                 "Approve & Log."

5. Log        →  Three writes happen simultaneously:
                   - Customer metafield updated (concession_issued, code, tier, ticket ID)
                   - Order note appended (never overwritten)
                   - recent_concession flag set (suppresses auto-suggestions for 30 days)
                   - Macro inserted into the Gorgias reply editor
```

### Order note format

```
[CONCESSION ISSUED 2026-04-09] Ticket: TKT-88234 | Type: canceled_order | Tier: $5 discount | Code: SAVE5SARA3K2X | Order: #4821 ($134.00)
```

### Macro mapping

Macros are tied to the **poor experience type**, not the concession tier. This ensures the reply language matches what the customer actually experienced.

| Poor Experience Type | Macro |
|---|---|
| `canceled_order` | `canceled_order` |
| `canceled_item` | `canceled_item` |
| `wrong_item` | `wrong_item` |

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
| Jordan Kim | Regular, missing order — carrier marked delivered, customer denies receipt | Fraud review gate → free shipping after clearance |

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

---

## Future Improvements

### Issue count decay

Currently, poor experience counts never decay — an incident from two years ago carries the same weight as one from last week. The recommended approach is a **rolling 12-month window**: only occurrences within the past 12 months count toward eligible instances. This is simple to explain to operators, straightforward to implement (filter breakdown entries by `last_occurrence` date at calculation time), and prevents a customer's historical record from permanently inflating their tier long after the relationship has recovered.

A shorter window (e.g. 6 months) could be considered for lower-severity types like `wrong_item`, while keeping a longer window for higher-severity types like `missing_order`. That said, starting with a single consistent window across all types is easier to communicate and audit.

### Segment-aware tier logic

Currently, customer segment (VIP / Regular / New) has no effect on the concession calculation — tier is determined solely by order value and eligible instance count. A future improvement would allow segment to influence the base tier or bump behavior. For example, a VIP's first issue might warrant starting one tier higher, or a New customer's first issue might be capped to avoid over-investing before a relationship is established. Any segment-based adjustments should be defined in the rulebook before being implemented here.

---

*Prototype built with simulated data. Not connected to live Shopify or Gorgias environments.*
