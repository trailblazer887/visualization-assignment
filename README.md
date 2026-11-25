# 中国人口可视化 · 双人并行任务分配

> 数据集：UN WPP 2024 + 世界银行 · 图表：D3 v7 · 交付：单页 HTML
> 版本：v4（年龄结构与跨国联动） · 2026-11-25 · 状态：待认领
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
| A 等 B 的页面骨架 | A 注册 `Viz.charts.pyramid` 和 `Viz.charts.ageStructure`，各图用 `init(host, ctx)` 自取尺寸。A 用 `dev-a.html` 挂载两图 |
| A 要改 B 的 `index.html` | `index.html` 只归 B；A 不碰，集成时由 B 加容器与 script 标签 |
| A 的图依赖 B 的 `state.js` | A 不 import 任何 B 的文件；状态通过 `ctx` 注入、事件通过 `ctx.emit` 发出 |
| 图表样式依赖页面样式 | A 用 SVG 属性及 `py-`、`age-` 前缀定义图表样式，B 负责页面布局 |
| 联调时才发现字段对不上 | 契约里给出**逐字段示例**（第 3.2 节），两人 Day1 各跑一次自测页确认 |

### 两个汇合点

| 检查点 | 时间 | 内容 | 谁主导 |
|---|---|---|---|
| **CP1 集成** | Day4 上午 | B 在 `index.html` 接入 A 的图表和数据，跑通四图与跨国联动 | B 主导，A 修数据问题 |
| **CP2 终检** | Day5 | 按第 8 节清单互查、复现步骤验证 | 共同 |

除这两个点外，**没有日常交接**。

---

## 1. 项目目标与验收底线

用 UN《World Population Prospects 2024》与世界银行数据，在单页呈现四张联动图，展示中国人口的年龄结构如何变化、自然增长率何时转负，以及中国与其他国家在人口分布和出生死亡趋势上的差异。

| 图 | 内容 | 负责人 |
|---|---|---|
| 图1 | 人口金字塔与右侧年龄人口合计折线（1950–2100）。同一幅图，可拖动年份滑块或播放 | A |
| 图2 | 出生率、死亡率、自然增长率折线（1960–2024），支持框选和跨国对比 | B |
| 图3 | 中、日、韩、美、印 65 岁及以上或 0–14 岁人口占比（1950–2050），可选择跨国对比对象 | B |
| 图4 | 中国 0–14 岁、15–64 岁、65 岁及以上人口占比堆叠面积图（1950–2100） | A |

**技术约束（两人都遵守）**

| 项 | 决定 |
|---|---|
| 图表库 | D3 v7.9.0，本地 `vendor/d3.v7.9.0.js` |
| 运行 | 双击 `index.html` 即可（`file://`），无服务器、无构建、无外网、无 CDN |
| 脚本 | 全部 classic `<script>`，不用 ES module、不用 `fetch` 读本地数据 |
| 图表技术 | 全部 SVG（不用 Canvas） |
| 文案 | 中文界面，字段名英文 |

**项目级验收底线**

- [ ] 双击可运行，四图均渲染，控制台无报错
- [ ] 四图按第 3.5 节联动，点选对比国后图1和图2同步显示对比曲线
- [ ] WPP 2024 年起的预测年份在界面有明确标识
- [ ] 页脚标注数据来源、版本、截止年份
- [ ] 使用同一批原始文件重跑构建脚本，得到同一份 `data/population.js`
- [ ] `verify_data` 与 `smoke` 全部通过
- [ ] 交付物不含 `.docx`，文档统一为本 README

---

## 2. 分工总览

### 2.1 职责边界

| | **A · 数据、金字塔与年龄结构** | **B · 页面与折线** |
|---|---|---|
| 职责 | 整理多国数据，实现人口金字塔、右侧合计折线和年龄结构面积图 | 实现页面、出生死亡趋势图和国际对比图，装配跨国联动 |
| 交付给最终页面的东西 | `Viz.data`、`Viz.charts.pyramid`、`Viz.charts.ageStructure` | 页面壳、状态机、`Viz.charts.trend`、`Viz.charts.compare`、KPI 与控件 |
| 独自可验收的证明 | `dev/dev-a.html` 单独运行图1和图4，可模拟选国 | `dev/dev-b.html` 单独运行图2和图3，用图1和图4的占位验证联动 |
| 估计工时 | 34 h + 共同 3.5 h | 30 h + 共同 3.5 h |

### 2.2 文件所有权（零重叠）

