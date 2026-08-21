# Chapter 14 — Capstone: Shipping a Real Feature End-to-End

*The whole book, executed once and then measured — spec to merged, with a reliability number attached*

---

This is the do-it chapter. Not a demo. Not "look what the agent built." A pipeline of gated phases, executed once, and then measured — because shipping a feature once tells you almost nothing about whether your workflow reliably ships features.

The feature is deliberately small. A `--json` output flag added to a small, well-tested Python CLI utility that currently prints only human-readable text. Think a tool with an existing pytest suite, a `CONTRIBUTING.md`, and a tidy `argparse` entry point. Small enough to run many times, which is what the reliability measurement in §14.7 needs.

Before touching the agent: run the Chapter 13 gate. *Should an agent do this at all?* Correctness is mechanically checkable — a JSON schema assertion. The change is reversible and low-blast-radius — additive flag, no existing behavior changed. The repo is trusted — your fork. For a single feature, the overhead barely justifies the leverage; but the goal here is to *measure a workflow* over many runs, so the overhead buys data, not just the feature. Decision: **use the agent, supervised, human merge gate.**

Now the spine of the whole chapter, stated once: **the end-to-end loop is a pipeline of gated phases, not a single prompt.** Spec → plan → failing test → minimal implementation → review → merge → CI, each with a gate that must pass before advancing. This is what separates "the agent generated something plausible" from "the workflow shipped something verified."

![Six phase boxes in a left-to-right row: spec, plan, red failing test, green passing implementation, review, merge. Human-checked gate diamonds sit after spec, plan, review, and at merge. The intra-TDD transitions plan-to-red and red-to-green are plain arrows. The red phase carries an open test marker, the green phase a filled marker, and merge feeds a terminal main node.](../images/14-capstone-shipping-a-feature-fig-01.png)
![The end-to-end gated pipeline: a feature ships through six phases, each separated by a gate that must pass before advancing.](images/14-capstone-shipping-a-feature-fig-01.png)
*Figure 14.1 — The end-to-end gated pipeline: a feature ships through six phases, each separated by a gate that must pass before advancing.*

---

## The spec: the behavior contract

The spec is the primary artifact. In spec-driven development, you generate code, tests, and docs *from* the spec — not as an afterthought after the code exists. The spec is the per-feature behavior contract; the "constitution" is an immutable high-level rules file applied to every change. For the `--json` feature:

```
FEATURE: --json output flag
WHAT:  When --json is passed, the tool emits a single JSON object to stdout
       conforming to the documented schema below. No --json: existing
       human-readable text output is byte-for-byte unchanged.
SCHEMA: { "results": [ {...} ], "count": <int>, "version": "<string>" }
NON-GOALS: no --json for the --help path; no streaming/NDJSON; no config-file output.
INVARIANT: existing text output and exit codes are unchanged when --json is absent.
```

Two things to notice. First, the spec encodes **intent** — "existing output byte-for-byte unchanged" is a backward-compatibility promise no single test fully states but the human reviewer will check. Second, it names **non-goals**, which scope the agent the way Chapter 13's reversibility criterion wants: a bounded feature with explicit edges.

**Gate 1 — human reviews the spec's intent.** Does the schema match what the issue actually asked for? Are the non-goals right? This gate is cheap and catches the most expensive error class: building the wrong thing correctly. *Pass before planning.*

There is a genuine dispute about who authors the spec. Tooling pushes toward agent-authored specs from a human constitution; skeptics argue the spec is exactly where human judgment must dominate. The book's position leans skeptic for the contract itself: let the agent *draft* the spec, but the human *owns* the intent and the non-goals. That is the part tests cannot recover later.

---

## The plan: explore in one session, save the artifact

Now the highest-leverage workflow move from Chapter 7: **plan in one session, execute in another.** Exploration is dirty — to plan a non-trivial change the agent reads widely and rejects most of what it reads, and that rejected material is exactly the sediment that produces context rot. So you quarantine it.

![Two side-by-side session containers. The left exploration session is full of scattered read glyphs and crossed-out dead ends, with a discard arrow to a trash node. Only one clean plan page crosses rightward into the execution session, which is near-empty and holds just the plan and a fresh agent. The polluted exploration window is thrown away.](../images/14-capstone-shipping-a-feature-fig-02.png)
![Plan in session A, execute in session B: only the saved plan crosses to a fresh session and the polluted window is discarded.](images/14-capstone-shipping-a-feature-fig-02.png)
*Figure 14.2 — Plan in session A, execute in session B: only the saved plan crosses to a fresh session and the polluted window is discarded.*

