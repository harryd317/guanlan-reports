# 观澜前端整体重写 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task.

**Goal:** 以用户确认的附件原型完整替换旧前端，仅保留“今天 / 股票池 / 个股详情 / 我的”四个界面，接入现有真实数据，并在不改动任何 `/api/*` 函数源码和业务模块的前提下完成双视口验收、部署与镜像同步。

**Architecture:** 新前端是单文件 `static/home.html` SPA，使用 hash 路由和内嵌的纯 JavaScript 数据适配层。FastAPI 只保留 `/` 页面路由，旧页面与旧静态资产路由删除后由框架返回默认 404。旧 `static/` 文件原样移入版本化归档并生成 SHA-256 清单。所有不可变边界用独立清单和测试校验。

**Tech Stack:** Python 3.11, FastAPI, 原生 HTML/CSS/JavaScript, Node.js 22（仅执行前端纯函数测试）, `unittest`, SQLite 一致性快照, macOS LaunchAgent, 项目现有报告镜像脚本。

## Global Constraints

- 视觉和交互唯一标准是 `~/Downloads/观澜新前端原型-唯一标准.md`。
- `service.py` 只能保留/调整 `/` 页面路由和删除已批准的旧页面/旧资产路由。
- 任何 `/api/*` 装饰器、请求模型、函数签名和函数源码都不得变化。
- 不修改策略、风控、数据管道、记账、通知或数据库结构。
- 新前端唯一写请求是 `POST /api/account`。
- 保留用户未跟踪的 `docs/AI评分校准-202608.md`，不读写、不纳入任何提交。
- 每个实现任务遵循 TDD：先运行定向测试观察预期失败，再写最小实现，然后回归。
- 不为了“一屏”隐藏真实行或引入内部滚动区。

---

### Task 1: 冻结 API、路由和运行数据基线

**Files:**
- Create: `scripts/guanlan_frontend_contract.py`
- Create: `test_guanlan_frontend_contract.py`
- Create: `reports/guanlan-frontend-20260801/baseline/api-source-manifest.json`
- Create: `reports/guanlan-frontend-20260801/baseline/runtime-hashes.json`
- Read only: `service.py`
- Read only: 生产 LaunchAgent plist 及其引用的三个数据库路径

**Step 1: Write the failing contract tests**

`test_guanlan_frontend_contract.py` 要求：

`ApiSourceManifestTests` 包含 `test_manifest_covers_every_api_route`、
`test_manifest_is_deterministic`、
`test_manifest_detects_one_character_function_change` 和
`test_runtime_hash_manifest_names_plist_and_three_databases`。

清单项至少包含 `paths` / `methods` / `function` / `start_line` / `end_line` / `sha256`。源文本范围从第一个 `/api/` 路由装饰器的首字节开始，到下一个同级函数或文件结尾前结束，不做格式化。

**Step 2: Run the focused tests and confirm RED**

Run:

```bash
.venv/bin/python -m unittest -v test_guanlan_frontend_contract.py
```

Expected: FAIL/ERROR，因为清单生成器和基线文件尚不存在。

**Step 3: Implement deterministic source extraction and hashing**

`scripts/guanlan_frontend_contract.py` 提供：

`scripts/guanlan_frontend_contract.py` 实现并导出
`extract_api_sources(service_path: Path) -> list[dict]`、
`hash_named_files(paths: dict[str, Path]) -> dict`、
`write_json(path: Path, payload: object) -> None` 和
`compare_api_manifests(before: dict, after: dict) -> list[str]`。

使用 `ast` 定位拥有 `/api/` 装饰器的顶层函数，使用文件的原始 bytes 切片计算 SHA-256；禁止使用 `inspect.getsource()` 或 AST 反向生成文本。

**Step 4: Resolve production-equivalent inputs without mutation**

读取项目规定的生产 LaunchAgent，解析工作目录、端口、启动脚本和三库路径。如果任何路径无法从 plist/当前配置权威地确定，本任务停在未部署状态，不猜测路径。

