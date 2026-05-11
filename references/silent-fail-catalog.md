# Silent-Fail Catalog

> Complete catalog of fields the ServiceTitan API silently drops on POST and/or PATCH.
> Every entry verified against the live production API. Dates indicate when the gotcha was first
> observed; updates record subsequent re-verification.

The defining characteristic of these gotchas: **the API returns HTTP 200 OK with a normal-looking response body, but the field was never accepted.** No 4xx, no warning header, no log entry. Detection requires reading the record back and comparing.

## How to read this table

- **Drops on POST?** — sending the field in a CREATE request: yes = silently ignored.
- **Drops on PATCH?** — sending the field in an UPDATE: yes = silently ignored.
- **n/a** — endpoint doesn't accept this verb for this resource.
- **Workaround** — the path that actually works.
- **First seen / Re-verified** — dates of confirmation. Memory fades; verification dates matter.

---

## The catalog

| Endpoint | Field | Drops on POST? | Drops on PATCH? | Workaround | First seen / Re-verified |
|---|---|---|---|---|---|
| `/services` | `cost` | yes | yes | UI-only, OR design around dynamic pricing (cost flows in via `serviceMaterials`) | 2026-04-17, re-verified 2026-05-08 |
| `/services` | `useStaticPrice` (singular) | yes | yes | Use `useStaticPrices` (plural). Even then, doesn't flip on freshly-created services. | 2026-05-08 |
| `/services` | `useStaticPrices` (plural) on freshly-created service | n/a | yes | Set in UI at creation time. Cannot be flipped via API after the service is created. | 2026-05-08 |
| `/services` | `price` (on dynamic-pricing service, `useStaticPrices=false` or `null`) | n/a | yes | Static prices only stick when `useStaticPrices=true` — which can't be flipped via API. UI is the workaround. | 2026-04-17 |
| `/materials` | `name` | yes | accepts | Use `displayName` on POST. PATCH `name` afterward if needed. | 2026-04 |
| `/materials` | `categoryId` (singular, integer field) | yes | n/a | POST without it. Then PATCH `categories: [<int>]` (bare integer array — see "categories shape" entry below) | 2026-04 |
| `/materials`, `/equipment` | `vendors: [...]` | yes | yes | Root-level `primaryVendor: { vendorId, cost, active }` (singular object). Without it on POST, ST returns 400 "There must be exactly one vendor selected as Primary" AND rolls back the create — follow-up GET on Sku.Id 404s. | canonical schema doc |
| `/materials`, `/equipment` | `vendorPricingLinks: [...]` | yes | yes | Same as `vendors:` — `primaryVendor` is the only writable vendor field. `vendorPricingLinks` is the read shape only. | canonical schema doc |
| `/equipment` | `typeId` | yes | yes | UI-only: Pricebook → Equipment → [item] → Type dropdown. | 2026-04 |
| `/equipment` | `equipmentTypeId`, `type`, `equipmentType` (misspellings) | yes | yes | None — these aren't ST field names. Use `typeId` (UI-only) | 2026-04 |
| `/equipment` | `variationsOrConfigurableEquipment` | yes | yes | UI-only: Pricebook → Equipment → [parent] → Configurable Equipment tab. Only `isConfigurableEquipment` (boolean) is writable. | 2026-04 |
| `/equipment` | `isConfigurable` (wrong field name) | yes | yes | Use `isConfigurableEquipment` (correct field name). | tool-catalog errata 2026-04-22 |
| `/categories` | `name` | n/a | yes | UI-only renames. `active` toggle PATCH works fine — verified across multiple categories 2026-04-27. | 2026-04-27 |
| `/sales/v2/.../estimates/{id}` | `status: "Dismissed"` / `"Sold"` / `"Open"` | n/a | yes | Use dedicated action endpoints: `PUT /{id}/sell` (body `{soldBy: <techId>}`), `PUT /{id}/dismiss` (body `{}`), `PUT /{id}/unsell` (body `{}`). `modifiedOn` doesn't even bump on the PATCH attempt. | 2026-05-10 |
| `/jpm/v2/.../job-types[/{id}]` | `isSmartDispatched` | yes | yes | UI-only: Settings → Operations → Job Types → edit → Smart Dispatch. Field is API-readable but not API-writable. | 2026-05-09 |
| `/settings/v2/.../technicians/{techId}` | `roleIds: [<int>]` (plural array) | n/a | yes | Use `{roleId: <int>}` (singular int) in the body. Read shape returns `roleIds: [<int>]` array; write shape expects singular. | 2026-04 |
| `/taskmanagement/...` | any field on PATCH/PUT | n/a | yes | Task management is **read + create only**. Updates fail silently across all fields. | 2026-04 |
| body root, any pricebook endpoint | `categories: [{id: <int>}]` (object array shape) | yes | yes | Use the bare integer array `categories: [<int>]`. Object shape returns 400 deserialization error — this one at least is loud. | canonical schema doc |
| URL path | `/task-management/` (with hyphen) | n/a | n/a | Correct path is `/taskmanagement/` (no hyphen). Wrong path 404s. Not silent, but easy to mis-type. | path-corrections middleware doc |

