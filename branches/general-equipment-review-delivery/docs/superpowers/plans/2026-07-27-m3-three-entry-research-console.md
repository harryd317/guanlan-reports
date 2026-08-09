# M3 Three-Entry Research Console Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Turn the existing Screener home page into a Material 3 single-page console with `#today`, `#pool`, `#me`, and a read-only stock detail side sheet while preserving every existing trading guard and legacy page.

**Architecture:** Keep `static/home.html` as the server-rendered entry and retain its existing action-card functions. Add a small inline `globalThis.M3Model` pure model for routing/grouping/sorting, an M3 shell around the existing today renderer, and read-only renderers for pool, account, and detail data. All requests use existing endpoints; no service or data-layer code changes.

**Tech Stack:** Static HTML, prefixed CSS, vanilla JavaScript, Python standard-library tests, Node.js only as a zero-dependency JavaScript execution host.

## Global Constraints

- Pure frontend refactor; no strategy, risk, backtest, data-pipeline, scheduler, notification, database, or parameter changes.
- Modify `static/home.html` only for production behavior.
- Do not modify `static/pool.html`, `static/nav.js`, `service.py`, dependency manifests, or legacy pages.
- New M3 read-only views may add calls only to `/api/pool`, `/api/trade/overview`, `/api/regime/current`, `/api/pool/{code}`, and `/api/stock_test/{code}`; preserve the existing home page’s `/api/brief`, intraday, action, OCR, rescan, and reaffirm calls unchanged.
- `#pool` is read-only; it must not contain `/transition`, rescan, or write controls.
- Missing ATR, sector-cluster, RS, seven-factor, or replay fields are omitted; never calculate substitutes.
- No hard-coded stock rows or demo payloads.
- No new npm or pip dependencies.
- Breakpoint is exactly `860px`; side sheet width is exactly `470px`.
- Existing server-side trade and circuit-breaker guards remain the only authority.
- Run `.venv/bin/python test_rules.py && .venv/bin/python test_rules_mid.py && .venv/bin/python test_web.py && .venv/bin/python test_regime.py` before completion.

---

### Task 1: Executable M3 Model and Hash Router

**Files:**
- Create: `test_m3_home.py`
- Modify: `static/home.html:1-390`
- Test: `test_m3_home.py`

**Interfaces:**
- Produces: `M3Model.routeOf(hash) -> "today"|"pool"|"me"`.
- Produces: `M3Model.groupPool(rows) -> [{key,label,rows}]`.
- Produces: `M3Model.sortWaiting(rows, quoteMap) -> row[]`.
- Produces: `applyM3Route()`, which updates visible views and `aria-current`.

- [ ] **Step 1: Write the failing model tests**

Create `test_m3_home.py` with a helper that extracts `<script id="m3-model">`, executes it with Node, and asserts literal results:

```python
from pathlib import Path
import json, re, subprocess, unittest

ROOT = Path(__file__).resolve().parent
HOME = (ROOT / "static" / "home.html").read_text(encoding="utf-8")

def run_model(expression):
    match = re.search(r'<script id="m3-model">([\s\S]*?)</script>', HOME)
    if not match:
        raise AssertionError("m3-model script missing")
    source = match.group(1)
    program = source + "\nprocess.stdout.write(JSON.stringify(" + expression + "));"
    result = subprocess.run(["node", "-e", program], text=True,
                            capture_output=True, check=True)
    return json.loads(result.stdout)

class M3ModelTests(unittest.TestCase):
    def test_hash_router_defaults_invalid_routes_to_today(self):
        self.assertEqual(
            run_model('["", "#today", "#pool", "#me", "#bad"].map(M3Model.routeOf)'),
            ["today", "today", "pool", "me", "today"])

    def test_pool_grouping_preserves_all_rows(self):
        rows = [
            {"id": 1, "state": "holding"},
            {"id": 2, "state": "watchlist"},
            {"id": 3, "state": "candidate"},
            {"id": 4, "state": "research"},
            {"id": 5, "state": "cooldown"},
        ]
        expr = "M3Model.groupPool(" + json.dumps(rows) + \
               ").map(g=>({key:g.key,ids:g.rows.map(r=>r.id)}))"
        self.assertEqual(run_model(expr), [
            {"key": "watchlist", "ids": [2]},
            {"key": "candidate", "ids": [3]},
            {"key": "research", "ids": [4]},
            {"key": "holding", "ids": [1]},
            {"key": "cooldown", "ids": [5]},
        ])

    def test_waiting_sort_uses_existing_distance_and_puts_missing_last(self):
        rows = [{"code": "A"}, {"code": "B"}, {"code": "C"}]
        quotes = {"A": {"dist_pct": 4.2}, "B": {"dist_pct": 1.1}}
        expr = "M3Model.sortWaiting(" + json.dumps(rows) + "," + \
               json.dumps(quotes) + ").map(r=>r.code)"
        self.assertEqual(run_model(expr), ["B", "A", "C"])
```

