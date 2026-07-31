# T4 Position Books Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add immutable speculative/investment position labels, separate book statistics, and fail-soft daily handling for system-external buys.

**Architecture:** Add only independent tables to the existing SQLite file and isolate every read/write behind `portfolio_books.py`. Existing strategy, risk, backtest, and market-data code never reads the new tables. New buys still enter the existing `positions` ledger so account-wide exposure remains unchanged; investment and system-external positions use the existing `strategy='manual'` boundary so S2 cannot manage them.

**Tech Stack:** Python 3, SQLite, FastAPI/Pydantic, existing vanilla HTML/CSS/JavaScript, `unittest`, existing APScheduler and push channels.

## Global Constraints

- Work only on `codex/t4-position-books`; do not merge, deploy, restart production, or start T5/T6.
- Add independent SQLite tables only. Do not alter any existing table, field, or index.
- Do not modify `rules_mid.py`, `strategies/`, `screener.py`, `backtest*.py`, `market_data.py`, `data.py`, `eod.py`, or `static/pool.html`.
- S2, risk, and backtest core must not read T4 tables.
- Preserve account-wide position, industry, and circuit-breaker behavior.
- Use no new dependency, no production demo data, and no retroactive label migration.
- Treat the approved start confirmation as the source of truth.

---

### Task 1: Immutable Book Ledger and Violation State

**Files:**
- Create: `portfolio_books.py`
- Create: `test_portfolio_books.py`

**Interfaces:**
- Produces: `init_db() -> None`
- Produces: `create_position(book_type, origin, code, name, buy_date, buy_price, stop_price, amount, rule_version, signal_type=None, industry=None, qty=None, external=False, exit_due_date=None, mistake_condition=None, buy_basis=None, recorded_on=None) -> int`
- Produces: `complete_discipline(position_id, mistake_condition, stop_price, buy_basis, completed_on) -> dict`
- Produces: `overview(account_size) -> dict`
- Produces: `legacy_manual_account() -> dict`
- Produces: `pending_violations(on_date=None) -> list[dict]`
- Produces: `mark_reminded(position_ids, on_date) -> None`
- Produces: `delete_record(position_id) -> None`

- [ ] **Step 1: Write failing tests for schema isolation and immutable labels**

```python
class PortfolioBooksTest(unittest.TestCase):
    def test_init_adds_only_independent_tables_and_label_cannot_change(self):
        before = existing_schema(self.db)
        pid = portfolio_books.create_position(
            "speculative", "system", "600001", "测试", "2026-07-30",
            10, 9, 10000, 6, recorded_on="2026-07-30")
        self.assertEqual(existing_schema(self.db), before)
        with self.assertRaises(sqlite3.IntegrityError):
            sql("UPDATE position_books SET book_type='investment' WHERE position_id=?", pid)
```

- [ ] **Step 2: Run the test and verify RED**

Run: `~/云慧养/zmu_gitee/screener/.venv/bin/python -m unittest -v test_portfolio_books.PortfolioBooksTest.test_init_adds_only_independent_tables_and_label_cannot_change`

Expected: import failure because `portfolio_books.py` does not exist.

- [ ] **Step 3: Implement the independent tables and atomic position+label insert**

Create `position_books`, `outside_buy_controls`, and the immutable-label trigger through `storage._conn()`. Insert the existing `positions` row and T4 rows in one SQLite transaction. Accept only `speculative` and `investment`. Force `strategy='manual'` for `investment` or `external=True`.

- [ ] **Step 4: Write failing tests for same-day completion and next-day-exit state**

```python
def test_external_buy_stays_violation_until_same_day_three_fields_complete(self):
    pid = portfolio_books.create_position(
        "investment", "outside", "600002", "外部票", "2026-07-30",
        20, None, 20000, 6, external=True, exit_due_date="2026-07-31",
        recorded_on="2026-07-30")
    self.assertEqual(portfolio_books.pending_violations("2026-07-30")[0]["position_id"], pid)
    with self.assertRaises(ValueError):
        portfolio_books.complete_discipline(pid, "业绩失速", 18, "", "2026-07-30")
    with self.assertRaises(ValueError):
        portfolio_books.complete_discipline(pid, "业绩失速", 18, "估值回落", "2026-07-31")
    got = portfolio_books.complete_discipline(
        pid, "业绩失速", 18, "估值回落", "2026-07-30")
    self.assertEqual(got["status"], "disciplined")
    self.assertEqual(portfolio_books.pending_violations("2026-07-30"), [])
```

- [ ] **Step 5: Run RED, implement validation, and rerun GREEN**

Run: `~/云慧养/zmu_gitee/screener/.venv/bin/python -m unittest -v test_portfolio_books`

Expected after implementation: all ledger and violation tests pass.

- [ ] **Step 6: Add literal-fixture tests for separate statistics and legacy filtering**

