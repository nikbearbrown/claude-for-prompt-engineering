# Chapter 2 — Setting Up a Coding Agent for a Real Repository

*Writing the onboarding map and wiring the verification loop so the agent can close it itself*

---

Here is the misconception that makes most agent setups fail before they begin: people think "setting up a coding agent" means installing a tool. It doesn't. The tool is already installed. What you haven't done is tell the agent how to verify its own work.

Let me show you what that costs.

An engineer points a fresh agent at a real service repo and types: *"Add validation to the `create_user` endpoint and make sure the tests pass."* The agent reads the endpoint, adds a validation block, and writes: *"I've added validation for email format and required fields. The tests should pass now."* It never ran a test. Not because it was lazy — because it had no idea how. Was it `pytest`? `make test`? `npm test` from a sub-directory? A tox environment? The agent guessed that running tests was someone else's job, declared victory, and stopped. This is the open loop from Chapter 1, and it is the default state of an un-onboarded agent: it cannot close a loop whose closing command you never told it.

Now the same repo, but with a `CLAUDE.md` at the root containing a Verification section that reads, in full:

```
## Verification — run these and do not stop until all pass
- Tests:      uv run pytest
- Type-check: uv run mypy src/
- Lint:       uv run ruff check src/
```

Same prompt. This time the agent adds the validation, runs `uv run pytest`, reads `1 failed` — its validation rejected a fixture the existing test expected to pass — narrows the rule, reruns, reads `12 passed`, runs `mypy` and `ruff`, and *then* stops. The loop closed. The difference between the two runs is not a smarter model or a cleverer prompt. It is six lines of a text file that contained the **exact command string** that produces ground truth, plus the standing instruction to keep going until that ground truth came back green.

That is the whole chapter in one contrast.

---

## The onboarding document, not the config file

The reframe that makes everything else intuitive: a persistent instruction file is the answer to the question *"what would I tell a competent new engineer on their first day?"*

You would not hand a new hire a 2,000-line manual restating the style guide your linter already enforces. You would tell them: here is what we build and roughly how it is laid out; here is *why* a couple of non-obvious choices are the way they are; here is how you run the tests, the type-checker, and the build; and here is what not to touch without asking. That is an onboarding map. Persistent instruction files — `CLAUDE.md` for Claude Code, the emerging cross-tool `AGENTS.md` standard — are exactly that, written for an agent instead of a person. They replace the chat system prompt for a stateful session: read at startup, scoped to directories in a monorepo, inherited and merged as the agent moves through the tree.

There is a mechanism fact behind why the file behaves the way it does, and it surprises most engineers. In current Claude Code, `CLAUDE.md` is not injected as the system prompt. It is delivered as a `<system-reminder>` block inside a user message, with a caveat to the effect of: *this context may or may not be relevant; ignore it if it isn't.* Your instructions are **influential, not absolute.** The model weighs them against everything else in the window and is told it may disregard sections it judges irrelevant. A rule that matters for one module, dropped into the global file, is either ignored when off-task or misapplied when the model over-reads its relevance. Vague or conflicting rules get applied inconsistently for the same reason.

The takeaway is: only broadly applicable, clearly stated instructions earn a place in the standing map.

This is also the human-authored layer of what SWE-agent calls the agent-computer interface. Chapter 1 used that work to argue that the interface — how the agent reads the repo and observes results — determines success as much as the model does. The instruction file is the read-side of that interface (where things are, how to find them) and its Verification section is the observe-side (how to get ground truth). Writing this file well is not optional polish. It is authoring the interface the SWE-agent result says is load-bearing.

---

## WHAT, WHY, HOW — and the verification loop as the payload

The map answers three questions. The third is the one this whole book turns on.

![One instruction file divided into stacked bands: WHAT, WHY, working rules, an enlarged emphasized HOW verification band holding three command lines, then handoff and pointers. The verification band is the payload that closes the agent loop. Two pointer arrows lead to load-on-demand documents.](../images/02-setting-up-a-coding-agent-fig-01.png)
![Anatomy of the onboarding map: WHAT, WHY, and an enlarged HOW verification band that carries the exact commands closing the loop.](images/02-setting-up-a-coding-agent-fig-01.png)
*Figure 2.1 — Anatomy of the onboarding map: WHAT, WHY, and an enlarged HOW verification band that carries the exact commands closing the loop.*

**WHAT** is the tech stack, the project structure, a directory map. Where things live, what the runtime and package manager are. A model can increasingly infer this by reading `package.json` or `pyproject.toml`, so keep it thin and pointer-heavy.

**WHY** is the purpose and the non-obvious design intent, so the agent can make judgment calls in situations the rules don't cover. *Why* is the part you cannot get from reading the code. "Dialogue rendering is gameplay-critical, so correctness there outranks performance" is a sentence that changes how an agent weighs a trade-off, and it appears nowhere in the source.

