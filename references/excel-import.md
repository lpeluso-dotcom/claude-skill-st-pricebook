# Excel / CSV Bulk Import

> Pattern for bulk-importing pricebook updates from Excel or CSV into ServiceTitan,
> with read-after-write verification that catches silent-fail drops in real time.

This is the most common "I need to update a lot of pricebook items at once" workflow. Operations or pricing analysts hand you a spreadsheet; you need to turn it into ST API calls without dropping fields, mis-categorizing items, or burning hours on retries.

## Recommended file shapes

Use one CSV per item type — services, materials, equipment — because the writable fields differ enough that mixing them in one sheet creates ambiguity.

### `services.csv`

```
code,displayName,description,hours,isLabor,taxable,account,paysCommission,categories,useStaticPrices,price,memberPrice
HV-RES-DIAG,HVAC Residential Diagnostic,Standard 1hr residential diagnostic,1.0,true,false,Revenue,true,"123,456",false,0,0
```

Notes:
- `cost` is intentionally omitted — ST silently drops it on services. Cost flows via linked materials.
- `categories` is a comma-separated list of category IDs (integers). Convert to `[123, 456]` (bare integer array) in code.
- `useStaticPrices` is included but **note**: on freshly-created services, the post-create flip silently fails. If you need static pricing, set it in the UI.

### `materials.csv`

```
code,displayName,description,cost,price,memberPrice,primaryVendorId,primaryVendorCost,unitOfMeasure,active,taxable,account,costOfSaleAccount,categoryId
RDP-6,6" Round Duct Pipe,6-inch round duct pipe,8.25,15.50,12.00,222,7.50,EA,true,true,Revenue,Materials,456
```

Notes:
- `primaryVendorId` + `primaryVendorCost` get packaged into the `primaryVendor` object on POST. Without `primaryVendorId`, the create fails with 400 and rolls back.
- `categoryId` is **read on the create path but silently dropped on POST**. The pattern is: POST without it, then PATCH `categories: [<int>]`. The CSV column carries it through both steps.

### `equipment.csv`

```
code,displayName,description,cost,price,memberPrice,primaryVendorId,brand,manufacturer,model,isConfigurableEquipment,hours,active,taxable,account,categoryId
UP16AZ24,Example 2-Ton Heat Pump,Features and benefits,1800,3500,3200,222,ExampleBrand,Example Mfg Inc,UP16AZ24AJ3CA,false,0,true,true,Revenue,789
```

Notes:
- Equipment is **always** statically priced — both `cost` and `price` matter and stick on POST.
- `typeId` and `variationsOrConfigurableEquipment` are read-only — don't include them. If equipment needs a Type, set it in the UI after create.
- `isConfigurableEquipment` (boolean) is writable. Only the flag — variant records are separate entities and the linkage is UI-managed.

---

## Pipeline

```
read CSV  →  validate  →  preview (dryRun)  →  operator confirm  →  execute in chunks  →  verify per-row  →  log + reconcile
```

### Step 1 — Read

Use any standard library — `csv-parse` for CSV in Node, `xlsx` for Excel sheets.

```js
import { parse } from 'csv-parse/sync';

const rows = parse(await fs.readFile('services.csv'), {
  columns: true,
  skip_empty_lines: true,
  trim: true,
});
```

### Step 2 — Validate

Catch the dumb errors before you hit the API:

```js
function validateRow(row, type) {
  const errors = [];

  // Required fields
  if (!row.code || row.code.length === 0) errors.push('missing code');
  if (!row.displayName || row.displayName.length === 0) errors.push('missing displayName');

  // Type coercion
  if (row.price && isNaN(Number(row.price))) errors.push(`bad price: ${row.price}`);
  if (row.cost && isNaN(Number(row.cost))) errors.push(`bad cost: ${row.cost}`);

  // FK existence (against your D1 mirror or a live API check)
  if (row.categoryId && !(await categoryExists(row.categoryId))) {
    errors.push(`unknown category ${row.categoryId}`);
  }

  // Vendor existence (materials/equipment only)
  if (type !== 'service' && row.primaryVendorId && !(await vendorExists(row.primaryVendorId))) {
    errors.push(`unknown vendor ${row.primaryVendorId}`);
  }

  // Code uniqueness (against your D1 mirror)
  const existing = await findByCode(type, row.code);
  if (existing && process.argv.includes('--create-only')) {
    errors.push(`code ${row.code} already exists (id ${existing.id})`);
  }

  // Cost > 0 sanity (except for services, where cost is silently dropped anyway)
  if (type !== 'service' && Number(row.cost) <= 0) {
    errors.push(`cost must be > 0 (got ${row.cost})`);
  }

  return errors;
}
```

