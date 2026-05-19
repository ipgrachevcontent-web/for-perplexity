# Drill 5.6 — Normalize fail (schema drift)

| Field | Value |
|---|---|
| **Drill ID** | `B.6` |
| **Runbook ref** | `docs/13_OPERATIONS.md §5.6` |
| **Risk** | Low — drill runs on a fixture in dev/staging, NOT against live exchange traffic |
| **Est duration** | 20 min (3 pre-flight + 5 induce + 5 observe + 3 recover + 4 verify/log) |
| **Maintenance window** | not required (does not touch production) |
| **Neighbour notify** | not required |
| **Owner** | <fill in before run> |

---

## 1. Scope & Goal

On a **dev/staging instance** (or a local pytest run against
production-like fixtures), introduce a schema drift in the Binance OI
fixture (rename `openInterest` → `oi_value`) and verify:

1. Normalizer detects the missing field, raises a typed
   `SchemaDriftError`, and emits `oi_normalize_errors_total{
   error_type="schema_drift", exchange="binance",
   failed_stage="parse"}` ≥ 1 per affected sample.
2. The bad sample is recorded in `normalization_errors` table with the
   triggering `error_message`.
3. Other 11 exchanges keep normalizing without regression.
4. After fixture revert + replay, errors stop, the affected
   period reprocesses cleanly, and `oi_samples` are present.

Drill is **deliberately not on prod**: we cannot fake an exchange
response mid-flight. The schema drift is simulated by amending a saved
fixture and re-running the normalizer pipeline.

---

## 2. Pre-flight

- [ ] Decide environment:
  - **A.** Local pytest harness against a real fixture file (preferred,
    fastest, least risk). Fixture path:
    `backend/tests/fixtures/exchange/binance/openinterest_*.json`.
  - **B.** Dedicated staging instance with replay harness pointed at a
    canned fixture set (only if available — most teams have only A).
- [ ] Working git tree clean (we'll touch a fixture file and want a
      clean revert):
      ```bash
      cd /var/www/oi-tracker && git status -s
      ```
      Expected: empty (or stash anything unrelated).
- [ ] If using path **A**, the test runner is set up:
      ```bash
      cd /var/www/oi-tracker/backend && uv run pytest --collect-only \
          tests/unit/normalizer/ tests/contract/ -q | head
      ```
- [ ] If using path **B**, staging instance backend services are healthy
      and Prometheus is scraping them.
- [ ] Baseline §3 captured.

---

## 3. Baseline capture

```bash
date -Iseconds

# Path A — local fixture (snapshot file before mutation):
cd /var/www/oi-tracker
ls backend/tests/fixtures/exchange/binance/ | head -5
git rev-parse HEAD
git status -s

# Path B — staging metrics (only if running on a live staging instance):
# Errors rate for binance:
curl -sG 'http://STAGING_HOST:9090/api/v1/query' \
    --data-urlencode 'query=sum by (error_type) (rate(oi_normalize_errors_total{exchange="binance"}[5m]))' \
    | jq -r '.data.result[] | [.metric.error_type, .value[1]] | @tsv'
# Expected baseline: 0 / negligible.

# normalization_errors table snapshot:
sudo -u postgres psql -d oi_tracker_staging -c "
  SELECT count(*) FROM normalization_errors
  WHERE exchange='binance' AND occurred_at >= now() - interval '15 minutes';
"
```

---

## 4. Induce procedure

### Path A — local fixture (preferred)

1. **t=0** — Locate a Binance OI fixture:
   ```bash
   cd /var/www/oi-tracker
   FIXTURE=$(ls backend/tests/fixtures/exchange/binance/openinterest_*.json | head -1)
   echo "FIXTURE=$FIXTURE"
   ```
2. **t=+1m** — Rename field `openInterest` → `oi_value`:
   ```bash
   # Surgical: only first occurrence of the JSON field name (jq is safer
   # than sed for JSON, but sed works for fixture files in repo since
   # we'll revert atomically via git checkout).
   sed -i 's/"openInterest":/"oi_value":/g' "$FIXTURE"

   # Confirm change:
   grep -E '"oi_value":|"openInterest":' "$FIXTURE" | head -3
   # Expected: only "oi_value":, no "openInterest":
   ```
3. **t=+2m** — Run the normalizer test suite that uses this fixture:
   ```bash
   cd /var/www/oi-tracker/backend
   uv run pytest tests/contract/test_binance_contract.py -v 2>&1 | tail -30
   # Expected: at least one FAIL with SchemaDriftError or
   # `KeyError: openInterest` if structured error not yet typed.
   ```
4. **t=+3m** — Check that the error path emits the expected metric.
   For unit-level verification (no Prometheus running locally), inspect
   the structured exception type in pytest output.

### Path B — staging instance (only if available)

1. Replace a fixture in the staging replay harness as in Path A.
2. Trigger replay: `python -m app.tools.replay --exchange=binance
   --since="..." --until="..." --version=test`.
3. Watch staging Prometheus for the metric uptick (§5).

---

## 5. Expected observations

### Pytest (Path A)
- ≥ 1 FAIL in normalizer/contract tests touching the fixture.
- Failure message contains `schema_drift` or
  `openInterest` (depending on whether typed error class is in use).
- `oi_normalize_errors_total` is incremented (visible if pytest fixture
  reads the in-process Prometheus registry):
  ```bash
  uv run pytest tests/contract/test_binance_contract.py -v -s 2>&1 \
      | grep -E 'normalize_errors|schema_drift'
  ```

### Staging metrics (Path B, if available)
- Within 1–2 min of replay:
  ```promql
  sum by (error_type) (rate(oi_normalize_errors_total{exchange="binance"}[2m]))
  ```
  Expected: `error_type="schema_drift"` rate > 0, others ≈ 0.

