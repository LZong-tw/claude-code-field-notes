## Session, Transcript & Restore

_Verified against Claude Code 2.1.178._

Claude Code persists each session as a newline-delimited JSON (`.jsonl`) transcript under the projects directory, one JSON record per line. The runtime self-documents this:

```text
Session transcripts are stored as .jsonl files under the projects directory. Each line is a JSON message;
user and assistant messages contain a "content" field with the conversation text.
The filename (without .jsonl) is the session ID.
```

The transcript path is `<projectDir>/<sessionId>.jsonl` (function `Ky` / `Tv_`):

```js
Ky(H){if(H===C_())return eV()??lz();let _=TA(W8());return cY.join(_,`${H}.jsonl`)}
// existence check:
Tv_(H){let _=wI()??TA(W8()),q=cY.join(_,`${H}.jsonl`)...statSync(q),!0...}
```

### Transcript record shape

Each line is a discriminated record keyed by `type`. The reader (`Y1T` = `parseTranscriptEntries`) accepts these `type`s and requires a string `uuid`:

```js
function Y1T(H){...let A=UjH(Y),w=A.type;
if((w==="user"||w==="assistant"||w==="progress"||w==="system"||w==="attachment")&&typeof A.uuid==="string")q.push(A)}
```

Conversation linkage fields, written onto every entry:

```js
{...G,cwd:Z.cwd,userType:Z.userType,entrypoint:Z.entrypoint,version:Z.version,
 gitBranch:Z.gitBranch,sessionId:K,timestamp:new Date().toISOString()}
... {...R,parentUuid:P,isSidechain:!1}
```

So a typical record carries: `parentUuid` (chain link, `null` at root), `uuid`, `isSidechain` (true for sub-agent/side branches), `userType` (`"external"` — `fn6()` returns the literal `"external"`), `cwd`, `version`, `gitBranch`, `sessionId`, `timestamp`, plus `type` and the `message` payload. A fast byte-scanner (`Y$T`/`A$T`) recognizes records by the literal prefixes `{"parentUuid":`, `"uuid":"`, `"isSidechain":true`, `","timestamp":"`, `"compact_boundary"`, and `"type":"last-prompt"`, so field *ordering* in the serialized line is fixed (`parentUuid` first, then `uuid`, etc.):

```js
function Y$T(H){let O=Buffer.from('{"parentUuid":'),T=Buffer.from('"uuid":"'),
z=Buffer.from('"isSidechain":true'),...Y=Buffer.from('","timestamp":"')...}
```

### How `message.model` is persisted (and what resume does NOT do)

For assistant turns, the model lives inside the nested `message` object (the raw Anthropic message), e.g. `message.model`. Synthetic / API-error assistant records are stamped with the sentinel model `"<synthetic>"`:

```js
QG="<synthetic>"
// synthetic/error assistant record:
type:"assistant",...message:{id:...,model:QG,role:"assistant",stop_reason:"stop_sequence",...usage:z,...}
```

`message.model` is read in **15** places, and every one is for accounting/display — never to choose the *active* model on resume. Examples:

```js
// per-model token aggregation (skips synthetic):
let a=U.message.model||"unknown";if(a===QG)continue;if(!f[a])f[a]={inputTokens:0,...}
// usage attribution / transcript-mode dim-color render of the model name:
let T=_6(q.message.model)+8,...createElement(V,{dimColor:!0},q.message.model)
// "find last real assistant model" for context display, skipping synthetic:
H.findLast((P)=>P.type==="assistant"&&P.message.model!==QG)
```

The active model for a resumed session comes from config / the `--model` flag, not the transcript. The model-picker UI confirms this is a session-level default:

```text
Switch between Claude models. Your pick becomes the default for new sessions.
For other/previous model names, specify with --model.
```

`restoreSessionMetadata` (`M5H`) restores title, tag, agent name/color, **mode**, and **permissionMode** from the session's metadata sidecar — but notably **not** the model:

```js
M5H(H){...if(H.mode)_.currentSessionMode=H.mode;if(H.permissionMode)_.currentSessionPermissionMode=H.permissionMode;...}
```

> Inference (strong): a restored session re-derives its model and therefore its context window from the current config/flag, not from `message.model` in the transcript. The transcript's per-message `model` is historical metadata only. The context window follows the chosen model's limit; resume does not pin it to whatever the old transcript ran on.

### `--continue` / `--resume` and how the file is located

Both `--continue` and `--resume[=<id>]` are recognized flags. A resume argument that is an absolute path ending in `.jsonl` is treated as a transcript file directly (`isTranscriptFileResumeArg` = `Xn6`):

