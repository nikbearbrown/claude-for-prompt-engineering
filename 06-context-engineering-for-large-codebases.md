# Chapter 6: Context Engineering for Large Codebases

## A scene: the agent that read too much

A senior engineer hands a CLI agent a task that should take twenty minutes: a partner's API changed a field name from `customerId` to `customer_id`, and the integration needs to follow. The repository is a monorepo — three thousand files, eleven services, a dozen teams, nobody holding the whole map in their head. The engineer types the task and goes to get coffee.

When she comes back, the agent is still working. It has read the integration module, fine. But then it read the shared HTTP client, then the auth layer the client uses, then the config system the auth layer reads, then a billing service that *also* talks to the same partner, then that billing service's tests, then a utility module that turned up in a grep, then the utility's tests. The context window is most full. Somewhere in there it lost the thread of the actual task. The patch it finally produces touches six files, two of which had nothing to do with the rename, and it has changed a `customerId` in the billing service that belonged to a *different* partner's API and should not have moved at all.

The agent was not stupid. It was *thorough*, in exactly the way that destroys performance at scale. This is the **overexploration trap**, and it is the central hazard of running an agent on a large codebase. The engineering teams that have measured it report numbers that should reframe how you think about "letting the agent figure it out." This chapter is about scoping context in a big repository so the agent stays on the twenty-minute task instead of touring the monorepo. The Bloom arc runs Apply to Analyze — you will use scoping techniques and analyze when an agent's context has gone wrong.

## The overexploration trap, with numbers

The intuition that more context is better dies here, and it dies with field evidence. The engineering team at Augment Code, building agents specifically for large codebases, reported a pattern that names the failure precisely: when an agent over-explores — pulling in broadly-related but task-irrelevant files — it can accumulate **80,000+ tokens of irrelevant context**, which correlated with roughly a **~25% drop in task completeness** and **doubled the time to completion**. `[verify — practitioner-sourced; Augment Code engineering]`

Read those three numbers as a single mechanism. The 80,000 tokens is the *cause*: a flood of nearly-relevant material that the model now has to reason over. The 25% completeness drop and the doubled time are the *effects*: with the signal of the actual task diluted by noise, the agent finishes less of the work and takes longer to do the part it finishes. This is the same finite-attention story you have met twice already — Chapter 3's middle-of-context blind spot, Chapter 4's instruction budget — now playing out at the scale of a whole repository's worth of files. The agent did not get worse at coding. Its context got worse, and on this book's thesis that is the same thing as the agent getting worse: **context is the bottleneck, not intelligence.** [High for the principle; `[verify]` on the specific figures]

![Two-path comparison of one rename task: a narrowest-first path reads the integration module and the one type definition then stops with a clean two-file patch, while a blind-expansion path pulls in client, auth, config, billing, tests, and utilities, accumulating over 80,000 tokens of irrelevant context with badges for ~25% completeness drop and ~2x time, ending in a corrupted patch.](images/06-context-engineering-for-large-codebases-fig-01.png)

*Figure 6.1 — The overexploration trap: blind expansion on a theory of relevance floods the window with 80k+ tokens of irrelevant context, correlating with a ~25% completeness drop and roughly doubled time (practitioner figures, [verify]).*

The flag on those numbers is the usual one and it is non-negotiable: they are practitioner field evidence from a vendor with a product in this space, not a controlled, independently-replicated experiment. The *direction* is robust and matches everything else in the book — irrelevant context degrades agents, and the degradation is non-trivial. The *magnitudes* — 80k, 25%, 2× — are estimates from one source's instrumentation and should be cited as such, and re-checked, before anyone leans on them in print.

The practical lesson the trap teaches is a rule you can apply on every large-codebase task:

> **Start at the narrowest plausible scope. Expand only when an explicit, unresolved dependency forces you to.** Do not let the agent (or yourself) pull a file into context on the theory that it *might* be relevant. Pull it in when the task has hit a concrete wall — a symbol that is defined elsewhere, a behavior that depends on code you have not yet seen — and not before. Relevance is proven by the task, not assumed in advance.

