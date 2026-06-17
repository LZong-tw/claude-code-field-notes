## Environment Variable Registry

Claude Code v2.1.178 reads almost all of its tunable knobs through a set of **lazy getter registry objects**. Each registry maps an env-var NAME to a typed parser, and exposes it as a property whose getter calls `O.parse(process.env[K])` on access. This means env vars are read **live** (every property access re-reads `process.env`), not snapshotted at startup.

### The registry wrapper (how env vars are read)

Found verbatim at the tail of the registry module (bundle offset ~139.59M):

```js
function plq(H,_){let q=Object.create(_);for(let[K,O]of Object.entries(H))
  Object.defineProperty(q,K,{get:()=>O.parse(process.env[K]),enumerable:!0,configurable:!0});
  return Object.defineProperties(q,{set:{value:(K,O)=>{process.env[K]=wgq(O)}}, ...
```

- Each registry object (`L38`, `h38`, `k38`, `R38`, plus a terminal/CI-detection object) is built with `j_(OBJ,{NAME:()=>VAR,...})`, then each `VAR` is assigned a typed parser via `pH.str()/.bool()/.int()/.triBool()/.enum()`.
- A property's getter runs `parser.parse(process.env[NAME])` — so **changing `process.env` mid-process changes the value**, and the `set` trap writes back to `process.env`.

### Parser/type semantics (str / bool / int / triBool)

```js
pH={str:()=>Th1(),bool:()=>zh1(),triBool:()=>$h1(),int:(H)=>H?fgq(H):Yh1(),
   enum:(H)=>N6.preprocess(Ej_,N6.string().optional().transform((_)=>_!==void 0&&H.includes(_.trim())?_.trim():void 0))
```

Boolean truthiness (shared by `bool` and `triBool`):

```js
function z_(H){if(!H)return!1;if(typeof H==="boolean")return H;
  let _=String(H).toLowerCase().trim();return["1","true","yes","on"].includes(_)
```

**Operator-critical:** a boolean env var is "on" only for `1`, `true`, `yes`, `on` (case-insensitive). Any other value (including `0`, `false`, `no`, empty) is treated as **false/unset**. `triBool` ($h1) returns `true` for those truthy strings and otherwise distinguishes explicit-false vs unset (three-state) — used where "unset" must differ from "explicitly disabled".

### Registry object 1 — models & fast-mode (`L38`)

Verbatim slice:

```js
var L38={};j_(L38,{FALLBACK_FOR_ALL_PRIMARY_MODELS:()=>zB1,CLAUDE_CONTEXT_COLLAPSE_MODEL:()=>OB1,CLAUDE_CONTEXT_COLLAPSE:()=>TB1,CLAUDE_CODE_SUBAGENT_MODEL:()=>_B1,CLAUDE_CODE_SKIP_FAST_MODE_ORG_CHECK:()=>DB1,CLAUDE_CODE_SKIP_FAST_MODE_NETWORK_ERRORS:()=>JB1,CLAUDE_CODE_OPUS_4_6_FAST_MODE_OVERRIDE:()=>jB1,CLAUDE_CODE_ENABLE_OPUS_4_7_FAST_MODE:()=>fB1,CLAUDE_CODE_EFFORT_LEVEL:()=>YB1,CLAUDE_CODE_DISABLE_LEGACY_MODEL_REMAP:()=>$B1,CLAUDE_CODE_DISABLE_FAST_MODE:()=>wB1,CLAUDE_CODE_DISABLE_1M_CONTEXT:()=>MB1,CLAUDE_CODE_BG_CLASSIFIER_MODEL:()=>KB1,CLAUDE_CODE_AUTO_MODE_MODEL:()=>qB1,CLAUDE_CODE_ALWAYS_ENABLE_EFFORT:()=>AB1,ANTHROPIC_SMALL_FAST_MODEL:()=>Up1,ANTHROPIC_MODEL:()=>Bp1, ... ANTHROPIC_DEFAULT_SONNET_MODEL/_OPUS_MODEL/_HAIKU_MODEL/_FABLE_MODEL(+_NAME/_DESCRIPTION),ANTHROPIC_CUSTOM_MODEL_OPTION(+_NAME/_DESCRIPTION)})
```

