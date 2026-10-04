# Coding-standard audit — Harmony Kira

Scope: all packages in `harmony-browser` (ai, ai-client, ai-harness, ai-protocol,
browser, core, shell). Method: read + pattern survey, not just the loop lint.
Verdict up front: the *principles* are mostly sound; the problem is **inconsistent
application** and **prose-padded comments**, not the ideas.

Honest framing: "match KCoreAnimation" is a fine heuristic but it is not a
standard — it is one package's output. Below I say where I think the idiom is
right, and where it is cargo-culted.

---

## F1 — Manual counter loops (KLINT002)  ·  real, mechanical

`var i = 0 / while i < xs.count { … i = i + 1 }` where a `for` says it plainly.
59 instances repo-wide at survey time. The toolchain already has the fix:
`kira lint <pkg> --warn=KLINT002 --fix`.

- Fixed by hand: **ai-harness** (all of it), verified compiling.
- Clean already: ai-protocol, ai-client.
- **Blocked**: ai, browser, core, shell do not currently compile (pre-existing
  WIP + a broken external dep `opacity-ui`), and `--fix` refuses a package that
  does not compile. Hand-editing a non-compiling tree is flying blind — a real
  mistake would hide behind the existing errors. These wait until they build.

## F2 — One concept, two standards: `ConversationId`  ·  the worst smell

`core/Protocol/Ai.kira:65` mints `distinct ConversationId = Int` with a paragraph
on *why* (an identity, not a count). The other AI protocol — `ai-protocol`,
`ai-harness`, `ai-client` — carries the exact same concept as bare `Int`, 22
sites. So the codebase argues both sides of its own rule. This is the clearest
evidence the "standard" is applied by vibe, not enforced.

Fix: mint `distinct ConversationId = Int` in `ai-protocol` and thread it through
the harness and client. All three compile, so it is verifiable. ~22 edits.

## F3 — Hand-rolled deep-copy helpers  ·  blocked, not the harness's fault

`ai-harness/app/Agent.kira` hand-writes `copyMessage/copyInputs/copyOutputs/
copyToolResults` — the exact boilerplate `@Derive(Clone)` generates. But the
types copied (`Message`, `Input`, `Output`, `ToolResult`) live in the external
`ai-sdk`, and nothing there derives `Clone`. So the boilerplate is the *cost of
the SDK not deriving Clone*, not a harness choice. Right fix is upstream:
`@Derive(Clone)` on those SDK types, then delete the helpers. Until then the
hand copies stand.

## F4 — Manual JSON-schema assembly  ·  acceptable

`ai-harness/app/Tools.kira` builds tool input schemas as 20 hand-assembled
`JsonMember` trees. Verbose, but it is genuine JSON Schema for the model's tool
API — there is no typed record to derive from, because the shape is the wire.
Low priority; a couple more builder helpers (`stringProperty`, `objectSchema`
already exist) would cut the noise. Not worth a rewrite.

## F5 — Sentinel `-1` index returns  ·  minor

9 `return -1` "index-or-absent" sites. Foundation has `Option<T>` (`.Some/.None`)
and KCoreAnimation uses both forms. `-1` for an *array index* is tolerable and
KCoreAnimation itself does it (`cachedTargetIndex`). Prefer `Option<T>` only
where the thing returned is a *value*, not an index. Low priority.

## F6 — Comment verbosity  ·  the subjective one, and I think it's real

