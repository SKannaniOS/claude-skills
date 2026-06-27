---
name: taskforce
description: >-
  Orchestrate a team of role-agents to take a task from vague request to verified result:
  gather requirements → plan → fan out parallel hackers → adversarial test → bounded fix-loop →
  plain-language summary → optional step-by-step walkthrough. Sizes each run lite or full so small,
  well-specified tasks skip the interview. Use when the user wants a task driven end-to-end by a
  coordinated agent team.
argument-hint: "<task description>"
allowed-tools: Agent, SendMessage, AskUserQuestion, Read, Write, Bash
user-invocable: true
---

# taskforce — orchestrator

You are the **orchestrator** (main loop). You own ALL control flow, user interaction, fan-out,
the bounded fix-loop, and artifact lifecycle. The five role-agents (`scout`, `mastermind`,
`hacker`, `lookout`, `frontman`) CANNOT talk to the user or to each other — every handoff
goes through you, and shared state lives in files under the run directory.

## Spawning the role-agents (use these exact `subagent_type` values — load-bearing)

This skill ships as a plugin, and plugin agents are reachable ONLY via the plugin namespace.
Whenever a step below says to invoke/spawn a role-agent, pass the **namespaced** `subagent_type`:

| role (shorthand used below) | `subagent_type` to pass |
|---|---|
| scout | `taskforce:scout` |
| mastermind | `taskforce:mastermind` |
| hacker | `taskforce:hacker` |
| lookout | `taskforce:lookout` |
| frontman | `taskforce:frontman` |

A **bare** name (`scout`, …) will NOT resolve to this plugin's agent for a fresh user — bundled
plugin agents are lowest-priority and only addressable through the `taskforce:` namespace. The bare
role names in the steps below are shorthand for these namespaced agents.

## SendMessage resume rule (load-bearing — verified by dry-run)

An agent can only be continued via `SendMessage` if it was spawned with `run_in_background: true`
— that is what returns the resumable `agentId` (format `a…`). A normal foreground `Agent` call
returns final text but CANNOT be resumed. Therefore spawn **in background** every agent you need
to hold a multi-turn conversation with: the `scout` (adaptive interview), any `hacker`
you'll re-engage in the fix-loop, and the `frontman` (walkthrough follow-ups). One-shot agents
that you never resume (e.g. the `mastermind`, and `lookout`s) may be foreground.
Resume with `SendMessage({ to: "<agentId>", summary: "…", message: "…" })`; you are notified when
the resumed agent stops again.

## Waiting on background agents — do NOT poll (load-bearing for speed)

You are **auto-notified** the moment a background agent stops. Do NOT `ScheduleWakeup` or sleep on
short intervals (e.g. 60–240s) to "check progress" — that adds dead wall-clock time, burns context,
and accomplishes nothing the notification doesn't. After spawning/resuming a background agent, simply
end your turn and wait for the completion notification. The ONLY legitimate `ScheduleWakeup` here is a
single **long fallback** (≥ 20 min) as a safety net against a genuinely hung agent — never as a poll.

## Tunable constants

- `MAX_ITERS = 3` — fix-loop cap before escalating to the user.
- `NUM_LOOKOUTS` — adversarial lookout count, sized by the triage gate (step 0.5): `3` on the
  **full path** (mandates `breaker`, `spec-literal`, `edge`) or `1` on the **lite path** (a single
  reviewer with a combined `breaker + spec-literal` mandate).
- `RETAIN_RUNS = 10` — keep this many most-recent runs.
- `RETAIN_DAYS = 14` — also keep anything newer than this.

## Run directory

All artifacts live in `~/.claude/tasks/taskforce/<run-id>/` where `<run-id>` is
`<UTC-timestamp>-<short-slug>` (e.g. `20260625-143002-fib-validation`). This is namespaced under
`taskforce/` so it never collides with Claude Code's own task dirs.
Artifacts: `task.md`, `spec.md`, `plan.md`, `findings.md`, `report.md`, `walkthrough.md`,
`diff.patch`; plus the `staging/` tree (git worktree where work is assembled and tested before
promotion) and the `before/`/`after/` snapshot subdirs used only on the non-git fallback.

