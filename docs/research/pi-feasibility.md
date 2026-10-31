# Pi session feasibility research

Research for [Check Pi session, messaging, permission, and transcript capabilities](https://github.com/markdlabrecque/pi-subagents-reloaded/issues/2), under [Specify parent-controlled Pi subagent sessions](https://github.com/markdlabrecque/pi-subagents-reloaded/issues/1).

## Scope and source version

This is a factual feasibility report, not a product specification or an implementation. Proposed mechanisms below are possibilities, not decisions. No model requests, paid-model experiments, MCP connections, or extension execution were used.

The installed package is `@earendil-works/pi-coding-agent` **1.4.0**, at `/home/mark/.local/lib/node_modules/@earendil-works/pi-coding-agent`. Its manifest pins `pi-agent-core`, `pi-ai`, `pi-codemode`, and `pi-mcp` to 1.4.0. Upstream tag `v1.4.0` resolves to commit **`4a080252f06caf63a525d7873c5d94d3c23887e0`**. All upstream citations below pin that commit, not moving `main`. [Manifest][manifest]

The installed manifest and 14 central documentation files were byte-compared with that upstream commit and matched. Nine downloaded source files also matched the `sourcesContent` embedded in installed JavaScript source maps: `sdk.ts`, `agent-session.ts`, `session-manager.ts`, extension `types.ts` and `runner.ts`, RPC `rpc-mode.ts`, `rpc-types.ts`, `rpc-client.ts`, and codemode `index.ts`. Installed declarations were inspected separately. This ties the cited release source to the installed package rather than assuming the tag and installation agree.

Read in full: README; SDK; CLI integration and CLI; RPC, commands, extension UI, JSON and message types; extensions; security and containerization; configuration and settings; sessions, session format and compaction; codemode and MCP; models and virtual models; skills, prompt templates and environment variables. Relevant cross-referenced declarations and source methods were checked. The SDK examples `01` through `14`, their README, RPC client example, permission gate and dynamic-tools examples were inspected. Upstream describes these as repository-typechecked examples. They were not executed or re-typechecked here. [SDK guide][sdk] [Examples][examples]

## Findings at a glance

- Separate fresh conversations, parent-supplied system instructions, selected resources, model control, streaming, queues and persistent transcripts are available without changing Pi. Fresh history does not mean resource isolation: default discovery imports local instructions, extensions and settings. [SDK][sdk] [Resource loader][resources]
- In-process SDK sessions offer direct control. RPC offers a separate process and typed JSONL transport, but fewer built-in controls. Neither establishes an OS sandbox or native parent-owned agent group. [RPC][rpc] [Security][security]
- An active tool list is not a complete authorization list. Codemode/deferred tools can stay callable when inactive; MCP has special allowlist rules. Pipeline `tool_call` gates cover direct and nested tool calls, but not arbitrary extension code or every model invocation. [Extensions][extensions] [MCP][mcp] [Codemode executor][code-execute]
- `agent_settled`, not `agent_end` or prompt acceptance, is the automatic-work boundary. Steering waits for the current turn's tools. A tool waiting for an answer delivered through that same session's queue can deadlock. [RPC commands][commands] [Session source][session]
- JSONL restores recorded conversation state, not running processes, promises, queues or exactly-once side effects. Stable transcript entry IDs support small cross-references; nested calls have bounded metadata, not independent transcript entries. [Session format][format] [Session manager][manager] [Message types][messages]

## Verified facts

### Independent context, instructions and resources

`createAgentSession()` creates an `AgentSession` owning one conversation, model, tools, queues and extension runtime. A new `SessionManager.create(cwd)` or `SessionManager.inMemory()` starts a new conversation. Opening, continuing, forking or cloning history is explicit. Supplying `parentSession` records lineage in the header; it is not automatic history copying, parent ownership, cancellation propagation or authorization. [SDK][sdk] [Session declarations/source][manager] [Session example][example-sessions]

The factory defaults to discovered resources, file-backed settings, persistent history and configured model credentials. Consequently, a child created with defaults has fresh conversation history but may load the same `AGENTS.md`, skills, extensions, prompt templates and model configuration as its parent. `cwd` determines discovery and built-in tool paths, not a filesystem access boundary. Context files load even when project trust is declined. [SDK][sdk] [Configuration][configuration] [Security][security]

The SDK can replace the system prompt with `DefaultResourceLoader.systemPromptOverride` and suppress inherited append text with `appendSystemPromptOverride: () => []`. The checked prompt example explicitly calls out `APPEND_SYSTEM.md`. Context files need their own override or suppression. The loader exposes `noExtensions`, `noSkills`, `noPromptTemplates`, `noThemes`, `noContextFiles`, resource-specific overrides and inline extension factories. A fully supplied `ResourceLoader` avoids standard discovery entirely, as example 12 shows. A parent can therefore supply selected context without copying its own history. [Prompt example][example-prompt] [Full-control example][example-control] [Resource loader][resources]

CLI/RPC startup supports `--system-prompt`, repeated `--append-system-prompt`, `--no-context-files` and resource suppression options. `--no-extensions` still permits explicit `-e` resources. `--no-mcp` only disables the built-in MCP implementation, not a replacement extension. `PI_CODING_AGENT_DIR` selects another agent directory, while session storage has its own setting/override. There is no standard RPC command to replace the system prompt or inject arbitrary resource loaders after startup. [CLI][cli] [RPC types][rpc-types] [Environment][environment]

`session.systemPrompt` is read-only effective state. Structured prompt/tool changes are recorded before the next model request. A `before_agent_start` handler returning a forced whole `systemPrompt` is a special case: providers receive it, but the transcript still records structured sections rather than that exact forced text. The transcript is therefore not guaranteed to be a byte-for-byte record of every final provider payload. [SDK][sdk] [Extensions][extensions] [Session source, forced prompt projection][session]

### In-process SDK versus subprocess RPC

| Property | In-process SDK | Subprocess RPC |
|---|---|---|
| Conversation | One `AgentSession` per conversation; multiple instances can be hosted | One active session per child runtime; new/switch/fork replace it |
| Control | Direct resource, tool, model, settings and session objects | Standard JSONL commands plus whatever trusted extensions expose |
| Observation | `AgentSessionEvent`, with cumulative SDK message snapshots | Responses and events; message updates are delta-only |
| Isolation | Same process, memory, environment and extension privileges as host | Separate process and lifecycle; same OS account permissions unless separately constrained |
| Tool policy | Initial filters, dynamic active tools, inline/file extension gate | Startup filters; no standard `set_active_tools` or grant/revoke command |
| Replacement | `AgentSessionRuntime` recreates services and session | `new_session`, `switch_session`, `fork`, `clone` replace active runtime |
| Lifecycle owner | Host must dispose owned sessions | Host must handle EOF, exits, signals, timeouts and stderr |

These comparisons follow the SDK/runtime declarations, RPC command union and security contract. A subprocess is a failure/lifetime boundary, not permission isolation. SDK sessions can also share process-global limits, including the default two codemode workers. [SDK][sdk] [Runtime][runtime] [RPC][rpc] [RPC types][rpc-types] [Codemode][codemode] [Security][security]

The exported `RpcClient` starts Pi, correlates command IDs, exposes typed methods and dispatches session events. Use continuous LF-only JSONL reading and stdin/stdout backpressure. Unicode line/paragraph separators are valid inside JSON strings; generic Node `readline` is unsuitable. Responses may arrive out of order, so request IDs matter. Session events generally carry no request ID. RPC has no JSON-mode session header; `get_state` gives session identity. [RPC][rpc] [JSON][json] [RPC client][rpc-client]

RPC is not a transparent SDK tunnel. No standard command installs a policy gate, sends a typed custom message, appends arbitrary custom state, or directly administers tool grants. An explicitly loaded trusted extension could provide additional behavior through registered commands or UI records. Extension commands execute immediately through `prompt`, even while streaming, and own their own concurrency. [Commands][commands] [RPC types][rpc-types] [Extensions][extensions]

### Tool authorization, indirect access, grants and revocation

#### Registered, active and callable are different sets

The initial `tools` filter accepts exact names and `*` patterns. `excludeTools` applies after selection. `noTools: "all"` and an explicit empty `tools` list disable MCP too. `noTools: "builtin"` disables default built-ins but retains extension/custom tools. Lists consisting only of `+name` and `-name` modify defaults rather than creating an allowlist. [SDK options][sdk-source]

Tool exposure has five values. [Extensions, tool exposure][extensions] [Extension declarations][extension-types]

| Exposure | Direct declaration | Indirect `ctx.executeTool` reach |
|---|---|---|
| `direct` | While active | While active |
| `model-only` | While active | Never |
| `codemode` | If explicitly activated | Whenever registered |
| `deferred` | After activation/search | Whenever registered |
| `hidden` | Never | Never |

`pi.setActiveTools()` changes model declarations, not all indirect reach. Unknown and hidden names are ignored. Tools cannot be unregistered through the documented tool API; re-registering with `hidden` withdraws reach. Tool declarations change before subsequent requests, not retroactively in an already-running model response. `prepareLoadout.hiddenDeclarations`, used by codemode-only presentation, hides declarations while tools remain active/callable. [Extensions][extensions] [Session source, loadout methods][session]

MCP adds a critical exception: a nonempty `--tools read,codemode` list leaves MCP tools registered for codemode and tool search. Including an entry starting `mcp__` makes the allowlist filter MCP too. `--exclude-tools` also filters MCP, including resource tools. `codemode` and `deferred` MCP exposures are reachable by either indirect mechanism. Servers connect asynchronously, tools can arrive late, and search-loaded tools remain declared on that transcript branch. MCP may auto-activate codemode/tool search. A static snapshot of startup declarations cannot describe future reach. [MCP][mcp] [CLI][cli] [SDK options][sdk-source]

#### What the pipeline actually gates

`tool_call` can block execution and handler failures fail closed. Direct model calls and nested calls made through `ctx.executeTool()` pass validation and the same `tool_call`/`tool_result` hooks. MCP tool/resource calls use that pipeline. Nested lifecycle events have `parentToolCallId`; IDs are assigned as `<parent id>/<n>`. `reachesTools` helps late registration finish before execution; it is a declaration of reach, not a security policy. MCP annotations are unverified hints, not proof that a tool is safe. [Extensions][extensions] [MCP][mcp] [Session source, `_executeNestedToolCall`][session] [Extension types][extension-types]

A trusted extension is executable Node/Bun code with the process's privileges. It can use filesystem/network APIs, spawn processes, call a tool implementation directly, use `pi.exec`, or invoke models outside `ctx.executeTool()`. The tool gate does not sandbox it. A child allowed `bash` can use other executables or launch another Pi process with whatever files, environment and credentials are accessible. Skill instructions are not an authorization boundary either. [Security][security] [Extension types][extension-types] [Skills][skills]

There is another path worth separating explicitly: codemode `models.classify()` and `models.generateImages()` call the model registry, not `ctx.executeTool()`. They do not produce a separate agent `tool_call` authorization opportunity for each model operation. The enclosing codemode call is gated, and model usage is aggregated. `createCodemodeExtension({ models: false })` can omit that namespace; the extension's default is `models: true`. Thus an allowed-tool-name policy alone does not establish a model-spending policy. [Codemode][codemode] [Codemode extension][code-index] [Codemode executor, `createModelGlobals`][code-execute]

User/RPC shell execution is also distinct from an LLM `bash` tool call. RPC `bash` uses `user_bash`; `session.executeBash()` executes and records shell output separately. Denying the model's `bash` tool is not a ban on a trusted RPC client or extension running shell commands. `abort_bash` is separate from run abort. [RPC mode source][rpc-mode] [Session source, `executeBash`][session] [Commands][commands]

#### Limits of revocation guarantees

Pi supplies mechanisms to change reach and deny a call before execution. It does not supply a parent-owned grant ledger, request-with-reason protocol, or child-proof self-grant prevention against arbitrary trusted extension code. Those concepts would be extension behavior, not native guarantees. Initial factory allow/exclude filters continue to constrain registration and activation, so `setActiveTools` cannot simply grant a name outside those filters. Widening access would require a compatible initial registry/filter arrangement or session recreation, in addition to controller authorization. [Extension API][extension-types] [Session source, `_isAllowedTool` and `_isActivatable`][session] [RPC types][rpc-types]

Changing active tools does not revoke registered codemode/deferred tools. Hiding or filtering a tool and enforcing a current policy in `tool_call` can prevent later admissions through the pipeline. Neither reverses side effects from an admitted/in-flight call. Tools from one response may run in parallel, and handlers run in registration order. Tool input can be mutated by later handlers without revalidation, so argument-sensitive approval also depends on trusted handler ordering. No atomic transaction joining policy mutation, admission and external side effects is documented. [Extensions][extensions] [Extension types, `ToolCallEvent`][extension-types]

Cancellation uses abort signals; it is not rollback. Codemode explicitly says calls before script failure are real and not undone. Remote operations may already have happened even if a response is lost. Revocation timing and treatment of admitted calls therefore still need a human-defined contract. [Codemode][codemode] [MCP, retry behavior][mcp]

### Parent and peer messages, queues, steering and settlement

Pi exposes message primitives for a session, not a native addressed parent/child/peer mailbox. `pi.events` is an extension event bus, backed by an in-process `EventEmitter`; it is not a durable, authenticated transport or an automatic bridge across sessions/processes. No parent group, recipient ACL, peer identity, delivery receipt or deduplication contract appears in the inspected SDK/RPC APIs. [Extension API][extension-types] [Event bus source][event-bus] [RPC types][rpc-types]

The SDK offers `prompt`, `steer`, `followUp`, `sendUserMessage`, and `sendCustomMessage`; extensions expose `sendUserMessage` and `sendMessage`. User messages initiate work; custom messages can carry context-visible content and non-model `details`. Custom content becomes user-role input for providers, not a new privileged system role. Extension send methods return `void`; direct session send methods return promises. Neither return type is a recipient reasoning/read receipt. [Session source/declarations][session] [Extension types][extension-types] [Message types][messages]

| Operation | Verified delivery/completion meaning |
|---|---|
| SDK `prompt()` when idle | Accepted run resolves after it finishes, including automatic retries; input/commands may instead be handled |
| RPC `prompt` success | Accepted, queued or handled; `disposition` distinguishes these, not model success |
| Steer while running | After the current assistant turn and its tool calls, before the next model request |
| Follow-up | After pending tools and steering finish; can be delayed by continuing work |
| Custom `deliverAs: "nextTurn"` | Held for the next user prompt; does not itself wake an idle session |
| Custom `triggerTurn: false` during a run | Appended after current turn results, without breaking tool-call/result ordering |
| `agent_end` | One low-level run ended; retries, recovery, compaction or queued work can follow |
| `agent_before_settle` | Actionable final boundary; can request continuation |
| `agent_settled` | No automatic work remains for that session-level run; not proof the task succeeded |

These rules come from the SDK guide, command reference and `sendCustomMessage` source. Queue modes are `all` or `one-at-a-time`. Busy prompts require an explicit steering/follow-up choice. Input hooks can consume or transform queued submissions, and extension commands run immediately instead of being queued normally. Completion must inspect final assistant stop reasons/errors as well as settlement. [SDK][sdk] [Commands][commands] [Session source][session] [JSON][json]

A subscription must exist before submission to avoid missing fast completion. `RpcClient.promptAndWait()` does that internally and avoids waiting for settlement when the prompt is handled. Standalone RPC `waitForIdle()` waits for a future settlement event; calling it after settlement can time out. The SDK's `waitForIdle()` checks current idle state first. [RPC][rpc] [RPC client source][rpc-client] [Session source][session]

`abort()` requests stopping and waits for idle. RPC documentation explicitly says queued messages can continue unless cleared. `clear_queue` before `abort` is the documented Esc-like sequence. Stopping a run, clearing queued work, aborting direct shell work, and terminating a process are different operations. [Commands][commands] [Session source, abort methods][session]

### Deadlock and liveness constraints

The following are deductions from the verified ordering, not observed paid-model experiments:

1. A tool in session A waits for a reply injected as steer/follow-up into A. Delivery waits for A's current tools to finish, including the waiting tool. The same problem applies to a tool awaiting a permission decision that only A's next model turn can make. [Queue ordering][commands]
2. A parent tool waits for child settlement while a child tool waits for the parent to process a queued question. Neither finishes its current tool batch. A peer wait cycle has the same shape. Pi provides no cross-session deadlock detector in the inspected interfaces. [Queue ordering][commands] [SDK][sdk]
3. Awaiting the current session's idle/replacement from an awaited lifecycle handler can wait for that handler itself. Pi explicitly reserves certain operations for command contexts because lifecycle use can deadlock. Even `agent_settled` extension handlers are awaited before the public settlement event and idle waiters are resolved. [Extensions][extensions] [Session source, `_emitAgentSettled`][session]
4. An RPC parent that waits for a response without draining stdout can fill the pipe and stall the child. A permission UI request without a reply can also block indefinitely unless it has a timeout. [RPC][rpc] [RPC extension UI][rpc-ui]
5. In-process codemode scripts share a default two-worker pool. A cancelled worker occupies its slot until it exits. Waiting for workers or long remote operations is not deadlock proof, and a later queued follow-up has no bounded delivery latency. [Codemode limits][codemode]

### Persistence, reload and crash recovery

Persistent sessions are version-3 JSONL trees. The active branch and compaction projection reconstruct model context. `SessionManager` is authoritative for finalized context; assigning `agent.state.messages` is not a durable history replacement. Custom entries persist non-context data; custom messages contribute model context. Branch-sensitive extension state is reconstructed from `getBranch()`, not every abandoned path. [SDK][sdk] [Format][format] [Extensions][extensions]

Current source creates the file only after a user or assistant message exists. Model/prompt/custom setup entries alone can remain solely in memory, even when a session has a planned file path. Once conversation exists, `_persist` synchronously writes initial entries and appends subsequent entries. `message_end` persists finalized messages; streaming `pending` assistant messages are not persisted as such. A crash can therefore preserve the first user prompt but lose an unfinished assistant stream. [Manager, `_hasConversation`/`_persist`][manager] [Session source][session] [Message types][messages]

The loader skips malformed JSON lines, validates the first parsed session header, and appends a newline if the file ends with an unterminated line. This is tolerant loading, not full recovery or a side-effect journal. The inspected session writer has no `fsync` durability barrier and no multi-writer transaction. Do not infer power-loss safety, safe concurrent writers, exactly-once tool execution or guaranteed recovery of partially written entries. [Manager, `loadEntriesFromFile` and persistence][manager]

The leaf is reconstructed as the last parsed entry. `branch()` and `resetLeaf()` only change the in-memory pointer until an entry is appended. A navigation-only selection can therefore be lost on reopen. This matters if a future controller treats a selected branch as durable control state. [Manager, `_buildIndex`, `branch`, `resetLeaf`][manager]

Pending input/custom queues, promises, event listeners, processes and active abort controllers are runtime state, not session entry types. The inspected interfaces do not persist an accepted-but-undelivered message queue or revive running tools after process restart. A tool may have performed an external side effect before its result was persisted. Reopening that transcript cannot establish whether repeating the call is safe. [Session source][session] [Format][format] [MCP retry cautions][mcp]

Reload emits `session_shutdown`, invalidates the old extension runtime, reloads resources and binds a replacement runtime. Session replacement recreates cwd-bound services and may fail after teardown; it is not a rollback transaction. Old extension contexts are invalid. Subscriptions to replaced `AgentSession` instances must be rebound. Extensions should use idempotent shutdown and reconstruct state at startup. An orderly hook cannot be assumed to run on SIGKILL, process crash or host loss. [Extensions][extensions] [Runtime][runtime] [Runtime example][example-runtime] [Session reload source][session]

RPC stdin EOF requests orderly disposal. Unexpected exits, parent exit policy, orphan cleanup, reconnect, cancellation escalation, ownership leases and recovery decisions belong to the host. A transcript's `parentSession` path supplies lineage only. Nothing in that field enforces a child's lifetime when its parent exits. [RPC][rpc] [Format][format]

### Transcript identities and minimal cross-reference markers

The session header has a session ID, normally a UUID but caller-supplied IDs are supported. Entries have stable `id`, `parentId` and ISO timestamp. IDs are usually short hex with UUID fallback, so they must not be assumed globally unique. Display names and timestamps are not identifiers. A transcript reference needs the session identity and entry ID, plus a way to locate that transcript. [Format][format] [Manager][manager]

RPC `get_entries({ since })` returns entries strictly after a stable cursor in append order, including abandoned branches and pre-compaction history, plus current `leafId`. An unknown cursor errors rather than silently guessing. `get_tree` exposes branches. `get_messages` is a context view without session-entry IDs and is not a substitute for the raw history. The SDK offers `getEntry`, `getEntries`, `getBranch`, `getLeafId`, and provenance-preserving `buildSessionProjection`. [Commands][commands] [Manager][manager]

The normal message events carry messages, not a universal persisted entry ID. `entry_appended` covers custom `pi.appendEntry()` entries, not every transcript append. Reconciliation through the authoritative manager or `get_entries` is available; timestamp matching alone would be ambiguous. Extension `message_end` transformations happen before final persistence, so reading the current leaf inside that hook is not a reliable universal pointer to the message being finalized. [JSON][json] [Session source, `_handleAgentEvent`][session] [Extension types][extension-types]

Compaction preserves original entries. `context_edit` changes future model context without modifying raw target metadata/history/accounting. Fork/clone create a new session and copy selected history; retained entry IDs can be repeated in the new session, while labels may be recreated. A session-qualified entry reference is therefore more precise than an entry ID alone. [Format][format] [Manager, `createBranchedSession` and `forkFrom`][manager]

Top-level assistant tool calls have `toolCall.id`, correlated with result `toolCallId`. Nested calls have `<parent id>/<n>` and `parentToolCallId` in lifecycle events, but no separate call/result transcript entries. Their parent result's `nestedCalls` is bounded to 256 records, omits oversized arguments above 8 KiB per call or 32 KiB per parent result, and marks loss with `complete: false`. It records metadata, never nested result payloads. A crash before the parent result can leave only transient nested execution events. A marker cannot promise a separate authoritative transcript payload for each nested result. [Extensions][extensions] [Message types][messages]

Pi supports labels and non-context custom entries, so minimal local markers can cross-reference authoritative transcripts without duplicating text, prompts, arguments or tool output. It supplies no documented browser-style universal `pi://session/entry` link scheme. A future resolver must account for transcript deletion, moves, copied session IDs, old-format migration, forks and missing entries. This is a feasibility constraint, not a chosen marker schema. [Format][format] [Manager][manager]

### Models and usage

Each session can select a model and thinking level. SDK options accept a `ModelRuntime`, explicit model, settings and scoped cycling models. `setModel()` validates authentication and records selection; session-only changes do not alter global defaults unless requested. Thinking levels clamp to model capabilities. RPC provides model enumeration, selection, cycling and thinking commands. Scoped/cycling lists are not documented as an authorization boundary for arbitrary extension model calls. [SDK][sdk] [Session source][session] [Commands][commands]

Virtual models may choose physical models per request. `model_change` records the selected virtual model; assistant messages record the physical model and thinking level, with provider response metadata where available. Consequently, selected model and billed/dispatched model need not be identical. Resuming a missing virtual model can fall back to the last physical model. [Virtual models][virtual] [Message types][messages]

`getSessionStats()` / RPC `get_session_stats` aggregates full-session usage, including compacted/abandoned history, tool-reported nested usage, summaries and standalone usage entries such as cache warming. This differs from current context-window usage and from an accounting total across multiple child sessions. `reasoning` tokens are already included in `output`. Streaming usage may remain zero until completion; context usage can be unknown after compaction. [Commands][commands] [Messages][messages] [Format][format] [Session source, `getSessionStats`][session]

Codemode classifier/image usage contributes through its tool result. Arbitrary custom tools need to report their own nested model usage. Pi automatically combines usage from nested tool results, so callers must not add it twice. Catalog-based costs can be zero when price metadata is missing, and they are not a provider invoice. A controller that also reports child usage through a parent tool must avoid summing both parent and child totals again. [Codemode][codemode] [Models][models] [Extensions][extensions]

No native cross-child hard resource budget or parent-controlled spending ledger is established by the inspected SDK/RPC contract. Cancellation and observing totals do not guarantee an exact monetary ceiling, particularly with parallel calls, final-only usage and external effects. [SDK][sdk] [RPC types][rpc-types] [JSON][json]

## Uncertainties and evidence limits

- No behavioral experiments were run. Queue, persistence and permission conclusions rely on release documentation, declarations and inspected source, not a reproduced race suite. Provider-specific cancellation, remote idempotency and OS process-tree behavior were not tested.
- Native APIs cover calls using Pi's pipeline, not arbitrary third-party extensions or MCP replacements. Any stronger claim needs an audit of the exact loaded resources and runtime policy.
- Immediate revocation of already-admitted work, bounded queue latency, message delivery across crashes, atomic multi-session state and exactly-once external effects are not guaranteed by the inspected contracts.
- `models.md` says generated images are not saved to disk, while the more specific codemode reference and installed executor say `image()` writes temp files. These describe different layers, but the broad statement must not be used to promise no disk writes. Codemode output can also spill to temp files. [Models][models] [Codemode][codemode]
- Whole-prompt forcing and provider request transforms can differ from the structured transcript. A minimal debug marker should not imply the transcript contains final HTTP bytes. [Extensions][extensions]
- Session ID and path conventions do not constitute globally managed identity or a retention promise. Legacy migration and externally copied/edited/deleted files limit marker resolution. [Format][format]

## Proposed mechanisms, not decisions

These options explain how an extension could use the verified APIs. None is selected here.

1. A controller could create a fresh manager and explicit resource loader for each child, pass only selected task/context, and maintain ownership outside `parentSession`. Either SDK hosting or a supervised RPC subprocess can implement this; the isolation and operational trade-offs above remain.
2. A controller-owned policy record could be checked at each direct/nested `tool_call`, while declarations/search exposure are separately minimized. An access-request operation could carry the requested tool and reason without changing the record. Only a controller path would approve grants. Fail-closed handling, current-policy checks after asynchronous approvals, handler ordering, and admission versus in-flight revocation would need explicit contracts. Model globals and trusted extension execution require separate treatment.
3. An addressed message broker could validate parent/group/recipient authority, attach message IDs, and map delivery onto custom messages or prompt/steer/follow-up operations. Transport acceptance, transcript insertion, model observation and reply would be distinct states. Durability, acknowledgements, deduplication and idle wakeup would need specification rather than being attributed to Pi queues.
4. Control-plane permission replies or cancellation signals could travel outside a waiting session's model-input queue. Tools could release before waiting for a later conversational reply, and waits could have deadlines. This avoids the identified self-queue and cross-session wait cycles without presuming an interruption policy.
5. Recovery could reconcile a durable ownership/policy/message record with session IDs, transcript cursors and process liveness. Interrupted external operations could remain explicitly uncertain rather than being blindly retried. Transcript restoration alone is insufficient.
6. A marker could retain only controller/run identity, event kind, session-qualified entry references and optional tool-call correlation. Payloads would stay in transcripts. Nested-call markers could refer to the containing top-level result and identify incomplete metadata, rather than inventing nonexistent nested entries. The final schema, retention and operator interface remain open.

## Validation performed

- Read the map and full factual ticket, including comments, through `gh-axi`.
- Verified installed version, upstream tag SHA, 15 installed manifest/doc byte matches and nine source-map matches. Inspected checked examples and installed declarations without running model-facing examples.
- Checked that document citations use pinned upstream files and that every cited file exists. Checked Markdown reference definitions and repository whitespace with `git diff --check`.
- Deliverable is one Markdown file only. No extension code, tests, product decisions, production access, credential lookup or paid requests were added. Code compilation/regression tests and upstream's expensive full suite are not applicable to this document-only branch. These are source/document checks, not final reviewer approval.

## Primary sources

All links below use Pi commit `4a080252f06caf63a525d7873c5d94d3c23887e0`, release `v1.4.0`.

[manifest]: https://github.com/earendil-works/pi/blob/4a080252f06caf63a525d7873c5d94d3c23887e0/packages/coding-agent/package.json
[sdk]: https://github.com/earendil-works/pi/blob/4a080252f06caf63a525d7873c5d94d3c23887e0/packages/coding-agent/docs/sdk.md
[sdk-source]: https://github.com/earendil-works/pi/blob/4a080252f06caf63a525d7873c5d94d3c23887e0/packages/coding-agent/src/core/sdk.ts
[resources]: https://github.com/earendil-works/pi/blob/4a080252f06caf63a525d7873c5d94d3c23887e0/packages/coding-agent/src/core/resource-loader.ts
[examples]: https://github.com/earendil-works/pi/blob/4a080252f06caf63a525d7873c5d94d3c23887e0/packages/coding-agent/examples/sdk/README.md
[example-prompt]: https://github.com/earendil-works/pi/blob/4a080252f06caf63a525d7873c5d94d3c23887e0/packages/coding-agent/examples/sdk/03-custom-prompt.ts
[example-control]: https://github.com/earendil-works/pi/blob/4a080252f06caf63a525d7873c5d94d3c23887e0/packages/coding-agent/examples/sdk/12-full-control.ts
[example-sessions]: https://github.com/earendil-works/pi/blob/4a080252f06caf63a525d7873c5d94d3c23887e0/packages/coding-agent/examples/sdk/11-sessions.ts
[example-runtime]: https://github.com/earendil-works/pi/blob/4a080252f06caf63a525d7873c5d94d3c23887e0/packages/coding-agent/examples/sdk/13-session-runtime.ts
[session]: https://github.com/earendil-works/pi/blob/4a080252f06caf63a525d7873c5d94d3c23887e0/packages/coding-agent/src/core/agent-session.ts
[runtime]: https://github.com/earendil-works/pi/blob/4a080252f06caf63a525d7873c5d94d3c23887e0/packages/coding-agent/src/core/agent-session-runtime.ts
[manager]: https://github.com/earendil-works/pi/blob/4a080252f06caf63a525d7873c5d94d3c23887e0/packages/coding-agent/src/core/session-manager.ts
[rpc]: https://github.com/earendil-works/pi/blob/4a080252f06caf63a525d7873c5d94d3c23887e0/packages/coding-agent/docs/rpc.md
[commands]: https://github.com/earendil-works/pi/blob/4a080252f06caf63a525d7873c5d94d3c23887e0/packages/coding-agent/docs/rpc-commands.md
[rpc-types]: https://github.com/earendil-works/pi/blob/4a080252f06caf63a525d7873c5d94d3c23887e0/packages/coding-agent/src/modes/rpc/rpc-types.ts
[rpc-client]: https://github.com/earendil-works/pi/blob/4a080252f06caf63a525d7873c5d94d3c23887e0/packages/coding-agent/src/modes/rpc/rpc-client.ts
[rpc-mode]: https://github.com/earendil-works/pi/blob/4a080252f06caf63a525d7873c5d94d3c23887e0/packages/coding-agent/src/modes/rpc/rpc-mode.ts
[rpc-ui]: https://github.com/earendil-works/pi/blob/4a080252f06caf63a525d7873c5d94d3c23887e0/packages/coding-agent/docs/rpc-extension-ui.md
[json]: https://github.com/earendil-works/pi/blob/4a080252f06caf63a525d7873c5d94d3c23887e0/packages/coding-agent/docs/json.md
[messages]: https://github.com/earendil-works/pi/blob/4a080252f06caf63a525d7873c5d94d3c23887e0/packages/coding-agent/docs/message-types.md
[format]: https://github.com/earendil-works/pi/blob/4a080252f06caf63a525d7873c5d94d3c23887e0/packages/coding-agent/docs/session-format.md
[extensions]: https://github.com/earendil-works/pi/blob/4a080252f06caf63a525d7873c5d94d3c23887e0/packages/coding-agent/docs/extensions.md
[extension-types]: https://github.com/earendil-works/pi/blob/4a080252f06caf63a525d7873c5d94d3c23887e0/packages/coding-agent/src/core/extensions/types.ts
[event-bus]: https://github.com/earendil-works/pi/blob/4a080252f06caf63a525d7873c5d94d3c23887e0/packages/coding-agent/src/core/event-bus.ts
[mcp]: https://github.com/earendil-works/pi/blob/4a080252f06caf63a525d7873c5d94d3c23887e0/packages/coding-agent/docs/mcp.md
[codemode]: https://github.com/earendil-works/pi/blob/4a080252f06caf63a525d7873c5d94d3c23887e0/packages/coding-agent/docs/codemode.md
[code-index]: https://github.com/earendil-works/pi/blob/4a080252f06caf63a525d7873c5d94d3c23887e0/packages/coding-agent/src/extensions/codemode/index.ts
[code-execute]: https://github.com/earendil-works/pi/blob/4a080252f06caf63a525d7873c5d94d3c23887e0/packages/coding-agent/src/extensions/codemode/execute.ts
[cli]: https://github.com/earendil-works/pi/blob/4a080252f06caf63a525d7873c5d94d3c23887e0/packages/coding-agent/docs/cli.md
[configuration]: https://github.com/earendil-works/pi/blob/4a080252f06caf63a525d7873c5d94d3c23887e0/packages/coding-agent/docs/configuration.md
[security]: https://github.com/earendil-works/pi/blob/4a080252f06caf63a525d7873c5d94d3c23887e0/packages/coding-agent/docs/security.md
[models]: https://github.com/earendil-works/pi/blob/4a080252f06caf63a525d7873c5d94d3c23887e0/packages/coding-agent/docs/models.md
[virtual]: https://github.com/earendil-works/pi/blob/4a080252f06caf63a525d7873c5d94d3c23887e0/packages/coding-agent/docs/virtual-models.md
[skills]: https://github.com/earendil-works/pi/blob/4a080252f06caf63a525d7873c5d94d3c23887e0/packages/coding-agent/docs/skills.md
[environment]: https://github.com/earendil-works/pi/blob/4a080252f06caf63a525d7873c5d94d3c23887e0/packages/coding-agent/docs/environment-variables.md
