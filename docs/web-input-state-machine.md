# Web Input State Machine

English | [中文](web-input-state-machine.zh.md)

The input state machine owns what happens between editing the composer and the server accepting or rejecting that input. It does not own the conversation history or model execution.

This page moves from user behavior to code responsibilities. "User behavior and system constraints" explains why one state machine is needed; "State model" explains what the system retains; "Events in and effects out" explains how state changes and external operations run; and "Implementation layers" maps those concepts to objects and source files. Later sections use ordinary messages, Slash commands, references, images, and the session lifecycle to verify the concrete paths covered by the model.

For the larger path after acceptance, see [From Web Prompt to Agent Reply](web-session-flow.md). For the browser request, Host admission, model request, event return path, and execution model, see [Web Request Flow and Execution Model](web-request-execution-model.md).

## 1. User behavior and system constraints

The composer must satisfy several requirements that interact with each other:

1. A failed send must leave the text and attachments available for correction and retry.
2. A successful send must remove committed content, while a late result must never erase newer draft state produced by another event.
3. Repeated Enter presses and late asynchronous results must not settle the same submission twice.
4. Ordinary messages, `/` commands, references, pasted content, and image attachments must share one editing and submission path.
5. Undo and redo must treat one edit, paste, or reference insertion as one user operation; accepted content must not return through Undo.
6. Each open conversation needs an independent draft and submission phase, and closing that conversation must cancel its unfinished work.

These requirements are coupled. Clearing the textarea directly after a request returns, for example, can erase newer text; letting command detection call the server independently can bypass the same failure and cancellation rules. The system therefore models composer changes as events processed by one per-session state machine.

## 2. State model

The state names describe whether the composer can accept another user action, not what the Agent is doing. Three concrete workflows show the distinctions.

### 2.1 Sending an ordinary message

The user types "Summarize this project." The composer is in `plain`: text and images remain editable, and the user can undo, paste, or insert a reference.

After the user presses Enter, the composer enters `submitting`. It still displays "Summarize this project" but is temporarily read-only. This internal intermediate state means the system is preparing references and images and waiting for the server to confirm acceptance; it does not mean that the model is answering.

- Server acceptance succeeds: the composer clears and returns to `plain`, ready for the next message.
- The server rejects the input or the network fails: the text and images remain, and the composer returns to `plain` for correction and retry.
- The user presses Enter repeatedly: only the first press has an effect because `submitting` rejects another submission.

This path is `plain` → `submitting` → `plain`.

### 2.2 Selecting a command from the menu

The user types `/` and selects `/goal`, a command that expects more input. The composer displays `/goal ` and its argument hint, then enters `claimed`. This state means only that the user explicitly selected `/goal` and that the system is waiting for the objective. The command has not executed.

The user can continue with `/goal Fix the login failure`. Deleting or changing the leading `/goal ` cancels the selection and returns to ordinary `plain` editing. Pressing Enter enters `submitting` and passes "Fix the login failure" to the `/goal` handler.

- The command succeeds: the composer clears and returns to `plain`.
- The command fails while the `/goal` draft remains unchanged: the composer returns to `claimed`, retaining the arguments for correction or retry.
- The command fails after another event has changed the draft: the composer returns to `plain`, preventing changed ordinary text from being sent to the old command.

The usual path is `plain` → `claimed` → `submitting` → `plain`. A failed command may instead return from `submitting` to `claimed`.

Not every menu command enters `claimed`. `/compact` accepts no further user input and executes immediately after selection. The system removes its token from the composer, starts compaction, and leaves the composer in `plain`. `/compact` therefore cannot illustrate `claimed`; only commands that declare further input, such as `/goal` or `/feedback`, need this state.

### 2.3 Typing a Slash command directly

The user does not select from the menu, but types `/goal Fix the login failure` directly and presses Enter. The system cannot assume from the text alone that it is a command: no plugin may have registered it, or a trigger may handle it itself. The composer therefore enters `adjudicating` first.

`adjudicating` is an internal intermediate state that the user normally sees only briefly. The composer is temporarily read-only while the system asks the loaded command sources which one handles the text. This prevents the same Enter press from both executing a command and sending an ordinary message.

- An input-accepting command such as `/goal` matches: enter `submitting` and invoke that command handler.
- An immediate command such as `/compact` matches: start it on the detached execution path, remove its token, and return the composer directly to `plain`.
- No command claims the text: enter `submitting` and send the original text as an ordinary message.
- A trigger already completed the action: return to `plain` without another send.
- Adjudication fails: retain the text, return to `plain`, and show the error.

