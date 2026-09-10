# Agent Note: User-centered Web session flow documentation

Status: implemented

English | [中文](2026-08-28-user-centered-web-session-flow.zh.md)

## Problem

A package-first or event-first account of the Web request path explains execution but leaves the reader to infer the product model. The visible conversation, browser-side Session state, durable Host Session, live Agent, child Session variants, and unrelated Terminal Session all use adjacent terminology. Without a user-need map, readers can mistake page state or live executors for separate conversation records.

## Decision

[`docs/web-session-flow.md`](../../../../docs/web-session-flow.md) first explains the complete experience through the user's needs for delivery feedback, busy-state queueing, continuous tasks, Tool continuity, streaming display, and refresh recovery. Each mechanism states the user problem before a "Name in code" paragraph introduces inbox/claim, Turn/Step, Session, and Agent. The role summary, branch behavior, and code-reading notes come after the main path so package names and types do not precede the needs they serve.

The page also distinguishes Core `Session` from Host: the former owns ordered facts for one conversation, while the latter owns a server runtime containing many sessions and services. This explains why Host may operate on a Session, but Session should not own networking, Workspaces, permissions, or coordination across sessions.

Each user need also carries its technical model: state lifecycle, authority, and cardinality select the input state machine, inbox, Agent, Turn, Step, Core `Session`, Client `Session`, or Host as the owner, followed by the concrete state and operations that object contributes to the main path. A consolidated object-responsibility section supports lookup, while complete APIs and descendant mechanisms remain in their subsystem pages and package READMEs.

The input state machine is a direct child mechanism with its own page at [`docs/web-input-state-machine.md`](../../../../docs/web-input-state-machine.md). That page derives the state and attempt model from composer requirements, distinguishes the pure `InputMachine` from `SessionInputShell` side effects and `InputHub` session ownership, and links each behavior to its source location. The main flow keeps only the responsibility summary and links to that owner.

The network and execution path is a sibling reference at [`docs/web-request-execution-model.md`](../../../../docs/web-request-execution-model.md). It derives the separation of browser-to-Host admission, Host-to-provider model calls, and Host-to-browser event delivery from user and performance needs for submission feedback, long tasks, streaming display, refresh recovery, multi-session routing, credential protection, uncertain admission results, and cross-site access control. It also makes explicit that input settlement ends at Host admission rather than model completion. Its thread chapter then derives four execution paths, synchronous main-thread segments, asynchronous I/O, worker threads, and subprocesses, from needs for page responsiveness, progress across sessions, ordering within one session, tool latency, blocking isolation, and resource release. An ordinary-message example shows one request switching between the main thread and I/O; a separate tool example maps network tools, Code Runtime, Shell-style tools, and synchronous plugins to their paths. The thread-versus-network comparison starts from user-observable problems and separates execution blocking from transport failures. Provider-specific protocol detail and Typert generation remain in their package README and API Gateway owners.

The page remains an ordered tutorial rather than an exhaustive subsystem reference. Exact types, event order, persistence rules, and provider-specific behavior stay in their owning subsystem pages and package READMEs. The session-classification section determines identity, history inheritance, navigation, and resume rules and leads into Host creation and resume, rather than listing types without a purpose.

## Alternatives considered

- **Start with the Web profile and package composition** — rejected because startup is a prerequisite, not the user's first concept, and it hides why Session and Agent are separate.
- **Start with the RPC and Agent-loop call chain** — rejected because it explains control flow before defining the user-visible object whose behavior the calls implement.
- **Call every retained activity a session** — rejected because Turn, Step, WorkflowRun, and Terminal Session have different identity, durability, and navigation semantics.

## Consequences

Readers can relate a sidebar conversation to its Client Session, Host Session, and Agent before following the request path. The Host and WebSocket section makes connection timing, stream separation, and reconnect value explicit; the inbox, Turn, and Step sections explain how queued input and multiple model operations form one observable user task, distinguishing pending intent, task boundaries, and individual requests; and the code-ownership table prevents API, state, and execution responsibilities from being conflated during code reading. Future edits preserve the user-need-first order and link detailed mechanics to their owners instead of expanding this page into a package catalog or event reference.

Readers who need to change composer behavior can follow the child page from a visible failure or success case to the state transition, effect executor, per-session registry, and UI feedback without turning the main session tutorial into an exhaustive input reference.

Readers who need to change network behavior can distinguish local attempt identity, RPC correlation, and durable event order before locating the browser fetch, Host admission, provider fetch, or WebSocket downlink that owns the change.
