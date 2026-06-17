## thinking-and-effort

How Claude Code v2.1.178 turns an operator's effort setting into request parameters. Two distinct things travel to the wire: an `effort` string (a real API field on capable models) and a `thinking` object (with `budget_tokens`). They are computed by separate code paths and clamped against the model's max-output budget.

### Effort levels and their valid set

The canonical effort enum has five members; the persisted setting (`settings.json` `effortLevel`) only accepts four (no `max`).

```
eh=["low","medium","high","xhigh","max"]
effortLevel:k.enum(["low","medium","high","xhigh"]).optional().catch(void 0).describe("Persisted effort level for supported models.")
```

`ultracode:true` in settings resolves to `"xhigh"`; otherwise the resolver reads `effortLevel`:

```
function Zd9(H){let _=KC(H.cli.effort);if(_!==void 0)return _;if(H.settings.ultracode===!0)return"xhigh";return IMH(H.settings.effortLevel)}
```

`IMH` only passes through low/medium/high/xhigh (so a settings value of `"max"` is dropped here); `KC` parses CLI/env effort and also accepts numeric strings:

```
function IMH(H){if(H==="low"||H==="medium"||H==="high"||H==="xhigh")return H;return ...
function KC(H){...let _=String(H).toLowerCase(),q=Cd9[_]??_;if(FvH(q))return q;let K=parseInt(_,10);if(!isNaN(K)&&Ed9(K))return K;return ...
```

### Env override and precedence (`CLAUDE_CODE_EFFORT_LEVEL`)

A session-only env override sits above settings. `auto`/`unset` clears it back to "model default":

```
function gvH(){let H=process.env.CLAUDE_CODE_EFFORT_LEVEL;return H?.toLowerCase()==="unset"||H?.toLowerCase()==="auto"?null:KC(H)}
```

The `/effort` slash command warns that it is session-scoped and does not persist or reach a remote process:

```
Use low, medium, high, or xhigh instead.
... is session-only (nothing saved) ...
Cleared effort from settings, but CLAUDE_CODE_EFFORT_LEVEL=
```

Spawned child sessions inherit effort via a different env var, `CLAUDE_EFFORT` (not `CLAUDE_CODE_EFFORT_LEVEL`):

```
if(H.effortLevel!==void 0)_.CLAUDE_EFFORT=H.effortLevel;
```

### Per-model effort clamp

The resolver `Rt` clamps unsupported levels down to `"high"`: `max` requires `UvH`, `xhigh` requires `uMH`:

```
function Rt(H,_){...let T=O??(q?K:void 0)??_??K;if(T==="max"&&!UvH(H))return"high";if(T==="xhigh"&&!uMH(H))return"high";return T}
```

Capability is model-keyed. `max` (`max_effort`) and `xhigh` (`xhigh_effort`) support differ by model:

```
function UvH(H){...if(q==="claude-fable-5"||q==="claude-mythos-5"||q==="claude-opus-4-8"||q==="claude-opus-4-7"||q==="claude-opus-4-6"||q==="claude-sonnet-4-6")return!0;return ...   // supports "max"
function uMH(H){...if(q==="claude-fable-5"||q==="claude-mythos-5"||q==="claude-opus-4-8"||q==="claude-opus-4-7")return!0;return ...                                                       // supports "xhigh"
```

Note the asymmetry: `claude-opus-4-6`/`claude-sonnet-4-6` support `max` but NOT `xhigh`, so an `xhigh` request on those models is silently downgraded to `high`. The per-model default effort:

```
function d36(H){if(T9(H)==="claude-fable-5")return"high";if(T9(H)==="claude-opus-4-8")return"high";if(T9(H)==="claude-opus-4-7")return"xhigh";return"high"}
```

A launch-effort "pin" (gated by `S_().unpinOpus48LaunchEffort` etc.) can make a model default to its top effort:

```
function QvH(H){let _=T9(H);if(_.includes("opus-4-7"))return!S_().unpinOpus47LaunchEffort;if(_.includes("opus-4-8"))return!S_().unpinOpus48LaunchEffort;...
```

