# T7 其余页面实施计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 把股票池、我的页、工程模式和股息率设置改成已批准的个人驾驶舱界面。

**Architecture:** 继续复用 `static/home.html` 的三入口壳和现有只读接口。股息率新增一个窄写接口，直接写既有设置键；其余功能只改展示层，不改变策略或数据结构。

**Tech Stack:** Python、FastAPI、原生 HTML/CSS/JavaScript、现有 SQLite 存储。

## Global Constraints

- 继续使用分支 `codex/t7-today-two-lines`，不合并、不部署。
- 今天页冻结，不改变已验收布局和数据口径。
- 零新增依赖。
- 不修改 S2、风控规则、回测、数据管道或数据库结构。
- 只使用既有 M3 令牌和四语义色。
- 新增可见句子不超过 15 个汉字。
- 每个页面提供桌面和手机截图。

---

### Task 1: 股票池纯表格

**Files:**
- Modify: `test_m3_home.py`
- Modify: `static/home.html`

**Interfaces:**
- Consumes: `GET /api/pool`
- Produces: `M3Model.poolRows(rows)` 和只读 `.m3-pool-table`

- [ ] **Step 1: Write the failing tests**

```python
def test_pool_is_one_read_only_table(self):
    self.assertIn('class="m3-pool-table"', HOME)
    self.assertNotIn('class="m3-pool-group"', HOME)
    self.assertNotIn("renderS3Clusters(S3_CLUSTERS)", pool_render)

def test_pool_table_keeps_state_order(self):
    result = run_model("M3Model.poolRows(rows).map(r=>r.id)")
    self.assertEqual(result, [watch_id, candidate_id, research_id, holding_id, cooldown_id])
```

- [ ] **Step 2: Run tests and verify RED**

Run: `./.venv/bin/python test_m3_home.py`  
Expected: FAIL because `poolRows` and `.m3-pool-table` do not exist.

- [ ] **Step 3: Implement the table**

Add `poolRows` to `M3Model`. Replace grouped card markup with one semantic table. Keep row click and keyboard access to the existing detail sheet.

- [ ] **Step 4: Run tests and verify GREEN**

Run: `./.venv/bin/python test_m3_home.py`  
Expected: PASS.

### Task 2: 我的页和工程模式

**Files:**
- Modify: `test_m3_home.py`
- Modify: `static/home.html`

**Interfaces:**
- Consumes: `GET /api/trade/overview`
- Produces: `.m3-books`, `.m3-risk`, `.m3-violations` 和默认关闭的 `.m3-engineering`

- [ ] **Step 1: Write the failing tests**

```python
def test_me_has_only_daily_sections_before_engineering(self):
    self.assertIn("投机仓", HOME)
    self.assertIn("投资仓", HOME)
    self.assertIn("违规处置", HOME)
    self.assertIn("无违规头寸", HOME)

def test_engineering_is_closed_and_keeps_old_links(self):
    self.assertIn('<details class="m3-engineering">', HOME)
    self.assertNotIn('<details class="m3-engineering" open>', HOME)
```

- [ ] **Step 2: Run tests and verify RED**

Run: `./.venv/bin/python test_m3_home.py`  
Expected: FAIL because the two-book and violation sections do not exist.

- [ ] **Step 3: Implement the daily sections**

Render `payload.books.speculative`, `payload.books.investment`, risk fields, and `payload.violations`. Move account statistics, validation progress, and old links inside engineering mode.

- [ ] **Step 4: Run tests and verify GREEN**

Run: `./.venv/bin/python test_m3_home.py`  
Expected: PASS.

### Task 3: 股息率读写

**Files:**
- Modify: `test_t7_overview.py`
- Modify: `service.py`
- Modify: `static/settings.html`

**Interfaces:**
- Consumes: `storage.load_settings()` and `storage.save_setting(key, value)`
- Produces: `GET /api/settings.investment_dividend_yield` and `POST /api/investment/dividend-yield`

- [ ] **Step 1: Write the failing API tests**

```python
def test_dividend_yield_saves_without_rule_version_change(self):
    before = params.rule_version()
    response = service.api_investment_dividend_yield(
        service.DividendYieldBody(value=3.25))
    self.assertEqual(response, {"ok": True, "value": 3.25})
    self.assertEqual(params.rule_version(), before)

def test_dividend_yield_rejects_out_of_range(self):
    response = service.api_investment_dividend_yield(
        service.DividendYieldBody(value=101))
    self.assertFalse(response["ok"])
```

- [ ] **Step 2: Run tests and verify RED**

Run: `./.venv/bin/python test_t7_overview.py`  
Expected: FAIL because the body and endpoint do not exist.

- [ ] **Step 3: Implement the API**

Return the saved value from `/api/settings`. Accept `None` to clear; otherwise accept only `0..100`, round to two decimals, and write `INVEST_DIVIDEND_YIELD`.

- [ ] **Step 4: Add the settings UI contract**

Test for `investment-dividend-yield`, the POST endpoint, a percent suffix, and default-closed `工程设置`. Then add the M3 investment card and fold the existing maintenance controls.

- [ ] **Step 5: Run tests and verify GREEN**

Run: `./.venv/bin/python test_t7_overview.py && ./.venv/bin/python test_m3_home.py`  
Expected: PASS.

### Task 4: Regression and screenshots

**Files:**
- Create: `docs/screenshots/t7/股票池-桌面-20260731.png`
- Create: `docs/screenshots/t7/股票池-手机-20260731.png`
- Create: `docs/screenshots/t7/我的-桌面-20260731.png`
- Create: `docs/screenshots/t7/我的-手机-20260731.png`
- Create: `docs/screenshots/t7/工程模式-桌面-20260731.png`
- Create: `docs/screenshots/t7/工程模式-手机-20260731.png`
- Create: `docs/screenshots/t7/设置-桌面-20260731.png`
- Create: `docs/screenshots/t7/设置-手机-20260731.png`

- [ ] **Step 1: Run the full test suite**

Run: `./.venv/bin/python test_web.py`  
Expected: `全部通过`.

- [ ] **Step 2: Check scope**

Run: `git diff --check` and `git diff --name-only 7e87228..HEAD`  
Expected: only T7 UI, API, tests, docs, and screenshots.

- [ ] **Step 3: Start an isolated preview**

Use cloned `market.db` and `screener.db`; point S3 reads at the existing read-only T1 database.

- [ ] **Step 4: Capture all page states**

Use 1440×900 and 390×844 viewports. Verify no browser errors, no horizontal overflow outside the pool table wrapper, and engineering mode is closed by default.

- [ ] **Step 5: Commit and stop**

Commit the implementation and screenshots. Do not merge or deploy.

