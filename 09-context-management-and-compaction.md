# Chapter 9 — Context Management and Compaction

You are six hours into a refactor that is going well. The agent understands the module, the tests are green, and you are deep in the satisfying rhythm of describing a change and watching it land. Then, without much ceremony, the work gets worse. The agent re-reads a file it edited an hour ago as if seeing it fresh. It reintroduces a bug you already fixed together. It asks you a question you answered at the start. Nothing crashed. The model did not get dumber. The *window filled up*, and somewhere around the point where the context got crowded, the agent stopped being able to reliably use what was in it.

This is context rot, and it is the failure that only stateful agents have. A chat turn starts clean every time; a six-hour agent session carries everything — every file read, every tool result, every dead end — until the accumulated weight degrades the very reasoning it was supposed to support. The whole book turns on this: **context is the bottleneck, not intelligence.** This chapter is about managing the bottleneck directly. You will learn what `/compact` does to your window, how to tell it what to keep, why the PostCompact hook from Chapter 8 is a necessary repair rather than a nicety, and — the highest-leverage habit in the chapter — why you should restart before the window degrades rather than ride it down.

This is an *Apply* chapter: you are managing a real budget across a real session, not designing a new artifact from scratch.

## The window is a budget, and long contexts are not neutral

Start from the constraint. An agent's context window is finite, and every turn spends from it. This would be merely an accounting problem if a full window worked as well as an empty one — but it does not. The research on how language models use long contexts is unambiguous that *position and length matter*: Liu et al.'s "Lost in the Middle" showed that models retrieve and use information unevenly across a long context, with material in the middle of a large window used least reliably [High]. A long context is not a neutral container that holds your instructions safely until needed. It is a crowded space where salience competes, and where a rule stated early can be functionally invisible by the time it is relevant.

Before you can manage the budget, you have to be able to *read* it — to recognize context rot as it starts rather than after it has cost you an hour. The symptoms are consistent enough to name. The agent re-reads a file it already has in context, as though seeing it for the first time. It reintroduces a change you reverted together earlier in the session. It asks a question you already answered. It contradicts a constraint it was honoring an hour ago. Its summaries of its own recent work go slightly wrong. None of these is a model failure; each is the visible surface of a window that has gotten too crowded for the model to use reliably. Learning to notice the *first* of these — usually the re-read or the re-asked question — is the practical skill, because it is the early-warning signal that says: the budget is the problem now, do something about the budget. The user who keeps prompting harder at this point is treating a context problem as an intelligence problem, and it will not work.

The practitioner community puts a number on the practical ceiling — that quality starts degrading well before the hard limit, in the neighborhood of eighty percent of capacity. Treat that figure with care: **restart at roughly 80% rather than degrade is practitioner-sourced, not a measured constant** `[verify — practitioner-converged; no controlled benchmark located]`. But you do not need the exact number to act on the principle. The principle is that the last twenty percent of the window is the worst twenty percent to be working in, and the discipline is to manage the budget so you are rarely there.

## What compaction actually does

When the window approaches its limit, the agent compacts. `/compact` (and the automatic compaction that triggers near the ceiling) replaces the long, verbatim conversation with a *summary* of it, then continues from that summary — buying back room to keep working [Medium; Claude Code documentation, current-state — verify command behavior before print]. Think of it as the agent writing itself a handoff note and then forgetting the original conversation in favor of the note.

This is the right move, and it is a lossy one. Two things follow from "lossy" that you have to manage.

**Compaction is a summary, and summaries make judgment calls.** The agent decides what mattered enough to keep. It is often right, but it is not reliably right about *your* priorities — and the things it is most likely to drop are exactly the standing constraints that have not come up in a while. A rule like "never edit `vendor/`" that you stated at minute three and have not needed since can vanish from the summary precisely because nothing recent reinforced it. The post-compaction agent then operates without a rule it once held, and the failure looks like the agent "forgetting" — but the mechanism is a summarization choice you never saw.

**Compaction resets your sense of where you are.** After a compaction you are working from the agent's reconstruction of the session, not the session itself. Details that were implicit in the full transcript — a half-finished decision, a constraint you negotiated in passing — may or may not have survived. You should treat the moment after a compaction the way you would treat picking up someone else's half-finished work: verify the state before you trust it.

## Telling compaction what to preserve

You are not helpless about what the summary keeps. Two levers matter.

**Steer the compaction at the moment you trigger it.** A bare `/compact` lets the agent choose what to preserve. A directed compaction tells it [Medium; current-state]:

```
/compact Preserve exactly: the acceptance criteria for this refactor, the
list of files we have already changed, the two bugs we fixed (timezone
handling in session.py and the off-by-one in pagination), and the rule
that vendor/ is read-only. Drop the exploratory reads and the dead-end
attempts.
```

This converts compaction from a black box into a controllable handoff. You are doing the agent's judgment for it on the one decision that matters most — what survives.