---

## Verification recipe (read-after-write hash compare)

The pattern that catches every silent drop:

```js
const writable = ['code', 'displayName', 'description', 'price', 'memberPrice',
                  'hours', 'isLabor', 'taxable', 'account', 'paysCommission',
                  'categories', 'serviceMaterials', 'serviceEquipment'];

function hash(obj, keys) {
  const slice = {};
  for (const k of keys) slice[k] = obj[k];
  return crypto.createHash('sha256').update(JSON.stringify(slice)).digest('hex');
}

// 1. Pre-write: hash current state
const pre = await stGet(`/services/${id}`);
const preHash = hash(pre, writable);

// 2. Execute the PATCH
await stPatch(`/services/${id}`, body);

// 3. Post-write: hash again
const post = await stGet(`/services/${id}`);
const postHash = hash(post, writable);

// 4. If the hash didn't change, the write didn't either — even if HTTP was 200
if (preHash === postHash && Object.keys(body).length > 0) {
  throw new SilentDropError(`PATCH on /services/${id} silently dropped ${Object.keys(body).join(',')}`);
}
```

Run this for every write. The 200ms cost of the follow-up GET is dwarfed by the cost of a 800-row "successful" import that didn't actually change anything.

For batch operations, hash the **diff** (writable fields you intended to change) rather than the whole object — that way unrelated changes from other actors don't trip the alarm. See [`write-safety.md`](write-safety.md) for the full pattern.

---

## When ST patches a gotcha

ST does occasionally fix these. If you discover a documented field now sticks, please file a PR with:

1. The gotcha you're invalidating (link to the row above)
2. Your re-verification date
3. Your tenant region if you know it (some fixes have been observed rolling out region-by-region)
4. (Optional) The request/response that confirmed it now works

We keep retired entries in the catalog with a strikethrough and a "fixed on [date], previously dropped on [verbs]" annotation rather than deleting them — they're useful for code that needs to support older tenant configurations.

---

## Not in this catalog

Some related ST gotchas live in adjacent reference files:

- **Estimate template internal API read↔write asymmetries** (`skuReference` wrapping, `isAddon` case, `summary` ↔ `description`) — these aren't drops, they're shape mismatches that return 500 NRE. See [`estimate-and-proposal-templates.md`](estimate-and-proposal-templates.md).
- **Service-link removal pattern** (no DELETE endpoint; GET → filter → PATCH full array) — not a silent drop, just a missing verb. See [`service-links.md`](service-links.md).
- **Dynamic vs static pricing model** (`useStaticPrices` tri-state) — see [`cost-and-pricing.md`](cost-and-pricing.md).
