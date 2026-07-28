# Changelog

## v0.3.4 — 2026-07-28

**方块大小上限全局化 + 屏占比换成全屏开关**

v0.3.3 第二波后又发现 2 个收尾问题：
- 100% 屏占比 + 星期数=1 时，季/年视图的 cellSize 不受 maxCellSize 限制（4k 大屏 + maxStripCols=7 → cellSize 算到 400+）
- 屏占比 5 档（100/83/75/66/50%）语义太阴间，用户搞不清楚每档差别，配置全屏与否直接用 on/off 就行

**1. 方块大小上限 (`maxCellSize`) 应用到所有视图：**
- 之前只在月视图截断 `cellSize`：`cellSize = min(rawCell, maxCellSize)`
- 季/年视图 `calculateStripLayout` 没截断 → 星期数=1 + 4k 屏 = cellSize 400+
- 修复：`calculateStripLayout` 加 `const cellSizeCap = state.maxCellSize || 80`，循环里 `cellSize = Math.min(raw, cellSizeCap)`
- 极端窄屏 fallback 不截（避免 cellSize 算成 < 20）

**2. 「月视图格子上限」→「方块大小上限」+ tooltip 改：**
- 旧 tooltip：月视图单个日期块的最大边长
- 新 tooltip：所有视图（月/季/年）日期块的最大边长，4k 大屏下防止格子过大，季/年视图也会按这个值截断 cellSize

**3. 屏占比 5 档 select → 单 checkbox「全屏」开关：**
- 旧 5 档语义不直观：100/83/75/66/50% 谁记得住比例
- 改成单一 checkbox，开/关二选一：
  - **开**（默认）：UI 撑满 viewport - 32
  - **关**：UI 由 `maxCellSize × 星期数 × 7` 反推，刚好包住日历年历
- 4k 大屏 + 星期数=5 + maxCellSize=80：UI = sidebar(240) + main(80×35+8×34=3072) + stats(280) + gaps/padding(32) = 3624
  - 全屏开：UI = 3808（满宽）
  - 全屏关：UI = 3624（缩了 184px，4k 屏下能看出差别）
- 4k 屏 + 星期数=1 + maxCellSize=80：UI = 240 + (80×7+8×6) + 280 + 32 = 1160
  - 全屏开：UI = 3808
  - 全屏关：UI = 1160（缩了 2648px，差异巨大 —— 这就是 100% 屏占比 + 星期数=1 看起来浪费的根因）
- 竖屏强制全屏（窄屏关掉会让 UI 比屏还宽更难看）

**4. state migration：**
- 旧 `state.screenRatio` (1.0/0.83/0.75/0.66/0.5) → 新 `state.fullscreen` (true/false)
- 旧值=1.0 → fullscreen=true（保持满宽体验）
- 旧值<1.0 → fullscreen=false（关全屏）
- migration 后 `delete state.screenRatio` 防止重复 migration

**5. CSS：**
- 加 `.settings-row input[type="checkbox"]` 样式：20×20px，accent-color 用主色，margin-left: auto 贴右对齐（跟 input/select 视觉一致）
- 竖屏下 `.screen-ratio-row` 仍然隐藏（强制全屏没意义给用户配置）

**保持不变：**
- 数据结构（除 fullscreen 替换 screenRatio）、storage key `count_calendar_v2`、点击循环、统计、a11y
- 4k 解锁、季/年 strip 跨月不跳星期、strip 居中（v0.3.3 的 justify-content: center）
- 月视图字号 CSS 写死、strip 字号算法 0.5/0.4 + cap 28/18、徽章阈值 cellSize ≥ 56
- 竖屏堆叠、min-width: 0 解决 flex 撑破

---

## v0.3.3 — 2026-07-28

**两波 bug 修复 + 4 个收尾改进**

v0.3.2 收尾后用户陆续发现一堆问题，分两波修完。第一波是 .ui-wrap 撑破 + maxCellSize 兜底两个隐蔽 bug（commit 16162a1），第二波是试用后的 4 个收尾改进（本次 commit）。

---

### 第二波：4 个收尾改进（本次 commit）

**a. 竖屏（< 900px）3 panel 改上下堆叠：**
- v0.3.2 把 `.app` 从 grid 改成 flex，v0.3.1 之前的 `@media (max-width: 900px) { .app { grid-template-columns: 1fr; } }` 失效
- 结果：689px 竖屏硬扛 3 列横排，main panel 缩到 100px，日历只能看 3 列
- **修复**：保留 flex 结构，media query 改 `.ui-wrap { grid-template-columns: 1fr; grid-auto-rows: auto; height: auto; }` + `.app { height: auto; min-height: 100vh; align-items: flex-start; }`
- 689px 竖屏：3 panel 竖排，各占满宽，日历能看完整 7 列
- max-width 仍然由 JS 算（appW - 32），保证两侧 16px 边距
- 横屏 (> 900px) 行为不变

