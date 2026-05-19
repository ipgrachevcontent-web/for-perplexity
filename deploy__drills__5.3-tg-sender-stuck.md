# Drill 5.3 — TG sender stuck (invalid token)

| Field | Value |
|---|---|
| **Drill ID** | `B.3` |
| **Runbook ref** | `docs/13_OPERATIONS.md §5.3` |
| **Risk** | Low — affects only Telegram delivery, no data loss (events stay in `delivery_queue`) |
| **Est duration** | 25 min (5 pre-flight + 10 induce/observe + 5 recover + 5 verify/log) |
| **Maintenance window** | not required |
| **Neighbour notify** | not required |
| **Owner** | <fill in before run> |

---

## 1. Scope & Goal

Replace the production TG bot token with an invalid one and verify:

1. tg-sender keeps running but every delivery attempt fails with
   `Unauthorized` from the Telegram API.
2. `delivery_queue` size grows (events not consumed).
3. Alertmanager fires `TGDeliveryBacklog` within ~5 min (per
   `12 §3.7`).
4. After restoring the real token + restart, the queue drains and the
   alert clears.
5. No alerts are dropped — the queue holds them until delivery
   succeeds.

---

## 2. Pre-flight

- [ ] Real bot token saved somewhere safe **before** swapping it.
      The env-file is `chmod 0600 root:oi-tracker` per F14, so make a
      backup copy:
      ```bash
      sudo cp /etc/oi-tracker/oi-tracker.env /etc/oi-tracker/oi-tracker.env.bak
      sudo chmod 0600 /etc/oi-tracker/oi-tracker.env.bak
      ```
- [ ] Pick a low-volume time. If the queue grows >1000 events during
      the drill we still want it drainable in 5 min after recovery.
- [ ] tg-sender is currently `active`, no backlog:
      ```bash
      sudo systemctl is-active oi-tracker-tg-sender
      sudo -u postgres psql -d oi_tracker -tAc \
          "SELECT count(*) FROM delivery_queue WHERE status='pending'"
      # Expected: 0 or low single digits.
      ```
- [ ] No active `TGDeliveryBacklog` alert.
- [ ] Baseline §3 captured.
- [ ] Recovery (§6) and Rollback (§8) re-read.

---

## 3. Baseline capture

```bash
date -Iseconds

# tg-sender service:
sudo systemctl is-active oi-tracker-tg-sender

# Queue depth:
sudo -u postgres psql -d oi_tracker -c "
  SELECT status, count(*) FROM delivery_queue GROUP BY status;
"

# Delivery rate (sent/sec last 5 min):
curl -sG 'http://127.0.0.1:9090/api/v1/query' \
    --data-urlencode 'query=sum by (status) (rate(oi_tg_delivery_total[5m]))' \
    | jq -r '.data.result[] | [.metric.status, .value[1]] | @tsv'

# Last 5 lines of tg-sender logs:
journalctl -u oi-tracker-tg-sender.service --since='2 min ago' --no-pager | tail -5

# Token sanity (without leaking it — just length):
sudo grep '^TELEGRAM_BOT_TOKEN=' /etc/oi-tracker/oi-tracker.env \
    | awk -F= '{print "token_len=" length($2)}'
# Real bot tokens are 46–47 chars.
```

---

## 4. Induce procedure

> Approach: in-place env replace via `sed`, then service restart.
> Reversible by restoring from `.bak` (created in pre-flight).

1. **t=0** — Backup verification:
   ```bash
   sudo test -f /etc/oi-tracker/oi-tracker.env.bak && echo backup_ok || echo MISSING
   # MUST: backup_ok
   ```
2. **t=+5s** — Replace token with a syntactically-valid but unauthorized
   placeholder:
   ```bash
   sudo sed -i 's|^TELEGRAM_BOT_TOKEN=.*|TELEGRAM_BOT_TOKEN=000000000:DRILL5_3_INVALID_TOKEN_AAAAAAAAAAAAAAAAA|' \
       /etc/oi-tracker/oi-tracker.env
   ```
3. **t=+10s** — Restart tg-sender:
   ```bash
   sudo systemctl restart oi-tracker-tg-sender
   sudo systemctl is-active oi-tracker-tg-sender
   # Expected: active (the service starts, but every delivery will fail).
   ```
4. **t=+15s ... +6m** — Observe (see §5).
5. **t=+6m** — Stop drill, proceed to recovery (§6).

---

## 5. Expected observations

### Within 30s
- Failure rate climbs:
  ```bash
  curl -sG 'http://127.0.0.1:9090/api/v1/query' \
      --data-urlencode 'query=sum(rate(oi_tg_delivery_total{status="error"}[1m]))'
  # Expected: > 0
  ```
- Logs show `Unauthorized` from telegram-bot-api:
  ```bash
  journalctl -u oi-tracker-tg-sender.service -f --grep='401|Unauthorized'
  ```
  Should NOT show the actual token (per `13 §10.3` log redaction).

### Within 2–5 min (depending on alert rate)
- `delivery_queue` pending count grows:
  ```bash
  sudo -u postgres psql -d oi_tracker -c "
    SELECT count(*) FROM delivery_queue
    WHERE status='pending' OR (status='failed' AND attempts < max_attempts);
  "
  ```
  Expected: monotonically increasing.

### Within 5 min
- Alertmanager fires `TGDeliveryBacklog`:
  ```bash
  curl -s 'http://127.0.0.1:9093/api/v2/alerts?active=true' \
      | jq '.[] | select(.labels.alertname=="TGDeliveryBacklog")'
  ```

### No regression elsewhere
- Other services unaffected:
  ```bash
  sudo systemctl is-active oi-tracker-scheduler oi-tracker-api
  # Expected: active × 2
  ```
