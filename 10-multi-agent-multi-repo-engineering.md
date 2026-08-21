# Chapter 10 — Multi-Agent and Multi-Repo Engineering

*Multi-agent multiplies the number of verification loops, not their nature — and adds one new loop nobody can close alone: the merge*

There is a mistake that looks like ambition. An engineer reads about multi-agent systems and immediately reaches for one. The task is a bug fix: the request parser, the validation layer, one test file. They spin up an orchestrator and four workers — one per concern — expecting the architecture to make things faster. What actually happens is that the orchestrator spends tokens decomposing a task that didn't need decomposing. The four workers each load overlapping context about the same small change. Their edits are *coupled* — the validator's fix depends on what the parser now returns — so the workers either step on each other or sit idle waiting. The merge surfaces a semantic conflict no individual worker could see. The whole thing costs several times what a single agent would have, takes longer, and produces a worse diff.

A single agent in a single session — read the three files, make the coupled change, run the test — would have closed the loop faster and cheaper.

![Three stacked decision gates test in order whether a task is parallelizable, breadth-first, and unpredictable in decomposition. Any No drops the task to a single-agent outcome. Only passing all three reaches the more costly multi-agent outcome.](../images/10-multi-agent-multi-repo-engineering-fig-02.png)
![Single-agent vs. multi-agent: a task routes to multi-agent only after passing all three gates; any failure drops to one strong agent.](images/10-multi-agent-multi-repo-engineering-fig-02.png)
*Figure 10.2 — Single-agent vs. multi-agent: a task routes to multi-agent only after passing all three gates; any failure drops to one strong agent.*

This is the thing to establish before anything else, because the field's loudest signal is "more agents, better results," and the loudest signals are the ones that need the most careful reading. Anthropic's own guidance leads with the correction: **use the simplest thing that works.** "Building Effective Agents" (Anthropic, December 2024) catalogs the workflow patterns and is explicit that you add agents only when the task's subtasks *can't be predicted in advance.* If you can predict the subtasks — parse, validate, test, and you knew that before starting — you don't need an orchestrator to discover them. You need a single agent to do them in the order they're coupled.

The decision rule that falls out: multi-agent is a deliberate, cost-justified choice for tasks that are parallelizable, breadth-first, and unpredictable in their decomposition. A small, sequentially-coupled change is none of those. Establish multi-agent as a spend decision, not a default, and the rest of the chapter is about when the spend pays.

---

## The orchestrator–workers pattern

When the task *is* big and parallelizable, the pattern is orchestrator–workers — and it is, when you see it clearly, Chapter 9's sequential migration loop run concurrently.

![One orchestrator fans out to three workers, each isolated in its own worktree closing its own passing test loop. The workers fan in to a merge node gated by an integration loop no worker can close, which leads down to a human gate.](../images/10-multi-agent-multi-repo-engineering-fig-01.png)
![Orchestrator-workers with worktree isolation: workers fan out into isolated worktrees, fan in to a merge loop no worker can close, then a human gate.](images/10-multi-agent-multi-repo-engineering-fig-01.png)
*Figure 10.1 — Orchestrator-workers with worktree isolation: workers fan out into isolated worktrees, fan in to a merge loop no worker can close, then a human gate.*

In the orchestrator–workers pattern, a central model dynamically breaks down a task, delegates subtasks to workers, and synthesizes their results. The motivating example is coding: the number of files that need changing depends on the task, so you can't hardwire the decomposition — the orchestrator discovers it. Map this onto the codebase-wide migration from Chapter 9: the orchestrator plays Rosie's role, defining the global change and splitting it into shards, and where Chapter 9 processed shards one after another, here the workers run in parallel.

But concurrency forces a problem sequential processing never had. Two agents editing the same working directory clobber each other. Worker A writes a file; worker B, mid-edit on the same file, overwrites A's change or reads a half-written state. This is a classic resource-contention race — two independent actors sharing a resource, corrupting it. Two agents sharing one checkout is a race condition, and the fix is mutual exclusion.

