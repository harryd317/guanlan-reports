# A股产业量能共振雷达 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 建立一个只读、可审计的产业周量能共振研究引擎，先验证产业发现能力，再排序产业核心股票，并在历史闸门通过后开始追加式前瞻影子记录。

**Architecture:** 新研究包独立放在 `research/industry_volume_resonance/`，复用第一阶段只读周线缓存和安全 I/O，不修改第一阶段结果或生产代码。数据流依次为历史产业成员缓存 → 股票周特征 → 产业周特征 → 产业事件与状态 → 前向标签和匹配对照 → 核心股排序 → 历史滚动复现 → 前瞻影子日志 → HTML 报告和独立校验。

**Tech Stack:** Python 3.11、pandas、numpy、sqlite3、unittest、现有 Tushare SDK、单文件 HTML；不增加第三方依赖。

## Global Constraints

- 研究版本固定为 `industry_volume_resonance_v1`，随机种子固定为 `20260727`。
- 主检验单位是“产业 × 完整市场周”，同产业八周内连续信号合并为一个事件。
- 主产业成员必须有 `in_date` 和 `out_date`；当前行业分类不得回填历史。
- 2022—2026-07-24 标记为 `seen_history_replication`，不得声称是全新样本外。
- 真正前瞻记录从冻结协议后的下一个完整市场周开始，只追加，不覆盖。
- CPO 等缺少历史成员证据的产业链主题只进入前瞻主题注册表，不回填历史。
- 前瞻主题注册表为空时，报告必须明确写“CPO 等主题尚未评价”，不得把申万行业结果冒充产业链主题结果。
- 结果只表示“值得研究”，不得输出“机构建仓”“推荐买入”或仓位建议。
- 正式 SQLite 只用 `mode=ro&immutable=1` 打开；所有新写入限制在 `research/industry_volume_resonance/` 和第二阶段报告文件。
- 第一阶段代码、结果、审计文件以及生产规则、正式数据库、页面、任务、参数、服务和 Vault 保持零修改。
- Token、账户信息和原始 API 错误不得写入缓存、日志、报告或 Git。
- 任意 `.py` 变更后运行专项测试以及 `.venv/bin/python test_rules.py && .venv/bin/python test_rules_mid.py && .venv/bin/python test_web.py && .venv/bin/python test_regime.py`。
- 任何网络调用只允许出现在显式 `prepare` 阶段；单元测试必须注入 Fake API。

---

## Per-Task Python Gate

Tasks 1—11 每次修改 Python 文件后，必须先运行该 Task 的专项测试，再分别运行以下四个现有回归门禁，全部通过后才允许执行该 Task 的提交步骤：

```bash
.venv/bin/python test_rules.py
.venv/bin/python test_rules_mid.py
.venv/bin/python test_web.py
.venv/bin/python test_regime.py
```

若任一门禁失败，保存失败输出并修复；不得用跳过、改弱断言或删除测试的方式继续提交。

---

## File Map

### 新建研究包

- `research/industry_volume_resonance/__init__.py`：公开研究版本。
- `research/industry_volume_resonance/config.py`：冻结日期、阈值、状态和判决常量。
- `research/industry_volume_resonance/models.py`：不可变阈值和成员区间数据类型。
- `research/industry_volume_resonance/sources.py`：Tushare 产业成员和沪深 300 OHLC 研究缓存。
- `research/industry_volume_resonance/membership.py`：历史成员区间校验、点时展开和前瞻主题注册表。
- `research/industry_volume_resonance/stock_features.py`：股票周量能和价格特征。
- `research/industry_volume_resonance/industry_features.py`：产业周广度、分散性、价格确认和匹配协变量。
- `research/industry_volume_resonance/states.py`：产业状态和八周事件合并。
- `research/industry_volume_resonance/labels.py`：固定成员集合的 13/26 周产业标签。
- `research/industry_volume_resonance/matching.py`：同周 1:3 产业对照。
- `research/industry_volume_resonance/core.py`：D1/D2/D3 核心股票排序。
- `research/industry_volume_resonance/metrics.py`：历史滚动指标、产业整簇 Bootstrap、稳定性和判决。
- `research/industry_volume_resonance/protocol.py`：冻结协议、哈希、前瞻起点和追加锁。
- `research/industry_volume_resonance/shadow.py`：每周前瞻影子快照和 JSONL 追加。
- `research/industry_volume_resonance/pipeline.py`：阶段编排和机器结果导出。
- `research/industry_volume_resonance/report.py`：第二阶段 HTML 报告。
- `research/industry_volume_resonance/validate.py`：文件、哈希、防前视、正式源零修改和敏感信息校验。
- `research/run_industry_volume_resonance.py`：显式阶段 CLI。

### 新建测试和版本化入口

- `test_industry_volume_resonance.py`：专项离线测试。
- `research/industry_volume_resonance/cache/.gitignore`：忽略缓存正文，保留目录。
- `research/industry_volume_resonance/results/.gitkeep`：保留结果目录。
- `research/industry_volume_resonance/themes/prospective_theme_members.csv`：前瞻多标签产业链注册表；初始只有表头，不伪造历史成员。
- `docs/研究-A股产业量能共振雷达-第二阶段.html`：最终人读报告。

### 只读复用

- `research/weekly_volume_radar/cache/weekly_cache.sqlite`
- `research/weekly_volume_radar/cache/panel.sqlite`
- `research/weekly_volume_radar/io.py`
- `research/weekly_volume_radar/metrics.py`
- `market.db`
- 仓库外 `.token`

---

### Task 1: Freeze the Research Contract

**Files:**
- Create: `research/industry_volume_resonance/__init__.py`
- Create: `research/industry_volume_resonance/config.py`
- Create: `research/industry_volume_resonance/models.py`
- Create: `research/industry_volume_resonance/cache/.gitignore`
- Create: `research/industry_volume_resonance/results/.gitkeep`
- Create: `test_industry_volume_resonance.py`

**Interfaces:**
- Produces: `ResonanceThresholds`, `RESEARCH_VERSION`, `PERIOD_BOUNDARIES`, `STATE_ORDER`, `VERDICTS`.
- Consumes: no earlier task.

- [ ] **Step 1: Write failing contract tests**

```python
class IndustryConfigTests(unittest.TestCase):
    def test_contract_is_frozen(self):
        self.assertEqual(config.RESEARCH_VERSION, "industry_volume_resonance_v1")
        self.assertEqual(config.RANDOM_SEED, 20260727)
        self.assertEqual(config.EVENT_MERGE_WEEKS, 8)
        self.assertEqual(config.PERIOD_BOUNDARIES["design_end"], "2018-12-31")
        self.assertEqual(config.PERIOD_BOUNDARIES["validation_end"], "2021-12-31")
        self.assertEqual(config.PERIOD_BOUNDARIES["seen_replication_end"], "2026-07-24")
        self.assertEqual(
            set(config.VERDICTS),
            {"值得进入前瞻影子雷达", "证据不足", "停止该版本"},
        )

    def test_threshold_values_match_design(self):
        t = ResonanceThresholds()
        self.assertEqual(t.activity_ratio_min, 1.50)
        self.assertEqual(t.amount_percentile_min, 0.80)
        self.assertEqual(t.active_breadth_min, 0.20)
        self.assertEqual(t.amount_share_shift_min, 1.25)
        self.assertEqual(t.top1_amount_share_max, 0.45)
```

- [ ] **Step 2: Run the tests and confirm the missing package failure**

Run:

```bash
.venv/bin/python -m unittest test_industry_volume_resonance.IndustryConfigTests -v
```

