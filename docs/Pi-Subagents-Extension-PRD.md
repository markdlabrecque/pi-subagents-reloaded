# PRD: Pi Subagents Extension

## Product goal
A reusable Pi extension to spawn isolated coding workers, communicate asynchronously and delegate tools at runtime within the owning Pi process's lifetime.

## Agreed requirements
1. Workers run in separate OS processes and Pi sessions, ideally in separate worktrees for coding. General file locking is out of scope.
2. Flat worker hierarchy: a worker cannot spawn another worker. Every top-level agent is a director, whether it owns workers or not.
3. A worker has one original assignment; it may receive related follow-ups, but not unrelated assignments.
4. Parent-worker messaging is asynchronous while the parent is running. Default delivery is at safe checkpoints; optional urgent delivery requires parent authorization. Durable delivery, replay and offline queues are out of scope.
5. Workers can request more tools, with rationale, scope and duration, without blocking unrelated work. Parent can approve, narrow or deny.
6. Sibling messaging is off by default. A worker may request a scoped messaging tool; the parent approves permitted recipients.
7. Completion and retirement are separate: completed workers idle, retain context, and can be re-engaged or retired by parent.
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
- Every follow-up references the original assignment and carries the parent's affirmation that it belongs to that assignment. The parent extension validates the reference and authorization, not semantic relatedness. The worker must question or refuse an apparently unrelated request.
- Tests for an assigned API change are a related follow-up. An unrelated billing fix requires a new worker, even if the parent requests it in the same conversation.
- Keep the original assignment and active approved instruction/context revision separately from conversational summaries and explicitly available after compaction or controlled worker-runtime recreation while the parent remains running. Restore the approved contents rather than rediscover current files or defaults. If restoration is impossible, pause for parent intervention. This does not promise worker reattachment after parent exit or crash.

