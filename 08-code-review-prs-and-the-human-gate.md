# Chapter 8 — Code Review, Pull Requests, and the Human Gate

*Routing the mechanical to the loop and the judgment-laden to the human who owns the merge*

Here is a finding that should reset your priors, and I want to start with it rather than bury it in a survey of techniques.

Pornprasit and Tantithamthavorn spent the better part of a research program trying to get an LLM to do code review well — not as a gimmick, but systematically, comparing zero-shot prompting, few-shot prompting, and fine-tuning across tasks like generating the review comment and refining the code. The result was published in *Information and Software Technology* (Elsevier, 2024), a refereed journal, not a preprint. Fine-tuning a GPT-3.5-era model outperformed the prompting approaches. That part you might have guessed. But here is the detail that turns this from a tuning note into a lesson: adding a *persona* to the prompt — the "you are an expert senior reviewer" move that prompt-engineering folklore tells you to reach for — *reduced* exact-match accuracy. The thing everyone does made it worse.

Sit with that for a moment. Most of this book is about prompting. The verification loop we've built, the task decomposition, the context discipline — those are prompting levers, and they pay off, because they're the right levers for *driving* an agent to produce code. But review is different. Review is not a generation task dressed in evaluation clothes. It's a judgment task, and it turns out you cannot prompt your way to a good machine reviewer; the quality of automated review is not principally a prompting problem.

So the question that drives this chapter is not "how do I prompt a better reviewer?" The question is: given that automated review has these limits, *what is the human reviewer actually for?* The naive answer — catching bugs — turns out to be wrong, and the wrongness is the whole chapter.

<!-- → [INFOGRAPHIC: Two-lane routing diagram — left lane labeled "Mechanical" (test/type/lint/SAST), right lane labeled "Human" (intent/design/context-security), with a merge gate at bottom owned by a human figure; arrows show every concern entering from the top and routing to exactly one lane] -->

---

## What review actually delivers

Bacchelli and Bird asked developers what code review was *for*, then tracked what it actually *delivered* — and the two lists don't match. The study is from ICSE 2013, and its central finding is still the most clarifying thing in the modern-code-review literature: what developers say review is for is finding defects. What review actually delivers is knowledge transfer, design discussion, and judgment about intent and maintainability. Defect-catching — the thing everyone names first — is the thing automated tools already do better.

This matters because it means the human gate has never been a slower, more expensive bug-finder. It exists for the class of judgments that tests, types, linters, and static analyzers *structurally cannot make*. Tools find what they were written to find. The human finds what no tool was written to find. Those are different jobs, and the mistake of treating review as "one undifferentiated act of looking at the code and feeling whether it's okay" is the mistake that makes review feel optional when CI is green.

Which brings me to the argument at the center of this chapter. It's short and it's load-bearing, so I want to state it precisely.

An agent that produces a diff has been optimizing toward a target. In this book, that target is the verification loop: pass the tests, satisfy the types, clear the linter, get CI green. That's the whole point of the loop — it gives the agent something to close against. But the consequence is mechanical and slightly unsettling: an agent that passes every checkable signal has, by construction, optimized *exactly* to those signals. Whatever the checks don't measure, the agent had no pressure to get right. The residual risk doesn't spread evenly across the diff. It *concentrates* in the dimensions the loop doesn't cover.

![A diff split into six bands. The three checked dimensions — tests, types, lint and SAST — are de-risked and nearly empty. The three unchecked dimensions — intent, design, context-security — are densely packed with residual risk.](../images/08-code-review-prs-and-the-human-gate-fig-02.png)
![An agent optimizes to the checkable signals, so residual risk pools in the three uncovered dimensions — intent, design, and context-security.](images/08-code-review-prs-and-the-human-gate-fig-02.png)
*Figure 8.1 — An agent optimizes to the checkable signals, so residual risk pools in the three uncovered dimensions — intent, design, and context-security.*

There are three of those dimensions, and they have names.

