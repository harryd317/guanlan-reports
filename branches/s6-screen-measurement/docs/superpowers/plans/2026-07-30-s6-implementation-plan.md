# S6 初筛有效性测量实施计划

> 分支：`codex/s6-screen-measurement`
>
> 性质：纯测量研究。不得产生买卖点、模拟交易、仓位、资金曲线、影子记录或生产接线。

## 目标

依据已冻结的《S6-初筛有效性测量研究-预声明》和动工确认，构建可复现的
2015—2025 初筛测量流水线，只写 `S6_research.db`，交付数据审计、全样本与
分层统计、冻结标准判定和验收证据。

## 全局约束

- 只允许新增 `s6_measurement/`、`s6_measurement_cli.py`、
  `test_s6_measurement.py` 和 S6 文档；不得修改任何既有业务文件或依赖文件。
- 新数据仅限已批准的 Tushare `income`、`forecast`、`daily_basic` 和批准字段；
  原始响应只写 `S6_research.db`。
- `market.db`、T1 `S3_research.db` 和 S3 v0.3 封存对象只读；禁止写入生产库、
  S2/S3/S5 数据库、制品或账本。
- 财务可见日使用 `f_ann_date`，缺失才回退 `ann_date`，且不得晚于快照日。
- 处罚否决项已经删除。报告必须披露：没有可复现的历史时点权威完整数据源；
  该项边际贡献低；证监会/交易所/巨潮接口仅登记为未来可选补充，不进入 S6。
- 快照严格从 T1 `breadth_daily` 的 2015-01-05 起每 10 个交易日一张，到
  2025-12-31 为止。
- 复权价统一为 `market_daily.close × adj_factors.adj_factor`；不得读取
  2025-12-31 之后数据。
- S3 控制组只允许读取封存的 `captures_by_hits[3]`，读取前核验：
  - `S3_v0_3_prepared.pkl`：
    `9403949d98bc70e459afe83a5a4da117216ced49ac11bb935559cdde3a2a54e0`
  - `s3_backtest/data.py`：
    `a8c0744db56e75d9f75122218e7b3ba08c964fd93b4bc96ee88c5e4b5e46e36c`
- 冻结判定三项同时通过才算有效：
  1. 60 日超额中位数 `>= 0.02`；
  2. 60 日正收益率 `>= 0.52`；
  3. 2015—2025 中 60 日年度超额为负的年份不超过 4 年。
- 零新增依赖；测试必须输出干净；每项代码任务遵循 RED→GREEN。

### Task 1：研究库、只读边界与获批数据源

**文件**

- 新建：`s6_measurement/__init__.py`
- 新建：`s6_measurement/config.py`
- 新建：`s6_measurement/db.py`
- 新建：`s6_measurement/sources.py`
- 新建：`s6_measurement_cli.py`
- 新建：`test_s6_measurement.py`

**要求**

1. 在 `config.py` 固化日期、阈值、三个源表字段、S3 哈希和数据库文件名。
2. `db.py` 拒绝任何非 `S6_research.db` 写路径；外部 SQLite 统一用
   `file:...?...mode=ro`；建立原始表、批次审计表、派生/统计表和运行清单表。
3. 原始表字段必须与审批完全一致：
   - `income_raw`：
     `ts_code,ann_date,f_ann_date,end_date,report_type,comp_type,revenue,oper_cost,n_income_attr_p,update_flag`
   - `forecast_raw`：
     `ts_code,ann_date,end_date,type,p_change_min,p_change_max,net_profit_min,net_profit_max,last_parent_net,first_ann_date`
   - `daily_basic_raw`：`ts_code,trade_date,total_mv`
4. `sources.py` 显式指定接口和字段，分批可断点续取；先在内存校验字段、日期、
   主键、数值和批次指纹，合格后原子写入；坏批次不得覆盖既有合格数据。
5. 调用参数、返回行数、去重情况、缺失、覆盖区间、SHA-256、状态和时间进入
   审计表；同一批准请求可安全重跑，不得扩展字段。
6. `daily_basic` 只允许按传入的 S6 快照日拉取；快照日来自只读 T1
   `breadth_daily`，不得自行生成另一套日历。
7. CLI 至少提供 `init-db`、`fetch-income`、`fetch-forecast`、
   `fetch-daily-basic`、`audit-data`；默认不执行网络请求。
8. 测试使用临时库和假客户端，证明严格字段、断点续取、坏批次隔离、只读外部
   连接、写路径拒绝和审计指纹。

**验收**

- `python -m unittest -q test_s6_measurement.S6DatabaseTests`
- `python -m unittest -q test_s6_measurement.S6SourceTests`
- `python -m unittest -q test_s6_measurement.S6CliBoundaryTests`

### Task 2：时点筛选、T1 成品复用与三组样本

**文件**

- 新建：`s6_measurement/screen.py`
- 修改：`s6_measurement/config.py`
- 修改：`s6_measurement/db.py`
- 修改：`s6_measurement_cli.py`
- 修改：`test_s6_measurement.py`

**要求**

1. 从 T1 `breadth_daily` 读取严格每 10 个交易日快照及门状态；从 T1 成品读取
   `hit_nh250`、活跃二级行业和成员关系，不重算 NH250、不读取
   `pool.industry`。
