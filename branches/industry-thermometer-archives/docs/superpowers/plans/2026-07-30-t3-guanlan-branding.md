# T3 Guanlan Branding Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 将所有用户可见品牌统一为“观澜”，并为网页与通知提供一个不可漂移的品牌配置源。

**Architecture:** 根模块 `branding.py` 保存唯一品牌配置并派生页面标题和通知签名；`service.py` 把该配置生成 `/brand.js`，所有静态页面通过 DOM 数据属性消费。Server酱、企业微信与 macOS 通知只在既有统一出站边界添加签名，不改变业务调用链。

**Tech Stack:** Python 3 标准库、FastAPI 既有栈、原生 HTML/CSS/JavaScript、`unittest`。

## Global Constraints

- 只改用户可见展示文案：网页标题、侧边栏品牌位、日报/通知签名、关于页。
- 品牌主文案固定为“观澜”，副文案固定为“A股纪律投研台”，品牌句固定为“观水有术，必观其澜”。
- 所有品牌文案必须收口到 `branding.py` 的一个 `BRAND` 配置常量。
- 禁止改代码标识符、文件名、数据库名、API 路径、环境变量、仓库名。
- 不改生产 S2 策略、风控规则、数据管道、回测核心和数据库结构。
- 零新增依赖，禁止硬编码演示数据。
- 只在 `codex/t3-guanlan-branding` 分支交付；不合并、不部署、不接生产、不开始 T4。

---

### Task 1: Brand Contract and Python Consumers

**Files:**
- Create: `branding.py`
- Create: `test_branding.py`
- Modify: `service.py`

**Interfaces:**
- Produces: `BRAND: dict[str, str]`
- Produces: `page_title(section: str = "") -> str`
- Produces: `notification_title(title: str) -> str`
- Produces: `brand_javascript() -> str`
- Produces: `GET /brand.js`

- [ ] **Step 1: Write the failing brand contract tests**

Add tests with literal expected results:

```python
def test_page_and_notification_titles_derive_from_one_brand():
    self.assertEqual(page_title("今日"), "观澜 · 今日")
    self.assertEqual(page_title(), "观澜")
    self.assertEqual(notification_title("日报已生成"), "[观澜] 日报已生成")
    self.assertEqual(notification_title("[观澜] 日报已生成"), "[观澜] 日报已生成")

def test_brand_javascript_exposes_config_and_updates_dom():
    script = brand_javascript()
    self.assertIn('"name":"观澜"', script)
    self.assertIn("window.GUANLAN_BRAND", script)
    self.assertIn("[data-brand-name]", script)
    self.assertIn("document.title", script)
```

- [ ] **Step 2: Run the contract tests and verify RED**

Run: `.venv/bin/python -m unittest -q test_branding.BrandingContractTests`

Expected: error because `branding` does not exist.

- [ ] **Step 3: Implement the minimal brand module**

Create `branding.py` with the exact `BRAND` dictionary from the design. Implement:

```python
def page_title(section: str = "") -> str:
    section = section.strip()
    return f"{BRAND['name']} · {section}" if section else BRAND["name"]

def notification_title(title: str) -> str:
    prefix = f"[{BRAND['name']}]"
    title = title.strip()
    return title if title.startswith(prefix) else f"{prefix} {title}".rstrip()
```

`brand_javascript()` must serialize `BRAND` with `json.dumps(..., ensure_ascii=False, separators=(",", ":"))`, assign `window.GUANLAN_BRAND`, update the three data-attribute node sets, and derive the browser title from `document.title` as the section-only input.

- [ ] **Step 4: Expose the JavaScript response**

In `service.py`, import `brand_javascript`, change FastAPI's visible OpenAPI title to `page_title(BRAND["subtitle"])`, and add:

```python
@app.get("/brand.js", include_in_schema=False)
def brand_js():
    return Response(
        content=brand_javascript(),
        media_type="application/javascript",
        headers={"Cache-Control": "no-store"},
    )
```

- [ ] **Step 5: Run the contract tests and verify GREEN**

