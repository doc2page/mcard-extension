# MCard 扩展线 ← Docker 线 同步更新实施细则

> **双线关系**：本仓库（扩展 MV3）是产品原始形态；Docker Web 版（mcard）2026-08 从扩展分叉重构。分叉后 Docker 线快速演进了多个版本（v1.0.0 → v1.2.2 + 分叉前重构期优化），本文件是把这些演进**同步更新**回扩展线的完整细则（14 项已全部完成）。
> **参考源**：Docker 线仓库（项目名 mcard，v1.2.2，根目录约定文档有全部架构约定）。
> 完成一项就在清单前打 `[x]` 并 commit。

---

## 一、模块对照表（同步时按此映射）

| 扩展文件 | Docker 对应 | 分化程度 | 同步策略 |
| --- | --- | --- | --- |
| `dropStats.js` | `src/lib/stats.js` | **高**（Docker 版大改） | 整文件对照重写：日界截断、双源聚合、称号映射 |
| `shared.js` | `src/lib/shared.js` | 低 | 逐函数 diff 补齐（computeUsable 等） |
| `marketStats.js` | `public/marketStats.js` | **高**（报告引擎/直方图新増） | 拷贝新函数 + 补 extractFacets 差异 |
| `portrait.js` | `public/portrait.js` | 中 | diff 补齐 |
| `background.js` | `src/lib/collector.js` + `trader.js` + `mteam.js` + `state.js`/`store.js` | **极高**（重构拆分） | 不做结构同步——只把「业务行为」同步进 background.js 对应逻辑 |
| `dashboard.js` | `public/app.js` | **极高** | 只同步「功能与交互规则」，UI 代码按 dashboard 既有风格实现 |
| `dashboard.css` | `public/app.css` | 高 | 同步设计值（对比度色值等），不搬布局 |
| `locales/` | `public/locales/{zh,en}.js` | 中 | 按 key 对照补齐新增文案 |
| `_locales/` | — | 扩展专属 | 不动 |

## 二、同步项清单

### P1 — 核心功能/修复（先行）

- [x] **S1 掉落统计：手动导入补全历史**
  - Docker：`collector.js` 的 `importDropMessages`（解析粘贴的 message search JSON → messages 全量合并，按 id 去重、只留能 parseDropContext 的、msgTotal 记录接口 total）+ `POST /api/drop-import` 入口 + 前端导入模态（粘贴 textarea → 导入 → 拉最新校验）
  - 补全判断用 `msgTotal`：`!msgTotal`（从未导入）或 `messages.length < msgTotal`（已导 < 接口总数）才显示导入 CTA。**勿用 rangeStart 判断**（不可靠，Docker 踩过坑）
  - 扩展落点：background.js 加消息处理 + chrome.storage 持久化；dashboard.js 加导入入口与模态
- [x] **S2 掉落双源去重不变量**
  - `feedCards` 只存 `createdDate > lastMsgDate`（messages 未覆盖的增量），三处维护此不变量：mergeDropFeed / importDropMessages / 存储加载。违反则双源重叠重复计数（Docker 修过）
- [x] **S3 掉落 dailyFull 日界**
  - `computeDropSummary` 的 since/refDay 起止必须截到日界 `slice(0,10)`（带时分会让日循环每天偏移、当天柱子被裁）
  - TZ 说明：Docker 版因容器 UTC 踩坑加了 TZ env；扩展跑在浏览器=用户本地时区，**天然免疫**，核对 `_todayStr` 等价物用的是本地日期即可，无需照搬
- [x] **S4 市场自挂单防自买**
  - 同款（filmId+rarity+provenance）同价 = 自己的挂单：单卡购买按钮 `disabled` 灰显 + title「这是你自己的挂单」；**批量模式也要拦截**（点选 toast + return + 隐藏勾选框），Docker 曾只灰显单卡漏了批量（v1.0.1 修复）
  - myAsks Set 构建：挂单列表过滤 `open+sell`，key=`filmId|rarity|provenance|price`
