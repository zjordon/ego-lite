# ego lite 与同类项目对比（使用者视角）

> 整理日期 2026-11-01。观点源自实际使用体验（购物比价、抖音视频上传、快照盲区调试、local-ego-daemon 逆向等任务），论证部分基于本仓库 docs/ 系列的源码分析与实测数据。
> 与 [agent-tooling-model.zh-CN.md](agent-tooling-model.zh-CN.md) 的分工：那篇是调用模型的**技术分析**，本篇是**六维对比 + 使用者优劣总结**。

## 0. 结论先行

ego lite 的护城河不在任何单项功能，而在**四个物种级选择的组合**：代码执行调用模型 × 引擎侧精简观察 × 复用宿主 agent 智力 × 引擎级人机控制权。前三条各有代价，第四条是其它物种**架构上无法复制**的。唯一硬伤（每轮进程启动）数量级上小于它买断的东西（LLM 往返）。

## 1. 对比坐标系：四类项目

| 类型 | 代表 | 形态 | 智能来源 |
|---|---|---|---|
| **工具提供型** | chrome-devtools-mcp（55 个离散工具）、vercel agent-browser | 常驻 server/CLI，把浏览器能力拆成工具给 agent 调 | 宿主 agent |
| **薄 harness 型** | browser-harness | 尽量做薄的执行层，截图优先 + agent 现场写 Python | 宿主 agent |
| **自带 loop 型** | browser-use（23.8k★） | 自实现 agent 循环 + 动作空间 | loop 中配置的 LLM（模型可自由换装） |
| **代码执行型** | **ego lite** | 每轮一个 JS 脚本，整程序进浏览器进程 | 宿主 agent |

## 2. 六维对比总表

| 维度 | ego lite | 工具提供型 | 薄 harness | 自带 loop |
|---|---|---|---|---|
| 调用粒度 | 多步封装一个脚本，一次执行 | 每个动作一次调用 | 脚本单位 | 每步一个动作 |
| 控制流归属 | 脚本内确定性代码 | LLM（每步往返） | 脚本内 | LLM（自有循环） |
| 观察层 | 引擎侧 AXTree 精简投影 | CDP 原始 DOM/AX 采集后序列化精简 | 截图优先，现场写 Python 过滤 | 原始 DOM 多源采集 + 多步过滤后编号（browser-use：三源采集、五步过滤） |
| token 经济学 | 省往返 + 省快照 | 每步一往返，观察靠后处理 | 过滤成本转嫁 LLM，截图贵 | 动作空间自有优化 |
| 进程模型 | 每轮 spawn ~300ms | 常驻，零 spawn | 常驻/低频 | 常驻 loop |
| 人机控制权 | **引擎级，四要素俱全** | 弱概念：有弹窗类 CDP 接口（handle_dialog），让权靠 LLM 自觉 | 无（仅有"如何操作弹窗"的 UI 技法） | 页面级检测，够不到浏览器 chrome |

## 3. 逐条展开

### 3.1 多次工具调用 → 一个脚本（vs 工具提供型）

省的不只是调用次数，是**把控制流从 LLM 手里搬进确定性代码**：重试、轮询、条件分支在脚本内消化，LLM 只负责"写脚本"和"读结果"两个往返。实测任务中 24 个轮次脚本（r1-r24）就是这么跑的。

**代价**：脚本由 LLM 生成，失败时调试单位变大——一轮废稿重写 vs 单步重试。封面折腾正是脚本生成质量问题被放大的例子。

### 3.2 引擎侧精简快照（vs 薄 harness）

真正差异不是"精简 vs 全部"（其它三类也都在做精简——工具提供型对 CDP 原始 DOM 序列化，browser-use 做三源采集五步过滤），而是**精简发生在哪一层**：ego 在引擎侧做 AXTree 投影（display:none 不进树、容器折叠、content/refs 双通道），是"编译期优化"；其它三类都是客户端拿到原始数据后再处理，或干脆把过滤成本转嫁给 LLM 现场写代码。

**代价**：省 token 和丢信息是同一枚硬币——隐藏的 file input、canvas 语义、交互状态会被精简掉（douyin 封面上传失败的根因）。详见 [snapshot-pipeline](snapshot-pipeline.zh-CN.md)、[snapshot-comparison](snapshot-comparison-dom-snapshot.zh-CN.md)。一句话：**引擎认为不重要的信息，agent 永远看不见**。

### 3.3 与宿主 agent 深度结合（vs 自带 loop 型）

"任务通常和日常工作混在一起，很少有纯浏览器事项"——这是比多数 browser agent 立项理由更清醒的观察。browser-use 也能被当作工具调用，但它要在自己的 loop 里另配一个 LLM（模型虽可换装，终究是第二个大脑）：两份上下文、两份计划、两次理解。ego 是**一个大脑干所有事**：文件、代码、浏览器共享同一个计划与记忆。

