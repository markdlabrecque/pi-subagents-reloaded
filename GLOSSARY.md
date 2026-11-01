# Pi subagents

Terms for parent-controlled workers and their assignment boundaries.

## Language

**Worker**:
A managed agent with one immutable original assignment. Its identity is distinct from its conversation and running executor.

**Turn**:
One episode of a worker's activity on its assignment, ending in completion, cancellation or failure. A related follow-up starts a new turn without changing the assignment.

**Completion**:
The successful end of a turn, distinct from the end of a worker's managed lifetime.
_Avoid_: Retirement as a synonym for completion

**Cancellation**:
A request to stop the current turn without ending the worker's managed lifetime. Cancellation does not imply reversal of effects already produced.

**Retirement**:
The permanent end of a worker's managed lifetime, distinct from completion or cancellation of a turn.

**Unavailable worker**:
A worker whose runtime has failed or disconnected and cannot accept work. Unavailability is distinct from being idle or retired.

**Assignment**:
The worker's immutable original objective and scope. Later messages and related follow-ups do not replace it.
_Avoid_: Message, follow-up as synonyms for assignment

**Follow-up**:
Related input that starts another turn on a worker's existing assignment. An authorized message to an idle worker is a follow-up, not a new assignment or a requirement for a separate wake operation.

**Peer messaging grant**:
Parent-approved authority for a worker to message specified workers in the same parent group. It includes permission to wake an idle recipient on its existing assignment.

**Session**:
The Pi conversation and its persisted transcript, distinct from the worker and the process executing it.
_Avoid_: Worker as a synonym for session

**Process**:
The running OS executor of a worker's session, distinct from its identity and persisted transcript. A retained transcript does not imply a recoverable managed worker.

**Parent**:
An agent in its role as the owner of a worker. Parent describes an ownership relationship, not a synonym for director.

**Child**:
A worker in relation to its owning parent, not a synonym for its session.

**Director**:
Any top-level agent, whether it owns workers or not, automatically identified without explicit registration. Director is distinct from parent, which describes worker ownership.

**Ownership**:
The fixed relationship between a parent and its worker for the worker's lifetime. Parent loss does not transfer ownership or promise reattachment.

**Owner**:
The parent responsible for a worker, its assignment and its delegated authority. Worker tool-grant and instruction/context-revision approvals remain owner-only in v1.
_Avoid_: Controller as a synonym for owner

**Control**:
Authorization to request particular management actions, distinct from viewing. In v1, a director manages only its own worker group.

**Controller**:
An actor authorized to request particular management actions. V1 has no separate cross-director controller or super-agent role.

**Instruction/context revision**:
A parent-approved version of a worker's standing instructions and selected context. Exactly one revision is active; revisions cannot replace the original assignment.
