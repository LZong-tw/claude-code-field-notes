## Settings Precedence & Config Resolution (Claude Code v2.1.178)

Claude Code resolves configuration by **deep-merging up to five named settings sources in a fixed order, where later sources override earlier ones (last-wins)**. The canonical order array is `YP`, and every consumer (the disk merge, the per-key source attribution, and the `env`-block application) iterates that same order.

### The five sources and the master order array

The authoritative ordered list is `YP`. The enabled set always force-adds `flagSettings` and `policySettings`, then filters `YP`, so the relative order is always preserved:

```
YP=["userSettings","projectSettings","localSettings","flagSettings","policySettings"]
```
```
function VN(){let H=mY_();...let _=new Set(H);_.add("flagSettings"),_.add("policySettings");let q=YP.filter((K)=>_.has(K));...}
```

Source → human label / file location mapping (`q_6`/`aJH`/`pm`/`al`):

```
case"userSettings":return"user";case"projectSettings":return"project";case"localSettings":return"project, gitignored";case"flagSettings":return"cli flag";case"policySettings":return"managed"
```
```
case"projectSettings":return Hv.join(".claude","settings.json");case"localSettings":return Hv.join(".claude","settings.local.json")
```
```
case"userSettings":return Hv.resolve($8()) ... case"policySettings":return fi1()
```

So the cascade, lowest-precedence first, is:

| Order | Source key | File / origin | Precedence |
|---|---|---|---|
| 1 (lowest) | `userSettings` | `~/.claude/settings.json` (`$8()` = user dir) | weakest |
| 2 | `projectSettings` | `<cwd>/.claude/settings.json` (checked in) | |
| 3 | `localSettings` | `<cwd>/.claude/settings.local.json` (gitignored) | |
| 4 | `flagSettings` | `--settings <path>` CLI flag | |
| 5 (highest) | `policySettings` | managed-settings.json + managed-settings.d (MDM) | **wins** |

Note there is a separate `globalConfig` (`~/.claude.json`, read via `S_()`) layer that supplies `env` (and other top-level config) applied *before* any settings source — see env section.

### The deep merge is last-wins (policy overrides everything)

`Vz8` is the on-disk loader. It walks `XD_(H)` (which is `YP` filtered, same order) and folds each source's settings into accumulator `K` via the `Us` deep-merge. Because user is folded first and policy last, **policy wins every conflicting key**:

```
for(let Y of XD_(H)){if(Y==="policySettings"){...if(w)K=Us(K,w,TqH);...continue}let A=aJH(Y,H);...if(f)K=Us(K,f,TqH)}
```
```
function XD_(H){let _=new Set(H.allowedSources);return _.add("flagSettings"),_.add("policySettings");return YP.filter((q)=>_.has(q))}
```

Per-key source attribution confirms last-wins: `$OH` scans the order array **from the end backwards** and returns the last source that defined a key:

```
function $OH(H){let _=VN();for(let q=_.length-1;q>=0;q--){let K=_[q];if(I6(K)?.[H]!==void 0)return K}return null}
```

The override-explanation strings make the hierarchy explicit to users:

```
case"flagSettings":return`Remove "${H.source}" from the --settings value — that flag overrides all settings files`;case"policySettings":return"Managed policy can't be overridden locally — contact your administrator"
```

### Managed (policy) settings: OS path + managed-settings.d

`policySettings` is loaded by `Leq`/`kz8` from a per-OS managed directory (`GW()`), reading `managed-settings.json` plus a `managed-settings.d/` drop-in directory of `*.json` files (dotfiles skipped):

```
GW=V6(function(){switch(a_()){case"macos":return"/Library/Application Support/ClaudeCode"...
```
(other OS branches: `/etc/claude-code`, Windows `ProgramData`.)
```
function fi1(){return Hv.join(GW(),"managed-settings.json")}
```
```
let{settings:q}=QF(AqH.join(_,"managed-settings.json"),...);...AqH.join(_,"managed-settings.d");z=n_().readdirSync($).some((Y)=>{if(!(Y.isFile()||Y.isSymbolicLink())||!Y.name.endsWith(".json")||Y.name.startsWith("."))return!1...
```

Managed source carries policy-only fields like `allowManagedPermissionRulesOnly` and `forceLoginOrgUUID` (`kz8`), and when a policy source exists but fails to parse the runtime **refuses to fall through** rather than silently dropping policy:

```
enforceAvailableModels: a policy source exists but failed to load; refusing cascade-tru...
```

### How the `env` block is applied

