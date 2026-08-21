# Introduction

A learner opens the first chapter of *Prompt Engineering with CLIs* with a familiar problem: there is too much information and not enough structure. The terms are available. The examples are available. The missing thing is a route through the material that turns exposure into understanding.

This book is about the gap between knowing the name of Prompt Engineering with CLIs's subject and being able to use its ideas with judgment.

The central argument is that Prompt Engineering with CLIs is best learned as a sequence of distinctions, practices, and recurring problems rather than as a list of topics. A reader who can name those distinctions can move through the field with more confidence than a reader who has only memorized definitions.

This is written for learners, teachers, practitioners, and builders who want a clear path through the material.

## What This Book Is

This book is a structured introduction to Prompt Engineering with CLIs. It teaches the vocabulary of the field, shows how the main ideas connect, and gives readers enough conceptual grip to continue with more specialized work. It is designed to be read as a book, used as a reference, and integrated into an intelligent textbook system.

## What This Book Is Not

This book is not a substitute for practice, mentorship, experimentation, or domain-specific judgment. It does not try to say everything. It tries to say enough, in the right order, so that the reader can recognize what matters next.

## The Concept Running Through the Book

The recurring idea is transfer: the movement from explanation to usable understanding. Each chapter should help the reader carry an idea from the page into a problem, a classroom, a project, or a decision.

## How This Book Is Organized

- **Chapter 1: Chapter 1 — Why CLI Agent Prompting Is a Different Discipline.** It is the third hour of a single agent session. The task was reasonable — refactor an authentication module, update its tests, and migrate the two call sites that use it. The first hour was a pleasure to watch. The agent read...
- **Chapter 2: Chapter 2 — The Agent's Context System: Static vs. Dynamic Context.** Same agent, same afternoon, two bug reports from a teammate. The first: "It keeps using tabs. We're a spaces shop — it's in the style guide." You check. The rule *is* in `CLAUDE.md`, plainly written. The agent followed it for the first...
- **Chapter 3: Chapter 3: Persistent Instruction Files — CLAUDE.md and AGENTS.md.** You open a fresh session on a service you maintain. You have done the responsible thing. Two months ago, after the agent kept reaching for `npm` in a `pnpm` repository, you wrote it down. You opened the project's `CLAUDE.md` and added the...
- **Chapter 4: Chapter 4: The Instruction-Capacity Budget.** A team has a `CLAUDE.md` they are proud of. It started clean — the kind of file Chapter 3 would endorse. Then it grew, the way good files do, one reasonable addition at a time. A rule about commit message format. A...
- **Chapter 5: Chapter 5: Progressive Disclosure and the agent_docs Pattern.** You finished Chapter 4 with a relocation table full of debts. The auth-decorator rule survived. So did "use the repository pattern." But a dozen entries said *move this somewhere else* — the full procedure for adding a payment provider, the seven gotchas...
- **Chapter 6: Chapter 6: Context Engineering for Large Codebases.** A senior engineer hands a CLI agent a task that should take twenty minutes: a partner's API changed a field name from `customerId` to `customer_id`, and the integration needs to follow. The repository is a monorepo — three thousand files, eleven services,...
- **Chapter 7: Chapter 7 — Task Prompt Design: Scope, Stopping Conditions, and Plan-then-Execute.** You ask the agent to "fix the flaky checkout test." It reads the failing test, reads the module under test, then reads the module's imports, then the configuration loader those imports pull in, then the environment shim the loader references. Forty minutes...
- **Chapter 8: Chapter 8 — Reusable Workflows: Slash Commands and Hooks.** It is Thursday, and for the third time this week you type the same paragraph into the agent. "Before you commit: run the full test suite, run the linter, run the type checker, and if any of them fail, stop and show...
- **Chapter 9: Chapter 9 — Context Management and Compaction.** You are six hours into a refactor that is going well. The agent understands the module, the tests are green, and you are deep in the satisfying rhythm of describing a change and watching it land. Then, without much ceremony, the work...
- **Chapter 10: Chapter 10 — Multi-Session Continuity and State Persistence.** You close the laptop on Friday with the migration half-done. The agent had a clear picture: which tables were already converted, which one was mid-flight and why it was tricky, the two edge cases you had agreed to handle later, the exact...
- **Chapter 11: Chapter 11 — Sub-Agents and Multi-Agent Orchestration.** You are three hours into a migration. The task is to move forty-one service modules from one logging library to another — mechanical, repetitive, but spread across a sprawling repository whose conventions you half-remember. You started in a single Claude Code session,...
- **Chapter 12: Chapter 12 — Security: Prompt Injection and the Lethal Trifecta.** A teammate asks you to review a pull request from an outside contributor. You point your agent at the branch: "Read the changed files, summarize what this PR does, and flag anything suspicious." The agent reads the diff. Buried in a markdown...
- **Chapter 13: Chapter 13 — Tool-Specific Practice.** A colleague who has read the first twelve chapters of this book sits down at a new job. The previous shop ran Claude Code; the new one runs Cursor and Aider. She knows the discipline cold — short scoped instruction files, the...
- **Chapter 14: Chapter 14 — Failure Modes, Templates, and Diagnostics.** It is the last week of the course, and a student drops a transcript in front of you. Her agent was supposed to add pagination to an API endpoint. Instead it rewrote three unrelated files, "fixed" a test by deleting its assertions,...

## How to Read This Book

Read the chapters in order if you are new to the subject. If you already know the area, use the chapter titles as a map and move directly to the parts where your understanding is weakest. The chapters are designed to be self-contained enough for reference, but they work best as a progression from Chapter 1 — Why CLI Agent Prompting Is a Different Discipline to Chapter 14 — Failure Modes, Templates, and Diagnostics.

## A Note About AI

AI matters to *Prompt Engineering with CLIs* because the modern textbook is no longer only a static container. It is also part of a learning system: searchable, remixable, explainable, and increasingly connected to tools such as Medhavy. For Bear Brown books, the relevant question is not whether AI can replace the learner or the teacher. It cannot. The useful question is what AI can make easier to inspect: definitions, worked examples, misconceptions, practice sequences, alternate explanations, and the structure of an argument. This book treats AI as infrastructure for practical AI-assisted authorship, analysis, and production. The chapters should still stand on their own as readable prose, but they are also designed to be legible to an intelligent textbook system.

## Closing Return

The learner at the opening does not need more noise. They need a path. This book is that path: not the whole territory, but a reliable way to begin moving through it.

Let's go.

## Tags

Prompt Engineering with CLIs, textbook, Medhavy, AI-assisted learning, Bear Brown