**Step 5: Generate and verify baseline evidence**

Run:

```bash
.venv/bin/python scripts/guanlan_frontend_contract.py baseline \
  --service service.py \
  --output reports/guanlan-frontend-20260801/baseline
.venv/bin/python -m unittest -v test_guanlan_frontend_contract.py
```

Expected: PASS；报告显示 API 条目数量与应用实际 `/api/*` 路由函数一致，且 plist/三库哈希已写入。

**Step 6: Commit the evidence tooling and baseline**

```bash
git add scripts/guanlan_frontend_contract.py test_guanlan_frontend_contract.py reports/guanlan-frontend-20260801/baseline
git commit -m "test: freeze guanlan frontend backend boundary"
```

---

### Task 2: 用新前端契约替换旧 M3 UI 锁定

**Files:**
- Create: `test_guanlan_frontend.py`
- Delete: `test_m3_home.py`
- Modify: `test_branding.py`
- Modify: `test_web.py`
- Test: `test_guanlan_frontend.py`
- Test: `test_branding.py`
- Test: `test_web.py`

**Step 1: Write the new static contract tests before changing the page**

`test_guanlan_frontend.py` 建立以下静态契约：

`GuanlanShellTests` 包含
`test_only_three_primary_nav_items`、
`test_exact_visual_tokens_are_present`、
`test_four_hash_views_exist_and_detail_is_not_nav`、
`test_forbidden_legacy_ui_terms_and_links_are_absent`、
`test_demo_tickers_and_demo_numbers_are_absent` 和
`test_only_account_endpoint_can_be_used_for_write`。

禁止词至少包含：工程模式、风控护栏、违规处置、复核、画像、日报、AI、维修间、回测、K线。读写扫描覆盖 `fetch` 请求方法，允许的非 GET 组合仅为 `POST /api/account`。

**Step 2: Retire obsolete UI-specific tests without weakening backend coverage**

- 删除 `test_m3_home.py` 中所有旧 Material/Apple/M3 壳契约，由 `test_guanlan_frontend.py` 完整接替。
- `test_branding.py` 保留品牌函数和通知测试；页面测试只检查新 `home.html`。删除对旧页面和 `/brand.js` 的可达性要求。
- `test_web.py` 仅替换故意锁定旧页面、旧导航和旧静态资产的断言；API 和业务测试原样保留。旧 URL 断言改为 FastAPI 默认 404 JSON。

**Step 3: Run the focused tests and confirm RED**

```bash
.venv/bin/python -m unittest -v test_guanlan_frontend.py test_branding.py
```

Expected: 新契约 FAIL，因为当前 `home.html` 仍含旧导航和禁止元素。

**Step 4: Commit test-contract replacement**

```bash
git add test_guanlan_frontend.py test_m3_home.py test_branding.py test_web.py
git commit -m "test: replace legacy ui contracts"
```

---

### Task 3: 原样化静态归档与新单页壳

**Files:**
- Create: `archive/frontend-legacy-20260801/static/SHA256SUMS`
- Move: `static/about.html` -> `archive/frontend-legacy-20260801/static/about.html`
- Move: `static/ai_drawer.js` -> `archive/frontend-legacy-20260801/static/ai_drawer.js`
- Move: `static/ask.html` -> `archive/frontend-legacy-20260801/static/ask.html`
- Move: `static/daily.html` -> `archive/frontend-legacy-20260801/static/daily.html`
- Move: `static/history.html` -> `archive/frontend-legacy-20260801/static/history.html`
- Move: `static/home.html` -> `archive/frontend-legacy-20260801/static/home.html`
- Move: `static/nav.js` -> `archive/frontend-legacy-20260801/static/nav.js`
- Move: `static/pool.html` -> `archive/frontend-legacy-20260801/static/pool.html`
- Move: `static/reaffirm.js` -> `archive/frontend-legacy-20260801/static/reaffirm.js`
- Move: `static/settings.html` -> `archive/frontend-legacy-20260801/static/settings.html`
- Move: `static/stock_detail.html` -> `archive/frontend-legacy-20260801/static/stock_detail.html`
- Move: `static/strategy.html` -> `archive/frontend-legacy-20260801/static/strategy.html`
- Move: `static/style.css` -> `archive/frontend-legacy-20260801/static/style.css`
- Move: `static/trade.html` -> `archive/frontend-legacy-20260801/static/trade.html`
- Create: `static/home.html`
- Modify: `test_guanlan_frontend.py`

