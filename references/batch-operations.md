# Batch Operations

> Patterns for safely running PATCH/POST in bulk against the ST pricebook API —
> idempotency, partial failures, rate limiting, recovery, and reconciliation.

A "batch" here means anything from ~10 to ~10,000 rows of writes. The patterns below scale across that range; for true bulk imports from a spreadsheet, see [`excel-import.md`](excel-import.md).

## Core principles

1. **Every batch is resumable.** A network blip at row 312 of 500 should never force you to restart from row 0.
2. **Per-row results, not batch-level success.** "200 OK" on the HTTP call ≠ "the write stuck." Surface per-row outcomes (including silent drops).
3. **Verify writes; don't trust them.** See [`write-safety.md`](write-safety.md) for the read-after-write hash compare pattern.
4. **Reconcile the mirror after the batch.** Don't mutate D1 inline.

---

## Idempotency

ST `POST` endpoints are **not** idempotent by default — same payload twice = two SKUs with auto-incremented `id`s but the same `code`. ST's UI will eventually flag the duplicate, but by then you've created a mess.

Make POSTs idempotent on your side using the SKU `code` as the dedup key:

```js
async function createOrUpdate(type, body) {
  const existing = await findByCode(type, body.code);
  if (existing) {
    return { action: 'PATCH', id: existing.id, response: await stPatch(`/${type}s/${existing.id}`, body) };
  }
  return { action: 'POST', response: await stPost(`/${type}s`, body) };
}
```

Pre-check in your D1 mirror if you have one — much faster than a live search. Refresh the mirror first if it's stale.

PATCHes are inherently idempotent (same input = same final state) so no dedup logic needed beyond hash-compare to skip no-ops.

---

## Partial failures: never abort, always continue

In a batch of N rows, expect that some will fail for non-fatal reasons (transient 5xx, rate limit, individual row data issues). The wrong response is to abort the batch.

```js
async function executeBatch(rows) {
  const results = [];
  for (const row of rows) {
    try {
      const r = await processRow(row);
      results.push({ ok: true, ...r });
    } catch (e) {
      results.push({ ok: false, row, error: e.message });
      // Continue. Do not throw, do not break.
    }
  }
  return results;
}
```

Group failures by error class at the end:

```js
function summarize(results) {
  return {
    total: results.length,
    success: results.filter(r => r.ok && !r.silent_drop).length,
    silent_drop: results.filter(r => r.silent_drop).length,
    transient: results.filter(r => !r.ok && r.error.match(/5\d\d|timeout/)).length,
    permanent: results.filter(r => !r.ok && r.error.match(/4\d\d/)).length,
  };
}
```

The transient bucket is the candidate for automatic retry. The permanent bucket needs human intervention (bad data, missing FK, etc.).

---

## Retries with exponential backoff

For HTTP 429 (rate limit) and 5xx (transient server error), retry with exponential delay:

```js
async function withRetry(fn, opts = {}) {
  const { maxRetries = 5, baseMs = 2000, capMs = 60000 } = opts;
  let lastErr;
  for (let attempt = 0; attempt < maxRetries; attempt++) {
    try {
      return await fn();
    } catch (e) {
      lastErr = e;
      if (!isRetryable(e)) throw e;
      const delay = Math.min(baseMs * 2 ** attempt, capMs);
      await sleep(delay + Math.random() * 500);  // jitter to avoid thundering herd
    }
  }
  throw lastErr;
}

function isRetryable(e) {
  return e.status === 429 || (e.status >= 500 && e.status < 600) || e.code === 'ECONNRESET';
}
```

5 retries with the cap at 60s gives up to ~62s of total wait per row (2 + 4 + 8 + 16 + 32 + retry on row 6 = retry budget exhausted). Adjust to your tolerance.

After max retries exhausted: log the row to the failure CSV with full error details, move on to the next row.

---

## Rate limiting reality check

ST publishes a "120 requests/minute" guideline. In practice:

- The enforced rate varies by endpoint and time of day
- Preemptive throttling at 2/sec is fine and rarely hits limits
- On a 429, back off and retry — don't preemptively rate-limit harder
- Read endpoints tolerate much higher rates than write endpoints

For sequential single-row writes, **250ms inter-request delay** has worked at production scale (258 sequential PUTs with 0 errors). For parallel writes (multiple workers writing concurrently), keep total throughput conservative — concurrent writers compound easily.

If you're seeing repeated 429s, the recovery pattern:

1. Pause all writes for 60s
2. Reduce concurrency by half
3. Resume from the next unprocessed row
4. Log the 429 burst for post-mortem (maybe your peak concurrency was wrong, or maybe ST hit a hot path)

---

## Chunking strategy

Recommended chunk size: **25 rows per chunk** for serialized writes.

