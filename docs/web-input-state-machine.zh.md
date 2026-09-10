# Web 输入状态机

[English](web-input-state-machine.md) | 中文

输入状态机负责从用户编辑输入框开始，到服务端接受或拒绝这次输入为止的过程。它不负责保存对话历史，也不负责执行模型。

本页按从用户行为到代码职责的顺序介绍这个子系统：“用户行为与系统约束”说明为什么需要统一状态机；“状态建模”说明系统需要保存什么；“输入事件，输出副作用”说明状态如何变化以及外部操作如何执行；“实现分层”再把这些概念对应到具体对象和源码。后续章节用普通消息、Slash 命令、引用、图片和会话生命周期验证这套模型覆盖的实际路径。

消息被接受之后的完整链路见[从网页输入到 Agent 回复](web-session-flow.zh.md)；浏览器请求、Host 接纳、模型请求、事件回传和执行模型见 [Web 请求链路与执行模型](web-request-execution-model.zh.md)。

## 1. 用户行为与系统约束

输入框需要同时满足几项互相影响的要求：

1. 发送失败后，文字和附件仍然可供用户修改和重试。
2. 发送成功后，已提交内容应被移除，但延迟返回的结果不能清掉其他事件产生的较新草稿状态。
3. 连续按 Enter 和延迟返回的异步结果不能让同一次提交结算两次。
4. 普通消息、`/` 命令、引用、粘贴内容和图片附件需要共用同一套编辑与提交流程。
5. 撤销和重做需要把一次编辑、粘贴或引用插入视为一次用户操作；已经被接受的内容不能通过撤销重新出现。
6. 每个打开的会话需要独立的草稿和提交阶段；关闭会话时必须取消尚未完成的输入工作。

这些要求不能分别处理。例如，请求返回后直接清空文本框可能删除用户后来输入的内容；让命令识别单独调用服务端，则可能绕开相同的失败处理和取消规则。因此，系统把输入框变化建模为事件，并由每个会话独有的状态机处理。

## 2. 状态建模

状态名描述的不是 Agent 在做什么，而是输入框此刻能否继续接受用户操作。先看三种实际用法。

### 2.1 发送普通消息

用户输入“总结这个项目”，输入框处于 `plain`：文字和图片都可以继续修改，也可以撤销、粘贴或插入引用。

用户按 Enter 后，输入框进入 `submitting`。界面仍显示“总结这个项目”，但暂时只读。这个内部中间态表示系统正在把引用和图片整理成可发送内容，并等待服务端确认已经接收；它不表示模型正在回答。

- 服务端接收成功：输入框清空并回到 `plain`，用户可以写下一条消息。
- 服务端拒绝或网络失败：原文字和图片保留，输入框回到 `plain`，用户可以修改后重试。
- 用户连续按 Enter：只有第一次有效，因为 `submitting` 不接受第二次提交。

这条路径是：`plain` → `submitting` → `plain`。

### 2.2 从菜单选择命令

用户输入 `/`，从菜单选择需要继续填写内容的 `/goal`。输入框显示 `/goal ` 和参数提示，并进入 `claimed`。这个状态只表示两件事：用户已经明确选择 `/goal`，系统正在等待用户填写目标。命令还没有执行。

用户可以继续输入 `/goal 修复登录失败问题`。删除或改坏开头的 `/goal ` 会取消这次选择，输入框回到普通编辑的 `plain`；按 Enter 则进入 `submitting`，并把“修复登录失败问题”作为参数交给 `/goal` 的处理器。

- 命令成功：清空输入框并回到 `plain`。
- 命令失败且 `/goal` 草稿没有变化：回到 `claimed`，保留参数供用户直接修改或重试。
- 命令失败时草稿已经被其他事件改变：回到 `plain`，避免把变化后的普通文字继续交给旧命令。

这条路径通常是：`plain` → `claimed` → `submitting` → `plain`。命令失败时也可能从 `submitting` 回到 `claimed`。

并非所有菜单命令都会进入 `claimed`。`/compact` 不接收用户继续填写的参数，属于选择后立即执行的命令。用户从菜单选择它时，系统会移除输入框中的命令词、立即执行压缩，并让输入框保持在 `plain`。因此 `/compact` 不能用来解释 `claimed`；只有声明了后续输入的命令，例如 `/goal` 或 `/feedback`，才需要这个状态。

### 2.3 直接输入 Slash 命令

用户没有从菜单选择，而是直接输入 `/goal 修复登录失败问题` 并按 Enter。系统不能只看文字就假定它一定是命令，因为插件可能没有注册该命令，也可能由触发器自行处理。因此输入框先进入 `adjudicating`。

`adjudicating` 是用户通常只会短暂看到的内部中间态：输入框暂时只读，系统询问已加载的命令来源“谁来处理这段文字”。它确保同一次 Enter 不会同时执行命令和发送普通消息。

