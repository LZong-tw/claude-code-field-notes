# Claude Code 2.1.178 — Context-Window & Auto-Compact Teardown

Reverse-engineering notes on how **Claude Code v2.1.178** (the native macOS
binary) decides:

1. the **context window** it reports (`/200k` vs `/1M`), and
2. when it fires **auto-compaction**.

All findings below were extracted by `strings`/`grep` over the bundled minified
JS inside the native Mach-O executable, then **adversarially re-verified** (a
second independent pass reproduced every cited snippet). Function names are the
minified identifiers as they appear in the binary; they are stable references
for *this* version only and will change across releases.

> Scope: this documents observable behaviour of the public Claude Code product
> for the purpose of operating it correctly (model routing, budget tuning). No
> proprietary source, assets, or credentials are included.

---

## TL;DR

| Question | Answer |
|---|---|
| What enables the 1M context window? | The **resolved model string must end in the literal suffix `[1m]`** (e.g. `claude-opus-4-8[1m]`). `Jf(H) = /\[1m\]/i.test(H)`. |
| Does `ANTHROPIC_1M_CONTEXT` do anything? | **No.** The string does not appear in the binary (0 occurrences). It is a dead env var in this version. |
| Is there an opt-out? | Yes: `CLAUDE_CODE_DISABLE_1M_CONTEXT` (truthy → forces non-1M). |
| Will CC auto-append `[1m]` to my model? | **Almost never.** Only inside `sX_()` for `claude-fable-5` / `claude-mythos-5`, and only onto the *opus default* model — never onto a user-supplied `--model claude-sonnet-4-6`. |
| Why does auto-compact fire at ~30–40%? | Almost always an **override knob** — `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE=<pct>` forces compaction at that percentage. Otherwise a **shrunk `autoCompactWindow`** (env/setting, floored at 100000) lowers the trigger. The default 200K-window trigger is ~144K (72%), *not* 35%. |
| Does preloaded context (system + tools + skills + MCP) cause early compaction? | **Partly.** It raises the *baseline* token count so you cross the threshold sooner, but it does **not** lower the threshold (reserves are constants, not tool-scaled). |
| Does a 1M window stop early compaction? | **No.** A percentage trigger (`CLAUDE_AUTOCOMPACT_PCT_OVERRIDE`) scales with the window, so 1M compacts at the same % — just at a higher absolute token count. Fix the trigger, not the window. |

---

## 1. The 1M-context gate

### 1.1 The decision function `w37`

The context window for a turn is computed by `w37(H, _)` where `H` = the
resolved model string and `_` = the request's `anthropic-beta` header array:

```js
function w37(H,_){
  if(Jf(H))return 1e6;                         // [1m] suffix → 1,000,000  (FIRST, wins over everything)
  if(_?.includes(yn.header)&&$g(H))return 1e6; // beta header context-1m-2025-08-07 AND model allow-listed
  if(sm(H))return 1e6;                          // certain first-party/opus models
  let q=IY6(H);if(q!==null)return q;            // remote-tunable window for sonnet-4-6 (experiment)
  return _Z_;                                   // _Z_ = 200000 default
}
```

The **first branch `if(Jf(H)) return 1e6` is evaluated before any provider,
gateway, or auth check**. That is the robust, provider-independent lever.

### 1.2 `Jf` / `QOH` — the `[1m]` test

```js
function QOH(){ return z_(process.env.CLAUDE_CODE_DISABLE_1M_CONTEXT) }
function Jf(H){ if(QOH())return!1; return /\[1m\]/i.test(H) }
```

- 1M is enabled **iff** the model string contains `[1m]` (case-insensitive) and
  `CLAUDE_CODE_DISABLE_1M_CONTEXT` is not truthy.
- `z_()` is the env-truthiness helper; set the disable flag to an unset/empty/
  falsy value to keep 1M available.

`grep -aoc` counts in the binary:

```
ANTHROPIC_1M_CONTEXT            → 0    (dead — does nothing)
CLAUDE_CODE_DISABLE_1M_CONTEXT  → 3    (the real toggle)
context-1m-2025-08-07           → 4    (the beta header, see §1.4)
```

### 1.3 Where the tested string comes from — `f9` / `c9` / `T9`

The configured model (from `--model`, the persisted/restore model, etc.) is
resolved through `f9()` before `Jf` sees it:

```js
function c9(){ let H=En(); if(H!==void 0&&H!==null)return f9(H); return yA() }

function f9(H){
  let _=H.trim(), q=_.toLowerCase(), K=Jf(q), O=K?K1(q).trim():q;
  if(uN(O))switch(O){ case "sonnet": return $Z()+(K?"[1m]":""); /* ...aliases... */ }
  /* ... */
  if(K) return _.replace(/(\[1m\])+$/i,"").trim()+"[1m]"; // preserve [1m] IFF input had it
  return _;
}

function K1(H){ return H.replace(/\[1m\]$/i,"") }        // strip one trailing [1m]
```