Expected: `ModuleNotFoundError: No module named 'research.industry_volume_resonance'`.

- [ ] **Step 3: Implement immutable constants and thresholds**

```python
from dataclasses import dataclass

RESEARCH_VERSION = "industry_volume_resonance_v1"
RANDOM_SEED = 20260727
EVENT_MERGE_WEEKS = 8
BOOTSTRAP_N = 2000
PERIOD_BOUNDARIES = {
    "design_start": "2015-01-01",
    "design_end": "2018-12-31",
    "validation_start": "2019-01-01",
    "validation_end": "2021-12-31",
    "seen_replication_start": "2022-01-01",
    "seen_replication_end": "2026-07-24",
}
STATE_ORDER = (
    "共振失败",
    "高潮风险",
    "趋势确认",
    "早期扩散",
    "产业苏醒",
    "单股异常",
)
VERDICTS = ("值得进入前瞻影子雷达", "证据不足", "停止该版本")

@dataclass(frozen=True)
class ResonanceThresholds:
    activity_ratio_min: float = 1.50
    amount_percentile_min: float = 0.80
    active_count_min: int = 3
    active_breadth_min: float = 0.20
    amount_share_shift_min: float = 1.25
    top1_amount_share_max: float = 0.45
    persistence_min: int = 2
    climax_amount_share_shift: float = 2.50
```

- [ ] **Step 4: Run contract tests**

Run:

```bash
.venv/bin/python -m unittest test_industry_volume_resonance.IndustryConfigTests -v
```

Expected: all `IndustryConfigTests` pass.

- [ ] **Step 5: Commit**

```bash
git add research/industry_volume_resonance test_industry_volume_resonance.py
git commit -m "test: freeze industry resonance research contract"
```

---

### Task 2: Build Point-in-Time Industry and Benchmark Caches

**Files:**
- Create: `research/industry_volume_resonance/sources.py`
- Create: `research/industry_volume_resonance/membership.py`
- Create: `research/industry_volume_resonance/themes/prospective_theme_members.csv`
- Modify: `test_industry_volume_resonance.py`

**Interfaces:**
- Consumes: `ResonanceThresholds`, safe I/O from `research.weekly_volume_radar.io`.
- Produces:
  - `build_source_cache(cache_db: Path, token_path: Path, api=None, pause: float = 0.2) -> dict`
  - `load_membership_intervals(cache_db: Path, level: str = "L2") -> pd.DataFrame`
  - `validate_membership_intervals(frame: pd.DataFrame) -> None`
  - `expand_membership_to_weeks(memberships: pd.DataFrame, calendar: pd.DataFrame) -> pd.DataFrame`
  - `load_prospective_themes(path: Path, as_of: str, registry_committed_at: str) -> pd.DataFrame`
  - cache tables `classifications`, `membership_intervals`, `benchmark_daily`, `fetch_audit`.

- [ ] **Step 1: Write failing point-in-time membership tests**

```python
class MembershipTests(unittest.TestCase):
    def test_interval_boundaries_are_point_in_time(self):
        members = pd.DataFrame([{
            "industry_code": "801780.SI",
            "industry_name": "银行",
            "level": "L2",
            "ts_code": "600000.SH",
            "in_date": "2019-01-01",
            "out_date": "2020-06-30",
            "source": "TUSHARE_index_member_all_SW2021",
        }])
        weeks = pd.DataFrame({"week_end": pd.to_datetime(
            ["2018-12-28", "2019-01-04", "2020-06-26", "2020-07-03"]
        )})
        got = expand_membership_to_weeks(members, weeks)
        self.assertEqual(
            list(got["week_end"].dt.strftime("%Y-%m-%d")),
            ["2019-01-04", "2020-06-26"],
        )

    def test_missing_in_date_is_rejected(self):
        bad = pd.DataFrame([{
            "industry_code": "x", "industry_name": "x", "level": "L2",
            "ts_code": "600000.SH", "in_date": None, "out_date": None,
            "source": "current_static",
        }])
        with self.assertRaisesRegex(ValueError, "in_date"):
            validate_membership_intervals(bad)

    def test_prospective_stock_can_keep_multiple_theme_roles(self):
        rows = pd.DataFrame([
            prospective_theme_row("CPO", "光模块", "300308.SZ"),
            prospective_theme_row("AI算力", "光互联", "300308.SZ"),
        ])
        path, committed_at = write_theme_fixture(rows)
        got = load_prospective_themes(path, "2026-08-07", committed_at)
        self.assertEqual(len(got.loc[got["ts_code"] == "300308.SZ"]), 2)
```

- [ ] **Step 2: Write a Fake API cache test**

```python
class SourceCacheTests(unittest.TestCase):
    def test_cache_uses_y_and_n_members_and_stores_no_token(self):
        class FakeApi:
            def index_classify(self, **kwargs):
                return pd.DataFrame([{
                    "index_code": "801780.SI", "industry_name": "银行",
                    "level": "L2", "industry_code": "480000",
                    "parent_code": "490000", "is_pub": "1", "src": "SW2021",
                }])

            def index_member_all(self, **kwargs):
                row = {
                    "l2_code": "801780.SI", "l2_name": "银行",
                    "ts_code": "600000.SH", "name": "浦发银行",
                    "in_date": "19991110",
                    "out_date": None if kwargs["is_new"] == "Y" else "20201231",
                    "is_new": kwargs["is_new"],
                }
                return pd.DataFrame([row])

            def index_daily(self, **kwargs):
                return pd.DataFrame([{
                    "ts_code": "000300.SH", "trade_date": "20200102",
                    "open": 10, "high": 11, "low": 9, "close": 10.5,
                }])

        with tempfile.TemporaryDirectory() as td:
            token = Path(td) / "token"
            token.write_text("DO_NOT_STORE", encoding="utf-8")
            db = Path(td) / "sources.sqlite"
            result = build_source_cache(db, token, api=FakeApi(), pause=0)
            self.assertGreaterEqual(result["membership_rows"], 1)
            self.assertNotIn(b"DO_NOT_STORE", db.read_bytes())
```

- [ ] **Step 3: Run the membership tests and confirm missing functions**

Run:

```bash
.venv/bin/python -m unittest \
  test_industry_volume_resonance.MembershipTests \
  test_industry_volume_resonance.SourceCacheTests -v
```

Expected: imports fail for `sources.py` and `membership.py`.

- [ ] **Step 4: Implement the research source cache**

Use the official Tushare calls:

```python
classifications = api.index_classify(level="L2", src="SW2021")
for row in classifications.itertuples():
    for is_new in ("Y", "N"):
        part = api.index_member_all(l2_code=row.index_code, is_new=is_new)
        # Normalize YYYYMMDD to YYYY-MM-DD and deduplicate exact intervals.
benchmark = api.index_daily(
    ts_code="000300.SH",
    start_date="20120101",
    end_date=data_cutoff.replace("-", ""),
    fields="ts_code,trade_date,open,high,low,close",
)
```

Write to a temporary SQLite file, validate row counts and required fields, then atomically replace the cache. On an API permission error, store only `type(error).__name__` plus a scrubbed summary in `fetch_audit`; never store the exception representation, URL, token, or request headers.

- [ ] **Step 5: Implement interval expansion and prospective theme validation**

```python
REQUIRED_MEMBERSHIP = {
    "industry_code", "industry_name", "level", "ts_code",
    "in_date", "out_date", "source",
}

def expand_membership_to_weeks(memberships, calendar):
    # Join where in_date <= week_end <= out_date.
    # Treat blank out_date as open-ended.
    # Return one row per industry_code, ts_code, week_end.
```

