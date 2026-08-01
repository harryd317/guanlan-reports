# S7 Streak Signal Measurement Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build and run the frozen 2015—2025 S7 streak-signal measurement without creating any trade, position, shadow, or production behavior.

**Architecture:** A dedicated `s7_measurement` package writes only `S7_research.local.db` and opens S6, T1, and market inputs read-only. Signal construction and fixed-horizon measurement each stream the 2674-day T1 calendar; derived rows, exclusion reasons, benchmarks, statistics, and the 16 qualification cells remain auditable in SQLite.

**Tech Stack:** Python 3.11 standard library, SQLite, existing Tushare client path, `unittest`, Markdown, CSV.

## Global Constraints

- Study interval: 2015-01-05 through 2025-12-31 on exactly 2674 T1 sessions.
- A/B thresholds: `5_000_000` and `10_000_000` ten-thousand yuan of `total_mv`.
- Streaks overlap: every 3-streak is also a 2-streak; every B signal is also an A signal.
- Signal-day `pct_chg <= 1.5`; one-price limit-up is equal OHLC and `pct_chg >= 9.5`.
- ST unknowns become `st_status_missing` exclusions; 2016-before limitations remain visible.
- Horizons: T+1, T+2, T+3, T+5 on the T1 calendar.
- Net return: adjusted gross return minus `0.0015` once; positive rate uses net return greater than zero.
- Qualification: the same one of 16 cells must pass net mean `>= 0.0015`, net positive rate `>= 0.52`, and no more than four negative-net-mean years with all eleven years present.
- No parameter scan, threshold movement, trade simulation, shadow record, production write, merge, or deploy.
- No new dependency.

---

### Task 1: Freeze constants and SQLite boundaries

**Files:**
- Create: `s7_measurement/__init__.py`
- Create: `s7_measurement/config.py`
- Create: `s7_measurement/db.py`
- Create: `test_s7_measurement.py`

**Interfaces:**
- Produces: `init_research_db(path)`, `connect_research(path, readonly=False)`, `connect_external_ro(path)`, `research_transaction(path)`.
- Produces frozen constants `STUDY_START`, `STUDY_END`, `EXPECTED_SESSIONS`, `CAP_A`, `CAP_B`, `HORIZONS`, `ROUND_TRIP_FEE`, and qualification thresholds.

- [ ] **Step 1: Write failing path and read-only tests**

```python
class S7DatabaseBoundaryTest(unittest.TestCase):
    def test_only_s7_local_database_is_writable(self):
        with self.assertRaisesRegex(ValueError, "S7 writes are restricted"):
            db.init_research_db(self.root / "wrong.db")

    def test_external_connection_rejects_write(self):
        source = self.root / "source.db"
        with sqlite3.connect(source) as con:
            con.execute("CREATE TABLE sample(value INTEGER)")
        with db.connect_external_ro(source) as con:
            with self.assertRaises(sqlite3.OperationalError):
                con.execute("INSERT INTO sample VALUES(1)")
```

- [ ] **Step 2: Run the tests and verify RED**

Run: `./.venv/bin/python -m unittest test_s7_measurement.S7DatabaseBoundaryTest -v`

Expected: import failure because `s7_measurement.db` does not exist.

- [ ] **Step 3: Implement constants, write guard, schema, and transactions**

Implement the exact target-name assertion:

```python
RESEARCH_DB_NAME = "S7_research.local.db"

def _assert_research_path(path):
    if Path(path).name != RESEARCH_DB_NAME:
        raise ValueError(f"S7 writes are restricted to {RESEARCH_DB_NAME}")
```

Create the approved raw/audit tables if absent and create `run_manifest`, `signal_observations`, `signal_cohorts`, `exclusion_counts`, `benchmark_measurements`, `return_measurements`, `drawdown_measurements`, `aggregate_statistics`, and `qualification_results` exactly as specified in the design. External connections use `mode=ro`, `row_factory=sqlite3.Row`, and `PRAGMA query_only=ON`.

- [ ] **Step 4: Run Task 1 tests and verify GREEN**

Run: `./.venv/bin/python -m unittest test_s7_measurement.S7DatabaseBoundaryTest -v`