Run: `.venv/bin/python -m unittest -q test_branding.BrandingContractTests`

Expected: all brand contract tests pass.

### Task 2: Existing Pages and Navigation

**Files:**
- Modify: `static/nav.js`
- Modify: `static/ask.html`
- Modify: `static/daily.html`
- Modify: `static/history.html`
- Modify: `static/home.html`
- Modify: `static/pool.html`
- Modify: `static/settings.html`
- Modify: `static/stock_detail.html`
- Modify: `static/strategy.html`
- Modify: `static/trade.html`
- Test: `test_branding.py`

**Interfaces:**
- Consumes: `window.GUANLAN_BRAND` from `/brand.js`
- Consumes: `data-brand-name`, `data-brand-subtitle`, `data-brand-tagline`
- Produces: browser-visible `观澜 · <页面>` titles and navigation label

- [ ] **Step 1: Write failing page integration tests**

For each existing page, request the real route through FastAPI's test client and assert the returned HTML loads `/brand.js`. Assert that the navigation script uses `window.GUANLAN_BRAND.name`, and that the home brand node uses `data-brand-name` instead of visible `Screener`.

For `stock_detail.html`, assert its dynamic title keeps only the stock name/code as the section:

```javascript
document.title = r.name || CODE;
```

- [ ] **Step 2: Run the page tests and verify RED**

Run: `.venv/bin/python -m unittest -q test_branding.PageBrandingTests`

Expected: failures showing missing `/brand.js`, hardcoded navigation brand, visible `Screener`, and old full document titles.

- [ ] **Step 3: Apply the minimal page changes**

In every existing HTML page:

1. Replace its `<title>` contents with the section only: `今日`, `问AI`, `日报`, `历史`, `股票池`, `设置`, `个股详情`, `市况`, or `交易台`.
2. Add `<script src="/brand.js"></script>` in `<head>` after `<title>`.
3. On the home page, replace the literal brand node with `<div class="m3-brand" data-brand-name></div>`.
4. In `stock_detail.html`, change the dynamic assignment to `document.title = r.name || CODE;` and call `window.GUANLAN_APPLY_BRAND()` after it.
5. In `nav.js`, render the brand label from `window.GUANLAN_BRAND.name`.

- [ ] **Step 4: Run page tests and verify GREEN**

Run: `.venv/bin/python -m unittest -q test_branding.PageBrandingTests`

Expected: all page branding tests pass.

### Task 3: About Page

**Files:**
- Create: `static/about.html`
- Modify: `service.py`
- Modify: `static/settings.html`
- Test: `test_branding.py`

**Interfaces:**
- Produces: `GET /about`
- Consumes: `/brand.js` DOM data attributes
- Produces: settings link `href="/about"`

- [ ] **Step 1: Write failing about-page tests**

Request `/about` and assert HTTP 200. Assert the returned real HTML loads `/brand.js` and contains the three data attributes, without embedding any of the three brand strings directly. Request `/settings` and assert it links to `/about`.

- [ ] **Step 2: Run the tests and verify RED**

Run: `.venv/bin/python -m unittest -q test_branding.AboutPageTests`

Expected: `/about` returns 404 and settings has no about link.

- [ ] **Step 3: Implement the about page and route**

Create `static/about.html` using existing local styles and navigation conventions. Its brand content must be:

```html
<h1 data-brand-name></h1>
<p data-brand-subtitle></p>
<blockquote data-brand-tagline></blockquote>
```

Add `GET /about` beside the other static page routes in `service.py`. Add a visible `/about` link in the settings page. Do not introduce network assets.

- [ ] **Step 4: Run the tests and verify GREEN**

Run: `.venv/bin/python -m unittest -q test_branding.AboutPageTests`

Expected: all about-page tests pass.

### Task 4: Notification Signatures

**Files:**
- Modify: `push.py`
- Modify: `eod.py`
- Modify: `test_web.py`
- Test: `test_branding.py`