Run validation on **all rows** before any API calls. A 500-row import with row 487 invalid should fail-fast, not fail after 486 successful writes.

### Step 3 — Preview (dryRun)

For each valid row, compare against the live or D1-mirrored current state. Surface only the rows that would actually change:

```js
async function buildPreview(rows, type) {
  const previews = [];
  for (const row of rows) {
    const proposed = csvRowToApiBody(row, type);
    const current = await findByCode(type, row.code);

    if (!current) {
      previews.push({ row, action: 'CREATE', body: proposed });
    } else {
      const diff = computeDiff(current, proposed, WRITABLE_FIELDS[type]);
      if (Object.keys(diff).length === 0) {
        previews.push({ row, action: 'SKIP', reason: 'no change' });
      } else {
        previews.push({ row, action: 'PATCH', id: current.id, body: diff });
      }
    }
  }
  return previews;
}
```

Output a summary:

```
500 rows in services.csv
  → 312 SKIP (no change)
  → 173 PATCH
  → 15 CREATE
  → 0 invalid
```

Show the first 10 PATCH/CREATE entries in full so the operator can spot-check. Then prompt for confirmation.

### Step 4 — Execute with chunked POST/PATCH

```js
async function executeImport(previews) {
  const results = [];
  for (const chunk of chunked(previews.filter(p => p.action !== 'SKIP'), 25)) {
    for (const p of chunk) {
      try {
        const resp = p.action === 'CREATE'
          ? await stPost(`/${type}s`, p.body)
          : await stPatch(`/${type}s/${p.id}`, p.body);
        results.push({ ...p, http: resp.status, ok: true });
      } catch (e) {
        results.push({ ...p, error: e.message, ok: false });
      }
    }
    await sleep(250);  // simple inter-chunk pause
  }
  return results;
}
```

### Step 5 — Verify (read-after-write)

For each successful HTTP response, GET the item and hash-compare:

```js
async function verifyResults(results) {
  const verified = [];
  for (const r of results) {
    if (!r.ok) { verified.push(r); continue; }

    const id = r.action === 'CREATE' ? r.http_response.id : r.id;
    const post = await stGet(`/${type}s/${id}`);
    const droppedFields = detectDropped(r.body, post);

    if (droppedFields.length > 0) {
      verified.push({ ...r, silent_drop: true, dropped: droppedFields });
    } else {
      verified.push({ ...r, silent_drop: false });
    }
  }
  return verified;
}

function detectDropped(body, post) {
  return Object.keys(body).filter(k =>
    JSON.stringify(body[k]) !== JSON.stringify(post[k])
  );
}
```

A row with `silent_drop: true` is a partial success — the HTTP write returned 200, but one or more fields didn't stick. Surface it.

### Step 6 — Log + recover

Append three CSVs to your output directory:

- `success.csv` — `code, id, action, fields_changed`
- `silent-drops.csv` — `code, id, action, dropped_fields, original_value, attempted_value`
- `failures.csv` — `code, action, error, http_status, response_body`

The recovery workflow:

- For `silent-drops.csv`: look up each dropped field in [`silent-fail-catalog.md`](silent-fail-catalog.md), apply the documented workaround (usually UI-only), then re-run the import to verify the second-pass writes do stick.
- For `failures.csv`: fix the root cause (bad data, missing FK, etc.) and re-run only those rows.
- For `success.csv`: nothing to do.

The importer must be **resumable**. If row 312 of 500 fails (network blip), you should be able to fix and re-run from row 312 without re-doing 311 successes. Easiest pattern: track a checkpoint file by row index, and on restart skip rows that already appear in `success.csv` or `silent-drops.csv`.

### Step 7 — Reconcile your D1 mirror

After a successful batch, trigger your sync to bring D1 up to date with ST:

```bash
curl -X POST "https://<your-worker>.example.workers.dev/api/sync-full?days=0" \
  -H "X-Sync-Key: <your-sync-key>"
```

Don't mutate D1 directly — see [`write-safety.md`](write-safety.md) for why.

---

## Column → field type coercion table

Common patterns that trip up CSV → JSON:

| CSV value | API expects | Coercion |
|---|---|---|
| `"true"` / `"false"` | boolean | `value.toLowerCase() === 'true'` |
| `"1"` / `"0"` | boolean | `value === '1'` |
| `"123,456"` | integer array | `value.split(',').map(s => parseInt(s.trim(), 10))` |
| `"8.25"` | decimal | `Number(value)` (ST uses native floats for prices) |
| `""` (empty cell) | null vs unset | If unset means "leave alone" → omit the key from body. If unset means null → explicit `null`. Match what your spreadsheet operator means. |
| `"$1,234.56"` | decimal | `Number(value.replace(/[\$,]/g, ''))` — Excel often adds currency formatting |
| ISO date `"2026-05-10"` | ST datetime | ST returns ISO 8601 with TZ; for writes, most date fields are read-only anyway |

