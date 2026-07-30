# T3 观澜品牌更名设计

## 1. 目标与边界

T3 只改变用户能够看到的品牌展示，把现有应用统一命名为：

- 主文案：观澜
- 副文案：A股纪律投研台
- 品牌句：观水有术，必观其澜

本任务不做全局重命名。代码标识符、文件名、数据库名、API 路径、环境变量、仓库名，以及内部 `screener` 术语保持不变。T3 不改策略、风控、回测、行情与研究数据管道、数据库结构，也不新增依赖。

## 2. 单一品牌源

新增根目录 `branding.py`，只在其中保存一份品牌配置：

```python
BRAND = {
    "name": "观澜",
    "subtitle": "A股纪律投研台",
    "tagline": "观水有术，必观其澜",
}
```

同一模块提供两个只读派生函数：

- `page_title(section: str = "") -> str`：返回 `观澜 · <页面>` 或 `观澜`。
- `notification_title(title: str) -> str`：返回 `[观澜] <标题>`；已带签名时保持不变，防止重复签名。

Python 页面服务、Server酱/企业微信推送和 macOS 通知都导入这一个配置源。

## 3. 前端注入

`service.py` 新增只读 `GET /brand.js`。响应内容由 `branding.py` 的 `BRAND` 序列化生成，向浏览器设置 `window.GUANLAN_BRAND`，并完成两项展示行为：

1. 根据页面自身的 section 标题把 `document.title` 统一生成为 `观澜 · <页面>`；
2. 把带 `data-brand-name`、`data-brand-subtitle`、`data-brand-tagline` 的节点填入对应文案。

九个既有 HTML 页面只增加 `/brand.js` 引用和语义化占位，不复制三条品牌文字。`static/nav.js` 从 `window.GUANLAN_BRAND` 读取品牌名。这样页面、导航、关于页没有第二份可漂移的品牌常量。

## 4. 关于页

新增 `static/about.html` 和只读 `GET /about`，展示主文案、副文案和品牌句，文案全部由 `/brand.js` 注入。`static/settings.html` 的“关于”入口指向 `/about`。

关于页沿用现有页面的轻量 CSS 与主导航，不增加第三方资产、依赖或数据接口。

## 5. 通知签名

所有 Server酱和企业微信消息已经汇聚到 `push.send_push(title, body)`，因此只在该边界对标题调用 `notification_title`，不修改各业务调用点。

所有本机通知已经汇聚到 `eod._macos_notify(title, body)`，因此只在该边界签名。`run_eod_scan`、日报生成和调度顺序保持原样。

验收测试用假的 HTTP 端点和假的 `osascript` 进程验证最终出站标题，不向真实通知渠道发送消息。

## 6. 测试与验收

新增 `test_branding.py`，覆盖：

- 页面标题和通知签名派生，含防重复；
- `/brand.js` 对真实 DOM 的品牌注入契约；
- 九个既有页面和关于页均加载品牌脚本；
- 导航、首页和关于页没有用户可见的旧品牌硬编码；
- Server酱、企业微信和 macOS 出站标题都带统一签名；
- 关于页路由与设置入口可达。

完成后运行 T3 新测试、既有规则/市况/T1/S3/全站回归、Python 编译、`git diff --check`、依赖与禁改文件审计。最后用隔离端口启动本地服务，分别截取桌面首页和手机关于页作为验收证据。

## 7. 交付纪律

所有修改只提交到 `codex/t3-guanlan-branding`。验收材料完成后停在该分支，不合并、不部署、不重启或切换生产、不开始 T4。