Declared types (from `xlq`): all `str` except `CLAUDE_CONTEXT_COLLAPSE`, `CLAUDE_CODE_DISABLE_LEGACY_MODEL_REMAP`, `CLAUDE_CODE_ALWAYS_ENABLE_EFFORT`, `CLAUDE_CODE_DISABLE_FAST_MODE`, `CLAUDE_CODE_ENABLE_OPUS_4_7_FAST_MODE`, `CLAUDE_CODE_SKIP_FAST_MODE_NETWORK_ERRORS`, `CLAUDE_CODE_SKIP_FAST_MODE_ORG_CHECK`, `CLAUDE_CODE_DISABLE_1M_CONTEXT` which are `bool`.

| Var | Effect (inferred from name+type) |
|---|---|
| `ANTHROPIC_MODEL` | primary model id override (str) |
| `ANTHROPIC_SMALL_FAST_MODEL` | background/haiku-class model override (str) |
| `ANTHROPIC_DEFAULT_{OPUS,SONNET,HAIKU,FABLE}_MODEL` | per-tier model id + display name/description overrides |
| `CLAUDE_CODE_SUBAGENT_MODEL` | model for sub-agents (str) |
| `CLAUDE_CODE_BG_CLASSIFIER_MODEL` | model for the background classifier (str) |
| `CLAUDE_CODE_AUTO_MODE_MODEL` | model used in auto mode (str) |
| `CLAUDE_CODE_EFFORT_LEVEL` | reasoning effort selector (str) |
| `CLAUDE_CODE_DISABLE_FAST_MODE` / `CLAUDE_CODE_ALWAYS_ENABLE_EFFORT` | force-off / force-on fast/effort modes (bool) |
| `CLAUDE_CODE_ENABLE_OPUS_4_7_FAST_MODE` / `CLAUDE_CODE_OPUS_4_6_FAST_MODE_OVERRIDE` | opt into Opus fast-mode variants |
| `CLAUDE_CODE_DISABLE_1M_CONTEXT` | disable the 1M-context beta (bool) — relevant to gateway operators on 1M |
| `CLAUDE_CODE_DISABLE_LEGACY_MODEL_REMAP` | stop remapping legacy model aliases (bool) |
| `FALLBACK_FOR_ALL_PRIMARY_MODELS` | fallback model for all primaries (str) |

### Registry object 2 — API limits, proxy, transport (`h38`)

```js
var h38={};j_(h38,{NO_PROXY:()=>WB1,MAX_THINKING_TOKENS:()=>xB1,MAX_STRUCTURED_OUTPUT_RETRIES:()=>uB1,MAX_MCP_OUTPUT_TOKENS:()=>mB1,HTTP_PROXY:()=>XB1,HTTPS_PROXY:()=>PB1,CLAUDE_STREAM_IDLE_TIMEOUT_MS:()=>cB1,CLAUDE_SLOW_FIRST_BYTE_MS:()=>dB1,CLAUDE_MOCK_HEADERLESS_429:()=>rB1,CLAUDE_ENABLE_STREAM_WATCHDOG:()=>QB1,CLAUDE_ENABLE_BYTE_WATCHDOG:()=>gB1,CLAUDE_CODE_SLOW_OPERATION_THRESHOLD_MS:()=>oB1,CLAUDE_CODE_RETRY_WATCHDOG:()=>lB1,CLAUDE_CODE_MAX_TURNS:()=>IB1,CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY:()=>bB1,CLAUDE_CODE_MAX_RETRIES:()=>vB1,CLAUDE_CODE_MAX_OUTPUT_TOKENS:()=>EB1,CLAUDE_CODE_MAX_CONTEXT_TOKENS:()=>SB1,CLAUDE_CODE_FORCE_SYNC_OUTPUT:()=>iB1,CLAUDE_CODE_FILE_READ_MAX_OUTPUT_TOKENS:()=>CB1,CLAUDE_CODE_EXTRA_METADATA:()=>VB1,CLAUDE_CODE_EXTRA_BODY:()=>NB1,CLAUDE_CODE_EAGER_FLUSH:()=>nB1,CLAUDE_CODE_CLIENT_KEY_PASSPHRASE:()=>RB1,CLAUDE_CODE_CLIENT_KEY:()=>GB1,CLAUDE_CODE_CLIENT_CERT:()=>ZB1,CLAUDE_CODE_CERT_STORE:()=>LB1,CLAUDE_CODE_ATTRIBUTION_HEADER:()=>yB1,API_TIMEOUT_MS:()=>pB1,API_TARGET_INPUT_TOKENS:()=>FB1,API_MAX_INPUT_TOKENS:()=>UB1,API_FORCE_IDLE_TIMEOUT:()=>BB1,ANTHROPIC_CUSTOM_HEADERS:()=>kB1,ANTHROPIC_BETAS:()=>hB1})
```