- [ ] **Step 2: Run the test and verify RED**

Run: `.venv/bin/python test_m3_home.py`

Expected: FAIL with `m3-model script missing`.

- [ ] **Step 3: Add the pure model and M3 view shell**

In `static/home.html`, add the exact three-route model before the main script:

```javascript
globalThis.M3Model = (() => {
  const routes = new Set(["today", "pool", "me"]);
  const groups = [
    ["watchlist", "等买点"], ["candidate", "候选"],
    ["research", "研究中"], ["holding", "持有中"], ["cooldown", "冷静期"],
  ];
  function routeOf(hash) {
    const key = String(hash || "").replace(/^#/, "");
    return routes.has(key) ? key : "today";
  }
  function groupPool(rows) {
    return groups.map(([key, label]) => ({
      key, label, rows: (rows || []).filter(row => row.state === key),
    }));
  }
  function sortWaiting(rows, quoteMap) {
    return [...(rows || [])].sort((a, b) => {
      const av = Number((quoteMap[a.code] || {}).dist_pct);
      const bv = Number((quoteMap[b.code] || {}).dist_pct);
      const aa = Number.isFinite(av), bb = Number.isFinite(bv);
      return aa && bb ? av - bv : aa ? -1 : bb ? 1 : 0;
    });
  }
  return {routeOf, groupPool, sortWaiting};
})();
```

Add the desktop rail, bottom navigation, and three `<section>` elements. `applyM3Route()` must set one active section, set `aria-current="page"` on the matching navigation items, preserve browser history through normal hash links, and normalize an empty/invalid hash with `history.replaceState(null, "", "#today")`.

- [ ] **Step 4: Run the model test and verify GREEN**

Run: `.venv/bin/python test_m3_home.py`

Expected: 3 tests PASS.

- [ ] **Step 5: Commit**

```bash
git add test_m3_home.py static/home.html
git commit -m "前端: 建立M3三入口与hash路由"
```

---

### Task 2: Read-Only Pool View

**Files:**
- Modify: `test_m3_home.py`
- Modify: `static/home.html:334-920`
- Test: `test_m3_home.py`

**Interfaces:**
- Consumes: `M3Model.groupPool(rows)`.
- Produces: `POOL_DATA`, `loadM3Pool()`, `renderM3Pool(payload)`.
- Produces: stock-row click contract `openM3Sheet(code)`.

- [ ] **Step 1: Add failing pool-view tests**

Add tests that parse the real HTML and require:

```python
def test_pool_view_is_read_only_and_uses_existing_list_endpoint(self):
    self.assertIn('id="m3-view-pool"', HOME)
    self.assertIn('fetch("/api/pool")', HOME)
    self.assertIn("M3Model.groupPool", HOME)
    self.assertNotIn('fetch("/api/pool/${id}/transition"', HOME)

def test_pool_view_has_legacy_management_link_without_embedding_actions(self):
    self.assertIn('href="/pool"', HOME)
    self.assertIn("池子管理", HOME)
```

