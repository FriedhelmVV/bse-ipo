# 北交所打新工作台 — 数据契约 SCHEMA v1

本文件是**唯一契约**。取数（`tools/fetch.cjs`）、构建（`build/build-seed.cjs`）、
页面（`北交所打新工作台.html`）三方都以本文件为准，任何字段改动必须同步改这里。

数据来源：同花顺 iFinD（`C:/Users/p/.agents/skills/ifind-finance-data/call-node.js`）。
覆盖度实测见 `docs/PROBE-FINDINGS.md`。

---

## 一、三层数据流

```
iFinD MCP
  │  tools/fetch.cjs         （增量、限速 3s、原始缓存）
  ▼
data/raw/universe.json       —— 股票池
data/raw/<code>.json         —— 每只股票的原始字段（逐字段缓存，已取到的不重取）
  │  build/build-seed.cjs    （纯计算，无网络）
  ▼
data/ipo.json / data/ipo.js  —— 成品记录数组
data/manifest.js             —— 元信息
  │  build/inject-seed.cjs
  ▼
北交所打新工作台.html         —— <script id="bse-seed"> 内联同一份数据
```

页面加载顺序：先读内联 seed（离线可用），再尝试加载 `data/manifest.js`；
若外部数据更新（`generatedAt` 更晚）则覆盖内联数据。**零外部依赖，不引 CDN。**

> ⚠️ 内联 seed 必须是 `data/ipo.json` 的**整对象序列化**（`JSON.stringify(ipo)`），
> 不许手工挑字段。2026-10-13 踩过：`inject-seed.cjs` 曾用 `{schemaVersion, generatedAt, today, counts, records}`
> 白名单，静默丢掉 `source`，页面 `source=undefined` 而 `data/ipo.json` 有值。
> `build/verify.cjs` 现在会逐个比对顶层键，缺键/多键/值不同都报错。

---

## 二、`data/raw/universe.json`

```jsonc
{
  "generatedAt": "2026-10-13T20:10:00+08:00",
  "source": "iFinD search_stocks",
  "years": [2025, 2026],
  "unlistedFetchedOn": "20261013",
  "items": [
    { "code": "920086.BJ", "name": "科马材料", "listingDate": "2026-01-16", "subscribeDate": "2026-01-06" },
    { "code": "920285.BJ", "name": "远大健科", "listingDate": null,         "subscribeDate": "2026-10-14" }
  ]
}
```
- `code`：`920xxx.BJ` 形式的北交所代码
- `listingDate`：`YYYY-MM-DD`，来自「新股上市日期」；未定为 `null`
- `subscribeDate`：`YYYY-MM-DD`，来自「申购日期」；**只有第二路问法会给**，按年问法为 `null`
- `unlistedFetchedOn`：`YYYYMMDD`，第二路问法最后一次执行的**市场日**（每个更新周期跑一次即可；键必须是市场日，不能用自然日，见 §二·补）

**股票池必须三路合并**（`tools/fetch.cjs`）：

| 路 | 来源 | 覆盖 |
| --- | --- | --- |
| ① 按年已上市 | `search_stocks`：`<年>年北交所上市的新股，按上市日期排序` | 已上市新股（每年一次调用） |
| ② 待上市 / 即将申购 | `search_stocks`：`北交所股票，上市日期为空，申购日期不为空` | 已申购未上市 + 即将申购 |
| ③ 手工关注清单 | `data/watch.json`（不消耗 iFinD 调用） | 已公告但 **iFinD 选股索引还没收录**的新股 |

⚠ 只用①时「即将申购 / 已申购待上市」永远是空的 —— 而那正是打新工作台的主打内容。
实测①漏掉 `920196.BJ`/`920296.BJ`/`920285.BJ` 三只，②恰好精确返回这三只。

### 二·补 `data/watch.json`（第三路，必须有）

2026-10-16 实测：交易所当天公告的 `920306.BJ` 天元重工（现金申购发行日期 2026-10-20、
发行价 4.48、发行市盈率 14.93、网上申购上限 181.9 万股）在 iFinD **选股索引**里完全查不到 ——
`北交所股票，股票代码大于920300` 只回 `920300.BJ` 辰光医疗，最宽的 `北交所股票，上市日期为空`
也只有 2 只；`search_stocks` 用简称、全称、代码问一律「抱歉，未找到符合要求的证券」。
但**按代码单查 `get_stock_info` / `get_stock_events` 字段齐全**。

