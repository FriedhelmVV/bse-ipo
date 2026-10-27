# 北交所打新工作台

一个**单文件、零依赖、可离线**的北交所（BJSE）打新看板：把「正在申购 / 已申购待上市 / 已上市」的新股放在一张表里，算清顶格资金、中签门槛、首日表现。

- 成品：`北交所打新工作台.html` —— 双击 `file://` 就能用，不联网、不装任何东西、不引任何 CDN。
- 数据：**全部由同花顺 iFinD 自动取数生成**，不含任何手工抄录或演示数据；取不到的字段一律显示 `—`。
- 更新：`tools/daily-update.cjs` **每天 00:00 起每小时**由 Windows 计划任务跑一遍（取数 → 构建 → 内联 → 断言 → 发布）；本「市场日」成功过就跳过，失败/被系统掐掉的下一小时自动补；跳过之前还会用 1 次调用探一下**有没有白天新公告的新股**，有就立刻补跑一轮（见 §六.20）。
- 固定网址：**<https://friedhelmvv.github.io/bse-ipo/>** —— 由 `tools/publish-pages.cjs` 发布到 GitHub Pages（仓库 `FriedhelmVV/bse-ipo`，配置见 `tools/pages.json`）。

> 数据来源：同花顺 iFinD。仅供研究参考，**不构成任何投资建议**。

---

## 一、先跑起来

### 1. 直接看（离线）

双击 `北交所打新工作台.html`。页面把数据内联在 `<script id="bse-seed">` 里，所以 `file://` 下也有完整内容。

### 2. 起本地服务（手机 / 局域网 / 隧道用）

```bat
node tools\serve.cjs                 :: http://127.0.0.1:8788
node tools\serve.cjs --port 9000
node tools\serve.cjs --host 0.0.0.0  :: 局域网可访问
```

只读静态服务：非 GET/HEAD 一律 405、不列目录、拦 `..`；`/`、`/index.html`、`/workbench.html`、`/dashboard.html` 都指向工作台页（给隧道和二维码用不带中文的 URL）；`/data/*` 自动 `no-store`，更新后刷新即见；`/healthz` 是 JSON 健康检查。

### 3. 一键从「取数」跑到「上线」

双击 `上线工作台.cmd`（起服务 + 隧道 + 公网自测），或：

```bat
node tools\daily-update.cjs            :: 完整跑一遍（计划任务用这个）
node tools\daily-update.cjs --dry-run  :: 只打印将要执行的命令，一个进程都不起
node tools\daily-update.cjs --no-publish
node tools\daily-update.cjs --from verify
node tools\daily-update.cjs --only publish
```

### 4. 发布到固定网址

```bat
node tools\publish-pages.cjs --init     :: 首次：建仓库 → 开 Pages → 全量推送
node tools\publish-pages.cjs            :: 之后：增量推送（只提交内容变了的文件）
node tools\publish-pages.cjs --status   :: 看仓库 / Pages 状态 / 网址
node tools\publish-pages.cjs --dry-run  :: 只组包并算差异，不发任何写请求
node tools\publish-pages.cjs --stage DIR:: 把站点落到 DIR 本地预览
```

令牌按优先级取自环境变量 `GITHUB_TOKEN` 或文件 `tools\.gh-token`（一行，**别提交、别外发**）。

### 5. 每天自动更新（计划任务）

```bat
node tools\schedule.cjs                        :: 查看状态（默认，只读）
node tools\schedule.cjs --install              :: 注册：00:00 起每小时跑 daily-update.cjs --catch-up --quiet
node tools\schedule.cjs --install --time 07:00 :: 换一个起始时间（改完记得同步 daily-update.cjs 的 DAY_START_HOUR）
node tools\schedule.cjs --run                  :: 立刻手工触发一次（等价于点「运行」）
node tools\schedule.cjs --uninstall            :: 卸载
node tools\schedule.cjs --dry-run              :: 只打印将注册的任务，绝不写系统
```

任务名 `BseIpoDailyUpdate`，`MultipleInstancesPolicy=IgnoreNew`、`ExecutionTimeLimit=PT45M`、`LogonType=InteractiveToken + LeastPrivilege`（**不需要管理员、不存密码**）、`StartWhenAvailable=true`（到点时没开机就开机后补跑）。

**触发器是「每天 00:00 起，每 1 小时一次，持续 16 小时（`<Repetition><Interval>PT1H</Interval><Duration>PT16H</Duration>`，即 00:00–16:00 共 17 次机会）**，不是一天赌一次。代价几乎为零，因为 `daily-update.cjs --catch-up` 是幂等的。

补跑闸门 `--catch-up` 的判据是**「市场日」**：本地 **00:00 之前算前一天** —— 也就是「一次更新周期 = 当天 00:00 到当天 23:59」，同一市场日已经成功跑过就 `SKIP`，跨市场日一定跑。于是：

- 00:00 那次失败/被掐掉 → 01:00、02:00… 每个整点自动重来，一直到 16:00；
- 00:00 成功之后，当天剩下的触点在 `SKIP` 之前会先跑一次 **新股探测**（`tools/probe-new.cjs`，只花 1 次 iFinD 调用）：发现还没进池子的新公告新股就**立刻补跑一轮**，没有才真的 `SKIP`。这是为了补上「发行公告白天才发」这个洞（2026-10-27 森峰激光就是这么漏的），见 §六.20；
- 起点小时**必须等于** `daily-update.cjs` 与 `tools/fetch.cjs` 的 `DAY_START_HOUR`（现在都是 0），否则「本市场日跑过没有」会和触发器错位 —— `build/verify.cjs [11]/[12]` 会核对三者相等。
- 起点取 0 时市场日的分界正好落在午夜，所以「市场日」就是 +08:00 的自然日，最容易理解；若改成别的整点（例如 `--time 07:00`），记得把 `daily-update.cjs` 与 `tools/fetch.cjs` 里的 `DAY_START_HOUR` 一起改，`verify.cjs [11]/[12]` 会三向比对。