The production change that makes these fail is removing grouping, adding a write call, or losing the legacy management link.

- [ ] **Step 2: Run and verify RED**

Run: `.venv/bin/python test_m3_home.py`

Expected: FAIL because `m3-view-pool`, `loadM3Pool`, and grouped rendering are absent.

- [ ] **Step 3: Implement read-only pool loading and grouping**

Implement:

```javascript
let POOL_DATA = null;
async function loadM3Pool() {
  const payload = await (await fetch("/api/pool")).json();
  if (!payload.ok) throw new Error(payload.error || "股票池加载失败");
  POOL_DATA = payload;
  renderM3Pool(payload);
  updateM3Counts();
}
```

`renderM3Pool` must:

- render all five groups returned by `M3Model.groupPool(payload.pool)`;
- omit empty group bodies or show an honest empty sentence;
- display name, code, `industry` when present, state, `last_close`, and existing distance fields when present;
- render `long_profile` only as existing reference markers;
- never render transition, rescan, buy, sell, or remove buttons;
- make each row call `openM3Sheet(row.code)`;
- include a visible `href="/pool"` “池子管理” link.

- [ ] **Step 4: Verify GREEN and legacy pool unchanged**

Run:

```bash
.venv/bin/python test_m3_home.py
git diff --exit-code main -- static/pool.html
```

Expected: tests PASS; `git diff` exits 0.

- [ ] **Step 5: Commit**

```bash
git add test_m3_home.py static/home.html
git commit -m "前端: 增加只读股票池视图"
```

---

### Task 3: Read-Only Detail Side Sheet

**Files:**
- Modify: `test_m3_home.py`
- Modify: `static/home.html:334-1000`
- Test: `test_m3_home.py`

**Interfaces:**
- Produces: `openM3Sheet(code)`, `closeM3Sheet()`, `renderM3Sheet(detail, test)`.
- Consumes: `/api/pool/{code}` and `/api/stock_test/{code}` only.

- [ ] **Step 1: Add failing side-sheet tests**

Add:

```python
def test_detail_sheet_has_three_close_paths_and_read_only_endpoints(self):
    self.assertIn('id="m3-sheet-mask"', HOME)
    self.assertIn('id="m3-detail-panel"', HOME)
    self.assertIn('aria-modal="true"', HOME)
    self.assertIn('fetch(`/api/pool/${encodeURIComponent(code)}`)', HOME)
    self.assertIn('fetch(`/api/stock_test/${encodeURIComponent(code)}`)', HOME)
    self.assertIn('event.key === "Escape"', HOME)
    self.assertIn("closeM3Sheet()", HOME)

def test_detail_sheet_geometry_and_optional_sections_are_explicit(self):
    self.assertRegex(HOME, r"\.m3-detail-panel\{[^}]*width:470px")
    self.assertIn("m3ClampPrice", HOME)
    self.assertIn("参考标记 · 不决策", HOME)
```

- [ ] **Step 2: Run and verify RED**

Run: `.venv/bin/python test_m3_home.py`

Expected: FAIL because mask, side sheet, and detail functions do not exist.

- [ ] **Step 3: Implement mask, sheet, and honest optional rendering**

`openM3Sheet(code)` must use `Promise.allSettled` so a stock-test failure does not erase pool details. `renderM3Sheet` must:

- show name, code, state, close, and change only when returned;
- build the price rail from existing `stop_price`, `platform_price`, and `close`;
- map marker percentages through:

```javascript
function m3ClampPrice(value, low, high) {
  if (![value, low, high].every(Number.isFinite) || high <= low) return null;
  return Math.max(4, Math.min(96, ((value - low) / (high - low)) * 100));
}
```

- use range `stop * 0.95` to `buy * 1.15`, without changing any price;
- show current stop wording, never calculate ATR;
- show planned risk amount only when `account_size` exists, using the specified display formula `account_size * 0.01`;
- show RS percentile only when directly present in returned objects;
- show cluster only when directly present in returned objects, with no calculation or fallback;
- show timeline from `detail.events`;
- show only present `long_profile.facts`;
- render `<details>` for existing thesis seven-factor and replay data, default closed;
- omit an entire optional block when its source is absent.

