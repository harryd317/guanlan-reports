# 产业温度计与产业档案页实施计划

> **执行边界：** 只读展示；不改策略、风控、选股、交易和收盘任务。

**目标：** 在新前端股票池中增加产业温度计与 Markdown 档案阅读器，并从个股详情深链到匹配档案。

**架构：** 新建 `industry_archives.py`，用 SQLite `mode=ro` 读 T1 库和既有 `market.db`，请求时计算四灯并使用内存缓存。同一模块扫描 `docs/产业档案/*.md`，解析限定 YAML 头并渲染安全 Markdown。`service.py` 只增加三个 GET API，`static/home.html` 增加第三子页签、双列阅读器和个股深链。

**技术栈：** Python 3.11、SQLite、FastAPI、`markdown-it-py`、原生 HTML/CSS/JavaScript、`unittest`。

---

## Task 1：锁定温度计数据合约

**文件：**

- 新建：`test_industry_archives.py`
- 新建：`industry_archives.py`

### Step 1：写最小 T1 fixture 和失败测试

在临时 SQLite 库中创建 T1 侧 `sw_industry`、`sw_membership`、`cluster_daily`、`daily_vol`、`stock_basic`，以及行情侧 `market_daily`、`adj_factors`。构造足够行业与 121 个交易日，使四灯、前 5 量比直接烫、冷、热、缺数都有确定样本。

测试必须断言：

- `data_date` 取 T1 集群/量比与行情成交额/复权表的最新公共日；
- 成员关系按 `in_date/out_date` 时点取值；
- 量比分母排除当日，前 5 不亮量比灯但等级为烫；
- 5 日成交占比严格递增才亮灯；
- RS 使用 121 个有效收盘，并按全市场横截面求分位；
- 必要指标缺失时温度为 `unknown`，不是 `cold`。

### Step 2：运行测试，确认 RED

```bash
.venv/bin/python -m unittest test_industry_archives.IndustryThermometerTests -v
```

预期：因 `industry_archives` 不存在或 API 缺失而失败。

### Step 3：实现只读连接与四灯计算

最小实现包含：

- `resolve_t1_path()`；
- `connect_t1_readonly(path)`；
- `latest_common_date(t1_con, market_con)`；
- `compute_thermometer(t1_path, market_path)`；
- 文件指纹内存缓存。

不创建表，不执行 `INSERT/UPDATE/DELETE/REPLACE`，不导入任何生产存储或交易模块。

### Step 4：运行测试，确认 GREEN

```bash
.venv/bin/python -m unittest test_industry_archives.IndustryThermometerTests -v
```

### Step 5：提交

```bash
git add industry_archives.py test_industry_archives.py
git commit -m "feat: compute read-only industry thermometer"
```

## Task 2：锁定产业档案协议

**文件：**

- 修改：`test_industry_archives.py`
- 修改：`industry_archives.py`
- 修改：`requirements.txt`

### Step 1：写失败测试

用临时目录构造合法与非法 Markdown 档案，断言：

- `industry` 标量、逗号列表、内联列表和多行列表都能解析；
- 无 YAML 头、缺字段、非法日期和重复 ID 被记入 `errors`；
- 新增文件下次扫描自动出现；
- 第一个 H1 作为标题，无 H1 时使用文件名；
- Markdown 表格渲染为 `<table>`，原始 `<script>` 不会执行；
- `../`、绝对路径和未知 ID 不能读取目录外文件；
- 个股代码按 T1 时点行业匹配单行业与多行业档案。

### Step 2：确认 RED

```bash
.venv/bin/python -m unittest test_industry_archives.ArchiveCatalogTests -v
```

### Step 3：实现限定 YAML 解析、目录扫描和 Markdown 渲染

显式把 `markdown-it-py>=4.0` 加入 `requirements.txt`。使用 `MarkdownIt("commonmark", {"html": False})` 并启用 `table`。

### Step 4：确认 GREEN

