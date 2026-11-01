# PRD: Pi Subagents Extension

## Product goal
A reusable Pi extension to spawn isolated coding workers, communicate asynchronously and delegate tools at runtime within the owning Pi process's lifetime.

## Agreed requirements
1. Workers run in separate OS processes and Pi sessions, ideally in separate worktrees for coding. General file locking is out of scope.
2. Flat worker hierarchy: a worker cannot spawn another worker. Every top-level agent is a director, whether it owns workers or not.
3. A worker has one original assignment; it may receive related follow-ups, but not unrelated assignments.
4. Parent-worker messaging is asynchronous while the parent is running. Default delivery is at safe checkpoints; optional urgent delivery requires parent authorization. Durable delivery, replay and offline queues are out of scope.
5. Workers receive a parent-removable default inspection set and can request more tools with rationale and scope without blocking unrelated work. Only the owning parent can approve, narrow or deny. Grants have no timed expiry; they end on explicit revocation, retirement, worker-runtime death or parent-session termination.
6. Sibling messaging is off by default. A worker may request a scoped messaging tool; the parent approves permitted recipients.
7. Completion and retirement are separate: completed workers idle and retain context. An authorized parent or peer message wakes an idle worker on its existing assignment; retirement remains parent-controlled.
8. Agent milestones and code-level heartbeats coexist. Suggested heartbeat every 15 seconds, suspect after three missed heartbeats.
9. The parent extension directly launches and manages workers. Parent exit shuts down workers. A parent crash may leave orphaned processes requiring manual cleanup; v1 promises neither continued operation nor automatic reattachment.
10. No supervisor, background service or boot startup is required in v1. Persisted transcripts are records, not durable job recovery.
11. Each director manages only its own workers. Worker grants and instruction revisions remain owner-only. Cross-director super-agent control is deferred.

## Extension boundary
Related PRD: [Pi Session Navigator](../../pi-session-navigator/docs/Pi-Session-Navigator-PRD.md). This link assumes sibling checkouts; replace it with a repository URL if published separately.

Each extension must remain useful without the other.

- Subagents directly owns worker spawning, assignments, lifecycle, live messaging and grants within the owning Pi process.
- Navigator owns global session discovery, search, viewing, switching and safe native resume. It is optional, not a required orchestration service.
- Without Navigator, Subagents provides tools and basic status for managing workers, messages, approvals and retirement.
- Subagents may offer an "open in Navigator" action when available; otherwise it returns session identity and basic status.
- Viewing never grants control. Navigator must not independently alter grants, kill managed workers or resume a session already executing under its parent.
- No shared supervisor API, durable event stream or global execution lease is required. The minimal optional integration contract remains to be decided.
- Missing or incompatible integration must not prevent standalone operation.

## Non-goals
Arbitrary worker nesting, generic file locks, automatic worktree deletion, cross-host orchestration, a full dashboard, OS containment, a supervisor or boot service, durable message delivery/replay, offline approval queues, automatic crash recovery/reattachment, global single-executor guarantees and cross-director super-agent control.

## Functional requirements
### Spawning and isolation
- SA-01 Spawn unique worker process/session with immutable assignment ID, parent ID, and optional worktree.
- SA-02 Prevent worker spawning. Apply configurable concurrency and resource budgets.
- SA-03 Keep worker ownership fixed for the worker's lifetime. Caller-supplied identities alone do not authorize worker management.

