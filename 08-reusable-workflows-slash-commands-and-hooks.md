# Chapter 8 — Reusable Workflows: Slash Commands and Hooks

It is Thursday, and for the third time this week you type the same paragraph into the agent. "Before you commit: run the full test suite, run the linter, run the type checker, and if any of them fail, stop and show me the output — do not commit." On Monday you typed it from memory. On Tuesday you forgot the type checker and the agent committed code that failed CI. On Wednesday you copied it from your Tuesday transcript, which still had the type-checker line missing. Each time, the same judgment — *these are my preconditions for a commit* — has to be reconstructed from scratch, slightly differently, slightly worse.

That paragraph is not a prompt. It is a *workflow*, and you have been re-deriving it by hand. The previous chapter ended on a rule that this chapter makes operational: **anything you prompt more than twice should become a command.** A repeated prompt is repeated judgment, and repeated judgment that lives only in your memory degrades exactly the way Thursday's did. The fix is to move that judgment out of your head and into a file the agent reads the same way every time.

This is a *Create* chapter. By the end you will encode a repeated workflow as a slash command in `.claude/commands/`, and you will register a lifecycle hook — including the one that re-injects your critical rules after the context has been compacted, the single most useful hook for keeping a long session honest.

## Why a repeated prompt is a liability, not a habit

The book's thesis says context is the bottleneck, and it has a corollary for workflows: *context that you re-type by hand is context you will eventually type wrong.* Every time you reconstruct a multi-step instruction from memory, you reintroduce the chance to drop a step, reorder a check, or soften a constraint. The agent is not the unreliable component in that loop. You are — not because you are careless, but because human working memory was never meant to be the storage medium for a precommit checklist.

A slash command turns the workflow into static, version-controlled context. It loads identically on every invocation, it lives in the repo where your teammates can see and improve it, and it removes the moment of reconstruction where errors creep in. This is the same logic as a persistent instruction file (Chapter 3), pushed down to the level of a single repeated task: stable, scoped context beats freshly-typed context, because freshly-typed context drifts.

## Slash commands: a workflow as a file

In Claude Code, a slash command is a Markdown file under `.claude/commands/`. The filename becomes the command name — `precommit.md` becomes `/precommit` — and the file's body is the prompt that gets injected when you invoke it [Medium; Claude Code documentation, current-state — verify path and syntax before print]. That is the whole mechanism. A slash command is just a prompt you wrote down once and named.

Here is the Thursday paragraph, finally promoted to a command. Save it as `.claude/commands/precommit.md`:

```markdown
---
description: Run all checks before committing; stop on any failure.
---

Before committing, run these in order and STOP on the first failure,
showing me the full output:

1. Tests:        `pytest -q`
2. Linter:       `ruff check .`
3. Type checker: `mypy src/`

Rules:
- Do NOT commit if any check fails.
- Do NOT modify code to make a check pass — report the failure and wait.
- If all three pass, stage the changes and show me the diff before
  committing. Wait for my go-ahead.
```

Now the workflow is `/precommit`. It is identical every time. The type-checker line cannot go missing, because it is not being retyped. When you discover next month that you also want a secrets scan before commits, you add one line to one file and every future invocation inherits it — including your teammates', because the file is in the repo.

![Before/after diagram: the same precommit workflow hand-typed across Monday, Tuesday, and Wednesday, with the Tuesday version dropping the type-checker check in red and the drift compounding, versus a single .claude/commands/precommit.md file invoked as /precommit that loads identically every time.](images/08-reusable-workflows-slash-commands-and-hooks-fig-01.png)

*Figure 8.1 — Anything you prompt more than twice should become a command: hand-typed context drifts and silently drops steps, while a named command file loads identically on every invocation.*

Commands take arguments, too. A command that scaffolds a new module benefits from a parameter for the module name. The Claude Code convention uses a placeholder (commonly `$ARGUMENTS`) that the invocation fills in [Medium; current-state — verify placeholder token before print]:

