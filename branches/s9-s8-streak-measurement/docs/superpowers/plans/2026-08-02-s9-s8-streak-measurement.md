# S9 S8池连阳尾盘信号测量 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 在冻结数据边界内测量 S8 池连阳尾盘的 16 个固定单元，并与同期 S7 全市场口径逐单元对照。

**Architecture:** 新建完全离线的 `s9_measurement` 包。输入 SQLite 均以只读模式打开，逐 S6 快照日重算历史温度、生成 S7 对照与 S8 子集队列，再计算固定期限收益和三门判定；所有中间表只写独立 S9 研究库。报告层从研究库导出 CSV、JSON 和 Markdown，不接入服务或生产调度。

**Tech Stack:** Python 3.11、标准库 `sqlite3`/`csv`/`json`/`hashlib`、现有 `industry_archives` 温度计公式、`unittest`。

## Global Constraints

- 性质固定为纯测量：零交易、无买卖点、无影子、无生产接入、不扫描参数。
- 输入仅为冻结 T1、S6、温度计 sidecar 和既有 S7 价格基线 `market.db`，全部只读。
- 观察日固定为 S6 的 268 个快照日；不得把结果称为逐日回测。
- 总体固定为 `s7_control` 与其严格子集 `s8_pool`。
- 单元固定为 `2/3 连阳 × <=1.5%/>1.5% × T+1/T+3/T+5/T+10`。
- 三门固定为净均值至少 0.15%、净正收益率至少 52%、11 年齐全且负净均值年份不超过 4。
- 唯一数据库写路径为 `S9_measurement.local.db`；不改生产、策略、风控、通知和数据管道。

---

### Task 1: 冻结配置与只读数据库边界

**Files:**
- Create: `s9_measurement/__init__.py`
- Create: `s9_measurement/config.py`
- Create: `s9_measurement/db.py`
- Create: `test_s9_measurement.py`

**Interfaces:**
- Produces: `FROZEN_CONFIG: dict`、`CONFIG_FINGERPRINT: str`。
- Produces: `connect_external_ro(path) -> sqlite3.Connection`。
- Produces: `init_research_db(path) -> Path`，仅接受文件名 `S9_measurement.local.db`。
- Produces: `research_transaction(path)`。

- [ ] **Step 1: 写失败测试**

  测试 `connect_external_ro` 的 `PRAGMA query_only` 为 1，并断言写 SQL 失败；测试 `init_research_db` 拒绝 `market.db`、`S3_research.db` 和任意非 S9 文件名。

- [ ] **Step 2: 验证红灯**

  Run: `../../.venv/bin/python -m unittest test_s9_measurement.S9DatabaseBoundaryTests -v`

  Expected: 因 `s9_measurement` 尚不存在而失败。

- [ ] **Step 3: 最小实现**

  在 `config.py` 固定区间、268 个观察日预期、11 年、两总体、两连阳层级、两涨幅桶、四期限、0.15% 费用和三门阈值；配置以排序 JSON 计算 SHA-256。`db.py` 使用 SQLite URI `mode=ro` 和 `PRAGMA query_only=ON`，研究库建立运行、覆盖、温度、观察、队列、收益、年度统计和判定表。

- [ ] **Step 4: 验证绿灯并提交**

  Run: `../../.venv/bin/python -m unittest test_s9_measurement.S9DatabaseBoundaryTests -v`

  Expected: PASS。

### Task 2: 历史温度和样本资格纯函数

**Files:**
- Create: `s9_measurement/thermometer.py`
- Create: `s9_measurement/eligibility.py`
- Modify: `test_s9_measurement.py`

**Interfaces:**
- Produces: `compute_heat_for_date(sidecar, market, trade_date) -> dict[str, dict]`。
- Produces: `streak_flags(bars) -> tuple[bool, bool]`。
- Produces: `last_gain_bucket(pct_chg) -> str`。
- Produces: `resolve_st_status(...) -> StStatus`。
- Produces: `previous_year_top10(price_rows) -> set[str]`。

- [ ] **Step 1: 写失败测试**

  用手算 SQLite 夹具证明：温度计缺 121 根价格时返回 `unknown`；四灯分别形成 cold/warm/hot/scorching；三连同时属于二连；1.5 边界落入 `le_1_5`；ST 缺口不被推断为非 ST；上一年收益并列时按代码升序定序。

- [ ] **Step 2: 验证红灯**

  Run: `../../.venv/bin/python -m unittest test_s9_measurement.S9EligibilityTests test_s9_measurement.S9ThermometerTests -v`

  Expected: 因函数缺失而失败。

- [ ] **Step 3: 最小实现**

  温度模块复用现有冻结公式的集群、21 日成交量、5 日成交额占比和 121 根龙头强度定义，但显式传入历史日期；资格模块保持纯函数，所有缺失状态返回明确原因。

- [ ] **Step 4: 验证绿灯并提交**

  Run: `../../.venv/bin/python -m unittest test_s9_measurement.S9EligibilityTests test_s9_measurement.S9ThermometerTests -v`

  Expected: PASS。

### Task 3: 观察、队列和固定期限收益流水线

**Files:**
- Create: `s9_measurement/pipeline.py`
- Modify: `test_s9_measurement.py`

**Interfaces:**
- Produces: `run_measurement(research_db, s6_db, t1_db, sidecar_db, market_db) -> dict`。
- Consumes: Task 1 的只读连接和研究表；Task 2 的温度与资格函数。

- [ ] **Step 1: 写失败测试**

  构造两年最小市场夹具，覆盖 S7 合格但 S8 因热度、行业或上年前十被排除的行，以及 S8 合格行；手算 T+1/3/5/10 复权收益、0.15% 扣减和停牌目标价。断言 S8 队列严格包含于 S7、一个信号只进一个涨幅桶、三连进入两个连阳层级、收益行数等于队列行数乘四。

