# Drill 5.7 — SLO breach (E2E latency P99 > 180s)

| Field | Value |
|---|---|
| **Drill ID** | `B.7` |
| **Runbook ref** | `docs/13_OPERATIONS.md §5.7` |
| **Risk** | Med — risk depends on path (A staging vs B prod throttle). Path A is preferred. |
| **Est duration** | 30 min (5 pre-flight + 10 induce + 5 observe + 5 recover + 5 verify/log) |
| **Maintenance window** | recommended (off-peak); required for Path B on prod |
| **Neighbour notify** | not required (no shared resources affected) |
| **Owner** | <fill in before run> |

---

## 1. Scope & Goal

Push end-to-end latency above the SLO threshold (P99 > 180s per
`01_PRODUCT_SPEC.md NFR-1`) and verify:

1. `oi_e2e_latency_seconds_bucket` / `oi_alert_e2e_latency_seconds`
   histogram shifts right; P99 query reads > 180s within 5 min.
2. Alertmanager fires `E2ELatencyHigh` (severity warning per `12 §3.7`).
3. Runbook mitigation works: temporarily disabling `consensus` rule
   reduces alert engine load, P99 falls back below SLO.
4. After recovery, alert clears and metrics return to baseline.

**Two induction paths**, choose based on environment:
- **Path A — staging (preferred)**: deploy the codebase to staging
  with a one-line `time.sleep(0.5)` injected into the normalizer hot
  path. Cleaner signal, no prod risk.
- **Path B — prod (only if no staging)**: lower the consensus rule's
  consensus_min_exchanges threshold + bump evaluation cadence to
  artificially overload the alert engine. Reversible, but messier
  signal.

---

## 2. Pre-flight

- [ ] Decide path:
  - Staging instance available? → **Path A**.
  - No staging? → **Path B** with explicit user authorization (this
    drill mildly degrades prod for 5–10 min).
- [ ] If **Path A**: have a separate branch / patch ready —
      see §4 step 2 for the exact diff.
- [ ] If **Path B**: have settings backup ready:
      ```bash
      sudo -u postgres psql -d oi_tracker -tAc \
          "SELECT key,value FROM settings WHERE key IN ('consensus_min_exchanges','evaluation_cycle_sec')" \
          | tee /tmp/drill-5.7-settings.bak
      cat /tmp/drill-5.7-settings.bak
      ```
- [ ] Capture baseline (§3) — important to know what "back to normal"
      looks like.
- [ ] Recovery (§6) and Rollback (§8) re-read.
- [ ] No active `E2ELatencyHigh` alerts (else we conflate).

---

## 3. Baseline capture

```bash
date -Iseconds

# E2E latency P99 (last 5 min). Per `12 §3.7` the histogram is
# `oi_alert_e2e_latency_seconds` (alert delivery side) and
# per-stage histograms exist for collector / normalizer / storage.
curl -sG 'http://127.0.0.1:9090/api/v1/query' \
    --data-urlencode 'query=histogram_quantile(0.99, sum by (le) (rate(oi_alert_e2e_latency_seconds_bucket[5m])))' \
    | jq -r '.data.result[0].value[1] // "no_data"'
# Expected: < 90 (p99 SLO from NFR-1 is 180; healthy baseline ~30–60).

# Per-stage P99:
for stage in collector normalize storage alert_engine; do
  case $stage in
    collector)    METRIC=oi_collector_request_duration_seconds_bucket ;;
    normalize)    METRIC=oi_normalize_duration_seconds_bucket ;;
    storage)      METRIC=oi_storage_insert_duration_seconds_bucket ;;
    alert_engine) METRIC=oi_alert_engine_cycle_duration_seconds_bucket ;;
  esac
  echo -n "stage=$stage p99="
  curl -sG 'http://127.0.0.1:9090/api/v1/query' \
      --data-urlencode "query=histogram_quantile(0.99, sum by (le) (rate(${METRIC}[5m])))" \
      | jq -r '.data.result[0].value[1] // "no_data"'
done

# Active alerts:
curl -s 'http://127.0.0.1:9093/api/v2/alerts?active=true' \
    | jq '.[] | select(.labels.alertname=="E2ELatencyHigh")'
# Expected: empty.

# Settings (if Path B):
sudo -u postgres psql -d oi_tracker -c \
    "SELECT key,value FROM settings WHERE key IN ('consensus_min_exchanges','evaluation_cycle_sec','polling_interval_sec');"
```