**Interfaces:**
- Consumes: `notification_title(title: str) -> str`
- Produces: `[观澜] <原标题>` at Server酱, enterprise WeChat, and macOS boundaries

- [ ] **Step 1: Write failing outbound notification tests**

Use a local fake HTTP server or patch only `requests.post` to capture the final Server酱 and enterprise WeChat JSON/form payloads, while keeping `send_push` real. Patch only `subprocess.run` to capture the real AppleScript built by `_macos_notify`.

Assert literal outbound titles:

```python
self.assertEqual(serverchan_form["title"], "[观澜] 日报已生成")
self.assertEqual(wecom_json["text"]["content"].splitlines()[0], "[观澜] 日报已生成")
self.assertIn('with title "[观澜] 日报已生成"', applescript)
```

- [ ] **Step 2: Run notification tests and verify RED**

Run: `.venv/bin/python -m unittest -q test_branding.NotificationBrandingTests`

Expected: final outbound titles are the unsigned originals.

- [ ] **Step 3: Add signing only at outbound boundaries**

In `push.send_push`, derive `signed_title = notification_title(title)` once and pass it to every enabled provider. In `eod._macos_notify`, derive the signed title before existing sanitization. Do not change any notification call site or scheduling logic.

Update existing `test_web.py` expectations only where they intentionally asserted the pre-T3 outbound title.

- [ ] **Step 4: Run notification and web tests**

Run:

```bash
.venv/bin/python -m unittest -q test_branding.NotificationBrandingTests
S3_RESEARCH_DB=/private/tmp/t3-web-tests.db .venv/bin/python -m unittest -q test_web
```

Expected: notification tests pass and the complete web suite passes.

### Task 5: Full Verification, Evidence, and Branch Handoff

**Files:**
- Create: `docs/验收-T3-观澜更名-20260730.md`
- Create: `docs/证据-T3-观澜-首页-桌面-20260730.png`
- Create: `docs/证据-T3-观澜-关于-手机-20260730.png`

**Interfaces:**
- Produces: self-contained T3 acceptance report and visual evidence

- [ ] **Step 1: Run all automated verification**

Run:

```bash
.venv/bin/python -m unittest -q test_rules_mid
.venv/bin/python -m unittest -q test_regime test_m3_home test_report_mirror_sync
.venv/bin/python -m unittest -q test_s3_research test_s3_ui
S3_RESEARCH_DB=/private/tmp/t3-full-web-tests.db .venv/bin/python -m unittest -q test_branding test_web
.venv/bin/python -m compileall -q branding.py service.py push.py eod.py
git diff --check
```

Expected: every test passes, compilation succeeds, and `git diff --check` prints nothing.

- [ ] **Step 2: Audit the exact diff boundary**

Run `git diff --name-only ea72736...HEAD` after the implementation commit and verify no strategy, risk, backtest, data-pipeline, database, requirement, S3, or S5 file is present. Run `git diff -- requirements.txt` and expect no output. Search static HTML for user-visible `Screener` and old brand phrases; internal identifiers are recorded separately and preserved.

- [ ] **Step 3: Capture browser evidence**

Start the branch on an isolated local port with temporary log/research paths. Capture a desktop home screenshot showing “观澜” and a mobile about screenshot showing the main name, subtitle, and tagline. Do not touch the production process or production port.

- [ ] **Step 4: Write the acceptance report**

Record:

- branch and base commit;
- exact changed-file list;
- every test command and count;
- old-brand search classification;
- notification payload evidence from automated tests;
- screenshot paths;
- checkboxes for every T3 requirement and every forbidden area;
- explicit statements: not merged, not deployed, production unchanged, T4 not started.

- [ ] **Step 5: Commit and stop**

Stage only tracked T3 source, tests, docs, and evidence. Exclude local `.venv`, `reports`, database and log symlinks/artifacts. Commit with:

```bash
git commit -m "feat: rename visible product brand to Guanlan"
```

Run the mirror sync hook for the branch documentation, report the commit hash, and stop on `codex/t3-guanlan-branding` pending user acceptance.
