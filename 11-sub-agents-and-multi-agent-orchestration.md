# Chapter 11 — Sub-Agents and Multi-Agent Orchestration

You are three hours into a migration. The task is to move forty-one service modules from one logging library to another — mechanical, repetitive, but spread across a sprawling repository whose conventions you half-remember. You started in a single Claude Code session, and for the first dozen files it went beautifully. Then the agent began to drift. It re-read files it had already edited. It "rediscovered" the project's import style for the fourth time and got it slightly wrong. By module thirty the session was carrying so much accumulated history — every diff, every test run, every dead end — that the model was reasoning against a context window two-thirds full of its own past. The work did not fail loudly. It degraded.

You already know the diagnosis from earlier chapters: context is the bottleneck, not intelligence. The same frontier model that fumbled module thirty would have handled it flawlessly in a fresh session. What you need is not a smarter agent. You need *more than one window* — a way to spread the work across several clean contexts and recombine the results without any single agent ever having to hold the whole job in its head.

That is what this chapter is about. The capability you are building is concrete: **split a task that exceeds one context window across coordinated agents, and merge their work back into a single trustworthy result.** Multi-agent orchestration is not a way to make the model think harder. It is a way to keep each unit of thinking inside the window where the model is reliable.

## When One Window Is Not Enough

There are exactly two reasons to reach for more than one agent, and both are about context, not capability.

The first is **size**. The task touches more code, or requires reading more files, than fits comfortably in one window before quality decays. A repository-wide refactor, an audit of every API endpoint, a survey of how a deprecated function is called across two hundred files — none of these fit in a single reliable session. You are not subdividing because the work is hard. You are subdividing because the *reading* is large.

The second is **parallelism**. The subtasks are genuinely independent, and you would rather not wait for them to run one after another. If module A and module B have nothing to do with each other, there is no reason a single agent should plod through both in sequence, accumulating the residue of A while it works on B.

Notice what is *not* on this list. "The task is complicated" is not a reason to add agents — a complicated task that fits in one window is best done in one window, where the agent can see all the relevant pieces at once. Splitting work across agents has a real cost: every boundary you draw is a place where context must be summarized, handed off, and reassembled, and every summary loses something. The rule that should govern every decision in this chapter is: **add an agent only when a single context window cannot hold the work reliably.** [High]

This is the natural extension of the parallel-versus-sequential rule from Chapter 10. There you learned to run two tasks sequentially only when B genuinely needs A's output files, and in parallel otherwise. Multi-agent orchestration is that rule applied at the level of whole context windows rather than whole sessions.

## The Operator-and-Workers Pattern

The dominant architecture for this is simple enough to draw on a napkin: one **operator** (also called a coordinator or orchestrator) and several **workers**.

The operator owns the task as a whole. It decomposes the job into subtasks, dispatches each to a worker, and assembles the results. The workers each receive one scoped subtask, do it in a fresh context, and return a result. Crucially — and this is the entire point — **the operator holds summaries, not full context.** [High]

Consider the logging migration. An operator-and-workers version looks like this:

- The **operator** reads the project's logging conventions once, produces a one-page migration spec (old call shape → new call shape, import changes, edge cases), and partitions the forty-one modules into, say, six groups.
- Each **worker** receives the spec plus one group of modules. It edits those files, runs the relevant tests, and returns a short report: which files it changed, which tests passed, anything that did not fit the spec.
- The **operator** collects the six reports. It never re-reads the 41 modified files in full. It reads six summaries, notices that worker four flagged an edge case the spec missed, and dispatches a follow-up.

The operator's context window contains the spec and six paragraph-length reports — a few thousand tokens. It never approaches the degradation threshold, because it never holds the raw material. Each worker's window contains one spec and a handful of files — well inside the reliable zone. No single agent ever carries the whole job.

![An operator holding the spec and short worker reports at top, dispatching scoped groups to workers in isolated git worktrees below, each returning a paragraph-length report that the operator reasons over before merging reviewed branches](images/11-sub-agents-and-multi-agent-orchestration-fig-01.png)

*Figure 11.1 — The operator reasons over the spec and short reports — never the full diffs — so it stays below the degradation threshold while workers do the deep reads in clean worktree contexts.*