The billing-service detour in the opening scene violated this on every read after the second. Nothing in the rename task had hit an unresolved dependency on billing; the agent expanded on a *theory* of relevance ("billing talks to a partner too"), and the theory was wrong in a way that corrupted the patch. Narrowest-first scoping would have stopped at the integration module and the one type definition the rename actually touched.

## Don't explore blind — read the map first

The deeper fix is to make sure the agent never *has* to explore blindly, because a large codebase that is documented for agents replaces exploration with reading. The cheapest, most reliable context in a big repository is the context a human already wrote down: **ADRs, READMEs, and index files.**

Architecture Decision Records (ADRs) are short documents that record *why* a structural choice was made — why billing and integrations are separate services, why the import pipeline is idempotent, why a particular boundary exists. For an agent, an ADR is gold, because it supplies the WHY (Chapter 3) for a whole region of the codebase in a few hundred words the agent reads once, instead of a WHY it would otherwise try to *infer* from reading ten thousand lines of the code that resulted from the decision. A directory `README` that says "this service owns partner-facing webhooks; it talks to `billing/` only through the event bus, never directly" prevents the exact billing detour from the scene — the agent learns the boundary from one sentence rather than discovering (wrongly) by reading across it.

The principle generalizes: **prefer reading human-authored structure over machine exploration.** A grep that returns forty hits invites the overexploration trap; a README that says "the rename you want lives in `integrations/acme/` and nowhere else" closes the task. When you own a large codebase that agents will work in, writing these index files is not documentation overhead — it is context engineering, and it pays for itself the first time an agent skips the monorepo tour.

## Directory-scoped instruction files

Progressive disclosure (Chapter 5) had one assumption that strains at monorepo scale: a single `CLAUDE.md` at the repository root, authored by someone who knows the whole layout. In a big repo, no one person does, and the conventions differ by area — the Python data service and the TypeScript web app and the Go billing service have genuinely different rules.

The mechanism that fits is the **directory-scoped instruction file.** Most CLI agents load not just the root instruction file but also instruction files found in subdirectories, applying the more specific file's guidance when the agent is working in that part of the tree. [Medium; current-state] So the monorepo gets a thin root file with the universal rules and the top-level map, and each service gets its own scoped file with its own conventions:

```
repo/
  CLAUDE.md                     # universal: monorepo layout, the cross-cutting
                                #   rules, "work within one service unless told"
  services/
    billing/
      CLAUDE.md                 # Go conventions, the event-bus-only rule,
                                #   "never import integrations/ directly"
      agent_docs/payments.md
    integrations/
      CLAUDE.md                 # the partner-integration conventions, idempotency
      agent_docs/acme.md
    web/
      CLAUDE.md                 # TS/pnpm conventions, component patterns
```

Two payoffs. First, **ownership scales**: the billing team maintains `services/billing/CLAUDE.md` without coordinating with the web team, the same way they own their code. Second, **scope is enforced by structure**: a directory-scoped file is the natural place to write the boundary rule — `services/billing/CLAUDE.md` can say "this service is reached only through the event bus; do not let a task pull you into `integrations/`" — so the instruction that prevents the overexploration detour lives exactly where the agent is when it would be tempted. The root file stays under the Chapter 4 budget because most of the project-specific weight has moved down into the scoped files, loaded only when the agent is actually in that subtree. Directory scoping is progressive disclosure along the *filesystem*, not just along a doc index.

## The canonical answer: the repository map

For the general problem — *how does an agent orient in a codebase too large to read* — the most mature, field-proven answer comes from Aider, and it is worth studying because it is the technique the whole chapter has been circling: the **repository map** (repo map). [Medium; current-state]

Aider does not paste the codebase into context, and it does not let the model explore blind. Instead it builds a compact, structural summary of the entire repository — for each file, the key symbols it defines (classes, functions, signatures) and their relationships, ranked by relevance to the current task — and puts *that* in context. [Medium] The repo map is the codebase's table of contents: not the text of every chapter, but enough structure for the agent to know what exists, where it lives, and what to read next *on purpose.* It is pointers over copies (Chapter 5) applied to an entire repository, and it is the narrowest-first rule made automatic — the map shows the whole shape cheaply, and the agent expands into the actual source of only the files the task proves it needs.

