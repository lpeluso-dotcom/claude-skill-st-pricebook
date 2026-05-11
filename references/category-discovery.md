# Category Discovery & Management

> Walking the ST pricebook category tree, building a parent-child map,
> renaming (UI only), deactivating, and bulk reorganization patterns.

Categories are how the ST pricebook organizes services, materials, and equipment for both UI navigation and reporting. Most established tenants have a couple hundred categories arranged in 2–4 levels of hierarchy.

## API endpoint

```
GET https://api.servicetitan.io/pricebook/v2/tenant/{tenantId}/categories?pageSize=200
```

Response shape:

```json
{
  "data": [
    {
      "id": 12345,
      "name": "Round Duct Pipe",
      "parentId": 6789,
      "active": true,
      "businessUnitId": null
    },
    ...
  ],
  "hasMore": false
}
```

Top-level categories have `parentId: null`. Use `parentId` to build the tree.

---

## Building the tree

```js
async function buildCategoryTree() {
  const all = [];
  let cursor = 1;
  while (true) {
    const resp = await stGet(`/categories?page=${cursor}&pageSize=200`);
    all.push(...resp.data);
    if (!resp.hasMore) break;
    cursor++;
  }
  // Build parent → children map
  const byParent = new Map();
  for (const c of all) {
    const p = c.parentId ?? 'root';
    if (!byParent.has(p)) byParent.set(p, []);
    byParent.get(p).push(c);
  }
  return { all, byParent };
}
```

For traversal:

```js
function descendants(byParent, rootId) {
  const out = [];
  const stack = [...(byParent.get(rootId) ?? [])];
  while (stack.length) {
    const c = stack.pop();
    out.push(c);
    stack.push(...(byParent.get(c.id) ?? []));
  }
  return out;
}
```

---

## Mirroring to D1

Recommended schema:

```sql
CREATE TABLE pb_categories (
  id              INTEGER PRIMARY KEY,
  name            TEXT,
  parent_id       INTEGER,                  -- NULL for top-level
  active          INTEGER,
  business_unit_id INTEGER,
  synced_at       TEXT,
  -- Convenience: flattened ancestry, computed at sync time
  full_path       TEXT                       -- "HVAC > Indoor > Air Handlers"
);

CREATE INDEX idx_pb_categories_parent ON pb_categories(parent_id);
CREATE INDEX idx_pb_categories_active ON pb_categories(active);
```

The `full_path` column is denormalized for ergonomics — recompute it on every sync from the parent chain.

---

## Useful queries

### Top-level categories

```sql
SELECT id, name, active
FROM pb_categories
WHERE parent_id IS NULL
ORDER BY name;
```

### All categories under a parent (HVAC, for example)

```sql
WITH RECURSIVE descendants(id, name, parent_id, full_path) AS (
  SELECT id, name, parent_id, name FROM pb_categories WHERE id = ?
  UNION ALL
  SELECT c.id, c.name, c.parent_id, d.full_path || ' > ' || c.name
  FROM pb_categories c JOIN descendants d ON c.parent_id = d.id
)
SELECT id, full_path FROM descendants ORDER BY full_path;
```

### Leaf categories (no children)

```sql
SELECT c.id, c.name, c.full_path
FROM pb_categories c
WHERE c.active = 1
  AND NOT EXISTS (
    SELECT 1 FROM pb_categories child WHERE child.parent_id = c.id AND child.active = 1
  )
ORDER BY c.full_path;
```

### Uncategorized item count by trade

