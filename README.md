# File Processing Pipeline

A FastAPI service that accepts file uploads and runs them through a configurable multi-step processing pipeline asynchronously.

## Architecture

```
POST /upload  →  save file to disk  →  insert job into SQLite  →  worker picks it up
                                                                        ↓
GET /jobs/{id}  ←─────────────────────────────────────────  step-by-step execution
GET /jobs/{id}/download                                           (validate → transform
                                                                   → convert → compress
                                                                   → notify)
```

**Components:**

- **API** (`app/api/`) — upload endpoint, status endpoint, download endpoint, webhook receiver
- **Worker** (`app/worker/`) — polls SQLite for PENDING jobs, executes pipeline steps with retry/backoff
- **Steps** (`app/worker/steps/`) — validate, transform, convert, compress, notify; each is a standalone async function
- **Storage** (`app/services/storage.py`) — streams files to `storage/input/` in 256 KB chunks
- **DB** (`app/db/database.py`) — SQLite via `aiosqlite`, WAL mode, three tables: `file_references`, `jobs`, `job_steps`

## Requirements

- Python 3.12+
- Docker + Docker Compose (for containerised run)

## Running locally

```bash
# Install dependencies
pip install -r requirements.txt

# Start the API + worker in one process (worker disabled via env var for separate mode)
cd file_pipeline
fastapi dev app/main.py
```

The worker runs as a background asyncio task inside the same process unless `RUN_WORKER=false` is set.

## Running with Docker Compose

```bash
docker-compose up --build
```

This starts two containers sharing a named volume (`pipeline_storage`):

| Service | Role |
|---------|------|
| `api` | FastAPI on port 8000, `RUN_WORKER=false` |
| `worker` | `python -m app.worker.runner`, polls SQLite for jobs |

The worker waits for the API healthcheck to pass before starting.

## API Usage

### Upload a file and define a pipeline

```bash
curl -X POST http://localhost:8000/upload \
  -F "file=@input/sample_transactions_100.csv" \
  -F 'pipeline=[
    {"step": "validate", "params": {"expected_type": "csv"}},
    {"step": "transform", "params": {
      "filter": {"column": "status", "operator": "=", "value": "completed"},
      "select_columns": ["transaction_id", "amount"]
    }},
    {"step": "convert", "params": {"output_format": "json"}},
    {"step": "compress", "params": {}},
    {"step": "notify",   "params": {"webhook_url": "http://localhost:8000/webhook"}}
  ]'
```

> **Docker Compose note:** The `worker` container cannot reach `localhost:8000` — `localhost` resolves inside the worker container itself, not the API container. Use the service hostname instead: `"webhook_url": "http://api:8000/webhook"`. The `localhost` URL works when running locally without Docker.

Response `202 Accepted`:

```json
{
  "job_id": "uuid",
  "status": "PENDING",
  "file": {"id": "...", "original_filename": "sample_transactions_100.csv", "size": 1234},
  "steps": [{"index": 0, "type": "validate", "status": "PENDING"}, ...]
}
```

### Check job status

```bash
curl http://localhost:8000/jobs/{job_id}
```

Returns full job state including per-step `status`, `duration_ms`, `error_message`, and `retry_count`.

### Download output

```bash
curl -OJ http://localhost:8000/jobs/{job_id}/download
```

Returns `409 Conflict` if the job has not yet completed.

### Health check

```bash
curl http://localhost:8000/health
```

## Pipeline Steps

| Step | Purpose | Key params |
|------|---------|------------|
| `validate` | Verify file type and structure | `expected_type: "csv"\|"json"` |
| `transform` | Filter rows, select columns, apply value mappings (CSV and JSON) | `filter`, `select_columns`, `apply` |
| `convert` | Convert between CSV and JSON | `output_format: "csv"\|"json"` |
| `compress` | Gzip the file | _(none required)_ |
| `notify` | HTTP POST to a webhook with job completion info | `webhook_url` |

All steps accept an optional `max_retries` param (default: 3). Steps that fail deterministically (wrong column name, type mismatch) raise `NonRetriableError` and are not retried.

### Transform step details

**filter** — keep only rows matching a condition:
```json
{"column": "amount", "operator": ">", "value": "100"}
```
Operators: `=`, `!=`, `>`, `<`, `>=`, `<=`. Numeric coercion is applied automatically.

**select_columns** — keep only named columns:
```json
["transaction_id", "amount"]
```

**apply** — apply a string transformation to all values in every column:
```json
{"type": "lowercase"}
```
Supported types: `lowercase`, `uppercase`, `trim`.

## Running tests

```bash
cd file_pipeline
pytest tests/ -v
```

13 tests covering upload validation, end-to-end pipeline execution, failure handling, status tracking, download, JSON transform, and error cases. Each test runs against an isolated temp-directory DB and storage.

## File layout

```
PMJob/
├── docker-compose.yml
├── Dockerfile
├── requirements.txt
├── README.md
├── DECISIONS.md
├── AI_USAGE.md
└── file_pipeline/
    ├── pytest.ini
    ├── app/
    │   ├── main.py              # FastAPI app + lifespan (worker task)
    │   ├── api/
    │   │   ├── upload.py        # POST /upload
    │   │   ├── status.py        # GET /jobs/{id}, GET /jobs/{id}/download
    │   │   └── webhook.py       # POST /webhook (local test target)
    │   ├── db/
    │   │   └── database.py      # Schema, init_db(), get_db()
    │   ├── services/
    │   │   ├── storage.py       # Chunked file save
    │   │   └── job_service.py   # Transactional job+steps insert
    │   └── worker/
    │       ├── runner.py        # Poll loop, claim, execute, retry
    │       └── steps/
    │           ├── registry.py
    │           ├── errors.py    # NonRetriableError
    │           ├── validate.py
    │           ├── transform.py
    │           ├── convert.py
    │           ├── compress.py
    │           └── notify.py
    └── tests/
        ├── conftest.py
        └── test_pipeline.py
```