Notice that the repo map resolves the chapter's tension directly. The overexploration trap comes from an agent that must reconstruct structure by reading source, accumulating 80k tokens of it in the process. The repo map *gives* the agent the structure up front, in a small fraction of those tokens, so it never has to reconstruct it by flooding context. A few thousand tokens of map replaces tens of thousands of tokens of exploratory reads — the exact trade the trap punishes you for not making. This is why a repository map, whether your tool builds one automatically (as Aider does) or you approximate one with READMEs and index files (as the previous sections described), is the canonical large-codebase context move. The lesson survives any specific tool: **give the agent a cheap, structural map first, and let the task — not a theory of relevance — drive every expansion into actual source.** [High for the principle]

![Resolution diagram: a 3,000-file repo too large to read is summarized into a compact repo map of per-file key symbols ranked by relevance (a few thousand tokens), from which the agent expands into the full source of only the one file the task proves it needs, in red, while everything else stays a pointer.](images/06-context-engineering-for-large-codebases-fig-02.png)

*Figure 6.2 — The repository map gives the agent the codebase's structure up front, so a few thousand tokens of map replace tens of thousands of exploratory reads and the agent expands into source only where the task forces it.*

## Analyzing a context that has gone wrong

The Analyze half of the chapter's Bloom level is a diagnostic skill: reading an agent's *context state* to determine whether overexploration has set in. The signals, in rough order of how early they appear:

1. **Reads with no edits.** The agent has read many files but proposed no change. Reading is cheap to start and expensive to accumulate; a long read-only streak on a large repo is the trap forming.
2. **Reads that cross a documented boundary.** The agent working on `integrations/` starts reading `billing/`. If no unresolved dependency forced the crossing, this is theory-of-relevance exploration, and it is the moment to intervene.
3. **Breadth without depth.** Many files touched shallowly rather than a few files understood. The rename task that ends up reading eight modules has lost its scope.
4. **The patch touches files the task never named.** The clearest retrospective signal: the diff includes changes outside the task's stated surface. The billing-service `customerId` that moved is this signal, and by then the damage is in the patch.

When you see these, the intervention is not "let it keep going, it will figure it out" — that is the bet the trap punishes. It is to stop, `/clear` if the context is already polluted (Chapter 9 makes this systematic), and restart with a *narrower* scope and an explicit pointer: "the rename is confined to `integrations/acme/`; do not read or modify any other service." You are doing by hand what the repo map and the boundary rules do automatically — proving relevance by the task and refusing expansion that the task has not forced.

## Exercises (Apply → Analyze)

The arc runs from applying scoping techniques to analyzing a degraded context.

**Exercise 6.1 — Scope a task narrowest-first (Apply).** On a large repo (yours, or a substantial open-source monorepo), take a real localized task — a rename, a single bug fix, a config change. *Before* running the agent, write the narrowest plausible scope: the exact files or directory the task should touch, and the one explicit dependency (if any) that would justify expanding. Then run the agent and compare what it actually read against your scope. *Deliverable:* your predicted scope, the agent's actual read list, and the delta. *What this surfaces:* whether you can predict the relevant surface — the skill that lets you set scope before the agent over-explores.

**Exercise 6.2 — Write the boundary into a directory-scoped file (Apply).** Pick two services/areas in a repo that should *not* reach into each other directly. Add a directory-scoped `CLAUDE.md` to one of them that states the boundary as a rule ("reached only via X; never import Y directly") with the WHY attached. Run a task in that directory that *tempts* the agent across the boundary and observe whether the scoped rule holds it back. *Deliverable:* the scoped file plus a session log. *What this surfaces:* whether a structurally-placed rule can prevent the overexploration detour at the point of temptation.

