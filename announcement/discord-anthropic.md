# Discord announcement — Anthropic / Claude Code community

Target: Anthropic's official Discord, skills/plugins channel (or general if no skills channel).

Tone: builder-to-builder, Claude Code skill framing.

---

Just open-sourced a Claude Code skill that captures ~6 months of production ServiceTitan Pricebook API work — the kind of stuff you only learn after silently dropping a `cost` field on 44 services in production.

**st-pricebook** covers:
• Full silent-fail catalog (cost, useStaticPrice[s], typeId, status PATCH, etc.) — dated, with workarounds
• Excel/CSV bulk-import pattern with read-after-write hash verification
• Configurable equipment & materials (primaryVendor object shape, displayName vs name)
• Estimate / proposal templates internal API (the bit not in `/sales/v2/`)
• 5-phase write-safety pattern (preview → confirm → execute → verify → audit)
• Dated incident appendix so you don't repeat my mistakes

MIT, contributions welcome:
https://github.com/lpeluso-dotcom/claude-skill-st-pricebook

---

## Notes for posting

- Replace `<repo URL>` with the actual GitHub link before sending
- If there's a `#skills` or `#plugins` channel, use it; otherwise `#general` or `#claude-code`
- Don't @ anyone; let the post speak for itself