Use one tagged speculative win, one tagged investment loss, and one untagged legacy manual trade. Assert each book's position count, realized P&L, win rate, and realized-equity drawdown from hand-calculated literals. Assert the legacy row appears only under `legacy_manual_account()`.

- [ ] **Step 7: Commit the model**

```bash
git add portfolio_books.py test_portfolio_books.py
git commit -m "feat(t4): add immutable position book ledger"
```

---

### Task 2: Enforce Labels Across Service Buy Paths

**Files:**
- Modify: `service.py`
- Modify: `test_web.py`
- Test: `test_portfolio_books.py`

**Interfaces:**
- `PositionBody.book_type`, `BriefActBody.book_type`, `TransitionBody.book_type`, and `OutsideActBody.book_type` accept `speculative|investment`
- `POST /api/portfolio/{position_id}/discipline` accepts `mistake_condition`, `stop_price`, and `buy_basis`
- `GET /api/trade/overview` adds `books` and `violations` without removing existing account-wide keys

- [ ] **Step 1: Write failing route tests**

```python
def test_new_buy_paths_reject_missing_or_unknown_book(self):
    self.assertFalse(service.api_position_add(position_body(book_type=None))["ok"])
    self.assertFalse(service.api_position_add(position_body(book_type="value"))["ok"])

def test_investment_is_manual_to_s2_but_still_counts_in_total_exposure(self):
    result = service.api_position_add(position_body(book_type="investment"))
    pos = storage.get_position(result["id"])
    self.assertEqual(pos["strategy"], "manual")
    self.assertEqual(storage.industry_exposure(pos["industry"]), pos["amount"])
```

- [ ] **Step 2: Run the focused tests and verify RED**

Run: `~/云慧养/zmu_gitee/screener/.venv/bin/python -m unittest -v test_portfolio_books test_web`

Expected: the new fields and routes are absent.

- [ ] **Step 3: Initialize T4 storage and enforce the service contract**

Call `portfolio_books.init_db()` after `storage.init_db()`. Validate the label in every post-T4 buy path:

- system card buy: speculative remains in the existing S2 path; investment becomes a manual ledger position and leaves the trading pool
- direct trade-desk buy: record as system-external and create next-trading-day disposition
- outside-ledger buy: record as system-external
- OCR buy: reaches one of the two routes above with an explicit label
- legacy CSV/JSON import: remains untagged legacy migration and does not trigger T4
- old `/pool` holding transition without a label: reject with a link to Today/Trade; do not edit `static/pool.html`

- [ ] **Step 4: Add discipline completion and close/delete lifecycle**

Add `POST /api/portfolio/{position_id}/discipline`. Use `position_id` when closing a tagged manual position. Keep the immutable label after a real close; delete the T4 record only when the underlying open position is deleted as an entry error.

- [ ] **Step 5: Add overview joins without changing account-wide risk keys**

Return:

```json
{
  "books": {
    "speculative": {"positions_n": 0, "position_amount": 0, "realized_pnl": 0, "win_rate": null, "drawdown_pct": null},
    "investment": {"positions_n": 0, "position_amount": 0, "realized_pnl": 0, "win_rate": null, "drawdown_pct": null},
    "positions": [],
    "trades": [],
    "legacy_unclassified_n": 0
  },
  "violations": []
}
```

Keep existing `drawdown`, `account_size`, `pos_max_pct`, `total_max_pct`, and all risk decisions unchanged.

- [ ] **Step 6: Run focused service tests GREEN**

Run: `~/云慧养/zmu_gitee/screener/.venv/bin/python -m unittest -v test_portfolio_books`

Run: `S3_RESEARCH_DB=/private/tmp/t4-service-tests.db ~/云慧养/zmu_gitee/screener/.venv/bin/python -m unittest -q test_web`

- [ ] **Step 7: Commit service integration**

```bash
git add service.py test_web.py test_portfolio_books.py
git commit -m "feat(t4): enforce position books on new buys"
```

---

### Task 3: Daily Violation Reminder

**Files:**
- Modify: `push.py`
- Modify: `service.py`
- Modify: `test_portfolio_books.py`

**Interfaces:**
- Produces: `push.notify_t4_violations(on_date=None) -> bool`
- Consumes: `portfolio_books.pending_violations()` and `portfolio_books.mark_reminded()`

- [ ] **Step 1: Write failing reminder tests**

```python
def test_daily_violation_push_deduplicates_only_after_success(self):
    sent = []
    push.send_push = lambda title, body: sent.append((title, body)) or {"ok": True}
    self.assertTrue(push.notify_t4_violations("2026-07-31"))
    self.assertFalse(push.notify_t4_violations("2026-07-31"))
    self.assertEqual(len(sent), 1)
    self.assertIn("下一交易日平仓", sent[0][1])

def test_push_failure_does_not_raise_or_mark_reminded(self):
    push.send_push = lambda title, body: (_ for _ in ()).throw(RuntimeError("offline"))
    self.assertFalse(push.notify_t4_violations("2026-07-31"))
    self.assertEqual(len(portfolio_books.pending_violations("2026-07-31")), 1)
```