In **session A**, let the agent explore freely: read the `argparse` setup, the output-formatting function, the existing tests, `CONTRIBUTING.md`. It produces a written plan:

```
PLAN: --json flag
- add --json to the argparse parser (file: cli.py, ~L40)
- branch output: if args.json -> json.dumps(build_payload()); else existing path
- build_payload() assembles the schema from the same data the text path uses
- FILES IN SCOPE: cli.py, tests/test_output.py
- VERIFY: pytest -q ; existing tests green ; new test green ; mypy clean
```

Save it to `plans/json-flag.md` — the durable artifact. The agent may have tried and rejected an approach in session A; that dead end now lives only in the polluted window, and you are about to discard it.

**Gate 2 — human approves the plan.** Is the scope right (only `cli.py` and the test file)? Is the approach sound (branch at output, not a parallel code path)? Are the verify commands the real ones? *Pass, then `/clear`.*

---

## TDD: red before green

Open **session B** — fresh window, exploration discarded. The execution session sees only the plan and a clean context, which is the condition under which agents perform well.

First, **red.** Per Chapter 4, the test is the agent's stopping condition and its acceptance criterion. The agent writes a failing test for `--json` against the schema, *before* any implementation:

```python
def test_json_flag_emits_schema(capsys):
    main(["--json"])
    out = json.loads(capsys.readouterr().out)
    assert set(out) == {"results", "count", "version"}
    assert isinstance(out["count"], int)
    assert out["count"] == len(out["results"])
```

Run it. It fails — `--json` is not a recognized argument yet. **Red confirmed.** This matters: a test that passes before you implement anything is testing nothing. The red state is evidence the test actually exercises the new behavior.

Then add the **behavior-preservation** guard from Chapter 5 — a test that the text path is unchanged:

```python
def test_text_output_unchanged(capsys):
    main([])                       # no --json
    assert capsys.readouterr().out == EXPECTED_TEXT  # golden, from before the change
```

This encodes the spec's invariant as a mechanical check. Now the tests, together with the spec, *are* the ground truth: the spec said "byte-for-byte unchanged," and this test enforces it.

---

## Implement: the minimal green

In the same clean session, the agent makes the **smallest change** that turns the new test green while keeping every existing test green — the Chapter 5 minimal-diff, behavior-preserving discipline. Add `--json` to the parser; branch the output; assemble `build_payload()` from the same data the text path uses.

Run the full suite:

```
$ pytest -q
........                                    [100%]
8 passed
$ mypy cli.py
Success: no issues found
```

New test green. Existing tests green — the text path is byte-for-byte unchanged. Types clean. The agent's stopping condition (Chapter 4) is satisfied: it does not keep going, because "done" was defined mechanically and the mechanism reports done.

A note on the failure mode this guards against, carried from Chapters 4 and 6: if the new test were weak — say it only checked that `--json` is *accepted*, not that the output *conforms to the schema* — the agent could turn it green with a stub that emits `{}`. The schema-and-count assertions close that hole. Tests are only as good a stopping condition as they are *specific*.

---

## Review and merge: the human gate

Green tests are necessary, not sufficient. Now the non-negotiable human gate of Chapter 8, checking the three things tests structurally cannot.

**Intent.** Does the emitted schema match what the issue and spec actually asked for? Tests check the schema you *wrote down*; the human checks whether the schema is the *right* one. Did the issue want `count` to include filtered-out results? The test can't know; you can.

**Security.** Does the JSON leak anything the text output didn't? A `--json` flag that helpfully serializes an internal object can expose a field — a path, a token, an internal ID — that the human-readable formatter omitted. New output surfaces are new exfiltration surfaces. Check the payload for anything sensitive.

**Design.** Flag naming (`--json` vs `--format json`), backward compatibility (the invariant holds, but is the flag discoverable in `--help`?), and whether the change fits the project's conventions from `CONTRIBUTING.md`.

**Gate 3 — human review: intent, security, design.** This is the gate no test replaces. *Pass before opening the PR for merge.*

Then the supervised-autonomy pipeline of Chapter 13: open the PR. CI runs in an ephemeral runner — isolated substrate, same checks. CI green. A human performs the **merge** (Gate 4, the only arrow that crosses into `main`). The agent proposed; CI verified; the human merged. The feature ships.

Pause on what just happened structurally. The agent is inconsistent — Chapter 13's pass^k — and yet the outcome was reliable, because the **architecture carried the inconsistent agent to a reliable result.** The gates, the tests, the isolation boundary, the human merge — *that* is what shipped, not the agent's per-run cleverness. Prompting tuned the agent's behavior *inside* the architecture; it was never the thing shipped on. This is the book's thesis, executed.