Key consequence: **`f9` only keeps `[1m]` if the input already had it.** It never
adds it to a bare concrete id. So `--model claude-sonnet-4-6` stays
`claude-sonnet-4-6` (200K); `--model claude-sonnet-4-6[1m]` stays 1M.

The on-wire API model id is normalized by `T9`/`lw`, which use `includes()`:

```js
function lw(H){ /* ... */ if(H.includes("claude-opus-4-8"))return "claude-opus-4-8"; /* ... */ }
```

So `claude-sonnet-4-6[1m]` → upstream sees `claude-sonnet-4-6`. **`[1m]` is a
client-local marker; the API/gateway never needs to understand it.**

### 1.4 Second path — the beta header (situational)

```js
yn = SJ("long_context","context-1m-2025-08-07");
function SJ(H,_){ return Object.freeze({name:H, header:_}) }

function $g(H){
  if(QOH())return!1; if(c28(H))return!1;
  let _=T9(H);
  if(_==="claude-fable-5"|| /* ... */ ||_==="claude-sonnet-4-6"||_==="claude-sonnet-4-5"||_==="claude-sonnet-4-0")return!0;
  return $v(Mw(H));
}
```

`w37`'s second branch returns 1M if the request carries the
`context-1m-2025-08-07` beta header **and** the model is allow-listed (sonnet-4-6
is). But CC only emits that header for models it already treats as 1M — so the
`[1m]` suffix remains the dependable trigger.

### 1.5 The auto-append remap `sX_` (why you can't rely on it)

```js
function sX_(H){
  let _=nH.ANTHROPIC_DEFAULT_OPUS_MODEL; /* ... */
  if((Jf(H)||sm(H))&&!Jf(_)&&!c28(_)) return _+"[1m]";   // appends to the OPUS DEFAULT, not your model
}
function $oK(){ /* ... */ let H=c9(); if(dj(H)||aX_(H)) return sX_(H); return H }
function dj(H){ return K1(T9(H))==="claude-fable-5"||zg(H) }
function aX_(H){ return K1(T9(H))==="claude-mythos-5" }
function rX_(H){ return!1 }   // a related guard, hard-coded dead
```

`sX_` is only reached when the model is `claude-fable-5` / `claude-mythos-5`, and
it appends `[1m]` to the *opus default* model, not to your selection.

### 1.6 Remote-tunable window for plain `sonnet-4-6` — `IY6`

```js
function IY6(H){
  if(QOH())return null; if(Jf(H))return null;
  if(T9(H)!=="claude-sonnet-4-6")return null;
  let _=S_().clientDataCache?.kelp_forest_sonnet; /* ... */ return q;
}
```

For **plain** `claude-sonnet-4-6` (no `[1m]`), the window can be remotely tuned by
a `kelp_forest_sonnet` experiment value pushed into `clientDataCache`. Once `[1m]`
is present, `Jf(H)` short-circuits and this is ignored. Worth scrubbing if a
gateway forwards experiment payloads.

### 1.7 How to actually get 1M

Make the model string Claude Code resolves **literally end in `[1m]`**, e.g.
`claude-sonnet-4-6[1m]`, injected where CC reads the model (`c9()→En()`: the
persisted/restore model and/or `--model`). The upstream API id stays clean. This
works through any provider/gateway because `if(Jf(H))return 1e6` precedes all
provider logic.

### 1.8 The per-session 1M rate-limit latch (verified in 2.1.179)

In **2.1.179** the gate is the same logic with renamed identifiers — `w37`→`dO7`,
`Jf`→`jf`, `QOH`→`xOH` (still only `CLAUDE_CODE_DISABLE_1M_CONTEXT`), and the
auto-1M model set is `uS` — but the window is now read through a wrapper `xM`
that applies a **latch cap before the model is ever consulted**:

```js
function xM(H,_){ let q=cO7(); if(q!==void 0) return q;   // explicit override (rare)
                  if(FV8(H,_)) return It;                 // It = 200000 — the latch cap
                  return dO7(H,_) }                        // only now: the model-based 1M gate
function FV8(H,_){ return dQH() && cO7()===void 0 && dO7(H,_) > It }
function dQH(){ return m_.longContext1mCreditsBlocked }   // a per-session in-memory flag
```

So when `dQH()` is true, **any** model that would resolve to >200K (including a
`[1m]` suffix or a `uS` model) is hard-capped to exactly `It = 200000`. The flag
is latched true inside the API **rate_limit** error path:

