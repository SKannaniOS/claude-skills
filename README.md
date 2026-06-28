# claude-skills

A [Claude Code](https://claude.com/claude-code) plugin marketplace — a small, curated collection of
skills and agents for AI-assisted software engineering. Add it once, install only what you want.

```bash
/plugin marketplace add SKannaniOS/claude-skills
```

## Plugins

| Plugin | What it does | Docs |
|--------|--------------|------|
| **taskforce** | Multi-agent orchestration that drives a task from a vague request to a verified result. | [details →](./taskforce) |
| **how-to-test** | Generates a hands-on, manual-testing guide for any repo — fresh clone to "I saw it work." | [details →](./how-to-test) |
| **pr-description** | Writes a PR description from your branch diff, filling the repo's own PR template. | [details →](./pr-description) |
| **commit-splitter** | Splits a messy working tree into clean, atomic commits — grouped by concern, not by file. | [details →](./commit-splitter) |
| **teach-me** | Patient tutor for any concept — adapts to your level, explains step by step, quizzes you, and leaves a recap. | [details →](./teach-me) |

## Installing a plugin

Once the marketplace is added, install any plugin by name:

```bash
/plugin install <name>@skannanios
```

For example, `/plugin install taskforce@skannanios`. Each plugin's own README (linked above) covers
what it does, how to run it, and any requirements. Newly installed skills and agents load at the
start of a session — start a fresh session after installing.

> Already added the marketplace? Run `/plugin marketplace update skannanios` to pick up newly
> published plugins.

## License

[MIT](./LICENSE) © Satheesh Kannan
