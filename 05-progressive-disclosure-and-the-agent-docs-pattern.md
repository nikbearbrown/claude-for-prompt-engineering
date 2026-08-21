# Chapter 5: Progressive Disclosure and the agent_docs Pattern

## A scene: the knowledge you cannot afford to load

You finished Chapter 4 with a relocation table full of debts. The auth-decorator rule survived. So did "use the repository pattern." But a dozen entries said *move this somewhere else* — the full procedure for adding a payment provider, the seven gotchas of the legacy import pipeline, the analytics warehouse's column-by-column schema, the reason the scheduling engine treats Sundays specially. All of it is real. All of it has cost you a production incident at least once. None of it is universal, so none of it earned a slot in the always-loaded file. You cut it. And now it lives nowhere the agent can reach.

So watch what happens next. A teammate asks the agent to add a new entry to the import pipeline. The agent has never seen the seven gotchas — you deleted them from `CLAUDE.md` to get under budget — so it does the only thing it can: it explores. It reads the import module, then the three modules it imports, then the config, then a test file, then a utility it misreads as relevant. Twenty file-reads later it has reconstructed maybe half of what your deleted paragraph said plainly, it has burned a third of the context window doing it, and it confidently writes code that trips gotcha number four.

This is the bind progressive disclosure exists to resolve. You have more knowledge than the budget can hold, but the agent needs that knowledge *exactly when the task demands it and not before.* The answer is not "load everything" (Chapter 4 forbade it) and not "load nothing" (the scene just showed the cost). The answer is a third thing: **load a tiny index always, and load the detail on demand.** This chapter is where you build it. The Bloom target is Apply moving to Create — you will not just use the pattern, you will design one for a real project.

## The principle: minimal index, on-demand detail

Progressive disclosure is an old idea (the Wayback box names its origin) with a precise meaning here. You split your project's agent-facing knowledge into two tiers:

- **Tier 1 — the always-loaded index.** This is your `CLAUDE.md`, kept under the Chapter 4 ceiling. Its job shrinks: it holds the universal WHAT/WHY/HOW *and a map* — a short list of pointers saying "for *this* kind of task, read *that* document first." It spends a slot or two on each pointer, not on the knowledge itself.
- **Tier 2 — the on-demand domain docs.** These are separate files, conventionally collected in an `agent_docs/` directory, each covering one domain in whatever depth it needs. They are *not* loaded into every session. The agent reads one only when the index has told it that this task touches that domain.

The economics are the whole point. A pointer costs one instruction slot. The document it points to might be three hundred lines — but those three hundred lines spend *zero* of the live instruction budget until the moment they are needed, and when they are read, they enter context as *data the agent consults*, not as standing imperatives it must hold against every action. You have converted an always-on cost into an on-demand one. That conversion is the lever. It is the same move Chapter 4's Wayback box called chunking, now made structural: the index is the chunk, the domain doc is the detail the chunk unpacks to.

And it directly defeats the opening scene. The agent asked to touch the import pipeline does not explore for twenty reads. It reads the index, sees "import pipeline → `agent_docs/imports.md`," reads that one document, learns all seven gotchas in one deliberate read, and writes code that respects gotcha four. The knowledge you "deleted" was never gone — it was *relocated to a place the agent reaches on purpose instead of reconstructs by accident.*

![Before/after diagram showing a bloated CLAUDE.md with every domain inlined being converted into a lean always-loaded index of conditional pointers that triggers an on-demand read of only the one agent_docs file the task touches.](images/05-progressive-disclosure-and-the-agent-docs-pattern-fig-01.png)

*Figure 5.1 — Progressive disclosure splits agent-facing knowledge into a tiny always-loaded index and on-demand domain docs; the pointer costs one slot, the doc spends zero budget until the task proves it needed.*

## Building the index

Here is the transformation, end to end. Start from the bloated file's relocation table and produce the two tiers.

The Tier-1 index — the part of `CLAUDE.md` that does the disclosure work — looks like this:

