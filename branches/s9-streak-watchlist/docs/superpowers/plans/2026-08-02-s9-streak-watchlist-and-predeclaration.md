# S9 连阳名单与测量预声明实施计划

> **For Codex:** 使用 `superpowers:executing-plans` 逐项执行；功能修改必须遵循 `superpowers:test-driven-development`。

**目标：** 上线只读 S8 池连阳名单与选股逻辑三卡，并在独立第二阶段只交付 S9 测量预声明。

**架构：** 服务端读模型复用冻结的 S8 猎场与现有只读数据源，GET 接口返回按数据日缓存的名单；单页前端只负责展示。正式测量在本任务中保持禁用。

**技术栈：** Python 3.11、FastAPI、SQLite `mode=ro`、原生 HTML/CSS/JavaScript、`unittest`

---

## Task 1：锁定阶段一读模型契约

**文件：**

- 新建：`s8_research/streak_watchlist.py`
- 修改：`test_s8_research.py`

1. 写失败测试，以完整 `DailyInputs` fixture 验证：只纳入 S8 猎场与温/热行业交集；排除烫档；阳线按 `close > open`；2 连和 3 连返回真实根数；末根涨幅与 20 日量比手算一致；排序稳定。
2. 运行聚焦测试并确认失败原因是读模型缺失。
3. 实现最小纯函数 `build_streak_watchlist(inputs)`。
4. 增加数据加载包装，读取最新完整快照日，通过现有获批源生成 `DailyInputs`，并按目标日做进程内缓存。
5. 写失败路径测试，验证陈旧快照或数据源异常返回错误，不写数据库。
6. 运行聚焦测试至通过。

## Task 2：增加只读 API

**文件：**

- 修改：`service.py`
- 修改：`test_s8_research.py`
- 修改：`test_guanlan_frontend_contract.py`
- 修改：`reports/guanlan-frontend-20260801/baseline/api-source-manifest.json`

1. 写失败测试，要求注册 `GET /api/s8/streak-watchlist`，并验证路由无 POST/写操作。
2. 在服务中调用读模型；异常经现有 `_fail` 返回真实空态。
3. 更新获批展示 API 基线清单，不改既有 API 源哈希。
4. 运行 S8 与前端 API 合同测试至通过。

## Task 3：实现三卡与今天页名单

**文件：**

- 修改：`static/home.html`
- 修改：`test_guanlan_frontend.py`
- 修改：`test_s8_research.py`

1. 写失败测试，验证三卡顺序、徽标、冻结文案、红利低波原文和新启动 GET 接口。
2. 写 JavaScript 模型失败测试，验证有数据、真实空态、错误态和温度中文标签。
3. 将逻辑页改成三卡；桌面三列，手机单列。
4. 在 S8 关注下增加只读名单区块与表格，空态显示“今日无≥2连阳”。
5. 将接口加入启动加载和 `pageState`，实现转义、安全格式化和渲染。
6. 确认页面 POST 请求数量与既有允许项不变，名单区没有交易入口。
7. 运行前端、S8、Web 聚焦回归至通过。

## Task 4：阶段一完整验证与生产等效验收

**文件：**

- 新建：`reports/s9-streak-watchlist-equivalent-20260802.json`
- 新建：`docs/screenshots/s9-streak-watchlist/20260802/阶段一-等效-桌面.jpg`
- 新建：`docs/screenshots/s9-streak-watchlist/20260802/阶段一-等效-手机.jpg`

1. 运行完整测试套件、`git diff --check` 和只读边界检查。
2. 记录生产 plist、`screener.db`、`market.db`、温度计 sidecar、`S8_research.db` 的启动前 SHA-256。
3. 克隆生产 plist，仅替换 Label、独立端口和工作目录；生产 plist 原件一字不改。
4. 将生产数据库复制到临时验收目录并只读打开。记录生产库和副本启动前哈希。
5. 通过同一 `start.sh` 启动等效实例，确认健康检查与连阳 GET 接口。
6. 用桌面和手机视口验收今天页、股票池选股逻辑页，并巡检全站主要页面。
7. 保存截图和 JSON 证据；停止等效实例。
8. 核对生产 plist、生产库和副本启动后哈希不变。

## Task 5：合并、部署与生产验收

**文件：**

- 新建：`reports/s9-streak-watchlist-production-20260802.json`
- 新建：`docs/screenshots/s9-streak-watchlist/20260802/阶段一-生产-桌面.jpg`
- 新建：`docs/screenshots/s9-streak-watchlist/20260802/阶段一-生产-手机.jpg`
- 新建：`docs/验收-S9阶段一-S8池连阳名单-20260802.md`

1. 提交阶段一代码与等效证据，确认分支测试全绿。
2. 合并主线前再次记录生产代码提交、plist 和四份数据库哈希。
3. 合并到 `main`，重启生产 LaunchAgent。
4. 验证 `/api/health` 的 `ok=true`、新 GET 接口、三卡和名单。
5. 用生产桌面、手机视口截图今天页与股票池逻辑页，并巡检全站。
6. 核对数据库和 plist 哈希不变。若任一检查失败，恢复上一代码提交和重启服务，并记录回滚。
7. 提交生产验收报告和证据。

## Task 6：阶段一镜像同步与验真

1. 从 `main` 运行镜像同步脚本并推送。
2. 全新克隆镜像仓库，验证阶段一报告、等效/生产证据和截图存在。
3. 记录镜像 HEAD；只有该哈希可被回报为阶段一完成凭据。

## Task 7：阶段二仅提交预声明

**文件：**

- 新建：`docs/S9-S8池连阳尾盘信号测量-预声明-v1.0.md`
- 新建：`docs/交付-S9测量预声明-20260802.md`

1. 从已同步的阶段一 `main` 建立独立分支。
2. 按 S7 原始预声明与正式报告冻结历史区间、费用、收益/回撤分布、切片、基准和三门标准。
3. 把样本空间改为历史时点 S8 猎场与温/热行业交集。
4. 明列连阳数、末根涨幅上限和持有期限的固定笛卡尔单元；禁止扫描、后移和优化重试。
5. 在文首和交付报告中写明“未获批准，不得正式测量”。
6. 只运行文档与仓库边界检查；确认没有新增测量脚本、数据库或结果文件。
7. 合并 `main` 并提交交付报告。

## Task 8：阶段二镜像同步与验真

1. 再次运行镜像同步脚本并推送。
2. 全新克隆镜像仓库，验证预声明和交付报告内容、阶段一产物仍在。
3. 记录第二个镜像 HEAD。
4. 最终汇报两个阶段的源仓提交、镜像提交、生产哈希、截图路径和“正式测量未启动”。
