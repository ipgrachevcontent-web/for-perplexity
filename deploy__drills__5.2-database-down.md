# Drill 5.2 — Database down

| Field | Value |
|---|---|
| **Drill ID** | `B.2` |
| **Runbook ref** | `docs/13_OPERATIONS.md §5.2` |
| **Risk** | **High** — touches shared PostgreSQL serving `detector` and `grach-ege` neighbours |
| **Est duration** | 30 min (10 pre-flight + 5 induce + 5 recover + 5 verify + 5 log) |
| **Maintenance window** | **required** |
| **Neighbour notify** | **required** — `detector`, `grach-ege` |
| **Owner** | <fill in before run> |

---

## 1. Scope & Goal

Stop PostgreSQL for ~30 seconds and verify that:

1. oi-tracker's three backend services (scheduler, api, tg-sender) hit
   `connection refused` and exit / restart-loop via systemd.
2. systemd `Restart=on-failure` brings them back up within their
   restart backoff window.
3. After PostgreSQL returns, all three services reconnect and resume
   normal collection / serving without manual intervention beyond what
   `13 §5.2 step 5` documents.
4. Alertmanager fires `DBDown` (or `ConnectorDown` cascade) within
   evaluation window, and clears after recovery.

The drill is **destructive to neighbour services** for ~30s. It must
run inside a coordinated maintenance window.

---

## 2. Pre-flight

- [ ] Maintenance window scheduled (≥ 5 min slot).
- [ ] **Neighbours notified explicitly** — `detector` owner and
      `grach-ege` owner have ACK'd the planned ~30s outage. Notify in
      writing (Slack/email) at least 1h ahead.
- [ ] `pg_dump` ran successfully within last 24h (recovery safety net).
      Manual snapshot if not:
      ```bash
      sudo -u postgres pg_dump --format=custom --file=/tmp/pre-drill-5.2.dump oi_tracker
      ```
- [ ] All 3 services are currently `active`:
      ```bash
      sudo systemctl is-active oi-tracker-scheduler oi-tracker-api oi-tracker-tg-sender
      ```