This path is `plain` → `adjudicating` → `submitting` → `plain`, or it may return directly from `adjudicating` to `plain`.

### 2.4 Four-phase quick reference

| Phase | User's current moment | Composer |
|---|---|---|
| `plain` | Writing an ordinary message or revising content retained after failure. | Editable. |
| `claimed` | Selected an input-accepting command such as `/goal` from the menu and is entering arguments; the command has not executed. | Editable; the command selection remains active. |
| `adjudicating` | Pressed Enter and the system is deciding who handles directly typed `/...` text. | Temporarily read-only; internal intermediate state. |
| `submitting` | Pressed Enter and the system is preparing content and waiting for server acceptance. | Temporarily read-only; internal intermediate state. |

A successful submission only means that the server accepted the input. Later queue, Turn, Step, and Agent-reply state belongs to the session flow, not these four phases.

### 2.5 What else the machine remembers

The page consumes `InputState`, defined at [`contract.ts:213`](../packages/client/ui-conversation/src/client/input/contract.ts). In addition to the `phase` values above, the machine remembers:

| Data | Concrete example | Problem it solves |
|---|---|---|
| `draft` | "Summarize this project" in the composer. | Retains the original text after failure and removes only the submitted content after success. |
| `imageIds` | `a.png` followed by `b.png`. | Retains both images after failure and removes only images included in the successful submission. The browser runtime separately owns their files and preview URLs. |
| `draftRev` | An internal counter incremented whenever the draft changes; users do not see it. | Gives plugin-observed character positions a validity marker so stale positions cannot modify a changed draft. |
| `claim` | The selected `/goal` command, its argument hint, and its handler. | Tells Enter which command to invoke; deleting the command token immediately invalidates the old selection. The machine retains the handler internally, while the page sees only display information. |
| `occurrences` | An `@session:abc` reference occupies characters 8 through 20. | Typing before it moves its range, and submission converts it to the model text required by its reference source. |
| `paste` | Newly pasted text is still being checked for references. | A late match upgrades text only while the draft and selection still correspond to that paste; further user editing makes the old result harmless. |
| `queue` | The current conversation has two pending messages. | Lets the page show pending messages in the same snapshot. This read-only projection does not change the input phase. |

> **Key idea:** Treat `draftRev` as the validity period for character positions in the draft. It is not a user phase and does not retain draft content. It answers one question: is the draft on which a plugin calculated `[start, end)` still the current draft?

For example, the current draft is `See @ses` and `draftRev` is `12`. A reference plugin observes `@ses` at `[4, 8)`, starts an asynchronous lookup, and retains `{ start: 4, end: 8, draftRev: 12 }`. Before the lookup returns, the user inserts `Please ` at the beginning. The draft becomes `Please see @ses`, the token moves to `[11, 15)`, and the machine increments `draftRev` to `13`.

The old lookup still asks to replace `[4, 8)`. Trusting that range would cut the wrong text. The machine first compares the lookup's `draftRev: 12` with the current `draftRev: 13`:

```text
12 === 13 -> false -> reject
```

The plugin's next tracking pass observes the current draft and obtains a new range with `draftRev: 13`, allowing a valid replacement. This compare-version-and-write-only-if-equal rule is the CAS guard used here. Typing, deletion, undo, redo, and every successful menu, reference, or command rewrite increment `draftRev`, invalidating all character ranges calculated earlier. Comparing only the text inside the range is insufficient because the same text may occur in several places and the plugin decision may also depend on the selection and surrounding context that it observed.

> **Do not confuse it with the submission number:** `draftRev` prevents a plugin from applying stale character positions to a new draft. `SubmitAttempt.seq` prevents an old network result from settling a new submission. The former protects editing; the latter protects submission.

The current UI's menu refresh, focus management, and synchronous event calls already prevent most revision mismatches, so users rarely observe a rejected operation. This check fixes a defensive plugin-interface rule: no plugin can use character positions from an older draft as permission to modify the current draft.

The machine also retains undo history and a number for each submission. For example, the first submission has already failed and a second is now active when a late result from the first arrives. Its number does not match, so the machine ignores it. Closing the conversation also cancels the work associated with the active number. This submission record is `SubmitAttempt`, defined at [`contract.ts:236`](../packages/client/ui-conversation/src/client/input/contract.ts).