结论：**索引滞后 ≠ 数据没有**，知道代码就能取全 → 兜底通道只能是「人工登记代码」。

```jsonc
{
  "items": [
    { "code": "920306.BJ", "name": "天元重工", "subscribeDate": "2026-10-20" }
  ]
}
```
- `code` 必填，`^\d{6}\.BJ$`；非法项静默跳过，文件缺失/损坏当空清单（**不能因为一个手写文件把整轮取数炸掉**）
- `name` / `subscribeDate` 可省；`subscribeDate` 在池子里还没有它时用来填申购日期
- 合并优先级：池子里的 `listingDate` / `subscribeDate` 更权威，关注清单只做「补进来」
- **这个文件有两个写入者**：① 人工登记（iFinD 索引滞后时）；② `tools/probe-new.cjs` 的**白天新股探测**自动追加（发现「已公告未上市」池里出现池子外的代码就写进来，让随后那一轮 fetch 能并入，见 README §六.20）。自动追加只补 `code`/`name`/`subscribeDate`，已存在的条目不会被覆盖，`_comment` 原样保留。
- 年份未知 → `codeYears` 记 `null`，任何 `--years` 过滤都保留它（否则未上市新股会被年份挡掉）
- 加完条目要跑一次 `node tools\fetch.cjs`（**别加 `--codes`**，那会跳过池子合并）
- `build/verify.cjs` 第 `[12]` 段断言「关注清单里的每一只都已并进 `universe.json`」

## 三、`data/raw/<code>.json`

字段一律「取到才写」，取不到写 `null`。`fetchedAt` 记录该字段最后一次成功抓取时间，
构建时用它判断是否值得重取（未上市的新股在上市后需要重新抓中签类字段）。

```jsonc
{
  "code": "920086.BJ",
  "name": "科马材料",
  "fetchedAt": "2026-10-13T20:10:00+08:00",
  "universe": { "listingDate": "2026-01-16", "subscribeDate": "2026-01-06" },
  "events": {
    "subscribeDate":      "2026-01-06",   // 现金申购发行日期
    "issuePrice":         11.66,          // 首发价格（元）
    "peRatio":            12.34,          // 首发市盈率(摊薄)
    "subscribeCapShares": 1000000,        // 网上申购数量上限（股）
    "onlineIssueShares":  18828000,       // 网上发行数量（股）
    "winRate":            0.0257,         // 网上发行中签率（%）
    "validSubscribers":   624494,         // 网上发行有效申购户数（户）
    "winners":            81904           // 网上获配户数（户）
  },
  "profile": {
    "industry": "汽车",                    // 申万一级行业
    "listingDate": "2026-01-16"
  },
  "perf": {
    "days": [
      { "date": "2026-01-16", "open": 50.0, "high": 79.93, "low": 49.0,
        "close": 54.95, "chgPct": 371.2693, "turnover": 96.1425,
        "amount": 987610000, "volume": 18101709, "floatMktCap": 1034600000 }
    ]
  }
}
```
- 数组/对象缺失的**子字段**用 `null` 占位，不要用 `""` 或 `"\t"`。
- `perf.days` 只保存「上市日起 ≤ 5 个自然交易日」的日线，用于算首日与 T+1。
- `volume` 单位**股**（iFinD 的「成交量」指标，返回值形如 `1810.1709万` 需按中文量纲解析）；`floatMktCap` 单位元（「流通市值」指标）。二者取不到写 `null`。
- **iFinD 日线是倒序返回的**（最新日期在前），入库前必须按 `date` 升序排。
- 停牌/非交易日的行只有 `close`，其余为 `null`（iFinD 原文是制表符 `\t`）——不要当成 0。

### 三·补 取数返回信封（写 fetch 必读）

