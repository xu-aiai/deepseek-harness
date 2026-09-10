# Web Trajectory View

English | [中文](web-trajectory-view.zh.md)

Users open Trajectory on a conversation page not to read low-level events, but to inspect what a task actually did: what the model received, why it called a Tool, where it waited, and which step produced the final result. This page starts from those user needs, explains the business objects derived from them, and then maps each object to its implementation.

See [From Web Input to Agent Reply](web-session-flow.md) for the complete path that delivers a message to the Agent, records it in the Session, and returns it to the browser. This page covers only how the browser organizes recorded Session facts into the Trajectory view.

## User needs

Consider a request to "analyze this project." One task may include user input, runtime context, an initial model request, several Tool calls, later model requests, and a final answer. Chat is suitable for reading the result, but it does not answer the following questions by itself.

### Need 1: understand work by task, not as an event stream

The user needs to see which records belong to the same task and why another model request followed the previous one. The page therefore preserves Turn and Request boundaries without adding a space-consuming card for every boundary.

### Need 2: inspect what the model actually saw

The user needs to inspect the System Prompt, Tool catalog, and input messages effective for a particular request, and whether they changed from the previous request. Trajectory cannot show only chat prose or substitute the current configuration for what a historical request actually used.

### Need 3: connect model output to Tool activity

The user needs to know which model output initiated a Tool, its arguments, its result or error, and whether the Tool invoked nested Tools. While streaming is incomplete, the page must also show the Assistant still being generated and Tools still running.

### Need 4: see where time was spent

The user needs to distinguish input preparation, time to first token, later generation, and Tool execution, and compare different Turns or Requests. The Overview above the ledger therefore uses Input, Model, and Tools lanes. Users can switch among sequence, duration, and actual-time projections and select an interval to focus the records below.

### Need 5: inspect long conversations

The user needs to search, fold a Turn or Assistant, load older history, and remain at the inspected location while live output continues. The page cannot load and mount every record at once merely because the complete conversation is long.

### Need 6: export original evidence when needed

Trajectory supports interactive inspection, but it is not the raw-log downloader. When the user clicks `Session log` in the header, the system separately exports Session records and attachments. That operation does not depend on how much history Trajectory has loaded and is unaffected by search, folding, or timeline focus.

## Business model

These needs are not modeled as one "event table." The system first distinguishes the entities the user inspects, their relationships, and the range of data currently available to the browser.

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

### Trajectory record

A `Trajectory record` is the smallest item a user can select and inspect in the ledger. It may be a System Prompt, USER, CONTEXT, ASSISTANT, TOOL, or compaction record. Every record has a stable identity. Search results, timeline focus, fold state, and detail selection refer to that identity rather than to a DOM row that might not be mounted.

### Turn and Request

A `Turn` represents one user task and groups its user input, model requests, Tool activity, and ending state. A `Request` represents one actual model call. Ordinary generation and compaction requests share one chronological numbering space. A Step is the runtime unit for ordinary model processing; Trajectory presents its corresponding request boundary to the user.

### Prompt state and Prompt change

`Prompt state` retains the System Prompt and Tool catalog actually used by one Request. `Prompt change` identifies whether it is the initial state or whether the System Prompt, Tool catalog, or both changed from the preceding Request. It belongs to the Request that introduced the change instead of becoming a detached log row.

### Tool call and Subtool

A `Tool call` associates arguments, execution result, error, timing, and Tool schema with one stable call identity. Calls produced inside a Tool are modeled as `Subtool` records and displayed through parent-child indentation rather than mixed with top-level Tools.

### Time span

A record with timing uses `startedAt` and duration as its active interval. The Overview maps records to Input, Model, and Tools lanes. After an Assistant completes, the first non-empty token time can split its interval into TTFT and decoding segments. When timing is incomplete, the page shows one Assistant interval and does not invent missing data.

### Loaded history window