The mutual-exclusion mechanism for coding agents is the **git worktree**. A worktree is a separate working directory on its own branch that shares the repository's history and remote. Worker A operates in `../task-parser/` on branch `fix/parser`; worker B operates in `../task-validator/` on branch `fix/validator`; their edits never touch the same files, because they're literally different directories. When each worker's branch is verified, the merge is a normal git operation. The isolation is the one non-negotiable. Point two agents at the same directory and watch them clobber each other's edits; give each its own worktree and the same two agents merge cleanly. Isolation is the safety property that makes parallelism safe — without it, multi-agent coding is a corrupted-state generator.

<!-- → [FIGURE: Fan-out / fan-in diagram — orchestrator at the top decomposes into N workers, each in its own labeled worktree/branch box with a small green checkmark (per-worker test loop); arrows reconverge at a merge node with an integration CI gate; final arrow leads to a human gate. Caption: The multi-agent verification structure. Per-worker loops are the old loops, replicated. The merge loop is new. The human gate sits on the synthesis.] -->

Now the structure of the verification loops, which is the chapter's central insight. Each worker has its own verification loop — its shard's tests, exactly as in Chapter 9. Those loops are independent precisely because the worktrees are isolated. But there is a loop no worker can close: **the merge.** Worker A's branch passes its tests. Worker B's branch passes its tests. The merged result can still fail, because A's change and B's change interact in a way neither worker could see from inside its own worktree. The orchestrator — or CI running on the merged branch — owns this integration verification loop, and it is genuinely new. It did not exist in single-agent work, and it cannot be delegated to any individual worker, because no worker holds the whole.

The mental model is fan-out / fan-in: orchestrator → N workers in isolated worktrees → merge node gated by integration CI → human gate on the synthesized result. The per-worker loops are the old loops, replicated. The merge loop is the new one. The human gate (Chapter 8) sits where it always sits — on the irreversible, judgment-laden step — which here is the synthesis, not each worker's individual diff.

---

## What the multi-agent numbers actually say

Multi-agent costs more, and the economics shape both whether you do it and how you structure it. The most actionable single fact in the literature: multi-agent systems burn on the order of approximately fifteen times the tokens of a chat interaction, from Anthropic's description of how they built their multi-agent research system (June 2025). That is the spend you are committing to. It means multi-agent only pays when the task's value justifies it — high-value, parallelizable, breadth-first work — and never as a reflex for a task a single loop closes.

![A two-tier hierarchy. One large orchestrator node, the strong and expensive model invoked once, sits above a row of four small uniform worker nodes, the cheaper and faster models invoked many times. Size contrast encodes the cost and invocation-frequency split.](../images/10-multi-agent-multi-repo-engineering-fig-03.png)
![Model tiering by role: one expensive orchestrator invoked once sits above many cheap workers invoked many times.](images/10-multi-agent-multi-repo-engineering-fig-03.png)
*Figure 10.3 — Model tiering by role: one expensive orchestrator invoked once sits above many cheap workers invoked many times.*

The same source reports that token usage alone explains roughly 80% of performance variance in their system, with tool-call count and model choice as the remaining factors. Read carefully, that is a statement about *spend*, and it motivates the standard economic structure: **model tiering by role.** Put the strong, expensive model where the hard thinking is — the orchestrator, which plans the decomposition and synthesizes the results — and cheaper, faster models on the workers, which execute bounded subtasks. Anthropic's research system used a Claude Opus 4 "LeadResearcher" spawning three to five Claude Sonnet 4 subagents, each with its own context window and tools. The role split is economically motivated: you pay for the orchestrator's judgment once and the workers' execution many times, so you want the expensive model in the role you invoke least.

<!-- → [TABLE: Role-based model tiering — columns: Role, Model tier, Job; rows: Orchestrator (Opus-tier, "decompose task / sequence workers / own merge loop"), Worker (Sonnet-tier, "execute bounded subtask in isolated worktree / close own test loop"). Caption: Economic rationale for tiering: orchestrator judgment is a one-time cost per task; worker execution scales with the number of shards.] -->