---

## Control loop

### 0. Set up
- Get a timestamp via `date -u +%Y%m%d-%H%M%S` (do not rely on internal clocks).
- Create `~/.claude/tasks/taskforce/<run-id>/`, write the raw task to `task.md`.
- **Retention prune:** delete a run if it is BOTH outside the most-recent `RETAIN_RUNS` AND older
  than `RETAIN_DAYS`. Equivalently: keep a run if it is among the most-recent `RETAIN_RUNS` OR
  newer than `RETAIN_DAYS`. Never prune the current run.

### 0.5 Triage — size the task, pick the pipeline weight
Before interviewing, size the work and choose a path. Set `NUM_LOOKOUTS` accordingly (step "Tunable
constants"). Default **conservative**: when in doubt, choose FULL. You MAY do a quick read-only
peek (Bash `ls`/`find`, `Read`) to ground the size estimate — but do NOT start building. If the
peek leaves any real uncertainty about size or clarity, choose FULL.
- **LITE path** — choose only when BOTH hold: (a) the change is small/tightly-coupled (≈ ≤ 3 files),
  AND (b) requirements are already unambiguous (the user pinned the design, or there are no real open
  decisions). On lite: SKIP the adaptive interview, draft `spec.md` directly from the task, go straight
  to the approval gate (step 2); set `NUM_LOOKOUTS = 1`; expect a single hacker (the mastermind will
  still produce a plan, often one subtask).
- **FULL path** — anything broad, ambiguous, risky, multi-file, or security-sensitive. Run the pipeline
  as written; `NUM_LOOKOUTS = 3`.
- If genuinely on the fence about size, you MAY ask the user once ("Lite or full taskforce?"), but bias
  to FULL for anything that isn't clearly a focused, well-specified fix. A mis-sized lite run gets a
  thinner safety net, so only go lite when the smallness AND clarity are both obvious.

### 1. Adaptive interview (skipped entirely on the LITE path)
- Only runs on the FULL path. On LITE, skip to step 2 with a directly-drafted `spec.md`.
- Invoke `scout` **with `run_in_background: true`** (so it's resumable) with the task and
  run-dir path. It returns one of: `STATUS: ASK` (one question + options),
  `STATUS: ASK_BATCH` (up to 4 INDEPENDENT questions + options), or `STATUS: DONE` (+ spec).
- To minimize user round-trips, the scout should BATCH all mutually-independent clarifications into a
  single `ASK_BATCH` and reserve one-at-a-time `ASK` only for a question whose wording genuinely
  depends on a previous answer.
- On `ASK_BATCH`: present all questions in ONE `AskUserQuestion` call (it accepts up to 4), WAIT for
  the answers, then `SendMessage` them back to the SAME scout instance together.
- On `ASK`: present the single question via `AskUserQuestion`, WAIT, then `SendMessage` the answer
  back to the SAME scout instance so it adapts the next question.
- On `DONE`: write the returned spec to `spec.md`.

### 2. Approval gate
- Show `spec.md` to the user. Get explicit go-ahead before building. If they want changes, loop
  back to step 1.

### 3. Plan
- Invoke `mastermind` with the run-dir path — on the FULL path spawn it with **`model: opus`**
  (decomposition correctness gates the entire fan-out; a bad partition cascades into collisions
  across every downstream hacker); `sonnet` is fine on LITE. It writes `plan.md` and returns
  `READY` or `BLOCKED`.
- If `BLOCKED`: (1) read the `## OPEN QUESTIONS` section from `plan.md`; (2) start a NEW
  scout instance (`run_in_background: true`) seeded with both the original task and the open
  questions; (3) run the interview loop (step 1) until the scout returns `STATUS: DONE`; (4)
  overwrite `spec.md` with the updated spec; (5) re-run the approval gate (step 2) on the updated
  `spec.md` and get explicit user go-ahead; (6) invoke a NEW mastermind instance. The old scout
  and mastermind instances are not resumed.
- Sanity-check that subtasks have **disjoint owned files** before fanning out.

### 4. Fan out
- **Non-git fallback only — baseline snapshot:** in a git repo skip this (the diff comes from
  `git`, step 7). On a non-git target, for every owned file that ALREADY exists, mirror it into
  `<run-dir>/before/` preserving its relative path
  (`mkdir -p "<run-dir>/before/<dir>" && cp "<file>" "<run-dir>/before/<relative-path>"`). This is
  both the diff baseline and the rollback point if the run escalates.
- **Create the staging tree FIRST** (git target): `git worktree add --detach "<run-dir>/staging" HEAD`.
  This is where hackers write and where the gate runs — never the user's live checkout. (Non-git
  target: the live tree doubles as staging; see step 5.)
- Spawn one `hacker` per subtask **in a single message** so they run in parallel, each with
  `run_in_background: true` (so the fix-loop can re-engage the same instance via `SendMessage`).
- **Where hackers write — pick by topology (this matters; the common case is the first one):**
  - *External git target* — a git repo, but the orchestrator session is NOT inside it (the usual
    case): point each hacker at the staging tree and have it write its disjoint owned files
    DIRECTLY into `<run-dir>/staging/`. Disjoint ownership makes concurrent writes collision-free
    (hackers run no git commands), so no per-subtask worktrees are needed. `isolation: worktree`
    does NOT apply here — it worktrees the *session* repo, not the external target.
  - *Session inside the target git repo*: you MAY additionally give each hacker its own
    `isolation: worktree` for belt-and-suspenders isolation, then copy owned files into staging at
    step 5. Optional — disjoint ownership is the real defense (see **Worktree isolation**).
  - *Non-git target*: hackers edit the live tree directly, serially, in strict file-ownership
    order; the `before/` snapshot is the rollback point.
- **Do not tear down staging or any worktree here** — they must stay alive through the fix-loop
  (step 7), which resumes hackers writing in place.
- Respect `Depends on:` ordering — run dependents only after their dependencies report `DONE`.
- If a hacker returns `NEEDS_INFO` / `OUT_OF_SCOPE`, resolve it (re-plan or gather) before
  proceeding.
- **Liveness:** if a hacker dies, never reports, or returns a `NEEDS_INFO` it cannot resolve after
  one re-engage, ESCALATE to the user rather than hanging. The only timer you may set is a single
  long fallback wakeup (≥ 20 min) as a hung-agent safety net — never a short poll (see the no-poll
  rule above).

### 5. Confirm the staging tree is assembled (NEVER the user's live tree)
The staging tree was created in step 4; unverified code lives only there until step 7 promotes it.
- **External git target (hackers wrote directly into staging — the common case):** assembly is
  already done. Just confirm every owned file from `plan.md` is present in `<run-dir>/staging/`
  before gating. No copy step.
- **Session-inside-target with per-subtask worktrees:** copy each hacker's owned files from its
  worktree into `<run-dir>/staging/`, serially. Disjoint ownership makes this a straight copy, no
  merge. The user's live checkout stays untouched.
- **Non-git target:** the live tree *is* the staging tree; the `before/` snapshot (step 4) is the
  rollback point. If step 7 escalates, restore from `before/` rather than leaving broken code behind.
- If a file collision appears, the plan's partition was wrong — fix the plan and re-run the
  affected subtask.
- **Do NOT tear down staging or any worktree yet** — the fix-loop reuses them (step 7).
- On a fix-loop re-assembly, only the changed subtasks' files change in staging (hackers edit in
  place, or you re-copy just those files).

