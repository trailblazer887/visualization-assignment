# 中国人口可视化 · 双人并行任务分配

> 数据集：UN WPP 2024 + 世界银行 · 图表：D3 v7 · 交付：单页 HTML
> 版本：v3（解耦并行版） · 2026-11-19 · 状态：待认领
>
> **这份文档解决一个问题**：让 A 和 B 从第一天起就能各干各的，谁也不用等谁；只有两个汇合点（集成、终检）。

---

## 0. 并行开发怎么落地（先读）

### 三条硬规则

1. **契约先行**：第 3 节的接口在 Day1 冻结。之后两人只对着契约写代码，不互相商量细节。
2. **文件零重叠**：每份文件只有一个主人（第 2.2 节）。需要对方改文件＝设计错了，改契约或加一个自己的新文件。
3. **各带自测页**：A 有 `dev/dev-a.html`，B 有 `dev/dev-b.html`，都能**单独跑起来**，不依赖对方的代码。

### 为什么能并行

| 本来会卡住的地方 | 解耦手段 |
|---|---|
| B 等 A 的真实数据 | 契约冻结数据 schema；B 自建 `data/b-fixture.js`（同结构假数据），集成时换一行 `<script>` |
| A 等 B 的页面骨架 | A 的图只暴露 `Viz.charts.pyramid` 对象，`init(host, ctx)` 自取尺寸；A 用 `dev-a.html` 自己挂载 |
| A 要改 B 的 `index.html` | `index.html` 只归 B；A 不碰，集成时由 B 加容器与 script 标签 |
| A 的图依赖 B 的 `state.js` | A 不 import 任何 B 的文件；状态通过 `ctx` 注入、事件通过 `ctx.emit` 发出 |
| A 的图靠 B 的 CSS | A 用 SVG 属性 + `py-` 前缀 class 自给自足；B 的 CSS 不碰 `py-*` |
| 联调时才发现字段对不上 | 契约里给出**逐字段示例**（第 3.2 节），两人 Day1 各跑一次自测页确认 |

### 两个汇合点

| 检查点 | 时间 | 内容 | 谁主导 |
|---|---|---|---|
| **CP1 集成** | Day3 上午 | A 的四个文件落库，B 在 `index.html` 接上，跑通三图 | B 主导，A 修数据问题 |
| **CP2 终检** | Day5 | 按第 8 节清单互查、复现步骤验证 | 共同 |

除这两个点外，**没有日常交接**。

---

## 1. 项目目标与验收底线

**一句话**：用 UN《World Population Prospects 2024》与世界银行数据，做一个单页、纯前端、D3 v7 的可视化，同页三张联动图，讲清"中国人口金字塔如何从正三角变成倒钟形、出生与死亡何时交叉、老龄化在国际上什么位置"。

| 图 | 内容 | 负责人 |
|---|---|---|
| 图1 | 人口金字塔 + 年份滑块（1950–2100，可播放） | A |
| 图2 | 出生率/死亡率折线（1960–2024，可框选） | B |
| 图3 | 中/日/韩/美/印 65+ 占比对比（1950–2050） | B |

**技术约束（两人都遵守）**

| 项 | 决定 |
|---|---|
| 图表库 | D3 v7.9.0，本地 `vendor/d3.v7.9.0.js` |
| 运行 | 双击 `index.html` 即可（`file://`），无服务器、无构建、无外网、无 CDN |
| 脚本 | 全部 classic `<script>`，不用 ES module、不用 `fetch` 读本地数据 |
| 图表技术 | 全部 SVG（不用 Canvas） |
| 文案 | 中文界面，字段名英文 |

**项目级验收底线**

- [ ] 双击可运行，三图均渲染，控制台无报错
- [ ] 三图交互齐全且联动（第 3.5 节矩阵）
- [ ] 2024 年起（预测年份）在界面有明确标识
- [ ] 页脚标注数据来源、版本、截止年份
- [ ] 数据可复现：重跑脚本得到同一份 `data/population.js`
- [ ] `verify_data` 与 `smoke` 全部通过
- [ ] 交付物不含 `.docx`，文档统一为本 README