**Intent.** Does the diff do the *right* thing — what the spec or the business rule actually requires — as opposed to *a* thing that passes the tests? This is Chapter 6's confident-wrong-fix arriving at the gate in a green suit. A test checks whether the output matches an expected value. It cannot check whether that value is the one the business needed, because that's a fact about the world the test was written from, not an independent oracle. The suite cannot refute itself.

**Design and maintainability.** Naming, structure, scope creep, future cost. A diff can pass every test while introducing a duplicated abstraction, a leaky module boundary, or a change three times larger than the task required. No test fails on bad naming. The cost of design debt is paid later, by whoever maintains it, and the agent has no signal for that.

**Context-sensitive security.** Injection, authentication and logic bypass, secrets in wrong places, unsafe deserialization — the class of flaws that only make sense *given the system*. A functional test suite passes right over a SQL injection vulnerability if the happy-path test uses benign input, because injection is not a functional failure. It's a failure of a property the tests don't assert. The agent optimized to the tests and never knew the property existed.

<!-- → [TABLE: Three residue classes — columns: Class, What it means, Why no test covers it, Example agent failure; rows: Intent, Design/Maintainability, Context-Sensitive Security] -->

The green checkmark certifies the mechanical lane and nothing else. A reviewer who treats green as "reviewed" is rubber-stamping exactly the dimensions where the risk lives — and for an agent's diff, those dimensions are where the agent was *free* to be confidently wrong, because it had no signal there.

---

## The gate as a routing decision

The discipline I want to teach is not "look harder at the code." Looking harder at everything is how you miss things — attention is finite and unguided attention distributes itself randomly. The discipline is to treat the gate as a *routing decision*: every concern a diff raises goes to exactly one of two lanes, and the two lanes have different owners.

![A concern entering a diff routes to one of two lanes — a mechanical lane of automated checks (test, type, lint, SAST) or a human lane branching into intent, design, and context-security. Both converge on a single human-owned merge.](../images/08-code-review-prs-and-the-human-gate-fig-01.png)
![Each concern routes to the mechanical lane or the human lane — intent, design, context-security — and the human alone owns the terminal merge.](images/08-code-review-prs-and-the-human-gate-fig-01.png)
*Figure 8.2 — Each concern routes to the mechanical lane or the human lane — intent, design, context-security — and the human alone owns the terminal merge.*

The mechanical lane handles anything a test, type-check, linter, or static scanner covers. Don't spend human attention re-checking what the suite already certifies. That's wasted judgment, and it dulls the reviewer for the concerns that need them. If a concern *can* be moved into the mechanical lane — write the missing test, add the lint rule, wire the scanner — move it. Every concern you mechanize is one fewer the human has to hold.

The human lane gets everything the mechanical lane can't cover. Route it to one of the three branches from above — intent, design, security — because each branch has a different question and a different reference point. The intent branch checks the diff against the stated task, which is why the PR description has to *state* the intent rather than just saying "fixes the bug." The design branch asks whether someone will curse this in six months. The security branch — and I want to be specific here about why this must be a checklist rather than raw judgment.

Paul and colleagues examined 135,560 review comments in a study of security-related coding weaknesses (arXiv:2311.16396, preprint as of this writing) and found that reviewers raised security concerns across 35 of 40 weakness categories. That sounds thorough. But they also found that reviewers *under-discussed* memory and resource-management classes relative to how often those cause actual vulnerabilities. And the related work found that developers can catch context-sensitive vulnerabilities that scanners miss — injection, authentication and logic flaws — but that detection improves sharply with structured support.

Put those two findings together and the design decision is clear: unstructured human security review has *systematic blind spots*. Not random ones. Systematic ones. Which means the blind spots are recoverable — but only if the structure forces you to look at the classes you'd otherwise skip. So the security branch must be a checklist (injection, auth and logic bypass, secrets, unsafe deserialization, memory and resource management), not "look it over for anything sketchy." The checklist exists precisely because raw judgment misses classes; the structure recovers the blind spots that the unstructured review leaves open.

