# Chapter 6 — Debugging with Agents

*Reproduce → localize → fix → verify, with external grounding instead of introspection*

---

Start with a result that should bother you.

Huang et al. (2024), "Large Language Models Cannot Self-Correct Reasoning Yet," tested a natural idea: take an LLM's answer to a reasoning problem, ask it to review its own reasoning and correct any mistakes — with no external feedback, no oracle telling it when to stop. Pure introspection. "Check your work."

It does not help. Performance on reasoning tasks does not improve and **frequently degrades** after a self-correction pass. Earlier work had seemed to show gains, but the paper explains why: those gains came from feedback that quietly smuggled in ground truth — for instance, an oracle stopping criterion that only let the model "stop correcting" once it had reached the right answer. Strip the oracle out, and the self-correction loop stops working.

Sit with how strange this is. We expect that asking a smart system to reconsider should, on average, help. For a human reviewer it usually does. For an ungrounded LLM it does not, because the same process that produced the first answer produces the "correction" — there is no independent vantage point. The model re-derives with equal confidence and is as likely to talk itself *out* of a correct answer as into one. Introspection without an external check is not error-correction. It is resampling with extra steps.

Now look at what this means for debugging, because the naive debugging instinct is *precisely* the thing Huang showed fails. "There's a bug — agent, look at your code again and find the mistake." That is intrinsic self-correction. If you debug by asking the agent to re-read its reasoning and try harder, you are running the exact loop that does not converge and can regress.

The fix is not a smarter model or a cleverer "really check this time" prompt. The fix is to stop asking the model to introspect and start giving it **external ground truth**. And here is the lucky fact about code: that ground truth is mechanical and cheap. You can run the program. You can read the actual stack trace. You can run the failing test and watch it stay red, then watch it turn green. The compiler is an oracle. The test runner is an oracle. Unlike the reasoning tasks in Huang's study, debugging comes with a free, external, non-fakeable signal — and that is what makes the loop close.

![Two panels. Left, ungrounded: a model node loops on itself and arrives at a regression outcome. Right, grounded: the same model node sends a round-trip signal to an external execution node and arrives at a convergence outcome. The only difference is the external execution node and its signal.](../images/06-debugging-with-agents-fig-02.png)
![Ungrounded self-correction resamples from the same process and can regress; wiring in an external execution signal lets the loop converge.](images/06-debugging-with-agents-fig-02.png)
*Figure 6.1 — Ungrounded self-correction resamples from the same process and can regress; wiring in an external execution signal lets the loop converge.*

Chen et al. (2023), "Teaching Large Language Models to Self-Debug," is the constructive half of the same picture. Their Self-Debugging method has the model debug its own program, partly by explaining its code in natural language — but the gains are largest precisely when the model is given **execution feedback**: run the code, see the result, debug against the real output. Same lesson from the positive side: introspection alone is weak; introspection *plus the run* works.

The engineering move for the rest of the chapter follows directly: **never debug by introspection; always debug against an external signal the model cannot fake.** Everything below is the structure that wires that signal in at every stage.

---

## The loop: four stages, four artifacts

The durable structure of debugging — agentic or human — is a four-stage loop, and the reason to name the stages is that each one must produce a *concrete artifact* you can point at. If a stage hasn't produced its artifact, you haven't done it; you've skipped to opinion.

![A four-stage ring: reproduce, localize, fix, verify. From verify, three branches: a clean exit on green, a red-retry arrow back to localize, and a branch to a hexagonal human gate when green but suspicious. An external test-runner marker feeds a ground-truth signal into verify from outside the loop.](../images/06-debugging-with-agents-fig-01.png)
![Debugging runs a four-stage loop — reproduce, localize, fix, verify — with verify branching back on red and out to a human gate.](images/06-debugging-with-agents-fig-01.png)
*Figure 6.2 — Debugging runs a four-stage loop — reproduce, localize, fix, verify — with verify branching back on red and out to a human gate.*

**Reproduce.** The first job is not to theorize about the bug. It is to make it happen on command. The artifact is a failing reproduction: a failing test, a script that triggers the crash, a stack trace you can regenerate. This mirrors TDD's red from Chapter 4 exactly — the bug, expressed as a thing that fails. An agent that starts proposing fixes before it can reproduce is theorizing, and you should stop it. No reproduction, no fix.

**Localize.** Given a reproduction, narrow *where* the defect lives. The artifact is a suspect region: a ranked set of lines, a module, a stack frame. For a crash, the stack trace is often a near-exact localizer for free. For a logic bug, you need more — which is where the next section's machinery comes in.

**Fix.** Propose a change. The artifact is a candidate diff. This is the only stage where the model's generation is doing the creative work — and it is the stage you trust *least* without verification, because of the Huang result.

