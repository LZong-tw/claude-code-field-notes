## Permissions, Auto-Mode & Sandbox

How a tool call is gated in Claude Code v2.1.178: a per-tool decision pipeline produces a `{behavior: "allow"|"deny"|"ask"|"passthrough", decisionReason}` verdict by combining (1) deny/ask/allow rules, (2) the active permission mode, (3) for Bash, a sandbox auto-allow path, and (4) in `auto` mode an LLM classifier. Identifiers below (`xiK`, `uiK`, `LL`, `uS`, `tlH`) are stable minified references for this build.

### Permission modes

Six modes exist. The canonical enum and the default permission context:

```
["default","acceptEdits","bypassPermissions","plan","dontAsk","auto"]).describe("Permission mode for control
```
```
LL=()=>({mode:"default",additionalWorkingDirectories:new Map,alwaysAllowRules:{},alwaysDenyRules:{},alwaysAskRules:{},isBypassPermissionsModeAvailable:!1})
```

The CLI-facing surface exposes `default/acceptEdits/plan/bypassPermissions` (auto/dontAsk are internal/opt-in):

```
"default","acceptEdits","plan","bypassPermissions"
```

Each mode is applied as a `decisionReason:{type:"mode",...}`. `acceptEdits` auto-allows edit/write tool calls; if the mode handler does not match, it returns `passthrough` so later checks run:

```
if(q.mode==="acceptEdits"&&f)return{behavior:"allow",updatedInput:_,decisionReason:{type:"mode",mode:q.mode}};let j=dPH(T,q,"edit");if(j)return{behavior:"allow",updatedInput:_,...
```
```
return{behavior:"passthrough",message:`No mode-specific handling for '${K}' in ${_.mode} mode`}
```

`bypassPermissions` is session-scoped and never persisted as the default, and can be hard-disabled by org policy or settings:

```
setMode:'bypassPermissions' is session-scoped; not persisting as defaultMode to
```
```
for(let X of j){if(X==="bypassPermissions"&&w){if(Y)N("bypassPermissions mode is disabled by feature gate",{level:"warn"}),J="Bypass permissions mode was disabled by your organization policy";else N("bypassPermissions mode is disabled by settings",{level:"warn"}),J="Bypass permissions mode was disabled by settings";continue}
```

Bypass mode availability is tracked on the context (`isBypassPermissionsModeAvailable`) and the CLI accepts `--dangerously-skip-permissions` / `--permission-mode acceptEdits`. Acceptance of the bypass dialog is recorded by the `skipDangerousModePermissionPrompt` setting (read from user/local/flag/policy layers):

```
function BI(){return!!(I6("userSettings")?.skipDangerousModePermissionPrompt||I6("localSettings")?.skipDangerousModePermissionPrompt||I6("flagSettings")?.skipDangerousModePermissionPrompt||I6("policySettings")?.skipDangerousModePermissionPrompt)}
```

**Plan mode** denies actions that would mutate; entering it is session-scoped and `ExitPlanMode` is itself an approval gate:

```
ExitPlanMode inherently requests user approval of yo...
```

### Allow / deny / ask rule matching

Rules live on the context in three buckets keyed by source layer (`alwaysAllowRules`, `alwaysDenyRules`, `alwaysAskRules`), each flattened to `{source, ruleBehavior, ruleValue}`:

```
function sc(H){return VU6.flatMap((_)=>(H.alwaysDenyRules[_]||[]).map((q)=>({source:_,ruleBehavior:"deny",ruleValue:y$(q)})))}function PwH(H){return VU6.flatMap((_)=>(H.alwaysAskRules[_]||[]).map((q)=>({source:_,ruleBehavior:"ask",ruleValue:y$(q)})))}
```

For Bash, all three behaviors are matched (deny/ask stripping env vars and skipping compound-command checks, allow honoring them), and precedence is **deny → ask → allow**:

```
z=sOq(H,T,q,{stripAllEnvVars:!0,skipCompoundCheck:!0,astCommand:O,ruleBehavior:"deny"}),$=vo(_,Z4,"ask"),Y=sOq(...ruleBehavior:"ask"}),A=vo(_,Z4,"allow"),w=sOq(H,A,q,{skipCompoundCheck:K,ruleBehavior:"allow"});return{matchingDenyRules:z,matchingAskRules:Y,matchingAllowRules:w}}
```

The master Bash gate `xiK` runs rule matching first and short-circuits on deny/ask before anything else, then runs the sandbox auto-allow path (`giK`):

