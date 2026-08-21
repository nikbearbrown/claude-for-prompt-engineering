# Chapter 4: The Instruction-Capacity Budget

## A scene: the rule that stopped working

A team has a `CLAUDE.md` they are proud of. It started clean — the kind of file Chapter 3 would endorse. Then it grew, the way good files do, one reasonable addition at a time. A rule about commit message format. A rule about not leaving `console.log` in. A rule about preferring composition over inheritance. A rule about which test directory mirrors which source directory. Each addition was defensible. Each landed, for a while.

Then something strange happened, and it is the symptom this entire chapter exists to explain. They added one more rule — a perfectly ordinary one, about wrapping new endpoints in an auth decorator — and the agent ignored it. That alone would not be alarming. What alarmed them was that the agent *also stopped obeying rules it had been following for weeks.* The commit format went sloppy. The `console.log` statements came back. It was as if adding the last rule had not buried one instruction but switched off the whole list.

They did the natural thing: they assumed the model had gotten worse, or that the new rule was badly worded, and they spent an afternoon rewording it. It did not help. Because the problem was not the wording of any one rule. **They had crossed a capacity threshold, and past that threshold an overloaded instruction set does not fail gracefully and locally — it fails globally.** The agent did not ignore the newest rule. It began ignoring *all* of them.

Chapter 3 told you to keep the file short and used soft words — "one screen," "small." This chapter replaces those words with numbers, a failure mode, and a budget. By the end you will be able to *estimate* how many instructions a given file is really asking the agent to hold, *diagnose* the global-ignoring failure when you see it, and *evaluate* a file against a practical ceiling rather than against your sense of thoroughness.

## Instructions are a budget, not a list

Start with the right mental model, because the wrong one is what gets teams into trouble. Most people treat an instruction file like a checklist: each line is an independent item, and adding one more costs nothing but a line of text. Under that model, a thorough file is a *better* file, and there is no reason to stop adding.

The correct model is a budget. The agent has a finite capacity to hold *followable instructions* — distinct constraints it can simultaneously track and apply while also reasoning about your actual task. Every rule you write spends from that capacity. So does every rule the agent is carrying from somewhere you did not write. When the spending exceeds the budget, the system does not throttle the least-important rule. The whole instruction-following behavior degrades.

This is a claim about *followable instructions*, not about tokens. A 50,000-token reference document the agent reads once and consults is not 50,000 instructions; it is data. But a list of "always do X / never do Y" directives is different — each is a standing constraint the model must check its every action against. It is that second kind, the imperative kind, that draws down the budget. The distinction matters for the rest of the book: Chapter 5's whole strategy is to convert standing imperatives (expensive) into on-demand reference (cheap), and you cannot make that move until you see *why* imperatives are the scarce resource.

## The numbers — and why they wear a flag

Now the figures. Read the flags as part of the sentence, not as footnotes; this is practitioner field evidence, not peer-reviewed measurement, and the book's discipline is to say so every time.

Frontier "thinking" models — the current generation that does extended reasoning — can reliably hold somewhere in the range of **~150–200 followable instructions** at once before instruction-following starts to break down. `[verify — practitioner-sourced; HumanLayer, "Writing a good CLAUDE.md," 2025]` That is the total budget, and it sounds generous until you learn what is already spent.

The agent's own system prompt — the standing instructions the *tool vendor* wrote to make the agent behave like a coding agent at all (how to use tools, when to ask permission, how to format output, the dozens of behavioral rules baked into the harness) — consumes a substantial fraction of the budget before your file is even loaded. The practitioner estimate is on the order of **~50 slots** already occupied by the system prompt. `[verify — practitioner-sourced]` You do not control these and you cannot reclaim them. They are rent.

Subtract the rent from the budget and you get the space available for *your* rules — plus whatever the conversation itself accumulates, because the running dialogue, the task description, and the tool results are all competing for the same finite attention. The headroom for a project instruction file is therefore much smaller than the headline 150–200 suggests.