**Step 1: Add archive integrity tests**

`LegacyArchiveTests` 包含
`test_runtime_static_contains_only_home`、
`test_every_archived_file_has_matching_sha256` 和
`test_manifest_has_no_missing_or_extra_entries`。

**Step 2: Run and confirm RED**

```bash
.venv/bin/python -m unittest -v test_guanlan_frontend.LegacyArchiveTests
```

Expected: FAIL，因为归档和清单尚不存在。

**Step 3: Move old files without changing their bytes and write the manifest**

先计算每个旧文件哈希，再移动，再重算一次。任何单项不一致都停止。`SHA256SUMS` 按相对文件名字典序排序。

**Step 4: Build the exact visual shell**

`static/home.html` 首先实现：

- `#today`, `#pool`, `#stock`, `#me` 四个 view；
- 桌面 200px 侧栏，手机顶部 3 个 tab；
- 三个一级导航按钮仅为今天、股票池、我的；
- 奶油/卡片/正文/陶土/鼠尾草令牌与 16px 圆角；
- 附件原型的信息层级、表格和卡片排列；
- 无演示数据，初始值统一显示 `—` 或“正在读取”。

**Step 5: Run shell and archive tests**

```bash
.venv/bin/python -m unittest -v test_guanlan_frontend.py test_branding.py
```

Expected: 壳、令牌、导航、归档和禁止词契约 PASS；与数据适配有关的测试仍待下一任务实现。

**Step 6: Commit the archive and shell**

```bash
git add archive/frontend-legacy-20260801 static test_guanlan_frontend.py
git commit -m "feat: replace legacy frontend shell"
```

---

### Task 4: 实现可独立测试的数据适配层和 hash 路由

**Files:**
- Modify: `static/home.html`
- Modify: `test_guanlan_frontend.py`

**Step 1: Add Node-executed pure-function tests**

从 `<script id="guanlan-model">` 提取纯函数源码，在 Node VM 中测试：

```javascript
routeFromHash('#stock/600000')
distanceToBuy(10.5, 10)
buildClusterIndex(clusterPayload)
buildWaitingRows(brief, pool, intraday, clusters)
buildPriceLadder(detail, intraday)
computeInvestmentPlan(nowShanghai, snapshotSeries, book)
formatStatus(rawStatus)
```

覆盖无 hash/未知 hash/不完整详情回到 today；距买点优先用接口现成字段；无值不计算；集群优先于行业；下一周三与当周 -2% 加倍；投资仓 `position_id` 去重且最多 5 批。

**Step 2: Run and confirm RED**

```bash
.venv/bin/python -m unittest -v test_guanlan_frontend.GuanlanModelTests
```

Expected: FAIL，因为纯函数尚未实现。

**Step 3: Implement the model script**

固定展示配置：

```javascript
const INVESTMENT_PLAN = Object.freeze({
  code: '512890',
  weekday: 3,
  batches: 5,
  amount: 14000,
  doubleDropPct: -2
});
```

适配器只接收 JSON 对象并返回 view model，不访问 DOM、`fetch`、`localStorage` 或时钟隐式全局。

**Step 4: Implement hash routing and return-state restoration**

使用显式内存状态：

```javascript
const navigationState = {
  poolTab: 'list',
  poolScrollY: 0,
  detailOrigin: 'pool'
};
```