`call('stock', tool, params)` → 外层 MCP 信封 `res.data.result.content[0].text` 是**一个 JSON 字符串**；
parse 后 `{ code:1, msg:'success', data:{ answer:'<markdown 表格>', indicators_params:{...} } }`。
**正文永远在 `data.answer`（markdown 表格），不是 `data.result.content`。** 表头会把单位写进列名
（`首发价格（单位：元）`、`换手率（单位：%）`），数值可能是 `1882.8万` / `9.8761亿` 中文量纲。
成功判定：外层 `res.ok !== false` 且内层 `code === 1`。细节见 `docs/PROBE-FINDINGS.md` §7/§8。

## 四、成品：`window.BSE_IPO`（= `data/ipo.json`）

```jsonc
{
  "schemaVersion": 1,
  "generatedAt": "2026-10-13T20:20:00+08:00",
  "source": "iFinD",
  "counts": { "total": 64, "listed": 60, "upcoming": 4 },
  "records": [ /* BseIpoRecord，按 subscribeDate 升序，最新在最后 */ ]
}
```

### BseIpoRecord

| 字段 | 类型 | 单位 | 来源/算法 | 缺失时 |
|---|---|---|---|---|
| `code` | string | — | universe | 必填 |
| `name` | string | — | universe | 必填 |
| `industry` | string\|null | — | raw.profile.industry | null |
| `subscribeDate` | `YYYY-MM-DD`\|null | — | raw.events.subscribeDate | null |
| `listingDate` | `YYYY-MM-DD`\|null | — | raw.profile.listingDate ?? universe.listingDate | null |
| `status` | `'upcoming'\|'subscribed'\|'listed'` | — | 见下 | 必填 |
| `listingToday` | boolean | — | `status==='listed' && listingDate === today`，见下 | 必填 |
| `subscribeToday` | boolean | — | `subscribeDate === today`，见下「四个时点」 | 必填 |
| `subscribeTomorrow` | boolean | — | `subscribeDate === today+1`，见下「四个时点」 | 必填 |
| `listingTomorrow` | boolean | — | `listingDate === today+1`，见下「四个时点」 | 必填 |
| `issuePrice` | number\|null | 元 | raw.events.issuePrice | null |
| `peRatio` | number\|null | 倍 | raw.events.peRatio | null |
| `subscribeCapShares` | number\|null | 股 | raw.events.subscribeCapShares | null |
| `onlineIssueShares` | number\|null | 股 | raw.events.onlineIssueShares | null |
| `onlineIssueWan` | number\|null | 万股 | `onlineIssueShares/1e4` | null |
| `winRate` | number\|null | % | raw.events.winRate | null |
| `validSubscribers` | number\|null | 户 | raw.events.validSubscribers | null |
| `winners` | number\|null | 户 | raw.events.winners | null |
| `maxFundWan` | number\|null | 万元 | **`subscribeCapShares*issuePrice/1e4`** | null |
| `fund100Wan` | number\|null | 万元 | **`100*issuePrice/(winRate/100)/1e4`** | null |
| `maxWinShares` | number\|null | 股 | `subscribeCapShares*winRate/100`，向下取整 | null |
| `shatterFundWan` | number\|null | 万元 | 碎股门槛估算，见下方 §五 | null |
| `totalFundYi` | number\|null | 亿元 | `(onlineIssueShares/(winRate/100))*issuePrice/1e8` | null |
| `avgWinPerHouse` | number\|null | 股 | `onlineIssueShares/winners` | null |
| `firstDay` | object\|null | — | 见下 | null |
| `t1ChgPct` | number\|null | % | 首日之后**第一个有涨跌幅的交易日**（跳过周末/停牌行） | null |
| `floatMktCapYi` | number\|null | 亿元 | 优先 `raw.perf.days[0].floatMktCap/1e8`；无则 `firstDay.close*onlineIssueShares/1e8`（口径：网上发行股本） | null |
| `plan` | object | — | 发行方案参数，供页面**实时**算「打新门槛测算 / 收益率弹性测算」，见 §五·补 | 必填 |
| `derived` | string[] | — | 本记录中由算法算出的字段名列表 | `[]` |
| `missing` | string[] | — | 本记录缺失字段名列表 | `[]` |

`status` 判定（三分法，`today` 取 Asia/Shanghai 当日）：
- `upcoming`：`subscribeDate` 为空，或 `subscribeDate >= 今天`（含"明天申购"和**"就是今天申购"**）
- `subscribed`：`subscribeDate < 今天` 且（`listingDate` 为空 **或** `listingDate > 今天`）
- `listed`：`listingDate <= 今天`

