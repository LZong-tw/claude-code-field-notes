## MCP (Model Context Protocol) Loading

How Claude Code v2.1.178 discovers MCP server configs, connects transports, names the resulting tools, and surfaces them to the model. All claims below are grepped from the minified bundle; identifiers (`Ux`, `_n`, `q1`, `um`, `ec9`) are stable for this version.

### Config sources, scopes, and discovery

MCP servers are read from multiple config sources, each tagged with a `scope`. The observed scope literals are `enterprise`, `user`, `local`, `project`, and `dynamic` (the last is `--mcp-config`):

```js
// project .mcp.json
{filePath:Y,expandVars:_,scope:"project"}
// user settings mcpServers
content:{mcpServers:K},expandVars:_,scope:"user"
// local settings mcpServers
content:{mcpServers:K},expandVars:_,scope:"local"
// managed/enterprise
{filePath:K,expandVars:_,scope:"enterprise"}
// --mcp-config (command-line) → dynamic
FK=v6_({filePath:$1,expandVars:!0,scope:"dynamic"});if(FK.config)z1=FK.config.mcpServers
```

Project-level discovery walks the directory tree for `.mcp.json` (it is also in the always-read filename list), resolving the project root and each ancestor:

```js
[".profile",".ripgreprc",".mcp.json"]
A.push(DO.resolve(D,".mcp.json"))   // per-directory resolution
case "project": return ar7.join(u_(),".mcp.json")
```

The project `.mcp.json` must have a top-level `mcpServers` **object**; a legacy `servers` key is explicitly rejected with a rename hint:

```text
.mcp.json is malformed (not valid JSON, or mcpServers is not an object)
Rename the top-level "servers" key to "mcpServers" in
```

The settings schema also exposes per-project approval gates for `.mcp.json` servers:

```js
enableAllProjectMcpServers ... "to automatically approve all MCP servers in the project"
enabledMcpjsonServers:k.array(k.string()).optional()
  .describe("List of approved MCP servers from .mcp.json")
disabledMcpjsonServers:k.array(k.string()).optional()
```

So a `.mcp.json` server is only used if it is in `enabledMcpjsonServers` (or `enableAllProjectMcpServers:true`) and not in `disabledMcpjsonServers`. (Observed: these three settings keys plus the helper names `isMcpServerDisabled`, `isMcpServerDenied`, `isMcpServerAllowedByPolicy`.)

### `--mcp-config` and `--strict-mcp-config`

`--mcp-config` takes one or more configs (file path or inline JSON), parsed under the `dynamic` scope and merged on top of disk config:

```text
--mcp-config <configs...>
--strict-mcp-config  Only use MCP servers from --mcp-config, ignoring all other MCP configurations
```

```js
mcpConfig:[],strictMcpConfig:!1   // arg-parser defaults
if($==="--strict-mcp-config"){K.strictMcpConfig=!0;continue}
```

Validation errors are surfaced as `--mcp-config validation failed (...)` / `--mcp-config: N entry warning(s)` and the warning channel `mcp-config-invalid`. With `--strict-mcp-config`, **string-spec** servers that resolve from disk config are dropped:

```text
' skipped: string specs resolve from disk config, which --strict-mcp-config ignores
```

`--strict-mcp-config` is mutually exclusive with an enterprise/managed MCP config:

```text
You cannot use --strict-mcp-config when an enterprise MCP config is present
```

In dispatched/subagent sessions `--mcp-config` is described as "Only use MCP servers from --mcp-config in dispatched sessions" (a separate, narrower help string), and the runtime carries `strictMcpConfig` and `mcpConnectNonBlocking` flags in process state (`WjH()`/`ye6()`, `PjH()`/`Ve6()`).

### Transports (stdio / sse / http / ws, + IDE variants)

Each server config is a discriminated union on `type`. Observed transport schemas:

```js
type:k.literal("stdio").optional(),command:k.string().min(1, ...)   // stdio (default)
type:k.literal("sse"),url:k.string(),headers:k.record(...)          // sse
type:k.enum(["http","streamable-http"]).transform(()=>"http")       // http
type:k.literal("ws"),url:k.string(),headers:...                     // ws
type:k.literal("sse-ide"),url:...,ideName:...                       // IDE-injected SSE
```

`type` defaults to `stdio` when omitted or when a `command` is present; url-based transports require `url`:

```js
let f=...typeof w.type==="string"?w.type:"stdio"
if(T.type==="stdio"||T.type===void 0&&"command"in T)T.type=...
if((f.type==="sse"||f.type==="http"||f.type==="ws")&&"url" ...
```

stdio configs go through env/arg variable expansion (`expandVars`), and the internal "ide" and "computer-use" servers are themselves injected as stdio servers spawning the CLI (`{type:"stdio",command:process.execPath,args:[...,"--computer-use-mcp"]}`). OAuth-bearing remote servers attach `oauth` to the sse/http config (`X={type:"sse",url:T,headers:j,oauth:D}`).

### Tool name prefixing and aliasing (`mcp__server__tool`)

MCP tools are surfaced with the `mcp__<server>__<tool>` naming convention, assembled by `_n` from a sanitized server prefix (`um`) and a sanitized tool name (`q1`):

```js
function um(H){return`mcp__${q1(H)}__`}
function _n(H,_){return`${um(H)}${q1(_)}`}
function YyH(H){return H.mcpInfo?_n(H.mcpInfo.serverName,H.mcpInfo.toolName):H.name}
```