```markdown
---
description: Scaffold a new service module with test and registration.
---

Create a new service module named `$ARGUMENTS`.

SCOPE: create only these files; do not edit anything else yet.
- src/services/$ARGUMENTS.py        (the service, following the
                                      pattern in src/services/billing.py)
- tests/services/test_$ARGUMENTS.py (one passing smoke test)

Then show me the two new files. Do NOT register the service in the
router until I approve the files.
```

Invoked as `/scaffold-service inventory`, this carries the scope discipline of Chapter 7 — named files, an explicit "do not go further" boundary, a human checkpoint — into a one-line, repeatable call. The command *is* the task-prompt template, frozen and reusable.

There is a team dimension here that is easy to undervalue. When the precommit workflow lives only in your head, it is *your* standard — and your teammate's commits run a different, undocumented standard, or none. When it lives in `.claude/commands/precommit.md` in the repo, it is the *project's* standard, visible in code review, improvable by anyone, and applied identically no matter who is driving the agent that day. A slash command is therefore not only a memory aid for you; it is a way to make a workflow a shared, inspectable convention rather than folklore passed between developers. The first time a new teammate runs `/precommit` and gets exactly the checks the rest of the team runs, the command has paid for itself in onboarding alone.

## The "more than twice" rule, and what not to commandify

The rule is deliberately low. Twice is the threshold because the second repetition is the evidence: the first time a prompt might be a one-off, the second time it is a pattern, and patterns belong in files. Waiting until the fifth or tenth repetition just means you have re-typed the workflow three or eight more times than you needed to, accumulating the same drift errors each time.

But the rule cuts both ways, and the discipline is knowing what *not* to encode. A slash command is static context, and static context has a cost: it is one more thing competing for the agent's attention and one more thing to keep current. Commandify the workflows that are *stable and procedural* — precommit checks, module scaffolds, the investigate-then-wait pattern, a release-notes generator. Do not commandify the judgment that has to be made fresh each time: which bug to fix, whether this architecture is right, whether the agent's plausible-looking patch is actually correct. The line is the same one this book keeps drawing — automate the repeatable procedure, reserve the contextual judgment for yourself. A command that tries to encode judgment becomes a stale rule the agent follows past the point where it makes sense.

## Hooks: behavior that enforces itself

A slash command still depends on you to invoke it. A *hook* does not. Hooks are scripts the agent runs automatically at defined points in its lifecycle — before a tool call, after an edit, when a session starts, when the context is compacted [Medium; Claude Code documentation, current-state — verify hook event names and config schema before print]. Where a command is a workflow you choose to run, a hook is a workflow the system runs *for* you, on a trigger, whether or not you remember.

This is the difference between a checklist taped to your monitor and a door that physically will not open until the checklist is done. The first depends on your attention; the second does not. For the constraints you cannot afford to forget, you want the door.

The trade-off to respect is that hooks fire whether or not you wanted them to *this time*. A command you invoke is a choice; a hook is a standing law. That makes hooks the right tool for invariants — things that should be true on *every* commit, *every* edit, *every* session start regardless of what you happen to be doing — and the wrong tool for anything contextual. A hook that runs the full test suite before every single tool call would be correct in spirit and unusable in practice; it would tax every action with a cost that only some actions warrant. The skill is matching the lifecycle event to the actual invariant: a session-start hook to load project context, a pre-commit hook for commit gates, a post-edit hook to format the file just changed. Pick the narrowest event that still guarantees the thing you cannot afford to forget, and let everything more contextual stay a command you choose to run.

The natural pairing with the precommit command, for instance, is a `pre-commit`-style hook that runs the same checks automatically, so the protection holds even on the day you forget to type `/precommit`. The mechanics vary by tool and version, so treat any specific config as current-state, but the shape is: an event name, a matcher for when it fires, and a command to run.

## The PostCompact hook: re-inject what compaction forgot