The terminal node of the routing is non-negotiable: **the human owns the merge.** The agent may draft the diff, draft the PR description, even draft a review. But a human makes the merge decision, and the agent never self-merges. That is the gate in one sentence. The merge is the irreversible action, and irreversible actions are where the human must stay in the loop — not as ceremony, but as the only arrangement where accountability survives into tomorrow.

---

## The structure that makes the gate work

The gate is only as good as the diff it receives. A two-thousand-line diff that does six things is unreviewable — nobody can hold the intent of six interleaved changes at once, and the routing collapses because there is no single intent to check against. So the gate's effectiveness is mostly determined *before* review starts, by the discipline of how the change is packaged.

**Small, single-purpose diffs.** One change, one reason, reviewable in one sitting. This is not aesthetics. It is what makes the intent check possible. You can ask "does this do what was asked" of a diff that does one thing. You cannot ask it of a diff that does six, because there are six asks and they're interleaved. Agents trend toward large diffs — the agent will happily refactor the whole neighborhood — so scope discipline is something you impose in the task prompt, before the diff exists.

**An intent-stating description.** The agent should author a PR description that states what the change does and what was verified — the spec or issue it satisfies, the tests it added, the checks it ran. This is not ceremony. It is the reference for the intent branch. The reviewer's first move is to confirm the diff matches the stated intent, which requires the intent to be stated. A description that says only "fixes the bug" gives the reviewer nothing to route against, and the intent check becomes a guess.

**A structured security checklist, not raw judgment.** Five items, every time: injection, auth and logic bypass, secrets, unsafe deserialization, memory and resource management. Record a finding or "n/a" for each. The study evidence says you will miss the memory and resource classes if you review unstructured. The checklist is the recovery mechanism.

**A human-owned merge.** The terminal node, stated as policy. Agents propose; humans dispose. The merge is the irreversible action.

The arrangement that falls out of all this is the operating standard for agent-assisted development: automated tooling runs the mechanical lane on every PR, an LLM reviewer may draft comments and pre-fill the checklist, but a human reads the residue branches and a human merges. Automate the lane that mechanizes; reserve the human for the lane that doesn't.

<!-- → [CHART: Bar or swimlane chart showing the hybrid workflow — x-axis: stages of PR lifecycle (diff created, CI runs, LLM reviewer drafts, human reads residue branches, human merges); y-axis or lane color: agent-owned vs. human-owned; makes visible exactly where handoff occurs and why the merge stays human] -->

<!-- → [INFOGRAPHIC: PR discipline checklist — four items as a vertical sequence: (1) small single-purpose diff, (2) intent-stating description, (3) five-item security checklist with slots for finding/n/a, (4) human-owned merge; styled as a reusable artifact, not prose] -->

---

## The history the gate comes from

None of this is new. Two people figured out the essential structure before software development as we practice it existed, and their work is worth naming because it explains *why* the structure works, not just that it does.

Michael Fagan invented the formal code inspection at IBM in 1976 — defined roles, a checklist, a logged defect list — and demonstrated empirically that it caught defects *before* testing, more reliably than testing caught them afterward. Everything the PR gate prescribes is a lightweight descendant: an author and a reviewer (roles), a security checklist, a record of what was found (review comments).

![Three elements of the 1976 Fagan inspection — defined roles, a checklist, a logged defect list — each map one-to-one by horizontal arrows onto modern PR counterparts: author/reviewer roles, a security checklist, and review comments.](../images/08-code-review-prs-and-the-human-gate-fig-03.png)
![Each element of the 1976 Fagan inspection — roles, checklist, defect log — maps onto a lightweight counterpart in agent-authored PR discipline.](images/08-code-review-prs-and-the-human-gate-fig-03.png)
*Figure 8.3 — Each element of the 1976 Fagan inspection — roles, checklist, defect log — maps onto a lightweight counterpart in agent-authored PR discipline.* What Fagan understood and what the green-checkmark era forgot is that *structure is what makes review catch things*. The inspection wasn't valuable because smart people looked. It was valuable because the process forced them to look at the right things in the right order. The agent era re-learns Fagan: unstructured review of a confident agent diff misses the residue; structured review recovers it.