Expected: all Task 1 tests pass.

- [ ] **Step 5: Commit Task 1**

```bash
git add s7_measurement/__init__.py s7_measurement/config.py s7_measurement/db.py test_s7_measurement.py
git commit -m "feat(s7): add isolated research database boundary"
```

### Task 2: Audit and resume the approved daily-basic source

**Files:**
- Create: `s7_measurement/sources.py`
- Modify: `test_s7_measurement.py`

**Interfaces:**
- Consumes: Task 1 database connections and constants.
- Produces: `t1_calendar(t1_db) -> list[str]`.
- Produces: `validate_data(research_db, market_db, t1_db) -> dict`.
- Produces: `ingest_daily_basic_frame(research_db, frame, trade_date) -> dict`.
- Produces: `fetch_daily_basic(research_db, t1_db, pro) -> dict`.

- [ ] **Step 1: Write failing calendar, field, idempotency, and 99% tests**

Use literal fixtures that cover 2673 sessions, a duplicate session, missing `gate_open`, a frame with an extra field, a repeated identical batch, a changed repeated batch, coverage exactly 99%, and coverage below 99%.

```python
def test_coverage_boundary_is_inclusive(self):
    result = sources._coverage_result(100, 99)
    self.assertEqual(result["coverage_pct"], 99.0)
    self.assertTrue(result["coverage_pass"])
    self.assertFalse(sources._coverage_result(10001, 9900)["coverage_pass"])
```

- [ ] **Step 2: Run Task 2 tests and verify RED**

Run: `./.venv/bin/python -m unittest test_s7_measurement.S7SourceAuditTest -v`

Expected: import or missing-symbol failures for `s7_measurement.sources`.

- [ ] **Step 3: Implement canonical rows and batch audit**

Only accept `ts_code`, `trade_date`, and `total_mv`; normalize `YYYYMMDD` to ISO; require nonempty primary keys and finite positive `total_mv`; hash canonical sorted JSON; reject a changed payload for an already successful request. `fetch_daily_basic` calls only missing T1 dates and passes `fields="ts_code,trade_date,total_mv"`.

- [ ] **Step 4: Implement the coverage gate**

`validate_data` attaches `market.db` and T1 read-only to the read-only S7 connection. It computes the conservative candidate superset after the 365-day and lifecycle checks, verifies one success audit per calendar day, verifies no raw date outside T1, and raises `DataCoverageError` below 99%.

- [ ] **Step 5: Run Task 2 tests and verify GREEN**

Run: `./.venv/bin/python -m unittest test_s7_measurement.S7SourceAuditTest -v`

Expected: all Task 2 tests pass.

- [ ] **Step 6: Run the real offline audit**

Run:

```bash
./.venv/bin/python s7_measurement_cli.py audit-data \
  --research-db S7_research.local.db \
  --market-db ../../market.db \
  --t1-db ../s3-stage1-data-clusters/S3_research.db
```

Expected JSON includes `2674`, `99.9780` or a more precise equivalent, zero failed audits, and `coverage_pass: true`.

- [ ] **Step 7: Commit Task 2**

```bash
git add s7_measurement/sources.py test_s7_measurement.py
git commit -m "feat(s7): enforce daily-basic coverage audit"
```

### Task 3: Implement pure frozen signal decisions

**Files:**
- Create: `s7_measurement/signals.py`
- Modify: `test_s7_measurement.py`

**Interfaces:**
- Produces dataclasses `Bar`, `StStatus`, `SignalInputs`, and `SignalDecision`.
- Produces: `streak_flags(bars) -> tuple[bool, bool]`.
- Produces: `resolve_st_status(...) -> StStatus`.
- Produces: `evaluate_signal(inputs) -> SignalDecision`.

- [ ] **Step 1: Write one failing test per frozen boundary**

Tests use literal bars and expected cohort tuples:

```python
def test_b3_expands_to_all_four_overlapping_cohorts(self):
    decision = signals.evaluate_signal(self.inputs(
        bars=[self.bull(), self.bull(), self.bull(pct_chg=1.5)],
        total_mv=10_000_000,
    ))
    self.assertEqual(decision.cohorts, (("A", 2), ("A", 3), ("B", 2), ("B", 3)))
```

