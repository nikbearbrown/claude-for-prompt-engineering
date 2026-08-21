# Chapter 3 — A Taxonomy of Agentic Coding Tasks

*Classifying a coding task by where its oracle is — and predicting the failure before the agent runs*

---

Here are five tickets — the kind that land in any backlog:

1. *"Scaffold a REST endpoint matching this OpenAPI schema, with a passing contract test."*
2. *"Rename `getUser` to `fetchUser` across the service and update all call sites."*
3. *"Make the checkout flow faster."*
4. *"Fix this flaky test that fails about 1 in 20 runs."*
5. *"Build us an internal admin dashboard."*

Before any theory, ask one question of each — not "how hard is this?" but: **when the agent says it's done, what tells me it's right?**

For (1), the answer is immediate and cheap: the contract test passes or it doesn't. The schema is the spec; the test is the oracle. The agent can run it, see red, edit, see green, and stop — and "green" means what you wanted. For (2), almost as cheap: a rename is correct exactly when behavior is preserved, and the existing test suite measures that. If the tests were green before and green after, the rename landed.

For (3), the question has no answer. *Faster* by how much, measured how, on what workload, at what cost to readability? There is no test that goes from red to green. Until a human says "p95 latency drops below 200 ms on the standard load test with no behavior change," there is nothing for the agent to close its loop against. It will produce something — confidently — and you will have no mechanical way to know if it's an improvement.

For (4), there *is* an oracle — the test stops being flaky — but reaching it requires judgment. Flakiness hides in timing, in shared state, in ordering, in a clock or a network call that isn't mocked. The stopping condition is clear; the localization is not. The agent will confidently propose a fix, the test will pass on the next ten runs by luck, and the flake will return on run thirty.

For (5), the failure is the most insidious. The agent will fill every gap in the spec itself, plausibly and wrongly. The code may compile, run, and look like a dashboard. The failure is not broken code — it is **the wrong dashboard built well.**

You just sorted these by *oracle availability* — by whether a ground-truth check exists and how cheaply you could build one — and that ranking predicted the failure mode. The rest of this chapter makes the two underlying axes explicit, hangs them on externally validated data, and kills the misconception that almost certainly leaked into your sort: that harder means bigger.

---

## The two axes: oracle availability in disguise

Every coding task sits somewhere on a plane defined by two axes.

![A classification plane: the horizontal axis runs bounded to open-ended, the vertical axis mechanical to judgment-laden. The bounded-mechanical quadrant is agent-strong with a cheap oracle; the open-ended-judgment quadrant needs a human gate first. Five tickets are plotted, with greenfield as squares and brownfield as circles. A diagonal arrow shows the move to drag a task toward bounded and mechanical.](../images/03-taxonomy-of-agentic-coding-tasks-fig-01.png)
![The oracle-availability 2x2: tasks sort on bounded-versus-open-ended and mechanical-versus-judgment, with a diagonal arrow showing how to drag a task into the agent-strong quadrant.](images/03-taxonomy-of-agentic-coding-tasks-fig-01.png)
*Figure 3.1 — The oracle-availability 2x2: tasks sort on bounded-versus-open-ended and mechanical-versus-judgment, with a diagonal arrow showing how to drag a task into the agent-strong quadrant.*

The **horizontal axis is bounded ↔ open-ended.** A bounded task has a stopping condition — a definite "done" the agent can recognize without further negotiation. "All call sites updated and the suite is green" is bounded. Open-ended tasks have no natural terminus: "make it faster," "improve the UX," "clean this up." The agent cannot know when to stop because *you* have not decided what stopping means.

The **vertical axis is mechanical ↔ judgment-laden.** A mechanical task has a correctness criterion checkable without human design judgment: behavior preservation, a passing test, a type-check, a successful build. Martin Fowler's definition of refactoring is the cleanest statement of the mechanical pole — a change to the internal structure of the code that doesn't modify its observable behavior, carried out as a series of small behavior-preserving transformations. Behavior preservation is a mechanical oracle: run the tests, compare. A judgment-laden task requires deciding what the code *should* do — a question no test can answer, because the test would have to encode the very judgment you are trying to make.

Now the move that turns this from a tidy 2×2 into a working diagnostic. Both axes are really one question asked twice: **does a ground-truth oracle exist, and how cheaply can I build one?** Bounded asks whether there's a stopping condition. Mechanical asks whether that condition is machine-checkable. A task that is bounded *and* mechanical has a cheap oracle sitting right there. A task that is open-ended *and* judgment-laden has no oracle at all until a human builds one. The quadrants are not difficulty bands. They are a map of oracle availability.

