# S3 T1 数据集群与 T2 回测实施计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 在独立研究库内完成 S3 广度、申万二级集群和冻结回测，向现有 M3 首页提供只读观察数据，并且不改变生产 S2、风控、生产数据库或交易行为。

**Architecture:** T1 新建 `s3_research` 旁路包，以只读连接消费 `market.db`，以 Tushare 补足已批准的研究数据，只写 `S3_research.db`；EOD 只增加一个失败隔离、异步、不等待的启动点。T2 从验收后的 T1 提交建立新分支，以 T1 数据契约为唯一输入，建立独立 S3 回测包、一次性 81 组样本内扫描、冻结选参和一次样本外验证。

**Tech Stack:** Python 3.11、SQLite、pandas、现有 Tushare/FastAPI/vanilla JavaScript；零新增 npm/pip 依赖。

## Global Constraints

- 排期和闸口以 `docs/superpowers/plans/2026-07-28-guanlan-t0-t6-delivery-plan.md` 为准。
- T1、T2 各自必须先提交“动工确认”；没有用户准确回复“动工”，不得执行对应任务的代码步骤。
- T1 分支固定为 `codex/s3-stage1-data-clusters`。
- T2 仅在 T1 验收后，从 T1 已接受提交建立 `codex/s3-stage2-backtest`。
- 不自动合并 `main`，不部署，不切换生产。
- 不修改生产 S2 策略、风控规则、生产数据管道步骤、回测引擎核心、生产数据库结构。
- T1 研究钩子是唯一批准例外：只在生产收盘编排完成后异步启动；主任务不等待；异常只记日志。
- 生产 `market.db`、`screener.db` 必须使用 SQLite `mode=ro`。
- 所有研究数据只写 `S3_research.db`。
- 申万二级是 S3 唯一行业口径；任何 S3 代码不得读取或映射 `pool.industry`。
- 暖窗只用于 2015 年指标初始化，不产生信号、交易或收益。
- 财务数据严格按 `f_ann_date <= signal_date` 取当时已公开年报。
- 缺失值不编造；数据不完整时标记、计数并写入报告。
- 零新增 npm/pip 依赖。
- 禁止硬编码股票名单、指数点位、集群名单、回测收益或演示数据。
- T2 参数空间恰好 81 组；结果产生后不得改变选择规则、追加组合或重跑调参。
- 每个代码任务执行 RED → GREEN → 回归 → 小提交。

---

## 文件结构

### T1 创建

- `s3_research/__init__.py`：公开版本和只读接口。
- `s3_research/db.py`：研究库模式、生产只读连接和研究事务。
- `s3_research/sources.py`：已批准 Tushare 数据源的获取、规范化和原始表写入。
- `s3_research/audit.py`：覆盖率、缺失率、重叠一致性、时点数据审计。
- `s3_research/breadth.py`：NH250/NL250 和广度闸门。
- `s3_research/clusters.py`：申万二级时点成员、60 日新高和集群状态机。
- `s3_research/runner.py`：回填、增量、校验、原子发布和任务记录。
- `s3_research/hook.py`：单飞、异步、失败隔离的 EOD 启动器。
- `s3_research/read_model.py`：界面只读查询模型。
- `s3_research_cli.py`：研究数据获取、回填、审计和人工复算入口。
- `test_s3_research.py`：T1 数据、时点、防写、状态机和隔离测试。
- `test_s3_ui.py`：T1 API/界面只读契约测试。
- `docs/验收-T1-S3阶段一-20260728.md`：逐条验收和证据。

### T1 修改

- `eod.py`：只在现有收盘编排末端增加研究启动调用。
- `service.py`：增加两个只读 S3 API。
- `static/home.html`：增加批准的 `#today` 观察卡和 `#pool` 活跃集群区。

### T2 创建

- `s3_backtest/__init__.py`：公开回测版本。
- `s3_backtest/config.py`：预声明 ID、基准参数和 81 组固定空间。
- `s3_backtest/ledger.py`：运行声明、数据指纹、单次执行和结果封存。
- `s3_backtest/selection.py`：冻结选参规则。
- `s3_backtest/signals.py`：资格、评分、浅回踩和深回调信号。
- `s3_backtest/portfolio.py`：组合、风险预算、仓位上限和退出。
- `s3_backtest/engine_adapter.py`：只调用现有成交/费用/涨跌停/停牌帮助函数。
- `s3_backtest/s2_adapter.py`：同数据、同区间调用现有 S2 回测入口。
- `s3_backtest/metrics.py`：年度、交易、回撤、信号分布指标。
- `s3_backtest/report.py`：八项报告和局限性。
- `s3_backtest_cli.py`：样本内、锁参、样本外和报告命令。
- `test_s3_backtest.py`：规则、无前视、81 组、选参和组合测试。
- `reports/s3_backtest_2015_2025.md`：最终报告。
- `reports/s3_parameter_grid.csv`：81 组样本内结果。
- `reports/s3_backtest_trades.csv`：逐笔交易。
- `reports/s3_signal_distribution.csv`：年度、市况和 60/120 日分布。
- `docs/验收-T2-S3回测-20260728.md`：逐条验收和 G1 材料。

### 永不修改

- `market_data.py`
- `storage.py`
- `screener.db`
- `market.db`
- `rules_mid.py`
- `strategies/`
- `backtest.py`
- `backtest_mid.py`
- `backtest_defense.py`
- `params.py`
- `run_eod_trigger.sh`
- `schedule.sh`
- `requirements.txt`
- `static/pool.html`

如果实际实施证明必须修改上述文件，立即停止并报告，不得变通。

---

## T1：阶段一数据、集群和只读展示

### Task 1：动工授权与基线固定

**Files:**
- Read: `docs/superpowers/plans/2026-07-28-t1-start-confirmation.md`
- Read: `docs/superpowers/specs/2026-07-28-s3-cluster-trend-research-design.md`
- No code changes

**Interfaces:**
- Consumes: 用户回复中的准确授权词“动工”。
- Produces: 可审计的任务基线提交和干净工作树。

- [ ] **Step 1: 验证用户授权**

检查当前对话。若没有在 T1 动工确认之后出现用户回复“动工”，停止，不运行后续步骤。

- [ ] **Step 2: 验证 T0 没有未收口的界面修改**

Run:

```bash
git status --short --branch
git log -1 --oneline
git diff --name-only main...HEAD
```

Expected: 工作树无未提交文件；当前分支是 `codex/s3-stage1-data-clusters`；相对基线只有已批准的设计和计划文档。

- [ ] **Step 3: 装配隔离工作树的既有测试资源**

工作树不复制 Git 忽略的虚拟环境和 golden 文件。若 `.venv` 不存在，运行：

```bash
ln -s ../../.venv .venv
```

若 `reports` 不存在，运行：

```bash
mkdir -p reports
```

逐个补入只读测试基准：

