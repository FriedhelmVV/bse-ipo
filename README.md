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
| 🧰 打新工作台 | 今日/近期可申购、待上市，含顶格资金、中签门槛、状态 |
| 📊 打新数据 | 全量打新记录表，点表头排序，可按代码/名称搜索 |
| 💰 资金档位分布 | 按顶格资金分档，标出近期重点与参考性弱的早期数据 |
| 🚀 上市首日表现 | 首日开/高/低/收/均价、涨跌幅、换手率、临停、T+1、成交额 |

- 四个主题：经典深色 / 浅色白底 / 米白纸感 / 深蓝石板，存 `localStorage`。
- 表格点表头排序；窄屏（<560px）卡片单列、表格内横向滚动。
- 图表是手写 SVG/CSS，**不引任何图表库**。

---

## 四、数据口径与已知缺口

**关键事实**：只有**已完成 IPO（已上市）**的新股才有中签类字段；已申购未上市的新股这些字段为空是正常的，不是取数失败。页面据此把记录分为 `upcoming / subscribed / listed` 三态。

| 字段 | 来源 |
| --- | --- |
| 申购日期 / 首发价格 / 首发市盈率 / 网上申购数量上限 | `get_stock_events` |
| 网上发行数量 / 中签率 / 有效申购户数 / 获配户数 | `get_stock_events`（**仅已上市**） |
| 上市日期 / 行业 | `get_stock_info` |
| 首日及 T+1 开高低收、涨跌幅、换手率、成交额、成交量、流通市值 | `get_stock_performance`（按明确日期区间逐只取） |

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
│  ├─ raw/                   取数原始缓存（universe.json + <code>.json + _errors.json）
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
9. **悬浮提示约定：柱子/点只显示「新股简称 + 对应数据」。** 提示内容由 `tipRows()` / `tipFor()` 生成，经 `drawChart` 的 `opt.tipName` 开关进入「只有简称 + 一个数值」模式（不带日期、不带附加行）；目前 `chartFundTrend()`（打新资金）与 `chartSubsTrend()`（有效申购户数）都用这个模式，指标名分别是 `打新资金(亿元)`、`有效申购户数(户)`。
   ⚠️ **`attrEsc()` 必须转义 `< >`，不能删除它们**：`data-tip` 属性里放的是 HTML 片段（`<div class="tt">…</div><div class="tr">…</div>`），`attachTip()` 用 `innerHTML` 渲染它。早期版本写成 `.replace(/[<>]/g,'')`，尖括号被删掉后提示会显示成一串裸文本（`div class="tt"星图测控/div…`）。正确写法（顺序不能变）：
   ```js
   function attrEsc(s){ return String(s==null?'':s).replace(/&/g,'&amp;').replace(/"/g,'&quot;').replace(/</g,'&lt;').replace(/>/g,'&gt;'); }
   ```
   `build/verify.cjs` 会检查这条（转义方式 + 顺序 + 两张趋势图用了 `tipName:true` 且不再拼 `sub` 附加行），`build/verify-real.cjs` 会逐根柱子核对提示的实际内容（标题必须是真实新股简称、恰好一行「指标名 + 数值」、不含日期/顶格资金/中签率/获配户数）。

### 交付前验收（照抄即可）

```bat
set N=C:\Users\p\.dsh\dsh-runtimes\dsh-primary-runtime\dependencies\node\bin\node.exe
%N% build\test-build-seed.cjs        :: 离线回归：派生公式 / status 三分法（不联网）
%N% build\verify.cjs                 :: 契约闸门：45 项，含内联 seed 与 ipo.json 顶层键逐项一致
%N% build\verify-real.cjs            :: 真实 93 条渲染：4 Tab 数值齐备 + 无乱码（无头 Chrome）
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
