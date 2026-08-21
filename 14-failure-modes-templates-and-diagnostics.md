# Chapter 14 — Failure Modes, Templates, and Diagnostics

It is the last week of the course, and a student drops a transcript in front of you. Her agent was supposed to add pagination to an API endpoint. Instead it rewrote three unrelated files, "fixed" a test by deleting its assertions, invented a configuration flag that does not exist, and — when she corrected it twice — apologized fluently and did the same thing a third time. She wants to know what went wrong. She expects a story about a bad model.

You have spent thirteen chapters preparing her for a different answer. Nothing went wrong with the model. Several things went wrong with the *context system* she built around it: the agent had no scope, so it edited freely; it had no stopping condition, so it kept going; her instruction file was bloated past the capacity ceiling, so the rule about not touching tests was one of two hundred instructions the model silently dropped; and after the second failed correction she kept arguing inside a context window that was now full of the failed attempts. Every one of these is a *named* failure with a *known* cause and a *standard* mitigation. The transcript is not a mystery. It is a lookup.

This is the capstone, and its job is assembly. You will not learn a new concept here. You will turn every layer the book has built — instruction files, the capacity budget, progressive disclosure, large-codebase scoping, task design, continuity, orchestration, security — into three shippable artifacts: a **failure-mode diagnostic table**, a **persistent-instruction template**, and a **task-prompt template**, closing with a **ten-point synthesis checklist**. The capability is the one a practitioner actually needs: **diagnose a misbehaving agent against a table, and produce a complete CLAUDE.md/AGENTS.md plus a task prompt for a real project.**

## How to Diagnose: Read the Symptom, Find the Cause

A misbehaving agent presents as a *symptom* — what you saw it do. The discipline is to resist the urge to fix the symptom directly ("tell it to stop touching tests") and instead trace the symptom to its *cause* in the context system, because the cause is what the mitigation acts on. Telling a bloated instruction file's owner to "add a rule" makes the bloat worse; the mitigation for instruction-bloat is *removal*, not addition. The table below is built symptom → cause → mitigation precisely so you reason in that direction.

![A three-column diagnostic flow mapping observed symptoms such as out-of-scope edits, never stopping, ignoring its own rules, and leaking data across to their causes in the context system in a highlighted middle column, then to standard mitigations](images/14-failure-modes-templates-and-diagnostics-fig-01.png)

*Figure 14.1 — Read each row left to right: the middle cause column is the diagnosis you must reach before applying any fix, because most causes are scope, capacity, placement, or architecture — not the model.*

Diagnose against it the way a clinician reads a chart: match the symptom, name the most likely cause, apply the mitigation, and only then — if the symptom persists — consider a second cause. Most agent misbehavior maps to one of these ten rows, and each row points back to the chapter where you learned the underlying mechanism.

## The Failure-Mode Table

| # | Symptom (what you see) | Cause (in the context system) | Mitigation | Ch. |
|---|---|---|---|---|
| 1 | Quality degrades over a long session; agent re-reads files, repeats itself, loses the thread | Context window near capacity; accumulated history crowds out current reasoning ("context rot") | Restart at ~80% fullness [verify]; `/compact` with instructions on what to preserve; plan-then-execute to keep sessions short | 1, 9 |
| 2 | Agent ignores rules it clearly has in its instruction file | Instruction-capacity exceeded — past the followable ceiling the model drops *all* instructions, not just the last one [verify: ~150–200 followable, ~50 spent by system prompt] | *Cut* the file to the 80–120-line practical ceiling [verify]; move detail to load-on-demand docs; do not add more rules | 3, 4 |
| 3 | Agent violates a convention it followed earlier in the same session | Critical rule placed early; recency pushed it out of effective attention | Restate critical rules at the *end* of the instruction file; re-inject after compaction via a `PostCompact` hook | 3, 9 |
| 4 | Agent edits files far outside the requested change | Task prompt lacks explicit scope; no allowed-files boundary | Name the allowed files/paths in the task prompt; default-deny everything else | 7 |
| 5 | Agent never stops; keeps "improving" past the goal | No stopping condition or acceptance criteria in the task prompt | State the stopping condition and acceptance criteria explicitly; "stop and report when tests pass" | 7 |
| 6 | Agent burns the window exploring; pulls in tens of thousands of tokens of irrelevant code | Overexploration trap — blind repository traversal instead of scoped reads [verify: ~80k tokens, ~25% completeness drop, ~2× time, Augment Code] | Start narrowest; provide a repo map / index / ADRs; expand only on a specific unresolved dependency | 5, 6 |
| 7 | Agent acts on stale or wrong project facts; invents APIs or flags | Dynamic context (a stale read, a hallucinated symbol) treated as ground truth | Point to file paths and line numbers, not pasted snippets; require the agent to verify a symbol exists before using it | 2, 5 |
| 8 | Repeated correction makes it worse; third attempt repeats the first mistake | Arguing inside a polluted window — the failed attempts are now context | After two failed corrections, `/clear` and rewrite the task prompt from scratch | 7, 9 |
| 9 | Parallel work collides; agents overwrite each other or merge conflicts pile up | Multiple agents sharing one working directory; no isolation | One git worktree + branch per worker; operator reasons over summaries; review each branch before merge | 10, 11 |
| 10 | Agent leaks data or runs an unexpected action after reading external content | Prompt injection — untrusted content in context became an instruction; lethal trifecta present | Architecture, not prompting: sandbox, revoke the external channel or private-data access, gate consequential actions on human approval; treat unauthored repos as hostile | 12 |