**代价**：宿主上下文窗口是共享资源，浏览器子任务的巨型快照会污染主对话（实操中靠 .mjs 文件 + cliLog 落盘规避）。另注：官方 daemon 自己也在长出 orchestration/loop——厂商自己也承认纯 harness 不够，市场在要"两者都要"（见 [local-ego-daemon-reverse](local-ego-daemon-reverse.zh-CN.md)）。

### 3.4 每轮进程启动（唯一硬伤，但账要算对）

实测（ego-decision-layer 数据）：每轮进程启动 **250-350ms**，每步 LLM 往返 **1.6-4.5s**——进程开销低一个数量级，所以 ego 敢用这个模型：

| | 工具提供型 | ego |
|---|---|---|
| 进程 | 常驻，零 spawn | 每轮 ~300ms |
| LLM 往返 | 每个动作一次 | 整个脚本一次 |
| **合计** | spawn 免费，LLM 大付 | spawn 付费，LLM 大省 |

净赚成立的前提是"脚本内多步"；密集短交互场景会亏（此时可走 loop 内嵌路线：一个 `ego-browser nodejs` 进程常驻、spawn 归零，jev-ego 与 90s 生存实验均已验证）。连官方 daemon 也在付同样的每轮 spawn 成本（createEgoBrowserCli 串行队列）。

### 3.5 人机控制权切换（结构性优势，其它项目难以复制）

这不是"做得好"，是**架构上只有浏览器宿主能做到**。自然切换要求四件事同时成立：

1. 人和 agent **共用同一个浏览器**（同窗口同 profile 同标签页，人在场是物理事实）
2. **引擎级意图检测**（权限弹窗、通知、手动接管 → reason key）
3. **agent 绕不过的信任边界**（原生桥逐调用检查所有权，`EGO_TASK_SPACE_USER_IN_CONTROL`）
4. **人永远知道谁在开车**（浏览器 UI 实时显示控制状态）

| 对比对象 | 缺哪块 |
|---|---|
| 工具提供型（CDP 连现有 Chrome） | 有弹窗/等待类接口（handle_dialog、wait_for），但"用户 vs agent"只存在于 LLM 的自觉里：CDP 是调试协议不是信任边界，浏览器不执法、不显示控制状态 |
| 薄 harness / Playwright 系 | 无控制权语义（文档里 dialogs 只是"怎么点弹窗"的 UI 技法）；页面内 JS 够不到浏览器 chrome 层；自动化实例通常不是人日常用的窗口 |
| computer use / Operator | 移交靠人在聊天框打断，agent 厂商中介，非环境语义 |

**代价**：对人是自然，对 agent 是粗暴——硬停止时本轮输出直接丢弃，loop 必须按协议设计（probe → 等待 → claim，不可重试）；beforeunload 归属仍是已知缺口。机制全解见 [user-control-transfer](user-control-transfer.zh-CN.md)。

## 4. 隐性成本清单（选型时一并计入）

- 脚本调试单位大（3.1）
- 精简即丢信息，快照盲区需自建观察层补（3.2）
- 宿主上下文污染（3.3）
- 硬停止对 loop 的协议要求（3.5）
- **闭源绑定**：引擎不可审计，版本自动迁移会破坏脚本路径（0.6.1.11→13→0.6.2.5 三次实测）；Linux 依赖社区复刻

## 5. 选型速查

| 场景 | 推荐 |
|---|---|
| 与编码/文件操作混合的浏览器任务 | ego lite（3.1+3.3） |
| 纯浏览器的批量自动化、给已有流水线插浏览器环节 | 自带 loop 型或工具提供型（无宿主 agent 时） |
| 需要人全程盯、随时接手的任务 | ego lite（3.5 独有） |
| 页面语义复杂、隐藏交互多（上传类） | ego + 自建观察层增强（补 3.2） |
| 极低延迟密集交互 | loop 内嵌 ego（3.4 的解法） |

## 相关文档

- [agent-tooling-model.zh-CN.md](agent-tooling-model.zh-CN.md)：调用模型光谱的技术分析（本篇的深度版）
- [user-control-transfer.zh-CN.md](user-control-transfer.zh-CN.md)：控制权机制全解
- [snapshot-pipeline.zh-CN.md](snapshot-pipeline.zh-CN.md) / [snapshot-comparison-dom-snapshot.zh-CN.md](snapshot-comparison-dom-snapshot.zh-CN.md)：观察层
- [ecosystem-scan.zh-CN.md](ecosystem-scan.zh-CN.md) / [local-ego-daemon-reverse.zh-CN.md](local-ego-daemon-reverse.zh-CN.md)：生态与官方动向