---

## 2. 分工总览

### 2.1 职责边界

| | **A · 数据与金字塔** | **B · 页面与折线** |
|---|---|---|
| 一句话 | 把数据搞对、把金字塔做出来 | 把页面搭起来、把两张折线做出来 |
| 交付给最终页面的东西 | `Viz.data`（数据访问层）+ `Viz.charts.pyramid` | 页面壳、状态机、`Viz.charts.trend`、`Viz.charts.compare`、KPI/控件 |
| 独自可验收的证明 | `dev/dev-a.html` 能单独跑金字塔 | `dev/dev-b.html` 能单独跑两张折线 + 布局 |
| 工时 | 23 h + 共同 3.5 h | 25 h + 共同 3.5 h |

### 2.2 文件所有权（零重叠）

| 文件/目录 | Owner | 说明 |
|---|---|---|
| `scripts/fetch_data.mjs`、`scripts/build_data.mjs` | A | 抓取与构建 |
| `src/data.js` | A | 数据访问层（`Viz.data`） |
| `src/chart-pyramid.js` | A | 图1 |
| `data/population.js`、`data/mock.js`、`data/raw/` | A | 产物与原始数据 |
| `docs/data-schema.md`、`tools/verify_data.mjs` | A | 数据文档与校验 |
| `dev/dev-a.html` | A | A 的自测页 |
| `index.html`、`src/style.css` | B | 页面壳（**A 不碰**） |
| `src/state.js`、`src/main.js`、`src/controls.js` | B | 状态、装配、KPI |
| `src/chart-trend.js`、`src/chart-compare.js` | B | 图2、图3 |
| `data/b-fixture.js` | B | B 的假数据（同契约结构） |
| `dev/dev-b.html` | B | B 的自测页 |
| `tools/smoke.mjs`、`tools/render_figures.mjs`、`images/` | B | 自检与预览 |
| `README.md` | 共同 | 各写自己那节，合并前互审 |

**唯一例外**：CP1 集成时，B 会修改 `index.html` 接线，A 只需要把文件放进对应路径，不参与编辑。

### 2.3 任务清单（详见第 4、5 节）

| 编号 | 任务 | 负责 | 工时 |
|---|---|---|---|
| A-1 | 抓取数据 | A | 3 h |
| A-2 | 构建数据 + 数据 API | A | 6 h |
| A-3 | 数据校验 | A | 2 h |
| A-4 | 图1 金字塔 | A | 7 h |
| A-5 | 图1 交互 + `dev-a.html` 自测 | A | 4 h |
| A-6 | README 数据章 | A | 1 h |
| B-1 | 页面骨架 + 状态机 + 假数据 | B | 5 h |
| B-2 | 图2 出生/死亡折线 | B | 7 h |
| B-3 | 图3 国际对比 | B | 6 h |
| B-4 | KPI 卡与控件 | B | 3 h |
| B-5 | `dev-b.html` + 自检 + 预览图 | B | 3 h |
| B-6 | README 使用章 | B | 1 h |
| C-1 | 契约冻结（30 min） | 共同 | 0.5 h |
| C-2 | CP1 集成 | 共同 | 2 h |
| C-3 | CP2 终检 | 共同 | 1 h |

---

## 3. 契约（Day1 冻结，逐字遵守）

### 3.1 全局命名与加载顺序

全局只允许一个命名空间 `window.Viz`；除此之外不产生任何全局变量。