Types (from `ulq`): `int` → `CLAUDE_CODE_MAX_RETRIES`, `CLAUDE_CODE_MAX_OUTPUT_TOKENS`, `CLAUDE_CODE_MAX_CONTEXT_TOKENS`, `CLAUDE_CODE_FILE_READ_MAX_OUTPUT_TOKENS`, `CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY`, `MAX_THINKING_TOKENS`, `MAX_STRUCTURED_OUTPUT_RETRIES`, `MAX_MCP_OUTPUT_TOKENS`, `API_TIMEOUT_MS`, `API_FORCE_IDLE_TIMEOUT`, `API_MAX_INPUT_TOKENS`, `API_TARGET_INPUT_TOKENS`, `CLAUDE_STREAM_IDLE_TIMEOUT_MS`, `CLAUDE_SLOW_FIRST_BYTE_MS`, `CLAUDE_CODE_SLOW_OPERATION_THRESHOLD_MS`. `bool` → `CLAUDE_ENABLE_BYTE_WATCHDOG`, `CLAUDE_CODE_RETRY_WATCHDOG`, `CLAUDE_CODE_EAGER_FLUSH`, `CLAUDE_CODE_FORCE_SYNC_OUTPUT`, `CLAUDE_MOCK_HEADERLESS_429`. `triBool` → `CLAUDE_ENABLE_STREAM_WATCHDOG`. Everything else `str`. Note `CLAUDE_CODE_MAX_TURNS` is **str** (`IB1=pH.str()`), not int.

| Var | Effect |
|---|---|
| `MAX_THINKING_TOKENS` | cap on thinking/reasoning tokens (int) |
| `CLAUDE_CODE_MAX_OUTPUT_TOKENS` | cap on output tokens per response (int) |
| `CLAUDE_CODE_MAX_CONTEXT_TOKENS` | cap on context window used (int) |
| `CLAUDE_CODE_FILE_READ_MAX_OUTPUT_TOKENS` | Read-tool output token cap (int) |
| `MAX_MCP_OUTPUT_TOKENS` | MCP tool output token cap (int) |
| `CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY` | parallel tool-call concurrency (int) |
| `CLAUDE_CODE_MAX_RETRIES` / `MAX_STRUCTURED_OUTPUT_RETRIES` | retry caps (int) |
| `API_TIMEOUT_MS` / `API_MAX_INPUT_TOKENS` / `API_TARGET_INPUT_TOKENS` / `API_FORCE_IDLE_TIMEOUT` | request tuning (int) |
| `CLAUDE_STREAM_IDLE_TIMEOUT_MS` / `CLAUDE_SLOW_FIRST_BYTE_MS` / `CLAUDE_CODE_SLOW_OPERATION_THRESHOLD_MS` | stream/latency watchdog thresholds (int) |
| `CLAUDE_ENABLE_STREAM_WATCHDOG` (triBool) / `CLAUDE_ENABLE_BYTE_WATCHDOG` / `CLAUDE_CODE_RETRY_WATCHDOG` | stream/byte/retry watchdogs — gateway-relevant for slow upstreams |
| `HTTP_PROXY` / `HTTPS_PROXY` / `NO_PROXY` | proxy config (str) |
| `ANTHROPIC_CUSTOM_HEADERS` / `ANTHROPIC_BETAS` | inject headers / anthropic-beta values (str) |
| `CLAUDE_CODE_EXTRA_BODY` / `CLAUDE_CODE_EXTRA_METADATA` | merge extra fields into request body/metadata (str) — useful for routing through gateways |
| `CLAUDE_CODE_CLIENT_CERT` / `CLAUDE_CODE_CLIENT_KEY` / `CLAUDE_CODE_CLIENT_KEY_PASSPHRASE` / `CLAUDE_CODE_CERT_STORE` | mTLS client cert config (str) |
| `CLAUDE_CODE_ATTRIBUTION_HEADER` | attribution header value (str) |
| `CLAUDE_CODE_EAGER_FLUSH` / `CLAUDE_CODE_FORCE_SYNC_OUTPUT` | stream-flush behavior (bool) — relevant when a gateway buffers SSE |