Now the critical caveat, and it is the one this chapter exists to insist on. The headline numbers — approximately 90.2% uplift over single-agent, fifteen times the token cost, 80% of variance from tokens — come from a **research system**, evaluated on a research and search task: breadth-first, parallel information gathering. They are not coding results. A research task is almost ideally suited to multi-agent: it decomposes into independent searches whose results compose by union, and parallel subagents genuinely cover more ground. Coding is frequently the opposite — sequentially coupled, where subtask B depends on the output of subtask A, and where parallel workers from the same base model may share blind spots and produce *correlated* errors rather than decorrelated coverage.

The honest claim is the open one: whether multi-agent beats a single strong agent for coding specifically is not established. There is no clean controlled comparison showing the research-system uplift transfers to a coding benchmark. The coordination overhead that a breadth-first research task amortizes, a tightly-coupled coding task may pay without return. Use the fifteen-times figure as a cost anchor — it is the most portable fact. Treat the 90.2% uplift as a domain-specific snapshot that says nothing reliable about your refactor.

One misconception to close here: "Anthropic's multi-agent system beat single-agent by 90%, so multi-agent coding is better." Two errors in one sentence. The 90.2% is a research-eval number, and the transfer to coding is unestablished. Coding's sequential coupling and correlated-error risk can erase or invert the advantage. Multi-agent for coding is an open empirical question, not a settled win — and the only thing the research numbers reliably tell you is that it will cost roughly an order of magnitude more.

---

## Cross-repo state: the under-published frontier

So far the workers shared one repository — concurrent shards of a monorepo change, isolated by worktree. The harder, less-charted case is when the units live in separate repositories with independent versioning, and there is no settled method here. That is itself worth saying plainly.

![A shared library sits above a left-to-right dependency chain of three repos. A horizontal compatibility-window band shows two repos already migrated to the new version and one still on the old version. Ordered dependency arrows mark the topological merge order; there is no atomic multi-repo merge, so every intermediate state must stay valid.](../images/10-multi-agent-multi-repo-engineering-fig-04.png)
![Cross-repo sequencing under a compatibility window: repos bump in dependency order while old and new library versions coexist, with no atomic multi-repo merge.](images/10-multi-agent-multi-repo-engineering-fig-04.png)
*Figure 10.4 — Cross-repo sequencing under a compatibility window: repos bump in dependency order while old and new library versions coexist, with no atomic multi-repo merge.*

The canonical case is a breaking library version that N downstream services — each its own repo, its own CI, its own release cadence — must bump and adapt within a compatibility window. Each repo is a shard with its own CI as local ground truth, and one agent per repo can bump and fix against that CI. But coordinating the fleet introduces problems a monorepo never had.

**Dependency ordering.** If service C depends on service B which depends on the library, you cannot merge them in arbitrary order without breaking the intermediate states. The orchestrator must sequence the merges to respect the dependency graph — a topological sort over repos, not a free-for-all.

**Compatibility windows.** During a staged rollout, both the old and new library versions must coexist for a period. Some repos run the old API while others run the new. The migration is not atomic across the fleet; it is a window during which the system is heterogeneous and must stay correct.

**No quasi-atomic multi-repo merge.** A monorepo gives you one commit that lands everything together. Across repos there is no equivalent — merges land independently, and the system passes through intermediate states that must each be valid.

<!-- → [FIGURE: Dependency graph diagram — nodes are services (Library, Service B, Service C) with directed dependency arrows; overlaid timeline showing staged migration order (Library merges first, then Service B, then Service C); a shaded "compatibility window" band where both old and new library APIs coexist. Caption: Cross-repo dependency ordering. Merges must respect the topological sort; the compatibility window is the period during which the system is heterogeneous and both API versions must be valid.] -->