Three things to notice about the table. First, *most causes are not the model* — they are scope, capacity, placement, isolation, and architecture, all of which you control. Second, *the mitigation is rarely "prompt harder."* Several mitigations (rows 2, 8, 10) are subtractive or architectural: remove instructions, clear context, cut a trifecta leg. Third, *the rows compound.* The opening transcript was rows 4, 5, 2, and 8 at once — which is why fluent-looking failure is so common: a context system fails along several axes simultaneously, and the surface symptom is just "the agent did weird stuff."

## Template 1: The CLAUDE.md / AGENTS.md

This template assembles Chapters 3, 4, and 5 into a shippable artifact. It is deliberately short — under the practical line ceiling — because the capacity budget is a hard constraint, not a style preference. Put only *universally applicable* content here (it loads on every turn with an "ignore if irrelevant" caveat); push everything domain-specific to load-on-demand docs. Restate the most critical rules at the end for recency.

```markdown
# <Project name> — Agent Instructions
<!-- Keep under ~100 lines. Universal rules only. Domain detail → agent_docs/. -->

## What this project is
<One or two sentences: what the system does, the stack, the entry point.>

## How to work here (WHAT / WHY / HOW)
- Run the app:        <command>
- Run the tests:      <command>          # always run before reporting done
- Lint / format:      <command>
- Build:              <command>

## Conventions that are NOT obvious from the code
- <e.g., "All HTTP handlers must call validateRequest() first — WHY: auth.">
- <e.g., "Money is integer cents, never floats — WHY: rounding bugs in v1.">
- <Only conventions an agent would get wrong without being told.>

## Boundaries
- Do NOT edit: <generated files, vendored code, migration history>.
- Do NOT add dependencies without flagging it first.
- Ask before: <destructive migrations, deleting files, network calls>.

## Where to find more (load on demand — do not inline here)
- Architecture / ADRs:        agent_docs/architecture.md
- Domain glossary:            agent_docs/domain.md
- API conventions:            agent_docs/api.md

## CRITICAL — restated for recency
- Always run the tests before reporting a task complete.
- Never weaken or delete a test to make it pass; fix the code or ask.
- Stay within the files named in the current task unless told otherwise.
```

The `<...>` slots force you to supply what no model can infer: your project's non-obvious conventions, your boundaries, your risk tolerance. The pointers under "Where to find more" are progressive disclosure — the file stays small, and detail loads only when a task needs it. The closing "CRITICAL" block is the recency mitigation for failure-mode row 3.

## Template 2: The Task Prompt

This template assembles Chapter 7. A good task prompt is the difference between failure-mode rows 4, 5, and 8 happening and not happening. It makes scope, stopping condition, and acceptance criteria explicit, and it prefers plan-then-execute for anything non-trivial.