Plot the five tickets:

**Bottom-left (bounded + mechanical) — the strong quadrant.** Tickets (1) and (2) land here. Cheap oracle, loop closes, agent strong.

**Top-right (open-ended + judgment) — the human-gate-first quadrant.** Tickets (3) and (5) land here. No oracle; the agent's output is unfalsifiable until a human supplies criteria.

**The mixed cells.** Ticket (4) is bounded-but-judgment-laden — the oracle exists, but localization is hard. A task like "design a caching layer for this service" is mechanical-in-the-end but open-ended in the middle. These are where agents stumble unpredictably.

The connection to the book's thesis is now exact. *An agent is only as good as the verification loop you build around the task.* Classifying a task is just asking where its loop closes. In the strong quadrant the loop is free; in the human-gate quadrant there is no loop until you make one.

### A misconception worth killing: harder means bigger

The instinct is to rank tasks by size. More files, more lines, more moving parts: harder. It is a reasonable prior and it is wrong often enough to be dangerous.

The counter-evidence comes from benchmark re-analysis of SWE-bench Verified. Practitioner work splitting those tasks into difficulty bands and asking what makes the hard ones hard finds that the dominant factor is frequently **specification quality, not size** — multi-file coordination and under-specified issues, not line count. A two-line change against an under-specified issue can defeat an agent that would cleanly land a hundred-line change against a crisp test. Why? Because size affects how *much* work the loop does; specification affects *whether the loop exists.* A small task with no oracle is harder for an agent than a large task with a cheap one. The axis that predicts failure is oracle availability, and oracle availability is mostly orthogonal to size.

---

## Grounding the taxonomy in a measured difficulty distribution

A taxonomy you invented is a taxonomy you can talk yourself out of. So hang it on numbers someone else collected.

**SWE-bench** (Jimenez et al., ICLR 2024) is the field's defining task landscape: 2,294 real issue-to-pull-request tasks drawn from 12 popular Python repositories, where the model must edit a whole repository given an issue description and is graded by the repository's own hidden test suite. Its central, load-bearing finding for this chapter is that resolving a real issue frequently requires understanding and coordinating changes across multiple functions, classes, and even files. That sentence is the empirical death of the snippet-scale mental model. Real software engineering is **repository-scale**, and the dominant sub-skill is *localization* — finding where in a large codebase the change belongs — which is exactly the pressure the taxonomy's axes try to capture.

This matters because the benchmarks that came before flattered everyone. Function-level benchmarks like HumanEval present a self-contained function signature and a docstring; correctness is local and the oracle is trivially present. The unit of evaluation then moved — function (HumanEval) → repository issue (SWE-bench) → human-validated repository issue (SWE-bench Verified, August 2024) — and each step tightened the ground-truth oracle and *lowered* apparent capability, because each step added the localization-and-coordination work that snippet benchmarks hid. The lesson for your own estimates: a model's HumanEval-style score tells you almost nothing about whether it will resolve a repository-scale issue in your codebase.

![Three stages left to right with growing scope: a self-contained function, a whole-repository issue, then a human-validated repository issue. As the evaluation unit widened, the ground-truth oracle tightened. A tapering arrow underneath shows apparent capability falling across the three stages.](../images/03-taxonomy-of-agentic-coding-tasks-fig-03.png)
![The tightening unit of evaluation: as the benchmark unit moved from function to repository issue to human-validated issue, the oracle tightened and apparent capability fell.](images/03-taxonomy-of-agentic-coding-tasks-fig-03.png)
*Figure 3.3 — The tightening unit of evaluation: as the benchmark unit moved from function to repository issue to human-validated issue, the oracle tightened and apparent capability fell.*

**SWE-bench Verified** is a human-curated 500-task subset whose annotators graded each task by estimated developer time — a ready-made, externally validated difficulty rubric with four bands: trivial / small / substantial / esoteric. The reported shape is roughly 39% trivial and about 52% small, putting on the order of 91% of tasks under an hour, with only a handful — roughly three — exceeding four hours [verify exact figures against the primary OpenAI post before printing]. Two things fall out of this distribution, pulling in opposite directions.

