# st-pricebook v0.1.0 — First public release

This is the initial public release of `st-pricebook`, a Claude Code skill capturing roughly six months of production ServiceTitan Pricebook v2 API work.

## What's included

- **SKILL.md** — overview, endpoints, critical safety rules, MCP tool map, and 7-step canonical update workflow
- **11 reference files** covering the full surface area:
  - `api-reference.md` — endpoints, schemas, OAuth, pagination, GL accounts, D1 mirror schemas, SQL queries
  - `silent-fail-catalog.md` — every field ST silently drops on POST or PATCH, dated and with workarounds
  - `write-safety.md` — the 5-phase safe-update pattern (preview → confirm → execute → verify → audit)
  - `excel-import.md` — bulk CSV/XLSX import with read-after-write verification
  - `configurable-equipment-and-materials.md` — variants, `primaryVendor` shape, `displayName` vs `name`
  - `cost-and-pricing.md` — dynamic vs static, the `useStaticPrices` tri-state, audit queries
  - `batch-operations.md` — idempotency, retries, partial failures, resumability
  - `category-discovery.md` — tree-walking, renames (UI-only), consolidation patterns
  - `service-links.md` — `serviceMaterials` / `serviceEquipment` array semantics, orphan detection
  - `estimate-and-proposal-templates.md` — internal `/app/api/` surface, read↔write asymmetries, cookie-injection auth
  - `incidents-and-lessons.md` — dated postmortems with lessons encoded back into the skill

## Highlights

**The silent-fail catalog** is the single most valuable piece. Every entry is dated, has a verification recipe, and includes the workaround. If you've ever PATCHed an ST pricebook item, received 200 OK, and quietly discovered the field didn't change — this catalog tells you why, when it was first seen, and what to do about it.

**The Excel/CSV import pattern** documents the full pipeline: read → validate → preview → confirm → execute → verify → log/recover. Resumability via checkpoint files, recovery from silent drops via the catalog, and reconciliation via your D1 mirror.

**The estimate-template internal API documentation** captures the surface area that **isn't** in `/sales/v2/`. The read↔write schema asymmetries (`skuReference` wrapping, `isAddon` case, `summary` ↔ `description`) are documented with the exact body shape verified against a live production tenant on 2026-05-05.

## What's NOT in v0.1

- Photo / asset upload patterns (different API surface)
- Membership pricing rules and discount fallback (separate skill candidate)
- Inventory tracking and material ordering (out of scope for pricebook)
- Reporting and dashboard patterns (separate concern)

## Compatibility

- Works as a Claude Code skill (`~/.claude/skills/st-pricebook/`)
- Also useful as plain markdown reference — every file stands on its own
- Tenant-agnostic — replace `{tenantId}`, `{d1DatabaseId}`, `<your-sync-key>` placeholders with your values

## Contributing

PRs welcome — especially:

- New silent-fail catalog entries (with verification date)
- Workarounds for documented gotchas
- Cross-tenant verification of catalog entries
- Fixes when ST patches a gotcha

See `AUTHORS.md` for original credits and `LICENSE` (MIT) for usage.