Here is the hook worth the chapter on its own, and the one that motivates the next chapter.

When a long session approaches its context limit, the agent *compacts*: it summarizes the conversation so far into a shorter form and continues from the summary, freeing room to keep working (the full mechanics are Chapter 9's subject). Compaction is necessary, but it is lossy. The summary is the agent's best guess at what mattered — and the agent is not always right about which of your rules were load-bearing. A critical constraint you stated forty turns ago ("never edit files under `vendor/`") can simply evaporate in the summary, and the post-compaction agent, no longer holding the rule, cheerfully edits `vendor/`.

A **PostCompact hook** fixes this by firing immediately after compaction and re-injecting your non-negotiable rules into the fresh, summarized context [Medium; Claude Code documentation, current-state — verify hook name and availability before print]. <!-- FACT-CHECK FLAG: The *capability* is real, but the exact event name is uncertain. As of mid-2026 Claude Code documents a `PreCompact` event (fires before compaction) and a `SessionStart` hook with `source=compact` (fires after compaction, used to re-inject context). A standalone event literally named "PostCompact" was an open feature request (anthropics/claude-code issue #32026) and may not be a stable official name; the canonical post-compaction re-injection pattern is `SessionStart` with the `compact` source. Verify the precise event name against current docs before print. --> It is the system's way of saying: *whatever the summary kept or dropped, here are the rules again.* A minimal version re-states the constraints you most need to survive a compaction:

```
# fires after the agent compacts the conversation
# re-inject the rules a summary is most likely to lose

CRITICAL RULES (re-asserted post-compaction):
- NEVER edit files under vendor/ or migrations/.
- All DB changes go through a reviewed migration — no direct schema edits.
- Run `pytest -q` before any commit; do not commit on red.
- The production config lives in config/prod.yaml — treat it as read-only.
```

The PostCompact hook is the architectural answer to a problem that no amount of careful prompting solves on its own. You can state a rule perfectly, and compaction can still drop it. Only something that fires *after* the lossy step and re-asserts the rule can guarantee the rule survives. This is the chapter's thesis in miniature: reliability comes from engineering the context system — here, a hook that repairs the context after a known lossy event — not from wording the original instruction more emphatically.

![Repair timeline: a full conversation holding the rule "never edit vendor/" passes through a lossy compaction that drops the rule; without a hook the agent edits vendor/, while a PostCompact hook fires after compaction and re-injects a short critical-rule card (shown in red) outside the summary, unconditionally.](images/08-reusable-workflows-slash-commands-and-hooks-fig-02.png)

*Figure 8.2 — The PostCompact hook is the architectural fix for lossy compaction: it fires after the summarization step and re-asserts only the non-negotiable rules, a lifeboat rather than a second copy of the whole instruction file.*

Keep the rule card short, and keep it different from your CLAUDE.md. The temptation is to re-inject everything — but a PostCompact hook that dumps your entire rule set back into the window after every compaction just refills the budget you were trying to free, and buries the genuinely critical rules among the merely nice-to-have ones (the "lost in the middle" problem this book returns to in Chapter 9). The rule card earns its place by being ruthless: only the constraints whose violation would do real, hard-to-undo damage — destructive edits, schema changes, touching production config. Everything else can live in CLAUDE.md and survive or not survive compaction on its own merits. The hook is a lifeboat, not a second copy of the ship; you put in it only what you cannot afford to lose.

> **Encode the judgment, not the moment**
>
> The instinct when a workflow goes wrong is to be more careful next time. Slash commands and hooks ask for the opposite: stop relying on care, and move the workflow into a file that does not get tired, distracted, or rushed on a Thursday. A command captures a procedure you would otherwise re-type; a hook captures a guarantee you would otherwise have to remember to enforce. Both convert a fragile act of attention into durable, version-controlled context — which is the only kind that survives a long session intact.

## Exercises (Create)

1. **Promote your most-repeated prompt.** Search your last two weeks of agent transcripts for a multi-step instruction you typed three or more times. Write it as a `.claude/commands/*.md` slash command, with a `description` and explicit scope. Invoke it twice. Confirm it loaded identically both times, and note any step your hand-typed versions had been dropping.

2. **Parameterize a scaffold.** Build a slash command that scaffolds a unit of work in your codebase (a module, an endpoint, a component) and takes the name as an argument. Bake in the Chapter 7 scope discipline: name the exact files it may create, and forbid it from going further until you approve. Run it for two different names.

3. **Write a PostCompact rule card.** List the three to five rules in your project that would do real damage if the agent forgot them mid-session (the "never touch X" rules). Draft the PostCompact hook body that re-injects exactly those rules and nothing else. In one sentence, justify why each rule earned a slot — and why your *other* rules did not.

4. **Audit a command for smuggled judgment.** Take a slash command you (or a teammate) wrote and find one line that encodes a decision that really should be made fresh each time. Cut it, and explain what would have gone stale if you had left it in.

## Bridge to Chapter 9

You have moved repeated workflows out of your memory and into commands, and you have met the compaction event through the back door — by building the PostCompact hook that repairs what it breaks. But you have been treating compaction as a black box that occasionally eats your rules. The next chapter opens that box. We look at what `/compact` actually does to your context, how to tell it what to preserve, when to let the agent compact versus when to restart from a clean session, and why the most reliable practitioners restart at roughly eighty percent of capacity rather than ride a degrading window to the limit.

## The AI Wayback Machine: Douglas Engelbart

> On December 9, 1968, in a hall in San Francisco, **Douglas Engelbart** gave a demonstration so far ahead of its time it is still called "The Mother of All Demos." He showed a working mouse, hypertext links, real-time collaborative editing, video conferencing — most of the interactive computer, decades early. But the deepest idea behind the demo was not any single gadget. Engelbart's life's work, framed in his 1962 report *Augmenting Human Intellect*, was that the point of interactive computing is not to replace human judgment but to *amplify* it — and that amplification comes from building up reusable tools and conventions, "tool systems," that let a person stop re-solving solved problems and operate at a higher level.
>
> A slash command is a small piece of Engelbart's vision made literal. The precommit workflow you used to retype is now a tool in a tool system; the judgment you encoded once is available at the cost of one word. Engelbart called this *bootstrapping*: you use your tools to build better tools, and the better tools free your attention for the work only a human can do. The hook is the same idea pushed further — a tool that runs itself, so the amplification does not depend on your remembering to reach for it. Engelbart saw, in 1962, what CLI-agent users rediscover every Thursday: the leverage is not in working harder on each request, but in building a context system that carries your accumulated judgment forward so you never have to derive it twice.

## Sources

- Engelbart, D. C. (1962). *Augmenting Human Intellect: A Conceptual Framework*. SRI Summary Report AFOSR-3223.
- Yao, S., et al. (2022/2023). ReAct: Synergizing Reasoning and Acting in Language Models. *arXiv:2210.03629*.
- Yang, J., et al. (2024). SWE-agent: Agent-Computer Interfaces Enable Automated Software Engineering. *NeurIPS 2024*. (Interface design changes performance.)
- Anthropic. *Claude Code documentation* (current online docs; accessed 2026-05). Slash commands (`.claude/commands/`), lifecycle hooks, PostCompact. [Current-state — verify file paths, argument placeholder, hook event names, and config schema before print.]
- Sweller, J. (1988). Cognitive Load During Problem Solving. *Cognitive Science*, 12(2), 257–285. (Chunking and offloading procedure.)
- Ericsson, K. A., Krampe, R. T., & Tesch-Römer, C. (1993). The Role of Deliberate Practice in the Acquisition of Expert Performance. *Psychological Review*, 100(3), 363–406.

## Tags

#cli-agents #slash-commands #hooks #reusable-workflows #PostCompact #context-engineering #automation #Engelbart
