# Drill 5.4 — Disk full

| Field | Value |
|---|---|
| **Drill ID** | `B.4` |
| **Runbook ref** | `docs/13_OPERATIONS.md §5.4` |
| **Risk** | **High** — real disk pressure on shared host. Mis-sized balloon can cause real outage. |
| **Est duration** | 45 min (15 pre-flight + math + 10 induce/observe + 10 recover/compress + 10 verify/log) |
| **Maintenance window** | recommended (off-peak) |
| **Neighbour notify** | not required for ≤ 88% target — but inform if ad-hoc spike to 90%+ |
| **Owner** | <fill in before run> |

---

## 1. Scope & Goal

Inflate `/var/lib/postgresql` usage to **81–85%** (without crossing
88%) using a regular file balloon, and verify:

1. Alertmanager fires `DBDiskUsageHigh` within ~5 min (per `12 §3.4`,
   threshold 80%).
2. The runbook's force-compress procedure (`13 §5.4 step 5`) reduces
   chunk size noticeably.
3. After removing the balloon, usage returns to baseline ± 1% and
   `DBDiskUsageHigh` clears.

**Non-goals:** we do NOT actually fill the disk to ≥ 95%. PostgreSQL
write protection kicks in around 95% and that's a real incident, not a
drill. The balloon is a regular file removable with `rm`.

---

## 2. Pre-flight

- [ ] Maintenance window scheduled (recommended off-peak — peak +
      balloon could push real OI growth into the danger zone).
- [ ] Latest `pg_dump` verified (`13 §9.2`).
- [ ] Capture **current** `df -h /var/lib/postgresql` BEFORE math.
      The balloon size depends on it.
- [ ] **Math (REQUIRED — paste your numbers into the drill log):**

      Let `total_kb`, `used_kb`, `avail_kb` be from `df -P
      /var/lib/postgresql`. Current usage:

      ```
      pct_now = used_kb / total_kb
      ```

      Target: **82%**. Compute balloon size in MB:

      ```
      target_used_kb = total_kb * 0.82
      delta_kb       = target_used_kb - used_kb
      balloon_mb     = delta_kb / 1024     # round DOWN
      ```

      **Hard caps (drill aborts if any are violated):**
      - `pct_now ≥ 0.78` — too close already, do not run.
      - `balloon_mb < 500` — drill won't move the needle, skip.
      - `target_used_kb / total_kb > 0.85` — recompute, target lower.
      - `total_kb < 50_000_000` (≈ 50 GB) — host too small, skip drill
        on this host.

- [ ] Calculator output written to drill log §9 BEFORE running `dd`.
- [ ] No active `DBDiskUsageHigh` alert.
- [ ] Baseline §3 captured.
- [ ] Recovery (§6) and Rollback (§8) re-read.
- [ ] You have `sudo` and `postgres`-user shell.

---

## 3. Baseline capture

```bash
date -Iseconds

# Disk usage:
df -h /var/lib/postgresql
df -P /var/lib/postgresql | awk 'NR==2 {printf "total_kb=%s used_kb=%s avail_kb=%s pct=%s\n", $2, $3, $4, $5}'

# DB size + biggest chunks:
sudo -u postgres psql oi_tracker -c "
  SELECT pg_size_pretty(pg_database_size('oi_tracker')) AS db_size;
"
sudo -u postgres psql oi_tracker -c "
  SELECT chunk_name, is_compressed,
         pg_size_pretty(pg_total_relation_size(format('%I.%I', chunk_schema, chunk_name)::regclass)) AS sz
  FROM timescaledb_information.chunks
  WHERE hypertable_name='oi_samples'
  ORDER BY range_start DESC LIMIT 10;
"

# Active alerts:
curl -s 'http://127.0.0.1:9093/api/v2/alerts?active=true' \
    | jq '.[] | select(.labels.alertname=="DBDiskUsageHigh")'
# Expected: empty.

# Disk-usage metric (Prometheus side, drives the alert):
curl -sG 'http://127.0.0.1:9090/api/v1/query' \
    --data-urlencode 'query=node_filesystem_avail_bytes{mountpoint="/var/lib/postgresql"} / node_filesystem_size_bytes{mountpoint="/var/lib/postgresql"}'
```

---

## 4. Induce procedure

