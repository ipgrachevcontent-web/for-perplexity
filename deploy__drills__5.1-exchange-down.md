# Drill 5.1 — Exchange down (DNS / circuit breaker)

| Field | Value |
|---|---|
| **Drill ID** | `B.1` |
| **Runbook ref** | `docs/13_OPERATIONS.md §5.1` |
| **Risk** | Low |
| **Est duration** | 20 min (5 pre-flight + 10 induce/observe + 3 recover + 2 log) |
| **Maintenance window** | not required |
| **Neighbour notify** | not required |
| **Owner** | <fill in before run> |

---

## 1. Scope & Goal

Induce DNS resolution failure for **one** exchange (Binance) and verify
that:

1. Connector circuit breaker opens after 5 consecutive failures (per
   adapter spec `_BASE_CONNECTOR.md §4`).
2. The remaining 11 exchanges keep collecting samples without
   regression.
3. Alertmanager fires `ConnectorDown{exchange="binance"}` within the
   alert evaluation window.
4. After restoring DNS, recovery completes within ≤ 2 min and the
   circuit breaker closes again.

---

## 2. Pre-flight

- [ ] Confirm 12 connectors are currently healthy:
      `oi_collector_request_total{exchange=~".+",status="ok"}` non-zero
      across all 12 exchanges in the last 5 min.
- [ ] No active `ConnectorDown` alerts in Alertmanager.
- [ ] Baseline §3 captured.
- [ ] Recovery (§6) and Rollback (§8) re-read.
- [ ] You have `sudo` for `/etc/hosts` and `systemctl restart
      oi-tracker-scheduler`.

---

## 3. Baseline capture

```bash
date -Iseconds
sudo systemctl is-active oi-tracker-scheduler

# Per-exchange request rate (last 5 min, requests/sec):
curl -sG 'http://127.0.0.1:9090/api/v1/query' \
    --data-urlencode 'query=sum by (exchange) (rate(oi_collector_request_total[5m]))' \
    | jq -r '.data.result[] | [.metric.exchange, .value[1]] | @tsv'

# Active alerts:
curl -s 'http://127.0.0.1:9093/api/v2/alerts?active=true' \
    | jq '[.[] | {alertname: .labels.alertname, exchange: .labels.exchange, state: .status.state}]'

# Snapshot last 10 lines of scheduler logs:
journalctl -u oi-tracker-scheduler.service --since='2 min ago' --no-pager | tail -10
```

---

## 4. Induce procedure

> Approach: hosts-file override. Pointing `fapi.binance.com` to
> `127.0.0.2` (RFC 5735 loopback) causes immediate connection refused
> at the connector. Reversible by removing one line. No CAP_NET_ADMIN
> required.

1. **t=0** — Add hosts override:
   ```bash
   sudo bash -c 'echo "127.0.0.2 fapi.binance.com  # DRILL 5.1" >> /etc/hosts'
   ```
2. **t=+5s** — Confirm resolution is poisoned:
   ```bash
   getent hosts fapi.binance.com
   # MUST print: 127.0.0.2  fapi.binance.com
   ```
3. **t=+10s** — Restart scheduler so connection pool flushes any cached
   sockets:
   ```bash
   sudo systemctl restart oi-tracker-scheduler
   ```
4. **t=+15s ... +5m** — Observe (see §5).
5. **t=+5m** — Stop drill, proceed to recovery (§6).

---

## 5. Expected observations

### Within 60s
- `oi_collector_request_total{exchange="binance",status="error"}` rate
  increases sharply (≥ 1/s — one failure per cycle):
  ```bash
  curl -sG 'http://127.0.0.1:9090/api/v1/query' \
      --data-urlencode 'query=sum(rate(oi_collector_request_total{exchange="binance",status="error"}[1m]))'
  ```
- Scheduler logs:
  ```bash
  journalctl -u oi-tracker-scheduler.service -f --grep='exchange=binance.*error'
  ```
  Should show structured fields like `error_type=connection_refused`
  or `getaddrinfo` failures.

### Within 2 min (after 5 consecutive failures)
- Circuit breaker opens for binance:
  ```bash
  curl -sG 'http://127.0.0.1:9090/api/v1/query' \
      --data-urlencode 'query=oi_connector_circuit_breaker_state{exchange="binance"}'
  # Expected value: 1 (open) — see `12 §3.1`.
  ```
- `oi_collector_request_total{exchange="binance"}` rate **drops to 0**
  while CB is open (no new fetches attempted).

### Within 5 min
- Alertmanager fires `ConnectorDown{exchange="binance"}` (severity
  `warning`):
  ```bash
  curl -s 'http://127.0.0.1:9093/api/v2/alerts?active=true' \
      | jq '.[] | select(.labels.alertname=="ConnectorDown" and .labels.exchange=="binance")'
  ```

