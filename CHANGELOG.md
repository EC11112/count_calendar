# Changelog

## v0.3.0 — 2026-07-28

**字体算法拆分 + 屏占比设置 + 徽章阈值改公式**

四个收尾项，都跟 v0.2.5 的"统一字号"路线对齐：把"全局一套"改成"按视图分治 + 用户可调"。

**1. 月视图独立，不参与全局字号算法：**
- v0.2.5 让月视图也吃 `calcLabelFontSize`，结果月视图在 1080p 半屏下被 cellSize * 0.7 算成 28-42px，配合 badge 一起在月视图本来就够大的格子里"再加粗"。
- 现在月视图字号 CSS 写死：`.weekdays span` 14px / `.day` 14px / `.day .badge` 10px。回到 v0.2.4 之前的月视图原貌。
- 后续如果 strip 字号算法成熟再统一，这块留着不阻塞 strip。

**2. strip 字号算法调小（去掉伪加粗感）：**
- `LABEL_FONT_RATIO`: 0.7 → **0.5**
- `BADGE_FONT_RATIO`: 0.5 → **0.4**
- `MIN_LABEL_FONT`: 20 → **14**
- `MAX_LABEL_FONT`: 40 → **28**（2k+ 大屏 cap 收紧，避免再"看着像加粗"）
- `MAX_BADGE_FONT`: 24 → **18**
- 根因：font-size 接近格子边长时浏览器抗锯齿退化、字看着像 bold。现在 cellSize=41 → 字号 20（之前 28），cellSize=80 → 字号 28（cap）。

**3. 徽章渲染阈值改公式（避免小格重叠）：**
- v0.2.5 的 `BADGE_THRESHOLD = 35` 太小，1080p 半屏竖屏触发后"半"跟日期数字在 41px 格子里确实会挤。
- 改 `shouldShowBadge(cellSize) = cellSize >= BADGE_MIN_CELLSIZE`，其中 `BADGE_MIN_CELLSIZE = REF_FONT_FOR_BADGE * 4 = 14 * 4 = 56`。
- 解读：cellSize 要能放下 4 个参考最小字号（14px）的字符宽度，才切月视图样式（数字右上 + 徽章左下）。小格保持居中、只显示浅色底。
- 实测阈值：1080p 半屏竖屏（cellSize ≈ 41）→ 不触发；1080p 全屏（60-80）→ 触发；4k 屏（80-150）→ 触发。

**4. 新增「屏占比」设置（横屏占比）：**
- 大屏拉到 100% 太满：年视图 14 列 cellSize 只到 41，看着密密麻麻。
- 新增 `state.screenRatio`（默认 1.0），5 档：100% / 83% / 75% / 66% / 50%。
- `calculateStripLayout` 在算 cellSize 前先 `effectiveWidth = panel * screenRatio`，让 grid 居中显示。
- 屏占比生效时 strip 内的 `.weekday-row` / `.day-strip` 改成 `width: fit-content; margin: 0 auto`，grid 不会占满 panel。
- 设置菜单第二行加 `<select>`，旁边跟一个 `?` 圆圈 hover 显示悬浮注释：「横屏占比：4k/2k 大屏拉到 100% 容易显得太满，降到 75%–83% 更舒服；平板/手机全屏场景可以保持 100%」。
- 持久化到 `state.screenRatio`，进 localStorage（`count_calendar_v2`），跟 `maxStripCols` 平级。

**5. strip 容器支持居中（屏占比生效需要）：**
- `.strip-wrap` 加 `margin: 0 auto`。
- `.weekday-row` / `.day-strip` 改 `width: fit-content; max-width: 100%; margin-left/right: auto`。grid 容器只占实际需要的宽度（cols × cellSize + gaps），居中显示。

**6. 3/4 列 → 3/4 星期（注释澄清）：**
- 之前 v0.2.5 注释"MAX_STRIP_COLS 默认 35（≈ 49 * 3/4）"里的 3/4 指的是"3/4 个完整星期数"（0.75 * 7 ≈ 5.25，向下取整到 7 的倍数就是 35 = 5 周）。本版本注释改成"3/4 个完整星期数"，避免歧义。

**实现细节：**
- 月视图 `renderSingleMonth` 不再设 `weekdays.style.fontSize` / `daysEl.style.fontSize` / `badge.style.fontSize`，清空内联让 CSS 接管。
- 屏占比档位是离散 5 档，`SCREEN_RATIO_OPTIONS = [1.0, 0.83, 0.75, 0.66, 0.5]`，`normalizeScreenRatio` 做最邻近吸附。
- `loadState` 验证 `screenRatio` 是合法档位，非法/缺失都回到 1.0。

