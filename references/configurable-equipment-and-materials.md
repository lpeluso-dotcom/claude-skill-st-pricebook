# Configurable Equipment & Materials

> Vocabulary, mental model, and worked examples for configurable items in the ST pricebook.

ST uses the word "configurable" two ways. They're related but distinct, and conflating them is a common source of failed PATCHes and 400 errors on create.

## Configurable equipment (parent + variants)

**Configurable equipment** is ST's name for a parent equipment record with sibling variant equipment records linked underneath it. The classic use case: a single "22kW Standby Generator" parent with three variants (Natural Gas, Liquid Propane, Diesel) — same nameplate, different fuel-source SKUs.

### The data model

- The **parent** is an equipment record with `isConfigurableEquipment: true`.
- Each **variant** is a separate, independently-priceable equipment record.
- The **link** between parent and variants is managed entirely in the ST UI. The `variationsOrConfigurableEquipment` field on the parent (which reads back as an array of variant references) is **read-only** via the API.

### What works via the API

- Creating the parent with `isConfigurableEquipment: true` ✅
- Creating each variant as an independent equipment record ✅
- Reading the parent and seeing its variants in `variationsOrConfigurableEquipment` ✅
- Toggling the parent's `isConfigurableEquipment` flag back to `false` (which un-marks it as a parent) ✅

### What doesn't

- Adding or removing variants by PATCHing `variationsOrConfigurableEquipment` ❌ — silently dropped
- Setting `typeId` on the parent or any variant ❌ — silently dropped
- Linking a variant to a parent ❌ — UI-only (Pricebook → Equipment → [parent] → Configurable Equipment tab)

### Worked example: 22kW generator with three variants

```js
// 1. Create the parent
const parent = await stPost('/equipment', {
  code: 'GEN-22-PARENT',
  displayName: 'Example 22kW Standby Generator',
  description: 'Air-cooled 22kW standby generator',
  isConfigurableEquipment: true,
  cost: 0, price: 0,                      // parent is non-priceable; variants carry the pricing
  active: true, taxable: true,
  account: 'Revenue',
  primaryVendor: { vendorId: 333, cost: 0, active: true },
});

// 2. Create each variant as an independent equipment record
const variants = await Promise.all([
  stPost('/equipment', {
    code: 'GEN-22-NG',
    displayName: 'Example 22kW Generator (Natural Gas)',
    cost: 4200, price: 7900,
    brand: 'ExampleBrand',
    model: 'EX-22-NG',
    active: true, taxable: true,
    account: 'Revenue',
    primaryVendor: { vendorId: 333, cost: 4100, active: true },
  }),
  stPost('/equipment', {
    code: 'GEN-22-LP',
    displayName: 'Example 22kW Generator (LP)',
    cost: 4250, price: 7950,
    brand: 'ExampleBrand', model: 'EX-22-LP',
    active: true, taxable: true,
    account: 'Revenue',
    primaryVendor: { vendorId: 333, cost: 4150, active: true },
  }),
  stPost('/equipment', {
    code: 'GEN-22-DSL',
    displayName: 'Example 22kW Generator (Diesel)',
    cost: 4800, price: 8600,
    brand: 'ExampleBrand', model: 'EX-22-DSL',
    active: true, taxable: true,
    account: 'Revenue',
    primaryVendor: { vendorId: 333, cost: 4700, active: true },
  }),
]);

// 3. STOP — link parent to variants in the ST UI.
//    Pricebook → Equipment → GEN-22-PARENT → Configurable Equipment tab → add each variant
//    There is no API path for this step.

// 4. Verify: re-GET the parent. variationsOrConfigurableEquipment should now contain
//    references to the three variants you linked in the UI.
const after = await stGet(`/equipment/${parent.id}`);
console.log(after.variationsOrConfigurableEquipment.length === 3);  // true after UI step
```

### Common mistakes

- **Trying to inline-create variants under the parent.** No nested-create endpoint exists. Each variant is an independent record.
- **PATCHing `variationsOrConfigurableEquipment` to add a variant reference.** ST returns 200; nothing changes.
- **Using `isConfigurable` instead of `isConfigurableEquipment`.** Misspelled field, silently dropped. The correct field name is `isConfigurableEquipment`.
- **Expecting `typeId` to stick.** Equipment Type (Air Handler, Heat Pump, etc.) is read-only via the API. Set in UI.

---

## Configurable materials

ST does **not** have a dedicated "configurable materials" data model the way it does for equipment. Material configurability comes from two adjacent patterns:

1. **Variant codes** — separate material records with related SKU codes (e.g. `M-MTR-480` and `S83-125` representing different variants of "the same" motor). The relationship is informal — there's no field on either record pointing at the other.
2. **`serviceMaterials` linking** — a service can carry multiple `serviceMaterials` entries (each with its own `materialId` and `quantity`), so the "configurable" choice between variants happens at the service level. Which materials a service requires is the configuration; the service picks them at invoice time.