**b. 月视图不应用屏占比（屏占比只对季/年视图有意义）：**
- 旧公式 `uiW = max(ratioUiW, cellUiW)`，月视图时 cellUiW 永远主导（maxCellSize 至少 80 → cellUiW=1160）
- 屏占比 75% 在月视图完全无效，配置了也没变化
- **修复**：`calcUiWidth` 里加 `const isMonth = state.viewMode === 'month'`，`ratio = isMonth ? 1.0 : state.screenRatio`
- 月视图：UI 由 cellUiW 决定（maxCellSize cap），屏占比不参与
- 季/年视图：UI = max(ratio * (appW-32), cellUiW)，屏占比生效
- 切换视图会自动重算（renderCalendar → applyScreenRatio）

**c. 季度/年视图 strip 在屏占比 < 100% 时居中：**
- v0.3.1 把 `.weekday-row` / `.day-strip` 从 `width: fit-content; margin: 0 auto` 改成 `width: 100%` 解决列对齐
- 副作用：75% 屏占比时 strip 比 panel 小，cells 贴左不居中（用户反馈）
- **修复**：保留 `width: 100%`（列对齐关键），加 `justify-content: center`（grid 内容居中）
- 两 grid 各自在 panel 宽度内居中 → strip 比 panel 小时 cells 在 panel 中央
- 顺带把 `.day-strip.year-strip` gap 从 1px 改 2px（跟 `.weekday-row` 一致）
  - 之前 1px 是为了 53 行紧凑，但跟 weekday 差 1px × 48 gap = 48px 列错位
  - 现在 2px + justify-content: center，年视图也列对齐

**d. 设置菜单 label/input 对齐统一：**
- 旧：`.settings-row` 用 `justify-content: space-between`，label 撑满、input 固定 100px
- label 长度不一致（"星期数" 3 字 vs "月视图格子上限" 6 字）导致视觉不齐
- **修复**：
  - label 容器 `<span>` 改 `flex: 1 1 auto; min-width: 0; overflow: hidden; text-overflow: ellipsis; white-space: nowrap`（左对齐 + 超长省略）
  - input/select 统一 `flex: 0 0 80px; width: 80px; margin-left: auto`（固定 80px + 永远贴右）
  - 所有行 label 起点 = 0，input 终点 = 菜单右边，视觉整齐

---

### 第一波：UI 整体宽度失控 + 月视图 maxCellSize 限制失效（commit 16162a1）

v0.3.2 收尾后还有两个隐蔽 bug，用户在 689px 竖屏窗口下发现：
- project / stats 块仍然贴屏幕两侧，16px 边距看不见
- 月视图不管窗口多窄，cellSize 永远是 34（maxCellSize 限制看起来"丢了"）

**a. `.ui-wrap` flex item 撑破 max-width：**
- `.ui-wrap` 是 `.app` 的 flex item，flex item 默认 `min-width: auto = min-content`
- min-content = sidebar(240) + main min-content + stats(280) + gaps ≈ 900px
- 即便 JS 写了 `style.maxWidth = '657px'`，flex item 还是被 min-content 撑大到 ~900px
- 后果：UI 永远撑满屏幕，屏占比 / 月视图限制都不生效
- **修复**：`.ui-wrap { min-width: 0; }`，允许 flex item 缩到 max-width 以下

**b. `measureMonthWidth()` / `measureStripWidth()` 兜底值过大：**
- 旧代码：`Math.max(inner, 280)` —— 本意是 init 时 main.panel 还没布局的应急
- 副作用：任何窄屏（包括 689px 竖屏 main 95px）都返回 280，导致 cellSize 永远按 280 算
- 月视图 `rawCell = (280-36)/7 = 34.86 → cellSize = 34`，看起来 maxCellSize 限制"丢了"
- 季/年 strip 同理：`calculateStripLayout(280)` → cellSize 至少 35-39
- **修复**：兜底从 280 降到 60（极端窄兜底，避免 cellSize 算成 0/负数），真实宽度如实返回
- 689px 竖屏实测：cellSize 现在正确缩到 28px 下限，maxCellSize=80 限制在 4k 屏下才显出价值

