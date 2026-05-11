# ServiceTitan Pricebook — API Reference

> Schemas, endpoints, auth flow, pagination, GL accounts, and database mirror schemas.
> The patterns documented here are **verified against the live ST production API** through 2026-05.

---

## OAuth2 client_credentials

Three credentials per integration:

- `ST_APP_KEY` — header `ST-App-Key`, identifies the developer app
- `ST_CLIENT_ID` — sent in the token-exchange body
- `ST_CLIENT_SECRET` — sent in the token-exchange body

Token endpoint:

```bash
curl -X POST https://auth.servicetitan.io/connect/token \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=client_credentials&client_id=${ST_CLIENT_ID}&client_secret=${ST_CLIENT_SECRET}"
```

Response: `{ "access_token": "...", "token_type": "Bearer", "expires_in": 57600 }` — 16 hours. Cache aggressively (KV with 15h TTL is a reasonable convention).

Per-request headers:

```
Authorization: Bearer ${access_token}
ST-App-Key: ${ST_APP_KEY}
```

Tenant ID is part of the URL path, not a header.

---

## Endpoint reference

Base URL: `https://api.servicetitan.io/pricebook/v2/tenant/{tenantId}`

| Method | Path | Description |
|---|---|---|
| `GET` | `/services?pageSize=200` | List services (paginated) |
| `GET` | `/materials?pageSize=200` | List materials (paginated) |
| `GET` | `/equipment?pageSize=200` | List equipment (paginated) |
| `GET` | `/categories?pageSize=200` | List categories (tree via `parentId`) |
| `GET` | `/services/{id}` | Single service |
| `GET` | `/materials/{id}` | Single material |
| `GET` | `/equipment/{id}` | Single equipment |
| `PATCH` | `/services/{id}` | Partial update (silent-drops common — see catalog) |
| `PATCH` | `/materials/{id}` | Partial update |
| `PATCH` | `/equipment/{id}` | Partial update |
| `POST` | `/services` | Create service |
| `POST` | `/materials` | Create material (`primaryVendor` required) |
| `POST` | `/equipment` | Create equipment (`primaryVendor` required) |

There is **no DELETE** for pricebook items. To remove a SKU, set `active: false` via PATCH.

### Pagination

Default and max `pageSize` is `200`. Response:

```json
{ "data": [ ... ], "hasMore": true, "totalCount": null }
```

Loop until `hasMore === false` or the current `data` array is empty. **Always set a safety cap** (e.g. 50 pages × 200 = 10,000 rows/table) — `hasMore` has occasionally been observed stuck at `true` on edge cases.

### Rate limiting

ST has historically published a "120 requests/minute" figure, but the enforced behavior is more lenient and varies by endpoint. The reliable pattern: don't preemptively throttle, **back off on 429**, retry with exponential delay (start at 2s, double, cap at 60s). At 250ms between sequential single-row writes, 258 PATCH/PUTs completed with 0 errors in real production batches.

---

## Response schemas

### Service object

```json
{
  "id": 12345678,
  "code": "HV-RES-ST",
  "displayName": "HVAC Diagnostic Fee (Residential)",
  "description": "Standard residential HVAC diagnostic...",
  "warranty": null,
  "categories": [{ "id": 123, "name": "HVAC Diagnostic Fee" }],
  "price": 85.00,
  "memberPrice": 0,
  "addOnPrice": 0,
  "addOnMemberPrice": 0,
  "taxable": false,
  "account": "Revenue",
  "hours": 0.5,
  "isLabor": true,
  "recommendations": [],
  "upgrades": [],
  "assets": [],
  "serviceMaterials": [{ "skuId": "MAT-123", "quantity": 1 }],
  "serviceEquipment": [{ "skuId": "EQ-456", "quantity": 1 }],
  "active": true,
  "crossSaleGroup": null,
  "paysCommission": true,
  "bonus": 0,
  "commissionBonus": 0,
  "modifiedOn": "2026-03-16T10:54:58Z",
  "source": null,
  "externalId": null,
  "externalData": null,
  "businessUnitId": null,
  "cost": 110.00,
  "createdOn": "2024-01-15T00:00:00Z",
  "soldByCommission": false,
  "defaultAssetUrl": null,
  "budgetCostCode": null,
  "budgetCostType": null,
  "isPriceLocked": false,
  "calculatedPrice": 85.00,
  "useStaticPrices": null
}
```

Verified key count: 38 keys returned on a list response (as of 2026-05-09). The keys `calculatedPrice` and `useStaticPrices` are tri-state-meaningful (see `cost-and-pricing.md`); `isPriceLocked` is read-only.