- 找到 `/goal` 这类接收参数的命令：进入 `submitting`，交给该命令处理器。
- 找到 `/compact` 这类立即执行的命令：命令在独立路径上开始执行，输入框移除命令词并直接回到 `plain`。
- 没有命令认领：进入 `submitting`，把原文当作普通消息发送。
- 触发器已经完成操作：回到 `plain`，不再发送。
- 判断过程失败：保留原文并回到 `plain`，同时显示错误。

这条路径是：`plain` → `adjudicating` → `submitting` → `plain`，也可能从 `adjudicating` 直接回到 `plain`。

### 2.4 四个阶段速查

| 阶段 | 用户所处的时刻 | 输入框 |
|---|---|---|
| `plain` | 正在写普通消息，或修改失败后保留的内容。 | 可编辑。 |
| `claimed` | 已从菜单选择 `/goal` 等接收参数的命令，正在填写参数；命令尚未执行。 | 可编辑；命令选择仍然有效。 |
| `adjudicating` | 已按 Enter，系统正在确认直接输入的 `/...` 由谁处理。 | 暂时只读；内部中间态。 |
| `submitting` | 已按 Enter，系统正在整理内容并等待服务端接收结果。 | 暂时只读；内部中间态。 |

成功提交只表示服务端已经接收这条输入。之后的排队、Turn、Step 和 Agent 回复属于会话流程，不属于这四个阶段。

### 2.5 状态机还记住什么

页面消费的 `InputState` 定义在 [`contract.ts:213`](../packages/client/ui-conversation/src/client/input/contract.ts)。除了上述 `phase`，状态机还要记住以下内容：

| 数据 | 具体例子 | 保存它解决什么问题 |
|---|---|---|
| `draft` | 输入框里的“总结这个项目”。 | 失败时保留原文；成功时只删除这次已经发送的部分。 |
| `imageIds` | 用户依次添加的 `a.png`、`b.png`。 | 失败时保留两张图；成功时只移除这次提交包含的图片。文件和预览地址由浏览器运行时另行保存。 |
| `draftRev` | 每次草稿改变时递增的内部计数；用户看不到它。 | 给插件观察到的字符位置标记有效期，防止旧位置修改已经变化的草稿。 |
| `claim` | 用户从菜单选择的 `/goal`、参数提示和处理器。 | Enter 知道要调用哪个命令；命令词被删除后，旧选择立即失效。处理器保存在状态机内部，页面只看到展示所需信息。 |
| `occurrences` | 草稿中的 `@session:abc` 引用位于第 8 至 20 个字符。 | 用户在引用前输入文字时，引用位置随之移动；发送时再把它转换成引用来源要求的模型文本。 |
| `paste` | 刚粘贴的一段文字仍在等待识别其中的引用。 | 识别结果晚到时，只有草稿和选区仍匹配才升级为引用；用户已经继续编辑时直接忽略旧结果。 |
| `queue` | 当前会话还有两条待处理消息。 | 页面在同一份快照中展示待处理消息；它是只读投影，不改变输入阶段。 |

> **理解重点：** 可以把 `draftRev` 看成“草稿字符位置的有效期”。它不表示用户进入了新阶段，也不保存草稿内容；它只回答一个问题：插件计算 `[start, end)` 时看到的草稿，还是不是当前草稿？

例如，当前草稿是“看 `@ses`”，`draftRev` 为 `12`。引用插件观察到 `@ses` 位于 `[2, 6)`，开始异步查询，并把 `{ start: 2, end: 6, draftRev: 12 }` 保存下来。查询返回前，用户在开头输入“请”，草稿变成“请看 `@ses`”，`@ses` 已经移动到 `[3, 7)`，状态机同时把 `draftRev` 增加到 `13`。

旧查询此时仍要求替换 `[2, 6)`。如果只相信字符位置，它会截断错误的文字。状态机因此先比较查询携带的 `draftRev: 12` 与当前的 `draftRev: 13`：

```text
12 === 13 -> false -> reject
```

插件下一次跟踪当前草稿时会得到新的区间和 `draftRev: 13`，再提交一次有效替换。这个“比较版本，相等才写入”的规则就是这里的 CAS 防护。每次输入、删除、撤销、重做，以及菜单、引用或命令插件成功改写草稿，都会增加 `draftRev`，使此前计算的字符区间全部过期。仅比较区间里的文字仍不充分，因为相同文字可能出现在多个位置，而且插件的决定还可能依赖当时的选区和上下文。

> **不要与提交编号混淆：** `draftRev` 防止插件用旧字符位置改写新草稿；`SubmitAttempt.seq` 防止旧网络结果结算新提交。前者属于编辑平面，后者属于提交平面。

当前 UI 的菜单刷新、焦点管理和同步事件调用已经避免了大多数版本不匹配，因此用户通常看不到拒绝过程。这项检查固定的是插件接口的防御约束：任何插件都不能凭旧草稿中的字符位置获得修改当前草稿的权限。

状态机还保存撤销记录和每次提交的编号。例如，第一次提交已经失败并结束，第二次提交正在进行，此时第一次请求的延迟结果到达；编号不匹配，状态机直接忽略它。关闭会话也会取消当前编号对应的工作。这个提交记录称为 `SubmitAttempt`，定义在 [`contract.ts:236`](../packages/client/ui-conversation/src/client/input/contract.ts)。

