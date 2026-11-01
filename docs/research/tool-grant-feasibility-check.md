# Tool grant feasibility and consistency check

Follow-up for [Define tool grants, revocation, and access requests](https://github.com/markdlabrecque/pi-subagents-reloaded/issues/6). Confirmed decisions in `1e4665d` remain unchanged. The user confirmed all three final clarifications below after reviewing this check. The permission decision is resolved; enforcement remains extension implementation work, not a verified end-to-end Pi capability.

## Evidence and limits

Reused the completed [Pi 1.4.0 feasibility research](https://github.com/markdlabrecque/pi-subagents-reloaded/blob/be45fb5d27f808123f168c50c114ff916e61c431/docs/research/pi-feasibility.md), including its permission findings and pinned source verification. Rechecked the installed 1.4.0 manifest, extension, SDK, codemode and MCP guides, permission-gate and dynamic-tools examples, and the relevant installed JavaScript, declarations and embedded source maps. Sources below use the same upstream release commit as the completed research.

This is static source verification. No extension was implemented or executed, and no model calls, MCP connections or authorization race experiments were run. It establishes available mechanisms and gaps, not an end-to-end authorization guarantee.

## Verified Pi capabilities versus required extension behavior

### Invocation-time authorization

Pi's `tool_call` handler can block valid direct calls and nested calls through `ctx.executeTool()`. Handler errors block execution. Nested calls receive a parent call ID and pass through the same validation and permission hooks. Unknown tools and invalid arguments fail before reaching that hook. The hook is therefore an execution gate, not a complete catalog or request-outcome mechanism. See [AgentSession][session], `_beforeToolCall` and `_executeNestedToolCall`, and [the agent loop][loop], `prepareToolCall` and `runToolCall`.

Pi does not supply worker ownership, an authenticated live-parent authorization connection, a grant ledger, permission request deduplication, revocation propagation, or fresh-runtime grant approval. Those remain required extension behaviors. An asynchronous hook can consult current policy, but its existence does not prove that policy changes, disconnection and invocation admission are ordered correctly.

There is a consequential timing gap. In parallel mode, `executeToolCallsParallel` prepares and authorizes each call before starting the prepared batch. An approved call can wait while a later call's hook awaits something else. `executePreparedToolCall` then uses the captured tool and arguments without another authorization check. A revocation or disconnect can occur after approval but before execution. Pi's `tool_execution_start` event is emitted before preparation, so it does not prove implementation execution has begun. See [the agent loop][loop].

Handlers also run in registration order on a shared mutable input. A later handler can alter previously approved arguments, or wait for MCP readiness, without a final authorization or argument-validation pass. See [extension runner][runner], `emitToolCall`, and [MCP registration][mcp-index], its `tool_call` handler. Trusted handler ordering and admission of the actual arguments must be addressed by the extension; they are not native guarantees.

### Live grant and catalog changes

Pi can register tools at runtime and change active declarations through `setActiveTools`. Declaration changes reach subsequent model requests, not an already-running response. Initial allow/exclude filters still constrain later registration and activation. Excluded tools cannot become usable merely because the parent approves them. See [extensions][extensions] and [AgentSession][session], `_isAllowedTool`, `_isActivatable` and `_setActiveTools`.

Inactive `codemode` and `deferred` tools remain indirectly callable. Re-registering a tool as `hidden` withdraws reach, but active-list edits alone do not revoke it. Codemode captures its available tool methods and discovery list when a script starts. Each later `tools.*` call uses `ctx.executeTool()` and resolves the current callable registry again. Existing scripts do not automatically gain newly registered methods. Neither a registry change nor a declaration change rechecks an already-prepared call. See [codemode executor][code], [AgentSession][session], `_getCallableTools`, and [extensions][extensions].

The extension must keep discovery distinct from permission, deny newly discovered tools without grants, and make authorization changes effective independently of model-visible declarations. Requestable, parent-forbidden and runtime-unavailable are extension classifications. Standard RPC has no grant/revoke or `set_active_tools` command. A separate worker process needs extension-owned control behavior, not merely startup flags or queued conversational messages. See the completed research and [RPC command types][rpc-types].

### Indirect access and broad capabilities

- Built-in codemode `tools.*` calls use the nested pipeline. Deferred discovery and definition loading do not constitute approval. `reachesTools` supports discovery/readiness, not authorization. MCP tools can remain reachable despite a nonempty built-in-only startup allowlist. Late connections, catalog additions and tool-search activation therefore require the same current-policy checks as direct calls. See [extensions][extensions], [MCP][mcp] and [codemode executor][code].
- Codemode's `models.classify()` and `models.generateImages()` call the model registry outside `ctx.executeTool()`. Their displayed nested rows are not separate `tool_call` authorization events. Model globals are enabled by default and can be omitted with `createCodemodeExtension({ models: false })`. An enclosing codemode approval does not prove per-model-call authorization or revocation. The existing broad-capability decision applies if these operations remain enabled without their own checks. This check does not select a configuration. See [codemode executor][code], `createModelGlobals`, and [codemode registration][code-index].
- MCP's generic `list_mcp_resources`, `list_mcp_resource_templates` and `read_mcp_resource` tools invoke resource connections directly. A list request without a server fans out across the current resource-server set, including providers added later. There is an outer tool gate, not a nested gate for each provider request. Exact outer tool names alone do not enforce provider-scoped authority. Supported provider checks or explicit approval of the actual broad capability must follow the confirmed wrapper rule. See [MCP resource tools][resources], `createMcpResourceToolDefinitions`.
- Arbitrary trusted extensions, replacement codemode/MCP implementations, direct implementation calls, `pi.exec`, SDK shell execution and RPC shell operations need not traverse the model-tool gate. Shell and network effects are not contained. Only audited paths can be claimed to enforce underlying authorization. Owner-only management and worker spawning remain forbidden by the extension contract even when a broad capability is granted. This is a trusted-local rule, not an OS-level prevention guarantee. See the completed research and [extensions][extensions].

### MCP provider identity

Pi names MCP tools using a configured server alias and tool name. Tool execution obtains the current client under that alias. Reconnection or replacement can select a new connection, and reconnect/retry can occur inside the MCP operation without another `tool_call`. These mechanisms do not establish a grant-bound verified provider identity. See [MCP tool definitions][mcp-tools], `createMcpToolDefinition`, [MCP registration][mcp-index], `registerTools`, and [MCP runtime][mcp-runtime], `withClient`.

The requirement that a changed provider receive fresh approval is therefore extension behavior. Neither matching aliases, matching advertised tool names nor successful reconnection proves it. The confirmed configured-provider identity contract below defines which properties establish sameness; Pi does not enforce that contract for the extension.

## Confirmed resolution of consistency findings

The user accepted all three recommendations in live discussion. These clarify the existing permission contract without selecting an implementation.

1. Revocation or disconnection blocks approved-but-not-started calls, including prepared parallel calls. Current authority and actual invocation arguments must be checked at execution admission. Only execution-admitted calls keep the non-interruption exception. An earlier hook return or `tool_execution_start` event is not that admission.
2. MCP grants bind to parent-approved transport, endpoint or executable configuration, and credential/account selection, not a server alias or claimed name. Changed identity requires fresh approval. Credential rotation for the same approved account does not itself change identity. This does not promise remote-software attestation or detection of every server-side change.
3. An explicit parent-local authorization policy defines the root director's delegable authority. Its initial contents are an explicit tool selection, not every configured tool. Registration, activation, exposure, discovery and callability do not grant delegation authority. Runtime availability remains a separate constraint.

The remaining confirmed requirements are consistent with the glossary and lifecycle section. No timed expiry or renewal is required. Completion and idle time preserve grants; runtime recreation requires fresh grants. Peer authorization is rechecked at delivery. Minimal transcript-linked history is evidence, not replayable authority.

Earlier issue text, map notes and comments mentioned duration, expiry and restoration of valid grants. The no-timed-expiry decision and `1e4665d` supersede those statements; they must not be used as implementation requirements. The final resolution preserves those decisions and records the three user-confirmed clarifications above.

## Source references

All upstream links pin Pi 1.4.0 at `4a080252f06caf63a525d7873c5d94d3c23887e0`.

[session]: https://github.com/earendil-works/pi/blob/4a080252f06caf63a525d7873c5d94d3c23887e0/packages/coding-agent/src/core/agent-session.ts
[loop]: https://github.com/earendil-works/pi/blob/4a080252f06caf63a525d7873c5d94d3c23887e0/packages/agent/src/agent-loop.ts
[runner]: https://github.com/earendil-works/pi/blob/4a080252f06caf63a525d7873c5d94d3c23887e0/packages/coding-agent/src/core/extensions/runner.ts
[extensions]: https://github.com/earendil-works/pi/blob/4a080252f06caf63a525d7873c5d94d3c23887e0/packages/coding-agent/docs/extensions.md
[mcp]: https://github.com/earendil-works/pi/blob/4a080252f06caf63a525d7873c5d94d3c23887e0/packages/coding-agent/docs/mcp.md
[code]: https://github.com/earendil-works/pi/blob/4a080252f06caf63a525d7873c5d94d3c23887e0/packages/coding-agent/src/extensions/codemode/execute.ts
[code-index]: https://github.com/earendil-works/pi/blob/4a080252f06caf63a525d7873c5d94d3c23887e0/packages/coding-agent/src/extensions/codemode/index.ts
[mcp-index]: https://github.com/earendil-works/pi/blob/4a080252f06caf63a525d7873c5d94d3c23887e0/packages/coding-agent/src/extensions/mcp/index.ts
[mcp-tools]: https://github.com/earendil-works/pi/blob/4a080252f06caf63a525d7873c5d94d3c23887e0/packages/coding-agent/src/extensions/mcp/tools.ts
[mcp-runtime]: https://github.com/earendil-works/pi/blob/4a080252f06caf63a525d7873c5d94d3c23887e0/packages/coding-agent/src/extensions/mcp/runtime.ts
[resources]: https://github.com/earendil-works/pi/blob/4a080252f06caf63a525d7873c5d94d3c23887e0/packages/coding-agent/src/extensions/mcp/resources.ts
[rpc-types]: https://github.com/earendil-works/pi/blob/4a080252f06caf63a525d7873c5d94d3c23887e0/packages/coding-agent/src/modes/rpc/rpc-types.ts
