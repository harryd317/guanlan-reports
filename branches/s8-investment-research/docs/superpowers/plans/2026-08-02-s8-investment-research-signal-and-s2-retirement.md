# S8 Investment Research Signal and S2 Retirement Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the S8 research-signal, archive-queue, and shadow-ledger system in an isolated sidecar, then retire S2 signal/review generation and switch the new frontend to S8 in one guarded production deployment.

**Architecture:** Phase A first executes `docs/superpowers/plans/2026-08-02-industry-thermometer-production-closure.md` through production-equivalent acceptance but stops before production deployment and mirror sync. Phase B creates a focused `s8_research` package whose SQLite writer can touch only three tables; it consumes the Phase A sidecar and production inputs through read-only adapters. Only after both equivalent environments pass does one deployment install both sidecars, migrate exactly nine S2 pool rows to legacy state, stop S2 signal/review generation, and expose S8 read models.

**Tech Stack:** Python 3.11, SQLite, `unittest`, existing Tushare Pro client, FastAPI, vanilla HTML/CSS/JavaScript, macOS LaunchAgent, in-app browser.

## Global Constraints

- Execute Phase A first. Its three insurance locks remain unchanged: frozen T1 SHA-256, sidecar-only writes, and no snapshot advance after any failed stage.
- S8 runtime creates and writes exactly `s8_signals`, `s8_shadow_ledger`, and `archive_queue` in `S8_research.db`. Do not use `AUTOINCREMENT`.
- Open every SQLite input with `mode=ro` and immediately set `PRAGMA query_only=ON`.
- Use only the existing Tushare `daily_basic` and `stock_basic` primary channel for current capitalization/name input. Never call a fallback provider.
- A date mismatch suppresses all new S8 signals for that day. Never reuse stale market-cap data as current.
- S8 has no real-order endpoint and cannot import real position/account mutation functions.
- Preserve the S2 source and parameter files. The only existing-database write is the separately authorized, guarded, one-time nine-row S2 legacy migration.
- Keep the market-data pipeline, market clock, account module, two-book labels, and the 1%/30%/60%/8% risk skeleton.
- Production deployment requires both phase reports, both production-equivalent acceptances, desktop/mobile evidence, hashes, health, and a successful rollback rehearsal.
- Final completion requires one HTTPS mirror push and a fresh-clone verification with a reported mirror commit hash.

---

## Phase A: Execute the approved thermometer closure plan

Use `docs/superpowers/plans/2026-08-02-industry-thermometer-production-closure.md` as the exact executable plan. Complete Tasks 1–8. In Task 9, produce an installable sidecar candidate and deployment manifest but do not modify the production plist, restart production, or push the mirror. Phase B may begin only when:

```text
frozen_t1_sha256 == 111f2017d476e87dfdc68d3541d7678630d466d410a91f4ab37d8bcb9ce049d5
snapshot.data_date == market.latest_trade_date
snapshot.is_stale == false
equivalent_health.ok == true
equivalent_health.match == true
```

Commit the Phase A equivalent report and screenshots as `test: accept thermometer production equivalent`.

---

### Task 1: Create the three-table S8 database boundary

**Files:**
- Create: `s8_research/__init__.py`
- Create: `s8_research/db.py`
- Create: `test_s8_research.py`

**Interfaces:**
- Produces: `default_s8_db() -> Path`
- Produces: `init_s8_db(path: str | Path) -> None`
- Produces: `connect_s8(path, readonly=False) -> sqlite3.Connection`
- Produces: `connect_external_ro(path) -> sqlite3.Connection`
- Produces: `s8_transaction(path)` context manager

- [ ] **Step 1: Write failing database-boundary tests**

```python
def test_init_creates_exactly_three_tables_without_sequence(self):
    init_s8_db(self.path)
    with sqlite3.connect(self.path) as con:
        names = {r[0] for r in con.execute(
            "SELECT name FROM sqlite_master WHERE type='table'")}
    self.assertEqual(names, {"s8_signals", "s8_shadow_ledger", "archive_queue"})

def test_rejects_non_s8_filename(self):
    with self.assertRaises(ValueError):
        init_s8_db(self.root / "screener.db")
```

- [ ] **Step 2: Verify RED**

Run: `.venv/bin/python -m unittest test_s8_research.S8DatabaseTests -v`
Expected: import failure because `s8_research.db` does not exist.

- [ ] **Step 3: Implement the minimal schema and path guard**

