## Usage, Cost & Token Counting

_Verified against Claude Code 2.1.178._

Claude Code tracks context size in two different ways depending on purpose: an **API-anchored count** (trusts the `usage` block returned by the last assistant turn) for the running context display and auto-compact decisions, and a **local 4-chars-per-token estimator** for the tail of messages that haven't been sent to the API yet and for cheap pre-flight checks. Cost is computed locally from the API `usage` block against a hardcoded per-model price table.

### The context-token summation `ae`

The single most important primitive is `ae`, which sums all four token buckets of an API `usage` object into one "how big is this turn's context" number:

```js
function ae(H){return H.input_tokens+(H.cache_creation_input_tokens??0)+(H.cache_read_input_tokens??0)+H.output_tokens
```

So the displayed context size includes `input_tokens + cache_creation_input_tokens + cache_read_input_tokens + output_tokens`. Cache reads and cache writes both count toward context occupancy — a fully cache-hit turn still shows a large context because `cache_read_input_tokens` is summed in. Operators routing through a gateway/CCR must preserve all four fields in the `usage` block or the context meter will under-report.

### API-anchored context with local tail: `TX` / `Jv7` / `R2`

The total context is not recomputed locally from scratch. `TX` finds the most recent assistant message that carries a real `usage` block (the "anchor"), takes the API-reported `ae(usage)` for everything up to and including it, then **only locally estimates the messages that came after the anchor**:

```js
function TX(H,_){let q=Jv7(H);if(!q)return R2(H,_);return ae(q.usage)+R2(H.slice(q.anchorIndex+1),_)}
function Jv7(H){let _=H.length-1;while(_>=0){let q=H[_],K=q?t7H(q):void 0;if(q&&K){let O=jv7(q);...return{usage:K,anchorIndex:_}}_--}return null}
function R2(H,_){let q=0;for(let K of H)q+=UCO(K,_);return q}
```

`Jv7` walks the history backward to find the last usage-bearing assistant turn. If none exists (`!q`), it falls back to a fully-local estimate via `R2`. This is why the context number is accurate right after an assistant reply (API truth) but drifts to an estimate as you queue local input.

### The local tokenizer is `length/4` (no real BPE)

`R2` → `UCO` → `rPH` → `cz`. The actual "tokenizer" `cz` is a character-count heuristic, **not** a BPE tokenizer:

```js
function cz(H,_=4){if(typeof H!=="string")return 0;return Math.round(H.length/_)}
function Vw3(H){switch(H){case"json":case"jsonl":case"jsonc":return 2;default:return 4}}
function aN7(H,_){return cz(H,Vw3(_))}
```

Default divisor is 4 chars/token; JSON-family content uses 2 chars/token (`Vw3`). Per-block estimation (`rPH`/`yw3`) assigns fixed costs to non-text blocks:

```js
function yw3(H,_){...if(H.type==="image"||H.type==="document")return 2000;if(H.type==="tool_result")return rPH(H.content,_);if(H.type==="tool_use")return cz(H.name+xH(H.input??{}),_);if(H.type==="thinking")return cz(H.thinking,_);...}
```

Images/documents are hardcoded at **2000 tokens each**. So the locally-displayed tail count can diverge substantially from real model tokenization — it's a budget heuristic, not ground truth.

### The real `count_tokens` API (with Haiku fallback)

For accurate counts Claude Code calls the server `count_tokens` endpoint, with a fallback chain:

```js
async function Y3_(H,_){try{let q=await jIH(H,_);if(q!==null)return q;N(`countTokensWithFallback: API returned null, trying haiku fallback (${_.length} tools)`)}catch(q){...}try{let q=await uaK(H,_);...}
```

Endpoints seen: `/v1/messages/count_tokens` and `/v1/messages/count_tokens?beta=true`. If the primary call returns null/fails, it retries against a Haiku model; if that also fails it logs `count_tokens_unreachable`. Gateways must proxy `/v1/messages/count_tokens` or token counting silently degrades to the `length/4` heuristic.

### Cost computation → `cost_usd_micros`

Cost is computed locally from the `usage` block. `uZ5` is the core formula (dollars), and the result is multiplied by `1e6` and rounded to produce `cost_usd_micros`:

```js
function uZ5(H,_){return _.input_tokens/1e6*H.inputTokens+_.output_tokens/1e6*H.outputTokens+(_.cache_read_input_tokens??0)/1e6*H.promptCacheReadTokens+xZ5(H,_)+(_.server_tool_use?.web_search_requests??0)*H.webSearchRequests}
```
```js
...cost_usd_micros:Math.round(D*1e6)
...cost_usd:i,cost_usd_micros:Math.round(i*1e6),duration_ms:n,...
```

`H` is the per-model price table (dollars per million tokens). Cache-write cost is split between 5-minute and 1-hour ephemeral rates by `xZ5`:

```js
function xZ5(H,_){let q=_.cache_creation_input_tokens??0,K=H.promptCacheWrite1hTokens,O=Math.min(_.cache_creation?.ephemeral_1h_input_tokens??0,q);if(K===void 0||O<=0)return q/1e6*H.promptCacheWriteTokens;return O/1e6*K+(q-O)/1e6*H.promptCacheWriteTokens}
```

Hardcoded price tables (USD per million tokens) include Sonnet-class `UOH` and Opus-class `_C9`:

```js
UOH={inputTokens:3,outputTokens:15,promptCacheWriteTokens:3.75,promptCacheWrite1hTokens:6,promptCacheReadTokens:0.3,webSearchRequests:0.01}
_C9={inputTokens:15,outputTokens:75,promptCacheWriteTokens:18.75,promptCacheWrite1hTokens:30,promptCacheReadTokens:1.5,webSearchRequests:0.01}
```

Because pricing is a static client-side table keyed by model id, **a gateway that rewrites the model name to something the table doesn't know will produce a wrong (or zero) cost** — the `usage` numbers stay correct but the dollar figure is only as good as the local table.

### Context window resolution: `QM` / `w37` / the `[1m]` suffix

The window the percentage is measured against is resolved by `QM`:

```js
function QM(H,_){let q=A37();if(q!==void 0)return q;if(zy8(H,_))return ct;return w37(H,_)}
function A37(){if(nH.DISABLE_COMPACT&&process.env.CLAUDE_CODE_MAX_CONTEXT_TOKENS){let H=parseInt(process.env.CLAUDE_CODE_MAX_CONTEXT_TOKENS,10);if(!isNaN(H)&&H>0)return H}return}
```