⚠ **申购日当天算 `upcoming`，不算 `subscribed`**：网上申购当天 9:15–15:00 才真正下单，写成
`subscribed` 会让工作台在当天显示「已申购，等待上市公告」，把当天最该看的那只标的说反。
页面 hero 里 `d===0 ? '就是今天申购'` 这个分支就是按这个语义写的（早期规则下它取不到）。
实测：`today=2026-10-14` 且 `920285.BJ` 远大健科 `subscribeDate=2026-10-14` → `upcoming`。

### `listingToday` —— 「当天上市」（`status` 仍是 `listed`，但详情不删）

`listingToday: boolean`，`true` 当且仅当 `status === 'listed' && listingDate === 今天`。
它是 `listed` 的**子类**，不新增枚举值：`counts`、`打新数据` Tab、`首日表现` Tab 全部照旧把它当已上市。

它存在的唯一理由：**上市当天，工作台仍要为这只标的给出和「已申购未上市」一模一样的完整分析**
（发行方案摘要 + 门槛测算 / 弹性测算 / 可比公司估值 / 公司基本情况 / 财务趋势 五张卡）。
次日起 `listingDate < 今天`，`listingToday` 变回 `false`，它自动退出申购清单，无需额外清理。

联动三处（`build/verify.cjs` 会逐条断言，缺一处就报错）：

| 位置 | 规则 |
|---|---|
| `build/build-seed.cjs` | `buildFund(fr, status, peRatio, listingToday)`：`status === 'listed' && !listingToday` 才返回 `null` |
| `tools/fetch.cjs` | fund 通道目标过滤是 `!it.listingDate \|\| it.listingDate >= todayDash`（**`>=`**，当天上市的也要取） |
| `北交所打新工作台.html` | `isListingToday(r)` / `inWorkbench(r)`；`ipocBodyHTML` 与 `detailCardsHTML` 的守卫是 `statusOf(r)==='listed' && !isListingToday(r)`；状态文案 `statusTxtOf(r)` 给「今日上市」 |

⚠ 判定一律读种子里的 `r.listingToday`，**不要在页面里用浏览器当天时间重算** —— 页面在种子重建
（每天 08:00）之前打开时，浏览器日期会和种子生成日差一天，两边会打架。

⚠ 不能只看"`listingDate` 是否非空"：已公告上市日但日期在未来的新股（实测 `920162.BJ` 莫森泰克
`listingDate=2026-10-15`，而 `today=2026-10-13`）此刻**还没有首日行情**，必须算 `subscribed`，
否则会凭空多出一条 `firstDay=null` 的"已上市"记录，也会把「已申购待上市」这个工作台主打分组清空。
`build/verify.cjs` 会按上面的规则逐条复算并断言。

### 「四个时点」高亮 —— `subscribeToday` / `subscribeTomorrow` / `listingToday` / `listingTomorrow`

四个布尔字段，全部由 `build/build-seed.cjs` 在**种子生成日**算好写进记录（`addDaysYmd(today,1)` 得到「明天」）：

| 字段 | 判据 | 页面文案 |
|---|---|---|
| `subscribeToday` | `subscribeDate === today` | 今日申购 |
| `subscribeTomorrow` | `subscribeDate === today+1` | 明日申购 |
| `listingToday` | `status==='listed' && listingDate === today` | 今日上市 |
| `listingTomorrow` | `listingDate === today+1` | 明日上市 |

页面用法（`北交所打新工作台.html`）：
- `hlOf(r)` 按优先级 **今日上市 > 明日上市 > 今日申购 > 明日申购** 返回命中的键（或 null）；
- `statusTxtOf(r)` = 命中文案 ?? 三分法文案；`statusSpan(r)` 在此基础上叠加 `.status.hl` 金色强调类；
- 申购清单整卡加 `.ipoc.hl`（描金边 + 序号块点亮），「打新数据 / 资金档位分布」表格行加 `tr.hl`，
  简称下方的 `.pill` 直接显示命中文案（`statusPill(st, hlKey)`）；
- `headTxt(r)` 优先输出「就是今天申购 / 明天申购 / 明天上市 · 上市日 …」。

