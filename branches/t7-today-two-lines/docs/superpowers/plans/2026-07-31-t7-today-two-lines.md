# T7 今天页双线实施计划

> 已获 2026-07-31 动工授权。仅实施今天页，截图验收前不改其他页面。

**目标：** 把今天页改成投机线、投资线双栏驾驶舱，并把冻结的五批建仓计划接到真实投资仓账本。

**结构：** 新增纯函数模块 `investment_plan.py` 负责固定日程、沪深300当周加倍与账本完成度。`portfolio_books.py` 增加只读买入汇总。`/api/trade/overview` 只追加投资线字段。`static/home.html` 用既有 M3 令牌重排今天页。

**依赖：** Python 标准库、现有 FastAPI、原生 HTML/CSS/JavaScript。零新增依赖。

---

### 任务 1：冻结建仓计划与自动推进

**文件：**

- 新增：`investment_plan.py`
- 新增：`test_investment_plan.py`
- 修改：`portfolio_books.py`

1. 先写失败测试：五个周三、基础 1.4 万、当周跌超 2% 加倍、真实买入达标推进、迟到不跳批、五批完成。
2. 运行 `python -m unittest -q test_investment_plan`，确认红灯。
3. 实现最小纯函数与只读账本查询。
4. 重跑测试，确认绿灯。

### 任务 2：把投资线数据追加到概览接口

**文件：**

- 修改：`service.py`
- 修改：`test_web.py`

1. 先写失败测试：`/api/trade/overview` 保留旧键，并追加 `investment_plan`、`investment_dividend_yield`。
2. 覆盖计划计算异常的失败隔离测试。
3. 实现接口追加字段；股息率只读设置值。
4. 跑相关测试与完整 `test_web.py`。

### 任务 3：重排今天页

**文件：**

- 修改：`static/home.html`
- 修改：`test_m3_home.py`

1. 先写失败契约：双线标题、计划字段、账本字段、广度单行、无旧长文区块。
2. 用现有 M3 卡片、状态容器和药丸按钮实现新布局。
3. 桌面两列，手机单列；压缩间距保证一屏。
4. 检查所有新增可见句子不超过 15 个汉字。
5. 运行 `test_m3_home.py` 与 `test_s3_ui.py`。

### 任务 4：回归与截图

**文件：**

- 不再改业务文件，除非测试证明有缺陷。

1. 运行 `test_investment_plan.py`、`test_m3_home.py`、`test_s3_ui.py`、`test_web.py`。
2. 运行 `git diff --check`，确认零新增依赖与禁改范围零差异。
3. 使用隔离数据库启动本地服务。
4. 截取桌面与手机视口。
5. 检查无滚动、无溢出、真实数据绑定、文案长度和语义色。
6. 提交截图与验收结果，停止等待用户确认。
