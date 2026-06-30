# Design Decisions

## 1. Large File Handling

**Approach chosen:** Streaming upload with 256 KB chunks written directly to local disk via `aiofiles`. The file is never fully loaded into memory.

**Why:** Files up to 100 MB would cause memory spikes and risk OOM errors if buffered entirely. Chunked streaming keeps memory usage flat regardless of file size — each chunk is written and discarded before the next is read.

**Memory usage:** Bounded to one 256 KB chunk at a time per upload. Concurrent uploads each hold one chunk, so peak memory scales with concurrency, not file size.

**Storage backend — local vs cloud:** Input and output files are stored on the local filesystem (`storage/input/`, `storage/output/`). In production this would be replaced with object storage (S3/GCS) so that multiple worker instances can all access the same files. The storage logic is isolated in `app/services/storage.py` so this swap would not touch the pipeline or worker code.

---

## 2. Step Failure Strategy

**Approach chosen:** Retry the failed step up to `max_retries` times (default: 3, configurable per step in the pipeline definition) with exponential backoff. If all retries are exhausted, mark the step `FAILED`, mark all remaining steps `SKIPPED`, and mark the job `FAILED`.

**Why:** Transient errors (network timeouts in the notify step, momentary disk errors) should not immediately kill a job. Three retries with backoff handles the common case. Deterministic failures raise `NonRetriableError` and skip all retries immediately — see Section 5 for the classification rule.

**Partial progress:** Completed steps write their output to disk and record `output_file_id` in the `job_steps` table before the next step begins. On failure, all completed step outputs are preserved. This also enables a future "resume from last successful step" feature without schema changes.

---

## 3. Transform Filter Operators

**Approach chosen:** Support exactly six comparison operators: `=`, `!=`, `>`, `<`, `>=`, `<=`.

**Why:** The assignment says "filter rows based on criteria" — equality alone is too narrow (e.g. "transactions above $500" requires `>`). Full SQL-like expressions (AND/OR, IN, LIKE, regex) are out of scope and would add significant complexity without matching any stated requirement. Six operators cover the realistic use cases cleanly.

**Numeric coercion:** CSV values are always strings. When comparing, the filter first attempts `float()` conversion on both sides; if that fails it falls back to lexicographic string comparison. This means `amount > 100` works correctly on a CSV column that contains `"150.00"`.

**Extensibility:** Adding a new operator is a one-line addition to the `OPERATORS` dict in `transform.py`.

---

## 4. Convert Step — JSON→CSV Memory Usage

**Approach chosen:** CSV→JSON streams row by row and never loads the full file. JSON→CSV loads the full JSON into memory before writing CSV.

**Why:** CSV is a row-oriented format that maps naturally to streaming iteration. JSON is a tree structure — to write CSV headers you need all field names, and `json.load()` is the standard way to parse it. A streaming alternative (`ijson`) would require an extra dependency for a marginal gain given the 100 MB file size constraint.

**Known limitation:** A 100 MB JSON file will use ~100 MB of heap during JSON→CSV conversion. Acceptable for this assignment; production would use `ijson` for streaming JSON parsing.

---

## 5. Retriable vs Non-Retriable Errors

**Approach chosen:** Steps raise `NonRetriableError` for deterministic failures. The worker catches it, skips all retries, and marks the step `FAILED` immediately.

**Why:** Retrying a deterministic error (wrong column name, unsupported file type, invalid params) wastes time and gives the user a misleading delay. Retries are only useful for transient failures (disk hiccup, network timeout in the notify step) where the same operation might succeed on a second attempt.

**Rule of thumb:** If the error would reproduce 100% of the time with the same input, raise `NonRetriableError`. If it could be caused by a temporary external condition, raise a plain exception and let the retry logic handle it.

---

## 6. Webhook Reliability

**Approach chosen:** The notify step retries on `5xx` and network errors (transient). It fails immediately on `4xx` (non-retriable — bad URL or auth). After `max_retries` exhausted, the job is marked `FAILED`.

**Why fail the job on notification failure?** Keeps the state machine simple — a job is only `COMPLETED` if every step including notification succeeded. The alternative (a `COMPLETED_WITH_NOTIFICATION_FAILURE` status) adds complexity without much benefit for this scope.

**Duplicate notification risk:** If the webhook POST succeeds but the worker crashes before writing `COMPLETED` to the DB, the step will be retried and the webhook called again. Mitigation in production: use an idempotency key in the payload (we include `job_id`) so the receiver can deduplicate.