```bash
ln -s ../../../reports/golden_p1_mid_before.json reports/golden_p1_mid_before.json
ln -s ../../../reports/golden_p1_short_before.json reports/golden_p1_short_before.json
ln -s ../../../reports/backtest_mid_trades_v5params.csv reports/backtest_mid_trades_v5params.csv
```

这些链接是被 Git 忽略的工作树测试资源，不进入提交。不得链接、复制或覆盖主线 `market.db`；真实研究命令显式使用 `../../market.db`。

- [ ] **Step 4: 复跑任务基线**

Run:

```bash
.venv/bin/python test_rules.py && .venv/bin/python test_rules_mid.py && .venv/bin/python test_web.py && .venv/bin/python test_regime.py
```

Expected: 8/8、20/20、834/834、6/6，命令退出 0。

- [ ] **Step 5: 记录任务基线**

把 `git rev-parse HEAD` 的值写入 T1 验收文档“基线提交”字段。此步骤只编辑验收文档，不改业务代码。

---

### Task 2：研究库模式与生产只读保护

**Files:**
- Create: `s3_research/__init__.py`
- Create: `s3_research/db.py`
- Create: `test_s3_research.py`

**Interfaces:**
- Produces: `FORMULA_VERSION = "s3-breadth-cluster-v1"`
- Produces: `default_research_db() -> str`
- Produces: `default_market_db() -> str`
- Produces: `connect_production_ro(path: str) -> sqlite3.Connection`
- Produces: `connect_research(path: str, readonly: bool = False) -> sqlite3.Connection`
- Produces: `init_research_db(path: str) -> None`
- Produces: `research_transaction(path: str) -> ContextManager[sqlite3.Connection]`

- [ ] **Step 1: 写生产库不可写的失败测试**

在 `test_s3_research.py` 建立临时生产库，调用待实现接口后尝试写表：

```python
class ReadOnlyBoundaryTests(unittest.TestCase):
    def test_production_connection_rejects_writes(self):
        with tempfile.TemporaryDirectory() as td:
            path = os.path.join(td, "market.db")
            sqlite3.connect(path).execute("CREATE TABLE market_daily(x)").connection.commit()
            con = connect_production_ro(path)
            with self.assertRaises(sqlite3.OperationalError):
                con.execute("CREATE TABLE forbidden(x)")

    def test_research_schema_never_opens_production_path_for_write(self):
        with tempfile.TemporaryDirectory() as td:
            market = os.path.join(td, "market.db")
            sqlite3.connect(market).close()
            with self.assertRaises(ValueError):
                init_research_db(market)
```

- [ ] **Step 2: 运行测试并确认 RED**

Run: `.venv/bin/python -m unittest test_s3_research.ReadOnlyBoundaryTests -v`

Expected: FAIL，原因是 `s3_research.db` 尚不存在。

- [ ] **Step 3: 实现只读 URI 和研究库保护**

`s3_research/db.py` 的核心必须是：

```python
PRODUCTION_DB_NAMES = {"market.db", "screener.db"}

def connect_production_ro(path):
    uri = Path(path).resolve().as_uri() + "?mode=ro"
    con = sqlite3.connect(uri, uri=True, timeout=30)
    con.row_factory = sqlite3.Row
    return con

def _assert_research_path(path):
    if Path(path).name in PRODUCTION_DB_NAMES:
        raise ValueError("S3 research must not write a production database")

def connect_research(path, readonly=False):
    _assert_research_path(path)
    if readonly:
        return connect_production_ro(path)
    con = sqlite3.connect(path, timeout=30)
    con.row_factory = sqlite3.Row
    return con
```

`s3_research/__init__.py` 定义唯一公式版本：

```python
FORMULA_VERSION = "s3-breadth-cluster-v1"
```

`default_research_db()` 返回仓库根的 `S3_research.db`，`default_market_db()` 返回仓库根的 `market.db`。测试通过显式路径注入，不改这两个默认值。

`init_research_db` 只创建设计文档列出的原始表、`breadth_daily`、`cluster_daily`、`cluster_members_daily`、`source_audit`、`job_runs`。所有派生表主键必须包含日期；原始表按来源自然键去重。

- [ ] **Step 4: 增加模式测试**

断言研究库包含全部表，且临时 `market.db`、`screener.db` 的 `sqlite_master` 在调用前后完全一致。

- [ ] **Step 5: 运行 GREEN**

Run: `.venv/bin/python -m unittest test_s3_research.ReadOnlyBoundaryTests -v`

Expected: PASS。

- [ ] **Step 6: 提交**

```bash
git add s3_research/__init__.py s3_research/db.py test_s3_research.py
git commit -m "research: add isolated S3 database boundary"
```

---

### Task 3：已批准数据源和原始表写入

**Files:**
- Create: `s3_research/sources.py`
- Modify: `test_s3_research.py`

**Interfaces:**
- Produces: `SourceSpec(name, start, end, fields, table)`
- Produces: `production_client() -> Any`
- Produces: `fetch_source(pro, spec: SourceSpec, **filters) -> pandas.DataFrame`
- Produces: `normalize_source(spec, frame) -> list[dict]`
- Produces: `upsert_source_rows(con, spec, rows) -> int`
- Consumes: 注入的 Tushare `pro` 客户端；测试禁止联网。

- [ ] **Step 1: 写来源和字段冻结测试**

```python
def test_source_specs_are_exactly_frozen(self):
    specs = source_specs()
    self.assertEqual(specs["daily_vol"].fields,
                     ("trade_date", "ts_code", "vol"))
    self.assertEqual((specs["warm_daily"].start, specs["warm_daily"].end),
                     ("2013-12-01", "2014-12-31"))
    self.assertEqual(specs["sw_daily"].start, "2014-01-01")
    self.assertEqual(specs["stock_st"].start, "2016-01-01")
    self.assertEqual((specs["annual_financials"].start,
                      specs["annual_financials"].end), ("2012", "2024"))
```

再用假的 `pro` 对象记录调用，断言只调用 `daily`、`adj_factor`、`index_classify`、`index_member_all`、`sw_daily`、`stock_basic`、`namechange`、`stock_st` 和年度财务接口。

- [ ] **Step 2: 运行 RED**

Run: `.venv/bin/python -m unittest test_s3_research.SourceTests -v`

Expected: FAIL，原因是来源接口尚不存在。

- [ ] **Step 3: 实现来源规格**

把以下范围写成不可变 `SourceSpec`，不从 UI 或回测参数覆盖：