进入详情前记录来源、股票池子页和页面位置。返回只改 hash，不访问旧 `/stock/{code}` URL。

**Step 5: Run model and shell tests**

```bash
.venv/bin/python -m unittest -v test_guanlan_frontend.py
```

Expected: PASS。

**Step 6: Commit**

```bash
git add static/home.html test_guanlan_frontend.py
git commit -m "feat: add guanlan frontend data model"
```

---

### Task 5: 接线今天页的真实数据

**Files:**
- Modify: `static/home.html`
- Modify: `test_guanlan_frontend.py`

**Step 1: Add request and rendering tests**

契约断言启动时并行 GET：

```text
/api/brief
/api/pool
/api/intraday
/api/regime/current
/api/s3/summary
/api/s3/clusters
/api/trade/overview
/api/snapshot_dates
```

且 `/api/snapshot/{date}` 只会读当前自然周所需日期。渲染测试使用完整伪响应对象，检查数据来自 fixture，不是 HTML 字面量。

**Step 2: Run and confirm RED**

```bash
.venv/bin/python -m unittest -v test_guanlan_frontend.GuanlanDataWiringTests
```

Expected: FAIL，因为页面尚未发起或渲染所有真实数据。

**Step 3: Implement isolated startup loading**

使用 `Promise.allSettled` 保留已成功区块。每个 endpoint 仅通过统一 `getJson` 发送 GET，错误存入对应区块状态。

**Step 4: Render the Today sections**

- 投机线：门、市况、持续日、沪深300 距 MA60，等待数据仅有值时显示。
- 投资线：`512890`、真实投资仓、批次、下一周三和 14,000/28,000 元。
- 待处理：`brief.cards` 保留顺序、一行化，不生成动作按钮。
- 等买点：`brief.waiting` 与 watchlist 池交集，显示真实现价/距离/板块/状态。
- 广度：`nh250`, `nl250`, `net`，用户界面只出现 NH/NL 语义。

**Step 5: Add honest error states**

区块失败显示“暂时取不到”，字段缺失显示 `—`。不使用默认股票、默认市场值或本地缓存。

**Step 6: Run and commit**

```bash
.venv/bin/python -m unittest -v test_guanlan_frontend.py
git add static/home.html test_guanlan_frontend.py
git commit -m "feat: wire guanlan today view"
```

---

### Task 6: 接线股票池与个股详情

**Files:**
- Modify: `static/home.html`
- Modify: `test_guanlan_frontend.py`

**Step 1: Add pool/detail fixture tests**

测试包含完整股票池顺序、集群命中、行业 fallback、缺失字段、按需 GET `/api/pool/{code}`、价格阶梯和返回状态。

**Step 2: Run and confirm RED**

```bash
.venv/bin/python -m unittest -v test_guanlan_frontend.GuanlanPoolDetailTests
```

Expected: FAIL，因为名单、固定选股逻辑子页和详情尚未完整渲染。

**Step 3: Implement the pool view**

- 默认子页“名单”，完整保留 `/api/pool.pool` 顺序。
- 五列：股票、状态、现价、距买点、板块。
- 板块：集群名称+命中数 > `industry` > `—`。
- “选股逻辑”仅使用附件原型的两张固定说明卡，不读取或暴露策略参数。

**Step 4: Implement the detail view**

详情仅显示名称/代码/行业/状态、现价/涨跌幅、止损-买点-现价阶梯、距买点、账户权益 1% 计划风险金额、有值时的相对强度、集群联动、状态时间线和一句依据。

**Step 5: Run and commit**

```bash
.venv/bin/python -m unittest -v test_guanlan_frontend.py
git add static/home.html test_guanlan_frontend.py
git commit -m "feat: wire guanlan pool and stock detail"
```

---

### Task 7: 接线“我的”与唯一写操作

