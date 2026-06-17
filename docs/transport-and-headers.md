## Transport and Headers (Claude Code v2.1.178)

All evidence is verbatim from the 156 MB minified bundle. Minified identifiers are stable for this version.

### Beta registry -- SJ() objects

Every beta is a frozen {name, header} pair built by SJ(); SW() projects a list to header strings; FS9 is a header->object map.

```
function SJ(H,_){return Object.freeze({name:H,header:_})}
function SW(H){return H.map((_)=>_.header)}
```

Full v2.1.178 registry (observed, single block):

```
ayH=SJ("claude_code","claude-code-20250219"), jMH=SJ("oauth_auth",yG),
frH=SJ("interleaved_thinking","interleaved-thinking-2025-05-14"),
yn=SJ("long_context","context-1m-2025-08-07"),
syH=SJ("context_management","context-management-2025-06-27"),
$t=SJ("structured_outputs","structured-outputs-2025-12-15"),
UX_=SJ("web_search","web-search-2025-03-05"),
FX_=SJ("tool_search","tool-search-tool-2025-10-19"),
jrH=SJ("effort","effort-2025-11-24"),
JrH=SJ("prompt_caching_scope","prompt-caching-scope-2026-01-05"),
gX_=SJ("redact_thinking","redact-thinking-2026-02-12"),
t46=SJ("thinking_token_count","thinking-token-count-2026-05-13"),
vn=SJ("mid_conversation_system","mid-conversation-system-2026-04-07")
```

(plus extended-cache-ttl, fast-mode, summarize-connector-text, afk-mode, advisor-tool, cache-diagnosis, context-hint, mcp-servers-2025-12-04, files-api, environments, ccr-byoc-2025-07-29, server-side-fallback, fallback-credit.)

### Per-request beta selection -- jy8()

Decides which betas go on each request. Model- and provider-aware (S8()/Mw() = provider mode, T9() = model id):

```
jy8=V6((H)=>{let _=[],q=T9(H).includes("haiku"),K=S8(),O=FN();
  if(!q)_.push(ayH);
  if(Lq()||fy8()&&!jX_()&&QD())_.push(jMH);
  if(Jf(H))_.push(yn);
  if(!z_(process.env.DISABLE_INTERLEAVED_THINKING)&&KZ_(H))_.push(frH);
  ... if($v(Mw(H))&&!DSH()&&(T||z))_.push(syH);
  ... if($v(Mw(H))&&!DSH()&&MSH(H)&&$)_.push($t);
  if(K==="vertex"&&ld5(H))_.push(UX_); if(K==="foundry")_.push(UX_);
  if(O)_.push(JrH); if(mY6(H))_.push(vn);
  if(process.env.ANTHROPIC_BETAS)_.push(...process.env.ANTHROPIC_BETAS.split(",")...map(k28));
  return _})
```

Observed rules:
- claude-code-20250219 (ayH): every non-Haiku model.
- oauth_auth (jMH): when OAuth-logged-in (Lq()).
- context-1m-2025-08-07 (yn, long_context): gated only on the model id matching [1m]:

```
function Jf(H){if(QOH())return!1;return/\[1m\]/i.test(H)}
```

- interleaved-thinking-2025-05-14 (frH): sent unless DISABLE_INTERLEAVED_THINKING; KZ_ disables it for claude-haiku-4-5 and claude-3-*, force-on for foundry.
- context-management-2025-06-27 (syH) and structured-outputs-2025-12-15 ($t): require provider support ($v(Mw(H))) AND !DSH() (experimental not disabled).
- web-search-2025-03-05 (UX_): only vertex (conditional) and foundry.

### Provider mode S8() (canonical, no gateway)

```
function S8(){return z_(process.env.CLAUDE_CODE_USE_BEDROCK)?"bedrock":
  z_(process.env.CLAUDE_CODE_USE_FOUNDRY)?"foundry":
  z_(process.env.CLAUDE_CODE_USE_ANTHROPIC_AWS)?"anthropicAws":
  z_(process.env.CLAUDE_CODE_USE_MANTLE)?"mantle":
  z_(process.env.CLAUDE_CODE_USE_VERTEX)?"vertex":"firstParty"}
```