- Smaller (5–10) = more API call overhead but finer-grained checkpoint
- Larger (50–100) = lower overhead but bigger blast radius on a chunk failure

Between chunks, sleep 250ms. Within a chunk, write sequentially (parallelism inside a chunk is risky given the unclear rate limit).

```js
async function chunkedWrite(rows, chunkSize = 25, interChunkMs = 250) {
  for (let i = 0; i < rows.length; i += chunkSize) {
    const chunk = rows.slice(i, i + chunkSize);
    for (const row of chunk) {
      await processRowWithRetry(row);
    }
    if (i + chunkSize < rows.length) await sleep(interChunkMs);
  }
}
```

---

## Resumability via checkpoint file

For batches over ~50 rows, write a checkpoint after each chunk so you can resume:

```js
async function batchWithCheckpoint(rows, checkpointPath) {
  let startIdx = 0;
  if (await fileExists(checkpointPath)) {
    startIdx = JSON.parse(await fs.readFile(checkpointPath, 'utf8')).nextIndex;
    console.log(`Resuming from row ${startIdx}`);
  }
  for (let i = startIdx; i < rows.length; i += 25) {
    const chunk = rows.slice(i, i + 25);
    for (const row of chunk) {
      await processRowWithRetry(row);
      // Append result to per-row log file (success.csv, silent-drops.csv, failures.csv)
    }
    await fs.writeFile(checkpointPath, JSON.stringify({ nextIndex: i + 25, ts: Date.now() }));
  }
}
```

On resume, the script picks up at the next unprocessed row. The per-row log files are the audit trail.

---

## Reconciling the mirror

After the batch completes (or pauses), trigger your D1 sync to reconcile:

```bash
# Whole-table reconcile
curl -X POST "https://<your-worker>.example.workers.dev/api/sync-full?table=pb_services" \
  -H "X-Sync-Key: <your-sync-key>"

# Or, if your sync endpoint supports it, only the rows you touched
curl -X POST "https://<your-worker>.example.workers.dev/api/sync-full?table=pb_services&ids=12,34,56" \
  -H "X-Sync-Key: <your-sync-key>"
```

**Don't** mutate D1 directly to reflect what you intended to write. If a write silently dropped, mutating D1 to the desired state hides the bug. Let the mirror catch up from ST's actual state.

---

## Detecting silent drops at batch scale

For a 500-row batch, the silent-drop detection step is itself ~500 API calls. To keep wall-time reasonable:

- Run the GETs in parallel chunks of 5-10 (reads tolerate higher concurrency than writes)
- Skip GETs for rows where the original PATCH/POST failed at the HTTP layer (already failure)
- For PATCHes where the only intended change was on a known-safe field (e.g. `active`), you can probabilistically sample: GET every 10th row, alarm if any drop, otherwise trust

```js
async function detectDropsParallel(results) {
  const verifyTargets = results.filter(r => r.ok && r.intended_changes);
  const droppedReports = [];
  for (const chunk of chunked(verifyTargets, 8)) {
    const post = await Promise.all(chunk.map(r => stGet(`/${r.type}s/${r.id}`)));
    chunk.forEach((r, i) => {
      const dropped = detectDroppedFields(r.intended_changes, post[i]);
      if (dropped.length > 0) droppedReports.push({ id: r.id, dropped });
    });
  }
  return droppedReports;
}
```

---

## Recovery from a half-applied batch

If a batch pauses partway (network failure, operator Ctrl-C, worker timeout), the state is:

- Rows up to the checkpoint: applied (success / silent-drop / failure all logged)
- Rows after the checkpoint: untouched

To recover:

1. Run the batch again with the same checkpoint file. It'll resume at the right row.
2. Don't re-do the already-applied rows — they're in success.csv / silent-drops.csv / failures.csv.
3. After completion, run the verification step against ALL rows (including ones from the previous run) — silent drops can be discovered later.
4. Apply workarounds for silent drops; re-run only the silent-drop subset against the documented workaround path.

---

## Anti-patterns

- **No retry on 429.** You'll have a fraction of your batch fail "permanently" that was actually transient.
- **Infinite retry on any error.** A row with bad data will retry forever; never make progress. Cap at 5 retries and move on.
- **Parallel writes without coordination.** Two workers writing to the same SKU = race conditions, last-write-wins, and audit confusion.
- **Skipping the checkpoint file for "small" batches.** Define "small" — anything under 50 maybe — but the rule is: if losing progress would be annoying, checkpoint.
- **Mutating D1 directly during the batch.** Either the write actually landed (next sync will catch up) or it didn't (and D1 now diverges from reality). Let the mirror reconcile from ST.
- **Treating per-chunk HTTP errors as batch failures.** One chunk-level error rarely means "the rest of the batch is poisoned" — keep going, surface per-row outcomes.
