# Chapter 7 — Greenfield Builds: From Specification to Working Software

*Manufacturing the verification loop up front, because empty space has no ground truth*

Here is something I want you to think about. Every other task an agent does in this book comes with something to push against. A failing test. A stack trace. A behavior that must be preserved. Something is *already there*, already broken or already working, and the agent's job is to move toward it or away from it. The oracle is given for free, like a light in the room you didn't have to install.

Greenfield removes the light.

When you type "build me a working URL-shortener service with a REST API, persistence, and rate limiting," there is no prior behavior, no failing test handed to you, no compiler complaint about a contract you violated — because you haven't written the contract. The agent generates plausible code. And plausibility is the only signal in the room, because plausibility is *all* there is to check against.

I want to be precise about why this is different, because "no existing code" sounds like a mere convenience problem — just write the tests yourself, and the problem goes away. It doesn't. The deeper issue is that the agent is optimizing toward "looks like working software," which is not the same thing as working software. These two things are hard to tell apart in the first ten minutes, and easy to tell apart in the eleventh. The URL-shortener starts. You shorten a URL. For about ten minutes this is the most impressive thing you have ever watched a machine do.

Then you ask whether the rate limiter works across processes — and the whole thing comes apart. The rate limiter is per-process and resets on restart. Two modules each define their own notion of a "short code" and they disagree on length. The tests cover the happy path the agent already knew worked. Nothing failed, because there was simply never anything that *could* fail. There was no specification of what "correct" meant beyond "it runs and shortens a URL."

<!-- → [FIGURE: Timeline diagram — left axis is time (minutes 0–11+), right axis is "confidence in correctness"; a curve rises sharply in the first 10 minutes then drops when the cross-process question is asked. Caption: The eleventh-minute collapse: the agent produces code that satisfies all present signals (runs, returns output) while failing the one constraint that was never stated.] -->

This is the exact shape of the book's central thesis. *An agent is only as good as the verification loop you build around the task* — and in greenfield, there is no loop yet. The resolution cannot be a better prompt or a smarter model. It has to be a change in the order of work. You have to stop generating into a vacuum and manufacture the ground truth first.

---

## The inversion of work

The naming move comes from GitHub's Spec Kit, open-sourced in September 2025. Its slogan states the inversion bluntly: "Specifications don't serve code — code serves specifications." In ordinary agentic coding, you prompt and the code is the artifact; the spec, if it exists at all, lives in your head and evaporates. Spec-driven development flips that. The spec is the durable, authoritative artifact, and the code is generated *from* it and held accountable *to* it.

![A left-to-right pipeline of four phases — Specify, Plan, Tasks, Implement — each emitting an artifact and separated by a gate. Curved back-arrows above show any gate can reopen work upstream.](../images/07-greenfield-builds-spec-to-software-fig-01.png)
![Spec-driven development runs four gated phases — Specify, Plan, Tasks, Implement — each behind a gate that can send work back upstream.](images/07-greenfield-builds-spec-to-software-fig-01.png)
*Figure 7.1 — Spec-driven development runs four gated phases — Specify, Plan, Tasks, Implement — each behind a gate that can send work back upstream.*

The structure is four gated phases. The word *gated* is load-bearing: each phase produces an artifact, and you do not advance until that artifact passes review.

**Specify.** You give the agent your goals and user journeys in prose. It drafts a specification — what the system does, for whom, with what acceptance criteria. The gate is human approval of the spec. This is where intent gets pinned.

**Plan.** You hand the approved spec to the agent. It produces a technical plan: architecture, stack choice, data model, the cross-cutting decisions. The gate is human approval of the architecture, and I'll explain in a moment why no unit test substitutes for that.

**Tasks.** The plan gets decomposed into small, independently verifiable units — ideally each one a module-plus-test you can run in isolation. Each task is checkable on its own.

**Implement.** The agent builds chunk by chunk, each task verified green before the next begins. The gate is: the verification passes before you move.