### 6. Verify — the orchestrator owns the pass/fail gate
A subagent's narrated "tests passed" is not evidence; an exit code is. YOU run the gate.
- **Run the real check yourself first:** in the staging tree, run the project's actual
  build/lint/test command via Bash and record its **exit code**. That exit code is the authoritative
  gate — a clean run is required to pass; a non-zero run is a failure no lookout can talk you out of.
  If there is no test command, run the most concrete check that exists (build / compile / typecheck /
  the acceptance command named in `spec.md`).
- **Then spawn the lookouts to probe what a single command can't:** `NUM_LOOKOUTS` lookouts in
  parallel in one message, pointed at the staging tree. FULL → 3 mandates (`breaker`, `spec-literal`,
  `edge`) with **`model: opus`**; LITE → 1 combined `breaker + spec-literal` lookout (`sonnet`). They
  hunt edge cases and walk each `spec.md` criterion; they do NOT own the verdict.
- Merge all findings into `findings.md`. A failure is **actionable** when backed by hard evidence —
  your own non-zero exit code, or a lookout's reproduced command + output, or an exact `spec.md`
  criterion violation. **Never discard an evidence-backed failure merely because only one source
  found it.** Use cross-lookout agreement only to **reconcile conflicting verdicts on the same
  criterion**: surface the conflict and re-test that one criterion rather than dropping either side.

