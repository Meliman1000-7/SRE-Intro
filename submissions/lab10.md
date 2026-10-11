# QuickTicket Reliability Review

Scope: QuickTicket on k3d (gateway Rollout ×5, events ×1, payments ×1, Postgres on a PVC, Redis), Labs 1–9. Load tests ran on 2026-10-11 with Locust 2.43.4 as an in-cluster Job against `http://gateway:8080`; Redis was flushed before every run.

## 1. SLO Compliance

SLOs from Lab 3: 99.5% availability (non-5xx), 95% of requests under 500 ms.

| SLO | Target | Observed | Status |
|---|---|---|---|
| Availability, normal load (5 min, 2590 req, incl. a live migration) | 99.5% | 100% | ✅ |
| Latency < 500 ms, normal load | 95% | 100% | ✅ |
| Availability, whole test day (36.6k req, incl. disaster drills and overload) | 99.5% | 91.7% | ❌ |
| Latency < 500 ms, whole test day | 95% | 93.9% | ❌ |

- At normal load the system has plenty of margin.
- The test day burned **3036 errors against a budget of 183** (0.5% of 36.6k), about 16.6× the budget. Almost all of them came from load above the capacity ceiling (section 2), not from the deliberate Postgres kills.
- History is short: the in-cluster Prometheus has no persistent storage and only held data from 14:00 that day (section 6).

## 2. Load Test Results

| Users | Ramp | RPS | p50 | p95 | p99 | 5xx error rate | 409 (inventory) |
|------:|-----:|----:|----:|----:|----:|---------------:|----------------:|
| 10    | 2/s  | 7.6  | 12 ms | 19 ms | 26 ms | 0% | 0 |
| 50    | 5/s  | 37.4 | 7 ms | 24 ms | 67 ms | 0% | 25 (1.1%) |
| 75    | 8/s  | 55.5 | 6 ms | 150 ms | 260 ms | 0.42% | 98 (3.0%) |
| **100** | 10/s | 60.9 | 270 ms | 610 ms | 800 ms | **37.0%** | 20 (0.5%) |
| 200   | 20/s | 78.5 | 85 ms | 5000 ms | 5000 ms | 43.8% | 33 (0.7%) |

**Breaking point: between 75 and 100 users. Capacity ceiling ≈ 55 RPS.** At 75 users both limits were nearly reached (0.42% 5xx, p99 260 ms); at 100 users both were far exceeded.

**What broke:** the single `events` pod.
- `psycopg2.pool.PoolError: connection pool exhausted` — 873 times in its log. The pool (`DB_MAX_CONNS=10`) raises immediately instead of queueing, so every extra concurrent request becomes a 500.
- CPU pinned at its 200m limit (section 8).
- At 200 users it stopped answering its liveness probe in time and was restarted, which turned errors into 502/504 for everyone.

Errors at 100 users: `/events` 502 ×857, `/reserve` 500 ×319, `/reserve` 502 ×48, `/health` 503 ×121. 409s were counted separately and are not failures.

**Verification of the fix** — same tests with `events` scaled to 3 replicas:

| Users | events | RPS | p50 | p95 | p99 | 5xx | 409 |
|------:|-------:|----:|----:|----:|----:|----:|----:|
| 100 | ×3 | 75.1 | 6 ms | 11 ms | 24 ms | 0% | 285 (6.3%) |
| 200 | ×3 | 149.2 | 5 ms | 15 ms | 61 ms | 0% | 1187 (13.3%) |

Ceiling moved from ~55 RPS to at least 149 RPS (2.7×) with zero 5xx. 409s grew only because 580 tickets sell out faster.

## 3. DORA Metrics