<!-- → [FIGURE: Four-phase funnel or pipeline diagram — Specify → Plan → Tasks → Implement, with the human-gate symbol (a person icon or eye icon) at Specify and Plan, and a green checkmark at each Implement step. Caption: The four gated phases of spec-driven development. The human gates sit at the front, where defects cost a sentence to fix. The automated gates run at every chunk.] -->

Now map the four gates onto the §7.1 URL-shortener, and watch the vacuum fill. "Rate limiting" was never specified — Specify would have forced "limit per API key, across processes, persisted in Redis." The cross-process question was never an architectural decision anyone approved — Plan would have surfaced it. The two disagreeing modules came from never decomposing into a shared unit — Tasks would have caught the overlap. The happy-path-only tests came from having no per-task acceptance criterion — Implement would have required a test for the cross-process case before the chunk was done. Each gate is a place the vacuum gets filled with ground truth *before* code commits to it.

It's worth noting that Spec Kit wasn't the first to do this. Harper Reed's "My LLM codegen workflow" (February 2025) describes the same arc, hand-rolled, months before Spec Kit named it: brainstorm a spec, generate a step-by-step blueprint, have the agent implement one small test-backed chunk at a time. When a tool and the independent grassroots practice converge, the practice is the signal and the tool is the snapshot. Spec Kit's specific commands will age. The principle — manufacture ground truth before you build — outlives the packaging.

One misconception to clear up now. "Spec-driven development is just waterfall with extra steps." This conflates SDD with a cartoon version of waterfall that Royce himself never described. Winston Royce's 1970 paper — the one routinely miscredited as inventing waterfall — actually argued for iteration and feedback between phases. Royce never used the word "waterfall." The misattribution is well-documented. SDD is the iterative reading: each gate can send you *back.* A failed verification in Implement can reopen Tasks. A wrong assumption in Tasks can reopen Plan. A missing requirement can reopen Specify. The arrows go both ways. Waterfall is the cartoon; SDD is the spiral, with the spec as the thing you re-enter rather than the thing you signed once and froze.

---

## Why planning and executing are different jobs

The Plan and Implement phases aren't bureaucracy. They correspond to a real architectural distinction. The pattern is Plan-then-Execute — a reasoning-heavy Planner deconstructs an ambiguous request into concrete subtasks, and a distinct Executor runs them step by step. The argument is that planning and executing separated buys three things over a reactive loop that decides each step on the fly: predictability (you can review the whole plan before any action), cost-efficiency (the expensive reasoning happens once, up front), and reasoning quality (planning isn't interleaved with execution noise).

![A dirty planning session cluttered with discarded approaches emits a saved plan file. A bold clear barrier separates it from a clean execution session that consumes only the plan.](../images/07-greenfield-builds-spec-to-software-fig-02.png)
![Planning happens in a dirty exploration window; its sediment is cleared before execution runs from only the saved plan in a fresh window.](images/07-greenfield-builds-spec-to-software-fig-02.png)
*Figure 7.2 — Planning happens in a dirty exploration window; its sediment is cleared before execution runs from only the saved plan in a fresh window.*

This maps directly onto context discipline you already know. Exploration is dirty: to plan a non-trivial build the agent reads widely, considers approaches, and rejects most of them — and that rejected material is exactly the sediment that produces context rot. If you then execute in the same window, the agent builds from a context full of the architectures it already discarded. The fix is to plan in one session, save the plan as a durable artifact, clear the context, and execute from *only* the plan in a clean window. The Plan and Implement gates are this split, formalized at project scale and written to a file instead of held in one session.

The principle that ties planning back to verification is: **execution-grounded verification.** Every change should survive sandboxed execution before it propagates to the next step. This closes the greenfield loop. The spec gives you acceptance criteria; the plan decomposes them into tasks; and execution-grounded verification means each task is *run and checked* — not assumed correct because it looks correct — before the next task builds on top of it.

