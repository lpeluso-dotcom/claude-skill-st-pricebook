# Service Links: `serviceMaterials` & `serviceEquipment`

> How services reference the materials and equipment they consume — adding, removing, and
> detecting orphan links. The "no DELETE endpoint" gotcha and the full-array PATCH pattern.

A service in ST often "consumes" linked materials and/or equipment when invoiced. Under dynamic pricing, this is also how cost flows into the service — the service's effective cost = sum of linked-material costs × markup rules.

## The shape

Service objects carry two link arrays at the root:

```json
{
  "id": 12345,
  "displayName": "Replace Blower Motor",
  "serviceMaterials": [
    { "skuId": 87654321, "quantity": 1 },
    { "skuId": 87654322, "quantity": 1 }
  ],
  "serviceEquipment": [
    { "skuId": 11223344, "quantity": 1 }
  ]
}
```

`skuId` is the material/equipment ID (not the SKU code). `quantity` is a decimal — usually `1` for materials, but can be fractional for consumables (e.g. `0.5` for half a roll of solder).

On reads, ST may also include enriched fields (`materialName`, `materialCost`) per entry — these are informational and not writable.

---

## Adding a link

There is no `POST /services/{id}/materials/{matId}` endpoint. To add a link, you GET the service, append to the array, and PATCH the whole array back:

```js
async function addServiceMaterial(serviceId, materialId, quantity = 1) {
  const service = await stGet(`/services/${serviceId}`);
  const existing = service.serviceMaterials ?? [];

  // Idempotency: if the link already exists, just update quantity
  const dupIdx = existing.findIndex(m => m.skuId === materialId);
  const newArray = dupIdx >= 0
    ? existing.map((m, i) => i === dupIdx ? { ...m, quantity } : m)
    : [...existing, { skuId: materialId, quantity }];

  await stPatch(`/services/${serviceId}`, { serviceMaterials: newArray });

  // Verify
  const after = await stGet(`/services/${serviceId}`);
  const persisted = after.serviceMaterials?.find(m => m.skuId === materialId);
  if (!persisted) throw new Error(`Link to material ${materialId} silently dropped`);
}
```

Same pattern for `serviceEquipment`.

---

## Removing a link (the "no DELETE" gotcha)

There is no `DELETE /services/{id}/materials/{matId}` endpoint — it returns 404. The pattern is GET → filter → PATCH:

```js
async function removeServiceMaterial(serviceId, materialId) {
  const service = await stGet(`/services/${serviceId}`);
  const filtered = (service.serviceMaterials ?? []).filter(m => m.skuId !== materialId);

  // Sanity check: did we actually remove anything?
  if (filtered.length === service.serviceMaterials.length) {
    return { removed: false, reason: 'link did not exist' };
  }

  await stPatch(`/services/${serviceId}`, { serviceMaterials: filtered });

  // Verify
  const after = await stGet(`/services/${serviceId}`);
  const stillThere = after.serviceMaterials?.find(m => m.skuId === materialId);
  if (stillThere) throw new Error(`Material ${materialId} link silently un-removed`);

  return { removed: true };
}
```

**Important:** PATCH on `serviceMaterials` has **full-array replace semantics**, not delta semantics. Sending `{ serviceMaterials: [...] }` replaces the entire array on the service. If you send `[{skuId: 999, quantity: 1}]` you've just nuked all the other links and replaced them with one to material 999. Always GET → modify → PATCH.

---

## Replacing an entire link set

Sometimes (e.g. during a CSV import) you want to declaratively set the full link set on a service:

```js
async function setServiceMaterials(serviceId, desiredLinks) {
  // desiredLinks: [{ skuId, quantity }, ...]

  // Optional pre-check: do all the materials exist?
  for (const link of desiredLinks) {
    const mat = await stGet(`/materials/${link.skuId}`);
    if (!mat || !mat.active) {
      throw new Error(`Material ${link.skuId} doesn't exist or is inactive`);
    }
  }

  await stPatch(`/services/${serviceId}`, { serviceMaterials: desiredLinks });

  // Verify by hashing the link set
  const after = await stGet(`/services/${serviceId}`);
  if (hashLinkSet(after.serviceMaterials) !== hashLinkSet(desiredLinks)) {
    throw new Error('serviceMaterials silently dropped during replace');
  }
}

