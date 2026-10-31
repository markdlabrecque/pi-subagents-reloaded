# PRD: Pi Subagents Extension

## Product goal
A reusable Pi extension to spawn isolated coding workers, communicate asynchronously, delegate tools at runtime and survive parent restarts. Pi Durable personal assistant is a key use case.

## Agreed requirements
1. Workers run in separate OS processes and Pi sessions, ideally in separate worktrees for coding. General file locking is out of scope.
2. Flat worker hierarchy: a worker cannot spawn another worker. A director is an independent session that can own workers.
3. A worker has one original assignment; it may receive related follow-ups, but not unrelated assignments.
4. Parent-worker messaging is asynchronous and durable. Default delivery is at safe checkpoints; optional urgent delivery requires parent authorization.
5. Workers can request more tools, with rationale, scope and duration, without blocking unrelated work. Parent can approve, narrow or deny.
6. Sibling messaging is off by default. A worker may request a scoped messaging tool; the parent approves permitted recipients.
7. Completion and retirement are separate: completed workers idle, retain context, and can be re-engaged or retired by parent.
8. Agent milestones and code-level heartbeats coexist. Suggested heartbeat every 15 seconds, suspect after three missed heartbeats.
9. If parent dies, children continue within existing permissions; pending approvals and messages queue. Parent reattaches and automatically processes durable unacknowledged events.
10. The supervisor starts on boot, even without Pi Durable, and never decides task outcomes.
11. Director orchestration is optional, preferably a skill/configuration above the extension. Personal assistant controls directors as peers; retiring a director cascades to all its workers.

## Extension boundary
Related PRD: [Pi Session Navigator](../../pi-session-navigator/docs/Pi-Session-Navigator-PRD.md). This link assumes sibling checkouts; replace it with a repository URL if published separately.

The extensions are designed primarily for use together, but each must remain useful without the other.

- Subagents owns orchestration through its supervisor: spawning, assignments, worker lifecycle, messaging, grants, recovery and managed execution.
- Navigator owns global session discovery, search, viewing, switching and safe native resume. It is an optional client of the supervisor, not another orchestration service.
- Without Navigator, Subagents provides tools and basic status output for managing workers, messages, approvals and retirement.
- With Navigator, the supervisor exposes relationships, status, assignments, events and authorized control actions. Navigator requests retirement or permission decisions through this API rather than killing processes or changing grants itself.
- Subagents may offer an "open in Navigator" action when that capability is available. Otherwise it returns the session ID and basic status. Navigator is never required to manage a worker.
- Runtime capability discovery must tolerate an absent, disconnected or incompatible counterpart. Missing integration must not prevent standalone operation.

The supervisor owns durable command queues, execution leases, authorization and single-executor enforcement for managed sessions. Viewing a session never grants control. Navigator does not duplicate these mechanisms.

## Non-goals
Arbitrary hierarchical workers, generic file locks, autonomous supervisor decisions, automatic worktree deletion, cross-host orchestration in v1, full dashboard UI.

## Functional requirements
### Spawning and isolation
- SA-01 Spawn unique worker process/session with immutable assignment ID, parent ID, and optional worktree.
- SA-02 Prevent worker spawning. Apply configurable concurrency and resource budgets.
- SA-03 Track `owner` separately from `controller`; no implicit delegation through controller edges.
### Messages
- SA-04 Persist each message before acknowledgment with ID, sender, recipient, assignment, type, priority, timestamp and state.
- SA-05 Types: instruction, progress, completion, question, permission_request, permission_decision, ack, cancel, system_event.
- SA-06 At-least-once delivery with deduplication, per-recipient ordering, retries, queue limits and dead-letter status.
- SA-07 Checkpoint delivery by default; urgent must not interrupt unsafe tool calls.
### Permissions
- SA-08 Requests specify tool, reason, scope, duration and correlation ID. Worker continues other tasks.
- SA-09 Parent grants, narrows or denies. Grants expire and can be revoked; check at tool invocation.
- SA-10 No parent may delegate greater authority than it possesses. Denied/unanswered requests never auto-grant.
- SA-11 Verify Pi supports runtime tool changes; otherwise enforce via guarded tool proxy or safe-point reconfiguration.
### Lifecycle
- SA-12 Normal states: created → starting → running → completing → idle → running (follow-up) or retiring → retired.
- SA-13 Exceptional states: disconnected, failed, cancelled, recovery_needed.
- SA-14 Completion is not retirement. Follow-ups must be tied to original assignment.
- SA-15 Director retirement gracefully cancels all workers, escalates after timeout, and retains logs/worktrees unless explicitly cleaned.
### Reliability
- SA-16 Heartbeats and process reconciliation do not consume model tokens.
- SA-17 Parent crash does not kill workers; queue completions and approval requests.
- SA-18 Reconnect returns active workers, outstanding approvals and ordered unacknowledged events.
- SA-19 Use durable operation IDs to prevent duplicate spawns and repeated actions during replay.
- SA-20 Supervisor restart reconciles existing processes rather than blindly spawning replacements.