```
async function xiK(H,_,q,K,O,T,z=[]){let $=rm6(H,_);if($.behavior==="deny"||$.behavior==="ask")return $;let Y=giK(H,_,K,O,T,z);if(Y.behavior==="deny"||Y.behavior==="ask")return Y;if(Y.behavior==="allow")return Y;...
```

### Auto mode (LLM classifier + auditor)

`auto` mode replaces the static prompt with an AI classifier that decides approve-vs-confirm. Custom rules use four categories — `allow`, `soft_deny`, `hard_deny`, `environment`:

```
Claude Code has an "auto mode" that uses an AI classifier to decide whether tool calls should be auto-approved or require user confirmation. Users can write custom rules in four categories:
- **allow**: Actions the classifier should auto-approve
- **soft_deny**: Destructive/irreversible actions the classifier should block unless clear user intent authorizes them
- **hard_deny**: Security-boundary actions the classifier should block unconditionally (user intent does not clear these)
- **environment**: Context about the user's setup that helps the classifier make decisions
```

There is a two-stage flow: a cheap **fast classifier** (stage "fast") that can short-circuit, then a fuller XML classifier. Denials surface as a mode decision and a user-facing reason:

```
Allowed by fast classifier",model:O,...stage:"fast"
```
```
Blocked by fast classifier",model:O,...stage:"fast"
```
```
Permission for this action was denied by the Claude Code auto mode classifier. Reason:
```

Auto-mode emits telemetry (`tengu_auto_mode_decision`, `tengu_tool_use_granted_by_classifier`, `tengu_classifier_summary_llm_emit`/`_heuristic_emit`) and is opt-in via a dialog recorded by `skipAutoPermissionPrompt`; accepting-as-default writes `permissions.defaultMode:"auto"`:

```
fq("userSettings",{skipAutoPermissionPrompt:!0,permissions:{defaultMode:"auto"}})
```

A **critique/auditor** path reviews the user's custom rules ("You are an expert reviewer of auto mode classifier rules…") via `autoModeCritiqueHandler` / `autoModeDefaultsHandler` / `autoModeConfigHandler`. Auto mode has a circuit-breaker that disables itself on repeated classifier failures ("auto mode disabled: …", `disableAutoMode in settings`), and the classifier transcript has a size cap (`classifierTranscriptTooLong`, aborting with "auto mode classifier transcript exceeded").

### Bash command sandbox (macOS seatbelt / Linux bubblewrap)

When sandboxing is enabled and the command qualifies, Bash runs under macOS `sandbox-exec` with a generated SBPL profile (default-deny):

