# S5 Breadth-Timing ETF Offline Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build and test the complete offline S5 research, strategy, simulation, reporting, and immutable-ledger code without downloading ETF data or running the declared 2015—2025 backtest.

**Architecture:** New `s5_research` and `s5_backtest` packages stay isolated from the production pipeline. All external inputs enter through explicit read-only SQLite adapters or injected test clients; pure strategy/risk functions feed a single-ETF portfolio engine; deterministic metrics, report, and ledger modules consume the engine result. No existing production, S2, S3, EOD, service, or UI module imports S5.

**Tech Stack:** Python 3.11 standard library, SQLite, pandas and numpy already present in `requirements.txt`, `unittest`; zero new dependencies.

## Global Constraints

- Predeclaration ID is exactly `s5-breadth-timing-etf-2026-07-29`.
- Assets are exactly `510300.SH`, `510500.SH`, and `159915.SZ`.
- Fixed parameters are exactly: 120 trading-day momentum, strict `> 2%` switch buffer, strict `NH/eligible > 10%` overheat threshold, and a 10-common-trading-day review cadence. No scan, grid, optimizer, or CLI parameter override is allowed.
- Samples remain 2015—2021 in-sample and 2022—2025 out-of-sample. This plan must not run either sample.
- The production `market.db` and T1 `S3_research.db` are opened with SQLite `mode=ro`; tests use temporary snapshots. Existing tables, indexes, triggers, and files receive no writes.
- No ETF network request is permitted. Source code may exercise an injected fake client only. A real-client CLI command remains approval-locked until a separate data approval is recorded.
- The backtest engine may run only against tiny synthetic fixtures in unit tests. It must not open the real ETF dataset or produce declared reports in this phase.
- No changes to `requirements.txt`, S2, S3, `backtest_mid.py`, `rules*.py`, `regime.py`, `market_data.py`, EOD, production risk, service/API/UI, deployment, or shadow mode.
- Fee is fixed at 0.15% round trip, represented as 0.075% on each executed buy or sell. ETF orders use 100-share lots.
- Decisions use the current close and execute at the next common trading day's open. The last unexecuted decision is not force-filled.
- Momentum ties keep the current ETF. When entering from cash with a tie, the fixed asset order above is the deterministic tie-break.

---

### Task 1: Freeze configuration and the approval-locked research boundary

**Files:**
- Create: `s5_research/__init__.py`
- Create: `s5_research/db.py`
- Create: `s5_research/source.py`
- Create: `s5_research_cli.py`
- Create: `s5_backtest/__init__.py`
- Create: `s5_backtest/config.py`
- Create: `test_s5_research.py`

**Interfaces:**
- Produces: `init_research_db(path)`, `connect_research(path, readonly=False)`, `connect_readonly(path)`, `research_transaction(path)`.
- Produces: `SourceManifest`, `SourceRequest`, `normalize_fund_daily(frame)`, `fetch_approved(client, request)`, and `ingest_rows(con, rows)`.
- Produces: immutable configuration constants plus `frozen_config_payload()` and `frozen_config_hash()`.
- The source manifest contains an explicit non-empty approval ID. `fetch_approved` raises `DataApprovalRequired` before calling the client when approval is absent.

- [ ] **Step 1: Write failing configuration and database-boundary tests**

```python
def test_frozen_configuration_has_no_parameter_grid_or_override():
    from s5_backtest import config
    assert config.ETF_CODES == ("510300.SH", "510500.SH", "159915.SZ")
    assert config.MOMENTUM_DAYS == 120
    assert config.SWITCH_BUFFER == 0.02
    assert config.OVERHEAT_RATIO == 0.10
    assert not hasattr(config, "parameter_grid")

def test_research_schema_refuses_production_names_and_ro_rejects_writes():
    from s5_research.db import init_research_db, connect_research
    with self.assertRaisesRegex(ValueError, "production"):
        init_research_db(tmp / "market.db")
    init_research_db(tmp / "S5_research.db")
    with connect_research(tmp / "S5_research.db", readonly=True) as con:
        with self.assertRaises(sqlite3.OperationalError):
            con.execute("DELETE FROM s5_etf_daily")
```