状态机不保存 DOM 光标。文本框在编辑时把变化区间或选区一起传入，状态机用它调整引用位置和撤销记录，React 继续管理屏幕上的真实光标。

## 3. 输入事件，输出副作用

这里的“输入事件”不是单指键盘事件，而是交给状态机处理的一项事实或请求。例如，`draft-changed` 表示草稿已经改变，`enter` 表示用户要求提交，`submit-settled` 表示此前的异步提交已经返回结果。所有事件都通过 [`contract.ts:251`](../packages/client/ui-conversation/src/client/input/contract.ts) 的 `InputEvent` 联合类型进入 [`machine.ts:170`](../packages/client/ui-conversation/src/client/input/machine.ts) 的 `dispatch()`。

“输出副作用”表示状态机要求外部执行、但自己不能执行的逻辑。这里的“副”不表示动作次要，而是表示它发生在内存状态计算之外，会调用命令插件、发送接口或 UI 提示存储。`dispatch()` 只同步完成状态变化，并把待执行动作作为零个或多个 [`contract.ts:287`](../packages/client/ui-conversation/src/client/input/contract.ts) 定义的 `InputEffect` 数据返回；`SessionInputShell` 再解释并执行这些数据。这样，同一事件的状态结果和外部操作要求都可以直接检查，而状态机本身不依赖网络、插件或 React。

这采用了命令模式的核心思想：把“要执行什么”封装成值，再交给统一执行者处理。状态机是命令创建者，`InputEffect` 是带判别字段和完整参数的命令描述，`SessionInputShell.execute()` 是统一调用者，shell 注入的命令来源、提交处理器、发送函数和提示存储是实际接收者。这里没有为每种命令建立一个带 `execute()` 方法的类，因此不是教科书式的命令对象层次；判别联合已经提供了同样关键的分离：状态机决定意图，运行时选择执行机制。命令值可以直接断言、在会话销毁后不执行，并由 shell 在一个位置统一处理异步回流和资源所有权。

四种副作用的执行语义如下：

| 副作用 | 何时产生 | `SessionInputShell` 实际执行什么 | 结果如何返回 |
|---|---|---|---|
| `adjudicate` | 用户在 `plain` 中直接提交以 `/` 开头的草稿。 | 调用会话的输入触发器，让已加载的命令来源判断草稿由谁处理；未挂载触发器时按“无人认领”处理。 | 判定结果成为 `adjudicated`，异常成为 `adjudication-failed`。 |
| `begin-submit` | 已有命令认领当前草稿，并开始提交其参数。 | 对声明接收图片的命令先序列化本次图片，再调用认领对象的 `submit` 处理器；只有成功时才移除这些图片。 | 成功、处理器错误或序列化错误都成为 `submit-settled`。 |
| `default-sink` | 草稿作为普通消息提交，包括 `/...` 无人认领的情况。 | 把内联引用替换为各来源提供的模型文本，再调用当前会话的默认发送函数；缺少序列化器或序列化失败会阻止发送并保留草稿。 | 发送成功、发送失败或序列化失败都成为 `submit-settled`。 |
| `notice` | 判定或提交结果需要向用户显示信息或错误。 | 同步写入 shell 的提示快照，由 `InputBar` 渲染短暂提示。 | 不需要异步回流事件。 |

`submit-settled` 本身不是副作用，而是异步提交完成后回到状态机的 `InputEvent`。它携带原 `SubmitAttempt`、成功标志和可选结果或说明文字；状态机先校验 attempt 是否仍是当前提交，防止旧请求的延迟结果改写较新的状态，再决定清理草稿、保留草稿以及回到 `plain` 还是 `claimed`。

状态本身没有“进入时自动执行”的逻辑。一次 `dispatch()` 会同时决定“状态变成什么”和“现在需要外部做什么”。适配层执行副作用；异步操作完成后，它再把结果包装成 `adjudicated`、`adjudication-failed` 或 `submit-settled` 事件送回状态机。完整循环如下：

```text
InputEvent
    ↓
InputMachine.dispatch()
    ├─ InputState → publish → React
    └─ InputEffect → SessionInputShell → external work
                                           ↓
                                       InputEvent
```

状态变化和副作用产生两类不同结果：

| 结果 | 由谁处理 | 具体逻辑 |
|---|---|---|
| 状态快照变化 | React 页面 | `plain` 和 `claimed` 允许编辑；`claimed` 显示命令参数提示；`adjudicating` 和 `submitting` 让输入框暂时只读。草稿、引用和附件变化也随快照重新渲染。 |
| `InputEffect` | `SessionInputShell` 适配层 | 调用命令来源、命令处理器或会话发送接口，或者发布一条提示；Promise 的结果随后作为新事件回到状态机。 |

主要状态流转实际触发的逻辑如下：

