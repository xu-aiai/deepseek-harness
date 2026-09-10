# Web 轨迹视图

[English](web-trajectory-view.md) | 中文

用户在对话页打开“轨迹”，通常不是为了阅读底层事件，而是想检查一项任务到底经历了什么：模型收到了哪些输入，为什么调用某个工具，等待发生在哪里，最终结果来自哪一步。本文先从这些用户需求出发，再说明系统把它们建模成哪些业务对象，最后把每个对象对应到具体实现代码。

消息从网页送达 Agent、写入 Session 并返回浏览器的完整过程见 [从网页输入到 Agent 回复](web-session-flow.zh.md)。本文只讨论浏览器如何把已经发生的 Session 事实整理成轨迹视图。

## 用户需求

以“分析这个项目”为例，一项任务可能包含用户输入、运行时上下文、第一次模型请求、多个工具调用、后续模型请求和最终回答。普通对话视图适合阅读结果，但不足以回答下面的问题。

### 需求一：按任务理解过程，而不是阅读事件流水

用户需要看出哪些记录属于同一项任务，以及一次模型请求结束后为什么又发生下一次请求。页面因此保留 Turn 和 Request 边界，但不为每个边界增加一张占空间的卡片。

### 需求二：检查模型实际看到了什么

用户需要查看某次请求生效的 System Prompt、工具目录和输入消息，还需要知道它们相对上一次请求是否发生变化。轨迹不能只显示聊天正文，也不能用当前配置代替历史请求当时真正使用的内容。

### 需求三：把模型输出和工具操作连起来

用户需要知道工具由哪次模型输出触发、参数是什么、结果或错误是什么，以及一个工具内部是否又调用了子工具。流式回答尚未结束时，页面还应显示正在生成的 Assistant 和正在执行的工具。

### 需求四：看清时间花在哪里

用户需要区分输入准备、等待首个 token、后续生成和工具执行时间，并比较不同 Turn 或 Request。页面上方的 Overview 因此使用 Input、Model 和 Tools 三条轨道；用户可以切换顺序、耗时和实际时间投影，并用区间选择聚焦下方记录。

### 需求五：长会话仍然可以检查

用户需要搜索、折叠 Turn 或 Assistant、加载更早历史，并在实时输出继续到达时停留在正在检查的位置。页面不能因为完整会话很长就一次性加载和挂载全部记录。

### 需求六：需要原始证据时可以导出

轨迹适合交互式检查，但它不是原始日志下载器。用户点击页头的 `Session log` 时，系统单独导出 Session 记录和附件；该功能不依赖轨迹当前加载了多少历史，也不受搜索、折叠或时间区间筛选影响。

## 业务建模

这些需求没有被建模成一张“事件表”。系统先区分用户要检查的实体、实体之间的关系，以及浏览器当前掌握的数据范围。

```text
Loaded Session window
└─ Turn: one user task
   ├─ Message: user input or runtime context
   ├─ Request / Step: one model request
   │  ├─ Prompt state/change: effective System Prompt and Tool catalog
   │  ├─ Assistant: model output
   │  └─ Tool call: Tool execution
   │     └─ Subtool: nested Tool execution
   └─ Turn end: task ending state
```

### 轨迹记录

`Trajectory record` 是用户可以在记录表中选择和检查的最小单位。它可能是 System Prompt、USER、CONTEXT、ASSISTANT、TOOL 或压缩记录。每条记录都有稳定标识；搜索结果、时间线选区、折叠状态和详情选择都引用这个标识，而不是引用可能尚未挂载的 DOM 行。

### Turn 与 Request

`Turn` 表示一项用户任务，负责把用户输入、若干次模型请求、工具活动和结束状态归在一起。`Request` 表示一次真实的模型调用；普通生成请求和 compaction 请求共用一套按时间排序的编号。代码中的 Step 是普通模型处理的运行单位，轨迹在展示层把对应的请求边界呈现给用户。

### Prompt state 与 Prompt change

`Prompt state` 保存一次 Request 真正使用的 System Prompt 和工具目录。`Prompt change` 描述它相对前一次 Request 是初始状态、System Prompt 变化、工具目录变化，还是两者都变化。它附着于引入变化的 Request，而不是作为脱离请求的独立日志行。

### Tool call 与 Subtool

`Tool call` 把调用参数、执行结果、错误、计时和工具 schema 关联到同一个稳定调用标识。工具内部产生的调用建模为 `Subtool`，通过父子关系显示缩进和层级，而不是与顶层工具混在一起。