Add distinct tests for strict candle positivity, 1.5% inclusion, A/B exact boundaries, day 364/365, delisting day, one-price limit-up, 2016 direct ST, pre-2016 name ST, and pre-2016 unknown status.

- [ ] **Step 2: Run Task 3 tests and verify RED**

Run: `./.venv/bin/python -m unittest test_s7_measurement.S7SignalDecisionTest -v`

Expected: missing module or symbols.

- [ ] **Step 3: Implement the minimal pure functions**

Use ordered exclusion codes. `st_status_missing` must be separate from `st_or_star_st`. A decision can create cohorts only when it has no exclusion reasons, has a valid adjusted close, and meets A.

- [ ] **Step 4: Run Task 3 tests and verify GREEN**

Run: `./.venv/bin/python -m unittest test_s7_measurement.S7SignalDecisionTest -v`

Expected: all Task 3 tests pass.

- [ ] **Step 5: Commit Task 3**

```bash
git add s7_measurement/signals.py test_s7_measurement.py
git commit -m "feat(s7): freeze streak signal decisions"
```

### Task 4: Stream the calendar into auditable signals

**Files:**
- Modify: `s7_measurement/signals.py`
- Modify: `test_s7_measurement.py`

**Interfaces:**
- Consumes: Task 2 validation and Task 3 decisions.
- Produces: `build_signal_observations(research_db, market_db, t1_db) -> dict`.

- [ ] **Step 1: Write a failing miniature-calendar integration test**

Create real temporary SQLite files with six dates and three stocks. Assert exact `signal_observations`, four-row B3 cohort expansion, separate `st_status_missing`, daily scanned counts, and no read of a seventh/future date.

- [ ] **Step 2: Run the integration test and verify RED**

Run: `./.venv/bin/python -m unittest test_s7_measurement.S7SignalStreamingTest -v`

Expected: `build_signal_observations` is missing.

- [ ] **Step 3: Implement day-by-day signal construction**

Preload only the modest T1 lifecycle, ST, namechange, and gate tables. Query one market day and one daily-basic day at a time. Maintain `deque(maxlen=3)` per code. Persist only raw 2-streak observations, expand valid cohorts, and write daily scan/exclusion counts. A failed run clears signal and downstream tables and records a failed manifest.

- [ ] **Step 4: Run Task 4 tests and verify GREEN**

Run: `./.venv/bin/python -m unittest test_s7_measurement.S7SignalStreamingTest -v`

Expected: all Task 4 tests pass.

- [ ] **Step 5: Commit Task 4**

```bash
git add s7_measurement/signals.py test_s7_measurement.py
git commit -m "feat(s7): stream auditable signal observations"
```

### Task 5: Measure returns, benchmarks, and five-day drawdown

**Files:**
- Create: `s7_measurement/measure.py`
- Modify: `test_s7_measurement.py`

**Interfaces:**
- Consumes: complete signal manifest and stored cohorts.
- Produces: `run_measurement(research_db, market_db, t1_db) -> dict`.
- Produces: `adjusted_return(start, target) -> float` and `net_return(gross) -> float`.

- [ ] **Step 1: Write failing hand-calculated return tests**

Use a six-session fixture with changing adjustment factors, one suspended target day, two equal-weight benchmark stocks, CSI300 closes, and known lows. Literal assertions include:

```python
self.assertAlmostEqual(row["gross_return"], 0.10)
self.assertAlmostEqual(row["net_return"], 0.0985)
self.assertAlmostEqual(row["equal_weight_return"], 0.05)
self.assertAlmostEqual(drawdown["max_adverse_excursion"], -0.08)
```

Also assert T+N uses the T1 date, a suspended stock retains the last reliable adjusted close, T+5 excludes signal-day low, and late-window rows receive `out_of_study_range`.

- [ ] **Step 2: Run Task 5 tests and verify RED**

Run: `./.venv/bin/python -m unittest test_s7_measurement.S7MeasurementStreamingTest -v`

Expected: missing `s7_measurement.measure`.

- [ ] **Step 3: Implement at-most-five-day active cohorts**

