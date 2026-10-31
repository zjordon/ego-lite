# Ego Lite 原生绑定 API 参考

> 中文翻译版 · 英文原文：[native-bindings-api.md](native-bindings-api.md)

本文档描述由已安装的 Ego Lite 应用注入到 `ego-browser nodejs` 环境中的底层
`globalThis.ego` API。

这些绑定由 Ego Lite 提供，并非本仓库实现。其可用性由已安装的 Ego Lite 构建
版本决定，与构建本地 `ego-browser` 运行时所用的 Git 标签或分支无关。

本仓库的运行时消费这些绑定，并可能只通过更高层的 helpers 暴露其中一个子集。
Agent 自动化通常应遵循 `ego-browser skill` 打印的使用指南；运行时维护者在
涉及原生桥的工作中可以直接访问 `globalThis.ego`。

## 运行时模型

多数方法返回 `Promise`，并把工作路由给浏览器侧的 `NodeClient`。Promise 的
结果由浏览器侧的 `base::Value` 对象转换而来，因此结果的形状取决于处理该请求
的命令实现。

任务域 API 作用于当前任务上下文。在脚本内用 `ego.useTaskSpace(id)` 选择该
上下文。`id` 是由 `ego.createTaskSpace()`、`ego.claimTaskSpace()` 或
`ego.listTaskSpaces()` 返回的数字型任务空间 id。选择任务只更新当前这次 Node
调用；它不会创建、认领或校验浏览器侧的任务空间。

Profile 在创建任务空间时选定，而不是通过 `ego.useTaskSpace()`。调用
`ego.listProfiles()` 发现可用 Profile，再把所选 Profile 的 `id` 传给
`ego.createTaskSpace(name, profileId)`。既有的任务域 API 继续在拥有所选任务
空间的那个 Profile 上运作。

任务空间结果对象目前同时暴露数字型定位符和用户可见名称：

```js
{
  id: 7,                    // 数字型唯一任务空间定位符
  taskId: 'checkout-flow',  // 与 name 同值
  name: 'checkout-flow',    // 用户可见的显示名
}
```

调用作用于既有任务空间的 API 时请使用 `id`。`taskId` 是显示名兼容字段，
不应当作定位符使用。

没有可用任务上下文时，任务域 API 依底层操作不同，或 resolve 一个错误对象，
或 reject。常规任务命令通常 resolve 形如
`{ error: "Task space not selected", error_code: "EGO_TASK_SPACE_NOT_SELECTED" }`
的对象；`ego.snapshot()` 则以带 `error_code` 属性的 `Error` reject。

当任务已移交给用户时，会修改浏览器状态的自动化调用会被阻塞，直到 agent
把任务取回。受影响的 API 有 `ego.createTab()`、`ego.listTabs()`、
`ego.snapshot()`、`ego.closeTaskSpace()`、`ego.completeTaskSpace()`、
`ego.animationHighlightMouseToPosition()`、`ego.setAgentTaskState()`，以及经
`ego.sendCDPMessage()` 的裸 CDP 发送。这些 API 多数以带 `error` 和
`error_code` 字段的错误对象 resolve；`ego.snapshot()` 以含 `error_code` 的
`Error` reject；裸 CDP 发送通过 `ego.onSendCDPMessageError` 报告失败。

## 错误码

浏览器侧的失败除人类可读的消息外，还会暴露稳定的 `error_code`。返回
Promise 的 API 使用以下两种形状之一：

```js
{ error: 'Task space not selected', error_code: 'EGO_TASK_SPACE_NOT_SELECTED' }
```

或者，对会 reject 的操作：

```js
try {
  await ego.snapshot();
} catch (error) {
  console.log(error.message);
  console.log(error.error_code);
}
```

参数校验失败抛出带 `error_code: 'EGO_INVALID_ARGUMENT'` 的 `TypeError`。

已知的错误码：

```text
EGO_BROWSER_UNAVAILABLE
EGO_CDP_CHANNEL_UNAVAILABLE
EGO_CDP_SEND_FAILED
EGO_INVALID_ARGUMENT
EGO_INVALID_RESULT_PAYLOAD
EGO_OPERATION_FAILED
EGO_PROFILE_NOT_FOUND
EGO_RESULT_CONVERSION_FAILED
EGO_SNAPSHOT_FAILED
EGO_TASK_HOST_DISCONNECTED
EGO_TASK_SPACE_INACTIVE
EGO_TASK_SPACE_NOT_FOUND
EGO_TASK_SPACE_NOT_SELECTED
EGO_TASK_SPACE_UNAVAILABLE
EGO_TASK_SPACE_USER_IN_CONTROL
EGO_WEB_CONTENTS_UNAVAILABLE
```