- [ ] No active critical alerts in Alertmanager (so we don't conflate).
- [ ] Baseline §3 captured.
- [ ] Recovery (§6) and Rollback (§8) re-read.
- [ ] You have `sudo systemctl` for postgresql + oi-tracker-*.
- [ ] Stopwatch ready — RTO budget is tight (≤ 2 min from `start
      postgresql` to all 3 services healthy).

---

## 3. Baseline capture

```bash
date -Iseconds

# Service state:
sudo systemctl is-active oi-tracker-scheduler oi-tracker-api oi-tracker-tg-sender postgresql

# PostgreSQL up + accepting connections:
sudo -u postgres psql -d oi_tracker -c 'SELECT now(), pg_is_in_recovery();'

# Connection count (so we know what "back to normal" looks like):
sudo -u postgres psql -c "
  SELECT datname, count(*) FROM pg_stat_activity
  WHERE datname IN ('oi_tracker','detector','grach_ege')
  GROUP BY 1 ORDER BY 1;
"

# Active alerts:
curl -s 'http://127.0.0.1:9093/api/v2/alerts?active=true' \
    | jq '[.[] | {alertname: .labels.alertname, severity: .labels.severity, state: .status.state}]'

# Service log tails (so deviations stand out):
journalctl -u oi-tracker-scheduler.service --since='2 min ago' --no-pager | tail -5
journalctl -u oi-tracker-api.service       --since='2 min ago' --no-pager | tail -5
journalctl -u oi-tracker-tg-sender.service --since='2 min ago' --no-pager | tail -5
```

---

## 4. Induce procedure

1. **t=0** — Stop PostgreSQL:
   ```bash
   sudo systemctl stop postgresql
   ```
2. **t=+5s** — Confirm down:
   ```bash
   sudo systemctl is-active postgresql
   # MUST: inactive
   sudo -u postgres psql -d oi_tracker -c 'SELECT 1;' 2>&1 | head -1
   # MUST: connection refused / "could not connect"
   ```
3. **t=+30s** — Stop drill, immediately proceed to recovery (§6).
   **Do not exceed 30s** without explicit reason — neighbour impact
   compounds.

---

## 5. Expected observations

> Capture during the 30s induce window — do not delay recovery to
> finish capturing.

### oi-tracker services
- Within 5–15s, scheduler / api / tg-sender hit DB error. Look at
  `journalctl -u oi-tracker-* -f` and grep for `connection refused`,
  `OperationalError`, or `asyncpg.exceptions.CannotConnectNowError`.
- systemd starts restart-looping (`Restart=on-failure` per
  `13 §3.X`):
  ```bash
  systemctl show oi-tracker-scheduler.service --property=NRestarts
  # Counter increments while postgres is down.
  ```

### Metrics
- Scrape errors on `/metrics` endpoints (services may be mid-restart):
  ```bash
  curl -sG 'http://127.0.0.1:9090/api/v1/query' \
      --data-urlencode 'query=up{job=~"oi-tracker.*"}'
  # 0 for at least one service during the window.
  ```
- `oi_storage_insert_duration_seconds_count` rate stops climbing.

### Alertmanager
- Within 1–2 min, expect at least one of:
  - `DBDown` (if defined as a synthetic check on `up{job="postgres"}`)
  - `ConnectorDown` cascade (scheduler can't write samples → all
    exchanges' `oi_collector_request_total{status="ok"}` rate drops)
  - `OITrackerServiceDown` (severity=critical) for the affected
    oi-tracker-* services
  ```bash
  curl -s 'http://127.0.0.1:9093/api/v2/alerts?active=true' \
      | jq '[.[] | .labels.alertname]' | sort -u
  ```

### Neighbours
- `detector` and `grach-ege` will also see `connection refused`. Their
  recovery is **out of scope** for this drill — we just confirm they
  recover after `start postgresql` returns. Sanity check at end:
  ```bash
  sudo systemctl status detector grach-ege 2>/dev/null | grep -E 'Active'
  ```

---

## 6. Recovery procedure

```bash
# 1. Bring postgres back:
sudo systemctl start postgresql

# 2. Verify accepting connections:
sudo -u postgres psql -d oi_tracker -c 'SELECT now();'

# 3. Per `13 §5.2 step 5` — restart oi-tracker services so they
#    reattach (some pools don't recover from cold connection refusal):
sudo systemctl restart oi-tracker-scheduler oi-tracker-api oi-tracker-tg-sender
```

**RTO target:** ≤ 2 min from `start postgresql` to all 3 oi-tracker
services back to `active` and serving traffic.

---

## 7. Post-drill verification

```bash
# All four services active:
sudo systemctl is-active oi-tracker-scheduler oi-tracker-api oi-tracker-tg-sender postgresql
# Expected: active × 4

# Health endpoint:
sudo -u oi-tracker curl -sf --unix-socket /run/oi-tracker/api.sock \
    http://localhost/api/v1/health
# Expected: HTTP 200, {"status":"ok"}

# Connection count back to baseline ± 20%:
sudo -u postgres psql -c "
  SELECT datname, count(*) FROM pg_stat_activity
  WHERE datname IN ('oi_tracker','detector','grach_ege')
  GROUP BY 1 ORDER BY 1;
"

# Sample insertion resumed:
sudo -u postgres psql -d oi_tracker -c "
  SELECT exchange, max(ts_processed)
  FROM oi_samples
  WHERE ts_processed >= now() - interval '5 min'
  GROUP BY 1 ORDER BY 1;
"
# Expected: rows for all 12 exchanges, max(ts_processed) within last 90s.

# Alertmanager cleared:
curl -s 'http://127.0.0.1:9093/api/v2/alerts?active=true' \
    | jq '[.[] | .labels.alertname]'
# Expected: empty (or only pre-existing unrelated alerts from baseline).

# Neighbours OK:
sudo systemctl status detector grach-ege 2>/dev/null | grep -E 'Active'
# Expected: active (running).
```

---

## 8. Rollback (emergency abort)

If `start postgresql` fails (corrupted WAL, OOM at boot, etc):

```bash
# 1. Diagnose immediately:
sudo journalctl -u postgresql -n 100 --no-pager

# 2. If WAL corruption / startup failure — escalate to PostgreSQL
#    playbook `13 §5.2 step 4`. Most likely action: pg_resetwal
#    (data loss possible) or restore from latest pg_dump (RTO ≤ 2h
#    per `13 §9.1`). DO NOT keep retrying `start` blindly.

# 3. If neighbours are screaming — they have their own runbooks; do
#    not touch their data. They reconnect once postgres is back.
```

If we cannot bring postgres back within 5 min, page on-call and treat
this as a **real incident**, not a drill. Stop the drill log; open an
incident timeline.

---

## 9. Drill log template

Copy into `deploy/drills/logs/<YYYY-MM-DD>-database-down.md`:

```markdown
# Drill 5.2 — <date>

- **Operator:** <name>
- **Maintenance window start:** <ISO>
- **Maintenance window end:** <ISO>
- **Result:** PASS / FAIL / PARTIAL

## Pre-flight
- [x] Maintenance window confirmed
- [x] detector owner ACK at <timestamp>
- [x] grach-ege owner ACK at <timestamp>
- [x] pg_dump verified <timestamp>

## Baseline (§3 output)
<paste>

## Induce log
- t=0    : systemctl stop postgresql (paste output)
- t=+5s  : connection refused confirmed (paste output)
- t=+15s : first scheduler error in logs (paste line)
- t=+30s : systemctl start postgresql

## Expected vs actual
| Expectation | Expected | Actual | Verdict |
|---|---|---|---|
| Services restart-loop | NRestarts ↑ | <X→Y> | OK / FAIL |
| up{job="oi-tracker.*"} = 0 during window | true | <Y/N> | OK / FAIL |
| Recovery RTO | ≤ 120s | <Xs> | OK / FAIL |
| Health endpoint 200 after recovery | yes | <Y/N> | OK / FAIL |
| All 12 exchanges resume insertion | yes | <Y/N> | OK / FAIL |
| Neighbours recover | yes | <Y/N> | OK / FAIL |
| Alertmanager fires + clears | yes | <Y/N> | OK / FAIL |

## Recovery (§6 output)
<paste>
RTO actual: <Xs> (budget: 120s)

## Post-drill verification (§7 output)
<paste>

## Findings
- <surprises / unexpected dependencies>
- <whether `Restart=on-failure` backoff was reasonable>

## Lessons added (if any)
- `tasks/lessons.md` — <pattern>

## Neighbour incident reports
- detector: <none / link to issue>
- grach-ege: <none / link to issue>
```

---

## 10. Cross-references

- Runbook: `docs/13_OPERATIONS.md §5.2`
- Dev plan: `tasks/development_plan.md §M5.B.2`
- systemd units: `deploy/systemd/*.service` — `Restart=on-failure`
- Alert rules: `deploy/observability/prometheus/rules/oi-tracker.yml` —
  groups `oi-tracker.collectors`, `oi-tracker.storage`
- Backups (recovery safety net): `docs/13_OPERATIONS.md §9`,
  `deploy/backups/`
- Decisions: `docs/00_DECISIONS_LOG.md F14` (UDS + ownership)

---

## 11. History

| Date | Operator | Result | Log |
|---|---|---|---|
| <YYYY-MM-DD> | <name> | <PASS/FAIL> | [link](logs/<file>.md) |
