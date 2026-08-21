# Chapter 11 — Evaluating Coding Agents

*pass@k flatters, pass^k is production: best-of-k versus all-of-k is the whole measurement story*

Suppose your agent has a 90% chance of solving a task on any given attempt. You run it eight times and ask: did at least one of the eight succeed? Almost certainly — the probability that all eight independently fail is $0.1^8$, essentially zero, so the "best of eight" number comes out at about 99.99%. Now ask the opposite question: did *all eight* succeed? That's $0.9^8 \approx 0.43$. A coin flip, near enough.

Same agent. Same task. Same eight runs. One framing gives you 99.99%; the other gives you 43%. Which number you report is the entire difference between a press release and a production decision.

![A line chart over attempts k from 1 to 8. For a single agent with fixed per-attempt success, the best-of-k probability rises and asymptotes toward 1, while the all-of-k probability decays toward 0. The two curves start close together at k equals 1 and scissor apart; a bracket marks the widening gap at a right-hand k.](../images/11-evaluating-coding-agents-fig-01.png)
![The pass@k / pass^k scissoring curves: best-of-k rises toward 1 while all-of-k falls toward 0, the gap between them being capability minus reliability.](images/11-evaluating-coding-agents-fig-01.png)
*Figure 11.1 — The pass@k / pass^k scissoring curves: best-of-k rises toward 1 while all-of-k falls toward 0, the gap between them being capability minus reliability.*

This is the chapter's thesis in its most compressed form, and the rest of the chapter is just unwinding what that gap means — for benchmarks, for leaderboards, for the only measurement that actually matters, which is whether your agent is reliable enough to deploy on your code.

<!-- → [CHART: Two curves on one axis — pass@k rising toward 1 as k grows, pass^k decaying toward 0 as k grows, for a hypothetical agent with p=0.9 per trial; the scissoring gap between them labeled "capability vs. reliability"; x-axis: k (1 to 10), y-axis: probability] -->

---

## Two metrics, opposite meanings

The optimistic number is **`pass@k`**, defined in Chen et al. (2021), "Evaluating Large Language Models Trained on Code" — the Codex/HumanEval paper. `pass@k` is the estimated probability that *at least one* of `k` sampled solutions passes the unit tests. To estimate it without bias, the paper generates $n \geq k$ samples per task (in the paper, $n = 200$, $k \leq 100$), counts $c$ that pass, and uses the estimator

$$\text{pass@}k = 1 - \frac{\binom{n-c}{k}}{\binom{n}{k}}$$

Read the structure of that formula: it's *one minus the probability that all $k$ drawn samples come from the failing set.* `pass@k` rises toward 1 as $k$ grows, because more tries make it more likely that *one* succeeds. It is a **best-of-k, capability** measure — it answers the question *can* the agent produce a correct answer, given several attempts?

The pessimistic number is **`pass^k`** (read "pass-hat-k"), introduced in Yao et al. (2024), "τ-bench." `pass^k` runs the same task $k$ independent times and counts a success only if *all $k$ trials succeed.* It falls toward 0 as $k$ grows, because every additional trial is another chance to fail. It is an **all-of-k, reliability** measure — it answers the question *does* the agent succeed every time?

These two curves scissor. That word is chosen carefully: `pass@k` goes up and to the right; `pass^k` goes down and to the right; they cross somewhere in the middle and move apart. The gap between them at any $k$ is the gap between *capable* and *reliable*, and τ-bench made that gap concrete and uncomfortable. Frontier function-calling agents, tested on real retail service tasks, hit **`pass^8` below 25%** [verify against the paper]. An agent that looks strong at `pass@1` can collapse at `pass^8`. The single most useful habit this chapter can give you: when you see a benchmark number, ask *best-of-k or all-of-k?* — and if it's the former, know you're reading the optimistic story.

Which one does production care about? When your agent ships a code change and a human reviews the diff, you do not get to sample eight diffs and pick the good one. You get the one it produced, and you need it right. **Production is `pass^k` country.** A leaderboard quoting `pass@1` — or worse, `pass@k` for large $k$ — is reporting capability: what the agent can do at its best. You are deploying reliability: what it does every time, under conditions that are not its best.

> **The misconception to kill at the door.** "The agent scores 90% on the benchmark, so it'll succeed 90% of the time in production." Almost certainly false, in both directions. If the 90% is `pass@k`, your single-attempt reliability is lower — possibly far lower. And the production number is a `pass^k`-style question on *your* task distribution, which the benchmark's best-of-k on *its* distribution doesn't answer. The headline is capability on someone else's tasks; you need reliability on yours.

---

## What a good benchmark looks like