```markdown
## Domain docs (read the matching file BEFORE starting the task)
Most project-specific detail lives in `agent_docs/`. Do not guess from the
code when a doc exists. Before you start, check this map and read the doc:

| If the task touches…              | Read first                     |
| --------------------------------- | ------------------------------ |
| The import pipeline               | `agent_docs/imports.md`        |
| Payment providers / billing flow  | `agent_docs/payments.md`       |
| The analytics warehouse / reports | `agent_docs/analytics.md`      |
| Scheduling rules & calendar logic | `agent_docs/scheduling.md`     |

If a task touches none of these, you do not need any domain doc — proceed.
```

Three design choices in that small block carry most of the value. First, the instruction is **imperative and specific**: "read the matching file BEFORE starting," not "docs are available." A passive mention gets the "ignore if irrelevant" treatment from Chapter 3; a clear conditional imperative ("if the task touches X, read Y") survives it, because the condition makes the relevance explicit. Second, the explicit **"do not guess from the code when a doc exists"** is load-bearing: it pre-empts the exploration spiral from the opening scene by telling the agent that reading the doc beats reconstructing it. Third, the closing line — "if none of these, proceed" — prevents the opposite failure, an agent that reads all four docs defensively on every task and re-bloats its own context. Disclosure must be *progressive*, which means *conditional*: the right amount, only when needed.

## Writing a domain doc

The Tier-2 document is where the deleted knowledge goes to live well. Here is `agent_docs/imports.md`, the home for the seven gotchas:

```markdown
# agent_docs/imports.md — Import pipeline

## What it does
Ingests partner CSVs from `s3://imports-inbound/` into `core.models.Record`
via `tasks/import_pipeline.py`. Runs nightly; also triggerable by hand.

## Before you change anything
- The pipeline is **idempotent by `(partner_id, external_id)`**. New writes
  MUST preserve that key or you create duplicate records on re-run.
- Partner "Acme" sends timestamps in local time with NO timezone. Convert
  with `acme_tz()` in `tasks/_tz.py`. Other partners send UTC. Do not assume.
- Rows failing validation go to `import_errors`, never dropped silently.
  Any new validation must follow this; silent drops have caused two incidents.

## How to verify a change
1. `make import-fixtures`  (loads the golden CSV set)
2. `pytest tasks/tests/test_import_pipeline.py`
3. Re-run the same fixtures a second time and assert record count is
   UNCHANGED — this is the idempotency check and the one people forget.