| 起点和输入事件 | 状态机立即做什么 | 返回的副作用 | 外部逻辑完成后 |
|---|---|---|---|
| `plain` 中编辑，收到 `draft-changed` | 更新草稿、引用位置、撤销记录和 `draftRev`，仍留在 `plain`。 | 无 | 页面显示新草稿。 |
| 菜单选中 `/goal`，收到 `begin-command` | 把命令词写入草稿并进入 `claimed`。 | 无 | 页面显示参数提示并继续允许编辑；命令尚未执行。 |
| `claimed` 中按 Enter，收到 `enter` | 创建本次提交记录并进入 `submitting`。 | `begin-submit` | 适配层调用 `/goal` 的提交处理器；结果以 `submit-settled` 回流，成功回到 `plain`，失败通常回到 `claimed` 并保留草稿。 |
| `plain` 中提交普通文字，收到 `enter` | 创建本次提交记录并进入 `submitting`。 | `default-sink` | 适配层序列化引用并调用会话发送接口；`submit-settled` 成功时清理已发送内容并回到 `plain`，失败时保留草稿并回到 `plain`。 |
| `plain` 中直接提交 `/...`，收到 `enter` | 进入 `adjudicating`，暂不决定它是命令还是普通消息。 | `adjudicate` | 命令来源返回 `adjudicated`：认领命令后进入 `submitting` 并产生 `begin-submit`；无人认领则进入 `submitting` 并产生 `default-sink`；来源已自行处理则直接回到 `plain`。判定失败返回 `adjudication-failed`，回到 `plain` 并产生错误 `notice`。 |
| `submitting` 中收到 `submit-settled` | 校验提交编号；成功清理已接收内容，失败保留草稿，再回到 `plain` 或 `claimed`。 | 结果带说明文字时返回 `notice`，否则无副作用。 | 页面解除只读；`notice` 被渲染为提示。 |

因此，“进入 `submitting` 会发送消息”只是简写。准确说法是：处理 `enter` 或 `adjudicated` 事件时，状态机进入 `submitting`，并同时返回 `begin-submit` 或 `default-sink`；真正的命令调用或消息发送由适配层执行。这样的拆分让状态转换代码不依赖 HTTP、附件存储、命令插件或 React。

## 4. 实现分层

实现方式是让一个具体的纯状态机位于多个较窄的公开接口之后，并没有定义 `IInputMachine` 接口和多个状态机实现。

| 层次 | 对象 | 职责 | 源码 |
|---|---|---|---|
| 类型与公开接口 | `InputState`、`InputEvent`、`InputEffect`、`SessionInput`、`SessionInputResolver` | 定义可观察状态、合法操作、待执行副作用和按会话访问的 API。 | [`contract.ts:25`](../packages/client/ui-conversation/src/client/input/contract.ts)、[`contract.ts:213`](../packages/client/ui-conversation/src/client/input/contract.ts)、[`contract.ts:251`](../packages/client/ui-conversation/src/client/input/contract.ts) |
| 纯转换引擎 | `InputMachine` | 保存草稿版本、阶段、命令认领、引用、粘贴尝试、撤销/重做事务和当前提交标识；接收事件并产生副作用。 | [`machine.ts:116`](../packages/client/ui-conversation/src/client/input/machine.ts)、[`machine.ts:170`](../packages/client/ui-conversation/src/client/input/machine.ts) |
| 运行时适配层 | `SessionInputShell` | 保存附件、发布快照、执行副作用、调用命令与 Host API、把 Promise 结果转换回事件，并发布提示。 | [`facade.ts:83`](../packages/client/ui-conversation/src/client/input/facade.ts)、[`facade.ts:420`](../packages/client/ui-conversation/src/client/input/facade.ts) |
| 按会话组装与注册 | `InputHub` | 通过依赖注入创建每个会话的适配层，注册会话范围的输入触发事件，路由默认发送，并把实例与监听器的销毁交给 Cordis session scope。 | [`hub.ts:39`](../packages/client/ui-conversation/src/client/input/hub.ts)、[`hub.ts:71`](../packages/client/ui-conversation/src/client/input/hub.ts) |
| UI 注册 | Conversation 插件组装 | 通过标准 Session provide 通道发布适配层的状态快照和操作。 | [`apply.ts:168`](../packages/client/ui-conversation/src/client/apply.ts) |
| UI 反馈 | `InputBar` | 把状态机提示和 Host 发送错误渲染为短暂错误提示。 | [`InputBar.tsx:69`](../packages/client/ui-conversation/src/client/skeleton/InputBar.tsx) |

因此，真正有用的抽象是把状态决定与运行时操作分开。`InputMachine` 保持包内私有，因为其他包需要的是公开状态和操作，而不是替换转换引擎的权限。Conversation 组装层通过 `SessionInput` 和 `SessionInputResolver` 访问输入系统。

文件名也反映了这项分工。`contract.ts` 只定义状态机、运行时适配层和页面共同依赖的类型与公开接口，例如页面可观察的 `InputState`、可派发的 `InputEvent` 和 shell 必须执行的 `InputEffect`；这些定义不属于某一个实现方，因此不放在 `machine.ts`。这里的 contract 指调用方和实现方共同遵守的类型约定，不表示网络协议或兼容性承诺。

