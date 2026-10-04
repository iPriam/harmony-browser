# Starlight Intelligence (SI) — Implementation Brief

> Audience: an autonomous coding agent implementing SI inside the Harmony codebase.
> SI was previously named **Dreaming Intelligence (DI)**. Rename any `DI`/`dreaming` identifiers you find to `SI`/`starlight` only if the task you were given says so.

---

## 0. Rules for the implementing agent

1. **Section 2 is settled architecture.** Do not redesign it, simplify it away, or substitute a "cleaner" pattern. If something there looks wrong, stop and ask. Do not work around it.
2. **Section 6 lists open decisions.** Do not guess them silently. If you need a default to keep moving, pick the one marked *proposed default*, isolate it behind an interface, and record it in `DECISIONS.md` so it can be reviewed.
3. **Match the existing repo.** Use the language, build system, module layout and conventions already present in Harmony. Do not introduce a new language or framework for SI.
4. **Execute instructions literally.**
   - If a command fails, report it. Do not substitute an "adjacent" command or loop through retries.
   - Do not rebuild speculatively.
5. **Never delete or overwrite files or data outside your worktree.** Inside your worktree, destructive operations on anything you did not create in this task need explicit approval.
6. **Stop early and report.** A clear report beats a long run that drifts.

---

## 1. What SI is

SI is the always-on autonomous core of **HarmonyAI**, which is the AI harness built into Harmony Browser.

- The user gives SI one or many tasks and can add more at any time. SI absorbs them and keeps working.
- SI runs 24/7 and tries to use all available token budget to finish outstanding work.
- SI wakes on:
  - provider session resets,
  - weekly quota resets,
  - a newly added provider,
  - other system events (see §6).
- **Single-session agents stay available** as the option for one focused task. SI is the default mode for ongoing work; it does not replace single-session agents.
- **Chat and Agentic are both control surfaces.**
  - From Harmony Chat, the user can dispatch work into SI, open threads, and ask for progress.
  - Agentic can read Chat history for context.

---

## 2. Settled architecture (non-negotiable)

### 2.1 Process placement

The processes `shell`, `browser`, `webkit` and `ai` are always separate. Inside `ai` there are three layers:

| Layer | Role |
|---|---|
| **AI (UI)** | Interface only. Talks to the harness and holds no orchestration logic. |
| **AI Runtime** | Client-side daemon. Always running, zero cost when idle, acts as the local bridge. Also serves Ask Harmony on iPhone. |
| **AI Harness** | Orchestration for Chat and Agentic. **SI lives here.** |

The server side is self-hosted on AWS and uses Harmony's own tunneling system for client connectivity.

**Persistence rule:** every piece of important state goes to disk and database. Nothing may exist only in memory. SI must survive a spot-instance interruption at any moment and resume cleanly.

### 2.2 Agent hierarchy

```
User / Event
   │
   ▼
Top-level orchestrator  (one per task or task group; conversational)
   │
   ▼
Low-level orchestrator ◄──── peer ────► Adversarial agent(s)
   │
   ▼
Workers (parallel, one worktree each)
   │
Parallel verifier runs alongside EVERY worker and EVERY orchestrator
```

**Top-level orchestrator**
- Spawned per task or task group.
- The user can talk to it directly. It observes and manages everything below it.
- **Has no adversarial peers.** It is started by the user or by an event, so it is directly accountable.

**Low-level orchestrator**
- Main coordinator: decomposes work, manages workers, validates the shape of the work.

**Adversarial agents**
- **Peers** of the low-level orchestrator, not subordinates.
- Criticize the orchestrator's decisions and the workers' output with equal weight.
- Run **alongside** the orchestrator, **not** as a separate validation pass afterwards.
- Exist only at the low level, where autonomous decisions happen without direct human oversight.

**Workers**
- Sub-agents that do the actual work.
- Run in parallel, each in its own worktree.
- Have **real bash access**. The shell is not abstracted away.

**Parallel verifier**
- Self-made models run alongside every worker and every orchestrator.
- They verify **every tool call and every file write in real time**, to ensure instructions are being followed.
- They **intercept inline** before the action takes effect. This is not a post-hoc review.

### 2.3 Agent lifecycle, stop, evaluate, compact

1. Agents are designed to **stop early**. The worst-case context is 999k tokens and must never be the normal case.
2. When an agent stops, a **free evaluator model** judges the work shape against the goal:
   - **Goal achieved** → report to the parent orchestrator.
   - **Not achieved, and context is above that model's configured optimal threshold** → compact, then continue.
   - **Not achieved, and context is fine** → continue.
3. **Compaction is agentic.**
   - A cheap or free model receives all model outputs and tool calls.
   - It may make a few tool calls of its own.
   - It writes a summary, plus an optional markdown handoff document.
   - The agent resumes from that summary and handoff.
4. **The evaluator is dynamic.** Use whichever free model is best at the time, set by configuration and never hardcoded. The current default is Muse Spark 1.3 Free.

### 2.4 Harmony Agent Queue (throttling)

**Purpose:** minimize prefill cost. The anti-pattern to avoid is spawning many parallel agents, hitting quota peaks, paying prefill on every resume, and killing agents over and over.

