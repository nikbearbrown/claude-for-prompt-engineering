# Chapter 7 — Task Prompt Design: Scope, Stopping Conditions, and Plan-then-Execute

You ask the agent to "fix the flaky checkout test." It reads the failing test, reads the module under test, then reads the module's imports, then the configuration loader those imports pull in, then the environment shim the loader references. Forty minutes and a few hundred thousand tokens later it has edited four files you never meant it to touch, broken two passing tests to make the flaky one green, and written a triumphant summary that says the issue is resolved. The test still fails on the next run.

Nothing here is a model failure. A frontier model is more than capable of fixing a flaky test. What failed is the *task prompt*. You handed the agent an open-ended verb and an undefined boundary, and it did what an unbounded agent does: it explored until the context filled with half-relevant reads, then acted on the polluted picture it had assembled. The lesson of this chapter is the spine of the whole book stated at the level of a single request. **Context is the bottleneck, not intelligence.** A task prompt is the one place where you, the human, decide what context the agent is allowed to gather, what counts as finished, and when its fluent confidence should be distrusted.

This is a *Create* chapter. By the end you will write task prompts with explicit scope, an explicit stopping condition, and explicit acceptance criteria, and you will run the highest-leverage workflow change available to a CLI-agent user: **plan in one session, clear the context, then execute from the plan alone.**

## A task prompt is a control specification, not a wish