- [ ] **Step 2: Run the tests and verify RED**

Run: `.venv/bin/python -m unittest -v test_s5_research`

Expected: import failure because `s5_research` and `s5_backtest.config` do not exist.

- [ ] **Step 3: Implement fixed configuration and the two-table sidecar schema**

The schema is:

```sql
CREATE TABLE s5_etf_daily(
  trade_date TEXT NOT NULL,
  ts_code TEXT NOT NULL,
  open REAL NOT NULL,
  high REAL NOT NULL,
  low REAL NOT NULL,
  close REAL NOT NULL,
  vol REAL,
  amount REAL,
  source_name TEXT NOT NULL,
  source_row_hash TEXT NOT NULL,
  PRIMARY KEY(trade_date, ts_code)
) WITHOUT ROWID;

CREATE TABLE source_audit(
  source_name TEXT NOT NULL,
  ts_code TEXT NOT NULL,
  requested_start TEXT NOT NULL,
  requested_end TEXT NOT NULL,
  actual_start TEXT,
  actual_end TEXT,
  row_count INTEGER NOT NULL,
  missing_count INTEGER NOT NULL,
  duplicate_count INTEGER NOT NULL,
  ohlc_invalid_count INTEGER NOT NULL,
  coverage_pct REAL,
  source_hash TEXT,
  status TEXT NOT NULL,
  fetched_at TEXT NOT NULL,
  PRIMARY KEY(source_name, ts_code, requested_start, requested_end)
);
```

`connect_readonly` uses an absolute URI with `mode=ro` and immediately enables `PRAGMA query_only=ON`. Research initialization rejects paths whose basename is `market.db` or `screener.db`.

- [ ] **Step 4: Add approval-gate and normalization tests**

The fake client exposes `fund_daily(**kwargs)` and records calls. Tests must prove:

- no approval ID means zero client calls;
- approved requests use only the fixed asset and field list;
- normalization rejects a wrong code, duplicate key, non-positive OHLC, `high < max(open, close)`, or `low > min(open, close)`;
- source rows are canonically hashed and idempotently upserted into a temporary sidecar.

- [ ] **Step 5: Implement the injected-client boundary and non-network CLI**

`s5_research_cli.py` exposes only `init`, `status`, and `validate-file` in this phase. It deliberately has no command that constructs a Tushare client or performs a fetch. `fetch_approved` remains callable only by tests with an injected client; the real fetch command is added after data approval.

- [ ] **Step 6: Run Task 1 tests**

Run: `.venv/bin/python -m unittest -v test_s5_research`

Expected: all Task 1 tests pass and no file named `S5_research.db` exists in the worktree.

- [ ] **Step 7: Commit Task 1**

```bash
git add s5_research s5_research_cli.py s5_backtest/__init__.py s5_backtest/config.py test_s5_research.py
git commit -m "feat(s5): add approval-locked research boundary"
```

### Task 2: Add read-only data adapters and a fail-loud prepared calendar

**Files:**
- Create: `s5_backtest/data.py`
- Modify: `test_s5_backtest.py` (create in this task)

**Interfaces:**
- Produces: `load_etf_daily(path, start, end)`, `load_breadth(path, start, end)`, and `load_regimes(path, start, end)`.
- Produces: `prepare_days(etf, breadth, regimes) -> tuple[PreparedDay, ...]`.
- `PreparedDay` contains `trade_date`, per-code OHLC/momentum, `gate_open`, `eligible_n`, `nh250`, and `regime`.

- [ ] **Step 1: Write failing read-only adapter tests**

Use three temporary SQLite files. Assert that:

- each adapter returns rows ordered by date;
- T1 `breadth_daily.trade_date` remains the decision date;
- production `regime_history.judged_date` is explicitly aliased to `trade_date`;
- connections reject writes;
- unsupported `formula_version` or `engine_ver` raises `InputVersionError`.

- [ ] **Step 2: Verify RED**

