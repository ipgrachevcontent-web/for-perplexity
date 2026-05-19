# Drill 5.5 — SSE disconnected (nginx down)

| Field | Value |
|---|---|
| **Drill ID** | `B.5` |
| **Runbook ref** | `docs/13_OPERATIONS.md §5.5` |
| **Risk** | Low — nginx outage of ≤ 60s, fully reversible |
| **Est duration** | 15 min (3 pre-flight + 5 induce + 2 recover + 5 verify/log) |
| **Maintenance window** | not required (off-peak still preferred) |
| **Neighbour notify** | not required (nginx is per-app on this host per `13 §4`) |
| **Owner** | <fill in before run> |

---

## 1. Scope & Goal

Stop nginx for ~60s and verify:

1. UI client (browser at `oi-tracker.robot-detector.ru`) shows
   `disconnected` badge / SSE reconnect indicator.
2. After nginx restart, the EventSource auto-reconnect kicks in within
   the configured retry budget (default 1–5s per `10 §5.6`).
3. SSE event stream resumes; the dashboard shows a fresh
   `connection_init` snapshot (ETA 2–5s after reconnect).
4. Backend API (over UDS) was running the whole time — nginx was the
   only failure.

---

## 2. Pre-flight

- [ ] One browser tab open at `https://oi-tracker.robot-detector.ru/`
      with dashboard live (you'll watch the badge).
- [ ] Backend API healthy via UDS direct check (without nginx):
      ```bash
      sudo -u oi-tracker curl -sf --unix-socket /run/oi-tracker/api.sock \
          http://localhost/api/v1/health
      ```
      Expected: `{"status":"ok"}`.
- [ ] No active alerts that would conflate (nginx down may surface as
      `OITrackerAPIUnreachable` if defined).
- [ ] Baseline §3 captured.
- [ ] Recovery (§6) memorized — it's a single command.

---

## 3. Baseline capture

```bash
date -Iseconds

# nginx state:
sudo systemctl is-active nginx

# Public endpoint (through nginx):
curl -sf -o /dev/null -w '%{http_code}\n' https://oi-tracker.robot-detector.ru/api/v1/health
# Expected: 200

# UDS direct (bypassing nginx, sanity that backend is independent):
sudo -u oi-tracker curl -sf --unix-socket /run/oi-tracker/api.sock \
    -o /dev/null -w '%{http_code}\n' http://localhost/api/v1/health
# Expected: 200

# Active SSE connections (gauge):
curl -sG 'http://127.0.0.1:9090/api/v1/query' \
    --data-urlencode 'query=sum by (channel) (oi_sse_connections)' \
    | jq -r '.data.result[] | [.metric.channel, .value[1]] | @tsv'
```

---

## 4. Induce procedure

1. **t=0** — Note the time on browser SSE badge (should be
   `connected`).
2. **t=+5s** — Stop nginx:
   ```bash
   sudo systemctl stop nginx
   sudo systemctl is-active nginx
   # Expected: inactive (or "failed" / "deactivating")
   ```
3. **t=+10s** — Verify public endpoint is unreachable:
   ```bash
   curl -sf -o /dev/null -w '%{http_code}\n' https://oi-tracker.robot-detector.ru/api/v1/health
   # Expected: connect failure / non-2xx
   ```
4. **t=+15s** — Verify backend API is STILL up via UDS:
   ```bash
   sudo -u oi-tracker curl -sf --unix-socket /run/oi-tracker/api.sock \
       -o /dev/null -w '%{http_code}\n' http://localhost/api/v1/health
   # Expected: 200 — proves nginx is the only thing down.
   ```
5. **t=+15s ... +60s** — Watch the browser:
   - SSE badge should show `disconnected` within 1–5s of nginx stop.
   - Browser should attempt auto-reconnect (visible in DevTools →
     Network → EventSource entries with `(failed)` and retry).
6. **t=+60s** — Stop drill, proceed to recovery (§6).

---

## 5. Expected observations

### Browser side (the human signal)
- SSE badge transitions: `connected` → `disconnected` (within ≤ 5s of
  nginx stop, depending on TCP keepalive).
- DevTools Network panel shows EventSource entries failing with
  `net::ERR_CONNECTION_REFUSED` or `net::ERR_EMPTY_RESPONSE`.

### Server side
- `oi_sse_connections` gauge drops to 0 within ~5s (server detects
  client disconnects):
  ```bash
  curl -sG 'http://127.0.0.1:9090/api/v1/query' \
      --data-urlencode 'query=sum(oi_sse_connections)'
  ```
- API access logs (via journald):
  ```bash
  journalctl -u oi-tracker-api.service --since='1 min ago' --no-pager | tail -20
  ```
  Should show normal log lines from before, then idle (no nginx →
  no proxied requests).

### Public endpoint
- HTTPS connection refused or `502 Bad Gateway` (depends on whether
  any other service is listening on 443).

---

## 6. Recovery procedure

```bash
# Atomic recovery — single command:
sudo systemctl start nginx
```

That's it. EventSource on the browser auto-reconnects.

**RTO target:** ≤ 30 sec from `start nginx` to:
- Public endpoint responsive.
- SSE badge back to `connected`.
- `oi_sse_connections` non-zero.

---

## 7. Post-drill verification

```bash
# nginx active:
sudo systemctl is-active nginx

# Public endpoint:
curl -sf -o /dev/null -w '%{http_code}\n' https://oi-tracker.robot-detector.ru/api/v1/health
# Expected: 200

# X-Robots-Tag still set (per `13 §4` / `15 §3.2`):
curl -sI https://oi-tracker.robot-detector.ru/ | grep -i 'x-robots-tag'
# Expected: noindex, nofollow, ...

# SSE connections returning:
curl -sG 'http://127.0.0.1:9090/api/v1/query' \
    --data-urlencode 'query=sum by (channel) (oi_sse_connections)'
# Expected: > 0 once browser reconnects.

# Verify no stuck nginx state:
sudo nginx -t
# Expected: configuration test successful.
```

**Browser sanity:** SSE badge `connected`, dashboard refreshing with
new bars / OI values.

---

## 8. Rollback (emergency abort)

If `start nginx` fails (config issue introduced since last reload):

```bash
# 1. Test config:
sudo nginx -t
# If errors — fix or revert config from git.

# 2. If config OK but service refuses to start:
sudo journalctl -u nginx -n 50 --no-pager
# Look for port-binding conflict (e.g., another process on 443).

# 3. Worst case — bypass nginx temporarily by running `oi-tracker-api`
#    on a TCP fallback port (only if F14 fallback debug mode is in
#    play). Note: this exposes API directly; do not leave in this
#    state. Revert as soon as nginx is repaired.
```

If nginx cannot be brought back within 5 min, treat as incident
(public endpoint outage). The dashboard outage time = nginx down time;
no data loss.

---

## 9. Drill log template

Copy into `deploy/drills/logs/<YYYY-MM-DD>-sse-disconnected.md`:

```markdown
# Drill 5.5 — <date>

- **Operator:** <name>
- **Start (UTC):** <ISO>
- **End (UTC):** <ISO>
- **Total duration:** <minutes>
- **Result:** PASS / FAIL / PARTIAL

## Baseline (§3 output)
<paste>

## Induce log
- t=0    : SSE badge = connected (visual)
- t=+5s  : systemctl stop nginx (paste output)
- t=+10s : public endpoint failed (paste curl output)
- t=+15s : UDS direct = 200 (proves backend independent)
- t=+Xs  : SSE badge → disconnected (paste DevTools screenshot ref)
- t=+60s : systemctl start nginx

## Expected vs actual
| Expectation | Expected | Actual | Verdict |
|---|---|---|---|
| Public endpoint fails | non-2xx | <X> | OK / FAIL |
| Backend stays up via UDS | 200 | <X> | OK / FAIL |
| SSE badge → disconnected within 5s | yes | <Xs> | OK / FAIL |
| Auto-reconnect on nginx restart | within 30s | <Xs> | OK / FAIL |
| oi_sse_connections drops + recovers | yes | <Y/N> | OK / FAIL |

## Recovery (§6 output)
<paste>
RTO actual: <Xs> (budget: 30s)

## Post-drill verification (§7 output)
<paste>

## Findings
- <browser reconnect timing>
- <any X-Robots-Tag drift>

## Lessons added (if any)
- `tasks/lessons.md` — <pattern>
```

---

## 10. Cross-references

- Runbook: `docs/13_OPERATIONS.md §5.5`
- Dev plan: `tasks/development_plan.md §M5.B.5`
- nginx config: `13 §4`, `deploy/nginx/oi-tracker.conf`
- SSE design: `docs/10_DELIVERY_LAYER.md §5` (especially §5.6
  reconnect / heartbeat semantics)
- Metrics: `docs/12_OBSERVABILITY_SLO.md §3.6` —
  `oi_sse_connections{channel}`
- F14 (UDS + fallback debug mode): `docs/00_DECISIONS_LOG.md F14`

---

## 11. History

| Date | Operator | Result | Log |
|---|---|---|---|
| <YYYY-MM-DD> | <name> | <PASS/FAIL> | [link](logs/<file>.md) |