```python
SPECS = {
    "warm_daily": SourceSpec("daily", "2013-12-01", "2014-12-31",
        ("trade_date", "ts_code", "open", "high", "low", "close", "amount", "vol"),
        "warm_daily"),
    "warm_adj": SourceSpec("adj_factor", "2013-12-01", "2014-12-31",
        ("trade_date", "ts_code", "adj_factor"), "warm_adj_factor"),
    "daily_vol": SourceSpec("daily", "2015-01-01", "2025-12-31",
        ("trade_date", "ts_code", "vol"), "daily_vol"),
    "sw_industry": SourceSpec("index_classify", None, None,
        ("index_code", "industry_code", "industry_name", "parent_code",
         "is_pub", "src"), "sw_industry"),
    "sw_membership": SourceSpec("index_member_all", None, None,
        ("l2_code", "l2_name", "ts_code", "in_date", "out_date", "is_new"),
        "sw_membership"),
    "sw_daily": SourceSpec("sw_daily", "2014-01-01", None,
        ("ts_code", "trade_date", "open", "high", "low", "close"),
        "sw_index_daily"),
    "stock_basic": SourceSpec("stock_basic", None, None,
        ("ts_code", "list_date", "delist_date", "list_status"),
        "stock_basic"),
    "namechange": SourceSpec("namechange", None, None,
        ("ts_code", "name", "start_date", "end_date", "ann_date"),
        "stock_namechange"),
    "stock_st": SourceSpec("stock_st", "2016-01-01", "2025-12-31",
        ("ts_code", "name", "trade_date", "type", "type_name"),
        "stock_st"),
    "annual_financials": SourceSpec("income", "2012-01-01", "2024-12-31",
        ("ts_code", "f_ann_date", "end_date", "report_type", "revenue",
         "n_income", "update_flag"), "annual_financials"),
}
```

日期统一存 ISO；代码保留 Tushare 后缀；数值缺失存 `NULL`。`None` 日期表示接口自身的全历史或“截至运行日”，实际范围必须写入 `source_audit`。

`production_client()` 只返回现有 `data._tushare_pro()` 句柄，不创建第二套鉴权或配置。

- [ ] **Step 4: 实现幂等写入和断点元数据**

`upsert_source_rows` 使用每个 `SourceSpec` 声明的自然主键执行 SQLite UPSERT。成功提交后写 `source_audit` 的最后成功日期和行数；空返回不更新成功进度。

- [ ] **Step 5: 运行 GREEN**

Run: `.venv/bin/python -m unittest test_s3_research.SourceTests -v`

Expected: PASS，且假客户端证明测试零联网。

- [ ] **Step 6: 提交**

```bash
git add s3_research/sources.py test_s3_research.py
git commit -m "research: add frozen S3 source ingestion"
```

---

### Task 4：时点规则和数据审计

**Files:**
- Create: `s3_research/audit.py`
- Modify: `test_s3_research.py`

**Interfaces:**
- Produces: `members_on(con, l2_code: str, trade_date: str) -> set[str]`
- Produces: `is_st_on(con, ts_code: str, trade_date: str) -> bool | None`
- Produces: `latest_public_annuals(con, ts_code: str, signal_date: str) -> tuple[Row | None, Row | None]`
- Produces: `quality_floor_fails(current, previous) -> bool | None`
- Produces: `audit_source(con, source_name: str) -> dict`
- Produces: `compare_overlap(prod_ro, research_ro, dates: list[str]) -> dict`

- [ ] **Step 1: 写时点边界测试**

```python
def test_membership_uses_historical_interval(self):
    seed_member("801010.SI", "000001.SZ", "2015-01-01", "2018-06-30")
    self.assertIn("000001.SZ", members_on(self.con, "801010.SI", "2018-06-30"))
    self.assertNotIn("000001.SZ", members_on(self.con, "801010.SI", "2018-07-01"))

def test_financials_obey_actual_publication_date(self):
    seed_financial("000001.SZ", f_ann_date="2020-03-20", end_date="2019-12-31")
    seed_financial("000001.SZ", f_ann_date="2021-03-25", end_date="2020-12-31")
    current, previous = latest_public_annuals(self.con, "000001.SZ", "2021-03-24")
    self.assertEqual(current["end_date"], "2019-12-31")
```

增加以下断言：

- 2016 年前 `namechange` 命中 ST 区间时返回 `True`；
- 2016 年后优先使用 `stock_st`；
- 净利润为负且营收同比下降才返回 `True`；
- 任一必需财务值缺失返回 `None`，不假定通过或失败；
- 成分历史缺口和财务缺失写入审计计数。

- [ ] **Step 2: 运行 RED**

Run: `.venv/bin/python -m unittest test_s3_research.PointInTimeTests -v`

Expected: FAIL。

- [ ] **Step 3: 实现时点查询**

成员条件固定为：

```sql
in_date <= :trade_date
AND (out_date IS NULL OR out_date = '' OR out_date >= :trade_date)
```

财务查询固定为 `f_ann_date <= :signal_date`，先选最新年报，再选其上一年度年报。报告展示时四舍五入，选择和过滤使用数据库原始精度。

- [ ] **Step 4: 实现覆盖和重叠审计**

`compare_overlap` 只抽取用户批准的重叠日期，比较原始 OHLC、amount 和复权因子；返回：

```python
{
    "sample_dates": ["2014-12-29", "2014-12-30", "2014-12-31"],
    "compared_rows": 0,
    "equal_rows": 0,
    "mismatch_rows": 0,
    "missing_left": 0,
    "missing_right": 0,
    "consistency_pct": None,
}
```

不得因不一致而写生产库；只把结果写 `source_audit`。

- [ ] **Step 5: 运行 GREEN**

Run: `.venv/bin/python -m unittest test_s3_research.PointInTimeTests -v`

Expected: PASS。

- [ ] **Step 6: 提交**

```bash
git add s3_research/audit.py test_s3_research.py
git commit -m "research: enforce point-in-time S3 data rules"
```

---

### Task 5：NH250/NL250 和广度闸门

**Files:**
- Create: `s3_research/breadth.py`
- Modify: `test_s3_research.py`

**Interfaces:**
- Produces: `load_adjusted_extremes(prod_ro, warm_ro, start: str, end: str) -> Iterable[dict]`
- Produces: `compute_breadth(rows: Iterable[dict]) -> list[dict]`
- Produces: `apply_breadth_gate(rows: list[dict], prior: bool = False) -> list[dict]`

- [ ] **Step 1: 写 250 日和闸门失败测试**

构造 251 个交易日、两只股票：

```python
def test_nh_nl_need_250_valid_rows_and_include_current_day(self):
    rows = fixture_adjusted_bars(days=251)
    out = compute_breadth(rows)
    self.assertEqual(out[248]["eligible_n"], 0)
    self.assertEqual(out[249]["eligible_n"], 2)
    self.assertEqual((out[250]["nh250"], out[250]["nl250"]), (1, 1))

def test_gate_open_close_and_zero_hysteresis(self):
    rows = [{"net_5d": 1, "net": 0}, {"net_5d": 2, "net": 1},
            {"net_5d": 3, "net": 0}, {"net_5d": 0, "net": -1},
            {"net_5d": -1, "net": -1}]
    states = [r["gate_open"] for r in apply_breadth_gate(rows)]
    self.assertEqual(states, [False, False, True, True, False])
```

- [ ] **Step 2: 运行 RED**

Run: `.venv/bin/python -m unittest test_s3_research.BreadthTests -v`

