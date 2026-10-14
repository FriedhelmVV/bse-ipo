# 北交所打新工作台

一个**单文件、零依赖、可离线**的北交所（BJSE）打新看板：把「正在申购 / 已申购待上市 / 已上市」的新股放在一张表里，算清顶格资金、中签门槛、首日表现。

- 成品：`北交所打新工作台.html` —— 双击 `file://` 就能用，不联网、不装任何东西、不引任何 CDN。
- 数据：**全部由同花顺 iFinD 自动取数生成**，不含任何手工抄录或演示数据；取不到的字段一律显示 `—`。
- 更新：`tools/daily-update.cjs` **每天早上 07:00** 由 Windows 计划任务跑一遍（取数 → 构建 → 内联 → 断言 → 发布）。
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
node tools\schedule.cjs --install              :: 注册：每天 07:00 跑 daily-update.cjs --catch-up --quiet
node tools\schedule.cjs --install --time 07:30 :: 换一个时间
node tools\schedule.cjs --run                  :: 立刻手工触发一次（等价于点「运行」）
node tools\schedule.cjs --uninstall            :: 卸载
node tools\schedule.cjs --dry-run              :: 只打印将注册的任务，绝不写系统
```

任务名 `BseIpoDailyUpdate`，`StartWhenAvailable=true`（07:00 没开机就开机后补跑）、`MultipleInstancesPolicy=IgnoreNew`、`ExecutionTimeLimit=PT45M`、`LogonType=InteractiveToken + LeastPrivilege`（**不需要管理员、不存密码**）。补跑由 `daily-update.cjs --catch-up` 的闸门兜住：**当天（Asia/Shanghai 自然日）已经成功跑过就跳过，跨天一定跑** —— 所以 07:00 没开机、09:00 才开机，会立刻补一次；一天不会跑两遍。（早期版本用「距上次成功满 20 小时」，会把「前一天晚上手工跑过一次」之后的第二天早上那次也吞掉，已改。）

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
node tools\fetch.cjs --codes=920086.BJ        :: 只取指定代码
node tools\fetch.cjs --limit=5 --interval=3000
node build\build-seed.cjs --today=2026-10-13
node build\inject-seed.cjs [--check]          :: --check 只校验不写
node build\verify.cjs
```

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
- 表格点表头排序；窄屏（<560px）卡片单列、表格内横向滚动。
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
- ⚠ 页面一律读种子里的 `r.listingToday`，**不要用浏览器当天时间重算**：页面在每天 07:00 种子重建之前打开时，浏览器日期会比种子生成日多一天，两边会打架（`status` 用的是种子生成日）。

### 五张测算卡（在展开体里）

只有 `status !== 'listed'`（即还没上市）的标的才渲染这五张卡；已上市的不再给申购测算。

| 卡片 | 内容 | 数据来源 |
| --- | --- | --- |
| 打新门槛测算 | 16 档「参与打新资金」→ 1 手正股门槛、2~5 手门槛、顶格申购中签股数。三个输入框（参与资金 / 网上发行数量 / 顶格资金）可改，表格实时重算 | 页面按 `plan` 实时算，**不预生成进数据** |
| 收益率弹性测算 | **只按顶格申购算**：在假设的 PE 区间（起始 PE / 步长可改）下，顶格申购的收益率，以及对应**年化收益率**（资金占用 9 天分段加权：申购日→中签公布日 2 天按顶格资金全额，公布日→上市日 7 天按中签股数 × 发行价）。**属情景假设，非预测** | 同上 |
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
│  ├─ ipo.json / ipo.js      成品数据
│  └─ manifest.js            索引与 hash
├─ tools/
│  ├─ fetch.cjs              iFinD 取数（唯一联网点）
│  ├─ serve.cjs              只读静态服务
│  ├─ publish-pages.cjs      发 GitHub Pages
│  ├─ daily-update.cjs       每日编排（取数→构建→内联→断言→发布）
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
    - `fund` 只对未上市标的产出；`noticeText` / `compCandidates` 是审计用的中间态，**绝不进 `data/ipo.json`、更不进内联 seed**（公告正文里偶有 U+FFFD，会污染源码文本卫生闸门）。
    - 「可比公司」名单来自 `get_stock_info` 的官方字段，名单里本来就有未上市/非 A 股主体，取不到代码是正常的，页面显示「未匹配」而不是报错。
11. **收益率弹性测算只按顶格申购口径算**，并同时给出年化收益率。
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

### 交付前验收（照抄即可）

```bat
set N=C:\Users\p\.dsh\dsh-runtimes\dsh-primary-runtime\dependencies\node\bin\node.exe
%N% build\test-build-seed.cjs        :: 离线回归：派生公式 / status 三分法（不联网）
%N% build\verify.cjs                 :: 契约闸门：60 项，含内联 seed 与 ipo.json 顶层键逐项一致
%N% build\verify-real.cjs            :: 真实 93 条渲染：4 Tab 数值齐备 + 申购清单展开体五张卡（无头 Chrome）
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