**Files:**
- Modify: `static/home.html`
- Modify: `test_guanlan_frontend.py`
- Modify: `test_web.py`

**Step 1: Add account interaction and persistence tests**

前端纯函数/DOM 契约覆盖 Enter 保存、blur 保存、Escape 取消、保存中禁用、错误恢复原值、成功后重新 GET overview，且不写 `localStorage`。

`test_web.py` 新增使用临时数据库/临时配置的端到端接口测试：

测试名为 `test_account_save_then_overview_refresh_keeps_value`。

测试不得指向生产数据库。

**Step 2: Run and confirm RED**

```bash
.venv/bin/python -m unittest -v \
  test_guanlan_frontend.GuanlanAccountTests \
  test_web.WebTests.test_account_save_then_overview_refresh_keeps_value
```

Expected: FAIL，因为内联编辑交互尚未实现。

**Step 3: Implement My view**

- 权益读取 `trade.overview.account_size`。
- 两仓读取 `books.speculative` / `books.investment` 持仓笔数和金额。
- 投资仓备注显示下一批日期、序号和本周金额。
- 验证期读取 `stats_current.n`，显示 `min(n, 20) / 20` 和对应进度。

**Step 4: Implement the only write path**

```javascript
await fetch('/api/account', {
  method: 'POST',
  headers: {'Content-Type': 'application/json'},
  body: JSON.stringify({account_size: parsedValue})
});
```

成功后必须重读 `/api/trade/overview`，不更改或复制后端任何账户设置逻辑。

**Step 5: Run and commit**

```bash
.venv/bin/python -m unittest -v test_guanlan_frontend.py
.venv/bin/python -m unittest -v test_web.py
git add static/home.html test_guanlan_frontend.py test_web.py
git commit -m "feat: add guanlan account inline editing"
```

---

### Task 8: 删除旧页面路由并证明 API 源码未变

**Files:**
- Modify: `service.py`
- Modify: `test_web.py`
- Modify: `test_branding.py`
- Modify: `test_guanlan_frontend_contract.py`
- Create: `reports/guanlan-frontend-20260801/verification/api-source-manifest.json`

**Step 1: Add route tests before route deletion**

测试 `/` 为 200，以下均为 FastAPI 默认 404：

```text
/pro /ask /strategy /settings /about /history /trade
/page/pool /pool /stock/600000 /daily
/nav.js /ai-drawer.js /reaffirm.js /brand.js
```

同时检查 `/doc/{key}` 和基线中所有 `/api/*` 路由仍存在。

**Step 2: Run and confirm RED**

```bash
.venv/bin/python -m unittest -v \
  test_guanlan_frontend_contract.py \
  test_branding.py
```

Expected: 旧 URL 404 断言 FAIL，因为当前旧路由仍可达。

**Step 3: Apply the only permitted `service.py` edit**

保留 `/` 函数，删除已批准的页面和 JS 资产路由函数。不重排 import，不运行格式化器，不触碰 API 函数上下文。

**Step 4: Regenerate and compare API manifests**

```bash
.venv/bin/python scripts/guanlan_frontend_contract.py verify \
  --service service.py \
  --baseline reports/guanlan-frontend-20260801/baseline/api-source-manifest.json \
  --output reports/guanlan-frontend-20260801/verification/api-source-manifest.json
```

Expected: exit 0，差异列表为空，每个 API 函数 SHA-256 与基线一致。

**Step 5: Run service-facing tests**

```bash
.venv/bin/python -m unittest -v \
  test_guanlan_frontend_contract.py \
  test_guanlan_frontend.py \
  test_branding.py \
  test_web.py
```

Expected: PASS。

**Step 6: Commit**

```bash
git add service.py test_web.py test_branding.py test_guanlan_frontend_contract.py reports/guanlan-frontend-20260801/verification/api-source-manifest.json
git commit -m "feat: retire legacy frontend routes"
```

---

### Task 9: 强化生产等效与双视口一屏验收器

