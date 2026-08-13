# Streak Volume-Ratio Review Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Reconstruct the frozen S8 streak watchlist from local read-only history, measure win rate and return paths across four fixed volume-ratio buckets, and publish a reproducible public evidence bundle.

**Architecture:** A new offline `streak_volume_review` package attaches read-only S7, T1, thermometer-sidecar, market, and S9-validation databases to one isolated writable research database. SQL window functions materialize exact streak counts, stock volume ratios, and daily historical industry heat; Python computes four forward horizons, drawdowns, ordered-trend tests, Holm corrections, and stable exports. No module is imported by the production service.

**Tech Stack:** Python 3.11 standard library, SQLite window functions, `unittest`, CSV/JSON/Markdown, existing HTTPS mirror synchronizer.

## Global Constraints

- The predeclaration commit `c933587` is immutable and precedes every data read.
- Signal dates end on 2026-08-12; local full-capitalization history ends on 2025-12-31 and the uncovered 2026 interval must remain visible.
- The four volume buckets are `<0.8`, `[0.8,1.0)`, `[1.0,1.5)`, and `>=1.5`.
- Horizons are T+1, T+5, T+10, and T+20; wins require gross adjusted return `>0`.
- The ±1% subgroup is inclusive on both boundaries and must be reported beside the full sample.
- All external databases are read-only; only `streak_volume_review.local.db`, `docs/`, and `reports/` may be written.
- The conclusion must retain: “买法已经 S7/S9 证伪；本研究是复盘学习，不是复活信号。”
- No network data fetch, parameter scan, production integration, deployment, or restart.

---

### Task 1: Frozen Configuration and Database Boundary

**Files:**
- Create: `streak_volume_review/__init__.py`
- Create: `streak_volume_review/config.py`
- Create: `streak_volume_review/db.py`
- Create: `test_streak_volume_review.py`

**Interfaces:**
- Produces: `FROZEN_CONFIG: dict[str, object]` and `CONFIG_FINGERPRINT: str`.
- Produces: `connect_external_ro(path) -> sqlite3.Connection`.
- Produces: `init_research_db(path) -> pathlib.Path`, restricted to `streak_volume_review.local.db`.
- Produces: `attach_external_ro(connection, alias, path) -> None`.

- [ ] **Step 1: Write a failing database-boundary test.**

```python
def test_external_inputs_are_query_only_and_research_name_is_fixed(self):
    with connect_external_ro(source) as connection:
        self.assertEqual(connection.execute("PRAGMA query_only").fetchone()[0], 1)
        with self.assertRaises(sqlite3.OperationalError):
            connection.execute("CREATE TABLE forbidden(x)")
    with self.assertRaises(ValueError):
        init_research_db(root / "market.db")
```

- [ ] **Step 2: Run `python -m unittest test_streak_volume_review.ReviewDatabaseTests -v`; verify import failure.**
- [ ] **Step 3: Implement the immutable config, URI `mode=ro` connections, read-only attachments, and isolated schema.**
- [ ] **Step 4: Re-run the database tests and commit the green boundary.**

### Task 2: Historical Feature Reconstruction

**Files:**
- Create: `streak_volume_review/reconstruction.py`
- Modify: `streak_volume_review/db.py`
- Modify: `test_streak_volume_review.py`

**Interfaces:**
- Produces: `build_stock_features(connection) -> dict[str, int]`.
- Produces: `build_historical_heat(connection) -> dict[str, int]`.
- Produces: `build_watchlist_rows(connection) -> dict[str, int]`.
- Consumes: S7 `signal_observations`, T1 lifecycle/ST/name history, sidecar membership/cluster/volume, and market price/adjustment/index history.

- [ ] **Step 1: Write failing fixtures for exact streak and stock volume ratio.**

```python
def test_stock_features_use_valid_bars_and_fixed_twenty_bar_ratio(self):
    # A suspended calendar day does not create a bar; the frozen v1 list counts
    # consecutive valid bullish bars and divides target volume by the prior 20.
    row = feature_for("A", "2025-02-03")
    self.assertEqual(row["streak_count"], 5)
    self.assertAlmostEqual(row["volume_ratio"], 42 / 21)
```

- [ ] **Step 2: Run the focused test and verify the missing reconstruction API fails.**
- [ ] **Step 3: Materialize `stock_features` with SQLite windows: a non-bullish cumulative group for exact streak, `LAG(close)` for last-day change, and a 20-row preceding average for volume ratio.**
- [ ] **Step 4: Write failing fixtures for all four thermometer lights and heat states.**
- [ ] **Step 5: Materialize daily heat without future data: member-summed volume and 20-day ratio/rank, five strictly rising turnover shares, carried 120-bar stock returns with `CUME_DIST`, and the frozen cluster/heat classification.**
- [ ] **Step 6: Compare optimized heat to all 33,232 rows in the completed S9 validation database; require exact `heat` equality and report any light-level limitation separately.**
- [ ] **Step 7: Build one watchlist row per code/date from S7 base eligibility plus prior-year top-10 exclusion, as-of membership, and warm/hot heat.**
- [ ] **Step 8: Run reconstruction tests and commit.**

### Task 3: Forward Returns, Benchmark, and Drawdown