| 文件/目录 | Owner | 说明 |
|---|---|---|
| `scripts/fetch_data.mjs`、`scripts/build_data.mjs` | A | 抓取与构建 |
| `src/data.js` | A | 数据访问层（`Viz.data`） |
| `src/chart-pyramid.js`、`src/chart-age-structure.js` | A | 图1、图4 |
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
| A-1 | 抓取五国数据 | A | 4 h |
| A-2 | 构建数据 + 数据 API | A | 8 h |
| A-3 | 数据校验 | A | 3 h |
| A-4 | 图1 金字塔与年龄人口合计折线 | A | 9 h |
| A-5 | 图1 交互 + `dev-a.html` 自测 | A | 4 h |
| A-6 | README 数据章 | A | 1 h |
| A-7 | 图4 年龄结构堆叠面积图 | A | 5 h |
| B-1 | 页面骨架 + 状态机 + 假数据 | B | 6 h |
| B-2 | 图2 出生率、死亡率与自然增长率对比 | B | 9 h |
| B-3 | 图3 国际对比与国家选择 | B | 7 h |
| B-4 | KPI 卡与控件 | B | 3 h |
| B-5 | `dev-b.html` + 自检 + 预览图 | B | 4 h |
| B-6 | README 使用章 | B | 1 h |
| C-1 | 契约冻结（30 min） | 共同 | 0.5 h |
| C-2 | CP1 集成 | 共同 | 2 h |
| C-3 | CP2 终检 | 共同 | 1 h |

---

## 3. 契约（Day1 冻结，逐字遵守）

### 3.1 全局命名与加载顺序

项目脚本统一用 `window.Viz` 注册模块。D3 提供 `window.d3`，生成的数据脚本提供 `window.POP_DATA`，供数据访问层读取。

```html
<!-- 最终 index.html 的脚本顺序（B 维护） -->
<script src="vendor/d3.v7.9.0.js"></script>
<script src="data/population.js"></script>    <!-- A：定义 window.POP_DATA -->
<script src="src/data.js"></script>           <!-- A：由 POP_DATA 生成 Viz.data -->
<script src="src/chart-pyramid.js"></script>  <!-- A：定义 Viz.charts.pyramid -->
<script src="src/chart-age-structure.js"></script> <!-- A：定义 Viz.charts.ageStructure -->
<script src="src/chart-trend.js"></script>    <!-- B -->
<script src="src/chart-compare.js"></script>  <!-- B -->
<script src="src/state.js"></script>          <!-- B -->
<script src="src/main.js"></script>           <!-- B：装配与分发 -->
```

### 3.2 数据访问层（A 实现，B 只调）

```js
window.Viz = window.Viz || {};
Viz.data = {
  meta(),                                        // { sources, fetchedAt, years:[1950,2100], ageGroups:[21 个], projectionFrom:2024 }
  pyramid(year, countryCode = 'CHN'),             // { male:[21], female:[21] }，单位：人
  total(year, countryCode = 'CHN'),                // WPP 总人口，单位：人
  share(year, minAge),                             // 中国该年 ≥minAge 岁人口占比（0~1）
  compare(countryCode, minAge, maxAge = Infinity), // { years:[1950..2050], values:[...] }，占比（0~1）
  ageStructure(countryCode = 'CHN'),              // { years:[1950..2100], age0to14, age15to64, age65plus }，各年龄段人数数组
  rates(countryCode = 'CHN'),                     // { yearsRates:[1960..2024], birth, death, natural, yearsPop:[1960..2025], pop }
  latest(),                                       // 中国最新值 { popYear, pop, birthYear, birth, deathYear, death }
  isProjection(year)                              // boolean，仅适用于 WPP 序列
};
```

- 国家代码统一为 `CHN`、`JPN`、`KOR`、`USA`、`IND`。人口数组按年份或年龄组顺序对齐。
- `compare('JPN', 0, 14)` 取 0–14 岁占比，`compare('JPN', 65)` 取 65 岁及以上占比。占比用同一国家、同一年的 WPP 人数和总人口计算。
- `rates` 的 `birth`、`death`、`natural` 单位均为 ‰，`natural = birth - death`，可为负数。`pop` 单位为人。世行缺测值保留为 `null`，出生率或死亡率缺测时，该年的自然增长率也为 `null`。

**逐字段示例**（两人 Day1 各跑一次自测页确认）