### Material object

```json
{
  "id": 87654321,
  "code": "RDP-6",
  "displayName": "6\" Round Duct Pipe",
  "description": "6-inch round duct pipe",
  "price": 15.50,
  "cost": 8.25,
  "memberPrice": 12.00,
  "active": true,
  "taxable": true,
  "unitOfMeasure": "EA",
  "account": "Revenue",
  "costOfSaleAccount": "Materials",
  "assetAccount": null,
  "categories": [{ "id": 456, "name": "Round Duct Pipe" }],
  "vendorPricingLinks": [{
    "id": 111, "vendorId": 222, "vendorName": "Example Vendor",
    "cost": 7.50, "vendorCode": "RDP6", "isPrimary": true
  }],
  "primaryVendor": { "vendorId": 222, "cost": 7.50, "active": true },
  "modifiedOn": "2026-03-16T10:55:17Z",
  "createdOn": "2024-06-01T00:00:00Z"
}
```

Note the read-shape asymmetry: `vendorPricingLinks` is the full read array; `primaryVendor` is the write-required summary object that ST returns alongside it. Materials POST/PATCH writes go through `primaryVendor`, not `vendorPricingLinks`. See [`silent-fail-catalog.md`](silent-fail-catalog.md).

### Equipment object

```json
{
  "id": 11223344,
  "code": "UP16AZ24AJ3CA",
  "displayName": "Example 2-Ton Heat Pump",
  "description": "Features and Benefits...",
  "price": 0,
  "cost": 0,
  "memberPrice": 0,
  "active": true,
  "taxable": true,
  "categories": [{ "id": 789, "name": "Heat Pump" }],
  "account": "Revenue",
  "brand": "ExampleBrand",
  "manufacturer": "Example Manufacturer Inc.",
  "model": "UP16AZ24AJ3CA",
  "isConfigurableEquipment": false,
  "typeId": null,
  "variationsOrConfigurableEquipment": [],
  "equipmentMaterialLinks": [],
  "primaryVendor": { "vendorId": 333, "cost": 0, "active": true },
  "modifiedOn": "2026-03-16T10:54:40Z",
  "createdOn": "2025-02-01T00:00:00Z"
}
```

`typeId` and `variationsOrConfigurableEquipment` are both **read-only**; only `isConfigurableEquipment` (boolean) is writable. See [`configurable-equipment-and-materials.md`](configurable-equipment-and-materials.md).

### Unit of measure values (materials)

`EA` (each), `BX` (box), `RL` (roll), `SH` (sheet), `BG` (bag), `FT` (foot), `GL` (gallon), `CS` (case), `PR` (pair), `PK` (pack).

---

## GL accounts

GL accounts are referenced by **string name** in the API (e.g. `"Revenue"`, `"Materials"`, `"Customer Prepayments:Yearly Prepaid-Pref Cust MA"`). The exact strings come from your ST tenant's chart of accounts. Common categories:

- **Revenue** — Standard revenue recognition (services, parts, equipment sales)
- **Materials** — Materials purchased and resold (cost-tracked separately)
- **Customer Prepayments:\*** — Deferred revenue for maintenance agreements (recognized when service performed)
- **Cust Prepayments:Purchase Accrual** — Prepaid service credits
- **Revenue:Inter-Company Revenue** — Inter-company billing
- **Interdepartmental Billing:1st Year Warranty** — Warranty repair cross-charges
- **Back Charges/Customer Repairs** — Warranty/rework items
- **Revenue:Returns & Allowances** — Refund/credit items
- **Retention Receivable** — Contract retention holdbacks
- **Deposits on Future Work** — Customer deposits not yet earned
- **Membership Agreements** — Membership-specific revenue

A material can have a separate `costOfSaleAccount` (for COGS recognition) and `assetAccount` (for inventory tracking) — both are nullable string fields. Equipment items typically inherit their cost-of-sale GL account from their category.

If you can't list GL accounts via the API: pull them from a CSV export of the pricebook, or read them from your D1 mirror after a full sync.

---

## D1 mirror schemas (reference)

This is one tested D1 schema for mirroring the pricebook. Adapt for your database of choice.

### `pb_services`