排查「数据怎么没更新」用只读诊断（一步都不执行）：

```bat
node tools\daily-update.cjs --explain-catch-up
node tools\daily-update.cjs --explain-catch-up --at=2026-10-17T03:00:00+08:00
:: → { nowLocal, marketDay, lastSuccessAt, lastSuccessDay, lastSuccessFetched, decision: "RUN…/SKIP…" }
```

（早期版本是「一天一次 07:00」+「自然日」判据，2026-10-16 被现实打穿：电脑 04:30 睡到 09:12，07:00 那次根本没触发；09:18 的补跑在第 10 秒被硬杀（`exit=0xC000013A`），`RestartOnFailure` 没兜住，站点 13 小时没更新。中间还用过「每晚 22:00」（2026-10-16 ~ 10-17），与短暂用过的「每天 02:00」（2026-10-27）；最终按使用者要求定为 **00:00**。更早还用过「距上次成功满 20 小时」，会把「前一天手工跑过一次」之后的第二天那次吞掉。都已改。）

---

## 二、数据管线（四层，职责分明）

```text
tools/fetch.cjs          iFinD 取数（唯一联网点）
        ↓               增量写 data/raw/universe.json + data/raw/<code>.json
build/build-seed.cjs     纯计算，无网络：算派生指标、判状态
        ↓               写 data/ipo.json / data/ipo.js / data/manifest.js
build/inject-seed.cjs    把 data/ipo.json 内联进 HTML 的 <script id="bse-seed">
        ↓
build/verify.cjs         契约断言（含基准股逐字段比对）；不过就绝不发布
```

为什么这样分层：**取数结果（`data/raw/`）只增不改**，派生逻辑（`build-seed.cjs`）可以随时重算、随时改公式，不用重新联网；页面既能 `file://` 离线看（内联种子），也能在线读 `data/*.js` 覆盖（右上角徽标会显示数据来自内联种子还是 `data/` 目录）。

```bat
node tools\fetch.cjs                          :: 增量：只补缺的
node tools\fetch.cjs --full                   :: 忽略缓存全量重取
node tools\fetch.cjs --codes=920086.BJ        :: 只取指定代码（⚠ 此模式不合并股票池）
node tools\fetch.cjs --limit=5 --interval=3000
node build\build-seed.cjs --today=2026-10-13
node build\inject-seed.cjs [--check]          :: --check 只校验不写
node build\verify.cjs
```

股票池来自三条路：按年已上市、待上市/即将申购、以及**手工关注清单 `data/watch.json`**（iFinD 选股索引滞后时登记新公告的代码）。详见 §六.17。

字段口径、公式、单位、缺失语义**全部**写在 [`SCHEMA.md`](SCHEMA.md)，改字段先改它。

---

## 三、页面结构

四个 Tab（对照参考站做了减法，把做不扎实的先砍掉）：

| Tab | 内容 |
| --- | --- |
| 🧰 打新工作台 | **申购清单**：待上市与**当天上市**新股（即将申购 / 已申购待上市 / 今日上市）逐只一行，点标题展开该只的完整分析（发行方案 + 五张测算卡，见下）。清单下方是「最近已上市新股 · 首日表现速览」 |
| 📊 打新数据 | 全量打新记录表，点表头排序，可按代码/名称搜索 |
| 💰 资金档位分布 | 按顶格资金分档，标出近期重点与参考性弱的早期数据 |
| 🚀 上市首日表现 | 首日开/高/低/收/均价、涨跌幅、换手率、临停、T+1、成交额 |

- 四个主题：经典深色 / 浅色白底 / 米白纸感 / 深蓝石板，存 `localStorage`。
- **所有「一排新股一行」的表格一律按「申购日期」从新到旧**（打新数据 / 资金档位分布 / 上市首日表现 / 首日表现速览都是），打新数据表点表头可改排序（见 §六.15）。
- 窄屏（<560px）卡片单列、表格内横向滚动。
- 图表是手写 SVG/CSS，**不引任何图表库**。

### 申购清单即容器

工作台顶部只有一份清单，**清单本身就是容器**：每只待上市新股一行（序号 / 简称 / 代码 / 状态 pill / 摘要行「发行价 · PE · 申购 · 上市 · 顶格」/ 右侧倒计时），点标题行展开，展开体里依次是

1. 发行方案摘要（8 个 `.elem`：发行价 / 首发市盈率 / 网上申购上限 / 顶格资金 / 申购日期 / 上市日期 / 中签率 / 行业）；
2. 五张测算卡（下表）。

