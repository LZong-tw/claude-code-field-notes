## Statusline

Claude Code can render a custom status line by running an operator-supplied shell command and printing its first non-blank stdout lines below the prompt. The command receives a single JSON object on **stdin** describing the current session; whatever it writes to **stdout** becomes the status line text. This is the contract `ccstatusline` and similar tools consume.

### Configuration shape and resolution

The statusline config lives under `statusLine` in settings. It is parsed by `fUH`, which gives **managed policy precedence**: if a managed `policySettings` exists, its `statusLine` wins over the user's:

```js
function fUH(H){return nD()?I6("policySettings")?.statusLine:H}
```

The executor only runs `type:"command"` configs; anything else is a no-op:

```js
let O=fUH(rq()?.statusLine);if(!O||O.type!=="command")return;
```

The config object's recognized fields (observed in the scheduler hook `qJT`): `command` (string), `type` (`"command"`), `refreshInterval` (seconds), `padding`, and the hook-common `shell` / `args` (handled by the shared executor `nn6`).

### When it runs (scheduling / throttle)

Execution is driven by the React hook `qJT`. The core trigger is a **300ms-debounced** callback `b`:

```js
B=vp(()=>{b()},300);
```

It re-fires whenever any of these change (effect deps): last-assistant message id, token usage, permission mode, vim mode, main-loop model, fast mode, effort value, thinking-enabled, PR status:

```js
DD.useEffect(()=>{if(_!==C.current.messageId||q!==C.current.tokenUsage||T!==C.current.permissionMode||K!==C.current.vimMode||f!==C.current.mainLoopModel||j!==C.current.fastMode||J!==C.current.effortValue||D!==C.current.thinkingEnabled||M!==C.current.prStatus)...B()},[_,q,T,K,f,j,J,D,M,B]);
```

It also re-runs immediately when the configured `command` string changes, and on an **optional fixed timer** from `refreshInterval` (seconds → ms, floored to 1s):

```js
let U=w?.refreshInterval;l1(B,U!==void 0?Math.max(1,U)*1000:null);
```

So: by default it runs reactively (debounced 300ms) on state changes; set `refreshInterval` to also poll on a wall-clock cadence. Each new run aborts the previous in-flight command (`O.current?.abort()`), so a slow command can be cut off by the next trigger.

### Timeout, trust gates, and output handling

The executor is `executeStatusLineCommand` (`r0q`), default timeout **5000ms**:

```js
function r0q(H,_,q=5000,K=!1){if(wx())return;if(k1("statusLine"))return;if(JfH()){N("Skipping StatusLine command execution - workspace trust not accepted");return}...let T=_||AbortSignal.timeout(q);...let z=xH(H),$=await nn6(O,"StatusLine","statusLine",z,zxH(H),T,...);
```

Gates, in order: `wx()` (a global disable), `k1("statusLine")` (the `disableAllHooks` / per-hook disable — string `Disable all hooks and statusLine execution`), and **workspace-trust** (`JfH()` → logs `Skipping StatusLine command execution - workspace trust not accepted`). The JSON payload is `xH(H)` (`JSON.stringify`) handed to the shared hook runner `nn6`, which spawns the command (`cn6.spawn(...,{env,cwd,...})`) and feeds the JSON on stdin.

Stdout is trimmed, split on newlines, blank lines dropped, then rejoined — so **multi-line status lines are supported**:

```js
if($.status===0){let A=$.stdout.trim().split(`\n`).flatMap((w)=>w.trim()||[]).join(`\n`);if(A){...return A}}
```

stderr is captured and logged (`StatusLine [<command>] stderr: ...`) but does not block rendering. A non-zero exit means no update (the prior status line stays). There is also a setup/mount telemetry event (`tengu_status_line_mount`) and a per-result event (`tengu_status_line_result`).

### Payload shape (what the command reads on stdin)

The payload is assembled by `_JT`, merged onto the common hook base `h3()`. Verbatim builder:

```js
return{...h3(),cwd:f,...R&&{session_name:R},model:{id:X,display_name:Dz(X)},workspace:{current_dir:f,project_dir:W8(),added_dirs:T,...$&&{git_worktree:$},...Y&&{repo:Y}},version:{...VERSION:"2.1.178"...}.VERSION,output_style:{name:P},cost:{total_cost_usd:gX(),total_duration_ms:J8H(),total_api_duration_ms:jW(),total_lines_added:H3H(),total_lines_removed:_3H()},context_window:HJT(Z,W),exceeds_200k_tokens:_,fast_mode:q,...WP(X)&&{effort:{level:UN(X,j)}},thinking:{enabled:J!==!1},...(y.five_hour||y.seven_day)&&{rate_limits:y},...A6H()&&{vim:{mode:w??"INSERT"}},...D&&{agent:{name:D}},...A3()!==null&&{remote:{session_id:C_()}},...A&&{pr:{number:A.number,url:A.url,...}},...M&&{worktree:{name:...,path:...,branch:...,original_cwd:...,original_branch:...}}}
```

The base (`h3`) contributes the session/identity fields:

```js
return{session_id:K,transcript_path:Ky(K),cwd:u_(),permission_mode:H,agent_id:...,agent_type:O,effort:$}
```

