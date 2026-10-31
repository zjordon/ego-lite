# 原生 JavaScript 对话框需求

> 中文翻译版 · 英文原文：[native-javascript-dialog-requirement.md](native-javascript-dialog-requirement.md)

Ego Lite 0.5.0.12 将页面自身弹出的 `alert`、`confirm`、`prompt` 对话框保持在
Agent 控制之下，包括由文件输入 change 处理器打开的 confirm。真实浏览器回归
测试覆盖这些路径，且全程无需接管或恢复步骤。

`beforeunload` 仍是未决项。在 0.5.0.12 中，即便经过可信点击与粘性用户激活
（sticky user activation），导航也会在没有对话框事件的情况下直接继续。期望的
行为是让该对话框同样处于 Agent 控制之下，并经由同一个 Page API 上报。

任何页面自身弹出的对话框都不得改变 TaskSpace 的所有权，也不得以
`fallback_site_dialog_required_notice` 上报
`EGO_TASK_SPACE_USER_IN_CONTROL`。

不需要新的绑定。既有的 CDP 桥必须：

- 在 Page 会话上转发 `Page.javascriptDialogOpening` 与
  `Page.javascriptDialogClosed`；
- 在对话框打开期间于该会话上接受 `Page.handleJavaScriptDialog`，即使触发
  它的命令仍未返回；
- 保留对话框的类型、消息、URL、prompt 默认值以及传入的 `promptText`。

权限提示与设备选择器必须继续把控制权移交给用户。

一个对话框可以比打开它的那次 `ego-browser nodejs` 调用活得更久。SDK 按进程
保存对话框状态；而对话框打开期间，Chromium 不会应答渲染器命令——包括
`Page.enable`。因此下一次调用无法发现这个对话框，它发出的每条命令都会
超时。Ego Lite 0.5.2.2 尚未满足本要求：本地运行中，调用退出时对话框被直接
关闭，有时 TaskSpace 还被切到用户控制；线上环境则对话框保持打开，下一次
调用超时。当某次调用在对话框打开的状态下退出时，桥必须：

- 保持对话框打开，且 TaskSpace 仍归 Agent 所有；
- 向下一个附着到该 Page 的会话上报这个打开中的对话框，例如在
  `Page.enable` 之后发出 `Page.javascriptDialogOpening`。

验收：通过真实的 Page 点击触发每一种 JavaScript 对话框，验证 TaskSpace 保持
Agent 所有；以 `promptText: "agent"` 接受一个 prompt；关闭一个 confirm；
接受一个 alert；并验证返回值以及打开/关闭事件。还要在一个被拦截的文件输入
上设置文件——其 change 处理器会打开一个 confirm——然后在
`DOM.setFileInputFiles` 尚未返回时接受它。对 `beforeunload`，验证打开事件
与动作回执先于"是否允许导航继续"的选择到达。另外单独验证定位、摄像头与
设备提示仍会把控制权移交给用户。
