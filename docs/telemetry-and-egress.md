## Telemetry & Egress / Kill Switches

Ground truth: the Claude Code v2.1.178 native bundle. Minified identifiers below are stable for this version.

### The master kill switch and the three-tier traffic mode

There is a single helper, `Iiq()`, that collapses three environment variables into a traffic posture. Everything else keys off it.

```
function Iiq(){if(process.env.CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC)return"essential-traffic";if(process.env.DISABLE_TELEMETRY)return"no-telemetry";if(z_(process.env.DO_NOT_TRACK))return"no-telemetry";return"default"}
function KK(){return Iiq()==="essential-traffic"}function FVH(){return Iiq()!=="default"}
```

- `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` (any truthy value) → `"essential-traffic"`. This is the **broadest** switch: `KK()` returns true, which short-circuits a large set of network calls (observed: 23 distinct `if(KK())return` guard sites). `GlH()` reports it as the named reason.
- `DISABLE_TELEMETRY` (truthy) **or** `DO_NOT_TRACK` (parsed by `z_`) → `"no-telemetry"`. These set `FVH()` true but **not** `KK()`. So they kill analytics/feature-eval traffic but leave other "essential" egress (e.g. auth, the actual model API) alone.
- `FVH()` true is what suppresses analytics event recording and feature-gate evaluation:

```
async function QPq(){if(KK()||FVH())return"skipped_privacy"
```

Observed precedence: `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` is strictly stronger than `DISABLE_TELEMETRY`/`DO_NOT_TRACK` (it gates the union of both, via `KK()` ⊂ `FVH()`).

### Egress #1 — Analytics / "tengu" events (first-party event logging)

Analytics events (the `tengu_*` event family) batch-upload to the **first-party API**, not to statsig.net. The endpoint is derived from `ANTHROPIC_BASE_URL`, so a gateway/proxy base URL redirects it:

```
process.env.ANTHROPIC_BASE_URL==="https://api-staging.anthropic.com"?"https://api-staging.anthropic.com":"https://api.anthropic.com");this.endpoint=`${_}${H.path||"/api/event_logging/v2/batch"}`
```

Statsig is **bundled in-process** (a local `statsig` state directory under the config dir alongside `todos`/`logs`), with **no direct `statsig.net`/`featuregates.org`/`featureassets.org` egress** (grep for those hosts returns nothing). Feature flags resolve from `S_().clientDataCache` (server-pushed cache), e.g. `S_().clientDataCache?.[H]===!0`.

- **Kill switch:** `DISABLE_TELEMETRY` / `DO_NOT_TRACK` (via `FVH()` → `"skipped_privacy"`), and of course `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC`.

### Egress #2 — Datadog RUM/event telemetry (first-party accounts only)

A Datadog event sink exists (`trackDatadogEvent`/`XsH`, plus `shutdownDatadog`), hosts `https://api.datadoghq.com` and `https://http-intake.logs.us5.datadoghq.com/api/v...`. It is hard-gated to first-party auth and a fixed event allowlist:

```
function XsH(H,_){if(S8()!=="firstParty")return;...if(!q||!Gn5.has(H))return;
Gn5=new Set(["tengu_feature_ok","tengu_feature_bad","tengu_feature_sad","chrome_bridge_connection_succeeded","chrome_bridge_connection_failed","chrome_bridge_disconnected"...])
```

- **Never fires** under Bedrock/Vertex/Foundry/Mantle/CCR-via-non-first-party because `S8()!=="firstParty"` returns early. So gateway/CCR operators on a non-first-party provider mode get zero Datadog traffic regardless of switches.

### Egress #3 — In-process error reporting

Uncaught errors are reported via `SQ1`/`uiq`. Gated off for cloud providers and by the error switch and the master switch:

```
...z_(process.env.CLAUDE_CODE_USE_MANTLE)||process.env.DISABLE_ERROR_REPORTING||KK())return;let K={error:_.stack||_.message,timestamp:new Date().toISOString()};
```