In a chat window, a prompt is a request for text. In a CLI agent, a prompt is the entry point to a loop — the read, reason, act, observe cycle (Yao et al.'s ReAct framing) that runs until the agent decides it is done [High]. The difference matters because every iteration of that loop *consumes context*. Each file the agent reads, each command it runs, each error it observes is appended to the window it must reason over next turn. A vague prompt does not just risk a vague answer; it licenses an unbounded number of reads, and unbounded reads are how a session fills with noise.

So treat the task prompt as a control specification with three load-bearing parts.

**Scope** names what the agent may touch and, just as importantly, what it may not. "Modify only `checkout/session.py` and its test file; do not edit configuration, do not touch other modules" is scope. Scope is the lever against the overexploration trap — the documented pattern where an agent pulls tens of thousands of tokens of irrelevant context into the window and measurably loses accuracy as a result [verify — practitioner-sourced; attributed to Augment Code field reports]. You cannot stop the agent from reasoning poorly over a clean context, but you can stop it from polluting the context in the first place.

**Stopping condition** names the event that ends the loop. Without one, "done" is whatever the agent decides looks plausible — and a plausible-looking patch is not evidence of correctness, which is the single most expensive misconception a CLI-agent user holds [High]. "Stop when `pytest tests/test_checkout.py` passes and you have run it twice with no other test newly failing" is a stopping condition the agent can check against the world rather than against its own optimism.

**Acceptance criteria** name what *you* will inspect before you accept the work. These overlap with the stopping condition but are pitched at the human: the diff stays inside the named files, no unrelated test regressed, the fix addresses the root cause rather than silencing the symptom. Acceptance criteria are where you encode the judgment a generic model cannot infer from text — your risk tolerance, your sense of what "really fixed" means for this codebase.

A task prompt that carries all three reads less like a wish and more like a work order.

It helps to see how each part fails when it is missing, because the three failures look different and you diagnose them differently. A prompt with no **scope** fails by *sprawl*: the diff is too large, files you never named got touched, the session's context filled with reads that had nothing to do with the task. A prompt with no **stopping condition** fails by *false completion*: the agent declares victory on a plausible-looking change that does not actually hold, because nothing told it what "actually holds" means in checkable terms. A prompt with no **acceptance criteria** fails most insidiously of all — by *silent drift*: the work passes the agent's own bar, the diff is plausible, the tests the agent chose to run are green, and yet the change is subtly not what you wanted, because you never wrote down the standard you would have judged it against. Sprawl you can see in the diff. False completion you catch on the next run. Silent drift you sometimes do not catch until production. Writing all three parts is not belt-and-suspenders; each guards against a failure the others cannot.

![Three-column mapping: a missing SCOPE produces sprawl (seen in the diff), a missing STOPPING CONDITION produces false completion (caught next run), and a missing ACCEPTANCE CRITERIA produces silent drift in red (not caught until production).](images/07-task-prompt-design-scope-stopping-conditions-and-plan-then-execute-fig-02.png)

*Figure 7.2 — The three load-bearing parts of a task prompt and the distinct failure each one prevents; silent drift, the failure acceptance criteria guard against, is the most insidious because nothing else catches it.*

## The worked example: from open verb to control spec

Here is the flaky-test request, first as most people write it, then as a control specification.

The version that fails:

```
Fix the flaky checkout test.
```

The version that holds:

```
TASK: Make tests/test_checkout.py::test_session_expiry deterministic.

SCOPE
- You may edit ONLY: checkout/session.py and tests/test_checkout.py
- You may READ: anything under checkout/, but do not edit it.
- Do NOT touch config/, do NOT touch other test files.

INVESTIGATE FIRST (do not edit yet)
- Run the test 5 times: `pytest tests/test_checkout.py::test_session_expiry -q`
- Report the failure pattern and your hypothesis for the root cause.
- WAIT for my approval before editing.

ACCEPTANCE CRITERIA
- The test passes 10 consecutive runs.
- No other test in tests/test_checkout.py newly fails.
- The diff touches only the two named files.
- The fix addresses the timing race, not the assertion threshold.

STOPPING CONDITION
- Stop and summarize once the acceptance criteria are met,
  OR after two failed fix attempts — do not keep patching.
```

Notice what the second prompt does. It bounds reads (scope), forces a diagnosis before any edit (the *output-a-plan-before-acting* move, below), defines done in terms the agent can verify against the test runner, and — crucially — installs a circuit breaker: *after two failed attempts, stop.* We will return to that circuit breaker; it is one of the most underused instructions in the discipline.

## Output a plan before acting

The cheapest way to catch a misaligned task is to make the agent state its plan before it edits anything. "Investigate and propose a fix; wait for my approval before changing code" costs you one extra turn and buys you a checkpoint at the moment the cost of correction is lowest — before any tokens have been spent on the wrong edit and before the context has been polluted with the wreckage of a bad attempt.

This works because of where the failure actually lives. When an agent goes wrong, it rarely goes wrong at the level of syntax. It goes wrong at the level of *intent*: it misread which behavior you wanted, or it assumed a constraint you never stated. A plan surfaces intent in plain language, where you can read it in seconds. A diff hides intent inside mechanics, where you have to reverse-engineer it. Reviewing a three-sentence plan is faster and more reliable than reviewing forty lines of changes, and it catches the same class of error earlier.

There is a second reason to demand a plan, and it is about evidence rather than intent. The most expensive belief a CLI-agent user can hold is that *a successful-looking result is evidence of success.* An agent that has edited code and reports "the issue is resolved" is making a claim, and the claim is fluent, confident, and frequently wrong — not because the model is dishonest but because it is reasoning over a context that may already be polluted with its own optimistic reads. A plan, stated before action, lets you check the agent's *reasoning* against your knowledge of the system before any code changes — the one moment where you can catch a wrong premise cheaply. Reviewing the plan is how you distrust the fluency on purpose, at the point where distrust costs the least.

The plan also becomes an artifact you can save — which is the hinge to the chapter's central workflow.

## Plan, then clear, then execute from the plan alone

Here is the single highest-leverage change you can make to how you drive a CLI agent. Anthropic's own Claude Code guidance endorses splitting planning from execution, and the practitioner community has converged on the same shape [High; Anthropic Claude Code documentation, current-state].

The move has three beats:

1. **Plan in one session.** Let the agent explore the codebase, ask questions, and produce a concrete written plan: the files it will change, the approach, the verification steps, the acceptance criteria. Exploration is *supposed* to be messy and read-heavy here. That is the cost of understanding.

2. **Clear the context.** Run `/clear` (or start a fresh session). Everything the agent read while planning — every half-relevant file, every dead-end, every error from a probe command — is now gone. What survives is the plan, because you saved it to a file.

3. **Execute from the plan alone.** In the clean session, hand the agent only the plan and the instruction to execute it. It now works against a context that contains the *conclusions* of exploration without the *debris* of exploration.

Why does this help so much? Because the planning session and the execution session have opposite needs. Planning needs breadth: the agent should read widely to understand the terrain. Execution needs focus: the agent should hold a small, clean set of instructions and act on them. If you run both in one session, the breadth you needed for planning becomes the noise that degrades execution. The window that helped you understand the problem now actively hurts you as you solve it. Separating the two lets each phase have the context it actually needs.

![Three-beat pipeline: a planning session accumulates wide reads, dead ends, and probe errors and emits PLAN.md; a red /clear boundary drops all that debris so only PLAN.md crosses into a clean execution session that holds just the plan's conclusions.](images/07-task-prompt-design-scope-stopping-conditions-and-plan-then-execute-fig-01.png)

*Figure 7.1 — The plan-then-execute split: the planning session reads wide and messy, the /clear boundary drops the exploration debris, and only the saved plan crosses into a clean execution window.*

Concretely, the planning session ends with something like:

```
Write your plan to PLAN.md. Include: files to change, the approach,
the exact verification commands, and acceptance criteria.
Do not edit any source files yet.
```

Then you `/clear`, open a fresh session, and say:

```
Read PLAN.md and execute it exactly. Run the verification commands in
the plan. If a step fails twice, stop and report — do not improvise
beyond the plan.
```

The saved `PLAN.md` is a durable handoff between two stateless sessions — a pattern this book develops fully in Chapter 10. For now, notice that the plan file is doing the work that the chat history *cannot* do reliably: carrying intent across the boundary without carrying the noise.

## The two-correction rule: clear and rewrite, don't pile on patches

When a fix attempt fails, the instinct is to correct it in place: "no, not like that — try the other approach." When that also fails, the instinct is to correct again. Resist it. **After two failed corrections, stop, `/clear`, and rewrite the task prompt from scratch** [Medium; practitioner-converged guidance].

The reasoning is, once again, about context rather than cleverness. Each failed correction leaves its wreckage in the window: the wrong edit, your objection, the second wrong edit, your second objection. By the third turn the agent is reasoning over a transcript dominated by failure, and the failures themselves bias it toward more of the same. You are not steering anymore; you are negotiating with an accumulating record of confusion. Piling a third instruction onto that pile rarely escapes it.

A `/clear` and a rewritten prompt costs you the few minutes of context the agent had built — but that context was mostly noise by the third turn anyway. What you gain is a clean window and a prompt sharpened by what the failures taught you. The first attempt revealed that your scope was too loose or your acceptance criteria too vague; the rewrite fixes that. Two failures is the signal that the *prompt* is wrong, not that the agent needs one more nudge.

This is the same logic as the plan-then-clear workflow, applied to repair instead of to scale. In both cases the move is: when the context has gone bad, do not try to argue your way out of it from inside — leave, and re-enter clean.

> **The map is not the territory — but write the map down anyway**
>
> A working developer's instinct is to skip the plan and "just let it run." The plan-then-execute workflow asks you to do the opposite: spend a whole session producing an artifact you then deliberately discard the context of. The payoff is not the plan as a document; it is the plan as a *filter* — the one thing allowed to cross the `/clear` boundary. The discipline is learning to trust that a clean window plus a good plan beats a full window plus the memory of how you got there.

## Exercises (Create)

These build toward a reusable personal practice, in the spirit of deliberate practice — repeated scoped tasks with real feedback, not admiration for the tool.

1. **Convert an open verb into a control spec.** Take a real request you would actually make of an agent on your own repo ("add input validation to the signup form"). Rewrite it with explicit Scope, Investigate-First, Acceptance Criteria, and a Stopping Condition that includes a two-attempt circuit breaker. Run both versions on a throwaway branch and record the diff sizes and the number of files touched by each.

2. **Run the plan-then-execute split.** Pick a moderately involved change in your codebase. In session one, drive the agent to write `PLAN.md` and forbid edits. `/clear`. In session two, execute from `PLAN.md` alone. Then run the *same* task in a single un-cleared session. Compare: which produced the tighter diff, and which session's context was noisier at the end?

3. **Trigger the two-correction rule on purpose.** Find a task you expect the agent to get slightly wrong. Correct it once, correct it twice, and on the third failure stop and rewrite the prompt from scratch. In two sentences, state what the two failures told you to change in the prompt.

4. **Write the stopping condition you keep forgetting.** Audit five recent agent sessions. For how many did you state, up front, the event that should end the loop? Draft a one-line stopping-condition template you can paste into future prompts.

## Bridge to Chapter 8

You now write task prompts that scope reads, define done, and split planning from execution — and you have noticed that you write some of the same instructions again and again. The investigate-first block, the two-attempt circuit breaker, the verification commands: these are workflows, not one-offs. The next chapter takes the rule that falls naturally out of this one — *anything you prompt more than twice should become a command* — and shows you how to encode a repeated workflow as a reusable slash command, and how to use lifecycle hooks to make the agent's behavior enforce itself rather than depending on you remembering to type it.

## The AI Wayback Machine: Lucy Suchman

> In the mid-1980s, the anthropologist **Lucy Suchman** was studying why people struggled with a "smart" photocopier at Xerox PARC. The machine had been designed around a plan: a fixed sequence of steps the user was expected to follow. Real users did not follow plans that way. They improvised, backed up, reinterpreted instructions in light of what the machine actually did, and got stuck precisely where the designers' plan diverged from the unfolding situation. Suchman's 1987 book *Plans and Situated Actions* drew the conclusion that has aged remarkably well: a plan is not a program that determines action; it is a *resource* for action that must be continually reinterpreted against a real, surprising environment.
>
> That is exactly the tension you manage in a task prompt. The plan you and the agent write is indispensable — but it is not the execution. The execution happens turn by turn, in contact with a test runner that fails in ways the plan did not anticipate, a codebase that contains a constraint nobody wrote down. Suchman saw, forty years early, why the plan-then-execute workflow works *and* why the two-correction rule is necessary: the plan carries your intent across the boundary, but when the situated action drifts far enough from the plan, no amount of in-place correction recovers it. You stop, you re-read the territory, you write a new map. Suchman would recognize the move. She spent her career arguing that intelligent behavior lives in the interaction, not in the instructions — which is another way of stating this book's thesis: the leverage is in the context system, not in the cleverness of the words.

## Sources

- Yao, S., et al. (2022/2023). ReAct: Synergizing Reasoning and Acting in Language Models. *arXiv:2210.03629*. (Read/reason/act/observe loop.)
- Shinn, N., et al. (2023). Reflexion: Language Agents with Verbal Reinforcement Learning. *arXiv:2303.11366*. (Accumulated context helps or hurts depending on what is preserved.)
- Yang, J., et al. (2024). SWE-agent: Agent-Computer Interfaces Enable Automated Software Engineering. *NeurIPS 2024*. (Interface and context, not raw model intelligence, drive task performance.)
- Anthropic. *Claude Code documentation* (current online docs; accessed 2026-05). Plan mode, `/clear`, task workflows. [Current-state — verify command names before print.]
- Suchman, L. (1987). *Plans and Situated Actions: The Problem of Human-Machine Communication*. Cambridge University Press.
- Sweller, J. (1988). Cognitive Load During Problem Solving. *Cognitive Science*, 12(2), 257–285. (Why scoped, chunked instruction beats overload.)
- Ericsson, K. A., Krampe, R. T., & Tesch-Römer, C. (1993). The Role of Deliberate Practice in the Acquisition of Expert Performance. *Psychological Review*, 100(3), 363–406. (Exercise design.)
- *Practitioner note:* the overexploration figures (~80k tokens of irrelevant context; accuracy degradation) are field-reported and attributed to Augment Code; marked `[verify]` pending controlled confirmation.

## Tags

#cli-agents #task-prompts #scope #stopping-conditions #plan-then-execute #context-engineering #two-correction-rule #Suchman