```html
<!-- 最终 index.html 的脚本顺序（B 维护） -->
<script src="vendor/d3.v7.9.0.js"></script>
<script src="data/population.js"></script>    <!-- A：定义 window.POP_DATA -->
<script src="src/data.js"></script>           <!-- A：由 POP_DATA 生成 Viz.data -->
<script src="src/chart-pyramid.js"></script>  <!-- A：定义 Viz.charts.pyramid -->
<script src="src/chart-trend.js"></script>    <!-- B -->
<script src="src/chart-compare.js"></script>  <!-- B -->
<script src="src/state.js"></script>          <!-- B -->
<script src="src/main.js"></script>           <!-- B：装配与分发 -->
```

### 3.2 数据访问层（A 实现，B 只调）

```js
window.Viz = window.Viz || {};
Viz.data = {
  meta(),                        // { sources, fetchedAt, years:[1950,2100], ageGroups:[21 个], projectionFrom:2024 }
  pyramid(year),                 // { male:[21], female:[21] }，单位：人
  total(year),                   // 中国该年总人口（人）
  share(year, minAge),           // 中国该年 ≥minAge 岁占比（0~1）
  compare(countryCode, minAge),  // { years:[...], values:[...] }，CHN/JPN/KOR/USA/IND
  rates(),                       // { yearsRates:[1960..2024], birth, death, yearsPop:[1960..2025], pop }
  latest(),                      // { popYear, pop, birthYear, birth, deathYear, death }
  isProjection(year)             // boolean
};
```

**逐字段示例**（两人 Day1 各跑一次自测页确认）

```js
Viz.data.meta()          // { years:[1950,2100], ageGroups:['0-4',...,'100+'], projectionFrom:2024 }
Viz.data.pyramid(2023)   // { male:[31_000_000, ...21 个], female:[29_000_000, ...] }
Viz.data.total(2023)     // 1_422_000_000
Viz.data.share(2023,65)  // 0.148
Viz.data.compare('JPN',65) // { years:[1950,...,2050], values:[0.049,...] }
Viz.data.rates()         // { yearsRates:[1960,...,2024], birth:[...], death:[...], yearsPop:[...], pop:[...] }
Viz.data.latest()        // { popYear:2025, pop:1406585000, birthYear:2024, birth:6.77, deathYear:2024, death:7.76 }
```

### 3.3 图表注册协议

```js
window.Viz = window.Viz || {};
Viz.charts = window.Viz.charts || {};

Viz.charts.pyramid = {          // A 写
  init(hostEl, ctx),            // ctx = { data: Viz.data, emit(evt, payload) }
  render(state),                // 状态变化时重绘
  resize()
};
```

**约束**
- 图表**不得**直接引用 `Viz.data`，只用 `ctx.data`（这样 B 的假数据页也能跑）。
- 图表**不得** import 或引用任何其他图表/状态文件。
- 图表尺寸必须每次 `render` 时从 `hostEl.getBoundingClientRect()` 读，不缓存死值。

### 3.4 容器与尺寸

| 容器 ID | 高度 | 归属 |
|---|---|---|
| `#chart-pyramid` | 560 px | A 的图渲染进去 |
| `#chart-compare` | 560 px | B |
| `#chart-trend` | 320 px | B |

B 在 `index.html` 中保证容器有显式高度；A 的图自适应宽高。

### 3.5 状态与事件

```js
{
  year: 1950..2100,
  playing: false,
  brushRange: null | [a, b],
  countries: Set<'CHN'|'JPN'|'KOR'|'USA'|'IND'>,
  hoverAge: string | null
}
```

| 事件 | 发起方 | 载荷 |
|---|---|---|
| `year` | 控件 / 图2 | number |
| `brush` | 图2 | `[start,end]` 或 `null` |
| `country` | 图3 | 国家代码 |
| `hoverAge` | 图1 | 年龄组或 `null` |
| `play` | 控件 | boolean |

| 触发 | 图1 | 图2 | 图3 | KPI |
|---|---|---|---|---|
| 拖年份滑块 | 重绘该年 | 十字线移动 | 十字线移动 | 更新 |
| 播放 | 逐帧重绘 | 跟随 | 跟随 | 更新 |
| 框选 | 区间内循环播放 | 选区高亮 | 跟随 | 更新 |
| 点国家 | 不变 | 不变 | 高亮切换 | 显示该国占比 |