---

## 4. Induce procedure

### Path A — staging deploy with injected sleep

1. **t=0** — Apply this one-line patch on the staging branch:
   ```diff
   --- a/backend/app/normalizer/pipeline.py
   +++ b/backend/app/normalizer/pipeline.py
   @@ <hot path inside normalize_one>
   +    import time; time.sleep(0.5)  # DRILL 5.7 — REMOVE BEFORE MERGE
   ```
2. **t=+2m** — Deploy to staging (per `13 §7`):
   ```bash
   # On staging host:
   sudo -iu oi-tracker
   cd /opt/oi-tracker/code/backend
   git pull origin drill-5.7-slo-breach
   uv sync
   exit
   sudo systemctl restart oi-tracker-scheduler oi-tracker-alert-engine
   ```
3. **t=+5m** — Observe (§5).
4. **t=+10m** — When `E2ELatencyHigh` fires and is captured, apply
   mitigation per runbook `13 §5.7` step 3 (disable consensus rule):
   ```bash
   sudo -u postgres psql -d oi_tracker -c "
     UPDATE alert_rules SET enabled=false WHERE rule_type='consensus';
   "
   ```
   Watch P99 trend down over the next 2–3 min.
5. **t=+15m** — Stop drill, proceed to recovery (§6).

### Path B — prod settings throttle (only if no staging)

1. **t=0** — Lower consensus threshold (artificially many alerts) +
   shrink evaluation cadence (more cycles per minute):
   ```bash
   sudo -u postgres psql -d oi_tracker -c "
     UPDATE settings SET value='2'::jsonb WHERE key='consensus_min_exchanges';
     UPDATE settings SET value='10'::jsonb WHERE key='evaluation_cycle_sec';
   "
   ```
   These settings are restart-required per F23, so:
   ```bash
   sudo systemctl restart oi-tracker-alert-engine oi-tracker-scheduler
   ```
2. **t=+5m** — Observe (§5). The alert engine cycle duration histogram
   should grow; E2E latency drift follows.
3. **t=+10m** — Apply mitigation per `13 §5.7` step 3 — disable the
   consensus rule itself:
   ```bash
   sudo -u postgres psql -d oi_tracker -c "
     UPDATE alert_rules SET enabled=false WHERE rule_type='consensus';
   "
   sudo systemctl restart oi-tracker-alert-engine
   ```
4. **t=+15m** — Stop drill, proceed to recovery (§6).

---

## 5. Expected observations

### Within 5 min
- E2E P99 climbs above SLO:
  ```bash
  curl -sG 'http://127.0.0.1:9090/api/v1/query' \
      --data-urlencode 'query=histogram_quantile(0.99, sum by (le) (rate(oi_alert_e2e_latency_seconds_bucket[5m])))'
  # Expected: > 180
  ```
- Per-stage histograms identify the bottleneck:
  - **Path A**: `oi_normalize_duration_seconds` P99 grows by ~500ms
    per sample (the injected sleep).
  - **Path B**: `oi_alert_engine_cycle_duration_seconds` P99 grows.

### Within 5–10 min
- Alertmanager fires `E2ELatencyHigh` (warning):
  ```bash
  curl -s 'http://127.0.0.1:9093/api/v2/alerts?active=true' \
      | jq '.[] | select(.labels.alertname=="E2ELatencyHigh")'
  ```

### After mitigation (consensus disabled)
- Within 2–3 min, P99 recedes.
- `oi_alert_engine_cycle_duration_seconds` cycle count by `rule_type`
  drops the consensus contribution:
  ```promql
  sum by (decision) (rate(oi_alert_engine_decisions_total{rule_id=~".*consensus.*"}[2m]))
  ```
  Expected: 0 (no decisions for consensus rule any more).