The machine does not retain the DOM caret. The textarea supplies the changed range or selection with each relevant edit; the machine uses it to move references and maintain undo history, while React continues to own the visible caret.

## 3. Events in and effects out

An "input event" here is not limited to a keyboard event. It is a fact or request handed to the machine. For example, `draft-changed` says that the draft changed, `enter` says that the user requested submission, and `submit-settled` says that a previous asynchronous submission returned. Every event enters through the `InputEvent` union at [`contract.ts:251`](../packages/client/ui-conversation/src/client/input/contract.ts) and reaches the `dispatch()` method at [`machine.ts:170`](../packages/client/ui-conversation/src/client/input/machine.ts).

An "output effect" is logic that the machine requires but cannot perform itself. "Effect" does not mean that the action is secondary. It means that the action occurs outside the in-memory state calculation and touches a command plugin, send API, or UI notice store. `dispatch()` only completes the state change synchronously and returns the requested work as zero or more `InputEffect` data values, defined at [`contract.ts:287`](../packages/client/ui-conversation/src/client/input/contract.ts); `SessionInputShell` then interprets and executes those values. This makes both the state result and the external-work request directly inspectable while keeping the machine independent of networking, plugins, and React.

This uses the central idea of the Command pattern: package what must be done as a value and hand it to one executor. The machine creates commands, an `InputEffect` is a command description with a discriminant and complete arguments, `SessionInputShell.execute()` is the invoker, and the injected command sources, submit handlers, send function, and notice store are the receivers. The implementation does not create a class with an `execute()` method for every command, so it is not the textbook command-object hierarchy. The discriminated union provides the important separation: the machine decides intent while the runtime selects the execution mechanism. Command values can be asserted directly, dropped after session disposal, and handled in one shell that owns asynchronous return events and resources.

The four effects have the following execution semantics:

| Effect | When it is produced | What `SessionInputShell` actually does | How the result returns |
|---|---|---|---|
| `adjudicate` | The user directly submits a `/`-prefixed draft from `plain`. | Calls the session input triggers so loaded command sources can decide who handles the draft; no mounted trigger pipeline is treated as no claim. | A decision becomes `adjudicated`; an exception becomes `adjudication-failed`. |
| `begin-submit` | A command has claimed the current draft and begins submitting its arguments. | For a command that accepts images, serializes the captured images before calling the claim's `submit` handler; those images are removed only after success. | Success, handler errors, and serialization errors all become `submit-settled`. |
| `default-sink` | The draft is submitted as an ordinary message, including an unclaimed `/...` draft. | Replaces inline references with model text supplied by each source, then calls the current session's default send function; a missing serializer or serialization failure blocks the send and retains the draft. | Send success, send failure, and serialization failure all become `submit-settled`. |
| `notice` | An adjudication or submission outcome needs to show information or an error. | Synchronously writes the shell's notice snapshot, which `InputBar` renders as transient feedback. | No asynchronous return event is needed. |

`submit-settled` is not an effect. It is an `InputEvent` that returns to the machine after an asynchronous submission finishes. It carries the original `SubmitAttempt`, a success flag, and an optional outcome or message. The machine first verifies that the attempt is still the active submission so a late result from an older request cannot overwrite newer state, then decides whether to clear or retain the draft and whether to return to `plain` or `claimed`.

A state has no automatic "on entry" action. One `dispatch()` call decides both the next state and any external work required at that moment. The shell executes the effects. When asynchronous work finishes, it wraps the outcome in an `adjudicated`, `adjudication-failed`, or `submit-settled` event and sends it back to the machine. The complete loop is:

```text
InputEvent
    ↓
InputMachine.dispatch()
    ├─ InputState → publish → React
    └─ InputEffect → SessionInputShell → external work
                                           ↓
                                       InputEvent
```

State changes and effects produce two different kinds of result:

| Result | Consumer | Concrete behavior |
|---|---|---|
| State snapshot change | React page | `plain` and `claimed` remain editable; `claimed` shows the command argument hint; `adjudicating` and `submitting` make the composer temporarily read-only. Draft, reference, and attachment changes also re-render from the snapshot. |
| `InputEffect` | `SessionInputShell` runtime adapter | Calls a command source, command handler, or conversation send API, or publishes a notice. Promise outcomes return to the machine as new events. |

The main transitions trigger the following logic:

| Starting point and input event | Immediate machine decision | Returned effect | After external work finishes |
|---|---|---|---|
| Edit in `plain`; receive `draft-changed` | Update the draft, reference ranges, undo record, and `draftRev`; remain in `plain`. | None | The page displays the new draft. |
| Pick `/goal` from the menu; receive `begin-command` | Write the command token into the draft and enter `claimed`. | None | The page shows the argument hint and remains editable; the command has not run. |
| Press Enter in `claimed`; receive `enter` | Create the submission attempt and enter `submitting`. | `begin-submit` | The shell calls the `/goal` submit handler. Its `submit-settled` result returns to `plain` on success and normally to `claimed` with the draft retained on failure. |
| Submit ordinary text in `plain`; receive `enter` | Create the submission attempt and enter `submitting`. | `default-sink` | The shell serializes references and calls the conversation send API. A successful `submit-settled` clears accepted content and returns to `plain`; failure retains the draft and returns to `plain`. |
| Submit a directly typed `/...` in `plain`; receive `enter` | Enter `adjudicating` without yet deciding whether the text is a command or an ordinary message. | `adjudicate` | Command sources return `adjudicated`: a claim enters `submitting` and emits `begin-submit`; no claim enters `submitting` and emits `default-sink`; an internally handled result returns directly to `plain`. `adjudication-failed` returns to `plain` and emits an error `notice`. |
| Receive `submit-settled` in `submitting` | Verify the attempt number, clear accepted content on success or retain the draft on failure, then return to `plain` or `claimed`. | A `notice` when the outcome carries text; otherwise none. | The page becomes editable; a `notice` renders as feedback. |

Therefore, "entering `submitting` sends the message" is only shorthand. More precisely, while processing `enter` or `adjudicated`, the machine enters `submitting` and also returns `begin-submit` or `default-sink`. The adapter performs the actual command call or message send. This separation keeps transition code independent of HTTP, attachment storage, command plugins, and React.

## 4. Implementation layers

The implementation uses one concrete pure machine behind narrower public faces. It does not define an `IInputMachine` interface with several machine implementations.

| Layer | Object | Responsibilities | Source |
|---|---|---|---|
| Types and public faces | `InputState`, `InputEvent`, `InputEffect`, `SessionInput`, `SessionInputResolver` | Define observable state, legal mutations, requested effects, and the per-session access API. | [`contract.ts:25`](../packages/client/ui-conversation/src/client/input/contract.ts), [`contract.ts:213`](../packages/client/ui-conversation/src/client/input/contract.ts), [`contract.ts:251`](../packages/client/ui-conversation/src/client/input/contract.ts) |
| Pure transition engine | `InputMachine` | Own draft revisions, phases, claims, references, paste attempts, undo/redo transactions, and active submission identity; accept events and produce effects. | [`machine.ts:116`](../packages/client/ui-conversation/src/client/input/machine.ts), [`machine.ts:170`](../packages/client/ui-conversation/src/client/input/machine.ts) |
| Runtime adapter | `SessionInputShell` | Own attachments, publish snapshots, execute effects, call command and Host APIs, convert promise outcomes back into events, and expose notices. | [`facade.ts:83`](../packages/client/ui-conversation/src/client/input/facade.ts), [`facade.ts:420`](../packages/client/ui-conversation/src/client/input/facade.ts) |
| Per-session assembly and registry | `InputHub` | Create each session adapter through dependency injection, connect scoped input-trigger events, route the default send, and delegate instance and listener disposal to the Cordis session scope. | [`hub.ts:39`](../packages/client/ui-conversation/src/client/input/hub.ts), [`hub.ts:71`](../packages/client/ui-conversation/src/client/input/hub.ts) |
| UI registration | Conversation plugin assembly | Publish the shell snapshot and actions through the standard session provide channel. | [`apply.ts:168`](../packages/client/ui-conversation/src/client/apply.ts) |
| UI feedback | `InputBar` | Render machine notices and Host prompt failures as a transient error toast. | [`InputBar.tsx:69`](../packages/client/ui-conversation/src/client/skeleton/InputBar.tsx) |

The useful abstraction is therefore the separation between state decisions and runtime operations. `InputMachine` remains package-private because other packages need the published state and actions, not permission to replace the transition engine. `SessionInput` and `SessionInputResolver` are the access abstractions used by the conversation wiring layer.

The filenames reflect this division. `contract.ts` contains only the types and public interfaces shared by the machine, runtime adapter, and page, including the observable `InputState`, dispatchable `InputEvent`, and `InputEffect` that the shell must execute. These definitions do not belong to one implementation, so they do not live in `machine.ts`. Here, contract means the type-level obligations shared by callers and implementations, not a wire protocol or a compatibility promise.