2. 减法任一命中即剔除：
   - 最新已公开年报亏损，或最近两个已公开年报营收连续下滑；
   - 快照日 ST/*ST；
   - 上市未满 365 个自然日；
   - `total_mv < 1_000_000` 万元；
   - 最近 20 个全市场交易日平均 `amount < 300_000` 千元，停牌日按零；
   - 120 个全市场交易日复权涨幅百分位 `> 80%`。
3. 必需字段缺失记为“不具备测量资格”，保存原因和计数，不插补、不猜测。
4. 加法要求盈利改善、趋势、活跃集群同时成立：
   - 盈利改善三条任一；单季值按 Q1、H1-Q1、Q3-H1、FY-Q3 还原；
   - 预增/略增 `p_change_min > 0`，扭亏
     `last_parent_net < 0 and net_profit_min > 0`；
   - 趋势为 T1 NH250 或同一快照合格行情宇宙 RS120 百分位 `>= 85%`；
   - 属于快照日 T1 已物化活跃申万二级集群。
5. 建立三组观察：
   - 主池：减法合格且加法全满足；
   - 质地合格池：只通过减法；
   - S3 同期捕获池：只读并校验两个冻结哈希后提取
     `captures_by_hits[3]` 的快照日代码。
6. S3 哈希不符、封存对象缺失或结构不符立即失败；不得重建或替代。
7. 持久化每个“快照×股票×组别”的入选、理由、门状态、所用可见财务记录和
   缺失原因，便于逐行复算。
8. 测试至少覆盖三种单季还原、可见日回退、未来公告排除、预告规则、
   六项减法、RS 边界 80/85、停牌零成交、T1 行映射和 S3 哈希失败。

**验收**

- `python -m unittest -q test_s6_measurement.S6FinancialPointInTimeTests`
- `python -m unittest -q test_s6_measurement.S6ScreenTests`
- `python -m unittest -q test_s6_measurement.S6FrozenControlTests`

### Task 3：收益测量、分层统计、冻结判定与文档生成

**文件**

- 新建：`s6_measurement/measure.py`
- 新建：`s6_measurement/report.py`
- 修改：`s6_measurement/db.py`
- 修改：`s6_measurement_cli.py`
- 修改：`test_s6_measurement.py`

**要求**

1. 对三组观察计算 20/60/120 个全市场交易日后的复权收益；停牌期间价格前向
   保持，退市后或目标日无可靠价格记 `N/A`。
2. 禁止使用 2025-12-31 之后价格；晚期越界窗口显式记 `N/A` 并计算覆盖率。
3. 每个快照和期限用当日已上市且起止价格有效的全部 A 股算术平均收益作为
   全市场等权基准；个股超额为个股收益减对应基准。
4. 输出三组的全样本、逐年度、门开/关分切，指标包括观察数、有效数、覆盖率、
   收益中位数、正收益率、超额中位数；另输出逐快照分布。
5. 冻结过闸结论只使用全样本主池 60 日指标和 11 个年度结果；不得后移阈值、
   换统计量或补参数。
6. `report.py` 从数据库内的运行清单和统计生成：
   - `docs/测量报告-S6-初筛有效性-20260730.md`
   - `docs/验收-S6-初筛有效性测量-20260730.md`
7. 报告必须披露处罚否决项删除理由、覆盖缺口和影响边界；将上交所/深交所监管
   查询登记为未来可选补充，明确“未调用、未入库、未参与本次 S6”。
8. 报告明确 S6 是测量而非策略，不含买卖、交易、仓位、净值或影子记录。
9. 测试覆盖复权公式、20/60/120 日目标日、停牌前向保持、年末越界、
   等权基准、分层指标、三项冻结判定和强制披露。

**验收**

- `python -m unittest -q test_s6_measurement.S6ReturnMeasurementTests`
- `python -m unittest -q test_s6_measurement.S6AggregationTests`
- `python -m unittest -q test_s6_measurement.S6ReportTests`
- `python -m unittest -q test_s6_measurement`

### Task 4：批准数据取数、全量测量与最终验收

**文件**

- 本地生成（不提交）：`S6_research.db`
- 生成并提交：`docs/测量报告-S6-初筛有效性-20260730.md`
- 生成并提交：`docs/验收-S6-初筛有效性测量-20260730.md`
- 必要时仅修复 S6 新文件中的实际运行缺陷，并补对应回归测试

**要求**

1. 对生产库、T1 数据库和两个 S3 封存文件记录取数前 SHA-256；取数和测量后
   再核验，必须完全一致。
2. 只调用批准的三个 Tushare 接口和字段，按 Task 1 审计机制断点续取。
3. 数据覆盖验收至少包括：日期范围、股票数、行数、主键重复、必需字段缺失、
   批次失败/重试、源指纹、快照覆盖和可见日越界。
4. 通过覆盖验收后执行一次 2015—2025 全量测量，生成三组、三期限、年度和
   门状态结果；不得根据结果修改规则或重跑不同口径。
5. 独立抽查并写入验收报告：
   - 财务可见性、单季还原、减法、加法各至少 3 例；
   - 快照节奏、T1 新高/集群/成员各至少 3 日；
   - S3 控制组三个日期逐行一致；
   - 20/60/120 日收益和等权基准各至少 3 例；
   - 晚期窗口、停牌、退市、财务缺失、ST 历史缺口计数。
6. 按冻结三项标准给出通过或不通过，不做补救版本。
7. 最终运行：
   - `python -m unittest -q test_s6_measurement`
   - 既有只读定向基线测试
   - `git diff --check`
   - 相对任务基线的新增依赖、文件清单、禁区差异审计
   - S3 制品、生产库、T1/S5 制品哈希前后一致
8. 提交代码、报告和验收证据，停在 S6 分支；不得合并、部署、接生产或影子。

**验收**

- 数据审计与全量测量命令退出码为 0；
- 两份报告完整生成；
- 所有测试和差异审计通过；
- `S6_research.db` 未进入 Git；
- 分支停在独立提交等待用户裁决。