## Pointers, not copies
The full partner-format spec is in `docs/partners/*.md`. Read the specific
partner's file only if the task names that partner. Do not paste it here.
```

Notice the final section, because it is the discipline within the discipline. Even a domain doc should *point* rather than *copy*. The full partner-format spec is large and itself task-specific; the import doc references it by path instead of inlining it, so the agent loads the partner spec only if the task names that partner. Progressive disclosure nests. You do not flatten the whole knowledge base into one big doc; you build a shallow tree the agent descends only as far as the task requires.

## Pointers over copies: prefer file-paths and line-numbers

That nesting reveals a principle worth stating on its own, because it applies everywhere in context engineering, not just to domain docs:

> **Prefer pointers to copies.** When you want the agent to know about a piece of code, give it the *path and line range* — `core/models/record.py:40–95` — not a pasted snippet. When you want it to know about a spec, give the file path, not the inlined text.

Three reasons, all flowing from the book's spine that context is the bottleneck. First, a pasted snippet is **stale the moment you paste it**; the file it came from keeps changing, and now your doc and the code disagree, with the agent trusting the wrong one. This is the static-versus-dynamic split from Chapter 2 biting from a new angle: a copy is static context that silently rots, and stale static context produces the most insidious failures, because the agent acts confidently on information that *used to be* true. A path is always current — the agent reads the live file, which is dynamic context that cannot rot because it is fetched fresh. Second, a pointer **spends almost no context** until it is followed, where a copy spends its full length whether or not it is relevant — the same on-demand-versus-always-on economics as the index itself, and the same instruction-budget arithmetic from Chapter 4: a path is one slot, a hundred-line paste is a hundred lines of competing material. Third, a pointer **teaches the agent to ground in reality**: it goes and reads the actual code rather than reasoning over your possibly-outdated transcription of it. The habit compounds — an agent trained by your docs to follow pointers to source is an agent that checks the territory instead of trusting the map, which is exactly the behavior you want when the stakes are a production change. This is why mature instruction files read like maps — "the auth logic lives in `core/auth/`, start at `middleware.py`" — rather than like anthologies of code excerpts. You are giving the agent coordinates, and trusting it to navigate, because the territory is more reliable than any copy of it you could keep. [Medium]

![Contrast diagram: a pasted code snippet in a doc diverges from its source file as the source changes, ending in a red STALE marker where doc and code disagree; a path-plus-line-range pointer instead reads the live file and stays current.](images/05-progressive-disclosure-and-the-agent-docs-pattern-fig-02.png)

*Figure 5.2 — A pasted copy is static context that silently rots as the source changes; a path-and-line pointer is dynamic context that cannot rot because the agent fetches the live file fresh.*

## The tool-level formalization: Anthropic Agent Skills (SKILL.md)

What you have built by hand — a conditional index that triggers an on-demand load — the tooling now formalizes. Anthropic's **Agent Skills** make progressive disclosure a first-class feature rather than a convention you maintain in Markdown. [Medium; current-state]

A Skill is a directory containing a `SKILL.md` file. The `SKILL.md` carries metadata — crucially a *name* and a *description* of when the skill applies — and a body of instructions and resources. The agent is shown the lightweight metadata (name + description) for available skills as part of its standing context, but the *body* is loaded only when the agent judges, from the description, that the skill is relevant to the current task. [Medium; current-state] That is precisely the two-tier pattern: the description is the index entry (cheap, always present), the body is the domain doc (expensive, loaded on demand), and the matching is automatic rather than something the model has to remember to do by reading your table.

A minimal `SKILL.md` for the import domain:

```markdown
---
name: import-pipeline
description: >
  Use when adding, modifying, or debugging partner CSV imports, the nightly
  ingestion job, or anything touching tasks/import_pipeline.py. Covers
  idempotency, timezone handling, and the error-routing rules.
---

# Import pipeline