`facade.ts` 体现门面模式：页面和插件面对的是按会话解析的 `SessionInput` 窄接口，不需要分别协调 `InputMachine`、附件对象、命令触发器、会话发送函数、状态存储和提示存储。`SessionInputShell` 作为统一入口，把 `setDraft()`、`submit()`、`addImages()` 等调用转换成状态机事件，执行状态机返回的命令，发布组合后的快照，并管理异步结果和会话释放。门面没有替代这些内部组件的职责，而是固定调用顺序、所有权和错误回流，使调用方不依赖子系统的协作细节。

`SessionInputShell` 这个名字还表达“纯函数核心、命令式外壳”的分层：`SessionInput` 表示它提供一个会话的输入操作，`Shell` 表示它包住只处理内存转换的 `InputMachine`，并且是允许接触命令插件、会话发送接口、附件资源和 React 提示存储的运行时层。它同时承担门面和命令执行者两个角色；`InputHub` 为每个 `SessionId` 保存一个 shell，并在会话释放时一并销毁。

### 4.1 `InputHub`：依赖注入与控制反转

`InputHub` 组合了按会话注册表、依赖组装和路由职责。它的 `Map<SessionId, SessionInputShell>` 保证每个活动会话解析到同一个 shell，但它不是通用依赖注入容器；Cordis 才是管理插件服务、事件和 effect 生命周期的框架。

依赖注入发生在 `InputHub.shellFor()` 创建 `SessionInputShell` 时。shell 不自行查找或构造命令触发器、队列投影、会话发送函数和图片序列化器，而是接收 `actx`、`inputTriggers`、`queue`、`defaultSink` 和 `commandImages` 等窄能力。`InputHub` 把 Cordis 服务和当前 session binding 适配成这些能力，因此 shell 依赖调用接口，不依赖 conversation、attachment 或 input-trigger 插件的具体实现。这也是依赖倒置：高层输入流程声明需要什么，组装层决定由谁提供。

控制反转描述的是“谁决定何时调用”。`InputHub` 不主动创建会话、不轮询事件，也不自行判断何时销毁实例：

```text
Cordis/session scope
  -> materialize -> InputHub.shellFor(...)
  -> dispatch event -> scoped listener -> SessionInputShell
  -> dispose -> effect disposer -> shell.dispose()
```

会话 provide materialization 是创建触发点；`InputHub` 在 session `actx` 上通过 `actx.on()` 注册输入修改监听器，事件随后由 Cordis 分发到这些监听器；所有监听器、shell、注册表条目和剩余附件清理都放在同一个 `actx.effect()` disposer 中，由 session scope 释放触发。因而准确说法是：**Cordis/session scope 控制创建、事件调用和销毁时机，`InputHub` 响应这些时机完成输入子系统的组装、路由与清理。**

> **延伸研究：控制反转。** 本页只说明它在输入子系统中的落点。继续研究时可从 [Cordis 入门](cordis-primer.zh.md)的“五个核心概念”开始，重点对照“上下文是服务的容器”“通过 `inject` 声明服务依赖”“类型化事件用于通信”和“注册是可逆的副作用”，再阅读其中“实践规则”对 disposer 与 teardown 顺序的说明。

## 5. 普通消息提交流程

下面用一条包含引用和图片的普通消息说明完整闭环。用户在输入框中写入“比较 `@session:abc` 的方案”，并附加 `diagram.png`。按 Enter 前，关键状态可以简化为：

| 字段 | 按 Enter 前的值 | 含义 |
|---|---|---|
| `phase` | `plain` | 输入框允许继续编辑和提交。 |
| `draft` | `比较 @session:abc 的方案` | 页面显示的草稿文本；引用仍是面向用户的显示形式。 |
| `occurrences` | `@session:abc` 的来源、标识和字符区间 | shell 稍后依靠这组区间替换引用，不重新扫描草稿。 |
| `imageIds` | `[diagram.png 对应的 id]` | 状态机只观察标识；文件、字节和预览地址由 shell 持有。 |
| `inflight` | 无 | 当前没有正在判定或提交的输入。 |

文本框此前每次调用 `SessionInputShell.setDraft()` 都会派发 `draft-changed`。状态机在同一事务内更新 `draft`、引用区间、`draftRev` 和撤销记录；这个编辑阶段始终留在 `plain`，不产生副作用。

### 5.1 Enter：从 `plain` 进入 `submitting`

`submitting` 是客户端输入状态机的“等待本次提交结算”阶段。它不表示消息已经发送成功，也不表示服务端或模型正在处理消息；它只表示状态机已经接受这次 Enter、记录了唯一的 `inflight` attempt，并发出了负责实际提交的副作用，但尚未收到该副作用返回的 `submit-settled`。

这里的两个词表示不同事物：

