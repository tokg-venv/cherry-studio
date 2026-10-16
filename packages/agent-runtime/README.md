# Agent runtime

Runs the Pi coding agent (`@earendil-works/pi-coding-agent`) with every model request made by the
host through the AI SDK. Pi owns the agent loop: tool execution, steering, compaction. The host owns
the model call, so Cherry's AI SDK plugin chain (`transformParams`, model middlewares, usage
accounting, retries) applies to Pi's requests unchanged.

**Status:** not wired into the app. Nothing in `src/` imports this package yet.

## Boundaries

- No imports from the app (`src/`, `@main`, `@shared`, `@logger`, `@application`, `@data`) and no
  logging. Everything host-specific comes in through the options below.
- Pi is ESM-only and the main bundle is CommonJS, so the host must load this package with a dynamic
  `import()`, as `src/main/ai/runtime/pi/piSdk.ts` does for Pi itself.
- External code enters through the package export only (`src/index.ts`).

## How a model request flows

1. Pi calls the provider's `streamSimple` once per model request (agent turns and compaction
   summaries alike).
2. `createAiSdkProvider` converts Pi's transcript into AI SDK `ModelMessage[]` (after Pi's own
   `transformMessages` replay rules), Pi's tools into AI SDK tools **without** `execute`, and calls
   the host's `ModelCallPort.streamText(request)`.
3. The host runs one single-step `streamText` (in Cherry: ai-core's executor). With no `execute`,
   the AI SDK returns tool calls instead of running them, and Pi stays the loop owner.
4. The bridge maps `fullStream` back into Pi events: text, reasoning (provider metadata is kept in Pi's
   signature slots), tool calls, usage, stop reason, errors and abort.

## Host-facing contracts

- **`ModelCallPort`** – the only way out to a model. The request carries a `requestId`, Pi's
  `sessionId`, and `options`, the host's own per-request settings. The bridge never hands the port
  a request whose signal is already aborted.
- **Reasoning** – the host decides it through `requestOptions` (forwarded verbatim on every request),
  never through Pi's thinking level, which lacks Cherry levels such as `ultra`.
- **Side channel** – `onError` receives the original error object (for example `APICallError` with
  `statusCode` and `responseBody`), and `onUnmappedPart` receives `source`, `file` and
  provider-executed tool parts. Both get the request identity. Pi keeps only the error message, as
  the provider wrote it, so Pi's context-overflow detection still works.
- **Invalid tool calls** – a call whose input is unparsable or not a JSON object, or whose input the
  stream never finalized (it ended after `tool-input-start`), is replayed with `{}`, and the tool
  never runs on it. When the tool's schema has required fields, Pi's argument validation rejects
  `{}`; otherwise the session builder's guard, fed by `onInvalidToolCall` and ahead of every host
  `tool_call` handler (an approval prompt never sees the call), answers with the AI SDK's error (for
  an unfinished call: that the stream ended before its input was complete). A call to an unknown
  tool gets Pi's `Tool <name> not found`. A host registering `createAiSdkProvider` itself must block
  the calls `onInvalidToolCall` reports the same way.
- **Usage** – the usage handed to Pi feeds its context accounting and compaction only. A foreground
  `subagent` result also carries its subagent's summed usage on the tool result, and a background
  run's notification (or the `stop_subagent` result of a run the model stopped) carries it in
  `details`, for budgets. The host's AI SDK calls already account all of it, so it must not be
  billed again.
- **Retries** – Pi's retry is always disabled. Retries belong to the AI SDK layer.
- **Model identity** – every model has a host `key` (in Cherry, the unique model id). Assistant
  entries store it, and on rebuild only replies with the current key replay their reasoning as
  reasoning, with signatures; other models get that reasoning as plain text. The Pi `provider`/`id`
  may change between sessions.

## Session builder

