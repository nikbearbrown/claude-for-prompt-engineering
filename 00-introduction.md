# Introduction

A learner opens the first chapter of *Prompt Engineering for CLI AI Coding Agents* with a familiar problem: there is too much information and not enough structure. The terms are available. The examples are available. The missing thing is a route through the material that turns exposure into understanding.

This book is about the gap between knowing the name of Prompt Engineering for CLI AI Coding Agents's subject and being able to use its ideas with judgment.

The central argument is that Prompt Engineering for CLI AI Coding Agents is best learned as a sequence of distinctions, practices, and recurring problems rather than as a list of topics. A reader who can name those distinctions can move through the field with more confidence than a reader who has only memorized definitions.

This is written for learners, teachers, practitioners, and builders who want a clear path through the material.

## What This Book Is

This book is a structured introduction to Prompt Engineering for CLI AI Coding Agents. It teaches the vocabulary of the field, shows how the main ideas connect, and gives readers enough conceptual grip to continue with more specialized work. It is designed to be read as a book, used as a reference, and integrated into an intelligent textbook system.

## What This Book Is Not

This book is not a substitute for practice, mentorship, experimentation, or domain-specific judgment. It does not try to say everything. It tries to say enough, in the right order, so that the reader can recognize what matters next.

## The Concept Running Through the Book

The recurring idea is transfer: the movement from explanation to usable understanding. Each chapter should help the reader carry an idea from the page into a problem, a classroom, a project, or a decision.

## How This Book Is Organized

- **Chapter 1: Chapter 1 — The Agentic Coding Loop.** *Read, reason, act, observe — and why the observe step is the whole game in code* A coding agent runs four moves in sequence, over and over. ![A four-stage cyclic process. READ at top, REASON at right, ACT at bottom, OBSERVE at...
- **Chapter 2: Chapter 2 — Setting Up a Coding Agent for a Real Repository.** *Writing the onboarding map and wiring the verification loop so the agent can close it itself* The reframe that makes everything else intuitive: a persistent instruction file is the answer to the question *"what would I tell a competent new engineer on...
- **Chapter 3: Chapter 3 — A Taxonomy of Agentic Coding Tasks.** *Classifying a coding task by where its oracle is — and predicting the failure before the agent runs* Every coding task sits somewhere on a plane defined by two axes. ![A classification plane: the horizontal axis runs bounded to open-ended, the vertical...
- **Chapter 4: Chapter 4 — Test-Driven Agentic Development.** *The test is the loop — and a loop the agent controls is no loop at all* The loop is Kent Beck's, unchanged in shape. In *Test-Driven Development: By Example* (2003), the cycle is: write a small failing test (red), write just...
- **Chapter 5: Chapter 5 — Refactoring and Large-Scale Code Transformation.** *Driving a mechanical multi-file change with scope discipline and a behavior-preservation oracle* Before trusting an agent with a refactor, you should know its actual track record. Liu et al. (2024) evaluated LLM refactorings against 180 real-world refactorings performed by human experts. ![A...
- **Chapter 6: Chapter 6 — Debugging with Agents.** *Reproduce → localize → fix → verify, with external grounding instead of introspection* The durable structure of debugging — agentic or human — is a four-stage loop, and the reason to name the stages is that each one must produce a *concrete...
- **Chapter 7: Chapter 7 — Greenfield Builds: From Specification to Working Software.** *Manufacturing the verification loop up front, because empty space has no ground truth* Here is something I want you to think about. Every other task an agent does in this book comes with something to push against. A failing test. A stack...
- **Chapter 8: Chapter 8 — Code Review, Pull Requests, and the Human Gate.** *Routing the mechanical to the loop and the judgment-laden to the human who owns the merge* Here is a finding that should reset your priors, and I want to start with it rather than bury it in a survey of techniques. Pornprasit...
- **Chapter 9: Chapter 9 — Migrations and Dependency / Framework Upgrades.** *Shard the change so each unit has its own pass/fail loop — and never advance a red shard* Google has been running migrations larger than your entire codebase for over a decade, and the infrastructure they built to do it safely is...
- **Chapter 10: Chapter 10 — Multi-Agent and Multi-Repo Engineering.** *Multi-agent multiplies the number of verification loops, not their nature — and adds one new loop nobody can close alone: the merge* There is a mistake that looks like ambition. An engineer reads about multi-agent systems and immediately reaches for one. The...
- **Chapter 11: Chapter 11 — Evaluating Coding Agents.** *pass@k flatters, pass^k is production: best-of-k versus all-of-k is the whole measurement story* Suppose your agent has a 90% chance of solving a task on any given attempt. You run it eight times and ask: did at least one of the eight...
- **Chapter 12: Chapter 12 — Security and Trust in Agentic Coding.** *The lethal trifecta applied to code, and why the sandbox — not the system prompt — is the boundary* In August 2025, Check Point Research disclosed a vulnerability in OpenAI's Codex CLI with a CVSS score of 9.8 — the high end...
- **Chapter 13: Chapter 13 — Production Engineering with Agents.** *Putting an agent in the pipeline, pricing its inconsistency, and deciding when not to use one* Here is the pattern, and I want to describe it precisely because the marketing describes something else. You assign a GitHub issue to a coding agent....
- **Chapter 14: Chapter 14 — Capstone: Shipping a Real Feature End-to-End.** *The whole book, executed once and then measured — spec to merged, with a reliability number attached* The spec is the primary artifact. In spec-driven development, you generate code, tests, and docs *from* the spec — not as an afterthought after the...

## How to Read This Book

Read the chapters in order if you are new to the subject. If you already know the area, use the chapter titles as a map and move directly to the parts where your understanding is weakest. The chapters are designed to be self-contained enough for reference, but they work best as a progression from Chapter 1 — The Agentic Coding Loop to Chapter 14 — Capstone: Shipping a Real Feature End-to-End.

## A Note About AI

AI matters to *Prompt Engineering for CLI AI Coding Agents* because the modern textbook is no longer only a static container. It is also part of a learning system: searchable, remixable, explainable, and increasingly connected to tools such as Medhavy. For Bear Brown books, the relevant question is not whether AI can replace the learner or the teacher. It cannot. The useful question is what AI can make easier to inspect: definitions, worked examples, misconceptions, practice sequences, alternate explanations, and the structure of an argument. This book treats AI as infrastructure for practical AI-assisted authorship, analysis, and production. The chapters should still stand on their own as readable prose, but they are also designed to be legible to an intelligent textbook system.

## Closing Return

The learner at the opening does not need more noise. They need a path. This book is that path: not the whole territory, but a reliable way to begin moving through it.

Let's go.

## Tags

Prompt Engineering for CLI AI Coding Agents, textbook, Medhavy, AI-assisted learning, Bear Brown
