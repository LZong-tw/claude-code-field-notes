## Tools & Deferred Tool-Search

Claude Code v2.1.178 keeps tools out of the model's context by **deferring** them: a deferred tool appears only as a *name* in a `<system-reminder>`, with no JSON schema, and is uninvocable until the model calls the `ToolSearch` tool to fetch its schema. This is the "tool-search-tool" beta. Deferred-but-unfetched tools are excluded from the prompt's tool-token budget. Whether deferral runs at all, and how aggressively, is controlled almost entirely by `ENABLE_TOOL_SEARCH` plus the API host.

### The ToolSearch tool (the fetch mechanism)

`ToolSearch` (minified name constant `u$="ToolSearch"`) is the one tool that is *never* deferred (so it's always callable). Its description spells out the whole lifecycle:

```
bf3=`Fetches full schema definitions for deferred tools so they can be called.

Deferred tools appear by name in <system-reminder> messages.`
xf3=` Until fetched, only the name is known — there is no parameter schema, so calling the tool fails with InputValidationError. When any instruction, system reminder, or other tool's description names a deferred tool, fetch it with query "select:<name>" before calling it.`
uf3=` ...returns the matched tools' complete JSONSchema definitions inside a <functions> block. Once a tool's schema appears in that result, it is callable exactly like any tool defined at the top of the prompt.
...
Query forms:
- "select:Read,Edit,Grep" — fetch these exact tools by name
- "notebook jupyter" — keyword search, up to max_results best matches
- "+slack send" — require "slack" in the name, rank by remaining terms`
```

So fetching is by exact name (`select:a,b,c`), keyword search, or `+required` term — and the result is encoded identically to the top-of-prompt tool list.

### Enablement gate (`DV` / `yh_`) — modes standard / tst / tst-auto

The mode is resolved by `yh_()`, then `DV()` decides if optimistic deferral is on:

```
function yh_(){if(DSH())return"standard";let H=process.env.ENABLE_TOOL_SEARCH,_=H?LB8(H):null;if(_===0)return"tst";if(_===100)return"standard";if(Of3(H))return"tst-auto";if(z_(H))return"tst";if(J4(process.env.ENABLE_TOOL_SEARCH))return"standard";return"tst"}
```

```
function DV(){let H=yh_();if(H==="standard"){...return!1}if(!process.env.ENABLE_TOOL_SEARCH&&S8()==="firstParty"&&!w3()){...N(`[ToolSearch:optimistic] disabled: ANTHROPIC_BASE_URL=${process.env.ANTHROPIC_BASE_URL} is not a first-party Anthropic host. Set ENABLE_TOOL_SEARCH=true (or auto / auto:N) if your proxy forwards tool_reference blocks.`);return!1}if(!process.env.ENABLE_TOOL_SEARCH&&S8()==="vertex"){...N("[ToolSearch:optimistic] disabled: Vertex AI does not accept the tool-search beta header. Set ENABLE_TOOL_SEARCH=true to override.");return!1}...return!0}
```

Observed behavior:
- **Kill switch:** `DSH()` (set by `CLAUDE_CODE_DISABLE_EXPERIMENTAL_BETAS` or hipaa mode) forces `"standard"` → no deferral. `function DSH(){return z_(process.env.CLAUDE_CODE_DISABLE_EXPERIMENTAL_BETAS)||cX_("hipaa")}`
- **Default (`ENABLE_TOOL_SEARCH` unset) on first-party host:** falls through to `"tst"` — deferral on.
- **Custom base URL / proxy:** if `ENABLE_TOOL_SEARCH` is unset and the host is `firstParty` but `w3()` (base-URL check) is false, optimistic search is **disabled** with the `[ToolSearch:optimistic] disabled: ANTHROPIC_BASE_URL=...` log. This is the gateway/CCR gotcha.
- **Vertex:** disabled (Vertex won't accept the `tool-search-tool-2025-10-19` beta header) unless forced.

### `ENABLE_TOOL_SEARCH` value parsing (`LB8` / `Of3` / `Izq`)

```
function LB8(H){if(!H.startsWith("auto:"))return null;let _=H.slice(5),q=parseInt(_,10);if(isNaN(q))return N(`Invalid ENABLE_TOOL_SEARCH value "${H}": expected auto:N where N is a number.`),null;return Math.max(0,Math.min(100,q))}
function Of3(H){if(!H)return!1;return H==="auto"||H.startsWith("auto:")}
function Izq(){let H=process.env.ENABLE_TOOL_SEARCH;if(!H)return bzq;if(H==="auto")return bzq;let _=LB8(H);if(_!==null)return _;return bzq}
```
with `bzq=10`. So:
- `auto:N` → N clamped to 0..100, used as a **percent of the context window** for the auto threshold.
- `auto:0` → mode `tst` (always defer); `auto:100` → mode `standard` (never defer); plain `auto` → `tst-auto` at default 10%.
- `Izq()` (the percent) defaults to `10` and is also the percent used when computing token thresholds.

### What is deferred — the predicate `Ux` (isDeferredTool)

```
function Ux(H){if(H.alwaysLoad===!0)return!1;if(kV7().includes(H.name))return!1;if(H.isMcp===!0)return!0;if(H.name===u$)return!1;if(H.name===G9){if((h$H(),x8(lV7)).isForkSubagentEnabled())return!1}if(H.name===Sf3)return!1;if(H.name===Cf3)return!1;if(H.name===WQ&&nt_())return!1;if(H.name===CY&&X$H())return!1;if(H.name===P$H&&process.env.CLAUDE_CODE_SESSION_KIND==="bg")return!1;return H.shouldDefer===!0}
```

Resolved tool-name constants: `u$="ToolSearch"`, `G9="Agent"` (Task), `WQ="PushNotification"`, `CY="ScheduleWakeup"`, `P$H="EnterWorktree"`. Rules, in order:
1. `alwaysLoad===true` → never deferred.
2. In `kV7()` (the `non_deferrable_builtins` allowlist) → never deferred.
3. **Any MCP tool (`isMcp===true`) → always deferred.** This is why every `mcp__*` tool shows up as a name-only reminder.
4. ToolSearch, Task (when fork-subagent is on), PushNotification/ScheduleWakeup (when their features are active), and EnterWorktree (in `bg` session kind) → never deferred.
5. Otherwise defer iff the tool's own `shouldDefer===true` flag is set.

Several built-ins carry `shouldDefer:!0` (e.g. NotebookEdit, WebFetch, WebSearch — `"content from a URL",maxResultSizeChars:1e5,shouldDefer:!0`), so they are deferred when tool search is active.

The `non_deferrable_builtins` allowlist is server-configurable and empty by default:
```
function kV7(){let H=new Set;try{let _=A_("tengu_non_deferrable_builtins",null);if(Array.isArray(_)){...H.add(q)}}catch{}try{let _=S_().clientDataCache?.non_deferrable_builtins;...}catch{}if(H.size===0)return $f3;return[...H]}
```
(`$f3=[]` default). A `tengu_tool_search_unsupported_models` list (`zf3()`) defaults to `Tf3=["claude-3-5-haiku","claude-3-haiku"]`, used by `$r()` to disable tool search on those models.

### Token accounting — deferred tools excluded until loaded

This is the core of "keeps tools out of context." Built-ins are split into non-deferred (`A`) vs deferred (`w`):

```
let{isDeferredTool:$}=...,A=T.filter((X)=>!$(X)),w=T.filter((X)=>$(X)),f=A.length>0?await iRH(A,_,q,K):0,...
if(w.length>0&&Y){...for(let[Z,W]of w.entries()){let G=Math.max(0,(P[Z]||0)-KB6),R=X.has(W.name);if(J.push({name:W.name,tokens:G,isLoaded:R}),M+=G,R)D+=G}}
...return{builtInToolTokens:f+D,deferredBuiltinDetails:J,deferredBuiltinTokens:M-D,systemToolDetails:j}
```

Observed:
- Non-deferred tools' tokens (`f`) always count toward `builtInToolTokens`.
- For each deferred tool, its estimated tokens get a flat discount of `KB6=500` (`Math.max(0,(P[Z]||0)-KB6)`).
- `R` = whether that deferred tool's name already appeared in a prior `tool_use` (i.e. it was fetched/used). **Loaded deferred tools (`D`) are added back into `builtInToolTokens`; only not-yet-loaded deferred tokens (`M-D`) live in the separate `deferredBuiltinTokens` bucket.** So a deferred tool costs nothing in the main budget until the model actually fetches and uses it.

The context-view shows them as separate rows: `push({name:"MCP tools (deferred)",...isDeferred:!0})` and `{name:"System tools (deferred)",...isDeferred:!0}`.

### Auto-search threshold math (`CCO` / `paK` / `BaK`)

In `tst-auto` mode the decision compares deferred-tool size to a percent of the context window:

```
function CCO(H,_,q,K){let O=await yCO(H,_,q,K);if(O!==null){let $=paK(K);return{enabled:O>=$,debugDescription:`${O} tokens (threshold: ${$}, ${Izq()}% of context)`,...}}let T=await vCO(H,_,q,K),z=BaK(K);return{enabled:T>=z,debugDescription:`${T} chars (threshold: ${z}, ${Izq()}% of context) (char fallback)`,...}}
function paK(H){let _=QM(H,SW(foH(H))),q=Izq()/100;return Math.floor(_*q)}
function BaK(H){return Math.floor(paK(H)*VCO)}
```
with `VCO=2.5`, `bzq=10`, `KB6=500`, and `yCO` subtracting 500 from the token estimate. So: the **token threshold = Izq()% of context window** (default 10%); if deferred-tool tokens exceed it, deferral turns on. If the token count can't be computed it falls back to a **char threshold = 2.5× the token threshold**. The `[source: ...]` / `Auto tool search enabled: N tokens (threshold: ...)` debug lines are emitted when this fires.

### Delta caching and the usage reminder

Deferred-tool changes are tracked as a delta attachment, capped at `DEFERRED_DELTA_LIST_CAP = e7H = 30` (`e7H=30`); `getDeferredToolsDelta` (`uzq`) walks `deferred_tools_delta` attachments, accumulating `addedNames`/`removedNames`/`readdedNames`. The cache is invalidated and re-emitted with `ToolSearchTool: cache invalidated - deferred tools changed` when the set changes. For groups like Chrome DevTools MCP, the prompt nudges the model to batch fetches into one call:

```
**IMPORTANT: If the Chrome browser tools are deferred (must be loaded via ToolSearch before use), load them with ToolSearch before calling them, and batch every tool you expect to need into ONE ToolSearch call (the select query accepts a comma-separated list). Do NOT load tools one at a time; each separate ToolSearch call wastes a full round-trip.**
```

### Operator notes
- **Gateways/CCR break deferral silently.** With `ANTHROPIC_BASE_URL` pointed at a non-first-party host and `ENABLE_TOOL_SEARCH` unset, optimistic tool search is auto-disabled (look for `[ToolSearch:optimistic] disabled: ANTHROPIC_BASE_URL=...`). Re-enable by either setting `ENABLE_TOOL_SEARCH=true` (or `auto`/`auto:N`) — only if your proxy actually forwards `tool_reference`/`tool_search` beta blocks — or set `_CLAUDE_CODE_ASSUME_FIRST_PARTY_BASE_URL=1` to make `w3()` treat the proxy as first-party.
- **Vertex** rejects the `tool-search-tool-2025-10-19` beta header; deferral is off there unless you force `ENABLE_TOOL_SEARCH=true`.
- **`ENABLE_TOOL_SEARCH` cheat-sheet:** unset = default on (first-party); `true`/`1` = force on; `auto` = threshold mode at 10% of context; `auto:N` = threshold at N% (0..100, parseInt of after `auto:`); `auto:0` = always defer; `auto:100` / `false` = never defer. `auto:foo` (non-numeric) logs an "Invalid ENABLE_TOOL_SEARCH value" warning and falls back to the 10% default.
- **Kill switch:** `CLAUDE_CODE_DISABLE_EXPERIMENTAL_BETAS=1` (and hipaa mode) forces standard mode — no tools deferred at all.
- **MCP tools are always deferred** when tool search is active (`isMcp===true → defer`). Heavy MCP setups therefore stay out of the token budget until the model fetches them; if your gateway disables deferral, those tools flood the context instead.
- **No fixed tool-count cap was found** — deferral is budget-driven (auto threshold = % of context window, default 10%; char fallback = 2.5×). The only count-like limit is the delta-list cap of 30.
- **Token-budget accounting:** deferred tools each get a flat 500-token discount and contribute to a separate `deferredBuiltinTokens` bucket until the model actually invokes them, at which point they move into the main `builtInToolTokens` count. So `/context`-style token views under-report tools the model hasn't touched yet.
- **Tool-search is disabled for `claude-3-5-haiku` / `claude-3-haiku`** by default (server-overridable via `tengu_tool_search_unsupported_models`).
