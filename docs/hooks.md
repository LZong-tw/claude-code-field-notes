## Hooks

_Verified against Claude Code 2.1.178._

Claude Code runs user/plugin-defined commands at fixed lifecycle points. Configuration lives under a top-level `hooks` key in settings; each event maps to an array of `{ matcher, hooks: [{ type, command, timeout, statusMessage }] }` groups. Ground truth here is the bundled Zod input/output schemas (`k.object(...)` / `k.literal(...)`) plus the embedded `## Hooks Configuration` documentation block — both are shipped verbatim in the bundle.

### Config structure & hook types

The documented shape (from the in-bundle docs block, recovered via `strings.txt`):

```
"hooks": { "EVENT_NAME": [ { "matcher": "ToolName|OtherTool",
  "hooks": [ { "type": "command", "command": "...", "timeout": 60, "statusMessage": "Running..." } ] } ] }
```

There are four hook `type`s. Three are documented (`command`, `prompt`, `agent`); a fourth (`http`) and an internal `mcp_tool` form exist in code:

```
**1. Command Hook** - Runs a shell command:
{ "type": "command", "command": "prettier --write $FILE", "timeout": 30 }
**2. Prompt Hook** - Evaluates a condition with LLM:
{ "type": "prompt", "prompt": "Is this safe? $ARGUMENTS" }
**3. Agent Hook** - Runs an agent with tools:
{ "type": "agent", "prompt": "Verify tests pass: $ARGUMENTS" }
Only available for tool events: PreToolUse, PostToolUse, PermissionRequest.
```

```
type:"http",command:_.url   // ...else if(_.type==="mcp_tool")return{type:"mcp_tool",command:`${_.server}/${_.tool}`}
```

`prompt`/`agent` hooks are restricted to the tool events (`PreToolUse`, `PostToolUse`, `PermissionRequest`).

### Events and when they fire

Documented event table (verbatim):

```
| PermissionRequest | Tool name | Run before permission prompt |
| PreToolUse | Tool name | Run before tool, can block |
| PostToolUse | Tool name | Run after successful tool |
| PostToolUseFailure | Tool name | Run after tool fails |
| Notification | Notification type | Run on notifications |
| Stop | - | Run when Claude stops (including clear, resume, compact) |
| PreCompact | "manual"/"auto" | Before compaction |
| PostCompact | "manual"/"auto" | After compaction (receives summary) |
| UserPromptSubmit | - | When user submits |
| SessionStart | - | When session starts |
```

The schema layer registers a broader set than the docs expose, including `PostToolBatch`, `PermissionDenied`, `SubagentStart`/`SubagentStop`, `StopFailure`, `Setup`, `UserPromptExpansion`, `TeammateIdle`, `TaskCreated`/`TaskCompleted`, `Elicitation`/`ElicitationResult`:

```
hook_event_name:k.literal("PostToolBatch"),tool_calls:k.array(...)  // "Fired once after every tool call in a batch has resolved, before the next model request. PostToolUse fires per-tool and may run concurrently for parallel tool calls; PostToolBatch fires exactly once with the full batch."
hook_event_name:k.literal("PermissionDenied"),tool_name,tool_input,tool_use_id,reason
```

A `Stop` hook running inside a subagent is rewritten to `SubagentStop`:

```
Converting Stop hook to SubagentStop for  (subagents trigger SubagentStop)
```

### stdin contract (input JSON)

Every event extends a common base object (`Aj()`) carrying session context, then adds event-specific fields:

```
session_id:k.string(),transcript_path:k.string(),cwd:k.string(),permission_mode:k.string().optional(),
agent_id:k.string().optional()...  // "Present only when the hook fires from within a subagent... Use this field (not agent_type) to distinguish subagent calls from main-thread calls."
agent_type:k.string().optional()...
effort:k.object({level:k.string()...}).optional()  // "...exposed to hook commands and Bash as the CLAUDE_EFFORT env var."
```

Per-event additions (selected, verbatim):