```js
function Xn6(H){return cY.isAbsolute(H)&&H.endsWith(".jsonl")}
```

Otherwise the id maps to `<projectDir>/<id>.jsonl`. The resumed session adopts that file and continues appending (`adoptResumedSessionFile` / `resumeSessionId` carried through session-launch metadata). Reading a transcript goes through `KG4` → `$1T` (read bytes, optionally only the post-compact-boundary tail) → `Y1T` (parse lines) → `qG4`/`A1T` (rebuild the chain):

```js
async function $1T(H,_){try{if(_>xJH&&!z_(process.env.CLAUDE_CODE_DISABLE_PRECOMPACT_SKIP))
  return(await Ce_(H,_)).postBoundaryBuf;return await HG4.readFile(H)}catch{return null}}
```

A consistency check (`checkResumeConsistency` = `Z1q`) compares an expected `messageCount` from a `turn_duration` checkpoint record against the actual chain length and emits telemetry `tengu_resume_consistency_delta`:

```js
Z1q(H){...if(q.subtype!=="turn_duration")continue;let K=q.messageCount;...
c("tengu_resume_consistency_delta",{expected:K,actual:O,delta:O-K,chain_length:H.length,...})}
```

### Chain reconstruction & compaction relinking (`A1T`)

`A1T` is the heart of restore. It indexes every record by `uuid`, then for each `compact_boundary` record it **relinks parent pointers** so the post-compaction summary supersedes the summarized range, then walks `parentUuid` from the newest leaf back to the root, reverses, and re-attaches tool-result / progress children:

```js
function A1T(H){let _=new Map;for(let j of H)_.set(j.uuid,j);
for(let j of _.values()){if(j.type!=="system"||j.subtype!=="compact_boundary")continue;
 let J=j.compactMetadata?.preservedMessages,D=j.compactMetadata?.preservedSegment;
 if(J){...let M=J.anchorUuid;for(let Z of J.uuids){let W=_.get(Z);_.set(Z,{...W,parentUuid:M}),M=Z}...}
 else if(D){let M=_.get(D.headUuid);if(M)_.set(D.headUuid,{...M,parentUuid:D.anchorUuid});
   for(let[X,P]of _)if(P.parentUuid===D.anchorUuid&&X!==D.headUuid)_.set(X,{...P,parentUuid:D.tailUuid})}}
 ...let Y=...$(T);... // pick newest non-sidechain leaf, then walk parentUuid back to root
```

The self-documenting comment matches this: instead of walking the chain, readers relink uuids directly:

```text
readers look up each UUID directly and relink uuids[i] to uuids[i-1] (uuids[0] to anchor_uuid)
instead of walking the parentUuid chain. Unset when compaction summarizes everything.
```

The leaf chosen as the active tip prefers records that are not sidechain, not team, not meta:

```js
let z=T.filter((j)=>!j.isSidechain&&!j.teamName&&!j.isMeta),...Y=z.length>0?$(z):$(T)
```

### Compaction records

Compaction writes two related records.

1. A `system` / `compact_boundary` record carrying `compactMetadata`:

```js
type:"system",subtype:"compact_boundary",content:"Conversation compacted",isMeta:!1,
timestamp:new Date().toISOString(),uuid:QE.randomUUID(),level:"info",
compactMetadata:{trigger:H,preTokens:_,userContext:K,messagesSummarized:O},...q&&{logicalParentUuid:q}}
```
- `trigger` — what caused compaction (e.g. auto vs manual `/compact`).
- `preTokens` — token count before compaction.
- `userContext` — extra user-provided compaction instructions.
- `messagesSummarized` — how many messages were folded in.
- optional `logicalParentUuid` — the pre-compaction parent for chain rewrites.

2. The summary itself is a **user-type** message flagged so it shows in the transcript but is treated as compaction output:

```js
return{ok:!0,summaryText:Y,...,messages:[F6({content:hk_(Y,...),isCompactSummary:!0,isVisibleInTranscriptOnly:!0})]}
```

Records with `isCompactSummary===!0` (and `isMeta===!0`) are skipped by command-history / "first meaningful user message" scanners:

```js
if(H.isMeta===!0||H.isCompactSummary===!0)return; // ZJ_
if(O.includes('"isCompactSummary":true')...)continue;
```

There is also a separate `summary` record type (e.g. `{"type":"summary",searchCount,readCount,replCount,uuid:"su..."}`) that links a generated summary text to a `leafUuid` — the conversation leaf it summarizes — used when listing/resuming sessions:

```js
leafUuid:w.uuid,summary:q,customTitle:K,tag:T,fileHistorySnapshots:O,...
leafUuid:y.uuid,summary:K.get(y.uuid),customTitle:O.get(C),aiTitle:T.get(...)
```

### Write path (append-only, dedup-aware)

Records are appended via `recordTranscript` (`B_H`), which de-dupes against the set of uuids already on disk before inserting a chain, returning the new leaf uuid:

```js
function B_H(H,_,q,K){let O=eBH(H,K),...for(let f of O)if(z.has(f.uuid)){...}else $.push(f),A=!0;
 if($.length>0)await W5().insertMessageChain($,!1,void 0,Y,_);return $.findLast(T5H)?.uuid??Y??null}
```

The per-type append policy (`ENTRY_APPEND_POLICY` = `DN4`) governs dedup: most types are `"dedup-transcript"`, summaries are `"always"`:

```js
DN4={user:"dedup-transcript",assistant:"dedup-transcript",attachment:"dedup-transcript",
 system:"dedup-transcript",progress:"dedup-transcript",summary:"always",...}
```

Persistence is suppressed entirely when `Qc()` is true:

```js
function Qc(){let H=z_(process.env.TEST_ENABLE_SESSION_PERSISTENCE);
 return RN4()==="test"&&!H||Am()||z_(process.env.CLAUDE_CODE_SKIP_PROMPT_HISTORY)||YvH()}
```

### Read/scan size limits (observed constants)

```js
Y0q=52428800   // MAX_TRANSCRIPT_READ_BYTES = 50 MiB cap on a single transcript read
MN4=256        // INDEX_HEAD_SCAN_BYTES
XN4=4096       // INDEX_BOUNDARY_SCAN_BYTES
PN4=1024       // INDEX_LAST_PROMPT_SCAN_BYTES
WN4=64         // LAST_PROMPT_PREFIX_SCAN_BYTES
```

Filtering of restored entries for the model context: `j1T` drops `isMeta`, `isSidechain`, and `teamName` records (system records only included when `includeSystemMessages` is set):

```js
function j1T(H,_){...if(H.isMeta)return!1;if(H.isSidechain)return!1;if(H.teamName)return!1;return!0}
```

`/resume`'s session list filters out sidechain entries (`filtered from /resume: isSidechain=true`).

### Operator notes

- **Transcript = source of conversation; model = config, not transcript.** A resumed/`--continue`'d session re-derives its active model (and thus context window) from your config/`--model` flag, not from `message.model` in the old transcript. If you resume a session that was run on a 1M-context model but your current default is 200k, the resumed turn uses 200k. Pass `--model` explicitly to pin it.
- **`message.model` is metadata only.** Gateways/CCR can safely rewrite/normalize the model string they report; Claude Code uses the per-message `model` only for usage stats and the dim model label in transcript view. `"<synthetic>"` marks locally-generated (non-API) assistant records and is excluded from token/cost aggregation.
- **Resume by id or by path.** `--resume <id>` resolves to `<projectDir>/<id>.jsonl`; an absolute path ending in `.jsonl` is accepted directly (`isTranscriptFileResumeArg`). `--continue` picks the most-recent session for the cwd's project.
- **Disable persistence with `CLAUDE_CODE_SKIP_PROMPT_HISTORY`** (truthy) — no `.jsonl` is written and prompt history is off. Useful for ephemeral/CI runs through a gateway where you don't want transcripts on disk.
- **Compaction is chain-rewriting, not truncation.** A `compact_boundary` system record + an `isCompactSummary` user message are appended; on restore, `parentUuid` pointers are relinked so the summary supersedes the summarized range. `compactMetadata.trigger/preTokens/messagesSummarized` are your forensic breadcrumbs for "why/when did context collapse."
- **`CLAUDE_CODE_DISABLE_PRECOMPACT_SKIP`** forces reading the whole transcript on resume instead of only the post-boundary tail — slower restore on big files, but reads everything before the last compaction. Leave unset for normal use.
- **50 MiB read cap** (`MAX_TRANSCRIPT_READ_BYTES`): extremely long sessions can hit this; transcripts larger than that won't be fully read on restore.
- **Resume consistency telemetry:** mismatches between a `turn_duration` checkpoint's `messageCount` and the rebuilt chain length emit `tengu_resume_consistency_delta` — a signal that something dropped/duplicated entries (e.g. a gateway killing the write mid-turn).
