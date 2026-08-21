# Chapter 9 — Migrations and Dependency / Framework Upgrades

*Shard the change so each unit has its own pass/fail loop — and never advance a red shard*

---

Here is the failure, because it is the one every engineer is tempted into the first time.

A team needs to migrate a service off a deprecated date-handling library onto its successor. The API shapes are similar — most call sites are a mechanical substitution — so the obvious move is to hand the agent the whole repository and one instruction: "replace every use of `oldtime` with `newtime`, following the migration guide." The agent reads the guide, scans the codebase, and produces a single enormous diff touching 340 files. It compiles. The agent runs the suite. **Green.** The diff looks plausible — thousands of near-identical substitutions, exactly what you asked for. You merge it.

Three weeks later a customer in a half-hour-offset timezone files a bug. The old library and the new one disagree on how they round sub-minute offsets; one call site in the 340 relied on the old rounding, and no test exercised that path. The agent rewrote it faithfully — it changed the *code* exactly as instructed — and in doing so changed the *behavior*, silently, in a place no human looked and no test covered. The migration was "correct" by every signal the loop checked and wrong in production.

Sit with why this happened. The problem was not the model's competence — each substitution was right. The problem was the **structure of the task**: one giant change, one pass/fail signal for the whole thing, one human review of a 340-file diff nobody reads line by line. A green suite over a 340-file diff tells you almost nothing, because the suite was written for the *old* behavior and a behavior change no test covers passes it trivially. The single global loop is too coarse to catch a single local regression.

The instinct — the one this book exists to dislodge — is that the fix is a better prompt: "be careful about timezone edge cases," "preserve rounding behavior." That instinct is wrong in the same way it was wrong for debugging and refactoring. You cannot prompt your way to safety on a change whose verification signal is too coarse to see the danger. **The skill is not prompting the migration well. It is designing the shard and its verification loop** so that each unit of change has its own pass/fail signal and its own human review — fine-grained enough that a local regression has somewhere to be caught.

And here is the reassuring part: this is a *solved* problem, solved before LLMs existed.

---

## Rosie: the migration loop, pre-derived

Google has been running migrations larger than your entire codebase for over a decade, and the infrastructure they built to do it safely is the mental model this chapter teaches. It is described in Sadowski et al., "Lessons from Building Static Analysis Tools at Google" (*Communications of the ACM*, 2018) — a refereed paper, not a blog post.

![A global change is transformed and fanned out into per-project shards. One shard runs a build, run-tests, fix inner loop that loops back on any red result, passes a human-review gate, then commits — and a next-shard arrow returns to the splitter.](../images/09-migrations-and-upgrades-fig-01.png)
![The Rosie staged-shard migration pipeline: transform, fan out to shards, per-shard build-test-fix loop, human review, commit, next shard.](images/09-migrations-and-upgrades-fig-01.png)
*Figure 9.1 — The Rosie staged-shard migration pipeline: transform, fan out to shards, per-shard build-test-fix loop, human review, commit, next shard.*

**ClangMR** is the transform engine. It runs an updated Clang compiler in a distributed MapReduce over the whole codebase to programmatically fix every instance of a problem at once. Because the transform is a compiler-level rewrite operating on the parsed structure of the code — not a text substitution — it is behavior-preserving by construction for the class of changes it handles. The Java equivalent, **JavacFlume**, is a javac-based MapReduce that emits suggested fixes as diffs.

**Rosie** is the part that matters most for us, because it is the *loop*, not the transform. Rosie takes one giant change and **splits it into per-project shards.** For each shard it runs that project's tests, routes the shard to that project's owners for review, and commits the shard on approval. One enormous global change becomes hundreds of small, locally-verified, locally-reviewed changes. The 340-file timezone disaster cannot happen under Rosie, because there is no 340-file diff — there are dozens of small diffs, each tested against its own project's suite and read by the people who own that code.

Name the loop explicitly, because it is the chapter's central artifact:

> **Global change defined → transform applied → sharded into per-project units → [per shard: build → run tests → fix failures → human review] → commit shard → next shard,** with a loop-back on any red shard and a human gate on the review step.