### No regression
- Normal alert types (threshold, divergence, exchange_health) still
  fire if their conditions are met:
  ```bash
  curl -sG 'http://127.0.0.1:9090/api/v1/query' \
      --data-urlencode 'query=sum by (signal_type) (rate(oi_alert_engine_fires_total[5m]))'
  ```
- No data loss in `oi_samples`:
  ```sql
  SELECT exchange, max(ts_processed) FROM oi_samples
  WHERE ts_processed >= now() - interval '5 min' GROUP BY 1;
  ```

---

## 6. Recovery procedure

### Path A — staging
```bash
# 1. Revert the patch on staging:
sudo -iu oi-tracker
cd /opt/oi-tracker/code/backend
git checkout main           # or whatever the staging-clean branch is
uv sync
exit

# 2. Restart services:
sudo systemctl restart oi-tracker-scheduler oi-tracker-alert-engine

# 3. Re-enable consensus rule (was disabled in §4 step 4):
sudo -u postgres psql -d oi_tracker -c "
  UPDATE alert_rules SET enabled=true WHERE rule_type='consensus';
"
sudo systemctl restart oi-tracker-alert-engine
```

### Path B — prod
```bash
# 1. Restore settings from baseline:
sudo -u postgres psql -d oi_tracker -c "
  UPDATE settings SET value='3'::jsonb WHERE key='consensus_min_exchanges';
  UPDATE settings SET value='30'::jsonb WHERE key='evaluation_cycle_sec';
"
# (Use the actual baseline values you captured; defaults shown.)

# 2. Re-enable consensus rule:
sudo -u postgres psql -d oi_tracker -c "
  UPDATE alert_rules SET enabled=true WHERE rule_type='consensus';
"

# 3. Restart alert-engine + scheduler:
sudo systemctl restart oi-tracker-alert-engine oi-tracker-scheduler
```

**RTO target:** ≤ 5 min from start of recovery to P99 < 180s and alert
cleared.

---

## 7. Post-drill verification

```bash
# E2E P99 back below SLO:
curl -sG 'http://127.0.0.1:9090/api/v1/query' \
    --data-urlencode 'query=histogram_quantile(0.99, sum by (le) (rate(oi_alert_e2e_latency_seconds_bucket[5m])))'
# Expected: < 180 (and ideally back near baseline)

# Alert cleared:
curl -s 'http://127.0.0.1:9093/api/v2/alerts?active=true' \
    | jq '.[] | select(.labels.alertname=="E2ELatencyHigh")'
# Expected: empty.

# Consensus rule re-enabled:
sudo -u postgres psql -d oi_tracker -c \
    "SELECT rule_id, rule_type, enabled FROM alert_rules WHERE rule_type='consensus';"
# Expected: enabled=t

# Path B — settings back to baseline:
sudo -u postgres psql -d oi_tracker -c \
    "SELECT key,value FROM settings WHERE key IN ('consensus_min_exchanges','evaluation_cycle_sec');"
# Expected: matches baseline file (/tmp/drill-5.7-settings.bak).

# Services healthy:
sudo systemctl is-active oi-tracker-scheduler oi-tracker-alert-engine \
                       oi-tracker-api oi-tracker-tg-sender postgresql

# Path A — patch fully reverted:
cd /var/www/oi-tracker && git status -s
grep -RE 'DRILL 5\.7' backend/app/ || echo no_patch_left
# Expected: no_patch_left
```

---

## 8. Rollback (emergency abort)

If E2E latency spikes far beyond expected (P99 > 600s, evaluation
queue overflows, or scheduler restart-loops):

### Path A
```bash
# Atomic — staging is throwaway; force-restart cleanly:
sudo systemctl restart oi-tracker-scheduler oi-tracker-alert-engine
sudo -iu oi-tracker bash -c 'cd /opt/oi-tracker/code/backend && git reset --hard origin/main && uv sync'
sudo systemctl restart oi-tracker-scheduler oi-tracker-alert-engine
```

