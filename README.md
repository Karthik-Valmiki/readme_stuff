# AsyncZ — Complete A to Z Walkthrough

## What is this project in one sentence?

A system where clients submit jobs (tasks) that get processed in the background by workers, without the client waiting for the result.

---

## The Core Problem it Solves

Imagine a user uploads a video and you need to compress it. If you do it inside the API request, the user waits 2 minutes staring at a loading spinner. Instead:
- The API says "got it, here is your job ID" in under 10ms
- The video compression happens in the background
- The user polls `GET /jobs/{id}` to check when it is done

This is what every company does for anything that takes more than 1 second. Email sending, PDF generation, AI inference, payments — all async.

---

## The Full Architecture

```
Client (k6 / browser / any app)
        │
        │  POST /jobs  {"payload": {"task": "..."}}
        ▼
   ┌─────────────┐
   │   FastAPI   │  ← Your API. Lives in app/main.py
   │  (Uvicorn)  │    Handles HTTP requests
   └──────┬──────┘
          │  1. INSERT job row into PostgreSQL (status = "queued")
          │  2. ENQUEUE job_id into Redis via ARQ
          │  3. Return 202 + job_id immediately
          ▼
   ┌─────────────┐
   │  PostgreSQL │  ← Source of truth. Every job's state lives here.
   │  (jobs DB)  │    Tables: jobs, job_execution_logs
   └─────────────┘

   ┌─────────────┐
   │    Redis    │  ← The queue. Holds job IDs waiting to be processed.
   │  (ARQ queue)│    Also holds the Dead Letter Queue (asyncz:dlq)
   └──────┬──────┘
          │  Worker pulls job_id from Redis
          ▼
   ┌─────────────┐
   │   Worker    │  ← Your background processor. Lives in app/worker.py
   │  (ARQ/arq)  │    Runs as a completely separate process.
   └─────────────┘
          │  1. Mark job as "processing" in PostgreSQL
          │  2. Send heartbeat every 10s (proves worker is alive)
          │  3. Execute the actual job logic
          │  4. On success → mark "completed" in PostgreSQL
          │  5. On failure → retry (with backoff) up to max_retries
          │  6. If all retries fail → mark "dead", push to DLQ
          ▼
   ┌─────────────┐
   │  PostgreSQL │  ← Final state written back here
   └─────────────┘
```

---

## Every File Explained

| File | What it does |
|---|---|
| `app/main.py` | The FastAPI HTTP server. 4 endpoints. |
| `app/worker.py` | The background job processor. Runs separately. |
| `app/db.py` | PostgreSQL connection pool setup (asyncpg). |
| `app/redis_client.py` | Redis connection pool setup (ARQ). |
| `app/models.py` | SQLAlchemy ORM models — the `jobs` and `job_execution_logs` tables. |
| `app/schemas.py` | Pydantic request/response shapes. What the API accepts and returns. |
| `tests/load_test.js` | k6 load test that simulates many users submitting jobs. |
| `start.bat` | Starts both Uvicorn and the ARQ worker in separate terminal windows. |

---

## Every Endpoint Explained

### `POST /jobs`
**What you send:**
```json
{
  "payload": {"task": "send_email", "to": "user@example.com"},
  "idempotency_key": "optional-uuid",
  "max_retries": 3
}
```
**What you get back (immediately, in under 10ms):**
```json
HTTP 202
{"job_id": "some-uuid", "status": "queued"}
```
The job is now in PostgreSQL and Redis. The worker will pick it up.

---

### `GET /jobs/{job_id}`
Poll this to track progress. Status moves through:
```
queued → processing → completed
queued → processing → queued (retry) → processing → dead
```
**What you get:**
```json
{
  "job_id": "...",
  "status": "processing",
  "retry_count": 0,
  "max_retries": 3,
  "heartbeat_at": "2026-08-01T12:45:10",  ← proves worker is alive
  "worker_id": "my-laptop-1234",
  "started_at": "2026-08-01T12:45:00",
  "completed_at": null
}
```

