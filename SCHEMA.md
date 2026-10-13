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
- `unlistedFetchedOn`：`YYYYMMDD`，第二路问法最后一次执行的日期（每天跑一次即可）

**股票池必须两路合并**（`tools/fetch.cjs`）：

| 路 | 问法 | 覆盖 |
| --- | --- | --- |
| ① 按年已上市 | `<年>年北交所上市的新股，按上市日期排序` | 已上市新股（每年一次调用） |
| ② 待上市 / 即将申购 | `北交所股票，上市日期为空，申购日期不为空` | 已申购未上市 + 即将申购 |

⚠ 只用①时「即将申购 / 已申购待上市」永远是空的 —— 而那正是打新工作台的主打内容。
实测①漏掉 `920196.BJ`/`920296.BJ`/`920285.BJ` 三只，②恰好精确返回这三只。

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
| `derived` | string[] | — | 本记录中由算法算出的字段名列表 | `[]` |
| `missing` | string[] | — | 本记录缺失字段名列表 | `[]` |

`status` 判定（三分法，`today` 取 Asia/Shanghai 当日）：
- `upcoming`：`subscribeDate` 为空，或 `subscribeDate > 今天`（含"明天申购"）
- `subscribed`：`subscribeDate <= 今天` 且（`listingDate` 为空 **或** `listingDate > 今天`）
- `listed`：`listingDate <= 今天`

⚠ 不能只看"`listingDate` 是否非空"：已公告上市日但日期在未来的新股（实测 `920162.BJ` 莫森泰克
`listingDate=2026-10-15`，而 `today=2026-10-13`）此刻**还没有首日行情**，必须算 `subscribed`，
否则会凭空多出一条 `firstDay=null` 的"已上市"记录，也会把「已申购待上市」这个工作台主打分组清空。
`build/verify.cjs` 会按上面的规则逐条复算并断言。

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
| `data/raw/universe.json` | 股票池缓存 |
| `data/raw/<code>.json` | 逐股原始缓存 |
| `data/ipo.json` | 成品（格式化，便于 diff） |
| `data/ipo.js` | `window.BSE_IPO = {...}`，与 ipo.json 内容一致 |
| `data/manifest.js` | `window.BSE_MANIFEST = {generatedAt,schemaVersion,counts,hash}` |
| `北交所打新工作台.html` | 单文件工作台，内联 seed |
