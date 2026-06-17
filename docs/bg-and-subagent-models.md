## Background / Small-Fast / Subagent Model Selection

Claude Code v2.1.178 runs several classes of traffic on a cheaper "small/fast" (haiku-class) model rather than the main conversational model. There are three distinct resolvers, each with its own precedence chain:

- `JP()` — the **small/fast model** id (haiku-class). Used by lightweight, non-interactive internal queries.
- `$oK()` / `zoK()` / `YoK()` — the **background-classifier** model + thinking selector, gated by the `tengu_bg_classifier_config` flag.
- `sHH(...)` — the **subagent / Task model** resolver, honoring `CLAUDE_CODE_SUBAGENT_MODEL` and per-agent overrides.
- `c9()` — the **main model** (what the interactive loop uses), for contrast.

### The small/fast model: `JP()`

```js
function JP(){if(process.env.ANTHROPIC_SMALL_FAST_MODEL)return process.env.ANTHROPIC_SMALL_FAST_MODEL;let H=S8(),_=H==="firstParty"&&(w3()||l28())||H==="anthropicAws";if(!process.env.ANTHROPIC_DEFAULT_HAIKU_MODEL&&!_)return c9();return FOH()}
```

Precedence:
1. `ANTHROPIC_SMALL_FAST_MODEL` env (verbatim, wins outright).
2. Otherwise, for first-party (with a feature gate `w3()||l28()`) or `anthropicAws` providers, OR if `ANTHROPIC_DEFAULT_HAIKU_MODEL` is set → fall through to `FOH()` (the haiku default).
3. Otherwise (other gateways/providers without a haiku model configured) → `c9()`, i.e. it **falls back to the main model**. This is the key operator gotcha: on a custom gateway, "small/fast" traffic silently runs on your *main* model unless you set `ANTHROPIC_SMALL_FAST_MODEL` or `ANTHROPIC_DEFAULT_HAIKU_MODEL`.

The haiku default itself:

```js
function FOH(){if(process.env.ANTHROPIC_DEFAULT_HAIKU_MODEL)return process.env.ANTHROPIC_DEFAULT_HAIKU_MODEL;return U28()}
function U28(H=NT()){return H[uqH]}
```

`FOH()` returns `ANTHROPIC_DEFAULT_HAIKU_MODEL` if set, else the haiku slot from the model catalog (`NT()`). The catalog contains `claude-haiku-4-5` / `claude-3-5-haiku` / `claude-3-haiku` entries (observed in strings: `haiku45`, `haiku35`, `claude-haiku-4-5`).

**What runs on `JP()` (small/fast):**