That bracketed inner loop is the per-shard verification loop. It is the same red→green loop from TDD (Chapter 4) and the same behavior-preservation discipline from refactoring (Chapter 5), now replicated once per shard. The agentic version of migration changes exactly *one* thing about Rosie: it swaps the transform engine. Where Rosie used a deterministic compiler pass for the rewrite, an agentic migration can use an LLM for the parts a deterministic recipe can't express. Everything else — the sharding, the per-shard tests as ground truth, the per-shard owner review — stays. You are not inventing a new method. You are re-deriving Rosie with a probabilistic transform engine, which means you inherit Rosie's safety *only if you keep Rosie's loop.*

The durable lesson, stated as a rule: **large-scale migration is a sharding problem, not a single-edit problem.** This predates LLMs and survives them.

---

## Recipe-first: where determinism still wins

Before you reach for an agent, ask whether you need one. This is the triage, and it is the most actionable thing in the chapter.

![Three ordered questions route each change. Does a recipe exist routes to running it. Is it recipe-able but unwritten routes to the agent writing a codemod. Otherwise the agent edits per shard under a human gate — escalating from safest to riskiest.](../images/09-migrations-and-upgrades-fig-02.png)
![The recipe-first triage: three ordered questions routing each change to run-a-recipe, agent-writes-a-codemod, or agent-edits-per-shard under a human gate.](images/09-migrations-and-upgrades-fig-02.png)
*Figure 9.2 — The recipe-first triage: three ordered questions routing each change to run-a-recipe, agent-writes-a-codemod, or agent-edits-per-shard under a human gate.*

A deterministic transform — a codemod or a recipe — operates on the *parsed structure* of the code, not its text, and it is **idempotent** (running it twice changes nothing the second time) and **verifiable** (the same input always produces the same output). For the mechanical bulk of a migration, that is exactly what you want.

**OpenRewrite** (from Moderne) operates on a typed Lossless Semantic Tree — a full parse that preserves formatting — and expresses migrations as composable, idempotent recipes. Its maintained upgrade recipes chain dozens of sub-transforms: `UpgradeSpringBoot_3_0`, for instance, bundles the `javax`→`jakarta` namespace migration, dependency-version bumps, deprecated-API rewrites, JUnit and Jackson upgrades, and build-file edits, each as a tested transform. Where a recipe like this exists, it is more reliable and far cheaper than an agent, because it cannot hallucinate a substitution and it is the same every run.

**jscodeshift** and the broader codemod tooling do the same for JavaScript and TypeScript on an AST. The norm in that ecosystem is to ship a codemod with your breaking change — React ships official `react-codemod` transforms for its own migrations. When a library author has done this, your migration is "run their codemod," not "have the agent rewrite call sites."

The triage, as three questions you ask in order:

**Does a deterministic codemod or recipe already exist?** If yes, *use it.* It is more reliable and cheaper than any agent. The agent's job, if any, is the residue the recipe deliberately leaves alone.

**Is the change mechanical and recipe-able, but no recipe is written?** Then have the agent **write the codemod**, not hand-edit thousands of call sites. A generated codemod is verifiable (you can read it, test it on a sample, and confirm it is idempotent) and reusable (the next service with the same migration runs the same transform). Asking the agent to hand-edit 3,000 sites is asking for 3,000 independent chances to err; asking it to write one transform is one artifact you can verify once and apply everywhere.

**Is the change judgment-laden — a behavioral choice the recipe can't safely make?** Then it is the agent's residue: **agent-edits per shard, human-reviews per shard.** Custom security config, an API whose replacement requires a design decision, a deprecated pattern with no mechanical successor — these are where the LLM earns its place, and where the per-shard loop and human gate are non-negotiable.

One misconception to kill: "An agent makes codemods obsolete — just have it rewrite everything." No. Where a deterministic recipe exists or can be written, it is the *safer and cheaper* tool, and a good agent should be directed toward writing or running the recipe, not toward hand-editing under it. Pointing it at the mechanical bulk wastes its strength and exposes you to per-site hallucination on exactly the part a deterministic transform would have nailed.

The contrast in plain terms:

| | Deterministic codemod / recipe | Agentic transform |
|---|---|---|
| Idempotent? | Yes | No |
| Verifiable? | Yes | Only via tests, per shard |
| Handles judgment? | No | Yes |
| Cost | One transform, applied N times | Per-shard tokens + review |

