# Agent Note: 以用户为中心的 Web 会话流程文档

Status: implemented

[English](2026-08-28-user-centered-web-session-flow.md) | 中文

## 问题

从包或事件开始介绍 Web 请求路径可以解释执行过程，却要求读者自行推断产品模型。用户看到的对话、浏览器中的页面会话状态、持久 Host Session、活跃 Agent、子 Session 变体和与对话无关的 Terminal Session 使用相近术语。缺少用户需求映射时，读者容易把页面状态或活跃执行器误认为另一份对话记录。

## 决策

[`docs/web-session-flow.md`](../../../../docs/web-session-flow.zh.md)先按用户需要的送达反馈、忙时排队、连续任务、工具衔接、流式显示和刷新恢复解释完整体验。每个机制先说明用户问题，再用“代码中的名称”引入 inbox/claim、Turn/Step、Session 和 Agent；角色总览、分支行为与代码阅读说明放在主流程之后，避免初读者在理解需求之前先记包名和类型。

页面还明确区分 Core `Session` 与 Host：前者拥有一段对话的有序事实，后者拥有包含多个会话和服务的服务端运行环境；这种边界解释了为什么 Session 可以被 Host 操作，却不应承担网络、Workspace、权限和多会话协调。

每个用户需求同时给出技术建模：先按状态的生命周期、可信度和数量关系选择输入状态机、inbox、Agent、Turn、Step、Core `Session`、Client `Session` 或 Host 作为拥有者，再列出该对象为主链路承担的具体状态和操作。集中式对象职责卡支持按对象查阅，但完整 API 和下级机制仍链接到各自的子系统页面与包 README。

输入状态机作为直接下级机制，由 [`docs/web-input-state-machine.md`](../../../../docs/web-input-state-machine.zh.md) 单独说明。该页面从输入框需求推导状态和 attempt 模型，区分纯 `InputMachine`、负责副作用的 `SessionInputShell` 和负责会话实例的 `InputHub`，并把各项行为链接到源码位置。主流程只保留职责摘要并链接到该归属文档。

网络与执行路径由同级参考页 [`docs/web-request-execution-model.md`](../../../../docs/web-request-execution-model.zh.md) 说明。该页从提交反馈、长任务、流式显示、刷新恢复、多会话路由、凭据保护、不明确接纳结果和跨站访问等用户与性能要求，推导浏览器到 Host 的接纳、Host 到 provider 的模型调用以及 Host 到浏览器的事件投递为什么必须分离，并明确输入结算结束于 Host 接纳而不是模型完成。线程章节再从页面响应、跨会话推进、单会话顺序、工具延迟、阻塞隔离和资源释放推导主线程同步片段、异步 I/O、worker thread 与 subprocess 四种路径；普通消息示例展示一次请求如何在主线程和 I/O 之间切换，工具示例再分别说明网络工具、Code Runtime、Shell 类工具与同步插件的路径。线程模型与网络架构的对照以用户可观察的问题为入口，分别定位执行阻塞和传输故障。提供方特有协议和 Typert 生成细节仍由各自的包 README 与 API Gateway 文档负责。

该页面始终是一篇有序教程，而不是完整的子系统参考。精确类型、事件顺序、持久化规则和提供方特有行为继续由各自的子系统页面和包 README 负责。会话分类章节用于确定身份、历史继承、导航和恢复规则，并承接到 Host 的创建/恢复流程，而不是单纯罗列类型。

## 曾考虑的替代方案

- **从 Web profile 和包组装开始**：否决。启动是运行前提，而不是用户需要建立的第一个概念，并且这种顺序掩盖了 Session 与 Agent 分离的原因。
- **从 RPC 和 Agent loop 调用链开始**：否决。这种顺序会在定义调用所实现的用户可见对象之前先解释控制流。
- **把所有保留状态的活动都称为会话**：否决。Turn、Step、WorkflowRun 和 Terminal Session 具有不同的身份、持久性和导航语义。

## 后果

读者可以先把一个侧边栏对话与其 Client Session、Host Session 和 Agent 对应起来，再沿着请求路径阅读。Host 与 WebSocket 小节让连接建立时机、下行流分工和断线重建的用户价值可见；inbox、Turn 和 Step 小节解释输入排队与多次模型处理如何组成一项可观察的用户工作，并区分待处理意图、任务边界和单次请求；代码归属表解释按文件阅读时的责任边界。后续修改保持用户需求优先的顺序，并把详细机制链接到各自归属文档，而不会把本页扩展成包清单或事件参考。

需要修改输入框行为的读者可以从子页面中的可见失败或成功场景，一直定位到状态转换、副作用执行器、按会话注册表和 UI 提示，而不必把会话主教程扩展成完整输入参考。

需要修改网络行为的读者可以先区分本地 attempt 身份、RPC 关联和持久事件顺序，再定位到拥有该变化的浏览器 fetch、Host 接纳、provider fetch 或 WebSocket 下行。
