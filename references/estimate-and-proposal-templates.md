# Estimate & Proposal Templates

> The bit that **isn't** in the public `/sales/v2/` API.
> Internal endpoints, read↔write schema asymmetries, status transitions, and Playwright-with-cookies auth.

ServiceTitan estimate templates and proposal templates power the "build me a proposal" UX that field techs use to present multi-tier options to customers. Critically, the **template authoring endpoints are not in the public API** — they live at `/app/api/estimates/templates/*` under the same auth surface the ST web app uses.

If you've tried to find an estimate-template endpoint in `/sales/v2/` and come up empty, this is why.

## The public-API gap

| What you want | Public API (`/sales/v2/`) | Internal API (`/app/api/`) |
|---|---|---|
| Read an estimate template | ❌ No endpoint | ✅ `GET /estimates/templates/{id}` |
| Create an estimate template | ❌ No endpoint | ✅ `POST /estimates/templates` |
| Update an estimate template | ❌ No endpoint | ✅ `PUT /estimates/templates/{id}` |
| Create a proposal template | ❌ No endpoint | ✅ `POST /estimates/templates/proposals` |
| Read an estimate | ✅ `GET /sales/v2/.../estimates/{id}` | (same data, different shape) |
| Transition estimate status | ❌ PATCH silently dropped | ✅ `PUT /sales/v2/.../estimates/{id}/sell` / `/dismiss` / `/unsell` |

The public `/sales/v2/estimates/*` API is **for transacting against existing estimates** (selling, dismissing, listing). Template authoring is `/app/api/`.

---

## Auth: cookie injection from a browser session

The `/app/api/` endpoints reject OAuth tokens — they require the same cookies a logged-in ST user's browser would send.

Two paths to get there:

### 1. Manual cookie capture (one-off scripts)

Log in to your ST tenant in Chrome. F12 → Network tab → find any `/app/api/*` request → right-click → "Copy as cURL" → extract the `Cookie:` header.

Use those cookies with `curl` or any HTTP client. **Important:** plain `curl` from a Linux box often fails with 401 anyway because ST's edge does TLS fingerprinting. You need a TLS fingerprint that matches Chrome.

### 2. Playwright with cookie injection (programmatic)

Use Playwright Chromium with `addCookies()` for the cookies you captured. The Chromium TLS handshake matches what ST expects, and `addCookies()` plus a target URL is enough to access `/app/api/*` paths.

```js
import { chromium } from 'playwright';

const browser = await chromium.launch();
const context = await browser.newContext();
await context.addCookies(CAPTURED_COOKIES);  // array of {name, value, domain, path, ...}
const page = await context.newPage();

// Use page.evaluate to make fetch() calls from within the page context —
// inherits the cookie jar automatically
const result = await page.evaluate(async (url, body) => {
  const r = await fetch(url, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(body),
  });
  return { status: r.status, body: await r.json() };
}, 'https://<tenant>.servicetitan.com/app/api/estimates/templates/proposals', payload);
```

This works because Playwright's `fetch` runs in the page context, picks up the cookie jar, and uses Chromium's TLS stack.

ST does enforce 2FA on logins. There's no programmatic login flow — you have to log in once interactively, capture cookies, then replay them. Cookies have a multi-day TTL, so this is workable for batch operations but not for fully-automated daemons.

For sessions where this pattern is used regularly, having a documented cookie-injection recipe (the QSC-internal version lives in the `st-forms` skill) is worth more than re-discovering it each time.

---

## Estimate template — item shape (write)

The write shape on estimate-template items is **dramatically different** from the read shape. This is the single biggest source of 500 errors when authoring templates programmatically.

### Write shape (POST / PUT body)

```json
{
  "id": -1709999999,
  "skuReference": { "skuId": 12345678, "skuType": "Service" },
  "businessUnitId": 100,
  "generalLedgerAccountId": 42,
  "description": "Replace blower motor",
  "order": 1,
  "taxable": false,
  "isAddon": false,
  "unitPrice": 850.00,
  "totalPrice": 850.00,
  "unitCost": 320.00,
  "totalCost": 320.00,
  "standardPrice": 850.00,
  "memberPrice": 800.00,
  "quantity": 1,
  "soldHours": 1.5,
  "estimatedLaborCost": 75.00,
  "projectLabels": [],
  "allowDiscounts": true,
  "summary": ""
}
```