⚠ 与 `listingToday` 同一条铁律：**判定一律读种子里的字段**，绝不用浏览器当天时间重算 —— 页面在次日
08:00 种子重建前打开时，浏览器日期比种子生成日多一天，重算会把「今日申购」显示成「明日申购」。
⚠ 「明日」按**自然日 +1**，不跳周末：申购日/上市日只可能是交易日，+1 落在非交易日时天然不会命中任何记录。

### firstDay

```jsonc
{
  "date": "2026-01-16",
  "open": 50.0, "high": 79.93, "low": null, "close": 54.95,
  "chgPct": 371.2693,        // 涨跌幅（%），相对发行价
  "highChgPct": 585.0,       // 最高价相对发行价涨幅（%）= (high/issuePrice-1)*100
  "avgPrice": 54.5589,       // 均价 = amount/volume（成交量单位股）；拿不到 volume 时为 null
  "avgChgPct": 367.92,       // 均价相对发行价涨幅（%）
  "turnover": 96.1425,       // 换手率（%）
  "amount": 987610000,       // 成交额（元）
  "amountYi": 9.8761,        // 成交额（亿元）
  "volume": 18101709,        // 成交量（股）
  "suspended": false         // 首日全天停牌（个别只有收盘价）
}
```

⚠ iFinD 另有「区间成交均价」指标，返回的是**整段区间**的均价而不是单日均价，**不要**用它当 `avgPrice`。

## 五、碎股门槛 `shatterFundWan` 算法（**估算，页面必须标注**）

北交所规则：按申购量比例配售，不足 100 股的碎股汇总后按申购量从大到小优先分配。
因此「稳拿碎股」需要在顶格档附近。估算：

```
if maxWinShares >= 100:  shatterFundWan = null        // 顶格已够整股，无需碎股
else:                    shatterFundWan = fd(100*issuePrice/(winRate/100)/1e4)
```
并同步写入 `derived`。页面标题上标「(估算)」。**不允许把估算值渲染成精确值。**

## 五·补 发行方案参数 `plan` 与两张测算表（页面实时计算）

对标参考站「实际发行打新门槛测算 / 收益率弹性测算」，**整张表不进 `ipo.json`**（会膨胀十几倍），
只在每条记录里存参数，由页面 JS 按下面公式实时渲染；用户可改输入框即时重算。

```jsonc
"plan": {
  "issuePrice": 20.3,            // 发行价（元）
  "peRatio": 12.0,               // 摊薄发行市盈率（倍）
  "epsAfter": 1.6917,            // 发行后 EPS（元/股）= issuePrice / peRatio
  "onlineIssueWan": null,        // 实际网上发行数量（万股），未披露时 null
  "onlineIssueWanUsed": 1800,    // 两表实际使用的值 = onlineIssueWan ?? subscribeCapShares*20/1e4
  "onlineIssueWanSource": "estimate",  // 'actual' | 'estimate' | null
  "subscribeCapShares": 900000,  // 网上申购上限（股）
  "topFundWan": 1827,            // 顶格申购资金（万元）= subscribeCapShares*issuePrice/1e4
  "lotFundWan": null,            // 1 手正股门槛（万元），需中签率；未出配售结果时 null
  "fundBaseYi": 11500,           // 「参与打新资金」默认档（亿元）
  "fundBaseSource": "estimate"   // 'actual'（用该股 totalFundYi 四舍五入）| 'estimate'（近 20 只中位数取整到百亿）
}
```

**⭐「网上发行数量」兜底估算的口径**：北交所规则是「网上申购上限 = 网上发行数量的 1/20」（即 5%）。
全量实测 92 条里 **90 条严格 ×20**，另 2 条 ×25（`920186.BJ` 中科仪、`920072.BJ` 科莱瑞迪）。
所以配售结果未披露时用 `subscribeCapShares*20/1e4` 估算，并必须把 `onlineIssueWanSource` 标成
`"estimate"`，页面在表头标注「网上发行数量为估算」。实测校对：远大健科 `900000*20/1e4 = 1800` 万股，
与参考站内嵌值 `1800.0` 万股逐字相同。
**`epsAfter` 的推导与校验**：`发行市盈率(摊薄) = 发行价 ÷ 发行后EPS`，反推 `epsAfter = issuePrice/peRatio`。
实测：远大健科 `20.30/12.00 = 1.6917`，×6900 万股 = 1.167 亿 ≈ 其 2025 扣非 1.17 亿 ✓；
文峰光电 `11.45/13.51 = 0.8475`，×8000 万股 = 0.678 亿 ≈ 其 2025 扣非 0.68 亿 ✓。