Recipe-first means: spend the deterministic tool where it dominates, and reserve the agent for where only it can go.

---

## The Google case: a calibrated anchor, not a promise

There is one large, published, real-world LLM migration to learn from, and the right way to use it is as a *calibration point* — what one team got, under what conditions — not as a promise of what you will get.

![Three horizontal bars on a shared zero baseline and 0-to-100 scale from the Google JUnit3-to-JUnit4 migration: about 80 percent of edits AI-authored, about 87 percent committed unedited, and about 50 percent time saved.](../images/09-migrations-and-upgrades-fig-04.png)
![The Google JUnit3-to-JUnit4 migration figures: about 80% of edits AI-authored, about 87% committed unedited, about 50% time saved — a dated snapshot.](images/09-migrations-and-upgrades-fig-04.png)
*Figure 9.4 — The Google JUnit3-to-JUnit4 migration figures: about 80% of edits AI-authored, about 87% committed unedited, about 50% time saved — a dated snapshot.*

Tehrani, Hu, et al. (Google, 2025) reports a JUnit3→JUnit4 and related test-framework migration. The headline figures: **5,359 files and roughly 149,000 lines migrated in three months**; about **80% of the code modifications in the landed changes AI-authored**; and overall migration time **reduced by an estimated ~50%.** The method: combining the LLM with AST-based tooling and heuristics, verified by per-shard tests — not the model alone.

Those are real, large numbers, and they are also a *dated snapshot from a specific setup*. Three conditions made them possible, and none is automatic for your team.

**A fine-tuned model.** The Gemini model was fine-tuned on Google's internal codebase. It knew Google's idioms before it started. Most teams cannot do this, and the open question is how far a strong stock base model plus a good rule list plus AST tooling closes that gap.

**A monorepo.** The migration ran in a single repository with a single build and test graph. The multi-repo case — independent versioning, compatibility windows, staged rollout — is far less solved.

**The LLM was one component, not the system.** The transform was driven by a list of human migration rules as the prompt, combined with AST-based tooling and heuristics, and verified by the existing tests per shard. The 80% figure describes the share of *edits* the AI authored inside a system that was AST-assisted and test-verified — not "an LLM did a migration on its own."

Choose the migration class carefully, too. JUnit3→JUnit4 is a behavior-preserving test-framework migration — bounded, mechanical, and mechanically checkable, with the existing tests as a ready-made ground-truth loop. That is the *safest* class of agentic migration. A migration that changes runtime behavior, or one whose correctness the existing suite doesn't cover, is a harder and riskier animal.

Use the Google numbers the way this book uses every benchmark: as evidence that the *method* works at scale, dated to 2025, conditioned on a fine-tuned model and a monorepo. Teach the staged-shard-verify loop; treat the percentages as perishable.

---

## Behavior preservation: what the tests don't catch

The per-shard test suite is the migration's ground truth, and it is *necessary but incomplete* — which is the seam where the opening disaster slipped through.