### 3.6 CSS 约定

- B 只定义布局与通用组件类：`.grid`、`.panel`、`.kpi`、`.toolbar`、`.footer` 等。
- A 的图只用 `py-` 前缀类（`.py-bar`、`.py-label`…）或 SVG 属性写样式；**不引用 B 的类**。
- B 的 CSS **不得**选择器命中 `svg` 内部元素或 `py-*`。
- 提示框：各地图自己实现（位置相对 host），不共用 B 的组件。

### 3.7 变更规则

任何字段、事件、容器 ID 的改动：发起方在聊天里说明 → 另一方确认 → 记入附录 A → 改代码。**未确认不得单方面改**。

---

## 4. A 的任务包（数据与金字塔）

> 全程可在 `dev/dev-a.html` 中自测，不需要 B 的任何代码。

### A-1 抓取数据 ｜ 3 h ｜ 前置：无

**要做**
- `scripts/fetch_data.mjs`：下载 WPP 2024 CSV（29,948,947 B）到 `data/raw/`，再取世行 3 个指标（总人口 1960–2025、出生率/死亡率 1960–2024）到 `data/raw/wb_*.json`
- 重试 5 次；文件已存在且大小一致则跳过
- `data/raw/fetch-meta.json`：URL、UTC 时间、字节数、sha256
- 脚本内把进程级 `TMP`/`TEMP` 指向 `tmp/temp`

**产出**：`scripts/fetch_data.mjs`、`data/raw/wpp2024_pop5_medium.csv.gz`、`data/raw/wb_*.json`、`data/raw/fetch-meta.json`

**验收**
- [ ] 重跑不重复下载
- [ ] 断网时错误信息明确，提示手动放 CSV 的位置
- [ ] `fetch-meta.json` 4 条记录、sha256 齐全
- [ ] `%TEMP%` 下无本次新增

### A-2 构建数据 + 数据 API ｜ 6 h ｜ 前置：A-1

**要做**
- `scripts/build_data.mjs`：`zlib.createGunzip()` + `readline` **流式**读（解压后约 200–300 MB）
- 手写 CSV 行解析（BOM、引号内逗号）；`LocID ∈ {156,392,410,840,356}`，`Variant=Medium`
- 千人 → 人（×1000 取整）；聚合 `pyramid` / `total` / `share65` / `compare`
- 合并世行 → `rates`、`pop`、`latest`
- 输出 `data/population.js`（`window.POP_DATA`）+ `data/mock.js`（3 年样本，同结构，供 `dev-a.html` 用）
- `docs/data-schema.md` 写清字段、单位、来源
- `src/data.js`：由 `POP_DATA` 组装 `Viz.data`（严格按 3.2 节签名）

**产出**：`scripts/build_data.mjs`、`data/population.js`、`data/mock.js`、`src/data.js`、`docs/data-schema.md`

**验收**
- [ ] `population.js` < 1 MB，151 年连续，21 组
- [ ] `Viz.data` 每个方法返回值与 3.2 节示例形状一致
- [ ] `mock.js` 结构与真实数据完全一致

### A-3 数据校验 ｜ 2 h ｜ 前置：A-2

**要做**：`tools/verify_data.mjs` 断言
- 每年 `Σ male`、`Σ female` ≈ `PopTotal`（误差 < 1%）
- 年份 1950–2100 连续 151 个，年龄组 21 个
- WPP 2023 中国总人口约 14.2 亿（±3%）；世行 2025 = 1,406,585,000
- 世行序列无断年；2022 死亡率 7.37 > 出生率 6.77；2024 出生率回升 6.39 → 6.77
- 无 `NaN`、无负值

**产出**：`tools/verify_data.mjs`

**验收**：全绿；故意改坏一个数能报错