Use text primary keys and the exact columns from the approved design. Set `PRAGMA foreign_keys=ON`; `s8_shadow_ledger.signal_id` references `s8_signals.signal_id`. Reject every writable basename except `S8_research.db` and `S8_research.test.db`.

- [ ] **Step 4: Add read-only mutation proof**

Open a fixture through `connect_external_ro()`, attempt `CREATE TABLE forbidden(x)`, and assert `sqlite3.OperationalError`.

- [ ] **Step 5: Verify GREEN and commit**

Run: `.venv/bin/python -m unittest test_s8_research.S8DatabaseTests -v`
Expected: all database tests pass and no fourth table exists.

Commit: `feat(s8): add isolated three-table research database`.

### Task 2: Freeze current inputs, hunter universe, and window semantics

**Files:**
- Create: `s8_research/sources.py`
- Create: `s8_research/universe.py`
- Create: `s8_research/windows.py`
- Modify: `test_s8_research.py`

**Interfaces:**
- Produces: `load_daily_inputs(target_date, thermometer_db, market_db, pro) -> DailyInputs`
- Produces: `build_hunting_ground(inputs) -> tuple[list[HuntingStock], list[Exclusion]]`
- Produces: `blood_window_state(net5_rows, target_date) -> BloodWindow`
- Produces: `evaluate_candidate(stock, industry_heat, blood_window) -> CandidateDecision`

- [ ] **Step 1: Write failing date/source tests**

```python
def test_daily_basic_date_must_equal_target(self):
    pro = FakePro(daily_basic_date="20260730")
    with self.assertRaisesRegex(ValueError, "market-cap date mismatch"):
        load_daily_inputs("2026-07-31", self.t1, self.market, pro)
    self.assertEqual(pro.called_endpoints, ["daily_basic", "stock_basic"])
```

Assert fields are exactly `ts_code,trade_date,close,total_mv` and `ts_code,symbol,name,list_date`; assert no fallback function is referenced.

- [ ] **Step 2: Verify RED**

Run: `.venv/bin/python -m unittest test_s8_research.S8SourceTests -v`
Expected: missing `load_daily_inputs`.

- [ ] **Step 3: Implement read-only adapters and strict live-source validation**

Normalize codes to six digits. Require finite positive `total_mv`, exact target date, current stock names, and a matching Phase A thermometer snapshot. Return immutable dataclasses; write nothing.

- [ ] **Step 4: Write hunter boundary tests**

Cover `total_mv == 5_000_000`, `== 10_000_000`, listing age 364/365 days, current ST, historical ST label, and previous-year rank 10/11. Compute annual rank from adjusted first/last closes in `market.db`.

- [ ] **Step 5: Write window RED tests**

```python
def test_reopen_is_third_positive_day_after_ten_negative_days(self):
    rows = net5_series([-1] * 10 + [1, 2, 3])
    state = blood_window_state(rows, rows[-1].trade_date)
    self.assertEqual(state.reopen_date, rows[-1].trade_date)
    self.assertTrue(state.active)
```

Also cover day 30/31 natural-window boundaries; 60-day high; volume-ratio 1.0/3.0; scorching exclusion; cooling drawdown 5%/20%; 120-day return zero; and volume ratio 0.8.

- [ ] **Step 6: Implement minimal hunter/window functions and verify GREEN**

Run: `.venv/bin/python -m unittest test_s8_research.S8SourceTests test_s8_research.S8UniverseTests test_s8_research.S8WindowTests -v`
Expected: all input, universe, and window tests pass.

Commit: `feat(s8): freeze hunter and signal windows`.

### Task 3: Implement archive gate, transition queue, and data skeleton

**Files:**
- Create: `s8_research/archives.py`
- Modify: `test_s8_research.py`
- Modify: `industry_archives.py`

**Interfaces:**
- Produces: `archive_gate(industry_name, archive_root) -> ArchiveGate`
- Produces: `extract_falsification(markdown_text) -> list[str]`
- Produces: `advance_archive_queue(s8_db, previous_snapshot, current_snapshot, archive_root) -> list[dict]`
- Produces: `build_industry_skeleton(inputs, industry_code) -> dict`
- Produces: `seed_core_archive_queue(s8_db, snapshot, archive_root) -> dict`

- [ ] **Step 1: Write failing archive-gate tests**

