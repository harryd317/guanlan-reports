# T7 生产等效环境修复实施计划

> 状态：设计已批准；本计划只执行复现、修复与等效验收，不部署生产。

**目标：** 用生产 LaunchAgent 的克隆副本在 `127.0.0.1:18787` 稳定复现 T7 今天页空白与手机端挤压，先红后绿完成最小修复，并取得五页桌面/手机验收证据。

**架构：** `start.sh` 保持生产入口，只把端口抽成默认值为 `8787` 的 `SCREENER_PORT`。新增一个小型等效模式边界模块，集中读取三个数据库副本路径、用 SQLite `mode=ro` 打开并禁止后台调度和 HTTP 写操作。等效运行脚本只复制生产 plist 到独立证据目录后修改副本；生产 plist、生产数据库、生产进程始终只读且不停止。

**技术栈：** Bash、launchd/LaunchAgent、FastAPI/Uvicorn、SQLite URI、Python `unittest`、浏览器自动化。

---

## Task 1：冻结运行基线和证据契约

**文件：**

- Create: `scripts/t7_equivalent_env.py`
- Create: `test_production_equivalent.py`
- Modify: `.gitignore`

**步骤：**

1. 在 `test_production_equivalent.py` 先写证据目录契约测试：
   - 运行目录必须位于工作树内的 `.runtime/t7-equivalent-20260731/`；
   - 必须记录生产 plist、三个生产数据库和三个副本的 SHA-256；
   - 必须拒绝生产 plist 路径作为输出目标；
   - 必须拒绝任一副本路径与生产数据库路径相同。
2. 运行：

   ```bash
   .venv/bin/python -m unittest -v test_production_equivalent.py
   ```

   保存 RED 输出，失败原因应为证据工具尚不存在。
3. 最小实现 `scripts/t7_equivalent_env.py`：
   - `hash_file(path)` 流式计算 SHA-256；
   - `snapshot_sqlite(source, destination)` 使用 SQLite backup API 生成一致性副本；
   - `record_hashes(...)` 输出 UTF-8 JSON；
   - `assert_isolated_paths(...)` 对真实路径做相等检查；
   - 副本完成后移除写权限。
4. 在 `.gitignore` 增加 `.runtime/`，避免数据库副本和运行日志进入 Git。
5. 重跑测试至 GREEN，并提交：

   ```bash
   git add .gitignore scripts/t7_equivalent_env.py test_production_equivalent.py
   git commit -m "test(t7): establish production-equivalent evidence boundary"
   ```

## Task 2：先红后绿实现 `start.sh` 端口兼容

**文件：**

- Modify: `test_production_equivalent.py`
- Modify: `start.sh`

**步骤：**

1. 增加启动脚本行为测试，用临时 `PATH` 注入假的 `lsof`、`pip`、`python` 和 `open`：
   - 不传 `SCREENER_PORT` 时，探测、提示、浏览器 URL 和 Uvicorn 参数均为 `8787`；
   - 传 `SCREENER_PORT=18787` 时，上述四处均为 `18787`；
   - `SCREENER_NO_BROWSER=1` 时不调用 `open`。
2. 运行单测并保存 RED 输出，确认失败来自脚本硬编码 `8787`。
3. 在 `start.sh` 进入端口探测前加入：

   ```bash
   SCREENER_PORT="${SCREENER_PORT:-8787}"
   ```

   将所有四处 `8787` 替换为同一变量，不改变其他启动行为。
4. 重跑单测至 GREEN，并提交：

   ```bash
   git add start.sh test_production_equivalent.py
   git commit -m "fix(t7): keep production port while allowing equivalent launch"
   ```

## Task 3：先红后绿建立只读数据库边界

**文件：**

- Create: `equivalent_mode.py`
- Modify: `storage.py`
- Modify: `market_data.py`
- Modify: `s3_research/read_model.py`
- Modify: `service.py`
- Modify: `test_production_equivalent.py`

**步骤：**

1. 增加测试并保存 RED 输出：
   - `SCREENER_EQUIVALENT_MODE=1` 时三个路径必须来自各自环境变量；
   - `storage._conn()`、`market_data._conn()` 和 S3 连接均使用 `file:<绝对路径>?mode=ro` 与 `uri=True`；
   - 连接可读，`INSERT`、建表和 `PRAGMA journal_mode=WAL` 均失败；
   - 任一路径缺失、不可读或指向生产默认路径时，应用导入/启动失败，不回退；
   - 普通模式仍使用原路径和原连接语义。