**HOW** is the exact verification commands. This is the payload. Not "run the tests" but the literal string: `uv run pytest`, `npm run typecheck`, `cargo test`, `make lint`. This is the Chapter 1 observe edge made concrete — the mechanically-checkable channel that distinguishes coding from softer agentic domains.

A minimal map in that shape, for a real repo, looks roughly like this — and the whole thing should fit comfortably under a hundred lines:

```
# Project: <name>
A <one-line description>. Runtime <X>, framework <Y>, package manager <Z>.

## Why a few things are the way they are
- <non-obvious design choice + reason the agent cannot infer from code>

## Working rules
- Inspect before you edit; make the smallest change that satisfies the task.
- Stay within the named paths; ASK before editing files outside them.
- Do NOT touch: generated files, migrations, secrets/.env, CI config — without explicit approval.

## Verification — run these; do not stop until all pass
- Tests:      <exact command>
- Type-check: <exact command>
- Lint:       <exact command>
- Build:      <exact command>

## Handoff — when done, report:
- Files changed, tests run and result, any risk or anything you skipped.

## Deeper docs (load on demand)
- See agent_docs/running_tests.md, agent_docs/service_architecture.md, ...
```

Two design choices in that skeleton carry most of the weight.

First, the Verification section carries the standing instruction "do not stop until all pass." That sentence is what converts the command strings from documentation into a *stopping condition.* Without it, the agent has the command and might still declare success without running it — the failure mode from §1.4. With it, the loop's closing condition is the ground-truth signal, exactly where you want it.

Second, the Working rules encode the human gate before you've reached the review chapter. "Stay within named paths; ask before editing outside them" plus a short do-not-touch list — generated code, migrations, secrets, CI config, production dependencies — marks the spots a human must approve. These are the irreversible, high-blast-radius actions no test will catch. A couple of lines in the instruction file keeps the agent inside a verifiable, reversible box by default.

What does *not* belong:

Code-style rules. A linter enforces "2-space indents, prefer `const`" far more reliably than an instruction the model might judge irrelevant. Putting style in the file both wastes instruction budget and gets enforced worse than `ruff` or `eslint` would. Say "run the linter," not "here are the linter's rules."

Task-specific or hotfix notes. "When editing combat, remember the 2024 damage-formula change" is irrelevant on every non-combat task and a misapplication risk when the model over-reads its relevance. It belongs in a load-on-demand document.

Anything that could be a pointer to a sub-file. That is the next section.

---

## Keeping the map small: progressive disclosure and scope

Here is the discipline that separates a setup that works from one that slowly stops working: the standing map must stay small, and the way you keep it small without losing detail is to make most detail load on demand.

![Left: a small root index points down to four documents, two loaded into context and two dormant. Right, separated by a rule: a monorepo tree where a root points to two per-package files, each holding its own verify command. The standing index stays small by pointing to detail pulled in only when relevant.](../images/02-setting-up-a-coding-agent-fig-02.png)
![Progressive disclosure: a thin standing index points to deeper docs and per-package files that load into context only when relevant.](images/02-setting-up-a-coding-agent-fig-02.png)
*Figure 2.2 — Progressive disclosure: a thin standing index points to deeper docs and per-package files that load into context only when relevant.*

The pressure to cut comes from two budgets worth naming separately.

The **token budget** from Chapter 1: everything in the always-loaded file occupies the context window every session, raising the consumed-token count before the task even starts. A bloated file eats the budget you need for actually reading code.

The **instruction budget**: a model follows only so many distinct directives before adherence degrades. Practitioner research puts the ceiling for frontier thinking models on the order of roughly 150–200 distinct instructions, with the agent's own system prompt reportedly consuming around 50 of those before your file even loads — leaving perhaps 100–150. These are practitioner heuristics, not measured constants. But the dangerous property reported past the threshold is worth taking seriously: adherence degrades roughly uniformly. It is not that your 201st rule gets dropped while the first 200 hold; all of them start to slip. If that is right, a bloated file degrades adherence to the rules you cared about most, not just the marginal ones.

Both budgets push the same direction: keep the standing file minimal. The practical ceiling is roughly 80–120 lines, with under 100 better. One production `CLAUDE.md` cited in practitioner writing runs under 60 lines. That is a striking, counter-intuitive claim worth stating plainly because it inverts the engineer's instinct: **shorter outperforms exhaustive.** The pull to "just tell the agent everything" is the strongest and most damaging instinct in this whole chapter.

![Two independent constraints, the token budget and the instruction-adherence budget, each send an arrow converging on a single shared outcome: a minimal standing instruction file. Both pressures point in the same direction, keeping the file small.](../images/02-setting-up-a-coding-agent-fig-03.png)
![Two budgets, token and instruction-adherence, are distinct pressures that both push the standing file toward minimal size.](images/02-setting-up-a-coding-agent-fig-03.png)
*Figure 2.3 — Two budgets, token and instruction-adherence, are distinct pressures that both push the standing file toward minimal size.*

