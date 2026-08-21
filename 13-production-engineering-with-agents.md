# Chapter 13 — Production Engineering with Agents

*Putting an agent in the pipeline, pricing its inconsistency, and deciding when not to use one*

Here is the pattern, and I want to describe it precisely because the marketing describes something else.

You assign a GitHub issue to a coding agent. The agent spins up its own ephemeral GitHub Actions environment — a disposable runner with its own filesystem. It explores the repository, edits files, runs the test suite and the linters inside that runner, and when it believes it is done, it opens a pull request. The PR shows up in your queue looking like any other PR. CI runs against it, the same CI that runs against a human's PR. You review the diff. Tests pass and the code looks right? You merge.

The agent autonomously *proposes.* CI autonomously *verifies* — mechanically, the same way it verifies everything. The agent does not autonomously *merge.* A human holds that gate.

This is **supervised autonomy**, and it is the convergent 2026 production consensus: agent acts inside guardrails, CI verifies, a human holds the merge. Not "fully autonomous ship-to-prod" — that is not mainstream practice. Not "the agent writes code and you trust it." The agent writes code and you review it, the same way you review anyone's code, because the pipeline does not care that the committer is an agent. The pipeline never trusted the committer. It trusted the test suite.

![A left-to-right pipeline. An assigned issue feeds an ephemeral isolated runner box containing the agent, test/lint runs, and file edits. The runner opens a pull request, continuous integration mechanically verifies it, and a human-held merge gate is the only crossing into the main branch.](../images/13-production-engineering-with-agents-fig-02.png)
![Supervised autonomy: the agent proposes inside an ephemeral runner, CI verifies, and only a human crosses the merge gate.](images/13-production-engineering-with-agents-fig-02.png)
*Figure 13.2 — Supervised autonomy: the agent proposes inside an ephemeral runner, CI verifies, and only a human crosses the merge gate.*

That last sentence is not a throwaway. It is the whole book in one line.

<!-- → [INFOGRAPHIC: Supervised autonomy pipeline — left to right: agent spins up ephemeral environment → edits files, runs tests/linter → opens PR → CI verifies → human reviews diff → human merges; human gate shown explicitly at merge step, not at any earlier step] -->

---

## The pipeline is the verification loop, industrialized

There is a Wayback moment worth a paragraph here because it clarifies what stays constant when an agent commits into the pipeline.

Grady Booch coined the phrase "continuous integration" in *Object-Oriented Analysis and Design with Applications* (1991), where it named the practice of integrating work frequently rather than in a big-bang merge at the end. Kent Beck and Extreme Programming later operationalized it into the automated build-and-test-on-every-commit pipeline we now mean by CI. Trace the idea across three eras — Booch's *integrate often*, XP's *automate the integration check*, and today's *an AI agent commits into that automated check* — and one thing is constant: the value was never in who writes the code. It was in the mechanical check that runs on every integration. The agent is a new kind of author. The check is the same institution it always was.

That is precisely why supervised autonomy works at all. The pipeline already existed for the right reasons before agents arrived, and those reasons are unchanged. The tests, types, lint, and build that a careful engineer runs by hand — the verification loop the whole book has been building task by task — are run automatically on every PR, and the human merge gate of Chapter 8 sits at the end. Production engineering with agents is not a new architecture. It is the verification loop scaled up and made continuous.

The agent's ephemeral, isolated environment also delivers the security argument from Chapter 12, now for free. A disposable Actions runner with no standing secrets is already a sandbox. The security chapter said "run the agent where a compromise is contained"; the production substrate provides that without additional work, because the runner is destroyed after every job. Architecture handled it before you had to ask.

---

## Agents are inconsistent: measure pass^k, not pass@1

The most important thing to internalize before deploying an agent is that it will not give you the same result twice. Your pipeline design has to price that.

Recall from Chapter 11: **pass@k** is the probability the agent succeeds on *at least one* of k independent trials — the optimistic metric, the one demos report. **pass^k** is the probability the agent succeeds on *all* k independent trials — the metric that tells you whether you can rely on a single unattended run. They diverge sharply for an inconsistent agent.