- **同一时刻只展开一只**（`IPOC_OPEN` 单值）——展开体里沿用固定 id（`dtThr` / `dtEl` / `dtFund` / `dtPe0`…），多开会串台。
- 默认展开 `pickedForHero()` 挑中的那只（即将申购/最近申购的），其余收起；收起态的 `.ipoc-body` 是空字符串，不预渲染。
- 清单按 `subscribeDate` 降序；**当天上市**的照旧留在清单里（见下），**上市第二天起不进清单**，只在下方速览与「打新数据」Tab。
- 旧的三个独立窗口 `#wHero` / `#wDetail` / `#wPlan` 已从 DOM 移除（`verify.cjs` 会断言其不存在）。
- ⛔ **`m03109` 起「该不该打 · 要准备多少钱」块已整体删除**（含 `planListHTML()`、`.fold` / `.list` / `.li` 样式、`details.fold` 探针字段）。它想说的信息并没有丢：顶格资金与顶格中签股数在展开体的发行方案摘要和门槛测算里，碎股门槛在「打新数据」Tab 的列里。三道闸门都加了「不许再出现 `planListHTML` / `details class="fold"` / 文案『该不该打』」的反向断言（`verify.cjs` / `verify-real.cjs` / `verify-live.cjs`）。

#### 当天上市的新股也留在清单里（`listingToday`）

用户要求：**上市当天的标的要和「已申购未上市」一样给完整数据**，不能只有一行壳子。

- `status` **不新增枚举值**：当天上市仍是 `listed`，另给一个布尔字段 `listingToday`（等价于 `status==='listed' && listingDate === today`）。`counts`、「打新数据」Tab、「上市首日表现」Tab 全部照旧把它当已上市，零改动。
- 页面侧：`isListingToday(r)` / `inWorkbench(r)`；`ipocList()` 用 `inWorkbench()` 收录；`ipocBodyHTML()` 与 `detailCardsHTML()` 的守卫是 `statusOf(r)==='listed' && !isListingToday(r)`（**写成裸 `statusOf(r)==='listed'` 就会把当天那只一起短路掉** —— 这正是本次修的 bug）；状态 pill 文案改走 `statusTxtOf(r)`，当天显示金色「今日上市」。
- 数据侧两处必须同步，缺一处当天就是空卡：
  - `build/build-seed.cjs` 的 `buildFund(fr, status, peRatio, listingToday)`：`status === 'listed' && !listingToday` 才返回 `null`；
  - `tools/fetch.cjs` 的 fund 目标过滤：`!it.listingDate || it.listingDate >= todayDash`（**`>=`**，`>` 会让当天上市的取不到基本面）。
- 次日起 `listingDate < today` → `listingToday=false` → 自动退出清单，不需要任何清理逻辑。
- ⚠ 页面一律读种子里的 `r.listingToday`，**不要用浏览器当天时间重算**：页面在每天 00:00 种子重建之前打开时，浏览器日期会比种子生成日多一天，两边会打架（`status` 用的是种子生成日）。

#### 「四个时点」高亮：今日申购 / 明日申购 / 今日上市 / 明日上市

用户要求（`m06106`）：**每天更新时，把「明日申购、当天申购、明日上市、当天上市」的新股高亮出来**，一眼就能看到今天该干什么、明天要准备什么。

- 实现方式与 `listingToday` 同一套铁律：**四个布尔字段在 `build/build-seed.cjs` 里按种子生成日算好**，页面只读，绝不拿浏览器日期重算。
  - `subscribeToday` ＝ `subscribeDate === today`
  - `subscribeTomorrow` ＝ `subscribeDate === today + 1`（`addDaysYmd(today, 1)`）
  - `listingToday` ＝ `listingDate === today`
  - `listingTomorrow` ＝ `listingDate === today + 1`
- 优先级（`HL_ORDER`）：今日上市 > 明日上市 > 今日申购 > 明日申购；一只标的只显示最靠前的那一个（不会同时挂两个徽标）。
- 页面表现（`北交所打新工作台.html`）：
  - 徽标文案走 `statusTxtOf(r)`；命中时 `statusSpan()` 输出 `.status.hl`（金色描边＋光晕），清单卡片整卡加 `.ipoc.hl`（描金边＋序号变金底），**「打新数据」表命中行加 `tr.hl`（金色左侧竖条＋淡金底）**；
  - 卡片右侧时点文案（`headTxt`）：`今天上市 · 上市日 …` / `明天上市 · 上市日 …` / `就是今天申购` / `明天申购`；
  - 「资金档位分布」表的「备注」列也带 `今天申购 / 明天申购 / 明天上市`。
- ⚠ 「明日」按**自然日 +1** 算，不跳周末：周五看到的「明日」是周六，而北交所不会在周六申购/上市，所以那天自然不命中 —— 这条是有意的，不要改成「下一个交易日」（页面没有交易日历，猜比不猜更糟）。
- 闸门：`verify.cjs` 静态契约＋逐条复算；`test-build-seed.cjs` 三个基准日覆盖四种命中；`verify-real.cjs` 的 `[C7]` 段逐卡逐行比对 `.hl` 类、文案、金色样式；`verify-live.cjs` 校验线上四个字段与标记齐全。

### 五张测算卡（在展开体里）

只有 `status !== 'listed'`（即还没上市）的标的才渲染这五张卡；已上市的不再给申购测算。

