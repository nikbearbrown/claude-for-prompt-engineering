# Chapter 3: Persistent Instruction Files — CLAUDE.md and AGENTS.md

## A scene: the file that should have helped

You open a fresh session on a service you maintain. You have done the responsible thing. Two months ago, after the agent kept reaching for `npm` in a `pnpm` repository, you wrote it down. You opened the project's `CLAUDE.md` and added the rule. Then you added the one about not editing generated files. Then the one about the staging database. Then, because you were tired of repeating yourself, you pasted in the team style guide, the onboarding doc, a paragraph on your personal preference for early returns over nested conditionals, and a long note-to-self about a refactor you might do someday. The file grew to four hundred lines. It felt thorough. It felt like diligence.

Today you ask the agent to add a field to an API response, and it runs `npm install`.

The rule was *right there*, in the file, loaded into the session before you typed a word. The model is not weak; it is one of the strongest available. So why did the instruction not land? The answer is the spine of this book, and it is the opposite of the intuition that made you keep adding lines. **The problem was not that the rule was missing. The problem was that it was buried — drowned among four hundred lines of mostly irrelevant text the agent was asked to hold all at once.** Context is the bottleneck, not intelligence. You did not have an instruction problem. You had a context-engineering problem, and you made it worse with every line you added.

This chapter is about the first and most durable lever you have over an agent's behavior: the persistent instruction file. By the end you will be able to author one that loads reliably and earns its place — and, just as important, you will know what to keep *out* of it.

## What a persistent instruction file actually is

In a chat interface, the model's standing instructions live in a system prompt you mostly do not control. In a CLI coding agent, that role is filled by a file that sits in your repository: `CLAUDE.md` for Claude Code, `AGENTS.md` for the cross-tool convention adopted by Codex CLI and others. [Medium; current-state] It is plain Markdown. It is version-controlled alongside your code. It is the static context the agent carries into *every* session in that project, before it has read a single file or run a single command.

That last clause is the whole point. Recall the distinction from Chapter 2: an agent's context divides into **static** content — the system prompt, the instruction file, standing rules that are the same every session — and **dynamic** content — the files it reads, the command output it observes, the conversation as it accumulates. The instruction file is the most important piece of static context you can write, because it is the only part of the agent's standing knowledge that *you* author and that persists without you re-typing it. [High]

Dynamic context fails one way: it goes stale, and the agent acts on information that was true three edits ago. Static context fails a different way: it encodes a convention, and when the convention is wrong, or buried, or contradicted, the agent violates it confidently across every task. [High] The instruction file is where you fight the static-context failure mode. Getting it right is not housekeeping. It is the difference between an agent that respects your project's reality and one that fluently ignores it.

## How the file loads — and why that changes what belongs in it

Here is the mechanism, and the mechanism is the lesson. When you start a session, the agent does not paste your `CLAUDE.md` in as a fresh user instruction the way you would paste a question. Based on reverse-engineering of the Claude Code system prompt by practitioners, the file's contents are injected into the conversation wrapped in a `<system-reminder>` block, delivered inside a user-role message, and — this is the part that should govern everything you write — accompanied by a caveat telling the model that the content may not be relevant to the current task and to *ignore it if it is not*. [Medium; practitioner-sourced, Piebald-AI reverse-engineering]

Read that again, because it inverts the naive model of how these files work. You did not write a command. You wrote a *standing note that the agent has been explicitly told it may disregard.* The agent decides, task by task, whether your instruction applies. This is not a bug; it is what makes the file usable at all — you would not want every rule about database migrations consulted when the task is fixing a typo in a comment. But it has a sharp consequence:

> **Only universally-applicable content belongs in a persistent instruction file.** If a line is true and important for *every* task in the project, the "ignore if irrelevant" caveat never fires against it, and it does its job. If a line is relevant to only one kind of task, the agent is being invited to ignore it most of the time — and the habit of ignoring lines does not stay neatly contained.

