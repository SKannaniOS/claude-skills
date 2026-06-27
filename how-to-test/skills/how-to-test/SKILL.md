---
name: how-to-test
description: >-
  Analyze an entire repository and generate a detailed, developer-friendly,
  step-by-step guide for MANUALLY testing it by hand — actually using the
  SDK/repo/project and observing that it works, NOT running its automated test
  suites. The guide covers prerequisites, external dependencies (databases,
  services, accounts/tokens), local setup, and a concrete hands-on walkthrough.
  If the repo already has a usable manual-test path (example app, sandbox, dev
  server, playground), the guide explains how to verify by hand using that;
  otherwise it explains how to create a minimal manual-test setup from scratch
  and verify by hand. Use this when the user asks "how do I manually test this
  repo/SDK/project", wants a hands-on testing/verification guide generated, or is
  onboarding and needs to know how to exercise the code by hand.
---

# how-to-test

Produce a single, human-readable Markdown document that lets a developer who has
never seen this repo **manually test it by hand** — go from a fresh clone to
actually using the SDK/app/project and confirming with their own eyes that it
works.

This is about **manual, hands-on verification**, not the automated test suite.
Do **not** document how to run unit/integration/e2e test cases. Instead, document
how a person sits down and exercises the thing: installs it, wires up the minimum
needed to call it, runs a real example or flow, and observes correct behavior.

Two situations, and the guide must handle whichever applies:

- **A manual-test path already exists** — an `examples/` or `samples/` app, a
  `playground/`, a sandbox script, a dev server, a demo page, a REPL/CLI. →
  Document how to use *that* to verify the project by hand.
- **No manual-test path exists** — → Document how to **create a minimal setup**
  (a fresh sample project, link/install the SDK locally, write a small script
  that imports and calls it) and verify by hand.

The output is a **guide for a human to follow**, not a report of work you did.
Every command must be copy-pasteable. Every prerequisite, credential, or manual
step must be called out explicitly and honestly — never hand-wave "set up the
database" when you can give the exact command.

## Operating principles

- **Fan out by default.** Collecting the facts is the bulk of the work — do it in
  parallel, not inline. Spawn subagents for every independent slice (each
  discovery dimension below, and in a monorepo each major package), and reach for
  fan-out again to verify commands or draft independent sections of the guide.
  Send the batch in one message so they run concurrently; the main thread
  synthesizes their findings, it shouldn't be the one reading every file. Only go
  fully inline when the repo is tiny.
- **Evidence over assumption.** Derive everything from the actual repo —
  `package.json`/`pyproject.toml`/`go.mod`/`Cargo.toml`, lockfiles, `README`,
  `examples/`, `docs/`, quickstart snippets, `Makefile`, `docker-compose.yml`,
  `.env.example`. Do **not** invent commands, APIs, or versions. If a step can't
  be verified from the repo, label it clearly as an assumption or a gap.
- **Show the real public API.** For an SDK/library, find the actual entry points,
  exported functions, and init/config a user calls — quote the real import paths,
  function names, and required arguments from the source or README, so the
  sample code in the guide actually compiles and runs.
- **Call out the un-automatable.** Interactive logins (SSO, `gcloud auth`,
  cloud tokens), secrets that aren't in the repo, paid accounts, or anything the
  user must do by hand → flag it with a ⚠️ and say exactly where to get it.
- **Respect monorepos.** If the repo has multiple packages/workspaces, make clear
  which package the user is manually testing and how to point the sample setup at
  the right one (local link, path dependency, workspace alias).
- **Honesty about cost/scope.** If exercising the project needs a live service,
  real credentials, or money, say so up front.
- **Real calls have real consequences.** Manually exercising the thing makes
  *real* side effects — a sample call can write to a production database, emit a
  live analytics event, send a real email/SMS, charge a card, or burn rate/usage
  limits. ⚠️-flag any step that touches a shared or production system, and steer
  the reader to the safe path: a sandbox/test account, a throwaway write key or
  test source, a local/disposable database, or a dry-run/mock mode. Never have
  the guide tell a reader to fire a real call at prod without warning.
- **Match the repo's own conventions.** Quote the real example/script names and
  the project's own quickstart; don't normalize them into generic placeholders.

## Execution plan

### Phase 1 — Discover (fan out)

Confirm the target repo (default: the current working directory). Then launch
parallel `Explore`/`general-purpose` subagents — one per dimension — so the
analysis is fast and broad. Each agent returns concise, quoted facts with
file paths; you synthesize. Cover at least these dimensions:

1. **Project shape & what it is** — is this an SDK/library, an app/service, a
   CLI, or a mix? Languages; package manager(s) and lockfiles; monorepo tool and
   the package list; runtime versions (`.nvmrc`, `.python-version`, `go.mod`,
   `.tool-versions`); how the thing is built/installed.
2. **Existing manual-test paths** — `examples/`, `samples/`, `demo/`,
   `playground/`, sandbox scripts, a dev server / `start` script, a hosted demo,
   a REPL or interactive CLI, quickstart snippets in the `README`/docs. For each,
   note exactly how it's launched and what it demonstrates. This decides whether
   the guide uses an existing path or builds one from scratch.
3. **The public API / how a user actually uses it** — for an SDK/library: the
   entry points and exported functions a consumer calls, how to initialize/
   configure it, required arguments and return values, and the smallest realistic
   call that proves it works. For an app: the main user-facing flow or endpoints.