Create `prospective_theme_members.csv` with this exact header:

```csv
theme_code,theme_name,node,role,ts_code,effective_from,effective_to,evidence_source,evidence_date
```

`load_prospective_themes()` must reject rows whose `effective_from` precedes the evidence date or precedes `registry_committed_at`, the registry file's first Git commit date supplied by the caller.

- [ ] **Step 6: Run tests**

Run:

```bash
.venv/bin/python -m unittest \
  test_industry_volume_resonance.MembershipTests \
  test_industry_volume_resonance.SourceCacheTests -v
```

Expected: all membership and cache tests pass without network access.

- [ ] **Step 7: Commit**

```bash
git add research/industry_volume_resonance/sources.py \
  research/industry_volume_resonance/membership.py \
  research/industry_volume_resonance/themes/prospective_theme_members.csv \
  test_industry_volume_resonance.py
git commit -m "feat: cache point-in-time industry membership"
```

---

### Task 3: Compute Causal Stock-Week Features

**Files:**
- Create: `research/industry_volume_resonance/stock_features.py`
- Modify: `test_industry_volume_resonance.py`

**Interfaces:**
- Consumes: weekly OHLCV rows and point-in-time eligibility from the first-stage caches.
- Produces:
  - `load_stock_week_base(panel_db: Path, weekly_db: Path) -> pd.DataFrame`
  - `compute_stock_week_features(frame: pd.DataFrame) -> pd.DataFrame`
  - fields `activity_ratio_2_26`, `amount_percentile_52`, `amount_slope_4`, `up_down_amount_ratio_8`, `pullback_amount_ratio`, `ret_1w`, `ret_4w_feature`, `above_ma20`, `new_high_26w`, `dist_ma20`, `long_upper`, `pullback_volume`.

- [ ] **Step 1: Write failing causal feature tests**

```python
class StockFeatureTests(unittest.TestCase):
    def test_incomplete_market_week_is_excluded_at_load_boundary(self):
        got = load_stock_week_base(
            panel_db=panel_fixture(complete_week_end="2026-07-24"),
            weekly_db=weekly_fixture(last_week_end="2026-07-31"),
        )
        self.assertEqual(got["week_end"].max().strftime("%Y-%m-%d"), "2026-07-24")

    def test_baseline_excludes_recent_four_weeks(self):
        frame = stock_week_fixture(90)
        frame.loc[86:89, "amount"] = [160, 180, 200, 220]
        got = compute_stock_week_features(frame)
        expected = np.mean([200, 220]) / np.median(frame.loc[60:85, "amount"])
        self.assertAlmostEqual(got.iloc[-1]["activity_ratio_2_26"], expected)

    def test_future_mutation_does_not_change_past_features(self):
        frame = stock_week_fixture(90)
        before = compute_stock_week_features(frame).loc[:70].copy()
        frame.loc[71:, ["open", "high", "low", "close", "amount"]] *= 100
        after = compute_stock_week_features(frame).loc[:70]
        pd.testing.assert_frame_equal(before, after)

    def test_long_upper_and_pullback_volume_are_exact(self):
        frame = stock_week_fixture(90)
        frame.loc[89, ["open", "high", "low", "close"]] = [10, 20, 9, 11]
        frame.loc[89, "amount"] = frame.loc[84:87, "amount"].mean() * 2
        got = compute_stock_week_features(frame)
        self.assertTrue(got.iloc[-1]["long_upper"])
        self.assertTrue(got.iloc[-1]["pullback_volume"])

    def test_amount_percentile_compares_current_week_with_prior_52(self):
        frame = stock_week_fixture(90)
        frame["amount"] = np.arange(1, 91, dtype=float)
        got = compute_stock_week_features(frame)
        self.assertEqual(got.iloc[-1]["amount_percentile_52"], 1.0)
```

- [ ] **Step 2: Run tests and confirm failure**

Run:

```bash
.venv/bin/python -m unittest test_industry_volume_resonance.StockFeatureTests -v
```

Expected: `compute_stock_week_features` is missing.

- [ ] **Step 3: Implement the feature function**

```python
def _one_stock(group: pd.DataFrame) -> pd.DataFrame:
    g = group.sort_values("week_seq").copy()
    amount = pd.to_numeric(g["amount"], errors="coerce")
    close = pd.to_numeric(g["close"], errors="coerce")
    baseline = amount.shift(4).rolling(26).median()
    g["activity_ratio_2_26"] = amount.rolling(2).mean() / baseline
    g["amount_percentile_52"] = amount.rolling(53).apply(
        lambda x: float(np.mean(x[:-1] <= x[-1])),
        raw=True,
    )
    g["amount_slope_4"] = rolling_slope(amount, 4) / baseline
    g["ret_1w"] = close / close.shift(1) - 1
    g["ret_4w_feature"] = close / close.shift(4) - 1
    g["ma20"] = close.rolling(20).mean()
    g["above_ma20"] = close >= g["ma20"]
    g["new_high_26w"] = close >= close.shift(1).rolling(26).max()
    g["dist_ma20"] = close / g["ma20"] - 1
    spread = (g["high"] - g["low"]).replace(0, np.nan)
    g["long_upper"] = (g["high"] - g[["open", "close"]].max(axis=1)) / spread >= 0.40
    g["pullback_volume"] = (g["ret_1w"] < 0) & (
        amount > amount.shift(1).rolling(4).mean()
    )
    return g
```

Compute `up_down_amount_ratio_8` and `pullback_amount_ratio` exactly as specified. Preserve input `eligible`, `total_mv_yi`, `cap_bucket`, `week_end`, and `week_seq`.

`load_stock_week_base()` must use the cached market calendar to determine the latest completed market week. It must not infer completeness from the presence of a partial Friday row or from the current wall-clock date.

- [ ] **Step 4: Run tests**

Run:

```bash
.venv/bin/python -m unittest test_industry_volume_resonance.StockFeatureTests -v
```

Expected: all stock feature tests pass.

- [ ] **Step 5: Commit**

```bash
git add research/industry_volume_resonance/stock_features.py \
  test_industry_volume_resonance.py
git commit -m "feat: compute causal stock-week resonance features"
```

---

### Task 4: Aggregate Industry-Week Breadth and Dispersion

**Files:**
- Create: `research/industry_volume_resonance/industry_features.py`
- Modify: `test_industry_volume_resonance.py`

**Interfaces:**
- Consumes: stock-week features and expanded membership rows.
- Produces:
  - `build_industry_week_features(stock_week: pd.DataFrame, expanded_membership: pd.DataFrame, thresholds: ResonanceThresholds) -> pd.DataFrame`
  - one row per `industry_code, week_end`
  - fields listed in design section 5.

- [ ] **Step 1: Write failing aggregation tests**

```python
class IndustryFeatureTests(unittest.TestCase):
    def test_three_stock_resonance_and_top_share(self):
        stock = industry_stock_fixture(
            amounts=[40, 30, 20, 10, 5],
            active=[True, True, True, False, False],
            caps=[500, 300, 100, 50, 20],
        )
        got = build_industry_week_features(stock, membership_fixture(stock), ResonanceThresholds())
        row = got.iloc[-1]
        self.assertEqual(row["eligible_count"], 5)
        self.assertEqual(row["active_count"], 3)
        self.assertAlmostEqual(row["active_breadth"], 0.60)
        self.assertAlmostEqual(row["top1_amount_share"], 40 / 105)
        self.assertEqual(row["active_cap_bucket_count"], 3)

    def test_single_stock_dominance_is_visible(self):
        stock = industry_stock_fixture(
            amounts=[90, 3, 3, 2, 2],
            active=[True, False, False, False, False],
            caps=[500, 300, 100, 50, 20],
        )
        row = build_industry_week_features(
            stock, membership_fixture(stock), ResonanceThresholds()
        ).iloc[-1]
        self.assertGreater(row["top1_amount_share"], 0.45)
        self.assertEqual(row["active_count"], 1)
```