### Registry object 3 — cloud providers / gateway / CCR / AWS (`k38`)

```js
var k38={};j_(k38,{_CLAUDE_CODE_ASSUME_FIRST_PARTY_BASE_URL:()=>KU1,CLOUD_ML_REGION:()=>jU1,CLAUDE_GATEWAY_LOG_LEVEL:()=>VU1,CLAUDE_GATEWAY_ALLOW_LOOPBACK:()=>NU1,CLAUDE_CODE_USE_VERTEX:()=>sB1,CLAUDE_CODE_USE_MANTLE:()=>HU1,CLAUDE_CODE_USE_FOUNDRY:()=>tB1,CLAUDE_CODE_USE_CCR_V2:()=>_U1,CLAUDE_CODE_USE_BEDROCK:()=>aB1,CLAUDE_CODE_USE_ANTHROPIC_AWS:()=>eB1,CLAUDE_CODE_SKIP_HFI_VERSION_CHECK:()=>xU1,CLAUDE_CODE_SIMULATE_PROXY_USAGE:()=>hU1,CLAUDE_CODE_PROXY_RESOLVES_HOSTS:()=>kU1,CLAUDE_CODE_PROVIDER_MANAGED_BY_HOST:()=>LU1,CLAUDE_CODE_GB_REFRESH_INTERVAL_MS:()=>EU1,CLAUDE_CODE_GB_BASE_URL:()=>vU1,CLAUDE_CODE_ENABLE_GATEWAY_MODEL_DISCOVERY:()=>yU1,CLAUDE_CODE_API_BASE_URL:()=>OU1,CCR_FORCE_BUNDLE:()=>bU1,CCR_ENABLE_BUNDLE:()=>CU1,CCR_AGENT_PROXY_ENABLED:()=>SU1,AWS_SHARED_CREDENTIALS_FILE:()=>ZU1,AWS_REGION:()=>MU1,AWS_PROFILE:()=>PU1,AWS_DEFAULT_REGION:()=>XU1,AWS_CONFIG_FILE:()=>WU1,ANTHROPIC_VERTEX_PROJECT_ID:()=>fU1,ANTHROPIC_VERTEX_BASE_URL:()=>wU1,ANTHROPIC_UNIX_SOCKET:()=>RU1,ANTHROPIC_SMALL_FAST_MODEL_AWS_REGION:()=>GU1,ANTHROPIC_FOUNDRY_RESOURCE:()=>DU1,ANTHROPIC_FOUNDRY_BASE_URL:()=>JU1,ANTHROPIC_BEDROCK_SERVICE_TIER:()=>zU1,ANTHROPIC_BEDROCK_MANTLE_BASE_URL:()=>$U1,ANTHROPIC_BEDROCK_BASE_URL:()=>TU1,ANTHROPIC_BASE_URL:()=>qU1,ANTHROPIC_AWS_WORKSPACE_ID:()=>AU1,ANTHROPIC_AWS_BASE_URL:()=>YU1,AGENT_PROXY_URL:()=>IU1})
```

