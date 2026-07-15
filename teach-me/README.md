# teach-me

Learn any concept from a patient expert tutor — without getting buried in a wall of text.

Run it with a topic and Claude becomes a tutor that **meets you at your level**, explains in **plain English, one step at a time**, and **pauses after every step** so you can ask questions or say "got it" before moving on. Nothing is dumped on you at once.

For fast-moving topics (platform SDKs, frameworks, tooling, vendor policies), it **checks current sources before teaching** — so the lesson covers what's true today, with links, instead of stale training-data facts.

When a concept clicks better by *doing*, it offers to **build a small runnable project** with you, growing it one piece at a time. Along the way it runs short **multiple-choice checks** — you pick the right answer, and it explains why that answer is right and the others are wrong — so gaps get caught and re-taught instead of quietly piling up.

The goal isn't a quick answer. It's that you walk away able to **use the concept** in your own work and **explain it to someone else**.

## Install

```bash
/plugin marketplace add SKannaniOS/claude-skills
/plugin install teach-me@skannanios
```

Then:

```
/teach-me:teach-me <concept or question>
```

For example: `/teach-me:teach-me closures in JavaScript`

## How a session goes

1. **Pick your level** — noob, junior, senior, or professional. Everything after this adapts to it.
2. **See the roadmap** — a short outline of what you'll cover, which you can reorder or trim.
3. **Choose hands-on or inline** — build a small runnable sample project, or stick to code examples in chat.
4. **Learn step by step** — one idea at a time, each followed by a pause for your questions.
5. **Quick understanding checks** — multiple-choice questions between steps and at the end, with explanations.
6. **Keep a recap** — at the finish, optionally save a cheat-sheet (mental model, key points, gotchas, and how to explain it to others) to revisit later.