τ-bench (Yao, Shinn, Razavi, and Narasimhan, 2024; ICLR 2025) made this concrete. Testing tool-using agents on realistic tasks, it found that even GPT-4o succeeds on fewer than half its tasks and is, in the authors' words, "terribly inconsistent" — **pass^8 fell below 25% in the retail domain.** Run the agent eight times on the same task. The probability it succeeds every single time is under one in four, even though any individual run might look fine.

The pipeline consequence is the load-bearing one: **you cannot ship on per-run cleverness; you ship on the loop.** A workflow that resolved an issue once in a demo has told you almost nothing about whether it resolves issues reliably. This is the same argument the book has been making since Chapter 1 — the agent is inconsistent, and the *loop* is what carries an inconsistent agent to a reliable outcome — now expressed as a number. If pass^k is low, you do not respond by trusting the agent more. You respond by tightening the gate: more mechanical checks, mandatory human review, a narrower autonomy band. The inconsistency is a structural fact to engineer around, not a temporary bug to wait out.

This is also the lens for reading every autonomy claim you will encounter in the wild.

<!-- → [CHART: pass@k vs pass^k curves for a hypothetical agent at p=0.5 per trial — x-axis k (1 to 8), y-axis probability; pass@k climbing toward 1, pass^k decaying toward 0; gap labeled "what demos report" vs "what production requires"] -->

---

## How to read an autonomy claim: the Devin case

In March 2024, Cognition AI launched Devin as "the first AI software engineer," reporting 13.86% of issues resolved on SWE-bench unaided — a roughly 7× jump over prior systems. The launch video was extraordinary. The phrase entered the discourse. And nearly every part of that sentence needs a label.