- [ ] **Step 2: 验证红灯**

  Run: `../../.venv/bin/python -m unittest test_s9_measurement.S9PipelineTests -v`

  Expected: 因 `run_measurement` 缺失而失败。

- [ ] **Step 3: 最小实现**

  流水线先记录四个输入哈希，再读取 S6 观察日和市值；逐日写温度与覆盖，逐股写唯一观察审计，生成两个总体的队列和四期限收益。异常时运行清单写 `failed`，不保留“complete”状态。

- [ ] **Step 4: 验证绿灯并提交**

  Run: `../../.venv/bin/python -m unittest test_s9_measurement.S9PipelineTests -v`

  Expected: PASS。

### Task 4: 年度统计、三门和成对结论

**Files:**
- Create: `s9_measurement/statistics.py`
- Modify: `s9_measurement/pipeline.py`
- Modify: `test_s9_measurement.py`

**Interfaces:**
- Produces: `aggregate_and_qualify(connection, run_id) -> dict`。
- Produces: 352 行 `annual_statistics` 和 32 行 `qualification_results`。

- [ ] **Step 1: 写失败测试**

  用独立字面量构造门槛上下边界，证明净均值恰为 0.15%、正收益率恰为 52%、负年份恰为 4 时通过；缺一年、5 个负年份或任一数值低于门槛时失败。证明只有 `s7_control` 失败且同单元 `s8_pool` 通过才标记 `reversed=1`。

- [ ] **Step 2: 验证红灯**

  Run: `../../.venv/bin/python -m unittest test_s9_measurement.S9StatisticsTests -v`

  Expected: 因统计函数缺失而失败。

- [ ] **Step 3: 最小实现**

  对所有总体、连阳层级、涨幅桶、期限和 11 年做笛卡尔补齐；无样本年份保留空指标并令 `years_complete=0`。总结果只读冻结阈值，不提供命令行覆盖。

- [ ] **Step 4: 验证绿灯并提交**

  Run: `../../.venv/bin/python -m unittest test_s9_measurement.S9StatisticsTests -v`

  Expected: PASS。

### Task 5: CLI、正式运行和可复读报告

**Files:**
- Create: `s9_measurement/report.py`
- Create: `s9_measurement_cli.py`
- Create: `reports/s9-s8-streak-measurements-20260802.csv`
- Create: `reports/s9-s8-streak-qualification-20260802.csv`
- Create: `reports/s9-s8-streak-source-coverage-20260802.json`
- Create: `reports/s9-s8-streak-run-manifest-20260802.json`
- Create: `docs/测量报告-S9-S8池连阳尾盘信号-20260802.md`
- Create: `docs/验收-S9-S8池连阳尾盘信号测量-20260802.md`
- Modify: `test_s9_measurement.py`

**Interfaces:**
- Produces: CLI `run` 与 `export` 子命令，所有输入路径必须显式传入。
- Produces: `export_artifacts(...) -> dict[path, sha256]`。

- [ ] **Step 1: 写失败测试**

  测试导出 CSV 行数与研究表一致、JSON 可解析、运行清单包含输入前后哈希和配置指纹、报告明确“纯测量”及覆盖缺口，并拒绝未完成运行。

- [ ] **Step 2: 验证红灯**

  Run: `../../.venv/bin/python -m unittest test_s9_measurement.S9ReportTests -v`

  Expected: 因报告接口缺失而失败。

- [ ] **Step 3: 最小实现并运行正式测量**

  Run: `../../.venv/bin/python s9_measurement_cli.py run --research-db S9_measurement.local.db --s6-db ../s6-screen-measurement/S6_research.db --t1-db ../s3-stage1-data-clusters/S3_research.db --sidecar-db ../../industry_thermometer_sidecar.db --market-db ../../market.db`

  随后运行 `export` 生成四份机器可读产物和两份 Markdown。报告逐项回答 16 对单元是否扭转，并披露所有无效与缺口状态。

- [ ] **Step 4: 验证正式产物**

  Run: `../../.venv/bin/python -m unittest test_s9_measurement.py -v`

  Run: `sqlite3 -readonly S9_measurement.local.db 'PRAGMA integrity_check;'`

  Run: `git diff --check`

  Expected: 全部 PASS/`ok`，四个输入的正式运行前后哈希逐项一致。

### Task 6: 验收、提交与公开镜像验真

**Files:**
- Modify: `docs/验收-S9-S8池连阳尾盘信号测量-20260802.md`
- Modify: `reports/s9-s8-streak-run-manifest-20260802.json`

**Interfaces:**
- Produces: 预声明提交、代码提交、结果提交和镜像提交组成的哈希链。

- [ ] **Step 1: 全量验证**

  运行 S9 测试、S8 定向回归、镜像同步测试、SQLite 完整性、表级守恒、输入复哈希、`git diff --check` 和敏感信息扫描。记录真实命令、测试数和结果。

- [ ] **Step 2: 提交正式产物**

  只暂存 S9 代码、测试、`docs/` 和 `reports/`；确认 `S9_measurement.local.db` 未暂存，且主工作树用户文件不受影响。

- [ ] **Step 3: 镜像预演与推送**

  先执行 `scripts/report_mirror_sync.py --dry-run` 核对公开白名单，再执行 `--push`。不得包含数据库、环境文件、令牌或未跟踪的用户文档。

- [ ] **Step 4: 全新克隆复读**

  从远端全新克隆 `guanlan-reports`，核对镜像 HEAD、S9 必需路径和逐文件 SHA-256；把复读结果写入验收报告并形成最终提交/镜像哈希。