Which is why the practical ceiling for a `CLAUDE.md`/`AGENTS.md` lands where field practitioners put it: roughly **~80–120 lines.** `[verify — practitioner-sourced]` That is not a vendor-enforced limit (those exist too, and are different — Chapter 13 catalogs the byte and character caps various tools impose). It is an *effectiveness* ceiling. Below it, your rules tend to land. Above it, you are gambling, and the stakes are not one rule but all of them.

> **A budget you can actually compute.** Total followable instructions a frontier thinking model holds reliably: **~150–200** `[verify]`. Already consumed by the agent's system prompt: **~50** `[verify]`. Remaining for your file *and* the live conversation: the difference, shared. Practical line ceiling for a persistent instruction file: **~80–120 lines** `[verify]`. Every figure here is practitioner-sourced field evidence (primarily HumanLayer, 2025), not controlled measurement — treat them as order-of-magnitude planning numbers, not constants, and re-check against current vendor guidance before you rely on them.

![A stacked budget bar: ~50 slots of system-prompt "rent," then your file and the conversation, against a ~150–200 ceiling; a second over-budget bar spills past the line into a red zone marked "uniform instruction-ignoring."](images/04-the-instruction-capacity-budget-fig-01.png)

*Figure 4.1 — The followable-instruction budget: the vendor's system prompt spends part before you arrive, and exceeding the ceiling collapses all rule-following at once, not just the last rule.*

A word on why these numbers must wear the flag, since the chapter's whole credibility turns on it. None of them comes from a controlled, published experiment. They come from practitioners who write these files for a living and report what stops working and roughly when. That is real evidence — it is the best evidence currently available — but it is field observation, and field observation drifts as models change. A model generation from now the budget may be 300, or the system-prompt rent may be 80. The *shape* of the claim is robust: there is a finite budget, the vendor spends part of it before you arrive, and exceeding the remainder degrades instruction-following globally. The *magnitudes* are estimates. Teach the shape with confidence; cite the magnitudes with the flag. A reader who later finds the number was 180 rather than 200 should still find the lesson intact.

## The failure mode that surprises everyone: uniform instruction-ignoring

Here is the part that makes this more than a "keep it short" reminder, and it is the diagnostic skill the chapter is really after.

When you exceed a normal resource limit, you expect graceful, *local* degradation: the newest or least-important item gets dropped, the rest survive. That is how a cache evicts, how a stack overflows the last frame, how a human forgets the bottom of a long list. Reasoning by analogy, the team in the opening scene expected the *new* auth-decorator rule to be the casualty.

That is not what happens. Past the threshold, the observed behavior is **uniform instruction-ignoring**: the model's adherence to the *entire* instruction set drops, not just the marginal rule. `[verify — practitioner-sourced]` Rules that worked for weeks stop working the moment the file tips over the edge. It is less like a list losing its last item and more like a signal degrading below the floor where any of it is legible.

Why would it behave this way? You do not need the mechanism to use the diagnostic, but a plausible account connects two threads you already have. From Chapter 3: long contexts are not neutral, and information competes for a finite, position-sensitive attention budget (Liu et al., "Lost in the Middle"). [High] When the instruction set grows past what the model can keep salient, no single rule reliably wins the competition for attention — so the failure is not "rule N lost" but "the instruction *region* lost coherence." The list stops functioning as a list. Whatever the true mechanism, the *practical* signature is what you must recognize: a sudden, broad collapse in rule-following that correlates with file growth rather than with any single rule's content.

This reframes debugging entirely. When an agent ignores a rule, the instinct is to fix *that rule* — reword it, bold it, add an exclamation point. But if you are over budget, rewording the offending rule is the worst possible move, because it does not reduce load and it costs you an afternoon. The correct first question is not "what is wrong with this rule?" It is **"how many followable instructions am I asking the agent to hold, and am I over budget?"** If you are, the fix is subtraction, not editing.

## Evaluating a file against the budget

Make this concrete with a worked evaluation. Here is a fragment of a real-feeling overloaded file. Your job, and the chapter's core skill, is to count and judge.