### Ownership and director authority
Resolved through [Define supervisor authority and director control](https://github.com/markdlabrecque/pi-subagents-reloaded/issues/8), which reduced v1 to parent-local management.

- Every top-level agent is a director, automatically identified at creation without registration or opt-in, whether it owns workers or not. A session resume preserves director identity; a fork or separate top-level agent gets a new identity.
- Each director can spawn workers within its existing tool authority and resource limits. It manages only its own worker group and cannot spawn on another director's behalf while claiming that director as owner.
- The owning parent alone approves worker tool grants and instruction/context revisions. There is no automatic ownership transfer, super-agent allowlist or cross-director control in v1.
- Director cancellation stops its current activity without automatically cancelling workers. Explicit director retirement cascades to its workers; normal parent exit also shuts workers down. Shutdown follows the lifecycle contract below.
- Parent loss never grants pending requests. Messaging and approvals are live-parent operations, not offline queues.
- This is a trusted-local-agent model, not OS containment.
- Automatic director identity across interactive, SDK and RPC entry points remains a feasibility check. It does not imply a global registry or remote management authority.

### Worker instructions and context
Resolved through [Define parent instructions and child context boundaries](https://github.com/markdlabrecque/pi-subagents-reloaded/issues/4). Terms are defined in [the glossary](../GLOSSARY.md).

#### Initial context and authority
- Spawn starts a fresh conversation. It retains Pi's base operating instructions and the extension's worker restrictions, then adds parent-authored role instructions and the immutable original assignment.
- The parent explicitly selects project instructions, available skills and supporting context. Reusable profiles may provide defaults, but the effective selection must be inspectable. Do not automatically inherit the parent's conversation, system prompt, user-global instructions or discovered skills.
- Selected project instructions apply within their scope. Skills guide work but cannot grant tools or override the assignment boundary. Instruction conflicts require clarification rather than silent precedence guessing.
- Separate instructions from supporting material. Label selected excerpts, documents and tool results with their source. Parent-supplied evidence does not become authoritative instructions merely because the parent supplied it.
- Missing instructional resources or parent discussion require a context request. The worker may read ordinary files within its existing permissions, but reading a file does not promote its contents into standing instructions. Parent history is never automatically available; only deliberately selected material may enter.

#### Approved revisions
- Capture the initial instruction set and selected instructional resource contents as the first approved revision. Keep them fixed unless the owning parent explicitly approves a revision. Ordinary source files being edited are not frozen by this rule.
- A worker may request a specific adjustment with a reason. The parent may approve, narrow or deny it, or initiate a revision without a request. Pending or denied requests leave the active revision unchanged.
- Approval identifies the actual contents being adopted, not merely mutable file paths. Apply approved revisions at safe checkpoints, never halfway through a tool call. File edits, reloads and restarts must not silently change the active selection.
- Preserve previous revisions for audit and identify exactly one active revision. Later conversational messages remain subordinate to standing instructions; a conflicting message requires clarification or the explicit revision path, not an implicit override.
- No revision can change the original assignment, remove the extension's worker restrictions or grant tool authority by text alone. Tool grants remain separately changeable through the permission mechanism.

#### Follow-ups and continuity
- Every parent follow-up references the original assignment and carries the parent's affirmation that it belongs to that assignment. An authorized parent message to an idle worker is a follow-up; no separate wake operation is required. A recipient-scoped peer grant also authorizes messages to wake that recipient on its existing assignment. The parent extension validates the assignment reference and authorization, not semantic relatedness. The worker must question or refuse an apparently unrelated request from either sender.
- Tests for an assigned API change are a related follow-up. An unrelated billing fix requires a new worker, even if the parent requests it in the same conversation.
- Keep the original assignment and active approved instruction/context revision separately from conversational summaries and explicitly available after compaction or controlled worker-runtime recreation while the parent remains running. Restore the approved contents rather than rediscover current files or defaults. If restoration is impossible, pause for parent intervention. This does not promise worker reattachment after parent exit or crash.

#### Feasibility boundary
The [Pi 1.4.0 feasibility research](https://github.com/markdlabrecque/pi-subagents-reloaded/blob/be45fb5d27f808123f168c50c114ff916e61c431/docs/research/pi-feasibility.md) verifies fresh conversations, prompt/resource selection and suppression of default discovery. Fresh history alone does not suppress discovered instructions, skills or extensions. Pi's lineage metadata does not enforce ownership or assignment boundaries.

Approved revisions, exact-content restoration, parent approval, follow-up validation and compaction continuity above are required extension behavior, not native Pi guarantees. Standard RPC has no system-prompt replacement command after startup. Safe revision application therefore needs a supported parent-managed SDK or extension mechanism, or controlled runtime recreation. This specification does not select an implementation or claim that plain RPC already provides one.

### Messages
Resolved through [Define parent and peer message delivery](https://github.com/markdlabrecque/pi-subagents-reloaded/issues/5). These are required extension behaviors within the live parent's lifetime, not native Pi delivery guarantees.

#### Addressing, authority and message meaning
- SA-04 Address the parent or stable worker identities within its worker group, not process or session IDs. The extension supplies authenticated sender identity rather than trusting a caller-supplied sender field. Peer grants authorize specific recipients; no cross-director messaging in v1.
- Identify messages by message ID, authenticated sender, recipient, recipient assignment where applicable, type, priority, timestamp and delivery state. Correlated requests and replies reference the request they answer.
- SA-05 Conversational messages may carry instructions, progress, questions, replies and notifications. Cancellation, permission decisions, instruction/context revisions and lifecycle transitions remain structured, authorized operations. Their notifications may appear in conversation, but a message type or its text never performs the operation or grants authority. A peer cannot cancel another worker or revise its standing instructions.
- Ordinary messages remain subordinate to the immutable assignment and active approved instructions. A completion notification alone does not establish that lifecycle completion was recorded.

#### Send, delivery and replies
- Send returns promptly with accepted or rejected and a message ID. Accepted means admitted to the live queue, not delivered or acted upon. Delivered means inserted into the recipient's conversation at an allowed checkpoint, not understood or completed.
- Delivery status and failure reasons are inspectable. Do not emit separate receipt messages or define generic acknowledged or processed states. When confirmation matters, request a correlated reply that states the relevant fact, such as whether the worker has stopped editing a file.
- Waiting for a correlated reply is a separate operation with a finite deadline. Incoming messages must remain deliverable while waiting, so mutual waits cannot deadlock the messaging system. A timeout means no reply arrived within the deadline; it does not cancel the message or requested work. A known delivery failure ends an explicit wait with that failure.

#### Wakeup, checkpoints and ordering
- An authorized parent message wakes an idle worker and starts a turn on its existing assignment, without requiring a separate follow-up operation. A recipient-scoped peer messaging grant includes the same wake permission by default. Messages to an active worker never start an overlapping turn. Shutdown still prevents new work.
- SA-07 Normal messages arrive between agent turns. An explicit reply wait must also permit incoming delivery rather than holding the recipient inaccessible until the turn ends. Parent-authorized urgent messages arrive at the next safe opportunity within an active turn. Neither priority interrupts an executing tool; urgent is not cancellation or a guarantee of immediate action.
- Preserve acceptance order within each priority for a recipient. Urgent messages may overtake normal messages.

#### Bounded live queues and duplicate sends
- SA-06 Bound each recipient's pending queue by both message count and bytes, not tokens. Pending means accepted but not yet delivered; delivery releases queue capacity. Also bound individual message size.
- Limits are configurable by the director within its overall resource budget; workers cannot raise them. Numeric defaults, aggregate budgets and bounded deduplication retention belong to [Define v1 resource, workspace and retention policies](https://github.com/markdlabrecque/pi-subagents-reloaded/issues/11).
- Reject a send that would exceed either queue limit immediately with `queue_full`, identifying the relevant limit and current usage. Reject oversized individual messages outright. Never silently evict accepted messages, block send waiting for capacity or automatically retry.
- A sender-supplied request ID supports bounded live deduplication. Within the retained deduplication window, retrying the same send returns its existing message ID rather than delivering it twice. Reusing that ID with different contents is an error. This is not durable deduplication or an exactly-once execution guarantee.

#### Failure, revocation and visibility
- Recheck recipient availability and peer authorization at delivery, not only admission. Recipient failure before delivery fails queued messages; do not replay them into a recreated runtime. Grant revocation or the end of its authorized lifetime before delivery fails affected queued peer messages. Already-delivered messages cannot be recalled.
- Parent exit ends live delivery, with no survival or replay guarantee. Normal shutdown rejects messages that would trigger work as soon as shutdown begins. Parent loss never turns a pending permission request into approval.
- Do not mirror peer exchanges or automatically push their failure notices into director context. Metadata, delivery failures and transcript-linked content remain inspectable, and the director may request a report or summary. Explicitly addressed messages to the director are distinct from mirroring peer exchanges.
### Permissions
Behavior agreed through live discussion in [Define tool grants, revocation, and access requests](https://github.com/markdlabrecque/pi-subagents-reloaded/issues/6). The ticket remains open for a Pi feasibility and final consistency check. These are required extension behaviors, not verified native Pi guarantees. The no-timed-expiry decision supersedes earlier expiry requirements, including the earlier messaging decision.

#### Default grants and catalog
- Workers receive `read`, `grep`, `find` and `ls` by default, plus narrowly defined worker-control operations for parent messaging, status reporting and permission requests. The owning parent can remove defaults. Defaults cannot exceed its authority and do not imply assignment-directory confinement.
- `bash`, `write`, `edit`, peer messaging and MCP tools are opt-in. Workers never receive permission to spawn descendants or approve grants.
- Grants identify exact tools, including provider identity for MCP tools. No wildcard grant automatically covers future catalog additions. Reconnecting the same verified provider does not renew or broaden authority; a changed provider identity requires fresh approval.
- Discovery is not permission. Distinguish requestable tools, parent-forbidden tools and tools unavailable in the runtime. Loading a definition does not authorize execution. Forbidden and unavailable tools return distinct explanations rather than repeated approval prompts.

#### Requests and decisions
- SA-08 Requests specify exact tool, rationale, requested scope and a worker-scoped correlation ID. No duration is required. Workers continue unrelated work. Repeating the same ID and contents returns the existing request; conflicting reuse is an error.
- SA-09 Only the owning parent grants, narrows or denies. Decisions identify the actual approved authority and notify the worker. Withdrawn requests and requests belonging to an ended runtime or lifetime cannot receive late approval.
- SA-10 No parent may delegate greater authority than it currently possesses. Loss of parent authority removes corresponding worker authority. Denied and unanswered requests never grant access.
- V1 supports named-tool grants and recipient-scoped peer messaging, without promising general path, command or network restrictions. A request's intended use is not an enforced restriction. In particular, `bash` authorizes shell execution, not only read-only commands or commands in a particular directory.
- If a requested restriction cannot be enforced, the owning director judges whether granting the tool without that restriction is permissible. It may explicitly approve the actual broader authority, select a supported narrower alternative or deny. Do not silently discard restrictions or automatically reject solely because a restriction is unsupported. Record and notify the worker of the actual authority granted.

#### Invocation enforcement and indirect access
- Check authorization at each invocation, requiring a live authenticated connection to the owning parent's authorization mechanism. A disconnected worker cannot start calls using cached grants. Disconnection does not authorize continued work or imply automatic reconnection.
- Direct calls and indirect calls through codemode, deferred tools, MCP or other wrappers follow the same authorization rules. Granting a wrapper does not implicitly grant its underlying tools; each underlying invocation must pass authorization.
- If a wrapper cannot enforce underlying checks, treat it as a broad capability requiring explicit parent approval, analogous to shell access. Such approval never authorizes owner-only management operations or worker spawning. This is trusted-local authorization, not OS containment against malicious workers or shell/network effects.
- Revocation blocks subsequent invocations, without automatically interrupting an already-authorized running tool or undoing effects. Cancellation and retirement use their separate shutdown rules. Peer authorization is also rechecked at message delivery.

#### Grant lifetime and history
- Grants have no timed expiry or renewal requirement. They survive turn completion, idle time and related follow-ups until explicitly revoked by the owning director or their authorized lifetime ends.
- Retirement, worker-runtime death and parent-session termination end all grants. A recreated runtime requires fresh owning-director grants, including fresh authorization of any default set. Retained transcripts cannot restore permission. Disconnection immediately blocks new invocations as described above.
- Parent and worker can inspect effective grants and request outcomes. Keep minimal transcript-linked history of grants, denials, explicit broader-access approvals, revocations and lifetime termination, recording who authorized what, for which worker, when and why. Do not duplicate tool payloads or transcripts. Retained history is evidence, not authority replayable after parent loss.

#### Remaining feasibility check
- SA-11 Reuse the completed Pi feasibility research to check invocation-time authorization, live tool changes and indirect access. Identify unsupported behavior or factual gaps before closing the permission decision. A visible tool allowlist or safe-point reconfiguration alone must not be treated as proof of invocation-time enforcement. No SDK hook, guarded proxy or other implementation is selected here.

### Lifecycle
Resolved through [Define one-shot and ongoing session lifecycle](https://github.com/markdlabrecque/pi-subagents-reloaded/issues/3). These are required extension behaviors, not claims about native Pi guarantees.

#### Worker states and turn outcomes
- `created`: identity, fixed ownership and immutable assignment recorded; no runtime launched yet.
- `starting`: launching or explicitly recreating the runtime; not ready for follow-ups.
- `running`: one turn is active, including tool execution or waiting for input or permission.
- `completing`: finalizing the turn's result and confirming runtime readiness. Only after the parent extension records the result and confirms readiness does the worker become idle. This is local bookkeeping, not acknowledgment from the parent model.
- `idle`: no active turn; context retained and ready for a related follow-up.
- `unavailable`: runtime failed or disconnected; no work accepted.
- `retiring`: permanently closing the worker and confirming shutdown.
- `retired`: managed lifetime ended; cannot resume or recreate.

Normal progression is created → starting → running → completing → idle. A related follow-up or authorized message moves idle → running. Retirement is separate from turn completion and may begin from any non-retired state.

One-shot and ongoing work use the same lifecycle, without a permanent mode flag. One-shot means no follow-up is expected, not automatic retirement. A separate explicit follow-up operation, if exposed, is accepted only when idle; reject early submissions with the current state rather than silently queueing new turns. Ordinary authorized messages are accepted through the live-messaging contract, wake idle workers and never start overlapping turns during running or completing. The messaging decision supersedes the earlier requirement that only a separate explicit follow-up could wake an idle worker.

Completed, cancelled and failed describe turn outcomes, not permanent worker states. Completion or safe cancellation leaves the worker idle. A failed turn may also leave it idle if the runtime remains usable. A failed tool command alone is not a failed turn. Cancellation pending is a flag on an active turn, not another worker state. Waiting for input or permission does not itself complete the turn.

#### Cancellation and shutdown
Cancellation stops the current turn without retiring the worker or discarding context. Report cancellation as pending until stopping is confirmed. Cancellation does not undo file changes or reverse external effects.

The parent extension's recorded ordering settles completion/cancellation races. If completion is recorded first, cancellation returns already completed. If cancellation is accepted first, the turn is cancelled once stopping is confirmed, even if completion arrives late. Keep partial results inspectable. A disconnect during completion is not silently recorded as success.

Cancellation, retirement and normal parent exit use bounded shutdown even during an active tool. Configurable defaults are five seconds for cooperative cancellation, then termination of the worker runtime and its managed subprocesses, then another two seconds before force-killing them. Detached processes and reversal of external effects are not covered. If cancellation kills the runtime, the worker becomes unavailable rather than idle.

Retirement and parent exit prevent new turns, messages that trigger work and tool invocations as soon as shutdown begins. Remain retiring until managed-runtime shutdown is confirmed; uncertainty must be reported rather than marked retired. Repeated retirement requests are safe and do not reset the deadline. Explicit director retirement cascades to its workers; director cancellation alone does not. Preserve transcripts, logs and worktrees; cleanup requires separate authorization.

#### Startup, runtime failure and disconnection
Startup failure makes that worker unavailable, reports the failure and shuts down any partially started process using the bounded sequence. There is no automatic retry. The parent may explicitly spawn a replacement with a new worker identity. Preserve transcripts and files already created.

Runtime death during a turn fails the turn and makes the worker unavailable. There is no automatic recreation. While the parent remains alive, it may explicitly recreate the runtime for the same worker only when the original assignment, active approved instruction/context revision and usable conversation can be restored. Never automatically replay an interrupted turn or tool call. Runtime recreation requires fresh owning-parent tool grants; restored conversation and instructions do not restore authority. If restoration is impossible, use a new replacement worker.

Loss of the parent-worker control connection makes the worker unavailable even if its process may still be alive. The interrupted turn's outcome is unknown unless a result was already recorded. Attempt bounded shutdown of the known process. Do not accept follow-ups or launch a second runtime while the original might still execute. No automatic reconnection or reattachment is promised.

Detailed detection, orphan reporting and manual cleanup remain with [Define parent-lifetime failure boundaries and orphan handling](https://github.com/markdlabrecque/pi-subagents-reloaded/issues/9). Idle timeout and resource policy remain with [Define v1 resource, workspace and retention policies](https://github.com/markdlabrecque/pi-subagents-reloaded/issues/11).
### Parent lifetime and failure boundaries
- SA-16 Code-level process monitoring does not consume model tokens.
- SA-17 Normal parent exit shuts down its workers and preserves transcripts and worktrees.
- SA-18 Parent crashes may leave orphaned worker processes requiring manual cleanup. Continued operation, automatic reattachment and replay are not supported guarantees.
- SA-19 Pending messages and permission requests need not survive parent loss and never become approvals because of it.
- SA-20 No durable operation store, restart reconciliation, execution lease or global single-executor mechanism is required.

## Data and API sketch
Parent-local records cover workers, sessions, assignments, messages, permission requests, grants and instruction/context revisions. No database or storage stack is selected.

Operations: spawn, list, message, complete, follow_up, retire, request_permission, decide_permission, revoke_permission.

Transcripts remain available after exit, but are not a recoverable job registry.

Example request:
```json
{"type":"permission_request","request_id":"r-123","tool":"message_agent","scope":{"recipients":["worker-8"]},"reason":"Coordinate API contract"}
```

## Security
Authenticate parent-worker communication and enforce owner-only operations, least privilege, invocation-time checks, explicit revocation and grant lifetime termination. Record permission decisions and retirements with minimal transcript-linked markers. Cooperative tool permissions do not sandbox shell/network access; OS containment is outside v1. Bound live message queues. Preserve artifacts after cancellation unless cleanup is explicitly authorized.

## Acceptance criteria
- [ ] Two workers run in distinct sessions/worktrees.
- [ ] A fresh worker receives only its explicit instruction/resource selection and selected supporting context, without unrelated parent history or automatic user-global instruction/skill discovery.
- [ ] Changed skill or project instruction files do not alter the active revision. A parent-approved exact-content revision takes effect only at a safe checkpoint; pending and denied changes have no effect.
- [ ] Parent-initiated revisions use the same mechanism. Ordinary messages cannot revise standing instructions, grant tools or change the original assignment.
- [ ] Parent follow-ups carry the original assignment reference and parent affirmation. Invalid references are rejected; the worker questions an apparently unrelated parent or peer request rather than accepting reassignment.
- [ ] An authorized parent message wakes an idle worker without a separate follow-up operation. A recipient-scoped peer grant includes wake permission by default. Neither starts overlapping turns or bypasses shutdown or assignment boundaries.
- [ ] Message addressing uses stable worker identities and extension-authenticated senders. Caller-supplied identities cannot impersonate a parent or peer; cross-director delivery is rejected.
- [ ] Send reports accepted or rejected with a message ID. Delivery is inspectable, not proof of action. Confirmation uses correlated replies rather than automatic receipt messages or generic processed states.
- [ ] Text labelled as a permission decision, cancellation or instruction revision cannot perform the corresponding authorized operation.
- [ ] Normal delivery occurs between turns; authorized urgent messages may overtake normal ones at safe opportunities without interrupting tools. Acceptance order is preserved within each priority. Explicit reply waits keep incoming delivery operational and always have a finite deadline.
- [ ] Configurable pending-message count and byte limits reject overload with `queue_full`, limit and usage; oversized messages are rejected. Delivery releases capacity. No eviction, blocked send or automatic retry occurs.
- [ ] Duplicate live sends within the retained deduplication window return the existing message ID; conflicting reuse is rejected. No exactly-once execution or persistence guarantee is implied.
- [ ] Worker failure with queued messages fails those messages without replay into a recreated runtime. Revocation or grant lifetime termination before delivery fails queued peer messages; already-delivered messages cannot be recalled.
- [ ] Peer messages and their failures are not mirrored into director context. The director can explicitly inspect them or request a report or summary.
- [ ] Compaction and controlled worker-runtime recreation preserve the assignment and active approved revision while the parent remains running. Missing approved contents pause work instead of silently loading replacements.
- [ ] Default access contains only the parent-authorized inspection set and worker-control operations. Defaults can be removed and never exceed parent authority; shell, writes, peers and MCP tools are opt-in.
- [ ] Discovery distinguishes requestable, forbidden and unavailable tools without granting execution. Exact tool and provider identities prevent future catalog additions or changed providers from inheriting grants.
- [ ] Worker requests a tool with rationale, scope and correlation ID while continuing unrelated work. Duplicate identical requests return the existing request; conflicting reuse fails. Withdrawn or ended-runtime requests cannot receive late approval.
- [ ] Unsupported restrictions trigger an explicit owning-director risk decision. Any approved broader access is accurately recorded and communicated; no restriction is silently dropped and unanswered requests never grant access.
- [ ] Direct and indirect invocations require current authority and a live authenticated parent authorization connection. A wrapper grant alone cannot authorize underlying tools; a wrapper lacking underlying checks requires explicit broad-capability approval without permitting owner-only operations or worker spawning.
- [ ] Revocation blocks new calls without interrupting already-authorized running tools or undoing effects. Sibling grants restrict recipients and are rechecked at delivery.
- [ ] Grants have no timed expiry and survive completion, idle time and related follow-ups. Revocation, retirement, runtime death or parent-session termination ends access. Runtime recreation requires fresh grants; transcripts cannot restore authority.
- [ ] Effective grants and request outcomes are inspectable with minimal transcript-linked authorization history, without duplicated payloads or replayable authority.
- [ ] One-shot and ongoing work share one lifecycle. Completion records a result and confirms readiness before idling; a related follow-up or authorized message from idle starts another turn on the same assignment.
- [ ] Safe cancellation preserves context and leaves the worker idle. Runtime-killing cancellation leaves it unavailable; cancellation remains pending until stopping is confirmed.
- [ ] Recorded completion/cancellation ordering determines the outcome; late completion cannot override accepted cancellation. Partial results remain inspectable.
- [ ] Startup failure reports unavailable, shuts down partial processes and preserves artifacts without automatic retry.
- [ ] Runtime death fails the active turn. Explicit recreation requires restorable assignment, approved context and conversation, a live parent and no possibly executing old runtime. Interrupted work is not automatically replayed.
- [ ] Control disconnection blocks work and reports an unknown interrupted outcome unless already recorded; no automatic reconnection or reattachment occurs.
- [ ] Retirement and parent exit immediately block new work and use configurable five-second cooperative and two-second termination grace periods before force-kill escalation, including active tools.
- [ ] Retirement is irreversible and reported complete only after confirmed shutdown. Repeated requests do not reset the deadline; transcripts and worktrees remain intact.
- [ ] Normal parent exit shuts down workers while preserving transcripts and worktrees.
- [ ] Parent crash is reported as unsupported for automatic recovery; orphan cleanup and lost live requests are documented without implying safe continued work.
- [ ] Parent loss never approves a pending request.
- [ ] Directors manage only their own worker groups. No cross-director authority or super-agent status exists in v1.
- [ ] Top-level agents receive director identities automatically, with no opt-in; session resume preserves identity and a separate agent or fork gets a new one.
- [ ] Director cancellation does not automatically cancel workers; explicit retirement cascades.
- [ ] Worker management, messaging, approvals and retirement work without Navigator, Pi Durable or a background service.
- [ ] Missing or incompatible Navigator integration does not break standalone tools; viewing never grants control.

## Specification and implementation boundary
This decision map specifies behavior only. Implementation planning follows once the remaining lifecycle, messaging, permission, failure-boundary, integration and policy decisions are resolved. Reuse the completed Pi feasibility research; verify uncovered automatic-identity entry points and safe live instruction/tool changes before treating them as supported.

A supervisor, durable delivery/recovery and cross-director control may be reconsidered later if needed. They are not prerequisites or scheduled implementation phases for v1.

## Open questions
Default idle timeout, maximum workers, live resource and queue-limit values, aggregate budgets and deduplication retention, transcript retention, worktree provisioning, crash/orphan reporting, actual Pi extension hooks, automatic director identity across entry points, safe live tool updates and the minimal optional Navigator contract.

---

# Architecture and conventions
These are product requirements, not verified claims about Pi APIs.

The extension runs within the owning Pi process and directly manages separate worker processes and sessions. There is no independent supervisor, boot service, durable command queue or global registry in v1. Storage and transport details remain implementation choices.

A director is any top-level agent. Parent describes an ownership relationship, not a synonym for director; a child is a worker, not its session. Identity is distinct from conversation and process identity. Ownership remains fixed, and workers cannot spawn workers.

Optional Navigator integration uses agreed identifiers and capabilities, not a supervisor API. Neither extension imports the other's implementation or requires the other for standalone use. Persisted transcripts support inspection, not automatic managed-job recovery.
