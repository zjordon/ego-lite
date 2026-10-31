# local-ego-daemon 逆向报告（ego lite 0.6.2.5）

> 调研日期 2026-10-31。静态逆向（REA 应用图分析，证据 `ev_c5e78eab…f5947b30`）+ 二进制字符串取证 + 被动进程/端口观测 + 授权运行时实验（本机手动拉起 daemon 并实测协议）。
> 背景：TUI browser agent 立项前调研——官方 daemon 是否可成为外部客户端的接入底座。
> 配套阅读：[ecosystem-scan.zh-CN.md](ecosystem-scan.zh-CN.md)（生态位判断，**其"空白区"结论需结合本文更新**）、[agent-tooling-model.zh-CN.md](agent-tooling-model.zh-CN.md)。

## 0. 结论先行

1. local-ego-daemon 是**官方内置的 agent 编排守护进程**——浏览器聊天 UI（"问问 ego"按钮）的后端，一个 Effect 4 编写的 Node 单体（Thread/Space/Project/Orchestration/Automation/Provider 全领域模型）
2. **对外接口已实测走通两层**：HTTP（/health、/ws/negotiate、/mcp）与 WS 升级；WS 内的 Effect-RPC 信封已破译到方法级，但合法 Request 会收到空 Close 帧（最后一公里未通）
3. **`/mcp` 是标准 MCP over JSON-RPC 活端点**（Bearer provider-session 凭据鉴权）——外部客户端（如 TUI）最有价值的接入点，剩凭据签发方式未知
4. 安全模型 = **仅回环 + Origin 白名单**，无 token（`EGO_AUTH_TOKEN` 已废弃）；运行时硬约束 **Node ≥24.21 <25**（与 ego-browser.exe 内嵌的 v24.22.1 严丝合缝）

## 1. 它是什么：浏览器聊天服务的后端

```
chrome.dll（浏览器 UI）
  ├─ 工具栏 AskEgoButton / 菜单项 kAskEgoForLite（zh-CN 显示"问问 ego"）
  │    └─ 打开 WebUI：ego://side-panel-chatbox-lite/（特权 scheme，普通标签页 ERR_UNKNOWN_URL_SCHEME）
  ├─ ego\browser\chat_service\chat_service_manager.cc
  │    └─ 从 app bundle 安装 daemon、失败请求修复、版本看护（bundle 比已跑过的旧 → 等待更新）
  └─ WebUI 资源：chatbot/ task-details/ newtab/ onboarding/ whats-new/ ...
         │
         ▼  本地回环
local-ego-daemon（Node ESM 单体，rolldown 打包，serverBuild 0.8.3）
  ├─ HTTP 127.0.0.1:3773（/health、/ws/negotiate、/mcp、/mcp/external）
  ├─ WS /ws —— @effect/rpc 4.0 私有协议（对聊天 UI）
  ├─ provider 层 —— codex(88处)/opencode/claude-code/gemini 等 agent 二进制 + openai/deepseek/xai/mistral/moonshot/qwen 模型 API
  │    （ACP 在这里：daemon 作为 ACP **客户端**驱动 provider agent，不是 ACP 服务端）
  ├─ MCP external —— 把自身 agent 工具经 /mcp 对外暴露（tools/call → invokeAgentTool，Bearer 鉴权）
  └─ 浏览器驱动 —— 串行队列 spawn `ego-browser` cli + `nodejs` 子命令（经典每轮进程模型）
```

Web 客户端是完整聊天 UI（KaTeX 公式、mermaid、几十种代码高亮语法、`BrowserWebSocketTransport`），随 bundle 附带但 daemon HTTP 不伺服它（GET / → 404）。

## 2. 静态逆向发现

### 2.1 启动与安装

- **无自启注册**（HKCU Run / 计划任务 / 服务均无）；浏览器启动也不拉起 daemon——仅在使用聊天功能时按需安装+启动
- 用户数据目录初始为空（`AppData\Local\Citro Labs\ego lite\User Data`），首次使用聊天才落 daemon
- 安装/拉起/版本看护全在 chrome.dll 的 `chat_service_manager.cc`（错误文案：`No entry script configured; the chat service is not running.` 等）

### 2.2 入口 URL（chrome.dll 提取）

| ego:// URL | 用途 |
|---|---|
| `ego://side-panel-chatbox-lite/` | 聊天侧面板 WebUI（特权 scheme，标签页打不开） |
| `ego://services/node/resources/index` | node 服务资源页（未深究） |
| `ego://welcome/` | 欢迎页 |