Three techniques keep the map small while depth stays available.

**Progressive disclosure — the `agent_docs/` pattern.** Keep `CLAUDE.md` as an index whose last section points at deeper documents the agent reads only when relevant:

```
agent_docs/
  running_tests.md
  code_conventions.md
  service_architecture.md
  database_schema.md
```

The agent loads `database_schema.md` when it is touching the database and ignores it otherwise. Standing context stays small; depth is on demand. This is the distinction between static context (always loaded, must be minimal) and dynamic context (pulled just-in-time for the task at hand).

**Prefer pointers to copies.** A path-plus-line reference — `see src/render/dialogue.py:142` — stays correct as the code evolves. A pasted snippet of that code goes stale the moment someone edits the original. A stale snippet in your instruction file is a context-rot generator you built on purpose. Point; don't paste.

**Directory-scoped files in a monorepo.** A single giant root file that pastes every package's build steps forces the agent working in `packages/api/` to carry `packages/web/`'s rules — and possibly misapply them. The disciplined layout is a thin root file as an index plus a per-package file each carrying that package's own verify commands:

```
CLAUDE.md                      # thin index for the whole repo
packages/api/CLAUDE.md         # api's exact test/typecheck/build commands
packages/web/CLAUDE.md         # web's exact commands
```

The agent in `api/` sees the right loop — its own test command, not the web package's. The misconception this kills is "give the agent all the context." In a monorepo, that is the token-budget blowout of Chapter 1 at architectural scale.

> **A note on tool specifics (verify before relying on these).** Claude Code merges `CLAUDE.md` files by directory scope and delivers them as the `<system-reminder>` user-message block described above. Other tools use `AGENTS.md`; at least one CLI tool has been reported to enforce a hard project-doc ceiling past which large files silently truncate — a too-long file can vanish without warning. Cursor and Windsurf use `.cursorrules` with their own character caps. `AGENTS.md` is an emerging cross-tool standard, not yet universal. Every byte cap and behavior here is a dated snapshot; the principle — minimal index, progressive disclosure, exact verify commands — is what the book teaches.

---

## What you actually built

Step back and see what "setup" produced, because it reframes the whole endeavor.

You did not configure a tool. You authored the conditions under which the agent can verify its own work and a human knows where the gate is. The exact `test`/`typecheck`/`lint`/`build` commands in the Verification section are the agent's ground truth — the mechanically-checkable signal that the observe step needs in order to close. The Working rules mark the spots where a human must approve. The `agent_docs/` index keeps depth available without drowning the budget.

This is also the most direct enactment of the book's thesis. Chapter 1 argued that an agent is only as good as the verification loop you build around the task. Chapter 2 built the loop: the literal commands that let the agent know whether it succeeded, and the rules that mark where it must stop and ask.

Every chapter in the rest of the book reuses this wiring. TDD makes the failing test the stopping condition. Refactoring leans on the build and the existing suite to prove behavior was preserved. Debugging feeds the stack trace and the failing test as observations. Review is where the human gate you sketched here becomes the chapter's whole subject. Each task type is, from here on, the same question in a new costume: *what is the verification loop, and where is the human gate?* That question is answerable because Chapter 2 established the loop as something written down and runnable.

One honest forward-looking note. As models get better at self-onboarding — reading `package.json`, inferring the test command, building their own repo map — the WHAT portion of your file will shrink, possibly to near nothing. What is unlikely to shrink is the part the model cannot infer from the code: the WHY judgment calls, the do-not-touch and human-gate rules, and arguably even the exact verify commands when a repo has several plausible ones and only one correct one. Don't over-invest in documenting what the agent will soon discover for itself. Do invest in the ground-truth channel and the gate, because those encode decisions, not discoverable facts.

---

## LLM Exercises

**Exercise 2.1 — Apply, produce something.** Take a real repository you work in. Write a `CLAUDE.md` (or `AGENTS.md`) under 100 lines in the WHAT / WHY / HOW shape from this chapter. It must include: a 2-line project overview, at least one WHY sentence the agent could not infer from reading the code, a Working-rules block with a minimal-scope default and an explicit do-not-touch list, and a Verification section with your repo's exact, copy-pasteable test/typecheck/lint/build commands plus the standing instruction "do not stop until all pass." Then prove it: run the same scoped task once with the file present and once with it removed, and report whether the agent ran the verification commands in each case.

**Exercise 2.2 — Analyze.** You are handed a 300+-line `CLAUDE.md`. Rewrite it under the capacity ceiling: produce a map under 100 lines plus an `agent_docs/` index of at least three load-on-demand files showing what you moved out and why. Replace at least two pasted snippets with pointers. In a short paragraph, classify each major cut: did it protect the token budget, the instruction budget, or both — and explain why a code-style rule protects neither, because it belongs in the linter.