| Metric | Value | Source |
|---|---|---|
| Deployment frequency | 21 deploys in 16 days (~1.3/day), on demand | 9 ArgoCD syncs (`argocd app history`, 2026-09-26) + 12 gateway Rollout revisions (2026-10-04); 67 commits and 8 merged PRs on `main` |
| Lead time for changes | **~4.7 min** code → running; 5 s – 5 min for manifest-only changes | Lab 5: push 12:09:47 → Healthy 12:14:28. CI build 2m40s–3m44s (`gh run list`) + ArgoCD poll |
| Change failure rate | **29%** (6 of 21) including deliberate failures; **~5%** (1 of 21) unplanned | 2 bad image tags (Lab 5), 2 manual aborts + 1 crash-looping canary + 1 `Failed` AnalysisRun (Lab 7). `kubectl get analysisrun`: 2 Successful, 1 Failed. Unplanned: amd64-only images crash-looping on arm64 |
| Time to restore | **0.3 s** manual abort, **107 s** automated analysis abort, **304 s** `git revert` | Lab 7 and Lab 5 measurements |

Against DORA targets: frequency, lead time and restore time are at elite level; unplanned change failure rate is within 0–15%. The 29% figure is inflated by lab exercises that required shipping broken versions.

## 4. Top 3 Reliability Risks

1. **`events` is a single-replica bottleneck that fails hard.** One pod, 200m CPU limit, a 10-connection pool that throws instead of waiting. It caps the whole system at ~55 RPS, and its restart is a full outage. Fix: 3 replicas, a higher CPU limit, pool wait with a timeout, an HPA on CPU. Measured effect: 2.7× capacity, 0% errors.
2. **Postgres is a single instance and its backups share its fate.** The data PVC and the backup PVC are both `local-path` volumes on the same node, so losing the node loses both. RPO is 5 min at best, and no restore has ever been tested automatically. Fix: ship dumps off-cluster (object storage), add WAL archiving for point-in-time recovery, run a scheduled restore drill.
3. **The checkout path has no protection against a degraded payments service.** Retries and the circuit breaker are stubs, payments runs one replica, and Lab 8 showed a 2 s payment delay cutting throughput by 85% with zero errors and zero alerts. Fix: 2+ payments replicas, idempotent retry with backoff, a circuit breaker, a `/pay` latency alert.

## 5. Toil Identification

| Toil | How often | How to automate | Saved |
|---|---|---|---|
| Re-seeding Postgres (`psql < seed.sql`) after every pod or cluster restart | 4+ times (Lab 4 twice, Lab 7, Lab 9) | Done in Lab 9: PVC. Remaining: run schema + seed as an init Job / Alembic migration in CI | ~5 min each, plus debugging "Events service unavailable" when it was forgotten |
| Waiting on and babysitting deploys: `argocd app get --refresh`, `rollouts get --watch`, 3–5 min ArgoCD polls | Every deploy, 20+ times | GitHub webhook → ArgoCD for instant sync; rely on AnalysisTemplate + a notification on abort instead of watching | 3–5 min per deploy; `git revert` recovery 304 s → seconds |
| Rebuilding local access and state: port-forwards, switching kube namespace for `argocd --core`, restarting Docker/k3d after laptop sleep, toggling ArgoCD auto-sync around manual experiments | 10+ times | A `Makefile`/script for port-forwards and context; ArgoCD `selfHeal` plus doing experiments through Git branches instead of `kubectl apply` | ~2–3 min each and a class of "wrong namespace" mistakes |

## 6. Monitoring Gaps

Wished for during the Lab 8 chaos experiments:
- **Latency alerting.** Only error-rate and burn-rate alerts exist (Lab 6). The 2 s payments delay was invisible to them.
- **Metrics from events, payments, Postgres and Redis.** The in-cluster Prometheus scrapes only gateway, so DB pool saturation could not be confirmed in Lab 8 and was only found in the events log today.
- **Client-side view.** Gateway counters only see requests that reached a gateway. Lab 8 needed a separate curl client to prove no connections were dropped.

Still missing:
- **Alerting inside the cluster.** The Lab 6 alert rules live in the docker-compose Grafana, which cannot reach k3d pods. Nothing pages for the system that actually runs.
- **Persistent metrics.** Prometheus has no PVC; SLO history disappears on restart, so a 30-day error budget cannot be computed.
- **Backup freshness.** A failing backup CronJob would go unnoticed.