### 2.3 浏览器驱动方式（createEgoBrowserCli）

- 二进制解析：`EGO_BROWSER_CLI` 环境变量 → PATH 上的 `ego-browser`
- **不用 `--sdk-path`**（0 处引用），spawn 原装 CLI：`spawn(binaryPath, [...connection, "nodejs"])`，三管道 stdio
- **串行队列**（一次一个子进程，按 coalesce key 合并），输出截断上限 + 超时
- bundle 注释自述"Serial runner for `ego-browser nodejs`, the same CLI the coding agents use"，并专门适配 Codex 用 login-shell 包裹命令的行为
- 轮次模型即经典 round 模式（注释可见 `ego-browser nodejs < round-02.mjs`）——**官方产品在付每轮进程启动成本，与 jev-ego 式"loop 内嵌"路线形成对照**

### 2.4 环境变量清单（bundle 提取）

| 变量 | 作用 |
|---|---|
| `EGO_HOME` | 数据目录（SQLite 等） |
| `EGO_HOST` / `EGO_PORT` | 绑定地址/端口（默认 127.0.0.1:3773） |
| `EGO_AUTH_TOKEN` | **已废弃**——设置即拒启（见 §3） |
| `EGO_ALLOW_INSECURE_REMOTE` | 远程绑定放开（未验证） |
| `EGO_LOG_WS_EVENTS` / `EGO_LOG_PROVIDER_EVENTS` | 调试日志开关 |
| `EGO_NO_BROWSER` | 疑似不驱动浏览器模式（未验证） |
| `EGO_AGENT_GATEWAY_BOOTSTRAP_TOKEN` | 网关引导 token（MCP 凭据签发线索） |
| `EGO_GROK_ACP_DEBUG` | Grok 走 ACP 的调试开关 |
| `CODEX_HOME` / `CLAUDE_CONFIG_DIR` | codex / claude-code 集成的配置目录 |

### 2.5 WS 方法目录（WS_METHODS，节选）

`projects.*`（12）、`git.*`（24）、`pullRequests.*`（9）、`server.*`（26）、`automation.*`（12）、`provider.*`（9）、`studio.*`、`filesystem.browse`、`shell.openInEditor`、`stats.*`——**完整 IDE 级 agent 后端**。服务器能力 6 项：`orchestration.cursor-safe-streams`、`orchestration.thread-detail-snapshot`、`rpc.typed-errors`、`projects.github-provisioning`、`projects.file-watch`、`transport.http-negotiate`。

## 3. 运行时实证（本机手动拉起）

### 3.1 启动配方（已验证）

```js
// 宿主：ego-browser nodejs（内嵌 Node v24.22.1 满足版本门槛；用户自装 24.14 被拒：
// RuntimeDependencyError: engines.node ">=24.21.0 <25"）
process.env.EGO_HOME = "<可写目录>";   // 首次失败会在 HOME 留残留 → 后续 StartupError，重试前清空
delete process.env.EGO_AUTH_TOKEN;      // 残留/误设会拒启："EGO_AUTH_TOKEN is no longer supported;
                                        //  Ego only accepts loopback connections."
await import("file:///…/local-ego-daemon/index.mjs");
```

浏览器本体不是必需的（daemon 独立可跑；拉起聊天 UI 才需要浏览器）。

### 3.2 接口契约卡（全部实测）

| 项 | 实测结果 |
|---|---|
| 端口绑定 | `TCP 127.0.0.1:3773 LISTENING`（仅回环） |
| `GET /health` | 200 JSON：`startupReady/pushBusReady/keybindingsReady/terminalSubscriptionsReady/orchestrationSubscriptionsReady` + projection 子系统（序列号/投影滞后/重试） |
| `GET /ws/negotiate` | 查询参数 `x-ego-client-build`、`x-ego-protocol-epoch=1`、`x-ego-protocol-min-revision=1`、`x-ego-protocol-max-revision=1`（可重复 `x-ego-required-capability`）→ 200：`{protocolEpoch, negotiatedRevision, serverBuild:"0.8.3", serverInstanceId:<uuid>, capabilities:[6项]}`；缺参 → 426 `WS_NEGOTIATION_REQUIRED` |
| `GET /ws` 升级 | 需查询参数 **`x-ego-server-instance=<negotiate 返回的 uuid>`** + epoch/revision/clientBuild + **同源 Origin** → 101 ✓ |
| Origin 白名单 | `http://127.0.0.1:3773` ✓；`http://localhost:3773` → **403**；无 Origin → 426 `WS_SERVER_GENERATION_CHANGED`；`ego://…` → 挂断 |
| `POST /mcp`、`/mcp/external` | **MCP over JSON-RPC**：无 Authorization → 401；假 Bearer → 401 + `caller_session_inactive: Missing, revoked, or invalid provider-session credential` |
| WS 消息信封 | @effect/rpc 4.0：`{_tag:"Request"\|"Ack"\|"Interrupt"\|"Eof", …}`；Request = `{_tag:"Request", tag:"<方法>", id:<数字>, payload:{…}}` |
| 合法 Request | **空 Close 帧**（未走通，见 §4） |