`closeM3Sheet()` must restore focus to the clicked row, remove the open class, and leave `location.hash` unchanged.

- [ ] **Step 4: Verify GREEN**

Run: `.venv/bin/python test_m3_home.py`

Expected: all side-sheet tests PASS.

- [ ] **Step 5: Commit**

```bash
git add test_m3_home.py static/home.html
git commit -m "前端: 增加个股只读详情侧栏"
```

---

### Task 4: Account and Engineering View

**Files:**
- Modify: `test_m3_home.py`
- Modify: `static/home.html:334-1050`
- Test: `test_m3_home.py`

**Interfaces:**
- Produces: `OVERVIEW`, `loadM3Overview()`, `renderM3Me(payload)`, `renderM3Validation(payload)`.
- Consumes: `/api/trade/overview`.

- [ ] **Step 1: Add failing account-view tests**

Add:

```python
def test_me_view_uses_overview_and_links_legacy_tools(self):
    self.assertIn('id="m3-view-me"', HOME)
    self.assertIn('fetch("/api/trade/overview")', HOME)
    for href in ("/pool", "/trade", "/ask", "/strategy", "/settings"):
        self.assertIn(f'href="{href}"', HOME)
    self.assertIn("🔒 策略中心 · 回测 · 影子调度 → 改参数需预声明", HOME)
    self.assertIn("平时不展示回测数字与换将建议", HOME)

def test_validation_copy_is_exact(self):
    self.assertIn("攒满 20 笔前，仓位上限与规则冻结", HOME)
```

- [ ] **Step 2: Run and verify RED**

Run: `.venv/bin/python test_m3_home.py`

Expected: FAIL because the `#me` renderer and exact copy are absent.

- [ ] **Step 3: Implement account, statistics, risk rails, and engineering links**

`renderM3Me` must read, not derive or mutate:

- `account_size`, latest `equity`, and `drawdown`;
- `stats_current.n`, `win_rate`, `profit_factor`, `expectancy_pct`, `verify_done`;
- `pos_max_pct`, `total_max_pct`, `ladder_capacity`, and current circuit-breaker data when present.

Unknown values render `—`; missing cards are omitted. Add an expandable engineering card containing links to `/pool`, `/trade`, `/ask`, `/strategy`, and `/settings`, plus the exact required copy. Use `stats_current.n` for the shared 0/20 validation card on `#today`.

- [ ] **Step 4: Verify GREEN**

Run: `.venv/bin/python test_m3_home.py`

Expected: account and copy tests PASS.

- [ ] **Step 5: Commit**

```bash
git add test_m3_home.py static/home.html
git commit -m "前端: 增加我的与工程模式视图"
```

---

### Task 5: Material 3 Today Layout and Responsive Behavior

**Files:**
- Modify: `test_m3_home.py`
- Modify: `static/home.html:1-1280`
- Test: `test_m3_home.py`
- Test: `test_web.py`

**Interfaces:**
- Consumes: existing `render`, `heroHtml`, `cardHtml`, `holdingsHtml`, `waitingHtml`.
- Produces: M3-prefixed shell styles and full-data today layout.

- [ ] **Step 1: Add failing layout and integrity tests**

Add:

```python
def test_three_routes_responsive_breakpoint_and_side_sheet_width(self):
    for route in ("today", "pool", "me"):
        self.assertIn(f'href="#{route}"', HOME)
    self.assertIn("@media (max-width:860px)", HOME)
    self.assertIn("grid-template-columns:5fr 7fr", HOME)
    self.assertIn("width:470px", HOME)

def test_home_has_no_hard_coded_stock_codes_or_pool_write_endpoint(self):
    self.assertIsNone(re.search(r'(?<!\\d)[036]\\d{5}(?!\\d)', HOME))
    self.assertNotIn("/transition", HOME)
```