The failure you are designing against is the operator that tries to *supervise by re-reading*. If the coordinator pulls every worker's full diff back into its own context to "check the work," you have rebuilt the single-bloated-session problem with extra steps. The coordinator's job is to reason over summaries and acceptance signals (tests passed, files in the allowed set, spec honored), and to escalate to a full read only on a specific, flagged exception. Summarize aggressively; read fully only on cause.

> **Worked sketch — operator prompt (excerpt).** The operator is itself an agent, driven by a task prompt:
>
> ```
> You are coordinating a logging-library migration across the modules
> listed in migration-targets.txt. Do NOT edit code yourself.
>
> 1. Read docs/logging-conventions.md and produce migration-spec.md:
>    old call shape, new call shape, import changes, known edge cases.
> 2. Partition the targets into 6 groups in migration-plan.md.
> 3. For each group, dispatch a worker with the spec and that group only.
> 4. Collect each worker's report (files changed, tests run, exceptions).
>    Do not re-read changed files unless a worker flags an exception.
> 5. Write migration-summary.md: per-group status, open exceptions,
>    recommended follow-ups. Stop. Do not start follow-ups without review.
> ```
>
> The stopping condition ("Stop. Do not start follow-ups without review") is doing real work: it keeps the operator from spiraling into autonomous fan-out you did not authorize.

## Split-and-Merge

The operator-and-workers pattern is one instance of a more general move: **split-and-merge**. You take a task, cut it along a clean seam into independent pieces, run the pieces in separate contexts, and stitch the outputs back together.

The art is entirely in choosing the seam. A good seam produces pieces that are genuinely independent — a worker can do its piece without needing to know what any other worker is doing. A bad seam produces pieces that secretly depend on each other, so the merge step becomes a second, harder task in which you reconcile decisions the workers made in isolation without coordinating.

![A clean seam where two workers on disjoint files merge without conflict, beside a leaky seam where two workers both call and rename one shared function in isolation, producing a merge conflict](images/11-sub-agents-and-multi-agent-orchestration-fig-02.png)

*Figure 11.2 — A clean seam is a clean interface; a hidden shared dependency leaking across the cut is a Liskov violation that surfaces as a merge conflict.*

Some seams are clean by nature:

- **By file or directory.** "Each worker handles one service module" — clean if the modules do not share state the change touches.
- **By concern.** One worker updates tests, another updates docs, another updates the implementation — clean only if the interface between them is fixed in advance (which is exactly what a shared spec does).
- **By phase.** One agent researches and produces a plan; a *separate* agent executes the plan in a fresh context. This is the plan-then-execute split from Chapter 7, now seen as a two-agent pipeline: the planning context and the execution context never contaminate each other.

Some seams only look clean. "Split the refactor by feature" sounds independent until two features both call the function you are changing, and your two workers rename it two different ways. When you cannot find a clean seam, that is a signal to keep the work in one window, or to add an explicit coordination artifact — the shared spec, a fixed interface, a frozen API — that *makes* the pieces independent before you split them.

The merge step deserves a named owner. Either the operator merges (reading summaries and resolving flagged conflicts), or you merge by hand. What you must not do is assume the pieces will simply compose because each one passed its own tests. Independent correctness does not imply joint correctness; that is the whole reason the seam mattered.

## Git Worktrees: Clean Parallel Contexts on Disk

Parallel agents that all edit the same working directory will collide. Worker A's half-finished edit to a shared file becomes worker B's confusing input; two agents stage conflicting changes; one agent's test run picks up another's uncommitted mess. You need each parallel worker to operate on its own copy of the repository — but cloning the whole repo six times is wasteful and slow.

Git's answer is the **worktree**. A single repository can have multiple working trees checked out at once, each on its own branch, sharing one underlying object store. `git worktree add ../migration-group-3 -b migration/group-3` gives you a second directory, on its own branch, that shares history with the original. [Medium; current-state-as-of-2026]

Worktrees are the natural substrate for parallel agents because the isolation is *also context isolation*. Each worker runs in its own directory, on its own branch, and therefore in its own clean filesystem context: its `git status` shows only its work, its tests run only against its changes, its diffs are uncontaminated by siblings. When a worker finishes, its branch is a self-contained, reviewable unit you can merge or discard.