2. 最小实现 `equivalent_mode.py`：
   - 只解析 `SCREENER_EQUIVALENT_MODE`；
   - 校验并返回 `SCREENER_DB_PATH`、`SCREENER_MARKET_DB_PATH`、`SCREENER_S3_DB_PATH`；
   - 生成只读 SQLite URI；
   - 普通模式不改变现有路径。
3. 在三个数据库模块仅改路径选择与连接工厂：
   - 普通模式保留原 `sqlite3.connect(path)` 和 WAL；
   - 等效模式用 `mode=ro`、`uri=True`、`PRAGMA query_only=ON`，不执行 WAL。
4. 在 `service.py` 启动阶段验证三个只读连接均可打开；任何失败直接中止启动。
5. 重跑测试至 GREEN，并提交：

   ```bash
   git add equivalent_mode.py storage.py market_data.py s3_research/read_model.py service.py test_production_equivalent.py
   git commit -m "fix(t7): isolate equivalent mode with read-only databases"
   ```

## Task 4：先红后绿禁用后台任务与写请求

**文件：**

- Modify: `service.py`
- Modify: `test_production_equivalent.py`

**步骤：**

1. 增加测试并保存 RED 输出：
   - 等效模式 lifespan 不调用 `storage.init_db()`、`portfolio_books.init_db()`、`params.apply()` 或 `scheduler.start()`；
   - 普通模式仍按原顺序执行初始化和调度；
   - 等效模式的 `POST`、`PUT`、`PATCH`、`DELETE` 返回 `403` 和固定错误码；
   - `GET /`、`GET /nav.js`、`GET /brand.js` 与五页只读 API 保持可用。
2. 在 `service.py` 只增加等效模式条件：
   - 等效模式只验证数据库，不初始化、迁移或启动 APScheduler；
   - 添加最窄的 HTTP 方法守卫；
   - 普通模式路径逐字保持原行为。
3. 重跑测试至 GREEN，并提交：

   ```bash
   git add service.py test_production_equivalent.py
   git commit -m "fix(t7): suppress writes in production-equivalent mode"
   ```

## Task 5：克隆生产 LaunchAgent 并记录启停前证据

**文件：**

- Modify: `scripts/t7_equivalent_env.py`
- Modify: `test_production_equivalent.py`
- Create: `.runtime/t7-equivalent-20260731/`（忽略，不提交）
- Create: `reports/t7-production-equivalent-20260731/before-hashes.json`

**步骤：**

1. 为 plist 克隆器写测试并保存 RED 输出：
   - 输入文件只读打开；
   - 副本仅改变 `Label`、`WorkingDirectory`、`ProgramArguments`、环境变量和日志路径；
   - 原 plist 字节哈希不变；
   - 生成副本通过 `plutil -lint`。
2. 实现最小 plist 克隆器，输出到 `.runtime/t7-equivalent-20260731/`，绝不覆盖 `~/Library/LaunchAgents/com.rui.stock-screener.web.plist`。
3. 只读采集生产状态：

   ```bash
   shasum -a 256 ~/Library/LaunchAgents/com.rui.stock-screener.web.plist
   launchctl print gui/$(id -u)/com.rui.stock-screener.web
   lsof -nP -iTCP:8787 -sTCP:LISTEN
   curl -fsS http://127.0.0.1:8787/api/health
   ```

4. 用 SQLite backup API 生成三个一致性副本，移除写权限，记录生产库与副本启前哈希。
5. 生成并 lint 等效 plist；人工核对生产 plist 哈希与步骤 3 一致。
6. 提交只含哈希和拓扑元数据的 `before-hashes.json`，不得提交数据库、令牌、绝对私有路径或日志。

## Task 6：启动等效实例并稳定复现

**文件：**

- Create: `reports/t7-production-equivalent-20260731/reproduction/`（截图和脱敏探针）
- Modify: `docs/事件记录-T7部署回滚-20260731.md`

**步骤：**

1. 用克隆 plist 启动：

   ```bash
   launchctl bootstrap gui/$(id -u) .runtime/t7-equivalent-20260731/com.rui.stock-screener.web.t7-equivalent.plist
   launchctl print gui/$(id -u)/com.rui.stock-screener.web.t7-equivalent
   lsof -nP -iTCP:18787 -sTCP:LISTEN
   ```