- [x] **S5 市场分析报告**
  - Docker：`marketStats.js` 的 `analyzeMarket(sum)` 返回语言无关 tier key（`expand/sell/divergence/cool/flat/nodata`），文案走 i18n `report.*`；HTML 报告前端生成（单文件、新 tab、可打印）；**排除当天**（基准日=昨天）；机制卡统一计入总 GMV 但稀有度筛选排除（归 mechMatrix）
  - `_priceHistogram` 返回桶边界（X 轴/tooltip 显示价格区间+占比）
- [x] **S6 我的卡册 view**
  - `computeCardBook`：持有+挂单(open sell)按 filmId 聚合、排除机制卡；排序四层：稀有度种数↓ → 稀有度加权和↓（`{UR:30,SSR:10,SR:5,R:3,N:1}`，有该档计一次）→ 称号加权分↓（薪王5>殘焰4>星火3>薪火2>傳火1，每称号计一次）→ 片名
  - 卡片：主题反色底、5 稀有度行×5 称号固定列×双格矩阵（左持有墨绿右挂单金，无称号组/空稀有度统一暗淡 0.28/0.22）；右上角放大镜=片名加入定向搜索（不跳 view）

### P2 — 交互与防护

- [x] **S7 市场刷新统一入口**：`triggerMarketRefresh()` 模式——所有刷新路径（按钮/切回市场/改 pageSize 走 8s 节流；买入后/保存 key 后 force 绕节流），按钮 spin 与请求生命周期同步（API 返回即停）
- [x] **S8 交易防滥用三锁**：购买流程单飞锁（确认弹窗+提交全程，防连点双买）；撤单/改价单飞锁（改价=撤+挂两步，防重复）；定向搜索 2s 节流（seq 丢弃旧结果不挡连发，防连打搜索接口）
- [x] **S9 预算池批量前置校验**：批量购买入口先查——未设预算池 toast「请先设置预算魔力池」；可用额度<所选总价 toast 区分原因（余额不足 vs 预算剩余不足）+ 金额明细，**不开模态**
- [x] **S10 运行锁防卡死**：采集轮 try/finally 兜底解锁 + 启动时重置运行状态（Docker 曾因崩溃锁卡 true，重启后市场采集永不触发）

### P3 — 产品级打磨

- [x] **S11 对比度 WCAG AA**：`--muted` 值 dark `#7b8296` / light `#5f6675`（约 5:1），`--faint` dark `#6e7689` / light `#6b7383`
- [x] **S12 错误文案行动指引**：余额不足→「到 M-TEAM 充值后重试」；预算不足→「提高总额或重置已花费」；令牌无效→「检查复制完整或到实验室重新生成」（locales 同步）
- [x] **S13 交互细节**：模态焦点管理（打开 focus 首控件、关闭归还触发元素）；toast `aria-live=polite`；请求超时（扩展侧 fetch 30s AbortController）
- [x] **S14 文案**：持有卡「锁定中」→「冷却中」（统计卡+卡片标签，中英；与手动锁定区分概念）

## 三、不适用项（勿同步）

- Docker 部署类：非 root、HEALTHCHECK、SIGTERM、compose 资源/日志限制、备份/恢复/升级文档
- ~~启动揭幕 splash~~（2026-08-19 已同步：双档时间轴同构，完整仪式档绑「首次保存令牌成功」——扩展无登录页，以 reload 重开面板替代 Docker 的登录跳转）
- 鉴权模块（AUTH_PASSWORD/HMAC——扩展由浏览器配置页权限管）
- SSE 相关（扩展无服务端推送）

## 四、同步纪律