Expected: FAIL。

- [ ] **Step 3: 实现后复权滚动极值**

使用 `high × adj_factor` 判 NH250，使用 `low × adj_factor` 判 NL250。每只股票按自身有效交易日计数；少于 250 根不进入分母。暖窗行与生产行只在内存查询结果中拼接，不复制生产行情字段到研究库。

核心判定必须等价于：

```python
eligible = valid_count >= 250
is_nh = eligible and adjusted_high >= rolling_high_250
is_nl = eligible and adjusted_low <= rolling_low_250
```

- [ ] **Step 4: 实现闸门状态机**

当前五日和大于 0 且最近三日每日净值均不小于 0时开；当前五日和小于 0 时关；等于 0 沿用前态；无前态默认关。

- [ ] **Step 5: 运行 GREEN**

Run: `.venv/bin/python -m unittest test_s3_research.BreadthTests -v`

Expected: PASS。

- [ ] **Step 6: 提交**

```bash
git add s3_research/breadth.py test_s3_research.py
git commit -m "research: compute S3 breadth and gate"
```

---

### Task 6：申万二级集群状态机

**Files:**
- Create: `s3_research/clusters.py`
- Modify: `test_s3_research.py`

**Interfaces:**
- Produces: `constituent_hits_10d(con, breadth_hits, trade_date) -> list[dict]`
- Produces: `industry_highs_60d(con, trade_date) -> dict[str, bool | None]`
- Produces: `compute_cluster_states(dates, hit_counts, industry_highs, threshold=3) -> list[dict]`

- [ ] **Step 1: 写集群失败测试**

```python
def test_cluster_activates_on_distinct_constituents_or_index(self):
    hits = fixture_hits(distinct_codes=3, within_days=10)
    out = compute_cluster_states(self.dates, hits, {}, threshold=3)
    self.assertTrue(out[-1]["active"])

def test_cluster_exits_after_twenty_consecutive_inactive_days(self):
    out = fixture_active_then_empty_days(20)
    self.assertTrue(out[18]["active"])
    self.assertFalse(out[19]["active"])

def test_missing_index_never_changes_taxonomy(self):
    out = compute_cluster_states(self.dates, self.stock_hits,
                                 {"801010.SI": None}, threshold=3)
    self.assertEqual(out[-1]["activation_arm"], "constituents")
```

增加一项源码边界测试：

```python
for path in Path("s3_research").glob("*.py"):
    self.assertNotIn("pool.industry", path.read_text(encoding="utf-8"))
```

- [ ] **Step 2: 运行 RED**

Run: `.venv/bin/python -m unittest test_s3_research.ClusterTests -v`

Expected: FAIL。

- [ ] **Step 3: 实现最近十日去重命中**

命中数是最近 10 个交易日内命中过 NH250 的不重复时点成分股数，不是命中事件行数。行业指数 60 日新高使用包含当日的 60 根有效行业指数收盘。

- [ ] **Step 4: 实现激活和移出**

```python
trigger = distinct_hit_count >= threshold or industry_nh60 is True
inactive_streak = 0 if trigger else previous_inactive_streak + 1
active = True if trigger else previous_active and inactive_streak < 20
```

行业指数缺失时 `industry_nh60=None`，只允许成分股臂决定，不得替换行业代码。

- [ ] **Step 5: 运行 GREEN**

Run: `.venv/bin/python -m unittest test_s3_research.ClusterTests -v`

Expected: PASS。

- [ ] **Step 6: 提交**

```bash
git add s3_research/clusters.py test_s3_research.py
git commit -m "research: derive point-in-time S3 clusters"
```

---

### Task 7：回填、增量和原子发布

**Files:**
- Create: `s3_research/runner.py`
- Create: `s3_research_cli.py`
- Modify: `test_s3_research.py`

**Interfaces:**
- Produces: `run_backfill(research_db, market_db, start, end, pro) -> dict`
- Produces: `run_incremental(research_db, market_db, trade_date, pro) -> dict`
- Produces: `validate_batch(payload) -> list[str]`
- Produces: CLI `fetch-sources`, `backfill`, `incremental`, `audit`, `verify`

- [ ] **Step 1: 写坏批次不可覆盖测试**

```python
def test_empty_or_partial_batch_preserves_last_good_rows(self):
    seed_published_breadth("2025-01-02", nh=10, nl=2)
    result = run_incremental(self.research_db, self.market_db,
                             "2025-01-02", EmptySource())
    self.assertFalse(result["published"])
    self.assertEqual(read_breadth("2025-01-02")["nh250"], 10)

def test_exception_rolls_back_derived_tables(self):
    with self.assertRaisesRegex(RuntimeError, "injected"):
        publish_with_injected_failure(self.research_db)
    self.assertEqual(snapshot_derived_tables(), self.before)
```

再测同一日期重复运行不增加行数，且 `job_runs` 记录 `success/failed/rejected`。

- [ ] **Step 2: 运行 RED**

Run: `.venv/bin/python -m unittest test_s3_research.RunnerTests -v`

Expected: FAIL。

- [ ] **Step 3: 实现先算后验再事务发布**

执行顺序固定：

```python
raw = load_inputs(research_db, market_db, start, end)
derived = derive_all(raw, formula_version=FORMULA_VERSION)
errors = validate_batch(derived)
if errors:
    record_rejected(research_db, start, end, errors)
    return {"published": False, "errors": errors}
with research_transaction(research_db) as con:
    replace_requested_date_range(con, start, end, derived)
    record_success(con, start, end, FORMULA_VERSION,
                   {name: len(rows) for name, rows in derived.items()})
return {
    "published": True,
    "errors": [],
    "row_counts": {name: len(rows) for name, rows in derived.items()},
}
```

任何下载、计算或校验异常只更新独立 `job_runs` 失败记录，不改上一份派生数据。

- [ ] **Step 4: 实现 CLI**

命令必须显式传研究库和生产库，默认值分别为仓库根的 `S3_research.db` 与 `market.db`。CLI 启动时打印：

- 生产库 URI 为只读；
- 研究库路径；
- 任务范围；
- 公式版本。

- [ ] **Step 5: 运行 GREEN**

Run: `.venv/bin/python -m unittest test_s3_research.RunnerTests -v`

Expected: PASS。

- [ ] **Step 6: 提交**

```bash
git add s3_research/runner.py s3_research_cli.py test_s3_research.py
git commit -m "research: add atomic S3 backfill runner"
```

---

### Task 8：单飞、异步、失败隔离的 EOD 钩子

**Files:**
- Create: `s3_research/hook.py`
- Modify: `eod.py`
- Modify: `test_s3_research.py`

**Interfaces:**
- Produces: `try_start_incremental(trade_date: str) -> bool`
- Produces: `_run_isolated(trade_date: str) -> None`
- Consumes: `runner.run_incremental`

- [ ] **Step 1: 写不等待、不抛错、单飞测试**