Before the metrics mean anything, the benchmark has to grade on ground truth — not on whether the output *resembles* a correct answer, but on whether it *is* one. SWE-bench is the field's anchor for doing that at repository scale, and its story is worth understanding because it shows both what right looks like and how fast right can expire.

Jimenez et al. (2023/2024) built **2,294 task instances** from real issue-and-pull-request pairs across twelve popular Python repositories. Each instance ships a Docker environment containing **Fail-to-Pass tests**: tests that fail before the PR's change and pass after it. The grading signal is mechanical and non-fakeable — a candidate patch is correct if and only if the Fail-to-Pass tests go from red to green. This is the book's thesis as a measurement principle: evaluate against ground-truth execution, not a similarity score against a reference solution. A patch that looks nothing like the human fix but passes the tests, passes. A patch that looks identical but doesn't, fails.

![Inside a single containerized environment, a repository at the pre-fix commit shows a column of failing test markers. A candidate patch is applied, after which the same test column passes. The before-to-after flip emits a single correct-if verdict — the grading oracle is execution, not string similarity.](../images/11-evaluating-coding-agents-fig-02.png)
![SWE-bench Fail-to-Pass grading: inside a container, a patch is correct only when held tests flip from failing to passing.](images/11-evaluating-coding-agents-fig-02.png)
*Figure 11.2 — SWE-bench Fail-to-Pass grading: inside a container, a patch is correct only when held tests flip from failing to passing.*

Two facts about SWE-bench reshape how you read any score from it.

The first: **the score measures the agent plus harness plus benchmark, not the model.** SWE-agent (Yang et al., NeurIPS 2024) lifted SWE-bench resolution from 3.8% — a RAG baseline — to 12.5% *by changing the agent-computer interface alone.* The scaffold changed; the underlying model didn't. The number tripled because the harness changed. So when a vendor quotes a resolution rate, the scaffold is doing unstated work. "Model X scores Y%" is meaningless without "under which scaffold."

The second: **instance quality is itself a measurement problem.** The original SWE-bench contained tasks that were underspecified, had unfair tests, or were unsolvable within the harness budget — noise that distorted scores in both directions. OpenAI and the SWE-bench authors released **SWE-bench Verified** (August 2024): a human-validated 500-instance subset, reviewed by 93 contracted developers, with roughly 68.3% of sampled instances filtered as flawed [verify these figures]. Verified became the dominant frontier-coding reporting target in late 2024 — a clean case of *fixing a benchmark's measurement validity* by removing the instances that didn't measure what they claimed.

For a moment, the field had a clean, execution-grounded, human-validated standard. And then the ground moved under it.

<!-- → [TABLE: SWE-bench family — columns: Variant, Year, Size, Contamination defense, Current status; rows: SWE-bench, SWE-bench Verified, SWE-bench-Live, SWE-bench Pro; every number stamped with source date] -->

---

## The benchmark OpenAI retired

A benchmark is a fair measurement only until the systems it measures have seen its answers. This is the contamination problem, and SWE-bench Verified's story is the cleanest public demonstration of it.

The critique arrived first. "The SWE-Bench Illusion: When State-of-the-Art LLMs Remember Instead of Reason" (arXiv:2506.12286, preprint) reported that models could identify the buggy file paths from the issue text alone — no repository access required — at up to about 76%, dropping to about 53% on repositories not in SWE-bench. A model that names the file to fix without seeing the code is not reasoning about the bug. It is recalling the benchmark. The paper also found direct solution leakage — solution code present in the issue text or comments — affecting 30%+ of "passes" without filtering, and that 135 of 500 Verified instances contain ground-truth file paths in the issue text [verify all figures]. A score that was a fair capability measurement in 2024 had drifted, by 2025, toward measuring training exposure.

![A time axis from 2023 to 2026 carries four benchmark variants whose bar heights encode relative instance count. The 2023 original is superseded (gray), the 2024 Verified variant is deprecated (emphasized) with a downward arrow to a barrier on the axis, and two 2025 variants remain active. Freshness decays as each ages past a training cutoff.](../images/11-evaluating-coding-agents-fig-03.png)
![The SWE-bench family on a dated freshness timeline: each variant is a perishable snapshot, with Verified deprecated and fresher variants succeeding it.](images/11-evaluating-coding-agents-fig-03.png)
*Figure 11.3 — The SWE-bench family on a dated freshness timeline: each variant is a perishable snapshot, with Verified deprecated and fresher variants succeeding it.*

Then, on February 23, 2026, OpenAI published "Why SWE-bench Verified no longer measures frontier coding capabilities" and deprecated the benchmark it had helped build. The stated reasons: roughly 60% of failed problems had flawed tests, and there was evidence frontier models had trained on benchmark solutions. OpenAI recommended shifting toward contamination-resistant and privately-authored evaluations [verify against the post].