Create fixtures for approved, draft, dormant, and missing-falsification archives. Assert only an eligible archive with at least one falsification bullet permits a stock signal. Assert the existing status `样板（格式验收用）` is eligible.

- [ ] **Step 2: Verify RED**

Run: `.venv/bin/python -m unittest test_s8_research.S8ArchiveTests -v`
Expected: missing archive gate.

- [ ] **Step 3: Implement parsing without writing Markdown**

Reuse the existing safe frontmatter parser. Extract bullets under the heading containing `证伪`; stop at the next heading of equal or higher level. Return normalized plain text only.

- [ ] **Step 4: Write queue transition and lifecycle tests**

Assert cold/unknown/scorching → warm/hot without archive creates one idempotent queue row and one industry notice; first snapshot creates no false transition; 20 consecutive cold days sets `dormant`; quarter boundary sets `review_due`; a committed archive resolves the queue.

- [ ] **Step 5: Write and implement skeleton tests**

The skeleton must contain member code/name, market cap, amount, 20/60/120-day return, and profit-margin field with an explicit `missing` marker when unavailable. Persist only `skeleton_json` in `archive_queue`.

Seed the exact ten predeclared core industries idempotently. Resolve the existing semiconductor/storage archive; leave the other nine as `pending` or `draft_ready`. Never classify an AI draft as an approved archive.

- [ ] **Step 6: Verify GREEN and commit**

Run: `.venv/bin/python -m unittest test_s8_research.S8ArchiveTests -v`
Expected: all archive, queue, and skeleton tests pass.

Commit: `feat(s8): add archive gate and build queue`.

### Task 4: Generate deterministic S8 cards under 3/2/20 constraints

**Files:**
- Create: `s8_research/engine.py`
- Modify: `test_s8_research.py`

**Interfaces:**
- Produces: `run_signals(s8_db, target_date, inputs, archive_root) -> RunResult`
- Produces: `format_stock_card(candidate, archive_gate) -> dict`
- Produces: `format_industry_notice(queue_event) -> dict`

- [ ] **Step 1: Write failing capacity and dedup tests**

Build candidates across three industries and assert blood candidates sort before cooling candidates, then market cap descending and code ascending. Assert maximum three stock cards, maximum two per industry, 30-natural-day dedup, and open-ledger capacity 20.

- [ ] **Step 2: Verify RED**

Run: `.venv/bin/python -m unittest test_s8_research.S8EngineTests -v`
Expected: missing `run_signals`.

- [ ] **Step 3: Implement atomic signal generation**

Build all decisions in memory. In one `BEGIN IMMEDIATE` transaction insert stock cards, queued stock cards, and industry notices. Use deterministic text IDs and `INSERT ... ON CONFLICT DO NOTHING`.

- [ ] **Step 4: Lock the card contract**

Assert every stock card contains name/code, cap tier, archive link, window, four lights, numeric reason, falsification condition, review date, data date, and exact disclaimer `研究信号，供学习，非交易建议`.

- [ ] **Step 5: Verify GREEN and commit**

Run: `.venv/bin/python -m unittest test_s8_research.S8EngineTests -v`
Expected: all ordering, capacity, idempotency, and card tests pass.

Commit: `feat(s8): generate bounded research cards`.

### Task 5: Implement next-open shadow entries and three exits

**Files:**
- Create: `s8_research/shadow.py`
- Create: `s8_research/g2.py`
- Create: `scripts/s8_g2_report.py`
- Modify: `test_s8_research.py`

**Interfaces:**
- Produces: `advance_shadow_ledger(s8_db, market_db, target_date, account_size=100_000) -> ShadowResult`
- Produces: `manual_falsify(s8_db, market_db, ledger_id, target_date, reason) -> dict`
- Produces: `build_review_card(signal, ledger, benchmark) -> dict`
- Produces: `g2_summary(s8_db, market_db, as_of_date) -> dict`

- [ ] **Step 1: Write failing entry-size tests**

Assert risk distance equals next open minus signal-date MA250; budget equals 1,000 yuan; shares round down to 100; virtual amount obeys single-stock 30%, industry 30%, and total 60%. Assert open ≤ MA250, missing MA250, missing next-day open, and fewer than 100 affordable shares produce `no_entry`.

- [ ] **Step 2: Verify RED**

Run: `.venv/bin/python -m unittest test_s8_research.S8ShadowEntryTests -v`
Expected: missing shadow module.

