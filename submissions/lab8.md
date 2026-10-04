# Lab 8 — Chaos Engineering

## Hypotheses (committed before any experiment ran)

**Experiment 1 — Pod kill under load.**
If one of the 5 gateway pods is deleted while mixedload runs, the error count stays at ~0 (at most 1–2 in-flight requests fail) and the other 4 pods absorb its traffic. The Service removes a terminating pod from endpoints right away, and the ReplicaSet starts a replacement within ~5–10 s. Total RPS should not dip noticeably.

**Experiment 2 — Payment latency 2000 ms.**
If payments takes 2 s per request, `/pay` p99 rises to ~2 s and no 5xx appear, because 2 s < `GATEWAY_TIMEOUT_MS=5000`. `/events` and `/reserve` p99 stay unchanged since they don't touch payments. Total RPS drops: each mixedload loop is sequential and now waits 2 s on `/pay`. At 6000 ms, `/pay` returns 504 after ~5 s. Gateway retries and circuit breaker are no-op stubs until Lab 11, so nothing fails fast — every request waits the full timeout.

**Experiment 3 — Redis down.**
If Redis is scaled to 0, `/reserve` fails with 5xx (504 seen in Lab 6, when events blocked on the Redis connection). `/health` reports `degraded`. Expected surprise: events' liveness probe hits `/health`, which returns 503 without Redis (same issue as Lab 4), so kubelet keeps restarting events. `/events`, which needs only Postgres, will likely start failing too after ~30 s. A Redis outage would then turn into a full outage of the read path.