Stream the T1 calendar. Maintain current reliable adjusted closes and last quote dates. For each signal date, snapshot valid all-market starts, signal starts, and CSI300 start. Resolve due T+1/2/3/5 rows on their exact calendar dates. Track adjusted lows only for T+1 through T+5. Write benchmark, return, and drawdown rows in transactions.

- [ ] **Step 4: Implement lifecycle and invalid-status handling**

A target on or after delisting is invalid. A stock with no reliable start or target is invalid. A missing benchmark is explicit. Late-window rows retain observation counts but no numeric return. No invalid observation disappears from the table.

- [ ] **Step 5: Run Task 5 tests and verify GREEN**

Run: `./.venv/bin/python -m unittest test_s7_measurement.S7MeasurementStreamingTest -v`

Expected: all Task 5 tests pass.

- [ ] **Step 6: Commit Task 5**

```bash
git add s7_measurement/measure.py test_s7_measurement.py
git commit -m "feat(s7): measure fixed horizons and drawdown"
```

### Task 6: Aggregate slices and apply the 16-cell decision

**Files:**
- Modify: `s7_measurement/measure.py`
- Modify: `test_s7_measurement.py`

**Interfaces:**
- Produces: `aggregate_statistics(research_db, run_id) -> dict`.
- Produces: `qualification_decision(research_db, run_id) -> dict`.

- [ ] **Step 1: Write failing statistic and decision tests**

Use literal net returns `[-0.02, 0.00, 0.01, 0.03]` to check mean, median, inclusive quartiles, and positive rate. Add fixtures for exact threshold equality, missing year, five negative years, cross-horizon criterion mixing, and one passing cell among fifteen failures.

- [ ] **Step 2: Run Task 6 tests and verify RED**

Run: `./.venv/bin/python -m unittest test_s7_measurement.S7AggregationDecisionTest -v`

Expected: missing aggregation symbols or tables.

- [ ] **Step 3: Implement complete slice materialization**

Materialize every group × streak × horizon × year/all × gate/all/open/closed cell, including empty cells. Join drawdown through `signal_cohorts`. Single-observation quartiles equal that observation; multi-observation quartiles use inclusive interpolation.

- [ ] **Step 4: Implement exact 16-cell qualification**

For each A2/A3/B2/B3 × horizon, use the overall net mean and net positive rate plus all eleven yearly net means. Store all three gate values and the cell result. Overall result is `any(cell_pass)` only.

- [ ] **Step 5: Run Task 6 tests and verify GREEN**

Run: `./.venv/bin/python -m unittest test_s7_measurement.S7AggregationDecisionTest -v`

Expected: all Task 6 tests pass.

- [ ] **Step 6: Commit Task 6**

```bash
git add s7_measurement/measure.py test_s7_measurement.py
git commit -m "feat(s7): aggregate frozen qualification cells"
```

### Task 7: Add the CLI and deterministic reports

**Files:**
- Create: `s7_measurement/report.py`
- Create: `s7_measurement_cli.py`
- Modify: `test_s7_measurement.py`

**Interfaces:**
- Produces: `generate_reports(research_db, measurement_report, acceptance_report, detail_csv) -> dict`.
- Produces CLI commands in the design with JSON stdout and nonzero failure status.

- [ ] **Step 1: Write failing report and CLI tests**

Seed a completed miniature run. Assert the report contains the coverage percentage, `st_status_missing` count, all 16 cells, the frozen thresholds, and the unique overall result. Assert that neither the database schema nor the CSV introduces order, position, account, or trade-execution fields. Assert `measure` refuses a non-complete signal manifest and `report` refuses a non-complete measurement manifest.

- [ ] **Step 2: Run Task 7 tests and verify RED**

Run: `./.venv/bin/python -m unittest test_s7_measurement.S7ReportCliTest -v`

Expected: missing report and CLI symbols.

- [ ] **Step 3: Implement deterministic Markdown and CSV generation**

Generate tables in stable group, streak, horizon, year, and gate order. The acceptance report lists commands, source hashes, test counts, table row counts, exclusions, and the 16-cell verdict. The CSV contains detailed cohort returns and statuses but no account or trade columns.