**Files:**
- Modify: `scripts/t7_equivalent_env.py`
- Modify: `test_production_equivalent.py`
- Create: `reports/guanlan-frontend-20260801/acceptance/acceptance-matrix.json`
- Create: `reports/guanlan-frontend-20260801/acceptance/screenshots/`

**Step 1: Add vertical-overflow and evidence completeness tests**

`test_production_equivalent.py` 新增：

新增 `test_visual_evidence_rejects_vertical_scroll`、
`test_visual_evidence_rejects_wrong_viewport_or_image_size`、
`test_visual_evidence_requires_console_and_screenshot_hash` 和
`test_acceptance_matrix_requires_four_views_at_two_viewports`。

**Step 2: Run and confirm RED**

```bash
.venv/bin/python -m unittest -v test_production_equivalent.py
```

Expected: FAIL，因为现有验证器只检查宽度/横向溢出。

**Step 3: Extend evidence validation**

`validate_visual_evidence` 额外要求：

```python
document_scroll_height <= inner_height
document_scroll_width <= inner_width
image_width == inner_width
image_height == inner_height
console_errors == []
len(screenshot_sha256) == 64
```

验收矩阵必须正好包含 today/pool/stock/me × desktop/mobile 八项。

**Step 4: Run and commit**

```bash
.venv/bin/python -m unittest -v test_production_equivalent.py
git add scripts/t7_equivalent_env.py test_production_equivalent.py
git commit -m "test: enforce guanlan one-screen acceptance"
```

---

### Task 10: 全量自动化回归与代码边界复核

**Files:**
- Modify only if a test exposes a scoped frontend/test defect: `static/home.html`, `service.py`, frontend-specific tests/helpers
- Create: `reports/guanlan-frontend-20260801/verification/test-results.txt`
- Create: `reports/guanlan-frontend-20260801/verification/final-runtime-hashes.json`

**Step 1: Run focused frontend and delivery tests**

```bash
.venv/bin/python -m unittest -v \
  test_guanlan_frontend_contract.py \
  test_guanlan_frontend.py \
  test_branding.py \
  test_portfolio_books.py \
  test_production_equivalent.py \
  test_report_mirror_sync.py
```

Expected: PASS。

**Step 2: Run the repository-mandated suite exactly**

```bash
.venv/bin/python test_rules.py && \
.venv/bin/python test_rules_mid.py && \
.venv/bin/python test_web.py && \
.venv/bin/python test_regime.py
```

Expected: 四段均 exit 0。把命令、开始/结束时间、用例数和结果写入 `test-results.txt`。

**Step 3: Re-run the immutable-boundary comparison**

```bash
.venv/bin/python scripts/guanlan_frontend_contract.py verify \
  --service service.py \
  --baseline reports/guanlan-frontend-20260801/baseline/api-source-manifest.json \
  --output reports/guanlan-frontend-20260801/verification/api-source-manifest.json
```

重算 plist 和三个生产数据库哈希到 `final-runtime-hashes.json`，必须与 baseline 一致。

**Step 4: Review the final diff**

```bash
git diff --check
git status --short
git diff --stat HEAD~8..HEAD
git diff HEAD~8..HEAD -- service.py
```

确认无意外的业务文件、无用户未跟踪文件、`service.py` 差异只有旧页面路由删除与 `/` 说明。

**Step 5: Commit verification evidence**

```bash
git add reports/guanlan-frontend-20260801/verification
git commit -m "test: record guanlan frontend verification"
```

---

### Task 11: 生产等效视觉和交互验收

**Files:**
- Create: `reports/guanlan-frontend-20260801/acceptance/launch-agent-clone.plist`
- Create: `reports/guanlan-frontend-20260801/acceptance/db-hashes.json`
- Create: `reports/guanlan-frontend-20260801/acceptance/acceptance-matrix.json`
- Create: `reports/guanlan-frontend-20260801/acceptance/screenshots/*.png`
- Create: `reports/guanlan-frontend-20260801/acceptance/account-persistence.json`