所有异步回调在调用 JS 之前都会先经 Node 事件循环中转。脚本可以放心地
`await` 这些 API，并允许进程在未决请求落定之后退出。

## 浏览器版本

### `await ego.getBrowserVersion()`

返回已安装的 Ego Lite 版本，以及其更新器是否有可用的新版本。该方法不属于
任务域，也不会启动更新。

```js
{
  currentVersion: '0.5.0.5',
  updateAvailable: false,
}
```

较旧的 Ego Lite 构建可能不暴露此方法。调用方应把方法缺失、请求失败或结果
无效一律视为更新状态不可用。

## 任务上下文

### `ego.useTaskSpace(id)`

为当前 Node 调用中后续的任务域 helper 调用选择任务上下文。这只是一次同步
的本地选择。

```js
const id = ego.useTaskSpace(task.id);
```

参数：

```text
id: number
```

同步返回所选的 `id`。

`id` 缺失、不是数字、或传入了多余参数时抛出 `TypeError`。抛出的错误带
`error_code: 'EGO_INVALID_ARGUMENT'`。

用 `ego.createTaskSpace(name)` 创建新的 agent 持有任务空间。用户创建的常规
Space 会以 `ownership: 'user'` 出现在 `ego.listTaskSpaces()` 的返回中——把
它们的数字 `id` 传给 `ego.claimTaskSpace(id, name?)` 之后再行选择。

## Profile

Profile 决定新的任务空间继承哪个浏览器身份、Cookie、存储和登录态。Profile
选择只在创建任务空间时生效。创建之后，照常用 `ego.useTaskSpace(task.id)`
选择返回的任务空间 `id`。

只要已安装的 Ego Lite 构建暴露了这些 Profile API，它们就可用——即使当前
checkout 的仓库运行时尚未提供等价的更高层 helper 方法。

### `await ego.listProfiles()`

列出可被 `ego.createTaskSpace()` 使用的 Profile。

```js
const result = await ego.listProfiles();
for (const profile of result.profiles) {
  console.log(profile.id, profile.name, profile.isDefault);
}
```

参数：无。

返回：

```js
{
  profiles: [
    {
      id: 'Profile 2',
      name: 'Work',
      isDefault: false,
    },
    {
      id: 'Default',
      name: 'Personal',
      isDefault: true,
    },
  ],
}
```

字段含义：

```text
id:
  ego.createTaskSpace(name, profileId) 接受的定位符，例如
  "Default" 或 "Profile 2"。

name:
  用户可见的 Profile 名称。可能重复，不能当定位符用。

isDefault:
  省略 profileId 时会被选中的那个 Profile 为 true。
  这并不意味着它的 id 一定是 "Default"。
```

结果只包含可用的常规 Profile。Ego 内部的认证 Profile 不会被暴露。

传入任何参数都抛出 `TypeError`。抛出的错误带
`error_code: 'EGO_INVALID_ARGUMENT'`。

要带 Profile 创建任务空间，把 `profile.id` 传给
`ego.createTaskSpace(name, profileId)`。完整的调用与错误处理见"任务空间"
一节。

## 标签页

### `await ego.createTab(url)`

在当前任务空间中创建一个标签页。

```js
const tab = await ego.createTab('https://example.com/');
console.log(tab.targetId);
```

参数：

```text
url: string
```

返回浏览器侧的结果对象。当前的任务实现在创建成功时返回一个至少包含
`targetId` 的标签页描述符。

`url` 缺失或不是字符串时抛出 `TypeError`。抛出的错误带
`error_code: 'EGO_INVALID_ARGUMENT'`。

### `await ego.listTabs()`

列出当前任务空间中的标签页。

```js
const result = await ego.listTabs();
for (const tab of result.tabs) {
  console.log(tab.index, tab.title, tab.url, tab.active);
}
```

参数：无。

典型结果：

```js
{
  tabs: [
    {
      index: 0,
      targetId: '...',
      url: 'https://example.com/',
      title: 'Example',
      active: true,
    },
  ],
}
```