```python
def test_hook_returns_before_slow_runner_finishes(self):
    started = time.monotonic()
    self.assertTrue(try_start_incremental("2025-01-02"))
    self.assertLess(time.monotonic() - started, 0.2)

def test_hook_is_singleflight(self):
    self.assertTrue(try_start_incremental("2025-01-02"))
    self.assertFalse(try_start_incremental("2025-01-02"))

def test_runner_exception_never_escapes_thread(self):
    self.runner.side_effect = RuntimeError("injected")
    self.assertTrue(try_start_incremental("2025-01-02"))
    wait_for_thread()
    self.assertIn("injected", captured_log())
```

使用 `inspect.getsource` 断言 `run_eod_scan` 不含 `s3_research`，启动点只存在于 `_generate_daily` 的生产动作之后。

- [ ] **Step 2: 运行 RED**

Run: `.venv/bin/python -m unittest test_s3_research.HookTests -v`

Expected: FAIL。

- [ ] **Step 3: 实现隔离线程**

`s3_research/hook.py`：

```python
_lock = threading.Lock()

def try_start_incremental(trade_date):
    if not _lock.acquire(blocking=False):
        return False
    threading.Thread(target=_run_isolated, args=(trade_date,),
                     daemon=True, name="s3-research").start()
    return True

def _run_isolated(trade_date):
    try:
        runner.run_incremental(default_research_db(), default_market_db(),
                               trade_date, sources.production_client())
    except Exception:
        logger.exception("S3 research incremental failed; production unaffected")
    finally:
        _lock.release()
```

- [ ] **Step 4: 增加唯一 EOD 启动点**

在 `_generate_daily` 现有生产步骤全部完成后增加：

```python
try:
    from s3_research.hook import try_start_incremental
    try_start_incremental(trade_date)
except Exception as exc:
    logger.warning("S3研究钩子启动失败(忽略,不影响日报/EOD):%s", exc)
```

不得修改 `_market_daily_job`、`run_eod_scan`、调度器和现有钩子的顺序。

- [ ] **Step 5: 运行 GREEN 和 golden master**

Run:

```bash
.venv/bin/python -m unittest test_s3_research.HookTests -v
.venv/bin/python golden_master.py --mode mid --out /tmp/s3_mid_golden.json
cmp /tmp/s3_mid_golden.json reports/golden_p1_mid_before.json
```

Expected: 测试 PASS；`cmp` 退出 0。

- [ ] **Step 6: 提交**

```bash
git add s3_research/hook.py eod.py test_s3_research.py
git commit -m "research: isolate S3 EOD hook"
```

---

### Task 9：只读 API

**Files:**
- Create: `s3_research/read_model.py`
- Modify: `service.py`
- Modify: `test_s3_ui.py`

**Interfaces:**
- Produces: `latest_summary(path: str) -> dict | None`
- Produces: `active_clusters(path: str, trade_date: str | None = None) -> list[dict]`
- Produces: `GET /api/s3/summary`
- Produces: `GET /api/s3/clusters`

- [ ] **Step 1: 写 API 只读与空态测试**

```python
def test_summary_returns_real_latest_row(self):
    seed_summary("2025-01-02", nh=12, nl=4, net_5d=21, gate_open=1)
    out = service.api_s3_summary()
    self.assertEqual(out["summary"]["trade_date"], "2025-01-02")
    self.assertEqual(out["summary"]["nh250"], 12)

def test_missing_research_db_returns_honest_empty_state(self):
    service.s3_read_model.RESEARCH_DB_PATH = self.missing
    out = service.api_s3_summary()
    self.assertTrue(out["ok"])
    self.assertIsNone(out["summary"])
```

断言 API 模块以 `mode=ro` 打开研究库，且不存在 POST/PUT/DELETE S3 路由。

- [ ] **Step 2: 运行 RED**

Run: `.venv/bin/python -m unittest test_s3_ui.S3ApiTests -v`

Expected: FAIL。

- [ ] **Step 3: 实现只读模型和路由**

路由只负责返回：

```python
{"ok": True, "summary": latest_summary(s3_read_model.RESEARCH_DB_PATH)}
{"ok": True, "trade_date": date,
 "clusters": active_clusters(s3_read_model.RESEARCH_DB_PATH, date)}
```

集群行包含 `l2_code, l2_name, hit_count_10d, threshold, industry_nh60, active_days, members`。字段缺失返回 `None`，不计算替代值。

- [ ] **Step 4: 运行 GREEN**

Run: `.venv/bin/python -m unittest test_s3_ui.S3ApiTests -v`

Expected: PASS。

- [ ] **Step 5: 提交**

```bash
git add s3_research/read_model.py service.py test_s3_ui.py
git commit -m "research: expose read-only S3 observations"
```

---

### Task 10：M3 首页只读展示

**Files:**
- Modify: `static/home.html`
- Modify: `test_s3_ui.py`
- Test: `test_m3_home.py`

**Interfaces:**
- Consumes: `/api/s3/summary`, `/api/s3/clusters`
- Produces: `loadS3Observation()`, `renderS3Summary(payload)`, `renderS3Clusters(payload)`

- [ ] **Step 1: 写界面契约失败测试**

```python
def test_today_has_non_trading_observation_card(self):
    self.assertIn("S3 观察指标 · 不影响交易", HOME)
    self.assertIn('fetch("/api/s3/summary")', HOME)
    for label in ("NH250", "NL250", "五日净值"):
        self.assertIn(label, HOME)

def test_pool_has_read_only_cluster_group(self):
    self.assertIn('fetch("/api/s3/clusters")', HOME)
    self.assertIn("活跃集群", HOME)
    self.assertNotIn("/api/s3/transition", HOME)
```

再断言 `static/pool.html` 相对基线无差异，页面中没有硬编码六位股票代码，没有新增 `2×ATR` 和“板块联动”计算函数。

- [ ] **Step 2: 运行 RED**

Run: `.venv/bin/python -m unittest test_s3_ui.S3UiContractTests -v`

Expected: FAIL。

- [ ] **Step 3: 实现 `#today` 观察卡**

现有生产闸门卡不改。新卡只显示 API 返回的研究闸门、NH250、NL250、五日净值、确认进度和数据日期。`summary=None` 时显示“研究数据尚未生成”，不得显示 0。

- [ ] **Step 4: 实现 `#pool` 集群区**

在现有状态分组上方显示活跃申万二级行业、`hit_count_10d/threshold`、行业指数 60 日新高、活跃天数和真实成员。`clusters=[]` 时显示“当前没有活跃集群”。

不增加按钮、状态迁移、重筛、买卖或删除事件。旧 `/pool` 入口仍只在“我的 → 工程模式”。

- [ ] **Step 5: 运行 GREEN 和 T0 回归**

Run:

```bash
.venv/bin/python -m unittest test_s3_ui -v
.venv/bin/python test_m3_home.py
git diff --exit-code 0a7be53 -- static/pool.html
```

Expected: 全部 PASS；旧 `static/pool.html` 零差异。