- [ ] **Step 2: Run RED, implement fail-soft reminder, run GREEN**

Call the reminder from the existing 08:00 orchestration helper without changing any strategy/EOD job. Include code, name, book, buy date, exit deadline, and missing fields. Mark rows only after a successful send.

Run: `~/云慧养/zmu_gitee/screener/.venv/bin/python -m unittest -v test_portfolio_books`

- [ ] **Step 3: Commit reminder integration**

```bash
git add push.py service.py test_portfolio_books.py
git commit -m "feat(t4): remind unresolved outside buys daily"
```

---

### Task 4: Desktop and Mobile UI

**Files:**
- Modify: `static/home.html`
- Modify: `static/trade.html`
- Modify: `test_m3_home.py`
- Modify: `test_web.py`

**Interfaces:**
- Home buy request sends `book_type`
- Trade direct/outside/OCR buys send `book_type`
- Trade page renders `books`, `violations`, and legacy-unclassified rows from the overview API

- [ ] **Step 1: Write failing behavior-contract tests**

Add tests that submit real FastAPI model payloads without/with the label, then assert response behavior. Add static-page tests only for required controls and endpoint names; do not assert cosmetic source text.

- [ ] **Step 2: Run UI contract tests RED**

Run: `~/云慧养/zmu_gitee/screener/.venv/bin/python -m unittest -v test_m3_home`

Run: `S3_RESEARCH_DB=/private/tmp/t4-ui-tests.db ~/云慧养/zmu_gitee/screener/.venv/bin/python -m unittest -q test_web`

- [ ] **Step 3: Add the Home selector**

For buy and gate-violation buy cards, render an unselected warehouse selector. Block confirmation until the user chooses. Show the investment warning beside the selector and send `book_type`.

- [ ] **Step 4: Add the Trade page controls and reports**

Add:

- required label selector on direct and OCR buys
- immutable book badge on every tagged position
- separate speculative/investment statistic cards
- “投资线规则未上线，仅记账” warning
- red violation panel with exit deadline and three-field completion form
- explicit “旧账未归类” section for pre-T4 rows

Do not render any label-edit control.

- [ ] **Step 5: Run UI tests GREEN and commit**

```bash
git add static/home.html static/trade.html test_m3_home.py test_web.py
git commit -m "feat(t4): add two-book trade interface"
```

---

### Task 5: Acceptance, Full Regression, and Screenshots

**Files:**
- Create: `docs/验收-T4-投机投资分仓-20260730.md`
- Create: `docs/证据-T4-投机投资分仓-桌面-20260730.png`
- Create: `docs/证据-T4-投机投资分仓-手机-20260730.png`

- [ ] **Step 1: Run T4 tests and compile checks**

```bash
~/云慧养/zmu_gitee/screener/.venv/bin/python -m unittest -v test_portfolio_books
~/云慧养/zmu_gitee/screener/.venv/bin/python -m compileall -q portfolio_books.py service.py push.py
git diff --check
```

- [ ] **Step 2: Run every frozen baseline**

```bash
~/云慧养/zmu_gitee/screener/.venv/bin/python test_rules.py
~/云慧养/zmu_gitee/screener/.venv/bin/python -m unittest -q test_rules_mid
~/云慧养/zmu_gitee/screener/.venv/bin/python -m unittest -q test_regime test_m3_home test_report_mirror_sync
~/云慧养/zmu_gitee/screener/.venv/bin/python -m unittest -q test_s3_research test_s3_ui
S3_RESEARCH_DB=/private/tmp/t4-full-web-tests.db ~/云慧养/zmu_gitee/screener/.venv/bin/python -m unittest -q test_branding test_web
```

- [ ] **Step 3: Audit forbidden files and existing schema**

Compare `main...HEAD`. The only permitted production changes are `portfolio_books.py`, `service.py`, `push.py`, `static/home.html`, and `static/trade.html`. Prove `static/pool.html` and all strategy/risk/backtest/data-pipeline files have zero diff. Compare every pre-existing `sqlite_master` table definition and index before/after T4 initialization.

- [ ] **Step 4: Capture desktop and mobile screenshots**

Run the branch against a disposable copy of the local database, create representative rows only through T4 public APIs, and capture `/trade` at 1440×1000 and 390×844. The screenshots must show both book cards, immutable labels, investment warning, and one unresolved violation. Never write the production database.

- [ ] **Step 5: Write and commit the acceptance report**

Record every checklist item, exact commands, exit codes, test counts, screenshot paths, branch commit, and “not merged/not deployed/not production” statement.

```bash
git add docs/验收-T4-投机投资分仓-20260730.md docs/证据-T4-*.png
git commit -m "docs(t4): deliver acceptance evidence"
```

- [ ] **Step 6: Stop for user screenshot acceptance**

Do not merge `main`, deploy, restart production, or begin T5/T6.