### 3.3 信封破译过程（供复现参考）

1. JSON-RPC 形式 → `Unknown request tag: undefined`（顶层找 `_tag`）
2. `{_tag:"bootstrap.negotiate"}` → `Unknown request tag: bootstrap.negotiate`（协议协商已在 HTTP 升级层完成，socket 上是 feature RPC）
3. 读服务器分发代码（run 模块 44775 行起）→ 正确信封 `{_tag:"Request", tag, id, payload}`
4. 正确信封 → 空 Close（悬案）

## 4. 未解之谜（诚实清单）

1. **合法 Request → 空 Close 帧**：疑缺初始化信封或 fork 版序列化细节；NDJSON 加 `\n` 无效
2. **MCP provider-session 凭据签发**：配对流程在聊天 UI 内（`/api/mcp/external/pair` 字样与 `EGO_AGENT_GATEWAY_BOOTSTRAP_TOKEN` 是线索），未找到外部签发口
3. WebUI 真实 Origin（daemon 不伺服 client，GET / 404；同源可连但不知浏览器侧用什么）
4. 第二次启动 StartupError 的残留机制（清 EGO_HOME 即愈，机制未查）
5. `EGO_ALLOW_INSECURE_REMOTE` / `EGO_NO_BROWSER` 行为未验证

## 5. 对 TUI 项目的意义

**三条接入路线（按推荐排序）**：

| 路线 | 优点 | 风险 |
|---|---|---|
| ① MCP 端点（`/mcp`） | 标准协议；Bearer；直通 `invokeAgentTool` | 凭据签发未破；官方可能收紧 |
| ② WS RPC | 最深（全量方法目录） | 最后一公里未通；私有协议随版本变 |
| ③ 完全自建（jev-ego 路线） | 最稳；观察层增强可移植 | 全部工程量自付 |

**生态位判断更新**（修正 ecosystem-scan 的"空白区"）：官方已占"给人用的 agent 客户端"位——但形态是**浏览器内聊天面板**，且浏览器驱动仍走每轮进程模型。独立 TUI 的差异化只剩：终端形态、观察层深度（dom-snapshot 式增强）、loop 效率（内嵌 vs 每轮 spawn）。窗口判断从"以周计"改为"已经开抢"。

## 6. 复现指南

探测脚本全套：`C:\Users\ZhuanZ\ego\daemon-probe\`

| 文件 | 作用 |
|---|---|
| `run-daemon.mjs` | 用 `ego-browser nodejs` 拉起 daemon（配 EGO_HOME/日志开关） |
| `probe.mjs`–`probe10.mjs` | /health → negotiate → Origin 矩阵 → 手工 WS 升级+帧收发 → 信封迭代全记录 |
| 复跑顺序 | `run-daemon.mjs`（后台）→ `probe4/9/10.mjs`（握手）→ curl 打 `/mcp` |

静态分析复现：REA `analyze_javascript_application`（目录 `local-ego-daemon/`，summary 视图）；chrome.dll 字符串用 `LC_ALL=C grep -a`（ASCII）与 `-P '\x00'` 模式（UTF-16）。

## 相关文档

- [ecosystem-scan.zh-CN.md](ecosystem-scan.zh-CN.md)：生态扫描（其空白区结论以本文 §5 为准）
- [agent-tooling-model.zh-CN.md](agent-tooling-model.zh-CN.md)：调用模型光谱
- [user-control-transfer.zh-CN.md](user-control-transfer.zh-CN.md)：控制权协议（自建路线必读）
- [native-bindings-api.zh-CN.md](native-bindings-api.zh-CN.md)：globalThis.ego 原生 API（daemon 底下的那层）