| 术语 | 它是什么 | 包含什么 |
|---|---|---|
| `attempt` | 一次 Enter 触发的提交尝试记录，类型为 `SubmitAttempt`。它是一个具体的数据对象，会随 `adjudicate`、`begin-submit` 或 `default-sink` 副作用传出，再随结果事件原样返回。 | `seq` 是递增序号；`draftSnapshot` 是按 Enter 时的草稿；`mode` 是 `queue` 或 `steer`；`signal` 用于取消异步工作。 |
| `inflight` | 状态机内部保存“当前尚未结束的 attempt”的唯一槽位。代码中的值是 `{ attempt, controller }`；`undefined` 表示当前没有正在判定或提交的输入。 | attempt 本身，以及可以触发其 `AbortSignal` 的 `AbortController`。 |

因此，`inflight attempt` 不是两个 attempt，而是“当前占用 `inflight` 槽位的那个 attempt”。例如，Enter 创建 attempt `#7` 后，`inflight` 指向 `#7`；`submit-settled(#7)` 到达并通过序号校验后，状态机清空该槽位。此后迟到的 `submit-settled(#6)` 与当前槽位不匹配，会被忽略。直接输入 Slash 命令时，同一个 attempt 会先在 `adjudicating` 中等待归属判断，再进入 `submitting`，不会为实际发送另建一个 attempt。

下面这些场景说明为什么需要同时保存 attempt 和 `inflight`：

| 场景 | attempt 和 `inflight` 怎样工作 | 解决的问题 |
|---|---|---|
| 普通消息发送 | Enter 创建 attempt `#7`，`inflight` 保存 `#7`；`default-sink` 和最后的 `submit-settled` 都携带同一个对象。 | 把异步结果对应到发起它的那次 Enter。 |
| 连续按 Enter | `inflight` 已存在且阶段是 `adjudicating` 或 `submitting` 时，第二次 Enter 不创建 attempt，也不产生发送副作用。 | 防止同一份草稿重复提交。 |
| 旧结果迟到 | 当前 `inflight` 是 `#7` 时，`submit-settled(#6)` 因序号不匹配而被忽略。 | 防止旧请求清除或覆盖较新的草稿。 |
| Slash 命令判定 | 直接输入 `/goal ...` 时，一个 attempt 依次经过 `adjudicating` 和 `submitting`；判定完成后不会再创建一个发送 attempt。 | 让命令识别与实际提交属于同一次用户操作。 |
| 会话关闭 | `inflight` 同时持有 `AbortController`；scope 释放时触发取消，引用序列化和浏览器请求都观察同一个 signal。 | 阻止已关闭会话继续发送或用迟到结果修改页面。 |
| 引用或附件正在准备 | attempt 保存按 Enter 时的草稿和模式，提交阶段禁止增删图片，shell 使用本次捕获的数据发送。 | 防止页面内容与实际请求内容在异步准备期间分叉。 |

| 观察角度 | `submitting` 期间的事实 |
|---|---|
| 已经确定 | 本次提交的序号、按 Enter 时的草稿快照、投递模式和取消信号已经保存在 `inflight` 中；状态机也已经选择 `begin-submit` 或 `default-sink`。 |
| 正在等待 | shell 在状态机之外执行命令处理，或者准备引用和附件并调用会话发送接口；状态机等待执行结果以 `submit-settled` 事件返回。 |
| 页面限制 | 输入框暂时只读，重复 Enter 被忽略，图片不能增删；原草稿、引用和附件仍然保留，便于失败后恢复。 |
| 退出条件 | 当前 attempt 对应的 `submit-settled` 到达后，成功或失败都会结束 `submitting`；成功提交已捕获的内容，失败保留可重试内容。会话销毁则取消 attempt 并释放整个输入实例。 |

单独保存这个阶段有三个作用：阻止同一输入被重复提交；防止附件在序列化和发送期间变化；用 attempt 序号拒绝迟到的旧结果。服务端接收成功之后的排队、Turn、Step 和 Agent 回复由会话流程表示，不会让输入框继续停留在 `submitting`。

Enter 使 shell 派发 `{ type: 'enter', mode }`。`onEnter()` 在 [`machine.ts:483`](../packages/client/ui-conversation/src/client/input/machine.ts) 按以下顺序处理：

1. 如果已经处于 `adjudicating` 或 `submitting`，返回空副作用数组，因此连续按 Enter 不会创建第二次提交。
2. 对普通消息去除首尾空白后检查内容；空草稿不创建 attempt。纯图片提交已由 `SessionInputShell.submit()` 在进入状态机前分流到独立发送路径。以 `/` 开头的草稿则走 Slash 判定路径，本例不进入这两个分支。
3. `beginAttempt()` 递增序号，创建 `{ seq, signal, draftSnapshot, mode }`，并把它写入唯一的 `inflight` 槽。`draftSnapshot` 保存按 Enter 时的显示草稿，`mode` 保存本次普通消息的 `queue` 或 `steer` 投递意图，`AbortSignal` 用于会话释放时取消后续工作。
4. 状态机把 `phase` 从 `plain` 改为 `submitting`，但不清空 `draft`、引用或附件。
5. 状态机返回 `{ type: 'default-sink', attempt, draft, mode }`。这是需要 shell 执行的命令值，不是另一个状态。