- normalization_errors table:
  ```sql
  SELECT error_type, failed_stage, error_message, count(*)
  FROM normalization_errors
  WHERE exchange='binance' AND occurred_at >= now() - interval '5 minutes'
  GROUP BY 1,2,3 ORDER BY 4 DESC;
  ```
  Expected: rows with `error_type='schema_drift'`,
  `failed_stage='parse'`.

### No regression
- Other 11 exchanges' error rates unchanged (Path B).
- For Path A, other contract tests still PASS.

---

## 6. Recovery procedure

### Path A
```bash
cd /var/www/oi-tracker

# Atomic revert — git restore the fixture (single source of truth):
git checkout HEAD -- backend/tests/fixtures/exchange/binance/

# Confirm revert:
git status -s
# Expected: empty (or only unrelated files).

grep -l '"openInterest":' backend/tests/fixtures/exchange/binance/*.json | head
# Expected: at least one file lists openInterest again.

# Re-run tests to confirm green:
cd backend
uv run pytest tests/contract/test_binance_contract.py -v 2>&1 | tail -10
# Expected: all PASS.
```

### Path B
1. Revert fixture in staging replay store (same `git checkout` if it's
   versioned, or restore from backup).
2. Replay the affected window with the **fixed** parser/fixture:
   ```bash
   python -m app.tools.replay --exchange=binance \
       --since="<window_start>" --until="<window_end>" --version=fixed
   ```
3. Verify the period now has clean `oi_samples` rows.

---

## 7. Post-drill verification

### Path A
```bash
cd /var/www/oi-tracker
git status -s
# Expected: empty.

cd backend
uv run pytest tests/unit/normalizer/ tests/contract/ -q 2>&1 | tail -5
# Expected: all PASS.

# Verify no leftover modifications anywhere:
git diff
# Expected: empty.
```

### Path B
- No `oi_normalize_errors_total{error_type="schema_drift"}` increments
  after replay finishes:
  ```bash
  curl -sG 'http://STAGING_HOST:9090/api/v1/query' \
      --data-urlencode 'query=sum(rate(oi_normalize_errors_total{error_type="schema_drift"}[5m]))'
  # Expected: 0
  ```
- Sample insertion for the affected window:
  ```sql
  SELECT count(*) FROM oi_samples
  WHERE exchange='binance'
    AND ts_exchange BETWEEN '<window_start>' AND '<window_end>';
  ```
  Expected: matches the original sample count for that window.

---

## 8. Rollback (emergency abort)

Drill 5.6 is the safest of the seven; rollback is unlikely to be
needed. But if pytest run hangs or staging replay loops:

```bash
# Path A — kill pytest, hard-reset the working tree:
cd /var/www/oi-tracker
git checkout HEAD -- backend/tests/fixtures/

# Path B — abort replay job:
# (replay tooling Ctrl-C; surviving partial rows in oi_samples are
# acceptable per `13 §6.3` — replay is idempotent on (exchange,
# native_symbol, ts_exchange, version)).
```

No persistent state damage in either path.

---

## 9. Drill log template

Copy into `deploy/drills/logs/<YYYY-MM-DD>-normalize-fail.md`:

```markdown
# Drill 5.6 — <date>

- **Operator:** <name>
- **Path:** A (local pytest) / B (staging replay)
- **Start (UTC):** <ISO>
- **End (UTC):** <ISO>
- **Total duration:** <minutes>
- **Result:** PASS / FAIL / PARTIAL

## Baseline (§3 output)
<paste; include git HEAD and `git status -s`>

## Induce log
- t=0    : fixture located at <path>
- t=+1m  : sed renamed openInterest → oi_value (paste grep output)
- t=+2m  : pytest run output (paste tail 30)
- t=+Xm  : SchemaDriftError class observed (Y/N)
- t=+Ym  : error metric increment observed (Y/N + value)

## Expected vs actual
| Expectation | Expected | Actual | Verdict |
|---|---|---|---|
| Test FAIL with schema_drift | yes | <Y/N> | OK / FAIL |
| Error class is typed (not bare KeyError) | yes | <Y/N> | OK / WARN |
| oi_normalize_errors_total{schema_drift} ↑ | yes | <Y/N> | OK / FAIL |
| normalization_errors row written | yes | <Y/N> | OK / FAIL |
| Other contract tests still PASS | yes | <Y/N> | OK / FAIL |
| Revert restores green tests | yes | <Y/N> | OK / FAIL |

## Recovery (§6 output)
<paste git checkout output>
<paste pytest re-run tail>

## Post-drill verification (§7 output)
<paste>

## Findings
- <error class typing — bare KeyError vs typed SchemaDriftError>
- <metric label correctness>
- <how quickly the error surfaced — relevant for `13 §5.6` recovery
   step expectations>

## Lessons added (if any)
- `tasks/lessons.md` — <pattern>
```

---

## 10. Cross-references

- Runbook: `docs/13_OPERATIONS.md §5.6`
- Dev plan: `tasks/development_plan.md §M5.B.6`
- Normalizer: `docs/07_NORMALIZER.md §3` (parsers), `docs/07 §7`
  (quality assessment / typed error classes)
- Replay tooling: `docs/13_OPERATIONS.md §6.2` (replay procedure)
- Metrics: `docs/12_OBSERVABILITY_SLO.md §3.3` —
  `oi_normalize_errors_total{exchange,error_type,failed_stage}`
- Adapter spec: `docs/11_EXCHANGE_ADAPTERS/binance.md` (canonical
  field names)

---

## 11. History

| Date | Operator | Path | Result | Log |
|---|---|---|---|---|
| <YYYY-MM-DD> | <name> | A/B | <PASS/FAIL> | [link](logs/<file>.md) |
