# SI — Decisions Log

Interim choices made while implementing SI.md, per its §0.2 and §6. Each is
isolated so it can be reviewed and swapped.

## D1 — Verifier is parallel and non-blocking (overrides SI.md §2.2 / C4)

SI.md §2.2 and C4 describe the parallel verifier as intercepting **inline**,
before an action takes effect ("A blocked action never executes"). The project
owner overrode this on 2026-09-16: **verifiers run in parallel and are
non-blocking.** They observe tool calls and file writes and flag/log, but do not
gate the action in real time. C4/C5 will be built to this model. Revisit if the
owner restores inline interception.

## D2 — Storage: event-sourced, one file per event (C1)  *(proposed default)*

SI.md §6 leaves "Database choice and schema for the task store" open. Interim:

- **Event sourcing.** Every change is one JSON event; in-memory state is a fold
  over the events in sequence order. Directly satisfies C1's "full event log"
  and "a kill at any point followed by a restart reproduces the same state."
- **One file per event** at `si/events/<seq>.json`, written `.tmp` then
  `renamePath`d in. Foundation has no append primitive; a single growing log
  would be O(n) per change and tearable mid-write. Per-event files make append
  O(1); the atomic rename means a present `<seq>.json` is always whole, so a
  kill leaves at most a stray `.tmp` (never read) and resume is clean.
- **Typed wire, not stringly.** Every event shape (`SiEvent` enum + one payload
  struct per variant, wrapped in `SiRecord`) carries `@Derive(Serializable)`,
  Foundation's blessed serde. `apply` dispatches by matching `SiEvent` — the
  compiler forces that match exhaustive. No hand-rolled JSON, no `if kind == …`.
- **Corruption fails fast.** `deserialize_` traps on malformed input (no partial
  parse). Acceptable because atomic writes make a torn `<seq>.json` impossible
  from our own crashes; genuine disk corruption stops the load loudly rather
  than silently resuming a half-state. Revisit if we need to quarantine-and-skip.
- **Shared id counter** for groups and tasks (`nextId`), so a task's parent/group
  reference is unambiguous across kinds.
- Recovered on load: `nextId` and `nextSeq` from the max seen in the events.

Swap target: a real embedded DB if/when one is chosen. The store's surface
(`SiStore` methods) is the seam; the on-disk format is behind it.

## D3 — Store root config seam: `HARMONY_SI_DIR`

Default `~/.harmony/si`; `HARMONY_SI_DIR` overrides the root. First of the
config seams SI.md §2.5 asks for, and what isolates a test run from a real one.

## D4 — Naming: SI / starlight, no DI migration yet

No existing `DI`/`dreaming` identifiers found in this tree, so nothing to
migrate (SI.md §6 "Naming migration"). New code is named `Si*` / `si*`.

## D5 — Numbers via derived serde

Moot in C1: `@Derive(Serializable)` tags each field with its exact width
(`Int(3)`, `SiTaskId(Int(3))`) and range-checks on read, so numbers are typed
end to end with no hand choice of Integer vs Number. Where genuine floats appear
later (e.g. context thresholds) they serialize as `Float`/`F32` bit-exact.

## D6 — Agent runtime & model router (C2 + C11)  *(proposed default)*

- **Agents persist in the same event-sourced store as C1**, not a separate one:
  `AgentSpawned`/`AgentStatusSet` events fold into `SiStore.agents`. C1's
  crash/resume guarantee covers agents for free. C2's runtime (spawn→run→stop
  via the gateway) drives these records; the store just remembers them.
- **One role axis, reused.** The hierarchy enum `SiRole` (top-orch/low-orch/
  adversarial/worker/verifier, §2.2) also drives model selection via
  `modelForRole`. The two non-hierarchy functions — evaluator (§2.3) and
  compactor — are read straight off `SiModelRoles`, not forced into the enum.
- **Model config seam: `si/models.toml`** (under `HARMONY_SI_DIR`), a `key =
  "value"` file; missing keys keep the defaults. Swapping any model — the
  evaluator included (§2.3 dynamic) — is a config edit, verified.
- **Default evaluator = "Muse Spark 1.3 Free"** (§2.3); worker/orch/etc. default
  to the free Nemotron the base harness uses (worker tokens are ~80% of spend,
  §2.5, so the default is free).
- **Deferred to the next slice:** actually running a worker through the gateway
  (reusing the harness `Agent`/`SubAgent` tool loop under the resolved model,
  capturing its result, marking it Done). The lifecycle, persistence and routing
  that a run hangs off are in place.