- Stop-hook / programmatic "condition satisfied?" checker agents — they default to the small/fast model when no model is specified:
```js
let j=F6({content:f}),J=H.model??JP(),D=(W)=>z&&z.length>0?[...r$T(z,J,W),j]:[j]
```
```js
ok: false with reason if the condition is not met`]),C=H.model??JP(),S=50,...mainLoopModel:C,isNonInteractiveSession:!0,thinkingConfig:{type:"disabled"}
```
(note `thinkingConfig:{type:"disabled"}` — these run thinking-off.)
- A non-interactive helper query path that hard-pins the small model with thinking disabled:
```js
thinkingConfig:{type:"disabled"},tools:[],...options:{...O,...,model:JP(),enablePromptCaching:...}
```

### The background classifier: `zoK` / `$oK` / `YoK`

```js
function zoK(){return A_("tengu_bg_classifier_config",{useSmallFastModel:!0,disableThinking:!0})}
function $oK(){if(zoK()?.useSmallFastModel)return JP();let H=c9();if(dj(H)||aX_(H))return sX_(H);return H}
function YoK(H){if(DoH(H))return[void 0,ToK];if(zoK()?.disableThinking)return[!1,0];return[void 0,ToK]}
```

- `zoK()` reads remote/gate config `tengu_bg_classifier_config`; **bundled defaults are `useSmallFastModel:true, disableThinking:true`**.
- `$oK()` is the classifier model selector: if `useSmallFastModel` is on → `JP()` (small/fast); otherwise the main model `c9()` (with a downgrade `sX_(H)` if the main model is a heavy/opus class via `dj`/`aX_`).
- `YoK()` returns `[thinking, thinkingTokens]`; with `disableThinking` on it returns `[false, 0]`.

**What uses the classifier model (`$oK()`):**

- `agent_namer` — generates a 2–4 word label for a background job:
```js
A=$oK(),[w,f]=YoK(A),J=(await eB({querySource:"agent_namer",model:A,thinking:w,max_tokens:32+f,maxRetries:1,skipSystemPromptPrefix:!0,messages:[{role:"user",content:`2-4 word lowercase label for this job.
```
- `agent_classifier` — classifies/routes (observed `querySource:"agent_classifier",model:X,...` where `X=$oK()`):
```js
...,X=$oK(),[P,Z]=YoK(X);A="apiError";for(let G=0;G<2&&!W;G++){...
```

The `CLAUDE_CODE_BG_CLASSIFIER_MODEL` env var is present in the env-binding table (alongside `CLAUDE_CODE_SUBAGENT_MODEL`), but classifier model choice in code flows through `$oK()`/`tengu_bg_classifier_config`; treat the env var as the named override hook for this classifier path.

### Subagent / Task model: `sHH(H,_,q,K,O)`

This is the resolver every Task/subagent invocation goes through. Arguments: `H` = the agent-config model (from `TAH`), `_` = parent `mainLoopModel`, `q` = an explicit per-call override (e.g. the Task tool's `model` param), `K` = permission mode, `O` = a "downgraded" callback.

```js
function sHH(H,_,q,K,O){let T=()=>wL({permissionMode:K??"default",mainLoopModel:_,exceeds200kTokens:!1}),...$=process.env.CLAUDE_CODE_SUBAGENT_MODEL;if($){if($==="inherit")return T();let j=f9($);if(!M4(j))return z($);return j}...if(q){if(q==="inherit")return T();if(fZK(q,_))return _;let j=A(wZK(f9(q)),q);if(!M4(j))return z(q,j);return j}let w=H??r6q();if(w==="inherit")return T();...}
function r6q(){return"inherit"}
function LTO(H){N(`Subagent model "${H}" is not in the availableModels allowlist; inheriting the parent model instead`,{level:"warn"})}
```

Precedence (highest → lowest):
1. **`CLAUDE_CODE_SUBAGENT_MODEL` env** — wins for *all* subagents. Value `"inherit"` ⇒ use the parent main-loop model (`T()` = `wL(...mainLoopModel:_...)`); any other value is resolved via `f9()`; if it's not in the allowlist (`M4`) it warns (`z()` → `LTO`) and falls back to the parent.
2. **Per-call override `q`** (the Task tool's `model` param) — same `inherit` / allowlist handling.
3. **Agent-config model `H`** (from the agent definition), defaulting to `r6q()` = `"inherit"` when absent.
4. Final fallback in every branch: the parent's main-loop model. So a subagent **inherits the main model by default** unless something explicitly downgrades it.

Bedrock-cross-region adjustment (`A`/`liH`) and opus-`[1m]` tagging (`wZK`) are applied to non-inherited choices.

Call sites all pass the same shape, e.g.:
```js
DH=qH.model?sHH(void 0,C.options.mainLoopModel,qH.model,PH.mode):void 0
u=sHH(TAH(C,A.options.mainLoopModel),A.options.mainLoopModel,...)
resolvedAgentModel:sHH(Bx.model,_.options.mainLoopModel,void 0,I8(_).mode)
```

#### Built-in "explore/search" agent forced to haiku

`TAH` maps the agent config to a model and **forces the built-in fast-search agent onto haiku**:

```js
function TAH(H,_){if(H.agentType!==p4H.agentType||H.source!=="built-in")return H.model;if(!A_("tengu_quartz_heron",!1))return"haiku";return LzO(_)?XGK:"inherit"}
```

So the built-in `Fast read-only search agent for locating code` runs on `"haiku"` by default (when the `tengu_quartz_heron` gate is off); with the gate on it uses `XGK` or inherits. All *other* (custom/user) subagents return their own configured model and otherwise inherit.

#### Teammate variant `u1q` / `ox6`

A parallel resolver for "teammate" agents honors the same env var first:

```js
function u1q(H,_){let q=process.env.CLAUDE_CODE_SUBAGENT_MODEL;if(q&&q!=="inherit"){let K=f9(q);if(M4(K))return K;return x1q(q),ox6(_)}if(H==="inherit")return _??ox6(_);...}
```

#### Spawned `claude` child processes

When a subagent is launched as a child CLI process, the same env var is translated to a `--model` flag:

```js
let A=process.env.CLAUDE_CODE_SUBAGENT_MODEL;if(A&&A!=="inherit")_.push(`--model ${lK([A])}`);else{let w=$f();if(w)_.push(`--model ${lK([w])}`)}
```

### Main model (for contrast): `c9()`

```js
function c9(){let H=En();if(H!==void 0&&H!==null)return f9(H);return yA()}
```
`En()` resolves the configured model (config override → `ANTHROPIC_MODEL` → catalog default), else `yA()`. This is what the interactive loop, and the non-haiku branch of every resolver above, fall back to.

### Traffic split summary (observed)

| Traffic | Resolver | Default model |
|---|---|---|
| Interactive main loop / Task subagents (default) | `c9()` / `sHH` inherit | main model |
| Built-in fast read-only search agent | `TAH` → `sHH` | `"haiku"` |
| Background `agent_namer` (job label) | `$oK` | small/fast (haiku) when `useSmallFastModel` |
| Background `agent_classifier` | `$oK` | small/fast (haiku) when `useSmallFastModel` |
| Stop-hook / programmatic condition agents | `H.model??JP()` | small/fast, thinking disabled |
| Non-interactive helper query | `JP()` | small/fast, thinking disabled |

### Operator notes

- **Pin the cheap model on gateways.** `JP()` falls back to the *main* model (`c9()`) when the provider isn't first-party/AWS and neither `ANTHROPIC_SMALL_FAST_MODEL` nor `ANTHROPIC_DEFAULT_HAIKU_MODEL` is set. On a CCR/custom gateway, set **`ANTHROPIC_SMALL_FAST_MODEL`** (or `ANTHROPIC_DEFAULT_HAIKU_MODEL`) or your background/classifier/hook traffic will hit your expensive main model.
- **Force every subagent's model with `CLAUDE_CODE_SUBAGENT_MODEL`.** It overrides per-agent and per-Task `model` settings. Use `CLAUDE_CODE_SUBAGENT_MODEL=inherit` to make all subagents reuse the parent main model. Non-allowlisted values warn (`"…not in the availableModels allowlist; inheriting the parent model instead"`) and silently inherit.
- **Subagents inherit the main model by default.** Absent an env override or an explicit agent/Task `model`, `sHH` returns the parent main-loop model. The one built-in exception is the fast search agent (`"haiku"`).
- **Background classifier defaults to small + no-thinking.** `tengu_bg_classifier_config` bundled default is `{useSmallFastModel:true, disableThinking:true}`; a remote gate can flip these, which would move `agent_namer`/`agent_classifier` onto the main model. `CLAUDE_CODE_BG_CLASSIFIER_MODEL` is the named env hook for this path.
- **Region pinning:** `ANTHROPIC_SMALL_FAST_MODEL_AWS_REGION` pins the small model's Bedrock region independently of the main model.
- **Child-process subagents** get `--model` injected from `CLAUDE_CODE_SUBAGENT_MODEL` (non-`inherit`), so the same knob propagates to spawned `claude` CLI instances.
- Related env present in the binding table but tangential here: `CLAUDE_CODE_FORK_SUBAGENT`, `CLAUDE_SUBAGENT_BG_SHELL_MAX_MS`.