This is Lamport's territory: independent processes that must agree on a consistent global state need an explicit protocol — ordering, consensus — and the orchestrator coordinating a multi-repo migration is solving a distributed-coordination problem at the level of git branches across repos. The consistent state the repos must converge on is every service on a compatible version of the library; the orchestrator's sequencing is the protocol that gets them there without passing through a broken intermediate state.

The honest status: cross-repo coordination is largely bespoke per organization. There is little rigorous public method. What teams do is hand-rolled. This is a genuine research gap, not a solved problem with a tool you can install.

The practical posture: treat each repo as an isolated worker with its own CI loop — you know how to do this, it is Chapter 9. Treat the *sequencing* — dependency order, compatibility windows, staged rollout — as a coordination problem you design explicitly per fleet, with the orchestrator or a human release manager holding the protocol. The per-repo loop is solid. The cross-repo protocol is the part you build by hand and the part the field hasn't standardized.

---

## What this adds up to

Multi-agent and multi-repo work scaled the verification loop without changing its nature. Each worker closes its own loop in an isolated worktree. The orchestrator closes the merge loop no worker can. The human gate sits on the synthesis. The non-negotiable is isolation — one worktree per worker. The open question is whether any of this beats a single strong agent for coding. And the cost anchor — roughly fifteen times the tokens — is the one figure that travels.

Notice the assumption underneath every claim in these last two chapters. "Never advance a red shard." "Gate the merge on integration CI." "Multi-agent's uplift is open." Every one of those rests on *measurement* — on knowing how reliably your agent or workflow actually succeeds. And we have been quoting numbers throughout: SWE-bench resolution rates, the 90.2% uplift, the migration percentages, all flagged as dated snapshots. Chapter 11 turns the book's thesis on the measurement itself. To know how good your agent is, you must build a verification loop around the measurement — and the distinction that organizes it is the one between *can it succeed* and *does it succeed every time.* That gap — pass@k versus pass^k — is where we go next, and it is the difference between a leaderboard number and a number you can ship on.

---

## What would change my mind

The chapter's load-bearing caution is that the headline multi-agent uplift is a research-eval result that does not establish a coding advantage, and that for sequentially-coupled coding tasks a single strong agent may match or beat a multi-agent fleet. A controlled comparison — same model family, same compute budget, a representative set of real coding tasks, a defensible correctness measure — showing that orchestrator–workers reliably outperforms a single strong agent on coding with the advantage surviving coordination and correlated-error costs, would overturn this chapter's "open question" framing and make multi-agent a recommended default for parallelizable coding rather than a cost-justified exception. I'd want it to control for the obvious confound: that multi-agent's apparent wins often come from more total compute, not from coordination. Show the uplift at matched token budget, and show error decorrelation — workers making genuinely different mistakes — rather than redundant agreement.

Separately, a settled, tool-supported method for quasi-atomic multi-repo coordination — dependency ordering, compatibility windows, and rollback as a standard primitive rather than a bespoke per-organization script — would move §10.4 from frontier to solved and change what I'd teach there.

---

## Still puzzling

Does multi-agent beat single-agent for coding, at matched compute? The strongest published uplift is a research and search task. Coding's sequential coupling and correlated-error risk may erase or invert it, and most apparent multi-agent wins may just be more total tokens. No clean controlled comparison at matched budget exists.

Does multi-agent add genuine error decorrelation, or redundant agreement? Parallel workers from one base model may share blind spots — agreeing confidently on the same wrong answer. Whether multi-agent buys real diversity or just expensive consensus is open, and it determines whether the pattern adds reliability or just cost.

What is the optimal worker count? The practitioner "two to four parallel sessions" ceiling tracks human review bandwidth and rate limits, not a measured agent-capability optimum. Whether there is a true coordination-cost curve with a sweet spot, and where it sits, is not characterized.

How do you verify a *semantic* merge conflict? Integration CI catches what the merged tests cover; inter-worker semantic conflicts — two correct-in-isolation changes that interact wrongly in a path no test exercises — are the migration-style residue at concurrency scale. The agentic discipline for resolving those is immature, and it is the merge loop's hard case.

---

## LLM Exercises