- [ ] **Step 2: Run tests and confirm failure**

Run:

```bash
.venv/bin/python -m unittest test_industry_volume_resonance.IndustryFeatureTests -v
```

Expected: aggregator import fails.

- [ ] **Step 3: Implement point-in-time join and weekly aggregation**

```python
def build_industry_week_features(stock_week, expanded_membership, thresholds):
    joined = stock_week.merge(
        expanded_membership,
        on=["ts_code", "week_end"],
        how="inner",
        validate="many_to_many",
    )
    joined = joined.loc[joined["eligible"].astype(bool)].copy()
    joined["active"] = (
        (joined["activity_ratio_2_26"] >= thresholds.activity_ratio_min)
        & (joined["amount_percentile_52"] >= thresholds.amount_percentile_min)
    )
    # Aggregate counts, equal-weight price breadth, market-cap top five,
    # amount concentration, market amount share, past volatility and strength.
```

Calculate `amount_share_shift` with the current two-week average industry market-share ratio divided by the prior 26-week median after shifting four weeks. Calculate `persistence_4` only after `active_breadth` exists. Retain rows with at least five eligible members; record smaller groups in an exclusion frame returned through `DataFrame.attrs["excluded"]`.

- [ ] **Step 4: Add a future-mutation aggregation guard**

Add a test that multiplies all stock inputs after a cutoff by 100 and asserts that industry rows through the cutoff remain identical.

- [ ] **Step 5: Run tests**

Run:

```bash
.venv/bin/python -m unittest test_industry_volume_resonance.IndustryFeatureTests -v
```

Expected: all industry feature tests pass.

- [ ] **Step 6: Commit**

```bash
git add research/industry_volume_resonance/industry_features.py \
  test_industry_volume_resonance.py
git commit -m "feat: aggregate point-in-time industry resonance"
```

---

### Task 5: Implement the State Machine and Event Merge

**Files:**
- Create: `research/industry_volume_resonance/states.py`
- Modify: `test_industry_volume_resonance.py`

**Interfaces:**
- Consumes: industry-week feature rows.
- Produces:
  - `classify_industry_weeks(frame: pd.DataFrame, thresholds: ResonanceThresholds) -> pd.DataFrame`
  - `merge_industry_events(frame: pd.DataFrame, max_gap_weeks: int = 8) -> pd.DataFrame`
  - event fields `event_id`, `industry_code`, `industry_name`, `event_week_seq`, `event_week_end`, `raw_signal_weeks`, `state_at_event`, `max_state`, `failed_within_4w`.

- [ ] **Step 1: Write failing state priority tests**

```python
class StateAndEventTests(unittest.TestCase):
    def test_state_priority_places_climax_before_confirmation(self):
        row = industry_week_row(
            active_count=5, active_breadth=.50, amount_share_shift=2.8,
            top1_amount_share=.30, active_cap_bucket_count=2,
            persistence_4=3, core_active_count=2,
            current_active_count_above_prior_median=True,
            ret_4w_excess=.12, up_share=.70, above_ma20_share=.80,
            new_high_26w_share=.30, leader_dist_ma20_median=.30,
            no_progress_active_share=.20, long_or_pullback_active_share=.20,
        )
        got = classify_industry_weeks(pd.DataFrame([row]), ResonanceThresholds())
        self.assertEqual(got.iloc[0]["state"], "高潮风险")

    def test_signals_within_eight_weeks_merge(self):
        frame = industry_signal_fixture(week_seq=[10, 12, 18, 27])
        events = merge_industry_events(frame, max_gap_weeks=8)
        self.assertEqual(list(events["event_week_seq"]), [10, 27])
        self.assertEqual(events.iloc[0]["raw_signal_weeks"].count("|"), 2)
```

- [ ] **Step 2: Run tests and confirm failure**

Run:

```bash
.venv/bin/python -m unittest test_industry_volume_resonance.StateAndEventTests -v
```

Expected: state functions are missing.

- [ ] **Step 3: Implement exact state predicates**

```python
def classify_industry_weeks(frame, thresholds):
    out = frame.copy()
    awake = (
        (out["active_count"] >= 3)
        & (out["active_breadth"] >= .20)
        & (out["amount_share_shift"] >= 1.25)
        & (out["top1_amount_share"] <= .45)
        & (out["active_cap_bucket_count"] >= 2)
    )
    diffusion = awake & (out["persistence_4"] >= 2) & (
        out["core_active_count"] >= 1
    ) & out["current_active_count_above_prior_median"]
    confirm_count = pd.concat([
        out["ret_4w_excess"] > 0,
        out["up_share"] >= .60,
        out["above_ma20_share"] >= .60,
        out["new_high_26w_share"] >= .20,
    ], axis=1).sum(axis=1)
    confirmed = diffusion & (confirm_count >= 3)
    climax = awake & (
        ((out["amount_share_shift"] >= 2.50)
         & (out["up_share"] < .50)
         & (out["leader_dist_ma20_median"] > .25))
        | ((out["no_progress_active_share"] >= .50)
           & (out["long_or_pullback_active_share"] >= .40))
    )
    # Assign in fixed priority: climax, confirmed, diffusion, awake, single.
```

After events are merged, inspect the following four industry weeks. Mark `failed_within_4w` only when `active_breadth < .10` and `ret_4w_excess <= 0`; do not rewrite the original signal week.

- [ ] **Step 4: Add single-stock and failure tests**

Verify that one active stock never becomes `产业苏醒`, and that changing week `t+5` does not change a failure decision whose four-week window ended at `t+4`.

- [ ] **Step 5: Run tests**

Run:

```bash
.venv/bin/python -m unittest test_industry_volume_resonance.StateAndEventTests -v
```

Expected: all state and event tests pass.

- [ ] **Step 6: Commit**

```bash
git add research/industry_volume_resonance/states.py \
  test_industry_volume_resonance.py
git commit -m "feat: classify and merge industry resonance events"
```

---

### Task 6: Label Fixed-Member Industry Outcomes and Match Controls

**Files:**
- Create: `research/industry_volume_resonance/labels.py`
- Create: `research/industry_volume_resonance/matching.py`
- Modify: `test_industry_volume_resonance.py`

**Interfaces:**
- Consumes: industry events, stock weekly OHLCV, event-time members, benchmark OHLC.
- Produces:
  - `label_industry_event(event: dict, members: pd.DataFrame, stock_week: pd.DataFrame, benchmark_week: pd.DataFrame) -> dict`
  - `match_industry_controls(event: dict, candidates: pd.DataFrame, k: int = 3) -> pd.DataFrame`
  - labels `industry_followthrough_13`, `industry_trend_26`, `core_emergence_26`, `reversal_risk_13`, censor flags and fixed member count.

- [ ] **Step 1: Write failing fixed-member label tests**