| 卡片 | 内容 | 数据来源 |
| --- | --- | --- |
| 打新门槛测算 | 16 档「参与打新资金」→ 1 手正股门槛、2~5 手门槛、顶格申购中签股数。三个输入框（参与资金 / 网上发行数量 / 顶格资金）可改，表格实时重算 | 页面按 `plan` 实时算，**不预生成进数据** |
| 收益率弹性测算 | **只按顶格申购算**：在假设的 PE 区间（起始 PE / **步长默认 10 倍**，档位 20/30/40/50/60，两个输入框可改）下，顶格申购的收益率，以及对应**年化收益率**（资金占用 9 天分段加权：申购日→中签公布日 2 天按顶格资金全额，公布日→上市日 7 天按中签股数 × 发行价）。**属情景假设，非预测** | 同上 |
| 可比公司估值 | 可比公司 / 证券代码 / PE / PB / 总市值 + 算术平均行 + 「发行 PE 较行业静态 PE 折价/溢价百分之几」 | 名单 `get_stock_info` 的官方「可比公司」字段；估值 `get_stock_info`；行业 PE `get_stock_events` |
| 公司基本情况 | 公司全称 / 成立日期 / 注册资本 / 注册地址 / 所属行业 / 实际控制人 / 保荐机构 / 员工总数 / 发行后总股本 / 主营业务 + 所属概念、主要产品名条 | `get_stock_info` 结构化字段为主，公告片段只补空洞 |
| 财务趋势 | 近三年 + 最新一期营收与净利柱图（`class="chartbox inline-chart"`）+ 6 列表（报告期 / 营收 / 净利 / 营收同比 / 毛利率 / ROE） | `get_stock_financials` |

> 门槛测算与弹性测算的公式逐字对齐参考站（Excel `A12:G26` 与 `A28:F46`），见 [`SCHEMA.md`](SCHEMA.md) §五·补。**估值与财务都是「发行公告口径」，不是实时行情**；取不到的一律渲染 `—`。

---

## 四、数据口径与已知缺口

**关键事实**：只有**已完成 IPO（已上市）**的新股才有中签类字段；已申购未上市的新股这些字段为空是正常的，不是取数失败。页面据此把记录分为 `upcoming / subscribed / listed` 三态。

| 字段 | 来源 |
| --- | --- |
| 申购日期 / 首发价格 / 首发市盈率 / 网上申购数量上限 | `get_stock_events` |
| 网上发行数量 / 中签率 / 有效申购户数 / 获配户数 | `get_stock_events`（**仅已上市**） |
| 上市日期 / 行业 | `get_stock_info` |
| 首日及 T+1 开高低收、涨跌幅、换手率、成交额、成交量、流通市值 | `get_stock_performance`（按明确日期区间逐只取） |
| 首发时所属行业静态市盈率 | `get_stock_events`（**仅未上市标的取**） |
| 公司全称 / 成立日期 / 注册资本 / 注册地址 / 主营业务 / 实际控制人 / 保荐机构 / 所属行业 / 所属概念 / 员工总数 | `get_stock_info` 两次结构化问法 |
| 近三年 + 最新一期营收 / 净利 / 营收同比 / 毛利率 / ROE | `get_stock_financials`（报告期必须显式写「最近三年」） |
| 可比公司名单 | `get_stock_info` 的官方「可比公司」字段（**不是**从公告正文正则捞的） |
| 可比公司 PE / PB / 总市值 | `get_stock_info`（按公司**全称**问，代码由 iFinD 回填） |
| 发行股数 / 发行后总股本 | `news.search_notice` 公告片段（不稳定，取不到即 `—`） |

- 上述「基本面」六行只对 `status !== 'listed'` 的标的取数，落在 `data/raw/<code>.fund.json`，成品里是 `record.fund`（详见 [`SCHEMA.md`](SCHEMA.md) §五·补二）。

- 顶格资金 = `网上申购数量上限 × 首发价格 ÷ 1e4`（万）。该公式已用参考站样本逐字校验通过。
- 首日均价 = `首日成交额 ÷ 首日成交量`。**注意**：query 里带「成交均价」返回的是整段**区间**均价，不能当单日均价，故不取。
- **碎股门槛是估算值**，列头必须带「(估算)」。
- 日线**倒序返回**（最新在前）；停牌行只有收盘价，勿当 0。

**明确缺口（页面如实标注，不做假数据）**：分时数据不可得 —— `stock_highfreq_quotes` 对北交所代码解析失败且只支持当日实时、无历史。所以参考站的「首日实时看板 / 首日动态复盘 / 最高价&换手率分时表」三个 Tab 本版**不做**。完整实测证据见 [`docs/PROBE-FINDINGS.md`](docs/PROBE-FINDINGS.md)。

---

## 五、目录结构

```text
bse-ipo/
├─ 北交所打新工作台.html     成品页面（单文件，数据内联）
├─ SCHEMA.md                 数据契约（改字段先改这里）
├─ README.md
├─ docs/
│  ├─ PROBE-FINDINGS.md      iFinD 能力实测证据与缺口表
│  └─ shots/                 无头 Chrome 验收截图与 verify*.json
├─ data/
│  ├─ raw/                   取数原始缓存（universe.json + <code>.json + <code>.fund.json + _errors.json）
│  ├─ watch.json             手工关注清单：iFinD 选股索引还没收录的新股代码（见 §六.17）
│  ├─ ipo.json / ipo.js      成品数据
│  └─ manifest.js            索引与 hash
├─ tools/
│  ├─ fetch.cjs              iFinD 取数（唯一联网点）
│  ├─ serve.cjs              只读静态服务
│  ├─ publish-pages.cjs      发 GitHub Pages
│  ├─ daily-update.cjs       每日编排（取数→构建→内联→断言→发布）
│  ├─ probe-new.cjs          新股探测：1 次调用查「已公告未上市」池，新代码写进 data/watch.json（见 §六.20）
│  ├─ schedule.cjs           注册/卸载计划任务 BseIpoDailyUpdate
│  ├─ pages.json             仓库配置
│  ├─ _peek.cjs              查看某只股票的原始缓存关键字段（排查用）
│  └─ .gh-token              GitHub 令牌（勿提交）
├─ build/
│  ├─ build-seed.cjs         raw → 成品数据
│  ├─ inject-seed.cjs        成品数据 → 内联进 HTML（整对象序列化，勿手工挑字段）
│  ├─ verify.cjs             契约断言（发布前的闸门）
│  ├─ test-build-seed.cjs    离线回归测试（不联网、不碰 data/）
│  ├─ sample-seed.cjs        边界假数据，仅用于页面自测
│  ├─ mkshot-bse.cjs         无头 Chrome 截图（--real 用真实内联种子）
│  ├─ verify-real.cjs        用**真实 93 条**验证渲染与加载降级（--shot 出图）
│  ├─ verify-boot.cjs        四种加载/降级路径（用 12 条假数据）
│  ├─ verify-serve.cjs       静态服务的真站点断言（路由/穿越/敏感路径）
│  ├─ verify-publish.cjs     发布组包的离线验收（绝不发请求）
│  ├─ verify-live.cjs        线上固定网址验收（只读 GET：标题编码/种子真伪/零 CDN/data 资源）
│  ├─ audit-deliverable.cjs  交付前体检（修复是否在位 / 事故残留 / 种子真伪）
│  └─ check-data-encoding.cjs 用 node 判断编码与中文是否正常（不靠 PS 肉眼）
├─ probe/                    iFinD 能力探针与原始返回存档
├─ 上线工作台.cmd
└─ 发布到固定网址.cmd
```

