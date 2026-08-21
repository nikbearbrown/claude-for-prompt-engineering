# Chapter 2 — The Agent's Context System: Static vs. Dynamic Context

## A scene: two failures that look identical and are not

Same agent, same afternoon, two bug reports from a teammate.

The first: "It keeps using tabs. We're a spaces shop — it's in the style guide." You check. The rule *is* in `CLAUDE.md`, plainly written. The agent followed it for the first dozen edits and then, deep in a long session, started emitting tabs anyway.

The second: "It deleted the retry logic in `payments.py` and said the function never had any." You check. The function absolutely has retry logic — right there on line 40. But the agent read that file an hour ago, then refactored a *different* module, and when it came back to `payments.py` it acted on the version it remembered, not the version on disk. Someone else had edited the file in between.

To your teammate these look like the same bug: "the agent ignored reality." They are not the same bug. The first is a **static-context** failure — a standing instruction got drowned. The second is a **dynamic-context** failure — the agent acted on a stale observation. They have different causes, different symptoms, and completely different fixes, and you will misdiagnose both unless you can tell which kind of context failed. That classification is the instrument this chapter hands you.

## The context system has two halves

Everything the model "knows" at any moment in a session lives in its context window, but that content arrives by two fundamentally different routes, and the routes are worth naming precisely.

**Static context** is everything written in advance and loaded the same way every session, independent of what the agent later does:

- the vendor's system prompt (not yours, but it is there — see Chapter 1);
- your persistent instruction file (`CLAUDE.md`, `AGENTS.md`, `CONVENTIONS.md`);
- any rules files, project conventions, or always-loaded docs the tool injects on startup.

Static context is *authored*. It is stable across the session. It is identical whether the agent is on turn 1 or turn 400. You control it completely, in advance, by editing a file.

**Dynamic context** is everything the agent gathers as it runs:

- file contents it reads;
- output from commands and tests it executes;
- results from tools and external services;
- and — this is the part people forget — *its own prior reasoning and messages*, which accumulate in the window just like everything else.

Dynamic context is *gathered*, not authored. It changes every turn. It reflects a particular moment in time — the state of a file *when it was read* — and it does not update itself when the world changes underneath it. You do not control it directly; you control it only indirectly, by shaping what the agent is allowed and inclined to read, and by deciding when to evict it.

Here is the single most useful sentence in the chapter: **static and dynamic context fail in opposite directions.** Static context fails by being *ignored* — drowned, outvoted, lost in the middle. Dynamic context fails by being *stale* — true once, false now, acted on anyway. Learn to ask, of any agent misbehavior, "did a standing instruction get ignored, or did the agent act on something that used to be true?" and you have already cut the diagnosis in half.

![Two columns — static (authored, stable) and dynamic (gathered, changing) — with failure arrows diverging in opposite directions: static toward "convention violation," dynamic toward "acts on expired fact."](images/02-the-agents-context-system-static-vs-dynamic-context-fig-01.png)

*Figure 2.1 — The two halves of the context system fail in opposite directions: static context by being ignored, dynamic context by going stale.*

## How static context fails: convention violations

Static context is, in principle, always present — it was loaded at startup and it is still in the window. So why does the tabs-versus-spaces rule get violated?

Because *present* is not the same as *attended to*. This is the *Lost in the Middle* result from Chapter 1, applied to your own instructions: a model does not weight every position in a long context equally, and a rule that sat near the top of an empty window at turn 1 is, by turn 400, buried under tens of thousands of tokens of dynamic material. It is still technically in context. The model just cannot reliably find it, weight it, and act on it against all the competing recent signal. `[High]`

The symptom is therefore distinctive: **convention violations that worsen as the session ages.** The rule is obeyed early and dropped late. The agent is not confused about what the code should do; it is producing perfectly functional code that simply violates a standing convention — wrong test runner, wrong import style, wrong directory for new files, wrong naming. If you start a fresh session and the same rule is suddenly obeyed again, you have confirmed a static-context failure: the instruction was fine, the *context conditions* under which it was read had degraded.

The fixes for static failures are all about *salience and budget*, and they are the subject of the next several chapters:

- Keep the instruction file short enough that everything in it stays attended to (Chapter 4's capacity budget).
- Restate the few truly critical rules at the *end* of the file, where recency helps the model weight them (Chapter 3).
- Move anything non-universal out of the always-loaded file into load-on-demand docs, so the static budget is spent only on rules that apply everywhere (Chapter 5).
- Restart or re-inject the instructions before the window fills past the danger zone (Chapters 7 and 9).

Notice what is *not* on that list: rewording the rule more forcefully. "ALWAYS use spaces, this is CRITICAL" does not survive being buried any better than a calm sentence does. The failure is positional and budgetary, not linguistic.

## How dynamic context fails: acting on stale information

Dynamic context fails in a way that is almost the mirror image. The `payments.py` problem is not that an instruction was drowned. It is that the agent's *picture of the file* was a snapshot taken an hour and a dozen edits ago, and the agent reasoned from the snapshot as though it were the present.

This is a structural property, not a bug you can phrase your way out of. When an agent reads a file, the contents enter the context as a frozen observation. Nothing in the loop refreshes that observation automatically. If the file changes — because the agent edited a different copy, because a parallel process touched it, because a teammate committed over it, because an earlier edit in the same session already changed it — the context still holds the old version, and the model has no built-in way to know it is stale. It will confidently describe retry logic as absent because, in the version it is holding, it was.

The symptom is distinctive in the other direction: **confident action on out-of-date facts.** The agent does not violate a convention; it asserts something false about the current state of the world and acts on it. "This function has no error handling." "There are no tests for this module." "I already imported that." Each was true at the moment of some earlier read and is false now. The tell is that the claim *was* accurate at some point — the agent is not hallucinating from nothing; it is reporting a real but expired observation.

The fixes for dynamic failures are about *freshness and verification*, and they are a different toolkit entirely:

- Have the agent re-read a file immediately before editing it, rather than trusting a read from earlier in the session.
- Prefer the *observe* move — actually run the test, actually re-grep — over reasoning from memory of a past observation.
- Avoid parallel work on the same files, where one agent's edits silently invalidate another's reads (Chapter 11's parallel-versus-sequential rule).
- When in doubt, treat any observation older than a few turns as suspect and re-acquire it.

Again, notice what is *not* on the list: editing `CLAUDE.md`. A standing instruction cannot fix a staleness problem, because the problem is not that the agent lacks a rule; it is that the agent is holding an expired fact. You debug static and dynamic failures with disjoint toolkits, which is exactly why telling them apart first is worth the effort.

## A diagnostic you can run in your head

Put the two failure modes in a single table and you have a field-usable diagnostic. When an agent misbehaves, ask the questions in order.

| Symptom | Likely class | Diagnostic question | First fix to try |
|---|---|---|---|
| Obeyed a rule early, violates it late in a long session | **Static** (ignored) | Does a fresh session obey the rule again? | Shorten/relocate the rule; restart near the budget ceiling |
| Produces working code that breaks a project convention | **Static** (ignored) | Is the rule actually in the instruction file, and is the file too long? | Trim the file to its universal rules (Ch. 4–5) |
| Confidently asserts a false fact about current file state | **Dynamic** (stale) | Was that assertion *true* at some earlier point? | Re-read the file immediately before acting |
| Reintroduces a bug it already fixed | **Dynamic** (stale) | Did an intervening edit or parallel process change the file? | Re-acquire state; avoid parallel edits to shared files |
| Both at once in a degraded long session | **Both** | Is the context near capacity? | `/clear` and re-execute from a saved plan (Ch. 7) |

The bottom row is the most common in practice and the most important. In a long, polluted session, *both* failure modes appear together — instructions get ignored *and* observations go stale — because the underlying condition (a full, noisy context window) drives both. That is why the universal repair, when the table sends you to the bottom row, is not a surgical edit but a reset: capture state, `/clear`, reload clean.

## The trap: putting dynamic content in a static file

Now the distinction stops being descriptive and starts changing what you build. The most common context-system mistake practitioners report is **smuggling dynamic content into a static file** — and it carries a specific, often invisible cost: it destroys cache stability.

Here is the mechanism. Static context is loaded identically every session, which means tools and the underlying model API can *cache* it — the stable prefix of the context is processed once and reused, which is faster and cheaper. Anthropic's own guidance on context engineering treats this prefix stability as something to protect. `[verify — Anthropic Engineering, current-state]` The moment you paste something that *changes* into your "static" file, you break that stability for no benefit, and you do it on every single session.

What does smuggled dynamic content look like? It is seductive because it feels helpful:

- Pasting the *current contents* of a key file into `CLAUDE.md` so the agent "always knows" it — except the file changes, and now your static context is permanently stale, reintroducing the dynamic failure mode *into the part of the system that was supposed to be reliable*.
- Embedding today's directory listing, or a snapshot of the test output, or "the current sprint's tickets."
- Hard-coding a line number — `the auth check is on line 142` — which is true until the next edit and then actively misleads the agent.

Each of these takes information that is inherently dynamic (it has a timestamp; it expires) and freezes it into the one part of the context that is supposed to be timeless. The result is the worst of both worlds: the content is stale like dynamic context, but it is reloaded every session and never refreshed like static context, and it has poisoned your cache prefix on the way.

The rule that falls out of this is sharp and worth memorizing: **static files hold only what is true regardless of time and task.** Conventions, architecture invariants, what the project is and why, durable "always/never" rules. Anything with an expiry date — file contents, line numbers, current state — belongs in *dynamic* context, which is to say: let the agent *read it fresh when it needs it*. Prefer a pointer over a paste. `the auth logic lives in src/auth/` is a stable pointer; pasting the auth code is a perishable copy. Chapter 5 turns this single instinct — pointers over copies — into the whole progressive-disclosure discipline.

## Worked example: classifying and fixing a real instruction file

Here is a `CLAUDE.md` fragment that mixes the two kinds of content. Your job is to classify each line.

```markdown
# Project: payments-service

## Conventions
- Python, formatted with black; 4-space indent, never tabs.        [STATIC ✓]
- Tests live in tests/ and run with pytest. Never use unittest.    [STATIC ✓]
- All money is handled as integer cents, never floats.            [STATIC ✓]

## Current state
- The retry logic is in payments.py on line 40.                   [DYNAMIC ✗ — expires]
- We are mid-migration; auth.py currently has both old and new    [DYNAMIC ✗ — "currently"]
  auth paths.
- See the full payments.py contents below:                        [DYNAMIC ✗ — a paste]
  def charge(amount): ...
```

The top three lines are legitimately static: they are true today, next week, and after the next refactor, and they apply to every task in the repo. They earn their place in a file the agent loads every session.

The bottom three are dynamic content masquerading as static. The line number will be wrong after one edit. "Currently has both paths" stops being true the day the migration finishes — and nobody will remember to update `CLAUDE.md`. The pasted function body is stale the moment someone touches `charge`. Every one of these will, on some future session, *confidently mislead* the agent, and because they live in static context the agent will trust them more than it should.

The fix is to evict the dynamic content and replace it, where needed, with a stable pointer the agent can resolve *fresh*:

```markdown
# Project: payments-service

## Conventions
- Python, formatted with black; 4-space indent, never tabs.
- Tests live in tests/ and run with pytest. Never use unittest.
- All money is handled as integer cents, never floats.

## Where things are
- Payment/charge logic: src/payments.py
- Auth: src/auth/ (a migration is in progress; read the module to see current state)
```

The pointer to `src/auth/` cannot go stale the way a line number can, and the parenthetical tells the agent to *acquire the current state dynamically* rather than trusting a frozen claim. You have moved the perishable facts out of the timeless file and back into the loop, where freshness is the loop's job.

## Exercises (Bloom: Analyze)

The Bloom target here is *analysis*: you should be able to classify context and predict its failure mode, not merely define the terms.

1. **Classify your own file.** Open your real `CLAUDE.md`/`AGENTS.md`. Tag every line STATIC or DYNAMIC. For each DYNAMIC line, write its expiry condition — the event that will make it false — and rewrite it as a stable pointer or move it out entirely.

2. **Predict before you reproduce.** Take a misbehavior you have seen from your agent. *Before* re-running it, predict whether it is a static (ignored) or dynamic (stale) failure, and write down the diagnostic question that would confirm it. Then reproduce it and check your prediction. Were you right? If not, what did you misread?

3. **Stage a dynamic failure.** Have the agent read a file, then edit a *different* file for several turns, then ask it to describe the first file from memory without re-reading. Modify the first file on disk in between. Capture the moment it asserts something false. Explain in two sentences why no wording of `CLAUDE.md` would have prevented this.

4. **Stage a static failure.** Put a clear, easy convention in your instruction file. Run a long session that fills the context with unrelated reads. Find the first violation of the convention. Then start a fresh session and confirm the rule is obeyed again. Write up why "fresh session fixes it" is the signature of a static failure.

5. **Audit the cache cost.** Find one piece of dynamic content hiding in a static file in any project you have access to (a pasted snippet, a line number, a "current state" note). Explain what it costs on *every* session — both the staleness risk and the cache-stability cost — and propose the pointer that replaces it.

## AI Wayback Machine: Barbara Liskov and the discipline of hiding what changes

> In the early 1970s, Barbara Liskov argued for a way of building software that now feels like air: *data abstraction* — building programs around abstract data types whose stable interface is separate from their implementation. (The closely related term *information hiding* is properly David Parnas's, coined in his 1972 work on modular decomposition; Liskov's own contribution was the abstract-data-type discipline and, later, the substitutability principle.) A module should expose a stable interface — what it does — and hide its implementation — how it does it, which is free to change. Her later substitutability principle made the same instinct formal: code should depend on the stable contract, never on the volatile internals. The whole point was to draw a line between *what holds across time* and *what is allowed to vary*, and to make sure the rest of the system depended only on the former.
>
> That is the static/dynamic distinction, four decades early, in a different costume. Your `CLAUDE.md` should expose the stable interface of your project — its conventions, its invariants, where things live — and hide the volatile internals — the current line numbers, the in-flight migration, the actual contents of a file that changes daily. When you paste dynamic content into a static file, you are doing exactly what Liskov warned against: making the durable layer depend on the volatile one, so that every change downstream silently breaks the thing that was supposed to be reliable. The fix is hers, too. Depend on the stable pointer, not the perishable copy. Let the implementation — the live file state — be fetched fresh, behind the interface, every time it is needed. Good agent context, it turns out, is just good information hiding aimed at a new kind of consumer.

## Bridge to Chapter 3

You now have the instrument the rest of Act Two is built on: any piece of an agent's context can be classified as static or dynamic, and that classification predicts how it will fail — convention violation versus stale-fact action — and which toolkit will fix it. You have also seen the cardinal sin, smuggling dynamic content into static files, and the cardinal habit, pointers over copies. What you have not yet done is *author* the static layer well. That is Chapter 3: how persistent instruction files actually load, why the `<system-reminder>` delivery and its "ignore if irrelevant" caveat mean only universally applicable content belongs, and how to write a `CLAUDE.md` that loads reliably and survives a long session. It is also where you build your first real artifact.

## Sources

- Liu et al., "Lost in the Middle: How Language Models Use Long Contexts," TACL 2024 — positional weighting; why a buried static rule loses force. [peer-reviewed]
- Yao et al., "ReAct," 2022/2023; Shinn et al., "Reflexion," 2023 — the loop and the role of preserved versus refreshed observations. [peer-reviewed/preprint]
- Yang et al., "SWE-agent," NeurIPS 2024 — interface/context determines behavior, not model intelligence alone. [peer-reviewed]
- Anthropic Engineering, "Effective context engineering for AI agents," 2025; Claude Code documentation, current — context engineering as a discipline; prefix/cache stability of static context. [vendor, current-state, `[verify]`]
- HumanLayer, "Writing a good CLAUDE.md," 2025 — what belongs in static context; keep it short and stable. [practitioner, `[verify]` on any numeric claim]
- Liskov, B. & Zilles, S., "Programming with Abstract Data Types," 1974; Liskov, B., Turing Award lecture / the Liskov Substitution Principle. [historical/peer-reviewed]