```markdown
# CLAUDE.md (excerpt — the back half of a 180-line file)

- Always use 2-space indentation.
- Prefer const over let; never use var.
- Use single quotes for strings except in JSON.
- Always destructure props in function components.
- Name boolean variables with is/has/should prefixes.
- Prefer early returns over nested conditionals.
- Always add JSDoc to exported functions.
- Wrap all new API endpoints in the @requireAuth decorator.   <-- the rule that "stopped working"
- Never commit commented-out code.
- Use the repository pattern for all DB access.
- ... (many more above) ...
```

Count followable instructions, not lines, but here they coincide: this excerpt alone is ten standing imperatives, and it is the *back half* of the file. Call the whole file roughly 100+ distinct directives. Add the ~50 the system prompt already spent. You are at or past 150 before the conversation contributes a single token. By the budget, this file is over the line, and the auth-decorator rule did not "stop working" because of anything about auth decorators — it was simply the straw, and its arrival pushed the set into uniform-ignoring territory, taking the indentation and quote-style and early-return rules down with it. [Medium]

Now evaluate *which* of these even belong, using Chapter 3's test. Indentation, quote style, `const`/`let`, JSDoc, early returns — these are **style**, and style does not belong in an instruction file at all; a formatter and linter enforce them deterministically and spend *zero* budget because the agent never has to hold them. Strike all of them. What remains — wrap endpoints in `@requireAuth`, use the repository pattern for DB access, never commit commented-out code — are genuine project constraints worth a few slots. The file should have been a handful of real rules plus "run `make lint`," and it would have sat comfortably under budget with every rule landing. Instead it spent its entire budget on directives a tool could have enforced for free, and then blamed the model.

That is the evaluation move: **count the followable instructions, subtract the rent, compare to the budget, and reclaim slots by moving style to tools and task-specifics to on-demand docs.** It is arithmetic, not taste.

## Why "just use a bigger model" does not help

A reader arriving from chat prompting will reach for the obvious escape: if the budget is the problem, wait for the next, larger model, which will surely hold more instructions. Sometimes the budget genuinely rises. But the reasoning misses the book's spine, so it is worth dismantling here.

First, the rent rises too. A more capable agent harness tends to ship a *larger* system prompt — more tools, more behaviors, more guardrails — so a chunk of any capacity gain is consumed before you see it. Second, and more fundamental: the failure was never that the model was too small to be smart. It was that you polluted its finite attention with directives that did not need to be there. A bigger budget spent the same wasteful way fails the same way, just later. The lever is not model size; it is what you choose to make the model hold. That is context engineering, and it is the whole point of the book. **Capacity is a constraint to design around, not a number to wait for.** [High]

## Exercises (Evaluate)

This chapter's Bloom level is Evaluate, so the exercises ask you to *judge* files against the budget and defend the judgment — not merely to shorten them.

**Exercise 4.1 — Estimate the budget consumption (Evaluate).** Take any real `CLAUDE.md`/`AGENTS.md` (yours, a teammate's, or an open-source project's). Count the *followable instructions* — distinct standing imperatives the agent must hold, not lines, not tokens. Then add the practitioner system-prompt estimate (~50). *Deliverable:* a number, the count broken down by category (style / project-constraint / task-specific / orientation), and a one-sentence verdict: over budget, comfortable, or borderline? *What this surfaces:* whether you can read a file as a budget rather than a checklist. Cite the ~150–200 and ~50 figures *with their `[verify]` flags* in your reasoning — using a flagged number honestly is part of the skill.

**Exercise 4.2 — Diagnose a degradation report (Evaluate).** You are handed this bug report: *"Our agent used to follow our commit-format and no-`console.log` rules reliably. Last week we added three new rules and now it ignores almost everything, including the old rules. We tried rewording the new rules — no change. The model version did not change."* Write the diagnosis. *Deliverable:* a paragraph naming the failure mode (uniform instruction-ignoring), explaining why rewording failed, and prescribing the fix — with the reasoning for *why* subtraction beats editing here. *What this surfaces:* whether you can recognize the global-ignoring signature from its symptoms rather than chasing the most recent rule.