Types (from `mlq`): `bool` → `CLAUDE_CODE_USE_{BEDROCK,VERTEX,FOUNDRY,MANTLE,ANTHROPIC_AWS}`, `CLAUDE_CODE_USE_CCR_V2`, `_CLAUDE_CODE_ASSUME_FIRST_PARTY_BASE_URL`, `CLAUDE_CODE_PROVIDER_MANAGED_BY_HOST`, `CLAUDE_CODE_SIMULATE_PROXY_USAGE`, `CLAUDE_CODE_PROXY_RESOLVES_HOSTS`, `CLAUDE_GATEWAY_ALLOW_LOOPBACK`, `CLAUDE_CODE_ENABLE_GATEWAY_MODEL_DISCOVERY`, `CCR_AGENT_PROXY_ENABLED`, `CCR_ENABLE_BUNDLE`, `CCR_FORCE_BUNDLE`, `CLAUDE_CODE_SKIP_HFI_VERSION_CHECK`. `int` → `CLAUDE_CODE_GB_REFRESH_INTERVAL_MS`. Everything else `str`.

| Var | Effect |
|---|---|
| `CLAUDE_CODE_API_BASE_URL` | API base URL override (str) — core gateway/CCR knob |
| `ANTHROPIC_BASE_URL` | base URL override (str) |
| `CLAUDE_CODE_USE_BEDROCK` / `_USE_VERTEX` / `_USE_FOUNDRY` / `_USE_MANTLE` / `_USE_ANTHROPIC_AWS` | select cloud provider backend (bool) |
| `CLAUDE_CODE_USE_CCR_V2` | route through CCR v2 (bool) |
| `CCR_ENABLE_BUNDLE` / `CCR_FORCE_BUNDLE` / `CCR_AGENT_PROXY_ENABLED` | CCR bundling / agent-proxy toggles (bool) |
| `CLAUDE_GATEWAY_LOG_LEVEL` (str) / `CLAUDE_GATEWAY_ALLOW_LOOPBACK` (bool) | gateway logging + loopback allow |
| `CLAUDE_CODE_ENABLE_GATEWAY_MODEL_DISCOVERY` | discover models from gateway (bool) |
| `CLAUDE_CODE_GB_BASE_URL` / `CLAUDE_CODE_GB_REFRESH_INTERVAL_MS` | GrowthBook feature-flag base URL + refresh (str/int) |
| `CLAUDE_CODE_PROXY_RESOLVES_HOSTS` / `CLAUDE_CODE_SIMULATE_PROXY_USAGE` / `CLAUDE_CODE_PROVIDER_MANAGED_BY_HOST` | proxy/host integration toggles (bool) |
| `_CLAUDE_CODE_ASSUME_FIRST_PARTY_BASE_URL` | treat custom base URL as first-party (bool) — affects auth/headers |
| `ANTHROPIC_BEDROCK_BASE_URL` / `_SERVICE_TIER` / `ANTHROPIC_VERTEX_BASE_URL` / `_PROJECT_ID` / `ANTHROPIC_FOUNDRY_BASE_URL` / `_RESOURCE` / `ANTHROPIC_AWS_BASE_URL` / `_WORKSPACE_ID` | provider-specific endpoint/identity (str) |
| `ANTHROPIC_UNIX_SOCKET` | connect over a unix socket (str) |
| `AGENT_PROXY_URL` | agent proxy URL (str) |
| `AWS_REGION` / `AWS_DEFAULT_REGION` / `AWS_PROFILE` / `AWS_CONFIG_FILE` / `AWS_SHARED_CREDENTIALS_FILE` / `CLOUD_ML_REGION` | standard AWS/GCP credential + region resolution (str) |

### Registry object 4 — the big behavioral/session/runtime set (`R38`)

This is the largest object (130+ entries). Selected verbatim head:

```js
var R38={};j_(R38,{VOICE_STREAM_BASE_URL:()=>Om1,VCR_RECORD:()=>Kp1,ULTRAPLAN_PROMPT_FILE:()=>Km1,TEST_ENABLE_SESSION_PERSISTENCE:()=>qp1,TEAM_MEMORY_SYNC_URL:()=>qm1,TASK_MAX_OUTPUT_LENGTH:()=>pp1,SLASH_COMMAND_TOOL_CHAR_BUDGET:()=>mp1,SESSION_INGRESS_URL:()=>_m1, ... MCP_TOOL_TIMEOUT:()=>up1,MCP_TIMEOUT:()=>xp1,MCP_SERVER_CONNECTION_BATCH_SIZE:()=>Ip1,MCP_CONNECT_TIMEOUT_MS:()=>Sp1,MCP_CONNECTION_NONBLOCKING:()=>Hp1, ... CLAUDE_CONFIG_DIR:()=>mu1,CLAUDE_CODE_SESSION_ID:()=>Gu1,CLAUDE_CODE_ENTRYPOINT:()=>rx1,CLAUDE_CODE_SHELL:()=>hu1,CLAUDE_CODE_SHELL_PREFIX:()=>ku1,CLAUDE_CODE_GLOB_TIMEOUT_SECONDS:()=>wp1,CLAUDE_CODE_OVERRIDE_DATE:()=>Tu1,CLAUDE_CODE_AUTO_COMPACT_WINDOW:()=>Fx1,CLAUDE_AUTOCOMPACT_PCT_OVERRIDE:()=>Rx1,CLAUDE_BASH_MAINTAIN_PROJECT_WORKING_DIR:()=>jm1,CLAUDE_AUTO_BACKGROUND_TASKS:()=>fm1, ... CLAUDE_TMPDIR:()=>tu1,CLAUDE_CODE_TMPDIR:()=>Cu1,CLAUDE_ENV_FILE:()=>gu1, ...})
```

Operator-relevant members (type from `Ilq`: tail block is `.str()`, then a `.bool()` run `Tm1..Kp1`, then an `.int()` run `Op1..pp1`):

