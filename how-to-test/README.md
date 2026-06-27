# how-to-test

Point it at any repo and it writes a single Markdown guide for **manually testing the project by hand** — going from a fresh clone to actually using the SDK/app and confirming, with your own eyes, that it works. It's deliberately *not* about running the automated test suite.

Great for **onboarding to an unfamiliar codebase** or **evaluating a library before you adopt it**.

It fans out parallel subagents to discover the project shape, existing example/sandbox paths, the real public API, external dependencies, and how to run things locally — then synthesizes a newcomer-friendly guide with prerequisites, copy-pasteable setup, a happy-path walkthrough, expected output, troubleshooting, and teardown. Steps that need credentials, money, or hit a production system are flagged with ⚠️.

## Install

```bash
/plugin marketplace add SKannaniOS/claude-skills
/plugin install how-to-test@skannanios
```

Then, from inside the repo you want a guide for:

```
/how-to-test:how-to-test
```

It saves the guide as `how-to-test.md` at the repo root (or into an existing `docs/`/`tasks/` folder) and gives you a short summary of the single fastest check to confirm the project works.