```js
Viz.data.meta()          // { years:[1950,2100], ageGroups:['0-4',...,'100+'], projectionFrom:2024 }
Viz.data.pyramid(2023)   // { male:[31_000_000, ...21 个], female:[29_000_000, ...] }
Viz.data.pyramid(2023, 'JPN') // 同样的 21 个年龄组，顺序与中国一致
Viz.data.total(2023)     // 1_422_000_000
Viz.data.share(2023,65)  // 0.148
Viz.data.compare('JPN',65) // { years:[1950,...,2050], values:[0.049,...] }
Viz.data.ageStructure() // { years:[1950,...,2100], age0to14:[...], age15to64:[...], age65plus:[...] }
Viz.data.rates('JPN')   // { yearsRates:[1960,...,2024], birth:[...], death:[...], natural:[...], yearsPop:[1960,...,2025], pop:[...] }
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

`ageStructure`、`trend`、`compare` 使用同样的 `init`、`render`、`resize` 协议。

**约束**
- 图表**不得**直接引用 `Viz.data`，只用 `ctx.data`（这样 B 的假数据页也能跑）。
- 图表**不得** import 或引用任何其他图表/状态文件。
- 图表尺寸必须每次 `render` 时从 `hostEl.getBoundingClientRect()` 读，不缓存死值。

### 3.4 容器与尺寸

| 容器 ID | 高度 | 归属 |
|---|---|---|
| `#chart-pyramid` | 560 px | A，金字塔和右侧折线共用这个容器 |
| `#chart-compare` | 560 px | B |
| `#chart-trend` | 320 px | B |
| `#chart-age-structure` | 360 px | A |

B 在 `index.html` 中保证容器有显式高度。各图自适应宽高，图1内部的金字塔和折线始终左右相邻、逐年龄组对齐。

### 3.5 状态与事件

```js
{
  year: 1950..2100,
  playing: false,
  brushRange: null | [a, b],
  compareCountry: null | 'JPN' | 'KOR' | 'USA' | 'IND',
  compareMetric: '65+' | '0-14',   // 默认 '65+'
  hoverAge: string | null
}
```

| 事件 | 发起方 | 载荷 |
|---|---|---|
| `year` | 控件 / 图2 / 图4 | number |
| `brush` | 图2 | `[start,end]` 或 `null` |
| `country` | 图3 | 非中国国家代码，或 `null`（取消对比） |
| `compareMetric` | 图3 | `'65+'` 或 `'0-14'` |
| `hoverAge` | 图1 | 年龄组或 `null` |
| `play` | 控件 | boolean |

| 触发 | 图1 | 图2 | 图3 | 图4 | KPI |
|---|---|---|---|---|---|
| 拖年份滑块或点图2/图4的年份 | 重绘中国金字塔和当前折线 | 年份线移动 | 年份线移动 | 年份线移动 | 更新 |
| 播放 | 逐帧重绘 | 年份线跟随 | 年份线跟随 | 年份线跟随 | 更新 |
| 框选 | 播放时在选区内循环 | 选区高亮 | 跟随播放年份 | 跟随播放年份 | 跟随播放年份 |
| 选择对比国 | 右侧加入该国同年合计折线 | 加入该国三条人口率折线 | 高亮中国与对比国 | 继续显示中国 | 增加对比国当前指标的占比 |
| 切换 65+ / 0–14 | 保持当前年份与对比对象 | 保持当前年份与对比对象 | 切换显示指标 | 继续显示三个年龄段 | 对比国占比切换为当前指标 |

中国是固定基准。点选一个非中国国家后，另选国家会替换对比对象。再次点当前对比国或点中国会清除对比。选国操作保留年份、播放状态及框选范围。年份超出图2或图3的时间轴时，该图隐藏年份线。

### 3.6 CSS 约定

- B 只定义布局与通用组件类：`.grid`、`.panel`、`.kpi`、`.toolbar`、`.footer` 等。
- A 的图只用 `py-`、`age-` 前缀类或 SVG 属性写样式。图1与图4统一用蓝色表示 0–14 岁、青绿色表示 15–64 岁、橙色表示 65 岁及以上。
- B 的 CSS **不得**选择器命中 `svg` 内部元素或 `py-*`、`age-*`。
- 提示框：各地图自己实现（位置相对 host），不共用 B 的组件。

### 3.7 变更规则

任何字段、事件、容器 ID 的改动：发起方在聊天里说明 → 另一方确认 → 记入附录 A → 改代码。**未确认不得单方面改**。

---

## 4. A 的任务包（数据、金字塔与年龄结构）

> 全程可在 `dev/dev-a.html` 中自测，不需要 B 的任何代码。

### A-1 抓取五国数据 ｜ 4 h ｜ 前置：无