Verified live 2026-05-05.

### Read shape (GET response)

```json
{
  "id": 987654321,
  "skuId": 12345678,
  "skuType": "Service",
  "businessUnitId": 100,
  "generalLedgerAccount": { "id": 42, "name": "Revenue" },
  "description": "Replace blower motor",
  "order": 1,
  "isTaxable": false,
  "isAddOn": false,
  "unitPrice": 850.00,
  ...,
  "calculation": { ... },
  "customFields": [...]
}
```

### The asymmetries

| Field | Write shape | Read shape | Note |
|---|---|---|---|
| ID on create | `id: -<timestamp>` (client-generated negative temp) | `id: <real integer>` | Send negative client-temp on POST; ST assigns the real ID and returns it |
| SKU reference | `skuReference: { skuId, skuType }` (nested) | `skuId`, `skuType` (flat) | Sending flat in body → 500 NRE |
| GL account | `generalLedgerAccountId: <int>` | `generalLedgerAccount: { id, name }` | Sending object in body → 500 NRE |
| Tax flag | `taxable` | `isTaxable` | Different name. Sending `isTaxable` in body silently ignored. |
| Add-on flag | `isAddon` (lowercase 'a') | `isAddOn` (capital O) | Case-sensitive. Sending `isAddOn` in body silently ignored. |
| Description | `summary: "..."` (in body) | `description: "..."` (in response) | Different field name. Sending top-level `description` in the body is silently ignored. To set the customer-facing description you see in the UI, use `summary` in the body. |

Sending the GET shape directly back as a PUT returns **500 Internal Server Error** with a NullReferenceException message — `calculation{}` and `customFields[]` aren't writable fields and confuse the deserializer.

**Recipe:** to update an existing template, GET it, then build the PUT body **from scratch** using only writable-shape fields, copying field values across as needed. Don't try to mutate the read object in place.

---

## Proposal template — create shape

Verified live 2026-05-05:

```
POST /app/api/estimates/templates/proposals
{
  "name": "HVAC Replacement Proposal",
  "internalName": "hvac-replace-proposal",
  "description": "Three-tier replacement options",
  "proposalTypeId": 5,
  "businessUnitIds": [100, 101],
  "categoryIds": [123, 456],
  "active": true,
  "status": 1,
  "estimateAssignments": [
    { "order": 1, "estimateTemplateId": 1001, "proposalTypeOptionId": 11 },
    { "order": 2, "estimateTemplateId": 1002, "proposalTypeOptionId": 12 },
    { "order": 3, "estimateTemplateId": 1003, "proposalTypeOptionId": 13 }
  ]
}
```

Returns: **200 OK with a bare integer as the body** (the new proposal ID). Not a wrapped `{id: ...}` object — just the raw int. This trips up clients that auto-parse JSON expecting an object.

Asymmetry to know:

- `proposalTypeOptionId` (write) vs `optionId` (read). Sending `optionId` in a write body is silently ignored.

---

## Status transitions on estimates (public API)

This is a different problem from template authoring, but it's closely related and worth documenting here because the **shape of the fix is the same**: use the dedicated action endpoint, not PATCH.

### Don't

```
PATCH /sales/v2/tenant/{tenantId}/estimates/{id}
Body: { "status": "Dismissed" }
→ 200 OK. modifiedOn doesn't change. The estimate is still Open.
```

Verified 2026-05-10 against multiple estimates. Silent ignore — does not bump `modifiedOn`, does not change `status`.

### Do

```
PUT /sales/v2/tenant/{tenantId}/estimates/{id}/sell
Body: { "soldBy": <technicianId> }
→ 200 OK. Estimate is now Sold.

PUT /sales/v2/tenant/{tenantId}/estimates/{id}/dismiss
Body: {}
→ 200 OK. Estimate is now Dismissed.

PUT /sales/v2/tenant/{tenantId}/estimates/{id}/unsell
Body: {}
→ 200 OK. Returns a Sold estimate to Open.
```

Verified at scale: 258 dismissals via PUT, 250ms inter-request delay, 0 errors (2026-05-10).