- **Kill switches:** `DISABLE_ERROR_REPORTING`, or `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` (`KK()`), or being on Bedrock-AWS / Mantle (auto-suppressed). There is **no first-party Sentry**: the only sentry host in the bundle is `https://mcp.sentry.dev`, which is the optional Sentry MCP/plugin, not Claude Code's own crash pipeline. `https://bun.report` (`BUN_ENABLE_CRASH_REPORTER`/`BUN_CRASH_REPORT_URL`) is the Bun runtime's native crash reporter, not Claude Code application code.

### Egress #4 — OpenTelemetry (operator-owned OTLP pipeline, opt-in)

OTEL is **off by default** and opt-in via `CLAUDE_CODE_ENABLE_TELEMETRY`:

```
function ZDK(){return z_(process.env.CLAUDE_CODE_ENABLE_TELEMETRY)}
...isTelemetryEnabled=${_} (CLAUDE_CODE_ENABLE_TELEMETRY=${process.env.CLAUDE_CODE_ENABLE_TELEMETRY})
```

Standard OTLP env vars are honored for routing to an operator's own collector (so this egress goes wherever the operator points it, not to Anthropic): `OTEL_METRICS_EXPORTER`, `OTEL_LOGS_EXPORTER`, `OTEL_EXPORTER_OTLP_ENDPOINT`, `OTEL_EXPORTER_OTLP_METRICS_ENDPOINT`, `OTEL_EXPORTER_OTLP_LOGS_ENDPOINT`, `OTEL_EXPORTER_OTLP_TRACES_ENDPOINT`, `OTEL_EXPORTER_OTLP_HEADERS`, `OTEL_EXPORTER_OTLP_PROTOCOL`, `OTEL_RESOURCE_ATTRIBUTES`, `OTEL_LOG_USER_PROMPTS`, and the `*_EXPORT_INTERVAL` / `OTEL_METRICS_INCLUDE_*` knobs. An additional enterprise/team auto-on path exists for metrics:

```
function GDK(){if(FVH())return!1;let H=fK(),_=Lq()&&(H==="enterprise"||H==="team");return IXH()||_}
function IXH(){if(!R1())return!1;if(Lq())return!1;return!0}    // R1()===firstParty
```

So even with metrics enabled, `GDK()` returns false when `FVH()` is true — meaning `DISABLE_TELEMETRY`/`DO_NOT_TRACK`/`CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` also suppress the auto-on enterprise OTEL metric reader.

### Egress #5 — Trace-context propagation (TRACEPARENT)

Outbound model requests can carry a W3C `traceparent` header. Default behavior plus an explicit opt-in:

```
function wX_(){return w3()||z_(process.env.CLAUDE_CODE_PROPAGATE_TRACEPARENT)}
...T=H&&(K||z_(process.env.CLAUDE_CODE_PROPAGATE_TRACEPARENT))?Pl8(H):void 0
```

`TRACEPARENT` (and `TRACESTATE`) are read from the inherited environment; `CLAUDE_CODE_PROPAGATE_TRACEPARENT` forces propagation onto API requests. Relevant for gateway operators who strip/inspect headers.

### Egress #6 — GrowthBook feature flags (Remote Control)

A GrowthBook client points at `https://cdn.growthbook.io`:

```
function iFq(H){let _=H.apiHost||"https://cdn.growthbook.io";return{apiHost:_.replace(/\/*$/,"")...
```

- **Kill switch:** `DISABLE_GROWTHBOOK`. Observed user-facing gate: `"Remote Control requires feature-flags ... because ${_} is set. Unset it ... if(nH.DISABLE_GROWTHBOOK)return"Remote Control requires..."`. This fetch is positively tied to the Remote Control / RemoteTrigger path; whether it also runs at plain startup was not fully traced (see uncertainties).

### Egress #7 — Auto-updater

Update checks/downloads hit `https://downloads.claude.ai/claude-code-releases` (and `https://claude.ai/download`). Disable precedence is explicit:

```
if(nH.DISABLE_UPDATES)return{type:"env",envVar:"DISABLE_UPDATES"};if(nH.DISABLE_AUTOUPDATER)return{type:"env",envVar:"DISABLE_AUTOUPDATER"};let H=GlH();if(H)return{type:"env",envVar:H};
```

- **Kill switches (in order):** `DISABLE_UPDATES` → `DISABLE_AUTOUPDATER` → `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` (`GlH()`), then `settings.json autoUpdates:false`. Note the migration path that sets `DISABLE_AUTOUPDATER:"1"` and `nH.set("DISABLE_AUTOUPDATER",!0)` when migrating auto-update config.

