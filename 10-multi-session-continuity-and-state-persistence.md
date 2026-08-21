# Chapter 10 — Multi-Session Continuity and State Persistence

You close the laptop on Friday with the migration half-done. The agent had a clear picture: which tables were already converted, which one was mid-flight and why it was tricky, the two edge cases you had agreed to handle later, the exact command that ran the migration in dry-run mode. On Monday you open a fresh session and that entire picture is gone. The agent does not remember Friday. It cannot. As far as it is concerned, Monday is the first time it has ever seen this repository. So you spend the first half hour of Monday reconstructing, badly and from memory, what Friday-you and the agent already knew — and you get some of it wrong, because the thing that actually held the state was a conversation that no longer exists.

This is the defining constraint of multi-session work, and it is easy to misread. The agent is not forgetful; it is *stateless across sessions* by design. Each session begins from the persistent files and nothing else. The whole book has been building to this realization: if context is the bottleneck, then the bottleneck does not disappear at the end of a session — it *resets to zero*. Continuity is not something the agent provides. It is something you engineer, with durable artifacts that live in the repository and outlast any single window.

This is a *Create* chapter. By the end you will build the three artifacts that carry state across the gaps — a session-notes handoff, a registry, and a durable work log — and you will know the one rule that decides whether multiple agent sessions can run in parallel or must run in sequence.

## State lives in files, not in the window

The mental correction to make is blunt: **the chat history is not where your project state lives, and treating it as storage is the root cause of Monday morning.** The window is working memory — fast, rich, and erased at the end of the session. Anything you need next week has to be written to disk, in the repo, in a form a fresh agent can read cold.

This is the same principle that drove the saved `PLAN.md` of Chapter 7 and the save-then-restart of Chapter 9, now generalized. There, the handoff crossed a `/clear` within a workday. Here it crosses days, branches, and machines. The artifact is doing what the conversation cannot: it is *durable, lossless on the things you chose to record, and legible to a session that has no memory of how it was produced.* A good handoff artifact answers the question a fresh agent actually has — "what is the state of this work and what do I do next?" — without requiring it to reconstruct anything.

![Two session windows separated by a memory gap, with a durable file layer of NOTES.md, REGISTRY.md, and WORKLOG.md spanning the gap underneath to carry state from session N to session N+1](images/10-multi-session-continuity-and-state-persistence-fig-01.png)

*Figure 10.1 — The agent forgets everything at the gap; the committed-file layer is what survives it, letting the next session resume in one turn instead of thirty.*

Three artifacts cover the common cases. They are not exotic; they are plain Markdown files you commit alongside the code.

## The session-notes handoff

A session-notes file is the note you write to the *next* session — which is to say, to a version of the agent (and of you) that knows nothing. Write it at the end of a session, or just before a restart, while the state is still in the window. The discipline is to write what a stranger would need, because the next session is, functionally, a stranger.

A worked example, saved as `NOTES.md` (or `.agent/session-notes.md`) at the end of the Friday migration session:

```markdown
# Session Notes — user-table migration
_Last updated: 2026-05-29 (Fri), end of session_

## Goal
Migrate the `users` table to the new schema (split `name` into
`given_name` / `family_name`; add `created_at`). Acceptance: all
existing rows migrated, no data loss, app tests green.

## State: where we are
- DONE: `accounts` and `sessions` tables migrated and verified.
- IN PROGRESS: `users` migration written in
  migrations/0007_split_user_name.py but NOT yet run against staging.
- The name-splitting heuristic is in migrations/lib/namesplit.py.
  It handles "First Last" but NOT mononyms or "Last, First" — see below.

## Deferred (agreed, do later)
1. Mononyms (single-token names): currently dumped into `given_name`,
   `family_name` left NULL. We decided this is acceptable for v1.
2. "Last, First" comma format: ~40 rows. Handle in a follow-up.

## Next action
Run the migration in dry-run against staging:
  `python manage.py migrate --plan --database=staging`
Inspect the row count it reports; it MUST equal 18,442 (current users).
Do NOT run for real until the dry-run count matches.

## Rules that still apply
- migrations/ is append-only. Never edit a committed migration.
- vendor/ is read-only.
```

Notice the structure: goal, current state split into done / in-progress, the deferred decisions (so Monday does not relitigate them or treat them as bugs), an unambiguous *next action* with the exact command and the number to check against, and the standing rules that must survive the gap. A fresh agent reading this on Monday can resume in one turn instead of thirty.

The next session begins by reading it:

```
Read NOTES.md. Confirm the current state back to me in three lines,
then perform the "Next action" exactly. Do not start new work.
```

The hard part of a handoff note is not the format; it is the discipline of writing for someone who knows nothing, while you still know everything. At the end of a session you are saturated with context — every constraint is obvious, every decision feels self-explanatory — and that very saturation is what produces a useless note. You write "finish the migration" because *of course* that means run the staging dry-run first and check the row count against 18,442; it is obvious to you, right now. It will not be obvious to Monday. The fix is a concrete test you can apply before you close the session: would a competent stranger, reading only this note and the repo, take the correct next action without guessing? If any step requires a fact that lives only in your head, that fact belongs in the note. The note that passes this test is longer and more explicit than your saturated end-of-session self wants to write — which is exactly why writing it is a discipline and not a formality.