### Time span

可计时记录使用 `startedAt` 与 duration 表示活动区间。Overview 将记录映射到 Input、Model、Tools 三条轨道。Assistant 完成后还可以用首个非空 token 的时刻，把总区间分成 TTFT 和 decoding 两段；计时信息不完整时只显示一个 Assistant 区间，不推测缺失数据。

### 已加载历史窗口

浏览器只持有一段连续的 Session Event 窗口。它包含当前已加载范围、是否还有更早历史和实时追加的尾部。轨迹中的请求编号、累计用量、搜索范围和时间范围都只描述这个窗口；加载更早一页后，这些投影随窗口扩展。

### 记录检查器

`Record inspector` 是选中记录的局部详情面板。消息可以显示 Markdown、来源和图片；工具可以显示参数、结果和 schema；Request 可以显示选项、用量、计时以及结果跳转。它属于轨迹自身，不与 Chat 的会话级详情栏共享选择状态。

轨迹不是服务端额外保存的第二份日志。这些对象都是浏览器根据同一份 Session Event 窗口计算出的可丢弃投影；刷新或补页时可以重新构建。

## 技术实现

实现分为四步：先得到连续事件窗口，再把事件归并为业务上下文，然后生成轨迹快照，最后派生时间线、记录表和检查器所需的页面模型。

### 第一步：维护连续的 Session 事件窗口

Client `Session` 打开会话时获取最近一页历史；实时事件到达后按 `seq` 去重并追加。如果发现序号缺口，它先缓存新事件，再重新获取尾页进行修复。用户向前加载历史时，更早事件被 prepend 到同一个连续窗口。入口分别位于 [`session.ts:618`](../packages/client/runtime/src/client/sessions/session.ts)、[`session.ts:657`](../packages/client/runtime/src/client/sessions/session.ts)、[`session.ts:672`](../packages/client/runtime/src/client/sessions/session.ts) 和 [`session.ts:683`](../packages/client/runtime/src/client/sessions/session.ts)。

这一步只回答“浏览器掌握哪些有序事实”，还没有生成 SYSTEM、ASSISTANT 或 TOOL 行。

### 第二步：把事件归并为业务上下文

公共抽象 `ConversationNodeDefinition` 声明某类事件如何匹配稳定业务 ID、如何开始或更新状态，以及如何产生 View Node，定义见 [`conversation.ts:171`](../packages/client/runtime/src/client/contract/conversation.ts)。`ConversationNodeAssembler` 根据这些 Definition 增量重放受影响的 Context，并为 Chat 和 Trajectory 分别建立 View Builder，核心组装位置见 [`conversation-assembler.ts:707`](../packages/client/runtime/src/client/sessions/conversation-assembler.ts) 和 [`conversation-assembler.ts:775`](../packages/client/runtime/src/client/sessions/conversation-assembler.ts)。

轨迹插件在 [`index.ts:30`](../packages/client/ui-trajectory/src/client/index.ts) 注册自己的全部 Definition、Builder 和页面标签。各业务对象的归并规则如下。

| 用户看到的对象 | 事件如何被解释 | 实现入口 |
|---|---|---|
| USER / CONTEXT / steering | 根据 `user/message` 的来源以及消息是否在当前边界被 claim 分类 | [`trajectory-message-definitions.ts:66`](../packages/client/ui-trajectory/src/client/trajectory-message-definitions.ts) |
| Prompt state/change | 读取 `request/header`，保存本次 prompt，并与前一次请求比较 | [`trajectory-request-header-definition.ts:45`](../packages/client/ui-trajectory/src/client/trajectory-request-header-definition.ts) |
| Assistant / Request | 把 Step、流式 chunk、最终消息、retry、用量和结束状态归到同一请求 | [`trajectory-assistant-definition.ts:279`](../packages/client/ui-trajectory/src/client/trajectory-assistant-definition.ts) |
| Tool / Subtool | 按稳定 call id 关联调用、结果与父子工具树 | [`trajectory-tool-definition.ts:219`](../packages/client/ui-trajectory/src/client/trajectory-tool-definition.ts) |
| Compaction / Session end | 把压缩请求和会话结束状态放入同一条时间序列 | [`trajectory-compaction-definition.ts:80`](../packages/client/ui-trajectory/src/client/trajectory-compaction-definition.ts) |

流式 token 只更新命中的 Assistant Context。发布在 [`trajectory-assistant-definition.ts:337`](../packages/client/ui-trajectory/src/client/trajectory-assistant-definition.ts) 合并为每个 animation frame 最多一次，避免每个 token 都触发整页 React 更新。