This is why the four-hundred-line file failed. Most of its lines were task-specific. The agent, told it could ignore irrelevant content, was practicing exactly that judgment across hundreds of lines on every single request. The one universally-applicable rule — *use pnpm* — was sitting in a haystack the model had been instructed to treat as mostly hay. The delivery mechanism turned your thoroughness into noise.

There is a second consequence, and it connects to a result you will meet formally in Chapter 4. Long contexts are not neutral containers. The empirical work on how models use long inputs — Liu and colleagues' "Lost in the Middle" being the canonical study — shows that information placed in the middle of a long context is retrieved and acted upon *less reliably* than information at the beginning or the end. [High; peer-reviewed, Liu et al. 2024, TACL] A rule buried on line 230 of 400 is not just competing for attention; it is sitting in the position the model is *least* likely to use. The file's length and the rule's position are doing measurable harm.

## What goes in: WHAT, WHY, HOW

If task-specifics and length are the enemies, what survives? A useful three-part test: a line earns its place if it tells the agent **WHAT** the project is, **WHY** a non-obvious constraint exists, or **HOW** to do something the agent cannot infer and would otherwise guess wrong.

**WHAT** is orientation. One or two lines naming what this codebase is and the shape of it, so the agent does not have to reconstruct your architecture from scratch on every task. "This is a Django REST backend for an internal scheduling tool; the API lives in `api/`, the workers in `tasks/`."

**WHY** is the rule with its reason attached. Not "don't touch `migrations/`" but "migrations are applied in CI against a shared staging DB; editing an existing migration corrupts other developers' state, so never edit one — always add a new one." The reason matters because an agent given a bare prohibition will often find a clever, technically-compliant way around it; an agent given the *reason* generalizes the constraint correctly to cases you did not enumerate. This is the situated-action problem in miniature, and we return to it in this chapter's Wayback box.

**HOW** is the non-inferable procedure. The exact build command, the test command, the way your project wants the agent to verify its own work. "Run `make check` before considering any change complete; it runs lint, type-check, and the fast test suite." An agent that knows your verification command will use it; an agent that does not will either skip verification or invent a plausible-looking command that does the wrong thing.

Notice what these three have in common: each is true for essentially every task, and each is something the agent genuinely cannot get right by reading the code alone. That is the bar. WHAT, WHY, HOW — universal and non-inferable.

## What stays out: style and task-specifics

Two large categories do *not* belong, and both are tempting precisely because they feel like good documentation.

**Style.** Formatting conventions — quote style, line length, import ordering, early-return preferences — do not belong in the instruction file, for two reasons. First, they are better enforced by tools that cannot be ignored: a formatter and a linter run in your verification command will fix style deterministically, where a prose rule only *asks*. Second, every style line you add spends capacity (Chapter 4) and pushes your load-bearing rules toward the unreliable middle of the file. If you find yourself writing a style guide into `CLAUDE.md`, you are using the wrong tool; point the agent at the linter instead. [Medium]

**Task-specifics.** Anything relevant to one kind of task — the detailed procedure for adding a new payment provider, the gotchas of the legacy import pipeline, the full schema of the analytics warehouse — does not belong in the always-loaded file. It is real, valuable knowledge. It is just not *universal*. Its home is a load-on-demand document the agent reads only when the task calls for it, which is the entire subject of Chapter 5. For now, hold the principle: **the persistent file is an index of what is always true, not a warehouse of everything that is sometimes true.**

## Restate the critical rules at the end

One more structural move, and it follows directly from the long-context result. If a small number of rules are genuinely non-negotiable — the ones whose violation causes real damage — restate them, briefly, at the very end of the file. [Medium; practitioner-sourced]

The reasoning is recency. The end of a context is a high-attention position; it is the last thing the model reads before it begins to act, and it is empirically among the most reliably used regions. [High] A short closing block — three or four lines, no more — that repeats your hardest constraints buys you a second placement in a position the model attends to. This is not redundancy for its own sake; it is deliberately spending a little length to defeat the middle-of-context blind spot for the rules you can least afford to have ignored. Keep the block tiny. If you find yourself restating fifteen rules, you do not have a recency problem; you have a length problem, and Chapter 4 is about to make that quantitative.