### A-4 图1 金字塔 ｜ 7 h ｜ 前置：A-2

**要做**
- `src/chart-pyramid.js`：男左女右、21 组、左右对称比例尺
- hover 左右同时高亮 + tooltip（人数、占比）
- 2024+ 柱体降透明度 + "预测"角标
- 只通过 `ctx.data` 取数，`init/render/resize` 三件套

**验收**
- [ ] 1950 正三角、2023 收缩、2050 倒钟形，肉眼可辨
- [ ] hover 数值与 `verify_data` 抽查一致
- [ ] 容器宽度变化后 `resize()` 正确

### A-5 图1 交互 + 自测页 ｜ 4 h ｜ 前置：A-4

**要做**
- 年份滑块 1950–2100 + 播放/暂停（约 80 ms/年，`requestAnimationFrame`，每帧最多 1 年）
- 快捷年份：1950 / 1980 / 2000 / 2023 / 2050
- 响应 `brush`：在框选区间内循环播放
- `dev/dev-a.html`：只加载 d3 + mock + data.js + chart-pyramid.js，自绘一个临时滑块，验证上述全部交互

**验收**
- [ ] `dev-a.html` 单独打开即可完整演示金字塔（含滑块/播放/hover）
- [ ] 30 fps 以上；播放/拖动互不打架
- [ ] 框选区间播放只在该区间循环

### A-6 README 数据章 ｜ 1 h ｜ 前置：A-3

写"数据来源与时效""口径与限制"两节：WPP 2024 为最新版、估算止于 2023、世行补到 2025/2024、两套口径不混算。

---

## 5. B 的任务包（页面与折线）

> 全程可在 `dev/dev-b.html` 中用 `data/b-fixture.js` 自测，不需要 A 的任何代码。

### B-1 页面骨架 + 状态机 + 假数据 ｜ 5 h ｜ 前置：无

**要做**
- `index.html`：顶部标题 + KPI 行 + 控件；第一行 `#chart-pyramid`（7 列）+ `#chart-compare`（5 列）；第二行 `#chart-trend`；页脚来源说明；脚本顺序按 3.1 节
- `src/style.css`：布局与通用组件类；不碰 `svg` 内部与 `py-*`
- `src/state.js`：唯一状态源 + 事件总线（`on`/`emit`），字段见 3.5 节
- `src/main.js`：遍历 `Viz.charts` 调 `init(host, ctx)`，状态变化时调 `render(state)`，窗口变化时调 `resize()`
- `data/b-fixture.js`：**按 3.2 节契约**造一份假数据（151 年 × 21 组可用正弦/线性生成），并实现一个临时的 `Viz.data` 桩（只在 `dev-b.html` 用）
- `dev/dev-b.html`：加载 d3 + b-fixture + 图2/图3 + 状态机 + 一个 10 行的假金字塔，验证布局与联动

**产出**：`index.html`、`src/style.css`、`src/state.js`、`src/main.js`、`data/b-fixture.js`、`dev/dev-b.html`

**验收**
- [ ] `dev-b.html` 单独打开：布局正确、两张折线能响应年份/点击
- [ ] 双击 `index.html` 无报错（此时仅两张折线 + 假金字塔）
- [ ] 窗口缩放不错位

### B-2 图2 出生/死亡折线 ｜ 7 h ｜ 前置：B-1

**要做**
- `src/chart-trend.js`：两条折线（‰），x = 1960–2024
- 标注：1963 高峰、1987 峰值、2016 二孩、**2022 死亡>出生**、**2024 龙年回升**
- `d3.brushX` → 发 `brush`；hover 显示出生率/死亡率/自然增长率
- 全局年份十字线；副轴可切"总人口"（1960–2025）
- 图注写明世行口径

**验收**
- [ ] 2022 交叉标注位置正确（7.37 vs 6.77）
- [ ] 框选触发 `brush`，取消发 `null`
- [ ] hover 数值与 `ctx.data.rates()` 一致