The browser holds one contiguous Session Event window. It records the loaded range, whether older history exists, and the live-appended tail. Trajectory request numbers, cumulative usage, search domain, and time domain describe only this window. Those projections expand when another older page is loaded.

### Record inspector

The `Record inspector` is a local detail panel for the selected record. Messages can expose Markdown, source fields, and images; Tools can expose arguments, results, and schema; Requests can expose options, usage, timing, and result navigation. It belongs to Trajectory and does not share selection state with Chat's conversation-wide details column.

Trajectory is not a second log stored by the server. These objects are disposable browser projections computed from the same Session Event window and can be rebuilt after refresh or pagination.

## Technical implementation

Implementation has four stages: obtain a contiguous event window, assemble events into business contexts, build a Trajectory snapshot, and finally derive the timeline, ledger, and inspector page models.

### Stage 1: maintain a contiguous Session Event window

Client `Session` fetches the latest history page when opening a conversation. Live events are deduplicated by `seq` and appended. When a sequence gap appears, it buffers the new events and refetches the tail page to repair the gap. Loading backward prepends older events into the same contiguous window. The entry points are [`session.ts:618`](../packages/client/runtime/src/client/sessions/session.ts), [`session.ts:657`](../packages/client/runtime/src/client/sessions/session.ts), [`session.ts:672`](../packages/client/runtime/src/client/sessions/session.ts), and [`session.ts:683`](../packages/client/runtime/src/client/sessions/session.ts).

This stage answers only "which ordered facts does the browser have?" It has not yet produced SYSTEM, ASSISTANT, or TOOL rows.

### Stage 2: assemble events into business contexts

The common `ConversationNodeDefinition` abstraction declares how one event family matches a stable business ID, starts or updates state, and produces a View Node. It is defined at [`conversation.ts:171`](../packages/client/runtime/src/client/contract/conversation.ts). `ConversationNodeAssembler` incrementally replays affected Contexts from those Definitions and creates separate View Builders for Chat and Trajectory. Its main assembly points are [`conversation-assembler.ts:707`](../packages/client/runtime/src/client/sessions/conversation-assembler.ts) and [`conversation-assembler.ts:775`](../packages/client/runtime/src/client/sessions/conversation-assembler.ts).

The Trajectory plugin registers all of its Definitions, Builder, and page tab at [`index.ts:30`](../packages/client/ui-trajectory/src/client/index.ts). Its business-object assembly rules are:

| User-visible object | Event interpretation | Implementation entry |
|---|---|---|
| USER / CONTEXT / steering | Classify `user/message` from its source and whether the message was claimed at the current boundary | [`trajectory-message-definitions.ts:66`](../packages/client/ui-trajectory/src/client/trajectory-message-definitions.ts) |
| Prompt state/change | Read `request/header`, retain its prompt, and compare it with the preceding Request | [`trajectory-request-header-definition.ts:45`](../packages/client/ui-trajectory/src/client/trajectory-request-header-definition.ts) |
| Assistant / Request | Associate the Step, streaming chunks, final message, retry, usage, and ending state with one request | [`trajectory-assistant-definition.ts:279`](../packages/client/ui-trajectory/src/client/trajectory-assistant-definition.ts) |
| Tool / Subtool | Associate calls, results, and the parent-child Tool tree by stable call id | [`trajectory-tool-definition.ts:219`](../packages/client/ui-trajectory/src/client/trajectory-tool-definition.ts) |
| Compaction / Session end | Place compaction requests and Session ending state into the same chronology | [`trajectory-compaction-definition.ts:80`](../packages/client/ui-trajectory/src/client/trajectory-compaction-definition.ts) |

Streaming tokens update only the matching Assistant Context. Publication at [`trajectory-assistant-definition.ts:337`](../packages/client/ui-trajectory/src/client/trajectory-assistant-definition.ts) is coalesced to at most once per animation frame so every token does not trigger a complete React update.

### Stage 3: build one Trajectory snapshot