- [ ] **Step 4: Implement the CLI gates**

Each command initializes only the S7 schema, validates required upstream state, calls one stage, and prints sorted JSON. `fetch-daily-basic` is the only command allowed to call the existing Tushare client.

- [ ] **Step 5: Run Task 7 tests and verify GREEN**

Run: `./.venv/bin/python -m unittest test_s7_measurement.S7ReportCliTest -v`

Expected: all Task 7 tests pass.

- [ ] **Step 6: Run the complete S7 unit suite**

Run: `./.venv/bin/python -m unittest test_s7_measurement.py -v`

Expected: all S7 tests pass with no warnings.

- [ ] **Step 7: Commit Task 7**

```bash
git add s7_measurement/report.py s7_measurement_cli.py test_s7_measurement.py
git commit -m "feat(s7): add offline measurement reports"
```

### Task 8: Run the frozen measurement and publish the evidence

**Files:**
- Create: `docs/测量报告-S7-连阳尾盘信号-20260801.md`
- Create: `docs/验收-S7-连阳尾盘信号测量-20260801.md`
- Create: `reports/s7-streak-signal-measurements-20260801.csv`

**Interfaces:**
- Consumes all earlier tasks and the already-audited local inputs.
- Produces final branch and mirror hashes.

- [ ] **Step 1: Recheck source hashes and data coverage**

Run the three source SHA-256 commands and `audit-data`. Compare them with the start-confirmation hashes. Stop on any mismatch.

- [ ] **Step 2: Build signals**

Run:

```bash
./.venv/bin/python s7_measurement_cli.py build-signals \
  --research-db S7_research.local.db \
  --market-db ../../market.db \
  --t1-db ../s3-stage1-data-clusters/S3_research.db
```

Expected: complete manifest, explicit signal/cohort counts, and separate `st_status_missing` count.

- [ ] **Step 3: Measure and report**

Run:

```bash
./.venv/bin/python s7_measurement_cli.py measure \
  --research-db S7_research.local.db \
  --market-db ../../market.db \
  --t1-db ../s3-stage1-data-clusters/S3_research.db

./.venv/bin/python s7_measurement_cli.py report \
  --research-db S7_research.local.db \
  --measurement-report docs/测量报告-S7-连阳尾盘信号-20260801.md \
  --acceptance-report docs/验收-S7-连阳尾盘信号测量-20260801.md \
  --detail-csv reports/s7-streak-signal-measurements-20260801.csv
```

Expected: reports derive from complete manifests and contain one overall conclusion.

- [ ] **Step 4: Run fresh verification**

Run:

```bash
./.venv/bin/python -m unittest test_s7_measurement.py -v
./.venv/bin/python -m unittest test_rules.py test_rules_mid.py test_regime.py
./.venv/bin/python -m unittest test_report_mirror_sync.py
git diff --check
sqlite3 -readonly S7_research.local.db 'PRAGMA integrity_check;'
```

Expected: every suite passes, diff check is empty, and SQLite returns `ok`.

- [ ] **Step 5: Audit scope and source immutability**

Confirm only S7 files and final artifacts changed. Recompute S6, T1, and market hashes; each must equal the start-confirmation value. Confirm no production, strategy, static, scheduler, or dependency file changed.

- [ ] **Step 6: Commit final artifacts**

```bash
git add s7_measurement s7_measurement_cli.py test_s7_measurement.py \
  docs/测量报告-S7-连阳尾盘信号-20260801.md \
  docs/验收-S7-连阳尾盘信号测量-20260801.md
git add -f reports/s7-streak-signal-measurements-20260801.csv
git commit -m "research(s7): measure frozen streak signal"
```

- [ ] **Step 7: Sync and independently verify the mirror**

Run the HTTPS mirror sync for `codex/s7-streak-signal-measurement`, then clone `guanlan-reports` into a new temporary directory. Verify the mirror HEAD, source commit metadata, both Markdown reports, CSV, and file hashes.

- [ ] **Step 8: Stop on the branch and report evidence**

Report the branch name, source commit, mirror commit, 16-cell decision, data/test evidence, and report paths. Do not merge, deploy, enter shadow mode, or propose optimized parameters.