### B-3 图3 国际对比 ｜ 6 h ｜ 前置：B-1

**要做**
- `src/chart-compare.js`：5 国 65+ 占比（1950–2050），中国加粗
- 点选国家高亮/淡化；hover 显示占比
- 切换 65+ / 0–14
- 与全局年份十字线同步

**验收**
- [ ] 5 条线齐全，2024+ 虚线
- [ ] 点选后其余淡化，KPI 同步

### B-4 KPI 卡与控件 ｜ 3 h ｜ 前置：B-1

总人口（世行 2025）、65+ 占比（WPP，标注年份与估算/预测）、0–14 占比、出生/死亡率（世行 2024）；年份滑块与播放按钮；国家开关。

**验收**
- [ ] 年份变化 KPI 同步；1960 年前出生/死亡率显示"—"
- [ ] 预测年份有文字标识

### B-5 自测页收尾 + 交互自检 + 预览图 ｜ 3 h ｜ 前置：B-2、B-3

**要做**
- 完善 `dev/dev-b.html`（给 A 也留一个可打开的空壳说明）
- `tools/smoke.mjs`：jsdom 加载整页（`runScripts:'dangerously'`、`resources:'usable'`、`pretendToBeVisual:true`），`beforeParse` 设 `window.__TEST__=true` 关动画；断言金字塔柱 42、图2 路径 2、图3 路径 5、KPI 非空；模拟改年份/点国家/框选
- `tools/render_figures.mjs`：jsdom → SVG → `@resvg/resvg-js` 出 `images/*.png`
- 两个工具用 `createRequire` 只读引用 `作业4\d3-quickstart\node_modules`

**验收**
- [ ] `smoke.mjs` 全绿
- [ ] 三张 PNG 中文不丢字

### B-6 README 使用章 ｜ 1 h ｜ 前置：B-5

写"运行方式""交互说明""自检方法"三节。

---

## 6. 检查点与集成

### 6.1 CP1 集成（Day3 上午，约 2 h）

| 步骤 | 操作 | 谁 |
|---|---|---|
| 1 | 交付四个文件：`data/population.js`、`src/data.js`、`src/chart-pyramid.js`、`docs/data-schema.md` | A |
| 2 | 在 `index.html` 里确认 3 个容器 + 按 3.1 顺序加 A 的 3 个 script；确认生产页不引用 `b-fixture.js`（它只服务 `dev-b.html`） | B |
| 3 | `node tools/verify_data.mjs`、`node tools/smoke.mjs` | 双方 |
| 4 | 各自按第 8 节清单抽查 15 分钟 | 双方 |
| 5 | 问题分类：数据字段 → A；事件/布局 → B；当天闭环 | 分工修 |

**集成失败预案**：若 2 小时内未跑通，B 先用一个 20 行桩金字塔保住页面可用，A 回 `dev-a.html` 继续调试，次日再集成；B 的其余任务不受影响。

### 6.2 CP2 终检（Day5，约 1 h）

- 双方按第 8 节清单互查
- 从零复现一次：`fetch → build → verify → 双击打开`
- 确认交付物清单与"不含 docx"

---

## 7. 数据与技术要点（两人都要知道）

### 7.1 数据来源与时效

| 数据 | 来源 | 覆盖 |
|---|---|---|
| 5 岁组人口 × 性别 | UN WPP 2024 `WPP2024_PopulationByAge5GroupSex_Medium.csv.gz`（29.9 MB） | 1950–2100 |
| 总人口 | 世行 `SP.POP.TOTL` | 1960–2025 |
| 出生/死亡率 | 世行 `SP.DYN.CBRT.IN` / `SP.DYN.CDRT.IN` | 1960–2024 |
| 降级备用 | OWID `population-by-age-group.csv`（5 大年龄段） | 1950–2100 |

**时效关键**：WPP 两年一版，**2026 版未发布**（官方 2026 CSV 为 404），WPP 2024 是当前最新版；估算止于 2023，2024–2100 为预测；2024–2025 实际值用世行补齐。