- `oi_collector_request_total` rate unchanged.

### Token leakage check (security regression)
- Logs MUST NOT contain the placeholder secret:
  ```bash
  journalctl -u oi-tracker-tg-sender.service --since='2 min ago' --no-pager \
      | grep -E 'DRILL5_3_INVALID_TOKEN|000000000:'
  # Expected: empty (no match)
  ```
  If matches found — that's a security finding (FAIL the drill on
  this alone, even if backlog alert fired correctly).

---

## 6. Recovery procedure

```bash
# 1. Restore env from backup:
sudo cp /etc/oi-tracker/oi-tracker.env.bak /etc/oi-tracker/oi-tracker.env
sudo chown root:oi-tracker /etc/oi-tracker/oi-tracker.env
sudo chmod 0600 /etc/oi-tracker/oi-tracker.env

# 2. Verify token restored (length sanity):
sudo grep '^TELEGRAM_BOT_TOKEN=' /etc/oi-tracker/oi-tracker.env \
    | awk -F= '{print "token_len=" length($2)}'
# Should match baseline.

# 3. Restart tg-sender; queue drains automatically:
sudo systemctl restart oi-tracker-tg-sender

# 4. Drop the backup file (no longer needed):
sudo rm /etc/oi-tracker/oi-tracker.env.bak
```

**RTO target:** ≤ 5 min from token restore to queue back to baseline
(per `13 §5.3`). Drain time depends on backlog size and TG rate-limits.

---

## 7. Post-drill verification

```bash
# tg-sender active:
sudo systemctl is-active oi-tracker-tg-sender

# Queue drained:
sudo -u postgres psql -d oi_tracker -c "
  SELECT status, count(*) FROM delivery_queue GROUP BY status;
"
# Expected: pending count back to baseline (≤ 5).

# Successful deliveries:
curl -sG 'http://127.0.0.1:9090/api/v1/query' \
    --data-urlencode 'query=sum(rate(oi_tg_delivery_total{status="ok"}[2m]))'
# Expected: > 0

# Alert cleared:
curl -s 'http://127.0.0.1:9093/api/v2/alerts?active=true' \
    | jq '.[] | select(.labels.alertname=="TGDeliveryBacklog")'
# Expected: empty

# Backup file gone:
sudo test -f /etc/oi-tracker/oi-tracker.env.bak && echo STILL_PRESENT || echo gone_ok
# Expected: gone_ok

# Permissions intact (F14):
stat -c '%a %U %G' /etc/oi-tracker/oi-tracker.env
# Expected: 600 root oi-tracker
```

---

## 8. Rollback (emergency abort)

If anything goes wrong while the env is swapped:

```bash
# Atomic restore — single command:
sudo cp /etc/oi-tracker/oi-tracker.env.bak /etc/oi-tracker/oi-tracker.env \
  && sudo chmod 0600 /etc/oi-tracker/oi-tracker.env \
  && sudo chown root:oi-tracker /etc/oi-tracker/oi-tracker.env \
  && sudo systemctl restart oi-tracker-tg-sender
```

If the backup is missing (didn't run pre-flight), the real token must
be re-entered manually. Until then, tg-sender stays broken — but
events keep accumulating safely in `delivery_queue` (no data loss). The
backlog can be drained later.

---

## 9. Drill log template

Copy into `deploy/drills/logs/<YYYY-MM-DD>-tg-sender-stuck.md`:

```markdown
# Drill 5.3 — <date>

- **Operator:** <name>
- **Start (UTC):** <ISO>
- **End (UTC):** <ISO>
- **Total duration:** <minutes>
- **Result:** PASS / FAIL / PARTIAL

## Baseline (§3 output)
<paste, with token_len and queue depth>

## Induce log
- t=0    : backup verified
- t=+5s  : sed replaced token
- t=+10s : tg-sender restarted
- t=+Xs  : first 401 / Unauthorized in logs
- t=+Ys  : queue depth = <N>
- t=+Zs  : TGDeliveryBacklog fired (paste alertmanager JSON)

## Expected vs actual
| Expectation | Expected | Actual | Verdict |
|---|---|---|---|
| Errors rate ↑ within 30s | > 0 | <X> | OK / FAIL |
| Queue backlog grows | monotonic | <Y/N> | OK / FAIL |
| TGDeliveryBacklog fires within 5 min | yes | <t=<Zs>> | OK / FAIL |
| No token leak in logs | no match | <Y/N> | OK / FAIL |
| Other services unaffected | active | <Y/N> | OK / FAIL |

## Recovery (§6 output)
<paste>
RTO actual: <Xs> (budget: 300s)
Queue drain time: <Ys>

## Post-drill verification (§7 output)
<paste>

## Findings
- <surprises / spec drift>
- <token-redaction quality>

## Lessons added (if any)
- `tasks/lessons.md` — <pattern>
```

---

## 10. Cross-references

- Runbook: `docs/13_OPERATIONS.md §5.3`
- Dev plan: `tasks/development_plan.md §M5.B.3`
- Alert rule: `deploy/observability/prometheus/rules/oi-tracker.yml` —
  group `oi-tracker.alerts` → `TGDeliveryBacklog`
- Metrics: `docs/12_OBSERVABILITY_SLO.md §3.7` —
  `oi_tg_delivery_total{status}`, `oi_tg_queue_depth`
- Secrets policy: `docs/00_DECISIONS_LOG.md F14`,
  `docs/13_OPERATIONS.md §10.3` (log redaction)

---

## 11. History

| Date | Operator | Result | Log |
|---|---|---|---|
| <YYYY-MM-DD> | <name> | <PASS/FAIL> | [link](logs/<file>.md) |