```sql
CREATE TABLE pb_services (
  id           TEXT PRIMARY KEY,           -- ST service ID
  code         TEXT,                       -- SKU code
  name         TEXT,                       -- displayName
  description  TEXT,
  price        REAL,                       -- Customer price ($)
  member_price REAL,                       -- Member price ($)
  cost         REAL,                       -- Internal cost ($) — read-only via API
  hours        REAL,                       -- Billable hours
  is_labor     INTEGER,                    -- Boolean (0/1)
  active       INTEGER,
  taxable      INTEGER,
  category_name TEXT,                      -- Flattened hierarchy ("HVAC > Diagnostic Fees")
  use_static_prices INTEGER,               -- Tri-state: 1 / 0 / NULL
  calculated_price REAL,                   -- ST's most-recent computed price (read-only mirror)
  account      TEXT,                       -- GL account string
  materials_json TEXT,                     -- JSON: [{ skuId, quantity }]
  equipment_json TEXT,                     -- JSON: [{ skuId, quantity }]
  recommendations_json TEXT,
  upgrades_json TEXT,
  business_unit_id INTEGER,
  pays_commission INTEGER,
  cross_sale_group TEXT,
  synced_at    TEXT,
  modified_on  TEXT
);
```

### `pb_materials`

```sql
CREATE TABLE pb_materials (
  id            TEXT PRIMARY KEY,
  code          TEXT,
  name          TEXT,
  description   TEXT,
  price         REAL,
  member_price  REAL,
  cost          REAL,                      -- Vendor/landed cost
  active        INTEGER,
  taxable       INTEGER,
  unit_of_measure TEXT,                    -- "EA", "BX", etc.
  category_name TEXT,
  account       TEXT,                      -- GL account
  cost_of_sale_account TEXT,
  asset_account TEXT,
  primary_vendor_id INTEGER,
  primary_vendor_name TEXT,
  primary_vendor_cost REAL,
  vendors_json  TEXT,                      -- Full vendorPricingLinks array
  is_configurable INTEGER,
  synced_at     TEXT,
  modified_on   TEXT
);
```

### `pb_equipment`

```sql
CREATE TABLE pb_equipment (
  id           TEXT PRIMARY KEY,
  code         TEXT,
  name         TEXT,
  description  TEXT,
  price        REAL,
  member_price REAL,
  cost         REAL,
  active       INTEGER,
  taxable      INTEGER,
  category_name TEXT,
  account      TEXT,
  cost_of_sale_account TEXT,
  asset_account TEXT,
  brand        TEXT,
  manufacturer TEXT,
  model        TEXT,
  hours        REAL,
  item_type    TEXT,                       -- "Air Handler", "Heat Pump", etc.
  is_configurable_equipment INTEGER,
  variations_json TEXT,                    -- variationsOrConfigurableEquipment
  equipment_materials_json TEXT,           -- equipmentMaterialLinks
  primary_vendor_id INTEGER,
  primary_vendor_name TEXT,
  primary_vendor_cost REAL,
  synced_at    TEXT,
  modified_on  TEXT
);
```

### `pb_categories`

```sql
CREATE TABLE pb_categories (
  id           INTEGER PRIMARY KEY,
  name         TEXT,
  parent_id    INTEGER,                    -- NULL for top-level
  active       INTEGER,
  business_unit_id INTEGER,
  synced_at    TEXT
);
```

See [`category-discovery.md`](category-discovery.md) for queries that walk the tree.

---

## Sync architecture (reference)

The pattern that's worked at production scale:

```
nightly cron (typically NOT bundled with transactional sync)
  └─→ handleFullSync()
        ├─→ paginate /pricebook/v2/{tenantId}/services?pageSize=200 → batchUpsert pb_services
        ├─→ paginate .../materials → batchUpsert pb_materials
        ├─→ paginate .../equipment → batchUpsert pb_equipment
        ├─→ paginate .../categories → batchUpsert pb_categories
        ├─→ embedPricebook() — vector index for semantic search (e.g. bge-small-en-v1.5, batch 100)
        └─→ warm KV / edge cache
```

Note: pricebook tables are commonly **gated out of an every-2-hour transactional sync** because 10K-row pagination dominates the runtime. Many teams run pricebook on a separate nightly cron at off-peak hours (typically 02:00–03:00 local). If you ever discover your pricebook mirror is 14–28 days stale (it happens), it's usually because the cron skipped silently or `updated_at` isn't being touched on unchanged rows. Run a manual full rebuild before any pricebook fix work.

### Manual rebuild endpoint convention

```bash
# Single table
curl -X POST "https://<your-worker>.example.workers.dev/api/sync-full?table=pb_services" \
  -H "X-Sync-Key: <your-sync-key>"

# Full rebuild (all pricebook tables)
curl -X POST "https://<your-worker>.example.workers.dev/api/sync-full?days=0" \
  -H "X-Sync-Key: <your-sync-key>"
```