`facade.ts` applies the Facade pattern. Pages and plugins use the narrow, per-session `SessionInput` interface instead of coordinating `InputMachine`, attachment objects, command triggers, the conversation send function, state stores, and notice stores separately. As the unified entry point, `SessionInputShell` translates calls such as `setDraft()`, `submit()`, and `addImages()` into machine events, invokes commands returned by the machine, publishes the composed snapshot, and manages asynchronous outcomes and session disposal. The facade does not take over the responsibilities of those internal components; it fixes their call ordering, ownership, and error return path so callers do not depend on their collaboration details.

The `SessionInputShell` name also expresses a functional-core, imperative-shell split. `SessionInput` names the input operations it provides for one session; `Shell` says that it wraps the in-memory-only `InputMachine` and is the runtime layer allowed to interact with command plugins, the conversation send API, attachment resources, and the React notice store. It serves as both the facade and the command invoker. `InputHub` retains one shell for each `SessionId` and disposes it with the session.

### 4.1 `InputHub`: dependency injection and inversion of control

`InputHub` combines a per-session registry with dependency assembly and routing. Its `Map<SessionId, SessionInputShell>` ensures that one live session resolves to the same shell, but the hub is not a general dependency injection container. Cordis is the framework that owns plugin services, events, and effect lifecycles.

Dependency injection occurs when `InputHub.shellFor()` creates a `SessionInputShell`. The shell does not locate or construct command triggers, the queue projection, the conversation send function, or image serializers. It receives narrow capabilities such as `actx`, `inputTriggers`, `queue`, `defaultSink`, and `commandImages`. `InputHub` adapts Cordis services and the current session binding into those capabilities, so the shell depends on callable interfaces instead of concrete conversation, attachment, or input-trigger plugin implementations. This is also dependency inversion: the high-level input flow declares what it needs, while the assembly layer decides which provider supplies it.

Inversion of control describes who decides when code runs. `InputHub` does not create sessions, poll for events, or decide independently when to destroy an instance:

```text
Cordis/session scope
  -> materialize -> InputHub.shellFor(...)
  -> dispatch event -> scoped listener -> SessionInputShell
  -> dispose -> effect disposer -> shell.dispose()
```

Session provide materialization triggers creation. `InputHub` registers input-mutation listeners on the session `actx` through `actx.on()`, after which Cordis dispatches events to those listeners. One `actx.effect()` disposer owns the listeners, shell, registry entry, and remaining attachment cleanup, and session-scope disposal invokes it. The precise statement is therefore: **Cordis/session scope controls creation, event invocation, and disposal timing; `InputHub` responds at those times to assemble, route, and clean up the input subsystem.**

> **Further study: inversion of control.** This page covers only its application in the input subsystem. Continue with "Cordis In Five Ideas" in the [Cordis Primer](cordis-primer.md), especially contexts as service repositories, dependency declaration through `inject`, typed events, and reversible registration effects, then compare the disposer and teardown-order guidance under "Practical Rules."

## 5. Ordinary message submission

The following example traces an ordinary message containing both a reference and an image. The user writes "Compare the approach in `@session:abc`" and attaches `diagram.png`. Immediately before Enter, the relevant state is:

| Field | Value before Enter | Meaning |
|---|---|---|
| `phase` | `plain` | The composer remains editable and can be submitted. |
| `draft` | `Compare the approach in @session:abc` | Display text in the composer; the reference is still in its user-facing form. |
| `occurrences` | Source, identity, and character range for `@session:abc` | The shell later replaces the recorded range without rescanning the draft. |
| `imageIds` | `[id for diagram.png]` | The machine observes identifiers; the shell owns files, bytes, and preview URLs. |
| `inflight` | None | No input is being adjudicated or submitted. |

Each earlier `SessionInputShell.setDraft()` call dispatched `draft-changed`. The machine updated `draft`, reference ranges, `draftRev`, and the undo record in one transaction. This editing stage remains in `plain` and emits no effect.

### 5.1 Enter: `plain` to `submitting`

`submitting` is the client input machine's phase for waiting until the current submission settles. It does not mean that the message has been sent successfully or that the server or model is processing it. It means only that the machine accepted this Enter press, recorded the sole `inflight` attempt, and emitted the effect that performs the submission, but has not yet received the effect's `submit-settled` result.

