# Cost & Pricing Model

> How the `cost`, `price`, `memberPrice`, and `useStaticPrices` fields actually behave —
> including the dynamic-pricing model that most established ST tenants run on.

## The three pricing models

ST supports three pricing models, and they behave differently per item type:

### 1. Dynamic pricing (services, default)

`useStaticPrices` is `false` or `null`. Customer price is **computed at invoice time** from cost + markup rules (configured in ST Pricing Settings).

- `price` on the service record is typically `0` and **not the billed amount**
- `cost` matters for margin reporting and flow-through to markup calc — but ST silently drops `cost` on POST and PATCH (see [`silent-fail-catalog.md`](silent-fail-catalog.md))
- Cost flows in through linked materials via `serviceMaterials` — the service's effective cost = sum of linked material costs

This is the default for most established ST tenants. If you see `price = 0` and `useStaticPrices = null/false` everywhere, that's dynamic pricing, not a data error.

### 2. Static pricing (services, opt-in)

`useStaticPrices` is `true`. `price` is the canonical customer-billed amount.

- `price` matters and sticks (you can set it via API)
- `cost` still silently drops on POST/PATCH; UI is the only path to set it
- **Cannot flip a service from dynamic to static via API post-create** — `useStaticPrices` is silently dropped on PATCH after the service exists. Must be set in the UI, or set at create time (and even then, behavior is unreliable per 2026-05-08 investigation).

### 3. Equipment (always static)

Equipment doesn't have a `useStaticPrices` field — it's effectively always static. Both `price` and `cost` are settable via API and matter:

- `price` is the customer-billed amount on the invoice
- `cost` is for margin reporting and feeds your COGS line on the GL

### Materials

Materials are typically static — both `cost` and `price` matter and stick on POST/PATCH. There's no `useStaticPrices` field on materials.

---

## The `useStaticPrices` tri-state

This field has three meaningful values: `true`, `false`, `null`. They look similar but behave differently.

| Value | Meaning | Common in |
|---|---|---|
| `null` | Field has never been explicitly set. Behavior defaults to dynamic. | Long-lived ST tenants (most rows) |
| `false` | Explicitly dynamic — set at some point in the past. | Tenants that ran a normalization pass |
| `true` | Explicitly static — `price` is canonical, billed directly. | Diagnostic fees, one-off services, items that bypass markup |

In a 317-row audit of active plumbing services (real production tenant, 2026-04-28), every row had `useStaticPrices = null`. Don't write code that branches on `=== false` and silently misses the null case.

### Inferring pricing model from price/cost shape

Since `useStaticPrices = null` is the most common state, behavior is usually inferred from the (price, cost) shape:

| `price` | `cost` | Inferred model |
|---|---|---|
| `> 0` | `= 0` | Static catalog price. `price` is the billed amount; cost not tracked. |
| `= 0` | `> 0` | Dynamic. Cost is the input; price computed at invoice. |
| `> 0` | `> 0` | Hybrid — static price billed directly, cost tracked for margin reporting. |
| `= 0` | `= 0` | Placeholder / incomplete. **Real risk** — service won't price correctly when invoiced. |

`(0, 0)` rows are the "data quality" hotspot you want to surface in audits. They're not always bugs (some are intentional templates) but they're the population most likely to bill at $0 by accident.

---

## Decision tree: setting prices on a new service

```
Need a static catalog price (diagnostic fee, one-off labor)?
├─ YES → Create in UI (cannot reliably flip useStaticPrices via API)
│        OR: design the service to live alongside a "fee" pricebook category
│        that's audited periodically for $0 entries
│
└─ NO  → Use dynamic pricing (default)
         ├─ POST without `useStaticPrices`, `price`, `cost`
         ├─ Link materials via serviceMaterials → that's where cost flows
         └─ ST computes customer price at invoice time
```

## Decision tree: setting prices on a new material

```
Standard material (vendor-sourced, resold at margin)?
├─ YES → POST with cost (vendor cost) and price (sell price).
│        Both stick. `primaryVendor.cost` should match `cost`.
│
└─ Markup-only material (no fixed sell price)?
    ├─ Rare on materials — usually you have a target sell price.
    └─ If truly markup-only, leave price=0 and let your invoicing logic compute.
```