## The registry pattern

Session notes handle one thread of work. When a project has *many* moving pieces — several features in flight, a set of modules each with their own conventions, a list of known issues — a single notes file either bloats past usefulness or forces every session to load state it does not need. The fix is a **registry**: a small index file that names the pieces and points to where each one's detail lives, rather than holding all the detail inline.

This is progressive disclosure (Chapter 5) applied to *state* instead of to rules. The registry is the table of contents; the agent loads only the entry relevant to the current task. A `REGISTRY.md`:

```markdown
# Work Registry
_Index of active work. Load only the entry you need._

| ID    | Thread                  | Status      | Detail / notes file                |
|-------|-------------------------|-------------|------------------------------------|
| W-007 | user-table migration    | in-progress | .agent/notes/W-007-migration.md    |
| W-011 | rate-limit middleware   | blocked     | .agent/notes/W-011-ratelimit.md    |
| W-014 | export-to-CSV feature   | not-started | (spec only) docs/specs/export.md   |

## Conventions index (load on demand)
- Service modules:   see src/services/CONVENTIONS.md
- Migrations:        append-only; see migrations/README.md
- API error shapes:  see docs/error-contract.md

## Blocked / waiting
- W-011 is blocked on the infra team enabling Redis on staging.
  Owner: infra. Do not start until the registry shows it unblocked.
```

The registry keeps each session's loaded context small and current. A session picking up W-007 reads the registry, then loads only `W-007-migration.md` — not the rate-limiter notes, not the export spec. The registry also encodes cross-session facts a single notes file cannot: what is blocked, on whom, and what must not be started yet. It is the project's state-of-the-world, kept short enough to load every session and pointed enough to expand only where needed.

## The durable work log

Session notes say *where we are*; the work log says *how we got here*. It is an append-only record of what was done, decided, and learned, with dates — closer to a captain's log or an ADR trail than to a status file. You append to it; you do not rewrite it.

The work log earns its keep for the questions that the current state cannot answer: *Why* is the name-splitter built this way? *Why* did we defer mononyms? *When* did we decide migrations are append-only, and what went wrong that made us decide it? A `WORKLOG.md`:

```markdown
# Work Log (append-only; newest at top)

## 2026-05-29 — migration name-split decision
Decided to split user names with a simple "First Last" heuristic and
DEFER mononyms + "Last, First". Reason: only ~40 rows are non-standard
and a perfect parser would blow the sprint. Revisit in W-014 follow-up.

## 2026-05-22 — migrations are now append-only
Editing a committed migration (0004) on staging desynced it from prod,
which had already applied the old version. Cost a half-day to untangle.
RULE going forward: migrations/ is append-only, full stop. (Also added
to CLAUDE.md and the PostCompact rule card.)
```

The log is what lets a future session — or a future you — understand a decision instead of merely encountering its consequences and being tempted to "fix" it. It also feeds the other artifacts: the append-only rule that started as a painful Tuesday in the log graduated into CLAUDE.md and the Chapter 8 PostCompact hook. The log is where institutional memory accumulates before it hardens into enforced rules.

## Parallel or sequential: the one rule

Multi-session work raises a scheduling question. You have several pieces in the registry; can you run agents on them at the same time, or must they go one after another? The rule is sharp:

**Run sequentially if and only if task B needs task A's output files. Otherwise, run in parallel.**

![A decision diagram testing two task pairs by the question whether either task reads a file the other writes — a shared output file routes a pair to sequential, while a shared topic but no shared file routes to parallel](images/10-multi-session-continuity-and-state-persistence-fig-02.png)

*Figure 10.2 — The schedule turns on one checkable fact: a shared output file forces sequencing; a merely shared topic does not.*

The dependency is *on files*, not on vague relatedness. Two tasks that touch "the same area of the code" but produce independent outputs can run in parallel. Two tasks where B reads a file A creates — B's migration consumes the model A defines, B's tests import A's new module — must run in sequence, because a parallel B would read a file that does not exist yet or, worse, a half-written one. When you do run in parallel, give each session a clean, separate working context (git worktrees are the standard tool for this, developed in Chapter 11) so the sessions cannot corrupt each other's state.

A concrete test against your registry makes the rule operational. Take W-007 (the user migration) and W-014 (the export-to-CSV feature). Do they share a file dependency? The migration produces the new `users` schema; the export feature reads user data through the application's model layer. If the export code imports the model definitions the migration changes, then W-014 *consumes an output of* W-007 and the two must run in sequence — a parallel export session would build against a schema that does not exist yet. But if the export feature reads through a stable query interface that does not change when the underlying columns split, the two share a topic ("users") without sharing an output file, and they can safely run in parallel. The judgment is not "are these related?" — almost everything in a codebase is related. The judgment is the narrow, checkable one: *does either task read a file the other task writes?* Answer that, and the schedule decides itself.