`TrajectorySnapshot` defines the data consumed by the page: finalized nodes, event locations, Requests, Tool schemas, the partial Assistant, and running Tools. See [`trajectory-contract.ts:61`](../packages/client/ui-trajectory/src/client/trajectory-contract.ts). `TrajectorySnapshotBuilder` receives the View Nodes produced by all Definitions and combines them into one snapshot. See [`trajectory-snapshot-builder.ts:138`](../packages/client/ui-trajectory/src/client/trajectory-snapshot-builder.ts) and [`trajectory-snapshot-builder.ts:175`](../packages/client/ui-trajectory/src/client/trajectory-snapshot-builder.ts).

The Builder separates event interpretation from page layout. When a new Session fact is introduced, a Definition decides which business object owns it; the Trajectory ledger does not directly switch over raw event types.

### Stage 4: derive page layout and interaction

`deriveTrajectoryLayout()` merges nodes, Requests, prompt changes, the partial Assistant, and running Tools into event-ordered Turns, Request groups, and records. Its entry is [`layout.ts:138`](../packages/client/ui-trajectory/src/client/layout.ts), and the ordered merge begins at [`layout.ts:218`](../packages/client/ui-trajectory/src/client/layout.ts).

`deriveTrajectoryTimeline()` projects the same records onto three timing lanes and supports sequence, equal-width duration, idle-compressed time, and complete actual-time modes. See [`timeline.ts:70`](../packages/client/ui-trajectory/src/client/timeline.ts). Interval selection is converted into a set of record indexes by [`timeline.ts:189`](../packages/client/ui-trajectory/src/client/timeline.ts), which then filters the ledger below. Records without reliable timing are not forced into a timed selection.

`TrajectoryView` reads `snapshot.views.get('trajectory')`, builds the search index and layout, and composes Toolbar, Overview, and Table. Its entry is [`TrajectoryView.tsx:120`](../packages/client/ui-trajectory/src/client/TrajectoryView.tsx); the three page regions are composed at [`TrajectoryView.tsx:448`](../packages/client/ui-trajectory/src/client/TrajectoryView.tsx).

`TrajectoryTable` owns Turn and Assistant folding, record selection, detail tabs, backward pagination, and scroll position. It uses `@tanstack/react-virtual` to mount only rows near the viewport. Its entry is [`TrajectoryTable.tsx:1693`](../packages/client/ui-trajectory/src/client/TrajectoryTable.tsx), and the virtualizer is created at [`TrajectoryTable.tsx:1786`](../packages/client/ui-trajectory/src/client/TrajectoryTable.tsx). Live records move the scroll position only while the user is already following the tail. After the user scrolls upward to inspect older records, new tokens do not pull the page back to the bottom.

### Session log download is a separate path

The header button is provided by [`HeaderAction.tsx:11`](../packages/session-query/session-log-export/src/client/HeaderAction.tsx). Its controller calls `/api/session.export`; see [`controller.ts:57`](../packages/session-query/session-log-export/src/client/controller.ts). The server yields ordered ZIP entries and streams them from [`session-export.ts:219`](../packages/host/apiproxy/src/session-export.ts). This path reads exportable Session data and does not reuse `TrajectorySnapshot`, so the export is unaffected by current view state.

## Trace one need into code

For "inspect why one Tool took time," follow this sequence:

1. `tool/call` and `tool/result` enter Client `Session`'s contiguous event window.
2. Tool Definition associates the call, result, timing, and Subtools into a Tool Context by call id.
3. `TrajectorySnapshotBuilder` places that Tool Context in the Trajectory snapshot.
4. `deriveTrajectoryLayout()` produces a TOOL or Subtool record.
5. `deriveTrajectoryTimeline()` produces a span in the Tools lane from `startedAt + duration`.
6. After the user selects the span or record, `TrajectoryTable` opens arguments, result, schema, and timing details using the same stable record id.

This path shows the core division of responsibility: Session retains the fact window, Definitions interpret business meaning, the Builder forms the view snapshot, pure derivation functions produce page models, and React components handle only interaction and rendering.