When the organization with the most to gain from a high SWE-bench Verified score retires the metric as a measure of capability, the lesson is not subtle. **Benchmark numbers are dated snapshots, and the date matters as much as the number.** A 2024 SWE-bench Verified figure and a 2026 one are not the same measurement, even for the same model, because the relationship between the score and genuine capability decayed in between.

<!-- → [FIGURE: Timeline of SWE-bench Verified's lifespan — left to right: Aug 2024 (Verified released, 500 human-validated instances), late 2024 (frontier reporting target), 2025 (Illusion paper: file-path recall ~76%, solution leakage in 135/500 instances), Feb 23 2026 (OpenAI deprecates); caption: "From gold standard to deprecated in eighteen months — the contamination half-life of a public benchmark"] -->

The field's response has been freshness and privacy. SWE-bench-Live (Microsoft, arXiv:2505.23419, NeurIPS 2025) is a contamination-resistant, continuously updated variant: 1,319 instances created January 2024 through April 2025, restricted to post-cutoff issues [verify counts and window]. The defense is temporal — a task created after a model's training cutoff cannot have leaked into training. SWE-bench Pro (harder, private split) and privately-authored evaluations are the other directions. No successor has been standardized; each trades realism, freshness, and accessibility differently, which is part of why the chapter's practical deliverable is not a better public leaderboard.

---

## Build your own regression suite

The stable skill — the one that outlives every leaderboard in the previous section — is to measure *your* agent on *your* tasks in a harness *you* control. This is contamination-immune by construction, because it's private. No one can train on it. No one can leak it. Its freshness window doesn't expire because you create new instances as you work.

![A four-stage pipeline for a private regression suite: curate your own Fail-to-Pass tasks, wrap each in a reproducible harness, run the whole workflow k times for an all-of-k reliability number, then gate every change on that number. The gate emits a merge arrow and a feedback arc back to the run stage so the suite re-runs on every change.](../images/11-evaluating-coding-agents-fig-04.png)
![Build-your-own private regression suite: curate own tasks, wrap in a reproducible harness, run k times for pass^k, and gate every change on that number.](images/11-evaluating-coding-agents-fig-04.png)
*Figure 11.4 — Build-your-own private regression suite: curate own tasks, wrap in a reproducible harness, run k times for pass^k, and gate every change on that number.*

The recipe is a small SWE-bench, built from your real history.

**Start with real tasks.** Take actual issues your team faced — bugs you fixed, features you shipped — each with the real Fail-to-Pass tests: the tests that failed before the fix and passed after. A dozen well-chosen tasks from your own codebase beats a thousand generic benchmark instances, because they're your distribution. The whole point of an operational profile, in John Musa's reliability engineering sense, is that you measure failure-free operation under *how the system is actually used*, not under cherry-picked public cases.

**Wrap each in a reproducible harness.** A container with the repo at the pre-fix commit and the Fail-to-Pass tests, exactly SWE-bench's structure. The grading is mechanical: did the workflow produce a patch that turns the tests green? Execution-grounded, not similarity-scored.

**Run your workflow $k$ times and report `pass^k`.** Not `pass@1` on a public benchmark — your end-to-end workflow, including the agent, the scaffold, the prompts, and any human-in-the-loop steps, run $k$ times per task, scored on the fraction of tasks it gets right *every* time. That is the all-of-k reliability number production actually depends on.

**Gate changes on it.** Every prompt tweak, model swap, or scaffold change re-runs the suite as a regression gate. A change that improves a public benchmark but regresses your suite is a regression *for you.*

<!-- → [INFOGRAPHIC: Four-step private regression suite recipe — (1) curate real tasks with Fail-to-Pass tests, (2) container harness at pre-fix commit, (3) run workflow k times, report pass^k, (4) gate every change on the suite; styled as a reusable checklist, not prose] -->

There is a temptation to shortcut this by having a model *judge* your agent's output — point an LLM at the result and ask whether it's correct. Resist it. An LLM judge scoring your agent's output is just another model with the same problem sitting one layer up. Its failures correlate with the system it's grading. You have purchased confidence, not a check. The Fail-to-Pass test is an execution-grounded verifier precisely because it is not a model: it answers mechanically, the way `OBSERVE` does in the book's first act. That mechanical quality is what makes the loop close rather than recurse.

The practical heuristics that follow from all of this:

Don't quote `pass@1` on a public benchmark as your production number. It's capability on someone else's tasks; your production number is `pass^k` on yours. When a prompt tweak "improves" a benchmark score, re-run with `pass^k` on fresh, post-cutoff tasks — real improvement shows up in all-of-k reliability on work the model hasn't seen. When reading a vendor claim, ask three questions: which harness (the scaffold tripled SWE-agent's number)? `pass@1` or `pass^k` (capability or reliability)? Public and possibly contaminated benchmark, or privately-authored? Then discount accordingly.

> **The misconception to close with.** "A high public-benchmark score means the agent will be reliable on my codebase." No — on three counts simultaneously. The score is best-of-k capability, not all-of-k reliability. It's measured on the benchmark's task distribution, not yours. And it may be contaminated, meaning the agent saw the answers. Your regression suite corrects all three: it measures reliability, on your tasks, fresh. Trust it over any leaderboard, and date every external figure you cannot replace with your own.

This is the book's thesis turned fully reflexive. The whole book has argued that an agent is only as good as the verification loop you build around the task. To know how good your agent *is*, you build a verification loop around the *measurement* — execution-grounded, run it $k$ times, on your own operational profile — and you trust that loop over any number a vendor hands you. The loop closes on the measurement the same way it closes on the code.

---

## LLM Exercises

**Exercise 11.1 (Apply).** You are given run-level results for one agent on a task set (in the course repo, `exercises/ch11/`): for each task, the pass/fail outcome of 8 independent attempts. Compute `pass@1` (average single-attempt success), `pass@8` (at least one of eight), and `pass^8` (all eight) across the set. Plot `pass@k` and `pass^k` as functions of `k` on one axis and identify the `k` where they cross the "looks healthy / is unreliable" gap. In two sentences, state which number you'd put in a press release, which in a deployment decision, and why they differ.

**Exercise 11.2 (Analyze).** You are given three vendor benchmark claims. For each, identify (a) which metric it likely reports (best-of-k or all-of-k), (b) which harness/scaffold the number depends on, and (c) whether the benchmark is in its freshness window or possibly contaminated. Then write the single follow-up question that would most change your trust in the number, and state how you'd discount it.

**Exercise 11.3 (Create).** Build a minimal private regression suite. Curate **three** real tasks from a repo you control (or the course fork) — each a real bug or feature with the actual Fail-to-Pass tests — and wrap each in a reproducible harness (container or script: repo at pre-fix commit plus the tests). Run your agent workflow **`k = 5`** times per task and report **`pass^5`** (fraction of tasks solved in *all five* runs) alongside `pass@1`. Append a paragraph: where do the two numbers disagree, and what does the disagreement tell you about deploying this workflow?

**Exercise 11.4 (Evaluate, optional).** Take a prompt or scaffold change you believe improves your agent. Re-run your Exercise 11.3 suite before and after, reporting `pass^5` (not just `pass@1`) on both. Then, if you can, run on at least one post-cutoff task (created after the model's training date) to control for leakage. Report whether the "improvement" survives the all-of-k, fresh-task test — or whether it was variance, leakage, or a `pass@k`/`pass^k` artifact.

---

## References

- Chen, M., et al. (2021). *Evaluating Large Language Models Trained on Code.* arXiv:2107.03374. — defines `pass@k` and its unbiased estimator (n = 200, k ≤ 100); the Codex/HumanEval paper.
- Yao, S., et al. (2024). *τ-bench: A Benchmark for Tool-Agent-User Interaction in Real-World Domains.* ICLR 2025. arXiv:2406.12045. — introduces `pass^k`; frontier function-calling agents hit `pass^8` < 25% in retail.
- Jimenez, C. E., et al. (2024). *SWE-bench: Can Language Models Resolve Real-World GitHub Issues?* ICLR 2024. arXiv:2310.06770. — 2,294 instances / 12 repos; Fail-to-Pass execution grading.
- Yang, J., et al. (2024). *SWE-agent: Agent-Computer Interfaces Enable Automated Software Engineering.* NeurIPS 2024. arXiv:2405.15793. — 3.8% RAG → 12.5% by changing the interface; the score measures agent + harness, not the model.
- OpenAI (2024). *Introducing SWE-bench Verified.* openai.com, Aug 2024. — 500 human-validated instances; ~93 developer annotators.
- Liang, S., Garg, S., & Moghaddam, R. Z. (2025). *The SWE-Bench Illusion: When State-of-the-Art LLMs Remember Instead of Reason.* arXiv:2506.12286. — file-path recall up to ~76% from issue text alone, ~53% off-distribution; memorization, not reasoning.
- OpenAI (2026). *Why SWE-bench Verified no longer measures frontier coding capabilities.* openai.com, Feb 23, 2026. — ~60% of audited failed problems had flawed tests; contamination evidence; OpenAI deprecates the benchmark.
- Zhang, L., et al. (2025). *SWE-bench Goes Live!* (SWE-bench-Live, Microsoft.) NeurIPS 2025 D&B. arXiv:2505.23419. — 1,319 contamination-resistant instances (Jan 2024–Apr 2025), 93 repos.
- Musa, J. D. *Software Reliability Engineering.* — the operational profile: measure reliability under actual usage.