**保持不变：**
- 数据结构、storage key `count_calendar_v2`、点击循环、统计、a11y。
- 季/年视图的跨月不跳星期、周末红色。
- 4k 解锁（v0.2.5 已删 `.app { max-width }`）。
- 季/年视图的算法骨架（从 MAX 往下试 7 的倍数、cellSize ≥ 41）。

---

## v0.2.5 — 2026-07-28

**字体统一 + 大格子切月视图样式 + 4k 解锁 + 设置菜单**

四个收尾项。

**1. 统一月/季/年日历的格子内字体：**
- `calcLabelFontSize(cellSize) = min(40, max(20, cellSize * 0.7))` — weekday 标签和 date 数字统一这个公式
- `calcBadgeFontSize(cellSize) = min(24, max(9, cellSize * 0.5))` — "半"/"整" 徽章
- 41px 最小格子 → 28.7px 字体（≈ 格子下限 * 0.7）；80px 格子 → 40px（cap）
- 月视图的 `.weekdays span` / `.day` / `.day .badge` 也改成内联设置，去掉 CSS 里的固定 12px/13px/10px

**2. 大格子切月视图样式（cellSize ≥ 35）：**
- 日期在右上角（`position: absolute; top: 2px; right: 4px;`），不再是居中
- "半" / "整" 徽章在左下角（`position: absolute; bottom: 2px; left: 4px;`）
- 小格子（cellSize < 35）保持居中渲染
- 阈值 `BADGE_THRESHOLD = 35` 约对应 4-5 个 10px 字符宽度，用户说"4.5个字你按合适的来"

**3. 解除 `.app` 的 `max-width: 1400px`：**
- 之前这个限制让 4k 屏下 main panel 只能分到 ~816px → 算法只能选 14 列
- 去掉之后 4k 屏 main panel ~3256px → 4k 屏能跑到默认上限 35 列
- 2k+ 大屏也终于能用上自适应带来的多列体验

**4. 右上角新增设置按钮（⚙）：**
- 点击弹出菜单，含「最大列数」输入框（7 的倍数，范围 7-49）
- 默认 35（≈ 49 * 3/4），用户可调
- 存在 `state.maxStripCols`，进 localStorage（`count_calendar_v2`）
- 改完自动 re-render
- 提示文字「7 的倍数，41-49 之间效果最佳」

**实现细节：**
- `calculateStripLayout` 改用 `state.maxStripCols`（默认 35）作上限
- `buildStripCell` 接 `cellSize` 参数，自动判断 `useBadgeStyle` 并加 `.with-badge` class
- `renderStripEl` 算 `labelFont` / `badgeFont` 传给 cell
- `renderSingleMonth` 同样算 cellSize + labelFont + badgeFont
- resize 监听涵盖月视图（之前只 re-render 季/年）

**保持不变：**
- 数据结构、storage key、点击循环、统计、a11y
- 季/年视图的跨月不跳星期
- 月视图的布局（仍然是 7 列 1 周/行）

---

## v0.2.4 — 2026-07-28

**日历布局自适应视口**

之前季/年视图的列数和格子大小都写死，写死导致：
- 电脑 1080p / 2k+ 大屏：格子过大，浪费空间
- 手机竖屏 / 电脑半屏竖屏：格子过小，看不清
- 平板横屏：要么太小要么太大，没有 sweet spot

**算法（按用户要求）：**
1. **列数必须是 7 的倍数**（保证星期整除）
2. **格子必须正方形**（`aspect-ratio: 1/1`）
3. **格子像素必须 `≥ 41`**（`1080*0.8/21 ≈ 41`，舒适下限）
4. **在保证 (3) 的前提下尽量多列填满屏幕**

实现：从 `MAX_STRIP_COLS = 49`（7 周/行）往下试 7 的倍数（49, 42, 35, 28, 21, 14, 7），第一个 `cellSize ≥ 41` 的列数即为最优。如果所有列数都达不到 41（极窄屏 < ~290px），兜底用 7 列、cell 尽可能大（允许 < 41）。

**字体也跟着缩放：** `fontSize = cellSize * 0.2`，夹在 9-24px 之间。保证大屏不糊、小屏不爆字。

**视口场景适配**（实测）：
| 场景 | 面板宽 | 选中列数 | cellSize | fontSize |
|------|--------|----------|----------|----------|
| 手机竖屏 (375px) | ~340 | 7 | ~46 | 9 |
| 手机横屏/小平板 (618px) | ~585 | 7 | ~79 | 16 |
| 1080p (1920px) | ~710 | 14-21 | 41-49 | 9-10 |
| 2k (2560px) | ~990 | 21-28 | 41-45 | 9 |
| 4k (3840px) | ~1490 | 35-42 | 41-42 | 9 |

**resize 监听：** 窗口缩放时 debounce 100ms 重算布局，季/年视图自动重排。月视图（固定 7 列）不受影响。