```python
class LabelAndMatchingTests(unittest.TestCase):
    def test_label_starts_at_next_open_and_keeps_loser(self):
        event = {"industry_code": "I1", "week_seq": 10, "week_end": "2020-03-13"}
        members = pd.DataFrame({"ts_code": ["A", "B", "C", "D", "E"]})
        stock = fixed_member_forward_fixture(
            entry_opens={"A": 100, "B": 100, "C": 100, "D": 100, "E": 100},
            max_26={"A": 1.60, "B": 1.40, "C": 1.35, "D": .90, "E": .50},
        )
        benchmark = benchmark_fixture(entry_open=100, max_26=110)
        got = label_industry_event(event, members, stock, benchmark)
        self.assertEqual(got["fixed_member_count"], 5)
        self.assertTrue(got["industry_trend_26"])
        self.assertIn("E", got["fixed_member_codes"])

    def test_right_censored_event_is_not_failure(self):
        got = label_industry_event(
            event_near_cutoff(), five_members(), short_forward_fixture(), benchmark_fixture_short()
        )
        self.assertTrue(got["right_censored_26w"])
        self.assertIsNone(got["industry_trend_26"])
```

- [ ] **Step 2: Write a no-future matching test**

```python
    def test_match_order_ignores_future_return(self):
        event = {
            "industry_code": "I1", "week_end": "2020-03-13",
            "eligible_count": 10, "industry_total_mv_yi": 5000,
            "past_vol_13w": .20, "industry_equal_weight_ret_13w": .10,
        }
        pool = matching_fixture_with_extreme_future_returns()
        got = match_industry_controls(event, pool, k=3)
        self.assertEqual(list(got["industry_code"]), ["I2", "I3", "I4"])
        self.assertNotIn("future_return", got.attrs["matching_columns"])
```

- [ ] **Step 3: Run tests and confirm failure**

Run:

```bash
.venv/bin/python -m unittest test_industry_volume_resonance.LabelAndMatchingTests -v
```

Expected: label and matching modules are missing.

- [ ] **Step 4: Implement fixed-member labels**

For each event:

1. Freeze members from the event week.
2. Use each member's next trading-day open as its start.
3. Calculate each member's cumulative weekly close return through 13 and 26 weeks.
4. Keep a delisted or suspended member in the fixed denominator; use its last tradable close and emit `terminal_price_gap=True`.
5. Build the industry equal-weight series from the fixed set.
6. Subtract the benchmark series started at the next benchmark open.
7. Apply the exact labels from design section 9.

Return `None` for labels whose full horizon is unavailable.

- [ ] **Step 5: Implement standardized 1:3 matching**

```python
MATCHING_FIELDS = (
    "eligible_count", "industry_total_mv_yi",
    "past_vol_13w", "industry_equal_weight_ret_13w",
)

def match_industry_controls(event, candidates, k=3):
    # Same week, not the event industry, not already signaled,
    # all MATCHING_FIELDS observed.
    # Filter by week_end first, then Z-score MATCHING_FIELDS only.
    # Sort by Euclidean distance then industry_code.
```

- [ ] **Step 6: Run tests**

Run:

```bash
.venv/bin/python -m unittest test_industry_volume_resonance.LabelAndMatchingTests -v
```

Expected: all label and matching tests pass.

- [ ] **Step 7: Commit**

```bash
git add research/industry_volume_resonance/labels.py \
  research/industry_volume_resonance/matching.py \
  test_industry_volume_resonance.py
git commit -m "feat: label and match industry resonance events"
```

---

### Task 7: Rank Core Stocks Without Changing Industry Events

**Files:**
- Create: `research/industry_volume_resonance/core.py`
- Modify: `test_industry_volume_resonance.py`

**Interfaces:**
- Consumes: frozen event rows and event-week member features.
- Produces:
  - `rank_core_stocks(event_members: pd.DataFrame) -> pd.DataFrame`
  - `choose_core_method(method_metrics: pd.DataFrame) -> str`
  - rows for D1, D2, D3 with `method`, `rank`, `score`, `reason_fields`, `risk_fields`.

- [ ] **Step 1: Write failing deterministic ranking tests**

```python
class CoreRankingTests(unittest.TestCase):
    def test_d3_uses_frozen_weights_and_returns_three(self):
        members = core_member_fixture()
        got = rank_core_stocks(members)
        d3 = got.loc[got["method"] == "D3"].sort_values("rank")
        self.assertEqual(len(d3), 3)
        self.assertEqual(list(d3["ts_code"]), ["A", "B", "C"])
        self.assertAlmostEqual(
            d3.iloc[0]["score"],
            .30 * 1 + .25 * 1 + .20 * 1 + .15 * 1 + .10 * 1,
        )

    def test_complex_method_must_beat_both_simple_methods(self):
        metrics = pd.DataFrame([
            {"method": "D1", "precision_at_3": .30, "median_excess": .04, "mdd": -.20, "n": 100},
            {"method": "D2", "precision_at_3": .32, "median_excess": .05, "mdd": -.22, "n": 100},
            {"method": "D3", "precision_at_3": .31, "median_excess": .08, "mdd": -.18, "n": 100},
        ])
        self.assertEqual(choose_core_method(metrics), "D2")
```

- [ ] **Step 2: Run tests and confirm failure**

Run:

```bash
.venv/bin/python -m unittest test_industry_volume_resonance.CoreRankingTests -v
```

Expected: core module is missing.

- [ ] **Step 3: Implement D1, D2 and D3**

```python
D3_WEIGHTS = {
    "relative_strength_13w": .30,
    "amount_persistence_8w": .25,
    "up_pullback_consistency": .20,
    "industry_drawdown_resistance": .15,
    "liquidity_26w": .10,
}

def rank_core_stocks(event_members):
    # D1 score: event-week historical total market cap percentile.
    # D2 score: event-week 13-week excess-strength percentile.
    # Convert each D3 field to an industry-week percentile.
    # Higher values are better for all normalized fields.
    # Use ts_code as deterministic final tie-breaker.
    # Emit top three per method; never mutate the event table.
```

`choose_core_method()` applies the fixed order `Precision@3 → median excess → MDD → coverage`. D3 wins only if both Precision@3 and median excess exceed D1 and D2.

- [ ] **Step 4: Add missing-data tests**

Verify that a stock missing one D3 input remains in D1/D2 but is excluded from D3 with `exclusion_reason="d3_feature_missing"`. Do not fill from cross-sectional future values.

- [ ] **Step 5: Run tests**

Run:

```bash
.venv/bin/python -m unittest test_industry_volume_resonance.CoreRankingTests -v
```

Expected: all core ranking tests pass.

- [ ] **Step 6: Commit**

```bash
git add research/industry_volume_resonance/core.py \
  test_industry_volume_resonance.py
git commit -m "feat: rank core stocks inside frozen industries"
```

---

### Task 8: Compute Walk-Forward Metrics and Automatic Verdict

**Files:**
- Create: `research/industry_volume_resonance/metrics.py`
- Modify: `test_industry_volume_resonance.py`

**Interfaces:**
- Consumes: labeled events, matched controls, core rankings.
- Produces:
  - `period_label(week_end: str, prospective_start: str | None = None) -> str`
  - `summarize_stage(events: pd.DataFrame, controls: pd.DataFrame) -> dict`
  - `industry_cluster_bootstrap(events: pd.DataFrame, controls: pd.DataFrame, n_boot: int = 2000) -> dict`
  - `stability_checks(events: pd.DataFrame) -> dict`
  - `decide_verdict(inputs: dict) -> dict`.

- [ ] **Step 1: Write failing period and verdict tests**