- [ ] **Step 3: Implement entry advancement**

Advance only signals whose planned next trade date equals the target. Do not roll a missing open forward. Insert or update ledger and signal status in one S8 transaction.

- [ ] **Step 4: Write three-exit tests**

Cover the first trading close on/after 90 natural days, the tenth consecutive below-MA250 close, a recovery on day nine, and manual falsification with mandatory reason/valid close. When two exits coincide, assert manual falsification, trend falsification, then 90-day review precedence.

- [ ] **Step 5: Implement review-card evidence**

Compute signal-to-exit return, maximum favorable/adverse excursion, CSI 300 relative return, original reason/condition, and mechanical conclusion. Store JSON in both ledger and signal rows.

- [ ] **Step 6: Verify GREEN and commit**

Run: `.venv/bin/python -m unittest test_s8_research.S8ShadowEntryTests test_s8_research.S8ShadowExitTests -v`
Expected: all ledger tests pass.

Add RED tests for “one quarter since first stock signal” and “30 stock signals” readiness, then implement the frozen 20/60/90-day, CSI 300, two-window, falsification-hit, and missing-archive aggregates. `scripts/s8_g2_report.py` opens both databases read-only and refuses to write a report while `g2_ready=false`.

Commit: `feat(s8): close the shadow learning loop`.

### Task 6: Add isolated post-close hook and read-only APIs

**Files:**
- Create: `s8_research/hook.py`
- Create: `s8_research/read_model.py`
- Modify: `s3_research/hook.py`
- Modify: `service.py`
- Modify: `test_s8_research.py`
- Modify: `test_guanlan_frontend_contract.py`

**Interfaces:**
- Produces: `try_start_s8(trade_date) -> bool`
- Produces: `today_payload(s8_db) -> dict`
- Produces: `observation_payload(s8_db) -> dict`
- Produces: `queue_payload(s8_db) -> dict`
- Produces: `review_payload(s8_db) -> dict`
- Produces: `g2_payload(s8_db, market_db) -> dict`

- [ ] **Step 1: Write failing hook-order tests**

Assert the thermometer snapshot publication finishes before `try_start_s8()`. Inject failure in S8 sources, ledger, signals, and push; assert the main post-close hook returns normally and the previous S8 rows remain intact.

- [ ] **Step 2: Implement isolated worker**

Use a non-blocking process lock and one daemon thread. Order: shadow advancement → archive lifecycle → signal generation → push committed cards. Push failure records pending status but does not duplicate rows.

- [ ] **Step 3: Write API contract tests**

Add GET contracts for `/api/s8/today`, `/api/s8/observations`, `/api/s8/archive-queue`, `/api/s8/reviews`; POST contracts for skeleton, draft, and falsify. Assert every POST writes only the S8 file.

Add `GET /api/s8/g2` and assert it returns the frozen aggregate, `g2_ready`, and the first trigger reason without writing either database.

- [ ] **Step 4: Add real-order source guards**

Inspect every `s8_research/*.py` file and assert it contains none of: `add_position`, `close_position`, `pool_transition`, `api_brief_act`, `/api/brief/act`, `orders`, `broker`.

- [ ] **Step 5: Verify GREEN and commit**

Run: `.venv/bin/python -m unittest test_s8_research.S8HookTests test_s8_research.S8ApiTests -v`
Expected: order, failure isolation, APIs, and source guards pass.

Commit: `feat(s8): expose isolated post-close research service`.

### Task 7: Replace the first-screen S2 review area with S8

**Files:**
- Modify: `static/home.html`
- Modify: `test_guanlan_frontend_contract.py`
- Modify: `test_s8_research.py`

**Interfaces:**
- Consumes: the four S8 GET payloads from Task 6.
- Produces: today S8 card section, pool source badges, archive queue panel.

- [ ] **Step 1: Write failing frontend contract tests**

Assert `S8 关注` occupies the first-screen review slot; stock cards render every §4 field; industry notices and review cards have distinct classes; the disclaimer is always visible; the S2 review heading/actions are absent.

- [ ] **Step 2: Add pool and archive tests**

Assert S8 rows show `S8观察`, legacy rows show `S2遗产`, and the archive page header renders queue status, skeleton button, and true data date.

- [ ] **Step 3: Verify RED**

Run: `.venv/bin/python -m unittest test_guanlan_frontend_contract test_s8_research.S8FrontendTests -v`
Expected: S8 DOM tokens absent.