Margaret Hamilton built the gate into Apollo. The flight software's priority-display design handled what it could automatically, then surfaced the decision to a human at the critical moment. That is the human gate in its purest form: automated recovery up to a threshold, human judgment at the call that matters. The merge is the agent-coding equivalent of Hamilton's threshold — the machine does the work right up to the irreversible decision, and a human owns the decision. Hamilton's insight was not distrust of automation. It was knowing *where* the human had to stay in the loop, and holding that boundary even when it was inconvenient.

---

## The trap that can undo it

There is one failure mode that threatens to dissolve the gate from the inside, and I want to name it clearly because it is the failure that the agent era makes newly likely.

![An author agent and reviewer agent sit inside a shared blind-spot region, giving correlated error. The reviewer's confident approved verdict flows to a human and suppresses scrutiny, shown by a faint diminished back-loop to the diff.](../images/08-code-review-prs-and-the-human-gate-fig-04.png)
![When an agent reviews an agent, correlated blind spots and a confident verdict compound, so the gate fails exactly where it is needed.](images/08-code-review-prs-and-the-human-gate-fig-04.png)
*Figure 8.4 — When an agent reviews an agent, correlated blind spots and a confident verdict compound, so the gate fails exactly where it is needed.*

The natural efficiency move is to have an LLM review the agent's diff. It helps with scale. It surfaces concerns faster than a human could read everything. But two risks compound in a way that's worth understanding mechanically.

The first is rubber-stamping. A confident LLM reviewer that says "looks good" trains the human to stop looking, which defeats the gate. A gate that nobody checks is a turnstile. The psychological literature on automation bias is well-established: humans shown a machine's confident verdict scrutinize the thing less. That's the *point* of automation bias — it's a rational response to reliable signals, and it becomes a liability when the reliable-seeming signal is wrong.

The second is correlated blind spots. An agent reviewing an agent's diff shares the *same* optimization pressures. Both were trained on similar data, both are weak on similar un-checked dimensions, and so the reviewer is least likely to catch exactly what the author was most likely to get wrong. The whole value of review is independence — a second pair of eyes that might have formed different intuitions, noticed different things. An LLM reviewer reviewing an LLM author's diff is not independent in the way that matters.

The consequence: an LLM reviewer can triage and draft, surfacing concerns and pre-filling the checklist. But it cannot own the merge, and the human must actually read the residue branches rather than trusting the agent's "approved." Automating the verification is just another model with the same problem sitting one layer up. The gate stays human at the terminal node, and the reason is not distrust of models in general but a specific structural fact about what review requires: independence, judgment about intent, and accountability that survives into tomorrow. Models don't carry accountability forward. Humans do.

<!-- → [IMAGE: Simple diagram showing automation-bias failure loop — agent authors diff → LLM reviewer approves → human sees green + approval and rubber-stamps → merge without residue check; caption: "The compounding failure: correlated blind spots meet automation bias"] -->

---

## What this chapter argues

I've been building one claim from three directions, so let me state it whole.

The verification loop is necessary but bounded. An agent optimizes to exactly what the loop checks, and the residue — intent, design, context-sensitive security — concentrates in what no test measures. The gate routes the mechanical to the loop and the judgment-laden to the human, made effective by PR discipline (small diffs, stated intent, a structured security checklist) and made non-negotiable at the merge. Fagan gave us the structure, Hamilton gave us the man in the loop, and Pornprasit and Tantithamthavorn gave us the warning that prompting won't substitute for either.

The sharpest exercise you can do to internalize this: take a diff that passes everything — green tests, clean types, no lint, static analysis silent — and find the failure anyway. A fix that suppresses a symptom rather than solving the problem (intent). A change that adds a third copy of an abstraction that should have been shared (design). A query built by string concatenation where every test happens to use a numeric ID (security). All green, all wrong. The green is *why* they slipped — they slipped through the lane the human was supposed to cover. Locating that failure is the skill the gate exists to exercise, and no amount of CI coverage buys it for you.

---

## LLM Exercises

**Exercise 8.1 (Analyze).** You are given four agent-authored diffs, all with green CI (in the course repo, `exercises/ch08/`): (A) a bug-fix that special-cases the test's exact input; (B) a feature that adds a third copy of a date-formatting helper; (C) an endpoint that builds a query by string concatenation, tested only with numeric IDs; (D) a refactor that quietly widens a public function's contract. For each, route every concern it raises to the mechanical lane or the human lane, and for the human-lane concerns, name which residue class (intent / design / context-security) it lands in. State, for each, the one check that would have moved the concern into the mechanical lane (and why nobody wrote it).

**Exercise 8.2 (Evaluate).** Take diff (C) from Exercise 8.1. Write the PR description *the agent should have written* (stating what it does and what it verified), then write the security-checklist section of the review, going item by item (injection, auth/logic, secrets, unsafe deserialization, memory/resource) and recording a finding or "n/a" for each. Show specifically how the *checklist* catches the injection that an unstructured "looks fine" review would have passed — i.e., demonstrate the §8.4 blind-spot recovery on a concrete diff.

**Exercise 8.3 (Create).** Design a reusable PR-review discipline for agent-authored changes in a real repo you have access to. Produce: (1) a PR-description *template* the agent fills in (intent, what changed, what was verified); (2) a review checklist with a mechanical-lane section (the exact test/type/lint/SAST commands) and a human-lane section (intent, design, the five-item security checklist); (3) a one-line merge policy stating the human owns the merge and the agent never self-merges. Then run it on one real agent-authored diff and append a paragraph: which residue-class concern did the structured checklist surface that you would *not* have caught reviewing by feel?

**Exercise 8.4 (Evaluate, optional).** Run an agent-reviews-agent experiment. Have agent A produce a diff with a deliberate intent defect (a confident-wrong-fix that passes the tests). Have agent B review it cold. Record whether B caught the intent defect, and whether B's "approval" made *you* less likely to scrutinize it (track your own time-on-diff with and without B's verdict shown). Report on both the correlated-blind-spot risk (did B miss what A got wrong?) and the automation-bias risk (did B's confidence reduce your scrutiny?), and state where you'd keep the human gate non-negotiable as a result.

---

## References

- Bacchelli, A., & Bird, C. (2013). *Expectations, Outcomes, and Challenges of Modern Code Review.* ICSE 2013, pp. 712–721. DOI 10.1109/ICSE.2013.6606617. — review's stated purpose (defect-finding) vs. realized value (knowledge transfer, design discussion).
- Fagan, M. E. (1976). *Design and Code Inspections to Reduce Errors in Program Development.* IBM Systems Journal 15(3), pp. 182–211. — the formal inspection: roles, checklist, logged defects; structure is what makes review catch things.
- Paul, R., et al. (2024). *Toward Effective Secure Code Reviews: An Empirical Study of Security-Related Coding Weaknesses.* Empirical Software Engineering (Springer); arXiv:2311.16396. — 135,560 review comments (OpenSSL, PHP); security concerns raised in 35 of 40 weakness categories; memory/resource classes under-discussed.
- Pornprasit, C., & Tantithamthavorn, C. (2024). *Fine-Tuning and Prompt Engineering for LLM-based Code Review Automation.* Information and Software Technology (Elsevier). arXiv:2402.00905. — fine-tuning beats prompting; adding a persona *reduced* exact-match accuracy.
- Margaret Hamilton — Apollo guidance software priority-display design (general history; MIT, Charles Stark Draper Lab). — automated recovery up to a threshold, human judgment at the critical call.