Without execution-grounded verification, decomposition is just a prettier vacuum. You'd be generating plausible chunks in sequence with nothing falsifying any of them. With it, the per-chunk run is the external signal — the greenfield analog of the test runner in a debugging session.

<!-- → [TABLE: Plan-then-Execute vs. ReAct loop — columns: property, Plan-then-Execute, reactive ReAct loop; rows: when expensive reasoning happens (up front / each step), reviewability before action (full plan visible / none), context cleanliness at execution (clean window / accumulated), defect discovery (pre-execution plan review / post-execution surprise). Caption: The structural advantage of plan-then-execute over reactive loops at project scale.] -->

The decomposition itself is the unit of reliability, and this is worth slowing down on. A task is well-formed only if it is *independently verifiable* — runnable and testable in isolation, with a clear pass/fail. "Build the persistence layer" is not a task; it's a fog. "Implement `ShortCode.generate()` returning a unique 7-char code, with a test asserting uniqueness across 10,000 calls and a test asserting length" is a task: it has a stopping condition and an oracle. The skill of greenfield-with-an-agent is largely the skill of cutting the spec into tasks of that grain — each small enough that its verification is obvious and cheap. Every task scaffolds module and test together, red before green, and the chunk isn't done until its own test passes.

---

## How much spec is enough

Here is the genuinely unsettled part, and the honest thing is to mark it as judgment, not formula. Two failure directions pull against each other.