**Exercise 4.3 — Cut a file under the ceiling and justify every cut (Evaluate→Apply).** Take an over-budget file and bring it under the ~80–120-line practical ceiling. For *every* line you remove, state where it goes — linter, on-demand doc (forward to Chapter 5), or deletion — and why that home is correct. The constraint is not "make it shorter"; it is "make every surviving line one that must be held, and relocate every line that need not be." *Deliverable:* the trimmed file plus a relocation table. *What this surfaces:* the judgment that distinguishes a slot worth spending from a slot wasted — the chapter's whole thesis, applied to your own repo.

## Bridge

You now have a budget, a ceiling, and a failure mode to watch for. But evaluation surfaced an awkward fact: a great deal of your relocation table pointed *somewhere else* — "move this to a domain doc," "this is task-specific, it shouldn't load every session." You kept deferring real, valuable knowledge to a place the chapter has not yet built. That place is the subject of Chapter 5. The trick that lets a project keep a hundred pages of hard-won detail while spending only a handful of instruction slots is **progressive disclosure**: a tiny always-loaded index that points to load-on-demand documents the agent reads only when the task calls for them. You have been writing the index entries already, in your relocation tables. Now you will build what they point to.

> **AI Wayback Machine — George A. Miller, "The Magical Number Seven, Plus or Minus Two" (1956).** Seventy years before anyone tripped a `CLAUDE.md` over its capacity threshold, the psychologist George Miller measured something that has never stopped being true: human working memory holds only a small, fixed number of items at once — famously, about seven, give or take two. Cross that limit and recall does not degrade gracefully; the whole span collapses. Miller's deeper move, the one that matters here, was *chunking* — you do not raise the limit by trying harder, you raise effective capacity by grouping many items into a few meaningful units, so the budget is spent on chunks rather than on raw items. That is exactly the move this book makes with instructions. You cannot make the model hold 200 rules by wording them better, any more than you can make a person hold 200 digits by concentrating. You hold a few high-value constraints, and you *chunk* the rest into on-demand documents and tool-enforced defaults so they no longer draw down the live budget. The number is different — the model's budget is larger and softer than Miller's seven — but the lesson is identical and it is old: capacity is finite, overload fails globally, and the engineering answer is not more effort but better chunking. What Miller saw in human cognition in 1956, CLI-agent users are rediscovering in machine context in 2026. `[verify the agent figures; Miller's result is High-confidence cognitive science]`

---

## Sources

- Miller, George A. "The Magical Number Seven, Plus or Minus Two: Some Limits on Our Capacity for Processing Information." *Psychological Review* 63, no. 2 (1956): 81–97.
- Liu, Nelson F., et al. "Lost in the Middle: How Language Models Use Long Contexts." *Transactions of the ACL*, 2024 — the position-sensitive, finite attention budget underlying a plausible mechanism for uniform instruction-ignoring.
- Sweller, John. "Cognitive Load During Problem Solving: Effects on Learning." *Cognitive Science* 12, no. 2 (1988): 257–285 — the foundational case that overloaded instruction sets exceed working-capacity limits and must be chunked.
- HumanLayer. "Writing a good CLAUDE.md." 2025 practitioner source — the primary origin of the ~150–200 followable-instructions figure, the ~50-slot system-prompt estimate, the ~80–120-line practical ceiling, and the uniform-instruction-ignoring observation. **All four figures are practitioner-sourced field evidence, marked `[verify]`, and must be re-checked against current vendor guidance before print.**
- Anthropic Engineering. "Effective context engineering for AI agents." 2025 — vendor articulation of context as the discipline governing reliable behavior; current practice, not timeless theory.
- Yang, John, et al. "SWE-agent: Agent-Computer Interfaces Enable Automated Software Engineering." *NeurIPS*, 2024 — interface and context, not raw capability, govern agent performance.
- Anthropic. "Claude Code documentation." Current online docs (accessed 2026-05) — vendor file-size caps (distinct from the effectiveness ceiling); treat as current-state, see Chapter 13.