**要做**
- `scripts/fetch_data.mjs` 下载 WPP 2024 CSV（29,948,947 B）到 `data/raw/`。按中国、日本、韩国、美国、印度抓取世行总人口（1960–2025）及出生率、死亡率（1960–2024），处理分页并按指标汇总到 3 个 `data/raw/wb_*.json` 文件。
- 重试 5 次；文件已存在且大小一致则跳过
- `data/raw/fetch-meta.json`：URL、UTC 时间、字节数、sha256
- 脚本内把进程级 `TMP`/`TEMP` 指向 `tmp/temp`

**产出**：`scripts/fetch_data.mjs`、`data/raw/wpp2024_pop5_medium.csv.gz`、`data/raw/wb_*.json`、`data/raw/fetch-meta.json`

**验收**
- [ ] 重跑不重复下载
- [ ] 断网时错误信息明确，提示手动放 CSV 的位置
- [ ] `fetch-meta.json` 的 4 项来源、sha256 齐全，世行数据包含约定的 5 个国家代码
- [ ] `%TEMP%` 下无本次新增

### A-2 构建数据 + 数据 API ｜ 8 h ｜ 前置：A-1

**要做**
- `scripts/build_data.mjs`：`zlib.createGunzip()` + `readline` **流式**读（解压后约 200–300 MB）
- 手写 CSV 行解析（BOM、引号内逗号）。提取 `LocID ∈ {156,392,410,840,356}` 的 1950–2023 年估算和 2024–2100 年 `Medium` 预测。
- 将千人换算为人（×1000 取整），按国家、年份保留 21 组男女分列人数。由同一批年龄组求和得到总人口、三个大年龄段人数和占比。
- 按国家和年份合并世行指标，通过 `birth - death` 得到自然增长率。缺测值和单位沿用第 3.2 节约定。
- 输出 `data/population.js`（`window.POP_DATA`）和 `data/mock.js`。自测数据按同一结构用少量年份锚点生成连续序列，足以演示选国、播放和面积图。
- `docs/data-schema.md` 写清字段、单位、来源
- `src/data.js`：由 `POP_DATA` 组装 `Viz.data`（严格按 3.2 节签名）

**产出**：`scripts/build_data.mjs`、`data/population.js`、`data/mock.js`、`src/data.js`、`docs/data-schema.md`

**验收**
- [ ] `population.js` < 1 MB，五国各有 151 个连续年份，每年 21 个年龄组
- [ ] `Viz.data` 每个方法返回值与 3.2 节示例形状一致
- [ ] `mock.js` 结构与真实数据完全一致

### A-3 数据校验 ｜ 3 h ｜ 前置：A-2

**要做**：`tools/verify_data.mjs` 断言
- 五国每年 `Σ(male + female)` 与 WPP 总人口误差小于 1%，且各年龄组男女之和与右侧折线的口径一致
- 五国 1950–2100 年连续且不重复，每年 21 个年龄组。三个大年龄段覆盖所有年龄组，人数之和等于分组人口总数
- WPP 2023 中国总人口约 14.2 亿（±3%）；世行 2025 = 1,406,585,000
- 世行年份索引连续，保留来源中的缺测值。逐年验证自然增长率等于出生率减死亡率，任一指标缺测时结果为 `null`
- 中国 2022 年死亡率 7.37‰、出生率 6.77‰、自然增长率约 −0.60‰。2024 年出生率由 6.39‰ 回升到 6.77‰
- 数值项均为有限数。人口、出生率和死亡率非负，自然增长率允许为负

**产出**：`tools/verify_data.mjs`

**验收**：全绿；故意改坏一个数能报错

### A-4 图1 金字塔与年龄人口合计折线 ｜ 9 h ｜ 前置：A-2

**要做**
- `src/chart-pyramid.js`：男左女右、21 组、左右对称比例尺
- hover 左右同时高亮 + tooltip（人数、占比）
- 2024+ 柱体降透明度 + "预测"角标
- 只通过 `ctx.data` 取数，`init/render/resize` 三件套

**右侧折线**