**已核实数值**：2025 总人口 1,406,585,000；2024 出生率 6.77‰、死亡率 7.76‰；2022 死亡率 7.37‰ 首次超过出生率 6.77‰。

### 7.2 口径限制（写进页面与 README）

1. 2024 年起为预测，界面必须标识；
2. WPP 的 "China" 不含港澳台；
3. 金字塔/对比图用 WPP，折线用世行，两套口径差约 1%，**不混算**；
4. UN Data Portal 数据接口需 API key（401），故走原始 CSV。

### 7.3 下载与落盘规范（不污染全局环境）

| 动作 | 位置 | 大小 |
|---|---|---|
| WPP CSV | `data\raw\wpp2024_pop5_medium.csv.gz` | 29.9 MB |
| 世行数据 | `data\raw\wb_*.json` | < 100 KB |
| 元信息 | `data\raw\fetch-meta.json` | < 1 KB |
| 构建产物 | `data\population.js` | < 1 MB |
| 临时文件 | `tmp\`（`TMP`/`TEMP` 指向 `tmp\temp`） | 可变 |
| d3 | `vendor\d3.v7.9.0.js`（从 `作业4` 复制，非下载） | 587 KB |

- 不在 `Downloads`、桌面、`%USERPROFILE%`、`AppData` 落文件
- 不跑 `npm install` / `npm i -g` / `pip install`
- 自检工具用 `createRequire` 只读引用 `作业4\d3-quickstart\node_modules`，缺失则跳过预览
- `TMP`/`TEMP` 只对进程生效

### 7.4 三图标注要点

| 图 | 必须出现 |
|---|---|
| 图1 | 1950 正三角、2023 收缩、2050 倒钟形；2024+ 预测角标 |
| 图2 | 1963 高峰、1987 峰值、2016 二孩、2022 死亡>出生、2024 龙年回升 |
| 图3 | 五国对比；2024+ 虚线 |

---

## 8. 验收标准

### 8.1 项目级

- [ ] 双击 `index.html` 可运行，三图渲染，无报错
- [ ] 三图交互齐全且联动正确
- [ ] 预测年份有标识，页脚有来源与截止年份
- [ ] `verify_data` / `smoke` 全绿
- [ ] 交付物不含 docx

### 8.2 A 的验收

- [ ] `dev/dev-a.html` 单独可完整演示金字塔（滑块/播放/hover/框选响应）
- [ ] `Viz.data` 全部方法符合 3.2 节签名与示例
- [ ] `verify_data` 全绿
- [ ] 图1 三个年份形态可辨，预测有标识
- [ ] 数据章节数字与脚本输出一致

### 8.3 B 的验收

- [ ] `dev/dev-b.html` 单独可运行（假数据）
- [ ] 图2 标注正确、brush 事件载荷正确
- [ ] 图3 五国线齐全、点选联动
- [ ] KPI 随年份更新、预测有标识
- [ ] `smoke` 全绿、三张 PNG 中文不丢字
- [ ] 使用章节完整

---

## 9. 风险与应对

| 风险 | 影响 | 应对 | 触发人 |
|---|---|---|---|
| 契约理解偏差 | CP1 返工 | 3.2 节逐字段示例 + 各自自测页 Day1 验证 | 双方 |
| WPP 30 MB 下载不稳 | 阻塞 A 下游 | 重试 5 次 + sha256；失败切 OWID 降级 | A |
| A 的图在 B 页面里尺寸不对 | 返工 | 约定容器显式高度，A 每次 render 读 host 尺寸 | A/B |
| 两套口径被误读 | 结论可信度 | 页面图注 + README 明确不混算 | A |
| 播放掉帧 | 体验差 | 每帧最多 1 年 + 测试模式关动画 | A/B |
| WPP 2026 期间发布 | 数据显旧 | README 记版本；重跑管线即可切换 | A |
| 预览图字体缺失 | PNG 丢字 | resvg 指定 `C:/Windows/Fonts` | B |

---

## 10. 目录结构与复现步骤

```
大作业/
├─ README.md
├─ index.html                    # B
├─ src/
│  ├─ state.js  main.js  controls.js  style.css   # B
│  ├─ chart-trend.js  chart-compare.js            # B
│  ├─ chart-pyramid.js                            # A
│  └─ data.js                                     # A
├─ data/
│  ├─ population.js  mock.js  raw/                # A
│  └─ b-fixture.js                                # B
├─ dev/
│  ├─ dev-a.html                                  # A
│  └─ dev-b.html                                  # B
├─ vendor/d3.v7.9.0.js
├─ scripts/fetch_data.mjs  build_data.mjs         # A
├─ docs/data-schema.md                            # A
├─ tools/verify_data.mjs                          # A
├─ tools/smoke.mjs  render_figures.mjs            # B
├─ images/                                        # B（可选）
└─ tmp/                                           # 临时，可删
```

```powershell
# 准备 d3（复制，不是下载）
New-Item -ItemType Directory -Force .\vendor | Out-Null
Copy-Item 'D:\可视化作业\作业4\histogram-html\d3.js' .\vendor\d3.v7.9.0.js