`createAgentRuntimeSession` builds an `AgentSession` entirely in memory: in-memory credentials,
models and settings, `SessionManager.inMemory` rebuilt from the host's transcript, and a
`DefaultResourceLoader` with extension, skill, prompt-template and theme discovery off. The host
supplies the system prompt (Pi's own prompt when omitted; Pi still appends a `<cwd>` section) and any
appended prompt, custom tools, the enabled built-in tools (none by default), extra extension
factories, the model descriptor, compaction and Pi settings such as `shellCommandPrefix`. It may
opt in to workspace `AGENTS.md` / `CLAUDE.md` context files and to explicit skill directories, as
Cherry's current Pi runtime does. `dispose()` aborts the running turn, emits `session_shutdown` to
extensions and disposes the session. `prompt()` and `compact()` wrap `session.prompt` and
`session.compact`; the host starts every user turn and manual compaction through them (see
Subagents). `stopSubagent()` and `backgroundSubagent()` are the user's controls for subagents.

## Transcript

The host owns storage. It passes the session's transcript in (`transcript`, the full active path,
oldest first, uncompacted) and persists what comes out (`onEvent`). A session is always rebuilt from
the transcript; nothing is kept between sessions except `sessionId`, which should stay stable per
host session because it is also the prompt-cache routing key.

`TranscriptEntry` is plain JSON:

| Kind | Holds | Pi entry |
| --- | --- | --- |
| `message` | AI SDK `ModelMessage` (`user`, `assistant`, `tool` with one `tool-result`), plus `modelKey`, `usage` (token counts and `cost`, the USD the reply was billed at; absent reads as 0), `stopReason`, `errorMessage`, `responseId` (assistant) and `details` and `usage` (tool: what the tool reported for its own work, e.g. a subagent's requests) | `message` |
| `message` with `custom` | a `user` message an extension added (`pi.sendMessage`), with its type, display flag and details | `custom_message` |
| `compaction` | `summary`, `firstKeptEntryId`, `tokensBefore`, `details` | `compaction` |
| `context-edit` | an omitted message (`replacement: null`) | `context_edit` |
| `state` | extension state: a Pi `custom` entry whose type starts with `cherry.` | `custom` |

- **What the model saw** – the message payload is what the bridge sends (one codec in
  `modelMessages.ts`). Reasoning keeps its provider options, so signatures replay to the same model
  key. Images are inline base64.
- **Rebuild** – `SessionManager.inMemory(cwd, { id }, entries)` with the host's ids and
  timestamps, so Pi's compaction, context accounting and extension state behave as in the live
  session. State entries are in place before extensions bind, so `session_start` sees them. The
  transcript is validated first; a bad one throws `TranscriptError` (`invalid_entry`,
  `duplicate_id`, `unsupported_content`, `orphan_tool_result`, `compaction_boundary_missing`,
  `edit_target_missing`) and nothing is created. Tool calls left without a result (a crash
  mid-turn) and failed or aborted replies are valid: Pi answers or skips them when it builds a
  request.
- **Output** – `onEvent` receives `transcript-append` with new entries in Pi order, each exactly
  once; `compaction-start` / `compaction-end` (after the compaction entry was appended); and
  `turn-complete` with the head entry id when a run settles; `subagents` with the background
  subagents after each change of one of them. Pi `system`, `model_change`,
  `thinking_level_change` and `usage` entries are never emitted: the host owns the prompt, the
  model and billing.
- **Tool loadout** – tools activated beyond the session's base tools (by `tool_search`) are
  recorded as a `cherry.tool-loadout` state entry and re-activated on rebuild, after the tools the
  host enables now. Deactivating a base tool is not recorded.

## Compaction

Pi's native compaction runs, configured by `compaction: { reserveTokens, keepRecentTokens,
enabled?, summarize? }`: it compacts once the context passes `contextWindow - reserveTokens` (also
mid-turn, before the next model request) and compacts and retries once when a request overflows.
The host computes both numbers from its own settings. Without `summarize`, Pi's default summarizer
runs on the session model through the port. With it, the host summarizes with its own prompt and
model: `summarize({ sessionId, reason, messages, previousSummary, instructions, signal })` receives
the compacting session's `sessionId` (a subagent's is the id `SubagentPort.configure` received, so a
host can tell its compaction from its parent's) and the folded messages as AI SDK messages, after the session model's replay rules (failed replies left out, tool
calls without a result answered), so they can be sent as they are, and returns the summary text, which is stored as is; Pi frames it when it
builds the context. A failing summarizer cancels that compaction (it never falls back to the session
model) and `compaction-end` carries the error: a threshold compaction leaves the turn running, an
overflow then ends the turn with the provider's error.

## Tool output offload

`offload: { store, thresholdChars }` keeps one oversized tool result from overflowing the context.
A `tool_result` hook (after every other extension's) saves text output longer than
`thresholdChars` (and than 3000 characters, so the marker is always shorter than the output) through
the host's `ToolOutputStore` and gives the model its head and tail around a
`<persisted-output>` note with the saved path, to read back with Pi's `read` tool. If the store
throws, the note gives its error instead and the model keeps only the head and tail. Names are content
addressed (`tool-output-<sha256>.txt`), so the same output gives the same marker and prompt caches
hold. Images and `structuredContent` (for the UI and codemode) are kept. Not offloaded: errors,
`read` results (reading an offloaded file back must not offload it again) and nested tool calls,
which reach the model only through their caller. The transcript stores the marker, which is what the
model saw. The host must let `read` open the store's paths without an approval prompt.

## Recall

`recall: true` registers `vcc_recall`, with pi-vcc's tool name and parameters (`query`, `range`,
`expand`, `page`, `scope`, `mode`), so existing tool settings and approvals apply; the host disables
it by name like any tool. It reads this session's transcript: the full replayed path plus every entry
since, including the running turn. `#N` is an entry's position among the message entries, so it stays
the same once the host stores the running turn and rebuilds. `query` is a keyword search (entries matching more of the
words first, 5 per page), `range: [from, to]` lists entries in order (20 per page) and `expand: [N]`
returns full text. `scope: 'all'` is the same as `'lineage'`: edits drop the turns after them.