## Decision tree: setting prices on new equipment

```
Equipment is always static — set both cost and price on POST.
├─ cost = your landed cost from primary vendor
├─ price = customer-billed amount
└─ Both stick via API. Don't skip cost — equipment cost feeds margin reports
   and is one of the most-watched numbers in field-service ops.
```

---

## Detecting cost/price problems at scale

Useful queries against your D1 mirror:

### Equipment with cost but no price (margin exposure)

```sql
SELECT code, name, cost, price, category_name
FROM pb_equipment
WHERE cost > 0 AND price = 0 AND active = 1
ORDER BY cost DESC;
```

These are the highest-risk rows — you're paying for the item but not billing for it. In a real production audit, 87 rows like this represented ~$146K of unbilled cost exposure.

### Services priced statically at $0 (billing risk)

```sql
SELECT id, code, name, price, member_price, category_name
FROM pb_services
WHERE active = 1 AND use_static_prices = 1 AND price = 0;
```

Static-priced services with `price = 0` will bill at $0 on the invoice. Usually a config error — someone flipped `useStaticPrices` but didn't set the price.

### Materials with cost but no vendor

```sql
SELECT code, name, cost, price
FROM pb_materials
WHERE active = 1 AND cost > 0 AND (primary_vendor_id IS NULL OR primary_vendor_id = 0)
ORDER BY cost DESC LIMIT 100;
```

Cost without a vendor link breaks auto-purchasing and obscures vendor performance reports.

### Services with no materials linked (potentially mis-modeled)

```sql
SELECT id, code, name, price, cost
FROM pb_services
WHERE active = 1
  AND is_labor = 0                                        -- exclude pure-labor services
  AND (materials_json IS NULL OR materials_json = '[]')
ORDER BY name LIMIT 50;
```

Non-labor services without `serviceMaterials` won't get cost flow-through. Under dynamic pricing they'll price at the minimum markup (or fail invoice validation).

---

## The dynamic pricing trap

The most common confusion point for new ST integrators:

> "Why is `price = 0` on all my services? Is the sync broken?"

It's not. Dynamic pricing means ST stores cost-side data on the service and computes the customer price at invoice generation time. The `price` field on the service is intentionally `0` because there is no static price to bill.

The **right** question to ask of a dynamic-priced service is "what's the cost path?" — usually the answer is "through linked materials in `serviceMaterials`." If a service has no linked materials and no static price, it has nothing to bill — that's the actual bug.

---

## `calculatedPrice` (read-only)

The API response includes a `calculatedPrice` field on each service — this is ST's **most-recent computed price** for the service, cached from the last invoice or rebuild. It's read-only (you can't set it via API) and treated as informational.

Useful for:
- Spot-checking that dynamic pricing is computing reasonable customer prices
- Listing "what does this service typically bill at" without invoking the full pricing engine
- Detecting drift between intended price and computed price after a markup rule change

Don't use it as the authoritative billing amount — it can lag behind the latest markup rules.

---

## Anti-patterns

- **Treating `useStaticPrices = null` as a bug.** It's the default. Most rows in long-lived tenants are null. Code that assumes `false` will miss the null case.
- **Trying to flip `useStaticPrices` via API on an existing service.** Silently dropped. UI-only.
- **Sending `cost` on a service POST.** Silently dropped. The path to set service cost is via linked materials, not the field directly.
- **Auditing equipment with the dynamic-pricing rule "price=0 is fine."** Equipment is always static — price=0 on active equipment is a real bug.
- **Computing markup from `price / cost`.** If `cost` is 0 (which it often is on services), you'll divide by zero or compute infinite markup. Compute on materials and let services inherit via linking.

---

## See also

- [`silent-fail-catalog.md`](silent-fail-catalog.md) — `cost` and `useStaticPrice[s]` drops
- [`service-links.md`](service-links.md) — how `serviceMaterials` carries cost into a dynamic-priced service
- [`incidents-and-lessons.md`](incidents-and-lessons.md) — incident 2026-05-08 (cost drop) and 2026-04-17 (motor replacement cost-loss)