当前任务已移交给用户时，该 Promise 会 resolve 为一个错误对象：

```js
{
  error: 'The task is under user control...',
  error_code: 'EGO_TASK_SPACE_USER_IN_CONTROL',
}
```

`error_code` 为 `'EGO_TASK_SPACE_USER_IN_CONTROL'`。对这一错误码，`error`
是一个稳定的原因键（reason key），而不是展示文案。当前的键有：

```text
notifications
location
camera
microphone
pan_tilt_zoom_microphone
midi
bluetooth
usb
serial
hid
protocol_handler
manual_takeover
```

例如，一个定位权限提示会返回：

```js
{
  error: 'location',
  error_code: 'EGO_TASK_SPACE_USER_IN_CONTROL',
}
```

调用方应使用 `error_code` 识别控制权边界，再用原因键选择引导方式。未知的
原因键必须回落到通用的用户控制引导。

页面自身弹出的 JavaScript 对话框不得走这条用户控制路径；参见
[原生 JavaScript 对话框需求](native-javascript-dialog-requirement.zh-CN.md)。

## 快照

### `await ego.snapshot(options?)`

为当前任务空间捕获一份结构化快照。

```js
const snapshot = await ego.snapshot({
  scope: 'only_within_viewport',
  includeActionMarks: true,
  interactiveOnly: true,
  includeStableLocator: true,
  maxResultLength: 4000,
});

console.log(snapshot.content);
console.log(snapshot.refs);

const subtree = await ego.snapshot({
  scope: 'subtree',
  root: snapshot.refs[0].backendNodeId,
});
```

参数：

```text
options?: object
```

支持的选项：

```text
scope:
  'full_page'            捕获整页。这是默认值。
  'only_within_viewport' 仅捕获视口内的内容。
  'subtree'              捕获由 root 指定节点及其后代。

root: number
  来自上一次快照 ref 的 backendNodeId。仅 subtree 范围需要。

includeActionMarks: boolean
  在文本快照中包含动作标记。

interactiveOnly: boolean
  只输出可交互元素。

includeStableLocator: boolean
  在可用时于 ref 上附带可复用的定位器提示。

maxResultLength: number
  结果内容的最大长度。为零或省略表示不限。
```

在 Ego Lite 0.5.0.19 中，subtree 范围要求浏览器以
`--enable-features=SnapshotSubtree` 启动。不要使用 `PageSnapshotSubtree`。
该版本在所有帧中按 `backendNodeId` 匹配 `root`——尽管该标识符只是帧内
局部的。若同一个值出现在多个帧中，调用会以歧义错误失败，而不是任选一个
节点。

结果：

```js
{
  content: '...',
  refs: [
    {
      backendNodeId: 17,
      role: 'button',
      name: 'Continue',
      loc: 'role:button[name=Continue]', // 仅在可用时出现
    },
  ],
}
```

当没有稳定、唯一的定位器可用时，请在 `refs` 和文本快照中都省略 `loc`。
不要返回 `unstable` 或 `ambiguous` 作为定位器的值。

浏览器侧报告快照错误时，该 Promise 以一个 `Error` reject。`message` 含
人类可读的错误，`error_code` 含稳定错误码，例如 `EGO_SNAPSHOT_FAILED`、
`EGO_TASK_SPACE_NOT_SELECTED`、`EGO_TASK_SPACE_NOT_FOUND`、
`EGO_TASK_SPACE_USER_IN_CONTROL` 或 `EGO_WEB_CONTENTS_UNAVAILABLE`。

## 任务空间

### `await ego.listTaskSpaces()`

列出原生任务空间桥可用的 Agent 任务空间与用户创建的常规 Space。列取是只读
操作：不会选择或认领任何 Space。

```js
const spaces = await ego.listTaskSpaces();
for (const space of spaces.taskSpaces) {
  console.log(space.id, space.name, space.ownership);
}
```

参数：无。

返回：

```js
{
  taskSpaces: [
    {
      taskId: 'research',
      id: 3,
      name: 'research',
      createdBy: 'user',
      ownership: 'user',
      profileId: 'Default',
      profileName: 'Work',
      recentTabTitles: ['Project notes', 'Reference'],
    },
    {
      taskId: 'checkout-flow',
      id: 7,
      name: 'checkout-flow',
      createdBy: 'agent',
      ownership: 'agent',
      recentTabTitles: ['Checkout', 'Cart'],
    },
  ],
}
```