**实现细节：**
- `calculateStripLayout(width)` 纯函数，输入容器宽，输出 `{ cols, cellSize }`
- `calcFontSize(cellSize)` 纯函数，输入格子边长，输出字号
- `measureStripWidth()` 读 `main.panel.clientWidth - 32`（去掉内边距），初次渲染拿不到 0 时兜底 280px
- `buildStripData(months, cols)` 接收动态列数
- `renderStripEl(data, cellSize, fontSize, stripClass)` 格子尺寸和字号都内联，不再依赖 CSS 变量
- `.day-strip` / `.weekday-row` 去掉 `grid-template-columns` 和 `font-size`（由 JS 动态算）
- `.strip-day` 去掉 `min-width: 0` 和固定 `font-size`
- `.day-strip.year-strip` 只保留 `gap: 1px`（年视图 53 行更紧凑）

**保持不变：**
- 数据结构、storage key `count_calendar_v2`、点击循环、统计、a11y
- 季/年视图的跨月不跳星期、周末红色
- 月视图（固定 7 列，单独处理）

---

## v0.2.3 — 2026-07-28

**季/年视图改造 + 今天按钮锚定**

三个收尾项。

**季视图去掉块名：**
- 原来季组件顶部还会显示「`7–9 月`」一行小字，和顶部 label「`2026 年 7–9 月`」重复。本版本拿掉。顶部 label 是单一来源。

**年视图独立实现（不再拼接季度组件）：**
- 旧年视图是「3 个 mini 季组件上下堆叠」，每个块还带自己的月份范围标题。看起来就像 3 个季视图拼起来。
- 新年视图是独立的 `renderYear()`：1 块，12 个月连续流，一周一行（7 列），整年 ≈ 53 行；只 1/1 加前缀空对齐周四，后续月份无缝接续。
- 视觉上加了 `.day-strip.year-strip` 紧凑样式：格子固定 20px 高、10px 字、gap 1px，53 行不爆高度。
- 季视图和年视图是两个独立函数（`renderQuarter` / `renderYear`），都通过 `buildStripData` + `renderStripEl` 共享同一套数据构造 + DOM 渲染逻辑，但入口分开，以后想各自演化不会牵动对方。

**今天按钮锚定到本季 / 本年：**
- 之前 `goToday` 不管 viewMode 都把 `viewMonth` 设成「今天的月份」。在年视图里这会导致 viewMonth=6 → 显示 7月2026-6月2027 整整错位一年；季视图里如果今天是 5 月，会落在「5-7月」这个奇怪的 3 月窗口里而不是 Q2。
- 现在按 viewMode 走：
  - `month` → `viewMonth = 今天`
  - `quarter` → `viewMonth = floor(今天月/3) * 3`（季首）
  - `year` → `viewMonth = 0`（年初）
- viewYear 仍然取今年（`t.getFullYear()`），所以从 2027 视图点今天会回到 2026 1-12 月。

**清理：**
- 删掉无用的 `WEEKS_PER_ROW` 常量、`buildStripBlock` 函数、`.strip-block-name` / `.strip-block-name .year-tag` CSS。

**保持不变：**
- 数据结构、storage key `count_calendar_v2`、点击循环、统计、a11y、季/年视图本身的跨月不跳星期。

---

## v0.2.2 — 2026-07-28

**修复季/年视图「跳星期」+ 清理 UI 噪音**

上一版流式 strip 仍有两个肉眼可感的 bug，本版本一起修掉。

**修复：**
- **跳星期（数据 bug）**：v0.2.1 的 `buildStripBlock` 在每个月前都按 `firstDow` 加前缀空。问题是 7月 1号 = 周三 → 3 个空，8月 1号 = 周六 → 6 个空，于是 8/1 实际被推到第 40 格（col 40 % 7 = 5 = 周五），与真实日期的周六列错位。本版本只在**第一个月**前加前缀空，后续月份无缝接续上个月最后一天，colIndex 用 `colIndex % 7` 算真实星期。
  - 验证：7/1 在 col 3（三）✅、7/31 在 col 12（五）✅、8/1 紧接 7/31 之后落在 col 13（六）✅、8/31 在 col 9（二，2026/8/31 是周一）✅
- **季/年视图锚定（交互 bug）**：从月视图切到季视图时，过去保留的是当前 `viewMonth`（可能是 7、8 或 9 任何一个），结果"本季"语义模糊。本版本切换时把 `viewMonth` 对齐到当前季起点（`Math.floor(m/3)*3`）或年初（0）。

**UI 清理：**
- 去掉块外的灰色大框（`.strip-block` 包裹层）、去掉块内的元信息文本（如「共 3 个月 · 3 周/行」），块头只剩干净的月份范围（`7–9 月` / `1–4 月`）。
- 周末日期数字本身也加红（之前只 weekday label 红，数字不变）。规则：`.strip-day.wkn:not(.logged) .strip-day-num` 用 `#ef4444`。
- 日/六 两种星期位置（日=列首、六=列尾）在跨周重复时一直保持红。

