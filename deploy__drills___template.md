# Drill X.Y — <Title>

> **Template.** Copy to `deploy/drills/X.Y-slug.md` for new drills.
> Replace placeholders, keep all 11 sections.

| Field | Value |
|---|---|
| **Drill ID** | `B.<n>` (per `tasks/development_plan.md §M5.B`) |
| **Runbook ref** | `docs/13_OPERATIONS.md §5.<n>` |
| **Risk** | Low / Med / High |
| **Est duration** | <e.g. 30 min including pre-flight + recovery + log writeup> |
| **Maintenance window** | required / not required |
| **Neighbour notify** | required (`detector` / `grach-ege`) / not required |
| **Owner** | <person running the drill> |

---

## 1. Scope & Goal

What we induce and what we prove. One paragraph.

> Example: «Induce DNS failure for one exchange to verify that connector
> circuit breaker opens after 5 consecutive failures, that the remaining
> 11 exchanges keep working, and that recovery time after restoring DNS
> is under 2 minutes.»

---

## 2. Pre-flight

- [ ] Maintenance window confirmed with owner.
- [ ] Neighbour services notified if relevant (`detector`, `grach-ege`).
- [ ] Latest backup verified (only for drills touching DB / state).
- [ ] All baseline metrics captured (see §3).
- [ ] Recovery procedure (§6) re-read before inducing.
- [ ] Rollback procedure (§8) re-read before inducing.
- [ ] Alertmanager not silenced for the alert we expect to fire.

---

## 3. Baseline capture

> Run BEFORE inducing. Paste output verbatim into the drill log (§9).

```bash
# Example baseline commands — replace per drill.
date -Iseconds
sudo systemctl is-active oi-tracker-scheduler oi-tracker-api oi-tracker-tg-sender
curl -s 'http://127.0.0.1:9090/api/v1/query?query=<metric>' | jq -r '.data.result'
journalctl -u oi-tracker-scheduler.service --since='5 min ago' --no-pager | tail -20
```

---

## 4. Induce procedure

Numbered, exact commands. Time-budget per step.

1. **t=0**: <command>
2. **t=+30s**: <observation>
3. **t=+5m**: <command>

> Mark each step as it completes with the wall-clock time in the drill log.

---

## 5. Expected observations

What MUST happen. PromQL queries with expected ranges.

- **Within 60s**: `<metric>` should reach `<value>`.
- **Within 5 min**: Alertmanager should fire `<AlertName>` (severity=`<sev>`).
- **Logs**: structured field `<field>=<value>` should appear in
  `journalctl -u <service>`.
- **No regression**: <other-metric> should remain within baseline ± <delta>.

---

## 6. Recovery procedure

How to return state to normal. Atomic command preferred; ≤ 3 steps.

```bash
# Example
sudo systemctl start postgresql
sudo systemctl restart oi-tracker-scheduler oi-tracker-api oi-tracker-tg-sender
```

Time-budget: ≤ <RTO target> per `13 §5.<n>`.

---

## 7. Post-drill verification

Proof that the system returned to baseline.

```bash
# Example
sudo systemctl is-active oi-tracker-scheduler oi-tracker-api oi-tracker-tg-sender
# All should report `active`.

curl -s 'http://127.0.0.1:9090/api/v1/query?query=<metric>' \
    | jq -r '.data.result[].value[1]'
# Should be back to baseline ± 5%.

# Confirm Alertmanager cleared the alert:
curl -s 'http://127.0.0.1:9093/api/v2/alerts?active=true' \
    | jq '.[] | select(.labels.alertname=="<AlertName>")'
# Should return empty.
```

---

## 8. Rollback (emergency abort)

If the drill goes off-script (real incident in parallel, recovery not
working, on-call paged), abort fast. Should be **strictly faster** than
the standard recovery.

```bash
# Example: emergency restore from snapshot, hosts file revert, etc.
```

---

## 9. Drill log template

Copy this into `deploy/drills/logs/<YYYY-MM-DD>-<slug>.md` and fill in.

```markdown
# Drill <X.Y> — <date>

- **Operator:** <name>
- **Start (UTC):** <ISO timestamp>
- **End (UTC):** <ISO timestamp>
- **Total duration:** <minutes>
- **Result:** PASS / FAIL / PARTIAL

## Baseline (§3 output)
<paste verbatim>

## Induce log (§4 with timestamps)
- t=0   : <action>
- t=+Xs : <observation>
- ...

## Expected vs actual (§5)
| Expectation | Expected | Actual | Verdict |
|---|---|---|---|
| <PromQL query> | `<expected range>` | `<actual>` | OK / FAIL |
| Alert fires | `<AlertName>` within <Xm> | <actual time> | OK / FAIL |

## Recovery (§6 output + elapsed)
<paste>
RTO actual: <Xs> (budget: <Ys>)

## Post-drill verification (§7 output)
<paste>

## Findings
- <surprise / regression / spec drift>
- <if FAIL — root cause + fix issue link>

## Lessons added
- `tasks/lessons.md` — <pattern name>
```

---

## 10. Cross-references

- Runbook: `docs/13_OPERATIONS.md §5.<n>`
- Dev plan: `tasks/development_plan.md §M5.B.<n>`
- Alert rule: `deploy/observability/prometheus/rules/oi-tracker.yml` — `<AlertName>`
- Metrics: `docs/12_OBSERVABILITY_SLO.md §3.<n>`

---

## 11. History

| Date | Operator | Result | Log |
|---|---|---|---|
| <YYYY-MM-DD> | <name> | PASS/FAIL | [link](logs/<file>.md) |