![Sort diagram: a full transcript fed into a directed /compact, with acceptance criteria, the files-changed list, the bugs fixed, and the CRITICAL rules (in red) preserved as the interface, while exploratory reads, dead ends, and probe errors are dropped; a note flags that the preserve-the-rules instruction is itself in the summary and the PostCompact hook is the guarantee outside it.](images/09-context-management-and-compaction-fig-02.png)

*Figure 9.2 — A directed compaction preserves the session's conclusions and drops the exploration debris, but because the "preserve the CRITICAL rules" instruction is itself inside the summary, only a PostCompact hook guarantees those rules survive.*

**Put compaction instructions in CLAUDE.md.** Because compaction happens automatically when you are not watching, the most reliable place to state what must survive is your persistent instruction file. A short standing instruction — "When compacting, always preserve the current task's acceptance criteria, the list of files already modified, and all rules marked CRITICAL" — applies to every compaction in the project, including the automatic ones you never typed `/compact` for [Medium; current-state]. This is static context doing what static context is for: encoding a standing policy so you do not have to re-state it under pressure.

## The PostCompact hook, revisited as the necessary repair

Steering helps, but it cannot fully solve the problem, for a simple reason: the instruction that says "preserve the CRITICAL rules" is itself part of the context that gets summarized. You are asking the lossy process to reliably preserve the instruction about what to preserve. Sometimes it does; the failure mode is exactly when it does not.

This is why the **PostCompact hook** from Chapter 8 is not a convenience but the architectural fix. The hook fires *after* compaction completes and re-injects your non-negotiable rules into the freshly summarized context — outside the summary, after the lossy step, unconditionally [Medium; Claude Code documentation, current-state — verify hook name and availability before print]. Steering the summary is a request the lossy process may honor; the PostCompact hook is a guarantee the lossy process cannot touch, because it runs after the process is done.

The pairing is the lesson. Steer the compaction so the summary is good; back it with a PostCompact hook so the rules that matter survive even when the summary is bad. The first improves the common case; the second protects against the case that does the damage. Neither is "a better prompt." Both are engineering applied to a known lossy step in the context system.

## Restart at 80%, don't ride it to zero

Here is the habit that separates practitioners whose long sessions stay sharp from those whose sessions rot: they do not treat compaction as the only tool. Compaction keeps a *degrading* session alive. A restart gives you a *clean* session. These are not the same thing, and conflating them is how people end up doing their hardest work in the worst part of the window.

The discipline is to watch the budget and, as you approach the practical ceiling — call it roughly eighty percent `[verify — practitioner-sourced]` — make a deliberate choice rather than coasting into automatic compaction. The choice has a clean decision rule:

- **If the remaining work is small and self-contained,** let it ride or run a directed `/compact` and finish.
- **If the remaining work is substantial,** do not compact — *restart.* Save the state you need (an updated `PLAN.md`, a short progress note, the list of files changed — the durable-handoff artifacts of Chapter 10), `/clear`, and re-enter with a clean window holding only the conclusions.

The reason to prefer restart over compaction for substantial work is that compaction inherits the session's accumulated noise into the summary, while a restart from a written handoff inherits only what you deliberately wrote down. Compaction is the agent's lossy summary of everything; a handoff is your lossless summary of what matters. When the work ahead is large, you want the second.

Make the decision concrete. Return to the six-hour refactor from the opening. You hit the practical ceiling with two modules still to convert and the test harness still to update — clearly *substantial* remaining work. The wrong move is to keep going and let automatic compaction summarize six hours of exploratory reads into a lossy digest you then build the rest of the refactor on top of. The right move is to spend five minutes writing the state down: update `PLAN.md` with the two remaining modules and the exact verification command, jot a three-line note on the two bugs already fixed and the read-only paths, then `/clear` and re-enter. The new session opens holding only those conclusions — no dead ends, no half-relevant reads, no accumulated optimism — and does the hardest remaining work in the *cleanest* available window instead of the most degraded one. The five minutes you spend writing the handoff is the cheapest insurance in the chapter, and the instinct it has to overcome is the strong, specific reluctance to "throw away" a session that still feels productive. It is not productive; it is degrading. Writing it down and leaving is how you convert a rotting asset into a clean one.

![A horizontal context-window-fill bar from 0 to 100 percent with a clean-reasoning zone, a degradation zone in grey from 80 to 100 percent labeled with context-rot symptoms, and a red ~80% restart line; below it a two-branch decision rule: small/self-contained work means directed compact and finish, substantial work means save a handoff, clear, and restart.](images/09-context-management-and-compaction-fig-01.png)

*Figure 9.1 — Quality degrades well before the hard limit, so at roughly 80% fill you make a deliberate choice rather than coast into auto-compaction: compact and finish only small work, restart from a written handoff for substantial work.*

