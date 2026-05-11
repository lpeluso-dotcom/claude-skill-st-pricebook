# Incidents & Lessons Learned

> A dated log of pricebook-adjacent incidents from production work, with root cause and
> the lesson encoded back into this skill. Dates are when the incident occurred or was discovered;
> some lessons reference earlier root-cause activity. Tenant-identifying details have been sanitized.

If you're using this skill in production, **adding your own incidents here is one of the most valuable contributions you can make**. Future-you (and the community) will thank you.

---

## 2026-04-07 — Tenant data-quality baseline audit

**What happened:** First systematic audit of an established multi-trade ST tenant. Surfaced multiple data-quality hotspots that had accumulated over years of organic pricebook growth.

**Findings:**
- 87 active equipment items with `cost > 0` and `price = 0` → ~$146K of cost exposure without a billed-revenue offset
- 341 services with `useStaticPrice = true` and `price = 0` → at risk of billing at $0
- 7,856 materials with no category assigned → poor findability in the UI
- 4,726 materials with no primary vendor → broke auto-purchasing and vendor performance reporting
- 26 OEM consumable code conflicts (`COL-*`, `COMP-*`, `DMS-*`) → SKU codes colliding across categories

**Lesson encoded:** Data-quality audits should be the **first step** before any pricebook automation. The audit queries in [`api-reference.md`](api-reference.md) and [`cost-and-pricing.md`](cost-and-pricing.md) are the ones that surfaced these findings.

---

## 2026-04-13 — D1 staleness discovery

**What happened:** During an audit, discovered the D1 pricebook mirror was 14–28 days stale: `pb_services` and `pb_equipment` had `synced_at` 28 days old; `pb_materials` was 14 days old. Live ST data was current.

**Root cause:** The nightly cron at `15 2 * * *` was paginating the ST API successfully but **not updating `updated_at` on rows where no fields had changed**. The `lastSyncedAt` column existed but wasn't being touched. The cron looked successful in the run log but was effectively a no-op for unchanged rows. **Compounding:** pricebook tables had been quietly excluded from the every-2-hour transactional sync because they're 10K rows and dominated the runtime — only the once-nightly cron touched them, and even that wasn't refreshing the staleness marker correctly.

**Lesson encoded:**
- Pricebook D1 mirror is opt-in for nightly cron — many teams gate it out of higher-frequency syncs. If yours does, your mirror can be days stale by design.
- Before any pricebook fix work, run a manual full rebuild: `POST /api/sync-full?days=0`.
- Track `synced_at` separately from `modified_on` — they answer different questions ("when did this row last get touched by sync" vs "when did this row's ST data last change").

See [`api-reference.md`](api-reference.md) "Sync architecture" section.

---

## 2026-04-17 — Motor replacement: cost field silent drop (first sighting)

**What happened:** Created 4 new motor-replacement service SKUs via POST to ST. All 4 returned 200 OK. Read-back showed all 4 services had `cost = 0` despite POST bodies including `cost: 330`, `cost: 133.10`, etc.

**Root cause:** ST silently drops the `cost` field on service POSTs. The 200 OK response body even echoed the cost values back, masking the failure.

**Detection:** A manual spot-check (was going to verify the prices looked right; noticed the cost was zero).

**Lesson encoded:**
- **Never trust the POST response body to confirm a write.** ST often echoes back your request fields even when they were silently dropped. Only an independent GET on the resource is authoritative.
- The 4 new services in this incident were left at `cost = 0` and worked under dynamic pricing (cost flowed in from linked materials).
- For services that need an explicit static cost, ST UI is the only path.

Captured in [`silent-fail-catalog.md`](silent-fail-catalog.md) as the canonical "cost drops on services POST" entry. Re-verified 2026-05-08 (see below).

---

## 2026-04-22 — Tool catalog errata: 10 corrections found in cold-context plan review

**What happened:** During an architecture review of an MCP tool catalog for ST, a cold-context plan-review pass surfaced 10 documented patterns that didn't match live ST behavior.

**Findings (the 10):**

1. Dispatch capacity is POST not GET
2. Tech assignments have no generic PATCH — must two-call unassign-all then assign-new via appointment-assignments endpoints
3. Appointment hold is a dedicated `/{id}/hold` sub-route, not PATCH
4. Customer notes API is append-only, no PATCH (use `add_customer_note` naming)
5. "Lead attribution" isn't a real ST object — it's a call POST with campaignId + customerId + leadCallId stitched in
6. `book_job` requires `campaignId` (ST rejects without)
7. Estimate `status=Sold` requires `soldBy` technicianId
8. `create_recurring_service` requires an Active membership
9. Configurable equipment vocabulary is "variations" / `isConfigurableEquipment` — not "variants"
10. `/payments/{id}` returns a payment object, not a status — invoice balance lives on the invoice