> Approach: regular file balloon in `/var/lib/postgresql` (same FS as
> data). Single `dd` write, single `rm` to remove. PostgreSQL itself
> is not modified.

1. **t=0** — Re-verify caps from §2 pre-flight math. Have you written
   `balloon_mb` to drill log? If not, stop.
2. **t=+1m** — Inflate balloon (replace `<N>` with your `balloon_mb`):
   ```bash
   N=<balloon_mb>      # paste number from your math
   sudo -u postgres dd if=/dev/zero of=/var/lib/postgresql/drill-5.4-balloon.bin \
       bs=1M count="$N" status=progress
   sync
   ```
3. **t=+1m30s** — Verify usage moved to ~82%:
   ```bash
   df -h /var/lib/postgresql
   ```
   If usage > 88% — **abort**: `sudo rm /var/lib/postgresql/drill-5.4-balloon.bin`
   and recompute math.
4. **t=+2m ... +6m** — Observe (see §5).
5. **t=+6m** — Optional: practice the force-compress mitigation
   (`13 §5.4 step 5`):
   ```sql
   -- As postgres user:
   SELECT compress_chunk(c)
   FROM show_chunks('oi_samples', older_than => INTERVAL '7 days') c
   WHERE c::regclass::text NOT IN (
     SELECT format('%I.%I', chunk_schema, chunk_name)
     FROM timescaledb_information.chunks
     WHERE is_compressed = true AND hypertable_name='oi_samples'
   );
   ```
   Capture how much disk this reclaimed (often 5–10× compression).
6. **t=+10m** — Stop drill, proceed to recovery (§6).

---

## 5. Expected observations

### Within 1 min of `dd` finishing
- `df -h` shows ~82% on `/var/lib/postgresql`.
- node_exporter scrape picks up new value:
  ```bash
  curl -sG 'http://127.0.0.1:9090/api/v1/query' \
      --data-urlencode 'query=1 - node_filesystem_avail_bytes{mountpoint="/var/lib/postgresql"} / node_filesystem_size_bytes{mountpoint="/var/lib/postgresql"}'
  # Expected: ~0.82
  ```

### Within 5 min
- `DBDiskUsageHigh` fires:
  ```bash
  curl -s 'http://127.0.0.1:9093/api/v2/alerts?active=true' \
      | jq '.[] | select(.labels.alertname=="DBDiskUsageHigh")'
  ```
- Severity should be `warning` at 80% threshold (per `12 §3.4`).

### Optional (force-compress step)
- After `compress_chunk` SQL — `pg_total_relation_size` for compressed
  chunks shrinks 5–10×. Recheck:
  ```bash
  sudo -u postgres psql oi_tracker -c "
    SELECT count(*) FILTER (WHERE is_compressed) AS compressed,
           count(*) FILTER (WHERE NOT is_compressed) AS uncompressed
    FROM timescaledb_information.chunks WHERE hypertable_name='oi_samples';
  "
  ```

### No regression
- Other oi-tracker services keep `active`.
- Sample insertion unaffected:
  ```bash
  curl -sG 'http://127.0.0.1:9090/api/v1/query' \
      --data-urlencode 'query=rate(oi_storage_insert_duration_seconds_count[2m])'
  # Expected: rate ≈ baseline.
  ```

---

## 6. Recovery procedure

```bash
# 1. Remove balloon (atomic):
sudo rm /var/lib/postgresql/drill-5.4-balloon.bin
sync

# 2. Verify usage back to baseline:
df -h /var/lib/postgresql

# 3. (Optional) If you ran force-compress in §4 step 5, the savings
#    are PERMANENT and beneficial — leave them. Do NOT decompress.
#    Note this in the drill log so future-you knows what changed.
```

**RTO target:** ≤ 30 sec from `rm` to df reading at baseline ± 1%.

---

## 7. Post-drill verification