## Subagents

`subagents: SubagentPort` registers three `model-only` tools: `subagent({ description, prompt,
fork?, background?, model? })` with DSH's name, so Cherry's renderer shows it as a Task card,
`message_subagent({ id, message })` and `stop_subagent({ id })` (`SUBAGENT_TOOL_NAME`,
`MESSAGE_SUBAGENT_TOOL_NAME`, `STOP_SUBAGENT_TOOL_NAME`). Design:
`docs/superpowers/specs/2026-10-14-agent-runtime-background-subagent-design.md`.

Each `subagent` call builds a session in-process from the parent's options (port, request options,
model, prompts, tools, compaction, offload, recall, settings) plus the extensions the host returns
from `SubagentPort.configure(subagent)`, and starts it with `session.prompt()` so
`before_agent_start` handlers run as in any turn. It shares the parent's `sideChannel`, told apart by
the request `sessionId`; the parent's `extensionFactories` and `onEvent` are not inherited. Its
session id is a random UUID: the id the model addresses it by and the key the host stores it under.
A run makes at most 200 model requests, counted where its session makes them: its turns and the
compaction summaries made for it (Pi's own summarizer through the port, or the host's
`compaction.summarize`) alike; the 201st is refused before it goes out. Calls of one step run at the
same time. A fresh subagent has
none of the three tools; a fork keeps them, so its tools match the parent's, and they fail with
`Subagents cannot start or manage subagents.`

- **Foreground** (the default) – the call blocks. Its result is the final message followed by
  `(id: … — continue this subagent with message_subagent.)`, `details`
  `{ sessionId, fork, background: false, outcome }` and `usage`, the sum of the subagent's replies.
  A run that fails, is cut off by its output limit, is aborted with the parent's turn or would start
  a 201st request returns an `isError` result with the reason and its last partial output
  (`outcome` `failed`, `truncated`, `aborted`, `limit_reached`). A host whose `configure` throws
  fails that call only. `backgroundSubagent(sessionId)` moves a running foreground subagent to the
  background: the call returns at once (`details.background: true`) and the same run goes on; the
  parent's turn no longer stops it.