- [ ] **Step 6: 提交**

```bash
git add static/home.html test_s3_ui.py
git commit -m "ui: show read-only S3 breadth and clusters"
```

---

### Task 11：真实数据回填、人工复算和 T1 验收

**Files:**
- Create: `docs/验收-T1-S3阶段一-20260728.md`
- Modify only if tests expose a defect: T1 files listed above

**Interfaces:**
- Consumes: T1 CLI and真实 Tushare/生产只读库。
- Produces: 完整 `S3_research.db`、数据审计、3 日宽度复算、2 集群复算和界面证据。

- [ ] **Step 1: 获取冻结外部数据**

Run:

```bash
.venv/bin/python s3_research_cli.py fetch-sources --research-db S3_research.db
```

Expected: 只写 `S3_research.db`；输出每个来源的实际起止日期、行数、缺失率和断点状态。

- [ ] **Step 2: 运行全量派生回填**

Run:

```bash
.venv/bin/python s3_research_cli.py backfill --research-db S3_research.db --market-db ../../market.db --start 2015-01-01 --end 2025-12-31
```

Expected: 生产库日志明确 `mode=ro`；暖窗不产生 2014 年信号；派生表覆盖 2015—2025。

- [ ] **Step 3: 运行数据审计**

Run:

```bash
.venv/bin/python s3_research_cli.py audit --research-db S3_research.db --market-db ../../market.db
```

把暖窗覆盖率、重叠一致性、`daily.vol` 覆盖率、成分历史覆盖、财务缺失率和 2015 ST 局限写入验收文档。

- [ ] **Step 4: 人工复算**

Run:

```bash
.venv/bin/python s3_research_cli.py verify --research-db S3_research.db --market-db ../../market.db --breadth-dates 2015-06-30,2018-10-19,2022-04-27 --cluster-count 2
```

输出每只参与复算股票的 250 日窗口边界、调整后高低价、命中判断，以及两个集群的 10 日成员命中和 20 日移出路径。把结果逐条贴入验收文档。

- [ ] **Step 5: 注入故障验证**

Run: `.venv/bin/python -m unittest test_s3_research.HookTests test_s3_research.RunnerTests -v`

Expected: 下载失败、SQLite 写失败、派生异常都不向生产 EOD 抛出，也不覆盖好数据。

- [ ] **Step 6: 浏览器验收**

在隔离端口启动工作树服务，验证：

- `#today` 真实数据态和空态；
- `#pool` 真实集群态和空态；
- 桌面宽屏和小于 860px 手机布局；
- 所有 S3 区块只读；
- 生产闸门和旧股票池分组未改变。

把截图路径和观察结果写入验收文档。

- [ ] **Step 7: 完整回归**

Run:

```bash
.venv/bin/python test_rules.py && .venv/bin/python test_rules_mid.py && .venv/bin/python test_web.py && .venv/bin/python test_regime.py
.venv/bin/python test_m3_home.py
.venv/bin/python -m unittest test_s3_research test_s3_ui -v
git diff --check
```

Expected: 全部退出 0。

- [ ] **Step 8: 禁区和依赖审计**

Run:

```bash
git diff --name-only 7888f8e...HEAD
git diff -- requirements.txt market_data.py storage.py rules_mid.py backtest.py backtest_mid.py params.py static/pool.html
```

Expected: 第一条只含 T1 允许文件；第二条零输出。`eod.py` 只含批准的窄启动点。

- [ ] **Step 9: 提交验收文档并停止**

```bash
git add docs/验收-T1-S3阶段一-20260728.md
git commit -m "docs: record T1 S3 stage-one acceptance"
```

向用户提交分支、提交、证据和未合并状态。等待 T1 验收；不得创建 T2 分支。

---

## T2：S3 全量回测

### Task 12：T2 动工确认、分支和 T1 数据指纹

**Files:**
- Create only after T1 acceptance: T2 动工确认文档
- No business code

**Interfaces:**
- Consumes: T1 用户验收、T2 用户回复“动工”、T1 数据库指纹。
- Produces: `codex/s3-stage2-backtest` 和不可变 T2 基线。

- [ ] **Step 1: 验证三项授权**

必须同时存在：

1. 用户明确验收 T1；
2. 已提交一页 T2 动工确认；
3. 用户在该确认后回复“动工”。

缺一项立即停止。

- [ ] **Step 2: 从已接受 T1 提交建分支**

Run:

```bash
git rev-parse HEAD
git switch -c codex/s3-stage2-backtest
```

Expected: 基线恰好是用户已接受的 T1 提交。

- [ ] **Step 3: 固定输入指纹**

对生产库只读表范围、研究原始表、派生表、公式版本和 T1 提交生成 SHA-256 清单。清单写入 T2 验收文档；不得修改数据后沿用旧指纹。

---

### Task 13：预声明配置、81 组和冻结选参

**Files:**
- Create: `s3_backtest/__init__.py`
- Create: `s3_backtest/config.py`
- Create: `s3_backtest/selection.py`
- Create: `s3_backtest/ledger.py`
- Create: `test_s3_backtest.py`

**Interfaces:**
- Produces: `PREDECLARATION_ID = "s3-cluster-v0.1-2026-07-28"`
- Produces: `BASELINE_PARAMS`
- Produces: `parameter_grid() -> tuple[Params, ...]`
- Produces: `select_params(results: Sequence[Result]) -> Selection`
- Produces: `claim_run(con, predeclaration_id, input_hash, grid_hash) -> str`
- Produces: `seal_run(con, run_id, result_hash) -> None`

- [ ] **Step 1: 写 81 组和选择纪律测试**

```python
def test_grid_is_exactly_eighty_one_unique_combinations(self):
    grid = parameter_grid()
    self.assertEqual(len(grid), 81)
    self.assertEqual(len(set(grid)), 81)

def test_selection_filters_drawdown_then_excess_then_drawdown(self):
    rows = [
        result("a", max_dd=24.9, annual_excess=9.0),
        result("b", max_dd=20.0, annual_excess=9.0),
        result("c", max_dd=25.0, annual_excess=99.0),
    ]
    self.assertEqual(select_params(rows).params_id, "b")

def test_no_passing_combination_uses_predeclared_baseline(self):
    rows = [result("a", max_dd=25.0, annual_excess=20.0)]
    self.assertTrue(select_params(rows).tuning_failed)
    self.assertEqual(select_params(rows).params, BASELINE_PARAMS)
```

增加：

- 最大回撤必须严格 `<25`，`25.0` 不通过；
- 比较使用未四舍五入原始值；
- 年化超额和回撤都完全相同时抛 `SelectionAmbiguityError`，停止等用户裁决，不自行增加第三排序键；
- 已封存的预声明 ID 拒绝第二次 `claim_run`；
- 参数空间哈希变化时拒绝运行。

- [ ] **Step 2: 运行 RED**

Run: `.venv/bin/python -m unittest test_s3_backtest.SelectionTests -v`

Expected: FAIL。