Run: `.venv/bin/python -m unittest -v test_s5_backtest.DataBoundaryTests`

Expected: import failure because `s5_backtest.data` does not exist.

- [ ] **Step 3: Implement adapters using `mode=ro` only**

Do not import `regime.py`, `market_data.py`, `s3_research.runner`, or any initializer. SQL selects only the materialized columns required by S5 and verifies the frozen versions `s3-stage1-v1` and `regime_v1`.

- [ ] **Step 4: Write failing maturity, missing-data, and momentum tests**

Fixtures use literal closes. Tests prove:

- 249 bars produce no prepared day; 250 bars produce the first;
- momentum is `close[t] / close[t-120] - 1`, independently hand-calculated;
- any missing ETF, breadth, or regime row after the common maturity date raises `MissingDailyInputError`;
- appending future rows does not change the serialized earlier `PreparedDay` values.

- [ ] **Step 5: Implement `prepare_days`**

Build the expected calendar from the intersection of breadth and regime dates, require all three ETF rows after the latest 250th valid-bar date, and never forward-fill. Compute each code's momentum from its own prior valid rows.

- [ ] **Step 6: Run Task 2 tests and commit**

Run: `.venv/bin/python -m unittest -v test_s5_backtest.DataBoundaryTests test_s5_backtest.PreparedDataTests`

```bash
git add s5_backtest/data.py test_s5_backtest.py
git commit -m "feat(s5): add read-only prepared data boundary"
```

### Task 3: Implement the mechanical strategy and R1 risk state machine

**Files:**
- Create: `s5_backtest/strategy.py`
- Create: `s5_backtest/risk.py`
- Modify: `test_s5_backtest.py`

**Interfaces:**
- Produces: `base_target(gate_open, regime, nh250, eligible_n) -> BaseTarget`.
- Produces: `is_review_day(common_index) -> bool`.
- Produces: `select_asset(momentums, current_asset, target_weight, review_due) -> SelectionDecision`.
- Produces: `RiskState.mark(equity, trade_date) -> RiskSnapshot`.

- [ ] **Step 1: Write failing three-axis target tests**

Table-drive the literal expected weights and reasons:

```python
cases = [
    (False, "bull_healthy", 5, 100, 0.0, "breadth_closed"),
    (True, "bull_stressed", 5, 100, 0.5, "regime_half"),
    (True, "neutral_stressed", 5, 100, 0.5, "regime_half"),
    (True, "bear_stressed", 5, 100, 1.0, "gate_open"),
    (True, "bull_healthy", 10, 100, 1.0, "gate_open"),
    (True, "bull_healthy", 11, 100, 0.0, "overheat"),
]
```

Invalid or zero `eligible_n` raises instead of silently opening the gate.

- [ ] **Step 2: Verify RED, then implement the minimal target function**

Run: `.venv/bin/python -m unittest -v test_s5_backtest.StrategyTests`

Expected first run: missing module. After implementation: all target cases pass.

- [ ] **Step 3: Write failing review and selection tests**

Assert review indices `0, 10, 20`; 2.00 percentage points does not switch; 2.01 does; cash reopening chooses the current leader even off-cycle; a momentum tie keeps the holding; a cash tie uses fixed ETF order.

- [ ] **Step 4: Implement selection and keep the tests green**

`select_asset` may not accept lookback, buffer, cadence, or asset-list overrides. It reads frozen constants only.

- [ ] **Step 5: Write failing R1 boundary and hysteresis tests**

For a peak of 100:

- equity `92.001` keeps multiplier `1.0`;
- `92.0` sets `0.5`;
- `88.001` stays `0.5`;
- `88.0` sets `0.25`;
- `99.999` remains at the lowest attained tier;
- `100.0` restores `1.0`;
- a later new peak of `105.0` becomes the next recovery reference.

- [ ] **Step 6: Implement R1 and commit**

Run: `.venv/bin/python -m unittest -v test_s5_backtest.StrategyTests test_s5_backtest.RiskTests`

