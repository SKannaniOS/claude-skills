---
name: commit-splitter
description: >-
  Split a messy working tree into clean, atomic commits with Conventional-Commits
  messages — grouping related changes together (even when spread across multiple
  files) and separating unrelated changes into different commits (even when they
  live in the same file, via hunk-level staging). It proposes a commit-by-commit
  plan and only creates commits after you approve. Use when the user asks to split,
  untangle, organize, or clean up uncommitted changes into multiple logical commits,
  wants atomic/bisectable history, or wants help staging hunks into separate commits.
argument-hint: "[hint | --dry-run | --detailed | --verify | --unpushed]"
---

# commit-splitter

Turn a pile of intermingled, uncommitted changes into a clean series of **atomic commits** — one
logical concern per commit, each with a Conventional-Commits message. You read the full set of
changes, group them by concern, **show the user a commit-by-commit plan, and create commits only
after they approve.**

The defining behavior is that grouping is **by logical concern, not by file**:

- **Related changes spread across multiple files → a single commit** (e.g. a new helper, its caller,
  and its export line).
- **Unrelated changes inside one file → multiple commits** (that file's hunks get split across
  several commits). This hunk-level splitting is the whole point and the reason a plain
  file-level `git add` is not enough.

## Flags (parsed from `$ARGUMENTS`)

| Flag | Effect |
|------|--------|
| *(none)* | Split the **uncommitted working tree**; one-line Conventional-Commits subjects. |
| free text | A grouping hint, e.g. `keep the refactor separate` — bias the grouping, don't treat as a flag. |
| `--dry-run` | Produce and show the commit plan, then stop. Create nothing. |
| `--detailed` | Write **full** messages: subject + blank line + body (what changed and why, bulleted) + optional footer (`BREAKING CHANGE:`, issue refs). Still strict Conventional Commits. |
| `--verify` | After each commit, run the repo's build/test command and confirm the commit is independently green; re-group if a split breaks a commit. |
| `--unpushed` | Also fold **local commits that are not on the remote** back into the working tree and re-split them. Rewrites only local, un-pushed history. Pushed commits are never touched. |

Flags compose (e.g. `--detailed --verify`). Anything not recognized as a flag is a grouping hint.

## Invariants (never violate)

- **Nothing is committed before the user approves the plan.** The approval gate is mandatory — it is
  also how this skill satisfies the rule "never commit without showing the message and getting
  confirmation."
- **Never `git push`.** Never force-push. Never add AI/assistant attribution to a commit message.
- **History is only rewritten with `--unpushed`, and only for local, un-pushed commits** — never a
  commit that already exists on the remote.
- **The working tree's content is never lost:** a `git stash create` snapshot is taken before any
  mutation, and the undo paths are always reported.
- Commit messages follow **Conventional Commits** (`type(scope): summary`).

---

## Control flow

### 0. Preflight & safety

1. Confirm you are in a git repo (`git rev-parse --git-dir`). If not, stop and say so.
2. Refuse if a merge/rebase/cherry-pick is in progress (`.git/MERGE_HEAD`, `.git/rebase-merge`,
   `.git/CHERRY_PICK_HEAD` exist) — tell the user to finish or abort it first.
3. Record the starting point: `ORIG_HEAD=$(git rev-parse HEAD)`.
4. **Determine the working state** and act on it:
   - `git status --porcelain` → is there anything uncommitted (staged/unstaged/untracked)?
   - `git rev-list --count @{upstream}..HEAD 2>/dev/null` → how many local commits are ahead of the
     remote (un-pushed)? (A non-zero exit means **no upstream is configured** — see below.)

   | State | Action |
   |-------|--------|
   | Uncommitted changes present | Split them (core path). |
   | Clean tree, but un-pushed commits exist, **no** `--unpushed` | Stop: "Nothing uncommitted to split. You have N un-pushed commit(s) — re-run with `--unpushed` to split those." |
   | Clean tree, un-pushed commits exist, **with** `--unpushed` | Soft-reset them into the tree (step 0.5), then split. |
   | Clean tree, nothing un-pushed (everything pushed) | Stop: splitting pushed commits means rewriting published history / force-pushing — out of scope. |
   | Mixed (uncommitted + un-pushed) | Default: split only the uncommitted. With `--unpushed`: combine and split both. |

### 0.5 `--unpushed` handling (only when the flag is set)

1. Determine the base of the un-pushed range:
   - If the branch has an upstream: `BASE=$(git rev-parse @{upstream})`. **Hard guard:** never
     include a commit reachable from the upstream — the un-pushed set is exactly `@{upstream}..HEAD`.
   - If there is **no upstream**, do NOT guess what is pushed. Ask the user for an explicit base ref
     (e.g. `origin/main`), or offer to fall back to working-tree-only. Only proceed once you have a
     base you are certain is already on the remote.
2. Confirm with the user that these specific local commits (`git log --oneline $BASE..HEAD`) will be
   un-committed and re-split.
3. `git reset --soft "$BASE"` — this moves those commits' changes into the index while leaving the
   working tree intact. Now continue as if they were uncommitted changes.

### 0.6 Snapshot & uniform baseline

1. Take the rollback anchor without disturbing anything: `SNAP=$(git stash create)`. This writes a
   commit object capturing the exact current state and prints its SHA; the working tree and stash
   list are untouched. Remember `$SNAP`. If it prints nothing, there is genuinely nothing to split —
   stop.
2. Unstage to a uniform baseline so analysis and staging both start from `index == HEAD`:
   `git reset -q`. (Working-tree contents are unchanged; only the index is cleared. The original
   staged/unstaged distinction is not preserved, but `$SNAP` still captures it — mention this.)

### 1. Collect all changes