#### 打新门槛测算（页面函数 `thrRows(fund, online, top)`，复刻参考站/Excel A12:G26）

单位：`fund` 参与打新资金**亿元**、`online` 网上发行总数**万股**、`top` 顶格申购资金**万元**。
`f = fund`、`on = online`、`tp = top`。

```js
var A = f + 100 * i;                              // 参与打新资金，每 +100 一行，共 16 行
var C = 100 * A / on;                             // 1手正股门槛(万元) = 100×打新资金/网上发行总数
var B = (C >= tp) ? 100 : Math.ceil(tp / C) * 100;// 顶格申购中签股数：门槛≥顶格→全员碎股=100；否则 ROUNDUP(顶格/1手门槛)×100
// N 手门槛（N=2,3,4,5）= (N*C < tp) ? N*C : "无"
```
表头：`参与打新资金（亿元） | 顶格申购中签股数 | 1手正股门槛(万元) | 2手 | 3手 | 4手 | 5手`。
默认输入 `fund = plan.fundBaseYi`、`online = plan.onlineIssueWanUsed`、`top = plan.topFundWan`。

#### 收益率弹性测算（页面函数 `elasticRows(pe0, step, eps, price, online, top, fund0)`，复刻 Excel A28:F46）

默认 `pe0 = 20`、`step = 5`、`eps = plan.epsAfter`、`price = plan.issuePrice`、`fund0 = plan.fundBaseYi`。

```js
var pe     = [pe0, pe0+step, pe0+2*step, pe0+3*step, pe0+4*step];
var priceJ = pe.map(p => p * eps);                 // 对应股价 = PE × 发行后EPS
var upJ    = priceJ.map(pj => pj / price - 1);     // 涨幅 = 对应股价 ÷ 发行价 − 1
var funds  = [0,3,6,9,12,15].map(k => f0 + 100*k); // 取门槛表的第 1/4/7/10/13/16 行
// 每个资金档位 f：c1 = 100*f/online；b = (c1>=top) ? 100 : Math.ceil(top/c1)*100
//   踩门槛中 1 手收益率 = price*up*100/c1/10000   （百分比，toFixed(4)）
//   顶格申购收益率     = price*up*b/top/10000
```
表结构：上半「踩门槛中 1 手的收益率弹性测算」四行 = `给予20XX年业绩PE / 对应股价(元) / 涨幅 / <各资金档位>`；
下半「顶格申购的收益率弹性测算」两行 = `涨幅 / <各资金档位>`。输入框：起始 PE、PE 步长、发行后 EPS、发行价。

⚠ 两张表的 `fundBaseYi` 在 `fundBaseSource === 'estimate'` 时是**估算档位**，页面必须标注；
`epsAfter` 是从市盈率反推的，公司实际扣非 EPS 以财报为准。

## 五·补二 基本面数据 `fund`（可比公司估值 / 公司基本面 / 财务趋势）

数据源全部是 iFinD，**只对「未上市」+「当天上市」两类标的取**
（`status !== 'listed' || listingToday === true`）；其余已上市记录的 `fund` 恒为 `null`，
页面在工作台 Tab 的申购清单展开体里对这两类标的渲染这几张卡。

### 取数：`data/raw/<code>.fund.json`（由 `tools/fetch.cjs` 的 fund 通道写）