- [ ] **Step 3: 实现固定参数空间**

```python
BASELINE_PARAMS = Params(cluster_hits=3, initial_stop_atr=2.0,
                         chandelier_atr=3.0, weights="base")

def parameter_grid():
    return tuple(Params(*values) for values in itertools.product(
        (2, 3, 4), (2.0, 2.5, 3.0), (2.5, 3.0, 3.5),
        ("base", "rs_heavy", "cluster_heavy")))
```

- [ ] **Step 4: 实现冻结选择**

```python
eligible = [r for r in results if r.max_dd < 25.0]
if not eligible:
    return Selection(BASELINE_PARAMS, tuning_failed=True)
best_excess = max(r.annual_excess for r in eligible)
top = [r for r in eligible if r.annual_excess == best_excess]
best_dd = min(r.max_dd for r in top)
finalists = [r for r in top if r.max_dd == best_dd]
if len(finalists) != 1:
    raise SelectionAmbiguityError(finalists)
return Selection(finalists[0].params, tuning_failed=False)
```

- [ ] **Step 5: 实现运行封存**

同一预声明允许中断后继续未完成的确定性分片，但不得删除已产生结果或从头重跑。样本内 81 组全部完成并封存后，任何第二次执行直接失败。

- [ ] **Step 6: 运行 GREEN 并提交**

Run: `.venv/bin/python -m unittest test_s3_backtest.SelectionTests -v`

```bash
git add s3_backtest/__init__.py s3_backtest/config.py s3_backtest/selection.py s3_backtest/ledger.py test_s3_backtest.py
git commit -m "backtest: freeze S3 parameter selection"
```

---

### Task 14：资格、机械排序和无前视信号

**Files:**
- Create: `s3_backtest/signals.py`
- Modify: `test_s3_backtest.py`

**Interfaces:**
- Produces: `eligible_candidates(snapshot: Snapshot, params: Params) -> list[Candidate]`
- Produces: `score_candidate(candidate, weights) -> float`
- Produces: `shallow_entry(candidate, bars) -> Signal | None`
- Produces: `deep_entry(candidate, bars, volumes) -> Signal | None`

- [ ] **Step 1: 写资格和无前视测试**

测试必须覆盖：

- 活跃集群；
- 20 日日均成交额不低于 3 亿；
- NH250 或距 NH250 不超过 5%；
- 120 日 RS 前 15%；
- ST、上市不足一年、财务质地底线；
- 集群外强票只进 B 观察池；
- 信号日之后追加任意行情，不改变历史信号。

```python
def test_future_rows_do_not_change_past_signal(self):
    before = eligible_candidates(snapshot_through("2020-06-30"), BASELINE_PARAMS)
    after = eligible_candidates(snapshot_with_future("2020-06-30", "2021-01-01"),
                                BASELINE_PARAMS)
    self.assertEqual(before, after)
```

- [ ] **Step 2: 运行 RED**

Run: `.venv/bin/python -m unittest test_s3_backtest.SignalTests -v`

Expected: FAIL。

- [ ] **Step 3: 实现 0–100 分量和权重**

RS、集群强度使用横截面百分位；首捕满足为 100，否则 0；平台价为 100，`平台+2.5ATR` 为 0，中间线性插值。三组权重固定为 `50/30/10/10`、`60/20/10/10`、`40/40/10/10`。

- [ ] **Step 4: 实现两类入场**

浅回踩要求捕获后真实发生回踩，收盘位于 `[平台, 平台+2.5ATR]`。深回调要求峰值至低点 `4–7ATR`、当日量不低于 20 日均量 1.5 倍并重新站上平台。两者都只在下一可交易日开盘下单。

- [ ] **Step 5: 运行 GREEN 并提交**

Run: `.venv/bin/python -m unittest test_s3_backtest.SignalTests -v`

```bash
git add s3_backtest/signals.py test_s3_backtest.py
git commit -m "backtest: implement S3 point-in-time signals"
```

---

### Task 15：组合、成交和退出

**Files:**
- Create: `s3_backtest/engine_adapter.py`
- Create: `s3_backtest/portfolio.py`
- Modify: `test_s3_backtest.py`

**Interfaces:**
- Produces: `fill_buy(code: str, signal_i: int, bars: pandas.DataFrame) -> Fill | None`
- Produces: `fill_sell(code: str, exit_i: int, bars: pandas.DataFrame) -> Fill`
- Produces: `size_position(equity, buy, stop, cash, limits) -> int`
- Produces: `Portfolio.process_day(date, candidates, bars) -> DayResult`

- [ ] **Step 1: 写成交和风险测试**

覆盖：

- 信号次日开盘；
- 一字涨停买不进；
- 一字跌停顺延卖出；
- 停牌顺延；
- 双边费用与现有引擎一致；
- 1% 风险倒算；
- 单票 30%、行业 30%、总仓 60%；
- 沪深 300 低于 MA60 时总仓 30%；
- 账户回撤 8% 停新仓；
- 同日按分数从高到低成交。

- [ ] **Step 2: 运行 RED**

Run: `.venv/bin/python -m unittest test_s3_backtest.PortfolioTests -v`

Expected: FAIL。

- [ ] **Step 3: 只调用现有成交帮助函数**

`engine_adapter.py` 导入并包装现有 `backtest_mid.FEE_PCT`、`_limit_up_open` 和 `_fill_sell`。不得复制后再改变其语义，不得修改 `backtest_mid.py`。

- [ ] **Step 4: 实现仓位和退出**

初始止损 `平台-k×ATR`；仓位为 `equity×1%/(buy-stop)`，再取现金、单票、行业和总仓上限的最小值。达到 1R 移到成本加双边费用；达到 2.5R 卖 1/3；余仓改用 4ATR；主吊灯只上移不下移。

- [ ] **Step 5: 运行 GREEN 并提交**

Run: `.venv/bin/python -m unittest test_s3_backtest.PortfolioTests -v`

```bash
git add s3_backtest/engine_adapter.py s3_backtest/portfolio.py test_s3_backtest.py
git commit -m "backtest: add S3 portfolio and fills"
```

---

### Task 16：确定性回测、样本内扫描和一次样本外

**Files:**
- Create: `s3_backtest/metrics.py`
- Create: `s3_backtest_cli.py`
- Modify: `test_s3_backtest.py`

**Interfaces:**
- Produces: `simulate_period(start, end, params, data) -> BacktestResult`
- Produces: `run_in_sample(run_id: str, data: BacktestData) -> tuple[Result, ...]`
- Produces: `lock_selection(run_id) -> Selection`
- Produces: `run_out_of_sample_once(run_id: str, selection: Selection, data: BacktestData) -> BacktestResult`

- [ ] **Step 1: 写确定性和区间隔离测试**

