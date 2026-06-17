## Model Resolution & Aliases

_Verified against Claude Code 2.1.178._

How a user-supplied `--model` value, a short alias (`sonnet`/`opus`/`haiku`/`fable`/`opusplan`/`best`), or a settings `model` field becomes a concrete wire model id, and how the `[1m]` (1M-context) suffix is detected, preserved, stripped, and re-applied.

### The core resolver `f9(H)` — alias → concrete id

`f9` is the single entry point that turns any user/alias/setting string into a concrete model id. It lowercases, detects a trailing `[1m]`, strips it, and switches on the bare alias:

```js
function f9(H){let _=H.trim(),q=_.toLowerCase(),K=Jf(q),O=K?K1(q).trim():q;
if(uN(O))switch(O){
 case"fable":{let T=Yt();return T+(K&&!Rn()&&!Jf(T)?"[1m]":"")}
 case"opusplan":return $Z()+(K?"[1m]":"");
 case"sonnet":return $Z()+(K?"[1m]":"");
 case"haiku":return FOH()+(K?"[1m]":"");
 case"opus":return K?Tg(cj()):cj();
 case"best":return YC9();
 default:}
if(AO()&&zvH(O)&&LrH())return K?Tg(cj()):cj();
if(K&&Rn()&&QZ5(O)&&sm(O))return _.replace(/(\[1m\])+$/i,"").trim();
if(K)return _.replace(/(\[1m\])+$/i,"").trim()+"[1m]";return _}
```

Key facts from this body:
- `K=Jf(q)` records whether the input carried `[1m]`; `O=K1(q)` is the alias with the suffix stripped (`K1` = `H.replace(/\[1m\]$/i,"")`).
- `uN(O)` gates the alias switch — the alias allowlist is `KvH=["sonnet","opus","haiku","fable","best","sonnet[1m]","opus[1m]","fable[1m]","opusplan"]`.
- `opusplan` and `sonnet` both resolve to `$Z()` (the Sonnet concrete id). So **opusplan's mainloop model is Sonnet** at resolution time; the opus-for-plan behavior is layered elsewhere (see the `En()` plan-mode branch below).
- `opus` resolves to `cj()`, and when `[1m]` was present it goes through `Tg()` which canonicalizes the suffix: `Tg(H)=H.replace(/(\[1m\])+$/i,"")+"[1m]"`.
- A non-alias concrete id (e.g. a raw `claude-opus-4-8`) falls through: if it had `[1m]` it is re-normalized to exactly one trailing `[1m]` (`..._.replace(/(\[1m\])+$/i,"").trim()+"[1m]"`); otherwise returned unchanged.

### Per-family concrete ids and the `ANTHROPIC_DEFAULT_*_MODEL` overrides

Each alias delegates to a family resolver that checks its env override first, then a catalog default `NT()`:

```js
function cj(){if(process.env.ANTHROPIC_DEFAULT_OPUS_MODEL)return process.env.ANTHROPIC_DEFAULT_OPUS_MODEL;return TvH()}
function $Z(){if(process.env.ANTHROPIC_DEFAULT_SONNET_MODEL)return process.env.ANTHROPIC_DEFAULT_SONNET_MODEL;return K16()}
function FOH(){if(process.env.ANTHROPIC_DEFAULT_HAIKU_MODEL)return process.env.ANTHROPIC_DEFAULT_HAIKU_MODEL;return U28()}
function Yt(){let H=process.env.ANTHROPIC_DEFAULT_FABLE_MODEL||B28();return Rn()?_2_(H):H}
```

The non-overridden defaults are provider-mode dependent (`S8()`), e.g. opus:

```js
function TvH(H=NT()){if(S8()==="mantle")return H.opus47;if(!AO())return H[IqH];if(S8()!=="firstParty")return H.opus47;return H.opus48}
function K16(H=NT()){if(!AO())return H[xqH];return H.sonnet46}
```

So `--model opus` on a non-first-party provider mode (Bedrock/Foundry/Mantle/Vertex, i.e. `S8()!=="firstParty"`) resolves to **opus47**, not opus48 — unless you set `ANTHROPIC_DEFAULT_OPUS_MODEL` to pin it. Note a plain CCR / `ANTHROPIC_BASE_URL` gateway does **not** change `S8()`; it stays `firstParty` unless a `CLAUDE_CODE_USE_*` flag is set. (`AO()=H==="firstParty"||H==="anthropicAws"||H==="gateway"` — the `"gateway"` branch is dead, since `S8()` only ever returns one of bedrock/foundry/anthropicAws/mantle/vertex/firstParty.)