---

## The reliability assessment: a number, not a vibe

Here is the move that separates this chapter from every "look, the agent built a thing" demo. Shipping the feature *once* tells you almost nothing about how reliably the workflow ships features like it. So you measure the workflow — you turn the book's own argument on the loop you just used.

![Two zero-based bar panels. The left panel shows a tall optimistic pass-at-one bar near 0.7 against a much shorter all-of-k reliability bar near 0.17, with a bracket marking the gap. The right panel is a five-bar histogram of failures by pipeline stage—spec, plan, red, green, review—with the dominant failing stages emphasized.](../images/14-capstone-shipping-a-feature-fig-03.png)
![The reliability assessment: the pass@1 versus pass^k gap and a failure-by-stage histogram showing where runs broke.](images/14-capstone-shipping-a-feature-fig-03.png)
*Figure 14.3 — The reliability assessment: the pass@1 versus pass^k gap and a failure-by-stage histogram showing where runs broke.*

Run the **whole pipeline N times** on N similar small features (or the same feature from N clean starts), where a run "passes" iff it reaches a mergeable, CI-green, review-clean PR. Record:

**pass@1** — fraction of runs that succeeded on the first try. The optimistic number. The demo number.

**pass^k** — fraction of sets of k runs in which *all k* succeeded. For an inconsistent agent, pass^k ≪ pass@1, exactly as τ-bench found (Chapter 13). This is the number that tells you whether to trust an *unattended* run.

**A failure-by-stage histogram** — *where* in the loop runs broke. Did they fail at spec (wrong intent), plan (bad scope), red (test didn't fail when it should), green (couldn't pass without regression), or review (intent/security/design caught it)?

A template for the required artifact:

```
RELIABILITY ASSESSMENT — --json flag workflow
Repo+version: owner/repo@v1.2.3 (SHA abc123)
Feature class: additive output flag, schema-checked
N runs: 10
pass@1: 0.7        (7/10 reached a clean PR on first try)
pass^5: 0.17       (P(all of 5 i.i.d. runs clean) — estimated)
Failures by stage:  spec 0 | plan 1 | red 0 | green 1 | review 1
Trust verdict: "Supervised, this loop ships this feature class reliably enough
  — every failure was caught by a gate (the green failure by CI, the review
  failure by a human), so no bad change reached main. Unattended, pass^5 = 0.17
  says do NOT auto-merge this class. Gate stays on."
```

The trust verdict is the whole point. Notice what it does: it refuses to say "the agent ships `--json` flags." It says "*this loop*, run supervised, ships this feature class reliably because the gates catch every failure; the same loop is *not* trustworthy unattended, and here is the pass^k that proves it." That is an honest, quantified claim an engineer can act on.

The failure-by-stage histogram is the diagnostic. If failures cluster at *review*, your tests are too weak — move checks left into TDD. If they cluster at *plan*, your spec is under-specified. If at *green*, the task may be near the agent's capability edge for this repo. The histogram tells you where to invest.

Watts Humphrey, often called the father of software quality, built the Personal Software Process on a single thesis: reliable software comes from measured, disciplined process — not individual heroics. His PSP had engineers record their own defect data, run after run, and improve the process against the numbers rather than against intuition. The reliability assessment is PSP for the agentic age. The agent is the new "individual"; trusting its per-run brilliance is the heroics Humphrey warned against. The pass@1 / pass^k / failure-by-stage record is your PSP defect data — evidence about the *process*, gathered run after run, that tells you where the loop breaks and how much to trust it. Humphrey's move was to stop arguing about whether a developer was good and start measuring whether the *process* was reliable. The capstone makes the same move about the agent: stop arguing about how smart it is; measure how reliably the loop ships.

One calibration note: there is **no standard reliability-assessment protocol** for personal or team agent workflows. pass^k exists for benchmarks, not as a turnkey "measure my own loop" method. This chapter's template partly fills that gap; validating it is open. And the widely-repeated "~4.5 hours to build an end-to-end product via Spec→Plan→Tasks" is a **practitioner anecdote, not a study** — use it only as an order-of-magnitude illustration. Measure your own loop; do not import someone else's number as if it were yours.

---

## LLM Exercises

**Exercise 14.1 — Apply / Create: the capstone.** Pick a real, **trusted** repository you control or have forked (pin `owner/repo@version` and SHA). Choose a single bounded, mechanically-checkable feature — a flag, a small endpoint, a formatter. Run the full gated pipeline: write the spec (with non-goals and an invariant); plan in session A and save the plan doc; `/clear`; in session B write a *failing* test (confirm red), add a behavior-preservation test, implement the minimal green; pass the three-part human review (intent / security / design); open the PR; merge on green CI. Submit the spec, the plan doc, the test diff, the implementation diff, and the merged PR link.