The `env` map from settings is pushed into `process.env`. `Na()` applies it in the **same precedence order** — global config first, then each settings source via `VN()` (user→project→local→flag→policy), each `Object.assign` overriding the prior, so the highest-precedence source's `env` wins:

```
function Na(){...Object.assign(process.env,$z_(S_().env,"globalConfig"));for(let H of VN())Object.assign(process.env,$z_(I6(H)?.env,H));...}
```

`$z_` runs each `env` block through a filter chain `_wT(HwT(eAT(sAT(...))))`:

- **`sAT`** — if `ANTHROPIC_UNIX_SOCKET` is set, it **strips connection-overriding keys** from any settings `env` block so they cannot redirect the endpoint:
```
function sAT(H){if(!H||!process.env.ANTHROPIC_UNIX_SOCKET)return H||{};let{ANTHROPIC_UNIX_SOCKET:_,ANTHROPIC_BASE_URL:q,ANTHROPIC_API_KEY:K,ANTHROPIC_AUTH_TOKEN:O,CLAUDE_CODE_OAUTH_TOKEN:T,...z}=H;return z}
```
- **`eAT`** — under host-managed mode, drops keys matched by `t07`/`e07` (host-restricted env names):
```
function eAT(H,_){if(!H)return{};if(!(Eg_.managedByHost||Eg_.desktopHost&&tAT.has(_)))return H;...for(...){if(t07(O))continue;if(Eg_.managedByHostFlag&&e07(O))continue;K[O]=T}...}
```
- **`HwT`** — won't overwrite env names that pre-existed in `process.env` at startup snapshot (`qi6`), and **`_wT`** special-cases `NO_COLOR`/`FORCE_COLOR`.

A second applier `mFH` exists that uses a reduced source set `qwT=["userSettings","flagSettings","policySettings"]` and **skips `policySettings`** for that pass:
```
for(let H of qwT){if(H==="policySettings")continue;if(!Dw(H))continue;Object.assign(process.env,$z_(I6(H)?.env,H))...
```

### Editing constraints (which sources are user-writable)

The user-editable/known source enum is `["userSettings","projectSettings","localSettings","flagSettings","policySettings"]`, but writes target only user/project/local; `policySettings` is read-only (managed). The runtime also blocks editing the settings files themselves and reports invalid JSON per-source rather than crashing:

```
~/.claude/settings.json, .claude/settings.json, and .claude/settings.local.json are blocked.
```
```
updateSettingsForSource: invalid JSON in settings file
```

### Operator notes

- **Precedence (lowest→highest):** user (`~/.claude/settings.json`) → project (`.claude/settings.json`) → local (`.claude/settings.local.json`) → `--settings` flag → **managed policy**. Deep-merge, last-wins per key — managed policy overrides everything, the `--settings` flag overrides all on-disk files.
- **Managed/MDM settings** live in an OS dir (macOS `/Library/Application Support/ClaudeCode/managed-settings.json`, Linux `/etc/claude-code`, Windows `ProgramData`), plus a `managed-settings.d/*.json` drop-in dir. You cannot override them locally; a malformed policy file makes the runtime refuse to cascade rather than silently ignore policy.
- **`env` blocks merge in the same precedence order** — a policy/flag `env` value beats a project/user one. Global `~/.claude.json` env is applied first (weakest).
- **Endpoint-redirect env keys in settings can be neutralized:** when `ANTHROPIC_UNIX_SOCKET` is set, `ANTHROPIC_BASE_URL`, `ANTHROPIC_API_KEY`, `ANTHROPIC_AUTH_TOKEN`, `CLAUDE_CODE_OAUTH_TOKEN` are stripped from settings `env` blocks. If your gateway relies on setting `ANTHROPIC_BASE_URL` via settings `env`, do not also set a unix socket — the socket wins and your base-URL override is silently dropped.
- **Settings `env` does not override pre-existing process env** (`HwT`): if a var was already in the launching shell, the settings `env` value is skipped. Set it in the shell OR in settings, not expecting settings to win over an already-exported var.
- **The settings files are write-protected by the runtime itself** (and by your Clawback hooks); the per-key "from (source)" attribution comes from `$OH`, so `/config`-style displays will tell you exactly which layer set a given key.
- **(Inferred, not directly traced):** the `mFH` env pass skipping `policySettings` and using the reduced `qwT` set appears to be an early/secondary bootstrap applier; the authoritative full-order applier is `Na()`. Treat `Na()`'s order as the ground truth for runtime env precedence.
