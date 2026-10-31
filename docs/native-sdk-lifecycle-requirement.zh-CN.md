# 嵌入式 SDK 宿主要求

> 中文翻译版 · 英文原文：[native-sdk-lifecycle-requirement.md](native-sdk-lifecycle-requirement.md)

## 销毁

当 Ego Lite 卸载或替换一个嵌入式 ego-browser Node 上下文时，应在销毁该
上下文之前调用并等待模块导出的 `disposeEgoSdk()` 函数。

该函数会让未完成的 CDP 请求 reject，清空 Page/会话状态，并且只在
`onCDPMessage` 与 `onSendCDPMessageError` 仍由该 SDK 实例持有时才移除它们。
这防止原生代码保留指向已被丢弃的 JavaScript 上下文的回调。

不需要新的绑定方法。宿主只需遵守这个模块生命周期钩子。没有
`disposeEgoSdk` 的旧版 SDK 构建可以照旧卸载。

验收：加载 SDK，完成一条 CDP 命令，留下另一条命令未决，调用
`disposeEgoSdk()`，然后验证在 Node 上下文被销毁之前，未决命令已经
reject、两个原生回调槽位均已清空。

## 语法诊断

`ego-browser nodejs` 宿主会在嵌入式 SDK 检查之前先编译 heredoc。编译失败
时，应报告错误消息、从 1 计数的行号与列号，以及对应的源码行。直接的
JavaScript 入口已经提供这一诊断，但一旦宿主把失败归约为泛化的
`node:internal/vm` 栈，它就无法恢复源码位置。

不需要新的 Agent API。宿主可以把未编译的源码传给 SDK 执行器，也可以自行
格式化 V8 的编译错误。

验收：运行一个第 4 行含非法 token 的 heredoc，验证 stderr 指明第 4 行、
指向该 token，并打印该源码行。