**c. UI 整体宽度公式重构 + JSDoc 列举所有影响因素：**
- 拆出 `calcUiWidth(appW)` 纯函数，输入 viewport 宽，返回 `{uiW, ratioUiW, cellUiW, ratio, maxCellSize, appW}`
- `applyScreenRatio()` 改成调用 + 落地（设置 .ui-wrap.style.maxWidth）
- JSDoc 列举全部 5 个影响因素：屏占比 / maxCellSize / sidebar+stats 固定宽 / padding+gap / APP_MARGIN
- 公式 `uiW = max(ratio*(appW-32), maxCellSize*7 + gap*6 + sidebar + stats + gap*2 + padding*2)` 不变
- 输出策略：太窄 fallback 撑满，否则永远 clamp 到 `appW-32` 留 16px 边距（**之前是 uiW >= appW 就清空 max-width 让 UI 撑满**，是导致贴屏幕的第二个原因）

---

## v0.3.2 — 2026-07-28

**间距全面收紧 + 屏占比改回 v0.3.0 思路 + 标签改名 + 竖屏隐藏屏占比**

v0.3.1 / v0.3.2 这一波全是 3 panel 布局的细节调优：sidebar / main / stats 紧贴屏幕两侧、紧凑排布、main 内部不留多余空白。

**1. 屏占比反向限制 main panel 居中（v0.3.1 改坏，v0.3.2 重做）：**
- v0.3.1 改 `.app` 的 `grid-template-columns: 240px mainWpx 280px`，3 panel 总宽 < .app 宽，**多余空间堆在 stats 右边**。用户反馈"项目/详情离日历非常远"、"对整个 body 都施加了屏占比限制"。
- v0.3.2 重做：**保持 `.app` grid 默认 `240px 1fr 280px` 不动**，只给 main panel 加 `max-width: mainW; margin: 0 auto`。
  - 1fr 列 = `frWidth`（= .app 宽 - sidebar 240 - stats 280 - 2 gap 8 - 2 padding 8，**自动撑满**）
  - main panel `max-width: mainW` + `margin: 0 auto` → 在 1fr 列内**居中**缩小
  - 1fr 列内 main panel 左右的空白属于 .app 背景（不是 stats 旁的"未用空间"）
  - sidebar / stats 始终紧贴 .app 边缘（grid 1fr 行为不变）
- 实测 4k 屏 75% 屏占比：sidebar 贴左、stats 贴右、main 居中 2442 宽，左右各 411 空白。

**2. 间距全面收紧：**
- `.app` gap 16 → **8**（panel 之间从 16px 减半到 8px）
- `.app` padding 0 → **8**（v0.3.1 改 0 让 panel 贴死屏幕边缘"很难看"，现在 8px 呼吸）
- `.cal-header` margin-bottom 16 → **8**：main panel 顶部 header 块（`<< 标题 >> 今天 月季年 ⚙`）之前 margin-bottom 16 跟 sidebar / stats 不对齐，main 顶部比两侧多 16+16+16=48px"标签占位"。改 8 后 3 panel 顶部对齐。
- `.panel` padding 16 → **14**（配合 gap 8 整体收紧）

**3. 「最大列数」→ 「星期数」（更准确）：**
- 配置项存的是"几个 7 列"，所以"星期数"比"最大列数"语义更直接。
- 提示文字保持 "列数 21-28（cellSize 41-49px）效果最佳"。

**4. 竖屏时屏占比配置隐藏 + 强制 100%：**
- 竖屏下 main panel 本来就窄，屏占比再缩就崩。
- CSS `@media (orientation: portrait) { .screen-ratio-row { display: none; } }` 隐藏设置行
- JS `applyScreenRatio` 用 `matchMedia('(orientation: portrait)')` 判断，竖屏时强制 ratio=1.0
- 切横屏自动恢复（监听 `orientationchange`）

**5. 窄窗口屏占比失效保护（保留自 v0.3.1）：**
- `if (frWidth < 1024)` 时屏占比失效，强制 100%。避免 800px 窗口 + 屏占比 75% → mainW=168px 季/年视图被压垮。

**6. 极小屏 main panel `min-width: 0`（保留自 v0.3.1）：**
- `main.panel { min-width: 0; }` 防止 .app grid 1fr 被季/年视图的 14-21 列 grid 内容撑大。