---

## 六、改动约定

1. **先改 [`SCHEMA.md`](SCHEMA.md)**，再同时改页面与 `tools/fetch.cjs`。
2. Node **不在 PATH** 上，用固定路径：
   `C:\Users\p\.dsh\dsh-runtimes\dsh-primary-runtime\dependencies\node\bin\node.exe`
3. 新增行情指标：把指标名加进 `tools/fetch.cjs` 的 `PERF_FIELDS_DESIRED` —— 旧缓存因 `perf.fields` 不含它会被自动判为「需重取」，不用手删 `data/raw/`。
4. 禁止引入任何 CDN / 第三方库；图表一律手写 SVG/CSS。`build/verify.cjs` 会把外部资源引用判为失败。
5. **禁止硬编码演示数据**。缺数据就渲染 `—`，并在 `missing[]` 里记录。
6. 发布前的闸门是 `build/verify.cjs`：**它不过，就不发布会**（`daily-update.cjs` 任一步失败即中止，`publish-pages.cjs` 排在最后）。
7. 改完跑 `node build/test-build-seed.cjs`（离线回归）与 `node build/verify.cjs`。
8. **布局约定：每个 Tab 内一律「图表在上、表格在下」。** 带 `<div class="chartbox">` 的卡片必须排在 `<table>` 卡片之前；工作台 Tab 的身份块（hero 大卡）仍置顶，其后的「首日表现速览」属于内容区，不参与这条约定。
   - **例外**：`class="chartbox inline-chart"` 是「跟着自己后面那张小表」的内联小图（工作台 Tab 的「财务趋势」卡里两张营收/净利柱图），不受这条约定约束，可以出现在表格之后。`build/verify.cjs` 的布局闸门只认看板级 `chartbox`（正则 `class="chartbox(?![^"]*inline-chart)`）。
9. **悬浮提示约定：柱子/点只显示「新股简称 + 对应数据」。** 提示内容由 `tipRows()` / `tipFor()` 生成，经 `drawChart` 的 `opt.tipName` 开关进入「只有简称 + 一个数值」模式（不带日期、不带附加行）；目前 `chartFundTrend()`（打新资金）与 `chartSubsTrend()`（有效申购户数）都用这个模式，指标名分别是 `打新资金(亿元)`、`有效申购户数(户)`。
   ⚠️ **`attrEsc()` 必须转义 `< >`，不能删除它们**：`data-tip` 属性里放的是 HTML 片段（`<div class="tt">…</div><div class="tr">…</div>`），`attachTip()` 用 `innerHTML` 渲染它。早期版本写成 `.replace(/[<>]/g,'')`，尖括号被删掉后提示会显示成一串裸文本（`div class="tt"星图测控/div…`）。正确写法（顺序不能变）：
   ```js
   function attrEsc(s){ return String(s==null?'':s).replace(/&/g,'&amp;').replace(/"/g,'&quot;').replace(/</g,'&lt;').replace(/>/g,'&gt;'); }
   ```
   `build/verify.cjs` 会检查这条（转义方式 + 顺序 + 两张趋势图用了 `tipName:true` 且不再拼 `sub` 附加行），`build/verify-real.cjs` 会逐根柱子核对提示的实际内容（标题必须是真实新股简称、恰好一行「指标名 + 数值」、不含日期/顶格资金/中签率/获配户数）。