**Production note:** In production this would target a Slack incoming webhook, a custom backend, or an email service HTTP endpoint (SendGrid, Mailgun). Email addresses cannot be used directly — webhooks are HTTP endpoints.

**Security note:** Webhook URL validation checks only that the URL starts with `http://` or `https://`. This is intentional for a local/demo context. A production system would add SSRF protection (blocking private IP ranges and internal hostnames), allowlist customer-registered domains, and optionally require HMAC-signed payloads for receiver authentication.

---

## 7. File Cleanup Strategy

**Approach chosen:** No automatic cleanup in this implementation. The `file_references` table has an `expires_at` column reserved for a future TTL-based eviction policy, but no background task currently reads it.

**Why deferred:** The assignment does not require cleanup, and adding it would mean another background task, a policy decision about retention windows, and test coverage for edge cases (delete while job in flight, etc.). The schema change cost to add cleanup later is zero — the column is already there.

**Production approach:** A periodic cleanup task (a second async task in `lifespan`, or a cron job) would scan `file_references WHERE expires_at < now()`, delete the files from disk, and remove the DB rows. Output files would get a longer TTL than intermediates.

---

## 8. Progress Tracking

**Approach chosen:** Step-level granularity tracked in the `job_steps` table. Each step records `status` (PENDING → RUNNING → COMPLETED/FAILED/SKIPPED), `started_at`, `completed_at`, and `duration_ms`. The parent `jobs` row tracks `current_step_index` and overall status.

**Why step-level and not byte-level:** Byte-level progress (e.g. "45% through the file") would require either streaming counters in each step function or a side-channel update mechanism. Step-level progress is accurate, zero-overhead, and sufficient for the use cases in this assignment. A client polling `/jobs/{id}` every few seconds gets a meaningful view of where the pipeline is.

**Trade-off:** Long-running single steps (e.g. a large gzip compress) show no progress within the step itself — the status stays `RUNNING` until the step completes. Acceptable at this scale; production would push intermediate progress events to a WebSocket or SSE stream.

---

## 9. Job State Storage

**Approach chosen:** SQLite via raw `aiosqlite` (no ORM).

**Why not an ORM (SQLAlchemy):** The schema is simple — three tables with straightforward inserts and lookups by primary key. An ORM adds setup overhead and a dependency without meaningful benefit at this scale.

**Why not Redis:** Redis would require an extra docker service. SQLite in WAL mode handles concurrent reads from the API and writes from the worker without contention, which is all we need.

**Why not in-memory:** Lost on restart. A job that was `PENDING` when the server crashes would be silently lost with no way to recover or even detect it.

**WAL mode:** Enabled on every connection (`PRAGMA journal_mode=WAL`) so the API process and the worker process can access the database concurrently without blocking each other.

---

## 10. One Thing I Would Do Differently With More Time

**I would replace the SQLite polling queue with a durable task queue.**

The current poller works at this scale, but has production blind spots: a job that crashes mid-step stays stuck in `PROCESSING` indefinitely with no automatic recovery, and queue depth is only visible by querying the DB directly.

A proper task queue (Redis/RQ, Celery, SQS, Cloud Pub/Sub) adds dead-letter queues, visibility timeouts that auto-recover stalled jobs, and queue-depth metrics out of the box. The `docker-compose.yml` already models API/worker separation; replacing the SQLite poll with a queue transport is the natural production upgrade.

---

## 11. Deliberate Omissions

**Decompress / zip extraction:** The assignment mentions "Compress/Decompress — Gzip compression, Zip extraction." This implementation covers gzip compression only. Decompression and zip extraction were omitted as the five implemented steps (validate, transform, convert, compress, notify) already exceed the four-step minimum and cover the core pipeline use case. Zip extraction would be a straightforward addition: a `decompress` step that calls `zipfile.ZipFile.extractall` in `asyncio.to_thread`.

**Job cancellation:** There is no `DELETE /jobs/{id}` or cancel endpoint. The `job_steps` table supports a `SKIPPED` status that the worker writes on failure, but there is no mechanism for a client to request cancellation of an in-flight job. This was omitted as the assignment does not require it; adding it would need a CANCELLED status, a way to signal the running worker coroutine, and handling for the case where a step is mid-execution when the cancel arrives.