**Behavior**
- Prefer fewer agents that run longer with fewer resumes.
- **Wake paused agents every 50 minutes.** Provider KV caches almost universally expire after 1 hour; subtracting a 10-minute buffer after the last agent response ends means the cache is still warm on resume.
  - Measure the timer from the end of the last response.
  - Keep the interval configurable, with 50 minutes as the default.

**MCP interface**
- External models (for example ChatGPT scheduled tasks) can **pull jobs** from the queue.
- Harmony remains the orchestrator no matter which model executes the work.

### 2.5 Multi-provider

SI routes work across several providers and models: Anthropic, OpenAI, GLM, and free tiers. Keep these decisions behind configuration and never scatter them through the code:

- which model is the worker,
- which model is the orchestrator,
- which model is the evaluator,
- which model does compaction.

The main cost is worker tokens, roughly 80% of spend.

---

## 3. Components to build

Each component needs tests and must restore its state from persistence after a crash.

| # | Component | Responsibilities | Done when |
|---|---|---|---|
| C1 | **Task store** | Tasks and task groups, status, parent/child agent links, full event log. | A kill at any point followed by a restart reproduces the same state. |
| C2 | **Agent runtime** | Spawn, pause, resume and stop agents; each agent is bound to a model, role, context budget and worktree. | Roles: top-orch, low-orch, adversarial, worker. |
| C3 | **Worktree manager** | One isolated worktree per worker; handles creation, cleanup and handing results back to the orchestrator. | Workers cannot write outside their worktree. |
| C4 | **Parallel verifier hook** | Inline interception of every tool call and file write for every agent; approve, block, or flag with a reason. | A blocked action never executes, and the decision is logged. |
| C5 | **Adversarial loop** | Runs concurrently with the low-level orchestrator; critiques both decisions and worker output. | Critiques reach the orchestrator before it commits a decision. |
| C6 | **Stop evaluator** | Runs the 3-way branch in §2.3; evaluator model comes from config. | All three branches are covered by tests. |
| C7 | **Agentic compactor** | Summary plus optional handoff markdown; limited tool calls. | A resumed agent continues correctly from the handoff. |
| C8 | **Agent Queue** | Scheduling, 50-minute warm-cache wakes, quota awareness, MCP job-pull endpoint. | An external MCP client can pull and complete a job. |
| C9 | **Event bus / wake triggers** | Session reset, weekly reset, new provider, and further events (§6). | Each trigger wakes the correct pending work. |
| C10 | **Control surface API** | Used by Chat and Agentic: dispatch tasks, add tasks, query progress, talk to a top-level orchestrator, read Chat history. | Chat can start work and report its status. |
| C11 | **Provider adapters + router** | Uniform interface across providers; model roles set in config. | Swapping the evaluator model is a config-only change. |

---

## 4. Suggested build order

1. C1 + C2 + C11. A single worker agent runs end to end with persistence.
2. C3 + C4. Isolation and inline verification, **before** any autonomy is enabled.
3. C6 + C7. The stop, evaluate and compact loop.
4. Low-level orchestrator with workers, then C5 (adversarial peers).
5. Top-level orchestrator, plus C10.
6. C8 + C9. The queue, wake triggers, and 24/7 operation.
7. Chaos testing: kill the process or instance at random points and verify clean resume.

---

## 5. Testing requirements

- **Crash and resume** at every lifecycle stage (§2.3), including in the middle of compaction.
- **Verifier coverage:** any tool call or write that goes around C4 is a test failure.
- **Stop-evaluator branches:** all three paths.
- **Queue timing:** the wake happens at or before 50 minutes after the last response ends. Use an injectable clock.
- **Top-level orchestrators** have no adversarial peers; **low-level orchestrators** always have at least one.
- **Worker isolation:** a write outside the worktree is blocked.

---

## 6. Open decisions — ask, do not guess

These are **not** specified yet. Record any interim choice in `DECISIONS.md`.

**Verification and evaluation**
- **Verifier models:** which self-made models, how they are served, their latency budget, and what happens if a verifier is unavailable (the *proposed default* is to fail closed and block).
- **"Work shape":** the exact criteria the evaluator and orchestrators use to decide that the goal has been reached.
- **Optimal context threshold:** the value per model, and where it lives in config.
- **Adversarial disagreement:** how deadlocks are resolved, and whether any case escalates to the top-level orchestrator or the user.

**Queue and scheduling**
- **Event list:** "other events" beyond session reset, weekly reset and new provider.
- **Quota model:** how per-provider limits and resets are tracked, and how the queue prioritizes across tasks.
- **Concurrency limits:** maximum parallel workers per provider or task.
- **MCP job-pull contract:** authentication, job schema, leasing and timeouts, and result submission.

**Workers and results**
- **Worker result integration:** how worktree outputs are merged, and who resolves conflicts.
- **Destructive-operation policy:** what counts as destructive, and which operations need human approval.

**Storage and schema**
- **Database choice** and schema for the task store (C1).
- **Naming migration:** whether existing `DI` identifiers are renamed now.

---

## 7. Out of scope for this brief

- AI (UI) visuals and per-platform presentation. The AI process already renders to its own GPU texture, and desktop attach/detach already exists; do not touch either.
- Single-session agent mode, beyond keeping it working.
- Harmony Chat, the Slack-style product for agents and humans. It is a separate product.