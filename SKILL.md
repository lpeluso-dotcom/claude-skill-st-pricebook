---
name: st-pricebook
description: ServiceTitan Pricebook v2 API knowledge — schemas, endpoints, the silent-fail field catalog, bulk import patterns, configurable equipment/materials, estimate templates, and safe-update workflows. TRIGGER on any ST pricebook task — services, materials, equipment, categories, GL accounts, bulk imports from Excel/CSV, vendor links, dynamic vs static pricing, configurable variants, estimate or proposal templates, D1 sync, or debugging a PATCH that returned 200 OK but didn't actually change anything.
---

# ServiceTitan Pricebook

A field-tested reference for working against the ServiceTitan Pricebook v2 API. Captures patterns that took roughly six months of production write operations against a multi-trade tenant to learn.

> **QSC users (this repo):** see `local.md` for tenant ID, D1 IDs, sync key, worker paths, scenario IDs.
> **Everyone else:** substitute your own values where you see `{tenantId}`, `{d1DatabaseId}`, or `<your-*>`.

## When to use this skill

- Reading or writing ST services / materials / equipment / categories via the public API
- Bulk-importing pricebook data from Excel / CSV
- Designing a D1 (or any database) mirror of the pricebook for low-latency reads or semantic search
- Creating or updating estimate / proposal templates (which live on a *different* API surface than the pricebook)
- Debugging a PATCH that returned 200 OK with no visible effect — almost certainly a silent-fail gotcha
- Cleaning up category trees, GL accounts, vendor links, or configurable equipment

## Endpoints at a glance

Base: `https://api.servicetitan.io/pricebook/v2/tenant/{tenantId}`

| Method | Path | Notes |
|---|---|---|
| `GET` | `/services?pageSize=200` | Paginate until `hasMore=false` |
| `GET` | `/materials?pageSize=200` | Same |
| `GET` | `/equipment?pageSize=200` | Same |
| `GET` | `/categories?pageSize=200` | Category tree |
| `GET` | `/{type}/{id}` | Single item |
| `PATCH` | `/{type}/{id}` | Partial update — **always GET after to verify** (see Silent-fail catalog) |
| `POST` | `/{type}` | Create new item |

Auth: OAuth2 `client_credentials` at `https://auth.servicetitan.io/connect/token`. App key, client ID, and client secret are three separate credentials — store all three securely (e.g. as worker secrets `ST_APP_KEY`, `ST_CLIENT_ID`, `ST_CLIENT_SECRET`). Tokens have a 16-hour expiry; cache them.

Page cap convention: 50 pages × 200 = 10,000 rows/table. Set a safety bound or you'll hit a runaway loop if `hasMore` is mis-reported.

Estimate templates and proposals live on a different surface — see [`references/estimate-and-proposal-templates.md`](references/estimate-and-proposal-templates.md).

## Critical safety rules

These three rules eliminate most production incidents:

1. **Every write needs a read-after-write verification.** ST returns 200 OK for many writes it silently ignores. After every PATCH / POST, GET the same item, hash the writable fields, compare. If the hash didn't change, the write didn't either — treat it as a failure, not a success.
2. **Two-phase writes for anything bulk.** Phase 1 builds the proposed change and a confirmation token; phase 2 submits the token and executes. This catches "wait, you're about to update 800 rows" mistakes before they become 800 silent failures.
3. **Use `displayName` on POST, not `name`.** ST silently drops `name` on material POSTs. Multiple other fields have the same pattern. See [`references/silent-fail-catalog.md`](references/silent-fail-catalog.md) for the full table.

Full pattern with HMAC token + worked example: [`references/write-safety.md`](references/write-safety.md).

## The silent-fail catalog (top 8)

ST returns 200 OK and silently drops these. The full catalog is in [`references/silent-fail-catalog.md`](references/silent-fail-catalog.md); these are the eight you'll hit first.

| Endpoint | Field | Drops on | Workaround |
|---|---|---|---|
| `/services` | `cost` | POST, PATCH | UI-only, or rely on dynamic pricing |
| `/services` | `useStaticPrice` (singular) | POST, PATCH | Use plural `useStaticPrices` — but it still won't flip post-create |
| `/services` (fresh) | `useStaticPrices` | PATCH | Set in UI at creation time; can't flip via API after |
| `/materials` | `name` | POST | Use `displayName` on POST; `name` works on PATCH |
| `/materials` | `categoryId` (singular) | POST | POST without it, then PATCH `categories: [<int>]` (integer array) |
| `/materials`, `/equipment` | `vendors: [...]` | POST, PATCH | Root-level `primaryVendor: {vendorId, cost, active}` object |
| `/equipment` | `typeId`, `variationsOrConfigurableEquipment` | POST, PATCH | UI-only |
| `/sales/v2/.../estimates/{id}` | `status: "Dismissed"` | PATCH | Use `PUT /{id}/sell` / `/dismiss` / `/unsell` |

## Dynamic vs static pricing

Most established ST tenants run on **dynamic pricing**: `useStaticPrices` is `null` (or `false`) on every service, and the customer price is computed at invoice time from cost + markup rules. In this model:

- `price` on a service is `0` by design. Don't flag it as a bug.
- `cost` is what matters — but ST silently drops it on POST and PATCH (see catalog). Materials drive the real cost number through `serviceMaterials` linking.
- Equipment is **always** statically priced — both `price` and `cost` matter and stick.