(…the same idempotency / timezone / error-routing content and verification
steps as agent_docs/imports.md, plus any helper scripts the skill ships…)
```

The relationship to your hand-rolled `agent_docs/` pattern is worth being precise about, since both will coexist for a while. The `agent_docs/` index is the *portable, tool-agnostic* form — plain Markdown and an explicit "read this file first" instruction that works in any CLI agent that can read files. Agent Skills are the *tool-native* form — the same idea with the relevance-matching and loading handled by the harness, plus the ability to bundle executable resources. Learn the hand-rolled pattern first because it teaches you what the mechanism *is* and works everywhere; reach for Skills when your tool supports them and you want the matching automated. Both are progressive disclosure. Neither changes the economics: index cheap and always present, detail expensive and on demand.

## Exercises (Apply → Create)

The Bloom arc here runs from applying the pattern to creating one from scratch. Do them in order.

**Exercise 5.1 — Convert a relocation table into an index (Apply).** Take the relocation table from Chapter 4 Exercise 4.3 (or build one now from any bloated instruction file). Turn every "move to a domain doc" entry into a Tier-1 index row, using the conditional-imperative form ("if the task touches X, read Y") and including the "do not guess when a doc exists" line and the "if none apply, proceed" line. *Deliverable:* the index block, ready to drop into a `CLAUDE.md`. *What this surfaces:* whether you can write a pointer that survives the "ignore if irrelevant" caveat.

**Exercise 5.2 — Write one real domain doc (Apply → Create).** Pick the single domain in a repo you know where an agent most often goes wrong from missing context. Write its `agent_docs/<domain>.md`: a "what it does," a "before you change anything" block of the non-obvious constraints, a "how to verify" block with the real commands, and a "pointers, not copies" section that references larger specs by path instead of inlining them. *Deliverable:* the committed doc. *What this surfaces:* whether you can capture hard-won knowledge as on-demand reference rather than as always-on rules — the core conversion of the chapter.

**Exercise 5.3 — Design the disclosure tree for a whole project, and test it (Create).** For a real repository, design the full two-tier system: the trimmed `CLAUDE.md` index plus the set of `agent_docs/` files (or `SKILL.md` skills if your tool supports them) it points to. Then *test the disclosure*: start a fresh session, give a task that touches exactly one domain, and confirm the agent reads the right doc and *only* that doc before acting — not zero (it explored instead) and not all of them (it loaded defensively). *Deliverable:* the file tree, plus a session log showing correct progressive loading and a one-paragraph diagnosis if it loaded wrong. *What this surfaces:* whether your index actually *triggers* the right disclosure under real conditions — the difference between a pattern that looks right on paper and one that controls behavior.

## Bridge

Progressive disclosure scales your knowledge without scaling your budget — for a project you can hold in your head. But the pattern was designed against an assumption the next chapter breaks: that *you* can author the index and the docs because you know where everything lives. In a monorepo of three thousand files and a dozen teams, nobody holds the whole map, the agent's first instinct is to explore, and exploration at that scale has a specific, measured cost — the **overexploration trap**, where an agent floods its own context with tens of thousands of tokens of nearly-relevant material and gets *worse*, not better. Chapter 6 takes progressive disclosure into the large codebase: directory-scoped instruction files, repo maps, and the rule that turns out to govern all of it — start at the narrowest plausible scope and expand only when an explicit, unresolved dependency forces you to.

> **AI Wayback Machine — John M. Carroll and the "minimal manual" (1980s, IBM).** The phrase *progressive disclosure* entered interface design through people like John M. Carroll, whose 1980s research at IBM's Watson Research Center on the "minimal manual" attacked a problem that should sound familiar: thick software manuals overwhelmed new users, who could not find the one thing they needed inside the everything they were handed. Carroll's radical proposal was to give people *less* — a slim manual that surfaced only what the immediate task required and pointed elsewhere for the rest — and users learned faster and made fewer errors than those given the comprehensive tome. The deeper insight, which interface designers from the Xerox and Apple lineages turned into the standing principle of *progressive disclosure*, was that information has a cost at the point of delivery: showing everything is not generous, it is expensive, because the recipient pays in attention to find the relevant fraction. That is exactly the cost Chapter 4 measured in instruction slots and this chapter spends an index to avoid. The agent reading a thousand-line `CLAUDE.md` is Carroll's overwhelmed novice with the thick manual; the agent reading a tiny index that points to one domain doc is the same novice handed the minimal manual. What Carroll demonstrated about human learners in the 1980s — that withholding the irrelevant is an act of design, not laziness — CLI-agent users are rediscovering as the economics of machine context. Disclose progressively, because attention is finite whether it is a person's or a model's. [High for the HCI principle; Medium on exact attribution of the term's origin]

---

## Sources

- Sweller, John. "Cognitive Load During Problem Solving: Effects on Learning." *Cognitive Science* 12, no. 2 (1988): 257–285 — the load argument for chunking detail behind an index.
- Carroll, John M. *The Nurnberg Funnel: Designing Minimalist Instruction for Practical Computer Skill.* MIT Press, 1990 — the minimal-manual research underlying progressive disclosure as an interface principle.
- Mayer, Richard E. *Multimedia Learning.* Cambridge University Press, 2001 — segmenting and signaling: deliver one layer at a time, then combine.
- Liu, Nelson F., et al. "Lost in the Middle: How Language Models Use Long Contexts." *Transactions of the ACL*, 2024 — why keeping the always-loaded layer small protects the rules that must be used.
- Anthropic. "Agent Skills" and `SKILL.md` documentation. Current online docs (accessed 2026-05) — the tool-level formalization of progressive disclosure (metadata always present, body loaded on demand). Treat as current-state; verify the front-matter fields before print. `[verify]`
- Anthropic. "Claude Code documentation" and "Effective context engineering for AI agents." 2025 / current docs — `CLAUDE.md` behavior and the prefer-pointers-over-copies practice; current-state.
- HumanLayer. "Writing a good CLAUDE.md." 2025 practitioner source — field guidance on splitting always-loaded context from on-demand docs. `[verify]` on numeric claims.