- **Background** (`background: true`) – the call returns its id at once, and stopping the parent's
  turn does not stop the subagent. At most `maxBackground` (default 20) run at once. When a run
  ends, its result reaches the parent as a `cherry.subagent-notification` custom message (a user
  message to the model; `details` is `SubagentNotificationDetails`): merged into the parent's run
  as it settles, or, when the parent is idle, as the start of a new run, which the host shows as an
  autonomous turn (`background-work`). Never in the middle of a run; a queued user message goes
  first, and results that end together go in one message. After the user stops a turn, nothing
  starts on its own until their next prompt, which pending results ride with (a goal aborting a
  run to withdraw a stale round is no such stop). Results that start no
  run (the user's stops, those a rebuild restores) ride with the next notification or prompt, or are
  merged as a run no prompt started (a command's, such as `/goal resume`) settles. A report over the
  offload threshold (20,000 characters without offload) is saved through the offload store, or cut
  to its head and tail.
- **Messages** – `message_subagent` queues a message for a running subagent: it reads it after its
  current tool batch, and one that comes as it settles starts its next run in the same session.
  For a finished subagent it loads the stored transcript (`SubagentPort.load`), configures the
  same subagent again (`continuedBy`, and `modelKey` from its origin entry), answers tool calls an
  interrupted run left open with an "interrupted" result and runs the message in the background,
  after a recovery hint when the last run was interrupted or aborted. Without `load`, finished
  subagents cannot be continued. The id must be one this conversation started. A message to a
  subagent being stopped waits for the stop, then continues it or is refused as for a finished one.
  `details` is `SubagentMessageDetails`: `{ sessionId, continued: false }` when queued,
  `{ sessionId, continued: true, run }` when it started a run.
- **Stopping** – `stop_subagent` stops a running subagent and returns its partial output; no
  notification follows, and it can be continued, unless the user also stopped that run (before or
  after the model). Its `details` (`SubagentStopDetails`) is
  `{ sessionId, stopped: true, run, usage }` with the stopped run's usage, or
  `{ sessionId, stopped: false }`. `stopSubagent(sessionId)` is the user's stop button for a
  background subagent: the parent hears of it only with its next message, and it cannot be
  continued. The user's stop is recorded when it is accepted (a `cherry.subagent-state` snapshot
  with `stoppedBy: 'user'` while the run winds down), so a parent rebuilt after closing meanwhile
  still refuses to continue that run and reports it as stopped by the user. Disposing the parent
  stops and disposes every subagent session; a run that had ended on its own by then keeps its
  result, which the rebuilt parent delivers.

With `SubagentPort.findModels`, `subagent` takes a `model` the user named: the package resolves it
among the exact candidates the host finds (none or several fail and tell the model what to ask) and
hands the key to `configure` as `Subagent.modelKey`, for which the host must return that model in
`SubagentSessionOptions.model`, unless the key is the parent's current model: then the parent's
model runs it when the host returns none, so a host that never picks models can still continue
subagents. Without a named model the host may return its default subagent model; otherwise the
subagent runs on the parent's. A fork always keeps the parent's model, and a continuation the model
of its first run.

The host's approval extension comes from `configure` and labels each request with `Subagent`.
`SubagentPort.onApproval` (the host's one approval registry) marks a subagent `awaiting_approval`
while one of its calls waits; the parent sees this in the `subagents` event, its status message and
errors that list subagents. Each event should name the session that runs the call
(`ToolApprovalEvent.sessionId`, `ctx.sessionManager.getSessionId()` in the approval extension);
without it, sessions are told apart by call id alone.

The parent records each background run in `cherry.subagent-state` snapshots (started; finished with
its outcome, result and usage), never sent to the model, and `onEvent` gets
`{ type: 'subagents', subagents: SubagentView[] }` after each change. A rebuilt parent restores its
subagents from them and from `subagent` results: a run whose last snapshot is `running` was cut off
(`interrupted`) and can be continued; such runs, and every run's result that never reached the
parent, ride with the user's next message (or the settle of a turn a command starts), once. After a
compaction a `cherry.subagent-status` message lists the background subagents still running and the
results not delivered yet.

A subagent's `onEvent` receives its whole transcript: the first `transcript-append` holds its start
entries, the forked prefix and a `cherry.subagent-origin` state entry
(`{ parentToolCallId, fork, description, modelKey }`, never sent to the model). The host stores it
per `sessionId` to implement `load`. `onSessionCreated` receives the live session before its first
request. `AskUserQuestion` ends as `unavailable` inside a subagent without asking the port.

Host contract: start user turns with `AgentRuntimeSession.prompt` and manual compactions with
`compact`, never `session.prompt` or `session.compact` directly (Pi reports a preparing prompt as
idle, and a notification run would collide with it); do not dispose or rebuild the parent while
background subagents run. A `prompt` or `compact` called while a run settles (from `onEvent`'s
`turn-complete`, where Pi itself would defer it and return at once) waits until that settle and any
run Pi started during it are over, then runs, and resolves or rejects as it does.

## Extensions

Factories the host passes in `extensionFactories`. Their tools are `model-only` (never callable from
codemode scripts, so every call stays its own tool part) and use the names Cherry's renderer already
handles.

- **`createTodoExtension()`** – `todo_write` with dsh's schema: `{ todos: { content, status }[] }`,
  status `pending | in_progress | completed`, no other item fields. Every call carries the whole list
  and replaces the previous one; content is trimmed and must be non-empty and unique, and at most one
  todo may be `in_progress`. Invalid lists come back as error results. `details` is
  `{ todos, counts }`. The extension keeps no state: the latest list is the last successful
  `todo_write` on the transcript's active path, so it follows forks and edits and the model sees it in
  the rebuilt history, until a compaction folds that call into the summary.
- **`createAskUserExtension(port)`** – `AskUserQuestion` with Claude Code's input schema (1–4
  questions, each with a `header`, 2–4 options and `multiSelect`; question texts must differ, since
  answers are keyed by them, and so must the option labels of a question, since answers name them).
  The tool waits on `AskUserPort.ask`, which resolves `answered` (`answers`, optional `annotations`
  notes), `declined` (optional `feedback`) or `unavailable` (no one can answer, e.g. a channel or
  scheduled run); inside a subagent's session it ends as `unavailable` without asking the port. The
  result text tells the model what happened; only `answered` is a success. Every result the tool
  returns, aborted included, has `details` `{ questions, answers, annotations?, outcome }`, `outcome`
  being `'answered' | 'declined' | 'unavailable' | 'aborted'`;
  invalid input and a rejecting port fail the call with empty `details`. Calls in
  one model step run one at a time. When the turn aborts, the tool stops waiting at once and the
  request's `signal` aborts so the host can withdraw the question; a late answer is ignored. A
  question pending at a crash is not persisted: the rebuilt session closes the call as failed.
- **`createGoalExtension(port)`** – a persisted objective the runtime keeps pushing while the
  model is idle. State lives in `cherry.goal` entries (one snapshot per change, a tombstone on
  clear); each round is a `cherry.goal-round` message the host maps to the "round N" header. Rounds
  continue only at `agent_before_settle` and count once Pi sends them on; when
  `port.defaults.autonomy()` returns `confirm-each-round`, every round ends paused. Every stop is
  the runtime's and sets `phase` and `lastStop`: round cap (the last allowed round is a wrap-up;
  a cap lowered to the running round ends the goal with it), token or USD budget, no progress
  (two rounds hint, three stop; the same tool call three times in
  a row within a round is blocked, five times ends the goal), a denied approval, an approval nobody
  can give or a dismissed question in a goal round, a round another extension's settle handler
  overrides, and a failed request or an abort (`session.abort()` or the host's own request signal)
  while the goal is armed. Usage is never
  kept: every check (the budget, `resume`, `/goal`, `get_goal`) reads it from the transcript, so a
  rebuilt session sees what the live one saw. It is every request made in one of the goal's
  admitted rounds while the goal was active, the request whose tool call stopped it included and
  requests after a stop (the summary after `complete`) never, priced at each entry's stored `cost`,
  plus the `usage` a tool result reports when the request that made its call counted (a foreground
  subagent still running under `/goal pause` counts when it returns). Snapshots record that usage
  when written (`tokensUsed`, `costUsed`) for display only, and a stop's snapshot the progress up to
  the stop. A round reserved at a settle is checked again when it starts: if a command changed the
  goal meanwhile (a later extension's settle handler was still running), the round is withdrawn
  before its request, and the goal goes on as it now is (a new round, or the stop its new limits
  call for). The host sees a withdrawal as a run that ends aborted (`turn-complete` with
  `aborted: true`) right after that round's prompt, its reply `stopReason: 'error'` with the
  message `This operation was aborted`; it should show it as a withdrawn round, not as a failure or
  a stop the user made. A round has run once a snapshot names its id (`admittedRound`, the round
  message's `details.id`). A round prompt no snapshot names (another extension vetoed the
  continuation, the run was aborted at the settle, or the round was withdrawn) stays in the
  transcript and is left out of later model requests and of the verifier's conversation; the
  compaction summarizer still reads it.
  `update_goal(complete)` passes the host's `GoalVerifier` (it
  gets the full current context, after compaction, and the call's `signal`; the package sets no
  timeout) or, without one, needs evidence and a round that changed something. An accepted
  completion also tells the model to end its turn with a summary, and a complete goal is final:
  pause, resume and further reports are refused, and new work needs a new goal.
  `update_goal(blocked)` is accepted at once in a human turn and in goal rounds only after
  `blockedAfter` consecutive rounds report it; several reports in one round count once, a human
  turn, `resume` or a user edit starts the count over, and the reasons are not compared. A queued
  user message always goes first, and its turn does not count for the goal. Tools are `get_goal` /
  `create_goal` / `update_goal` with DSH's output shape; the model can create, edit, pause or
  resume a goal only in a human turn. The command
  `/goal [<objective> | edit <objective> | pause | resume | clear | rounds <n> | budget <tokens|$usd>]`
  makes no model request itself and reports through `cherry.goal-notice` entries; creating or
  resuming a goal on an idle session starts its first round, while during a run it only arms the
  goal for that run's settle. `/goal pause` and `/goal clear` abort the running turn only when the
  goal drives it (active and armed). `session.prompt('/goal …')` returns once the command is
  handled: a command that starts a round ends with `turn-complete` as usual, the others (the
  status, a pause while idle, …) produce none. A rebuilt session restores the goal disarmed and
  writes that down (`armed: false` in a snapshot of the same revision); `/goal resume` continues it.
  The budget also counts the `usage` a tool result carries, so a
  `subagent` call's requests count toward it. While background subagents run, an armed goal waits
  (no round, no stop; `/goal` shows `Waiting for N background subagents`) and goes on after their
  notification's turn settles. That turn is neither a human turn nor a goal round (a notification
  that rides with a person's message or a round's start belongs to that turn); while the goal is
  armed its usage counts toward the budget. So does each background run launched, moved or
  continued by a request that counts for the goal (a person's turn's runs are not the goal's):
  once, when its result reaches the model (its notification or a `stop_subagent` result;
  overlapping stops report one run), whether the goal is running then or not (disarmed, paused or
  blocked). Once the budget is used up, background launches fail and the goal's running background
  subagents stop. `port.onApproval` events of other sessions (by `sessionId`, or calls not
  executing in this session when an event has none) are ignored. When `port.modelSupportsTools()`
  is false, no tools or command are registered and an active goal is blocked with
  `unsupported_model`. In a subagent's session the goal tools fail with `in_subagent`, nothing
  continues and `port.onApproval` is not subscribed, so a fork ignores the goal entries it inherits.
  Design:
  `docs/superpowers/specs/2026-10-11-agent-runtime-goal-design.md`.

The host's approval layer must not ask approval for `AskUserQuestion` (the question is itself the
interaction, or the user would be asked twice) and may auto-allow `todo_write` (it only changes the
session's list). `ASK_USER_TOOL_NAME` and `TODO_TOOL_NAME` hold the names.

The host sets `create_goal` to ask for approval, so a goal the model proposes is confirmed by the
user through the existing approval card. `GOAL_TOOL_NAMES` holds the goal tools' names.

The approval layer may auto-allow `subagent`, `message_subagent` and `stop_subagent` as long as
`SubagentPort.configure()` returns the approval extension, so each call a subagent makes is approved
on its own.

## Known gaps

- `vcc_recall` has no regex search, `mode: 'touched'` (files worked on) or `#N:path` file
  drill-down yet; compaction summaries do not cite `#N` entries.
- `read` results are never offloaded, and Pi caps them at 50 KB, which can still overflow a small
  context window.
- A parallel tool batch whose results together exceed the context window, each below the
  threshold, still overflows; offloading at `turn_end` is deferred.
- Pi's `bash` tool saves the full output of a truncated command to a temp file and names it in the
  result; that file is not moved into the offload store and may be gone when read later.
- Pi's `after_provider_response` extension event never fires for these models: `streamText` does not
  expose response headers before the body is consumed.
- Tool call ids from another provider are replayed as-is (no `normalizeToolCallId`). A provider
  with stricter id rules may reject history recorded by a different one.
- Pi's resource loader still scans its discovery directories (`cwd/.pi`, `agentDir`,
  `~/.agents/skills`) while loading. The `no*` flags and prompt overrides discard everything it finds.