This connects directly to Chapter 7's plan-then-execute split and Chapter 8's commands. The same move — *when the context goes bad, leave and re-enter clean* — appears as a workflow split (Ch. 7), as the response to repeated failure (Ch. 7's two-correction rule), and here as proactive budget management. It is one discipline wearing three hats: do not try to fix a polluted window from inside it.

> **Compaction buys time; a restart buys a clean slate — know which you need**
>
> The tempting reading of compaction is "infinite context, automatically managed." It is not. Compaction is a lossy summary that trades fidelity for room, made by a process that does not reliably know your priorities. The skill is not getting better at compaction; it is knowing when compaction is the right tool (small remaining work, steer it, finish) and when it is the wrong one (large remaining work, restart from a written handoff instead). Riding a window to its limit and letting it auto-compact is the default — and the default is how good sessions quietly rot.

## Exercises (Apply)

1. **Watch a session degrade, then restart it.** Run a deliberately long session — keep feeding the agent related tasks until you see context rot (a re-read file, a reintroduced bug, a forgotten constraint). Note the approximate fill level when quality dropped. Then save a handoff, `/clear`, and re-enter. Compare the agent's first three turns before and after.

2. **Compare bare vs. directed compaction.** At a natural mid-session checkpoint, run a bare `/compact` and record what the summary kept. Reset, return to the same point, and run a *directed* `/compact` naming exactly what to preserve. Diff the two resulting states: what did the bare compaction drop that you needed?

3. **Add a compaction policy to CLAUDE.md.** Write a short standing instruction for your project naming what every compaction must preserve. Trigger a compaction and check whether the policy held. If it did not fully hold, that is the case the PostCompact hook exists for — note it.

4. **Write your restart decision rule.** For your own workflow, draft the one-line rule you will follow when you hit the practical ceiling: under what condition do you compact and finish, and under what condition do you save-and-restart? Make it specific enough that you would not have to think on the day it matters.

## Bridge to Chapter 10

You can now manage a single session's budget: steer compaction, guard it with a hook, and restart before the window rots. But notice what every restart depended on — a handoff you wrote *before* you cleared: the updated plan, the progress note, the list of files changed. That handoff is the seam between one session and the next, and so far you have been improvising it. The next chapter makes it deliberate. We treat the agent as fundamentally stateless across sessions and build the durable artifacts — session-notes files, a registry, a work log — that carry state across the gaps, so that "restart" never means "start over."

## The AI Wayback Machine: Barbara Liskov

> In the 1970s, **Barbara Liskov** was confronting a problem that looks, in hindsight, a great deal like context rot in software form: programs had grown too large for any one person to hold in their head, and changes in one part rippped through the whole because everything depended on the messy internal details of everything else. Her answer, developed in the CLU language and crystallized in the principle of *data abstraction*, was that a module should expose a small, stable interface and hide its implementation behind it. You should be able to use a module — and reason about it — by knowing what it *guarantees*, not by reading every line of how it works. The work won her the Turing Award, and the idea is now so basic to software that it is hard to see how radical it was: the claim that *hiding* information is what makes large systems tractable.
>
> Compaction is data abstraction applied to a conversation. The full transcript is the implementation — every read, every dead end, the messy how. The summary is the interface — a small, stable surface that says what the session *guarantees* without carrying how it got there. Liskov's insight tells you why a good compaction works (a clean interface lets you keep reasoning without the implementation underneath) and why a bad one fails (an interface that drops a load-bearing guarantee silently breaks every caller that relied on it — which, post-compaction, is the agent itself). The PostCompact hook is you re-asserting the part of the interface the summary must never drop. Liskov saw, fifty years early, the move that CLI-agent users make every time they steer a `/compact`: complexity becomes manageable only when you decide, deliberately, what to hide and what to keep visible at the boundary.

## Sources

- Liu, N. F., et al. (2024). Lost in the Middle: How Language Models Use Long Contexts. *Transactions of the Association for Computational Linguistics*, 12, 157–173. (Position and length affect how models use context.)
- Liskov, B., & Zilles, S. (1974). Programming with Abstract Data Types. *ACM SIGPLAN Notices*. (Data abstraction and the stable-interface principle.)
- Yao, S., et al. (2022/2023). ReAct: Synergizing Reasoning and Acting in Language Models. *arXiv:2210.03629*.
- Shinn, N., et al. (2023). Reflexion: Language Agents with Verbal Reinforcement Learning. *arXiv:2303.11366*. (Preserved state helps or hurts.)
- Anthropic Engineering. (2025). *Effective Context Engineering for AI Agents*. (Current vendor articulation of context engineering.) [Current-state.]
- Anthropic. *Claude Code documentation* (current online docs; accessed 2026-05). `/compact`, automatic compaction, compaction instructions in CLAUDE.md, PostCompact hook. [Current-state — verify command behavior and hook availability before print.]
- HumanLayer. (2025). *Writing a Good CLAUDE.md* (practitioner source). Source of the ~80%-capacity restart heuristic and instruction-capacity figures. [Numeric claims `[verify]` — practitioner, not benchmarked.]
- Sweller, J. (1988). Cognitive Load During Problem Solving. *Cognitive Science*, 12(2), 257–285.

## Tags

#cli-agents #context-management #compaction #context-rot #PostCompact #restart-at-80 #lost-in-the-middle #Liskov