### Effort -> request param `effort` (NOT reasoning_effort)

The resolved effort string is sent to the API verbatim as `effort`, only on models where `WP()` (the `effort` model capability) is true, together with the `effort-2025-11-24` beta header (`jrH`). On unsupported models the field is deleted:

```
function RAT(H,_,q,K,O){if(!WP(O)){delete _.effort;return}if("effort"in _)return;if(H===void 0)K.push(jrH);else if(typeof H==="string")_.effort=H,K.push(jrH)}
jrH=SJ("effort","effort-2025-11-24")
```

There is no `reasoning_effort` field — the param is named `effort` and is distinct from the thinking budget below. The resolved value is plumbed through as `effortValue` and re-resolved at request time:

```
KH=Rt(A,T.effortValue),$H=WP(A)&&KH!==void 0?ATH(KH):void 0;
function ATH(H){if(typeof H==="string")return FvH(H)?H:"high";return"high"}
```

### Thinking budget -> request `thinking.budget_tokens`

Thinking is a SEPARATE object. Per-model thinking limits live in `CXH` (`_` = default, `q` = upperLimit); `f37` returns `upperLimit-1`:

```
function CXH(H){...if(K==="claude-opus-4-8")_=64000,q=128000;else if(K==="claude-sonnet-4-6")_=32000,q=128000;...else if(K==="claude-opus-4-5"||K==="claude-sonnet-4-5"||K==="claude-haiku-4-5")_=32000,q=64000;else if(K==="claude-opus-4-1"||K==="claude-opus-4-0")_=32000,q=32000;...let O=$37(H);if(O?.max_tokens&&O.max_tokens>=4096)q=O.max_tokens,_=Math.min(_,q);return{default:_,upperLimit:q}}
function f37(H){return CXH(H).upperLimit-1}
```

The thinking config `GK` is built per model. Newer models (e.g. sonnet-4-6, and anything `MoH` marks adaptive) use `type:"adaptive"` (the model self-budgets); otherwise it's `type:"enabled"` with an explicit `budget_tokens` clamped to `max_tokens-1`:

```
KO=v_8(T.model);if(KO!==void 0?KO==="adaptive":MoH(A)&&!$4)GK={type:"adaptive",display:d9};else{let R9=f37(A);if(q.type==="enabled"&&q.budgetTokens!==void 0)R9=q.budgetTokens;R9=Math.min(Wq-1,R9),GK={budget_tokens:R9,type:"enabled",display:d9}}
```

The final request body carries them side by side; `max_tokens` (`Wq`) is from the model/override, not from effort:

```
metadata:c0H(),max_tokens:Wq,thinking:GK,...
Wq=Math.min(k6?.maxTokensOverride||T.maxOutputTokensOverride||j8,j8),...
```

### `MAX_THINKING_TOKENS` override and enable/disable gates

`MAX_THINKING_TOKENS` overrides the configured `maxThinkingTokens` and, if `>0`, force-enables thinking; `0` disables it:

```
process.env.MAX_THINKING_TOKENS?parseInt(process.env.MAX_THINKING_TOKENS,10):z.maxThinkingTokens;if(n6!==void 0){if(n6>0)g3=!0,v3={type:"enabled",budgetTokens:n6};else if(n6===0)g3=!1,v3={type:"disabled"}}
```

The "is thinking on" gate: `MAX_THINKING_TOKENS>0` forces on; otherwise `alwaysThinkingEnabled===false` turns it off; default is on:

```
function sqH(){if(process.env.MAX_THINKING_TOKENS)return parseInt(process.env.MAX_THINKING_TOKENS,10)>0;let{settings:H}=cF();if(H.alwaysThinkingEnabled===!1)return!1;return!0}
```