**Step 1: Clone the production environment safely**

使用 `scripts/t7_equivalent_env.py` 克隆 LaunchAgent 配置，保持相同工作目录规则、启动脚本和静态服务方式，只替换为独立端口/独立标签/独立快照路径。三个数据库先一致性快照，默认只读。

**Step 2: Start the isolated instance and perform health probes**

验证 `/` 200、一个现有健康 API 200、所有已退役 URL 404。记录端口、PID、工作目录、启动命令和快照哈希。

**Step 3: Capture eight ordinary-viewport screenshots**

使用浏览器在 `1440×900` 和 `390×844` 依次访问：

```text
#today #pool #stock/<a real pool code> #me
```

每项记录 viewport、截图实际像素、`innerWidth/Height`、`scrollWidth/Height`、控制台错误、截图 SHA-256。不使用 full-page 截图。

**Step 4: Verify pool-detail-back behavior**

在名单中记录真实行代码、当前子页和位置，点击进入详情，点击返回，检查代码、子页和位置都恢复。

**Step 5: Verify account save/refresh on a disposable writable copy**

复制只读快照为一次性可写验收副本，用与生产等效实例隔离的第二个进程运行。修改权益 -> 保存 -> 刷新 -> 读回同值，记录前/保存/刷新后值。停止进程后删除一次性副本；生产三库哈希必须未变。

**Step 6: Validate and commit acceptance evidence**

```bash
.venv/bin/python -m unittest -v test_production_equivalent.py
git add reports/guanlan-frontend-20260801/acceptance
git commit -m "test: record guanlan production-equivalent acceptance"
```

Expected: 八项全部 `scrollHeight <= innerHeight` 且 `scrollWidth <= innerWidth`；交互两项 PASS；生产哈希不变。任意一项失败都不进入部署。

---

### Task 12: 完成报告、部署、复验和公开镜像

**Files:**
- Create: `docs/reports/2026-08-01-guanlan-frontend-rewrite.md`
- Create: `reports/guanlan-frontend-20260801/deployment/post-deploy.json`
- Modify only if required by the existing mirror workflow: none

**Step 1: Write the completion report before deployment**

报告链接归档清单、API 前后清单、plist/三库哈希、测试结果、八张视觉证据、详情返回和权益持久化结果，并写明源提交、待部署提交。报告不使用未运行的计划代替证据。

**Step 2: Final pre-deploy gate**

重新运行 Task 10 的全量测试、API 清单比对和生产哈希比对，然后校验 Task 11 验收矩阵。只有全绿时允许继续。

**Step 3: Deploy with the project-prescribed restart path**

将已验收的提交部署到现有生产工作树，然后在生产工作目录执行：

```bash
./restart.sh
```

不修改 LaunchAgent 语义、不迁移数据库、不执行策略或数据管道命令。

**Step 4: Post-deploy probes**

记录 `/` 200、全部旧 URL 404、健康 API 200、页面仅三个导航。再计算生产 plist 和三库哈希，必须与部署前一致。写入 `post-deploy.json`。

**Step 5: Commit the final report and deployment evidence**

```bash
git add docs/reports/2026-08-01-guanlan-frontend-rewrite.md reports/guanlan-frontend-20260801/deployment
git commit -m "docs: report guanlan frontend deployment"
```

记录该提交为部署/报告源提交。

**Step 6: Sync and verify the public report mirror**

按仓库现有规范执行镜像脚本，使用“只有 HEAD 触及交付文档才同步”模式并推送。然后从镜像远端重新读取报告文件和提交对象，比较本地/远端文件 SHA-256。

Expected:

- 远端报告内容与本地相同；
- 镜像提交可从远端解析；
- 最终向用户回报源/部署提交、API 清单哈希、三库哈希比对、八项验收结果和镜像提交哈希。
