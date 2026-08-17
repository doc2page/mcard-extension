# MCard 扩展线 ← Docker 线 同步更新实施细则

> **双线关系**：本仓库（扩展 MV3）是产品原始形态；`~/workspace/mcard`（Docker Web，v1.2.2）2026-08 从扩展分叉重构。分叉后 Docker 线演进三个月（v1.0.0→v1.2.2 + 分叉前重构期优化），本文件是把这些演进**同步更新**回扩展线的完整细则。
> **参考源**：`~/workspace/mcard`（其本地根 CLAUDE.md 有全部架构约定，可读）。
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

- [ ] **S1 掉落统计：手动导入补全历史**
  - Docker：`collector.js` 的 `importDropMessages`（解析粘贴的 message search JSON → messages 全量合并，按 id 去重、只留能 parseDropContext 的、msgTotal 记录接口 total）+ `POST /api/drop-import` 入口 + 前端导入模态（粘贴 textarea → 导入 → 拉最新校验）
  - 补全判断用 `msgTotal`：`!msgTotal`（从未导入）或 `messages.length < msgTotal`（已导 < 接口总数）才显示导入 CTA。**勿用 rangeStart 判断**（不可靠，Docker 踩过坑）
  - 扩展落点：background.js 加消息处理 + chrome.storage 持久化；dashboard.js 加导入入口与模态
- [ ] **S2 掉落双源去重不变量**
  - `feedCards` 只存 `createdDate > lastMsgDate`（messages 未覆盖的增量），三处维护此不变量：mergeDropFeed / importDropMessages / 存储加载。违反则双源重叠重复计数（Docker 修过）
- [x] **S3 掉落 dailyFull 日界**
  - `computeDropSummary` 的 since/refDay 起止必须截到日界 `slice(0,10)`（带时分会让日循环每天偏移、当天柱子被裁）
  - TZ 说明：Docker 版因容器 UTC 踩坑加了 TZ env；扩展跑在浏览器=用户本地时区，**天然免疫**，核对 `_todayStr` 等价物用的是本地日期即可，无需照搬
- [ ] **S4 市场自挂单防自买**
  - 同款（filmId+rarity+provenance）同价 = 自己的挂单：单卡购买按钮 `disabled` 灰显 + title「这是你自己的挂单」；**批量模式也要拦截**（点选 toast + return + 隐藏勾选框），Docker 曾只灰显单卡漏了批量（v1.0.1 修复）
  - myAsks Set 构建：挂单列表过滤 `open+sell`，key=`filmId|rarity|provenance|price`
- [ ] **S5 市场分析报告**
  - Docker：`marketStats.js` 的 `analyzeMarket(sum)` 返回语言无关 tier key（`expand/sell/divergence/cool/flat/nodata`），文案走 i18n `report.*`；HTML 报告前端生成（单文件、新 tab、可打印）；**排除当天**（基准日=昨天）；机制卡统一计入总 GMV 但稀有度筛选排除（归 mechMatrix）
  - `_priceHistogram` 返回桶边界（X 轴/tooltip 显示价格区间+占比）
- [ ] **S6 我的卡册 view**
  - `computeCardBook`：持有+挂单(open sell)按 filmId 聚合、排除机制卡；排序四层：稀有度种数↓ → 稀有度加权和↓（`{UR:30,SSR:10,SR:5,R:3,N:1}`，有该档计一次）→ 称号加权分↓（薪王5>殘焰4>星火3>薪火2>傳火1，每称号计一次）→ 片名
  - 卡片：主题反色底、5 稀有度行×5 称号固定列×双格矩阵（左持有墨绿右挂单金，无称号组/空稀有度统一暗淡 0.28/0.22）；右上角放大镜=片名加入定向搜索（不跳 view）

### P2 — 交互与防护

- [ ] **S7 市场刷新统一入口**：`triggerMarketRefresh()` 模式——所有刷新路径（按钮/切回市场/改 pageSize 走 8s 节流；买入后/保存 key 后 force 绕节流），按钮 spin 与请求生命周期同步（API 返回即停）
- [ ] **S8 交易防滥用三锁**：购买流程单飞锁（确认弹窗+提交全程，防连点双买）；撤单/改价单飞锁（改价=撤+挂两步，防重复）；定向搜索 2s 节流（seq 丢弃旧结果不挡连发，防连打搜索接口）
- [ ] **S9 预算池批量前置校验**：批量购买入口先查——未设预算池 toast「请先设置预算魔力池」；可用额度<所选总价 toast 区分原因（余额不足 vs 预算剩余不足）+ 金额明细，**不开模态**
- [ ] **S10 运行锁防卡死**：采集轮 try/finally 兜底解锁 + 启动时重置运行状态（Docker 曾因崩溃锁卡 true，重启后市场采集永不触发）

### P3 — 产品级打磨

- [ ] **S11 对比度 WCAG AA**：`--muted` 值 dark `#7b8296` / light `#5f6675`（约 5:1），`--faint` dark `#6e7689` / light `#6b7383`
- [ ] **S12 错误文案行动指引**：余额不足→「到 M-TEAM 充值后重试」；预算不足→「提高总额或重置已花费」；令牌无效→「检查复制完整或到实验室重新生成」（locales 同步）
- [ ] **S13 交互细节**：模态焦点管理（打开 focus 首控件、关闭归还触发元素）；toast `aria-live=polite`；请求超时（扩展侧 fetch 30s AbortController）
- [ ] **S14 文案**：持有卡「锁定中」→「冷却中」（统计卡+卡片标签，中英；与手动锁定区分概念）

## 三、不适用项（勿同步）

- Docker 部署类：非 root、HEALTHCHECK、SIGTERM、compose 资源/日志限制、备份/恢复/升级文档
- 启动揭幕 splash（Web 打开仪式，扩展无此场景）
- 鉴权模块（AUTH_PASSWORD/HMAC——扩展由浏览器配置页权限管）
- SSE 相关（扩展无服务端推送）

## 四、同步纪律

1. **纯逻辑模块同构**：`dropStats/marketStats/portrait/shared` 同步后与 Docker 版逐函数对照一致，文件头加同源注释（`// 与 mcard(Docker) src/lib/stats.js 同源，同步自 v1.2.x`），Docker 侧对应文件也已有反向注释
2. **宿主薄适配**：只动 storage/消息/UI 入口层，不引入 Docker 宿主概念（Express/SQLite/SSE）
3. **不破坏既有功能**：扩展侧既有 dashboard 结构与交互习惯保持；同步的是「能力与规则」不是「代码原样搬」
4. **逐项 commit**：一项一 commit（信息注明 SYNC-Sx），commit/push 等用户指令
5. 卡壳就对照 Docker 源码（路径锚点见各条目），不确定的设计决策问用户