**10.1 (Analyze).** You are given four tasks (in the course repo, `exercises/ch10/`): (A) a two-line bug fix coupling the parser and validator; (B) a codebase-wide rename across nine independent modules; (C) a breadth-first investigation mapping an unfamiliar codebase before a design; (D) a breaking library version bumped across six downstream repos. For each, decide single-agent vs. multi-agent and justify it from §10.1's three criteria (parallelizable? breadth-first? decomposition unpredictable?). For the multi-agent ones, name the isolation primitive, the per-worker loop, and the loop the orchestrator must own that no worker can.

**10.2 (Apply).** Take a real codebase-scale change (or the course fork) that decomposes into at least three independent modules. Set up one git worktree per module, run a worker per worktree, and merge. Keep a log: each worker's branch, what its verification loop checked, and — critically — whether the *merge* surfaced a failure no individual worker's tests caught. Then deliberately point two workers at the *same* directory (no worktree) and report what breaks. Append a paragraph on what the merge loop caught that the per-worker loops couldn't.

**10.3 (Create — produce something).** Design and run a multi-repo dependency bump. Produce a written coordination plan for bumping a shared library across three (real or simulated) downstream repos: the dependency order, the compatibility window, the per-repo verification loop (each repo's CI), and the merge sequencing protocol the orchestrator follows. Identify at least one intermediate state during rollout and confirm it is valid (both library versions coexisting). Then execute the sequence (or simulate it) and append a note on where your hand-rolled protocol was fragile — the part no tool gave you.

**10.4 (Evaluate, optional).** Take the same parallelizable coding task and run it two ways: (a) a single strong agent in one session; (b) an orchestrator with workers in worktrees. Measure wall-clock time, token cost (estimate the multiple), and correctness (regressions, merge conflicts, semantic conflicts). Report whether multi-agent paid for its overhead on *this* task, at what task size you'd expect the answer to flip, and whether you observed correlated errors — the same mistake from multiple workers sharing a base model.

---

## References

- Anthropic (2024). *Building Effective Agents.* anthropic.com/research/building-effective-agents. (The workflow-pattern taxonomy — prompt chaining, routing, parallelization, orchestrator–workers, evaluator–optimizer — and the restraint principle: use the simplest pattern that works; add agents only when subtasks can't be predicted in advance.)
- Anthropic Engineering (2025). *How we built our multi-agent research system.* anthropic.com/engineering/multi-agent-research-system. (~90.2% uplift, ~15× token cost, ~80%-of-variance-from-tokens. **These are research-eval figures, NOT coding results** — do not relabel them; verify against the post.)
- Yang, J., Jimenez, C. E., Wettig, A., Lieret, K., Yao, S., Narasimhan, K., Press, O. (2024). *SWE-agent: Agent-Computer Interfaces Enable Automated Software Engineering.* NeurIPS 2024. arXiv:2405.15793. (The agent-computer interface as a first-class design variable; multi-agent coding inherits the ACI problem per worker — worktree isolation is an ACI-level decision about what filesystem each worker sees.)
- Claude Code documentation. *Worktrees* — `/worktrees`, `--worktree`, per-subagent `isolation: worktree`. code.claude.com/docs/en/worktrees. (Git-worktree-per-worker isolation; the "2–4 parallel sessions" ceiling is a practitioner heuristic. Exact flag/frontmatter spelling dates fast — verify against current docs.)
- Lamport, L. (1978). *Time, Clocks, and the Ordering of Events in a Distributed System.* (Independent processes converging on a consistent global state need an explicit protocol — the cross-repo coordination framing; interpretive lineage, not a claim about coding agents.)
- Dijkstra, E. W. (1968). *Cooperating Sequential Processes.* (Mutual exclusion for concurrent processes sharing a resource — worktree isolation as the resolution of a two-agents-one-directory race; interpretive lineage.)
- *Companion volume:* Brown, N. B. *Prompt Engineering with CLIs* — multi-agent context-system discipline and the operator-plus-workers pattern at the context level.