**Exercise 6.3 — Diagnose a polluted context from a transcript (Analyze).** Take a real session transcript where an agent produced a wrong or bloated result on a large repo (run one deliberately if you must — give a localized task with *no* scoping and let it wander). Annotate the transcript against the four overexploration signals: where the read-only streak began, where it crossed a boundary, where breadth replaced depth, where the patch exceeded the task's surface. Then state the single earliest point at which you should have intervened, and what you would have said. *Deliverable:* the annotated transcript and the intervention point. *What this surfaces:* whether you can *read a context* — diagnose the trap from its signals while it is forming, not only after the patch is wrong. This is the analysis the rest of the book's scaling chapters depend on.

## Bridge

You can now scope a single task in a large codebase: start narrow, read the map and the boundaries before exploring, expand only on a proven dependency, and recognize the overexploration trap as it forms. But every technique in this chapter assumed a *task* worth scoping — and so far you have been handing the agent tasks the way you hand a colleague a Slack message, informally. At scale that informality leaks: a task without an explicit stopping condition is an invitation to wander, and a task without acceptance criteria has no definition of done for the agent to aim at. Chapter 7 turns the task itself into an engineered artifact — explicit scope, explicit stopping conditions, acceptance criteria — and introduces the single highest-leverage workflow in the book: plan in one session, clear the context, and execute from the plan alone.

> **AI Wayback Machine — David Parnas, "On the Criteria To Be Used in Decomposing Systems into Modules" (1972).** Half a century before anyone watched an agent flood its context touring a monorepo, the computer scientist David Parnas asked the question that this chapter is really about: when you split a large system into parts, what should each part be allowed to *know* about the others? His answer founded modern software architecture — *information hiding*: a module should expose a small interface and hide its internals, so that the rest of the system can work against the interface without needing to know, or read, what lies behind it. The point was never tidiness. It was that a system whose parts must each understand all the others does not scale — every change forces every developer to reason about everything, and the cognitive load explodes. Now read the overexploration trap through Parnas. The agent that pulls billing into a rename task is a system component refusing to respect information hiding — it insists on reading internals across a boundary that exists precisely so nobody, human or model, has to. The directory-scoped instruction file that says "reached only via the event bus" is an interface contract written for an agent. The repo map is information hiding made navigable: it exposes the interfaces (the symbols, the structure) and hides the bodies until the task proves it needs one. What Parnas saw in 1972 — that scale is survivable only when each part can ignore most of the whole — is exactly what CLI-agent users rediscover the first time an unscoped agent tries to hold an entire monorepo in its head and gets *worse* for it. The boundaries that make codebases maintainable for humans turn out to be the same boundaries that make them legible to agents. [High]

---

## Sources

- Parnas, David L. "On the Criteria To Be Used in Decomposing Systems into Modules." *Communications of the ACM* 15, no. 12 (1972): 1053–1058.
- Yang, John, et al. "SWE-agent: Agent-Computer Interfaces Enable Automated Software Engineering." *NeurIPS*, 2024 — the peer-reviewed anchor that the interface through which an agent perceives a codebase governs its performance.
- Liu, Nelson F., et al. "Lost in the Middle: How Language Models Use Long Contexts." *Transactions of the ACL*, 2024 — the finite-attention basis for why irrelevant context degrades the agent, scaled to a whole repository.
- Augment Code. Engineering writing on context management for large-codebase agents, 2025 — the primary practitioner source for the overexploration figures (**~80,000+ tokens of irrelevant context, ~25% completeness drop, ~2× time to completion**). **All three figures are practitioner-sourced field evidence, marked `[verify]`, and must be re-checked against the source and ideally independent replication before print.**
- Aider (Paul Gauthier). Repository-map documentation and design notes. Current online docs (accessed 2026-05) — the canonical repo-map technique; treat as current-state.
- Anthropic. "Claude Code documentation." Current online docs (accessed 2026-05) — directory-scoped `CLAUDE.md` loading behavior; treat as current-state, see Chapter 13.
- HumanLayer. "Writing a good CLAUDE.md." 2025 practitioner source — field guidance on scoping context to the working area. `[verify]` on numeric claims.