4. **External dependencies & local setup** — `docker-compose*.yml` and the
   services it starts (DBs, redis, queues, mocks); required databases/brokers and
   how they're seeded/migrated; `.env`/`.env.example` and every var that matters;
   secrets, API keys, tokens, registry auth, cloud/SSO logins; third-party
   services the project talks to. Everything needed to make a real call succeed.
   Also find the **safe path**: any sandbox/test mode, dev source or throwaway
   write key, mock server, or dry-run flag that lets the reader exercise the
   project without hitting production or incurring cost — and note what a sample
   call actually does (writes where, sends what) so its side effects are known.
5. **Run / serve locally** — how to start the app, dev server, or a local build
   of the SDK; ports; how to point a sample project at the local build
   (`npm link`/`pnpm link`, `pip install -e .`, path/workspace dependency); how
   to confirm it's up (health check, demo page, sample request, expected output).

Scale the fan-out to repo size: a small library may need 2–3 agents; a large
monorepo warrants one per dimension (and possibly per major package).

If anything is ambiguous after discovery (e.g. whether a usable example exists,
which package the user means, or whether real credentials are available), ask the
user a focused question rather than guessing.

### Phase 2 — Synthesize the guide

Write ONE Markdown file (see structure below) that any developer can follow start
to finish without prior knowledge of the repo — plain language, defined terms,
numbered do-this-then-that steps (see **Style**). Lead with the fastest hands-on
happy path — the shortest route to "I called it and saw it work" — then layer in
detail. Prefer tables and fenced code blocks. Keep prose tight. For a
large/monorepo repo, fan out the drafting too — one agent per package or per
major section — then stitch the pieces into the single document yourself.
Where practical, fan out a quick verification pass that actually runs the
read-only setup commands (install/link/`--help`, not destructive ones) and, if
feasible, the sample call itself, so the guide's steps are confirmed, not assumed.

### Phase 3 — Deliver

Save the guide to the repo (default: `how-to-test.md` at the repo root; if the
repo already uses a `tasks/` or `docs/` folder for such notes, prefer that and
say where you put it). Then give the user a 3–5 line summary: where the file is,
the single fastest hands-on check to confirm the project works, and any ⚠️
blockers that need their action (credentials, accounts, manual logins).

## Required structure of the generated guide

Adapt section presence to the repo (skip what doesn't apply), but follow this order:

1. **Overview** — 3–6 lines: what the project is (SDK / app / CLI), the tech
   stack, and the one-paragraph mental model of how you test it by hand. Note if
   it's a monorepo and which package this guide targets.
2. **What "testing by hand" means here** — 2–3 lines stating the approach: either
   "use the existing example/sandbox at `<path>`" or "create a small sample
   project that uses the SDK" — so the reader knows the plan before the steps.
3. **Prerequisites** — a table of tools + exact versions to install (with the
   source of truth, e.g. "Node 20.18.3 — from `.nvmrc`") and install hints.
4. **External dependencies & local setup** — ⚠️-flagged, step by step:
   Docker services, databases (start + migrate + seed), env vars (which to set
   and where to get values), secrets/tokens/registry auth, and any interactive
   logins. Give exact commands. State which steps are one-time vs every-run.
5. **Get the project ready** — clone → install → build, as ordered copy-paste
   commands. For an SDK, include how to make a local build consumable (link or
   local install).
6. **The hands-on walkthrough** — the heart of the doc. Pick the path that fits:
   - **Existing example/sandbox/app:** how to launch it, what to do in it, and
     exactly what you should see (output, UI state, response) that proves it works.
   - **No example — build one:** ordered steps to create a minimal sample project,
     install/link the SDK into it, and a small, complete, copy-pasteable sample
     program that imports the real API and makes the smallest meaningful call.
     Show the command to run it and the **expected output**.
   - Include a couple of follow-on things to try (a second function, an error
     case) so the reader can explore beyond the happy path.
   - ⚠️ Before the first real call, state its side effects (writes where, sends
     what, costs what) and point at the safe path — sandbox/test credentials, a
     throwaway write key or dev source, a local/disposable DB, or a dry-run/mock
     mode — so the reader never unknowingly fires at production.
7. **What success looks like** — concretely: the output, status, log line, or UI
   state that means "it works." Call out anything that looks alarming but is fine.
8. **Cleanup / teardown** — how to undo the manual setup once done: stop and
   remove services (`docker-compose down [-v]`), unlink local builds
   (`npm unlink`/`pnpm unlink`, uninstall the editable install), drop or reset any
   seeded/test data, and remove the throwaway sample project. Mark anything that
   leaves global state behind (e.g. a global `npm link`) so nothing lingers.
9. **Troubleshooting** — a table of common failures → cause → fix (e.g. missing
   env var, port in use, registry 401, unlinked/stale local build, auth error).
10. **Quick reference (cheat sheet)** — the handful of commands the developer will
    reuse to set up and run a manual check, in a compact block.

## Style

- **Write for a newcomer, not an expert.** Assume the reader has never seen this
  repo and may be junior or from another stack. Plain language, no unexplained
  jargon — the first time a project-specific term, tool, or acronym appears,
  define it in a few words. If a step has a "why," give it in one short clause so
  the reader understands, not just obeys.
- **Numbered, do-this-then-that steps.** The walkthrough and setup are an ordered
  sequence a reader follows top to bottom without prior knowledge. One action per
  step; after each meaningful step, say what they should now see so they can tell
  it worked before moving on. Never assume an implied step — spell it out.
- Developer-friendly and skimmable: short sentences, real command and API names,
  tables, fenced code blocks with the right language hint.
- Every command and code sample copy-pasteable and runnable; show expected output
  wherever it builds confidence. Prefer a command the reader can paste over a
  sentence describing what to do.
- Use ⚠️ for anything needing credentials, money, or manual action.
- Be explicit about one-time vs repeated steps.
- Don't pad. A small library's guide can be one page; a large project's may be
  several — length should track real complexity, not filler.