First, it is *reassuring* for the strong quadrant: most real, curated tasks are short and bounded, which is the agent-friendly profile. Second, it is *cautionary*: the long tail — the esoteric, multi-hour tasks — clusters in the open-ended/judgment quadrant, where agents reliably underperform and the human gate becomes non-negotiable. The annotator time-grade is a usable proxy for "how far up-and-right on the 2×2," and the tail is where oracle availability runs out.

![A four-bar chart with the y-axis from zero showing the share of curated repository tasks by estimated time. About 39 percent are trivial and 52 percent small; together over 90 percent take under an hour. A substantial band is smaller, and the esoteric long tail over four hours is a tiny bar, emphasized to mark where agents underperform.](../images/03-taxonomy-of-agentic-coding-tasks-fig-02.png)
![The SWE-bench Verified difficulty distribution: most curated tasks are short and bounded, with only a tiny long tail exceeding several hours.](images/03-taxonomy-of-agentic-coding-tasks-fig-02.png)
*Figure 3.2 — The SWE-bench Verified difficulty distribution: most curated tasks are short and bounded, with only a tiny long tail exceeding several hours.*

> **A figure to draw on your own backlog.** X-axis: bounded (left) to open-ended (right). Y-axis: mechanical (bottom) to judgment-laden (top). Shade the bottom-left green and the top-right red. Plot your last ten tickets as points — squares for greenfield, circles for brownfield. Draw one diagonal arrow from top-right toward bottom-left and label it *"drag the task left and down by building an oracle."* That arrow is the bridge out of this chapter.

---

## The greenfield/brownfield overlay

The two axes capture oracle availability. A third distinction — **greenfield versus brownfield** — does not add a new axis so much as it changes *which failure mode* a quadrant produces.

![Two identical 2x2 grids. The left grid uses square markers for greenfield, with the top-right marker flagged for a wrong-scope failure. The right grid uses circle markers for brownfield, the top-right marker carrying a double ring for wrong-scope plus collateral damage. The bottom-left strong cell is shaded in both, unchanged by the overlay.](../images/03-taxonomy-of-agentic-coding-tasks-fig-04.png)
![The greenfield/brownfield overlay rotates the failure mode within each quadrant rather than adding a new axis.](images/03-taxonomy-of-agentic-coding-tasks-fig-04.png)
*Figure 3.4 — The greenfield/brownfield overlay rotates the failure mode within each quadrant rather than adding a new axis.*

Greenfield means new code with no legacy entanglement: a fresh service, a new module, a from-scratch script. Brownfield means changing existing code inside a living system with its conventions, its hidden coupling, its existing test suite, and its accumulated decisions. SWE-bench is almost entirely brownfield, which is itself a gap worth naming: greenfield agentic success has far less rigorous evidence than brownfield issue-resolution, because no widely used benchmark measures it the way SWE-bench measures the brownfield case.

Watch how this overlay rotates the failure mode within each quadrant.

**Greenfield + bounded + mechanical** is the strongest possible cell. Clear oracle, no legacy to break, nothing to coordinate against. When agents look magical, this is usually the cell.

**Brownfield + bounded + mechanical** is still strong, but the oracle now has to guard against breaking unrelated things. The behavior-preservation check has to cover the whole affected surface, which is why scope discipline becomes load-bearing — the Chapter 5 territory.

**Greenfield + open-ended**: the failure is **plausible wrong scope.** The agent fills the spec's gaps confidently and builds a coherent, working, *wrong* thing. The code isn't broken; the *decision* is, and no compiler catches a decision.

**Brownfield + open-ended/judgment**: the hardest cell. No oracle, plus a living system whose existing behavior the agent might silently change while chasing the unstated goal. Failure is *contested success* — you cannot even agree on whether it worked.

The pattern is worth stating plainly: greenfield open-ended failures are wrong-scope; brownfield open-ended failures are wrong-scope plus collateral damage. The cell where agents reliably win is the same regardless of greenfield/brownfield: bounded and mechanical, where the oracle is cheap.

---

## The interface confound: when "too hard" is really "loop too weak"

There is a trap in this whole enterprise, and it has a name in the literature. When you say "the agent fails at this task," you are making a claim about the task. But the same task can move across the success/fail boundary *without the task changing at all* — just by improving the loop around the model.

This is the central finding of **SWE-agent** (Yang et al., NeurIPS 2024): a purpose-built agent-computer interface — a file viewer scoped to readable chunks, an editor that lints on every edit, a search tool, a guarded way to run tests — raised issue-resolution sharply above non-interactive baselines on the *same* tasks. The model didn't get smarter. The interface let it search, edit safely, and observe results. Tasks that were "too hard" became tractable because the loop got better at feeding the model ground truth.