- [ ] **Step 4: Implement the minimal Claude-style UI**

Reuse existing design tokens, pills, card spacing, mobile single-column breakpoint, and hash navigation. Add no new client write except the three approved research POST actions.

- [ ] **Step 5: Verify DOM and syntax**

Run the focused tests and extract all inline `<script>` blocks with `r'<script(?:\\s[^>]*)?>(.*?)</script>'`; run `node --check` on each.

- [ ] **Step 6: Commit**

Commit: `feat(s8): take over today and pool research views`.

### Task 8: Retire S2 with an exact nine-row guarded migration

**Files:**
- Create: `scripts/retire_s2.py`
- Modify: `service.py`
- Modify: `eod.py`
- Modify: `brief.py`
- Modify: `test_s8_research.py`

**Interfaces:**
- Produces: `retire_s2(db_path, expected_ids, expected_sha256) -> dict`
- Produces: `expire_s2_legacy(trade_date) -> int`
- Produces: `S2_SIGNAL_ENABLED = False` deployment constant outside frozen parameter registry.

- [ ] **Step 1: Write guarded migration RED tests**

Build exactly nine active rows. Assert the migration changes only `strategy` and appends one note marker. Assert a count mismatch, ID mismatch, original SHA mismatch, or any unexpected field rolls back all rows.

- [ ] **Step 2: Implement the migration**

Require explicit database path, expected full SHA-256, and the nine IDs. Start `BEGIN IMMEDIATE`; re-read every guarded field; update; verify changed row count nine; commit. Print before/after hash and exact IDs.

- [ ] **Step 3: Write S2 shutdown tests**

Assert the scheduled production job does not call `eod.run_scan_blocking()` when `S2_SIGNAL_ENABLED` is false; no new S2 snapshot, review card, review action, or candidate row is produced. Assert market/thermometer/S8 hooks still run.

- [ ] **Step 4: Implement narrow legacy expiry**

Only `strategy='s2_legacy'` rows can move to `removed`, and only when `expire_at < trade_date`. Clear no price or history fields. Record the transition note; never run thesis/review generation.

- [ ] **Step 5: Preserve-code evidence**

Record SHA-256 of `rules.py`, `rules_mid.py`, `strategies/`, and `params.py` before/after. They must remain unchanged. Run existing risk/account tests to prove the 1%/30%/60%/8% skeleton still passes.

- [ ] **Step 6: Verify GREEN and commit**

Run: `.venv/bin/python -m unittest test_s8_research.S2RetirementTests -v`
Expected: migration, shutdown, expiry, and preservation tests pass.

Commit: `feat: retire s2 signals and preserve legacy pool`.

### Task 9: Run full verification and create Phase B evidence

**Files:**
- Create: `docs/验收-S8投资线研究信号与S2退役-20260802.md`
- Create: `reports/s8-verification-20260802.json`

- [ ] **Step 1: Run focused tests**

Run: `.venv/bin/python -m unittest test_s8_research test_industry_archives test_guanlan_frontend_contract test_production_equivalent -v`.

- [ ] **Step 2: Run required regressions in an isolated test worktree**

Run separately: `test_rules.py`, `test_rules_mid.py`, `test_web.py`, `test_regime.py`, and all discoverable unit tests. Never run `test_web.py` from the production main worktree.

- [ ] **Step 3: Run structural gates**

Run `git diff --check`, JSON parse with duplicate-key rejection, JavaScript syntax checks, schema exact-table check, S8 forbidden-token scan, and before/after S2 code hashes.

- [ ] **Step 4: Write report evidence**

Record test commands/counts, source dates, formula versions, three-table schema, TDD red/green evidence, Phase A dependency hashes, and explicit zero-trading proof.

- [ ] **Step 5: Commit**

Commit: `test(s8): record full research-system verification`.

### Task 10: Pass Phase B production-equivalent acceptance

**Files:**
- Create: `reports/s8-equivalent-20260802.json`
- Create: `docs/screenshots/s8-equivalent/20260802/S8关注-等效-桌面.jpg`
- Create: `docs/screenshots/s8-equivalent/20260802/S8关注-等效-手机.jpg`
- Create: `docs/screenshots/s8-equivalent/20260802/S8观察-等效-桌面.jpg`
- Create: `docs/screenshots/s8-equivalent/20260802/S8观察-等效-手机.jpg`
- Create: `docs/screenshots/s8-equivalent/20260802/建档队列-等效-桌面.jpg`
- Create: `docs/screenshots/s8-equivalent/20260802/建档队列-等效-手机.jpg`