10. **基本面数据（`record.fund`）的口径不能自作主张**：
    - 金额与同比列**同名**（`营业收入` vs `营业收入(同比增长率)`，`normHeader` 会剥掉括号）→ `parseMdTables` 重名键存成 `key#2`，`parseFinancials` 再按量级区分（`|v| ≥ 1e5` 是金额、`≤ 1e4` 是比率）。改这里必须两个函数一起改。
    - `company` 的合并优先级是 **`get_stock_info` 结构化字段 > 公告片段**（`mergeCompany(parseCompanyFromNotice(txt), parseCompanyFromInfo(ans))`）。反了就等着公告片段里的偶发垃圾覆盖好数据。
    - 取数**禁止猜股票代码**，也**不要用 `search_stocks` 做「简称 → 代码」解析**（多个名字一起问只回一条）。一律把公司**全称**丢给 `get_stock_info`。
    - 财务问法必须显式写「最近三年」，否则 iFinD 只给最新 MRQ；ROE 必须显式在 query 里要「净资产收益率」。
    - **保荐机构（`company.sponsor`）要单独补问一次**：它时灵时不灵 —— 2026-10-16 实测同一轮里 `920296.BJ` 正常返回「国元证券股份有限公司」，而 `920285.BJ` / `920196.BJ` 的合并问法（`<code>的保荐机构、所属行业、所属概念、员工人数`）返回空。修法是 `tools/fetch.cjs` 步骤 **5b**：合并问法结束后若 `!isPresent(infoCompany.sponsor)`，再单独问 `${code}的保荐机构` 并用 `parseCompanyFromInfo` 取 `首发保荐人`（`stats.fundCalls++`）。**只在缺失时才多发一次，稳态成本 0**；仍拿不到就留 null，页面按「缺项不渲染」整行不显示，**不猜**。`build/verify-real.cjs` 的断言因此写成「有值就逐字对上、没值则必须整行不渲染」。
    - `fund` 只对未上市标的产出；`noticeText` / `compCandidates` 是审计用的中间态，**绝不进 `data/ipo.json`、更不进内联 seed**（公告正文里偶有 U+FFFD，会污染源码文本卫生闸门）。
    - 「可比公司」名单来自 `get_stock_info` 的官方字段，名单里本来就有未上市/非 A 股主体，取不到代码是正常的，页面显示「未匹配」而不是报错。
11. **收益率弹性测算只按顶格申购口径算**，并同时给出年化收益率。PE 步长默认 **10 倍**（档位 20/30/40/50/60）——`var DT = { …, peStep:10, … }`，改默认值要在 `verify.cjs` 的「PE 步长默认 10」段同步。
    - 表一「顶格申购的收益率测算」：行 = 6 个资金档（`f0` 起每 +100 亿），列 = 5 个 PE 档；`顶格收益率 = 发行价 × 涨幅 × 顶格中签股数 ÷ 顶格资金`。
    - 表二「顶格申购的年化收益率」：形状同上，**资金占用按两段加权**（单位 万元·天）：
      ```
      资金占用 = 顶格资金 × 2 天  ＋  (中签股数 × 发行价) × 7 天
      年化收益率 = 收益 ÷ 资金占用 × 365
      ```
      前 2 天（申购日 → 中签公布日）冻结的是**顶格资金全额**；后 7 天（中签公布日 → 上市日）只占用**中签股数 × 发行价**。行标签里会显示「等效占用 X.XX 天」（= 资金占用 ÷ 顶格资金）。**不要退化成 `收益率 × 365 ÷ 9`** —— 两者差别很大（示例：加权 4.05% vs 9 天口径 0.90%）。
    - 天数与口径都写在源码顶部的常量 / `elasticRows()` 里，别把数字散进公式：
      ```js
      var EL_HOLD_DAYS = 2, EL_SETTLE_DAYS = 7;   // 申购日→中签公布日；公布日→上市日
      var occWanDays = tp * EL_HOLD_DAYS + bWan * EL_SETTLE_DAYS;   // bWan = price*b/1e4
      ann.push(t.map(function(v){ return v * 365 * tp / occWanDays; }));
      ```
      `build/verify.cjs` 会检查这两个常量仍是 `2` / `7`、`elasticRows()` 里出现 `bWan = price * b / 1e4` 与 `occWanDays = tp * EL_HOLD_DAYS + bWan * EL_SETTLE_DAYS` 与 `v * 365 * tp / occWanDays`、并且页面**不再**出现「踩门槛中 1 手」那张表。
    - `build/verify-real.cjs` 会逐格复算（`el[0]` 是收益率表首行、`el[6]` 是年化表首行），并断言年化**不等于**简单按 9 天折算的结果。
12. **页面脚本整体包在一个 IIFE 里 → 禁止用内联事件处理器调用内部函数。**
    - 页面末尾是 `(function(){ … })();`，所以 `onclick="ipocToggle('…')"` / `oninput="dtSet(…)"` 里的函数**在全局作用域不存在**，点击/输入时只会抛 `Uncaught ReferenceError: ipocToggle is not defined`，而**初始渲染完全正常**（所以肉眼和「读 DOM 初值」的断言都发现不了）。这个坑实际踩过两次：申购清单点标题不展开、门槛测算的输入框改了不重画。
    - 正确做法是**事件委托**：`ipocRender()` 在 `#wList` 上挂一个 `onclick`（用 `ipocHdOf()` 从 `ev.target` 往上找 `.ipoc-hd`），`ipocOpen()` 在展开体上调 `bindDtInput()` 挂 `input` 监听。
    - 输入框的键名**不能直接用中文标签**：`dtInput(id, 标签, …)` 的第二个参数是给人看的，`dtSet()` 要的是 `DT` 的键。用 `DT_KEY = { dtFund:'fund', dtOnline:'online', dtTop:'top', dtPe0:'pe0', dtStep:'peStep', dtEps:'eps', dtPrice:'price' }` 由 id 反查。
    - `build/verify.cjs` 会扫 `on(click|input|change)="(ipocToggle|dtSet|ipocOpen|dtPaintThr|dtPaintEl)"` 并判失败；`build/verify-real.cjs` 会真的派发一次 `input` 事件，断言表格首格从 `11,500` 变成 `12,000`（也就是「改输入框确实重画」）。