### 7. Bounded fix-loop, then promote only what passed
```
iter = 0
loop:
  if gate is green (clean exit code AND no actionable findings): break    # success
  if iter == MAX_ITERS: ESCALATE — show findings.md to user, ask how to proceed; break
  iter += 1
  for each subtask with a failure:
     SendMessage its hacker (same instance — keeps context, still alive in its worktree)
       the specific findings
  re-assemble the fixed subtasks into staging (step 5), then re-run the gate:
     the orchestrator's own check + fresh lookouts (same NUM_LOOKOUTS, mandates, model as step 6)
```
> **Shell-portability rule (load-bearing — verified by live run):** never rely on unquoted
> `$var` word-splitting to expand a file list — the run shell may be `zsh`, where `$files` does
> NOT split into multiple arguments (it stays one string) and silently breaks `git add`, `cp`,
> and `diff`. ALWAYS pass owned-file paths as **explicit, individually-quoted arguments** (one
> `cp` per file, or a literal multi-arg `git add -- "a" "b" "c"`). Quote every path.

**On success, run this exact ordered sequence — do NOT reorder, and do NOT tear down early:**
1. **Produce `diff.patch` from staging (before promoting or tearing down):**
   - **Git:** stage and diff **only the owned files** from `plan.md` (never `add -A` — running the
     gate leaves build artifacts like `__pycache__`, `*.o`, `dist/` in staging, and `add -A` would
     sweep that junk into the patch and the promotion). Pass each owned file as its own quoted arg:
     `git -C "<run-dir>/staging" add -- "<file1>" "<file2>" … && git -C "<run-dir>/staging" diff --cached -- "<file1>" "<file2>" … > "<run-dir>/diff.patch"`
     — a real diff scoped to taskforce's output, with rename detection and correct handling of
     new / deleted / binary / mode changes. Do not hand-roll with `cp` + `diff`.
   - **Non-git fallback only:** mirror changed files into `<run-dir>/after/`, then
     `diff -ruN "<run-dir>/before" "<run-dir>/after" > "<run-dir>/diff.patch"` (exit status `1` on
     differences is normal, not an error).
2. **PROMOTE — copy each owned file individually** from staging into the user's live working tree,
   one explicit quoted `cp` per file (`cp "<run-dir>/staging/<file>" "<live>/<file>"`). This is the
   single mutation of the real tree, and only verified code reaches it. (Non-git fallback: the live
   tree already holds the verified code — nothing to copy.)
3. **VERIFY IN THE LIVE TREE — re-run the gate there and read the exit code.** Promotion is a copy
   and copies can fail silently (wrong paths, a no-op expansion); a green staging gate does NOT
   prove the live tree is correct. Only an exit-0 gate **in the live tree** confirms the promotion.
4. **TEAR DOWN worktrees ONLY after step 3 is green.** Gate the teardown on the live-tree exit code:
   if it is non-zero, KEEP `staging/` and every worktree intact, tell the user the promotion did not
   verify, and hand them the `staging/` path — never destroy the verified work before the live tree
   is confirmed. (Premature teardown after a silently-failed promotion can lose the only good copy.)

- **On escalation (fix-loop hit `MAX_ITERS`) — DO NOT promote.** Leave the live tree clean (git) /
  restore it from `before/` (non-git), KEEP `staging/` and the worktrees, and hand the user the
  `<run-dir>/staging` path plus `findings.md` so they can inspect the broken state. Do not pretend
  it's done, and do not tear down the evidence.