`id` 是 `ego.claimTaskSpace()` 及其他任务空间 API 的定位符。`name` 和
`taskId` 是展示值，可能重复。在 Ego Lite UI 中创建的常规 Space 带
`createdBy: 'user'` 与 `ownership: 'user'`。`recentTabTitles` 或许可帮助
调用方在不认领的前提下辨认出目标 Space。

传入任何参数都抛出 `TypeError`。抛出的错误带
`error_code: 'EGO_INVALID_ARGUMENT'`。

### `await ego.createTaskSpace(name, profileId?)`

按名称创建 agent 持有的任务空间，可选地使用特定 Profile。

```js
const { profiles } = await ego.listProfiles();
const profile = profiles.find(item => !item.isDefault) ?? profiles[0];
if (!profile) {
  throw new Error('No available profile');
}

const taskName = `profile-demo-${Date.now()}`;
const task = await ego.createTaskSpace(taskName, profile.id);
if (task.error) {
  throw new Error(`${task.error} (${task.error_code})`);
}
ego.useTaskSpace(task.id);
console.log(JSON.stringify({ profile, task }, null, 2));
```

参数：

```text
name: string, 非空
profileId?: string
  ego.listProfiles() 返回的唯一 Profile id。省略、传空串或传 "default"
  时，选择当前活动/最近使用的常规 Profile。
```

返回：

```js
{
  taskId: 'checkout-flow',
  id: 7,
  name: 'checkout-flow',
}
```

`name` 缺失、不是字符串或为空；`profileId` 存在但不是字符串；或传入了多余
参数时，抛出 `TypeError`。抛出的错误带
`error_code: 'EGO_INVALID_ARGUMENT'`。

`profileId` 无法对应任何可用 Profile 时，该 Promise resolve 为：

```js
{
  error: 'Profile not found',
  error_code: 'EGO_PROFILE_NOT_FOUND',
}
```

浏览器无法创建任务空间时，该 Promise resolve 为一个错误对象：

```js
{
  error: 'No active browser',
  error_code: 'EGO_BROWSER_UNAVAILABLE',
}
```

### `await ego.claimTaskSpace(id, name?)`

认领 `ego.listTaskSpaces()` 返回的一个 Space，将其变为 agent 持有的任务
空间。这包括用户创建的常规 Space。传入 `name` 可在认领的同时重命名该
Space。

```js
const spaces = await ego.listTaskSpaces();
const checkout = spaces.taskSpaces.find(space => space.name === 'checkout-flow');
const task = await ego.claimTaskSpace(checkout.id, 'checkout-flow');
ego.useTaskSpace(task.id);
```

参数：

```text
id: number
name?: string
```

返回：

```js
{
  taskId: 'checkout-flow',
  id: 7,
  name: 'checkout-flow',
}
```

认领常规用户 Space 会保留其浏览器 Profile、标签页、活动标签页和数字
`id`；不会创建替代的 Space。取得所有权之前，调用方必须已获得用户的批准。

`id` 缺失或不是数字、`name` 存在但不是字符串、或传入了多余参数时，抛出
`TypeError`。抛出的错误带 `error_code: 'EGO_INVALID_ARGUMENT'`。

Space 不存在或无法认领时，该 Promise resolve 为一个错误对象。错误消息会
指明查找所用的数字 id：

```js
{
  error: 'Task space not found: 7',
  error_code: 'EGO_TASK_SPACE_NOT_FOUND',
}
```

`claimTaskSpace()` 本身不会选择任务上下文。认领成功后，先调用
`ego.useTaskSpace(task.id)` 再使用任务域 helpers。

### `await ego.closeTaskSpace()`

关闭当前任务空间并移除任务持有的浏览器状态。

```js
await ego.closeTaskSpace();
```

参数：无。

返回浏览器侧的状态值，当前实现成功时是形如
`"<id> task space closed."` 的字符串；任务不存在时是错误对象。错误对象含
`error` 与 `error_code`。

传入任何参数都抛出 `TypeError`。抛出的错误带
`error_code: 'EGO_INVALID_ARGUMENT'`。

### `await ego.completeTaskSpace()`

完成当前任务空间：移除 agent 任务状态，但 Space 对用户保持打开。它仍可
通过 `ego.listTaskSpaces()` 以相同的数字 `id`、用户所有权被发现。