> **Worked sketch — parallel worktrees.**
>
> ```
> # Operator sets up one worktree per worker group:
> git worktree add ../wt-group-1 -b migration/group-1
> git worktree add ../wt-group-2 -b migration/group-2
> git worktree add ../wt-group-3 -b migration/group-3
>
> # Each worker is launched in its own worktree directory with the
> # shared spec and its group's file list. Workers never see each other.
>
> # On completion, branches are independently reviewable:
> git log --oneline migration/group-1
> git diff main..migration/group-2
>
> # Merge clean branches; leave flagged ones for human review:
> git merge --no-ff migration/group-1
> ```
>
> When you are done, `git worktree remove ../wt-group-1` cleans up. The discipline is one branch per worker, reviewed before merge — the same discipline you would apply to human contributors, which is exactly the point.

The worktree pattern reframes the whole exercise. You are not "running many AIs." You are running many ordinary feature branches, each authored by an agent, each subject to the same review gate you already trust for human-authored branches. The orchestration disappears into Git workflow you already know.

## The Cost Split: Right-Sizing the Model to the Role

Operator and worker roles differ not just in context size but in the *kind* of thinking each requires — and that difference is also a cost lever.

The operator reasons about structure: how to decompose, what the spec should say, how to interpret summaries, when to escalate. This is the high-judgment work, and it benefits from the strongest available reasoning model. The workers do scoped, well-specified execution: apply this spec to these files, run these tests, report. With a good spec, this is lower-judgment work that a smaller, faster, cheaper model can do reliably.