```python
def test_same_inputs_produce_byte_identical_result(self):
    first = serialize(simulate_period("2019-01-01", "2019-12-31",
                                      BASELINE_PARAMS, self.data))
    second = serialize(simulate_period("2019-01-01", "2019-12-31",
                                       BASELINE_PARAMS, self.data))
    self.assertEqual(first, second)

def test_out_of_sample_not_read_during_selection(self):
    data = GuardedData(forbidden_after="2021-12-31")
    run_in_sample("run-1", data=data)
    self.assertEqual(data.forbidden_reads, [])
```

再断言样本内结果恰好 81 行，每个参数 ID 一行；样本外执行第二次时报错。

- [ ] **Step 2: 运行 RED**

Run: `.venv/bin/python -m unittest test_s3_backtest.EngineTests -v`

Expected: FAIL。

- [ ] **Step 3: 实现日序事件循环**

每天顺序固定：更新可见数据 → 处理可成交退出 → 更新止损 → 生成收盘信号 → 记录次日待成交单 → 记权益。所有排序加稳定的 `ts_code` 只用于同分成交顺序，不参与参数选择。

- [ ] **Step 4: 实现运行命令和封存点**

CLI 分为：

```bash
.venv/bin/python s3_backtest_cli.py run-is
.venv/bin/python s3_backtest_cli.py lock-selection
.venv/bin/python s3_backtest_cli.py run-oos
```

`run-is` 只填未完成的 81 个确定性分片；全部完成后自动封存网格哈希。`lock-selection` 只执行冻结规则。`run-oos` 检查锁参记录并只允许一次。

- [ ] **Step 5: 运行 GREEN 并提交**

Run: `.venv/bin/python -m unittest test_s3_backtest.EngineTests -v`

```bash
git add s3_backtest/metrics.py s3_backtest_cli.py test_s3_backtest.py
git commit -m "backtest: add deterministic S3 walk-forward run"
```

---

### Task 17：S2 同口径对照

**Files:**
- Create: `s3_backtest/s2_adapter.py`
- Modify: `test_s3_backtest.py`

**Interfaces:**
- Produces: `run_s2_comparison(data, start, end, fee) -> BacktestResult`
- Consumes: `backtest_mid.simulate_mid`，只调用不修改。

- [ ] **Step 1: 写同口径测试**

断言传给 S2 和 S3 的：

- 起止日期相同；
- 股票域相同；
- OHLCV 数据指纹相同；
- 费用相同；
- 涨跌停和停牌适配器相同；
- 沪深 300 序列相同。

- [ ] **Step 2: 运行 RED**

Run: `.venv/bin/python -m unittest test_s3_backtest.S2ComparisonTests -v`

Expected: FAIL。

- [ ] **Step 3: 实现输入转换和现有引擎调用**

把 T1 只读数据转换为 `backtest_mid.simulate_mid(bars, uni, gate, use_gate, fee)` 所需结构。不得把 S3 广度门传给 S2；S2 保持自身既有门，但区间、费用和股票域必须一致。

- [ ] **Step 4: 运行 GREEN 并提交**

Run: `.venv/bin/python -m unittest test_s3_backtest.S2ComparisonTests -v`

```bash
git add s3_backtest/s2_adapter.py test_s3_backtest.py
git commit -m "backtest: add same-period S2 comparison"
```

---

### Task 18：报告、人工复核和 G1 交付

**Files:**
- Create: `s3_backtest/report.py`
- Create: `reports/s3_backtest_2015_2025.md`
- Create: `reports/s3_parameter_grid.csv`
- Create: `reports/s3_backtest_trades.csv`
- Create: `reports/s3_signal_distribution.csv`
- Create: `docs/验收-T2-S3回测-20260728.md`
- Modify: `test_s3_backtest.py`

**Interfaces:**
- Produces: `build_report(run_id) -> str`
- Produces: 八项报告、数据审计、局限性和 5 笔复核。

- [ ] **Step 1: 写报告完整性测试**

```python
REQUIRED = (
    "2015—2025 全量", "2015、2018、2022", "分年度统计",
    "S2 平行对照", "81 组参数敏感性", "样本外验证",
    "信号分布", "过闸结论",
)
for heading in REQUIRED:
    self.assertIn(heading, report)
```

再断言报告包含财务公告日、2015 ST 局限、幸存者偏差、暖窗覆盖、重叠一致性、`daily.vol` 覆盖、财务缺失、冻结选参和禁止重跑。

- [ ] **Step 2: 运行 RED**

Run: `.venv/bin/python -m unittest test_s3_backtest.ReportTests -v`

Expected: FAIL。

- [ ] **Step 3: 实现八项报告**

报告必须逐项输出：

1. 2015—2025 全量年化、超额、回撤、胜率、盈亏比、单笔期望、笔数；
2. 2015、2018、2022；
3. 11 个年度行；
4. S2 同口径对照；
5. 81 组和孤岛最优检查；
6. 2015—2021 样本内与 2022—2025 样本外；
7. 每年信号数、市况、后 60/120 日分布；
8. 四项过闸判断。

- [ ] **Step 4: 执行一次冻结流程**

按顺序运行：

```bash
.venv/bin/python s3_backtest_cli.py run-is
.venv/bin/python s3_backtest_cli.py lock-selection
.venv/bin/python s3_backtest_cli.py run-oos
.venv/bin/python s3_backtest_cli.py report
```

若任一步发现输入指纹变化、重复运行、参数数目不是 81、选择完全并列或数据不完整，停止报告用户，不改规则、不补组合、不重新开始。

- [ ] **Step 5: 人工复核 5 笔**

固定随机种子从交易明细抽 5 笔，逐笔展示：

- 信号日可见数据；
- 集群和资格；
- 平台、ATR、止损；
- 次日开盘与是否受板制/停牌影响；
- 仓位倒算和上限；
- 退出线、分批和费用。

复核脚本只能读封存结果，不能改交易。

- [ ] **Step 6: 完整测试**

Run:

```bash
.venv/bin/python -m unittest test_s3_backtest -v
.venv/bin/python test_rules.py && .venv/bin/python test_rules_mid.py && .venv/bin/python test_web.py && .venv/bin/python test_regime.py
git diff --check
```

Expected: 全部退出 0。

- [ ] **Step 7: 禁区审计**

Run:

```bash
git diff --name-only "$(git merge-base codex/s3-stage2-backtest codex/s3-stage1-data-clusters)"...HEAD
git diff -- backtest.py backtest_mid.py rules_mid.py strategies market_data.py storage.py eod.py service.py static
```

Expected: 第一条只含 T2 创建文件和报告；第二条零输出。

- [ ] **Step 8: 提交并停在 G1**

```bash
git add s3_backtest s3_backtest_cli.py test_s3_backtest.py reports/s3_backtest_2015_2025.md reports/s3_parameter_grid.csv reports/s3_backtest_trades.csv reports/s3_signal_distribution.csv docs/验收-T2-S3回测-20260728.md
git commit -m "research: deliver frozen S3 backtest report"
```

向用户提交报告、分支、提交、数据指纹和未合并状态。停止等待 G1；无论结果好坏，都不得动工 T5。