`SessionInputShell.run()` 先调用 `execute()` 启动这项副作用，再发布组合后的状态快照。React 因而看到 `phase: submitting` 和原草稿、原附件，把输入框设为只读。此时的闭环只完成了前半段：

```text
plain
  └─ enter
       ├─ state: submitting + inflight attempt
       └─ effect: default-sink
```

### 5.2 `default-sink`：准备并发送消息

`SessionInputShell.execute()` 把 `default-sink` 分派给 [`facade.ts:456`](../packages/client/ui-conversation/src/client/input/facade.ts) 的 `sinkSerialized()`。这项副作用分六步执行：

1. shell 复制当前 `imageIds`，形成这次发送的附件集合，并读取状态机中的 `occurrences`。提交期间禁止增删图片，因此页面显示的附件与该集合保持一致。
2. 没有引用时，shell 直接对草稿执行 `trim()`，调用注入的 `defaultSink(text, imageIds, mode, attempt.signal)`；这条快速路径没有额外的异步序列化。
3. 有引用时，shell 通过输入触发器并行调用每个来源的 `serializeReference(source, ref, signal)`。`source` 指注册这类引用的插件；该插件的 codec 决定怎样把页面标签转换成后续会话管线能够识别的文本。对于本例的会话引用，页面显示 `@session:abc`，occurrence 中的 `ref` 则保存类似 `@[session:abc](dsh-session:<encoded-id>)` 的规范引用，统一 `@` 引用来源的 serializer 返回这个 `ref`。
4. 所有引用完成后，shell 按记录的字符区间从左到右拼接草稿，用序列化结果替换显示文本，再对最终文本执行 `trim()`。发送的用户消息因此携带规范引用，而不是只携带页面标签。模型请求组装前，`session-reference` 插件识别该规范引用，把正文中的令牌替换回可读的 `@session:abc`，并把被引用会话的有限内容快照作为额外上下文加入请求。缺少来源序列化器或任一序列化失败都会停止发送，不会退回为只发送页面标签。
5. `InputHub` 的默认发送函数在 [`hub.ts:165`](../packages/client/ui-conversation/src/client/input/hub.ts) 调用当前会话的 `conversation.sendSession(session, text, imageIds, mode, signal)`。空文本且无图片时直接返回成功，否则由 conversation service 接收消息。
6. `settleSubmit()` 观察这个 Promise。返回 `SubmitOutcome` 时，它把 `outcome.kind === 'success'` 转成成功标志；Promise 拒绝时，它提取错误文字。两条分支都携带原 attempt 派发 `submit-settled`。

真正的浏览器网络发送不在 `InputMachine`、`default-sink` 对象或 `InputHub` 中。调用继续经过 `ConversationService.sendSession()` 和 Client `Session.prompt()`；最终由 `AbstractApiClient.callUnary('session.prompt')` 组装 RPC envelope，并由 `postJson()` 中的 `doFetch()` 发出 `POST /api/session.prompt`。Host 返回“已接纳”结果后，这个 Promise 才结算并产生 `submit-settled`。这一步不等待 Agent 完成任务，也不等待模型回复；后续网络与执行阶段见 [Web 请求链路与执行模型](web-request-execution-model.zh.md)。

| 所处阶段 | 文本或数据 | 谁使用它 |
|---|---|---|
| 输入框显示 | `比较 @session:abc 的方案` | 用户和 React 页面。 |
| 会话收到的用户消息 | `比较 @[session:abc](dsh-session:<encoded-id>) 的方案` | `session-reference` 插件依靠其中的稳定会话标识找到被引用会话。 |
| 模型请求 | 正文中的 `比较 @session:abc 的方案`，外加被引用会话的内容快照 | 模型同时看到可读问题和实际引用内容。 |

所以原句中的“模型可见文本”不是简单地给 `@session:abc` 换一种显示格式。序列化的目的，是把只有页面 occurrence 才知道的引用身份写入可持久化的用户消息，使后续插件能够解析身份并为模型准备实际上下文。

引用序列化完成时，已销毁的 shell 不再调用发送函数；发送 Promise 完成时，`dead()` 检查 shell 是否已销毁或 attempt 是否已取消，迟到结果不再派发事件。成功结果到达时，shell 还会先从运行时附件列表中移除本次捕获的图片；失败时图片保持不变。

### 5.3 `submit-settled`：提交或保留草稿

`submit-settled` 回到 [`machine.ts:541`](../packages/client/ui-conversation/src/client/input/machine.ts) 的 `onSubmitSettled()` 后，状态机首先要求当前阶段仍是 `submitting`、`inflight` 仍然存在，并且事件携带的 `attempt.seq` 等于当前序号。任一条件不满足都返回空副作用数组，旧请求不能结算较新的提交。校验通过后先释放 `inflight`，再处理结果：

