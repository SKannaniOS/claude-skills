# taskforce

A team of role-agents that takes one task and drives it end to end:

> **scout** (interview you for requirements) → **mastermind** (plan + freeze the interface) →
> **hackers** (build, in parallel) → **lookouts** (adversarially test) → bounded **fix-loop** →
> plain-language **report** → optional step-by-step **walkthrough**.

It sizes each run automatically: a small, well-specified task skips the interview; anything broad
or ambiguous runs the full pipeline.

## Why it's different

- **The orchestrator owns the gate.** Pass/fail is a command *it* runs and an exit code *it* reads —
  never an agent's self-reported "tests passed."
- **Promote-on-green.** Work is assembled and tested in an isolated staging worktree; verified code
  is copied into your working tree **only after** the gate is green, and re-verified there before
  anything is torn down. Unverified code never touches your tree.
- **Adversarial verification.** Up to three independent lookouts (a breaker, a spec-literal checker,
  and an edge-case hunter) probe the result against a frozen spec; single-source, evidence-backed
  failures count, and conflicts are reconciled rather than dropped.
- **Disjoint-ownership parallelism.** The plan partitions work so parallel builders never touch the
  same file, with a frozen interface pinned as code so the pieces fit.

## Install

```bash
/plugin marketplace add SKannaniOS/claude-skills
/plugin install taskforce@skannanios
```

Then start a new session and run:

```
/taskforce <describe your task>
```

<details>
<summary>Local / offline install (clone, no GitHub fetch)</summary>

Clone the repo, then add it as a marketplace from the local path — this still goes through the
plugin system, so agent namespacing resolves correctly:

```bash
git clone https://github.com/SKannaniOS/claude-skills.git
# then, inside Claude Code:
#   /plugin marketplace add ./claude-skills
#   /plugin install taskforce@skannanios
```

Copying the skill and agents straight into `~/.claude/` is **not** supported: the skill spawns its
agents through the `taskforce:` plugin namespace, which only exists when it's installed as a plugin.
</details>

## The agents

| Agent | Role |
|-------|------|
| **scout** | Runs an adaptive requirements interview (batched questions), produces a frozen `spec.md`. |
| **mastermind** | Decomposes the spec into disjoint subtasks and a frozen shared interface. |
| **hacker** | Implements exactly one subtask; runs in parallel with other hackers. |
| **lookout** | Adversarially tests the result against the spec (breaker / spec-literal / edge mandates). |
| **frontman** | Writes the optional plain-language, step-by-step walkthrough. |

## Requirements & scope

- **Claude Code only.** The orchestration relies on Claude Code primitives (subagents, background
  agent resume, structured user prompts, git-worktree isolation) that other tools (Codex, Cursor,
  etc.) don't have. The agent prompt files are still reusable as templates if you're adapting by hand.
- **Best in a git repo.** A git target gets the full safety path: an isolated staging worktree, a
  real `git diff`, and promote-only-after-green. A non-git target falls back to in-place edits with a
  snapshot-based rollback.
- **No commits or pushes without your confirmation.**

## Known limitations

- Newly installed/edited **agent definitions load at session start** — start a fresh session after
  installing.
- The pipeline asks for your input at a few gates (requirements interview, spec approval, optional
  walkthrough/cleanup); it is interactive by design, not fully autonomous.
