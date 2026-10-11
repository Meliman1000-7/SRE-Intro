# Lab 9 — Stateful Services & DB Reliability

Setup: `mixedload` (2 replicas, ~16.7 RPS) ran during every step. Alembic 1.18.4 ran from the host through `kubectl port-forward svc/postgres 5432:5432`. All times are UTC, 2026-10-11.

## Task 1 — Migrations & Backup/Restore

**1. `alembic history`:**
```
83c8377a61dc -> f8fd975e1b76 (head), add email column to events
<base> -> 83c8377a61dc, baseline - pre-existing schema
```
The baseline is an empty revision applied with `alembic stamp head`, so Alembic treats the `seed.sql` schema as already in place.

**2. `\d events` after the migration:**
```
    Column     |           Type           | Nullable |              Default
---------------+--------------------------+----------+------------------------------------
 id            | integer                  | not null | nextval('events_id_seq'::regclass)
 name          | text                     | not null |
 venue         | text                     | not null |
 event_date    | timestamp with time zone | not null |
 total_tickets | integer                  | not null |
 price_cents   | integer                  | not null |
 email         | character varying(255)   |          |
```

**3. `time alembic upgrade head`:**
```
INFO  [alembic.runtime.migration] Running upgrade 83c8377a61dc -> f8fd975e1b76, add email column to events
0.20s user 0.05s system 65% cpu 0.392 total
```

**4. 5xx around the migration** (applied at 15:23:23):
```
at 15:23:22 (before):  5xx last 1min: 0   rps: 16.75
at 15:24:22 (after):   5xx last 1min: 0   rps: 16.71
5xx over the 5 min around it: 0
```
Zero impact: adding a nullable column is a metadata-only change, so it needs no table rewrite and holds its lock for milliseconds.

**5. Backup** (taken at 15:24:51):
```
168K quickticket.dump
quickticket.dump: PostgreSQL custom database dump - v1.16-0

$ pg_restore --list /tmp/backup.dump
; Archive created at 2026-10-11 15:24:51 UTC
;     dbname: quickticket
;     TOC Entries: 18
;     Compression: gzip
;     Format: CUSTOM
;     Dumped from database version: 17.11
220; 1259 16423 TABLE public alembic_version quickticket
218; 1259 16390 TABLE public events quickticket
217; 1259 16389 SEQUENCE public events_id_seq quickticket
219; 1259 16398 TABLE public orders quickticket
3474; 0 16423 TABLE DATA public alembic_version quickticket
3472; 0 16390 TABLE DATA public events quickticket
3473; 0 16398 TABLE DATA public orders quickticket
3320; 2606 16397 CONSTRAINT public events events_pkey quickticket
3322; 2606 16406 CONSTRAINT public orders orders_pkey quickticket
3325; 2606 16407 FK CONSTRAINT public orders orders_event_id_fkey quickticket
```
The dump was restored into a scratch database to count its rows exactly: **4552 orders**.

**6. Row counts:**

| Moment | events | orders | API |
|---|---:|---:|---|
| Before disaster (15:24:52) | 5 | 4558 | 200 |
| After `DROP TABLE orders CASCADE` | 5 | `relation "orders" does not exist` | `/events` 502, `/reserve` 500 |
| After `pg_restore --clean --if-exists` (15:24:57) | 5 | 4553 | `/events` 200, `/reserve` 200 |

The restore itself took under 1 s. The dump held 4552 orders; the 4553rd was written by mixedload right after the restore. The 6 orders created in the 1 s between the dump and the drop were lost.

**7. What's the RPO of the current setup (single `pg_dump`)? How to improve it?**
RPO equals the age of the last dump, and with one manual dump that is unbounded: everything written after 15:24:51 is gone on the next failure. At ~5.6 orders/s, even 1 s cost 6 orders. Improvements, in order:
- a scheduled `pg_dump` CronJob, which caps RPO at the schedule interval (5 min in the Bonus);
- a PVC, so a pod restart loses nothing at all;
- WAL archiving / point-in-time recovery or a streaming replica, for an RPO of seconds against volume loss.

## Task 2 — Disaster Recovery Under Load

`kubectl delete pod -l app=postgres --grace-period=0 --force`, then restore from the 15:24:51 dump and restart events.

**Timestamps:**
```
healthy at 15:25:34, orders: 4762

Disaster at      15:25:34
New pod ready    15:25:35   (+1s)
Restored         15:25:36   (+2s)
App fully up     15:25:47   (+13s)
```
New pod contents before the restore: `Did not find any relations.`

- **Actual RTO: 13 s.** Only 2 s were Postgres itself; the other 11 s were the `events` rollout restart needed to drop stale DB connections. This holds only with a human ready with the dump and the commands — unattended, the outage lasts until someone notices.
- **Actual RPO: 43 s** (dump at 15:24:51, disaster at 15:25:34). **4762 orders before, 4552 after restore: 210 paid orders lost.**

**5xx rate around the incident** (`sum(rate(gateway_requests_total{status=~"5.."}[30s]))`):
```
15:25:30  0.000
15:25:40  0.040
15:25:50  0.160
15:26:00  0.206
15:26:10  0.121
15:26:20  0.000
```
About 5.5 failed requests in total. The count is low because both mixedload clients are sequential and spent the outage blocked on slow requests.