**保持不变：**
- 数据结构、storage key `count_calendar_v2`、点击循环、统计、a11y。
- 季/年视图的跨月不跳星期、周末红色。
- 4k 解锁（v0.2.5 已删 `.app { max-width }`）。
- strip 字号算法（v0.3.0 的 0.5/0.4）、徽章阈值（v0.3.0 的 cellSize >= 56）。
- 月视图字号 CSS 写死（v0.3.0 的 14/14/10）。

---

## v0.3.1 — 2026-07-28

**屏占比反向限制 main.panel + 日期格子和星期几严格对齐 + 选项框宽度一致**

四个收尾项，跟 v0.3.0 收尾项呼应：v0.3.0 给了"屏占比"开关但语义有偏差，本版本把语义修正成"反向控制 main panel 整体宽度"，同时修了 4k 下 grid 列对齐的问题。

**1. 屏占比反向限制 main.panel 整体宽度（v0.3.0 实现的实际只缩了 cellSize 算法）：**
- v0.3.0 屏占比只缩了 `calculateStripLayout(measureStripWidth() * ratio)`，main.panel 仍 100% 占满，sidebar/stats 跟着贴两侧，**视觉上没真正变窄**。
- v0.3.1 改成：屏占比 < 1 时，**直接改 `.app` 的 `grid-template-columns`**，把 1fr 列换成精确 px 宽度（`frWidth * ratio`），sidebar/stats 固定 240/280 不动，main 列 = `frWidth * ratio`。
- 同时给 main.panel 加 `max-width: mainW` + `margin: 0 auto` 兜底（防止某些场景 grid 1fr 又被挤压）+ 内部内容居中。
- 100% 屏占比：清除 inline，恢复默认 `240px 1fr 280px`，行为不变。
- 月视图 4k 下格子太大：**自动解决**。屏占比生效时 main panel 缩，月视图 cellSize 按缩小后的 main panel 算，格子不会爆。

**2. 日期格子和星期几严格对齐（4k 尤为明显）：**
- v0.3.0 给 `.weekday-row` / `.day-strip` 加了 `width: fit-content; max-width: 100%; margin-left/right: auto`，grid 容器自己按内容算宽度。
- 理论上两 grid 列数 + cellSize 一致应该 align，但 `.strip-wrap` 仍 100% 父容器 + box-sizing 差异 + content-box vs border-box 在某些场景（4k 大屏）会失效。
- v0.3.1 改成 `width: 100%`，两个 grid 都 stretch 到 main.panel 宽度，列严格对齐（v0.2.5 的行为）。
- 实测 100% 屏占比 + 14 列：weekday "日 一 二 三 四 五 六" 跟下面 7 个日期格的列边界完全重合，4k 大屏也保证对齐。

**3. 窄窗口屏占比失效保护：**
- 屏占比本来就是给 2k/4k 大屏设计的，窄窗口（如 800px）下应用屏占比会把 main panel 压到 ~100px，季/年视图直接崩。
- v0.3.1 加 `if (frWidth < 1024) return;` —— 窗口太窄时屏占比强制当 100% 处理，main panel 默认 1fr 全宽。
- 4k 屏（frWidth ≈ 3256）下屏占比 75% / 50% 正常工作；1080p 半屏（frWidth ≈ 700）下屏占比失效保护生效。

**4. 设置选项框宽度一致 + 提示文字改：**
- `.settings-row input[type=number]` 宽度 70px → **100px**（跟 select 一致）。
- 提示文字「7 的倍数，41–49 之间效果最佳」→「**列数 21–28（cellSize 41–49px）效果最佳**」：21-28 是推荐列数（3-4 周/行），cellSize 41-49px 是其对应的格子边长。

**实现细节：**
- `applyScreenRatio()` 函数在 `renderCalendar()` 入口先调用，resize 监听器也会触发。
- 算法：`frWidth = appWidth - 240 - 280 - 32 - 32`（sidebar + stats + 2 gap + 2 padding）；`mainW = floor(frWidth * ratio)`。
- `.app` 的 `grid-template-columns` 用 inline style 覆盖，100% 时清除让 CSS 默认生效。
- main panel 同时设 `max-width` + `margin auto` 兜底。

**保持不变：**
- 数据结构、storage key `count_calendar_v2`、点击循环、统计、a11y。
- 季/年视图的跨月不跳星期、周末红色。
- 4k 解锁（v0.2.5 已删 `.app { max-width }`）。
- strip 字号算法（v0.3.0 的 0.5/0.4）、徽章阈值（v0.3.0 的 cellSize >= 56）。
- 月视图字号 CSS 写死（v0.3.0 的 14/14/10）。

---

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