```bash
git add s5_backtest/strategy.py s5_backtest/risk.py test_s5_backtest.py
git commit -m "feat(s5): add fixed signals and drawdown tiers"
```

### Task 4: Implement next-open portfolio execution and the offline engine

**Files:**
- Create: `s5_backtest/portfolio.py`
- Create: `s5_backtest/engine.py`
- Modify: `test_s5_backtest.py`

**Interfaces:**
- Produces: `Portfolio.execute(order, open_prices) -> RebalanceEvent | None`.
- Produces: `Portfolio.mark_to_market(close_prices) -> float`.
- Produces: `run_fixture(days, initial_capital=500_000.0) -> BacktestResult`.
- `run_fixture` is the same deterministic engine later used by the declared runner; the phase restriction is enforced operationally by not calling it with real input in this task.

- [ ] **Step 1: Write failing fee, lot, cash, and event tests**

Use two literal days and prove:

- a close decision produces no same-day trade;
- the next open executes with a 0.075% side fee;
- buys round down to a 100-share lot and never make cash negative;
- a 100%→50% reduction, 50%→0% exit, and same-day ETF switch each create one execution-date event;
- a switch sells before buying and records total fees once per leg;
- unchanged code and weight creates no event.

- [ ] **Step 2: Verify RED and implement `Portfolio`**

Run: `.venv/bin/python -m unittest -v test_s5_backtest.PortfolioTests`

Expected: missing module, then all literal account identities pass after implementation.

- [ ] **Step 3: Write failing engine sequencing tests**

Synthetic `PreparedDay` fixtures prove:

- yesterday's target executes at today's open;
- today's close updates equity and R1 before today's target is queued;
- final weight equals base target times the current R1 multiplier;
- overheat and gate-close decisions liquidate at next open;
- cash reopening selects the momentum leader immediately;
- future fixture rows do not change past decisions or events.

- [ ] **Step 4: Implement the engine and deterministic benchmark helper**

The 510300 buy-and-hold helper enters on the next open after the first prepared decision day, uses the same fee and lot rule, and never force-liquidates at period end.

- [ ] **Step 5: Run Task 4 tests and commit**

Run: `.venv/bin/python -m unittest -v test_s5_backtest.PortfolioTests test_s5_backtest.EngineTests`

```bash
git add s5_backtest/portfolio.py s5_backtest/engine.py test_s5_backtest.py
git commit -m "feat(s5): add next-open ETF portfolio engine"
```

### Task 5: Add metrics, G1, invalidation, reporting, and strong sealing

**Files:**
- Create: `s5_backtest/metrics.py`
- Create: `s5_backtest/report.py`
- Create: `s5_backtest/ledger.py`
- Create: `s5_backtest_cli.py`
- Modify: `test_s5_backtest.py`

**Interfaces:**
- Produces: `compute_metrics(equity, events)`, `evaluate_g1(strategy, benchmark)`, and `evaluate_invalidation(cycles, strategy, benchmark)`.
- Produces: `render_report(context)` and `write_deliverables(...)`.
- Produces: `claim_run`, `store_period_once`, `register_artifact`, `seal_run`, and `verify_run`.
- The CLI exposes `status`, `verify`, and approval-safe preparation. A real `run-is` or `run-oos` command refuses unless a complete approved data manifest exists.

- [ ] **Step 1: Write failing metrics and gate tests**

Use hand-derived equity arrays to verify annualization, peak-to-trough drawdown, yearly returns, positive-year count, and events/11 average rebalances. Assert each G1 criterion independently, and all-four conjunction.

- [ ] **Step 2: Verify RED and implement metrics**

Run: `.venv/bin/python -m unittest -v test_s5_backtest.MetricsTests`

- [ ] **Step 3: Write failing invalidation tests**

At 29 cycles the result is always `eligible=False`. At 30 cycles, strategy drawdown strictly greater than benchmark or annual return strictly below benchmark by more than 4 percentage points returns `stop=True`; equality does not.

- [ ] **Step 4: Write failing immutable-ledger tests**