The two terms identify different things:

| Term | What it is | What it contains |
|---|---|---|
| `attempt` | A record of one submission attempt triggered by Enter, typed as `SubmitAttempt`. This concrete data object leaves with an `adjudicate`, `begin-submit`, or `default-sink` effect and returns unchanged with the result event. | `seq` is a monotonic sequence; `draftSnapshot` is the draft at Enter time; `mode` is `queue` or `steer`; `signal` cancels asynchronous work. |
| `inflight` | The machine's sole internal slot for the attempt that has not finished. Its code value is `{ attempt, controller }`; `undefined` means that no input is being adjudicated or submitted. | The attempt and the `AbortController` that can trigger its `AbortSignal`. |

An `inflight attempt` is therefore not two attempts. It is the attempt currently occupying the `inflight` slot. For example, Enter creates attempt `#7` and `inflight` points to `#7`; once `submit-settled(#7)` passes the sequence check, the machine clears that slot. A later `submit-settled(#6)` does not match the current slot and is ignored. For a directly typed Slash command, the same attempt first waits for ownership adjudication in `adjudicating` and then enters `submitting`; the actual send does not create another attempt.

The following scenarios show why the machine retains both an attempt and the `inflight` slot:

| Scenario | How the attempt and `inflight` work | Problem solved |
|---|---|---|
| Ordinary message send | Enter creates attempt `#7`, and `inflight` stores `#7`; `default-sink` and the final `submit-settled` both carry that same object. | Correlates an asynchronous result with the Enter press that initiated it. |
| Repeated Enter | While `inflight` exists and the phase is `adjudicating` or `submitting`, another Enter creates no attempt and emits no send effect. | Prevents duplicate submission of the same draft. |
| Late old result | While the current `inflight` is `#7`, `submit-settled(#6)` is ignored because its sequence does not match. | Prevents an old request from clearing or overwriting a newer draft. |
| Slash command adjudication | For directly typed `/goal ...`, one attempt passes through `adjudicating` and `submitting`; adjudication does not create another send attempt. | Keeps command recognition and submission within one user operation. |
| Session closes | `inflight` also owns the `AbortController`; scope disposal aborts it, and reference serialization and the browser request observe the same signal. | Prevents a closed session from continuing to send or applying a late result to the page. |
| References or attachments are being prepared | The attempt retains the draft and mode captured at Enter, image changes are rejected during submission, and the shell sends the captured data. | Prevents visible content and request content from diverging during asynchronous preparation. |

| View | Facts during `submitting` |
|---|---|
| Already fixed | The submission sequence, draft snapshot at Enter time, delivery mode, and cancellation signal are stored in `inflight`; the machine has also selected `begin-submit` or `default-sink`. |
| Work in progress | Outside the machine, the shell runs a command handler or prepares references and attachments before calling the session send API; the machine waits for the result to return as `submit-settled`. |
| UI restrictions | The composer is temporarily read-only, repeated Enter presses are ignored, and images cannot be added or removed. The original draft, references, and attachments remain available for failure recovery. |
| Exit condition | `submit-settled` for the current attempt ends `submitting` on either success or failure. Success commits the captured content; failure retains retryable content. Session disposal instead cancels the attempt and releases the entire input instance. |

Representing this phase separately prevents duplicate submission, keeps attachments stable during serialization and sending, and rejects late results by attempt sequence. Queueing, Turn, Step, and Agent replies after server acceptance belong to the conversation flow and do not keep the composer in `submitting`.

Enter makes the shell dispatch `{ type: 'enter', mode }`. `onEnter()` at [`machine.ts:483`](../packages/client/ui-conversation/src/client/input/machine.ts) processes it in this order:

1. If the phase is already `adjudicating` or `submitting`, it returns an empty effect array, so repeated Enter presses cannot create another submission.
2. For an ordinary message, it trims the draft to check for content. An empty draft creates no attempt. `SessionInputShell.submit()` has already diverted an image-only submission to a separate send path before entering the machine. A `/`-prefixed draft follows Slash adjudication; neither branch is part of this example.
3. `beginAttempt()` increments the sequence and creates `{ seq, signal, draftSnapshot, mode }`, then occupies the single `inflight` slot. `draftSnapshot` retains the display draft at Enter time, `mode` retains this ordinary message's `queue` or `steer` delivery intent, and the `AbortSignal` cancels later work during session disposal.
4. The machine changes `phase` from `plain` to `submitting` without clearing the draft, references, or attachments.
5. It returns `{ type: 'default-sink', attempt, draft, mode }`. This is a command value for the shell, not another state.