### No regression
- The other 11 exchanges keep their baseline rate (± 10%):
  ```bash
  curl -sG 'http://127.0.0.1:9090/api/v1/query' \
      --data-urlencode 'query=sum by (exchange) (rate(oi_collector_request_total{exchange!="binance",status="ok"}[2m]))'
  ```
- No new `ConnectorDown` alerts for any other exchange.

---

## 6. Recovery procedure

```bash
# 1. Remove hosts override:
sudo sed -i '/# DRILL 5\.1/d' /etc/hosts

# 2. Verify DNS resolves correctly:
getent hosts fapi.binance.com
# MUST resolve to a real Binance IP (not 127.0.0.2).

# 3. Restart scheduler so the connection pool sees fresh DNS:
sudo systemctl restart oi-tracker-scheduler
```

**RTO target:** ≤ 2 min from `sed` to first successful binance fetch
(per `13 §5.1`).

---

## 7. Post-drill verification

```bash
# Scheduler is up:
sudo systemctl is-active oi-tracker-scheduler

# Binance request rate is non-zero again (within 90s of recovery):
curl -sG 'http://127.0.0.1:9090/api/v1/query' \
    --data-urlencode 'query=sum(rate(oi_collector_request_total{exchange="binance",status="ok"}[1m]))'
# Expected: > 0

# Circuit breaker closed:
curl -sG 'http://127.0.0.1:9090/api/v1/query' \
    --data-urlencode 'query=oi_connector_circuit_breaker_state{exchange="binance"}'
# Expected: 0 (closed)

# Alertmanager cleared the alert:
curl -s 'http://127.0.0.1:9093/api/v2/alerts?active=true' \
    | jq '.[] | select(.labels.alertname=="ConnectorDown" and .labels.exchange=="binance")'
# Expected: empty

# Confirm /etc/hosts is clean:
grep -E 'DRILL 5\.1|fapi\.binance\.com' /etc/hosts
# Expected: empty
```

---

## 8. Rollback (emergency abort)

If the drill triggers cascading alerts or other connectors start
failing for unrelated reasons:

```bash
# Atomic abort: revert hosts immediately, then restart.
sudo sed -i '/# DRILL 5\.1/d' /etc/hosts
sudo systemctl restart oi-tracker-scheduler
```

If recovery hangs (scheduler restart loops): check
`journalctl -u oi-tracker-scheduler -n 50` and follow runbook `13 §5.1`
step 5 (DNS / firewall fix). Worst case — scheduler stays down, the
other 11 exchanges keep their last-known state, no data corruption.

---

## 9. Drill log template

Copy into `deploy/drills/logs/<YYYY-MM-DD>-exchange-down.md`:

```markdown
# Drill 5.1 — <date>

- **Operator:** <name>
- **Start (UTC):** <ISO>
- **End (UTC):** <ISO>
- **Total duration:** <minutes>
- **Result:** PASS / FAIL / PARTIAL

## Baseline (§3 output)
<paste>

## Induce log
- t=0    : added hosts override
- t=+5s  : getent confirmed 127.0.0.2
- t=+10s : scheduler restarted
- t=+Xs  : first error in logs (paste line)
- t=+Ys  : circuit breaker opened (CB metric value)
- t=+Zs  : ConnectorDown alert fired (paste alertmanager JSON)

## Expected vs actual
| Expectation | Expected | Actual | Verdict |
|---|---|---|---|
| Errors rate ↑ within 60s | > 1/s | <X> | OK / FAIL |
| CB opens within 2 min | metric=1 | <X> at t=<Ys> | OK / FAIL |
| ConnectorDown alert fires within 5 min | yes | <Y/N at t=<Zs>> | OK / FAIL |
| Other 11 exchanges no regression | rate ± 10% baseline | <paste> | OK / FAIL |

## Recovery (§6 output)
<paste>
RTO actual: <Xs> (budget: 120s)

## Post-drill verification (§7 output)
<paste>

## Findings
- <surprises / spec drift>

## Lessons added (if any)
- `tasks/lessons.md` — <pattern>
```

---

## 10. Cross-references

- Runbook: `docs/13_OPERATIONS.md §5.1`
- Dev plan: `tasks/development_plan.md §M5.B.1`
- Alert rule: `deploy/observability/prometheus/rules/oi-tracker.yml` —
  group `oi-tracker.collectors` → `ConnectorDown`
- Metrics: `docs/12_OBSERVABILITY_SLO.md §3.1` — `oi_collector_request_total`,
  `oi_connector_circuit_breaker_state`
- Adapter circuit breaker spec: `docs/11_EXCHANGE_ADAPTERS/_BASE_CONNECTOR.md §4`

---

## 11. History

| Date | Operator | Result | Log |
|---|---|---|---|
| <YYYY-MM-DD> | <name> | <PASS/FAIL> | [link](logs/<file>.md) |