- Not carried in the transcript: context edits that replace content (Pi itself only omits
  messages), `bashExecution` and `branch_summary` entries (no Cherry producer), and `isError` on a
  tool result that also holds images.
- The CherryIN Anthropic endpoint omits the thinking block of tool-only replies but requires one on
  replay; Cherry's Pi runtime rebuilds it from the response id (`piThinkingReplay.ts`). The
  transcript keeps `responseId` for this, but requests do not expose it to the port yet; how they
  will is decided when the host is wired up.
- A fork's requests carry the subagent's session id, so a host that routes prompt caches by session id
  (OpenAI's `prompt_cache_key`) does not reach the parent's cache partition.
- Goals: no fresh-context (Ralph) mode. The package does not know the permission mode; keeping
  goals from running rounds in plan mode is up to the host.
- A `session.abort()` that lands in the run withdrawing a stale goal round (it ends before any
  request) is taken for that withdrawal: the goal goes on instead of pausing.
- Every fork stores the parent's prefix in its own transcript; a host may deduplicate it.
- A background result reaches a busy parent only when its run settles, after the whole turn.
- Shutdown writes no snapshots for the runs it cuts off, so a rebuilt parent cannot tell a closed
  app from a crash.
- Messages queued for a running subagent before it is stopped are dropped with it.
- Approval events without `sessionId` find their session by call id: two sessions running calls
  with the same id at the same moment cannot be told apart. For a foreground subagent that moment
  is its whole run, since the parent's `subagent` call runs all the while.
- A subagent on another model gets the parent's `requestOptions`.

## Tests

```bash
pnpm test:agent-runtime                              # from the repository root
pnpm --filter @cherrystudio/agent-runtime test       # or the package's own Vitest config
pnpm --filter @cherrystudio/agent-runtime typecheck
```

Tests drive real `streamText` over `MockLanguageModelV3` (and one real provider over local HTTP)
through the same port shape the host implements.