## A real worked example

Here is the four-hundred-line file from the opening scene, rebuilt. Not expanded — collapsed. This is roughly sixty lines, and every line earns its place by the WHAT/WHY/HOW test.

```markdown
# CLAUDE.md — scheduling-service

## What this is
Internal scheduling backend. Django + DRF.
- API endpoints: `api/`
- Background jobs: `tasks/` (Celery)
- Shared models: `core/models/`
Generated files live in `*/generated/` — never edit by hand; they are
rebuilt by `make codegen`.

## How to work here
- Package manager is **pnpm** for the frontend in `web/`. Never run `npm`.
- Python deps are managed with **uv**, not pip. Use `uv add`, not `pip install`.
- Before any change is "done", run `make check`
  (ruff + mypy + the fast pytest suite). A change that fails `make check`
  is not finished.
- Style is enforced by ruff; do not hand-format. Run `make fmt` if unsure.

## Why some things are off-limits
- Migrations in `*/migrations/` are applied in CI against a shared staging
  DB. Editing an existing migration corrupts other developers' state.
  Always create a NEW migration; never edit one that already exists.
- `core/billing/` is under a compliance freeze. Do not modify it without an
  explicit instruction naming a ticket. If a task seems to require touching
  it, stop and say so rather than working around the freeze.

## Domain docs (load on demand — see Chapter 5)
For task-specific detail, read the matching file before starting:
- Adding a scheduling rule  → `agent_docs/scheduling.md`
- Touching the import pipeline → `agent_docs/imports.md`
- Analytics warehouse schema → `agent_docs/analytics.md`

## Critical rules (restated)
1. Never run `npm`; use `pnpm`. Never use `pip`; use `uv`.
2. Never edit an existing migration — always add a new one.
3. Never modify `core/billing/` without an explicit ticket reference.
4. A change is not done until `make check` passes.
```

Walk the file against the principles. The "What this is" block is WHAT — pure orientation, true for every task. "How to work here" is HOW — the non-inferable commands, plus a pointer to the linter *instead* of an inline style guide. "Why some things are off-limits" is WHY — each prohibition carries its reason, so the agent generalizes the constraint (it will not "work around" the billing freeze, because it knows the freeze is the point). The "Domain docs" block is the index that keeps the four hundred lines of task-specific knowledge accessible without loading it — a forward reference to Chapter 5. And the closing "Critical rules" block restates the four constraints whose violation costs the most, in the high-attention end position.

![Candidate lines pass a "universal & non-inferable" gate: WHAT/WHY/HOW enter the file while style is diverted to the linter and task-specifics to on-demand docs; the resulting file ends with a restated critical-rules block.](images/03-persistent-instruction-files-claude-md-and-agents-md-fig-01.png)

*Figure 3.1 — Only universal, non-inferable lines earn a place; style goes to the linter, task-specifics to on-demand docs, and the few critical rules are restated at the end where recency helps.*

What is *gone*: the onboarding doc, the team-history paragraph, the personal early-return preference, the someday-refactor note, and the inline style guide. None of them was true for every task. None survived the universal-and-non-inferable bar. The file went from four hundred lines the agent was told to mostly ignore, to sixty lines it can hold and use.

## Exercises (Apply)

These exercises ask you to *apply* the chapter's test to real artifacts. You will produce a file, not an opinion about files.

**Exercise 3.1 — Triage an existing file (Apply).** Take a `CLAUDE.md` or `AGENTS.md` you (or a teammate, or an open-source project) already has — or the four-hundred-line file is fine if you have one. Go line by line and tag each line WHAT, WHY, HOW, STYLE, TASK-SPECIFIC, or OTHER. *Deliverable:* the annotated file plus a one-sentence verdict on each line that is *not* WHAT/WHY/HOW: where should it live instead (linter, domain doc, deleted)? *What this surfaces:* whether you can tell universal context from everything else — the single judgment the rest of the chapter depends on.