2. 记录等效 PID、cwd、完整参数；确认生产 8787 PID 和健康状态未变。
3. 从 `http://127.0.0.1:18787` 采集首页、`nav.js`、`brand.js` 的状态码、MIME、ETag、Last-Modified、Cache-Control。
4. 用桌面 `1440×900`、手机 `390×844` 对今天、股票池、我的、工程模式、设置逐页探针，至少记录：
   - page error、console error、失败请求；
   - 根容器和关键卡片 bounding box；
   - `scrollWidth/clientWidth`；
   - 活动路由、匹配 media query；
   - 正常缓存、禁用缓存、硬刷新结果。
5. 若今天页空白与手机挤压未稳定复现，停止代码修改，只核对 LaunchAgent、cwd、资源响应、缓存和数据库路径，直至能用同一操作连续复现三次。
6. 把失败截图、探针 JSON 和排除实验写入事件记录；此阶段截图明确标为“修复前失败证据”。

## Task 7：把真实根因固化为失败测试并做最小修复

**文件：**

- Modify: `test_production_equivalent.py`
- Modify: 根因对应的单个或最小文件集合（只有 Task 6 证据确定后才填写）
- Modify: `docs/事件记录-T7部署回滚-20260731.md`

**步骤：**

1. 根据 Task 6 的证据写用户可见行为测试，至少断言：
   - 今天页关键内容在等效启动条件下可见宽高大于零；
   - 手机主容器有效宽度不退化为单字列；
   - 页面无非设计横向溢出；
   - 根因涉及的实际资源/路由从 T7 工作目录加载。
2. 连续运行三次，均应 RED，保存完整输出和失败测量值。
3. 只修改证据指向的路径、模板、脚本或 CSS；每次实验只改变一个条件。
4. 连续运行三次新增测试，均应 GREEN。
5. 运行项目全量回归：

   ```bash
   .venv/bin/python test_rules.py
   .venv/bin/python test_rules_mid.py
   .venv/bin/python test_web.py
   .venv/bin/python test_regime.py
   ```

6. 把 RED/GREEN 输出、根因机制、为何隔离预览未暴露问题写入事件记录，并提交最小修复。

## Task 8：五页十图生产等效验收

**文件：**

- Create: `reports/t7-production-equivalent-20260731/acceptance/`
- Create: `reports/t7-production-equivalent-20260731/acceptance-probes.json`
- Modify: `docs/事件记录-T7部署回滚-20260731.md`
- Create: `docs/验收-T7-生产等效环境修复-20260731.md`

**步骤：**

1. 在同一等效 LaunchAgent 实例上逐页验收：
   - 今天 `/#today`
   - 股票池 `/#pool`
   - 我的 `/#me`
   - 工程模式 `/#me` 并展开
   - 设置 `/settings`
2. 每页保存桌面 `1440×900` 和手机 `390×844` 截图，共十张；文件名包含页面、视口、T7 源提交和日期。
3. 每页断言内容可见、无单字竖排、无非设计横向滚动、导航无遮挡、无控制台/page error、无失败资源/API。
4. 任一断言失败即停止验收，保留失败证据，返回 Task 6 或 Task 7；不得选择性提交通过截图。
5. 五页全过后记录生产 plist、生产数据库和数据库副本的启后哈希，并逐项与启前哈希比较。
6. 卸载等效 LaunchAgent：

   ```bash
   launchctl bootout gui/$(id -u)/com.rui.stock-screener.web.t7-equivalent
   ```

   确认 18787 不再监听，生产标签、PID、8787 健康状态仍正常。
7. 完成事件记录与验收报告，明确状态为“修复已验收，等待镜像复核和重新部署批准”。

## Task 9：提交、镜像同步与用户复核门禁

**文件：**

- Modify: `docs/事件记录-T7部署回滚-20260731.md`
- Modify: `docs/验收-T7-生产等效环境修复-20260731.md`
- Add: `reports/t7-production-equivalent-20260731/**`

**步骤：**

1. 核对提交范围不含生产配置、数据库、令牌和未脱敏绝对路径。
2. 提交最终报告和十张截图。
3. 通过 HTTPS 把 T7 分支 `docs/`、`reports/` 和 `AGENTS.md` 同步到公开镜像。
4. 从镜像远端新克隆，核对：
   - 事件记录；
   - 最终验收报告；
   - 十张截图文件数、尺寸和 SHA-256；
   - T7 分支 `AGENTS.md`；
   - 镜像 HEAD 提交哈希。
5. 只有远端验证通过并回报镜像哈希后，才可向用户申请重新部署；本计划不执行生产合并、重启或部署。