- [ ] **Step 2: Run and verify RED**

Run: `.venv/bin/python test_m3_home.py`

Expected: FAIL on missing 860px M3 layout.

- [ ] **Step 3: Add exact M3 tokens and adapt the visible today renderer**

Add the required token block and only `m3-` prefixed new selectors. Make the visible `#today` grid exactly `5fr 7fr`, with:

- left: gate card, full existing action cards, validation card, holdings;
- right: complete waiting list sorted through `M3Model.sortWaiting(waiting, INTRA_MAP)`;
- exact empty copy: `今日无操作 / 门关着，继续等待。没有机会也是一种正确结果。`;
- exact holdings empty copy: `持仓 · 0 / 骑牛留仓机制待命 · 涨幅达 30% 后激活`;
- stale pill in the navigation foot and warning sentence when `d.is_stale`;
- no “查看更多”.

Keep all existing action helpers and sentinel strings so `/api/brief/act`, OCR, reaffirm, circuit breaker, close confirmation, and local audit behavior remain intact.

- [ ] **Step 4: Verify new and existing contracts**

Run:

```bash
.venv/bin/python test_m3_home.py
.venv/bin/python test_web.py
```

Expected: M3 tests PASS and existing Web tests PASS 834/834.

- [ ] **Step 5: Commit**

```bash
git add test_m3_home.py static/home.html
git commit -m "前端: 完成M3今日页与响应式布局"
```

---

### Task 6: Real Browser Acceptance and Scope Audit

**Files:**
- Modify only if a failing acceptance test requires it: `static/home.html`
- Test: `test_m3_home.py`
- Test: project regression scripts

**Interfaces:**
- Produces: verified browser behavior and final diff audit.

- [ ] **Step 1: Start the branch service without touching production**

Run a temporary local service on an unused port with the worktree’s existing environment and fixture links. Do not call `./restart.sh` because no Python file changed and the production service must remain untouched.

- [ ] **Step 2: Exercise desktop behavior**

In a browser at desktop width:

- open `/#today`, `/#pool`, and `/#me`;
- use back/forward and refresh on each hash;
- confirm full waiting and pool lists contain the API row counts;
- click rows from today and pool;
- close the sheet by button, mask, and Esc;
- confirm close keeps the same hash;
- confirm missing ATR/cluster/RS sections are absent.

- [ ] **Step 3: Exercise mobile behavior**

At 859px and 375px:

- confirm rail is hidden and bottom three-key navigation is visible;
- confirm today and me grids become one column;
- confirm no horizontal overflow;
- confirm the detail sheet fills the available width.

- [ ] **Step 4: Run all automated verification**

Run:

```bash
.venv/bin/python test_m3_home.py
.venv/bin/python test_rules.py
.venv/bin/python test_rules_mid.py
.venv/bin/python test_web.py
.venv/bin/python test_regime.py
git diff --check
```

Expected: all tests PASS and `git diff --check` is clean.

- [ ] **Step 5: Audit exact changed-file scope and dependencies**

Run:

```bash
git diff --name-only main...HEAD
git diff --exit-code main -- static/pool.html static/nav.js service.py
git diff --exit-code main -- requirements.txt pyproject.toml package.json package-lock.json
rg -n '(?<![0-9])[036][0-9]{5}(?![0-9])' static/home.html
```

Expected:

- changed production file is only `static/home.html`;
- no legacy pool, shared navigation, backend, strategy, risk, backtest, pipeline, or dependency file changed;
- the hard-coded stock-code search returns no matches.

- [ ] **Step 6: Commit any acceptance-only fix and record final evidence**

If Step 2 or Step 3 required a fix, first add a reproducing test to `test_m3_home.py`, watch it fail, implement the fix, rerun all tests, then commit:

```bash
git add test_m3_home.py static/home.html
git commit -m "修复: 收口M3投研台验收问题"
```

If no fix was required, do not create an empty commit.