```
PreToolUse:  hook_event_name:k.literal("PreToolUse"),tool_name,tool_input,tool_use_id
PostToolUse: hook_event_name:k.literal("PostToolUse"),tool_name,tool_input,tool_response,tool_use_id,duration_ms  // "Excludes permission-prompt and hook time."
PostToolUseFailure: ...tool_name,tool_input,tool_use_id,error:k.string(),is_interrupt,duration_ms
UserPromptSubmit: ...prompt:k.string(),session_title?
SessionStart: ...source:k.enum(["startup","resume","clear","compact"]),agent_type?,model?,session_title?
Stop: ...stop_hook_active:k.boolean(),last_assistant_message?,background_tasks?,session_crons?
SubagentStop: ...stop_hook_active,agent_id,agent_transcript_path,agent_type,last_assistant_message?,background_tasks?,session_crons?
PreCompact:  ...trigger:k.enum(["manual","auto"]),custom_instructions:k.string().nullable()
PostCompact: ...trigger:k.enum(["manual","auto"]),compact_summary:k.string()  // "The conversation summary produced by compaction"
Notification: ...message:k.string(),title?,notification_type:k.string()
```

The docs show the minimal stdin example with the `// PostToolUse only` annotation on `tool_response`:

```
"session_id": "abc123", "tool_name": "Write",
"tool_input": { "file_path": "...", "content": "..." },
"tool_response": { "success": true }  // PostToolUse only
```

`background_tasks` and `session_crons` on `Stop`/`SubagentStop` let a hook tell "session done" from "paused waiting on background work" (`describe(...)` strings confirm this).

### stdout contract (output JSON)

The hook may print JSON to stdout. Top-level fields:

```
continue:k.boolean().optional(),suppressOutput:k.boolean().optional(),stopReason:k.string().optional(),
decision:k.enum(["approve","block"]).optional(),systemMessage:k.string().optional(),
terminalSequence:k.string().optional()...  // "Only notification/title OSCs (0,1,2,9,99,777) and BEL are permitted; anything else is dropped."
reason:k.string().optional(),hookSpecificOutput:k.union([...])
```

Documented field semantics (verbatim):

```
- `continue` - Set to `false` to block/stop (default: true)
- `stopReason` - Message shown when `continue` is false
- `suppressOutput` - Hide stdout from transcript (default: false)
- `decision` - "block" for PostToolUse/Stop/UserPromptSubmit hooks (deprecated for PreToolUse, use hookSpecificOutput.permissionDecision instead)
- `hookSpecificOutput` ... must include `hookEventName`
```

`hookSpecificOutput` is a discriminated union keyed by `hookEventName`. Key per-event outputs:

```
PreToolUse:  permissionDecision:k.enum(["allow","deny","ask","defer"]).optional(),permissionDecisionReason?,updatedInput?,additionalContext?
UserPromptSubmit: additionalContext?,sessionTitle?,suppressOriginalPrompt?  // 'When decision is "block", omit the original prompt from the block message'
SessionStart: additionalContext?,initialUserMessage?,sessionTitle?,watchPaths?,reloadSkills?  // "Re-scan skill and command directories after SessionStart hooks complete"
PostToolUse: additionalContext?,updatedToolOutput?  // "Replaces the tool output before it is sent to the model"; updatedMCPToolOutput? (MCP only)
Stop: additionalContext?  // "non-error feedback delivered to the model; the conversation continues so the model can act on it."
```

`permissionDecision` carries a fourth value `defer` beyond the documented allow/deny/ask. It is print-mode-only and solo-only — ignored interactively or when more than one tool call is in the batch:

```
Hook ... returned permissionDecision=defer in interactive mode; ignoring (defer is print-mode only)
Hook ... returned permissionDecision=defer but N tool calls are in this batch; ignoring (defer is solo-only — siblings would be orphaned on resume)
```

### Exit-code 2 (blocking convention)

Besides JSON output, a non-zero exit feeds stderr back as feedback; exit code `2` is the blocking signal. Confirmed for the `Stop` path:

```
if(J.code===2){let P=`Stop hook blocking error from command "${T}":`,Z="Stop hook feedback";...}
```

PostToolUse blocking surfaces as `Execution stopped by PostToolUse hook` and a `tengu_post_tool_hook_error` event.

### Runner / execution model

Hook commands are spawned with a per-hook timeout and a controlled environment. Default timeout is `WO=600000` ms (600 s); per-hook `timeout` is in **seconds** and multiplied by 1000:

```
S=H.timeout?H.timeout*1000:WO   // WO=600000
I={...qV(),...I8_(O),CLAUDE_PROJECT_DIR:G(R)}; if(u)I.COLUMNS=String(u); if(b)I.LINES=String(b);
if(Y){I.CLAUDE_PLUGIN_ROOT=G(Y); if(A)I.CLAUDE_PLUGIN_DATA=G(d7H(A))}
... I[`CLAUDE_PLUGIN_OPTION_${PH}`]=String(MH) ...
if(!Z&&(_==="SessionStart"||_==="Setup"||_==="CwdChanged"||_==="FileChanged")&&$!==void 0)I.CLAUDE_ENV_FILE=await CE7(_,$)
```