### 第三步：生成统一轨迹快照

`TrajectorySnapshot` 定义页面消费的数据：已完成节点、事件位置、请求、工具 schema、未完成 Assistant 和正在运行的工具，见 [`trajectory-contract.ts:61`](../packages/client/ui-trajectory/src/client/trajectory-contract.ts)。`TrajectorySnapshotBuilder` 接收各 Definition 产生的 View Node，再把它们汇总为一个快照，见 [`trajectory-snapshot-builder.ts:138`](../packages/client/ui-trajectory/src/client/trajectory-snapshot-builder.ts) 和 [`trajectory-snapshot-builder.ts:175`](../packages/client/ui-trajectory/src/client/trajectory-snapshot-builder.ts)。

Builder 让事件解释与页面排版分开：新增一种 Session 事实时，Definition 决定它属于哪个业务对象；轨迹表格不需要直接 switch 原始事件类型。

### 第四步：派生页面布局与交互

`deriveTrajectoryLayout()` 把节点、请求、提示词变化、未完成 Assistant 和运行中工具合并为按事件顺序排列的 Turn、Request group 与 record，入口见 [`layout.ts:138`](../packages/client/ui-trajectory/src/client/layout.ts)，有序合并从 [`layout.ts:218`](../packages/client/ui-trajectory/src/client/layout.ts) 开始。

`deriveTrajectoryTimeline()` 把相同记录投影到三条时间轨道，并支持顺序、等宽耗时、压缩空闲时间和完整实际时间几种模式，见 [`timeline.ts:70`](../packages/client/ui-trajectory/src/client/timeline.ts)。区间选择通过 [`timeline.ts:189`](../packages/client/ui-trajectory/src/client/timeline.ts) 转换为一组 record index，再过滤下方记录表；没有可靠计时的记录不会被强行放进计时选区。

`TrajectoryView` 从 `snapshot.views.get('trajectory')` 读取快照，建立搜索索引和布局，并组装 Toolbar、Overview 与 Table，入口见 [`TrajectoryView.tsx:120`](../packages/client/ui-trajectory/src/client/TrajectoryView.tsx)，三个页面区域的组合见 [`TrajectoryView.tsx:448`](../packages/client/ui-trajectory/src/client/TrajectoryView.tsx)。

`TrajectoryTable` 负责 Turn/Assistant 折叠、记录选择、详情标签页、向前分页和滚动位置。它通过 `@tanstack/react-virtual` 只挂载视口附近的行，入口见 [`TrajectoryTable.tsx:1693`](../packages/client/ui-trajectory/src/client/TrajectoryTable.tsx)，虚拟化器创建于 [`TrajectoryTable.tsx:1786`](../packages/client/ui-trajectory/src/client/TrajectoryTable.tsx)。实时记录只在用户已经跟随末尾时推动滚动位置；用户向上检查旧记录后，新 token 不会把页面拉回底部。

### Session log 下载是独立旁路

页头按钮由 [`HeaderAction.tsx:11`](../packages/session-query/session-log-export/src/client/HeaderAction.tsx) 提供。下载控制器调用 `/api/session.export`，见 [`controller.ts:57`](../packages/session-query/session-log-export/src/client/controller.ts)；服务端在 [`session-export.ts:219`](../packages/host/apiproxy/src/session-export.ts) 按顺序生成 ZIP 条目并流式返回。它读取可导出的 Session 数据，不复用 `TrajectorySnapshot`，因此导出结果不受当前视图状态影响。

## 从一个需求追到代码

以“查看某个工具为什么耗时”为例，可以按下面的顺序阅读：

1. `tool/call` 与 `tool/result` 进入 Client `Session` 的连续事件窗口。
2. Tool Definition 用 call id 把调用、结果、计时和子工具组装为 Tool Context。
3. `TrajectorySnapshotBuilder` 把 Tool Context 放入轨迹快照。
4. `deriveTrajectoryLayout()` 生成 TOOL 或 Subtool record。
5. `deriveTrajectoryTimeline()` 根据 `startedAt + duration` 在 Tools 轨道生成 span。
6. 用户选择 span 或记录后，`TrajectoryTable` 用同一稳定 record id 打开参数、结果、schema 和计时详情。

这条路径体现了实现的核心分工：Session 保存事实窗口，Definition 解释业务含义，Builder 形成视图快照，纯派生函数生成页面模型，React 组件只负责交互和渲染。