1. **纯逻辑模块同构**：`dropStats/marketStats/portrait/shared` 同步后与 Docker 版逐函数对照一致，文件头加同源注释（`// 与 mcard(Docker) src/lib/stats.js 同源，同步自 v1.2.x`），Docker 侧对应文件也已有反向注释
2. **宿主薄适配**：只动 storage/消息/UI 入口层，不引入 Docker 宿主概念（Express/SQLite/SSE）
3. **不破坏既有功能**：扩展侧既有 dashboard 结构与交互习惯保持；同步的是「能力与规则」不是「代码原样搬」
4. **逐项 commit**：一项一 commit（信息注明 SYNC-Sx），commit/push 等用户指令
5. 卡壳就对照 Docker 源码（路径锚点见各条目），不确定的设计决策问用户

## 五、增量批次 v1.2.2 → v1.4.0（16 项，2026-08-26 由 Docker 线会话写入）

> Docker 线在 v1.2.2 后发了 v1.3.0 / v1.4.0 两个 minor。以下按价值排优先级；完成打 `[x]`，commit 注明 SYNC-S15 起。
> 参考锚点均为 Docker 线文件（`~/workspace/mcard`）。

### P1 — 核心新功能

- [ ] **S15 普通卡兑换机制符（10 换 1）**
  - Docker：`trader.js` 的 `redemption`（POST redemption/submit：cardIds 恰好 10 个不同 + recipeId 1=魔力符N/2=置顶免费符SR/3=VIP符UR）+ 双重校验（inventory 回查普通卡/稀有度匹配/未手动锁定；挂单卡不在 inventory 回查即拒；**冷却卡可兑**）+ 成功记 `redemptions` 历史
  - 前端：持有页兑换统计卡（第一行 3 配方小卡均分——可兑数量=非mech+对应稀有度+剔除手动锁定÷10；第二行「已兑换」嵌套大卡×3 紧凑小卡 ×n 计数）；兑换子模式（点小卡进入：列表强制该稀有度、其它筛选灰显称号可用、显示锁定+解锁通道保留）；自动选（无称号→傳火→薪王升序前 10）+ 手动（恰好 10 封顶、选满灰显其余 `batch-locked`）；浮动面板常驻（X/10 + 自动选 + 兑换按钮未满 10 禁用主题色高亮）；确认模态（10 卡清单白字称号 + 前端最后校验 + 单飞锁提交 + 成功 toast 获得符名 + LOAD_INVENTORY 刷新）
  - 扩展落点：background.js 加 redemption 消息处理（fetch redemption/submit + chrome.storage 校验与历史）；dashboard.js 兑换 UI（按扩展既有模态风格）
  - **注意**：批量锁定禁选逻辑要放开冷却卡（`tradeLocked` 卡兑换模式走可选分支且不灰显）
- [ ] **S16 变化角标（轻量探测）**
  - Docker：后端 `POST /api/totals`（probeTotals：5 小请求拿 total，不写 state，8s 冷却同参数复用缓存；**myorders 的 lastId/status 参数接口均忽略**——挂单=拉一页 100 条数 id>本地 max 的新增；**ordersAll 是插入序最新在尾部，max 遍历取非 [0]**）+ 前端启动探测对比本地 → 侧栏按钮红底白字胶囊角标（99+ 封顶、弹入动画）→ 进 view 清零；移动端抽屉收起时汉堡按钮红点汇总（无数字）
  - **形态注意**：曾有 applyPatch 实时 diff + 5s 闪现层，**已撤**（进 view 后采集冷却 skip 导致重复提示烦扰）——只同步探测持久角标形态
  - 扩展落点：background.js 加 probe 消息（fetch 各接口 total）；dashboard.js 角标渲染 + 进 view 清零