function hashLinkSet(arr) {
  return JSON.stringify(arr.map(l => ({ skuId: l.skuId, quantity: l.quantity })).sort((a, b) => a.skuId - b.skuId));
}
```

---

## Detecting orphan links

An "orphan" link is a `serviceMaterials` or `serviceEquipment` entry pointing at a material/equipment that no longer exists (deactivated or deleted). ST doesn't proactively clean these up — they just sit there, and may cause invoice generation to silently skip the line item.

Detection via your D1 mirror:

```sql
-- Services with serviceMaterials pointing at deactivated/missing materials
SELECT s.id AS service_id, s.code, s.name,
       json_each.value AS material_link
FROM pb_services s, json_each(s.materials_json) AS json_each
LEFT JOIN pb_materials m ON m.id = json_extract(json_each.value, '$.skuId')
WHERE s.active = 1
  AND (m.id IS NULL OR m.active = 0)
LIMIT 100;
```

(SQLite syntax; adapt for Postgres / D1 dialect — D1 supports `json_each` and `json_extract`.)

Same query for equipment, against `pb_services.equipment_json` and `pb_equipment`.

Once found, the remediation is "GET the service, filter the bad link out, PATCH back" — same pattern as removing any link.

---

## Linking equipment to a configurable parent's materials

A bonus gotcha for configurable equipment: each variant equipment can carry its own `equipmentMaterialLinks` for install kits, accessories, etc. These follow the same array semantics as `serviceMaterials`.

```js
// Add a material to a configurable equipment variant's install kit
async function addEquipmentMaterial(equipmentId, materialId, quantity) {
  const eq = await stGet(`/equipment/${equipmentId}`);
  const existing = eq.equipmentMaterialLinks ?? [];
  const newArray = [...existing.filter(m => m.skuId !== materialId), { skuId: materialId, quantity }];
  await stPatch(`/equipment/${equipmentId}`, { equipmentMaterialLinks: newArray });
}
```

Same full-array PATCH semantics. Same need to verify after write.

---

## The cost-flow chain

Putting it together, the typical dynamic-pricing data flow for a service invoice:

```
Service (price=0, useStaticPrices=false)
  └─ serviceMaterials: [{skuId: 1, qty: 2}, {skuId: 2, qty: 1}]
       ├─ Material 1: cost=$10 → contributes $20
       └─ Material 2: cost=$5  → contributes $5
                                  ────────────
                                  Effective cost: $25
                                  Customer price = $25 × markup_rule
```

If `serviceMaterials` is empty on an active non-labor service, the cost-flow chain breaks — there's nothing to feed into the markup engine, and the service prices at the minimum (or fails invoice validation depending on tenant settings).

The audit query:

```sql
SELECT id, code, name, account, hours, is_labor
FROM pb_services
WHERE active = 1
  AND is_labor = 0                                          -- not a pure-labor service
  AND use_static_prices IS NOT 1                            -- expects dynamic pricing
  AND (materials_json IS NULL OR materials_json = '[]')     -- but has no material links
  AND (equipment_json IS NULL OR equipment_json = '[]')     -- nor equipment links
ORDER BY name;
```

These rows are the most likely "$0 invoice line" candidates.

---

## Anti-patterns

- **Treating PATCH on `serviceMaterials` as delta.** It's full-array replace. Always GET → modify → PATCH the whole array.
- **Appending to the array without dedup.** You'll end up with duplicate `{skuId, quantity}` entries that bill twice on the invoice.
- **Skipping the verify-after-write step.** Like every other ST write, the PATCH can silently drop. Read back and hash-compare.
- **Trying `DELETE /services/{id}/materials/{matId}`.** Returns 404 — the endpoint doesn't exist. GET → filter → PATCH is the only path.
- **Storing materials_json / equipment_json as denormalized strings in D1 without an index.** For any non-trivial orphan detection, you'll want the `json_each` virtual table or a proper join table.

---

## See also

- [`silent-fail-catalog.md`](silent-fail-catalog.md) — the silent-drop pattern that applies to these PATCHes
- [`cost-and-pricing.md`](cost-and-pricing.md) — how `serviceMaterials` carries cost into dynamic-priced services
- [`configurable-equipment-and-materials.md`](configurable-equipment-and-materials.md) — variant equipment with their own `equipmentMaterialLinks`
- [`batch-operations.md`](batch-operations.md) — bulk-applying link changes across many services