### 8. Summary
- Write `report.md`: a plain-language summary — what was asked, what was built, what was tested,
  what passed/failed, and any escalations. No jargon.
- Present `report.md` to the user.

### 9. Optional detailed walkthrough
- Offer via `AskUserQuestion`: "Want a step-by-step walkthrough of how this was built?"
- If yes:
  - Invoke `frontman` **with `run_in_background: true`** (so step-question follow-ups can resume
    it) — point it at `spec.md`, `plan.md`, `diff.patch` in the run directory, `findings.md`;
    it writes `walkthrough.md`.
  - For each step in order: show the step, then `AskUserQuestion` with options
    **Next / I have a question / Stop**.
    - *Next* → reveal the next step.
    - *I have a question* → `SendMessage` it to the frontman, relay the answer, re-offer.
    - *Stop* → exit the walkthrough.

### 10. Lifecycle / cleanup
- Offer via `AskUserQuestion`:
  - **Keep everything** (default) — full audit trail.
  - **Keep only report.md** — delete `spec.md`, `plan.md`, `findings.md`, `walkthrough.md`,
    `diff.patch`, and the `before/` and `after/` snapshot subdirs.
- Never auto-delete the current run silently; deletion here is explicit, and cross-run pruning
  already happened in step 0.

---

## Invariants (do not violate)

- No role-agent talks to the user or spawns another agent — that is YOUR job. If an agent's
  output implies it tried to, treat it as a bug and route the interaction yourself.
- The frozen `spec.md` is the single source of truth for the mastermind and lookouts.
- Implementers only ever touch their owned files; disjoint ownership is the primary
  collision-defense, with worktree isolation as a conditional safeguard (git repos only — see the
  Worktree isolation section).
- **Unverified code never enters the user's working tree — on a git target.** Assembly and testing
  happen in the staging worktree; promotion to the live tree happens only after the gate is green
  (step 7). The single documented exception is a **non-git target**, where no worktree is available
  so the live tree doubles as staging — there the `before/` snapshot is the rollback path (step 5).
- **The pass/fail gate is a command the orchestrator runs and an exit code it reads** — never a
  subagent's self-reported success.
- **Promotion is verified before teardown.** After copying into the live tree, re-run the gate
  *in the live tree*; tear down `staging/` and the worktrees ONLY on a green live-tree gate. Never
  destroy the staged work before the live tree is confirmed (a silently-failed promotion would
  otherwise lose the only good copy).
- **Never expand a file list via unquoted `$var`.** The run shell may be `zsh` (no word-splitting);
  pass owned-file paths as explicit, individually-quoted arguments.
- The fix-loop is bounded by `MAX_ITERS` and always escalates rather than looping forever.
- No git commits or pushes without explicit user confirmation.

## Worktree isolation — scope and fallback (clarified by dry-run)
`isolation: worktree` worktrees the **session's** repo, so it only isolates hackers when the
orchestrator is running *inside the target git repo*. Two independent guarantees, in priority
order:
1. **Disjoint-file ownership is the primary collision-defense** — proven in dry-run: two parallel
   background hackers writing disjoint files against a fixed interface integrated with zero
   conflict and zero drift. The mastermind MUST enforce disjoint ownership regardless of isolation.
2. **Worktree isolation is belt-and-suspenders**, useful only when (a) the session is inside the
   target repo AND (b) you can't fully guarantee disjointness. When the target repo is external to
   the session (or not a git repo at all), worktree isolation does not apply — rely on (1).
3. **The `staging/` worktree is a separate tree from the user's live checkout** (and from any
   per-subtask worktrees, when those are used). On an external git target — the common case —
   hackers write their disjoint owned files DIRECTLY into `staging/`, so no per-subtask worktrees
   exist or are needed. Its purpose is that integration and testing never mutate the user's tree
   until the work is verified and promoted (steps 5–7).
If the target isn't a git repo, offer `git init` (which unlocks staging + a real `git diff`), or
fall back to running hackers **serially** with strict file-ownership ordering and `before/`-based
rollback.