# A 的流程
node .\scripts\fetch_data.mjs
node .\scripts\build_data.mjs
node .\tools\verify_data.mjs
Start-Process .\dev\dev-a.html

# B 的流程（无 A 也能跑）
Start-Process .\dev\dev-b.html

# 集成后
Start-Process .\index.html
node .\tools\smoke.mjs
node .\tools\render_figures.mjs
```

**交付物**：`index.html`、`src/`、`data/population.js`、`vendor/d3.v7.9.0.js`、`scripts/`、`docs/`、`README.md`。
**开发期文件（不交付）**：`dev/`、`data/mock.js`、`data/b-fixture.js`、`data/raw/`、`tools/`、`tmp/`。

---

## 附录 A · 变更记录

| 日期 | 变更 | 提出/确认 |
|---|---|---|
| 2026-11-19 | v1：确定数据集、三张图、双人分工 | — |
| 2026-11-19 | v2：改为任务卡片结构，补交接点 | — |
| 2026-11-19 | v3：解耦并行——契约冻结、各自自测页、文件零重叠、汇合点收敛为 CP1/CP2 | — |
| 2026-11-19 | 移除 `.docx` 交付物，文档统一为 README | — |

> 接口变更也记这里：日期 / 改了哪个字段 / 谁提出 / 谁确认。

---

## 附录 B · 任务勾选表（可打印）

| 编号 | 任务 | 负责人 | 工时 | 完成 |
|---|---|---|---|---|
| A-1 | 抓取数据 | A | 3 h | ☐ |
| A-2 | 构建数据 + 数据 API | A | 6 h | ☐ |
| A-3 | 数据校验 | A | 2 h | ☐ |
| A-4 | 图1 金字塔 | A | 7 h | ☐ |
| A-5 | 图1 交互 + 自测页 | A | 4 h | ☐ |
| A-6 | README 数据章 | A | 1 h | ☐ |
| B-1 | 骨架 + 状态机 + 假数据 | B | 5 h | ☐ |
| B-2 | 图2 出生/死亡折线 | B | 7 h | ☐ |
| B-3 | 图3 国际对比 | B | 6 h | ☐ |
| B-4 | KPI 卡与控件 | B | 3 h | ☐ |
| B-5 | 自测页 + 自检 + 预览图 | B | 3 h | ☐ |
| B-6 | README 使用章 | B | 1 h | ☐ |
| C-1 | 契约冻结 | 共同 | 0.5 h | ☐ |
| C-2 | CP1 集成 | 共同 | 2 h | ☐ |
| C-3 | CP2 终检 | 共同 | 1 h | ☐ |