Decision tree, detection patterns, and tri-state behavior: [`references/cost-and-pricing.md`](references/cost-and-pricing.md).

## D1 (or any database) mirror pattern

Recommended pattern for low-latency reads and semantic search:

- Nightly worker cron paginates each ST pricebook table → `batchUpsert` into your DB → optionally embed into a vector index for search → optionally warm a KV/edge cache.
- **Pricebook tables are typically NOT on the same nightly cron as transactional data** (jobs, invoices, customers). Designs that share a single cron tend to skip pricebook because of the runtime cost of paginating 10K rows nightly. If yours does, that's fine — just confirm it.
- Trigger pricebook sync manually via a `POST /api/sync-full?days=0` style endpoint (with header auth) before any pricebook fix work — the mirror is often stale.
- Schemas, expansion columns (linked items, GL accounts, primary vendor, `is_configurable_equipment`, etc.), and SQL queries: [`references/api-reference.md`](references/api-reference.md).

## Reference index

| Task | Open |
|---|---|
| Endpoints, schemas, OAuth, pagination, GL accounts, D1 schemas | [`references/api-reference.md`](references/api-reference.md) |
| Full catalog of fields ST silently drops, with dates and verification recipe | [`references/silent-fail-catalog.md`](references/silent-fail-catalog.md) |
| Safe-update pattern: dryRun + HMAC + read-after-write hash compare | [`references/write-safety.md`](references/write-safety.md) |
| Bulk import from Excel / CSV — file shapes, validation, recovery | [`references/excel-import.md`](references/excel-import.md) |
| Configurable equipment & materials — `primaryVendor`, `displayName`, variants | [`references/configurable-equipment-and-materials.md`](references/configurable-equipment-and-materials.md) |
| Dynamic vs static pricing, `useStaticPrices` tri-state, cost behavior | [`references/cost-and-pricing.md`](references/cost-and-pricing.md) |
| Batched writes — idempotency, retries, partial failures, resume | [`references/batch-operations.md`](references/batch-operations.md) |
| Categories — walking the tree, renames, deactivation pattern | [`references/category-discovery.md`](references/category-discovery.md) |
| `serviceMaterials[]` / `serviceEquipment[]` — adding, removing, orphans | [`references/service-links.md`](references/service-links.md) |
| Estimate templates / proposals — internal API, read↔write asymmetries | [`references/estimate-and-proposal-templates.md`](references/estimate-and-proposal-templates.md) |
| Dated incidents with lessons | [`references/incidents-and-lessons.md`](references/incidents-and-lessons.md) |

## MCP tool map (when using an MCP server)

If you've built or are using an MCP wrapper around ST writes, map the tool to the silent-fail risks it inherits:

| Tool (typical naming) | Silent-fail risks |
|---|---|
| `st_patch_service` | `cost` drops, `useStaticPrice[s]` drops on fresh services, category object-shape rejection |
| `st_create_service` | `cost` drops, `useStaticPrice[s]` drops, `name` works (use over `displayName` on POST? — verify in your tenant) |
| `st_patch_material` | Category object-shape rejection, vendor-array drops |
| `st_create_material` | `name` drops (use `displayName`), `categoryId` (singular) drops, must include `primaryVendor` object or ST rolls back the create |
| `st_patch_equipment` | `typeId` drops, `variationsOrConfigurableEquipment` drops, `isConfigurable` (misspelled) drops |
| `update_estimate_status` (or equiv.) | `status` PATCH drops — switch to `PUT /sell` / `/dismiss` / `/unsell` |

The wrapper returning HTTP 200 is **not** confirmation the field stuck. Read-after-write hash compare is the only reliable signal.

## Canonical update workflow (7 steps)

For any non-trivial write (multi-row, multi-field, or cross-table):

1. **Snapshot** — pull current state of the affected items from your D1 mirror (or live API) into a working file. This is your rollback reference.
2. **CSV plan** — write the proposed changes to a CSV / spreadsheet, including only the fields you intend to change.
3. **Validate** — schema check, FK check (categories, GL accounts, vendors exist), code uniqueness, dynamic-pricing exceptions.
4. **dryRun** — for each row, compute the proposed PATCH/POST body, diff against current state using only writable fields. Skip rows where the diff is empty. Surface a summary to the operator.
5. **Confirm token** — operator signs off. If using HMAC tokens, generate one bound to the dryRun hash; if not, just a "type CONFIRM to proceed" prompt.
6. **Execute in chunks** — POST/PATCH in batches of ~25, GET + hash-compare each, log per-row outcome. On hash mismatch (silent-fail) treat the row as FAILED even if HTTP was 200.
7. **Reconcile** — refresh your D1 mirror slice for the affected rows. Don't mutate D1 directly; let the mirror catch up from ST.

The full implementation pattern (in any framework, with QSC's Cloudflare Workers example): [`references/write-safety.md`](references/write-safety.md).

## Note on Make.com integrations

The Make.com `service-titan:makeAnApiCall` module **silently drops request bodies** — confirmed on multiple workflows. Don't use it for writes. If you're orchestrating via Make, have Make trigger a worker (or any HTTP endpoint you control) that does the actual ST call via `https`.

For full historical context on which Make scenarios are deprecated and why: see [`references/incidents-and-lessons.md`](references/incidents-and-lessons.md).