| 结果 | 状态流转 | 草稿、引用和附件 | 后续副作用 |
|---|---|---|---|
| 成功 | `submitting` → `plain` | 清除命令认领和引用；当前草稿等于 `draftSnapshot` 时清空，只多出纯后缀时保留后缀；清空撤销/重做记录。shell 只移除本次捕获的图片标识，运行时列表中的其他标识不受影响。 | `SubmitOutcome` 带文字时产生 `notice`，否则无。 |
| 发送、序列化或网络失败 | `submitting` → `plain` | 普通消息的当前草稿、引用和图片保持不变，不用旧快照覆盖较新的程序化编辑。 | 错误文字存在时产生 error `notice`。 |
| 迟到或已取消 | 状态不变 | 不修改任何输入数据。 | 无。 |

成功分支只保留纯后缀，是因为“已提交快照 + 后来追加文字”可以无歧义拆分；其他交错改写无法可靠判断哪些字符已经被接收。当前 `InputBar` 在 `submitting` 期间本来就只读，这项规则主要防御异步或程序化输入。`notice` 由 shell 同步写入提示快照，`InputBar` 在 [`InputBar.tsx:89`](../packages/client/ui-conversation/src/client/skeleton/InputBar.tsx) 把它渲染为短暂反馈。

因此，完整状态与副作用闭环是：

```text
plain
  └─ enter
       ├─ submitting + inflight attempt
       └─ default-sink
            ├─ capture images
            ├─ serialize references
            ├─ conversation.sendSession(...)
            └─ submit-settled
                 ├─ success → plain + commit captured content
                 ├─ failure → plain + retain current content + notice
                 └─ stale/cancelled → ignore
```

## 6. 其他输入路径

### 6.1 Slash 命令

裁剪后的文本以 `/` 开头时，`onEnter()` 先进入 `adjudicating` 并产生 `adjudicate`。命令匹配后，同一个 attempt 进入 `submitting`；没有匹配时，捕获文本作为普通消息发送；触发器已经自行处理时回到 `plain`；判定出错时保留草稿并产生错误提示。这些分支分别位于 [`machine.ts:494`](../packages/client/ui-conversation/src/client/input/machine.ts)、[`machine.ts:504`](../packages/client/ui-conversation/src/client/input/machine.ts) 和 [`machine.ts:533`](../packages/client/ui-conversation/src/client/input/machine.ts)。

### 6.2 引用与粘贴

引用 occurrence 保存来源拥有的标识和草稿区间，每次编辑都会重新计算这些区间。普通消息发送前，适配层把展示区间替换为各来源序列化后的模型文本；缺少序列化器或序列化失败都会阻止发送并保留草稿。粘贴匹配可以在初始粘贴事务中创建引用，也可以稍后把已匹配文字升级为一项单独可撤销的引用事务。

事件定义及其事务规则位于 [`contract.ts:253`](../packages/client/ui-conversation/src/client/input/contract.ts)，序列化位于 [`facade.ts:449`](../packages/client/ui-conversation/src/client/input/facade.ts)。

### 6.3 图片

纯状态机通过 `InputState` 暴露附件标识，但运行时列表由 `SessionInputShell` 保存，因为文件对象、字节和预览地址需要浏览器侧清理。`adjudicating` 和 `submitting` 期间拒绝添加或移除图片，防止可见附件列表与正在执行的命令序列化不一致。附件操作从 [`facade.ts:126`](../packages/client/ui-conversation/src/client/input/facade.ts) 开始。

### 6.4 会话生命周期

`InputHub` 为每个 `SessionId` 保存一个 `SessionInputShell`。会话实例化时创建适配层并发布状态/操作；scope 释放时取消当前 attempt、注销输入触发监听器、释放剩余图片对象并删除注册表条目。创建和释放逻辑集中在 [`hub.ts:71`](../packages/client/ui-conversation/src/client/input/hub.ts)。

## 7. 代码阅读地图

| 问题 | 阅读入口 |
|---|---|
| 页面能观察或调用什么？ | [`contract.ts:25`](../packages/client/ui-conversation/src/client/input/contract.ts) |
| 状态机保存哪些状态？ | [`machine.ts:116`](../packages/client/ui-conversation/src/client/input/machine.ts) |
| 所有事件从哪里进入？ | [`machine.ts:170`](../packages/client/ui-conversation/src/client/input/machine.ts) |
| Enter 如何分类？ | [`machine.ts:483`](../packages/client/ui-conversation/src/client/input/machine.ts) |
| 成功、失败和新编辑如何合并？ | [`machine.ts:541`](../packages/client/ui-conversation/src/client/input/machine.ts) |
| 网络和插件操作在哪里执行？ | [`facade.ts:425`](../packages/client/ui-conversation/src/client/input/facade.ts) |
| 异步结果怎样回到状态机？ | [`facade.ts:495`](../packages/client/ui-conversation/src/client/input/facade.ts) |
| 每个会话在哪里挂载一个状态机？ | [`hub.ts:71`](../packages/client/ui-conversation/src/client/input/hub.ts) |
| React 从哪里取得快照和操作？ | [`apply.ts:181`](../packages/client/ui-conversation/src/client/apply.ts) |