**保持不变：**
- 顶部 weekday 行仍用 `日 一 二 三 四 五 六` 重复 N 次（N = 季 3 / 年 4）
- 数据结构、storage key `count_calendar_v2`、点击循环、统计、a11y

---

## v0.2.1 — 2026-07-28

**重做大视图布局：连续流式 strip**

之前 v0.2.0 的季/年视图是"3 个独立 mini-month 块并排"，每个块自己带 `日一二三四五六` 标签、自己内部 7 列换行。现在改成"流式连续 strip"：

**变更：**
- 季视图：3 个月作为 **1 块** 横向流式展开，顶部一行 `日一二三四五六` 重复 3 次（21 列）。月份之间无缝，1 号用前缀空对齐到对应周几。
- 年视图：12 个月分成 **3 块**（每块 4 个月），每块 28 列（`日一二三四五六` 重复 4 次）。3 块堆叠。
- 块头信息更丰富：`1–4 月` + `共 4 个月 · 4 周/行`。
- 跨年时月名仍自动带年份标签（v0.2.0 已有）。

**示例（季视图，3 个月一字排开）：**
```
7–9月         共 3 个月 · 3 周/行
日 一 二 三 四 五 六 │ 日 一 二 三 四 五 六 │ 日 一 二 三 四 五 六
.  .  .  1  2  3  4 │ .  .  1  2  3  4  5 │ .  1  2  3  4  5  6
5  6  7  8  9 10 11 │ 6  7  8  9 10 11 12 │ 7  8  9 10 11 12 13
...
```

**优势：**
- 一行能放 3-4 个完整周（21 或 28 格），1080p 横向视野更宽
- 跨月连续，无独立块分隔，视觉密度更高
- 6 行就能展示一整个季度，比 v0.2.0 的"3 块 × 5-6 行"更紧凑

**数据兼容：**
- storage key 仍为 `count_calendar_v2`，旧数据无缝保留
- 数据结构未变，只改了渲染布局

---

## v0.2.0 — 2026-07-28

新增**季度视图（3 个月并排）**和**年视图（12 个月 4 行 × 3 列）**。

**新增：**
- 视图切换器（分段控件：`月` `季` `年`），位于月导航右侧
- `mini-month` 组件：每月一段，顶部 `月名` + `日 一 二 三 四 五 六` 七列标签，紧凑日期格
- 季视图：3 个月横向并排，prev/next 步进 ±3 月
- 年视图：12 个月 4 行 × 3 列（每行一个季度），prev/next 步进 ±12 月
- 跨年时月份名自动带年份标签（如 `1月 2027`）
- 视图标题根据模式动态变化：
  - 月：`2026 年 7 月`
  - 季：`2026 年 7–9 月`（跨年时 `2026.7 – 2027.1`）
  - 年：`2026 年`
- 统计"本 X"标签也跟随模式（本月 / 本季度 / 本年）
- 响应式：< 700px 季/年视图改 2 列；< 480px 改 1 列

**保持不变：**
- 点击循环 0→1（半天）→2（整天）→0
- 颜色深浅、统计、持久化
- 项目管理、删除确认
- a11y（mini-day 同样有 `role="button" tabindex="0" aria-label`）

---

## v0.1.0 — 2026-07-28

初始版本，单 HTML 文件实现。

**功能：**
- 左侧项目列表（默认 3 个：学习 / 运动 / 工作，可新建 / 删除）
- 项目主题色 8 色调色板
- 中间日历组件：
  - 跨月 / 跨年
  - 点击循环计数：0 → 半天（浅色 28% 透明度）→ 整天（深色 85% 透明度）→ 0
  - 今天有边框高亮
  - 未来日期不可点击
  - 周末列标签红色
  - 每个月第一天对齐周几
- 右侧统计：
  - 本月（整天 + 半天 → 折合天数）
  - 总计（同上）
  - 折合小时（按 1 天 = 8h 估算）
- localStorage 持久化
- 响应式布局（< 900px 单列）

**实现细节：**
- 单文件 `index.html`，内联 CSS + JS，零依赖
- 数据结构：`{ projects: [], currentProjectId, logs: { projectId: { "YYYY-MM-DD": 0|1|2 } } }`
- 颜色：CSS 变量 + 主题色按 alpha 转 RGBA
- a11y：日期格 `role="button" tabindex="0" aria-label="年月日"`

**已知限制：**
- file:// 下 Chrome 对 localStorage 行为偶尔不稳，建议用 http(s) 协议打开
- 没有暗色模式
- 没有导入 / 导出 / 跨设备同步