```js
...error:"rate_limit"}) } if(T && bU6(H.message) && !dQH()) MH8(!0)   // MH8(x){ m_.longContext1mCreditsBlocked = x }
```

i.e. when the server returns a rate-limit response whose message matches `bU6`
(the 1M-long-context allowance signal), CC sets `longContext1mCreditsBlocked` for
the rest of that session. Properties:

- **Per-session, in-memory (`m_`).** It never clears within the session; a *fresh*
  session starts un-latched (1M again).
- **Applied before model resolution**, so switching models with `/model` does
  nothing — the cap precedes `dO7`.
- **Precise 200K**, readable by a statusline that reports `context_window`.

**Fingerprint:** same model + same account + same config, yet one live session is
1M and another is 200K → it can only be this latch, nothing in the model/config
layer. There is no client toggle to un-block a latched session; the lever is the
account's 1M-context usage tier (1M context has its own tighter rate limit,
separate from normal usage). Do not confuse this with the `[1m]`-suffix loss on
plain `--resume` of a session whose persisted model is a non-1M id (a different
200K path — that one *is* fixable by switching to a `uS` model or re-applying the
suffix; the latch is not).

---

## 2. The auto-compact threshold

### 2.1 Effective window → reserve → trigger

```js
// window:
function QM(H,_){ let q=A37(); if(q!==void 0)return q; if(zy8(H,_))return ct; return w37(H,_) }
// nc()/model-default path caps non-[1m] sonnet-4-6 / opus-4-6 to ct = 200000

// constants:
_Z_ = 200000;  ct = 200000;  CaK = 20000;  Gzq = 0.2;   // (plus qZ_=20000, gd5=32000, Qd5=128000)

// reserve & thresholds:
//   reserve = min(maxOutputTokens, CaK=20000)      → 20000 in practice
//   T       = L1H(model) = window - reserve

function qB6(H,_){                                  // H = T
  let q = H-13000, K = _.testPctOverride;
  if(K!==void 0 && !isNaN(K) && K>0 && K<=100)
    return Math.min(Math.floor(H*(K/100)), q);      // OVERRIDE: a PERCENTAGE of T (scales with window)
  return q;                                          // default: T - 13000
}
function Rzq(H,_){ return Math.min(H - Math.round(H*_.precomputeBufferFraction), qB6(H,_)) } // Gzq = 0.2
```

> **Critical:** `testPctOverride` (from `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE`) feeds
> `qB6` as a **percentage of T**, so it scales with the window. Since a small
> percentage (e.g. 35%) is below the 0.2 buffer's effective 80%, the override
> *wins* and a **larger window does NOT escape early compaction** — it only moves
> the absolute trigger up proportionally (see §2.2).

Two arms:

- **Arm A (precompute, `aiK`):** compact when `currentTokens H >= Rzq(T)`.
- **Arm B (legacy/blocking, `aBH→yaK`):** the `compact` level at `H >= qB6(T) = T - 13000`.

### 2.2 Worked numbers

**True 200,000 window:**

```
T   = 200000 - 20000 = 180000
Rzq = min(180000 - round(180000*0.2), 180000 - 13000)
    = min(144000, 167000) = 144000
→ Arm A fires at 144,000 tokens  (72% of 200K)
→ Arm B at 167,000               (83%)
```

So a **default** 200K session compacts near **72%**, *not* 35%.

**With `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE=35` (the percentage knob) — note it scales with the window, so 1M does NOT escape:**

```
qB6(T) = min(floor(T*0.35), T-13000)
Rzq(T) = min(T-round(T*0.2), qB6(T))

200K window:  T=180000 → Rzq = min(144000, 63000)  = 63000   (~31.5% of 200K)
1M   window:  T=980000 → Rzq = min(784000, 343000) = 343000  (~34%   of 1M)
```