- 在原金字塔右侧加一条与原年龄纵轴平行的零基线，两部分画在同一个 SVG 中。右侧折线按横轴为年龄组、纵轴为人数的方式构造，再旋转显示，使年龄组沿新基线竖直排列；每个点与金字塔的同龄条形在同一水平线上。
- 折线点的横向位移对应同龄男性人数与女性人数之和。右侧用独立的线性人数比例尺缩放，刻度以万人显示。金字塔保留原来的左右对称比例尺。对比两国时，右侧两条折线共用同一把人数尺，播放期间保持比例尺稳定。
- 将中国合计折线与零基线之间的区域沿年龄分为 0–14 岁、15–64 岁、65 岁及以上三段，分界线位于相邻年龄组之间。三区分别填蓝色、青绿色、橙色，并配图例。
- 选择对比国后，右侧同一坐标系叠加该国同年、同年龄组的男女合计折线。图例用实线标识中国、虚线标识对比国，三色填充对应中国曲线。2024 年起两国曲线随图1统一显示“预测”标识。
- 悬停任一年龄组时，同步高亮该组的男女条形和右侧同龄点。提示框显示中国男女分项及合计，选国时一并显示该国合计。

**验收**
- [ ] 1950 正三角、2023 收缩、2050 倒钟形，肉眼可辨
- [ ] 右侧 21 个折线点逐组和金字塔对齐，人数为该年龄组男女合计，三段填色边界准确
- [ ] 选国、换国、取消后，右侧曲线和图例对应正确。两国使用同一年份和统一人数刻度
- [ ] 悬停时原条形和同年龄折线点同步高亮，数值与 `verify_data` 抽查一致
- [ ] 容器宽度变化后 `resize()` 正确

### A-5 图1 交互 + 自测页 ｜ 4 h ｜ 前置：A-4、A-7

**要做**
- 年份滑块 1950–2100 + 播放/暂停（约 80 ms/年，`requestAnimationFrame`，每帧最多 1 年）
- 快捷年份：1950 / 1980 / 2000 / 2023 / 2050
- 响应 `brush`：在框选区间内循环播放
- `dev/dev-a.html` 加载 D3、自测数据、数据访问层及图1和图4。自测页提供年份滑块、模拟框选和对比国选择，验证双图联动。

**验收**
- [ ] `dev-a.html` 单独打开即可演示金字塔、右侧折线、年龄结构图及选国后的变化
- [ ] 30 fps 以上；播放/拖动互不打架
- [ ] 框选区间播放只在该区间循环

### A-6 README 数据章 ｜ 1 h ｜ 前置：A-3

写清五国数据来源、覆盖年份、年龄段聚合方法、自然增长率和缺测值口径。WPP 估算止于 2023，世行总人口与出生死亡率分别取到 2025 年和 2024 年。

### A-7 图4 年龄结构堆叠面积图 ｜ 5 h ｜ 前置：A-2

**要做**

- `src/chart-age-structure.js` 注册 `Viz.charts.ageStructure`。横轴为 1950–2100 年，纵轴为占总人口比例（0%–100%）。从下到上依次堆叠 0–14 岁、15–64 岁、65 岁及以上三个年龄段，配色与图1一致。
- 用 `ctx.data.ageStructure()` 的中国各年龄段人数计算当年占比。悬停显示该年三个年龄段人数及占比。年份线跟随滑块和播放，点击图中的年份可发出 `year` 事件。
- 2024 年起以边界线和“预测”文字标识，并降低预测区填充透明度。选国时图4继续显示中国的年龄结构。

**验收**

- [ ] 三个年龄段覆盖 1950–2100 年，每年合计 100%，提示框中的人数、占比可与图1数据核对
- [ ] 预测边界、年份高亮和图1联动正确

---

## 5. B 的任务包（页面与折线）

> 全程可在 `dev/dev-b.html` 中用 `data/b-fixture.js` 自测，不需要 A 的任何代码。

### B-1 页面骨架 + 状态机 + 假数据 ｜ 6 h ｜ 前置：无

**要做**
- `index.html` 保留顶部标题、KPI 和控件。第一行放 `#chart-pyramid`（7 列）及 `#chart-compare`（5 列），第二行放 `#chart-trend`，第三行放 `#chart-age-structure`。后两图各占整行，窄屏时第一行也改为单列。页脚列出两处数据来源，脚本顺序按第 3.1 节。
- `src/style.css` 负责布局与通用组件，样式边界按第 3.6 节。
- `src/state.js`：唯一状态源 + 事件总线（`on`/`emit`），字段见 3.5 节
- `src/main.js`：遍历 `Viz.charts` 调 `init(host, ctx)`，状态变化时调 `render(state)`，窗口变化时调 `resize()`
- `data/b-fixture.js` 按第 3.2 节提供五国的假数据和临时 `Viz.data`。覆盖 151 年 × 21 组、负自然增长率及对比国个别年份缺测。
- `dev/dev-b.html` 加载 D3、假数据、图2、图3和状态机。用简易的图1、图4占位显示收到的年份及对比国，验证布局和事件。