| Var | Type | Effect |
|---|---|---|
| `CLAUDE_CONFIG_DIR` | str | config directory override |
| `CLAUDE_CODE_OVERRIDE_DATE` | str | override the "today's date" injected into the prompt |
| `CLAUDE_CODE_AUTO_COMPACT_WINDOW` / `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE` | str/int | auto-compact window + percentage trigger override |
| `CLAUDE_CODE_SHELL` / `CLAUDE_CODE_SHELL_PREFIX` | str | shell binary + command prefix for Bash tool |
| `CLAUDE_BASH_MAINTAIN_PROJECT_WORKING_DIR` | bool | keep Bash cwd pinned to project dir |
| `CLAUDE_CODE_GLOB_TIMEOUT_SECONDS` | int | Glob tool timeout |
| `CLAUDE_CODE_PWSH_PARSE_TIMEOUT_MS` / `CLAUDE_CODE_PLUGIN_GIT_TIMEOUT_MS` / `CLAUDE_SUBAGENT_BG_SHELL_MAX_MS` | int | per-subsystem timeouts |
| `MCP_TIMEOUT` / `MCP_TOOL_TIMEOUT` / `MCP_CONNECT_TIMEOUT_MS` | int | MCP startup + per-tool + connect timeouts |
| `MCP_SERVER_CONNECTION_BATCH_SIZE` / `MCP_REMOTE_SERVER_CONNECTION_BATCH_SIZE` | int | MCP connection batching |
| `MCP_CONNECTION_NONBLOCKING` | bool | non-blocking MCP connect |
| `MCP_TRUNCATION_PROMPT_OVERRIDE` | str | override MCP truncation prompt |
| `TASK_MAX_OUTPUT_LENGTH` / `SLASH_COMMAND_TOOL_CHAR_BUDGET` | int | Task tool / slash-command output budgets |
| `CLAUDE_TMPDIR` / `CLAUDE_CODE_TMPDIR` / `CLAUDE_STAGE_FILE_ROOT` / `CLAUDE_JOB_DIR` | str | temp/stage/job directories |
| `CLAUDE_ENV_FILE` | str | env file to load |
| `CLAUDE_CODE_SESSION_ID` / `_SESSION_NAME` / `_SESSION_KIND` | str | session identity (also exposed to hooks) |
| `CLAUDE_CODE_ENTRYPOINT` | str | entrypoint label (cli/sdk/action…) — visible in hook env |
| `CLAUDE_CODE_AUTO_MODE_EXTERNAL_PERMISSIONS` | str | auto-mode external permission control |
| `CLAUDE_CODE_SUBPROCESS_ENV_SCRUB` | bool | scrub env passed to subprocesses |
| `CLAUDE_CODE_DONT_INHERIT_ENV` | bool | don't inherit parent env into tools |
| `CLAUDE_CODE_SESSIONEND_HOOKS_TIMEOUT_MS` | int | SessionEnd hook timeout |
| `CLAUDE_CODE_USER_DIALOG_TIMEOUT_MS` | int | user dialog/prompt timeout |
| `CLAUDE_CODE_RESUME_TOKEN_THRESHOLD` / `_RESUME_THRESHOLD_MINUTES` / `_IDLE_TOKEN_THRESHOLD` / `_IDLE_THRESHOLD_MINUTES` | int | resume/idle thresholds |
| `CLAUDE_CODE_PLAN_V2_AGENT_COUNT` / `_PLAN_V2_EXPLORE_AGENT_COUNT` | int | plan-mode parallel agent counts |
| `CLAUDE_AUTO_BACKGROUND_TASKS` | bool | auto-background long tasks |
| `CLAUDE_CODE_HIDE_SETTINGS_HINT` / `CLAUDE_CODE_FORCE_FULL_LOGO` / `CLAUDE_CODE_SYNTAX_HIGHLIGHT` | bool | TUI display toggles |
| `CLAUDE_CODE_MANAGED_SETTINGS_PATH` / `CLAUDE_CODE_REMOTE_SETTINGS_PATH` | str | settings file path overrides |
| `CLAUDE_CODE_MCP_ALLOWLIST_ENV` | str | MCP allowlist source |

### Separate mechanism — feature-gate / GrowthBook flag table

A large flat string table (bundle offset ~14.94M) lists gate/flag names, NOT process.env registry getters. These are GrowthBook-style feature gates whose names can also act as env overrides. Verbatim sample:

```
DISABLE_TELEMETRY ... DISABLE_PROMPT_CACHING(_SONNET/_OPUS/_HAIKU/_FABLE/_MYTHOS) ... DISABLE_INTERLEAVED_THINKING ... DISABLE_AUTOUPDATER ... DISABLE_AUTO_COMPACT ... DISABLE_COST_WARNINGS ... DISABLE_BUG_COMMAND ... CLAUDE_CODE_ENABLE_TELEMETRY ... USE_API_CONTEXT_MANAGEMENT / USE_API_CLEAR_TOOL_USES / USE_API_CLEAR_TOOL_RESULTS ... ENABLE_TOOL_SEARCH ... ENABLE_PROMPT_CACHING_1H(_BEDROCK) ... CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC ... CLAUDE_CODE_DISABLE_TERMINAL_TITLE ... CLAUDE_CODE_DISABLE_THINKING ... CLAUDE_CODE_DISABLE_ADAPTIVE_THINKING ...
```

Confirmed by user-facing strings:

```
DesignSync is unavailable while nonessential network traffic is restricted (CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC is set). Unset it to use /design-sync.
/feedback has been disabled via the DISABLE_BUG_COMMAND environment variable
/feedback has been disabled via the CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC environment variable
```