The house style over-narrates. Much of it explains *what* the next line does
(which the line already says) in multi-sentence prose. The good comments — and
there are many — explain *why* (see `core/Protocol/Ai.kira`'s `ConversationId`
note, or `Transaction.kira`'s default-action paragraph). Recommendation: keep
why-comments, cut what-comments. My own first pass at `TaskStore.kira` had this
fault; the rewrite trimmed it. This is where "the standard isn't good" bites
hardest, because it is the most-copied and least-questioned habit.

## F7 — The lint runner's own hand-loops  ·  NOT a bug

`LintRunner.kira:339` (`while k > 0 { k = k - 1 … }`) and `:361`
(`var i = 0 / while i < stmts.count / i = i + 1`) are hand-counted loops inside
the very lint that flags them. They are correct: the first counts *down* (no
`for i in 0..n` shape) and the second *appends to `stmts` mid-iteration* (the
collection grows, which the lint documents as an exclusion). An audit that
flagged these would be wrong. Also: that file is toolchain source
(`~/.kira/toolchains/.../foundation`), a different project — not ours to edit
here.

---

## Bottom line

Sound and worth keeping: `distinct` ids, enums-not-strings for state, exhaustive
`match`, `for` over hand counters, `@Derive(Serializable)` for persistence,
atomic-write persistence.

Genuinely weak: (1) the rules are not applied consistently — F2 is the proof;
(2) comments narrate the obvious — F6; (3) boilerplate that a derive would erase
survives because the derive was never added — F3.

Cheapest high-value fixes, in order: F1 (autofix, once packages build) → F2
(mint `ConversationId`, verifiable now) → F6 (trim what-comments as files are
touched) → F3 (upstream `@Derive(Clone)` in ai-sdk).

---

# Part 2 — API shape & design quality

The lint-level stuff above is the easy half. These are the shape problems — how
the types and functions are drawn — which no lint catches.

## A1 — One message, modeled three times  ·  the structural smell

A "delta" exists as **three** structs across three layers:
`AiDelta` (`core/Protocol/Ai.kira`) → `AiWireDelta` (`ai-protocol`) →
`AiStreamDelta` (`ai-client`), and `clientEvent` copies it field by field by
hand. Same for activity/ready. A wire-vs-domain split is legitimate — the wire
shape should be free to change without the domain following. But a *third* copy
plus hand-mapping is pure boilerplate, and every new field is now three edits and
a mapping line that a typo silently breaks. Options: collapse the client to reuse
the wire struct where the split buys nothing, or generate the mapping. Not a
quick fix; it is the biggest shape decision here.

## A2 — Invalid states are representable: `done`/`failed` two-bool  ·  FIXING NOW

`AiWireDelta { … done: Bool, failed: Bool }` — two bools, four states, and one
of them (`failed && !done`) is nonsense. `core/Protocol/Ai.kira` *already solved
this*: `AiDeltaKind { Continuing, Finished, Failed }`, with a comment explaining
exactly why the two-bool form is wrong. The wire protocol regressed to the form
core rejected, and `sendDelta(…, done, failed)` threads the two bools through the
harness. Fix: an `AiDeltaKind` enum on the wire, mirroring core. Verifiable in
the AI stack now — implemented below.

## A3 — Stringly `kind`/`state` on activity  ·  half-defensible

`AiWireActivity { kind: String, state: String }`. Carrying *text on the wire* is
defended (older clients still render an unknown category) and that argument holds.
But nothing types it on the *harness* side either: `sendActivity(…, "started")`
passes bare literals, so `"finsihed"` compiles and silently misbehaves. Right
shape: harness-side enums (`ActivityKind`, `ActivityState`) that serialize to the
tolerant text at the wire — types where authored, text where transmitted.

## A4 — Parameter plumbing  ·  free-function heavy

`runTurn`, `runVisibleToolCalls`, `sendDelta`, `sendActivity`, `turnCancelled`
each re-thread some of `(gateway, channel, conversation, sessions, agents)`.
`sendDelta(channel, conversation, …)` alone recurs ~5×. A small `Turn`/
`Conversation` context struct bundling `channel + conversation` (and a harness
holding `gateway + agents`) would shrink every signature and kill a class of
arg-order mistakes. This is the "objects vs loose functions" call at
architecture scale — the counterpart to F-series at the line scale.

## A5 — `harmonyDirectory()` defined twice  ·  trivial

The same function lives in `ai-client/app/Client.kira:75` and
`ai-harness/app/Auth.kira`. One concept, two copies that can drift. Hoist to
`ai-protocol` (both already depend on it) and delete the duplicate.

## A6 — `sessionEnsure` returns a raw index  ·  aliasing hazard

It hands back an `Int` index into `sessions`, and the caller then does
`sessions[index].history.append(...)` many lines later. Any mutation that
reorders/removes a session between mint and use silently corrupts. Today nothing
does, so it is latent — but the API invites the bug. A handle or a closure-over-
the-session would not.

## A7 — Naming & symmetry  ·  minor

Three prefixes for the same domain — `Ai*` (core), `AiWire*` (protocol), `Ai*`
(client) — and encode is per-message (`encodeTurn`, `encodeDelta`) while decode
is per-category (`decodeRequest`, `decodeEvent`). Not wrong, but the asymmetry is
a small tax on every reader.

## Priorities

Fix now (small, verifiable): A2 (done below), A5. Medium: A3 (harness-side
enums). Design calls worth a conversation before touching: A1 (the triple model)
and A4 (context struct) — both ripple, so they deserve a decision, not a
drive-by.