```python
class MetricsAndVerdictTests(unittest.TestCase):
    def test_period_labels_are_honest(self):
        self.assertEqual(period_label("2018-12-28"), "design")
        self.assertEqual(period_label("2021-12-31"), "validation")
        self.assertEqual(period_label("2024-01-05"), "seen_history_replication")
        self.assertEqual(period_label("2026-07-24"), "seen_history_replication")
        with self.assertRaisesRegex(ValueError, "prospective protocol"):
            period_label("2026-07-31")
        self.assertEqual(
            period_label("2026-08-07", prospective_start="2026-08-07"),
            "prospective",
        )

    def test_positive_verdict_requires_every_gate(self):
        good = {
            "lift": 1.60, "ci_low": 1.05, "mature_events": 180,
            "positive_years": 6, "top_decile_median_excess_26w": .06,
            "recall": .25, "reversal_risk_delta": .02,
            "max_year_success_share": .25, "max_industry_success_share": .15,
            "leave_max_year_lift": 1.20, "leave_max_industry_lift": 1.15,
            "material_mechanism_contradiction": False,
            "material_data_gap": False,
        }
        self.assertEqual(decide_verdict(good)["verdict"], "值得进入前瞻影子雷达")
        good["ci_low"] = .98
        self.assertEqual(decide_verdict(good)["verdict"], "证据不足")

    def test_strong_negative_stops_version(self):
        bad = {
            "lift": .95, "ci_high": 1.10, "only_reduces_events": True,
            "climax_or_reversal_dominates": True, "material_data_gap": False,
        }
        self.assertEqual(decide_verdict(bad)["verdict"], "停止该版本")
```

- [ ] **Step 2: Write a deterministic cluster Bootstrap test**

Create four industries with paired events and controls. Call the Bootstrap twice with `n_boot=100`; assert exact dict equality and `ci_low <= estimate <= ci_high`.

- [ ] **Step 3: Run tests and confirm failure**

Run:

```bash
.venv/bin/python -m unittest test_industry_volume_resonance.MetricsAndVerdictTests -v
```

Expected: metrics functions are missing.

- [ ] **Step 4: Implement paired industry-cluster Bootstrap**

```python
def industry_cluster_bootstrap(events, controls, n_boot=2000):
    industries = np.array(sorted(events["industry_code"].unique()))
    rng = np.random.default_rng(20260727)
    values = []
    for _ in range(n_boot):
        draw = rng.choice(industries, size=len(industries), replace=True)
        event_parts = []
        control_parts = []
        for draw_index, code in enumerate(draw):
            part = events.loc[events["industry_code"] == code].copy()
            part["bootstrap_draw"] = draw_index
            event_parts.append(part)
            ids = set(part["event_id"])
            paired = controls.loc[controls["event_id"].isin(ids)].copy()
            paired["bootstrap_draw"] = draw_index
            control_parts.append(paired)
        event_sample = pd.concat(event_parts, ignore_index=True)
        control_sample = pd.concat(control_parts, ignore_index=True)
        values.append(binary_lift(event_sample, control_sample))
    # Return estimate and percentile 95% interval.
```

Keep censored labels out of every denominator. Calculate annual, market-regime and state strata, Recall, top-decile metrics, rank correlation, leave-one-year-out, leave-one-industry-out and success concentration.

`period_label()` must not infer a prospective start from the current date. Dates after `seen_replication_end` require a frozen `prospective_start`; dates between those boundaries are rejected instead of being relabeled after the fact.

- [ ] **Step 5: Implement the three-way verdict**

Encode every positive, negative and evidence-insufficient gate from design section 11. Return:

```python
{
    "verdict": "值得进入前瞻影子雷达",
    "reason_codes": ["lift_gate", "bootstrap_gate", "stability_gate"],
    "failed_gates": [],
}
```

Never infer a positive verdict when an input is missing.

- [ ] **Step 6: Run tests**

Run:

```bash
.venv/bin/python -m unittest test_industry_volume_resonance.MetricsAndVerdictTests -v
```

Expected: all metrics and verdict tests pass.

- [ ] **Step 7: Commit**

```bash
git add research/industry_volume_resonance/metrics.py \
  test_industry_volume_resonance.py
git commit -m "feat: evaluate industry resonance walk-forward evidence"
```

---

### Task 9: Freeze and Append Prospective Shadow Evidence

**Files:**
- Create: `research/industry_volume_resonance/protocol.py`
- Create: `research/industry_volume_resonance/shadow.py`
- Modify: `test_industry_volume_resonance.py`

**Interfaces:**
- Consumes: historical verdict, current source hashes, current complete-week rows.
- Produces:
  - `freeze_shadow_protocol(path: Path, summary: dict, source_hashes: dict, next_complete_week: str) -> dict`
  - `append_shadow_record(log_path: Path, record: dict, protocol: dict) -> dict`
  - `build_shadow_record(industry_weeks: pd.DataFrame, core_rows: pd.DataFrame, protocol: dict) -> dict`.

- [ ] **Step 1: Write failing freeze and append tests**

```python
class ProtocolAndShadowTests(unittest.TestCase):
    def test_negative_history_cannot_start_shadow(self):
        with tempfile.TemporaryDirectory() as td:
            with self.assertRaisesRegex(PermissionError, "historical gate"):
                freeze_shadow_protocol(
                    Path(td) / "protocol.json",
                    {"verdict": {"verdict": "停止该版本"}},
                    {"code": "abc", "data": "def"},
                    "2026-07-31",
                )

    def test_shadow_is_monotonic_and_idempotent(self):
        with tempfile.TemporaryDirectory() as td:
            log = Path(td) / "shadow.jsonl"
            protocol = {"prospective_start": "2026-07-31", "protocol_sha256": "p"}
            first = append_shadow_record(log, shadow_record("2026-07-31"), protocol)
            second = append_shadow_record(log, shadow_record("2026-07-31"), protocol)
            self.assertEqual(first["status"], "appended")
            self.assertEqual(second["status"], "already_present")
            with self.assertRaisesRegex(ValueError, "monotonic"):
                append_shadow_record(log, shadow_record("2026-07-24"), protocol)
```

- [ ] **Step 2: Run tests and confirm failure**

Run:

```bash
.venv/bin/python -m unittest test_industry_volume_resonance.ProtocolAndShadowTests -v
```

Expected: protocol and shadow modules are missing.

- [ ] **Step 3: Implement protocol freeze**

`freeze_shadow_protocol()` must:

- require historical verdict `值得进入前瞻影子雷达`;
- reject an existing protocol path;
- store code, source, membership and thresholds hashes;
- store the first allowed complete week;
- store `prospective_horizon_weeks=52` and `mature_event_min=100`;
- atomically write JSON.

- [ ] **Step 4: Implement locked JSONL append**

```python
def append_shadow_record(log_path, record, protocol):
    # Acquire an adjacent O_EXCL lock file.
    # Parse all existing lines and verify their hashes.
    # Reject dates earlier than protocol start or earlier than the last line.
    # Return already_present for an identical week and hash.
    # Reject a conflicting duplicate week.
    # fsync the appended line and parent directory, then release the lock.
```

Each line contains `week_end`, `generated_at_utc`, `protocol_sha256`, `source_hashes`, `industries`, `record_sha256` and no recommendation fields.

- [ ] **Step 5: Add forbidden-copy tests**

Serialize a shadow record and assert it contains none of:

```python
("推荐买入", "机构建仓", "仓位", "止损", "目标价")
```

- [ ] **Step 6: Run tests**

Run:

```bash
.venv/bin/python -m unittest test_industry_volume_resonance.ProtocolAndShadowTests -v
```

Expected: all protocol and shadow tests pass.

- [ ] **Step 7: Commit**

```bash
git add research/industry_volume_resonance/protocol.py \
  research/industry_volume_resonance/shadow.py \
  test_industry_volume_resonance.py
git commit -m "feat: lock prospective industry shadow evidence"
```