```json
{
  "code": "920285.BJ", "name": "远大健科", "fetchedAt": "2026-10-14T19:30:00+08:00",
  "fundFetchedOn": "20261014",
  "industryPe": 21.83,
  "company": {
    "fullName": "远大健康科技(天津)股份有限公司",
    "founded": "2001-03-21",
    "address": "天津市静海区经济开发区金海道29号",
    "controller": "魏恩雨", "controllerType": "个人", "controllerPct": 39.4725,
    "business": "冰箱滤水器、泳池类滤芯、家用净水滤芯、咖啡机滤芯、整机等多品类净水器件及相关功能材料的研发、生产和销售。",
    "sponsor": "光大证券股份有限公司",
    "industryThs": "家用电器", "industrySw": "家用电器",
    "regCapital": "5100.0万", "employees": 604,
    "concepts": ["专精特新", "京津冀一体化", "人民币贬值受益", "净水概念", "跨境电商"],
    "issueSharesWan": null, "postIssueSharesWan": 6900,
    "products": []
  },
  "fin": [
    { "date": "2025-12-31", "revenue": 1010940000, "netProfit": 118390000, "deducted": null,
      "revenueYoy": 0.6282, "profitYoy": null, "grossMargin": 54.8673, "roe": 26.85 }
  ],
  "comps": [ { "name": "开能健康科技集团股份有限公司", "code": "300272.SZ",
               "pe": 64.3032, "pb": 2.8434, "mktCapYi": 36.0156, "shortName": "开能健康" } ],
  "compCandidates": ["A. O. Smith Corporation", "厦门建霖智慧家居股份有限公司"],
  "notice": { "title": "远大健科:向不特定合格投资者公开发行股票并在北京证券交易所上市发行公告", "date": "2026-10-13" },
  "noticeText": "……（公告正文拼接，截断 20000 字；**只留在 raw 里，不进 data/ipo.json、不进内联 seed**）",
  "missing": ["company.issueSharesWan", "company.products"]
}
```

⚠ `noticeText` 里偶有上游的 U+FFFD 替换字符（如 `科目具体情况及分<U+FFFD><U+FFFD><U+FFFD>说明`），解析前统一 `replace(/\uFFFD+/g,'')`；
它也**绝不能进成品**，否则会污染 `build/verify.cjs` 的「源码文本卫生」闸门。

**取数口径（实测见 `probe/probe-fund-out/` 与 `docs/PROBE-FINDINGS-FUNDAMENTALS.md`）**
| 字段 | 通道 | 说明 |
|---|---|---|
| `industryPe` | `stock.get_stock_events`，问法 `<code>的行业平均市盈率、首发时所属行业市盈率` | 实锤指标名 **`首发时所属行业市盈率`**；远大健科 21.83、文峰光电 22.33、南方乳业 21.45、莫森泰克 22.47 |
| `company.*`（结构化，**权威**） | `stock.get_stock_info` **两次**：`<code>的公司全称、成立日期、注册资本、注册地址、主营业务、实际控制人` 与 `<code>的保荐机构、所属行业、所属概念、员工人数` | 列名 `公司中文名称 / 成立日期 / 注册资本 / 注册地址 / 主营业务 / 实际控制人 / 实际控制人类型 / 实际控制人持股比例 / 首发保荐人 / 所属同花顺行业 / 所属申万行业 / 所属概念 / 员工总数`。⚠ **成立日期是 `20010321` 紧凑格式**，要转 `YYYY-MM-DD` |
| `company.issueSharesWan` / `postIssueSharesWan` / `products` | `news.search_notice`（公告片段兜底） | 公告片段**每次返回的段落都不一样**，这三项经常为 null，属正常；`time_start/time_end` 必填且窗口要 ~13 个月（-400 天） |
| `fin` | `stock.get_stock_financials`，问法 `<code>最近三年的营业收入、扣除非经常性损益后的归属母公司股东净利润、净利润、销售毛利率、净资产收益率` | 返回**按报告期逐行**（20231231/20241231/20251231），单位元。⚠ 金额列与同比列**归一化后同名**（`营业收入` vs `营业收入(同比增长率)` → `parseMdTables` 存成 `key#2`），取值必须按量级区分：金额 `|v| ≥ 1e5`、比率 `|v| ≤ 1e4` |
| `comps[].name` | `stock.get_stock_info`，问法 `<code>的可比公司、竞争公司` | **官方字段**，逗号分隔的**公司全称**（远大健科 13 家、文峰光电 11 家、南方乳业 10 家）；比从招股书正文正则捞名字可靠得多 |
| `comps[].code/pe/pb/mktCapYi` | `stock.get_stock_info`，全称 **5 个一批**：`<全称1>、<全称2>…的市盈率、市净率、总市值` | **禁止用 `search_stocks` 做「名称→代码」解析**（多个名字只回一条）；**禁止猜代码**（猜 `301469.SZ` 拿望圆科技，返回的是「恒达新材」）。解析不到的（未上市/非 A 股）不进 `comps`，留在 `compCandidates` |
| 全部 | 每只 **8 次调用** | 实测 4 只 95~110s（间隔 2500ms，含重试退避 5s/15s/45s） |