**产出**：`index.html`、`src/style.css`、`src/state.js`、`src/main.js`、`data/b-fixture.js`、`dev/dev-b.html`

**验收**
- [ ] `dev-b.html` 单独打开，图2、图3可联动，两处占位能收到状态
- [ ] 集成后双击 `index.html`，四图加载无报错
- [ ] 窗口缩放不错位

### B-2 图2 出生率、死亡率与自然增长率对比 ｜ 9 h ｜ 前置：B-1

**要做**
- `src/chart-trend.js` 默认绘制中国出生率、死亡率和自然增长率三条线。横轴为 1960–2024 年，三项共用 ‰ 纵轴。纵轴覆盖负值，并标出 0‰ 基线。
- 保留中国的 1963 高峰、1987 峰值、2016 二孩、**2022 死亡率高于出生率**、**2024 龙年回升**标注。标注位置沿用中国序列。
- 图3选国后叠加该国的三条线，与中国使用同一年轴和 ‰ 纵轴。颜色区分指标，实线表示中国，虚线表示对比国。图例同时说明指标和国家，悬停按国家列出同年三项指标。缺测时折线留空，提示框显示“—”。
- 保留 `d3.brushX` 与 `brush` 事件、全局年份十字线和点击选年。右侧副轴可切换显示中国总人口（1960–2025）；开启时横轴可延至 2025 年，框选播放区间仍限于 1960–2024 年。
- 图注写明出生率、死亡率来自世行，自然增长率为两者之差。

**验收**
- [ ] 中国三条线齐全，2022 年交叉标注正确（7.37‰ 与 6.77‰），自然增长率显示约 −0.60‰
- [ ] 选国显示两国共六条序列，换国和取消时曲线、图例、提示框同步更新，缺测年份正确留空
- [ ] 框选触发 `brush`，取消发 `null`
- [ ] 悬停数值与对应国家的 `ctx.data.rates(countryCode)` 一致

### B-3 图3 国际对比与国家选择 ｜ 7 h ｜ 前置：B-1

**要做**
- `src/chart-compare.js` 展示五国 1950–2050 年的 65 岁及以上占比，中国加粗。悬停显示各国占比。
- 点选非中国国家时发出 `country` 事件，同时高亮中国和该国并淡化另外三国。国家图例标示当前对比对象。点选、替换、清除的行为按第 3.5 节。
- 用 `compareMetric` 事件切换 65 岁及以上或 0–14 岁占比。
- 与全局年份十字线同步

**验收**
- [ ] 两个指标均有五国序列，2024 年起的预测段为虚线
- [ ] 选日本、换韩国、取消对比时，图1、图2、图3与 KPI 同步更新，年份和选区保留

### B-4 KPI 卡与控件 ｜ 3 h ｜ 前置：B-1

保留中国总人口（世行取至 2025 年）、65 岁及以上与 0–14 岁人口占比（WPP）、出生率和死亡率（世行取至 2024 年），以及年份滑块和播放按钮。选国后用 WPP 同年数据补充对比国在图3所选年龄段的占比，显示国家名和年份。

**验收**
- [ ] KPI 随年份更新，世行指标超出覆盖范围或缺测时显示“—”
- [ ] WPP 预测年份有文字标识，选国和切换年龄段后对比国占比正确

### B-5 自测页收尾 + 交互自检 + 预览图 ｜ 4 h ｜ 前置：B-2、B-3

**要做**
- 完善 `dev/dev-b.html`（给 A 也留一个可打开的空壳说明）
- `tools/smoke.mjs` 用 jsdom 加载整页（`runScripts:'dangerously'`、`resources:'usable'`、`pretendToBeVisual:true`），在 `beforeParse` 设 `window.__TEST__=true` 关动画。验证图1有 42 根柱和 21 点合计折线，图2默认三条指标序列，图3有五国序列，图4有三个面积层，KPI 有值。
- 模拟改年份、框选与清除、选择和切换对比国，验证图1右侧出现对比曲线、图2显示对应国家的六条序列、取消后恢复中国序列。
- `tools/render_figures.mjs`：jsdom → SVG → `@resvg/resvg-js` 出 `images/*.png`
- 两个工具用 `createRequire` 只读引用 `作业4\d3-quickstart\node_modules`

**验收**
- [ ] `smoke.mjs` 全绿
- [ ] 四张 PNG 中文齐全

### B-6 README 使用章 ｜ 1 h ｜ 前置：B-5

写"运行方式""交互说明""自检方法"三节。

---

## 6. 检查点与集成