A synthetic complete run stores exactly one in-sample result and one out-of-sample result, registers temporary artifacts, seals, verifies a 64-character root, and then proves INSERT/UPDATE/DELETE fail for every protected table. A second out-of-sample store raises `OutOfSampleAlreadyRunError`; a second claim after sealing raises `RunAlreadySealedError`; changing input or config hashes rejects resume.

- [ ] **Step 5: Implement the fixed-run ledger**

Unlike S3, S5 has no parameter tables. The protected tables are `backtest_runs`, `period_results`, and `run_artifacts`. The root hash is canonical JSON over the predeclaration ID, input/config hashes, run creation time, both period rows, and artifact hashes. The sealed root is computed internally, never accepted from a caller.

- [ ] **Step 6: Write failing report-contract tests**

The rendered report must contain:

- full-period and yearly strategy/benchmark rows;
- 2015, 2018, and 2022 sections;
- all rebalance events with signal date, execution date, direction, and reasons;
- data/config/code fingerprints;
- four mechanical G1 conclusions;
- the 30-cycle invalidation rule;
- no sample result hardcoded into source or tests.

- [ ] **Step 7: Implement report and approval-safe CLI**

The CLI must never create a production DB. Without a data approval manifest it exits before loading ETF rows. Unit tests exercise argument parsing and refusal only; no declared backtest command is run.

- [ ] **Step 8: Run Task 5 tests and commit**

Run: `.venv/bin/python -m unittest -v test_s5_backtest`

```bash
git add s5_backtest s5_backtest_cli.py test_s5_backtest.py
git commit -m "feat(s5): add metrics reporting and sealed ledger"
```

### Task 6: Prove scope isolation and prepare the data approval packet

**Files:**
- Create: `docs/S5-ETF数据审批单-20260730.md`
- Modify only if tests reveal an S5 defect: files created in Tasks 1—5.

**Interfaces:**
- No new runtime interface.

- [ ] **Step 1: Run all S5 tests**

Run: `.venv/bin/python -m unittest -v test_s5_research test_s5_backtest`

Expected: all pass using temporary files and fake clients only.

- [ ] **Step 2: Run syntax and dependency checks**

Run: `.venv/bin/python -m compileall -q s5_research s5_backtest s5_research_cli.py s5_backtest_cli.py`

Run: `git diff 7dd422b -- requirements.txt`

Expected: compilation succeeds and dependency diff is empty.

- [ ] **Step 3: Run existing focused and full regression suites**

Run: `.venv/bin/python test_rules.py`

Run: `.venv/bin/python test_rules_mid.py`

Run: `.venv/bin/python test_regime.py`

Run: `.venv/bin/python -m unittest -v test_s3_research test_s3_ui`

Run: `.venv/bin/python test_web.py`

Expected: `8/8`, `20/20`, `9/9`, `50/50`, and `834/834`.

- [ ] **Step 4: Prove production and retired-artifact immutability**

Record SHA-256 before and after for:

- `market.db`;
- T1 `S3_research.db`;
- all tracked S3 v0.1/v0.2/v0.3 artifacts visible from their retained branches.

Run: `git diff --name-only 7dd422b..HEAD` and verify every path belongs to S5 code, S5 tests, or S5 documentation.

- [ ] **Step 5: Write the separate data approval packet**

The packet states:

- source endpoint and official field definitions;
- fixed codes and approved listing-date-to-2025-12-31 request intervals;
- expected and actual coverage fields (actual values remain `待取数`, never invented);
- destination schema and production-zero-write proof;
- duplicate/OHLC/common-calendar/coverage/SHA-256 checks;
- the exact command that remains prohibited until approval.

- [ ] **Step 6: Commit verification documents**

```bash
git add docs/S5-ETF数据审批单-20260730.md
git commit -m "docs(s5): submit ETF data approval packet"
```

- [ ] **Step 7: Stop**

Do not execute a network request, initialize the real `S5_research.db`, load real ETF data, run in-sample, run out-of-sample, create declared result artifacts, seal a real run, merge, deploy, or attach to production. Report the branch state and wait for explicit data approval.