The consequence for your taxonomy is a discipline: **before you file a task in the failure quadrant, ask whether you've given the agent a real loop.** Can it run your tests? Can it see the error output? Can it search the repo rather than guessing at paths? A surprising amount of apparent task hardness is loop inadequacy — and loop inadequacy is fixable by you, which is the better problem to have.

The honest version of the taxonomy carries an asterisk: a task's quadrant is a function of the task *and the loop you've built*, and the two are hard to disentangle with current data. This interface-versus-task confound is an open research gap, not a solved measurement.

---

## Brooks's spine: essence, accident, and the failure boundary

The taxonomy has a 40-year-old intellectual spine. Naming it makes the failure boundary feel less like a property of today's models and more like a property of software itself.

In *No Silver Bullet* (1987), Frederick Brooks split the difficulty of software into two kinds. **Accidental complexity** is the difficulty of the implementation — the boilerplate, the language ceremony, the mechanical edits, the wiring that the problem doesn't inherently require but the tools impose. **Essential complexity** is the difficulty of *deciding what the system should do* — modeling the domain, resolving conflicting requirements, choosing among defensible designs. Brooks's claim was that essential complexity is irreducible: no tool removes the hard part of deciding, only the hard part of typing.

Map that onto the 2×2 and it lines up almost exactly. Agents excel at accidental complexity — the bottom-left quadrant is accidental complexity with a cheap oracle. Agents struggle with essential complexity — the top-right quadrant is essential complexity with no oracle, because the oracle would have to encode the decision you haven't made. A 1987 essay predicts the 2026 failure boundary because the boundary was never about the model. It is about whether the hard part of the task is *typing* (which agents now do well) or *deciding* (which they don't, and can't, because deciding is where the missing oracle lives).

Mary Shaw's framing of well-structured versus ill-structured engineering problems maps directly onto bounded↔open-ended: an agent can engineer a well-structured problem because the structure supplies the oracle; an ill-structured problem has to be structured by a human first. Both say the same thing the 2×2 says: the agent's strong quadrant is the part of software that was already structured enough to be checkable.

---

## What's still genuinely unsettled

It would be dishonest to present this taxonomy as more validated than it is.

The *structure* — oracle availability predicts the failure mode — is well supported by the benchmark trajectory and Brooks's essence/accident frame. The *quantitative predictors* are not. There is no validated, repository-agnostic predictor of agentic-task success. Size, specification quality, and oracle availability all correlate with failure, but none is established as the causal or dominant driver.

Benchmark-to-your-repo transfer is unquantified: a high SWE-bench Verified resolution rate does not forecast success on your codebase with its idiosyncratic conventions and weaker test suite. The greenfield axis is under-measured. The interface-versus-task confound is unresolved.

The honest operating posture: use the taxonomy to *predict and then check*, never to *label and trust.* Sort the task, predict the failure mode, build the loop, run it, and update on what actually happened. The taxonomy earns its keep by making your prediction falsifiable — which is the habit the whole book is trying to build.

And the one number to treat with the most suspicion: any leaderboard resolution rate. All benchmark scores are dated snapshots. Through 2025, Verified scores inflated to the point where the benchmark stopped separating frontier systems — which is itself a signal that the *taxonomy of remaining-hard tasks* now matters more than the aggregate number. When the score saturates, the only interesting question left is *which tasks are still hard, and why* — and that question is this chapter.

---

## LLM Exercises

**Exercise 3.1 — Analyze.** Take the ten most recent tickets from a real backlog you have access to. For each, write the answer to the single question "when the agent says it's done, what tells me it's right?" Then place each on the 2×2 and predict its dominant failure mode (broken code / plausible-wrong-scope / contested-success / clean) *before* running anything. Mark greenfield tickets as squares and brownfield as circles. Keep this sheet — a later exercise will ask you to run two of these and check your predictions.

**Exercise 3.2 — Analyze.** Find one task in your backlog that is *small but ill-suited* to an agent (few lines, but no oracle) and one that is *large but well-suited* (many lines, but a crisp test or behavior-preservation check). For each, write two sentences: where size would have misled you, and where oracle availability gives the correct prediction.

