# 计数日历

一个本地运行的网页日历，每天按格子记录半天/整天/空。多项目、多视图、可导出可同步。

> **License**: [MIT](./LICENSE) © 2026 EC11112
> **版本**: v0.3.16（单文件，零依赖，双击 `index.html` 就能用）

## 怎么用

**最快上手**：双击 `index.html` 用浏览器打开，无需安装、无需部署。

> 用 **Chrome / Edge 86+** 效果最好（File System Access API 需要）。Firefox 也能用但本地同步功能不可用。
> 数据存浏览器 `localStorage`，**每个浏览器独立**——换浏览器/清数据前请先导出 JSON（设置 → ⚙ → 导出数据）。

## 功能

### 基础

- **点击计数**：日期格子点击循环 `0 → 半天 → 整天 → 空`
- **两种项目分类**：
  - **工时类**（学习/工作）：半天 = 0.5 天，整天 = 1 天；同一天可记录多个项目
  - **非工时类**（健身/阅读）：0 → √ → 0
- **多项目**：左侧可新建/编辑/删除项目，8 主题色 + 自定义取色
- **同日多项目**：用色条区分同一天多个项目
- **统计**：右侧实时显示本月 / 本季 / 本年 / 总计 / 折合小时（按 1 天 = 8h）

### 视图

- **月视图**（默认）：单月，prev/next 步进 ±1 月
- **季视图**：3 个月连续流，**列数随视口自适应**（手机 7 列，平板 14 列，1080p 21 列，4K 35 列）。格子强制正方形，字体自动缩
- **年视图**：12 个月连续流，同样自适应列数
- **resize 监听**：窗口缩放时 debounce 重算布局，所有视图自动重排

### 编辑与协作

- **默认只读**：防止误触，点顶部 ✎ 进入编辑模式（**刷新自动关闭**，避免忘了关还继续记录）
- **项目编辑模式**：点击 sidebar 底部 ✎ 编辑项目，可改名/改色/改分类/删除
- **项目显隐**：sidebar 点 👁 切换显隐（统计/日历自动只算可见项目）
- **非编辑态双击日期块**：弹蓝色"温馨提示"toast 提示如何进入编辑

### 数据保护

- **导入/导出 JSON**（⚙ 设置菜单）：跨浏览器迁移、备份
  - 导出文件含 schemaVersion + appVersion，**导入会校验版本兼容性**
  - 字段含完整 state（项目/日志/视图设置）
- **清除数据**（sidebar 底部 🗑）：完全恢复到首次启动状态
  - 清空所有项目 + 日志 + 欢迎页"不再提醒"
  - **保留视图设置**（全屏/格子大小/隐藏下载按钮/隐藏同步按钮）
  - 同步配置会清掉（高级功能，每次清除需重新授权）
- **欢迎页**：首次进入自动弹 7 区块功能介绍 + 数据本地保存提醒
  - 勾"不再提醒"后下次不弹；点顶部 ? 可重新打开

### 部署与协作

- **下载/部署按钮**（顶部 ↓）：浏览器支持 PWA install 时弹安装提示，否则下载当前 HTML
  - 配合网盘（OneDrive/iCloud/Dropbox）实现多端同步
  - ⚠ 单文件 HTML 无 manifest + service worker，大部分浏览器走下载分支
- **本地同步**（顶部 ⇄，**高级功能**）：
  - 选本地 JSON 文件，启动时自动读入对比，不一致弹 confirm 让你选择是否导入
  - 开启自动导出后每 N 秒对比一次，数据变化就写回文件
  - **需 Chrome/Edge 86+ + HTTPS**，Firefox/Safari/file:// 不支持
  - 同步配置（syncEnabled/syncAutoExport/syncInterval）**不进 export/import JSON**
- **隐藏按钮**（⚙ 设置菜单）：不需要某个顶部按钮可以隐藏
  - 隐藏下载按钮 / 隐藏同步按钮（不影响设置项本身）
  - 勾选/取消都不关闭菜单（跟全屏 checkbox 一致）

## 界面与可访问性

- 顶部 5 个图标按钮：`✎` 编辑 / `?` 帮助 / `↓` 下载 / `⇄` 同步 / `⚙` 设置
- icon 风格：界面按钮统一线性黑白，欢迎页/介绍文案保留彩色 emoji
- 字体公式：所有视图字号一致 `clamp(cellSize × 0.5, 10, 14)`
- 周末（`日` / `六`）红色显示
- 响应式：< 900px 单列，< 700px 季/年 2 列，< 480px 1 列

## 快捷键

- 日期块 hover → tooltip 显示日期 + 项目记录
- 项目名输入框 → 回车提交 / Esc 取消
- 非编辑态双击日期块 → 温馨提示 toast（提示如何进入编辑）

## 数据说明

- **存哪**：浏览器 `localStorage`，key = `count_calendar_v2`
- **结构**：`{ projects, currentProjectId, logs, viewYear, viewMonth, viewMode, maxStripCols, maxCellSize, fullscreen, calPanelWidth, dismissedIntro, hideDownloadButton, hideSyncButton, syncEnabled, syncAutoExport, syncInterval, schemaVersion, appVersion }`
- **导出**（⚙ → 导出数据）：下载 JSON 备份
- **导入**（⚙ → 导入数据）：选 JSON 文件，**覆盖**当前所有数据（弹 confirm 显示 N 项目 / M 日志）
- **本地同步**（⇄ 高级功能）：File System Access API 选本地 JSON 文件做自动备份，详见顶部 ⇄ 弹窗
- **想清空**：用 sidebar 🗑 清除数据按钮（项目编辑模式下显示）

## 文件结构

```
count_calendar/
├── index.html      # 全部代码（HTML + CSS + JS），单文件 ~2000 行
├── README.md       # 本文件
├── CHANGELOG.md    # 版本历史
├── LICENSE         # MIT License
├── want.md         # 原始需求
└── .gitignore
```

## Built with

- 设计 + 实现：[EC11112](https://github.com/EC11112) 主理
- AI 协作：[MiniMax-M3 (Mavis)](https://github.com/minimax) 参与设计 / 编码 / 调试
- 部分 commit 由 AI 生成后人工审核

## 想改？

直接编辑 `index.html`（单文件，搜关键字）：

| 想改什么 | 在哪改 |
|---------|--------|
| 主题色 | CSS 顶部的 `--bg` `--accent` 等 CSS 变量 |
| 项目色板 | JS 的 `COLORS` 数组（16 预设 + 自定义取色） |
| 1 天 = 多少小时 | JS 的 `HOURS_PER_DAY` 常量（默认 8） |
| 默认项目 | JS 的 `DEFAULT_PROJECTS` 数组 |
| 字号公式 | `calcLabelFontSize` / `calcBadgeFontSize` 函数 |
| 月份名称 | i18n 在 `MONTH_NAMES` / `WEEKDAY_NAMES`（如果有） |

## 已知限制

- **file:// 协议下**：Chrome 对 `localStorage` 行为有差异；如数据没保存，请用 https:// 或 http:// 打开
- **Firefox / Safari**：本地同步（⇄）功能不可用（File System Access API 不支持）
- **跨终端同步**：依赖第三方网盘软件（OneDrive/iCloud/Dropbox）做文件同步，不做云端
- **多端同时写入**：会**后写覆盖前写**，已在 UI 警告
- **无暗色模式**：配色很易改（CSS 变量），但当前不提供
- **无云端账户系统**：纯本地工具，所有数据在用户设备

## License

MIT © 2026 EC11112 — 详见 [LICENSE](./LICENSE)