An explicit per-request budget converts to a thinking object via `jaK` (0 => disabled): `function jaK(H){return H===0?{type:"disabled"}:{type:"enabled",budgetTokens:H}}`. A `null` budget with thinking enabled yields adaptive: `if(H===null)return _!==void 0&&sqH()?{type:"adaptive",...}`. The disabled path is honored on first-party: `q.type==="disabled"&&S8()==="firstParty"&&...GK={type:"disabled"}`. There is also a hard kill switch `CLAUDE_CODE_DISABLE_THINKING` read alongside this logic (`C7=z_(process.env.CLAUDE_CODE_DISABLE_THINKING)`).

Note: `budget_tokens` is the field name in the actual streaming request (`thinking:{type:"enabled",budget_tokens:Fzq}`), while internal objects use `budgetTokens`. `Fzq=1024` is only the probe budget used by the token-counting helper, not the runtime budget.

### Interleaved-thinking beta

The `interleaved-thinking-2025-05-14` beta (`frH`) is added when the model supports it (`KZ_`) and `DISABLE_INTERLEAVED_THINKING` is not set:

```
frH=SJ("interleaved_thinking","interleaved-thinking-2025-05-14")
if(!z_(process.env.DISABLE_INTERLEAVED_THINKING)&&KZ_(H))_.push(frH);
function KZ_(H){...if(q==="claude-haiku-4-5"||q.includes("claude-3-"))return!1;return!0}
```

Adaptive thinking and the interleaved beta are mutually exclusive on the request: when the thinking config resolves to a non-default form, the code removes the interleaved beta token from the beta list (`if(GK&&d9){let $4=z8.indexOf(gX_);if($4!==-1)z8.splice($4,1)}`), consistent with the migration note in the bundle ("remove `interleaved-thinking-2025-05-14` once you're on adaptive thinking").

### Operator notes

- Effort and thinking are TWO independent request parameters. Effort goes out as a top-level `effort` string (with the `effort-2025-11-24` beta); thinking goes out as `thinking.budget_tokens` (or `thinking.type:"adaptive"`). A gateway/proxy that strips betas will lose `effort-2025-11-24` and the effort param will be silently dropped server-side even though the client still set it.
- Precedence for effort: `CLAUDE_CODE_EFFORT_LEVEL` (session env, set to `auto`/`unset` to clear) > CLI `--effort` > `ultracode:true` (=> xhigh) > `settings.json effortLevel`. `effortLevel` in settings only accepts low/medium/high/xhigh; `max` is reachable only via env/CLI/launch-pin, never persisted.
- Effort is clamped per model: `xhigh`/`max` fall back to `high` on models that don't advertise that capability. Opus-4-6 and Sonnet-4-6 support `max` but not `xhigh` — requesting xhigh there gives you high. Don't assume your `--effort xhigh` survived; it depends on the active model.
- To force a specific thinking budget regardless of model defaults, set `MAX_THINKING_TOKENS` (>0 forces thinking on with that budget, 0 disables). The budget is still hard-clamped to `max_tokens-1`, and to the model upperLimit (`CXH`): opus-4-8/sonnet-4-6/fable/mythos cap at 128000; opus-4-5/sonnet-4-5/haiku-4-5 at 64000; opus-4-1/4-0 at 32000.
- Newer models use adaptive thinking (no explicit `budget_tokens` on the wire) — `MAX_THINKING_TOKENS` matters mainly for the explicit-budget path; on adaptive models the budget is server-decided.
- Kill switches: `CLAUDE_CODE_DISABLE_THINKING`, `alwaysThinkingEnabled:false` (settings), `MAX_THINKING_TOKENS=0`, and `DISABLE_INTERLEAVED_THINKING` (drops only the interleaved beta, not thinking itself).
- Child/agent sessions read effort from `CLAUDE_EFFORT`, not `CLAUDE_CODE_EFFORT_LEVEL` — set that var if you wrap the harness and want spawned agents to inherit a non-default effort.
- This is a base-URL/beta-header story, not an S8 provider-mode story: the thinking `disabled` shortcut is specifically gated on `S8()==="firstParty"`, and interleaved-capability has a `foundry` special-case (`KZ_` returns true for foundry), but effort itself is gated purely on the model's `effort` capability plus the beta header surviving to the wire.