13. **`build-seed.cjs` 的 `generatedAt` 用 `--now=<ISO>` 固定**：默认取当前时刻，同一份输入两次运行必然差几毫秒，`test-build-seed.cjs` 的「重跑字节一致」幂等断言会随机失败。回归测试固定传 `--now=2026-10-13T00:00:00+00:00`。
14. **「一天一次」的计划任务不可靠 —— 电脑会睡觉。** 2026-10-16 实测：本机 04:30 进入睡眠，当天的触发**根本没发生**；09:12 唤醒后 `StartWhenAvailable=true` 在 09:18 补跑，`fetch` 跑到第 10 秒被系统**硬杀**（任务「上次结果」= `-1073741510` = `0xC000013A` = `STATUS_CONTROL_C_EXIT`，日志停在 `[fetch] iFinD 取数（增量） → …` 没有任何收尾行，`raw/` 只写了 2 个文件），`RestartOnFailure`（3 次 / 5 分钟）**没有兜住**。结果：站点 13 小时旧数据，直到用户来问。教训：`InteractiveToken` 的计划任务在睡眠中不执行，任何「一天只有一个机会」的编排都会被打穿。对策 = 触发器加 `<Repetition><Interval>PT1H</Interval><Duration>PT16H</Duration>`（起始小时起每小时一次，共 17 次机会）+ 幂等的 `--catch-up`；判据同时从「自然日」改成「市场日」（本地 `DAY_START_HOUR` 前算前一天），否则凌晨被唤醒的补跑会把当天那次吃掉。**开跑小时必须与 `DAY_START_HOUR` 相等**，`build/verify.cjs` 第 `[11]` 段把这条钉死（读两个常量比对 + 静态扫 XML 片段 + 用只读的 `--explain-catch-up` 验证市场日分界，且分界小时是从源码读出来的、不写死）。当前开跑时刻 = **每天 00:00**（`DEFAULT_TIME = '00:00'` / `DAY_START_HOUR = 0`，共 17 个机会到当天 16:00）。
15. **所有「一排新股一行」的数据表格，一律按「申购日期」从新到旧（降序）。** 涉及四处，改的时候别只改一半：① 打新数据表 `S2 = { k:'subscribeDate', dir:-1, … }`（`#resetQ` 重置也要写 `-1`）；② 资金档位分布表 `fundRender()` 里 `fundRows().reverse()`；③ 上市首日表现表 `fdRender()` 里把行按 `subscribeDate` 倒序重排；④ 工作台「首日表现速览」`quickHTML()` 自己按 `subscribeDate` 倒序取前 8。申购清单 `ipocList()` 本来就是这个口径。
    ⚠ **只许在渲染函数里重排副本，不许改 `fundRows()` / `fdRows()` 的返回顺序** —— 它们的升序原序是给 `chartFundTrend()` / `chartSubsTrend()` / 首日散点与柱图当时间轴用的，改了图表就倒着画。`build/verify-real.cjs` 有 4 条断言钉住（两条 `isDesc()` 扫 `#tblData` / `#tblFund` 的日期列，两条拿种子算出的期望序列逐位比对 `#tblFd` 的代码序与速览简称序）。
16. **窄屏闸门要用 iframe 量，不能用 `--window-size`。** Windows 上 Chrome 的 `--window-size` 有最小宽度（实测被夹到 ~500px）：`--window-size=390,2400` 会按 500px 布局却只截 390px 宽画布，看起来「被截断」其实是假象。`build/verify-real.cjs` 的 `narrowProbe(width)` 改成把页面塞进 390px 宽的 iframe（父页 1200 宽，需 `--allow-file-access-from-files` 才能读 `contentDocument`）再量 `scrollWidth`。
17. **新股发现有三条路，缺一条就会漏股。** `data/raw/universe.json` 的股票池 = ① 按年已上市（`search_stocks`）+ ② 待上市/即将申购（`UNLISTED_QUERY`）+ ③ **手工关注清单 `data/watch.json`**。
    - 为什么必须有第 ③ 条（2026-10-16 天元重工事故）：交易所当天公告的 `920306.BJ` 天元重工（现金申购发行日期 2026-10-20），在 iFinD 的**选股索引**里**完全查不到** —— `"北交所股票，股票代码大于920300"` 只回 `920300.BJ` 辰光医疗，连最宽的 `"北交所股票，上市日期为空"` 也只有 2 只；`search_stocks` 用简称、全称、代码问全是「抱歉，未找到符合要求的证券」。但**按代码单查 `get_stock_info` / `get_stock_events` 字段齐全**（发行价 4.48、发行市盈率 14.93、网上申购上限 181.9 万股、网上发行数量仍为空）。也就是说：**索引滞后 ≠ 数据没有**，只要知道代码就能取全，所以兜底通道必须是「手工登记代码」。
    - `data/watch.json` 形状：`{ "items": [ { "code": "920306.BJ", "name": "天元重工", "subscribeDate": "2026-10-20" } ] }`。`name` / `subscribeDate` 可省；`code` 必须是 6 位数字 + `.BJ`（非法项静默跳过，坏文件当空清单，绝不因为一个手写文件炸掉整轮取数）。**不消耗任何 iFinD 调用** —— 它只是把代码塞进池子，后面 events / profile / perf / fund 与池子里来的标的完全同路。iFinD 池子收录之后这里保留也无害。
    - 加完条目后要跑一次 `node tools\fetch.cjs`（别加 `--codes`，那样会跳过池子合并）才能让它进池子；`build/verify.cjs` 第 `[12]` 段会断言「关注清单里的每一只都已并进 `universe.json`」。