(Assumes you've added a `trade` column on your pb_* tables, or use a category-name pattern match.)

```sql
SELECT 'services' as type, COUNT(*) as uncategorized
FROM pb_services WHERE active = 1 AND (category_name IS NULL OR category_name = '')
UNION ALL
SELECT 'materials', COUNT(*)
FROM pb_materials WHERE active = 1 AND (category_name IS NULL OR category_name = '')
UNION ALL
SELECT 'equipment', COUNT(*)
FROM pb_equipment WHERE active = 1 AND (category_name IS NULL OR category_name = '');
```

### Empty categories (candidates for cleanup)

```sql
SELECT c.id, c.name, c.full_path
FROM pb_categories c
WHERE c.active = 1
  AND NOT EXISTS (SELECT 1 FROM pb_services s WHERE s.category_name LIKE '%' || c.name || '%' AND s.active=1)
  AND NOT EXISTS (SELECT 1 FROM pb_materials m WHERE m.category_name LIKE '%' || c.name || '%' AND m.active=1)
  AND NOT EXISTS (SELECT 1 FROM pb_equipment e WHERE e.category_name LIKE '%' || c.name || '%' AND e.active=1)
ORDER BY c.full_path;
```

(LIKE matching on flattened `category_name` is a heuristic — for precision, query the actual `categories: [<int>]` array stored on each item record.)

---

## Renames are UI-only

The big gotcha: `PATCH /categories/{id}` with `{name: "New Name"}` returns 200 OK and **silently does nothing**. The category name doesn't change.

Verified on multiple categories 2026-04-27.

**Workaround:** rename in the ST UI (Pricebook → Categories → [category] → Edit). Then re-sync your D1 mirror.

If you discover this has been fixed in your tenant (ST does occasionally patch gotchas), please open a PR against [`silent-fail-catalog.md`](silent-fail-catalog.md) with your verification date.

---

## What DOES work via PATCH

- **`active: true/false`** — toggling category active state works via PATCH (verified 2026-04-27, used in production deactivations during a tenant consolidation).
- **`businessUnitId`** — assigning categories to business units works.
- **`parentId`** — moving a category to a different parent works (cascades — all child items move with it).

So the path to "rename" a category is actually a three-step:

1. Create a new category with the desired name (POST).
2. Move every child item / category to the new parent.
3. Deactivate the old empty category (PATCH `active: false`).

Annoying, but tractable for one-off renames. For bulk renames, the UI is faster.

---

## Bulk consolidation pattern (verified 2026-04-27)

Real production pattern for collapsing 5 duplicate categories into 1:

```js
async function consolidateCategories(sourceIds, targetId) {
  // 1. For each source category, find all items in it
  const items = [];
  for (const srcId of sourceIds) {
    const services = await searchD1(`SELECT id FROM pb_services WHERE categories_json LIKE ?`, [`%${srcId}%`]);
    const materials = await searchD1(`SELECT id FROM pb_materials WHERE categories_json LIKE ?`, [`%${srcId}%`]);
    const equipment = await searchD1(`SELECT id FROM pb_equipment WHERE categories_json LIKE ?`, [`%${srcId}%`]);
    items.push(
      ...services.map(s => ({ type: 'services', id: s.id, srcId })),
      ...materials.map(m => ({ type: 'materials', id: m.id, srcId })),
      ...equipment.map(e => ({ type: 'equipment', id: e.id, srcId })),
    );
  }

  // 2. For each item, GET → replace source category ID with target → PATCH
  for (const item of items) {
    const live = await stGet(`/${item.type}/${item.id}`);
    const newCategories = live.categories
      .map(c => c.id)
      .filter(id => id !== item.srcId)
      .concat([targetId]);
    await stPatch(`/${item.type}/${item.id}`, { categories: newCategories });
    // verify-after-write: read back, confirm categories contains targetId, doesn't contain srcId
  }

  // 3. Deactivate the now-empty source categories
  for (const srcId of sourceIds) {
    await stPatch(`/categories/${srcId}`, { active: false });
  }

  // 4. Reconcile D1
  await stPost('/api/sync-full?table=pb_categories');
}
```

Real example: a tenant collapsed 3 duplicate "Push-To-Connect Fittings" categories (IDs `62184739`, `62291289`, `62289918`) into one canonical category. After moving items and deactivating, the canonical category remained and the three duplicates dropped out of the UI dropdown.

---

## Categories on items: the integer array

Items reference categories as a **bare integer array** at the body root:

```js
// RIGHT
await stPatch('/services/123', { categories: [456, 789] });

// WRONG — object array shape returns 400 deserialization error
await stPatch('/services/123', { categories: [{ id: 456 }, { id: 789 }] });

// WRONG — singular categoryId field is silently dropped (POST), and not a valid PATCH field
await stPatch('/services/123', { categoryId: 456 });
```

On reads, the category comes back as objects (`[{ id: 456, name: "..." }]`); on writes, it's integers only. Asymmetric.

An item can belong to multiple categories (uncommon but valid).

---

## Category-related data quality alarms

Run these periodically:

### Items with categories that don't exist (orphans)

```sql
SELECT s.id, s.code, s.name, s.category_name
FROM pb_services s
LEFT JOIN pb_categories c ON c.name = s.category_name
WHERE s.active = 1 AND c.id IS NULL
LIMIT 50;
```

If your sync resolves `categories: [<int>]` to a `category_name` string, mismatches can mean: (a) the category was renamed in the UI but you haven't re-synced, or (b) the category was deactivated and items weren't migrated off it first.

### Categories with very large item counts (consider splitting)

```sql
SELECT c.id, c.name, COUNT(s.id) as service_count
FROM pb_categories c
LEFT JOIN pb_services s ON s.category_name LIKE '%' || c.name || '%' AND s.active = 1
WHERE c.active = 1
GROUP BY c.id, c.name
HAVING COUNT(s.id) > 100
ORDER BY service_count DESC;
```

Categories with 100+ items are usually candidates for sub-categorization — they overwhelm the UI dropdown and hurt findability.

### Inactive categories with active items still pointing at them

```sql
SELECT c.id, c.name, COUNT(s.id) as orphan_services
FROM pb_categories c
JOIN pb_services s ON s.category_name LIKE '%' || c.name || '%' AND s.active = 1
WHERE c.active = 0
GROUP BY c.id, c.name
HAVING COUNT(s.id) > 0;
```

These are usually leftovers from incomplete consolidations. Move the items to an active category before deactivating any further.

---

## Anti-patterns

- **PATCHing `name` on a category and trusting the 200 OK.** Silently no-ops. UI-only.
- **Deactivating a category without first checking for active items.** The items become "uncategorized" in the UI and harder to find.
- **Using LIKE-matching on `category_name` strings.** Fast for exploratory queries, but `"HVAC"` matches `"HVAC > Maintenance"` and `"Plumbing > HVAC Workaround"`. For precision, query the `categories` integer array on the item record.
- **Building category-tree views by chaining LEFT JOINs.** Use the recursive CTE pattern instead — it scales cleanly to arbitrary depth.
- **Treating top-level "(uncategorized)" as a real category.** Some pricebook views label items with no category as "(uncategorized)" — that's a UI label, not a real category record. Don't try to PATCH it.
