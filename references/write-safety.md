# Write Safety

> The canonical pattern for safely mutating ServiceTitan pricebook items at scale.
> Framework-agnostic; example implementations in Node.js + Cloudflare Workers.

ST's API is high-throughput and well-documented, but it has a uniquely brittle write surface — see [`silent-fail-catalog.md`](silent-fail-catalog.md). The pattern below catches every silent drop and gives you idempotency, audit, and rollback. It's overkill for a one-row PATCH but well worth the cost when you're touching dozens or hundreds of rows.

## The five phases

```
1. Preview  →  2. Confirm  →  3. Execute  →  4. Verify  →  5. Audit
```

Each phase has a specific job. Skipping any phase shows up later as a silent failure that's hard to track down.

### Phase 1 — Preview (dryRun)

For each proposed change, build a "what would happen" object **without making the API call**.

```js
async function preview(changes) {
  const previews = [];
  for (const { id, body } of changes) {
    const current = await stGet(`/services/${id}`);  // or material, equipment, etc.
    const diff = computeDiff(current, body, WRITABLE_FIELDS);
    previews.push({
      id,
      current_hash: hashWritable(current),
      proposed_body: body,
      diff,                                            // {field: [before, after], ...}
      will_change: Object.keys(diff).length > 0,
    });
  }
  return previews;
}
```

Why this matters: a 500-row CSV import often has 200 rows where the CSV value already matches the live value. Skipping the no-op rows saves 200 API calls, 200 audit-log entries, and 200 chances for a silent drop on an unrelated field.

The preview output is **never** thrown away — keep it as the rollback reference.

### Phase 2 — Confirm

The operator (or upstream system) reviews the preview and confirms.

For interactive use: print the diff summary, prompt for `CONFIRM` typed verbatim.

For automated pipelines: bind the preview to a short-TTL HMAC token. The token is the only way to execute.

```js
function mintConfirmToken(previewHash) {
  const expires = Date.now() + 15 * 60 * 1000;  // 15 min
  const payload = `${previewHash}|${expires}`;
  const sig = hmacSha256(SECRET, payload);
  return `${payload}|${sig}`;
}

function verifyConfirmToken(token, expectedPreviewHash) {
  const [hash, expires, sig] = token.split('|');
  if (hash !== expectedPreviewHash) throw new Error('preview drifted');
  if (Date.now() > Number(expires)) throw new Error('token expired');
  if (sig !== hmacSha256(SECRET, `${hash}|${expires}`)) throw new Error('bad signature');
  return true;
}
```

The 15-minute TTL keeps stale plans from being replayed against a moved-on pricebook. Tighter is fine; longer is risky.

### Phase 3 — Execute

POST/PATCH in batches. Recommended chunk size: 25 rows. Smaller chunks = more API calls, but tighter blast radius if something goes wrong.

```js
async function execute(previews, token) {
  verifyConfirmToken(token, hashPreview(previews));

  const results = [];
  for (const chunk of chunked(previews, 25)) {
    for (const p of chunk) {
      if (!p.will_change) { results.push({ id: p.id, skipped: true }); continue; }
      try {
        const resp = await stPatch(`/services/${p.id}`, p.proposed_body);
        results.push({ id: p.id, http: resp.status, ok: true });
      } catch (e) {
        results.push({ id: p.id, error: e.message, ok: false });
      }
    }
    // Brief breather between chunks. Don't preemptively throttle hard;
    // ST's published 120/min is folklore — back off on 429s instead.
    await sleep(250);
  }
  return results;
}
```

### Phase 4 — Verify (read-after-write hash compare)

This is the phase that catches every silent drop. After each write, read the record back, hash the writable fields, and compare to expected.

```js
async function verify(results, previews) {
  const verified = [];
  for (const r of results) {
    if (r.skipped || !r.ok) { verified.push(r); continue; }
    const p = previews.find(p => p.id === r.id);
    const post = await stGet(`/services/${r.id}`);
    const postHash = hashWritable(post);

    // Hash matches the proposed-post state? Write actually landed.
    const proposedPost = { ...post, ...p.proposed_body };
    const proposedPostHash = hashWritable(proposedPost);

    if (postHash !== proposedPostHash) {
      verified.push({
        ...r,
        silent_drop: true,
        dropped_fields: detectDroppedFields(p.proposed_body, post),
      });
    } else {
      verified.push({ ...r, silent_drop: false });
    }
  }
  return verified;
}

function detectDroppedFields(body, post) {
  return Object.keys(body).filter(k =>
    JSON.stringify(body[k]) !== JSON.stringify(post[k])
  );
}
```

