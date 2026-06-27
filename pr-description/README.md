# pr-description

Generates a complete pull-request description from your current branch — no copy-pasting diffs into a chat.

It auto-detects the base branch (upstream → remote default → `main`/`master`), reads the branch diff and commit history, and **fills your repository's own PR template** (`.github/pull_request_template.md` and friends) — or a sensible default if you don't have one. The title is written in Conventional Commits format (`type(scope): summary`).

It's strict where it counts: **every template section is preserved, in order**, no placeholders left behind, and checkbox/option lists are never trimmed — only their checked state changes. The finished description is saved to a file in the repo root.

## Install

```bash
/plugin marketplace add SKannaniOS/claude-skills
/plugin install pr-description@skannanios
```

Then, from your feature branch:

```
/pr-description:pr-description [base-branch]
```

`base-branch` is optional — omit it and the skill detects it for you.