### 6.1 CP1 集成（Day4 上午，约 2 h）

| 步骤 | 操作 | 谁 |
|---|---|---|
| 1 | 交付 `data/population.js`、`src/data.js`、`src/chart-pyramid.js`、`src/chart-age-structure.js`、`docs/data-schema.md` | A |
| 2 | 在 `index.html` 确认 4 个图表容器，按第 3.1 节接入 A 的数据和图表脚本，生产页使用真实数据 | B |
| 3 | `node tools/verify_data.mjs`、`node tools/smoke.mjs` | 双方 |
| 4 | 各自按第 8 节清单抽查 15 分钟，至少走一次选国、换国、取消和框选播放 | 双方 |
| 5 | 问题分类：数据字段 → A；事件/布局 → B；当天闭环 | 分工修 |

**集成失败预案**：若 2 小时内未跑通，B 继续用 `dev/dev-b.html` 的图1、图4占位调试联动，A 在 `dev/dev-a.html` 排查两图与数据访问层。双方修正后再接入真实图表。

### 6.2 CP2 终检（Day5，约 1 h）

- 双方按第 8 节清单互查
- 从零复现一次：`fetch → build → verify → 双击打开`
- 确认交付物清单与"不含 docx"

---

## 7. 数据与技术要点（两人都要知道）

### 7.1 数据来源与时效

| 数据 | 来源 | 覆盖 |
|---|---|---|
| 5 岁组人口 × 性别 | UN WPP 2024 `WPP2024_PopulationByAge5GroupSex_Medium.csv.gz`（29.9 MB） | 五国，1950–2100 年 |
| 总人口 | 世行 `SP.POP.TOTL` | 五国，1960–2025 年 |
| 出生率、死亡率 | 世行 `SP.DYN.CBRT.IN`、`SP.DYN.CDRT.IN` | 五国，1960–2024 年 |
| 自然增长率 | 同一国家、同一年世行出生率减死亡率 | 两项世行指标均有值的年份 |
| 降级备用 | OWID `population-by-age-group.csv`（5 大年龄段） | 1950–2100 年，可用于核对大年龄段趋势 |

项目固定使用 WPP 2024，1950–2023 年为估算，2024–2100 年采用中方案预测。世行总人口与出生死亡率各自采用上表的历史序列范围。

**中国数据校验参照**：2025 年总人口 1,406,585,000。2024 年出生率 6.77‰、死亡率 7.76‰。2022 年死亡率 7.37‰，高于出生率 6.77‰，自然增长率约 −0.60‰。

### 7.2 口径限制（写进页面与 README）

1. 图1、图3、图4和基于 WPP 的 KPI 标记 2024 年起的预测数据。图2的 2024 年世行数据按历史序列显示。
2. WPP 的 "China" 不含港澳台。
3. 金字塔、年龄占比和面积图采用同一版本的 WPP 人口数据。出生、死亡、自然增长率和副轴总人口采用世行口径，各自标明来源。
4. WPP 的 21 个年龄组从 0–4 岁延续到 100 岁及以上。大年龄段由原始五岁组汇总，依次为 0–14 岁、15–64 岁、65 岁及以上。
5. WPP 批量下载使用原始 CSV。OWID 备用数据可用于核对大年龄段趋势，图1的年龄和性别明细仍需 WPP 文件。

### 7.3 下载与落盘规范（不污染全局环境）