**Exercise 14.2 — Evaluate: the reliability assessment.** Run your capstone workflow **N ≥ 8 times** (same feature from clean starts, or N similar features). Produce the §14.7 artifact: pass@1, an estimated pass^k for a k of your choice, a failure-by-stage histogram, and a one-paragraph trust verdict that states explicitly whether you would auto-merge this class unattended and *why the number says so*. Then name the single change to your loop that the histogram tells you would most improve reliability.

**Exercise 14.3 — Evaluate.** Before running 14.1, apply Chapter 13's when-not-to-use checklist to your chosen feature in writing: is correctness mechanically checkable? reversible / low-blast-radius? trusted repo? does leverage exceed overhead for a single instance? Then answer the harder question: does the calculus change when the goal is to *measure a workflow* (many runs) rather than ship one feature? State at what feature size the loop's overhead would make you decline the agent even for measurement.

**Exercise 14.4 — Create.** Write the one-page **capstone checklist** your team would use for every agent-shipped feature: the four gates (spec-intent, plan-scope, review-intent/security/design, merge), the red-before-green requirement, the behavior-preservation test, and the required reliability-assessment artifact. Map each line back to its source chapter. This is the book, compressed to a checklist you would actually run.

---

## What would change my mind

A demonstration that **skipping the gates does not cost reliability** — that for a representative class of bounded features, an ungated single-prompt "build me the `--json` flag and open a PR" run achieves pass^k *equal to* the full spec→plan→red→green→review pipeline, measured honestly across many runs and an adversarial review of the merged code — would overturn this chapter's central claim that the *pipeline of gates*, not any single run, is the product. My position is that the gates are what convert an inconsistent agent into a reliable outcome; the falsifiable form is: show me equal pass^k with the gates removed and a clean adversarial review of the ungated output, and I will concede the gates were ceremony. I expect the opposite — that ungated runs fail more, and fail *silently*, which is exactly the failure the gates exist to catch.

Separately, a validated, turnkey reliability-assessment protocol that outperforms the §14.7 template — better at predicting a workflow's real-world failure rate from a small number of runs — would replace my template, and I would adopt it.

---

## Still puzzling

**There is no standard reliability-assessment protocol for personal or team agent workflows.** pass^k is a benchmark metric; "measure my own loop" is not yet a turnkey method. How many runs N is enough to estimate pass^k for a feature class — and how to weight a failure caught by a gate versus one that escaped — is unsettled.

**How small is "too small" for the loop?** Chapter 13's when-not-to-use crossover, applied to capstone-sized features, is not quantified. For a one-line change the gated pipeline is absurd overhead; for a multi-file feature it is essential; the crossover point is judgment, not measurement.

**Does an agent-maintained spec stay in sync with shipped code over time?** Spec-driven development treats the spec as the durable primary artifact, but whether agent-maintained specs drift from the code they generated — and how you would detect the drift — is unstudied. A spec that has silently diverged from `main` is a stale-context generator at the project scale.

**Is spec-driven development genuinely new, or BDD and contract-first development rebranded?** The practitioner debate is unresolved. It may not matter for shipping — the gated loop works regardless of the label — but it matters for knowing which decades of prior practice you can borrow from.

---

## References

- GitHub (2025). *Spec Kit* (open-sourced Sept 2025) — Constitution → Specify → Plan → Tasks. [vendor/open-source]
- Fowler, M. / Thoughtworks (2025–2026). *Spec-driven development* discussion. [practitioner — non-peer-reviewed]
- *Spec-Driven Development* academic treatment. arXiv:2602.00180 (2026). [preprint — flag]
- Jimenez, C., et al. (2023/2024). *SWE-bench* and *SWE-bench Verified* — the F→P invariant as a done-criterion.
- Yao, S., et al. (2024). *τ-bench.* arXiv:2406.12045 — pass@k vs. pass^k, the reliability-assessment method.
- Humphrey, W. (1995/1997). *A Discipline for Software Engineering* / *Introduction to the Personal Software Process.*
- Boehm, B. (1988). *A Spiral Model of Software Development and Enhancement.*

---

**Tags:** capstone, spec-driven-development, tdd, plan-then-execute, behavior-preservation, human-gate, supervised-autonomy, reliability-assessment, pass-k, failure-by-stage, swe-bench-verified, humphrey-psp, spiral-model