So a common configuration is a **capable model as the orchestrator and a lighter model for the workers** — in current Anthropic terms, an Opus-class orchestrator dispatching Sonnet-class workers. [Medium; current-state-as-of-2026] Because the workers are where the token volume lives (six workers each reading files and running tests dwarfs the operator's six-paragraph diet), routing that volume to the cheaper model can cut the cost of a large parallel job substantially while keeping the expensive reasoning where it actually matters.

Do not over-fit to specific model names — they will change, and the [verify]-and-date discipline from the rest of the book applies to every model identifier here. The durable principle is role-based right-sizing: **spend reasoning capacity on decomposition and judgment, spend cheap throughput on scoped execution.** The architecture, not the model choice, is what makes this safe — a worker model that is too weak for its scoped task will fail its tests, and the operator will catch it in the summary. That is the system working as designed.

## More Agents, More Attack Surface

Every advantage in this chapter has a shadow, and you should see it clearly before the next chapter forces you to.

A single agent reading a single repository has one injection surface: the files it reads. (You will learn in Chapter 12 that this single surface is already larger and more dangerous than most practitioners assume.) A multi-agent system multiplies that surface. Each worker reads files — so each worker is a place where untrusted repository content could become an instruction. Workers pass summaries to the operator — so a compromised or confused worker can feed the coordinator a poisoned summary, and the operator, by design, *trusts summaries it does not re-verify.* The very property that makes orchestration efficient — the coordinator reasoning over summaries instead of raw content — is also a channel an attacker can target.

Worktrees and branches do not change this. A malicious instruction embedded in a file does not respect your branch boundaries; it executes wherever the file is read. Parallelism means more reads, running with less direct human attention per read, often against repositories you did not author.

This is the bridge to Chapter 12. Orchestration buys you scale, and scale is exactly what an attacker wants from you. Before you wire up a fleet of workers against a repository you did not write, you need a way to reason about what each of those agents can read, what it can do, and where it can send the results. Architecture, not prompting, is what will contain the blast radius — and that is the subject of the next chapter.

## Exercises

These exercises emphasize **Create** and **Evaluate** — designing orchestrations and judging when not to.

1. **(Evaluate) The seam test.** Take a real task in a repository you know — a dependency upgrade, a rename, a lint-rule rollout. Write down a candidate split into worker subtasks. Then, for each pair of subtasks, answer in one sentence: *could these two workers make conflicting decisions in isolation?* If yes for any pair, the seam is not clean. Either redraw it or name the coordination artifact (spec, frozen interface) that would make it clean. Submit the before-and-after split.

2. **(Create) An operator prompt.** Write a complete operator task prompt for a task that genuinely exceeds one window. It must: forbid the operator from doing the work itself, specify the decomposition, define what each worker receives, state that the operator reasons over summaries (with the one condition under which it may escalate to a full read), and include an explicit stopping condition before any unauthorized follow-up. Keep it under one page.

3. **(Create) A worktree run.** On a scratch repository, set up three worktrees on three branches, make a distinct trivial change in each, and merge two while discarding one. Capture the `git worktree`, `git diff`, and `git merge` commands you used. Write two sentences on how branch-per-worker review changes your trust in agent-authored changes versus a single agent editing in place.

4. **(Evaluate) The cost-and-risk trade.** For your task from Exercise 1, decide: which role gets the strong model and which gets the light one, and *why*. Then write the one paragraph you would add to Chapter 12's injection-surface map for this orchestration: every place a worker reads untrusted content, and every summary the operator trusts without re-verifying.

5. **(Evaluate) When not to orchestrate.** Describe a task that *looks* like it needs multiple agents but does not — one where the right answer is a single well-scoped session. Explain, in terms of context size and seam-cleanliness, why adding agents would make it worse.

> ### The AI Wayback Machine — Barbara Liskov and the Discipline of the Seam
>
> In 1972, Barbara Liskov was arguing for something the field had not yet internalized: that the way to build large software systems was to decompose them into modules that hide their internals behind clean interfaces, so that one part could be reasoned about — and changed — without holding the whole system in mind. Her work on abstraction and modularity, later crystallized in the substitution principle that bears her name, was fundamentally about *where you draw the boundary* so that pieces compose.
>
> Half a century later, you are rediscovering Liskov's insight at the level of context windows. An operator-and-workers orchestration succeeds or fails exactly on the quality of its seams — and a "clean seam" is precisely a clean interface: a boundary across which a worker can do its job knowing only the spec, not the internals of its siblings. The split-and-merge pattern is information hiding wearing new clothes. When a refactor splits badly because two workers both touch a shared function, you are watching a Liskov violation in real time: the modules were not actually independent, because their hidden dependencies leaked across the boundary.
>
> *Wayback prompt:* Connect multi-agent orchestration to Liskov's argument for abstraction and modularity. What did she see about decomposition and interfaces in the 1970s that CLI-agent users are now rediscovering when they choose where to split a task across agents?
>
> *(Liskov's foundational role in software abstraction is well documented [High]; the connection drawn here to context-window decomposition is the author's pedagogical framing, not a claim Liskov made.)*

## Sources

- Yang, J., et al. "SWE-agent: Agent-Computer Interfaces Enable Automated Software Engineering." *NeurIPS* 2024. — Peer-reviewed anchor for the claim that interface and context design, not raw model capability, drive agent performance; grounds the "context is the bottleneck" framing of decomposition.
- Yao, S., et al. "ReAct: Synergizing Reasoning and Acting in Language Models." 2022/2023. — Vocabulary for the read/reason/act/observe loop that each worker runs and the operator coordinates.
- Shinn, N., et al. "Reflexion: Language Agents with Verbal Reinforcement Learning." 2023. — Evidence that accumulated context can help or hurt depending on what is preserved; motivates summaries-not-full-context for the operator.
- Liskov, B. Foundational work on data abstraction and modularity (1970s onward); the Liskov Substitution Principle. — Anchor for the AI Wayback Machine box on clean seams as clean interfaces. [High for Liskov's role; the orchestration analogy is editorial]
- Anthropic, "Claude Code documentation," current online docs (accessed 2026). — Current-state reference for sub-agents, model roles (Opus/Sonnet), and worktree workflows. **Tool specifics, model names, and cost behavior must be rechecked before print. [verify]**
- Git documentation, `git worktree`. — Reference for the parallel-worktree pattern. [Medium; current-state-as-of-2026]

> **Note on evidence.** The orchestration patterns in this chapter are practitioner field knowledge, not peer-reviewed results. Specific model-role pairings (Opus orchestrator / Sonnet workers) and any cost figures are current-state-as-of-2026 and tagged [verify]. The stable core is architectural: split only when a window cannot hold the work, keep the operator on summaries, and draw clean seams.
