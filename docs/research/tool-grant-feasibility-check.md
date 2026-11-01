# Tool grant feasibility and consistency check

Follow-up for [Define tool grants, revocation, and access requests](https://github.com/markdlabrecque/pi-subagents-reloaded/issues/6). Confirmed decisions in `1e4665d` remain unchanged. The ticket is not ready to close because admission timing, provider identity and the root parent's authority need clarification.

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

The requirement that a changed provider receive fresh approval is therefore extension behavior. Neither matching aliases, matching advertised tool names nor successful reconnection proves it. The specification needs a definition of which configured provider properties establish identity before an implementation can verify sameness.

## Consistency findings requiring discussion

1. **Approved but not started calls.** The PRD exempts an "already-authorized running tool" from interruption. Pi separates approval from execution and can defer execution of an approved batch. Does authorization admission make such a call in-flight, or must revocation/disconnection prevent it from starting? Recommendation: require current authority at execution admission; only calls already admitted to execution keep the non-interruption exception. Do not equate an earlier hook return or `tool_execution_start` event with that admission.
2. **Verified provider identity.** What counts as the same MCP provider? Recommendation: bind grants to a parent-approved configured provider identity, including transport and endpoint or executable configuration plus credential/account selection, rather than a server alias or claimed name. Credential rotation for the same approved account need not itself change identity. This does not promise cryptographic attestation of remote software or detection of every server-side change. The exact identity contract still needs confirmation.
3. **Root parent authority.** The PRD prohibits delegation beyond the parent's current authority but does not define that authority for a top-level director. Pi distinguishes registered, active and callable sets and provides no native delegation policy. Recommendation: an explicit parent-local authorization policy defines delegable authority; exposure and discovery do not grant it. Runtime availability remains a separate constraint. Whether the initial policy includes all configured tools or only an explicit selection remains a user decision.

The remaining confirmed requirements are consistent with the glossary and lifecycle section. No timed expiry or renewal is required. Completion and idle time preserve grants; runtime recreation requires fresh grants. Peer authorization is rechecked at delivery. Minimal transcript-linked history is evidence, not replayable authority.

Earlier issue text and map notes still mention duration, expiry and restoration of valid grants. The final confirmed comment and `1e4665d` supersede those statements; they must not be used as implementation requirements. No product decision has been changed by this check.

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
