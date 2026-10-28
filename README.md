> 在线阅读（GitHub Pages）：https://maynor996.github.io/codex-reset-monitor/

# 在 codexinstall.asia 做 Codex 额度重置监控：公开推文 + 启发式窗口 + Workers 注入

> 个人工程笔记。记录我们如何基于 @thsottiaux 的公开推文，整理 Codex / ChatGPT 额度重置事件，并在 Cloudflare Workers 上挂一个可对照的日历页。数据非官方、不接 OpenAI 私有 API；文中链接仅作对照，不当购买入口。撰写时（2026-10-25）仓库与线上页一致。

## 一、引子：额度何时重置，大家常盯 @thsottiaux

用 Codex 写代码时，额度用尽是常态。官方仪表盘能看到「还剩多少」，却很难说清「下一次全员重置大概落在哪一天」。社区里大家习惯去刷 **Tibo（@thsottiaux）** 的公开帖——他常会发「Day N / Reset」「all propagated」「It's done」一类信号。

问题是：推文时间线散、时区绕（常见 PST / UTC 表述，国内习惯对照北京时间）、有时先宣布再确认，人工刷容易漏。我们想要的是：

1. 把确认过的重置事件落成结构化数据；
2. 用很朴素的启发式估一个「下次窗口」；
3. 页面上能一眼看到日历、最近确认帖，以及「刚重置不久 / 接近均值」的状态。

于是在 **codex中文网**（主站安装/指南类内容）旁边，加了一条独立路径：`/zh/codex-reset/`。目标不是做第二个状态中心，而是把「盯梢公开信号」这件事工程化一点。

![重置监控页](imgs/01.png)
*图 1：重置监控页——最近确认、启发式预测、十月日历与公开帖流*

## 二、做了什么：监控页长什么样

页面职责很窄，刻意不做「官方状态页」包装：

- **最近一次确认**：显示距今多久、北京时间戳、是否全员重置，并链回确认帖；
- **启发式预测（未来 24h）**：基于历史确认间隔的均值，估一个窗口与粗概率；
- **月历**：把 `confirmed` 事件标在日期上，可翻月；
- **公开帖流**：同一天内 announce / follow-up 按时间展开，可跳到 X 原文；
- **页脚声明**：基于 @thsottiaux 公开推文整理；**非官方**，不代表 OpenAI。

以撰写当天为例：最新一条是 **Day 20 / Rest and reset**。约 `2026-10-25T03:39Z` 宣布（含「By midnight PST」跟帖），`2026-10-25T06:06:16Z`（推文 `2114237083652882932`）确认已 propagated。再往前还有 Day 18、Day 17、Day 15、Day 13 等，都写在同一份事件表里。

预测侧当前均值大约 **68 小时**量级（随样本变化），刚重置后「未来 24h 内再重置」的启发式概率会压得很低——这符合直觉，也提醒：**数字只是统计投影，不是承诺。** 页面上的「预测窗口」通常取「上次确认 + 均值」附近一段（略提前几小时、略延后几小时），方便对照，而不是精确到分钟的闹钟。

## 三、数据怎么来：events.json，而不是 Worker 里调 X

事件源是仓库里的 `data/events.json`，构建时内嵌进静态 HTML。大致形状：

```json
{
  "updatedAt": "2026-10-25T08:40:00+08:00",
  "sourceNote": "基于 @thsottiaux 公开推文整理；非官方，不代表 OpenAI。",
  "resets": [
    {
      "id": "2114237083652882932",
      "confirmedAt": "2026-10-25T06:06:16Z",
      "announcedAt": "2026-10-25T03:39:31Z",
      "label": "Day 20 / Rest and reset",
      "status": "confirmed",
      "scope": "global",
      "posts": [
        { "id": "...", "at": "...", "text": "...", "url": "https://x.com/thsottiaux/status/..." }
      ]
    }
  ]
}
```

字段约定（实践中摸出来的）：

- `announcedAt`：第一次明确宣布的帖时间；
- `confirmedAt`：propagated / done 类确认帖时间；没有单独确认帖时，可与宣布时间相同；
- `id`：优先用确认帖 id，便于去重与外链；
- `posts[]`：同一轮重置的相关公开帖原文摘要与 URL。

同步流水线刻意放在 **Grok Bot 定时例程**（约每 4 小时），而不是 Cloudflare Worker cron：