### Path B
```bash
# Atomic — restore from /tmp/drill-5.7-settings.bak in a single TX:
sudo -u postgres psql -d oi_tracker <<'EOF'
BEGIN;
UPDATE settings SET value='3'::jsonb WHERE key='consensus_min_exchanges';
UPDATE settings SET value='30'::jsonb WHERE key='evaluation_cycle_sec';
UPDATE alert_rules SET enabled=true WHERE rule_type='consensus';
COMMIT;
EOF

sudo systemctl restart oi-tracker-alert-engine oi-tracker-scheduler
```

If rollback succeeds but the system stays slow → real underlying issue,
treat as incident, follow `13 §5.7` runbook proper.

---

## 9. Drill log template

Copy into `deploy/drills/logs/<YYYY-MM-DD>-slo-breach.md`:

```markdown
# Drill 5.7 — <date>

- **Operator:** <name>
- **Path:** A (staging) / B (prod throttle)
- **Start (UTC):** <ISO>
- **End (UTC):** <ISO>
- **Total duration:** <minutes>
- **Result:** PASS / FAIL / PARTIAL

## Pre-flight
- Path: <A/B>
- Settings backup at /tmp/drill-5.7-settings.bak (Path B only):
  <paste contents>

## Baseline (§3 output)
<paste — note baseline P99 values per stage>

## Induce log
- t=0    : path-specific induce step (paste output)
- t=+5m  : E2E P99 = <X>s (paste PromQL output)
- t=+Xm  : E2ELatencyHigh fired (paste alertmanager JSON)
- t=+Ym  : mitigation applied (paste UPDATE rule SQL)
- t=+Zm  : P99 receded to <X>s

## Expected vs actual
| Expectation | Expected | Actual | Verdict |
|---|---|---|---|
| P99 > 180s within 5 min | yes | <Xs at t=<Ys>> | OK / FAIL |
| E2ELatencyHigh fires within 10 min | yes | <t=<Xs>> | OK / FAIL |
| Bottleneck stage clearly identified | yes | <stage> | OK / FAIL |
| Mitigation reduces P99 | yes | <X→Y in <Z>s> | OK / FAIL |
| Recovery RTO | ≤ 300s | <Xs> | OK / FAIL |
| Other rules keep firing during drill | yes | <Y/N> | OK / FAIL |

## Recovery (§6 output)
<paste>
RTO actual: <Xs> (budget: 300s)

## Post-drill verification (§7 output)
<paste>

## Findings
- Bottleneck attribution accuracy (per-stage histograms vs reality):
- Mitigation effectiveness (`13 §5.7 step 3` works as documented?):
- Did Alertmanager group/inhibit correctly (`E2ELatencyHigh` should
  inhibit `TGDeliveryFailing` per `12 §5.5` inhibit_rules)?

## Lessons added (if any)
- `tasks/lessons.md` — <pattern>
```

---

## 10. Cross-references

- Runbook: `docs/13_OPERATIONS.md §5.7`
- Dev plan: `tasks/development_plan.md §M5.B.7`
- SLO source: `docs/01_PRODUCT_SPEC.md NFR-1` (P95 90s, P99 180s)
- Alert rule: `deploy/observability/prometheus/rules/oi-tracker.yml` —
  group `oi-tracker.alerts` → `E2ELatencyHigh`
- Inhibit rule: `deploy/observability/alertmanager/config.yml` —
  `E2ELatencyHigh` inhibits `TGDeliveryFailing`
- Metrics: `docs/12_OBSERVABILITY_SLO.md §3.5` (alert engine), §3.7
  (E2E + delivery)
- F23 (settings restart-required): `docs/00_DECISIONS_LOG.md F23`

---

## 11. History

| Date | Operator | Path | Result | Log |
|---|---|---|---|---|
| <YYYY-MM-DD> | <name> | A/B | <PASS/FAIL> | [link](logs/<file>.md) |