**Files:**
- Create: `streak_volume_review/measurement.py`
- Modify: `test_streak_volume_review.py`

**Interfaces:**
- Produces: `measure_watchlist(connection) -> dict[str, int]`.
- Produces: `measure_one_path(signal_date, base_price, calendar, prices, benchmark) -> dict[int, HorizonResult]`.
- Writes: `measurements`, one row per watchlist sample and horizon.

- [ ] **Step 1: Write failing literal-path tests.**

```python
def test_horizons_excess_and_mdd_are_close_to_close(self):
    result = measure_one_path("2025-01-02", 100, calendar, prices, benchmark)
    self.assertEqual(tuple(result), (1, 5, 10, 20))
    self.assertAlmostEqual(result[5].gross_return, 0.08)
    self.assertAlmostEqual(result[5].excess_return, 0.06)
    self.assertAlmostEqual(result[5].max_drawdown, -0.10)
```

- [ ] **Step 2: Verify the expected RED result.**
- [ ] **Step 3: Implement calendar advancement, last-reliable-close carry-forward, index excess, path MDD, `out_of_calendar`, and price-missing states.**
- [ ] **Step 4: Stream candidates by stock code so each price history is loaded once; write four horizon rows per sample.**
- [ ] **Step 5: Run measurement tests and commit.**

### Task 4: Fixed Buckets and Statistical Decisions

**Files:**
- Create: `streak_volume_review/statistics.py`
- Modify: `test_streak_volume_review.py`

**Interfaces:**
- Produces: `aggregate_statistics(connection) -> dict[str, object]`.
- Produces: `cochran_armitage(groups) -> tuple[float, float]`.
- Produces: `holm_adjust(p_values) -> list[float]`.
- Writes: `group_statistics`, `trend_results`, and `bucket_significance`.

- [ ] **Step 1: Write failing boundary tests for bucket assignment and the inclusive ±1% subgroup.**
- [ ] **Step 2: Verify RED, implement the four literal bucket boundaries, and re-run GREEN.**
- [ ] **Step 3: Write failing tests with literal win/loss counts for Cochran–Armitage z, two-proportion z, Holm monotonic adjustment, and the `<10` inference block.**
- [ ] **Step 4: Implement mean, median, MDD p10, ordered trends, 16 bucket-vs-rest comparisons, and frozen stable-direction verdicts.**
- [ ] **Step 5: Add auxiliary rows for exact 5, exact 6, 7+, warm, and hot without letting them affect the primary verdict.**
- [ ] **Step 6: Run statistics tests and commit.**

### Task 5: CLI, Full Run, and Public Artifacts

**Files:**
- Create: `streak_volume_review/report.py`
- Create: `streak_volume_review_cli.py`
- Create: `reports/streak-volume-ratio-detail-20260813.csv`
- Create: `reports/streak-volume-ratio-groups-20260813.csv`
- Create: `reports/streak-volume-ratio-overlap-20260813.json`
- Create: `reports/streak-volume-ratio-run-manifest-20260813.json`
- Create: `reports/streak-volume-ratio-artifact-hashes-20260813.json`
- Create: `docs/测量报告-投机线连阳名单胜率与量比关系-20260813.md`
- Create: `docs/验收-投机线连阳名单胜率与量比关系-20260813.md`
- Modify: `test_streak_volume_review.py`

**Interfaces:**
- Produces: `run` and `export` CLI commands with explicit input paths.
- Produces: `export_artifacts(research_db, repository_root) -> dict[str, str]`.

- [ ] **Step 1: Write failing export tests that require every artifact, the exact conclusion sentence, all groups, script hashes, input hashes, and explicit 2026 coverage gaps.**
- [ ] **Step 2: Verify RED and implement deterministic CSV/JSON/Markdown exports.**
- [ ] **Step 3: Run the full measurement with the production `market.db`/sidecar, frozen T1/S7 inputs, completed S9 validation database, and the two saved watchlist evidence JSON files.**
- [ ] **Step 4: Re-hash all read-only inputs and require before/after equality; run SQLite `integrity_check` and table conservation checks.**
- [ ] **Step 5: Export, independently recompute aggregate counts from the detail CSV, and commit code plus artifacts while excluding the local research database.**

### Task 6: Regression, Mirror Delivery, and Fresh-Clone Verification

**Files:**
- Modify: `docs/验收-投机线连阳名单胜率与量比关系-20260813.md`
- Modify: `reports/streak-volume-ratio-run-manifest-20260813.json`

**Interfaces:**
- Produces: source commit, mirror commit, and a fresh-clone verification record.

- [ ] **Step 1: Run `test_streak_volume_review.py`, S8 research tests, mirror tests, `compileall`, `git diff --check`, the sensitive-information blocker, and an independent CSV recomputation.**
- [ ] **Step 2: Record exact test counts, database hashes, script hashes, coverage limits, and the non-revival conclusion in the acceptance report.**
- [ ] **Step 3: Commit final docs/reports and run the HTTPS mirror synchronizer with its existing system-proxy inheritance.**
- [ ] **Step 4: Clone `guanlan-reports` into a new temporary directory; verify mirror HEAD, required paths, SHA-256, CSV row counts, and JSON parsing.**
- [ ] **Step 5: Preserve the research branch/worktree for audit and report source and mirror hashes.**