```
Bh7.default.quote(["env",...R,"/usr/bin/sandbox-exec","-p",G,y,"-c",_]);return nq(`[Sandbox macOS] Applied restrictions - network: ${...}, read: ${w?"allowAllExcept"in w?"allowAllExcept":"denyAllExcept":"none"}, write: ${f?"allowAllExcept"in f?...
```

The profile begins `(version 1)` + `(deny default (with message "…"))` and grants Chrome-sandbox-style base permissions, then layers filesystem read/write policy (each as `allowAllExcept` / `denyAllExcept` / `none`) and a network policy:

```
D=["(version 1)",`(deny default (with message "${J}"))`,"",`; LogTag: ${J}`,..."(allow process-exec)","(allow pr...
```
Filesystem rules emitted include `(allow file-read*)`, `(deny file-read* …)`, `(allow file-write* …)`, `(deny file-write* …)`, `file-write-unlink`, `file-write-create`. Network is default-blocked; Unix sockets and a localhost proxy port can be selectively allowed:

```
D.push("(allow system-socket (socket-domain AF_UNIX))"),D.push('(allow network-bind (local unix-socket (path-regex #"^/")))'),...
```
```
D.push(`(allow network-bind (local ip "localhost:${q}"))`),D.push(`(allow network-inbound (local ip "localhost:${q}"))`),D.push(`(allow network-outbound (remote ip "localhost:...
```

Sandbox config shape (from the settings schema) includes `allowUnixSockets`, `allowAllUnixSockets`, `allowLocalBinding`, `allowedHosts`, `deniedHosts`, `allowedDomains`, `allowManagedReadPathsOnly`, `enableWeakerNestedSandbox`. Linux uses bubblewrap (`bwrap`, `CLAUDE_CODE_BUBBLEWRAP`, "[Sandbox Linux]…"). WSL1 is unsupported ("sandbox is enabled but WSL1 is not supported (requires WSL2)").

### Sandbox auto-allow & `dangerouslyDisableSandbox`

If sandboxing is enabled and `autoAllowBashIfSandboxed` is on, a *sandboxable* Bash command is auto-approved without a prompt (rather than asking), via `uiK`:

```
function uiK(H,_,q,K){if(!dq.isSandboxingEnabled()||!dq.isAutoAllowBashIfSandboxedEnabled()||!iV(H))return null;...
```
```
tlH="Auto-allowed with sandbox (autoAllowBashIfSandboxed enabled)"
tJ_="Read-only command is allowed"
```
`uiK` refuses to auto-allow if the command writes unknown env vars or redirects to `/dev/tcp|/dev/udp` (so it cannot smuggle network egress past the sandbox):

```
$=q.some((w)=>w.redirects.some((f)=>/^\/dev\/(tcp|udp)\//.test(f.target)));if(z||$)return null;
```

The `dangerouslyDisableSandbox` Bash tool param opts a single command out of the sandbox (its prompt text says retry with it on sandbox-caused failure). It can be hard-disabled by policy:

```
All commands MUST run in sandbox mode - the `dangerouslyDisableSandbox` parameter is disabled by policy.
```

Sandbox-state detection (`zn5`) treats the process as sandboxed when `CLAUDE_CODE_SANDBOXED` is truthy:

```
function zn5(){if(z_(process.env.CLAUDE_CODE_SANDBOXED))return!0;if(iQH())return!0;if(k7())return!0;...
```

`sandbox.failIfUnavailable` (managed-settings) makes startup error out if sandboxing can't start instead of silently running unsandboxed: "Exit with an error at startup if sandbox.enabled is true but the sandbox cannot start … Intended for managed-settings deployments that require sandboxing as a hard gate."

### Sandbox network classifier (auto mode + network)

When auto mode is active and a sandboxed command needs network, an "iron_gate" classifier decides allow/deny; if the classifier is unavailable the gate closes (deny) by default:

```
Sandbox network classifier unavailable for ${H}; iron_gate → ${A?"allow":"deny"}`,{level:"warn"});if(!A)N(`Auto mode classifier blocked sandbox network access to ${H}: ...
```

### Operator notes
- **Mode precedence is mode→rules per tool, but rules win at the boundary**: for Bash the order is deny rules → ask rules → allow rules → sandbox auto-allow → classifier (`xiK`). A `deny` rule cannot be overridden by `acceptEdits`/`auto`.
- **`bypassPermissions` is killable**: org `policySettings` or `settings` disable it (`bypassPermissions mode is disabled by feature gate/settings`); it never becomes `defaultMode`.
- **`auto` mode is a real LLM call per gated action** (fast stage + XML stage). It adds latency and tokens, has a transcript size cap, and a circuit-breaker that self-disables on classifier failures. Through a gateway/CCR these classifier requests hit the model endpoint like normal turns (separate `classifierModel`/`classifierStage` telemetry).
- **`hard_deny` rules are unconditional**; `soft_deny` can be cleared by user intent; design custom auto-mode rule sets accordingly.
- **Sandbox auto-allow lets Bash run without prompts** when `autoAllowBashIfSandboxed` is enabled — but only for commands that pass static analysis (no unknown env writes, no `/dev/tcp` redirects). To force prompts back, disable the setting.
- **Hard-require sandbox** in managed deployments with `sandbox.failIfUnavailable:true` and disable `dangerouslyDisableSandbox` by policy ("disabled by policy"). WSL must be WSL2.
- **Network is default-deny inside the sandbox**; egress is mediated via a localhost proxy port and `allowedHosts`/`deniedHosts`/`allowedDomains`. The `CLAUDE_CODE_*PROXY*` env vars route that egress. In auto mode the iron_gate classifier additionally gates network and fails closed.
- **Env flags**: `CLAUDE_CODE_SANDBOXED`, `CLAUDE_CODE_FORCE_SANDBOX=1`, `CLAUDE_CODE_BUBBLEWRAP` (Linux), `CLAUDE_CODE_BASH_SANDBOX_SHOW_INDICATOR` (UI), `IS_SANDBOX`/`SANDBOX_RUNTIME` markers.
- **`skipDangerousModePermissionPrompt` / `skipAutoPermissionPrompt`** record that the user accepted the bypass/auto opt-in dialogs (read across user/local/flag/policy layers); setting these pre-accepts the dialogs for non-interactive runs.