**`company` 的合并优先级**：结构化字段（`get_stock_info`）优先，公告片段只补空洞
（`mergeCompany(parseCompanyFromNotice(noticeText), infoCompany)`）。不要反过来——公告片段每次不一样。

### 成品：`BseIpoRecord.fund`

```json
"fund": {
  "industryPe": 21.83,
  "peDiscountPct": -45.03,
  "comps": [ { "name": "开能健康科技集团股份有限公司", "code": "300272.SZ",
               "pe": 64.3032, "pb": 2.8434, "mktCapYi": 36.0156 } ],
  "compsMeanPe": 34.15, "compsMeanPb": 2.04,
  "company": {
    "fullName": "远大健康科技(天津)股份有限公司", "founded": "2001-03-21",
    "address": "天津市静海区经济开发区金海道29号",
    "controller": "魏恩雨", "controllerType": "个人", "controllerPct": 39.4725,
    "business": "…", "sponsor": "光大证券股份有限公司",
    "industryThs": "家用电器", "industrySw": "家用电器",
    "regCapital": "5100.0万", "employees": 604,
    "concepts": ["专精特新", "…"], "products": [],
    "issueSharesWan": null, "postIssueSharesWan": 6900
  },
  "fin": [ { "…同 raw…" } ],
  "notice": { "title": "…", "date": "…" },
  "missing": ["company.issueSharesWan", "company.products"]
}
```

⚠ `comps[].name` 在成品里是**官方全称**（与 raw 一致），页面直接用；`shortName` 只留在 raw。
⚠ `noticeText` / `compCandidates` **不进成品**（成品只保留 `notice:{title,date}`）。

**规则（`build/build-seed.cjs` 负责，`build/verify.cjs` 逐条复算）**
- 只有 `status !== 'listed' || listingToday === true` 的记录才有 `fund`；其余为 `null`。
- `peDiscountPct = round((peRatio / industryPe - 1) * 100, 2)`；`peRatio` 或 `industryPe` 缺任一 → `null`。
- `compsMeanPe` = `comps` 中 `pe` 非 null 者的**算术平均**（2 位）；同理 `compsMeanPb`。样本为 0 → `null`。
- `comps` 里 `code` 解析不到的公司**保留在数组里**（`code:null`、估值全 `null`），让页面能显示「招股书披露但未上市/未匹配」。
- `fin` 排序：按 `date` 升序；只保留最近 3 期年报 + 之后的最新非年报期。
- 任何缺项一律 `null`，页面渲染 `—`；`missing[]` 记录缺失路径。
- ⚠ 页面必须标注：可比公司名单与行业市盈率**取自发行公告/招股书口径**，均值为算术平均，非指数化。

## 六、缺失即 `null`，绝不编造

- 任何算不出的字段一律 `null`，页面渲染为 `—`。
- 严禁用 0、`""`、参考站数值填充。
- 严禁硬编码任何演示数据进 HTML；页面只渲染 seed。
- 分时类字段（最高价时间、09:30~15:00 换手率）**本版不做**：iFinD 的
  `stock_highfreq_quotes` 不支持北交所，且只支持当日实时、无历史。
- `get_stock_summary` 不支持北交所，禁用。

## 七、文件清单（构建产物）

| 文件 | 说明 |
|---|---|
| `data/raw/universe.json` | 股票池缓存（三路合并的结果） |
| `data/raw/<code>.json` | 逐股原始缓存 |
| `data/watch.json` | **输入**（不是产物）：手工关注清单，第三路股票池，见 §二·补 |
| `data/ipo.json` | 成品（格式化，便于 diff） |
| `data/ipo.js` | `window.BSE_IPO = {...}`，与 ipo.json 内容一致 |
| `data/manifest.js` | `window.BSE_MANIFEST = {generatedAt,schemaVersion,counts,hash}` |
| `北交所打新工作台.html` | 单文件工作台，内联 seed |