`SessionInputShell.run()` starts this effect through `execute()` and then publishes the composed state snapshot. React therefore observes `phase: submitting` with the original draft and attachments and makes the composer read-only. At this point only the first half of the loop has completed:

```text
plain
  └─ enter
       ├─ state: submitting + inflight attempt
       └─ effect: default-sink
```

### 5.2 `default-sink`: prepare and send

`SessionInputShell.execute()` dispatches `default-sink` to `sinkSerialized()` at [`facade.ts:456`](../packages/client/ui-conversation/src/client/input/facade.ts). The effect runs in six steps:

1. The shell copies the current `imageIds` as this send's attachment set and reads `occurrences` from the machine. Image additions and removals are rejected during submission, so the visible attachments stay aligned with that set.
2. With no references, the shell trims the draft and immediately calls the injected `defaultSink(text, imageIds, mode, attempt.signal)`. This fast path has no additional asynchronous serialization.
3. With references, the shell asks the input triggers to run `serializeReference(source, ref, signal)` for every occurrence in parallel. `source` identifies the plugin that registered that reference kind, and its codec decides how to turn the page label into text that the later conversation pipeline can recognize. For this session reference, the page displays `@session:abc`, while the occurrence's `ref` retains a canonical mention such as `@[session:abc](dsh-session:<encoded-id>)`; the unified `@` reference source's serializer returns that `ref`.
4. After all references resolve, the shell rebuilds the draft from left to right using the recorded character ranges, substitutes the serialized values, and trims the final text. The accepted user message therefore carries the canonical mention rather than only the page label. Before assembling the model request, the `session-reference` plugin recognizes that mention, replaces the token in the direct text with readable `@session:abc`, and adds a bounded snapshot of the referenced session as extra context. A missing serializer or any serialization failure stops the send instead of falling back to the page label.
5. The default send function in `InputHub` calls `conversation.sendSession(session, text, imageIds, mode, signal)` for the current session at [`hub.ts:165`](../packages/client/ui-conversation/src/client/input/hub.ts). Empty text with no images resolves as success; otherwise the conversation service accepts the message.
6. `settleSubmit()` observes the promise. A returned `SubmitOutcome` maps `outcome.kind === 'success'` to the success flag; a rejected promise supplies its error text. Both branches dispatch `submit-settled` with the original attempt.

The actual browser network send is not in `InputMachine`, the `default-sink` value, or `InputHub`. The call continues through `ConversationService.sendSession()` and Client `Session.prompt()`; `AbstractApiClient.callUnary('session.prompt')` then builds the RPC envelope, and `doFetch()` inside `postJson()` sends `POST /api/session.prompt`. Only after the Host returns its admission result does this promise settle and produce `submit-settled`. This point does not wait for the Agent task or the model reply; [Web Request Flow and Execution Model](web-request-execution-model.md) traces the later network and execution stages.

| Stage | Text or data | Consumer |
|---|---|---|
| Composer display | `Compare the approach in @session:abc` | The user and React page. |
| User message accepted by the conversation | `Compare the approach in @[session:abc](dsh-session:<encoded-id>)` | The `session-reference` plugin uses the stable session identity to locate the referenced session. |
| Model request | Direct text containing `Compare the approach in @session:abc`, plus a snapshot of the referenced session | The model receives both the readable question and the referenced content. |

"Model-visible text" therefore does not mean merely changing the display format of `@session:abc`. Serialization records reference identity, which otherwise exists only in the page occurrence, in the durable user message so a later plugin can resolve that identity and prepare actual context for the model.

When reference serialization finishes, a disposed shell does not call the send function. When the send promise finishes, `dead()` checks whether the shell has been disposed or the attempt aborted and drops a late result without dispatching another event. A successful result first removes the images captured for this send from the shell's runtime attachment list. Failure leaves those images intact.

### 5.3 `submit-settled`: commit or retain the draft

When `submit-settled` reaches `onSubmitSettled()` at [`machine.ts:541`](../packages/client/ui-conversation/src/client/input/machine.ts), the machine first requires the phase to remain `submitting`, the `inflight` slot to exist, and the event's `attempt.seq` to equal the active sequence. A failed check returns an empty effect array, so an old request cannot settle a newer submission. A valid result releases `inflight` before applying the outcome:

| Outcome | State transition | Draft, references, and attachments | Further effect |
|---|---|---|---|
| Success | `submitting` → `plain` | Clear the command claim and references; clear the draft when it equals `draftSnapshot`, or retain only a pure suffix; clear undo and redo. The shell removes only image identifiers captured by this send; other identifiers in its runtime list are unaffected. | Emit `notice` when `SubmitOutcome` contains text; otherwise none. |
| Send, serialization, or network failure | `submitting` → `plain` | Retain the current ordinary-message draft, references, and images; do not overwrite newer programmatic edits with the old snapshot. | Emit an error `notice` when error text exists. |
| Late or cancelled result | No transition | Change no input data. | None. |

The success branch retains only a pure suffix because "submitted snapshot followed by appended text" can be separated without ambiguity. Other interleaved rewrites cannot reliably identify which characters were accepted. The current `InputBar` already makes the composer read-only during `submitting`; this rule primarily protects against asynchronous or programmatic input. The shell writes a `notice` synchronously to its notice snapshot, and `InputBar` renders it as transient feedback at [`InputBar.tsx:89`](../packages/client/ui-conversation/src/client/skeleton/InputBar.tsx).

The complete state-and-effect loop is therefore:

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

## 6. Other input paths

### 6.1 Slash commands

When trimmed text begins with `/`, `onEnter()` first enters `adjudicating` and emits `adjudicate`. A matching command moves the same attempt into `submitting`; a miss sends the captured text as an ordinary message; a handled trigger returns to `plain`; an adjudication error retains the draft and emits an error notice. These branches are implemented at [`machine.ts:494`](../packages/client/ui-conversation/src/client/input/machine.ts), [`machine.ts:504`](../packages/client/ui-conversation/src/client/input/machine.ts), and [`machine.ts:533`](../packages/client/ui-conversation/src/client/input/machine.ts).

### 6.2 References and paste

Reference occurrences store source-owned identity and draft ranges. Every edit reconciles those ranges. Before ordinary submission, the shell replaces display ranges with each source's serialized model text; a missing serializer or serialization failure blocks sending and retains the draft. Paste matching may create references in the initial paste transaction or upgrade a matched token later as a separate undoable transaction.

The event definitions and their transaction rules are at [`contract.ts:253`](../packages/client/ui-conversation/src/client/input/contract.ts); serialization is at [`facade.ts:449`](../packages/client/ui-conversation/src/client/input/facade.ts).

### 6.3 Images

The pure machine exposes attachment identifiers in `InputState`, but `SessionInputShell` owns their runtime list because file objects, bytes, and preview URLs require browser-side cleanup. Adding or removing images is refused during `adjudicating` and `submitting`, preventing the visible list from diverging from an in-flight command serialization. The attachment operations start at [`facade.ts:126`](../packages/client/ui-conversation/src/client/input/facade.ts).

### 6.4 Session lifecycle

`InputHub` stores one `SessionInputShell` per `SessionId`. Session materialization creates the shell and publishes its state/actions; scope disposal aborts the active attempt, unregisters input-trigger listeners, releases remaining image objects, and removes the registry entry. Creation and teardown are together at [`hub.ts:71`](../packages/client/ui-conversation/src/client/input/hub.ts).

## 7. Code-reading map

| Question | Start here |
|---|---|
| What can the page observe or invoke? | [`contract.ts:25`](../packages/client/ui-conversation/src/client/input/contract.ts) |
| What state does the machine retain? | [`machine.ts:116`](../packages/client/ui-conversation/src/client/input/machine.ts) |
| Where does every event enter? | [`machine.ts:170`](../packages/client/ui-conversation/src/client/input/machine.ts) |
| How is Enter classified? | [`machine.ts:483`](../packages/client/ui-conversation/src/client/input/machine.ts) |
| How are success, failure, and newer edits reconciled? | [`machine.ts:541`](../packages/client/ui-conversation/src/client/input/machine.ts) |
| Where are network and plugin operations executed? | [`facade.ts:425`](../packages/client/ui-conversation/src/client/input/facade.ts) |
| How does an asynchronous result return to the machine? | [`facade.ts:495`](../packages/client/ui-conversation/src/client/input/facade.ts) |
| Where is one machine attached to each session? | [`hub.ts:71`](../packages/client/ui-conversation/src/client/input/hub.ts) |
| Where does React receive the snapshot and actions? | [`apply.ts:181`](../packages/client/ui-conversation/src/client/apply.ts) |
