# Governed Ingest: One Door for New Pricebook Items

> How to stop recurring creation defects by sending every new material, equipment, and service
> through one pipeline with gates. The same gates are then used to clean up the existing book.
> Pattern from a multi-trade production tenant, 2026-09-26 to 2026-09-27.

Most pricebook defects are not typos. They are the same few mistakes, made again each time someone creates an item by hand, from a spreadsheet, or from a script. A creation audit on one tenant ranked 12 recurring defects: labor cost missing, items taxed that should not be, no vendor part, no category, per-100 units of measure priced per unit, kits that doubled a system material, descriptions copied from the display name, no photo or spec sheet, and more. Fixing each row by hand did not stop them. A provider sync also reverted some of the fixes.

The fix was structural. Every new item enters through **one door**, and that door has gates.

---

## Rule 1: one door

Every new item, whatever its source, goes through the same pipeline. Typical sources:

- The **AP invoice gap list**: parts bought and billed that are not in the book
- **Vendor quotes** sent to a pricing mailbox
- **Build specs** written during a pricebook session (for example, "add the tankless water heater ladder")
- **Vendor equipment sheets**

No side doors. A script that POSTs straight to `/materials` is a side door. If a direct edit is approved as an exception, **register it afterwards** (see [Out-of-pipeline edits](#out-of-pipeline-edits)) so that the drift watch still covers it.

---

## The gates

```
reconcile → normalize → policy → enrich → approve → write → verify → watch
```

Each gate either passes a row, or **holds** it with a reason code (`held_duplicate`, `held_cost_source`, `held_copy`, and so on). A held row never reaches the write gate. It waits for a person or for a later pass.

### 1. Reconcile: is it already in the book?

- Match against the **live API**, not only your mirror. Mirrors keep ghost codes after renames (see [`incidents-and-lessons.md`](incidents-and-lessons.md), 2026-04-13), so a code-only match against a stale mirror creates duplicates.
- Match on vendor part number as well as code. Normalize both (case, spaces, dashes). Be careful with leading digits: size-prefixed parts (`3/4` versus `1`) collide if you strip them.
- Near-duplicates across vendors are common. The same ¾" PEX coupling can exist once per distributor. Hold these rows for a person to decide.

### 2. Normalize: shape the row

- Code, display name, and category follow your naming standard.
- **Unit of measure.** Check whether the vendor price is per each, per 100 (`C`), or per 1,000 (`M`). A per-100 cost on an item that kits consume one at a time inflates every kit that links it. Convert the unit, or set kit quantities as fractions of the pack. For example, 15 ft of a 50 ft coil is `0.3`.
- Description lint (see [Descriptions](#descriptions)). A row that fails is held as `held_copy`.

### 3. Policy: apply the defaults from a table, not from memory

Keep one policy table, keyed by trade and item type, that sets the fields people usually forget:

| Field | Example policy |
|---|---|
| `taxable` | Pick one rule for your tenant and apply it everywhere. The origin tenant's rule is "nothing is taxed". Reject a per-row override, or the policy is not really one. |
| GL `account` / cost of sale | One rule per item type. The origin tenant uses Revenue / Materials on every material in every trade. |
| Service labor cost | `cost = hours × <labor rate>` on new creates. **Never change `hours` to make the numbers work.** Hold rows whose hours are not on your step (for example, 0.25 h). |
| `isLabor` | `true` on install and repair services that carry a kit. |
| Static or dynamic, member price, add-on | From the table. Most services stay dynamic. See [`cost-and-pricing.md`](cost-and-pricing.md). |

**Cost source is required.** Every cost carries its source (`invoice`, `quote`, `list`, `web`), the reference (invoice or quote number), the date, and the unit. Useful rules:

- A quote older than **90 days** is not a cost source. If an operator overrides this, record the override in the manifest.
- MSRP or list price may go live if it is tagged with its source.
- A web price is allowed only when the vendor catalog has none.

### 4. Enrich: photo, spec sheet, clean copy

- Manufacturer photo, spec PDF, and a clean description.
- **Check the photo before you reuse one.** ST has generic stock images that are shared by more than a thousand items. If you copy an image from a source material to a service, you may copy the stock placeholder. Compare the asset path against a list of known shared assets first.
- `PATCH {assets}` accepts the path of an asset already hosted by ST and rehosts it on the target. This lets you reuse a sibling item's photo or series spec sheet without uploading it again. Verified 2026-09-27; one item returned 400 and wrote nothing, so read back every row.

### 5. Approve: a person signs off on the batch

- Show each row next to its current state (or "new"), with approve, edit, hold, or reject for each row.
- Batch sizes: no more than 5 rows for a first canary, and about 100 rows for backfills.
- A confirmation token is bound to the approved rows. See [`write-safety.md`](write-safety.md).
- Deactivations, deletions, outbound quote requests, and spending money stay with a named person, even inside an approved batch.

### 6. Write: guarded and complete

- **Before-state freshness.** Read the live item no more than about 60 minutes before the write. Compare `modifiedOn` right before the PATCH. If it moved, re-plan the row.
- **Full-array PATCH** for `serviceMaterials` and `serviceEquipment`. The array is replaced, not merged. See [`service-links.md`](service-links.md).
- **Create inactive.** Switch a new item on only after it has been read back, and, for tiered sets, after a price check that Good < Better < Best still holds.
- Route equipment creates to the **pricebook** (`/pricebook/v2/tenant/{tenantId}/equipment`). The `/equipmentmanagement/` API is for equipment installed at customer locations. A wrapper that routes `equipment.create` there creates the wrong kind of record.

### 7. Verify: read back and diff everything

- Diff the full model before and after. **Only the intended fields and `modifiedOn` may change.** For example, a PATCH of `{account}` on a material should change only `account`, `generalLedgerAccountId`, and `modifiedOn`.
- **Also read back the linked estimate templates.** Kit edits can reprice templates without a template write. See [`estimate-and-proposal-templates.md`](estimate-and-proposal-templates.md).
- Stamp the ledger or registry row. **Key the stamp by item code, taken from the pre-write read.** Do not key it by ST id, and do not take the code from the caller's request. One pipeline keyed registry updates by ST id while the registry held codes, so no patch was ever stamped.

### 8. Watch: detect drift after the write

Provider and catalog syncs keep feeding cost. They can also revert structure. Items whose `source` is the provider catalog (with `externalId` set) had kit fixes reverted by a sync once. A nightly watch job checks seeded rules and alerts on a mismatch:

| Rule type | Example |
|---|---|
| **Must not contain** | Service kit `<code>` must not link material `<skuId>` (a link removed on purpose) |
| **Subaccount cost ratio** | Alert when a vendor subaccount cost is at least 3× the item cost (a per-100 price leaking in) |
| **UOM trap** | Alert on UOM `C` or `M` with a whole-number kit quantity |
| **Policy drift** | `taxable = true`, labor cost ≠ hours × rate, wrong GL account |

Prove the watch works by seeding a known drift and checking that it alerts.

---

## Descriptions

Descriptions are the biggest backfill. In the origin tenant, every active service failed the voice rule when it was first checked. Keep the rule mechanical so that a linter can enforce it:

- **One shape per item type.** Services, estimate-template summaries, and proposal descriptions use future tense, with the company as the subject ("`<Company>` will install …", then "This includes …", then any exclusions). Materials and equipment are plain present-tense spec sentences and never use "will".
- A reading-grade limit (the origin tenant uses Flesch-Kincaid ≤ 10). Leave out model numbers, units, the fixed company opener, and trade terms when you score the grade, or they inflate it.
- No HTML, no prices, no marketing words, at most three sentences.
- Check tier sets and size runs **as a set**. They should read alike except for the facts that differ.
- **Keep drafting and checking separate.** One model drafts, and a separate checker returns PASS or FAIL with the rule number only. If the checker suggests wording, it ends up steering the drafter. Choose the drafting model with a small side-by-side test on live rows.
- "Description = display name" is not a default. Send those rows to enrichment for a draft.

---

## Out-of-pipeline edits

An operator will sometimes approve a direct edit, such as an urgent kit fix. When that happens:

1. Write it with the same guards: `modifiedOn` check, full-array PATCH, and read-back diff.
2. Save a manifest: rows, before-state, cost source, and reason.
3. Register the rows in the pipeline's ledger or registry, flagged as edited, so that the drift watch covers them.
4. Add any must-not-contain rules the edit implies.

Without step 3, the watch skips these rows, and they are exactly the rows most likely to be reverted by a sync.

---

## Review of pipeline code and batches

- Use more than one independent reviewer on each goal or batch, working blind, and require PASS verdicts from all of them.
- A reviewer that cannot read the files (for example, one running in a sandbox without file access) returns "unproven", not "defect". Answer those findings in writing. Do not let them block the pipeline forever.
- Use a circuit breaker. If the same goal fails verification three times in a row, stop and ask a person.

---

## See also

- [`write-safety.md`](write-safety.md): dryRun, confirmation token, read-after-write hash compare
- [`batch-operations.md`](batch-operations.md): idempotency, retries, resume
- [`service-links.md`](service-links.md): full-array kit edits and provider-sourced reverts
- [`cost-and-pricing.md`](cost-and-pricing.md): static and dynamic pricing, and why `useStaticPrices: null` on the public API is not proof of dynamic pricing
- [`incidents-and-lessons.md`](incidents-and-lessons.md): the 2026-09 entries that led to this pattern