---

## Worked example: 145-row HVAC material bulk-create

Real production pattern (paraphrased; identifiers genericized):

```
1. Operator drops materials.csv (145 rows: code, displayName, cost, price, vendorId, vendorCost, category) in inbox
2. Validation: 145 valid, 0 invalid. All vendor IDs and category IDs resolved against D1 mirror.
3. Preview: 145 CREATE (no existing codes), 0 SKIP, 0 PATCH.
4. Operator confirms.
5. Execute: 145 POSTs across 6 chunks of 25 (last chunk 20), 250ms between chunks. Total wall time ~90s.
6. Verify: 145/145 200 OK on POST, but read-back finds:
   - 142 successful (categories set, vendor set, costs stuck)
   - 3 silent-drop on `categoryId` (singular field — should have been `categories: [<int>]`)
7. Fix-up: PATCH the 3 silent-drop rows with the correct `categories: [<int>]` shape. Re-verify. All stick.
8. Reconcile: POST /api/sync-full?table=pb_materials. Mirror catches up.
```

Without read-after-write verification, those 3 silent-drop rows would have looked successful and shown up as "uncategorized" later in a data-quality audit. The 5-minute cost of the verify phase is dwarfed by the cost of debugging "why are some of my new materials uncategorized" a week later.

---

## Sheet → API field mapping (services)

| CSV column | API field | Notes |
|---|---|---|
| `code` | `code` | Unique SKU identifier — also doubles as dedup key for idempotency |
| `displayName` | `displayName` | Customer-facing name. Use this on POST, not `name`. |
| `description` | `description` | Free-text description |
| `hours` | `hours` | Decimal hours billed |
| `isLabor` | `isLabor` | Boolean — true for labor services |
| `taxable` | `taxable` | Boolean |
| `account` | `account` | GL account string (e.g. `"Revenue"`) |
| `paysCommission` | `paysCommission` | Boolean |
| `categories` | `categories: [<int>]` | Comma-separated category IDs → bare integer array |
| `useStaticPrices` | `useStaticPrices` | **Sticks on create but cannot be flipped via API post-create** |
| `price` | `price` | Only meaningful if `useStaticPrices=true` |
| `memberPrice` | `memberPrice` | Member-tier price |
| ~~`cost`~~ | (silently dropped — omit from POST body) | UI-only |
| `serviceMaterials` | `serviceMaterials: [{materialId, quantity}]` | Full-array PATCH semantics — see [`service-links.md`](service-links.md) |
| `serviceEquipment` | `serviceEquipment: [{equipmentId, quantity}]` | Same |

---

## Sheet → API field mapping (materials)

| CSV column | API field | Notes |
|---|---|---|
| `code` | `code` | Unique SKU identifier |
| `displayName` | `displayName` | Use on POST. `name` is silently dropped. |
| `description` | `description` | Free-text |
| `cost` | `cost` | Decimal — sticks on materials (unlike services) |
| `price` | `price` | Customer price |
| `memberPrice` | `memberPrice` | Member-tier |
| `primaryVendorId` | `primaryVendor.vendorId` | Required on POST. |
| `primaryVendorCost` | `primaryVendor.cost` | Cost from this vendor — typically equals `cost` |
| `unitOfMeasure` | `unitOfMeasure` | `EA`, `BX`, `RL`, `SH`, `BG`, `FT`, `GL`, `CS`, `PR`, `PK` |
| `active` | `active` | Boolean |
| `taxable` | `taxable` | Boolean |
| `account` | `account` | GL account string |
| `costOfSaleAccount` | `costOfSaleAccount` | COGS GL account |
| `categoryId` | (drop from POST; follow with PATCH `categories: [<int>]`) | Category drops on POST |

---

## Anti-patterns

- **Bulk-creating without read-after-write verification.** You'll have silent-drop rows in your data and won't know until a data-quality audit reveals them weeks later.
- **Trusting the response body to confirm your write.** ST often echoes back your request body verbatim in the response, even when the field was silently dropped. Only a fresh GET on the resource is authoritative.
- **Mixing CREATE and PATCH in one CSV without an `action` column.** Operators inevitably mis-flag rows. Either split into two CSVs (creates.csv, updates.csv) or compute the action automatically based on whether the code exists.
- **Setting `useStaticPrices=true` on a fresh service via API.** It'll appear to work but won't stick. Either set in UI at creation time or design around dynamic pricing.
- **Skipping the reconcile step.** Your D1 mirror diverges, downstream reports go wrong, you blame the sync rather than the missing reconcile call.