- [ ] **Step 1: Create read-only consistent copies**

Clone the production LaunchAgent plist without editing the original. Use the same `start.sh`, working-directory semantics, static serving, and a unique port. Snapshot `screener.db`, `market.db`, the thermometer sidecar, and the S8 sidecar; chmod input copies read-only.

- [ ] **Step 2: Hash every boundary before start**

Record original plist, protected DBs, frozen T1, both sidecars, and all equivalent copies. Run the S2 migration only against a disposable copy.

- [ ] **Step 3: Exercise the two-stage chain**

Inject one successful target day and failures at each thermometer/S8 stage. Prove failed days keep the previous thermometer snapshot and S8 signal set.

- [ ] **Step 4: Capture three pages in two viewports**

Use desktop `1440×900` and mobile `390×844`. Require no horizontal overflow, no console error, true data date, S8 disclaimer, S8/S2 source badges, and visible queue status.

- [ ] **Step 5: Verify writes and hashes**

The protected input copies and production originals must retain their hashes. The equivalent S8 DB must contain exactly three tables. The equivalent screener copy may contain only the guarded nine-row S2 migration diff.

- [ ] **Step 6: Commit evidence**

Commit: `test(s8): accept production-equivalent takeover`.

### Task 11: Merge, install both sidecars, migrate S2, and deploy once

**Files:**
- Modify outside Git under authorization: production LaunchAgent clone/original environment values only after preflight.
- Create: `reports/s8-production-deployment-20260802.json`
- Create: `docs/screenshots/s8-production/20260802/*.jpg`

- [ ] **Step 1: Merge the feature branch into main**

Use `--no-ff`. Preserve `docs/AI评分校准-202608.md`. Run merged-result tests in a detached temporary worktree.

- [ ] **Step 2: Capture rollback evidence**

Record previous main commit, production plist bytes/hash, `screener.db`, `market.db`, `S3_research.db`, frozen T1, sidecar candidates, current health fingerprint, current process PID, and exact nine S2 IDs.

- [ ] **Step 3: Install Phase A and Phase B artifacts**

Atomically install a new thermometer sidecar and a new initialized `S8_research.db`. Update production environment paths. Do not overwrite an existing sidecar without an explicit versioned backup.

- [ ] **Step 4: Run the exact S2 migration**

Call `scripts/retire_s2.py` with preflight hash and IDs. Abort before restart if any guard fails.

- [ ] **Step 5: Restart once and verify health**

Run `./restart.sh`. Require `ok=true`, `match=true`, one process on port 8787, current thermometer date, S2 disabled state, and readable S8 endpoints.

- [ ] **Step 6: Capture six production screenshots**

Capture today, stock pool, and industry archive at desktop/mobile sizes. Verify exact card fields, source badges, queue, no S2 review controls, no overflow, and no console errors.

- [ ] **Step 7: Rehash and decide**

Protected databases and frozen T1 must match preflight. `screener.db` may differ only by the guarded nine rows. If any assertion fails, restore previous code/config/database backup, restart, and report the rollback.

- [ ] **Step 8: Commit final production report**

Commit: `docs(s8): record production takeover and hashes`.

### Task 12: Push one mirror update and verify a fresh clone

**Files:**
- Mirror allowlist: final `docs/`, `reports/`, `AGENTS.md`, and screenshots.

- [ ] **Step 1: Run final local gates**

Confirm main is clean except the preserved user file, production health remains matched, reports parse, screenshot hashes match, and no database/token/plist body is staged.

- [ ] **Step 2: Push through HTTPS**

Run `.venv/bin/python scripts/report_mirror_sync.py --push` once after both phase reports and all production evidence are committed.

- [ ] **Step 3: Fresh-clone verification**

Clone `https://github.com/harryd317/guanlan-reports.git` into a new `mktemp -d` directory. Verify remote HEAD, both phase reports, S8/S2 design and plan, all acceptance screenshots, JSON semantics, frozen T1 hash, production health, and the exact mirror file hashes.

- [ ] **Step 4: Finish**

Report main merge commit, final report commit, production health/fingerprint, protected before/after hashes, thermometer sidecar hash, S8 sidecar hash, S2 migration diff, screenshot paths, and verified mirror commit hash.