![A horizontal axis of increasing spec density. Cost rises at both extremes — under-specifying invites hallucinated architecture, over-specifying re-imposes waterfall rigidity — with a project-dependent sweet spot at the valley's low point in the center.](../images/07-greenfield-builds-spec-to-software-fig-03.png)
![Spec density has failure at both extremes — under-specify hallucinates architecture, over-specify re-imposes waterfall rigidity — with a sweet spot between.](images/07-greenfield-builds-spec-to-software-fig-03.png)
*Figure 7.3 — Spec density has failure at both extremes — under-specify hallucinates architecture, over-specify re-imposes waterfall rigidity — with a sweet spot between.*

Under-specify, and the agent hallucinates architecture. Leave "rate limiting" as one word and the agent invents a per-process in-memory limiter, because that's the most common pattern in its training distribution and nothing in the spec rules it out. The vacuum doesn't disappear when you write a thin spec; it retreats into the gaps you left. Every under-specified requirement is a place the agent fills with a plausible default that may be wrong for *your* system — and you won't find out until minute eleven.

Over-specify, and you've reinvented waterfall's actual pathology: a spec so detailed it pins decisions that should have stayed open, costs more to write than the code it governs, and freezes choices before you've learned anything from building. Boehm's spiral model exists precisely because you *can't* know everything up front; some decisions are cheaper to make after a slice is real. A spec that specifies the color of every bikeshed is a spec nobody updates, which means it stops being ground truth — and a stale spec is worse than a thin one, because it lies with authority.

The sweet spot is project-dependent and there is no validated guidance for where it sits. Empirical reliability data comparing SDD to ad-hoc builds is thin and mostly practitioner-reported as of 2026. But the heuristic the gated phases give you is usable: **specify what's expensive to get wrong, defer what's cheap to change.** The cross-process semantics of the rate limiter is expensive to get wrong — it's an architectural commitment that, discovered late, costs a rewrite. The exact error message format is cheap to change — leave it to implementation. Spec density should track cost-of-being-wrong, not uniform thoroughness.

There's a second axis the literature flags as contested: who authors the spec. SDD's default is that the agent drafts the spec from your goals and the human refines it — faster than writing by hand, and the drafting surfaces ambiguities you'd have missed. But how much the human must own the spec for it to be trustworthy is an open question. The conservative posture: the agent may draft, but the human must read every line and own the contract. The drafting is delegable. The ownership is not.

---

## The two gates no test recovers

Step back to the structure of the four-gate pipeline and ask: what does it actually guarantee? It manufactures a lot of ground truth. The spec is the acceptance contract. Each task has a per-chunk test. Execution-grounded verification runs every change. That's a real loop, and most of the build is genuinely checkable. But two things in a greenfield build are not recoverable by any downstream test, and they sit at the front of the pipeline for exactly that reason.

**A wrong spec is unrecoverable.** Every test you write, every task you decompose, every chunk you verify is checked *against the spec.* If the spec is wrong — if it says "limit per IP" when the business needs "limit per API key" — then a perfectly green build implements the wrong thing perfectly. The verification loop confirms the code matches the spec; it cannot confirm the spec matches the intent, because the spec *is* the agent's notion of intent. There is no oracle upstream of the spec except a human who knows what the system is actually for. This is why the Specify gate is a human gate: not a process nicety, but the single point where the only available oracle is human judgment.

**A wrong architecture is unrecoverable cheaply.** No unit test exercises "should this be a monolith or three services," "is this rate limiter's state in-process or in Redis," "does this data model normalize or denormalize." These are the decisions that, made wrong, cost a rewrite to undo — and they pass every test, because tests check behavior *within* an architecture, not the choice of architecture. The Plan gate is a human gate for the same reason the Specify gate is: it's the verification-hardest residue, the part where green tells you nothing.

<!-- → [FIGURE: Nested verification diagram — outer ring: "spec intent (human gate)"; middle ring: "architecture soundness (human gate)"; inner ring: "per-chunk behavior (automated tests)". Caption: Tests verify behavior within an architecture, not the choice of architecture, and not whether the spec matches business intent. The human gates cover the residue no test can reach.] -->

A misconception worth catching here: "if every task's test is green, the build is correct." Green per-chunk means each chunk satisfies its test, which was written from the spec. The build can be uniformly green and wholly wrong if the spec is wrong or the architecture is unsound — and those are precisely the two things no chunk-test checks. Greenfield's green is necessary, not sufficient, in exactly the way a passing test suite is necessary but not sufficient: the residue concentrates upstream, at spec-approval and architecture, where the human gate sits.

Margaret Hamilton coined "software engineering" to insist that code deserved the rigor of other engineering disciplines. The Apollo flight software was built to a hard spec under hard constraints, and its reliability came from specification and rigor, not heroics. That is the posture greenfield demands. The agent supplies the heroics — it will happily write the code. The human supplies the specification and the rigor at the two gates the heroics can't substitute for.

---

## Why front-loading correctness is where the leverage is

Step back further to the economics, because it explains why the entire gated structure is worth its overhead.

Barry Boehm's *Software Engineering Economics* (1981) established the empirical shape of the cost-of-change curve: a defect costs more to fix the later you catch it. A requirements error caught at definition costs near nothing — you edit a sentence. The same error caught in the field, after it's been built on, tested around, and shipped, costs far more — you unwind everything downstream of it.

![A line chart from zero origin. Cost-to-fix rises monotonically and steepens from Specify through Plan, Tasks, Implement to shipped. The two earliest gates, Specify and Plan, sit at the cheap end of the curve.](../images/07-greenfield-builds-spec-to-software-fig-04.png)
![The cost of fixing a defect rises the later it is caught, so the Specify and Plan gates sit at the cheap end of Boehm's curve.](images/07-greenfield-builds-spec-to-software-fig-04.png)
*Figure 7.4 — The cost of fixing a defect rises the later it is caught, so the Specify and Plan gates sit at the cheap end of Boehm's curve.*

A caution the honest version of this chapter requires: the often-quoted "100×" multiplier is widely paraphrased and much milder on small projects. Boehm's own data shows something closer to 1:4 for small efforts, with the large multipliers appearing on big, formal projects. Don't quote the curve as a precise law. Quote it as a direction: cost-of-fix rises monotonically with how late you catch the defect, and the rise is steeper the larger the system.

<!-- → [CHART: Cost-of-change curve (schematic, not data) — x-axis: phase (Requirements, Design, Code, Test, Maintenance); y-axis: relative cost to fix a defect; two curves, one for large projects and one for small, both rising left-to-right but the large-project curve rising more steeply. Label the Specify and Plan gates on the left (cheap) end. Caption: Boehm's cost-of-change direction: defects are cheapest at definition. The steep curve justifies the gated overhead. Note the 100× figure applies to large formal projects; small project multipliers are closer to 1:4.] -->

That direction is the entire economic argument for spec-driven development. The four gates are positioned at the cheap end of Boehm's curve. The URL-shortener's rate-limiter defect was a requirements error — caught at Specify it costs one line, caught after two thousand lines of code build on the wrong assumption it costs a partial rewrite. SDD isn't process for its own sake. It's moving the verification gates to where defects are cheapest to catch, which is exactly where greenfield's residual risk lives.

This is also the answer to "isn't all this planning overhead?" Yes — and the overhead is the point, because it's spent at the cheap end of the curve to avoid cost at the expensive end. The payoff scales with how expensive your defects are to fix late, which is why greenfield is exactly the task where front-loading earns the most. An agent will write the code regardless; the question is whether the code it writes implements anything you can verify was the right thing to build.

---

## Where this leads

Greenfield is the verification-hardest task in Act Two because it starts with no ground truth, and the chapter's whole move is to manufacture it: write the spec as the acceptance contract, plan the architecture as the second gate, decompose into independently verifiable tasks, and close the loop per chunk with execution-grounded verification.

But two gates in that pipeline are human gates, and we've asserted rather than fully earned them here. That assertion is the subject of the next chapter. When the tests are green, what does a human still have to check, and why can't a test catch it? Chapter 8 makes the gate explicit and structured — a routing decision that sends mechanical concerns to the loop and intent, design, and context-sensitive security to the human who owns the merge. Greenfield gave you the spec as the reference. Chapter 8 is where you review a diff *against* it, and decide what no green checkmark certifies.

---

## What would change my mind

The chapter's spine is that greenfield has no given ground truth, so you must manufacture it up front, and that the gated pipeline's overhead pays for itself by catching defects at the cheap end of Boehm's curve. A controlled study showing that, holding the model fixed, one-shot prompting produces builds of equal correctness and maintainability to a four-gate SDD pipeline — across a representative set of project-scale tasks and a defensible scoring standard — would force a re-specification. It would mean frontier models have internalized enough specification discipline that the explicit gates are redundant overhead, the way reasoning-tuned models eroded the value of explicit chain-of-thought. I'd want it with the gates genuinely removed (not smuggled back in as a detailed prompt), at project scale where the eleventh-minute collapse actually appears, scored on the defects that don't show up in a casual run. Until then, the practitioner evidence and Boehm's economics both point the same way: front-loading correctness is where the leverage is.

Separately, a verify-time check that reliably caught spec-intent and architecture defects without a human — something upstream of the per-chunk tests that could falsify "this spec doesn't match the business need" — would shrink the human gate this chapter insists on. I don't know what that check would be (the spec is the agent's own notion of intent, so an agent checking it shares its blind spot), which is exactly why the spec gate must be human.

---

## Still puzzling

The right spec density for a given project size is unsettled. "Specify what's expensive to get wrong" is a heuristic, not a measured threshold. Nobody has validated guidance for where the under-/over-specify sweet spot sits, and it plainly depends on project size, stakes, and team.

Whether agent-drafted specs are trustworthy enough to scale without line-by-line human authorship is genuinely open — and it's the same question Chapter 8 asks about diffs, one level upstream.

How you verify an architectural decision before thousands of lines commit to it is unsettled. We route it to the Plan human gate, but that's placing the residue somewhere, not measuring it. Whether some architectural choices are checkable by a cheap prototype-slice rather than human judgment alone is an open question.

Whether multi-agent execution of a single spec helps or hurts is unresolved. Decomposed tasks invite parallel agents, but a shared spec with parallel executors raises coordination cost — the two modules that disagreed on "short code" in the URL-shortener are exactly the failure parallelism makes worse.

---

## LLM Exercises

**7.1 (Analyze).** You are given three one-paragraph feature requests (in the course repo, `exercises/ch07/`): (A) "a CLI tool that watches a directory and uploads new files to S3"; (B) "a webhook receiver that deduplicates events"; (C) "a billing module that prorates a mid-cycle plan change." For each, identify the *one requirement most expensive to get wrong* (the one that, left under-specified, would produce a §7.1-style eleventh-minute collapse), and write the single spec sentence that pins it. Then name one requirement in each that is *cheap to change* and should be deferred to implementation. Justify each call from cost-of-being-wrong, not from thoroughness.

**7.2 (Apply).** Take request (B) from 7.1. Run the four gates by hand with an agent: Specify (draft the spec, then *you* edit it and note what you changed and why), Plan (have the agent propose architecture; approve or reject the dedup-storage choice explicitly), Tasks (get a decomposition and check each task is independently runnable — rewrite any that aren't), and Implement (execute the first two tasks only, each red→green). Keep an artifact log: the approved spec, the approved plan, the task list, and the two green chunks. Report which gate caught the most ambiguity.

**7.3 (Create — produce something).** Build a small library from scratch (the course's `shortcode` package, or your own) using a plan→execute split. In session A, plan from a spec you wrote and approved; save `plans/shortcode-build.md` with the task list and per-task verification commands. `/clear`. In session B, execute *only* from the plan, in a clean window, each task green before the next. Then deliberately introduce a *spec* defect (e.g., specify 6-char codes when you needed 7) and run the build again. Append a paragraph: did any per-task test catch the spec defect? Why or why not — and what does that tell you about where the human gate has to sit?

**7.4 (Evaluate, optional).** Take a non-trivial greenfield task and build it two ways: (a) one-shot — "build me X" in a single prompt; (b) the four-gate SDD pipeline. Hold the model fixed. Compare on three axes: how many requirements the one-shot version silently got wrong (count the eleventh-minute defects), the total wall-clock and token cost of each, and the size of the resulting diff. Report whether the gated overhead paid for itself, and at what task size you'd expect one-shot to be the right call instead.

---

## References

- Boehm, B. W. (1981). *Software Engineering Economics.* Prentice-Hall. And Boehm, B. W., Papaccio, P. N. (1988). *Understanding and Controlling Software Costs.* IEEE TSE. (The cost-of-change curve: defects are cheapest to fix at definition. The ~100× multiplier is widely paraphrased and much milder on small projects — verify exact multipliers against the original tables.)
- GitHub (2025). *Spec-driven development with AI: get started with a new open source toolkit.* GitHub Blog (Sept 2025); repo github.com/github/spec-kit. (Spec Kit and the four gated phases Specify → Plan → Tasks → Implement; "code serves specifications." Treat tool specifics as a dating snapshot.)
- *Architecting Resilient LLM Agents: A Guide to Secure Plan-then-Execute Implementations.* arXiv:2509.08646 [PREPRINT]. (The Planner/Executor decomposition; predictability, cost-efficiency, reasoning quality over ReAct; execution-grounded verification as a first-class principle.)
- *Spec-Driven Development: From Code to Contract in the Age of AI Coding Assistants.* arXiv:2602.00180 [PREPRINT — very recent, unrefereed]. (The "spec as executable contract" framing; treat claims as provisional given preprint status and 2026 date.)
- Reed, H. (2025). *My LLM codegen workflow atm.* harper.blog (Feb 2025). [BLOG — practitioner evidence.] (Spec → blueprint → small test-backed chunks; the grassroots SDD that predates Spec Kit.)
- Royce, W. W. (1970). *Managing the Development of Large Software Systems.* (Routinely miscredited as inventing "waterfall"; in fact argues for iteration and feedback. Royce never used the word — verify the misattribution before citing.)
- *Companion volume:* Brown, N. B. *Prompt Engineering with CLIs* — the plan→execute split (§14.5) this chapter applies at project scale.