If you're building an MCP tool, plugin, or any abstraction over ST estimates, make sure your tool routes status changes through the action endpoints. Don't route them through a generic PATCH wrapper — the wrapper will return 200 OK and the operation will silently fail.

---

## Why this matters for pricebook work

Bulk pricing changes flow through estimate templates as well as the pricebook itself. A scenario like "we just raised diagnostic-fee pricing by 10%" requires touching:

1. The `pb_services` items (PATCH each service's `price` if static, or update the cost-per-unit on linked materials if dynamic)
2. **Every estimate template that references those services** — the per-item `unitPrice` on the template is a *snapshot* of the service price at template-creation time. Updating the pricebook doesn't auto-update existing templates.

If you're orchestrating pricebook changes, planning the template-update pass is part of the work. The internal API access pattern (cookies + Playwright) is the path.

---

## Worked example: update template item prices after a pricebook bump

```js
// 1. Read all proposal templates referencing the affected business units
const proposals = await fetchInternalApi(
  `/app/api/estimates/templates/proposals?businessUnitIds=100,101&active=true&pageSize=200`
);

// 2. For each proposal, walk its estimate assignments → fetch each template
for (const p of proposals) {
  for (const assignment of p.estimateAssignments) {
    const template = await fetchInternalApi(`/app/api/estimates/templates/${assignment.estimateTemplateId}`);

    // 3. Rebuild items array using the WRITE shape
    const writeItems = template.items.map((item, idx) => {
      const service = pricebookLookup[item.skuId];           // your fresh pricebook snapshot
      return {
        id: -(Date.now() + idx),                              // negative client temp
        skuReference: { skuId: item.skuId, skuType: item.skuType },
        businessUnitId: item.businessUnitId,
        generalLedgerAccountId: item.generalLedgerAccount?.id,
        description: item.description,
        order: item.order,
        taxable: item.isTaxable,                              // read isTaxable → write taxable
        isAddon: item.isAddOn,                                // read isAddOn → write isAddon
        unitPrice: service.price,                             // BUMP applied
        totalPrice: service.price * item.quantity,
        unitCost: service.cost,
        totalCost: service.cost * item.quantity,
        standardPrice: service.price,
        memberPrice: service.memberPrice,
        quantity: item.quantity,
        soldHours: item.soldHours,
        estimatedLaborCost: item.estimatedLaborCost,
        projectLabels: item.projectLabels ?? [],
        allowDiscounts: item.allowDiscounts ?? true,
        summary: item.description,                            // read description → write summary
      };
    });

    // 4. PUT the rebuilt template
    await fetchInternalApi(`/app/api/estimates/templates/${assignment.estimateTemplateId}`, {
      method: 'PUT',
      body: { ...template, items: writeItems },
    });
  }
}
```

Annoying, but tractable. The "rebuild items array from scratch using the write shape" is the core trick.

---

## What about post-create UI-only fields?

Some template fields are only settable via the UI and not via either the public or internal API. The two most common:

- **Pricing-toggle on the template's underlying service** (`useStaticPrices=true/false`) — UI only post-create
- **Brand / vendor metadata on linked equipment** — UI fields that don't have a write surface

If your bulk update needs these, plan a UI follow-up pass after the API-driven bulk changes.

---

## Anti-patterns

- **PATCHing an estimate to change status.** Silently ignored. Use the action endpoints.
- **Sending the read shape back as a write.** 500 NRE. Always rebuild the body from scratch in the write shape.
- **Using the OAuth token for `/app/api/` calls.** Returns 401. Cookies only.
- **Trying to script a fresh login.** 2FA gate. Capture cookies from a real interactive session.
- **Assuming `proposalTypeOptionId` and `optionId` are interchangeable.** They are not — one is the write name, one is the read name.
- **Trusting `description` (the top-level field) on PUT to set the customer-facing description.** Silently ignored. Use `summary` in the body; ST returns it as `description` on subsequent GETs.

---

## See also

- [`silent-fail-catalog.md`](silent-fail-catalog.md) — estimate status PATCH drop (2026-05-10 entry)
- [`incidents-and-lessons.md`](incidents-and-lessons.md) — incident 2026-05-10 (estimate status silent-fail tool)
- [`write-safety.md`](write-safety.md) — the same read-after-write hash-compare pattern applies to template updates