- [ ] **S17 定向搜索词条筛选**
  - Docker：词条结构 `{name, rarities[], titles[], maxPrices{}}`（旧字符串兼容=无筛选）；新增/编辑模态（`+` 与词条 `✎` 共用：片名 + 稀有度勾选+每档最大价 + 称号多选，不选=全要）；runSearch 逐词条查询+前端过滤（勾称号时无称号卡排除，同市场语义）+合并去重；卡册放大镜联动（片名锁定只读、稀有度默认勾该影片未拥有档、已设词条自动转编辑+图标变铅笔）；胶囊按词条名 hash 稳定随机色（深浅主题各配 8 色板，**实色彩底白字**，非色点非 tint——曾两版返工）；卡册 `_sig` 拼入词条名单（加/删词条重绘同步图标）
  - 扩展落点：dashboard.js 词条模态 + runSearch 过滤 + 胶囊配色；背景侧 searchTags 存储结构兼容旧字符串
- [ ] **S18 挂买管理补全**
  - Docker：`trader.relistBuy`（纯限价挂买：撤单后 `/api/pt-card/market/buy` 限价重挂，open 即挂上**不撤**、无预算/阈值门——BUY_CARD 是吃单语义 open 即撤+预算门，**不能用于改价重挂**，曾踩坑）；改价按 side 分流（sell=撤+挂卖净价 / buy=撤+relistBuy）；`confirmDialog` 加 `onConfirm` 模式（确认后按钮 loading 禁用 → await 网络操作 → 才关模态；busy 期间 Esc/取消/遮罩全拦）；挂单 view 挂买/挂卖互斥单选筛选按钮（排序按钮前，再点取消，清除按钮重置）
  - 扩展落点：background.js relistBuy 消息；dashboard.js 改价分流 + confirmDialog onConfirm + 筛选按钮

### P2 — 修复

- [ ] **S19 inventory 采集 >200 张漏尾**：单页 pageSize=200 上限 → 翻页拿全（`while rawItems.length < total`，页间 randSleep，上限 20 页）——变化角标恒差的根因
- [ ] **S20 runSearch 渲染竞态**：搜索慢返回时 `renderCards()` 无条件渲染会写进挂单等其它 view 的 grid——三处调用（清空/loading/结果）加 `if (view === 'market')` 守卫
- [ ] **S21 持有称号过滤对机制卡生效**：normalizeMechanism 缺 title 映射（空 title 全放行点任何称号 mech 都出现）→ 补 `title: it.title || '傳火'`（机制卡固定傳火，与交易/挂单记录接口数据一致）
- [ ] **S22 批量改价/卖出单框双态**：输入框 focus=净卖价可编辑；blur/初始/执行中=显示挂单价（净×1.05，muted 弱化 + tooltip 双价）；顶部批量设价应用后行未聚焦直接显示挂单价；数据层 netPrice 永远存净价
- [ ] **S23 兑换模式冷却卡可选**：buildInventoryCard 的 `!tradeLocked && !userLocked` 可选分支——兑换模式放开 tradeLocked（官方允许冷却卡兑换）且不灰显；`batchSelected.size===0 && !redeemMode` 才收面板（兑换模式面板常驻）

### P3 — 打磨

- [ ] **S24 侧栏版本行**：当前版本（GET /api/version）+ 最新版本（GitHub releases 实时读，失败直说「获取失败」不强求）；有新版金色高亮 + 点击新 tab 打开 release 页；版本值缓存 `_curVersion/_latestVersion` 挂 renderStatus（切语言实时切换）。扩展落点：manifest version 当当前版本 + fetch releases API（CORS 允许）
- [ ] **S25 移动端预算面板重排**（若扩展侧上轮已含预算面板）：移动端顶栏预算卡隐藏（300px 悬浮卡遮盖顶栏），改市场工具行右侧紧凑版（数字 K/M/B 紧凑格式保 2 位小数、进度条分级配色与桌面同）
- [ ] **S26 价格阈值模态移动端稀有度输入框收窄**（120px→82px 防溢出，机制卡行不动）

### 不适用项（本批新增）

- Docker 部署类：Dockerfile npmmirror、npm ci 层缓存 rm -rf /root/.npm、SIGTERM/HEALTHCHECK（已在上轮不适用清单）
- 角标实时层（5s 闪现）——Docker 线已撤，勿同步