There is a second opus override reader, `sX_`, used for the 1M-fallback decision; note it reads the **same** `ANTHROPIC_DEFAULT_OPUS_MODEL`:

```js
function sX_(H){let _=nH.ANTHROPIC_DEFAULT_OPUS_MODEL;if(_===void 0){let q=NT();if(_=q.opus48,S8()==="firstParty")_=nR9.map((K)=>q[K]).find((K)=>M4(K))??q.opus48}
if((Jf(H)||sm(H))&&!Jf(_)&&!c28(_))return _+"[1m]";return _}
```

`c28(H)=d28(T9(H))` lists ids that **cannot** take 1M context (`claude-3-*`, `claude-opus-4-0/4-1/4-5`, `claude-haiku-4-5`, …), so `[1m]` is only appended when the target id actually supports it.

### The `[1m]` (1M-context) suffix lifecycle

Detection and the global kill-switch:

```js
function QOH(){return z_(process.env.CLAUDE_CODE_DISABLE_1M_CONTEXT)}
function Jf(H){if(QOH())return!1;return/\[1m\]/i.test(H)}
function K1(H){return H.replace(/\[1m\]$/i,"")}
function Tg(H){return H.replace(/(\[1m\])+$/i,"")+"[1m]"}
```

- `Jf` = "does this string carry `[1m]`". If `CLAUDE_CODE_DISABLE_1M_CONTEXT` is truthy, `Jf` (and `sm`) **always return false**, so `[1m]` is never detected nor re-appended anywhere downstream — the suffix is effectively stripped from the whole resolution path.
- `[1m]` eligibility for first-party-only behaviors is gated by `Rn()=S8()==="firstParty"&&w3()` and `gM()` (false unless `S8()==="firstParty"`). The fable case in `f9` only appends `[1m]` when `!Rn()` and the resolved fable id doesn't already carry it.
- The opus-to-opus1m migration path rewrites stored settings (`tengu_opus_to_opus1m_migration`), and there are legacy fix-ups that map bare `sonnet[1m]` → `sonnet-4-5-20250929[1m]`.

### On-wire normalization: `T9` / `lw`

Before a model id is compared or displayed, it is canonicalized to a family token by `T9` → `lw`:

```js
function T9(H){let _=iiH(H);if(_!==H)return lw(_);if(H.includes("application-inference-profile")){let q=oY_(aO(H));if(q)return lw(q)}return lw(_)}
function lw(H){if(H=H.toLowerCase(),H.includes("claude-fable-5"))return"claude-fable-5";if(H.includes("claude-mythos-5"))return"claude-mythos-5";if(H.includes("claude-opus-4-8"))return"claude-opus-4-8"; … if(H.includes("claude-3-haiku"))return"claude-3-haiku";return H.replace(/-\d{8}$/,"")}
```

- `lw` does **substring `includes()` matching** against a fixed ordered list (fable-5, mythos-5, opus-4-8 → 4-7 → 4-6 …, sonnet, 3-7/3-5 variants). This is why a Bedrock/Vertex id like `us.anthropic.claude-opus-4-8-v1:0[1m]` still maps to `claude-opus-4-8` — and why the `[1m]` suffix does **not** break family detection (it's a substring match, and `lw` lowercases but doesn't strip `[1m]`).
- `iiH` first applies the user's `modelOverrides` map (reverse lookup: if a wire id equals an override value, the alias key is used). For `application-inference-profile` (Bedrock) ids it extracts the underlying model via `oY_(aO(H))`.
- Anything unmatched falls through to `H.replace(/-\d{8}$/,"")` — stripping a trailing date stamp. Capability checks like `sm`, `c28`, family equality, and the model display labels (`CqH`, e.g. `case"claude-opus-4-8":… + (endsWith("[1m]")?" (1M context)":"")`) all run through `T9`.

### Session-model selection: `En` / `c9` / `yA`

The active mainloop model id is chosen by precedence in `En`, then resolved by `c9`:

```js
function En(){let H,_=$f();if(_!==void 0)H=_;else{let q=XF();H=q!==void 0?q:process.env.ANTHROPIC_MODEL??rq()?.model??void 0}if(H&&!M4(H))return;return H}
function c9(){let H=En();if(H!==void 0&&H!==null)return f9(H);return yA()}
function yA(){return f9(CW())}
```

Precedence (highest first): `$f()` = `mainLoopModelOverride` (set by `/model` mid-session) → `XF()` = `initialMainLoopModel` → `process.env.ANTHROPIC_MODEL` → `rq()?.model` (config) → otherwise the baseline `CW()` (computed from `availableModels`/cascade allowlist via `T16`). `M4(H)` validates the candidate is a known/available model. Everything ultimately passes through `f9` so aliases and `[1m]` are resolved identically regardless of source.

There is a separate **plan-mode opusplan** branch (outside `f9`) that upgrades to Opus for plan mode:

```js
O=En();if((O==="opusplan"||O==="opusplan[1m]")&&_==="plan"&&!K){let $=O==="opusplan[1m]"||gM()?Tg(cj()):cj(); …
```

So `opusplan` runs Sonnet (`$Z()`) for normal turns but switches to Opus (`cj()`, with `[1m]` if the alias carried it or `gM()` is true) during plan mode.

### `CLAUDE_CODE_AUTO_MODE_MODEL`

This name is **registered** in the env-var schema table (`CLAUDE_CODE_AUTO_MODE_MODEL:()=>qB1`) and appears in a binary symbol-table region, but I found **zero** `process.env.CLAUDE_CODE_AUTO_MODE_MODEL` reads in executable JS and no inline consumer adjacent to model-resolution logic (`grep -aoc 'process.env.CLAUDE_CODE_AUTO_MODE_MODEL' → 0`). Its runtime effect could not be traced this session — treat it as a recognized-but-unverified knob (inferred: read via the registry accessor `qB1` rather than a direct env read).

### Operator notes

- **Pin concrete ids with `ANTHROPIC_DEFAULT_{OPUS,SONNET,HAIKU,FABLE}_MODEL`.** These are checked *first* in `cj()/$Z()/FOH()/Yt()`; without them the default depends on provider mode (`S8()`), and notably `opus` falls back to **opus47** on non-first-party providers, not opus48.
- **`opusplan` resolves to the Sonnet id** for normal turns (`$Z()`); Opus only kicks in during plan mode. Gateways tracking spend per model should expect Sonnet traffic for opusplan sessions outside plan mode.
- **`CLAUDE_CODE_DISABLE_1M_CONTEXT=1` globally erases `[1m]`** — `Jf`/`sm` return false, so the suffix is never detected, preserved, or appended. Use it to force non-1M routing regardless of the user's `--model …[1m]`.
- **`[1m]` survives family normalization.** `T9`/`lw` use substring `includes()`, so `claude-opus-4-8[1m]` and Bedrock/Vertex-prefixed ids still classify as `claude-opus-4-8`. Don't rely on `[1m]` being stripped before your gateway sees the model field — it is carried on the wire id.
- **Provider mode drives defaults.** `S8()` returns exactly one of `bedrock`/`foundry`/`anthropicAws`/`mantle`/`vertex`/`firstParty` (selected by the `CLAUDE_CODE_USE_*` flags) — there is **no** `gateway` mode (the `"gateway"` literal in `AO()` is a dead branch). A CCR / `ANTHROPIC_BASE_URL` proxy stays `firstParty` unless a `USE_*` flag is set. `Rn()`/`gM()` (which enable some `[1m]` behaviors) require strictly `firstParty` (or `_CLAUDE_CODE_ASSUME_FIRST_PARTY_BASE_URL`); a non-first-party mode uses different opus defaults and won't auto-append `[1m]` on the fable path.
- **Session model precedence:** `mainLoopModelOverride` (`/model`) > `initialMainLoopModel` > `ANTHROPIC_MODEL` > config `model` > computed baseline. All are funneled through `f9`, so any of these may be an alias or a concrete id.
- **`CLAUDE_CODE_AUTO_MODE_MODEL`** is a recognized env var but has no traceable resolution consumer in this build (unverified effect).