### Egress #8 — Interactive commands that phone home

`/bug` and `/feedback` post to first-party endpoints and are individually gateable:

```
function we_(){... } // /feedback disabled reasons:
"/feedback has been disabled via the CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC environment variable"
...if(nH.DISABLE_BUG_COMMAND)return{kind:"disabled",reason:"/feedback has been disabled..."}
...DISABLE_FEEDBACK_COMMAND||nH.DISABLE_BUG_COMMAND
```

- **Kill switches:** `DISABLE_BUG_COMMAND`, `DISABLE_FEEDBACK_COMMAND`, or the master `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC`. Projects/feedback APIs also surface a `policy_disabled` reason for HIPAA/compliance orgs ("Projects is disabled for this organization by compliance policy").

### Provider-mode note

Several egress paths self-suppress by provider mode via `S8()` (returns one of `bedrock, foundry, anthropicAws, mantle, vertex, firstParty` — no `gateway`). Datadog requires `firstParty`; error reporting suppresses on Bedrock-AWS/Mantle. A gateway/CCR is a base-URL concept (`ANTHROPIC_BASE_URL`), so analytics (`/api/event_logging`) follows the redirected base URL rather than being a separate provider mode.

### Operator notes

- **One switch to kill almost everything: `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC=1`.** It is the superset — gates analytics, error reporting, enterprise OTEL auto-on, GrowthBook, auto-update, `/bug`, `/feedback`, and ~23 other guarded calls. Locked-down networks should set this first.
- **`DISABLE_TELEMETRY=1` (or `DO_NOT_TRACK=1`) is narrower:** it stops analytics/feature-eval and the auto-on OTEL metric reader, but does **not** by itself stop error reporting, the auto-updater, or `/bug`. Add `DISABLE_ERROR_REPORTING=1`, `DISABLE_AUTOUPDATER=1` (or `DISABLE_UPDATES=1`), and `DISABLE_GROWTHBOOK=1` for a fuller lockdown without the master switch.
- **No direct statsig.net / featuregates.org / featureassets.org egress.** Statsig is in-process; flags arrive in `clientDataCache` and analytics ship to `api.anthropic.com/api/event_logging/v2/batch`. A gateway that intercepts `ANTHROPIC_BASE_URL` will also see these analytics batches — budget/firewall accordingly.
- **Datadog telemetry only exists on first-party auth.** Operators running through Bedrock/Vertex/Foundry/Mantle (or a non-first-party gateway mode) get zero Datadog egress with no extra config.
- **OTEL is opt-in and operator-routed.** `CLAUDE_CODE_ENABLE_TELEMETRY=1` + `OTEL_EXPORTER_OTLP_ENDPOINT` sends to *your* collector, not Anthropic. Use `OTEL_EXPORTER_OTLP_HEADERS` for auth; `OTEL_LOG_USER_PROMPTS` controls whether prompt text is included.
- **Trace headers leak context to the API.** If your environment sets `TRACEPARENT` or you set `CLAUDE_CODE_PROPAGATE_TRACEPARENT=1`, that header rides on model requests; strip it at the gateway if you don't want correlation IDs leaving.
- **Auto-updater hits `downloads.claude.ai`.** Air-gapped installs must set `DISABLE_AUTOUPDATER=1` (or `DISABLE_UPDATES=1`); the master traffic switch also covers it. Watch for the config migration that persists `DISABLE_AUTOUPDATER` into settings.
- **Crash reporting is dual-origin:** Claude Code's own error reporting (gateable via `DISABLE_ERROR_REPORTING`) is separate from the Bun runtime's `bun.report` crash reporter — disable the latter with `BUN_ENABLE_CRASH_REPORTER=0` / `BUN_CRASH_REPORT_URL` if you need belt-and-suspenders. There is no first-party Sentry (the only sentry host is the optional MCP plugin).
- **`mcp.sentry.dev`, `cdn.growthbook.io`, `app.corridor.dev`, `api.datadoghq.com`** are the non-Anthropic telemetry/feature hosts to allow or block at the firewall depending on which features you keep.