Most "configurable material" workflows in practice are just careful naming conventions plus disciplined `serviceMaterials` linking. There's no parent-child equipment-style relationship.

If you genuinely need variants with parent-child semantics, model it as equipment instead of materials — even non-equipment items (e.g. a "tool kit" bundle) can be modeled as configurable equipment with non-priceable variants.

---

## Material POST cheat sheet

The two non-obvious gotchas on material POST:

### 1. `displayName`, not `name`

```js
// WRONG — ST silently drops `name` on POST
await stPost('/materials', { name: 'Round Duct Pipe 6"', ... });
// → 200 OK, but the created material has displayName: null
//   (Read-back shows the row exists but has no visible name in the UI)

// RIGHT — use `displayName`
await stPost('/materials', { displayName: 'Round Duct Pipe 6"', ... });
// → 200 OK, displayName: "Round Duct Pipe 6\""
```

On PATCH, `name` works fine — only POST drops it.

### 2. `primaryVendor` is required and must be the singular-object shape

```js
// WRONG — vendors array silently dropped, then 400 + create rollback
await stPost('/materials', {
  displayName: 'Round Duct Pipe 6"',
  cost: 8.25,
  vendors: [{ vendorId: 222, cost: 7.50, isPrimary: true }],  // dropped
});
// → 400 "There must be exactly one vendor selected as Primary"
// → The Sku.Id ST reports in the error response will 404 on follow-up GET
//   because ST already rolled back the create

// WRONG — vendorPricingLinks array also silently dropped
await stPost('/materials', {
  displayName: 'Round Duct Pipe 6"',
  vendorPricingLinks: [{ vendorId: 222, cost: 7.50, isPrimary: true }],
});
// → Same 400 + rollback

// RIGHT — primaryVendor as a singular object at the body root
await stPost('/materials', {
  displayName: 'Round Duct Pipe 6"',
  cost: 8.25,
  price: 15.50,
  unitOfMeasure: 'EA',
  active: true,
  account: 'Revenue',
  primaryVendor: { vendorId: 222, cost: 7.50, active: true },
});
// → 200 OK, material created with vendor wired up
```

To attach **additional** vendors after creation, PATCH `vendorPricingLinks` on the existing material (PATCH accepts the array shape; POST doesn't). The first vendor in `vendorPricingLinks` with `isPrimary: true` becomes the de-facto primary.

### 3. Categories are integer arrays at the body root

```js
// WRONG — singular `categoryId` is silently dropped on POST
await stPost('/materials', { ..., categoryId: 456 });
// → 200, but the material is uncategorized

// WRONG — categories as object array returns 400 deserialization error (at least loud)
await stPost('/materials', { ..., categories: [{ id: 456 }] });
// → 400 "Cannot deserialize the current JSON object..."

// RIGHT — bare integer array
await stPost('/materials', { ..., categories: [456] });
// → 200, material assigned to category 456
```

Same rule applies to services and equipment.

---

## Configurable equipment decision tree

When should you use configurable equipment vs. just creating independent SKUs?

| Scenario | Use configurable equipment? |
|---|---|
| Same nameplate, different fuel source / size / color | Yes — parent + variants gives the operator one entry point in the UI |
| Bundled install kit (different parts depending on customer choice) | Maybe — variants make UI selection clean, but if pricing logic is complex, model it as services with `serviceMaterials` |
| Subtle SKU variants with same price/cost | No — just create them as siblings with related codes. Configurable equipment overhead isn't worth it for pure cosmetic variation. |
| Pricing depends on options (e.g. 2-ton vs 3-ton vs 4-ton) | Yes — variants let the operator pick the right size at sale time |
| Variants that need different vendors | Yes — each variant has its own `primaryVendor` |

If in doubt, create the items as independent siblings first. Promoting siblings to a parent + variants relationship later is straightforward (toggle `isConfigurableEquipment: true` on the chosen parent, link variants in UI). Going the other way is harder.

---

## What's in the data, not the schema

A few patterns that are real but aren't directly enforced by the API:

- **Code-naming convention.** Variants typically inherit a stem from the parent (e.g. `GEN-22-NG`, `GEN-22-LP`, `GEN-22-DSL` from `GEN-22-PARENT`). The API doesn't enforce this — it's a convention for human findability.
- **Active flag on the parent.** Most teams set the parent's `active: true` even though it's never sold directly — toggling the parent inactive often hides the variants in the UI even if the variants are individually `active: true`. Verify in your tenant before relying on either behavior.
- **Cost / price on the parent.** Usually `0`, since the parent is non-priceable. Setting them to anything else doesn't break anything but is visually confusing in the UI.

---

## See also

- [`silent-fail-catalog.md`](silent-fail-catalog.md) — all the silent-drop fields touched on this page
- [`service-links.md`](service-links.md) — `serviceMaterials` / `serviceEquipment` linking pattern
- [`api-reference.md`](api-reference.md) — full equipment / material response schemas