**Verify.** Re-run the reproduction and the full suite. The artifact is the result: green means the bug, as reproduced, is gone; red means loop back to localize with new information. This is where the external grounding lives. The whole point of the loop is that verify is grounded in the run, not in the model's belief that it fixed it.

The loop's discipline is that you don't let the agent collapse the stages. The most common collapse is jumping from a glance at the code straight to a fix — skipping reproduce and localize. That is introspection wearing a hurry. The four artifacts are the checklist that keeps the agent honest: show me the failing reproduction, show me the suspect region, show me the diff, show me the green re-run. Each is a thing, not a claim.

There is a side-branch off verify you must hold in your head from the start: **green but suspicious → human gate.** Sometimes the suite goes green and the fix is still wrong (more on this in a moment). The loop's normal exit is "green," but green is necessary, not sufficient.

---

## Localization: a 20-year-old idea, now agent context

Localization has a real history, and the agent's biggest localization risk is a specific, nameable mistake.

![A vertical stack of five call frames. The top frame is the symptom site where the failure surfaced. The frame three positions down is the true origin where the bad value started. A bad value propagates upward to the symptom; reading upstream leads down to the origin. Fixing the top frame only suppresses the symptom.](../images/06-debugging-with-agents-fig-03.png)
![A crash surfaces at the top stack frame, but the defect's origin is often several frames upstream, so fixing the top frame suppresses the symptom.](images/06-debugging-with-agents-fig-03.png)
*Figure 6.3 — A crash surfaces at the top stack frame, but the defect's origin is often several frames upstream, so fixing the top frame suppresses the symptom.*

**The free localizer: the stack trace.** For a crash-class bug, the stack trace is an exact, no-cost localizer. The exception was thrown *here*, called from *here*, called from *here*. Feed it to the agent verbatim — the whole trace, not a paraphrase. But here is the trap, and it is the agent's characteristic localization error: **treating the top frame as the cause.** The top frame is where the program *noticed* the problem, which is often downstream of where the problem *was introduced*. A null dereference crashes in the function that dereferences, but the null was produced three frames up by a lookup that should have errored and didn't. An agent that "fixes" the top frame — adds a null check at the crash site — has suppressed the symptom and left the cause. The discipline: read the *whole* trace and ask "where did the bad value originate?" not "where did it blow up?"

**The historical localizer: spectrum-based fault localization.** Before LLMs, the field already had a method for "let the tests tell you where the bug is." Tarantula (Jones & Harrold, ASE 2005) ranks each statement by a suspiciousness score derived from how often it appears in *failing* test executions versus *passing* ones. A line that runs in every failing test and no passing test is highly suspicious; a line that runs in everything is not. The output is a heat map over the source: the failing-vs-passing spectrum points at the defect. This is twenty years old, and it is the direct ancestor of how an agent ranks suspect lines.

The modern move is not to replace spectrum-based fault localization but to **repurpose it as agent context.** AutoCodeRover (Zhang et al., ISSTA 2024) is the clean example: an LLM agent that combines structure-aware code search with spectrum-based localization when a test suite exists, feeding the model a ranked suspect region rather than the whole codebase. The old technique narrows the search; the new model fills it in. Classical SBFL as a filter, LLM as the patcher.

When does this approach mislead? On flaky or low-coverage suites. If tests pass and fail nondeterministically, the failing-vs-passing spectrum is noise, and the suspiciousness scores point nowhere reliable. Know which kind of suite you have before you trust the heat map.

---

## Feeding Observations: traces, logs, failing tests

The agent debugs against what you put in front of it, and the rule from the Huang result says that should be *real output*, not your summary of it. Three Observation types, each with a discipline.

**The failing test, verbatim.** Paste the actual test failure — the assertion, the expected-vs-actual, the full traceback — not "the test about empty input is failing." The exact output is the external signal; your paraphrase is introspection by proxy. If the bug surfaced as a red test, the reproduction already exists. Start there.

**The stack trace, whole.** As the last section warned: the entire trace, top to bottom, because the cause is frequently *not* the top frame. Truncating to the top frame hands the agent the symptom and hides the cause.

**The logs, scoped.** Logs are the dynamic record of what actually ran. They're high-value and high-volume; feed the relevant window — the lines around the failure, the request that broke — not the entire log file, which blows the token budget and dilutes attention. Selectivity is the skill: enough log to localize, not so much that the signal drowns.

The unifying principle: the agent's first job at every stage is to reproduce and read the real Observation, not to theorize. Theorizing is generation; reading the run is grounding.

---

## The confident wrong fix

This is the failure mode the chapter exists to catch, and it is the residue the loop alone does not eliminate.

![A taxonomy tree. A green-but-wrong fix branches into two failure shapes: symptom suppression, where the suppressed symptom returns, and test-overfit, where the exact input is special-cased. A genuinely correct fix stands to the side as the contrast both failures are measured against.](../images/06-debugging-with-agents-fig-04.png)
![A green-but-wrong patch fails in two ways — symptom suppression and test-overfit — both of which pass the loop's normal exit.](images/06-debugging-with-agents-fig-04.png)
*Figure 6.4 — A green-but-wrong patch fails in two ways — symptom suppression and test-overfit — both of which pass the loop's normal exit.*

Picture it. The agent reproduces the bug (red test), localizes (a plausible suspect region), proposes a fix, re-runs — **green.** The loop says exit. And the fix is *wrong.* It turned the one test green by handling that one case while breaking a case the suite doesn't cover, or by suppressing the symptom instead of fixing the cause, or by special-casing the exact input the test uses. The patch is plausible, the prose explaining it is confident, the suite is green. Everything the loop checks is satisfied, and the code is wrong.

The reason the confident prose is worthless as evidence is the same reason it was worthless in §6.1: it is generated by the same process that generated the fix. Its assurance is a property of the text, not of the code. The same dynamic that makes "the tests should now pass" a wish rather than a result makes "here is why this fix is correct" a story rather than a proof. Stop being surprised by it.

Two failure shapes worth naming separately.

**Symptom suppression.** The null check at the crash site that makes the crash stop without addressing why the null appeared. The bug "goes away" and returns in a different costume.

**Test-overfit.** The patch special-cases the test's specific input — `if input == "the exact test string": return expected` in spirit if not in letter. Green, useless.

This is the exact analog of Chapter 5's unsafe refactor — the model proposes confidently and cannot independently validate that the fix is *correct* rather than merely *test-passing.* Survey work on bug-fixing agents documents this directly: patches that pass narrow tests while being semantically wrong or overfit to the test.

The guard is one you already met in TDD and refactoring: **reproduce-first, with a new failing test that pins the real behavior before any fix is accepted.** For a logic bug that passes the existing suite — the worst case — the move is: *write a new failing test that captures the correct behavior first*, then fix against it. No new red test, no fix. This forces the agent to satisfy the actual requirement, not the accident of which inputs the old suite happened to check.

But even reproduce-first has a hole. The confident-wrong-fix's natural habitat is exactly the region *no existing test catches* — and a sufficiently subtle wrong fix can pass even a thoughtfully written new test, because you wrote the test from the same flawed understanding that let the bug through. This is the "green but suspicious" branch off verify. The honest engineering posture: a green suite *lowers* the probability of a wrong fix but does not drive it to zero, and for high-stakes fixes — security, money, data integrity — a human must inspect *intent*, not just outcome.

The Heisenbug deserves a final warning, because it is where the agent is most dangerous. A flaky or timing-dependent bug misleads spectrum-based localization (the spectrum is noise) and single-run traces (the next run differs). An agent will *confidently* "fix" a timing bug by suppressing its symptom — add a retry, widen a timeout, swallow the race — and report green, because on this run it passed. Flakiness is the signal that the loop's normal grounding is itself unreliable, and you cannot ground a fix on a signal that won't hold still. The human gate matters most precisely here.

One misconception to kill: "The test passed, so the bug is fixed." The test passing means *that test's* condition is satisfied. The confident-wrong-fix passes the test and is wrong anyway — by overfitting to it or by suppressing a symptom. Green is the loop's necessary exit condition, not its sufficient one. Reproduce-first with a new test raises the bar; the human gate on intent is what's left for the cases no test reaches.

---

## Bridge to greenfield builds

Debugging closed the same loop refactoring did, aimed the other way: you start red and earn green, with the test runner and the stack trace as the external ground truth that ungrounded introspection can't provide. Huang's negative result is the spine — "check your work" alone regresses — and the cure is mechanical grounding the model can't fake, plus a human gate for the wrong-fix residue no test catches.

So far every task in Act Two has had something to push against: a failing test, an existing behavior to preserve, a bug to reproduce. The ground truth was *given*. Greenfield is the hard case, Chapter 7's case: there is no existing code, no failing test handed to you, no behavior to preserve — you are building from a specification into empty space. The question becomes: when nothing exists yet to verify against, how do you manufacture the verification loop *as you build*, so the agent is never generating into a vacuum? That is plan-then-execute at project scale, and it is where the loop-first habit gets its sternest test.

---

## LLM Exercises

**Exercise 6.1 — Analyze.** You are given three failures (in the course repo, `exercises/ch06/`): (A) a stack trace ending in a null dereference, where the null originated three frames upstream; (B) a flaky test that fails roughly 1 run in 5; (C) a logic bug where the existing suite is fully green but a user-reported behavior is wrong. For each, identify the first loop stage that's hard, name the artifact that stage must produce, and state the specific way an agent is likely to go wrong — top-frame fixation, symptom suppression, or test-overfit. For (C), write the new failing test you'd require before any fix.

**Exercise 6.2 — Apply.** Take a real bug (or a seeded one in the course fork) and drive the full reproduce → localize → fix → verify loop with an agent, feeding it the *verbatim* failing test and whole stack trace as Observations. Keep a four-artifact log: the failing reproduction, the suspect region, the candidate diff, the green re-run. Then re-run the same bug a second way — by asking the agent to "look at the code and find the bug" with no reproduction first — and report the difference in outcome and in how many wrong turns each took.

**Exercise 6.3 — Create, produce something.** Build a confident-wrong-fix and then catch it. Write a small function with a bug and a thin test suite that the bug *passes*. Have the agent "fix" something nearby in a way that keeps the suite green but is semantically wrong — symptom suppression or test-overfit, your choice. Confirm the suite stays green. Then produce the **smallest new failing test** that exposes the wrong fix, and re-run the loop reproduce-first. Append a paragraph: what did the new test pin that the old suite missed, and is there a residual wrong-fix this test *still* wouldn't catch?

**Exercise 6.4 — Evaluate (optional).** On a repo with a real test suite, compare two localization strategies for the same bug: (a) hand the agent a spectrum-based suspect ranking, vs. (b) let the agent search the codebase itself. Measure time-to-localize and whether each landed on the true defect site versus a downstream symptom. Then deliberately introduce flakiness and rerun: report how much the SBFL signal degrades and at what flakiness rate you'd stop trusting it.

---

## What would change my mind

The chapter's spine is Huang et al.'s negative result: ungrounded self-correction does not converge and can regress. A controlled study showing that frontier models *can* reliably self-correct reasoning intrinsically — improving, not degrading, with no external feedback and no oracle stopping criterion, across a representative task set — would force a re-specification. It would mean the agent's introspection is itself a usable grounding signal, and the strict "never debug by introspection, always against the run" rule would soften toward "introspection is a real first pass, the run is the confirmation." I'd want it demonstrated with the oracle genuinely removed — Huang's central methodological point was that earlier gains leaked ground truth — on reasoning tasks, not just on code where the run is available anyway.

Separately, a verify-time check that reliably catches the confident-wrong-fix *without* a human — mutation testing or property-based testing driving the wrong-fix rate near zero — would shrink the human gate this chapter insists on. That's promising and under-measured in the agent setting, and it is the falsifiable form of "some checks no test catches need a human."

---

## Still puzzling

**How strong must the external check be to catch a confident-wrong-fix that passes the existing suite?** Reproduce-first with a new test raises the bar, but a subtle wrong fix can pass a test written from the same flawed understanding. There is no settled answer to how much oracle is "enough," and the human gate is where we put the residue rather than a measured threshold.

**Does mutation or property-based testing as the verify signal actually reduce overfit patches?** The intuition is yes — a patch that overfits one test should fail a mutant or a property — but this is under-measured in the agent setting. If it holds, it shrinks the human gate; if it doesn't, the gate stays.

**How reliable is spectrum-based fault localization as agent context under real flaky or low-coverage suites?** SBFL is a strong prior on a healthy suite and noise on a flaky one. Whether to feed ranked suspects or let the agent search is unsettled, and the crossover — how flaky is too flaky — isn't characterized.

**What's the true cost and value of running the full loop in CI?** Running reproduce→localize→fix→verify automatically per failure costs compute and latency; the value is bugs caught earlier. The trade depends on current tooling and dates fast. Nobody has a clean break-even.

---

## References

- Huang, J., Chen, X., Mishra, S., Zheng, H. S., Yu, A. W., Song, X., & Zhou, D. (2024). *Large Language Models Cannot Self-Correct Reasoning Yet.* ICLR 2024. arXiv:2310.01798.
- Chen, X., Lin, M., Schärli, N., & Zhou, D. (2023). *Teaching Large Language Models to Self-Debug.* ICLR 2024. arXiv:2304.05128.
- Jones, J. A., & Harrold, M. J. (2005). *Empirical Evaluation of the Tarantula Automatic Fault-Localization Technique.* ASE 2005. ACM DOI 10.1145/1101908.1101949.
- Zhang, Y., Ruan, H., Fan, Z., & Roychoudhury, A. (2024). *AutoCodeRover: Autonomous Program Improvement.* ISSTA 2024. arXiv:2404.05427.
- *LLM-based Agents for Automated Bug Fixing: How Far Are We?* arXiv:2411.10213 [preprint]. — documents the plausible-but-wrong / test-overfit patch failure mode.

---

**Tags:** debugging, reproduce-localize-fix-verify, self-debug, self-correction, external-grounding, stack-trace, spectrum-based-fault-localization, tarantula, autocoderover, confident-wrong-fix, test-overfit, human-gate