**Exercise 2.3 — Apply.** Take a small monorepo (or simulate one with two packages that have different test commands — `pytest` in one, `npm test` in the other). Set up a thin root `CLAUDE.md` index plus two directory-scoped files, each carrying its own verify commands. Run one task in each package and confirm the agent used the correct command for the package it was in. Write one sentence on what would have gone wrong with a single root file pasting both packages' commands.

**Exercise 2.4 — Evaluate (optional).** Set up the same repo two ways: (a) a kitchen-sink `CLAUDE.md` that documents everything — style rules, pasted schema, every hotfix note, around 300 lines; (b) a minimal map with `agent_docs/` progressive disclosure, around 70 lines. Run a representative set of small tasks through both. Track context usage at task start and count rule-adherence misses in each. Report whether the minimal file's adherence actually beat the exhaustive one — testing the "shorter outperforms exhaustive" claim against your own stack rather than taking it on faith.

---

## What would change my mind

The chapter's operational claim is that a minimal onboarding map with exact verify commands outperforms an exhaustive one — that "shorter outperforms exhaustive" is a real effect, not just tidiness. A controlled comparison holding the model and tasks fixed, in which a long, thorough instruction file produced equal or better rule-adherence and task success than a minimal map across a representative task set, would force me to retract the minimalism advice and concede that the instruction-capacity ceiling either doesn't bite at realistic file sizes or is swamped by the value of more complete instructions. The capacity figures are practitioner heuristics, not measured curves; a measurement showing adherence is robust to file length up to far higher limits would specifically weaken this chapter's argument toward "keep it reasonable" rather than "keep it under 100 lines."

Separately, if a generation of models reliably self-onboarded — inferring the correct verify commands, building an accurate repo map, and respecting an inferred do-not-touch boundary without a hand-authored file — then the "you must author the loop" thesis of this chapter would shrink to "you must author the gate and the WHY." The day that agents stop silently declaring success without running tests when handed a repo with no instruction file is the day this chapter's payload starts to become optional.

---

## Still puzzling

**How short is too short?** The chapter pushes hard on minimalism, but there is presumably a floor below which the agent lacks the WHY to make good judgment calls and starts guessing in the other direction. Where the optimum sits — and whether it is the same for a thinking model and a smaller one — is a measurable question this chapter asserts an answer to without a curve behind it.

**Is the "uniform degradation past the instruction ceiling" claim real, or is it selective dropping that looks uniform?** The advice survives either way, but the mechanism — all instructions slip versus the marginal ones get dropped — implies different things about which rules to cut first if you must trim. It deserves a controlled experiment, not a repeated heuristic.

**As self-onboarding improves, what is the irreducible core of the file?** The chapter bets it is the verify commands, the WHY, and the human-gate rules. But if models get good enough at running a repo's own CI to infer the verify commands too, the irreducible core might shrink to just the gate — the one thing that encodes a human's risk decision rather than a discoverable fact.

**Does the `<system-reminder>` delivery position actually matter for behavior, or is it an implementation detail?** The "influential, not absolute" framing leans on reverse-engineered, tool-specific delivery semantics. If `CLAUDE.md` instructions behaved identically whether delivered as system prompt or user-message reminder, the design advice would still hold for budget reasons — but the "the model may judge it irrelevant" justification would be wrong. The two justifications point the same direction today; whether they always will is unconfirmed.

---

## References

- Anthropic. *Best practices for Claude Code.* Claude Code documentation. https://code.claude.com/docs/en/best-practices
- Yang, J., Jimenez, C. E., Wettig, A., Lieret, K., Yao, S., Narasimhan, K., & Press, O. (2024). *SWE-agent: Agent-Computer Interfaces Enable Automated Software Engineering.* NeurIPS 2024. arXiv:2405.15793.
- Anthropic (2025). *Effective context engineering for AI agents.* Anthropic Engineering blog (Sep 2025). https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents
- Anthropic (2025). *Equipping agents for the real world with Agent Skills.* Anthropic Engineering blog. https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills
- HumanLayer (2025). *Writing a good CLAUDE.md* (Nov 2025). https://www.humanlayer.dev/blog/writing-a-good-claude-md
- Piebald-AI. *claude-code-system-prompts.* https://github.com/Piebald-AI/claude-code-system-prompts
- *AGENTS.md* — emerging cross-tool instruction-file convention. [verify current per-tool adoption]

---

**Tags:** claude-md, agents-md, onboarding-map, verification-loop, exact-verify-commands, ground-truth, progressive-disclosure, agent-docs, directory-scoping, instruction-capacity, human-gate, minimal-scope, cli-coding-agents