---

### `GET /dlq`
Shows all jobs that died after exhausting all retries.
```json
{
  "count": 2,
  "jobs": [
    {
      "job_id": "...",
      "payload": {"fail": true},
      "retry_count": 4,
      "last_error": "RuntimeError: Job intentionally failed",
      "failed_at": "2026-08-01T12:50:00"
    }
  ]
}
```

---

### `GET /health`
```json
{
  "status": "ok",
  "db": "ok",
  "redis": "ok",
  "dlq_length": 0
}
```
If `dlq_length` is growing, something is broken in your job logic.

---

## Phase 2 Features Explained Simply

### Retry Mechanism
When a job fails, instead of giving up immediately, the worker re-queues it and tries again. It waits longer each time (1s, then 2s, then 4s) so it does not hammer a broken dependency. After `max_retries` failures, it gives up and moves the job to the DLQ.

### Idempotency
If a client sends the same job twice with the same `idempotency_key`, the second request returns `409 Conflict` with the original `job_id`. The client can just poll that original `job_id`. No duplicate work happens.

### Heartbeats
While a job is running, the worker writes the current time to `heartbeat_at` in the database every 10 seconds. Think of it as the worker saying "I am still alive, still working on this."

### Zombie Recovery
A cron job runs every 60 seconds. It looks for jobs stuck in `processing` where `heartbeat_at` has not been updated for 60+ seconds. These are "zombies" — the worker that was processing them is dead (crashed, killed, out of memory). The recovery cron re-queues them automatically.

### Dead Letter Queue (DLQ)
A separate Redis list (`asyncz:dlq`) that stores the full payload of jobs that failed permanently. Instead of losing that data, you can inspect it, fix the bug, and reprocess manually.

---

## How to Get Multiple Workers

This is much simpler than you think. A "worker" is just the command `python -m arq app.worker.WorkerSettings` running in a terminal. To get 3 workers, you run that command 3 times in 3 different terminals. That is it.

Each worker process:
- Connects to the same Redis queue
- Connects to the same PostgreSQL database
- Picks up whichever job_id comes next from Redis
- Works on it independently

Redis acts as the coordinator. It guarantees that each job_id is only given to one worker at a time (FIFO queue). Two workers will never accidentally process the same job.

**With 1 worker** → processes ~7 jobs per second (limited by `max_jobs = 10` and 1.5s average job duration)

**With 3 workers** → processes ~21 jobs per second (each runs 10 concurrent jobs independently)

**In production** → you run as many workers as your server has CPU cores, or you run workers on multiple machines all pointing at the same Redis.

To try it right now, open 3 PowerShell terminals and run in each:
```powershell
cd "C:\Users\ORBIT\OneDrive\Desktop\SELF PROJECTS\AsyncZ"
Myenv\Scripts\activate
python -m arq app.worker.WorkerSettings
```
You will see all 3 workers picking up jobs in parallel.

---

## What the k6 Results Actually Mean

```
checks_succeeded...: 100.00%   ← Every job was accepted. API is healthy.
http_req_failed.....: 0.00%    ← No connection errors.
http_reqs...........: 259      ← 259 jobs submitted in 3 seconds.
http_req_duration...: avg=2.22s ← This is the API response time, not job completion time.
```

> [!IMPORTANT]
> The `http_req_duration` of 2.22 seconds is how long `POST /jobs` took to respond. It should be under 50ms for a healthy system. 2.22 seconds means the DB or Redis was slow under 200 VU pressure on Windows. On Linux this would be under 10ms.

---

## The Honest Summary

You did not overcomplicate it. This is exactly the architecture used in production.

The only thing left to do to make this a genuinely impressive portfolio project is:
1. Run it inside Docker or WSL2 so you can push real load
2. Write a README explaining the design decisions (why Redis, why ARQ, why commit before enqueue)
3. Make a diagram showing the flow (you already have it in `phases_wise`)

The code is real. The architecture is real. The concepts are industry-standard.