1. 用公开 X 读取拉 @thsottiaux 近期帖；
2. 只收录明确带 reset / propagated / Day N + reset 等信号的帖，不臆造，忽略无关回复与转推；
3. 合并 announce + 确认帖到同一 `resets[]` 条目；
4. `python3 scripts/build_page.py` 重生成 `public/**/index.html`；
5. `npm run deploy` 推到 Worker `codexinstall-gateway`；
6. 若无新增/变更，静默跳过，避免空部署刷屏。

**为什么不把 X token 塞进 Worker？** 这页只需要公开信息；Worker 侧也不该为「刷时间线」持有 bearer。例程改 JSON → 构建 → 部署，边界更干净：密钥不进边缘、静态页可回滚、离线打开仍能看到内嵌的最近一批事件。

前端启发式也很直白：对已确认时间排序，算相邻间隔均值；窗口取「上次 + 均值」附近一段；「未来 24h 概率」用「已过时间 / 均值」做一个偏 logistic 的映射。样本不足时默认均值回退约 48h。代码就在页面内联脚本里，方便对照，也方便以后换成更稳的模型而不改数据层。

## 四、工程坑：TanStack Start 水合会清掉 SSR 注入

主站本体是 **TanStack Start（React）**，部署在原 Worker `codexinstall`。我们另起 Gateway Worker `codexinstall-gateway`：

- `/zh/codex-reset`、`/codex-reset` 由 Gateway 自己的 ASSETS 托管；
- 其余流量 **service binding** 到 `codexinstall`；
- 仅对首页等少数 HTML 路径做注入（避免污染文档子页）。

最初用 `HTMLRewriter` 往 `nav[aria-label="主导航"]`、hero `.cd-actions`、`body` 里塞「重置监控」链接和右侧固定侧栏 `#gw-side-rail`。SSR HTML 看起来没问题，但 React 水合一跑，**不在虚拟 DOM 里的节点会被拆掉**——注入「闪一下又没了」。这是边缘注入遇上 CSR/水合框架的经典坑：Rewriter 改的是响应字节流，框架认的是自己的树。

解决办法是双保险：

1. **SSR 注入仍保留**：首屏与无 JS 场景能看到；
2. **client bootstrap**：在 `body` 末尾再塞一段带 `data-gw` 标记的脚本，在 `DOMContentLoaded` / `load`、若干延迟、`MutationObserver`、甚至低频 `setInterval` 上幂等重挂节点（用 `data-gw="gw-reset"` 等选择器判重）。

侧栏本身是右侧固定竖向胶囊：`position: fixed; writing-mode: vertical-rl`，移动端主站导航常被隐藏，侧栏成了稳定入口。样式与节点都带 `data-gw`，重复注入时先查再挂，避免 Gateway 多层代理时叠两份。

![首页注入与右侧侧栏](imgs/02.png)
*图 2：首页导航 / hero 按钮与右侧固定侧栏中的「重置监控」入口（Gateway 注入）*

![站点图标](imgs/03.png)
*图 3：codex中文网图标（目录投稿可用）*

## 五、自己怎么对照

需要看日历或最近确认帖时，直接打开：

[https://codexinstall.asia/zh/codex-reset/](https://codexinstall.asia/zh/codex-reset/)

主站仍是安装与指南向内容；若从首页侧栏看到「充值」外链，那是另一条业务入口，与本文监控逻辑无关，这里不展开。

## 六、免责与边界

1. **预测仅供参考**：均值间隔会随样本漂移，官方节奏也可能突变；
2. **不抓私有 API / 账号额度接口**，只整理公开推文；
3. **非 OpenAI 官方页面**，页面上的 `sourceNote` 已写明；
4. 推文措辞有时暧昧（玩笑、延期、局部范围），同步规则宁可漏、不要瞎补。

## 七、小结

这是一次很「窄」的工程实践：**公开信号 → 结构化事件 → 静态内嵌页 → Gateway 挂载与水合友好的导航注入**。没有神秘黑盒，也没有官方背书。若你也在盯 Codex 额度节奏，可以把该页当对照板；若自己搭类似「社区公开时间线 → 日历」小工具，Workers + 站外例程同步 + `data-*` 幂等重挂，是我们踩坑后比较稳的组合。

*（个人实践分享，非官方内容；事件与预测以页面实时数据为准。）*