```js
await ego.completeTaskSpace();
```

参数：无。

返回浏览器侧的状态值，当前实现成功时是形如
`"<id> task space completed."` 的字符串；任务不存在时是错误对象。错误对象
含 `error` 与 `error_code`。

传入任何参数都抛出 `TypeError`。抛出的错误带
`error_code: 'EGO_INVALID_ARGUMENT'`。

## 任务控制

### `await ego.handOffTaskSpace()`

把当前任务空间移交给用户控制。

```js
await ego.handOffTaskSpace();
```

参数：无。

返回移交操作在浏览器侧的状态值。当前实现返回形如
`"<id> has been handed off to the user."` 的字符串。

传入任何参数都抛出 `TypeError`。抛出的错误带
`error_code: 'EGO_INVALID_ARGUMENT'`。

### `await ego.takeOverTaskSpace()`

把当前任务空间收回 agent 控制。

```js
await ego.takeOverTaskSpace();
```

参数：无。

返回接管操作在浏览器侧的状态值。当前实现返回形如
`"<id> has been taken over by the agent."` 的字符串。

传入任何参数都抛出 `TypeError`。抛出的错误带
`error_code: 'EGO_INVALID_ARGUMENT'`。

### `await ego.setAgentTaskState(state)`

更新当前任务空间中用户可见的 agent 进度文案。

```js
await ego.setAgentTaskState('Waiting for checkout page');
```

参数：

```text
state: string
```

返回任务状态更新在浏览器侧的状态值。当前实现返回形如
`"<id> state updated."` 的字符串。

`state` 缺失、不是字符串、或传入了多余参数时，抛出 `TypeError`。抛出的
错误带 `error_code: 'EGO_INVALID_ARGUMENT'`。

## UI 辅助

### `await ego.animationHighlightMouseToPosition(x, y)`

在当前任务空间中，于给定屏幕位置显示一段动画高亮。

```js
await ego.animationHighlightMouseToPosition(120, 240);
```

参数：

```text
x: number
y: number
```

返回高亮操作在浏览器侧的状态值。当前实现返回形如
`"<id> mouse highlight shown."` 的字符串。

任一坐标缺失、不是数字、或传入了多余参数时，抛出 `TypeError`。抛出的
错误带 `error_code: 'EGO_INVALID_ARGUMENT'`。

## Chrome DevTools Protocol

### `ego.sendCDPMessage(message)`

经 Node 的 CDP 通道发送一条裸 CDP 消息。消息必须已是 JSON 编码。当前任务
必须已用 `ego.useTaskSpace(id)` 选择，且其所有权为 `agent`。若任务缺失、
未激活、已委托给用户或不可用，消息不会被发送，
`ego.onSendCDPMessageError` 会收到重构后的错误消息与错误码。

```js
const task = await ego.createTaskSpace('cdp-demo');
ego.useTaskSpace(task.id);
ego.sendCDPMessage(JSON.stringify({
  id: 1,
  method: 'Runtime.enable',
}));
```

参数：

```text
message: string
```

返回 `undefined`。

`message` 缺失或不是字符串、或 CDP 通道未连接时，抛出 `TypeError`。抛出的
错误，参数非法时 `error_code` 为 `EGO_INVALID_ARGUMENT`，通道缺失时为
`EGO_CDP_CHANNEL_UNAVAILABLE`。

注意：

```text
裸 CDP 的用户控制检查使用 ego.useTaskSpace(id) 所选的任务。
若尚未选择任何任务，发送以 EGO_TASK_SPACE_NOT_SELECTED 失败。
```

### `ego.onCDPMessage`

用于接收浏览器侧发来的裸 CDP 消息的属性。

```js
ego.onCDPMessage = message => {
  const event = JSON.parse(message);
  console.log(event.method);
};
```

赋一个函数以接收消息。赋任何非函数值会清除回调。传入的消息经 Node 事件
循环异步送达。

### `ego.onSendCDPMessageError`

用于接收 `ego.sendCDPMessage()` 本地发送失败的属性。

```js
ego.onSendCDPMessageError = (message, error_code) => {
  console.error(message, error_code);
};

const task = await ego.createTaskSpace('cdp-version');
ego.useTaskSpace(task.id);
ego.sendCDPMessage(JSON.stringify({
  id: 2,
  method: 'Browser.getVersion',
}));
```

送达回调的参数：