Exactly six modes, no gateway. CCR/proxy is a base-URL concept (ANTHROPIC_BASE_URL / CLAUDE_CODE_USE_CCR_V2, both in the bundle), orthogonal to S8().

### Provider-specific filtering -- Pg / Jy8 (Bedrock)

```
Pg=V6((H)=>{let _=jy8(H);if(Mw(H)==="bedrock")return _.filter((q)=>!N28.has(q));return _}),
Jy8=V6((H)=>{return jy8(H).filter((q)=>N28.has(q))});
N28=new Set([frH,yn,FX_]);
```

On Bedrock, interleaved_thinking, long_context and tool-search-tool are removed from the main header (Pg) and split off (Jy8). Experimental betas gated by:

```
function DSH(){return z_(process.env.CLAUDE_CODE_DISABLE_EXPERIMENTAL_BETAS)||cX_("hipaa")}
```

### Agentic + SDK betas -- foH()

```
function foH(H,_){let q=[...Pg(H)];if(_?.isAgenticQuery){if(!q.includes(ayH))q.push(ayH)}
  let K=ZJ(); ... let O=K.map(k28);
  if(!FN())O=O.filter((T)=>{if(X37.has(T))return!0;
    return N(`SDK beta '${T.header}' dropped on 3P`,{level:"debug"}),!1});
  return[...q,...O.filter((T)=>!q.includes(T))]}
X37=new Set([ayH,frH,yn,syH,$t,UX_,jrH,FX_,xG,jv]);
function ZJ(){return GG()?.sdkBetas??p_.sdkBetas}
```

X37 is the third-party SDK-beta allowlist.

### ANTHROPIC_BETAS validation -- D37 / dd5

```
function D37(H){...if(Lq()){console.warn("Warning: Custom betas are only available for API key users. Ignoring provided betas.");return}
  let{allowed:_,disallowed:q}=dd5(H);
  for(let K of q)console.warn(`Warning: Beta header '${K}' is not allowed. Only the following betas are supported: ${SW([...J37]).join(", ")}`);
  return _.length>0?_:void 0}
J37=new Set([yn]);
```

Under OAuth (Lq()), ANTHROPIC_BETAS is ignored wholesale; otherwise only context-1m-2025-08-07 is allowed.

### ANTHROPIC_CUSTOM_HEADERS

Newline-separated Key: Value lines; merged so SDK defaultHeaders WIN over custom:

```
let z=Ff("ANTHROPIC_CUSTOM_HEADERS");if(z){let Y={};for(let A of z.split(`\n`)){let w=A.indexOf(":");if(w>=0)Y[A.substring(0,w).trim()]=A.substring(w+1).trim()}O.defaultHeaders={...Y,...O.defaultHeaders}}
```

Presence logged at request time (no values):

```
[API:request] Creating client, ANTHROPIC_CUSTOM_HEADERS present: ${!!process.env.ANTHROPIC_CUSTOM_HEADERS}, has Authorization header: ${!!A.Authorization}
```

Merge order {...custom, ...defaults}: a custom header colliding with a runtime-set header (anthropic-beta, Authorization, x-api-key) is overridden by the runtime.

### Beta header serialization

SDK spreads the request betas array and comma-joins via toString(). Verified on the environments client (identical SDK shape):

```
headers:J9([{"anthropic-beta":[...K??[],"managed-agents-2026-04-01"].toString()},q?.headers])
```

### Retry / backoff