```bash
# Disk usage at baseline:
df -h /var/lib/postgresql

# Balloon file gone:
sudo test -f /var/lib/postgresql/drill-5.4-balloon.bin && echo STILL_PRESENT || echo gone_ok
# Expected: gone_ok

# Disk-usage metric back below 80%:
curl -sG 'http://127.0.0.1:9090/api/v1/query' \
    --data-urlencode 'query=1 - node_filesystem_avail_bytes{mountpoint="/var/lib/postgresql"} / node_filesystem_size_bytes{mountpoint="/var/lib/postgresql"}'
# Expected: < 0.80

# Alert cleared:
curl -s 'http://127.0.0.1:9093/api/v2/alerts?active=true' \
    | jq '.[] | select(.labels.alertname=="DBDiskUsageHigh")'
# Expected: empty.

# All services healthy:
sudo systemctl is-active oi-tracker-scheduler oi-tracker-api oi-tracker-tg-sender postgresql

# Sample insertion still flowing:
sudo -u postgres psql oi_tracker -c "
  SELECT exchange, max(ts_processed) FROM oi_samples
  WHERE ts_processed >= now() - interval '5 min'
  GROUP BY 1 ORDER BY 1;
"
```

---

## 8. Rollback (emergency abort)

If at any point disk usage crosses 88%, OR a real `DBDown` /
write-failure alert fires:

```bash
# Atomic abort:
sudo rm -f /var/lib/postgresql/drill-5.4-balloon.bin
sync
df -h /var/lib/postgresql
```

If postgres has already taken write protection action (rare at <95%
but possible with WAL pressure):

```bash
# After balloon removed, check postgres writability:
sudo -u postgres psql -d oi_tracker -c "INSERT INTO settings (key, value) VALUES ('drill_5_4_check', '1'::jsonb) ON CONFLICT DO NOTHING;"
# Should succeed. If "disk full" error — page on-call, treat as incident.
```

---

## 9. Drill log template

Copy into `deploy/drills/logs/<YYYY-MM-DD>-disk-full.md`:

```markdown
# Drill 5.4 — <date>

- **Operator:** <name>
- **Start (UTC):** <ISO>
- **End (UTC):** <ISO>
- **Total duration:** <minutes>
- **Result:** PASS / FAIL / PARTIAL / **ABORTED**

## Pre-flight math
- total_kb = <X>
- used_kb  = <X>
- pct_now  = <X>
- target_used_kb = total_kb * 0.82 = <X>
- delta_kb       = target_used_kb - used_kb = <X>
- **balloon_mb   = <X>**
- All hard caps: PASSED

## Baseline (§3 output)
<paste>

## Induce log
- t=0    : math validated, balloon_mb = <N>
- t=+1m  : dd start (paste output incl. status=progress final line)
- t=+1m30s : df after dd (paste)
- t=+5m  : DBDiskUsageHigh fired (paste alertmanager JSON)
- t=+6m  : (optional) compress_chunk SQL run; chunks compressed = <N→M>
- t=+10m : drill stop, proceeding to recovery

## Expected vs actual
| Expectation | Expected | Actual | Verdict |
|---|---|---|---|
| Usage ≈ 82% after dd | 0.80–0.85 | <X> | OK / FAIL |
| DBDiskUsageHigh fires within 5 min | yes | <t=<Xs>> | OK / FAIL |
| Force-compress reduces chunk size | yes | <ratio> | OK / N/A |
| No regression in inserts | rate ≈ baseline | <X vs Y> | OK / FAIL |
| Other services unaffected | active | <Y/N> | OK / FAIL |

## Recovery (§6 output)
<paste df after rm>
RTO actual: <Xs> (budget: 30s)

## Post-drill verification (§7 output)
<paste>

## Findings
- <surprises / unexpected lag in node_exporter scrape>
- <whether force-compress reclaim matched expectation>
- Compressed chunks created during drill: <list> (these are permanent and good)

## Lessons added (if any)
- `tasks/lessons.md` — <pattern>
```

---

## 10. Cross-references

- Runbook: `docs/13_OPERATIONS.md §5.4`
- Dev plan: `tasks/development_plan.md §M5.B.4`
- Alert rule: `deploy/observability/prometheus/rules/oi-tracker.yml` —
  group `oi-tracker.storage` → `DBDiskUsageHigh`
- Metrics: `docs/12_OBSERVABILITY_SLO.md §3.4` (storage), node_exporter
  filesystem metrics
- TimescaleDB: `docs/08_TIME_SERIES_STORAGE.md §6` (compression
  policies), `docs/08 §7` (retention)

---

## 11. History

| Date | Operator | Result | Log |
|---|---|---|---|
| <YYYY-MM-DD> | <name> | <PASS/FAIL/ABORTED> | [link](logs/<file>.md) |