Key operator gates (effect from name): `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` (kill telemetry + nonessential network + disables /feedback, /design-sync, Projects), `DISABLE_TELEMETRY` / `CLAUDE_CODE_ENABLE_TELEMETRY` (telemetry), `DISABLE_AUTOUPDATER` / `DISABLE_UPDATES` (auto-update), `DISABLE_AUTO_COMPACT` / `DISABLE_COMPACT` (compaction), `DISABLE_PROMPT_CACHING(_<tier>)` (prompt caching — gateway-relevant), `DISABLE_INTERLEAVED_THINKING`, `DISABLE_BUG_COMMAND` / `DISABLE_FEEDBACK_COMMAND` / `DISABLE_DOCTOR_COMMAND` / `DISABLE_LOGIN_COMMAND` / `DISABLE_LOGOUT_COMMAND` (hide slash commands), `DISABLE_ERROR_REPORTING`, `DISABLE_COST_WARNINGS`, `CLAUDE_CODE_DISABLE_TERMINAL_TITLE`, `CLAUDE_CODE_DISABLE_THINKING` / `_DISABLE_ADAPTIVE_THINKING`, `USE_API_CONTEXT_MANAGEMENT` / `USE_API_CLEAR_TOOL_USES` / `USE_API_CLEAR_TOOL_RESULTS` (server-side context management betas).

### Operator notes

- **Live reads:** every var is re-read from `process.env` on each access (`get:()=>O.parse(process.env[K])`) — you can mutate `process.env` at runtime and it takes effect; the registry's `set` trap also writes back to `process.env`.
- **Boolean syntax matters:** only `1/true/yes/on` (case-insensitive) enable a bool var; `0/false/no`/empty disable it. `=false` is the same as unset.
- **Type surprises:** `CLAUDE_CODE_MAX_TURNS` is parsed as a STRING (`IB1=pH.str()`), and `CLAUDE_CODE_AUTO_COMPACT_WINDOW` is a string, not int. Most `_MS`/`MAX_*_TOKENS`/`*_TIMEOUT*` vars are ints.
- **Gateway/CCR core knobs:** `CLAUDE_CODE_API_BASE_URL`, `ANTHROPIC_BASE_URL`, `CLAUDE_CODE_USE_CCR_V2`, `CCR_*`, `CLAUDE_CODE_PROXY_RESOLVES_HOSTS`, `CLAUDE_CODE_PROVIDER_MANAGED_BY_HOST`, `_CLAUDE_CODE_ASSUME_FIRST_PARTY_BASE_URL`, `CLAUDE_GATEWAY_*`, `CLAUDE_CODE_ENABLE_GATEWAY_MODEL_DISCOVERY`.
- **Budget tuning:** `MAX_THINKING_TOKENS`, `CLAUDE_CODE_MAX_OUTPUT_TOKENS`, `CLAUDE_CODE_MAX_CONTEXT_TOKENS`, `CLAUDE_CODE_FILE_READ_MAX_OUTPUT_TOKENS`, `MAX_MCP_OUTPUT_TOKENS`, `BASH_MAX_OUTPUT_LENGTH`, `TASK_MAX_OUTPUT_LENGTH`, `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE`.
- **Caching through a gateway:** if your gateway strips/forwards cache breakpoints incorrectly, `DISABLE_PROMPT_CACHING` (and per-tier variants) and `ENABLE_PROMPT_CACHING_1H[_BEDROCK]` are the relevant gates.
- **Air-gapped / quiet mode:** `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC=1` is the single biggest "go quiet" switch (also disables several slash commands).
- **Hook authors:** `CLAUDE_CODE_SESSION_ID`, `CLAUDE_CODE_ENTRYPOINT`, `CLAUDE_CODE_SESSION_KIND`, `CLAUDE_CODE_SESSIONEND_HOOKS_TIMEOUT_MS` are exposed/relevant when scripting hooks; `CLAUDE_CODE_SUBPROCESS_ENV_SCRUB` and `CLAUDE_CODE_DONT_INHERIT_ENV` change what env reaches your hook/tool subprocesses.
- **Effects with `(inferred)` caveat:** per-variable effects in the tables are inferred from name+declared type; the registry structure, parser semantics, names, and types are all verbatim-grounded. The ~14.94M gate table is a distinct feature-flag mechanism, not the process.env registry.