![The code's total behavior space contains a large covered region pinned by the passing test suite and a smaller uncovered region. A silent regression sits inside the uncovered region where no test reaches, even with every test green.](../images/09-migrations-and-upgrades-fig-03.png)
![The preservation gap: a passing suite pins only covered behavior, leaving an uncovered region where a silent regression hides.](images/09-migrations-and-upgrades-fig-03.png)
*Figure 9.3 — The preservation gap: a passing suite pins only covered behavior, leaving an uncovered region where a silent regression hides.*

Frame it as Fowler's refactoring rule, scaled up: never transform code you cannot re-verify; every behavior-preserving change is backed by a test that proves behavior is unchanged. A migration is that rule applied at industrial scale — thousands of behavior-preserving transforms, each backed by the shard's test suite. The temptation is to break Fowler's rule exactly as §9.1 broke it: transform code whose behavior the suite *doesn't* actually pin, and call green a proof of preservation it isn't.

The hole is precise: **a migration can keep every test green and still change untested behavior.** The suite was written for the old code's behavior, and a behavior change in a path no test exercises passes trivially. Green means "the behavior the tests pin is unchanged." It does not mean "all behavior is unchanged." The timezone-rounding regression lived exactly in that gap.

This is the migration analogue of Chapter 6's confident-wrong-fix and Chapter 5's unsafe refactor: the same green-but-wrong shape, where the loop's normal signal is satisfied and the code is still wrong. The defenses are partial and worth naming honestly.

**Per-shard review by code owners.** The people who own a shard know which untested behaviors matter. Routing each shard to its owners — not pooling 340 files into one diff nobody reads — is the single highest-value defense, because it puts a human who knows the danger zones in front of a diff small enough to actually read.

**Characterization testing before the migration.** Where stakes are high, capture the old behavior as a characterization test before migrating — record actual outputs across a range of inputs and assert the new code reproduces them. This manufactures coverage for the untested paths *before* the transform. It is the agentic analogue of differential testing, and it is under-practiced.

**The recipe-first triage itself.** A behavior-preserving recipe gives you preservation by construction for its class — a stronger guarantee than tests. Maximizing the share handled by a deterministic transform shrinks the surface where untested behavior can drift.

The honest posture: a green per-shard suite *lowers* the probability of a silent regression but does not drive it to zero. The residue — the untested behavior a migration can quietly change — is exactly where the per-shard human gate and characterization tests do their work. You do not get to skip the human just because the suite is green. Green is the loop's necessary exit, not its sufficient one.

---

## Bridge to multi-agent and multi-repo

A single-repo migration is the sequential version of Rosie: shards processed one after another, each through its own verification loop and human gate, with deterministic recipes carrying the mechanical bulk and the agent handling the residue. The whole discipline reduces to one rule — *never advance a shard whose tests don't go green* — and one triage — *recipe-first, agent-for-the-residue.*

But notice what we kept assuming: one repository, one build graph, shards processed in sequence. Two pressures break that frame, and they are Chapter 10's subject.

The first is **concurrency.** If the shards are independent, why process them one at a time? Run a worker per shard, each in its own isolated checkout, in parallel — which forces the question of how independent agents avoid clobbering each other's files and how their separately-verified shards merge. A single shard's verification loop cannot close the integration question; that needs another loop.

The second is **multiple repositories.** When a shared library releases a breaking version, the downstream services live in separate repos with independent versioning and a compatibility window to respect. Each repo is a shard with its own CI as local ground truth, but sequencing the fleet — dependency order, compatibility windows, staged rollout — is the under-published problem this chapter explicitly hands off. The migration loop you just learned is the per-unit primitive; coordinating many of those loops, concurrently and across repos, is multi-agent and multi-repo engineering.

---

## LLM Exercises

**Exercise 9.1 — Analyze.** You are given three proposed migrations (in the course repo, `exercises/ch09/`): (A) a Spring Boot 2→3 upgrade for which an OpenRewrite recipe exists; (B) an internal API rename with no published codemod but a fully mechanical mapping; (C) a date-library swap where two call sites depend on undocumented rounding behavior the test suite never exercises. For each, place it in the recipe-first triage (run-a-recipe / agent-writes-a-codemod / agent-edits-per-shard-with-human-gate), name the verification loop you'd run per shard, and for (C) identify the specific untested behavior a green suite would *miss* and the characterization test you'd write before touching it.

**Exercise 9.2 — Apply.** Take a real dependency bump or framework upgrade in a repo you control. First, search for an existing recipe or codemod and, if one exists, run it and report what it did and did *not* handle — the residue. Then shard the residue by module, drive the agent through one shard's edit→test→review loop, and keep a per-shard log: which tests were fail-before/pass-after, what the agent edited, and what you caught in review that the tests didn't. Report how much of the total change the deterministic tool carried versus the agent.

**Exercise 9.3 — Create, produce something.** Plan a staged migration end to end. Produce a written migration plan that (a) defines the global change, (b) states the recipe-first triage decision for each class of change with justification, (c) shards the change into independently-buildable, independently-testable units, (d) specifies the per-shard verification loop — exact build and test commands — and the per-shard review gate, and (e) names at least two untested-behavior risks and the characterization tests that cover them. Then execute *one* shard with an agent and append a paragraph on whether your shard boundary was the right size — too coarse to review, or so fine the overhead dominated.

**Exercise 9.4 — Evaluate (optional).** Take the Google "80% AI-authored / ~50% faster" figures and write a one-page discount analysis: which conditions in your environment differ from theirs (fine-tuned vs. stock model, monorepo vs. multi-repo, behavior-preserving vs. behavior-changing migration, clean rule list vs. none), and what AI-authorship rate you would actually predict for your next migration, with reasoning. Then, if you can, run a small migration both ways — agent-only one-pass vs. recipe-first sharded — and report the defect counts.

---

## What would change my mind

The chapter's spine is that migration safety comes from *structure* — sharding with a per-shard verification loop and human gate — not from prompting, and that deterministic recipes should carry the mechanical bulk wherever they exist. A controlled study showing that a strong stock frontier model, given only a good rule list and *no* AST tooling and *no* sharding, produces migrations with regression rates statistically indistinguishable from the recipe-first sharded approach — across a representative set of real migrations with a defensible behavior-preservation check — would force a real re-specification. I'd want it demonstrated with behavior-preservation checked *beyond* the existing suite, because the whole opening-section danger is precisely the regression a green pre-existing suite can't see.

Separately, a verify-time check that reliably catches silent migration regressions *without* a human — differential testing driving the untested-behavior-drift rate near zero — would shrink the per-shard human gate this chapter insists on. That is the falsifiable form of "the human gate catches what the tests can't."

---

## Still puzzling

**Can the Google figures generalize without fine-tuning?** The 80%/50% numbers come from a model fine-tuned on the target codebase in a monorepo. Whether a stock frontier model plus good rules plus AST tooling reaches comparable AI-authorship and time-savings is unmeasured publicly. The method generalizes; the numbers may be fine-tuning artifacts.

**What is the agentic analogue of differential testing for migrations?** Tests are necessary but incomplete, and a migration can preserve every green test while changing untested behavior. Characterization tests help, but a principled, automatable way to verify behavior preservation *beyond the existing suite* at migration scale is immature.

**How fine should a shard be?** Too coarse and the human review is a rubber stamp on a diff nobody reads; too fine and the per-shard overhead dominates. Rosie shards per-project; whether that is the right granularity for an LLM-driven migration isn't characterized.

**What's the real cost comparison?** There is no clean published accounting of agent-token cost vs. deterministic-codemod cost vs. human-hours for a like-for-like migration. The recipe-first triage assumes deterministic is cheaper where it applies, which is almost certainly true, but the crossover — when the residue is large enough that writing a codemod no longer pays — isn't measured.

---

## References

- Sadowski, C., Aftandilian, E., Eagle, A., Miller-Cushon, L., & Jaspan, C. (2018). *Lessons from Building Static Analysis Tools at Google.* Communications of the ACM 61(4):58–66. DOI 10.1145/3188720. — Tricorder / ClangMR / JavacFlume / Rosie: the canonical staged, sharded, independently-verified large-scale-change loop.
- Tehrani, S., Hu, et al. / Google (2025). *How is Google using AI for internal code migrations?* arXiv:2501.06972. — JUnit3→JUnit4 etc.: ~5,359 files, >149k lines, ~80% AI-authored, ~50% faster. Treat all figures as a dated, fine-tuned-monorepo snapshot — verify against the PDF before quoting.
- Wright, H., Jaspan, C., et al. / Google (2013). *Large-Scale Automated Refactoring Using ClangMR.* — the foundational distributed-compiler-pass migration technique. [Confirm venue/citation against the PDF.]
- OpenRewrite / Moderne. *Upgrade recipes* — `UpgradeSpringBoot_3_0`, etc. docs.openrewrite.org.
- jscodeshift / `codemod` (Meta origin); `react-codemod` (github.com/reactjs/react-codemod, React team + Codemod.com).
- Fowler, M. (1999). *Refactoring: Improving the Design of Existing Code.* Addison-Wesley.

---

**Tags:** migrations, dependency-upgrades, framework-upgrades, sharding, rosie, clangmr, openrewrite, codemod, recipe-first, behavior-preservation, characterization-testing, per-shard-verification, human-gate, llm-ast-hybrid