**Always-present fields:**
- `session_id` (string), `transcript_path`, `cwd` (the base `cwd` plus a second `cwd:f` override), `permission_mode`, `agent_id`, `agent_type`
- `model` = `{ id, display_name }`
- `workspace` = `{ current_dir, project_dir, added_dirs[] }` (+ optional `git_worktree`, `repo`)
- `version` (just the version string, e.g. `"2.1.178"`)
- `output_style` = `{ name }`
- `cost` = `{ total_cost_usd, total_duration_ms, total_api_duration_ms, total_lines_added, total_lines_removed }`
- `context_window` (see below)
- `exceeds_200k_tokens` (bool), `fast_mode` (bool), `thinking` = `{ enabled }`

**Conditional fields** (present only when relevant): `session_name`, `effort` = `{ level }` (only for models that support effort), `rate_limits` = `{ five_hour?, seven_day? }` (each `{ used_percentage, resets_at }`), `vim` = `{ mode }` (vim mode only), `agent` = `{ name }`, `remote` = `{ session_id }`, `pr` = `{ number, url, review_state?, kind? }`, `worktree` = `{ name, path, branch, original_cwd, original_branch }`.

`workspace.repo` (when in a git repo with a remote) is `{ host, owner, name }`, parsed from the remote URL by `HqH`:

```js
function HqH(H){...return{host:q[1],owner:q[2],name:q[3]}...}
```

Note: there is no top-level `git`/`gitBranch` field in this build's statusline payload — branch/PR/worktree info comes through `workspace.git_worktree`, `worktree.branch`, and `pr`. (`gitBranch` exists elsewhere in session metadata, not in this payload.)

### context_window object

`context_window` is built by `HJT(Z,W)` where `Z` is current `usage` and `W` is the resolved window size:

```js
function HJT(H,_){let q=xY6(H,_);return{total_input_tokens:H?H.input_tokens+H.cache_creation_input_tokens+H.cache_read_input_tokens:0,total_output_tokens:H?.output_tokens??0,context_window_size:_,current_usage:H,used_percentage:q.used,remaining_percentage:q.remaining}}
```

So the object is:
- `total_input_tokens` = input + cache-creation + cache-read input tokens
- `total_output_tokens`
- `context_window_size` (the cap)
- `current_usage` (the raw usage object: `input_tokens`, `cache_creation_input_tokens`, `cache_read_input_tokens`, `output_tokens`)
- `used_percentage` / `remaining_percentage` (integers 0–100)

Percentages from `xY6`:

```js
function xY6(H,_){if(!H)return{used:null,remaining:null};let q=H.input_tokens+H.cache_creation_input_tokens+H.cache_read_input_tokens,K=Math.round(q/_*100),O=Math.min(100,Math.max(0,K));return{used:O,remaining:100-O}}
```

`context_window_size` comes from `QM(model, headers)`, which an operator can override:

```js
function QM(H,_){let q=A37();if(q!==void 0)return q;...return w37(H,_)}
function A37(){if(nH.DISABLE_COMPACT&&process.env.CLAUDE_CODE_MAX_CONTEXT_TOKENS){let H=parseInt(process.env.CLAUDE_CODE_MAX_CONTEXT_TOKENS,10);if(!isNaN(H)&&H>0)return H}return}
```

The 1M-context (`[1m]`) path forces `1e6` (`w37`: `if(Jf(H))return 1e6`), so a 1M-context model reports `context_window_size: 1000000` and the statusline percentages are computed against 1M.

### Subagent statusline

There is a separate `subagentStatusLine` config/path (anchors `subagentStatusLine exited`, `subagentStatusLine emitted non-JSON line:`, `subagentStatusLine emitted invalid schema:`). Unlike the main statusline, the subagent variant **expects JSON output** validated against a schema — it is not a free-text status line. It is also workspace-trust gated (`Skipping subagentStatusLine execution - workspace trust not accepted`).

### Operator notes

- **Payload is on stdin as one JSON object**; emit your status line on stdout. Only non-blank lines are kept; multi-line output is allowed (joined with `\n`).
- **Default timeout 5000ms.** A slow command silently yields no update and the previous line persists; it can also be aborted by the next trigger.
- **Cadence:** reactive, 300ms-debounced on session-state changes by default. Add `"refreshInterval": <seconds>` to your `statusLine` config for an additional wall-clock poll (min 1s). Changing the `command` string triggers an immediate re-run.
- **Disable knobs:** `disableAllHooks` (also disables the status line — emits `Status line is configured but disableAllHooks is true`); workspace trust must be accepted or it is skipped.
- **Managed policy wins:** a `policySettings.statusLine` overrides the user's config entirely.
- **`context_window.context_window_size`** can be forced with `CLAUDE_CODE_MAX_CONTEXT_TOKENS` (only when `DISABLE_COMPACT` is set); a 1M-context model reports `1000000` and percentages are relative to that. Gateways that lie about the window won't change this number unless you set the env var.
- **Field availability is conditional** — guard for missing `effort`, `rate_limits`, `vim`, `pr`, `worktree`, `agent`, `remote`, `session_name`, and `workspace.repo`/`workspace.git_worktree` in your consumer.
- **Cost fields are cumulative-session** (`total_cost_usd`, durations, lines added/removed), not per-turn.
- **stderr is logged, not shown** (`StatusLine [<cmd>] stderr: ...`); non-zero exit = no update.