The child-session env builder injects the session/effort/trace vars:

```
function I8_(H){let _={CLAUDECODE:"1",CLAUDE_CODE_SESSION_ID:H.sessionId,CLAUDE_CODE_CHILD_SESSION:"1"};
if(H.source==="agent")_.AI_AGENT=Qt6("agent");
if(H.effortLevel!==void 0)_.CLAUDE_EFFORT=H.effortLevel;
if(wX_()){let q=a06();if(q!==void 0)_.TRACEPARENT=q}return _}
```

cwd is resolved with a fallback if the session cwd no longer exists:

```
Hooks: cwd ${B} not found, falling back to original cwd
```

Timeouts abort per-hook (not the whole batch); each event has its own message, e.g. `PostToolUse hook timed out (per-hook abort)`, `PostToolUseFailure hook cancelled (parent abort)`. Internal executors are named per event (`executeUserPromptSubmitHooks`, `executeSessionStartHooks`, `executePostToolUseFailureHooks`, `executePostCompactHooks`).

`$ARGUMENTS` / `$FILE` substitution and `${CLAUDE_PROJECT_DIR}` / `${CLAUDE_EFFORT}` expansion are supported in command strings (both tokens appear in shipped command templates).

### Policy gates

Hooks can be globally disabled or restricted to managed (admin-policy) hooks via settings:

```
n8T(){return I6("policySettings")?.allowManagedHooksOnly===!0}
i8T(){return rq()?.disableAllHooks===!0&&I6("policySettings")?.disableAllHooks===!0}
... // "are restricted (disableAllHooks or allowManagedHooksOnly is set in settings or by policy)."
```

Plugin SessionStart hooks fail soft:

```
Warning: Failed to load plugin hooks. SessionStart hooks from plugins will not execute. Error: 
```

HTTP hooks are gated by an allowlist; outbound URLs and forwarded env are restricted:

```
... does not match any pattern in allowedHttpHookUrls   // settings keys: allowedHttpHookUrls, httpHookAllowedEnvVars
```

### Operator notes

- **Timeout units differ from the field name:** config `timeout` is in **seconds** (`*1000` internally); the global default is **600 s** (`WO=600000`). A hook that hangs blocks that event for up to 10 minutes unless you set `timeout`.
- **Blocking has two channels:** exit code `2` (stderr → fed back as feedback) and JSON (`{"continue":false}` or, on tool events, `hookSpecificOutput.permissionDecision:"deny"/"ask"`). For `PreToolUse`, prefer `permissionDecision`; top-level `decision:"block"` is deprecated there.
- **`permissionDecision` has a 4th value `defer`** — only honored in print/non-interactive mode and only when exactly one tool call is in the batch; silently ignored otherwise.
- **PostToolUse can mutate tool output** via `updatedToolOutput` (all tools) / `updatedMCPToolOutput` (MCP only), and `PreToolUse` can rewrite tool input via `updatedInput` before execution.
- **Env available to hook commands:** `CLAUDE_PROJECT_DIR`, `CLAUDE_EFFORT`, `CLAUDE_CODE_SESSION_ID`, `CLAUDECODE=1`, `CLAUDE_CODE_CHILD_SESSION=1`, plus `CLAUDE_PLUGIN_ROOT/_DATA/_OPTION_*` for plugin hooks and `CLAUDE_ENV_FILE` (SessionStart/Setup/CwdChanged/FileChanged only). `CLAUDE_CODE_SHELL_PREFIX` is read to wrap the shell command.
- **Subagents:** a configured `Stop` hook fires as `SubagentStop` inside a subagent; `agent_id` (not `agent_type`) is the reliable "this came from a subagent" signal. Use `last_assistant_message` instead of parsing the transcript.
- **`Stop`/`SubagentStop` get `background_tasks` and `session_crons`** so a stop hook can distinguish a finished session from one parked waiting on background work or a scheduled wakeup.
- **Policy kill-switches:** `disableAllHooks` (settings + policy must both set it) and `allowManagedHooksOnly` (admin-policy hooks only). HTTP hooks need `allowedHttpHookUrls` and env passthrough is filtered by `httpHookAllowedEnvVars`.
- **`reloadSkills:true`** on SessionStart output re-scans skill/command dirs so a hook that installs skills makes them usable in the same session.
- **`terminalSequence`** lets a hook emit a desktop-notification OSC, but only OSC 0/1/2/9/99/777 and BEL pass the filter.