**Exercise 3.2 — Author from scratch (Apply).** For a repository you actually work in, write a `CLAUDE.md`/`AGENTS.md` from a blank file. Constraint: it must fit on one screen without scrolling (you will get a hard number for this in Chapter 4; for now, one screen). It must contain a WHAT block, a HOW block with the real build/test/verify commands, at least one WHY rule with its reason attached, and a restated critical-rules block at the end. *Deliverable:* the file, committed to the repo. *What this surfaces:* whether you can resist the urge to be thorough and instead be *universal*.

**Exercise 3.3 — Test the load and the recency effect (Apply).** Start a fresh session on the repo from 3.2. Without referencing the file, give the agent a task that should trigger one of your critical rules (e.g., ask it to "install the new dependency" in a repo where you forbid `npm`/`pip`). Observe whether it obeys. Then move that rule from the end of the file into the middle, restart, and try again. *Deliverable:* a short log of both runs and what changed. *What this surfaces:* the recency effect in your own repo — not as a claim you read, but as a behavior you produced.

## Bridge

You have a file that loads, and a test for what belongs in it. But you have also been leaning on words like "short" and "one screen" without a number behind them. That softness is about to end. The reason a persistent file must stay small is not taste and not aesthetics — it is a hard capacity constraint with an alarming failure mode: past a threshold, an overloaded file does not cause the agent to ignore the *newest* rule. It causes the agent to ignore *all of them at once.* Chapter 4 puts the number on it.

> **AI Wayback Machine — Lucy Suchman, *Plans and Situated Actions* (1987).** Long before anyone wrote a `CLAUDE.md`, the anthropologist Lucy Suchman sat at Xerox PARC watching people fight with a "helpful" copier whose instructions assumed it knew what the user was doing. Her argument, developed across studies of human-machine interaction, was that *plans are not the same as the actions that carry them out.* A plan is a resource — a standing intention consulted against the messy specifics of a real situation — not a script that executes itself. People do not follow instructions; they *interpret* them against what is actually happening in front of them, discarding the ones that do not fit. That is, almost exactly, what the `<system-reminder>` "ignore if irrelevant" caveat formalizes: the instruction file is a *plan*, and the agent treats it as a situated resource, consulting it against the task and setting aside what does not apply. What Suchman saw in 1987 is what every CLI-agent user is rediscovering now — write your standing instructions as resources for a situated interpreter, not as commands for an obedient executor, because an obedient executor is not what you have. The four-hundred-line file failed for the reason Suchman could have predicted: it confused a warehouse of plans with the single, universal plan the situation could actually use.

---

## Sources

- Yao, Shunyu, et al. "ReAct: Synergizing Reasoning and Acting in Language Models." arXiv:2210.03629, 2022/2023 — the read/reason/act/observe loop that makes instruction files a control discipline rather than a wording trick.
- Yang, John, et al. "SWE-agent: Agent-Computer Interfaces Enable Automated Software Engineering." *NeurIPS*, 2024 — the peer-reviewed anchor that interface design, not just model capability, governs agent performance.
- Liu, Nelson F., et al. "Lost in the Middle: How Language Models Use Long Contexts." *Transactions of the ACL*, 2024 — the empirical basis for the middle-of-context blind spot and the recency placement of critical rules.
- Suchman, Lucy A. *Plans and Situated Actions: The Problem of Human-Machine Communication.* Cambridge University Press, 1987.
- Anthropic. "Claude Code documentation." Current online docs (accessed 2026-05) — `CLAUDE.md` behavior; treat as current-state.
- OpenAI. "Codex CLI / AGENTS.md repository guidance." Current online docs (accessed 2026-05) — the cross-tool `AGENTS.md` convention; treat as current-state.
- Piebald-AI. Claude Code system-prompt reverse-engineering, 2025 — practitioner source for the `<system-reminder>` delivery and the "ignore if irrelevant" caveat. Mark as practitioner-sourced; verify against vendor docs before print. `[verify]`
- HumanLayer. "Writing a good CLAUDE.md." 2025 practitioner source — field guidance on file length and what belongs in persistent context. `[verify]` on numeric claims.