| 动作 | 位置 | 大小 |
|---|---|---|
| WPP CSV | `data\raw\wpp2024_pop5_medium.csv.gz` | 29.9 MB |
| 世行数据 | `data\raw\wb_*.json` | 随五国记录数变化 |
| 元信息 | `data\raw\fetch-meta.json` | < 1 KB |
| 构建产物 | `data\population.js` | < 1 MB |
| 临时文件 | `tmp\`（`TMP`/`TEMP` 指向 `tmp\temp`） | 可变 |
| d3 | `vendor\d3.v7.9.0.js`（从 `作业4` 复制，非下载） | 587 KB |

- 不在 `Downloads`、桌面、`%USERPROFILE%`、`AppData` 落文件
- 不跑 `npm install` / `npm i -g` / `pip install`
- 自检工具用 `createRequire` 只读引用 `作业4\d3-quickstart\node_modules`，缺失则跳过预览
- `TMP`/`TEMP` 只对进程生效

### 7.4 四图标注要点

| 图 | 必须出现 |
|---|---|
| 图1 | 1950 正三角、2023 收缩、2050 倒钟形。右侧合计折线、三个年龄段、两国图例与 2024 年起的预测标识 |
| 图2 | 1963 高峰、1987 峰值、2016 二孩、2022 死亡率高于出生率、2024 龙年回升。显示自然增长率和 0‰ 基线，区分指标与国家 |
| 图3 | 五国对比、2024 年起预测虚线、选国后的高亮 |
| 图4 | 三个年龄段堆叠到 100%，标出 2024 年估算与预测的分界 |

---

## 8. 验收标准

### 8.1 项目级

- [ ] 双击 `index.html` 可运行，四图渲染，无报错
- [ ] 共享年份、播放与框选联动正确。选国、换国、取消时图1与图2同步响应
- [ ] 预测年份有标识，页脚有来源与截止年份
- [ ] `verify_data` / `smoke` 全绿
- [ ] 交付物不含 docx

### 8.2 A 的验收

- [ ] `dev/dev-a.html` 单独可完整演示图1和图4（滑块、播放、悬停、模拟框选和选国）
- [ ] `Viz.data` 全部方法符合 3.2 节签名与示例
- [ ] `verify_data` 全绿
- [ ] 图1 三个代表年份形态可辨，预测有标识。右侧折线逐龄对齐，三段填色和两国数据可核对
- [ ] 图4 年份线同步，三段人口占比在每个年份合计 100%
- [ ] 数据章节数字与脚本输出一致

### 8.3 B 的验收

- [ ] `dev/dev-b.html` 单独可运行（假数据）
- [ ] 图2 标注正确，三条指标和对比国曲线可核对，`brush` 事件载荷正确
- [ ] 图3 五国序列齐全，可切换指标。选国同步更新图1、图2和 KPI
- [ ] KPI 随年份更新、预测有标识
- [ ] `smoke` 全绿，四张 PNG 中文齐全
- [ ] 使用章节完整

---

## 9. 风险与应对

| 风险 | 影响 | 应对 | 触发人 |
|---|---|---|---|
| 契约理解偏差 | CP1 返工 | 3.2 节逐字段示例 + 各自自测页 Day1 验证 | 双方 |
| WPP 30 MB 下载不稳 | 阻塞 A 下游 | 重试 5 次并校验 sha256。OWID 只用于核对大年龄段趋势 | A |
| A 的图在 B 页面里尺寸不对 | 返工 | 约定容器显式高度，A 每次 render 读 host 尺寸 | A/B |
| 两套口径被误读 | 结论可信度 | 页面图注 + README 明确不混算 | A |
| 播放掉帧 | 体验差 | 每帧最多 1 年 + 测试模式关动画 | A/B |
| 对比国世行指标缺测 | 折线部分年份留空 | 保留 `null`，图上留空并在提示框注明缺测 | A/B |
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
│  ├─ chart-pyramid.js  chart-age-structure.js    # A
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
| 2026-11-25 | v4：图1增加右侧分段合计折线，A 增加年龄结构面积图，图2增加自然增长率，图3选国联动图1和图2 | 本次需求调整 |

> 接口变更也记这里：日期 / 改了哪个字段 / 谁提出 / 谁确认。

---

## 附录 B · 任务勾选表（可打印）

| 编号 | 任务 | 负责人 | 工时 | 完成 |
|---|---|---|---|---|
| A-1 | 抓取五国数据 | A | 4 h | ☐ |
| A-2 | 构建数据 + 数据 API | A | 8 h | ☐ |
| A-3 | 数据校验 | A | 3 h | ☐ |
| A-4 | 图1 金字塔与年龄人口合计折线 | A | 9 h | ☐ |
| A-5 | 图1 交互 + 自测页 | A | 4 h | ☐ |
| A-6 | README 数据章 | A | 1 h | ☐ |
| A-7 | 图4 年龄结构堆叠面积图 | A | 5 h | ☐ |
| B-1 | 页面骨架 + 状态机 + 假数据 | B | 6 h | ☐ |
| B-2 | 图2 出生率、死亡率与自然增长率对比 | B | 9 h | ☐ |
| B-3 | 图3 国际对比与国家选择 | B | 7 h | ☐ |
| B-4 | KPI 卡与控件 | B | 3 h | ☐ |
| B-5 | `dev-b.html` + 自检 + 预览图 | B | 4 h | ☐ |
| B-6 | README 使用章 | B | 1 h | ☐ |
| C-1 | 契约冻结 | 共同 | 0.5 h | ☐ |
| C-2 | CP1 集成 | 共同 | 2 h | ☐ |
| C-3 | CP2 终检 | 共同 | 1 h | ☐ |