A row with `silent_drop: true` looks like a success at the HTTP layer but is a failure for your purposes. Treat it as one — surface to caller, write to a failure log, retry against the documented workaround if one exists.

### Phase 5 — Audit

Every write (including silent-drops) goes to an audit log. Schema (adapt to your stack):

```sql
CREATE TABLE st_audit_log (
  ts             TEXT NOT NULL,        -- ISO 8601
  actor          TEXT NOT NULL,        -- who initiated (operator email, system name)
  op             TEXT NOT NULL,        -- 'PATCH /services/{id}'
  resource_id    TEXT,
  request_body   TEXT,                 -- JSON
  response_body  TEXT,                 -- JSON
  http_status    INTEGER,
  silent_drop    INTEGER,              -- 0/1
  dropped_fields TEXT                  -- JSON array of field names if silent_drop=1
);
```

**Important pitfall:** if you're calling ST through a worker / proxy that auto-audits reads, double-check it also audits writes. We've seen production systems where the gateway audited reads automatically but skipped writes, leaving 258 production dismissals with zero audit rows ([incident 2026-05-10](incidents-and-lessons.md)). The fix is to call your audit function explicitly at the write callsite.

---

## Idempotency

ST POSTs are **not idempotent by default** — same payload twice = two SKUs. Make them idempotent on your side using the external `code` (SKU code) as the dedup key:

```js
async function createServiceIdempotent(body) {
  // Check if a service with this code already exists
  const existing = await searchByCode('services', body.code);
  if (existing) {
    // Decide: skip, update, or fail. Almost always update.
    return await stPatch(`/services/${existing.id}`, body);
  }
  return await stPost('/services', body);
}
```

Pre-check in your D1 mirror if you have one — much faster than a live API search.

---

## Rate limiting

ST's published 120/min figure is conservative and varies by endpoint. Reliable approach:

- Don't preemptively throttle (you'll be slower than you need to be).
- On 429, back off with exponential delay: 2s → 4s → 8s → 16s → 32s → 60s cap.
- Retry up to 5 times. After 5 retries, mark the row failed and move on.

At 250ms between sequential single-row writes, 258 PATCH/PUTs completed with 0 errors against a production tenant. That's a reasonable starting point.

---

## Two-phase via HTTP API

If you're exposing this as a service, the canonical shape:

```
POST /pricebook/write/preview
  body: { changes: [...] }
  → 200 { preview_id, previews: [...], expires_at }

POST /pricebook/write/confirm
  body: { preview_id, token }
  → 200 { results: [...] }   // includes silent_drop flags
```

The preview response includes the token caller needs to confirm. Two-phase makes accidental "I clicked the wrong row" mistakes recoverable.

---

## Reconciling your D1 mirror

After a successful write batch, your D1 mirror is stale by up to one nightly-sync cycle. Two options:

1. **Trigger a partial sync** of only the affected rows: `POST /api/sync-full?ids=<comma-separated>` if your sync endpoint supports it.
2. **Trust ST** as the source of truth for the next ~24h and don't worry about it; the next nightly cron will reconcile.

**Don't** mutate D1 directly to match what you "thought" the write did. If a row was silently dropped, mutating D1 to the desired state hides the drop. The mirror should reflect reality, not aspiration.

---

## Anti-patterns

These look safe but aren't:

- **Trusting 200 OK.** Half the gotchas in the catalog return 200. Always hash-compare.
- **Skipping the preview phase for "small" changes.** A "small" change without preview can still be the wrong row, with no rollback reference.
- **Mutating D1 alongside the write.** Either the write succeeds (next sync will reconcile) or it silently fails (D1 now diverges from reality and you can't tell). Let the mirror catch up from ST.
- **Bundling unrelated fields into one PATCH.** If one field silently drops, the others succeeded — but your diff says the whole row failed. Surface per-field results.
- **Re-running a "successful" import.** If you re-run a batch where some rows silently dropped, you'll hit the same silent drops again. Fix the drops first (UI fallback, etc.), then re-run only the actually-failed rows.

---

## Reference implementation (Cloudflare Workers)

For an end-to-end implementation in Cloudflare Workers + Durable Objects with HMAC tokens, D1 audit log, and per-tool rate limiting: see the QSC-internal `mcp-servicetitan` worker source. The pattern is portable to any framework — the phases are framework-agnostic.

Key components:
- `WriteGate` — handles preview → confirm → execute → audit
- `ConfirmationToken` — HMAC bound to preview hash + expiry
- `obs.audit()` — writes per-call rows to `audit_log` D1 table
- `validateConfirmToken()` — verifies token + hash match before executing

The pattern is broadly the same regardless of your runtime — adapt the implementation to your needs.