**The new Postgres pod was empty. Why? How to eliminate this failure mode?**
The Deployment had no volume, so the data directory lived in the container's writable layer and died with the pod. The replacement pod ran `initdb` on an empty directory. It became `Ready` in 1 s and looked healthy to Kubernetes while holding zero tables. Fix: mount a PersistentVolumeClaim on the data directory, so the data outlives the pod (Bonus). Backups are then needed only for volume loss and human error such as the `DROP TABLE` above.

## Bonus — Persistent Storage + Automated Backups

**`k8s/postgres.yaml` diff:**
```diff
 spec:
   replicas: 1
+  strategy:
+    type: Recreate
   selector:
 ...
             - name: POSTGRES_PASSWORD
               value: "quickticket"
+            - name: PGDATA
+              value: "/var/lib/postgresql/data/pgdata"
+          volumeMounts:
+            - name: data
+              mountPath: /var/lib/postgresql/data
+      volumes:
+        - name: data
+          persistentVolumeClaim:
+            claimName: postgres-data
+---
+apiVersion: v1
+kind: PersistentVolumeClaim
+metadata:
+  name: postgres-data
+spec:
+  accessModes: [ReadWriteOnce]
+  resources:
+    requests:
+      storage: 1Gi
```
`strategy: Recreate` was added so a rolling update never starts a second Postgres on the same data directory. The fresh volume was filled from a `pg_dump` of the live DB instead of `seed.sql`, which kept the `email` column and `alembic_version`.

**Same disaster test with the PVC** (in-cluster client polling `/events` every 0.5 s):
```
healthy at 15:27:34, orders: 4991

Disaster at        15:27:35
New pod ready      15:27:36 (+1s)
Postgres accepting 15:27:36 (+1s)
\dt → alembic_version, events, orders   (3 rows)
orders right after restart: 4993

poller: 128× 200, 1× 502, 3× timeout
non-200 window: 15:27:35 → 15:27:40
```

| | No PVC | With PVC |
|---|---|---|
| RTO | 13 s (restore + events restart, manual) | **~6 s** (automatic) |
| RPO | 43 s / 210 orders lost | **0** |
| Human action | `kubectl cp`, `pg_restore`, `rollout restart` | none |

events reconnected on its own; no restart was needed.

**`k8s/backup-cronjob.yaml`:**
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: postgres-backup
spec:
  schedule: "*/5 * * * *"
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 3
  jobTemplate:
    spec:
      backoffLimit: 2
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: backup
              image: postgres:17-alpine
              env:
                - name: PGHOST
                  value: "postgres"
                - name: PGUSER
                  value: "quickticket"
                - name: PGDATABASE
                  value: "quickticket"
                - name: PGPASSWORD
                  value: "quickticket"
              command: ["sh", "-c"]
              args:
                - |
                  set -e
                  cd /backups
                  FILE="quickticket_$(date -u +%Y%m%dT%H%M%SZ).dump"
                  pg_dump -Fc -f "$FILE.tmp"
                  mv "$FILE.tmp" "$FILE"
                  echo "created $FILE ($(du -h "$FILE" | cut -f1))"
                  ls -1t quickticket_*.dump | tail -n +6 | xargs -r rm -v
                  echo "kept: $(ls -1 quickticket_*.dump | wc -l) dumps"
              volumeMounts:
                - name: backups
                  mountPath: /backups
          volumes:
            - name: backups
              persistentVolumeClaim:
                claimName: postgres-backups
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-backups
  ...
```
- The dump is written to `.tmp` and renamed, so a failed `pg_dump` never leaves a half-written file that rotation would count as a valid backup.
- The `postgres-backups` PVC is declared in the same file, so the manifest works on its own when ArgoCD syncs `k8s/`.

**`kubectl logs job/manual-7`:**
```
created quickticket_20261011T152931Z.dump (208.0K)
removed 'quickticket_20261011T152910Z.dump'
kept: 5 dumps
```

**`ls -la /backups` after 7 runs:**
```
-rw-r--r--    1 root     root        207195 Oct 11 15:29 quickticket_20261011T152914Z.dump
-rw-r--r--    1 root     root        208003 Oct 11 15:29 quickticket_20261011T152919Z.dump
-rw-r--r--    1 root     root        208879 Oct 11 15:29 quickticket_20261011T152923Z.dump
-rw-r--r--    1 root     root        209684 Oct 11 15:29 quickticket_20261011T152927Z.dump
-rw-r--r--    1 root     root        210588 Oct 11 15:29 quickticket_20261011T152931Z.dump
```
Exactly 5 files. `manual-1` … `manual-4` were already garbage-collected by `successfulJobsHistoryLimit: 3`.

**Scheduled run** (no manual trigger) fired at 15:30:01 and rotated again:
```
postgres-backup-29862210   Complete   1/1   3s
quickticket_20261011T152919Z.dump … quickticket_20261011T153001Z.dump   (5 files)
```

With both changes, a pod restart loses nothing, and volume loss or a bad `DROP` loses at most 5 minutes. The CronJob and its PVC were left running; only `mixedload`, `backup-inspector` and the port-forward were cleaned up.