## Data and API sketch
Tables: sessions, relationships, assignments, messages, acknowledgments, permission_requests, grants, operations, audit_events.

Operations: spawn, list, message, complete, follow_up, retire, request_permission, decide_permission, revoke_permission, heartbeat, reattach, ack_event.

Example request:
```json
{"type":"permission_request","request_id":"r-123","tool":"message_agent","scope":{"recipients":["worker-8"]},"reason":"Coordinate API contract","duration_seconds":1800}
```

## Security
Authenticated local IPC; least privilege; expiry/revocation; audit all grants, control transfers and retirements. Cooperative tool permissions alone do not sandbox shell/network access; consider OS-level isolation for untrusted workers. Protect message queues against unbounded growth. Preserve artifacts after cancellation unless cleanup is explicitly authorized.

## Acceptance criteria
- [ ] Two workers run in distinct sessions/worktrees.
- [ ] Worker requests a tool and continues work; approval takes effect safely.
- [ ] Sibling grant allows only specified recipient; revocation and expiry work.
- [ ] Completion idles; related follow-up resumes; parent retires.
- [ ] Parent crash leaves workers running and messages durable.
- [ ] Parent and supervisor restart recover without duplicate jobs.
- [ ] Controller retires independent director and cascades to its workers.
- [ ] Supervisor never autonomously approves or assigns work.
- [ ] Worker management, messaging, approvals and retirement work without Navigator installed.
- [ ] Installing both extensions uses one authoritative supervisor and executor per managed session; unavailable or incompatible integration does not break standalone tools.

## Implementation phases
1. Feasibility spike: Pi hooks, Pi Durable compatibility, tool enforcement, session persistence.
2. Supervisor + spawn + messaging + lifecycle.
3. Durable outbox, heartbeats, recovery and reconciliation.
4. Dynamic permissions and sibling messaging.
5. Director control edges and Navigator integration.
6. Security, fault injection and documentation.

## Open questions
Default idle timeout, maximum workers, offline resource budget, retention, worktree cleanup confirmation, actual Pi extension hooks, safe live tool updates and OS sandbox choice.

---

# Architecture and conventions
Draft v0.1, 2026-10-31. These are product requirements, not verified claims about Pi APIs. Validate extension hooks and live session control in a spike.

The supervisor is a non-LLM service, started by systemd on Omarchy, with a durable registry, message outbox, process monitoring and authorization. Suggested implementation: TypeScript/Node, SQLite WAL and Unix socket IPC. This is a proposal, not a locked technology choice.

A **worker ownership** relationship is distinct from a **director control** relationship. Directors are independent top-level Pi sessions, controlled by the personal assistant and owning flat worker groups. Workers cannot spawn workers.

## Shared integration contract
- Stable session, agent, assignment, operation and message IDs.
- Authenticated local IPC with per-operation authorization, never trust caller-supplied IDs alone.
- Registry records sessions, processes, owner/controller edges, status, heartbeat, and audit history.
- Event stream uses durable sequence numbers, acknowledgments and idempotency keys.
- Session Navigator reads registry data through the supervisor API, never its database or other private storage.
- The supervisor API is versioned independently of extension releases and advertises capabilities. Exact discovery, transport and Pi hooks remain subject to feasibility validation.
- Neither extension imports the other's implementation. Shared IDs and the public API connect them; installation together must not create duplicate supervisors or competing session executors.
- Supervisor enforces policy but never makes LLM decisions.
