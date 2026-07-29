# A股周线量能雷达第一阶段实施计划

> **执行方式：** 在 `codex/weekly-volume-radar-research` 独立 worktree 连续执行。每个任务先写失败测试，再写最小实现；2015—2021 冻结后只运行一次 2022+。

**目标：** 交付只读、可复现的全市场周线事件研究，完成 A—E 消融、单次样本外检验、机器结果和 HTML 报告，不改生产系统。

**架构：** `research/weekly_volume_radar` 是独立研究包。正式库只读，周线和外部元数据写入被忽略的独立缓存；冻结协议和最终结果写入可跟踪目录。流水线分成 `prepare → design → freeze → oos → report → validate`，OOS 由审计锁阻止重跑。

**技术栈：** Python 3、SQLite、pandas、NumPy、SciPy、Tushare；HTML/SVG 由本地模板生成，不引入前端运行时。

---

## Task 1：建立研究包骨架和冻结配置

**文件：**

- 新建：`research/weekly_volume_radar/__init__.py`
- 新建：`research/weekly_volume_radar/config.py`
- 新建：`research/weekly_volume_radar/io.py`
- 新建：`research/weekly_volume_radar/results/.gitkeep`
- 新建：`research/weekly_volume_radar/.gitignore`
- 新建：`test_weekly_volume_radar.py`

**步骤：**

1. 测试固定随机种子、设计/OOS 日期、参数离散范围和三种判决文本。
2. 运行测试，确认因模块缺失而失败。
3. 实现冻结常量、路径、原子 JSON/CSV 写入和文件哈希。
4. 运行测试，确认通过。

## Task 2：完整周重采样和只读源审计

**文件：**

- 新建：`research/weekly_volume_radar/weekly.py`
- 新建：`research/weekly_volume_radar/source_audit.py`
- 修改：`test_weekly_volume_radar.py`

**步骤：**

1. 测试节假日短周、未结束周排除、周 OHLC/成交额、复权、市场周末对齐。
2. 测试 SQLite URI `mode=ro`，以及研究前后的 size/mtime/SHA 比较。
3. 运行新增测试，确认失败。
4. 实现流式日线聚合到独立 `weekly_bars` 缓存；正式库不建表、不写入。
5. 运行测试和小样本集成检查。

## Task 3：历史市值和证券状态缓存

**文件：**

- 新建：`research/weekly_volume_radar/metadata.py`
- 修改：`test_weekly_volume_radar.py`

**步骤：**

1. 测试 `total_mv` 万元转亿元、四档边界、上市 52 周、退市区间和脱敏错误。
2. 测试 API 缺周重试、缓存命中不重复请求、Token 不进入 manifest。
3. 实现 Tushare `stock_basic`、`daily_basic` 周快照和可选 `namechange` 缓存。
4. 运行测试。

## Task 4：特征、标签和事件合并

**文件：**

- 新建：`research/weekly_volume_radar/features.py`
- 新建：`research/weekly_volume_radar/labels.py`
- 新建：`research/weekly_volume_radar/events.py`
- 修改：`test_weekly_volume_radar.py`

**步骤：**

1. 测试 A 的历史基线排除最近 4 周、斜率和分位次数。
2. 测试 B 的量价效率、涨跌周额比和缩量整理。
3. 测试 C 的 MA、平台、安全位置和过热。
4. 测试标签从下一交易日开盘开始、右删失和最大不利波动。
5. 测试连续信号 8 周合并，事件日期取首周。
6. 实现并运行测试。

## Task 5：案例、匹配和指标

**文件：**

- 新建：`research/weekly_volume_radar/cases.py`
- 新建：`research/weekly_volume_radar/matching.py`
- 新建：`research/weekly_volume_radar/metrics.py`
- 修改：`test_weekly_volume_radar.py`

**步骤：**

1. 测试匹配不读取未来字段，优先同周/行业/市值/上市年限。
2. 测试 Precision、Recall、Base rate、Lift、提前量和分层统计。
3. 测试按股票整簇 Bootstrap 的确定性和 CI。
4. 测试不稳定归因和 leave-one-year-out。
5. 实现 40 案例选择、人工阶段注释模板及 1:3 对照。

## Task 6：A—D 设计段选择和冻结协议

**文件：**

- 新建：`research/weekly_volume_radar/experiments.py`
- 新建：`research/weekly_volume_radar/protocol.py`
- 修改：`test_weekly_volume_radar.py`

**步骤：**

1. 测试 A→B→C→D 只能逐级增加条件。
2. 测试稳定高原选择的 5% 规则和确定并列顺序。
3. 测试设计选择器拒绝读取 2022+ 行。
4. 测试冻结协议包含参数、代码/数据哈希和 `oos_viewed=false`。
5. 实现设计段有限搜索和协议冻结。

## Task 7：OOS 一次性审计和 E 对照

**文件：**

- 新建：`research/weekly_volume_radar/oos.py`
- 新建：`research/weekly_volume_radar/system_compare.py`
- 修改：`test_weekly_volume_radar.py`

**步骤：**

1. 测试无冻结协议拒绝、无确认参数拒绝、已有审计拒绝。
2. 测试审计开始文件先于指标写入，失败也不能静默重跑。
3. 测试共同覆盖范围和 `technical_capture_proxy` 标记。
4. 测试首次日期、周差、价格、MA20 和平台差。
5. 实现单次 OOS 和 E 比较。

## Task 8：CLI、机器结果和 HTML 报告

**文件：**

- 新建：`research/run_weekly_volume_radar.py`
- 新建：`research/weekly_volume_radar/pipeline.py`
- 新建：`research/weekly_volume_radar/report.py`
- 新建：`research/weekly_volume_radar/validate.py`
- 新建：`docs/研究-A股周线量能雷达-第一阶段-20260726.html`
- 修改：`test_weekly_volume_radar.py`

**步骤：**

1. 测试 CLI 阶段顺序和恢复点。
2. 测试机器输出字段字典、失败清单、manifest SHA。
3. 测试 HTML 必含一页结论、案例/失败图、消融、OOS、分层、E、风险和唯一判决。
4. 实现流水线和内嵌 SVG 报告。
5. 运行设计段，人工检查 40 案例标注，冻结协议。
6. 运行一次 OOS，生成所有结果和报告。

## Task 9：终验和提交

**步骤：**

1. 运行研究测试、原生产四套测试。
2. 运行 `py_compile`、`git diff --check`。
3. 比较正式数据库研究前后 size/mtime/SHA。
4. 检查 Git diff 只含研究代码、测试、数据结果和报告。
5. 扫描 Token、密钥、账户信息和绝对密钥路径。
6. 检查 OOS 审计只有一次、所有周完整、事件已合并、失败清单非空。
7. 可视化检查 HTML 桌面和窄屏布局。
8. 独立复核研究口径和结论映射。
9. 删除 worktree 本地 `.venv` 软链接，提交最终分支。

## 暂停/回滚条件

- 正式数据库发生 size、mtime 或 SHA 变化：立即停止，报告污染，不继续 OOS。
- 历史市值缺失导致 OOS 可用事件少于 200：不放宽阈值，判为证据不足。
- OOS 命令已经产生审计文件：不得删除、重跑或改参数。
- 行业时点化失败：行业结果保持探索性，不得升级为主证据。
- 现有系统只有成交记录：E 标为下界/案例对照，不得声称全市场 Precision 优劣。
