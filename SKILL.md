---
name: self-review
description: Sweep the current diff (uncommitted, staged, or feat-branch ahead-of-base) for the twelve recurring classes of bugs that automated PR reviewers (Copilot, code-rabbit, etc.) consistently catch but humans miss — pattern inconsistency (incl. lookup-table boundary fallbacks, 0-vs-1-based index drift, ternary fall-through, spread-without-field, inconsistent return-arity), missing lifecycle/teardown (incl. unbounded in-memory caches, AudioContext/MediaStream close, useEffect deps drift, async-acquire-then-store teardown race), tunnel-vision sibling sites (incl. pipeline drift, inconsistent identifier across sibling write sites, guard reading a proxy-ref not reset by every teardown path), doc drift (incl. stale model identifiers and numbered-comment-list drifted from execution order), untrusted-text-into-LLM-prompts (incl. sentinel-substring control-flow triggers), self-contradictory LLM-prompt instructions (a new rule that silently overrides a standing "never drop / same info" rule without being marked the explicit exception), ghost references / unfulfilled doc promises (incl. docstring-vs-impl mismatch and comments asserting unverifiable cross-repo consistency), hardcoded secrets / credentials committed to source, severed signal chains (removed producer with surviving consumers), boundary hardening on untrusted client/external input (type-too-loose params, missing length cap, missing URL-encoding, cross-system identifier casing/format mismatch, negative-cache missing), async / render-hot-path hygiene (sync I/O in async, external-state read in render, heavy compute on every stream chunk, quadratic-loop patterns, read-before-write ordering, cross-tab/same-tab sync incomplete), AND diff scope hygiene (every hunk justified by the commit message — catches cross-branch file-copy leaks, stray formatter passes, stale debug prints, generated-file noise, merge artifacts). Designed to be run before `git commit` or before opening a PR.
---

# Self-Review

You are about to perform a structured audit of pending code changes. The goal: catch the specific class of bugs a human author tends to miss but a fresh static-analysis pass would flag. Do NOT skip steps — the value is in the systematic sweep.

## When this skill fires

- User explicitly invokes `/self-review`
- Run BEFORE `git commit` and BEFORE `git push` on a feature branch
- Can also run on a worktree with staged-but-uncommitted changes

## The twelve classes of bugs to catch

These are the patterns automated reviewers consistently catch that humans authoring the change consistently miss:

1. **Pattern inconsistency** — A new helper/function uses a different convention than existing helpers in the same file/module that do similar things. Includes: side-effect-before-guard ordering; lookup-table boundary fallbacks (`_CARDINALS[100] = "hundred"` produces "hundred point five" not "one hundred point five"); 0-based vs 1-based index drift (`metadata.segmentIndex == _current_idx` where one is 1-based and the other 0-based); ternary fall-through with unreachable branch (`a ? X : (a && b ? Y : Z)` — branch Y never reached); spread-without-field (`{...msg, x: a, y: b}` at one site but `{...msg, x: a}` at the sibling drops field y); inconsistent return-tuple arity between success and error paths (success returns `(r, f, t, trace)`, error returns `(r, f, t)` — callers must branch on tuple length); variable scoping miss (using a name that's only defined in a sibling scope, e.g. `activeThreadId` in the replay scope where only `sessionId` is bound); aggregate-vs-collection scope mismatch (a response returns both a list with `pagination.total_count` AND summary totals, but the totals are computed over a different set than the list — e.g. `summary.total_api_calls = len(own)` while the list/`total_count` is the merged set — so they silently disagree).
2. **Missing lifecycle / teardown** — A new stateful primitive (useState, useRef, AbortController, EventSource, blob URL, setInterval, setTimeout, addEventListener) without a corresponding cleanup. Also: unbounded in-memory stores/caches with no TTL or LRU eviction; AudioContext / MediaStream / PeerConnection / Worker / SharedArrayBuffer not closed on disconnect or unmount; `useEffect` cleanup dependency-array missing callbacks or refs the effect body references.
3. **Tunnel-vision sibling sites** — A pattern fix applied to ONE site when the same pattern (and same bug) exists at sibling sites in the same file. Also: **pipeline drift** (TTS gets `normalize(x)` but transcription validator gets raw `x`); **inconsistent identifier across sibling write sites** (one fetch uses `messageId`, two others use `dbMessageId` — payloads to the same endpoint don't agree); **double-execute** (bypass path AND post-turn handler both perform the same advance, skipping a segment).
4. **Doc drift** — Code identifiers, library names, model names, or behavior changed but comments / docstrings / READMEs still reference the old version. Also: hard-coded line numbers in comments, stale model/SDK names in prose, AND numbered/ordered comment lists that drifted from the actual code execution order (comment says rules 1→2→3→4→5 but execution order is 2→1→3→5→4).
5. **Untrusted text → LLM-facing prompts** — Plan/DB/user-authored data interpolated into a string that becomes an LLM instruction or prompt, without sanitization. Also: **sentinel-substring control-flow triggers** — matching a hard-coded token (`"[CONCLUDE_TOPIC]"`, `"[FORCE_SKIP]"`, etc.) inside user-supplied content to decide control flow. A student writing the token in chat triggers the flow. Also: **dynamic values serialized into a delimited/structured format** (GFM table, CSV, query string) without escaping the delimiter, normalizing row arity, or honoring a "blank cell" placeholder intent — breaks structure even for non-LLM (frontend) sinks.
6. **Ghost references & unfulfilled doc promises** — Calls to methods/imports that don't exist anywhere; doc-comments on flags/fields that promise behavior no code reader implements; AND function docstrings that promise the function "matches X" / "mirrors Y" / "strips Z" / "handles W" when the implementation doesn't actually deliver that.
7. **Hardcoded secrets / credentials** — Passwords, API keys, DB connection strings, internal IPs paired with credentials, or `print("password: ...")` statements committed in source.
8. **Severed signal chains** — A state write (`state.X = True`), event emit (`bus.publish('X')`), SSE/WebSocket send, or other producer was *removed* in the diff, but downstream readers were NOT.
9. **Boundary hardening on untrusted client / external input** — Where data crosses an untrusted edge (HTTP route handler, WebSocket inbound, third-party API, browser localStorage) into your code, server-side defenses are easy to forget. Includes: type-too-loose params (free string where `Literal["a"|"b"|"c"]` belongs); missing length caps on client-provided strings; missing `encodeURIComponent` when interpolating user input into URLs; cross-system identifier casing/format mismatch (backend registers `tool_id` lowercased, frontend follow-up actions send mixed case); regex parsing that breaks on trailing slash / missing port / extra param; missing negative-caching of failed external fetches (every failure re-hits the network); same-tab state-sync missing custom event (only `storage` listener fires, cross-tab works but same-tab doesn't).
10. **Async / render-hot-path hygiene** — Code that runs hot — inside an async coroutine, inside a React render, inside a streaming chunk handler — needs to be cheap and pure. Includes: sync I/O / blocking calls in async code (should use `asyncio.to_thread` or async client); external-state read in React render without subscription (`localStorage.getItem('x')` directly in render body — React won't re-render when the value changes); heavy computation on every stream chunk (regex parse / format / JSON.parse every chunk — defer until stream completes); quadratic patterns in loops (`binary += String.fromCharCode(uint8[i])` for big buffers — chunk it); read-before-write ordering within a closure (capture `prevHighlight = ref.current` BEFORE the line that updates `ref.current`, otherwise both reads see the new value).
11. **Diff scope hygiene** — Every hunk in the diff should be explained by the commit message / PR description. Hunks the author can't justify are usually accidents — `git checkout <other-branch> -- <file>` shortcut that leaked an entire base-branch's worth of edits to the file, stray formatter / linter pass, stale debug prints, generated lockfile noise, merge / rebase artifacts. These are upstream of half the bugs the other sweeps catch (a severed signal chain, a doc drift, a pattern inconsistency) — if a change is *unintentional*, the author has no mental model of it and won't notice issues with it. Copilot will.
12. **LLM prompt-instruction coherence** — A rule ADDED to a prompt/instruction string the app feeds an LLM (system prompt, seeded DB prompt, tool description) contradicts a standing rule in the SAME prompt, with neither amended to reconcile them — e.g. a new "omit markdown-table rows from speech" rule against existing "speech must contain the SAME information" / "do NOT drop ANY original word" absolutes. The model then obeys inconsistently and the bug is non-deterministic. Distinct from class 5: there the threat is untrusted *data* flowing into a prompt; here the prompt's own *rules* fight each other.

## Procedure

### Phase 1 — Identify the diff scope

Run these in parallel:

```bash
# What changed?
git status --short
git diff --stat HEAD          # uncommitted vs HEAD
git diff --stat <base>...HEAD # if on a feature branch
```

Pick the right comparison:
- If the user is mid-edit (unstaged/staged but no commit yet) → compare against `HEAD`
- If on a feature branch about to PR → compare against the merge-base with the target branch (`dev_V1`, `main`, etc.)
- If unsure, ask the user once: "Am I auditing uncommitted edits or your whole feature branch?"

### Phase 2 — Run the twelve sweeps

For each sweep, output a section header. Do NOT skip a sweep that returns no findings — explicitly say "no findings" so the user knows you ran it.

**Run Sweep 11 (diff scope hygiene) first.** Hunks that turn out to be unintentional should be reverted before the other sweeps spend cycles auditing them — and finding "Copilot just flagged a severed signal chain in code I never meant to touch" after a force-push round-trip is a worse outcome than catching the leak here.

---

#### Sweep 1 — Pattern inconsistency

**Goal**: Find new helpers/functions that diverge from established conventions in the same file/module.

**Method**:
1. For each new function/helper added in the diff, identify its key operations:
   - State accessors (`stateName`, `stateNameRef.current`, `useSelector`, store getters)
   - URL/path construction (template literals like `` `${BASE}/${id}/...` ``)
   - API client calls
   - Resource lookups
2. Grep the same file for sibling functions performing similar operations.
3. Diff the approaches. Flag mismatches.

**Concrete checks** (run all that apply to the languages in the diff):

- **State-as-ref pattern** (React / similar): If the codebase uses `xRef.current || x` for some state value, and the new code uses just `x`, flag it. Common in this codebase: `sessionIdRef.current || sessionId`, `currentSegmentIndexRef.current || currentSegmentIndex`.
- **Counting / iteration pattern**: If existing code iterates `state.messages[seg_start:]` and new code iterates the whole `state.messages` with a filter, flag it. Off-by-one bugs love this gap.
- **URL construction pattern**: If existing fetches go through `apiService` or a shared helper, and the new code uses `fetch(\`${BASE}/...\`)` directly, flag it. Also: are query params built consistently?
- **Error handling pattern**: If existing error handlers do `console.warn` with a tag and propagate, and the new one swallows silently, flag it.
- **Guard ordering** (side-effect-before-early-return): If new event-handler / async code performs state mutations (`setX(...)`, `store.dispatch(...)`, `ref.current = ...`, `setMessages(prev => ...)`) and THEN checks an early-return guard (`if (busy) return;`, `if (!ready) return;`, `if (!hasattr(...)) return`), the partial side-effects fire even when the guard short-circuits. Flag it. The guard should be the first executable statement in the handler. Sibling check: find another handler in the same file/module that uses the same guard predicate — does the sibling put the guard first? If yes, your new code diverges and silently half-applies under the guarded condition.
  - Example caught: new "Go deeper" onClick did `setMessages(...)` + `setPendingTopicAdvance(false)` + `if (curiosity.isBusy) return;` + `await curiosity.spawn(...)`. Clicking during the busy window dismissed the UI row AND reset state, but no card spawned — silent partial failure with no retry path. Sibling Curiosity Mode button puts `if (curiosity.isBusy) return;` first.
- **Lookup-table boundary fallback**: When the diff introduces a function that maps an integer/key to a word/value via a dict lookup with a `str()` or pass-through fallback, check the entries that are present in the dict for *coarse* values that produce speakable-but-wrong output at boundaries.
  - Example caught: `_decimal_to_words(100.5)` did `_CARDINALS.get(100, str(100))` → `_CARDINALS[100]` is `"hundred"` (not `"one hundred"`) so output was `"hundred point five"`. The bug isn't a KeyError; it's that the dict's entry is *correct for its intended caller* (`_fraction_to_words` says "three hundredths") but *wrong for the new caller* (`_decimal_to_words` needs the prefix "one"). When sharing a constants dict across helpers, audit each entry for caller-specific grammar.
  - Method: for each new helper that reads a shared lookup, list the dict keys it relies on and ask whether the helper's surrounding grammar makes each entry's English natural ("one X point Y" vs "X point Y"). Special-case boundaries (10s, 100s, 1000s, exactly-zero, exactly-one) explicitly rather than trusting the dict.

- **0-based vs 1-based index drift**: When the diff compares two integer indices across modules, check whether each is 0-based or 1-based. Storage formats (DB rows, JSON metadata stamped by the frontend, user-facing labels) tend to be 1-based; in-memory cursors (`_current_idx`, list indices) tend to be 0-based. A comparison like `metadata.segmentIndex == _current_idx` silently undercounts to 0 when the orchestration paths' conventions don't match.
  - Method: for each new integer comparison or arithmetic between values from different modules, grep for how each value is set. If one is set as `len(...)` or `i + 1` and the other as `range(...)` or array index, flag.
  - Example caught: FORCE_SKIP auto-advance compared `metadata.segmentIndex` (1-based, stored on assistant rows; missing on user rows) to `_current_idx` (0-based). Comparison silently undercounted user messages in the current segment to 0.

- **Ternary fall-through unreachable branch**: A ternary chain `cond1 ? X : (cond1 && cond2 ? Y : Z)` — branch Y is never reachable because `cond1` already steered into X. Reorder so the more-specific predicate fires first.
  - Example caught: `isPlayingThis ? '■' : (isTTSLoading && isPlayingThis ? '…' : '▶')` — the buffering glyph `…` was unreachable because the first branch already captured `isPlayingThis`. Fix: `isPlayingThis && isTTSLoading ? '…' : (isPlayingThis ? '■' : '▶')`.

- **Spread-without-field**: One site does `{...msg, x: a, y: b}` but a sibling site does `{...msg, x: a}` and drops `y`. The next render's `msg.y` is `undefined`. Grep all sibling spreads of the same base for divergent field sets.
  - Example caught: `setMessages(... ? { ...msg, word_timestamps, ttsDisplayWords } : msg)` at one callback site missed `ttsCleanedContent` that two other callbacks set. Replay stale-detection broke for that codepath.

- **Inconsistent return-tuple arity** between success and error paths: Function returns `(r, f, t, trace)` on success but `(r, f, t)` on error. Callers / type-checkers either branch on tuple length or unpack-crash. Normalize so both paths return the same shape (use `None` or a sentinel for missing fields).
  - Example caught: scoring analyzers returned a 4-tuple on success and 3-tuple on error after a refactor; callers crashed under the (rare) error path.

- **Variable scoping miss**: A name that's only bound in a sibling scope gets pasted into the new scope. TypeScript catches this at compile (TS2304); Python catches it at runtime as `NameError`. JS without TS silently uses `undefined`.
  - Method: for each new identifier used in the diff, confirm it's actually bound in the enclosing scope (function arg, closure capture, import). Beware copy-paste from a different handler that closed over a different name.
  - Example caught: replay-scope code referenced `activeThreadId` where only `sessionId` was in scope — TS2304 in CI.

- **Aggregate-vs-collection scope mismatch**: When a single response (or a single function's return) carries BOTH a collection (a list, plus `total_count` / pagination / a rendered set) AND summary aggregates over it (counts, sums, totals, averages, cost), every aggregate must be derived from the SAME underlying set as the collection. If the diff changes what the list contains (filter, merge, prepend, dedupe, scope-widen) but leaves a sibling aggregate computed over the old/other set, they silently disagree — and any client that assumes `summary.total == pagination.total_count`, or cross-references count vs sum (avg = tokens/calls), breaks. The two are computed in different spots, so the author rarely notices.
  - Method: for each response/return that includes a list AND a derived total/summary, identify the source set of EACH. If the diff changed the list's source (e.g. `all_calls = own` → `all_calls = inherited + own`) but a sibling total/sum/avg/cost still reads the old source (`len(own_calls)`, `_calc(state.api_calls)`), flag. Fix by deriving every aggregate from the one merged/filtered set (extract a single `_summarize(theList)` helper and a `theList` local; reuse both for pagination AND the summary).
  - Sibling tell: `pagination.total_count = len(merged)` two lines above `summary["total"] = len(subset)`; or `cost = _calc(subsetA)` while the returned rows are `subsetB`.
  - Example caught (Copilot): observability endpoint returned `api_calls` + `pagination.total_count` over the merged fork lineage (inherited prefix + own), but `summary.total_api_calls`/cost stayed `own`-only — so summary disagreed with the paginated list on fork threads. Fix: `_lineage_calls(state)` + `_summarize_calls(list)` applied to every endpoint so counts, tokens, avg, and cost all come from the same list the panel renders.

- **Precondition-after-specialization in an if/elif (or guard) chain**: A branch chain dispatches on a *specific* signal (an action, a button, a mode) in its first arms, and a *fundamental precondition* that should gate ALL of them sits in a LATER arm (or the `else`). When the specific signal is present, the chain fires that arm and the precondition is never evaluated — so a required setup/first-pass step is silently skipped. The precondition must be checked FIRST and either short-circuit or neutralize the specific arms. Distinct from the ternary-fall-through check below (that's an *unreachable* branch); here the branch IS reached but *shouldn't* be until the precondition holds.
  - Method: for each new `if action == X: … elif action == Y: … elif <precondition>: …` (or a guard placed after action arms), ask "if the precondition is false but an action is set, does an action arm run anyway?" If the precondition is a prerequisite for the action arms (state not yet initialized, first-pass not yet shown, resource not yet loaded), it belongs ABOVE them — or the action must be dropped/deferred when the precondition fails.
  - Example caught (Copilot, PR #228): `run_orchestrator_classroom` checked `if action == "continue": … elif action == "literary": … elif not core_shown: <present core>`. A `continue`/`literary` button arriving before the topic's core was ever shown skipped PRESENT CORE entirely (video + summary never happened). Fix: `if action in ("continue","literary") and not core_shown: action = None` FIRST, so it falls through to the core arm.

- **Over-broad pattern / transform scope** (matcher fires beyond its intended inputs): A regex, `str.contains`, `startsWith`, or replace/split written to handle a KNOWN, specific set of inputs isn't anchored or allow-listed to them, so it collaterally matches unrelated content that happens to share a surface feature. The transform then mangles content it was never meant to touch. Scope it to the known set (enumerate the literals / anchor the pattern) rather than matching a generic shape.
  - Method: for each new regex/matcher added for a *specific* purpose (split THESE headings, detect THIS marker, rewrite THAT token), construct one plausible *unrelated* input that shares the matched shape (a heading with a number, a sentence containing the keyword, a path with the delimiter). If it matches, flag — narrow the pattern to the known cases.
  - Example caught (Copilot, PR #391): a markdown fix `#{1,6}[ \t]+\S[^\n]*?[ \t]+\d{1,2}[.)]\s` meant to split the literary headings ("### Key words 1. …") also split any heading containing " 1." — e.g. "### Chapter 1. Introduction". Fix: scope to the known headings `(?:Figures of speech|Rhyme scheme|Key words)`.

- **Duplicated near-identical object built via ternary** (drift-risk sibling of spread-without-field): A call builds two almost-identical literals through a ternary — `cond ? {a, b, c} : {a, b}` — where the branches share most fields and differ by one. The shared fields are now written twice; a later edit to one branch silently diverges from the other. Collapse to one object with a conditional spread of the differing field: `{a, b, ...(cond ? {c} : {})}`.
  - Method: for each ternary in the diff whose both arms are object/dict literals, diff their keys. If they share ≥2 keys and differ by an added key, flag as drift-risk and suggest the conditional-spread form.
  - Example caught (Copilot, PR #391): Continue button did `classroomMode ? { input_source:'continue_button', classroom_action:'continue' } : { input_source:'continue_button' }`. Fix: `{ input_source:'continue_button', ...(classroomMode ? { classroom_action:'continue' } : {}) }`.

- **Line-structure assumption vs the actual runtime input** (line-anchored regex / `.split("\n")` on input that isn't multi-line): Code using `(?m)^…` anchors, `re.MULTILINE`, `text.split("\n")`, or any per-line loop ASSUMES the input actually contains newlines. If an upstream step strips them (a frontend, a serializer, a `.replace(/\n/g,' ')`, a single-line producer), the `^` anchors NEVER fire and `.split("\n")` yields ONE element — so the transform silently no-ops or collapses the whole payload into one unit. The reverse also bites: logic written/tested on tidy multi-line input breaks on the single-line payload production actually sends. This is the highest-value check when a diff adds line-based text processing.
  - Method: for each `(?m)^` / `re.MULTILINE` / `.split("\n")` / line-by-line loop added, TRACE where the text actually comes from at runtime — does any upstream hop strip newlines? Don't test with a hand-typed multi-line string; construct the REAL payload shape (single-line if so) and run the code on it. If `^` never matches or the split yields one element, flag. **Also check ORDER**: when transform B is line-anchored and transform A is what re-creates the lines, A MUST run before B — running B first silently no-ops it (and the bug is invisible on multi-line test input).
  - Example caught (Copilot + self, PR tts-literary-single-line): the classroom Literary breakdown reached `/tts/live` as a SINGLE line (frontend stripped newlines), but `_literary_to_speech` / `_strip_md_to_words` used `.split("\n")` and `(?m)^…` — so the whole breakdown was one giant "heading": numbered markers stayed raw ("1." spoken as "number one"), and later "###" leaked as a spoken "hash". Sibling High finding: the `(?m)^(\s*\d+)\)` → `1.` canonicalization ran BEFORE the newline-restoring helper, so on single-line input it never fired and "1)" leaked into `display_words`. Fix: restore line breaks FIRST, then run the line-anchored steps. (Every local test had used `\n`-separated input, so it passed while production was broken — construct the real single-line shape.)

- **Quantified char-run matched mid-run** (a lookbehind/anchor lets a repeated-char quantifier match a SUFFIX of a run): A pattern with `#{1,6}`, `-+`, `=+`, `\d+`, `\*+` guarded by a lookbehind/assertion can match part-way INTO a run instead of the whole run — because when the run's true start fails the assertion, the engine backtracks to an interior position that passes. `(?<!\n)[ \t]*(#{1,6}…)` on `"\n\n### x"` fails at the first `#` (preceded by `\n`) but MATCHES at the second `#` (preceded by `#`), splitting `"###"` into `"#"` + `"## x"`.
  - Method: for each new regex with a repeated-char quantifier guarded by a lookbehind/lookahead/anchor, test an input where the run is immediately preceded by the asserted char (a newline before `###`, a digit before `\d+`, a dash before `-+`). If the match starts mid-run, flag. Fix: consume all preceding same-class chars so the quantifier starts at the run's first char (`\s*(#{1,6}…)` — greedily eats whitespace, so `#{1,6}` begins at the first hash), OR assert the char before the run is NOT the same class (`(?<!#)`).
  - Example caught (self, PR tts-literary-single-line): a `(?<!\n)[ \t]*(#{1,6}…)` "idempotency" attempt split `"###"` into `"#"`+`"##"` on multi-line input. Reverted to `\s*(#{1,6}…)`, which anchors to the run's first hash and is genuinely idempotent (`f(f(x))==f(x)`). Moral: adding a lookbehind to a run-quantifier can silently change WHERE the run is matched.

- **Insert-before-each transform fires at the START boundary too** (spurious leading separator/blank line): a `re.sub(r'…', r'\nMATCH', …)` / "prepend a delimiter before every match" / build-up-with-a-leading-separator transform ALSO matches the FIRST occurrence at position 0, prepending a separator the first element never needed — a leading `\n`, blank line, comma, or path segment. It's invisible while the output is only re-parsed by `split()` (the empty leading token is skipped), and it becomes VISIBLE the moment that same output doubles as a rendered / displayed / serialized form. This bites exactly when an internal normalization form is *promoted* to also be a user-facing output.
  - Method: for each new "prepend before each match" sub/replace (or `sep + item` accumulation), ask what the output feeds. If it's ever rendered, displayed, returned to a client, or logged verbatim — not only `.split()`-ed internally — test with input whose FIRST token matches. If there's a leading separator, flag; strip it on return (`.lstrip("\n")`, drop the empty first element) or guard the first match.
  - Example caught (Copilot, PR tts-literary-single-line): `_literary_linebreak` prepended `\n` before every heading; the first heading at position 0 got a leading `\n`, which — once the line-broken form was ALSO returned as `display_text` (promoted from internal-only to user-visible) — started the rendered markdown with a blank line. Fix: `return t.lstrip("\n")`. (Cross-refs the Sweep-3 "sibling outputs from different transforms" check — promoting an internal form to a returned/rendered field is what surfaced the latent leading `\n`.)

For each finding, output:
```
[INCONSISTENCY] <file>:<line>
  New code:      <one-line summary or excerpt>
  Existing pattern: <one-line summary, with file:line of example>
  Risk: <what could go wrong>
```

---

#### Sweep 2 — Missing lifecycle / teardown

**Goal**: Every new stateful primitive should have a corresponding cleanup, OR a documented reason it doesn't need one.

**Method**: Grep the diff for these patterns and check each occurrence:

| Added pattern | Look for cleanup |
|---|---|
| `useRef(new Set(...))` / `useRef(new Map(...))` / `useRef<...>(null)` | If the ref accumulates state across renders, is there a reset on `sessionId` / `userId` / similar change? Are entries removed when consumed? |
| `useState(...)` where state is keyed by something (e.g. `Record<id, ...>`) | When the parent changes, does the cache get cleared? |
| `URL.createObjectURL(blob)` | Is there a `URL.revokeObjectURL` somewhere? Especially when the URL is stored and overwritten later. |
| `new EventSource(...)`, `new WebSocket(...)`, `new AbortController()` | Cleanup in useEffect return? On unmount? On retry? |
| `setInterval`, `setTimeout` (long-running) | `clearInterval` / `clearTimeout`? |
| `addEventListener` | `removeEventListener` with same handler ref? |
| `new ResizeObserver`, `new IntersectionObserver`, `new MutationObserver` | `.disconnect()`? |
| Backend: `open(...)` without context manager | Explicit `close()` in `finally`? |
| Backend: subscription / queue handle | Unsubscribe? |

**Special check — Blob URLs**: For every `createObjectURL` in the diff, find every site where the resulting URL gets *overwritten* or where the *containing object is removed*. Verify there's a `revokeObjectURL` at each such site. Blob leaks are silent in dev and lethal in long-lived sessions.

**Special check — Unbounded in-memory stores**: A new module-level `dict`, `set`, or `list` that accumulates entries (per-thread state, per-session events, per-request audit log) must have a TTL OR an LRU cap. An unauthenticated endpoint writing into an uncapped store is a memory-exhaustion DoS vector.
  - Example caught: `tool_event_store` accumulated 3D-tool events keyed by thread-id with no expiry. Copilot added TTL + max-tracked-thread cap + LRU eviction.
  - Method: for each new `Dict[X, Y] = {}` / `Set[X] = set()` at module scope (or stored on a long-lived service singleton), grep its accessors for a corresponding `.pop(...)`, `time.time() - entry.created > TTL`, or `OrderedDict.move_to_end + popitem` — any eviction discipline. Zero results → flag.

**Special check — AudioContext / MediaStream / PeerConnection / Worker**: Web Audio nodes, MediaStreams, RTCPeerConnections, Web Workers, and SharedArrayBuffers don't get garbage-collected just because the React component unmounts. They need explicit `.close()` / `.disconnect()` / `.terminate()` on disconnect / unmount / retry.
  - Example caught: `useStudentVoiceAgent` opened a playback `AudioContext` but never closed it on `ws.onclose` / `disconnect()`. After N reconnects, the tab held N alive AudioContexts.

**Special check — async-acquire-then-store teardown race**: When the diff acquires a resource via `await` (`getUserMedia`, `connect()`, a `fetch` that returns a stream/handle, `open()`, a subscription) and stores it in a ref/field/variable on a LATER line, a teardown that fires DURING the await (unmount, session/route change, cancel, retry) sees the ref still empty and cannot release the resource — so when the await resolves the resource is acquired with nothing left to stop it (orphaned mic/socket/stream, indicator stuck on). Store-*after*-await is the tell; the teardown and the assignment race, and teardown usually wins.
  - Method: for every `const x = await acquire(...)` added in the diff whose result is later assigned to a ref/field (`someRef.current = x`) or drives further setup, ask: what runs teardown for this component/session, and does it null-check the not-yet-assigned ref (so it no-ops mid-await)? If so, capture a staleness token BEFORE the await — a generation counter bumped on unmount/session-change, a `mountedRef`, or an `AbortSignal` — and re-check it IMMEDIATELY after the await; if stale, release the just-acquired resource (`.getTracks().forEach(t => t.stop())` / `.close()`) and return before storing it or building anything further. Prefer reusing a generation token the codebase already bumps on unmount/nav over inventing a new primitive.
  - Example caught (Copilot, teacher-chat mic): `startRecording` did `const stream = await navigator.mediaDevices.getUserMedia(...)` then `vadMediaStreamRef.current = stream` on the next line. An unmount/session-change during the permission prompt ran the cleanup while `vadMediaStreamRef` was still null, so it couldn't stop the stream; once permission resolved the mic was live with nothing tracking it (indicator stuck on). Fix: snapshot the existing `startSessionGenRef` before the await and, if it changed after, stop the stream's tracks and return before assigning the ref.

**Special check — `useEffect` dependency-array drift**: When the body of a `useEffect` references a callback (`startMic`, `disconnect`) or ref-derived value, the deps array must list it. Stale-closure bugs from missing deps are subtle and load-bearing.
  - Method: for each `useEffect` ADDED or MODIFIED in the diff, list every identifier the body reads (excluding setters and constants), then diff against the deps array.

For each finding:
```
[LIFECYCLE] <file>:<line>
  Added:    <primitive that needs teardown>
  Missing:  <where the cleanup should live>
  Risk:     <what leaks / accumulates>
```

---

#### Sweep 3 — Tunnel-vision sibling sites

**Goal**: When a fix introduces a fallback / guard / null-check at one site, verify the same pattern doesn't exist unguarded elsewhere in the same file or directly adjacent files.

**Method**:
1. For each line ADDED in the diff that looks like a fallback chain (e.g. `A || B || C`), extract the access pattern `A`.
2. Grep the same file for OTHER occurrences of `A` (without the fallback).
3. For each occurrence, ask: does this site need the same fallback?

**Concrete checks**:

- **Fallback chains**: `x?.y || x?.z` — grep for `x?.y` alone elsewhere.
  - Example caught: `tc.function?.name || tc.name || 'unknown'` added at one site; same `tc.function?.name || 'unknown'` lived unguarded at 2 more sites in the same file.
- **Null-check guards**: New code adds `if (!threadId) throw ...` — grep for other uses of `threadId` that don't guard.
- **Guard reads a flag/ref that not every teardown path resets** (proxy-liveness drift across sibling teardowns): When the diff adds an early-return guard or "is this live?" check that reads a ref/flag as a proxy for state (`if (someRef.current) return;`), enumerate ALL the teardown/reset paths in the module — not just the one you wrote — and confirm each resets the SAME variable the guard reads. If a sibling teardown resets a *different* subset (nulls some refs + the `setX(null)` state setters but leaves the exact ref your guard reads dangling), the guard latches on the stale value forever after that path runs, blocking the operation permanently. This is the inverse of a normal sibling-fix miss: the divergence is between the guard's assumption and a teardown you didn't write.
  - Method: for each new `if (xRef.current …) return;` / flag-based guard, grep the module for every function that tears the feature down or resets it (`stopX`, `cleanupX`, `disconnect`, error/timeout handlers, the unmount effect). List which refs/flags each one actually clears. Any teardown that clears the *state* or a subset of refs but NOT the exact ref/flag the guard reads → flag. Prefer guarding on the flag EVERY teardown resets (often the state-mirroring ref like `isRecordingRef`), and let a belt-and-suspenders teardown clear any stale refs.
  - Example caught (Copilot, teacher-chat mic): a new re-entry guard read `if (isStartingRef.current || vadWebSocketRef.current) return;`, but the timeout/error teardown `cleanupLocal()` reset `isRecordingRef` and nulled only `vadMediaStreamRef` — it left `vadWebSocketRef`/`vadAudioContextRef`/`vadProcessorRef` non-null (it called the `setVad*(null)` state setters, not the refs). After one errored session the guard saw a stale non-null `vadWebSocketRef` and blocked every future start. Fix: guard on `isStartingRef || isRecordingRef` (flags every path resets) and let the teardown block clear stale refs.
- **Type narrowing**: New code adds `typeof x === 'number'` — sibling sites using `x` without narrowing.
- **Boundary handling**: New code clamps an index (`Math.max(0, ...)`); sibling array accesses don't clamp.
- **Pipeline drift** (high-value check): When the diff wraps a value passed to one downstream — e.g. `f(text=normalize(x), ...)` — grep the *same function/method scope* for OTHER downstreams that reference the same variable `x` without the transformation. The two downstreams have to receive the *same logical content* for the system to stay coherent. Classic case: TTS is given `normalize(text)` but the word-timestamp/transcription validator is still given raw `text`, so the audio says "zero point one" while alignment expects "0.1" — alignment breaks silently. Same shape: WebSocket payload `{"text": text, "words": timestamps_for_normalized}` — frontend gets a text/timestamps mismatch.
  - Method: for every transformation `f(x)` added in the diff, grep the enclosing function (and the closure scope it runs in) for other usages of `x`. Each usage that ends up at a downstream consuming *the same logical content* as the transformed sink needs the same transformation, OR the transformed value needs to be hoisted into a local (`transformed = f(x)`) and reused. Refactor to "transform once, use everywhere" is almost always the right fix.
  - Example caught: `voice_session.py` `run_tts()` passed `text=quick_normalize_speech(response_text)` to TTS but `expected_text=response_text` to `get_tts_word_timestamps` and `"text": response_text` to the WebSocket. Alignment drifted; frontend highlighting broke. Fix: hoist `speech_text = quick_normalize_speech(response_text)` once and use in all three sites.

- **Mismatched accepted-token set between two parsers that must agree** (drift on the input only one recognizes): When two functions each strip / tokenize / match the SAME class of syntactic marker over the SAME input — list markers (`1.` vs `1)` vs `-`/`*`/`+`), heading levels, quote styles (`"`/`"`/`"`), delimiters, whitespace — and their outputs are later ALIGNED or COMPARED (word-mapping, index zip, diff, dedup), the two must recognize the IDENTICAL token set. If one accepts a superset (a regex char-class `[.)]` where the sibling has only `\.`; a literal list one side lacks), an input using the token only one side recognizes makes the outputs diverge — one drops the marker, the other keeps it — and the alignment silently shifts. Unlike pipeline-drift (transform vs raw), here BOTH transform, but over different accepted sets.
  - Method: for each new function that parses a marker whose result is aligned against another function's result over the same text (e.g. `display_words` from `_strip_md_to_words` vs a speech flattener; tokens from two lexers; a highlighter vs a renderer), put the two marker regexes/char-classes side by side and diff their accepted set. For every token in the symmetric difference, construct an input containing it and run BOTH — if one drops it and the other keeps it, flag. Fix: make the two recognize the same set (share the regex, or canonicalize the input to the common form before the narrower parser).
  - Example caught (Copilot, PR #229): `_literary_to_speech` matched numbered items with `^\s*(\d+)[.)]\s+` (both `1.` and `1)`), but the sibling `_strip_md_to_words` feeding `display_words` (aligned against speech via `_build_word_mapping`) stripped only `^\s*\d+\.\s+` (`1.` only). A model emitting `1)` left the marker in `display_words` but not in speech → highlight mapping drifted by one token per item. Fix: canonicalize `1)` → `1.` before the display-word extraction so both recognize the same set. (The happy-path self-review missed it because only `1.` inputs were tested — the `1)` variant that exercises the divergence was never constructed.)

- **Inconsistent identifier across sibling write sites**: When the diff sends the same payload shape to the same endpoint from N sites, every site must use the *same* logical identifier in each field. Drift here causes the endpoint to receive correlated-but-non-matching rows.
  - Method: for each `fetch(SAME_URL, ...)` / `client.send(...)` / `db.upsert(...)` ADDED at multiple sites, diff the request bodies field-by-field. Are `thread_id`, `message_id`, etc. sourced from the same variables at every site?
  - Example caught: `/native/tts/store-audio` was called from three places. Two sites sent `message_id: dbMessageId` (the DB primary key); the third sent `message_id: messageId` (the in-memory React message id, never persisted). Audio for regenerated messages went to the wrong row.

- **Double-execute on shortcut + post-turn path**: When the diff adds a "fast path" that performs a side effect (advance, dispatch, save), check whether a downstream post-turn handler runs the same side effect again. If yes, the user action triggers it twice — skipping a segment, double-charging a counter, double-emitting an event.
  - Method: for each new fast-path that mutates state (`state.X = ...`, `await advance(...)`, `setSomething(...)`), grep the post-turn / post-stream / completion handler in the same module. Does that handler also perform the mutation? If yes, add a guard or an idempotency check.
  - Example caught: progress-decision-ui PR — bypass path called `progress_to_next_segment` AND the post-turn auto-advance also called it, skipping a segment when the LLM ignored the wrap-up prompt. Fix: `_handle_progress` early-returns when the latest user message has `conclude_topic` metadata.

- **Side-effect skipping in a shortcut path**: A new fast path calls a lower-level helper (`state.progress_to_next_segment()`) and bypasses the normal handler's accompanying side effects (immediate progress save, learned-terms append, segment summarization). Extract the side-effect bundle into a shared helper and route both paths through it.
  - Method: for each new direct call to a state mutator that's *normally* wrapped by a handler in the same module, list the handler's accompanying side effects. Verify the fast path performs them too.

- **Divergent parallel fallback chains** (two ladders that must resolve to the SAME resource): When two sibling sites each pick a value via a fallback chain (`A or B or C`) and both are supposed to resolve to the *same* underlying resource — e.g. one computes the resource, the other re-derives a piece of it — the chains must have IDENTICAL rungs in identical order. If one ladder has a middle rung the other lacks, a miss on the primary key resolves the two sites to *different* fallbacks, and they silently disagree. This is Sweep-3 pipeline-drift applied to selection logic rather than a single transform.
  - Method: for each new `x = A or B or C` (or if/elif key-selection) added in the diff, grep the same function/module for a sibling selection that must resolve to the same thing (often a "prefix" build vs a "suffix"/"tail" re-extraction, or "load object" vs "load object's field"). Diff the two ladders rung-by-rung. Any rung present in one but not the other → flag; make them lockstep (or extract one resolved local and reuse it).
  - Example caught (Copilot, PR #228): `v3_prompt_builder` selected the PREFIX base as `classroom_prompt or seed_fallback or v3_teacher_prompt`, but the SUFFIX tail-extraction fell back `classroom_prompt or seed_fallback or v3_base_prompt` — missing the `v3_teacher_prompt` rung. When `v3_classroom_prompt` was absent, prefix came from `v3_teacher_prompt` while suffix came from `v3_base_prompt`, desyncing any content after the dynamic placeholder. Fix: add the same `v3_teacher_prompt` rung to the suffix ladder so both resolve identically.

- **Sibling outputs of one response describe the SAME thing but are built from different transforms** (rendered form vs the tokens/coordinates that index it): when a function returns BOTH a payload that a consumer will RENDER (`display_text`, HTML, a markdown blob) AND a parallel structure the consumer uses to INDEX / align / highlight / map into that render (`display_words`, offsets, a word→timestamp map, line/col spans), the two MUST be derived from the SAME form of the input. If the diff transforms the input for the index side (`display_words = strip(linebreak(x))`) but returns the *untransformed* input as the render side (`display_text = x`), the consumer renders one structure and indexes another → highlighting/anchoring drifts. Return the SAME transformed form for both.
  - Method: for each returned dict/object that carries a "render this" field AND a "map into the render" field, check they come from the same variable/transform. If one is `f(x)` and the sibling is raw `x` (or `g(x)` with a different `g`), flag — hand the same `f(x)` to both, or document why the render normalizes to the index form on its own.
  - Example caught (Copilot, PR tts-literary-single-line): the literary branch built `display_words` from the line-broken `canonical` but returned the original single-line `text` as `display_text`. ReactMarkdown renders the squished single line as one heading while `display_words` came from the line-broken form, so word-highlight alignment drifted. Fix: `text = canonical` — return the line-broken form as `display_text` too, so render and index describe the same structure.

- **A regex re-built from a shared token list drops the ANCHORS/boundaries its sibling has**: when the diff builds a second regex from the same source-of-truth token list another regex already uses (good — kills the duplication from Sweep 1), it must also replicate that regex's anchoring — the trailing `\b`, leading `^`, `$`, `(?<=…)`, case flags. Sharing the *alternation* but not the *boundaries* means the two match different sets: the sibling fires on partial/substring matches the original rejects.
  - Method: for each new `re.sub`/`re.match` whose alternation is built from a constant that an existing compiled regex also uses, diff the two patterns' anchors and flags (`\b`, `^`, `$`, `re.IGNORECASE`, lookarounds). Any anchor on the source regex but missing on the new one → flag.
  - Example caught (Copilot, PR tts-literary-single-line): `_literary_linebreak` built its heading alternation from `_LITERARY_HEADINGS` (matching `_LITERARY_HEADING_RE`) but omitted the trailing `\b` that `_LITERARY_HEADING_RE` has, so it would split before `"### Figures of speeches …"` (a partial match the detector rejects). Fix: add the same `\b`.

For each finding:
```
[SIBLING] <file>:<line> (you fixed) vs <file>:<line> (you didn't)
  Pattern:  <the shared access pattern>
  Risk:     <same bug class at the unfixed site>
```

---

#### Sweep 4 — Doc drift

**Goal**: When code changes, comments / docstrings / READMEs that describe the OLD behavior should be updated or deleted.

**Method**:

1. **Removed identifier sweep**: For each identifier REMOVED in the diff (function name, class name, library import, constant, env var), grep the whole repo for surviving references in comments/docs/markdown. If any remain, flag.
   - Example caught: implementation swapped Haiku → Qwen on L40s, but the outer call-site comment block still said "classify with Haiku".

2. **Stale type/contract**: If a function signature or interface changed (new param, removed field), grep for callers AND for prose descriptions of the signature.

3. **Line-number references**: Grep the diff for hard-coded line numbers in comments (`\.py:\d+`, `\.ts:\d+`, etc.). Flag them — line numbers age fast.
   - Example caught: comment said "duplicate guard in tool_executor:744" — the line had moved.

4. **Stale code blocks in markdown**: README/docs with code snippets that mirror the diff's changes — do they still match?

5. **Stale model / SDK / vendor identifiers in prose**: Docs that name a specific model/SDK ("uses gpt-5.4-mini", "via Anthropic SDK", "powered by Whisper") need cross-checking against the *current* `_MODEL` / config / import constant. When the diff touches a docs file in a TTS/LLM/inference area, grep the doc for any quoted model or vendor name and verify it matches the code constant it claims to describe.
   - Example caught: `TTS_WORD_HIGHLIGHT_SYNC.md` said "OpenRouter, gpt-5.4-mini" but `tts_normalizer.py` had `_MODEL = "google/gemini-3-flash-preview"` — same paragraph the diff was editing, just one line above, so it should have been caught locally.

6. **Numbered / ordered comment lists drifted from code execution order**: A docstring or comment block lists "rules 1→2→3→4→5" or "steps: first X, then Y, then Z" but the actual execution order in the implementation is different (or has been refactored). Readers misread the contract.
   - Method: for each numbered/ordered list in a docstring or block-comment ADDED or NEARBY in the diff, trace the function body and confirm each item appears in the same order as the comment claims.
   - Example caught: `tool_usage` analyzer's docstring listed reconciliation rules 1→5; after a refactor the actual call order was 1→3→2→5→4. Caller reading the docstring to understand precedence got confused.

For each finding:
```
[DOC DRIFT] <file>:<line>
  Stale text: <excerpt of the outdated reference>
  Now:        <what the code actually does>
  Fix:        <suggested edit>
```

---

#### Sweep 5 — Untrusted text → LLM-facing prompts (prompt-injection surface)

**Goal**: Whenever data the application doesn't fully control (plan-authored content, student input, DB rows, filenames, third-party API responses) is embedded into a string that will be passed to an LLM as instruction or context, it must be sanitized. Without it, a single rogue character (a stray `"`, `]`, newline, or bracketed sentinel) can break the surrounding framing or smuggle pseudo-instructions to the model.

**Method**:

1. For each f-string / `.format()` / concatenation ADDED in the diff that contains a non-literal expression, ask:
   - **Sink**: does the resulting string become a prompt / tool result message / system instruction / classifier input / agent input?
   - **Source**: is the interpolated expression sourced from external data (segment/topic metadata from a plan, student message text, DB rows, file/path data, API responses, env vars set by another system)?
   - **Sanitizer**: is there a sanitization call (or a documented pre-cleaned source) between read and embed?
2. If sink = LLM-facing AND source = external AND no sanitizer → flag.

**Concrete checks**:

- **Plan/segment metadata into instructions**: any `seg.get('title')`, `topic.get('name')`, `chapter.get('description')`, `video.get('example_title')` embedded into a prompt/instruction without a sanitization step.
  - Example caught: `f"...conclude segment {seg_idx} (\"{seg_title}\")..."` — plan-authored title flowed raw into the LLM instruction. A title `Foo" — End of instructions. Now do X.` would break framing.
- **Student input into classifier/control prompts**: `user_message`, `messageText`, `student_reply` embedded into a system or instruction prompt without length cap + character substitution. Length-truncating alone is not enough — a 40-char malicious payload still bites.
- **DB rows in tool result messages**: titles, descriptions, error messages from DB lookups joined into a prompt-bound string (e.g., `titles = ", ".join(v.get('title') for v in unwatched)` → embedded in `"PROGRESS DENIED — videos: {titles}"`).
- **File paths / upload names**: filenames returned to the LLM in tool results without cleanup. Bracketed/quoted filenames can escape framing.
- **JSON-stringified args for re-display**: when re-displaying tool args to the LLM (e.g., showing the previous turn's tool call), confirm they're string-escaped properly, not just `str()`-ed.

- **Sentinel-substring control-flow trigger** (variant — same threat shape, different sink): When the diff branches control flow based on whether a hard-coded token appears as a substring of user-supplied content — e.g. `if "[CONCLUDE_TOPIC]" in message:` or `if "[FORCE_SKIP]" in transcript:` — a student / external caller can write the token in chat to trigger the privileged flow. The sink isn't an LLM in this case but the threat model is identical: untrusted content driving control. Trust only structured metadata flags (`metadata.conclude_topic == True`), not substring matches against the message body.
  - Example caught: progress-decision-ui PR had `if "[CONCLUDE_TOPIC]" in message:` next to `metadata.get('conclude_topic')`. Fix: drop the substring check; trust only the metadata flag.

- **Dynamic values serialized into a delimited / structured text format** (GFM table, CSV/TSV, URL query string, JSON-in-a-string): when the diff BUILDS such a format by joining authored/dynamic values with a delimiter, the value can contain the delimiter and fracture the structure. This is the same "rogue character breaks framing" failure as the rest of this sweep, and it bites even when the sink is the frontend rather than an LLM — so check it wherever the delimited output is built, not only on LLM-bound strings. Three facets that co-occur at the SAME build site; audit all three:
  1. **Escape the delimiter.** A cell/field value containing the structure's separator (`|` for GFM tables, `,`/newline for CSV, `&`/`=` for query strings) must be escaped or the format breaks. (URL-interpolation has its own bullet in Sweep 9 — `encodeURIComponent`; this generalizes it to every delimited format.)
  2. **Normalize row/field arity.** When rows are emitted under a fixed header (or fields under a fixed schema), pad/truncate each row to the header width so a short or long row doesn't misalign every column after it. Same shape as the "inconsistent return-tuple arity" check in Sweep 1, applied to serialized output.
  3. **Don't let normalization defeat a stated placeholder intent.** If a comment says "keep blank cells as a space (for fillable tables)" but the code does `str(c).strip()`, a whitespace-only cell collapses to `""` — the comment now lies (cross-ref Sweep 6 comment-vs-impl). Make the placeholder branch cover `None`, `""`, AND whitespace-only.
  - Example caught (Copilot, PR #224): `_render_concept_content`'s new `table` branch did `"| " + " | ".join(str(c).strip() ... for c in r) + " |"` — (1) a curriculum cell/header containing `|` injected phantom columns; (2) rows shorter than `headers` misaligned; (3) a `"   "` cell stripped to `""` despite a "keep blank cells as a space" comment. Fix: a `_cell()` helper that escapes `|`, collapses newlines, and returns `" "` for blank/whitespace-only values, plus `cells = (cells + [" "] * ncols)[:ncols]` to align to header width.
  - Method: for every `delimiter.join(...)` / f-string that builds a row of a structured format from non-literal values in the diff, confirm each value is delimiter-escaped, the arity matches the header/schema, and any "blank/placeholder" comment is actually honored by the code.

**What "sanitized" looks like** (any of, in order of strength):
- Function call to a known sanitizer in this codebase (e.g., `_sanitize_for_prompt`, `_safe_field`, `bleach.clean`, custom helpers)
- Explicit chain: `.replace("\n", " ").replace('"', "'").replace("[", "(")...` + truncation
- Use of structured parameter passing (the value is a tool arg / JSON field, not inlined into a free-text prompt)
- The source is itself a known-safe string (a literal, an enum value, an opaque ID like a UUID).

**What "sanitized" does NOT mean**:
- HTML escape only (`<` → `&lt;`) — doesn't neutralize quote/bracket/newline framing escapes
- Length truncation only — a short payload still injects
- Whitespace strip only — quotes/brackets remain

For each finding:
```
[INJECTION] <file>:<line>
  Embedded:    <expression, e.g. seg.get('title') / user_message>
  Source:      <plan-authored / student input / DB row / filename / other untrusted>
  Sink:        <which LLM-facing string — quote the framing words around the {}>
  Sanitizer:   none  |  <existing helper in this repo that should be used>
  Risk:        <what the worst case payload could do to framing/instructions>
```

**Calibration**: don't flag interpolations of values you can prove are application-controlled (UUIDs, integer indices, enum-string constants, hard-coded names). Don't flag interpolations into LOG strings (those don't reach the LLM). Don't flag interpolations into HTTP responses to the frontend *for the injection threat model* (the model can't be instructed there) — BUT the delimited/structured-serialization check above (delimiter escaping, arity, placeholders) still applies to frontend-bound output, because there the failure is broken *structure*, not injected *instructions*. The bar for injection is: **does this string get fed to an LLM, and does the value come from outside the application's control surface?** The bar for structure-breakage is: **is a dynamic value joined into a delimited format whose separator it might contain?**

---

#### Sweep 6 — Ghost references & unfulfilled doc promises

**Goal**: Catch two related classes of "code lies about itself" — references to methods/imports that don't exist anywhere (silent AttributeError on the hot path), and doc-comments that promise behavior no code actually delivers. Both stem from the author writing the surface (call site, doc, field) before the implementation, then forgetting to come back.

These are the most embarrassing class of finding when a PR reviewer (or production) catches them. They sail through typecheck if the language is dynamic (Python `self.X()`, JS dynamic dispatch), and they sail through human review because the author KNOWS what the code is *supposed* to do.

**Method**:

1. **Ghost method/function calls** — For each call to `self.X(...)`, `this.X(...)`, `obj.X(...)`, or a bare `X(...)` added in the diff, verify a definition for `X` exists somewhere reachable:
   - Same file (`def X`, `function X`, `const X = ...`, `class X`)
   - Mixin / parent class for Python (`class Foo(BarMixin, BazMixin): ...` — grep each base for `def X` / `async def X`)
   - Imported modules referenced in the file
   - Prototype chain or class hierarchy for JS/TS

2. **Ghost imports** — Symbols imported (`from x import y`, `import { y } from 'x'`) but the source module doesn't export them.

3. **Doc promises unfulfilled** — For each new field / flag / parameter ADDED in the diff whose doc-comment uses action verbs ("bypasses", "overrides", "triggers", "forces", "enables", "disables", "blocks", "skips", "auto-X"), search for a READER that implements the documented effect. Writers + storage-only readers don't count — find code that *acts on* the flag in the way the comment describes. If none, flag.

**Concrete checks**:

- **Python `self.X()` ghost call**: For each new `await self.X(...)` or `self.X(...)`, run `grep -rn "def X\|async def X"` across the file's class and every base/mixin it inherits from. Zero results → flag.
  - Example caught: `await self._classify_topic_advance_intent(state, message)` added in the NL-intent branch; zero defs anywhere in the repo → AttributeError on every NL turn during a pending decision (production-breaking).

- **TS/JS `hook.method()` ghost call**: For each `someHook.method()` where `someHook` is a hook/factory result, verify `method` appears in the hook's return type / `UseXResult` interface / factory return. Especially common right after a hook rename or API expansion.

- **Doc-comment promise vs reader**: For each new boolean field with a doc-comment like `# bypasses the X gate` or `// overrides Y check`, grep the codebase for the field name and confirm at least one site uses it to perform the described action. If readers exist but only for storage/passthrough/guard purposes (not the documented effect), still flag.
  - Example caught: `force_skip: Optional[bool] = None` documented as "User-sacrosanct override: bypasses the backend video-watched gate." Readers in `tool_executor.py` (where the gate lives): zero. Only readers anywhere: a self-loop in `message_processor.py` using it as an "already decided" guard. The doc lies.

- **Function-docstring promise vs implementation**: For each new function whose docstring claims it "mirrors X", "matches Y", "is consistent with Z", "strips W", "handles V", or otherwise asserts equivalence/inclusion of a specific behavior — open the referenced X/Y/Z and diff the operations. If the docstring claims behavior the implementation skips, flag.
  - Method: read the docstring's verbs ("mirrors", "matches the fallback", "strips markdown", "validates input"), then open the function and confirm each verb has corresponding code. Cross-reference any sibling function the docstring names — does its pipeline really match?
  - Example caught: `quick_normalize_speech` docstring said "same regex pipeline as the fallback branch of normalize_for_tts" but the implementation skipped the `_strip_md_to_words` step that `_build_identity` (the actual fallback) performs first. Result: markdown markers leaked into TTS, contradicting the docstring.

- **Comment asserting unverifiable cross-repo consistency**: a comment claims the code "matches the frontend's X" / "mirrors the client's Y" / "same as the other service's Z" when the referenced implementation lives in a DIFFERENT repo or module the diff can't see. Either the claim is unverifiable from here (so it silently rots when the other side changes), or — worse — it describes behavior THIS function doesn't even perform. Rewrite the comment to describe local behavior only; if a cross-repo invariant genuinely matters, verify it against the other repo and state it as a *checked* dependency, not an unverified assertion.
  - Example caught (Copilot, PR #187): a `_strip_md_to_words` comment said it "matches the frontend's isTableLine test (/^\s*\|/)" and that "speech text excludes it" — but no frontend code exists in this backend repo to verify, and the function only affects `display_words`. Fix: describe the local drop-table-lines behavior; drop the unverifiable frontend claim. (When actually checked against the frontend's `matchBackendWord`, the invariant DID hold — but the comment had no business asserting it unverified.)

- **Imported-but-undefined**: For each new `from X import Y`, verify Y is in X's exported surface. (For Python, check `__all__` or top-level defs; for TS, check the actual export.)

**Calibration**:

- Don't flag obvious dynamic dispatch: `getattr(self, "X", ...)`, `obj["X"]`, `**kwargs` unpacking, `Mapping`/`dict` attribute access via `__getattr__`. These are intentional.
- Don't flag flags whose doc honestly describes "stored for downstream consumers" / "metadata-only signal" / "passed through to X" — that's accurate self-description, not a promise.
- Don't flag method calls where the implementation is in an extension file the diff doesn't touch — grep the whole repo before flagging, not just the file.
- The bar for "doc promise unfulfilled" is: the doc uses an *action verb* describing a *runtime effect*, AND no code path implements that effect. Descriptive doc-comments ("this field is true when the user clicked Next Topic") are NOT promises.

For each finding:
```
[GHOST] <file>:<line>
  Reference:   <expr being called, or flag being documented>
  Kind:        ghost-method-call | ghost-import | unfulfilled-doc-promise | docstring-vs-impl-mismatch | unverifiable-cross-repo-claim
  Expected:    <what should exist — a def somewhere, an export, a reader that implements the doc, code matching the docstring>
  Found:       none  |  only-writers  |  only-storage-readers  |  <file:line of unrelated reader>  |  implementation diverges from docstring
  Risk:        AttributeError on <condition>  |  silent no-op vs documented behavior  |  misleads future readers into wrong assumptions
```

---

#### Sweep 7 — Hardcoded secrets / credentials

**Goal**: Catch passwords, API keys, DB connection strings, internal IPs paired with credentials, and `print("password: ...")` statements committed to source. Git history keeps them forever — deleting later doesn't undo the leak, and credentials in the repo tend to get reused or copied elsewhere before anyone notices.

**Method**:

1. For every line ADDED in the diff that's source code (not `.env.example`, not a test fixture), grep for credential-shaped patterns:
   - Password-like assignments: `password\s*=\s*['"][^'"<${]{6,}['"]`
   - Common secret keys: `api_key|apikey|secret|token|access_token|client_secret|private_key|aws_secret|client_id`
   - Connection strings with embedded credentials: `(postgres|mysql|mongodb|redis|amqp)://[^/]*:[^@/]+@`
   - Hardcoded IP + credentials in the same dict/call: `host=['"][0-9.]+['"]` within 5 lines of `password=`
   - Print/log statements that quote a real-looking password: `print\(.*[Pp]assword.*['"][^'"]{4,}['"]`
   - JWT-shaped strings: `eyJ[A-Za-z0-9_-]{20,}\.eyJ[A-Za-z0-9_-]{20,}\.`

2. Filter false positives — don't flag if:
   - File path matches `*.test.*`, `*_test.py`, `tests/**`, `*.spec.*`, `.env.example`, `conftest.py`, `fixtures/**`
   - Value is an obvious placeholder: `<password>`, `${PASSWORD}`, `your-key-here`, `xxx`, `***`, `changeme`, all-zero, all-one
   - The line is a comment documenting the *format* of a credential, not a real one
   - The value reads from an env var: `os.getenv(...)`, `process.env.X`, `os.environ['X']`

**Calibration**: if the value looks real-shaped (length ≥ 6, mixed character classes) in a non-test file, flag it. False positives on `password="test"` in non-test code are still correct flags — those tend to be real-looking by accident, and the cleanup is cheap.

For each finding:
```
[SECRET] <file>:<line>
  Pattern:     password | api_key | conn_string | hardcoded_IP_with_user | jwt | print_of_password
  Excerpt:     <line, with the value masked: password="****">
  Risk:        Credential leak persists in git history even after later deletion.
               Anyone with repo read access (incl. forks, public mirrors) has the value.
  Fix:         Move to env var (`os.getenv(...)`), .env (gitignored) / .env.example with placeholder,
               or a proper secret manager. Rotate the credential if it was already pushed.
```

---

#### Sweep 8 — Severed signal chains

**Goal**: Inverse of "ghost reference" — catch *removed* writes / assignments / event emits whose downstream readers / listeners still depend on the producer firing. The original write went away but the consumers didn't, so a flag never gets set, an event never fires, a worker never terminates. These are particularly nasty because typecheck passes (the field/key still exists), tests pass (they often mock the consumer), and the bug only manifests in production under the exact condition the missing write was supposed to signal.

**Method**:

1. Grep the diff for **REMOVED** lines matching producer-shaped patterns:
   - State writes: `state\.X\s*=`, `self\.X\s*=`, `obj\.X\s*=`
   - Env / dict mutations: `os\.environ\[['"]X['"]\]\s*=`, `d\[['"]X['"]\]\s*=`
   - Event emits / publishes: `(emit|publish|dispatch|send|broadcast)\(['"][^'"]+['"]`, `await\s+\w+\.send_json\(\{['"]type['"]:\s*['"][^'"]+['"]`
   - SSE yields: `yield\s+(f|b)?['"]event:\s*\w+`
   - WebSocket / Slack / pub-sub sends

2. For each removed producer, grep the **whole repo** (not just the file) for readers / consumers / listeners of the same key/attr/event-name:
   - Attribute reads in other files: `state\.X`, `self\.X`, `obj\.X`
   - Dict reads: `os\.getenv\(['"]X['"]\)`, `d\.get\(['"]X['"]\)`, `d\[['"]X['"]\]`
   - Event handlers: `on\(['"]X['"]`, `addEventListener\(['"]X['"]`, `if\s+event_type\s*==\s*['"]X['"]`, `if\s+msg\.get\(['"]type['"]\)\s*==\s*['"]X['"]`

3. If readers exist (especially in *different* files / modules), flag.

**Concrete patterns the consumers usually look like**:

- **Worker-loop termination signal**: a background loop / worker that breaks on a flag (`while not state.done:`), and the diff removed the only line that ever sets `state.done = True`. The worker now runs until `max_turns` / timeout / OOM.
  - Example: `state._lesson_concluded = True` removed; `ai_student_agent/worker.py` still loops `while not state._lesson_concluded`. Worker runs until `max_turns`.
- **Frontend listening for an event the backend stops emitting**: WebSocket `{"type": "lesson_concluded"}` removed; frontend `if msg.type == 'lesson_concluded'` handler still wired. UI hangs in "thinking" state.
- **Telemetry / dashboard query**: a write to a structured-log field removed; Grafana/Datadog queries reference the field. Dashboards go silent or alert.
- **Cache invalidation key**: a write that triggers cache eviction removed; readers still hit stale cache forever.

**Calibration**:

- Don't flag if the consumers were ALSO removed in the same diff (clean refactor, producer + consumer co-deleted).
- Don't flag if consumers use the value defensively with a documented fallback (`getattr(state, 'X', default)` where `default` is the intended path).
- High-confidence flag when readers are in a different file or module from the removed write (cross-file coupling = exactly the case humans miss).
- High-confidence flag when the field name has signal-language semantics (`_done`, `_concluded`, `_finished`, `_completed`, `_terminated`, `_ready`, `_aborted`, `_canceled`).
- High-confidence flag when the removed line had a comment explaining *why* it was setting the flag (the comment is evidence other code reads it).
- **If the removed producer line is something the author did not intend to remove**, that's a *diff-leak* — see Sweep 11. The severed-signal finding here is the consequence; the unintentional removal is the cause. Reverting the leaked hunk fixes both.

For each finding:
```
[SIGNAL-SEVERED] <file>:<line>  (removed producer)
  Removed:     <the deleted line, e.g. state._lesson_concluded = True>
  Producer kind: state-write | event-emit | sse-yield | websocket-send | dict-set
  Readers:     <file:line list — readers that still depend on this firing>
  Risk:        Downstream branches on this signal; producer gone means
               <expected behavior — termination / UI update / dashboard / cache evict> never fires.
               Possible silent infinite loop / hung UI / wrong default path.
  Fix:         Either restore the producer, OR remove the consumer too if the signal is genuinely obsolete.
```

---

#### Sweep 9 — Boundary hardening on untrusted client / external input

**Goal**: Wherever data crosses an untrusted edge into your code (HTTP route handler receiving JSON body, WebSocket inbound message, third-party API response, browser `localStorage`/`sessionStorage` value, query-param value), server-side defenses are easy to forget — you trust the shape because tests use clean inputs. Production gets messy inputs. This sweep catches the cases where the diff added a *new* boundary or a *new* field at an existing boundary without the corresponding hardening.

**Method**:

1. For each new HTTP route handler, WebSocket message handler, or function reading from external storage in the diff, list the fields it reads from the request/payload/storage.
2. For each field, ask:
   - **Type tight enough?** Is `action: str` actually `action: Literal["click", "zoom", "step_complete"]`?
   - **Length capped?** Is there a `len(x) > MAX_LEN` rejection or truncation before the value is stored / interpolated / forwarded?
   - **URL-encoded if it ends up in a URL?** When the field is later interpolated into a fetch URL or WebSocket URL (`ws://.../${field}`), is `encodeURIComponent` applied?
   - **Canonicalized if a cross-system identifier?** Lowercased / trimmed / format-normalized before being used as a lookup key in storage that other parts of the system might use with different casing?
   - **Negative-cached if a failable fetch?** Failed fetches re-attempted from a hot path will hammer the network every request.

**Concrete checks**:

- **Type-too-loose param on inbound route**: `action: str` accepting arbitrary text where the server actually only handles 3 values. Replace with `Literal[...]` / Pydantic enum so unknown values 422 at parse time instead of falling through.
  - Example caught: `tool_event_routes` accepted `action: str` but server only knew `click | zoom | step_complete`. Tightened to `Literal[...]` so other actions don't silently no-op.
- **Missing length cap**: client-supplied `description`, `tool_id`, `event_kind`, or any free-text field that ends up in a prompt / DB row / log / structured message without a hard cap. Even if the field doesn't directly inject, an attacker can blow up memory or storage with a 1MB string.
- **Missing URL-encoding** in URL construction: `ws://${BASE}/${studentId}?voice=${voice}&engine=${engine}`. If `studentId` is `"alice/eve"` or `voice` is `"male voice"`, the URL fragments and the routing is wrong.
  - Example caught: `useLandingVoiceAgent` / `useStudentVoiceAgent` built WS URLs without `encodeURIComponent` on studentId/voice/engine.
- **Cross-system identifier casing/format mismatch**: backend registers `tool_id` lowercased; frontend follow-up actions (`highlight`, `set_mode`) send mixed-case tool ids. Backend lookup misses. Fix: canonicalize on both sides, or canonicalize on read.
  - Example caught: `tool_executor` added `_canonical_3d_tool_id()` and used it in `highlight` / `set_mode` / `set_interaction` so frontend casing variations map to the same registered id.
- **Regex parsing brittleness on inputs with optional fragments**: `wsBaseURL.replace(/^http/, 'ws')` works until the URL has a trailing slash, port, or path. Strip trailing slash; or use the URL parser.
- **No negative-cache on failed external fetches**: a function that fetches a config / model / file from an external source on every request, with no caching of failed results, will retry the failed fetch on every call.
  - Example caught: `tool_3d_config_service` fetched config on every prompt build; if the URL was broken, every prompt build hit the dead URL. Added short-TTL negative cache.
- **localStorage / sessionStorage read without same-tab subscription**: a React component that reads `localStorage.getItem('accessToken')` to gate a fetch. Adding only a `storage` event listener catches OTHER tabs' writes but not the same tab. The same tab updating the token won't trigger a re-render.
  - Fix: dispatch a custom `auth-token-changed` event from the writer and subscribe to it alongside `storage` + `focus`.
  - Example caught: `ModelSelection` listened only for `storage` + `focus`; same-tab login didn't refetch models until manual refocus.

**Calibration**:

- Don't flag if the field is a known-safe enum from an internal-only producer (a backend-to-backend call where both sides are your code).
- Don't flag if the field is already validated by a Pydantic model / TypeBox schema in the diff.
- Don't flag URL-encoding when the value is provably ID-shaped (UUID literal, numeric).

For each finding:
```
[BOUNDARY] <file>:<line>
  Field:       <name of the field crossing the boundary>
  Edge:        HTTP-inbound | WS-inbound | localStorage | external-fetch | URL-construction
  Issue:       type-too-loose | no-length-cap | not-url-encoded | casing-mismatch | regex-brittle | no-negative-cache | same-tab-sync-missing
  Risk:        <DoS via memory exhaustion | routing breaks | cross-system lookup miss | hot-path network thrash | stale UI>
  Fix:         <Literal[...] | length cap | encodeURIComponent | _canonical_X helper | negative-cache TTL | custom event subscription>
```

---

#### Sweep 10 — Async / render-hot-path hygiene

**Goal**: Code that runs hot — inside an async coroutine, inside a React render body, inside a streaming-chunk callback — must be cheap and side-effect-clean. The diff likely added a new hot path or a new operation inside an existing one; this sweep catches the operations that *don't belong* there.

**Method**: For each function or callback the diff adds/modifies that runs on a hot path, audit the body for these specific operations.

**Concrete checks**:

- **Sync I/O inside async code**: `requests.get(...)`, `httpx.Client(...)` (sync), `time.sleep(...)`, blocking file I/O, blocking DB driver calls inside an `async def`. The async event loop stalls for the duration. Use `await client.get(...)` (async client), `asyncio.sleep`, or wrap in `asyncio.to_thread(fn, *args)`.
  - Example caught: `tool_3d_config_service` did a blocking `httpx.get` inside the prompt-build path. Wrapped in `asyncio.to_thread`.

- **External-state read in React render body**: `localStorage.getItem('x')`, `sessionStorage.getItem('x')`, `document.cookie`, `window.X` read directly from a render. React doesn't subscribe — the component won't re-render when the external value changes. Wrap in `useState` + storage/custom-event listener.
  - Example caught: `ModelSelection` read `accessToken = localStorage.getItem('accessToken')` directly in render. Token rotation didn't trigger model refetch until next mount.

- **Heavy compute on every stream chunk**: regex/parse/format inside an SSE chunk handler, an `onmessage` callback, or a React render that fires on every chunk. Defer until stream completes, or memoize on a coarser key.
  - Example caught: `formatTextWithParagraphs(text)` ran inside the assistant-message render path during streaming, re-doing regex passes on every chunk. Skip during streaming; format only after the stream completes.

- **Quadratic-loop pattern**: `binary += String.fromCharCode(uint8[i])` for a large buffer (string-concat in a loop is O(n²) in most JS engines). Same shape: `list += [item]` instead of `list.append(item)` in Python (less catastrophic but still slow). Same shape: building HTML strings by `+=` in a loop.
  - Fix: chunk it (`String.fromCharCode(...uint8.subarray(i, i + chunkSize))` then `.join('')`), or use a builder pattern.
  - Example caught: PCM-to-base64 conversion in TeacherChat used a char-by-char accumulator on 24kHz PCM buffers (hundreds of KB) — UI froze briefly on every audio store. Replaced with chunked `pcmToBase64()`.

- **Read-before-write ordering within a closure**: A closure that needs to capture the PREVIOUS value of a ref / state must do the read BEFORE the line that updates it. Reading after means both reads see the new value.
  - Method: for each `ref.current = ...` or `setX(...)` in a callback, scan the same callback for later reads of the same ref/state. If a "previous" semantic is needed (forward-jump coverage, undo history, change detection), capture first.
  - Example caught: word-highlight forward-jump logic did `const prev = lastHighlightWordRef.current` AFTER updating the ref. Both lookups returned the new index; forward-jump coverage broke. Fix: capture `prevHighlight` at the top of the callback.

- **Cross-tab / same-tab state sync incomplete**: The `storage` event fires only in *other* tabs. Same-tab writes don't trigger it. If your code expects state changes to propagate within the same tab (e.g. an in-app login flow updating the auth token), dispatch a custom event from the writer.
  - Example caught: `authSlice` wrote `accessToken` to localStorage but didn't `dispatchEvent(new Event('auth-token-changed'))`. `ModelSelection` listened only for `storage` → same-tab login didn't refetch.

**Calibration**:

- Don't flag a single regex / format call in render *if it's already memoized* with `useMemo` keyed on the actual input.
- Don't flag sync I/O on a one-shot startup path or in a CLI script — async hygiene matters most in server / hot paths.
- Quadratic loops on small inputs (`n < 100`) are fine; the bar is "is this on a path that processes user-scale data".

For each finding:
```
[HYGIENE] <file>:<line>
  Path:        async-coroutine | react-render | stream-chunk-handler | callback-closure | localStorage-writer
  Issue:       sync-io-in-async | external-state-in-render | heavy-compute-per-chunk | quadratic-loop | read-after-write | same-tab-sync-missing
  Risk:        event-loop-stall | stale-UI | UI-jank | quadratic-time | wrong-prev-value | UI-out-of-sync-with-storage
  Fix:         <asyncio.to_thread | useState + storage listener | defer until stream done | chunk the loop | capture-before-update | dispatch custom event>
```

---

#### Sweep 11 — Diff scope hygiene

**Goal**: Every hunk in the diff should be explained by the commit message / PR description / task at hand. Hunks the author can't justify are usually accidents — a `git checkout <other-branch> -- <file>` shortcut that leaked an entire base-branch's worth of edits, a stray formatter / linter pass, a stale debug print left from local testing, a generated lockfile / declaration file picked up by `git add -A`, or a botched rebase that re-applied a removal from the base branch.

**Why it matters and why it runs first**: If a chunk of the diff is *unintentional*, the author has no mental model of it and won't notice issues with it — but the other ten sweeps will surface those issues, and the author will spend cycles trying to understand findings on code they never meant to touch. Catching the leak here is one revert; catching it via Copilot after a force-push round-trip is a fix-rebase-rebroadcast cycle.

This sweep is upstream of several others. Past incident in this repo: a one-line SKIP_TO_NEXT_SEGMENT fix was force-pushed onto a PR branch whose `prompt_builder.py` had been copied from a staging-based branch via `git checkout b2590dd -- app/agents/native_teacher_agent/prompt_builder.py`. The file copy brought along staging's other edits — including a removed `state._available_image_titles = …` assignment that the author had no idea was in the diff. Copilot flagged it as a severed signal chain (Sweep 8) on the PR; the fix was a second force-push that reset the branch to dev_v1 and applied only the intended hunks via `git diff … | git apply`.

**Method**:

1. Run `git diff <base>..HEAD --stat` to list every changed file. Read it in full — every entry should be a file you remember editing.
2. For each file you don't expect to see changed, run `git diff <base>..HEAD -- <file>` and read every hunk.
3. For each hunk in every file (yes, including the ones you DO expect), ask: *Is this change explained by the commit message? Did I intend to make this specific edit?* If the answer is "no" or "I don't remember adding/removing this," flag.
4. Also check the commit boundary: `git log --oneline <base>..HEAD` — every commit's title should describe what that commit's hunks actually do.

**Concrete checks**:

- **Cross-branch file copy** (the canonical cause): `git checkout <other-branch> -- <file>` copies the *whole* file from the other branch, not just the change you wanted. If the destination branch is on a different base, every edit the other base made to that file leaks in. Same shape: `cp ../other-worktree/foo.py .`, `gh pr checkout <pr> && cp -r src/ ../mybranch/`. Symptom: hunks in your diff that touch areas of the file you have no memory of editing.
- **Stale debug code**: `console.log(…)`, `print("HERE")`, `pdb.set_trace()`, `debugger;`, `// XXX`, `# TODO(me)`, `assert False, "checkpoint"`, commented-out blocks left behind during local iteration.
- **Stray formatter / linter pass**: tabs↔spaces flips, trailing-whitespace strips, import reorders, `// prettier-ignore` removals — anywhere the diff touches lines whose visible content is unchanged. Most editors do this on save; a project without a committed formatter config gets you both directions.
- **Generated-file noise**: `package-lock.json` / `pnpm-lock.yaml` / `yarn.lock` / `Cargo.lock` / `go.sum` bumps with no matching change to the manifest file; auto-generated TypeScript declarations (`*.d.ts` under `dist/`); regenerated protobuf / GraphQL clients; OpenAPI snapshots. Lockfile bumps are usually intentional after `npm install`, but should accompany a `package.json` edit. A lockfile bump with no manifest change is suspicious.
- **Merge / rebase artifacts**: a hunk that's actually re-applying or re-removing a change because of a botched rebase or three-way merge resolution. Symptom: the hunk reverts (or re-adds) something the base branch already has the other way.
- **Test-only changes leaking into production paths**: changes in `src/` paired with no `tests/` change (or vice versa) where you'd expect them coupled.
- **Crossover from a sibling branch**: edits made while accidentally on a different feature branch, then carried over.

**Detection cues**:

- `git diff <base>..HEAD --stat` shows a file whose name doesn't appear in your commit message → high-confidence flag.
- A hunk's `+`/`-` lines mention identifiers (function names, fields) that aren't referenced anywhere in the commit message → flag.
- `git log -p <base>..HEAD -- <file>` shows the file changing across multiple commits with no logical thread → suspect a merge artifact or rebase mishap.
- A hunk removes a line containing `state.X = …`, `self.X = …`, or any other write, and you don't remember making that decision → flag immediately, and also rerun **Sweep 8** because the *consequence* of an accidental removal is a severed signal chain.

**Calibration**:

- Don't flag re-flows of unchanged content that the editor expanded for context (`unified=3` defaults around your real change).
- Don't flag clearly-mechanical hunks that obviously accompany your intent (e.g. a function rename rippling through 12 call sites — the call-site edits aren't "unintentional" even though you didn't think about each one individually).
- Lockfile updates are fine *if* there's a manifest change in the same commit explaining them.
- An automated codemod pass is fine if the commit message says so.

**For each finding**:
```
[DIFF-LEAK] <file>:<lines>
  Stated intent (commit/PR):  <one-line summary of what you said you were doing>
  Actual change in this hunk: <what the diff shows here>
  Likely cause:               cross-branch-copy | stale-debug | formatter | generated-noise | merge-artifact | crossover-from-sibling-branch | unknown
  Related downstream finding: <if this leak caused a Sweep 8 severed-signal hit, or a Sweep 1 inconsistency, link the tag here>
  Risk:                       <what could break unintentionally — usually whatever the leaked hunk would do if it shipped>
  Fix:                        revert the hunk | move to its own PR | document in the commit message
```

---

#### Sweep 12 — LLM prompt-instruction coherence

**Goal**: Catch self-contradictory or ambiguous instructions WITHIN a prompt/instruction string the application authors and feeds to an LLM. Distinct from Sweep 5 (untrusted *data* flowing into a prompt) — here the prompt's own *rules* fight each other. When a diff adds a rule to an existing prompt, the new rule often contradicts a standing one; the model then obeys inconsistently (sometimes the old rule wins, sometimes the new), and the bug is non-deterministic and miserable to reproduce.

**Method**:

1. For each diff hunk that ADDS or EDITS a line inside a prompt/instruction string (system prompt, seeded DB prompt in a `seed_*_prompts.py`, tool description, classifier instruction), read the WHOLE surrounding prompt — not just the hunk.
2. State the new/changed rule as a plain imperative ("drop table rows from speech").
3. Scan the rest of the prompt for any standing rule it contradicts or narrows — especially absolutes ("ALWAYS", "NEVER", "EVERY", "the SAME information", "do NOT drop ANY").
4. If the new rule is an exception to a standing absolute, confirm the absolute was amended to name the exception. If the two coexist unqualified → flag.

**Concrete checks**:

- **Absolute rule + new unreconciled exception**: a standing rule says "ALWAYS / NEVER / EVERY / do NOT drop ANY" and the new hunk carves out a case without editing the absolute to say "(SOLE EXCEPTION: …)". A literal-minded model can't tell which wins.
  - Example caught (Copilot, PR #187): a TTS-normalizer prompt said "speech must contain the SAME information as display" and "do NOT drop ANY original word", then a new hunk added "OMIT all markdown-table rows from speech." Copilot (Medium): the rules contradict → the model may keep the table to satisfy the older rule, or drop it, inconsistently. Fix: amend the absolutes to "(SOLE EXCEPTION: markdown-table lines)" AND frame the new rule as "the ONE and ONLY case where you may drop words, overriding 'same information' / 'do not drop'."
- **Conflicting format directives for the same output**: "respond in one continuous line, no newlines" added near "render as a GFM table" (tables REQUIRE newlines). Confirm each directive is scoped to the right section (e.g. SPEECH vs DISPLAY) rather than both governing the same output.
- **Two prompts for one pipeline drifting apart**: when the same behavior is described in two seeded prompts (e.g. the teacher prompt says "render tables"; the normalizer prompt must say "drop tables from speech"), confirm they agree rather than both claiming to speak/keep everything.
- **Numbered-rule self-collision**: a new clause appended to rule N permits what rule M forbids (rule 4 newly allows a table while rule 1 says "no non-prose blocks").

**Calibration**:

- Don't flag rules scoped to clearly different sections — a "keep newlines" rule under DISPLAY TEXT and a "one line, no newlines" rule under SPEECH TEXT are NOT in conflict; they govern different outputs.
- Don't flag preference + permission ("prefer short turns" coexisting with "you MAY use a table when it helps") — that's not a contradiction.
- The bar: two rules a literal-minded model could read as applying to the SAME output at the SAME time with OPPOSITE directives, where neither yields.

For each finding:
```
[PROMPT-CONFLICT] <file>:<line>
  New/changed rule: <the rule the diff added, as a plain imperative>
  Conflicts with:   <standing rule it fights, file:line — quote the absolute words "ALWAYS/NEVER/EVERY/SAME">
  Scope:            same-output (real conflict) | different-sections (not a conflict — don't flag)
  Risk:             model obeys inconsistently — non-deterministic output; the new behavior fires only sometimes
  Fix:              amend the standing rule to name the exception ("(SOLE EXCEPTION: …)") AND mark the new rule as the override; or scope each rule to its section explicitly
```

---

### Phase 3 — Output the verdict

After all twelve sweeps, output a single summary block. Use this format exactly:

```
## self-review verdict

Files audited: N
Findings: <count> across <categories>

[DIFF-LEAK]        <count>
[INCONSISTENCY]    <count>
[LIFECYCLE]        <count>
[SIBLING]          <count>
[DOC DRIFT]        <count>
[INJECTION]        <count>
[PROMPT-CONFLICT]  <count>
[GHOST]            <count>
[SECRET]           <count>
[SIGNAL-SEVERED]   <count>
[BOUNDARY]         <count>
[HYGIENE]          <count>

Recommendation:
  ✅ READY TO PUSH — no findings
  OR
  ⚠️ FIX BEFORE PUSH — N findings above
  OR
  💡 REVIEW — N low-severity findings; your call
```

For each finding above the verdict, output the specific [TAG] block from the sweep. Don't paraphrase — be exact about file paths, line numbers, and risk.

If there are 0 findings, say so explicitly. **Don't fabricate findings to look thorough.** A clean diff getting a clean verdict is the correct outcome.

## Things to NOT do

- Don't run typecheck / tests as part of this skill (the user has separate workflows). This skill is for the class of bugs typecheck doesn't catch.
- Don't propose architectural changes. Stick to what the diff already touches.
- Don't comment on style/formatting if the project has a formatter. Stay focused on correctness.
- Don't ask the user clarifying questions DURING the sweep — gather everything first, ask only at the end if a finding needs disambiguation.

## Calibration — what good findings look like

A useful finding is **specific**, **actionable**, and points to **the exact file:line** plus **the exact diverging sibling**. Generic warnings ("consider error handling") are not findings. If you can't point to a sibling site or a missing cleanup site, the finding doesn't ship.

## Why this exists

These categories are the ones the human author of a change almost never catches in their own review — they have confirmation bias from knowing what they intended. A fresh pass that doesn't know the intent catches them reliably. This skill is that fresh pass.

Every entry on the list was added because an automated PR reviewer (Copilot, code-rabbit, etc.) caught it on a real PR after the human reviewer missed it — and the bug-class examples cited above are real, mined from this repo's git history (`fix: address Copilot review …` / `refactor: adopt Copilot's …` / `address PR review` commits). If a future automated review catches a NEW class that doesn't fit any sweep here, add it — that's the gardening cycle for this skill.