```
async shouldRetry(H,_){...if(H.status===401&&this._authState.tokenCache&&q.usedTokenCache&&!q.didRefreshFor401)return q.didRefreshFor401=!0,this._authState.tokenCache.invalidate(),!0;
  let K=H.headers.get("x-should-retry");if(K==="true")return!0;if(K==="false")return!1;
  if(H.status===408)return!0;if(H.status===409)return!0;if(H.status===429)return!0;if(H.status>=500)return!0;return!1}
calculateDefaultRetryTimeoutMillis(H,_){let O=_-H,T=Math.min(0.5*Math.pow(2,O),8),z=1-Math.random()*0.25;return T*z*1000}
```

```
async retryRequest(H,_,q,K){let O,T=K?.get("retry-after-ms");if(T){...O=$}
  let z=K?.get("retry-after");if(z&&!O){...O=$*1000;else O=Date.parse(z)-Date.now()}
  if(O===void 0){let $=H.maxRetries??this.maxRetries;O=this.calculateDefaultRetryTimeoutMillis(_,$)}...}
```

Observed:
- Retried statuses: 408, 409, 429, >=500. A single 401 triggers one OAuth token refresh+invalidate then retry (didRefreshFor401 one-shot).
- Server override: x-should-retry: true|false forces/blocks a retry.
- Delay precedence: retry-after-ms -> retry-after (seconds or HTTP-date) -> exponential backoff min(0.5*2^attempt, 8) s with 1 - rand*0.25 jitter (cap 8 s).
- Default maxRetries = 2 (this.maxRetries=O.maxRetries??2); timeout parseInt(API_TIMEOUT_MS) || 600000 ms. Non-streaming requests projected >10 min throw and force streaming (calculateNonstreamingTimeout).

### Streaming / SSE

Consumer requires Content-Type: text/event-stream, decodes via streaming TextDecoder, dispatches MessageEvent keyed on the event: field:

```
if(!(A.get("content-type")||"").startsWith("text/event-stream")){...'Invalid content type, expected "text/event-stream"'...}
let f=new TextDecoder,j=z.getReader()... M&&$Y(this,j__).feed(f.decode(M,{stream:!D}))
new MessageEvent(O.event||"message",{data:O.data,...})
```

### stream_options / include_usage -- NOT USED

The tokens stream_options and include_usage do not occur anywhere in the bundle (grep -ac = 0). Claude Code does not use the OpenAI-style stream_options.include_usage; usage comes from native Anthropic message_start / message_delta SSE events.

### Operator notes

- The 1M-context beta (context-1m-2025-08-07) is emitted only when the model id contains [1m] (regex /\[1m\]/i); a gateway rewriting the model name drops it.
- S8() has exactly six provider modes (bedrock, foundry, anthropicAws, mantle, vertex, firstParty) -- no gateway. Proxy/CCR routing is purely ANTHROPIC_BASE_URL / CLAUDE_CODE_USE_CCR_V2.
- On Bedrock, interleaved_thinking, long_context and tool-search-tool are filtered out of the main anthropic-beta header (N28 via Pg) and split into a separate channel (Jy8).
- ANTHROPIC_BETAS is ignored under OAuth login, and otherwise allows only context-1m-2025-08-07 (J37); anything else logs a not-allowed warning and is dropped.
- ANTHROPIC_CUSTOM_HEADERS is Key: Value per line, but runtime defaultHeaders override custom values ({...custom, ...defaults}) -- you cannot override Authorization, x-api-key, or anthropic-beta this way.
- CLAUDE_CODE_DISABLE_EXPERIMENTAL_BETAS (or HIPAA mode) strips context_management, structured_outputs and other experimental betas (DSH()).
- Retries fire on 408/409/429/>=500 and a one-time 401-with-token-refresh. Gateways control backoff via retry-after / retry-after-ms and can force/suppress with x-should-retry. Default maxRetries=2, backoff capped at 8 s, timeout API_TIMEOUT_MS (default 600000 ms).
- No stream_options/include_usage anywhere -- usage must be read from native Anthropic SSE message_start/message_delta events; gateways synthesizing OpenAI-style usage will not be consumed.
- Stream parsing hard-requires Content-Type: text/event-stream; a gateway returning a different content-type on a streamed response triggers an Invalid content type error.