Because `0.35 < 0.8` (the buffer's effective fraction), the override always wins.
A bigger window only raises the *absolute* trigger (63K → 343K); the percentage
stays the same. **Removing the override — not switching to 1M — is what restores
the normal ~72% / ~78% trigger.**

**Floored `autoCompactWindow = 100000` (no pct override):**

```
T   = 100000 - 20000 = 80000
Rzq = min(80000 - 16000, 80000 - 13000) = min(64000, 67000) = 64000
→ fires at ~64–67K  (≈32% of a 200K display)
```

### 2.3 Precedence of the window used (`nc()`)

```
env CLAUDE_CODE_AUTO_COMPACT_WINDOW (clamped 1e5..1e6)
  > autoCompactWindow setting
    > model-default 200000
      > auto
```

### 2.4 The override that explains "~35%"

```js
function vzq(){
  let H=process.env.CLAUDE_AUTOCOMPACT_PCT_OVERRIDE,
      _=process.env.CLAUDE_CODE_BLOCKING_LIMIT_OVERRIDE;
  return { enabled:qh(), precomputeBufferFraction:wCO(),
           testPctOverride: H?parseFloat(H):void 0, /* ... */ };
}
```

`CLAUDE_AUTOCOMPACT_PCT_OVERRIDE=<n>` forces compaction at **n %** of the window,
independent of model and window size. A value like `35` reproduces "compacts at
30–40%" on **every** session — including a real 200K Sonnet — which is the
fingerprint that distinguishes this knob from a window/model issue.

> Practical lesson: if you see early compaction on *both* a routed/cheap model
> and a real first-party model, suspect a global `env` override
> (`CLAUDE_AUTOCOMPACT_PCT_OVERRIDE` / `CLAUDE_CODE_AUTO_COMPACT_WINDOW`) before
> blaming the model, the gateway, or preloaded context.

---

## 3. Does preloaded context cause early compaction?

**Partly — it's a baseline inflator, not a threshold-lowerer.**

- Current-context counting (`ae`) sums `input + cache_creation + cache_read +
  output` tokens, so the system prompt, **every tool schema**, loaded skills, and
  MCP tool definitions are all permanent input that raises the baseline.
- A heavily-loaded session (many skills + MCP servers) therefore reaches the same
  threshold **sooner**.
- **But** the threshold itself does not scale with tools: the reserves are
  constants (`qB6` subtracts 13000; `L1H` subtracts `CaK=20000`). There is no
  tool/skill-proportional reserve.

So preload contributes, but the dominant lever for a "~35%" symptom is the
override / shrunk window, not the preload.

Mitigation for the baseline: keep the active tool surface small, and prefer
deferred tool loading (`isToolSearchEnabled` / `isDeferredTool`) so tool schemas
stay out of the token count until actually fetched.

---

## 4. Operator cheat-sheet

**Get 1M context**
- Make the resolved model string end in `[1m]` (e.g. `claude-sonnet-4-6[1m]`).
- Ensure `CLAUDE_CODE_DISABLE_1M_CONTEXT` is unset/falsy.
- `ANTHROPIC_1M_CONTEXT` does nothing — don't bother.

**Stop premature auto-compaction** (this is independent of the window size)
- **Remove `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE`** — it forces compaction at *that
  percentage of the window*, so it fires early on **every** window size. This is
  the usual culprit when a session compacts at ~30–40% on *both* a 200K and a 1M
  model.
- Don't set `autoCompactWindow` / `CLAUDE_CODE_AUTO_COMPACT_WINDOW` near the
  100000 floor; leave unset for the 200000 default.
- Scrub any gateway-forwarded `kelp_forest_sonnet` experiment value.
- Disable entirely (last resort): `DISABLE_AUTO_COMPACT=1`, `autoCompactEnabled=false`, or `DISABLE_COMPACT` (`A37`/`qh` gate).

> **Switching to 1M does NOT stop early compaction by itself.** The percentage
> override (and any percentage-based trigger) scales with the window, so a 1M
> model still compacts at the same percentage — just at a higher absolute token
> count. 1M only fixes the `/1M` display and gives more absolute headroom; the
> compaction *timing* is governed by the override / threshold, not the window.

**Other knobs seen in the binary**
- `precomputeBufferFraction` default `0.2` (gate `tengu_amber_rokovoko`).
- `CLAUDE_CODE_BLOCKING_LIMIT_OVERRIDE`, `CLAUDE_CODE_MAX_OUTPUT_TOKENS`.
- Truthful `usage` matters: zeroed usage → the counter (`Jr()`) reads 0 and compaction never fires; inflated `cache_read` fires it early.

---

## Method & caveats

- Extraction: `grep -aoE ".{N}TOKEN.{M}" <binary>` over the native executable;
  every claim above is backed by a verbatim minified snippet.
- Adversarial pass reproduced: `Jf`/`QOH`, `w37`, `f9` (head+tail), `Rzq`, `qB6`,
  `L1H`, `CaK=20000`, `Gzq=0.2`, the `$g`/`$CO` allow-list, the model-default cap,
  the `nc` env branch, the `autoCompactWindow` zod schema (`min 1e5`, `max 1e6`),
  `ae` usage summation, `ANTHROPIC_BETAS` passthrough, `z_` truthiness, the
  `A37` `DISABLE_COMPACT` gate, and the counts `ANTHROPIC_1M_CONTEXT=0` /
  `DISABLE_1M=3`.
- Not pinned from strings alone: the exact config key/path that persists the
  restore model (it is whatever feeds `En()`/`c9()`); and live confirmation that
  CC emits the `context-1m-2025-08-07` header on the wire for a `[1m]` model
  (inferred from `w37` + `$g`, not observed in a captured request).
- Minified identifiers are version-specific to **2.1.178**.