---

### Task 10: Assemble the Historical Pipeline and CLI

**Files:**
- Create: `research/industry_volume_resonance/pipeline.py`
- Create: `research/run_industry_volume_resonance.py`
- Modify: `test_industry_volume_resonance.py`

**Interfaces:**
- Consumes: all Tasks 1–9.
- Produces CLI stages:
  - `prepare`
  - `historical`
  - `freeze-shadow`
  - `shadow`
  - `report`
  - `validate`
- Produces machine files specified in design section 12.
- Produces:
  - `run_negative_controls(stock_week: pd.DataFrame, memberships: pd.DataFrame, benchmark_week: pd.DataFrame, primary_events: pd.DataFrame, seed: int = 20260727) -> dict`
  - deterministic results for membership shuffling, event-week shifting and the single-stock-plus-peer-confirmation ablation.

- [ ] **Step 1: Write a failing synthetic end-to-end test**

```python
class PipelineTests(unittest.TestCase):
    def test_historical_pipeline_writes_required_machine_files(self):
        with tempfile.TemporaryDirectory() as td:
            result = run_historical(
                stock_week=synthetic_stock_week_panel(),
                memberships=synthetic_memberships(),
                benchmark_week=synthetic_benchmark(),
                results_dir=Path(td),
                bootstrap_n=50,
            )
            required = {
                "industry_membership_history.csv.gz",
                "industry_week_features.csv.gz",
                "industry_events.csv.gz",
                "matched_industry_controls.csv.gz",
                "industry_experiment_metrics.csv",
                "industry_strata.csv",
                "negative_controls.json",
                "core_stock_rankings.csv.gz",
                "failed_industry_events.csv.gz",
                "excluded_industry_weeks.csv.gz",
                "industry_resonance_summary.json",
            }
            self.assertTrue(required.issubset({p.name for p in Path(td).iterdir()}))
            self.assertIn(
                result["verdict"]["verdict"],
                {"值得进入前瞻影子雷达", "证据不足", "停止该版本"},
            )
```

- [ ] **Step 2: Run the pipeline test and confirm failure**

Run:

```bash
.venv/bin/python -m unittest test_industry_volume_resonance.PipelineTests -v
```

Expected: `run_historical` is missing.

- [ ] **Step 3: Implement deterministic historical orchestration**

```python
def run_historical(
    *,
    stock_week,
    memberships,
    benchmark_week,
    results_dir,
    bootstrap_n=2000,
):
    stock_features = compute_stock_week_features(stock_week)
    expanded = expand_membership_to_weeks(
        memberships, stock_features[["week_end"]].drop_duplicates()
    )
    industry_weeks = build_industry_week_features(
        stock_features, expanded, ResonanceThresholds()
    )
    classified = classify_industry_weeks(industry_weeks, ResonanceThresholds())
    events = merge_industry_events(classified, max_gap_weeks=8)
    # Label, match, rank cores, summarize A/B/C/D, decide verdict and export.
```

Use atomic temporary-file replacement for JSON, CSV and gzip CSV outputs. Sort every output by stable keys before writing.

- [ ] **Step 4: Implement frozen negative controls**

Add `run_negative_controls()` to the historical pipeline:

1. group membership rows by `source` and `level`, sort by `industry_code, ts_code, in_date, out_date`, and deterministically permute the `ts_code` column within each group using seed `20260727`; keep industry codes, interval boundaries and group sizes fixed, then rerun aggregation, states, labels and matching;
2. deterministically circular-shift each industry's event weeks by a non-zero offset, then relabel and rematch without changing price data;
3. run the single-stock-trigger-plus-peer-confirmation ablation without changing A/B/C event definitions.

Persist sample counts, Precision, matched Lift and confidence intervals for the real result and both randomized controls. Assert that no shuffled or shifted row enters the primary event or primary metric tables. Negative controls are diagnostic evidence: their only permitted verdict input is `material_mechanism_contradiction`; when true it may turn the verdict into `证据不足`. They must never be used to retune thresholds.

Set `material_mechanism_contradiction=True` when either randomized control's matched-Lift point estimate is greater than or equal to the primary matched Lift. If either comparison cannot be calculated, set `material_data_gap=True`; never silently treat a missing control as a pass.

- [ ] **Step 5: Add negative-control tests**

On a seeded synthetic fixture with an embedded industry effect, assert:

- two calls return byte-identical JSON;
- the real matched Lift exceeds both randomized-control Lifts;
- shuffled membership changes no original input frame;
- a non-zero circular shift is used for every industry;
- the ablation writes a separate method name and never changes A/B/C event IDs.

- [ ] **Step 6: Implement explicit CLI paths and stages**

Default paths:

```python
root = Path(args.repo_root).resolve()
base = root / "research" / "industry_volume_resonance"
weekly_cache = root / "research" / "weekly_volume_radar" / "cache" / "weekly_cache.sqlite"
panel_cache = root / "research" / "weekly_volume_radar" / "cache" / "panel.sqlite"
source_cache = base / "cache" / "sources.sqlite"
results = base / "results"
html = root / "docs" / "研究-A股产业量能共振雷达-第二阶段.html"
```

`prepare` may call Tushare. Every other stage must operate from local caches. `freeze-shadow` requires `--confirm-freeze` and a positive historical verdict. `shadow` refuses an incomplete market week.

- [ ] **Step 7: Add CLI guard tests**

Patch `sys.argv` and assert:

- `freeze-shadow` without `--confirm-freeze` exits with parser error;
- `historical` opens formal DBs read-only;
- `shadow` before a positive protocol fails;
- no stage named `oos` exists.

- [ ] **Step 8: Run pipeline and CLI tests**

Run:

```bash
.venv/bin/python -m unittest test_industry_volume_resonance.PipelineTests -v
```

Expected: all pipeline tests pass.

- [ ] **Step 9: Commit**

```bash
git add research/industry_volume_resonance/pipeline.py \
  research/run_industry_volume_resonance.py \
  test_industry_volume_resonance.py
git commit -m "feat: assemble industry resonance research pipeline"
```

---

### Task 11: Render the Report and Validate the Audit Chain

**Files:**
- Create: `research/industry_volume_resonance/report.py`
- Create: `research/industry_volume_resonance/validate.py`
- Modify: `research/industry_volume_resonance/pipeline.py`
- Modify: `test_industry_volume_resonance.py`

**Interfaces:**
- Consumes: machine outputs and source snapshots.
- Produces:
  - `render_report(output_path: Path, summary: dict, metrics: pd.DataFrame, strata: pd.DataFrame, events: pd.DataFrame, cores: pd.DataFrame, failures: pd.DataFrame) -> Path`
  - `validate_research(repo_root: Path, base_dir: Path, formal_sources_before: dict, protected_paths_before: dict) -> dict`
  - final HTML and `run_manifest.json`.

- [ ] **Step 1: Write failing report tests**

```python
class ReportValidationTests(unittest.TestCase):
    def test_report_has_required_sections_and_forbidden_copy_is_absent(self):
        with tempfile.TemporaryDirectory() as td:
            path = render_report(
                Path(td) / "report.html",
                summary=summary_fixture(),
                metrics=metrics_fixture(),
                strata=strata_fixture(),
                events=events_fixture(),
                cores=core_fixture(),
                failures=failures_fixture(),
            )
            text = path.read_text(encoding="utf-8")
            for heading in (
                "一页结论", "产业量能消融", "年度与市况",
                "核心股票排序", "高潮与失败案例", "数据边界",
            ):
                self.assertIn(heading, text)
            for forbidden in ("推荐买入", "机构建仓", "仓位建议"):
                self.assertNotIn(forbidden, text)
```

