# ego-lite 生态扫描：谁在它上面做浏览器 agent

> 检索日期 2026-10-28，GitHub CLI（`gh search repos`）实测数据，star 数为当日快照。
> 背景：评估"TS 全栈 + TUI 客户端 + 自研 ReAct loop，动作层直接调用 ego 库"这一项目构想的前置调研。
> 配套阅读：[agent-tooling-model.zh-CN.md](agent-tooling-model.zh-CN.md)（调用模型光谱）、[user-control-transfer.zh-CN.md](user-control-transfer.zh-CN.md)（控制权机制）。

## 0. 结论先行

1. **在 ego 上做自带 loop 的浏览器 agent 这件事，2026 年 10 月前后已出现 3+ 个先行项目**，其中 `jev-ego` 与本文构想架构几乎一致（TS + loop 内嵌 ego 进程 + daemon）
2. **但"面向人的独立 TUI 产品"（Claude Code 式客户端）在 ego 生态中完全空缺**——现有全部是决策层引擎、插件或给其他 agent 调用的服务
3. 动作空间设计已有事实标准（browser-use 的 **Jev 编号元素表范式**），不建议自创
4. 窗口正热（项目全部 ≤1 个月新），格局未定

## 1. 第一梯队：上游范式源头

| 项目 | 热度 | 定位 | 与本构想的关系 |
|---|---|---|---|
| [browser-use/jev-ultrafast](https://github.com/browser-use/jev-ultrafast) | 23.8k★ | "Fastest and cheapest web agent"。**编号元素表 + 一次请求同时回答"操作 + 目标"**（TypeSafe System One / Jev 小模型）；代码负责观测/执行/校验/退出条件 | 本 niche 的动作空间范式源头，ego 系项目普遍移植它 |
| [vercel-labs/agent-browser](https://github.com/vercel-labs/agent-browser) | 44.7k★ | 独立浏览器自动化 CLI 的巨头（ego README benchmark 中的 "agent-browser" 即此） | 它是"给 agent 用的工具 CLI"，**不带自有 loop**——不同物种 |

**Jev 动作空间范式**（ego 系项目通用）：

```
page → 编号元素表                      一次 TypeSafe 请求
  [1] button  换票类型 · 往返          → operation（CLICK/TYPE_TEXT/SELECT/
  [2] combobox 出发地 · 旧金山            SCROLL/WAIT/DONE/BLOCKED）
  [3] textbox  出发日期 · 空           → target（点击哪个编号）
                                      模型输出永不变成选择器/坐标/JS
```

## 2. 第二梯队：直接在 ego 上做 agent loop（构想的直接同类）

### 2.1 romaluev/jev-ego（17★，TypeScript）——架构同款，动工前必读

- TS 浏览器 agent；jev-ultrafast 的动作空间；浏览器层是 ego lite，无 Chrome/Playwright/Browser Harness
- **"The whole loop — TypeSafe, the text helper, and CDP — runs inside one `ego-browser nodejs` process"**——loop 内嵌 ego 上下文 + `serve` 环回 daemon 常驻 TaskSpace/会话（跳过进程启动），与本系列文档推导的推荐架构完全一致
- 卖点：每快照一次浏览器调用（可见控件/名称/值/文本原子读取）；guard/几何/点击合并单次 evaluate；循环内不用截图
- 定位边界（它自己的话）：不是通用自动化 SDK，上传/标签/对话框/canvas/iframe 仍回落 ego-browser skill
- **行动项：精读 `src/agent.ts`（loop）与 `docs/design.md`（daemon）再写第一行代码**

### 2.2 jiangkoumo/ego-decision-layer（29★，中文）——工程最严谨

- 可插拔决策层：小模型/步（默认 Jev，可换本地 OpenAI 兼容后端），代码管观测/执行/陈旧校验/退出
- 能力面：结构化抽取（Schema 校验）、动作缓存（命中跳过模型请求）、批处理、Canvas/SVG 区域兜底（定位器可注入）、**Windows Git Bash/MSYS 路径兼容**
- 评测纪律范本：16 套基准、预注册配对实验（对 browser-harness+jev-ultrafast 100 轮：点击链 −378ms、wiki 搜索 −819ms）、原始数据公开、**changelog 记录被实测推翻的旧结论**
- 它的"为什么"段落精确量化了本构想要逃的坑：**每步 LLM 往返 1.6–4.5s + 每步进程启动 250–350ms**；HN 两步导航 4.9s（1 进程）vs 9.7s（3 进程经典循环）

### 2.3 其他同类

| 项目 | 热度 | 形态 |
|---|---|---|
| shikaizhong-design/ego-jev-ultrafast | 3★ | jev-ultrafast 单文件零依赖移植到 ego（证明最小可行版很小） |
| Fisfzy/dsh-ego-browser | 342★ | DSH（DeepSeek Harness）插件：ego 做成 13 个结构化 `ego_*` 工具（**工具调用模型**，与 action loop 相反）；现已改名 dsh-browser-cdp 转向真 Chrome CDP |
| hjanuschka/pi-ego | 14★ | 给 Pi coding agent 的持久会话层（截图画廊/任务空间面板/learnings 录制） |

## 3. 周边基建（可直接利用）

| 项目 | 热度 | 价值 |
|---|---|---|
| iagogfe/ego-lite-linux（及 franky5440-afk、imleon/ego-lite-bridge） | 2★ | **社区在填 Linux 空白**（Chromium supervisor + CLI shim + CDP 上的 Task Spaces）——部署面短板可能被社区解决 |
| citrolabs/ego-browser-benchmark-framework | 9★ | 官方评测框架（beta），自建 agent 的评测可以直接挂 |
| Antigravity / Agent Remote / Codex(DSH) / Go MCP 等集成插件 | 0-2★ | 各家 agent 接 ego 的桥，佐证生态位分布 |

## 4. 空白区分析（本构想的差异化）

现有 ego 系项目清一色是**引擎或插件**：决策层（给别的 agent 调）、DSH/Pi/Codex/Antigravity 插件、CLI daemon。三个无人占据的位置：

1. **面向人的独立 TUI 产品**（Claude Code 式客户端）——完全空缺。jev-ego 的 daemon 本质是"给其他 agent 调的服务"；做"给人用的客户端"是不同产品
2. **观察层深度**——没有任何一家移植 dom-snapshot 级 DOM 观察（jev-ego 元素表是 Runtime.evaluate 现做的）；本仓库 `snapshot-pipeline` / `snapshot-comparison` 两文档沉淀的增强方案（file input 注入、状态标记、遮挡标记）是独家资产
3. **code-gen 模式**——现有全走动作空间路线；ego 原生的"整程序轮次"无人使用。loop 可两态支持（动作空间走 Jev 范式 + code-gen 走 ego 原生优势）

## 5. 风险与时机

- **撞车风险**：jev-ego 同语言同架构（17★ = 验证了路但没占住场）；ego-decision-layer 迭代快（v0.13.0）且同在中文圈，但它是决策层不是产品
- **上游依赖**：全部项目绑闭源 ego lite app（版本迁移坑已有实证：0.6.1.11→0.6.1.13 路径变更、符号链接未刷新）
- **时机信号**：同类项目全部出现于 2026-09/10——窗口期以周计

## 6. 行动建议

1. **动工前**：精读 jev-ego 源码（避免重复造已验证的轮子）+ ego-decision-layer 的评测方法（预注册配对实验可直接抄）
2. **动作空间**：继承 Jev 范式（编号表 + 一次请求定操作与目标），不自创
3. **评测**：挂官方 benchmark framework；把本仓库调试沉淀的踩坑清单（快照盲区、`EGO_TASK_SPACE_USER_IN_CONTROL` 协议、路径/版本坑）作为 loop 的 prompt 资产
4. **切角**：TUI 产品体验 + 观察层增强（fork 的 P0/P1）双轮驱动，避开与决策层项目的正面竞争

## 相关文档

- [agent-tooling-model.zh-CN.md](agent-tooling-model.zh-CN.md)：工具调用 vs 代码执行 vs 薄 harness 的三项目光谱
- [user-control-transfer.zh-CN.md](user-control-transfer.zh-CN.md)：自建 loop 必须处理的控制权协议
- [snapshot-pipeline.zh-CN.md](snapshot-pipeline.zh-CN.md) / [snapshot-comparison-dom-snapshot.zh-CN.md](snapshot-comparison-dom-snapshot.zh-CN.md)：观察层的现状与增强路线