**Lesson encoded:** Plan-review by a cold-context agent (one that hasn't been deep in the implementation) catches drift between documented behavior and live API. The pattern is broadly applicable: before shipping a new write tool, have an independent reviewer cross-check the doc against a live test against the actual API surface — not the mock used in unit tests.

---

## 2026-04-28 — Wave-1 mapper regression (silent revert by merge)

**What happened:** A long-lived feature branch was merged into main, and the merge resolution kept the older version of `gate-st.js` — silently reverting 19 mapper functions that had been added two weeks earlier on a different branch. Production deploys started writing incomplete pricebook columns.

**Root cause:** The feature branch had forked from before the wave-1 mapper merge, and its conflict resolution favored the older file. Tests in `gate-st-mappers.test.ts` still referenced the wave-1 mapper outputs, so the test suite started failing — but `tsc --noEmit` wasn't part of the preflight gate, and the failing tests weren't blocking deploys.

**Detection:** A week later, the next pricebook audit showed columns that should have been populated were empty for new rows.

**Recovery:** Re-applied the wave-1 mapper portion of the missing commit to the current `gate-st.js`. Confirmed all 19 mapper tests passed against the restored implementation. Migration `0010_*` was already applied (it only changed schema, not the JS that writes to it) so no DB recovery needed.

**Lesson encoded:**
- `tsc --noEmit` (or your language's equivalent) belongs in your preflight gate. It catches silent reverts where the new code compiles but tests assume code that was removed.
- For long-lived branches with significant divergence, manual rebase + manual conflict review is safer than a `merge -X theirs`-style resolution. If you can't tell which side of a conflict is right, ask the original author.

Not directly a pricebook API gotcha — but it illustrates the broader pattern: silent failures cluster around codepaths where success/failure isn't loudly signaled.

---

## 2026-05-08 — `useStaticPrice[s]` is unflippable post-create

**What happened:** Investigation around freshly-POSTed services confirmed that after a service is created (without `useStaticPrices` set, since POST drops it), you **cannot** flip it to `useStaticPrices=true` via PATCH. Both `mcp-servicetitan st_patch_service` and direct API PATCH returned success, but `pre_hash === post_hash` on `{cost, price, useStaticPrice, useStaticPrices}`.

**Verified across:** 4 newly-created services on 2026-05-08 (`WT-PREFILT-INS`, `HALO5-INS`, `WT-LNDRYPRO-INS`, `WT-BUNDLE-SETUP`).

**Workaround:**
- For services that need static pricing, set the toggle in ST UI immediately after API create
- OR design the import to use dynamic pricing (most existing services run on dynamic anyway)

**Lesson encoded:**
- API can land the structural pieces of a new service (code, displayName, description, hours, isLabor, taxable, account, paysCommission, categories, serviceMaterials, serviceEquipment) but **UI is required for the pricing toggle + cost**
- Design imports accordingly. Don't promise "fully via API" if static pricing is required.
- The correct field name is `useStaticPrices` (plural). Singular `useStaticPrice` is silently dropped everywhere — that gotcha has been around since at least 2026-04.

Captured in [`silent-fail-catalog.md`](silent-fail-catalog.md) and [`cost-and-pricing.md`](cost-and-pricing.md).

---

## 2026-05-09 — Multi-track HVAC pricebook cleanup (parallel sub-audits)

**What happened:** A large-scale HVAC pricebook reorganization that consolidated 1,241 services into a new 103-category tree and ran five parallel audit tracks (categories, costs, configurability, discounts, pricing model).

**Audit findings:**
- 22 configurable equipment items, bundled into UX flows
- 52 materials with zero-cost exposure
- 341 services priced at $0 with `useStaticPrices=true` (billing risk on Repair + Service buckets)
- Diagnostic-fee discount-code fallback pattern needed (member discount auto-fail)

**Lesson encoded:** Bulk pricebook reorgs benefit from parallel cold-context sub-audits — one agent each for:

- Category tree consistency
- Cost / margin exposure
- Configurability and variant relationships
- Discounts and membership pricing
- Static vs dynamic pricing audit

Each agent reads only its slice of the data and produces a focused report. Running them in parallel completes the audit in roughly the time of the longest single track, instead of serial summation. The merged report is the operator's punch list.

Not a write incident — but a positive pattern worth capturing.

---

## 2026-05-10 — Estimate status PATCH silent-fail (bulk dismissal)

**What happened:** Building a bulk-dismissal pipeline for ~400 stale Open estimates. The MCP tool `update_estimate_status` returned HTTP 200 for the first 5 test PATCHes (each with `{status: "Dismissed"}` in the body). UI spot-check showed all 5 estimates were still Open.

**Root cause:** `PATCH /sales/v2/.../estimates/{id}` accepts `{status: ...}` in the body and returns 200, but **silently ignores the field**. `modifiedOn` doesn't even bump. The tool wrapper had been wired since the MCP server's first release and was never end-to-end tested — unit tests mocked the upstream and covered only the dryRun → token flow, not actual ST acceptance.

**Detection:** Operator spot-check of the ST UI after the 5-row test batch.

**Workaround:** Switched to the dedicated action endpoints:
- `PUT /sales/v2/{tenantId}/estimates/{id}/sell` (body `{soldBy: <techId>}`)
- `PUT /sales/v2/{tenantId}/estimates/{id}/dismiss` (body `{}`)
- `PUT /sales/v2/{tenantId}/estimates/{id}/unsell` (body `{}`)

Final run: 258 stale Plumbing+Electrical estimates dismissed across 250ms-spaced PUT calls. 0 errors. The HVAC stale-estimate set (700+ rows) was scoped to a separate plan.

**Cost:** ~2 hours building harness + permission rules + per-row results CSV around a broken primitive.

**Lessons encoded:**
- **End-to-end verify every write tool against the real upstream before bulk-running.** Unit tests with mocked upstreams catch the framework but not the API mismatches.
- **Dedicated action endpoints are not optional.** When the API design has `PUT /{id}/sell`, `PUT /{id}/dismiss`, `PUT /{id}/unsell`, that's the path. `PATCH /{id}` with a status field is a trap that returns 200.
- **Write tools wrapping a generic PATCH are higher-risk than tools wrapping a specific action.** A generic `update_estimate_status` tool can route to PATCH (broken) or PUT (working) — encode the routing inside the tool, not at the call site.

Captured in [`silent-fail-catalog.md`](silent-fail-catalog.md) and [`estimate-and-proposal-templates.md`](estimate-and-proposal-templates.md).

**Audit-trail observation:** The 258 dismissals produced ZERO audit_log rows on the proxy because the write gateway only auto-audits **reads**, not writes — writes have to call `audit()` explicitly at the caller. The full bulk run had no per-row trail in the audit table. Lesson: if your gateway / wrapper auto-audits one direction, double-check the other. Don't assume parity.

---

## Patterns across incidents

Looking at the incident log as a whole, three patterns dominate:

### 1. "200 OK doesn't mean the write stuck"

Appeared in: 2026-04-17 (cost drop), 2026-05-08 (useStaticPrices), 2026-05-10 (estimate status), and at least half a dozen smaller drops captured in the catalog.

**Defense:** read-after-write hash compare, on every write. See [`write-safety.md`](write-safety.md).

### 2. "The wrapper looks fine but doesn't verify against reality"

Appeared in: 2026-05-10 (MCP tool wired to broken PATCH), 2026-04-28 (mapper regression with tests not gating deploys).

**Defense:** end-to-end smoke tests against the live upstream as a release gate. Mocks lie.

### 3. "The data was stale and nobody noticed"

Appeared in: 2026-04-13 (D1 staleness), 2026-04-28 (wave-1 mapper drift).

**Defense:** explicit freshness checks — `synced_at` column, daily heartbeats, drift detection — and the discipline to run them before relying on the data.

---

## How to contribute an incident

When you encounter a new gotcha or recover from an incident:

1. Add an entry above using the same headings (What happened / Root cause / Detection / Workaround / Lesson encoded).
2. Sanitize tenant-specific data — replace real SKU codes, business unit IDs, etc. with `<example-code>` placeholders.
3. Date-stamp the entry. Future readers will use dates to gauge whether the lesson is still current.
4. If the incident reveals a new silent-drop field, also add a row to [`silent-fail-catalog.md`](silent-fail-catalog.md).
5. Submit as a PR. Even one-paragraph entries are valuable.