#### Feasibility boundary
The [Pi 1.4.0 feasibility research](https://github.com/markdlabrecque/pi-subagents-reloaded/blob/be45fb5d27f808123f168c50c114ff916e61c431/docs/research/pi-feasibility.md) verifies fresh conversations, prompt/resource selection and suppression of default discovery. Fresh history alone does not suppress discovered instructions, skills or extensions. Pi's lineage metadata does not enforce ownership or assignment boundaries.

Approved revisions, exact-content restoration, parent approval, follow-up validation and compaction continuity above are required extension behavior, not native Pi guarantees. Standard RPC has no system-prompt replacement command after startup. Safe revision application therefore needs a supported parent-managed SDK or extension mechanism, or controlled runtime recreation. This specification does not select an implementation or claim that plain RPC already provides one.

### Messages
- SA-04 Identify live messages by sender, recipient, assignment, type, priority, timestamp and state. Exact acknowledgment semantics remain a messaging decision; persistence before acknowledgment is not required.
- SA-05 Types: instruction, progress, completion, question, permission_request, permission_decision, ack, cancel, system_event.
- SA-06 Bound live message queues and define ordering, failure reporting and wait behavior. No durable delivery, replay or offline queue guarantee.
- SA-07 Checkpoint delivery by default; urgent must not interrupt unsafe tool calls.
### Permissions
- SA-08 Requests specify tool, reason, scope, duration and correlation ID. Worker continues other tasks.
- SA-09 Parent grants, narrows or denies. Grants expire and can be revoked; check at tool invocation.
- SA-10 No parent may delegate greater authority than it possesses. Denied/unanswered requests never auto-grant.
- SA-11 Verify Pi supports runtime tool changes; otherwise enforce via guarded tool proxy or safe-point reconfiguration.
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

Normal progression is created → starting → running → completing → idle. A related follow-up moves idle → running. Retirement is separate from turn completion and may begin from any non-retired state.

One-shot and ongoing work use the same lifecycle, without a permanent mode flag. One-shot means no follow-up is expected, not automatic retirement. Explicit follow-ups are accepted only when idle; reject early submissions with the current state rather than silently queueing them. Input during running or completing uses the separately defined live-messaging contract and cannot start overlapping turns.

Completed, cancelled and failed describe turn outcomes, not permanent worker states. Completion or safe cancellation leaves the worker idle. A failed turn may also leave it idle if the runtime remains usable. A failed tool command alone is not a failed turn. Cancellation pending is a flag on an active turn, not another worker state. Waiting for input or permission does not itself complete the turn.

#### Cancellation and shutdown
Cancellation stops the current turn without retiring the worker or discarding context. Report cancellation as pending until stopping is confirmed. Cancellation does not undo file changes or reverse external effects.

The parent extension's recorded ordering settles completion/cancellation races. If completion is recorded first, cancellation returns already completed. If cancellation is accepted first, the turn is cancelled once stopping is confirmed, even if completion arrives late. Keep partial results inspectable. A disconnect during completion is not silently recorded as success.

Cancellation, retirement and normal parent exit use bounded shutdown even during an active tool. Configurable defaults are five seconds for cooperative cancellation, then termination of the worker runtime and its managed subprocesses, then another two seconds before force-killing them. Detached processes and reversal of external effects are not covered. If cancellation kills the runtime, the worker becomes unavailable rather than idle.

Retirement and parent exit prevent new turns, messages that trigger work and tool invocations as soon as shutdown begins. Remain retiring until managed-runtime shutdown is confirmed; uncertainty must be reported rather than marked retired. Repeated retirement requests are safe and do not reset the deadline. Explicit director retirement cascades to its workers; director cancellation alone does not. Preserve transcripts, logs and worktrees; cleanup requires separate authorization.

#### Startup, runtime failure and disconnection
Startup failure makes that worker unavailable, reports the failure and shuts down any partially started process using the bounded sequence. There is no automatic retry. The parent may explicitly spawn a replacement with a new worker identity. Preserve transcripts and files already created.

Runtime death during a turn fails the turn and makes the worker unavailable. There is no automatic recreation. While the parent remains alive, it may explicitly recreate the runtime for the same worker only when the original assignment, active approved instruction/context revision and usable conversation can be restored. Never automatically replay an interrupted turn or tool call. If restoration is impossible, use a new replacement worker.

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
{"type":"permission_request","request_id":"r-123","tool":"message_agent","scope":{"recipients":["worker-8"]},"reason":"Coordinate API contract","duration_seconds":1800}
```

## Security
Authenticate parent-worker communication and enforce owner-only operations, least privilege, expiry and revocation. Record permission decisions and retirements with minimal transcript-linked markers. Cooperative tool permissions do not sandbox shell/network access; OS containment is outside v1. Bound live message queues. Preserve artifacts after cancellation unless cleanup is explicitly authorized.

## Acceptance criteria
- [ ] Two workers run in distinct sessions/worktrees.
- [ ] A fresh worker receives only its explicit instruction/resource selection and selected supporting context, without unrelated parent history or automatic user-global instruction/skill discovery.
- [ ] Changed skill or project instruction files do not alter the active revision. A parent-approved exact-content revision takes effect only at a safe checkpoint; pending and denied changes have no effect.
- [ ] Parent-initiated revisions use the same mechanism. Ordinary messages cannot revise standing instructions, grant tools or change the original assignment.
- [ ] Follow-ups carry the original assignment reference and parent affirmation. Invalid references are rejected; the worker questions an apparently unrelated request rather than accepting reassignment.
- [ ] Compaction and controlled worker-runtime recreation preserve the assignment and active approved revision while the parent remains running. Missing approved contents pause work instead of silently loading replacements.
- [ ] Worker requests a tool and continues work; approval takes effect safely.
- [ ] Sibling grant allows only specified recipient; revocation and expiry work.
- [ ] One-shot and ongoing work share one lifecycle. Completion records a result and confirms readiness before idling; only an explicit related follow-up from idle starts another turn.
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
Default idle timeout, maximum workers, live resource and queue limits, retention, worktree provisioning, crash/orphan reporting, actual Pi extension hooks, automatic director identity across entry points, safe live tool updates and the minimal optional Navigator contract.

---

# Architecture and conventions
These are product requirements, not verified claims about Pi APIs.

The extension runs within the owning Pi process and directly manages separate worker processes and sessions. There is no independent supervisor, boot service, durable command queue or global registry in v1. Storage and transport details remain implementation choices.

A director is any top-level agent. Parent describes an ownership relationship, not a synonym for director; a child is a worker, not its session. Identity is distinct from conversation and process identity. Ownership remains fixed, and workers cannot spawn workers.

Optional Navigator integration uses agreed identifiers and capabilities, not a supervisor API. Neither extension imports the other's implementation or requires the other for standalone use. Persisted transcripts support inspection, not automatic managed-job recovery.