```text
message: string
error_code: string
```

该回调不携带原因键。因此一次用户控制的发送失败只能识别出控制权边界，
无法得知是哪个权限触发的。

赋一个函数以接收错误。赋任何非函数值会清除回调。错误经 Node 事件循环
异步送达。

该回调用于消息到达 CDP **之前**发生的失败，包括：

```text
- 任务当前处于用户控制之下
- 尚未选择任何任务空间
- 所选任务空间不存在
- 所选任务空间未被 agent 认领
- 浏览器侧的 CDP agent 宿主已不可用
```

用户控制示例：

```js
const task = await ego.createTaskSpace('user-control-demo');
ego.useTaskSpace(task.id);
await ego.handOffTaskSpace();

const result = await ego.createTab('https://example.com/');
console.log(result.error);
console.log(result.error_code); // "EGO_TASK_SPACE_USER_IN_CONTROL"

try {
  await ego.snapshot();
} catch (error) {
  console.log(error.message);
  console.log(error.error_code); // "EGO_TASK_SPACE_USER_IN_CONTROL"
}

ego.onSendCDPMessageError = (message, error_code) => {
  console.log(message);
  console.log(error_code); // "EGO_TASK_SPACE_USER_IN_CONTROL"
};
ego.sendCDPMessage(JSON.stringify({
  id: 1,
  method: 'Browser.getVersion',
}));

await ego.takeOverTaskSpace();
```

## 完整示例

```js
const { profiles } = await ego.listProfiles();
const profile = profiles.find(item => !item.isDefault) ?? profiles[0];
if (!profile) {
  throw new Error('No available profile');
}

const task = await ego.createTaskSpace('checkout-flow', profile.id);
if (task.error) {
  throw new Error(`${task.error} (${task.error_code})`);
}
ego.useTaskSpace(task.id);

await ego.createTab('https://example.com/');
await ego.setAgentTaskState('Reading page');

const snapshot = await ego.snapshot({
  scope: 'only_within_viewport',
  includeActionMarks: true,
  interactiveOnly: true,
});

console.log(snapshot.content);

await ego.completeTaskSpace();
```

## 原生 API 兼容性说明

### 支持 Profile 的任务空间创建

- 新增 `ego.listProfiles()`，以 `{id, name, isDefault}` 返回可用 Profile。
- `ego.createTaskSpace(name)` 现接受可选的 Profile 定位符：
  `ego.createTaskSpace(name, profileId?)`。
- Profile 的显示名不可作为定位符，因为它们不唯一。调用方必须传
  `ego.listProfiles()` 返回的 `id`。
- 省略 `profileId`、传空串或传 `"default"` 时，选择当前默认的常规
  Profile。
- 未知的 Profile id 会 resolve 一个带
  `error_code: 'EGO_PROFILE_NOT_FOUND'` 的错误对象。

### 任务空间 id 是数字定位符

- `ego.useTaskSpace(taskId)` 变更为 `ego.useTaskSpace(id)`。
  `id` 是任务空间 API 返回的数字型任务空间定位符。

- `ego.claimTaskSpace(name)` 变更为 `ego.claimTaskSpace(id, name?)`。
  第一个参数按数字 id 标识既有的任务空间。`name` 可选，提供时会在认领
  的同时重命名任务空间。

- 任务空间结果对象现已区分定位符字段与展示字段：

  ```js
  {
    id: 7,
    taskId: 'checkout-flow',
    name: 'checkout-flow',
  }
  ```

  调用 API 时使用 `id`。`taskId` 当前与 `name` 同值，仅作为显示名兼容
  字段保留。

- 任务域示例现在都用 `task.id` 选择任务空间，包括 `createTab`、
  `listTabs`、`snapshot`、`closeTaskSpace`、`completeTaskSpace`、
  `handOffTaskSpace`、`takeOverTaskSpace`、`setAgentTaskState`、
  `animationHighlightMouseToPosition`，以及裸 CDP 发送。

- 浏览器侧的操作消息现在以数字 id 标识任务空间，例如
  `"<id> task space completed."`、`"<id> state updated."` 和
  `"Task space not found: <id>"`。

### 错误码

- 浏览器侧的错误结果现在包含稳定的 `error_code` 字符串。
- `ego.snapshot()` 以带 `error_code` 属性的 `Error` reject。
- `ego.onSendCDPMessageError` 接收 `(message, error_code)`。