**Exercise 3.3 — Analyze → Evaluate.** Take a task you classified as a failure-quadrant task in Exercise 3.1 and run it with your agent under two loops: (a) a weak loop — no test access, you paste errors manually; (b) a strong loop — the agent can run the suite and see output itself, per Chapter 2. Report whether the task moved quadrants — whether the "failure" was task hardness or loop inadequacy. One paragraph, with the transcripts attached.

**Exercise 3.4 — Create, produce something.** Write a one-page **task-triage rubric** for your team: a checklist that, given a ticket, outputs (i) its quadrant, (ii) the predicted failure mode, (iii) whether a cheap oracle exists or must be built, and (iv) the recommended human gate. The rubric must be usable by someone who has not read this chapter. Test it by handing it, plus three unsorted tickets, to a colleague and checking whether their sort matches yours; revise where it doesn't.

---

## What would change my mind

A controlled study that fixed the model and the loop, held task size constant, and showed that specification quality did *not* predict agentic success — that under-specified and well-specified tasks of the same size resolved at statistically indistinguishable rates — would break the central claim of this chapter. The taxonomy stakes everything on oracle availability being the dominant predictor of failure. If size, or repository familiarity, or some unmeasured factor turned out to dominate while specification did not, the 2×2 would still be a useful sorting heuristic but a false causal story, and the Chapter 4 move — drag the task left by building an oracle — would lose its justification.

Separately, a repository-agnostic predictor of agentic success — a measurable feature of a task that forecasts resolution across codebases — would strengthen the chapter by replacing the qualitative 2×2 with a calibrated score. I'd welcome that and retire the hand-drawn axes the moment it exists.

---

## Still puzzling

**Is oracle availability really one thing, or two?** I've folded "a stopping condition exists" and "the condition is machine-checkable" into a single underlying question. But a task can be bounded-yet-uncheckable and unbounded-yet-mechanical-once-bounded. Whether these are one axis seen twice or two genuinely independent things is unresolved, and it matters for whether the 2×2 is the right shape.

**How much of "greenfield is easier" survives contact with real systems?** Greenfield looks like the strong cell because there's no legacy to break — but greenfield also has the least pre-existing oracle (no test suite, no behavior to preserve), so the agent's wrong-scope failure is most dangerous exactly where it looks safest.

**Does the taxonomy decay as models improve?** If frontier models get better at structuring ill-structured problems themselves — turning open-ended into bounded without a human — the human-gate quadrant could shrink. Or better models invite us to hand them ever-more-open-ended work, keeping the failure boundary fixed in relative terms even as it moves in absolute terms. Which trend wins is unclear.

**Can quadrant placement be made adversarially robust?** A task that *looks* bounded-mechanical but whose test is weak or gameable is misclassified — it will sit in the green quadrant and fail. The existence of the oracle isn't enough; the oracle has to be *good.* That gap is the entire subject of the next chapter.

---

## References

- Jimenez, C. E., Yang, J., Wettig, A., Yao, S., Pei, K., Press, O., & Narasimhan, K. (2024). *SWE-bench: Can Language Models Resolve Real-World GitHub Issues?* ICLR 2024. arXiv:2310.06770.
- OpenAI & the SWE-bench authors (2024). *Introducing SWE-bench Verified.* openai.com, August 2024. [Verify the 39% / 52% / ~91% / ~three-over-four-hours figures against the primary post.]
- Yang, J., Jimenez, C. E., Wettig, A., Lieret, K., Yao, S., Narasimhan, K., & Press, O. (2024). *SWE-agent: Agent-Computer Interfaces Enable Automated Software Engineering.* NeurIPS 2024. arXiv:2405.15793.
- Fowler, M. (2018). *Refactoring: Improving the Design of Existing Code*, 2nd ed. Addison-Wesley.
- Brooks, F. P. (1987). *No Silver Bullet: Essence and Accidents of Software Engineering.* *Computer* 20(4), IEEE.
- Ganhotra, J. (2025). *Cracking the Code: How Difficult Are SWE-Bench-Verified Tasks Really?* jatinganhotra.dev. — practitioner re-analysis; non-peer-reviewed, cite as documented case, not foundation.
- Epoch AI (2024–2025). *What skills does SWE-bench Verified evaluate?* epoch.ai. — independent skill-decomposition. Research-org analysis; cite as corroboration.

---

**Tags:** agentic-coding, task-taxonomy, oracle-availability, bounded-vs-open-ended, mechanical-vs-judgment, greenfield-brownfield, swe-bench, swe-bench-verified, swe-agent, essence-vs-accident, failure-mode-prediction