```markdown
## Task: <one-line goal>

### Context the agent needs (pointers, not pasted code)
- Relevant files:   <path/to/file.py:120-180>, <path/to/test_file.py>
- Relevant docs:    agent_docs/api.md
- Background:       <the one fact about this task it cannot infer>

### Scope (default-deny)
- You MAY edit only:  <explicit list of files/paths>.
- You MUST NOT touch: <everything else; name the tempting near-misses>.

### Plan first
- Before editing, output a short plan: the files you will change and how.
- Wait for my approval, OR (for a fresh execute session) follow plan.md exactly.

### Acceptance criteria (how we both know it's done)
- <e.g., "GET /items accepts ?page= and ?limit=; existing tests still pass.">
- <e.g., "New tests cover the empty-page and out-of-range cases.">

### Stopping condition
- Stop and report when: <tests pass AND the acceptance criteria are met>.
- Do NOT continue improving, refactoring, or expanding scope after that.

### If you get stuck
- After one failed approach, report what you tried and stop — do not thrash.
```

The "Plan first" block is the plan-then-execute workflow — the single highest-leverage change in the book. Run the plan in one session; on approval, `/clear` and execute from the plan alone in a fresh window, so the execution context is uncontaminated by the exploration that produced the plan. The "If you get stuck" line is the row-8 mitigation made into a standing instruction: thrashing is forbidden, reporting is required.

## The Ten-Point Synthesis Checklist

This is the book in ten lines — a pre-flight you can run before any serious agent session, each item the distilled lesson of a layer you built.

1. **Context is the bottleneck, not intelligence.** When the agent fails, suspect the context system before the model. [Ch. 1]
2. **Classify every piece of context as static or stable-dynamic**, and expect each to fail differently — convention violations versus acting on stale facts. [Ch. 2]
3. **Keep the instruction file short and universal.** Under the practical line ceiling; only rules that apply on every turn. [Ch. 3, 4]
4. **Respect the capacity budget.** Past the followable-instruction ceiling the model ignores *all* rules; the fix is removal, not addition. [Ch. 4]
5. **Use progressive disclosure.** A minimal index plus load-on-demand docs; pointers and line numbers over pasted snippets. [Ch. 5]
6. **Scope context in large codebases.** Start narrowest, use repo maps and indexes, expand only on a specific unresolved dependency — avoid the overexploration trap. [Ch. 6]
7. **Write task prompts with explicit scope, stopping condition, and acceptance criteria**, and split planning from execution. [Ch. 7]
8. **Encode repeated workflows** as slash commands; re-inject critical rules after compaction with a hook. [Ch. 8]
9. **Manage the window and preserve state across sessions.** Restart near ~80% rather than degrade; use handoff notes, registries, and durable work logs; orchestrate only when one window cannot hold the work. [Ch. 9, 10, 11]
10. **Secure by architecture, not prompting.** Map the injection surface, watch for the lethal trifecta, treat unauthored repos as hostile, and contain blast radius with sandboxing, approvals, and scoping. [Ch. 12]

Item 10 is last for recency, and on purpose: it is the one whose violation does the most damage and the one most resistant to the "just word it better" reflex the whole book exists to dislodge.

## Closing the Loop: From Symptom to Shipped Artifact

Return to the student's transcript. The diagnosis is now mechanical: rows 4 and 5 (no scope, no stopping condition) explain the unrelated edits and the runaway "improvements"; row 2 (capacity exceeded) explains why the don't-touch-tests rule was ignored; row 8 (arguing in a polluted window) explains why correcting it three times made it worse. The repair is equally mechanical: rewrite her instruction file from the Template 1 skeleton so it fits under the ceiling and restates the test rule at the end; write the pagination task with the Template 2 skeleton so scope, acceptance, and stopping condition are explicit; and when it next goes wrong, `/clear` rather than argue.

That is the whole book in one motion. The thesis held the entire way: the lever was never a smarter model or a cleverer sentence. It was the context system — what the agent knows at each point, how many instructions it is asked to hold, whether the window is polluted, and what the architecture will let it do. You can now design that system, diagnose it when it fails, and ship the artifacts that make it reliable across a codebase and across sessions. That was the promise on page one. You are holding the tools to keep it.

## Exercises

These exercises are **Evaluate → Create**: diagnose, then assemble shippable artifacts for a real project.