The reason the rule keys on output files rather than on topic is that *files are the medium of continuity in this whole chapter.* State lives in files; handoffs are files; the dependency that forces sequencing is a file one task produces and another consumes. Get the file dependency right and parallel work is safe; get it wrong and you have two agents fighting over a shared artifact, which is Monday morning multiplied by the number of sessions you launched.

> **The agent is stateless; your repo is not — put the state where it survives**
>
> The reflex after a lost session is to wish the agent had a longer memory. It will not, across sessions, by design — and waiting for that feature is how Monday mornings keep happening. The discipline is to stop using the window as storage and start writing durable artifacts: a handoff that lets a stranger resume, a registry that keeps each session's context small and current, a log that explains decisions long after the conversation is gone. Continuity is not a model capability you are missing. It is a context system you build, file by committed file.

## Exercises (Create)

1. **Write the handoff a stranger could use.** End your next real session by writing a `NOTES.md` with goal, done / in-progress, deferred decisions, an exact next action, and the standing rules. The test: hand it to a teammate (or a fresh agent) cold and ask them to state the next action without asking you a question. Revise until they can.

2. **Build a registry for a multi-thread project.** Create a `REGISTRY.md` for a project with at least three threads of work. Make each entry point to its own detail file rather than inlining it, and include a blocked/waiting section. Run a session that picks up one thread and confirm it loaded *only* that thread's detail.

3. **Start a work log and graduate a rule from it.** Begin an append-only `WORKLOG.md`. Record one real decision with its reason and date. Then find one entry in your log that has become a standing rule, and graduate it: add it to CLAUDE.md and to a PostCompact rule card (Chapter 8). Note the path it took from log to enforced rule.

4. **Apply the parallel/sequential rule.** List three current or planned tasks. For each pair, decide parallel or sequential by the file-dependency test, and write one sentence naming the specific output file that forces (or fails to force) sequencing. Identify which pair, if any, you would have wrongly run in parallel by judging "relatedness" instead of files.

## Bridge to Chapter 11

You can now carry one project's state across many sessions, and you have the file-dependency rule that says when sessions may run at once. But running sessions in parallel raises a harder question than scheduling: when a single task is too large for one context window, how do you split it across *coordinated* agents without each one drowning in the others' context? The next chapter turns the parallel/sequential rule into an architecture — an operator agent that holds only summaries while worker agents do the deep reads, split-and-merge across git worktrees, and the cost discipline of matching model size to job. The artifacts you built here become the shared state those agents coordinate through.

## The AI Wayback Machine: Douglas Engelbart

> **Douglas Engelbart**'s 1962 framework *Augmenting Human Intellect* is usually remembered for the gadgets it eventually produced — the mouse, hypertext, the networked workstation. But the conceptual core was about *memory and continuity*: Engelbart was obsessed with how a knowledge worker could externalize their thinking into a shared, persistent structure that survived the individual work session and accumulated over time. He wanted the by-products of yesterday's work — notes, links, decisions, the trail of how a conclusion was reached — to be captured in a durable structure that today's work could build on, so that effort compounded instead of evaporating. He called the larger ambition *bootstrapping*: a community gets smarter by improving the structures that hold its accumulated knowledge.
>
> That is exactly what a session-notes file, a registry, and a work log do for an agent that forgets everything at the end of each session. The agent has no continuity of its own — so you build the external structure that holds it, and the structure is what compounds. The work log is Engelbart's trail of decisions; the registry is his index into accumulated work; the handoff note is the externalized thinking that lets the next session start where the last one stopped. Engelbart saw, six decades early, the move that defeats Monday morning: the intelligence that persists across sessions does not live in any one mind or any one window — it lives in the durable structure you deliberately maintain outside of them. The agent is stateless; the augmented system is not.

## Sources

- Engelbart, D. C. (1962). *Augmenting Human Intellect: A Conceptual Framework*. SRI Summary Report AFOSR-3223. (Externalized, accumulating knowledge structures; bootstrapping.)
- Yao, S., et al. (2022/2023). ReAct: Synergizing Reasoning and Acting in Language Models. *arXiv:2210.03629*.
- Shinn, N., et al. (2023). Reflexion: Language Agents with Verbal Reinforcement Learning. *arXiv:2303.11366*. (Self-generated state across attempts.)
- Yang, J., et al. (2024). SWE-agent: Agent-Computer Interfaces Enable Automated Software Engineering. *NeurIPS 2024*.
- Liu, N. F., et al. (2024). Lost in the Middle: How Language Models Use Long Contexts. *TACL*, 12, 157–173. (Why handoff files must be short and pointed.)
- Anthropic. *Claude Code documentation* (current online docs; accessed 2026-05). Session management, memory files, worktrees. [Current-state — verify before print.]
- HumanLayer. (2025). *Writing a Good CLAUDE.md* (practitioner source). Field guidance on durable state files and handoffs.

## Tags

#cli-agents #multi-session #state-persistence #session-notes #registry #work-log #parallel-vs-sequential #Engelbart