**Label one: the benchmark.** The 13.86% was on **SWE-bench *Lite* — a 300-issue subset**, not the full SWE-bench. "13.86% on SWE-bench" and "13.86% on a 300-issue subset of SWE-bench" are different claims, and the shorter one is the one that traveled. Always ask *which* benchmark and *which subset*.
<!-- FACT-CHECK FLAG: CONTRADICTED — see factchecks/13-production-engineering-with-agents-assertions.md (per Cognition's technical report, the 13.86% was on a RANDOM 25% subset of SWE-bench (~570 tasks), not SWE-bench Lite (300). Correct the subset name/size — the "label the subset" point is right, the attribution is not) -->

**Label two: the framing.** "First AI software engineer" and "~7×" are Cognition self-reported marketing — not independent findings, not peer-reviewed claims. The 13.86% success rate was led with; the **86% failure rate** — the more informative figure for anyone deciding whether to deploy — was undercontextualized. A correctly skeptical reader inverts the headline: a system that fails 86% of unaided attempts is a system you supervise, not trust.

**Label three — the most important one: independent replication.** In January 2025, Answer.AI ran an independent evaluation of Devin 1.0 across 20 real-world tasks. The result: **3 successes, 14 failures, 3 inconclusive.** The three data scientists who ran it also reported they could find no pattern predicting which tasks would succeed. That is the number to anchor on, because it is the one a vendor did not produce. It is broadly consistent with "fails most of the time" and sharply inconsistent with "first AI software engineer" as that phrase is heard.

The contrast between the launch framing and the independent eval *is* the lesson. Demand independent replication and pass^k before believing an autonomy claim.

A fourth data point sharpens the cost angle. Princeton's SWE-agent (Yang et al., NeurIPS 2024) is widely reported at around 12% resolution on comparable tasks at far lower latency — roughly 90 seconds per task versus Devin's several minutes (treat those figures as reported, not independently re-verified). Comparable resolution, a fraction of the wall-clock. "How well" and "how expensively" are separate axes, and the marketing optimizes the first while you pay for the second.

Fred Brooks made the structural prediction in 1986: there would be no silver bullet yielding even a tenfold improvement in software productivity, because most remaining difficulty is *essential* — deciding what to build — rather than *accidental* — the labor of building it. Read the "~7×" marketing against that 40-year-old frame and the skepticism writes itself. Agents demonstrably crush *accidental* complexity: the typing, the boilerplate, the mechanical edits. They demonstrably struggle with *essential* complexity: the deciding. That is why the independent eval found a failure-heavy result on real tasks. The essential complexity is where real tasks live.

---

## Cost and latency, because turns compound

A coding agent's cost is not one model call. It is many, in sequence, each re-serializing the accumulated context, and that structure is why agentic work is expensive in a way that surprises people pricing it like a chat completion.

![Six stacked-bar columns on a common baseline, one per turn, each taller than the last. The top block of each column is newly added context; the blocks beneath are carried-over context re-processed every turn. Arrows chain the columns to show sequential dependency, and a lengthening arrow beneath shows wall-clock latency stacking.](../images/13-production-engineering-with-agents-fig-03.png)
![Why agent cost compounds: each sequential turn re-processes a growing context while latency stacks.](images/13-production-engineering-with-agents-fig-03.png)
*Figure 13.3 — Why agent cost compounds: each sequential turn re-processes a growing context while latency stacks.*

GitHub's engineering blog reports that agentic workflows can consume on the order of **10–20× more compute per task** than a single completion. Label that multiplier as illustrative, not a constant — it is vendor-stated, and independent task-normalized cost studies are scarce. But the *direction* is solid and the *mechanism* is clear. Latency compounds across turns. An agent that takes twenty turns to resolve an issue makes twenty round-trips, each re-processing a growing working context. Sequential turns also mean wall-clock latency stacks: you wait for turn nineteen before turn twenty starts.

The practice converging in production is **cost observability**: GitHub now emits a per-call `token-usage.jsonl` artifact recording input, output, cache-read, and cache-write tokens for every call in a workflow. That is the discipline to adopt. You cannot manage what you do not measure, and "the agent did it" is not a cost report. Emit per-task token telemetry. Cap the number of turns. Watch the cache-read fraction — a high cache-read rate means you are re-paying to re-process stable context, which is a signal to restructure the prompt rather than pay again.

A real-world reminder that this is a *capacity* decision, not only a quality one: GitHub temporarily paused Copilot sign-ups in 2025 as agentic load strained capacity [journalistic account — flag]. Whether or not that specific episode generalizes, the principle does. Putting agents in a pipeline is a compute-budget decision at the organizational scale, and "it works" does not imply "it scales affordably."

The question that every placement decision has to answer: **does the leverage exceed the overhead?** Overhead is token cost plus wall-clock latency plus human review burden plus the injection surface from Chapter 12. For a large mechanical migration across a monorepo, the leverage is enormous and the overhead earns its keep. For a one-line config change with a known fix, it does not.

<!-- → [TABLE: Cost-benefit placement examples — columns: Task type, Estimated turns, Leverage, Overhead, Decision; rows: mechanical migration, formatting PR, one-line config fix, multi-file refactor, dependency major-bump; illustrates where leverage-exceeds-overhead boundary sits] -->

---

## When not to use an agent

The most senior move in agentic engineering is declining to use the agent when the agent is the wrong tool. The chapter's payoff is a decision procedure, and that procedure includes the option to say no.

![Five decision diamonds stacked vertically: mechanically checkable, reversible and low-blast-radius, trusted repo, leverage exceeds cost, and not tiny-and-obvious. Each passes downward to the next; any single failure branches right into one shared do-not-deploy sink. Passing all five reaches a deploy-supervised terminal.](../images/13-production-engineering-with-agents-fig-04.png)
![The when-not-to-use-an-agent decision tree: any one failed gate routes the task to "don't deploy the agent."](images/13-production-engineering-with-agents-fig-04.png)
*Figure 13.4 — The when-not-to-use-an-agent decision tree: any one failed gate routes the task to "don't deploy the agent."*

Here is the checklist. Any one item can be disqualifying.

**1. Is correctness mechanically checkable?** If there is no test, type, compiler, or CI signal that can tell the agent — and you — whether the output is right, you have removed the verification loop the entire book depends on. No oracle means no closed loop means you are trusting output you cannot check. Either there is a human, or you build the oracle first. This is the deepest criterion; it is the book's thesis stated negatively.

**2. Is the change reversible and low-blast-radius?** A formatting PR is reversible. A database migration, a production canary promotion, a feature-flag flip, a force-push are not. Irreversible or high-blast-radius means a human gate is mandatory, and the autonomy band drops accordingly. Nancy Leveson's systems-safety framework makes the underlying point: safety is a property of the *control structure*, not the component. Making the agent more capable does not fix this — the gate has to sit in the architecture, ahead of the irreversible act, regardless of how reliable the agent is.

**3. Is the repository trusted?** From Chapter 12: an untrusted repo means the agent ingests attacker-controllable content while holding the trifecta. Untrusted means sandbox, no autonomy, no auto-approve.

**4. Does leverage exceed cost?** From the previous section: if token cost plus latency plus review burden plus injection surface exceeds the value of having the agent do it, the agent is the expensive way to do a cheap thing. The canonical example: a one-line, well-understood config change with a known fix. The agent's setup, turns, review, and risk all cost more than just making the edit.

**5. Is the task small and well-understood with a known fix?** The smaller and more certain the task, the worse the agent's overhead ratio. Agents earn their keep on *bounded-but-laborious* work — a large mechanical refactor, a systematic test migration — not on *tiny-and-obvious* work where the human solution is cheaper and faster.

<!-- → [INFOGRAPHIC: Decision tree — "When not to use an agent" — five yes/no nodes in sequence: (1) Is correctness checkable? (2) Is change reversible? (3) Is repo trusted? (4) Does leverage exceed cost? (5) Is task bounded-but-laborious? Each "no" routes to "don't deploy the agent here"; all five "yes" routes to autonomy band selection] -->

Run a candidate task down that list. If every answer points toward "agent," choose an autonomy band — assist, propose-and-review, policy-gated auto-merge, or autonomous deploy. If any answer is a disqualifier, the correct output of the decision procedure is "don't deploy the agent here." The book has insisted since Chapter 8 that architecture decides; this is that insistence applied to the prior question of *whether the architecture should include an agent at all.*

---

## The autonomy spectrum and where the gate sits

To make "choose an autonomy band" concrete, here is the spectrum:

![A four-band horizontal spectrum: assist, propose-and-review, policy-gated auto-merge, and autonomous deploy. The human gate moves progressively later across the bands. Propose-and-review is marked as the current production norm; autonomous deploy is flagged rare and high-risk. An arrow below shows increasing autonomy.](../images/13-production-engineering-with-agents-fig-01.png)
![The supervised-to-autonomous spectrum: the human gate sits later on each band, with propose-and-review the current norm.](images/13-production-engineering-with-agents-fig-01.png)
*Figure 13.1 — The supervised-to-autonomous spectrum: the human gate sits later on each band, with propose-and-review the current norm.*

| Band | What the agent does | Where the human gate sits | Status (2026) |
|---|---|---|---|
| **Assist** | Autocomplete / inline suggestion | Human writes every committed line | Ubiquitous |
| **Propose-and-review** | Agent opens a PR; CI verifies | Human reviews and merges | **Current production norm** |
| **Policy-gated auto-merge** | Low-risk classes auto-merge on green CI | Human gate on escalated classes only | Emerging |
| **Autonomous deploy** | Agent ships to prod unattended | Post-hoc only | **Rare / high-risk** |

<!-- → [FIGURE: Autonomy spectrum as a horizontal continuum — four bands labeled Assist, Propose-and-review, Policy-gated auto-merge, Autonomous deploy; human gate position shown as a moveable threshold; reversibility risk increasing left to right; 2026 "current norm" marker on Propose-and-review] -->

The interesting research frontier is the third band. A 2025 preprint, "AI-Augmented CI/CD Pipelines" (arXiv:2508.11867), proposes a **policy engine** as the mechanism: agent actions are evaluated against defined criteria, some auto-allowed, some requiring human approval, some denied, with adjustable confidence thresholds. A policy engine might auto-allow a formatting PR or a dependency patch-bump with green CI, while *escalating* a rollback, a canary promotion, or a feature-flag flip to a human. Flag the paper as a preprint — a useful framework, not a validated result. But notice that the escalation rule is just the reversibility criterion from §5 above, encoded as machine-checkable policy. Leveson's control structure, written down as thresholds.

How far up the spectrum production *should* go is genuinely disputed. The book's position is conservative, for the reason §2 gave: until pass^k on *your* tasks is high enough to stake an irreversible action on — and it rarely is — the gate stays. τ-bench's sub-25% pass^8 is the number that licenses conservatism. A high pass^k, independently measured on your operational profile, is the number that would justify moving the gate.

---

## What the chapter argues

I have been developing one claim from several directions: putting an agent in production is not a capability question, it is a placement decision. The decision has two governing numbers — pass^k (how reliably does it succeed every time?) and cost-per-task (does the leverage exceed the overhead?) — and a checklist of disqualifiers that includes the option of saying no.

The supervised-autonomy pattern — agent proposes, CI verifies, human merges — is the current production norm because it routes the mechanical to the loop and the judgment-laden to the human, the same argument Chapter 8 made for code review, now at pipeline scale. The verification loop the book has been building task by task is the same institution as continuous integration; the agent is a new kind of author; the check is unchanged. Grady Booch's insight from 1991, Kent Beck's operational pipeline, Nancy Leveson's control-structure argument — none of them anticipated agents, and all of them point to the same design.

The most common mistake I see is treating a passing demo as a deployment decision. A demo samples pass@k. A deployment decision requires pass^k. Those are different questions, and the τ-bench data says the gap between them is large.

---

## LLM Exercises

**Exercise 13.1 (Evaluate).** Find a current vendor autonomy claim — a coding-agent benchmark number from a launch post or a model card. Decompose it the way the chapter decomposes Devin: which benchmark and which subset? self-reported or independent? pass@1, pass@k, or pass^k? what is the *failure* rate implied by the success rate, and is it stated? Write a one-paragraph verdict on how much you would trust the claim for an unattended production run, and name the single piece of evidence you would most want before trusting it further.

**Exercise 13.2 (Evaluate).** Take a real agentic task you have run — or run one — and estimate its cost profile: number of turns, approximate tokens per turn, wall-clock latency, and, if your tool emits it, the per-call token telemetry and cache-read fraction. Compare the total against doing the task by hand. State whether leverage exceeded overhead, and identify the one structural change (fewer turns, more caching, narrower scope) that would most improve the ratio.

**Exercise 13.3 (Create).** Write your team's "when not to use an agent" checklist as a decision procedure, grounded in the five criteria above but specialized to your stack — name your real CI signals, your irreversible actions, your untrusted-repo sources. Then apply it to three real tasks from your backlog: classify each as autonomous-band, supervised-band, or no-agent, and for at least one, show the procedure *disqualifying* the agent. Justify each classification against reliability, cost, reversibility, and trust — not vibes.

**Exercise 13.4 (Create).** Design a policy table for a propose-and-review pipeline in your repo. List at least five action classes (formatting PR, dependency patch-bump, dependency major-bump, schema migration, feature-flag flip), and for each specify: auto-allow on green CI, require human approval, or deny. Justify each escalation by the reversibility-and-blast-radius criterion, and state the pass^k threshold you would require before promoting any class from "require approval" to "auto-allow."

---

## References

- Booch, G. (1991). *Object-Oriented Analysis and Design with Applications.* Benjamin/Cummings. — coined "continuous integration"; Kent Beck / Extreme Programming later operationalized it into the automated build-and-test-on-every-commit pipeline.
- Cognition AI (2024). *Introducing Devin* / *SWE-bench technical report.* cognition.ai, March 2024. — 13.86% resolved on a **random 25% subset of SWE-bench (~570 tasks)**, unassisted; self-reported "first AI software engineer."
- Bornstein, J., et al. / Answer.AI (2025). *Thoughts On A Month With Devin.* answer.ai, Jan 8 2025. — independent eval of Devin 1.0: 3 success / 14 fail / 3 inconclusive of 20; no predictive pattern.
- Yang, J., et al. (2024). *SWE-agent: Agent-Computer Interfaces Enable Automated Software Engineering.* NeurIPS 2024. arXiv:2405.15793. — ~12% resolution at far lower latency (~90s/task reported).
- Yao, S., et al. (2024). *τ-bench.* ICLR 2025. arXiv:2406.12045. — pass@k vs. pass^k; retail pass^8 < 25%.
- Brooks, F. P. (1986/1987). *No Silver Bullet: Essence and Accidents of Software Engineering.* IFIP / IEEE Computer. — no tenfold productivity bullet; essential vs. accidental complexity.
- Leveson, N. (2011). *Engineering a Safer World: Systems Thinking Applied to Safety.* MIT Press. — safety is a property of the control structure, not the component.
- GitHub Engineering (2025). *Agentic workflow compute and token telemetry* (`token-usage.jsonl`). github.blog. — agentic workflows ~10–20× compute per task (vendor-stated, illustrative).
- *AI-Augmented CI/CD Pipelines.* (2025). arXiv:2508.11867 [preprint]. — policy-engine framework for graduated autonomy.