- Tracked edits, deletions, and renames: `git diff` (now everything is unstaged after the reset).
- Untracked files: `git ls-files --others --exclude-standard`.
- Note **binary** files and whole-file adds/deletes/renames — these cannot be hunk-split and will be
  staged as a whole within whichever commit they belong to.

### 2. Analyze & group

Read the diff and partition every hunk into an ordered list of logical commits. Apply the grouping
heuristics below. Scale the effort to the size of the change: for a large diff, fan out
`Explore` / `general-purpose` subagents to summarize each file's hunks in parallel, then synthesize
the grouping yourself on the main thread.

Grouping heuristics (one concern per commit; order them so history reads well and patches apply
cleanly — typically dependencies/refactors first, then features, then tests, then docs/chore):

- Separate **feat** vs **fix** vs **refactor** vs **docs** vs **test** vs **chore/build** vs
  **style**.
- Put **formatting / whitespace-only** changes in their own `style:` commit, never mixed with logic.
- Keep **lockfile/dependency** changes (`package-lock.json`, `go.sum`, `Cargo.lock`, `poetry.lock`,
  …) with their manifest change or cleanly on their own — never buried in unrelated logic.
- **Group by concern, not by file** (the two directions described at the top).
- By default keep a test in the same commit as the code it exercises, unless it is clearly
  independent; honor any free-text hint the user gave.

For each group produce: an ordered index, the Conventional-Commits message (single-line subject by
default; full subject+body+footer under `--detailed`), the exact files/hunks it owns, and a
one-line rationale.

### 3. Present the plan & get approval (the commit gate)

Show the user an ordered list. For each proposed commit:

```
[1] feat(auth): add token refresh helper
     files: auth/token.py (hunk 2), auth/__init__.py (export), app.py (call site)
     why:   the new refresh path and the code that uses it

[2] style: reformat unrelated block in app.py
     files: app.py (hunk 1)
     why:   whitespace-only, kept out of the feature commit
...
```

Let the user **approve, reorder, merge or split groups, or edit any message**. Create nothing until
they explicitly approve. **If `--dry-run` was passed, stop here** after showing the plan.

### 4. Execute — non-interactive selective staging

Starting from the clean index (step 0.6), for each approved group **in order**:

1. Stage exactly that group's content:
   - **Hunk-level edits:** build a patch containing only this group's hunks and apply it to the
     index: `git apply --cached --recount <patch>` (see *Git mechanics*). Never use the interactive
     `git add -p`.
   - **Whole new/binary file:** `git add -- "<file>"`.
   - **Deletion:** `git rm -- "<file>"` (or stage the deletion hunk).
   - **Rename:** stage both sides so git records the rename.
2. Sanity-check the index matches intent: `git diff --cached --stat`.
3. Commit: `git commit -m "<subject>"` (or `-m "<subject>" -m "<body>"` for `--detailed`). Pass each
   owned file path explicitly and individually-quoted; never expand a file list via an unquoted
   shell variable.
4. If a group's patch fails to apply cleanly (overlapping/dependent hunks that can't be isolated),
   fall back to staging the offending file whole for that commit and tell the user that file was not
   hunk-split.

After the last group the working tree should be clean. If anything is left over, report it
explicitly — do not hide it.

### 5. `--verify` — per-commit green gate (only when the flag is set)

After creating each commit, run the repo's build/test command and read its **exit code** — that exit
code is the authority, not any narration:

- Auto-detect the command (e.g. an obvious `make test` / `npm test` / `pytest` / `go build ./...`),
  or use a command the user names. If none can be found, say so and skip verification for that run.
- If a commit is **not** independently green, surface the failure and offer to **re-group** — e.g.
  fold the breaking split back into the commit it depends on, then redo from that point. Never leave
  a knowingly-broken commit in the sequence.

### 6. Report & how to undo

- Show the result: `git log --oneline $ORIG_HEAD..HEAD`.
- State the two undo paths explicitly:
  - `git reset --soft $ORIG_HEAD` — drop the new commits, putting all the changes back in the index
    as one (returns you to exactly the pre-run HEAD).
  - `git stash apply $SNAP` — restore the exact pre-run working tree from the snapshot.
- **Do not push.** If the user wants to push, that's their separate, explicit action.

---

## Git mechanics (load-bearing — get these right)

- **Selective staging is done with patches, never interactively.** To stage a subset of hunks:
  generate the diff, assemble a patch containing only the chosen hunks (correct file headers +
  `@@` hunk headers), and run `git apply --cached --recount <patch>`. `--recount` lets git fix up
  line numbers as earlier groups shift the file, so later patches still apply. (Use `git diff -U0`
  + `--unidiff-zero` if you isolate hunks with zero context.) This is the reliable, scriptable
  equivalent of `git add -p`.
- **Write patch/temp files OUTSIDE the working tree** (a system temp dir, not the repo) — otherwise a
  stray `.patch` file becomes an untracked file that can get swept into a commit.
- **Never stage with `git add -A` / `git add .`.** Stage each group explicitly by its owned paths
  (individually quoted) or by applying its patch. A blanket add would pull in unrelated untracked
  files and defeat the whole purpose.
- **Uniform baseline first:** `git reset -q` so the index equals HEAD before you start; every group
  is then staged from the same known state.
- **Things that can't be hunk-split** — untracked whole files, deletions, renames, and binary files
  — are staged whole within the commit they belong to.
- **Order matters:** a later hunk can fail to apply if an overlapping earlier change wasn't applied
  yet. Order groups to respect dependencies; if a group's patch rejects, fall back to file-level for
  that file and note it.
- **Rollback anchor:** `git stash create` writes a commit capturing the working state and prints its
  SHA **without** touching the working tree or the stash list — the safety net taken before any
  mutation.
