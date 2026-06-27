---
name: pr-description
description: Generate a PR description from the current branch diff and any repo PR template. Use when the user asks to generate, write, or create a PR description, pull request description, or PR body.
argument-hint: "[base-branch]"
---

Generate a PR description by following these steps carefully:

## Step 1: Detect the base branch

If a base branch was provided as an argument, use it: $ARGUMENTS

Otherwise, auto-detect by running these in order:
```bash
# Check upstream tracking branch
git rev-parse --abbrev-ref --symbolic-full-name @{u} 2>/dev/null

# Fallback: get remote default branch
git remote show origin 2>/dev/null | grep 'HEAD branch' | awk '{print $NF}'

# Fallback: check if main or master exists
git branch -r | grep -E 'origin/(main|master)' | head -1
```

Use the detected branch name (strip `origin/` prefix if present). Default to `main` if all fail.

## Step 2: Get the diff
```bash
BASE_BRANCH=<detected-branch>
MERGE_BASE=$(git merge-base HEAD origin/$BASE_BRANCH)

# Summary of changed files
git diff $MERGE_BASE..HEAD --stat

# Full diff
git diff $MERGE_BASE..HEAD
```

## Step 3: Get commit history on this branch
```bash
git log $(git merge-base HEAD origin/$BASE_BRANCH)..HEAD --oneline --no-merges
```

## Step 4: Get current branch name
```bash
git rev-parse --abbrev-ref HEAD
```

## Step 5: Load the PR template

Check these paths in order and use the first one found:
```bash
cat .github/pull_request_template.md 2>/dev/null
cat .github/PULL_REQUEST_TEMPLATE.md 2>/dev/null
cat docs/pull_request_template.md 2>/dev/null
```

If no template is found, use this default structure:
```
## Summary
<!-- What does this PR do and why? -->

## Changes
<!-- List key changes made -->

## Testing
<!-- How was this tested? -->

## Screenshots
<!-- If applicable -->

## Notes
<!-- Anything reviewers should know -->
```

## Step 6: Parse every section from the template

Before writing anything, explicitly list out every section header found in the template in the exact order they appear.
This is your checklist — every single section MUST appear in the final output in the same order, no exceptions.

## Step 7: Generate the PR description

Start with a **PR Title** on the very first line using Conventional Commits format:
```
<type>(<scope>): <short description>
```

Where:
- **type** — inferred from the diff: `feat`, `fix`, `refactor`, `chore`, `docs`, `test`, `perf`, `ci`, `build`, `style`
- **scope** — the module, component, or domain affected (e.g. `auth`, `api`, `checkout`, `user-profile`) — inferred from changed file paths
- **short description** — imperative, lowercase, no period, max 72 chars

After the title, add a blank line, then fill every template section.

STRICT RULES — violating any of these is not acceptable:

- **NEVER skip a section.** Every section from the checklist in Step 6 must be present in the output. A missing section is always wrong, even if you have nothing to say.
- **NEVER reorder sections.** Preserve the exact order they appear in the template — do not rearrange, group, or move any section for any reason.
- **NEVER leave placeholder text.** No `<!-- comments -->`, no `TBD`. Replace every section with real content.
- **NEVER drop options from a checkbox or multi-option list.** If the template lists five `- [ ]` checkbox items (e.g. "Type of change" with `Bug fix`, `New feature`, `Breaking change`, `Documentation`, `Refactor`), the output MUST contain all five lines in the same order — only toggle the box state between `- [ ]` and `- [x]`. The same applies to radio-style lists and any bullet list copied verbatim from the template: preserve every line, only update its check state or trailing content. Removing an unchecked option is always wrong.
- For every section, derive content from the diff, commits, and branch name. When direct information is unavailable, write a brief factual statement rather than dropping the section:
  - **Summary** — explain the *why* and *what*, 2–4 sentences
  - **Changes** — bullet each meaningful change with file/component/function context
  - **Testing** — infer test steps from what changed; if purely a config or doc change, write what to manually verify
  - **Screenshots** — if UI files changed, note which screens are affected; otherwise write "No UI changes in this PR"
  - **Notes / anything else** — flag breaking changes, migrations, config changes; if none, write "No breaking changes or migration steps required"
  - **Type of change / checkbox lists** — keep every option from the template; mark the applicable ones with `- [x]` and leave the rest as `- [ ]`. Never delete unchecked options.
  - **Any other section** — write a concise factual statement derived from the diff; never drop the section because it seems inapplicable
- After filling all sections, do a final self-check: re-read your checklist from Step 6 and confirm every section header appears in your output **in the same order** before saving.

## Step 8: Save to file
```bash
# Slugify the branch name for the filename
BRANCH=$(git rev-parse --abbrev-ref HEAD | sed 's/[^a-zA-Z0-9]/-/g')
FILENAME="pr-description-${BRANCH}.md"
```

Write the complete PR description (title + all sections) to `$FILENAME` in the repo root:
```bash
cat > $FILENAME << 'EOF'
<generated content here>
EOF
```

Then confirm to the user:
```
✅ PR description saved to: <filename>
   Title: <the conventional commit title>
```

Do NOT print the full content to the terminal — only print the confirmation line and the PR title so the user knows what was generated.