```bash
.venv/bin/python -m unittest test_industry_archives.ArchiveCatalogTests -v
```

### Step 5：提交

```bash
git add industry_archives.py test_industry_archives.py requirements.txt
git commit -m "feat: load repository industry archives"
```

## Task 3：增加只读 API

**文件：**

- 修改：`service.py`
- 修改：`test_industry_archives.py`
- 修改：`test_web.py`

### Step 1：写 API 失败测试

使用临时 T1 库和档案目录 mock 新模块路径，断言：

- `/api/industry-thermometer` 返回日期、口径版本、覆盖率与行业列表；
- `/api/industry-archives` 返回目录，传 `code` 时返回匹配 ID；
- `/api/industry-archives/{id}` 返回渲染 HTML，未知 ID 返回 404；
- 新路由只有 GET，并在生产等效模式下可读。

### Step 2：确认 RED

```bash
.venv/bin/python -m unittest test_industry_archives.IndustryArchiveApiTests -v
```

### Step 3：在 `service.py` 增加薄路由

路由只转调 `industry_archives.py`。对可预期缺数返回诚实空态；对未知档案 ID 返回 404。不改 lifespan、scheduler、EOD 或写中间件。

### Step 4：确认 GREEN 并运行相关回归

```bash
.venv/bin/python -m unittest test_industry_archives.IndustryArchiveApiTests -v
.venv/bin/python test_s3_ui.py
```

### Step 5：提交

```bash
git add service.py test_industry_archives.py test_web.py
git commit -m "feat: expose read-only industry archive APIs"
```

## Task 4：加入两篇初始档案

**文件：**

- 新建：`docs/产业档案/产业档案-存储与算力硬件链.md`
- 新建：`docs/产业档案/研究-五年涨幅王画像与系统对照-20260718.md`
- 修改：`test_industry_archives.py`

### Step 1：先写仓库实例校验

断言 `docs/产业档案` 中两篇文档都能被目录加载，元数据齐全，复制的五年涨幅王正文 SHA-256 在去除 YAML 头后与原文一致。

### Step 2：放入档案

存储与算力硬件链档案保留用户附件原文；如附件仍未落到可读文件系统，不伪造“附件原文”，先放入明确标注的待替换空态文档，并在最终报告单列为未满足项。

五年涨幅王文档只在头部添加：

```yaml
---
industry: 全市场
updated: 2026-07-18
status: 研究归档
---
```

### Step 3：运行目录测试并提交

```bash
.venv/bin/python -m unittest test_industry_archives.RepositoryArchiveTests -v
git add docs/产业档案 test_industry_archives.py
git commit -m "docs: seed industry archive library"
```

## Task 5：按新前端令牌实现第三子页签

**文件：**

- 修改：`static/home.html`
- 修改：`test_guanlan_frontend_contract.py`
- 修改：`test_industry_archives.py`

### Step 1：写前端合约失败测试

锁定：

- 股票池有“名单 / 选股逻辑 / 产业档案”三子页签；
- 启动读取温度计与档案目录 API；
- 药丸点击只改变档案过滤，不调用任何 POST API；
- `#pool/archive/<id>` 可深链到阅读区；
- 个股详情只在 `matched_archive_ids` 非空时显示链接；
- 手机媒体查询下档案布局为单列，药丸和列表横向滚动。

### Step 2：确认 RED

```bash
.venv/bin/python -m unittest test_guanlan_frontend_contract.py -v
.venv/bin/python -m unittest test_industry_archives.FrontendArchiveContractTests -v
```

### Step 3：实现 HTML/CSS/JavaScript

复用现有 `--bg`、`--card`、`--line`、`--tc`、`--sage`、`--amber`、`--serif` 和 `--sans`。添加：

- 温度计标题、截止日和药丸带；
- 桌面双列档案区；
- 手机紧凑卡片带与阅读区；
- 安全插入后端已渲染 HTML 的阅读区；
- 个股详情匹配链接和深链路由。

### Step 4：确认 GREEN