`CLAUDE_CODE_MAX_CONTEXT_TOKENS` overrides the window **only when `DISABLE_COMPACT` is also set**. The default window comes from `w37`, where the `[1m]` model-id suffix (e.g. the user's `claude-opus-4-8[1m]`) forces a 1,000,000-token window:

```js
function w37(H,_){if(Jf(H))return 1e6;if(_?.includes(yn.header)&&$g(H))return 1e6;if(sm(H))return 1e6;let q=IY6(H);if(q!==null)return q;return _Z_}
function Jf(H){if(QOH())return!1;return/\[1m\]/i.test(H)}
```

Constants: `_Z_=200000` (default window), `ct=200000` (the cap used when long-context isn't enabled).

### Auto-compact threshold and reserved headroom

The auto-compact target is computed by `fx6`→`qB6`, which reserves a **fixed 13,000 tokens** below the usable window (and `L1H` first subtracts max output tokens, capped at `CaK=20000`):

```js
function qB6(H,_){let q=H-13000,K=_.testPctOverride;if(K!==void 0&&!isNaN(K)&&K>0&&K<=100)return Math.min(Math.floor(H*(K/100)),q);return q}
function L1H(H,_){let q=Math.min(S$H(H),CaK),K=qh()?_:void 0,{window:O}=nc(H,K);return O-q}
```

`CaK=20000`. A separate cheaper gate (used for microcompact / background-task eligibility) compares `ae(last usage)` against `contextTokenThreshold ?? RCq` with `RCq=1e5` (100k):

```js
...let K=_.contextTokenThreshold??RCq;if(q<K)return!1;...
```

`CLAUDE_CODE_AUTO_COMPACT_WINDOW` overrides the compact window directly (clamped to `[kzq, baK]` = `[100000, 1000000]`), taking precedence over the settings value:

```js
function nc(H,_){...if(process.env.CLAUDE_CODE_AUTO_COMPACT_WINDOW){let Y=s7H("CLAUDE_CODE_AUTO_COMPACT_WINDOW",process.env.CLAUDE_CODE_AUTO_COMPACT_WINDOW,kzq,baK);if(Y.status!=="invalid"){let A=Math.max(kzq,Y.effective);return{window:Math.min(O,A),configured:A,source:"env"}}}...
```

Constants: `kzq=1e5`, `baK=1e6`. Auto-compact is disabled when `DISABLE_AUTO_COMPACT` or `DISABLE_COMPACT` env is truthy:

```js
...if(z_(process.env.DISABLE_AUTO_COMPACT))return!1;return $5("autoCompactEnabled",!0).value}
```

### The displayed percentage

The status bar / `/context` percentage is `(window - used)/window * 100`, where the window is `L1H` (full window minus reserved output), shown as either "context used" or "until auto-compact":

```js
...W=Math.round((Z-q)/Z*100),_[5]=Z,_[6]=q,_[7]=W;...let M=J?`${100-j}% context used`:`${j}% until auto-compact`;
```

So "X% until auto-compact" is measured against the **output-reserved** window, not the raw 200k/1M — the meter hits 0% before raw token count reaches the nominal window size.

### Max output tokens

`CLAUDE_CODE_MAX_OUTPUT_TOKENS` controls the per-request output budget, clamped to a model default/upper limit, and is what gets subtracted (capped at 20k) from the window for context-meter math:

```js
function S$H(H){let _=CXH(H);return s7H("CLAUDE_CODE_MAX_OUTPUT_TOKENS",process.env.CLAUDE_CODE_MAX_OUTPUT_TOKENS,_.default,_.upperLimit).effective}
```

### Operator notes

- **All four usage fields must survive the proxy.** `ae` sums `input_tokens + cache_creation_input_tokens + cache_read_input_tokens + output_tokens`. A gateway that drops `cache_*` fields makes the context meter under-report and skews cost.
- **`cost_usd_micros` is computed client-side from a static price table** (`uZ5`/`UOH`/`_C9`). Rewriting the model id to one the table doesn't know yields wrong or zero cost even though token usage is correct.
- **Cache-write cost splits 5m vs 1h** via `cache_creation.ephemeral_1h_input_tokens` (`xZ5`). Preserve that nested field for accurate write-cost accounting.
- **Local context is `length/4` (JSON `length/2`), images/docs = 2000 tokens flat** (`cz`/`Vw3`/`yw3`). The displayed tail count is a heuristic, not real tokenization; only the API-anchored part (`ae`) is exact.
- **Accurate counting needs `/v1/messages/count_tokens` proxied.** Failure silently falls back to Haiku, then to the `length/4` estimate (`Y3_`).
- **`[1m]` in the model id forces a 1,000,000-token window** (`Jf` → `w37`). Defaults: window `_Z_=200000`, cap `ct=200000`.
- **`CLAUDE_CODE_MAX_CONTEXT_TOKENS` only takes effect when `DISABLE_COMPACT` is set** (`A37`). Otherwise it's ignored.
- **`CLAUDE_CODE_AUTO_COMPACT_WINDOW` overrides the compact window**, clamped to `[100000, 1000000]`, precedence over settings (`nc`). Unset it to let settings/experiment values apply.
- **Reserved headroom:** auto-compact target is `window − 13000` (`qB6`); the context meter window also subtracts max-output (capped at 20000, `CaK`). The "% until auto-compact" therefore reaches 0 before raw tokens reach the nominal window.
- **`CLAUDE_CODE_MAX_OUTPUT_TOKENS`** sets the output budget (clamped to model default/upper limit) and directly shrinks the meter window (`S$H`/`L1H`).
- **Disable auto-compact** with `DISABLE_AUTO_COMPACT` or `DISABLE_COMPACT`.
- **Cheap pre-gate threshold** is `RCq=1e5` (100k) via `contextTokenThreshold` for microcompact/background eligibility.