---

## Common SQL queries

### Count by type

```sql
SELECT 'services' as type, COUNT(*) as cnt FROM pb_services WHERE active = 1
UNION ALL SELECT 'materials', COUNT(*) FROM pb_materials WHERE active = 1
UNION ALL SELECT 'equipment', COUNT(*) FROM pb_equipment WHERE active = 1;
```

### Search pricebook (all types)

```sql
SELECT 'service' AS type, code, name, price, category_name, account
FROM pb_services WHERE active = 1 AND (name LIKE '%query%' OR code LIKE '%query%')
UNION ALL
SELECT 'material', code, name, price, category_name, account
FROM pb_materials WHERE active = 1 AND (name LIKE '%query%' OR code LIKE '%query%')
UNION ALL
SELECT 'equipment', code, name, price, category_name, account
FROM pb_equipment WHERE active = 1 AND (name LIKE '%query%' OR code LIKE '%query%')
ORDER BY name LIMIT 50;
```

### GL account summary

```sql
SELECT account, COUNT(*) AS items,
  SUM(CASE WHEN price > 0 THEN 1 ELSE 0 END) AS priced,
  SUM(CASE WHEN price = 0 THEN 1 ELSE 0 END) AS zero_price
FROM pb_services WHERE active = 1
GROUP BY account ORDER BY items DESC;
```

### Equipment with cost but no price (data-quality check)

```sql
SELECT code, name, cost, price, category_name
FROM pb_equipment
WHERE cost > 0 AND price = 0 AND active = 1
ORDER BY cost DESC;
```

This query is the standard early-warning for cost exposure — equipment is always statically priced, so non-zero cost with zero price means real margin risk.

### Materials with no primary vendor

```sql
SELECT code, name, cost, price
FROM pb_materials
WHERE active = 1 AND (primary_vendor_id IS NULL OR primary_vendor_id = 0)
ORDER BY cost DESC LIMIT 50;
```

### Services priced statically at $0 (billing risk)

```sql
SELECT id, code, name, price, member_price, account
FROM pb_services
WHERE active = 1 AND use_static_prices = 1 AND price = 0;
```

These are services flagged as static-priced where the price is $0 — they'll bill at $0 on invoices. Almost always a configuration error (someone flipped useStaticPrices but didn't set price).

---

## Quick reference: creating pricebook items via API

### Create a service

```bash
curl -X POST "https://api.servicetitan.io/pricebook/v2/tenant/{tenantId}/services" \
  -H "Authorization: Bearer $TOKEN" \
  -H "ST-App-Key: $ST_APP_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "code": "MY-SVC-001",
    "displayName": "My New Service",
    "description": "Description here",
    "price": 0,
    "memberPrice": 0,
    "hours": 1.0,
    "isLabor": true,
    "active": true,
    "taxable": false,
    "account": "Revenue",
    "categories": [12345]
  }'
```

Note `categories: [<int>]` is a **bare integer array** at the body root. Sending `[{"id": <int>}]` (the read shape) returns 400 deserialization error.

`cost` is intentionally omitted — ST silently drops it on POST. See [`silent-fail-catalog.md`](silent-fail-catalog.md).

### Create a material

```bash
curl -X POST "https://api.servicetitan.io/pricebook/v2/tenant/{tenantId}/materials" \
  -H "Authorization: Bearer $TOKEN" \
  -H "ST-App-Key: $ST_APP_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "code": "MAT-001",
    "displayName": "My Material",
    "description": "Description",
    "cost": 10.00,
    "price": 25.00,
    "unitOfMeasure": "EA",
    "active": true,
    "taxable": true,
    "account": "Revenue",
    "primaryVendor": { "vendorId": 222, "cost": 8.00, "active": true }
  }'
```

Without `primaryVendor`, ST returns 400 "There must be exactly one vendor selected as Primary" **and rolls back the create** — a follow-up GET on the reported Sku.Id will return 404. `categories` can be added on a follow-up PATCH.

### PATCH (partial update)

```bash
curl -X PATCH "https://api.servicetitan.io/pricebook/v2/tenant/{tenantId}/services/12345678" \
  -H "Authorization: Bearer $TOKEN" \
  -H "ST-App-Key: $ST_APP_KEY" \
  -H "Content-Type: application/json" \
  -d '{ "price": 95.00, "memberPrice": 80.00 }'
```

**Always GET after PATCH and compare hashes of the writable fields** — see [`silent-fail-catalog.md`](silent-fail-catalog.md) for the verification recipe and [`write-safety.md`](write-safety.md) for the full pattern.