```bash
.venv/bin/python -m unittest test_guanlan_frontend_contract.py -v
.venv/bin/python -m unittest test_industry_archives.FrontendArchiveContractTests -v
```

### Step 5：提交

```bash
git add static/home.html test_guanlan_frontend_contract.py test_industry_archives.py
git commit -m "feat: add industry archive experience to new frontend"
```

## Task 6：回归、真库对账与性能验证

**文件：**

- 修改：`test_industry_archives.py`
- 新建：`reports/industry-thermometer-realdata-20260802.json`

### Step 1：运行全部新测试

```bash
.venv/bin/python -m unittest test_industry_archives.py -v
```

### Step 2：运行项目强制四测

```bash
.venv/bin/python test_rules.py
.venv/bin/python test_rules_mid.py
.venv/bin/python test_web.py
.venv/bin/python test_regime.py
```

### Step 3：对完整 T1 库运行只读计算

记录：

- T1 库与 `market.db` 计算前后 SHA-256；
- `data_date`、行业总数、四档分布和缺数数；
- 首次计算与缓存命中耗时；
- 量比前 5 是否全部为烫；
- 四灯与等级合成不变式。

将可公开的聚合结果写入 JSON，不写原始行情、交易或账户数据。

### Step 4：提交对账证据

```bash
git add reports/industry-thermometer-realdata-20260802.json
git commit -m "test: verify industry thermometer on frozen T1 data"
```

## Task 7：生产等效环境视觉验收

**文件：**

- 新建：`docs/screenshots/industry-archives-equivalent/20260802/*.png`
- 新建：`reports/industry-archives-equivalent-20260802.json`

### Step 1：准备只读副本与 LaunchAgent 克隆副本

使用现有生产等效工具生成独立运行目录和数据库副本。克隆 plist 只改 label、端口、工作目录、日志路径与只读数据库环境变量；生产 plist 字节不改。

启动前记录：

- 生产 plist SHA-256；
- 生产 `screener.db`、`market.db`、`S3_research.db` SHA-256；
- 三个副本 SHA-256；
- 完整 T1 库 SHA-256。

### Step 2：通过克隆 LaunchAgent 与 `start.sh` 启动独立端口

验证 `/api/health` 与三个新 API，确认 POST 仍被等效模式 403 拦截。

### Step 3：浏览器验收与截图

桌面 1440×900 和手机 390×844 至少验收：

- 产业档案总览；
- 点行业药丸后的过滤态；
- 打开一篇带表格的档案；
- 个股详情“产业档案”链接；
- 无匹配个股隐藏链接。

检查 0 个控制台错误、0 个失败网络请求和手机无水平挤压。

### Step 4：停止等效实例并复核哈希

再次计算上述 plist、生产库、副本和 T1 库哈希。任一变化即验收失败。

## Task 8：最终报告、分支提交与镜像闭环

**文件：**

- 新建：`docs/验收-产业温度计与产业档案页-20260802.md`

### Step 1：写最终报告

报告包含：分支与提交、口径、数据截止日、只读证据、测试结果、等效环境哈希、桌面/手机截图清单、边界和未满足项。

### Step 2：运行完成前验证

```bash
.venv/bin/python -m unittest test_industry_archives.py -v
.venv/bin/python test_rules.py
.venv/bin/python test_rules_mid.py
.venv/bin/python test_web.py
.venv/bin/python test_regime.py
git diff --check
git status --short
```

### Step 3：提交最终报告

```bash
git add docs reports
git commit -m "docs: verify industry thermometer and archive page"
```

### Step 4：同步公开报告镜像

运行仓库镜像同步脚本，明确同步本分支的 `docs/` 与 `reports/`。通过 HTTPS 推送后，在全新临时目录克隆 `https://github.com/harryd317/guanlan-reports.git`，核对最终报告、设计、计划、初始档案、截图和对账 JSON。

记录并回报镜像 `HEAD` 完整提交哈希。没有该哈希不得宣布任务完成。