Alerts that would have caught what actually broke:
- `events` DB pool in use / max > 80% for 1 min → the capacity ceiling, before any 5xx.
- Container restarts > 0 in 5 min (`kube_pod_container_status_restarts_total`) → liveness kills of events (Lab 8 Redis outage, today's overload) and the Lab 5 crash loops.
- `/pay` p99 > 1 s for 2 min → the payments latency fault.
- Time since last successful backup > 15 min.

## 7. Capacity Plan

- **Current ceiling: ~55 RPS** (75 concurrent users), limited by the single `events` pod.
- **For 2× traffic (110 RPS):** scale `events` to 3 replicas. Measured: 149 RPS at p99 61 ms and 0% 5xx, so 110 RPS runs at ~74% of the proven level.
- **Rough cost:** 9 pods → 12 pods, **$45 → $60 per month** at $5/pod. Details in section 8.

## 8. Task 2 — Capacity Plan with Numbers

**Per-pod CPU at the breaking point** (`kubectl top pods`, 100 users, 40 s into the run):
```
events-7cd9fdc86c-4nv6z    200m    52Mi     ← at its 200m limit
gateway-6dcd8798dc-mqljx   59m     41Mi
gateway-6dcd8798dc-tvdqd   29m     40Mi
gateway-6dcd8798dc-vgxwr   42m     41Mi
gateway-6dcd8798dc-xbpss   32m     41Mi
gateway-6dcd8798dc-z957k   49m     42Mi
payments-6bb5cdc9c-4xwpt   9m      36Mi
postgres-dbb54497f-x5nvz   149m    29Mi
redis-c46d5dffc-kvdz2      6m      11Mi
```
`events` is the CPU-constrained service (100% of its limit). Gateway uses 15–30% of its limits, payments and Redis are idle, Postgres has room.

**Same check with events ×3 at 200 users / 149 RPS:**
```
events ×3    19m / 71m / 121m      (uneven: gateway keeps long-lived connections)
gateway ×5   50m – 76m each
postgres     215m
payments     15m        redis   6m
```

**Plan for 2× traffic (110 RPS):**

| Service | Now | Plan | Requests / limits | Reason |
|---|---|---|---|---|
| events | 1 | **3** | 100m / 500m CPU, 64Mi / 256Mi | The bottleneck. 3 replicas measured at 149 RPS; also removes the single point of failure. Limit raised so one hot pod is not throttled at 200m |
| gateway | 5 | 5 | 50m / 200m (unchanged) | ≤ 76m per pod at 149 RPS. 3 would be enough on CPU; 5 kept for 20% canary steps |
| payments | 1 | **2** | 50m / 200m (unchanged) | Idle on CPU. Second replica is for availability: Lab 8 showed a 503 on every restart |
| postgres | 1 | 1 | add 250m / 1 CPU, 256Mi / 512Mi | 215m at 149 RPS with no requests set today, so the scheduler guarantees it nothing |
| redis | 1 | 1 | unchanged | 6m CPU, 11Mi |

- **Redis:** a single pod is still fine. Load is negligible and it only holds reservations with a 5-minute TTL. It remains a single point of failure for `/reserve`; since Lab 8 an outage no longer takes browsing down. Replication (Sentinel) is worth it only when reservation downtime becomes unacceptable.
- **DB connections:** not a bottleneck at this size. 3 events pods × 10 connections = 30 of Postgres' default `max_connections = 100`. The real problem is that the pool fails instead of waiting. A pooler (PgBouncer) becomes necessary at ~9 events pods or if `DB_MAX_CONNS` is raised.
- **Cost:** 12 pods × $5 = **$60/month**, up from $45. That is +33% cost for 2.7× measured capacity.
- **Next limit:** Postgres CPU (215m at 149 RPS, single instance). Beyond ~300 RPS it needs a bigger instance or read replicas for `/events`.

## Bonus — SRE Handbook

Option B: [`submissions/runbooks/quickticket-handbook.md`](runbooks/quickticket-handbook.md).
