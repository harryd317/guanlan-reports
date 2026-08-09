# Today Line Labels Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Correct the Today page labels so S8 is explicitly the investment line and the market gate is strategy-neutral.

**Architecture:** Keep the existing single-file frontend and data flow unchanged. Add one static subtitle inside the existing S8 card, replace the two visible headings, and protect the resulting DOM contract with a focused test before running the existing global acceptance pipeline.

**Tech Stack:** Static HTML/CSS/JavaScript, Python `unittest`, FastAPI production runtime, macOS LaunchAgent, SQLite read-only equivalent snapshots, in-app browser acceptance.

## Global Constraints

- Change display copy only; do not change data, API, database, route, engine, scheduler, or pool-logic card behavior.
- Today S8 title: `投资线 · S8 关注`.
- Today S8 subtitle: `产业核心股研究信号 · 供学习 · 非交易建议`.
- Market gate title: `市况 · 开仓门`; the Today page must not label it as the speculative line.
- Keep the three pool-logic cards unchanged.
- Both equivalent and production acceptance must pass global zero-scroll at `1440×900` and `390×844`.
- Commit the final report, synchronize the public report mirror over HTTPS, verify through a fresh clone, and report the mirror commit hash.

---

### Task 1: Freeze the visible naming contract

**Files:**
- Modify: `test_guanlan_frontend.py`
- Modify: `static/home.html`

**Interfaces:**
- Consumes: the real checked-in `static/home.html` parsed by `GuanlanShellTests`.
- Produces: visible DOM text under `#s8-focus`, a subtitle node under the same card, and the exact text of `#spec-key`.

- [ ] **Step 1: Write the failing test**

Add a shell test that parses the real Today DOM and asserts:

```python
from bs4 import BeautifulSoup


def test_today_names_s8_as_investment_and_market_gate_as_infrastructure(self):
    soup = BeautifulSoup(home_source(), "html.parser")
    s8 = soup.select_one("#s8-focus")
    self.assertIn("投资线 · S8 关注", s8.get_text(" ", strip=True))
    self.assertEqual(
        s8.select_one(".s8-subtitle").get_text(" ", strip=True),
        "产业核心股研究信号 · 供学习 · 非交易建议",
    )
    self.assertEqual(
        soup.select_one("#spec-key").get_text(" ", strip=True),
        "市况 · 开仓门",
    )
    today = soup.select_one("#view-today")
    self.assertNotIn("投机线 · 开仓门", today.get_text(" ", strip=True))
```

Keep the existing pool-logic test as the invariant for the three unchanged cards.

- [ ] **Step 2: Run the test and verify RED**

Run:

```bash
~/云慧养/zmu_gitee/screener/.venv/bin/python -m unittest test_guanlan_frontend.GuanlanShellTests.test_today_names_s8_as_investment_and_market_gate_as_infrastructure -v
```

Expected: FAIL because the current title is `S8 关注`, `.s8-subtitle` is absent, and `#spec-key` still carries the speculative-line prefix.

- [ ] **Step 3: Implement the minimal DOM change**

Change only the Today markup:

```html
<h3>投资线 · S8 关注 ...</h3>
<div class="s8-subtitle">产业核心股研究信号 · 供学习 · 非交易建议</div>
```

Replace the market key with:

```html
<div class="key" id="spec-key">市况 · 开仓门</div>
```

Add only the compact subtitle style needed to preserve the current card height and visual hierarchy.

- [ ] **Step 4: Run the focused and frontend suites**

Run:

```bash
~/云慧养/zmu_gitee/screener/.venv/bin/python -m unittest test_guanlan_frontend.GuanlanShellTests.test_today_names_s8_as_investment_and_market_gate_as_infrastructure -v
~/云慧养/zmu_gitee/screener/.venv/bin/python test_guanlan_frontend.py
```

Expected: focused test PASS; frontend suite 39/39 PASS.

- [ ] **Step 5: Commit the TDD change**

```bash
git add static/home.html test_guanlan_frontend.py
git commit -m "fix(ui): correct today line labels"
```

### Task 2: Verify, deploy, and deliver evidence

**Files:**
- Create: `reports/evidence/today-line-labels/equivalent/*`
- Create: `reports/evidence/today-line-labels/production/*`
- Create: `reports/验收报告-今天页两线冠名修正-20260804.md`

**Interfaces:**
- Consumes: production LaunchAgent, `start.sh`, production databases, global zero-scroll route matrix, report mirror hook.
- Produces: immutable hashes, double-viewport screenshots, zero-scroll JSON, final report, verified mirror commit.

- [ ] **Step 1: Run focused and mandatory regression suites**

Run the frontend/equivalent suites, then the repository-mandated commands:

```bash
~/云慧养/zmu_gitee/screener/.venv/bin/python -m unittest -v test_guanlan_frontend test_production_equivalent
~/云慧养/zmu_gitee/screener/.venv/bin/python test_rules.py
~/云慧养/zmu_gitee/screener/.venv/bin/python test_rules_mid.py
~/云慧养/zmu_gitee/screener/.venv/bin/python test_web.py
~/云慧养/zmu_gitee/screener/.venv/bin/python test_regime.py
```

Expected: every suite exits 0.

- [ ] **Step 2: Run production-equivalent acceptance**

Create read-only SQLite snapshots for all production inputs, clone the production plist while changing only label, port, and isolated database paths, launch on an independent port, and record hashes before and after. Verify health, capture Today at both frozen viewports, and run the 20-route global zero-scroll matrix.

Expected: health `ok=true`; 20/20 zero-scroll; console errors empty; production plist, production databases, and copies retain their hashes.

- [ ] **Step 3: Merge and deploy**

Merge `ui/today-line-labels` into `main`, record production hashes, run `./restart.sh`, and confirm health fingerprint equality.

Expected: production `ok=true`; immediate pre/post database and plist hashes equal.

- [ ] **Step 4: Repeat production visual acceptance**

Capture Today at `1440×900` and `390×844`, run the same 20-route zero-scroll matrix, and write screenshot hashes and dimensions to a manifest.

Expected: exact viewport dimensions, 20/20 zero-scroll, no console errors, and the two new labels visible.

- [ ] **Step 5: Commit report and verify mirror**

Commit the final report and evidence. Let the HTTPS mirror hook publish the delivery tree, then clone `https://github.com/harryd317/guanlan-reports.git` into a new temporary directory and compare the report, screenshots, design, plan, and `AGENTS.md`.

Expected: fresh-clone HEAD equals the mirror commit, the clone is clean, all required files exist, and delivery hashes match after the mirror's documented path redaction.