- [ ] **Step 2: Write failing manifest validation tests**

Create temporary machine outputs, write correct hashes, validate success, then alter one byte and assert `all_ok=False` with `result_hashes=False`.

- [ ] **Step 3: Run tests and confirm failure**

Run:

```bash
.venv/bin/python -m unittest test_industry_volume_resonance.ReportValidationTests -v
```

Expected: report and validation modules are missing.

- [ ] **Step 4: Implement a self-contained HTML report**

Use escaped text and inline CSS only. Include:

- verdict, Lift and confidence interval;
- A/B/C industry event counts and matched metrics;
- `design`、`validation`、`seen_history_replication` labels;
- industry and market-regime strata;
- D1/D2/D3 comparison;
- success, false-positive, climax and failure charts;
- real-versus-randomized negative-control results;
- source coverage, missing membership, ST and terminal-price gaps;
- explicit statement that 2022—2026-07-24 is not pristine OOS;
- explicit statement that CPO and other prospective themes were not evaluated when their registry is empty;
- exact condition for starting or refusing prospective shadow.

- [ ] **Step 5: Implement audit validation**

Validate:

- all required machine files exist;
- manifest hashes match;
- source DB size, mtime and SHA equal their pre-run snapshots;
- membership source contains interval fields and no `current_static` rows in historical main analysis;
- every feature date is on or before its event date;
- every label starts after its event date;
- negative-control rows are isolated from primary event and metric tables; only the frozen `material_mechanism_contradiction` boolean may enter the verdict;
- first-stage tracked file hashes equal `protected_paths_before`;
- production tracked file hashes equal `protected_paths_before`;
- token bytes and common key patterns are absent;
- shadow log is monotonic and internally hashed;
- HTML verdict equals summary verdict.

- [ ] **Step 6: Run report tests**

Run:

```bash
.venv/bin/python -m unittest test_industry_volume_resonance.ReportValidationTests -v
```

Expected: all report and validation tests pass.

- [ ] **Step 7: Commit**

```bash
git add research/industry_volume_resonance/report.py \
  research/industry_volume_resonance/validate.py \
  research/industry_volume_resonance/pipeline.py \
  test_industry_volume_resonance.py
git commit -m "feat: report and validate industry resonance evidence"
```

---

### Task 12: Run Historical Research and Deliver the Frozen Result

**Files:**
- Create: `research/industry_volume_resonance/results/*`
- Create: `docs/研究-A股产业量能共振雷达-第二阶段.html`
- Modify: `.gitignore` only if the new cache paths are not already covered; do not ignore result summaries or the HTML report.

**Interfaces:**
- Consumes: completed pipeline and real read-only caches.
- Produces: historical verdict, machine outputs, report, manifest, and—only after a positive gate plus explicit freeze command—the prospective protocol.

- [ ] **Step 1: Run the complete专项 test suite**

Run:

```bash
.venv/bin/python test_industry_volume_resonance.py
```

Expected: all专项 tests pass.

- [ ] **Step 2: Run existing project regression gates**

Run each command separately:

```bash
.venv/bin/python test_rules.py
.venv/bin/python test_rules_mid.py
.venv/bin/python test_web.py
.venv/bin/python test_regime.py
```

Expected: all existing tests pass with their current counts or higher.

- [ ] **Step 3: Snapshot formal sources**

Record size, mtime and full SHA-256 for:

```text
market.db
research/weekly_volume_radar/cache/weekly_cache.sqlite
research/weekly_volume_radar/cache/panel.sqlite
research/weekly_volume_radar/results/oos_audit.json
research/weekly_volume_radar/results/oos_summary.json
```

Also hash every protected first-stage and production path named by the design. Write the snapshots into the new research cache as `formal_sources_before.json` and `protected_paths_before.json`; never edit the sources.

- [ ] **Step 4: Prepare point-in-time sources**

Run:

```bash
.venv/bin/python research/run_industry_volume_resonance.py prepare
```

Expected:

- Tushare `index_member_all` returns `in_date` and `out_date`;
- `benchmark_daily` contains沪深 300 open/high/low/close;
- no token appears in the cache or terminal output;
- permission failure yields a safe “证据不足” preparation report and stops before historical analysis.

- [ ] **Step 5: Run historical rolling replication**

Run:

```bash
.venv/bin/python research/run_industry_volume_resonance.py historical
```

Expected:

- result files are deterministic and sorted;
- rows through 2026-07-24 that were already visible carry `seen_history_replication`;
- summary has one of the three frozen verdicts;
- no prospective log is created.

- [ ] **Step 6: Render and validate**

Run:

```bash
.venv/bin/python research/run_industry_volume_resonance.py report
.venv/bin/python research/run_industry_volume_resonance.py validate
```

Expected: validator returns `all_ok: true`. If the historical verdict is negative or evidence-insufficient, do not run `freeze-shadow`.

- [ ] **Step 7: Run syntax and whitespace gates**

Run:

```bash
.venv/bin/python -m py_compile \
  research/run_industry_volume_resonance.py \
  research/industry_volume_resonance/*.py \
  test_industry_volume_resonance.py
git diff --check
```

Expected: both commands succeed.

- [ ] **Step 8: Independently inspect the result boundaries**

Confirm in the machine summary and HTML:

- the verdict follows the frozen gates;
- current static industries are absent from the historical main sample;
- failure and climax events remain present;
- core ranking never changes industry events;
- the real result and all three negative-control/ablation results are separately identified;
- an empty prospective theme registry is reported as “CPO 等主题尚未评价”;
- the report contains no trading recommendation;
- first-stage result hashes equal the pre-run snapshot.

- [ ] **Step 9: Commit the reproducible result**

Stage only source, tests, summaries, manifest and the HTML report. Exclude large caches and token-bearing files.

```bash
git add research/industry_volume_resonance \
  research/run_industry_volume_resonance.py \
  test_industry_volume_resonance.py \
  docs/研究-A股产业量能共振雷达-第二阶段.html
git commit -m "research: evaluate industry volume resonance"
```

- [ ] **Step 10: Start prospective shadow only when authorized by evidence**

If and only if the historical verdict is `值得进入前瞻影子雷达`, run:

```bash
.venv/bin/python research/run_industry_volume_resonance.py \
  freeze-shadow --confirm-freeze
```

Otherwise, record the negative or evidence-insufficient verdict and stop this version. Do not weaken gates to force a shadow launch.

---

## Final Verification Checklist

- [ ] `test_industry_volume_resonance.py` passes.
- [ ] Existing 8/20/web/regime suites pass.
- [ ] `py_compile` passes.
- [ ] `git diff --check` passes.
- [ ] Formal source hashes are unchanged.
- [ ] First-stage result hashes are unchanged.
- [ ] Historical industry membership has valid point-in-time intervals.
- [ ] 2022—2026-07-24 is labeled `seen_history_replication`.
- [ ] Fixed-member labels retain delisted and suspended members.
- [ ] Matching uses no future fields.
- [ ] Bootstrap is deterministic and clustered by industry.
- [ ] D3 cannot win unless it beats D1 and D2 on both required metrics.
- [ ] Prospective shadow cannot start after a negative or evidence-insufficient verdict.
- [ ] Shadow JSONL is monotonic, idempotent and append-only.
- [ ] Reports contain failures, climax cases and data gaps.
- [ ] Reports contain no buy, position, stop-loss or target-price advice.
- [ ] Final verdict is generated mechanically from the frozen gates.
