# T1 动工确认：广度指标与申万二级集群

## 任务与分支

- 任务：T1 阶段一——广度指标、板块集群、研究库和只读展示
- 基线提交：提交本确认文件后记录
- 工作分支：`codex/s3-stage1-data-clusters`
- 前置依赖：T0 已有 M3 三入口代码保持冻结；若 T0 产生新的 `static/home.html` 修订，T1 立即暂停界面步骤
- 用户授权状态：等待用户回复准确的“动工”
- 本次授权范围：只允许 T1 分支内开发、测试和研究库回填；不包含合并、部署、切换生产或 T2 开发

## 将创建

- `s3_research/__init__.py`：S3 研究包版本和公开接口
- `s3_research/db.py`：生产库只读连接、独立研究库模式和事务
- `s3_research/sources.py`：已批准 Tushare 来源和原始表写入
- `s3_research/audit.py`：覆盖率、缺失率、重叠一致性和时点审计
- `s3_research/breadth.py`：NH250、NL250、五日净值和研究闸门
- `s3_research/clusters.py`：申万二级历史成员和集群状态机
- `s3_research/runner.py`：全量回填、每日增量、校验和原子发布
- `s3_research/hook.py`：单飞、异步、失败隔离的 EOD 启动器
- `s3_research/read_model.py`：界面只读查询
- `s3_research_cli.py`：数据获取、回填、审计和复算命令
- `test_s3_research.py`：数据库边界、时点、公式、原子发布和钩子测试
- `test_s3_ui.py`：S3 API 和 M3 展示契约测试
- `docs/验收-T1-S3阶段一-20260728.md`：逐条自查和证据
- `S3_research.db`：独立研究库；作为本地数据产物，不进入生产数据库

## 将修改

- `eod.py`：只在现有收盘编排全部完成后增加一个研究线程启动调用；不进入 `run_eod_scan`
- `service.py`：只增加 `/api/s3/summary` 和 `/api/s3/clusters` 两个 GET 接口
- `static/home.html`：只增加 `#today` 的“S3 观察指标 · 不影响交易”和 `#pool` 的只读活跃集群

## 明确不动

- 生产策略：`rules_mid.py`、`strategies/`、`screener.py`
- 风控：现有仓位、熔断、开仓门和服务端拦截
- 生产数据管道：`market_data.py`、`data.py`、现有 EOD 数据处理步骤
- 定时配置：`service.py` 中既有调度条件、`schedule.sh`、`run_eod_trigger.sh`
- 回测核心：`backtest.py`、`backtest_mid.py`、`backtest_defense.py`
- 生产数据库：`market.db`、`screener.db` 及其表结构和数据
- 生产存储层：`storage.py`
- 参数：`params.py`
- 依赖：`requirements.txt`，不安装 npm/pip 包
- 旧股票池：`static/pool.html`
- T2：不创建 `s3_backtest/`，不跑 81 组，不产生回测结论
- 行业口径：不读取、不映射 `pool.industry`

## 验收自查表模板

- [ ] 生产库连接全部为 `mode=ro`
- [ ] 外部数据只写 `S3_research.db`
- [ ] 暖窗只初始化指标，不产生信号或交易
- [ ] `daily.vol` 只保存日期、代码、成交量
- [ ] 财务严格使用实际公告日可见数据
- [ ] 2015 ST、成分历史、财务缺失和幸存者偏差如实报告
- [ ] 3 个历史日期 NH250/NL250 人工复算一致
- [ ] 2 个申万二级集群激活和移出人工复算一致
- [ ] 坏批次不覆盖上一份好数据
- [ ] 钩子不等待、不抛错、单飞、失败只记日志
- [ ] `#today` 真实态和空态正确
- [ ] `#pool` 真实态和空态正确，且完全只读
- [ ] 桌面和手机布局通过
- [ ] `static/pool.html` 零差异
- [ ] 禁区文件零差异
- [ ] 零新增依赖
- [ ] 无演示数据和硬编码股票
- [ ] T1 新增测试全部通过
- [ ] 既有 8/8、20/20、834/834、6/6 基线回归不退化
- [ ] `git diff --check` 通过
- [ ] 文件清单审计通过
- [ ] 分支未合并、未部署、未切换生产

用户回复“动工”前，本确认不授权任何业务代码、测试代码、数据库或数据写入。
