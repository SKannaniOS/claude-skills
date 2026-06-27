# commit-splitter

You hacked all day and your working tree is a pile of intermingled changes — a feature, an unrelated
bug fix, some reformatting, a new file — all tangled together. `commit-splitter` untangles it into a
clean series of **atomic commits**, one logical concern each, with Conventional Commits messages.

It groups **by concern, not by file**:

- **Related changes spread across multiple files → one commit** (a new helper, its caller, its export).
- **Unrelated changes inside one file → multiple commits** — that file's hunks get split across
  several commits. This hunk-level splitting is the whole point, and the reason a plain
  `git add <file>` can't do it.

It always **shows you the commit-by-commit plan first** and creates commits only after you approve —
so you stay in control, and nothing is ever committed behind your back. It never pushes.

## Install

```bash
/plugin marketplace add SKannaniOS/claude-skills
/plugin install commit-splitter@skannanios
```

Then, from a repo with uncommitted changes:

```
/commit-splitter:commit-splitter
```

## Flags

| Flag | Effect |
|------|--------|
| *(none)* | Split the uncommitted working tree into atomic commits with one-line Conventional Commits messages. |
| `your hint here` | Free text biases the grouping, e.g. `keep the refactor separate`. |
| `--dry-run` | Show the proposed commit plan and stop — create nothing. |
| `--detailed` | Write full messages: subject + body (what changed and why) + optional footer (`BREAKING CHANGE:`, issue refs). |
| `--verify` | After each commit, run your build/test command and confirm the commit is independently green; re-group if a split breaks one. |
| `--unpushed` | Also fold local commits that aren't on the remote back into the working tree and re-split them. Never touches pushed history. |

Flags compose, e.g. `/commit-splitter:commit-splitter --detailed --verify`.

## Safety

- **Approval gate:** nothing is committed until you approve the plan.
- **Snapshot first:** a `git stash create` snapshot is taken before any change, so your working-tree
  content can always be restored.
- **Easy undo:** every run ends by telling you exactly how to undo it — `git reset --soft <orig>`
  (drop the new commits, changes back in one) or `git stash apply <snap>` (restore the exact
  pre-run tree).
- **Never pushes**, never force-pushes, and only rewrites *local, un-pushed* history — and only when
  you pass `--unpushed`. Anything already on the remote is never touched.

## How it works

1. Snapshots your state and unstages to a uniform baseline.
2. Reads every change (tracked edits, deletions, renames, untracked files) — scaling to large diffs
   by analyzing files in parallel.
3. Groups hunks into logical commits and shows you the plan.
4. On approval, stages each group non-interactively (`git apply --cached`, the scriptable equivalent
   of `git add -p`) and commits it.
5. Reports the new history and the undo paths.
