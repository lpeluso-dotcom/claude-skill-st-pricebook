# Discord announcement — ServiceTitan developer / integrator community

Target: any ServiceTitan API / developer Discord. Adjust framing if posting to an operator-focused server (less code-heavy).

Tone: ST-API-developer-focused, lead with the gotcha catalog because that's the differentiator.

---

Sharing a free reference for anyone integrating with the ServiceTitan Pricebook v2 API.

**st-pricebook** documents the things that took us months to learn from production:

• **Silent-fail field catalog** — every field ST drops on POST or PATCH (cost on services, name on materials, status PATCH on estimates, useStaticPrices not flippable post-create, etc.) with dates and workarounds
• **Internal `/app/api/estimates/templates/*` API** — the read↔write schema asymmetries that aren't documented anywhere else (`skuReference` wrapping, `isAddon` case, `summary` ↔ `description`)
• **Status transition endpoints** — `PUT /sell`, `/dismiss`, `/unsell` instead of PATCH (verified at scale, 258 dismissals with 0 errors)
• **Two-phase write pattern** (preview → confirm → execute → verify) that catches every silent drop before it hits prod
• **Dated incident log** — including the 2026-05-10 estimate-status fiasco where the PATCH wrapper returned 200 OK on all 5 test rows while ST silently ignored every one

Six months of production lessons from a multi-trade tenant, sanitized for public release. MIT-licensed, PRs welcome with verified-on-prod observations from your tenant.

https://github.com/lpeluso-dotcom/claude-skill-st-pricebook

---

## Notes for posting

- Verify which Discord servers Luke is in via `mcp__discord__discord_login` + `discord_get_server_info` before sending
- For an operator-focused channel, trim the code-heavy language and lead with "if you've ever done a bulk pricebook update and wondered why some changes didn't stick…"
- Skip if the server has an explicit "no self-promotion" rule