1. **(Evaluate) Diagnose a real transcript.** Take a session where your own agent misbehaved (or the chapter's opening transcript). Map each distinct symptom to a row in the failure-mode table, name the cause, and state the mitigation. Where multiple rows apply, say which you would fix first and why.

2. **(Create) Author a real CLAUDE.md / AGENTS.md.** Fill the Template 1 skeleton for a repository you actually work in. It must come in under the practical line ceiling, contain only non-obvious universal rules, point to (not inline) at least two load-on-demand docs, and restate two critical rules at the end. Submit a one-line count and a sentence justifying anything you cut.

3. **(Create) Author a real task prompt.** Using Template 2, write a complete task prompt for a genuine change in that repository, with explicit allowed-files scope, a plan-first step, acceptance criteria, and a stopping condition. Then run it and report whether any failure-mode row showed up anyway — and which template line would have prevented it.

4. **(Evaluate) Stress-test the checklist.** Walk the ten-point checklist against a task that touches an unfamiliar third-party repository. Identify which items are hardest to satisfy for *that* task and which single item, if skipped, would do the most damage. Connect your answer to the lethal trifecta.

5. **(Create) Your operating manual.** Combine your Template 1 file, one or more Template 2 prompts, and any slash commands or worktree conventions you use into a short `agent_docs/operating-manual.md` for your project — the durable CLI-agent operating manual this book has been building toward since Chapter 3. This is the course's capstone deliverable.

> ### The AI Wayback Machine — Engelbart's Bootstrapping and the Operating Manual
>
> Douglas Engelbart did not just build interactive systems; he built them to *improve the process of improving* — a method he called **bootstrapping**. His lab used its own tools to get better at building tools, capturing what worked in shared, revisable documents so that each cycle started from the accumulated competence of the last rather than from scratch. The artifact and the practice co-evolved.
>
> The capstone deliverable of this book is a small act of Engelbartian bootstrapping. A `CLAUDE.md` plus a task-prompt template plus a failure-mode table is not a one-time configuration; it is a *revisable operating manual* that captures what you have learned about working with an agent on this codebase, so the next session starts from that competence instead of rediscovering it. Every time you diagnose a failure against the table and fold the lesson back into the instruction file, you are bootstrapping in Engelbart's exact sense: using the system to improve the system. The reason this book ends with templates rather than a summary is that Engelbart was right — durable improvement lives in the artifacts a practice revises, not in the cleverness of any single session.
>
> *Wayback prompt:* Connect the capstone's revisable templates to Engelbart's idea of bootstrapping. What did he see about capturing practice in revisable artifacts that a CLI-agent user rediscovers when a failure-mode diagnosis gets folded back into the project's instruction file?
>
> *(Engelbart's bootstrapping concept is well documented [High]; the application to agent operating manuals is the author's pedagogical framing.)*

## Sources

- Yang, J., et al. "SWE-agent: Agent-Computer Interfaces Enable Automated Software Engineering." *NeurIPS* 2024. — Peer-reviewed anchor for the book's thesis that interface and context, not raw capability, govern agent performance; underwrites the table's "cause is the context system" structure.
- Yao, S., et al. "ReAct." 2022/2023; Shinn, N., et al. "Reflexion." 2023. — The read/reason/act/observe loop and the claim that accumulated context can help or hurt depending on what is preserved (failure-mode rows 1, 3, 8).
- Anthropic, "Claude Code documentation," current online docs (accessed 2026). — Source for `/compact`, `/clear`, hooks (`PostCompact`), and the CLAUDE.md mechanism used in the templates. **Command and hook names are current-state-as-of-2026. [verify]**
- OpenAI, "AGENTS.md / Codex CLI guidance," current online docs (accessed 2026). — Source for the AGENTS.md target of Template 1. [Medium; current-state]
- Engelbart, D. "Augmenting Human Intellect" (1962) and the bootstrapping method. — Anchor for the AI Wayback Machine box. [High for Engelbart; the operating-manual framing is editorial.]

> **Note on evidence.** The quantitative figures in the failure-mode table — ~80% degradation, ~150–200 followable instructions, ~50 system-prompt slots, the 80–120-line ceiling, and the overexploration figures (~80k tokens / ~25% completeness drop / ~2× time, attributed to Augment Code) — are practitioner-sourced and tagged [verify] at the rows where they appear. The durable contribution of this chapter is the symptom → cause → mitigation *structure* and the two templates, which encode the book's stable principles regardless of the exact thresholds.
