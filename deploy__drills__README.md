# oi-tracker runbook drills

> Executable playbooks for the 7 production runbooks in
> `docs/13_OPERATIONS.md §5.1–5.7`. Each drill induces a known failure
> mode, verifies that monitoring/recovery works as documented, and
> closes with a verification step that proves the system returned to
> baseline.

| Drill | Runbook | Risk | Est dur | Maintenance window | Notify neighbours |
|---|---|---|---|---|---|
| [5.1 Exchange down](5.1-exchange-down.md) | `13 §5.1` | Low | 20 min | not required | not required |
| [5.2 Database down](5.2-database-down.md) | `13 §5.2` | **High** | 30 min | **required** | **required** (`detector`, `grach-ege`) |
| [5.3 TG sender stuck](5.3-tg-sender-stuck.md) | `13 §5.3` | Low | 25 min | not required | not required |
| [5.4 Disk full](5.4-disk-full.md) | `13 §5.4` | **High** | 45 min | recommended | not required |
| [5.5 SSE disconnected](5.5-sse-disconnected.md) | `13 §5.5` | Low | 15 min | not required | not required |
| [5.6 Normalize fail](5.6-normalize-fail.md) | `13 §5.6` | Low | 20 min | not required (dev/staging) | not required |
| [5.7 SLO breach](5.7-slo-breach.md) | `13 §5.7` | Med | 30 min | recommended (dev/staging preferred) | not required |

Total: ~3h calendar (± pre-flight + log writeup).

## Cadence

- **Initial validation:** all 7 drills must PASS before declaring M5
  DoD complete (`tasks/development_plan.md §M5.B.1–B.7`).
- **Quarterly:** rerun all drills the first week of every quarter.
- **Post-incident:** rerun the relevant drill within 7 days of any
  incident touching that runbook (validates the fix + the runbook
  description match reality).
- **After connector or schema changes:** rerun 5.1, 5.6 within the
  release window.

## How to run a drill

1. **Open the playbook** (`5.X-*.md`) and read **all 11 sections**
   before starting. In particular, §6 (recovery) and §8 (rollback)
   should be already loaded in your head before you induce anything.
2. **Pre-flight** (§2): confirm maintenance window, notify neighbours,
   capture baseline (§3).
3. **Induce** (§4) — exact commands, no improvisation.
4. **Observe** (§5) — record actual values per `Expected observations`.
5. **Recover** (§6) — atomic if possible.
6. **Verify** (§7) — system back to baseline.
7. **Write the drill log** — copy template from §9 of the playbook into
   `deploy/drills/logs/<YYYY-MM-DD>-<slug>.md`. Fill in every section.

## Drill log retention

- Logs stay in repo (`deploy/drills/logs/`) — plain markdown, no PII.
- Naming convention: `YYYY-MM-DD-<drill-slug>.md` (e.g.
  `2026-05-15-exchange-down.md`).
- Each drill playbook §11 has a "History" table — append one row per
  run with link to the log.

## When a drill FAILs

A drill fails if (a) the induced failure is not detected, (b) the
observed metrics/alerts don't match `Expected observations`, or (c)
recovery exceeds RTO budget.

1. **Do NOT mark the drill as PASS.** Mark FAIL or PARTIAL.
2. **Open an issue** describing the deviation. Title: `[drill X.Y FAIL]
   <symptom>`.
3. **Fix** — either the runbook (`13 §5.X`), the alerting rule
   (`deploy/observability/prometheus/rules/oi-tracker.yml`), the
   metric emission code, or the recovery procedure.
4. **Add a lesson** to `tasks/lessons.md` if the failure exposes a
   systemic gap (drill expectation diverged from actual behaviour
   — that's a spec/runbook drift signal).
5. **Rerun the drill** within 7 days to confirm the fix.

## Safety guarantees enforced by the playbooks

Every playbook is structured so:

- **Pre-flight** is a hard checklist — skipping it invalidates the run.
- **Induce** procedure is exact, no free-form commands.
- **Recovery** is faster than RTO target per `13 §5.X`.
- **Rollback** is strictly faster than recovery — for use when
  something else breaks mid-drill.
- **Post-drill verification** must reach baseline before the drill is
  closed.

If at any point the system enters an unfamiliar state, switch to
**rollback** (§8 of the playbook). Do not improvise.

## Adding a new drill

Use `_template.md` as the starting point. All 11 sections are required.
Update this README and `tasks/development_plan.md` if the new drill
covers a runbook not currently in `13 §5`.

## Cross-references

- Runbooks: `docs/13_OPERATIONS.md §5`.
- Alert rules: `deploy/observability/prometheus/rules/oi-tracker.yml`.
- Metrics: `docs/12_OBSERVABILITY_SLO.md §3`.
- Dev plan: `tasks/development_plan.md §M5.B.1–B.7`.
- Decisions: `docs/00_DECISIONS_LOG.md` (esp. F14 secrets, F18 default
  rules, F30 metrics ports).