Sanitization replaces any char outside `[a-zA-Z0-9_-]` with `_` (special-casing `claude.ai ` connector names by collapsing/trimming underscores). Note: **no length truncation is performed in `q1` in this build** — it is purely a character filter:

```js
function q1(H){let _=H.replace(/[^a-zA-Z0-9_-]/g,"_");
  if(H.startsWith("claude.ai "))_=_.replace(/_+/g,"_").replace(/^_|_$/g,"");return _}
```

Parsing back the other direction (`mcp__server__tool` → `{serverName, mcpToolName}`) splits on `__`, requiring at least 3 segments, and rejoins everything after the server segment so tool names may themselves contain `__`:

```js
function ec9(H){if(!H.startsWith("mcp__"))return;let _=H.split("__");
  if(_.length<3)return;let q=_[1],K=_.slice(2).join("__");
  if(!q||!K)return;return{serverName:q,mcpToolName:K}}
```

The permission/matcher layer uses regex `\bmcp__[A-Za-z0-9_-]+__([A-Za-z0-9_-]+)` (rewriting to `mcp__<server>__$1`) and `^mcp__(.+?)__` to extract the server for per-server permission rules. So permission rules can target a whole server (`mcp__github__*`) or a single tool.

### MCP tools are DEFERRED by default (ToolSearch gating)

This is the most operationally important behavior: **every MCP tool is deferred** — i.e. its schema is not loaded into the model's tool list up front; it must be fetched via ToolSearch. The deferral predicate `Ux` returns `true` for any tool flagged `isMcp`:

```js
function Ux(H){if(H.alwaysLoad===!0)return!1;
  if(kV7().includes(H.name))return!1;
  if(H.isMcp===!0)return!0;   // ← all MCP tools deferred
  ...}
```

MCP tools are flagged with `isMcp:!0` at creation (e.g. `isMcp:!0,mcpInfo:{serverName:H,toolName:...}`, and the generic MCP tool wrapper `Y7({isMcp:!0,...,name:"mcp",...})`).

The override is the per-server `alwaysLoad` config flag — `alwaysLoad:k.boolean().optional()` in the server schema — which makes that server's tools load eagerly. The connect pipeline distinguishes the two passes explicitly:

```js
"--mcp-config alwaysLoad servers"   // eager connect/load pass
"--mcp-config servers"              // regular (deferred) pass
```

When the model needs a deferred MCP tool, ToolSearch resolves it; the search code filters specifically on the `mcp__` prefix and reports `mcpServersConfigured` count, and emits `tengu_tool_search_mcp_wait` telemetry while waiting for non-blocking connects. The `mcp__ide__*` tools are special-cased (kept non-deferred / always present in IDE sessions): `!H.name.startsWith("mcp__ide__")||FI3.includes(H.name)`.

### Connection timing and telemetry

Connections can be non-blocking (`mcpConnectNonBlocking`), with lifecycle hooks `before_mcp_connect_connector` / `after_mcp_connect_user` and status telemetry `mcpServersConfigured` / `mcpServersConnected` / `mcpServersPending` (plus session-diff `mcpServersAdded` / `mcpServersRemoved`). A failed server is retained as a stub with empty tools: `name:z,config:{...$,scope:"user"}},tools:[]`.

### Operator notes

- **Expect MCP tools to be invisible until searched.** Every MCP tool is deferred (`Ux`: `isMcp===!0 → defer`). The model sees only the tool *name* in a system-reminder and must call ToolSearch (`select:<name>`) to load the schema before it can call the tool. Budget for one extra round-trip per new MCP tool.
- **Force eager loading with `alwaysLoad:true`** on the server entry in `.mcp.json`/`--mcp-config` if a server's tools must always be present (no ToolSearch hop). This is the only documented escape from deferral besides the built-in `mcp__ide__*` allow-list.
- **`.mcp.json` servers need explicit approval.** They are gated by `enabledMcpjsonServers` / `disabledMcpjsonServers` / `enableAllProjectMcpServers` in settings — an unapproved project server silently won't load. Use the top-level `mcpServers` key (not `servers`).
- **`--mcp-config` accepts file paths or inline JSON**, merged under scope `dynamic` on top of disk config. **`--strict-mcp-config`** ignores all disk/user/project/enterprise config (and drops string-spec servers), and **cannot** be combined with an enterprise MCP config.
- **Tool name shape is `mcp__<server>__<tool>`.** Both server and tool names are sanitized with `[^a-zA-Z0-9_-] → _`. Gateways/permission rules should match this exact form; per-server wildcards work because the matcher extracts the server via `^mcp__(.+?)__`. Tool names may contain `__` (rejoined after the server segment).
- **No client-side name-length truncation observed in `q1`** for this version — if your server/tool names are long, the full `mcp__server__tool` string is sent as-is; any length cap is enforced upstream, not in the bundle.
- **Transports:** `stdio` (default; needs `command`), `sse`/`http`(`streamable-http`)/`ws` (need `url`, optional `headers`/`oauth`), plus IDE-internal `sse-ide`/`ws-ide`. Omitting `type` with a `command` present implies `stdio`.
- **Minimal/`--slim` mode (`CLAUDE_CODE_SIMPLE=1`) still honors `--mcp-config`** — it skips plugin sync, hooks, LSP etc., but MCP via `--mcp-config` is explicitly listed as a supported context source.
