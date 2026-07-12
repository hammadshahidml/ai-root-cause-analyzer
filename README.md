# STEM Academic Workspace Pro — Backend

## Stack
- FastAPI (API tier) · Redis (broker + result backend) · Celery (workers)
- Anthropic API called only from workers, never from the browser in this architecture.

## Files
- `celery_app.py` — queue config (json-only serialization, acks_late, result TTL)
- `tasks.py` — 3 workers: deterministic table conversion, math derivation (LLM), log analysis (LLM)
- `main.py` — API surface: schemas, auth gate, CORS, submit + status endpoints
- `useAsyncTask.js` — React polling hook + integration guide for the existing frontend

## Run (development)
```bash
pip install -r requirements.txt

# 1. Redis
redis-server                          # or: docker run -p 6379:6379 redis:7

# 2. Environment
export ANTHROPIC_API_KEY=sk-ant-...
export API_KEYS=dev-key               # comma-separated list
export ALLOWED_ORIGINS=http://localhost:5173

# 3. Worker
celery -A celery_app worker --loglevel=info --concurrency=4

# 4. API
uvicorn main:app --reload --port 8000
```

## Deliberate decisions (do not "fix" without understanding)
- `task_acks_late=True` + `task_reject_on_worker_lost=True`: a crashed worker
  requeues its task. For LLM tasks this can double-bill inference. Chosen:
  reliability over cost.
- `result_expires=3600`: results vanish from Redis after 1h. Clients must
  fetch within that window.
- Table conversion endpoint exists but interactive UIs should convert
  client-side (deterministic, ~2ms locally vs 1-2s+ through the queue).
- Auth is a static API-key check. Sufficient to stop drive-by abuse of paid
  LLM endpoints; NOT sufficient to charge customers. No rate limiting yet.

## Known gaps before production
1. Per-key rate limiting and usage metering (the "premium tier" has no teeth without it).
2. Observability: no logging/metrics/tracing (add Flower for Celery at minimum).
3. Polling wastes requests; SSE or WebSockets would push results and enable
   true token streaming from the LLM.
4. Two parser implementations exist (JS client-side, Python worker). They are
   mirrored today; without a shared test suite they will drift.