18. **所有「当日缓存」的失效键是「市场日」，不是自然日。** `tools/fetch.cjs` 里的 `marketYmd()`（与 `tools/daily-update.cjs` 的 `marketDay()` 同一套算法）用在两处：股票池第二路的 `cachedUnlistedOn !== marketYmd()`、基本面 `fundFetchedOn === marketYmd()`。写成 `todayYmd()` 的后果是：新周期开跑那天会**复用前一天那份属于上一个周期的池子**，等于整轮没重问 —— 这正是 2026-10-16 天元重工没被发现的第二个成因（`unlistedFetchedOn` 仍是 `20261016`，而那天 20:47 已经写过一次）。
    - **三处开跑小时必须相等**：`tools/schedule.cjs` 的 `DEFAULT_TIME`（触发器）、`tools/daily-update.cjs` 的 `DAY_START_HOUR`（catch-up 判据）、`tools/fetch.cjs` 的 `DAY_START_HOUR`（缓存周期）。改一处必须改三处，`build/verify.cjs` 第 `[12]` 段会三向比对。
19. **「四个时点」高亮由种子算，页面只读。** `subscribeToday` / `subscribeTomorrow` / `listingToday` / `listingTomorrow` 四个布尔全部在 `build/build-seed.cjs` 按种子生成日算出（`addDaysYmd(today, 1)` 拿「明天」），页面的 `hlOf()` / `statusSpan()` / `statusPill()` 只读字段。**页面里绝对不许出现 `addDays` 或 `new Date().getDate() + 1`** —— 页面在每天 00:00 重建种子之前打开时浏览器日期会比种子多一天，两边打架（`verify.cjs` 有反向断言）。详见 §三「四个时点」小节。
20. **`--catch-up` 每天跑成功之后，白天还要靠「新股探测」兜住新公告。** 计划任务是 00:00 开跑，之后每小时的触点一旦判定「本市场日已成功」就 SKIP —— 但**发行公告是白天才发的**：2026-10-27 实测 `920085.BJ 森峰激光`（申购 2026-10-28）是在当天 00:00 那一轮**之后**才进 iFinD 选股索引的，于是当天全部触点都 SKIP，用户在页面上整天看不到这只新股（来问「为什么没有更新」）。
    - 修法：`tools/probe-new.cjs` 只花 **1 次** iFinD 调用，用与 `tools/fetch.cjs` **一字不差**的 `UNLISTED_QUERY`（`北交所股票，上市日期为空，申购日期不为空`）问一次，拿回来的代码和 `data/raw/universe.json` 比对。
    - 发现新代码时它**写进 `data/watch.json`**（第三路股票池），随后那一轮 `fetch` 走关注清单把新股票并进池子 —— 所以补跑**不额外花配额**（走 watch 那条现有通道）。
    - `tools/daily-update.cjs` 的 `--catch-up` 在「本市场日已成功」分支里，**先 `probeNew()` 再决定 SKIP**：有新股就照常跑完整五步并记日志「探测到 N 只新公告新股…立刻补跑一轮」，没有才 SKIP。探测失败（断网 / iFinD 抖动）一律按「没有新股」处理，**绝不阻断 SKIP**（`probe-new.cjs` 自己 catch 后 `exit 0`）。
    - `--explain-catch-up` 保持**只读**，不触发探测。
    - `build/verify.cjs` 第 `[12]` 段钉住 4 条：`probe-new.cjs` 存在、两边 `UNLISTED_QUERY` 一致、探测结果会写 `data/watch.json`、`--catch-up` 的 SKIP 分支里 `probeNew()` 必须在 `SKIP` 输出**之前**（顺序反了等于没修）。

### 交付前验收（照抄即可）

```bat
set N=C:\Users\p\.dsh\dsh-runtimes\dsh-primary-runtime\dependencies\node\bin\node.exe
%N% build\test-build-seed.cjs        :: 离线回归：派生公式 / status 三分法 / 四个时点（71 项，不联网）
%N% build\verify.cjs                 :: 契约闸门：80 项，含内联 seed 与 ipo.json 顶层键逐项一致、自动更新自愈配置、新股发现通道（含白天新股探测）、四个时点、PE 步长
%N% build\verify-real.cjs            :: 真实 93 条渲染：4 Tab 数值齐备 + 申购清单展开体五张卡 + 表格排序 + 四时点高亮（113 项，无头 Chrome）
%N% build\verify-real.cjs --shot     :: 同上，并出 docs/shots/real-*.png
%N% build\verify-boot.cjs            :: 外部数据加载/降级四条路径
%N% build\verify-serve.cjs           :: 静态服务真站点断言（mode=real）
%N% build\verify-publish.cjs         :: 发布组包离线验收（不发任何请求）
%N% build\verify-live.cjs            :: 线上网址验收（首次发布后加 --wait 等 Pages 构建）
%N% build\audit-deliverable.cjs 北交所打新工作台.html   :: 交付体体检
%N% tools\daily-update.cjs --no-publish                :: 全链路（不在本机跑取数时加 --only）
```

### 排查

- 取数失败看 `data/raw/_errors.json`；限速很常见，`fetch.cjs` 已内置 3s 间隔 + 退避重试。
- 计划任务现场：`tools/last-run.log`（最近一次完整输出）、`tools/daily-update.log`（追加日志）、`tools/daily-update.state.json`（结构化状态）。
- 用 PowerShell 看本项目 JSON **必须**指定 `-Encoding UTF8`（默认按 GBK 解会乱码，看着像文件坏了，其实没坏）。
