# Chapter 14 — Prompt Engineering for CLI Coding Agents: Why Context Is the Bottleneck
*The frontier model inside the loop is rarely the limiting factor — what limits it is what it knows at a given step.*

---

A Wordsville engineer opens a coding agent in the project repository to fix a bug in the dialogue renderer. The first ten minutes go cleanly. The agent reads the renderer module, locates an off-by-one in a loop that truncates the last line of NPC dialogue, proposes a one-line fix, runs the test suite, and reports green. Good.

Then the engineer keeps going in the same session. "While you're here, also fix the inventory serializer." The agent reads three more files, tries an approach, hits a failing test, tries another, hits a different one, reads two more files to understand why, finds a third approach, and lands it. "Now the save-game migration." More reads. A failed attempt. A correction. Another failed attempt. A correction to the correction.

By the time the engineer says "okay, now go back and also handle the empty-dialogue edge case in the renderer" — the very module the agent fixed cleanly an hour ago — the agent does something strange. It re-edits the renderer using the old, buggy loop bound, reintroducing the off-by-one it had already fixed. It cites a test command the project does not use. It writes a helper function that duplicates one it read forty minutes earlier. Nothing in its output announces a problem. The prose is confident. The diffs look plausible.

The engineer's instinct is to blame the model: it "got dumber," or "got tired," or "lost the thread." None of that is what happened. The model is exactly as capable as it was in minute one. What changed is the context it is reasoning from. The window now holds a sediment of failed attempts, superseded approaches, stale file contents that no longer match disk, and corrections layered on corrections. The agent is not confused in the human sense. It is working faithfully from a context that no longer represents the current state of the world. This failure mode is called **context rot**, and it has no equivalent in one-shot chat, because chat has no accumulating state to rot.

The diagnostic move that proves it: open a fresh session and ask the agent, with no history, to handle the empty-dialogue edge case in the renderer. It does it cleanly, the way it did in minute one. Same model, same task, different context — and the difference is the whole story.

That is this chapter's thesis: **in CLI coding agents, context is the bottleneck, not intelligence.** Frontier models are capable enough. The failures cluster around what the model knows at a given step, how many instructions it is being asked to hold at once, and whether the context has been polluted. The engineering discipline that follows is not about wording. It is about designing and defending the context system the agent reasons from.

---

## Statefulness and the token budget

Start with the structural difference, because everything else follows from it.

A chat prompt is a single instruction with a single response. A CLI coding agent runs a loop: it reads a file, reasons about what it read, takes an action — an edit, a shell command, a tool call — observes the result, and repeats. Each pass appends to a context the engineer is responsible for managing. The agent is not answering a question; it is conducting an investigation whose notes pile up in front of it.

This is the same Thought→Action→Observation loop from Chapter 11, now running against a filesystem. The new fact at coding-agent scale is that the loop fills the context fast. A single bug investigation can read a dozen files, run the test suite three times, and accumulate several failed-attempt diffs — tens of thousands of tokens, none of which the model chose to forget.

Practitioner experience and Anthropic's own auto-compaction defaults converge on a working rule of thumb: output quality begins to degrade as the context approaches roughly 80% of capacity. Treat that as a soft architectural constraint rather than a sharp cliff. The practical implication is that you should plan to restart or compact around 80% rather than watch quality erode while you fill the last fifth.

Let $C$ be the context window size in tokens and $u(t)$ the tokens consumed after $t$ loop iterations. Useful work continues while:

$$u(t) < 0.8\,C$$

The trouble is that $u(t)$ is monotonically increasing — the loop only ever adds. Every file read, every tool result, every error message, every prior turn stays in the window unless you explicitly remove it. So the engineering question is not "how do I make the model smarter" but "how do I keep $u(t)$ below the degradation threshold while still giving the agent what it needs." Two levers: put less in (be selective about reads, prefer pointers to copies) and take stale material out (compaction, fresh sessions). Both are context engineering, not prompt wording.

![The stateful loop and the monotonic token budget: each read, tool result, and prior turn only adds to the window, so consumption rises monotonically toward the ~80% degradation line — the levers are putting less in and taking stale material out.](../images/14-prompt-engineering-for-cli-coding-agents-fig-01.png)
*Figure 14.1 — The stateful loop and the monotonic token budget*

This connects directly to Chapter 10. Long-context models do not attend uniformly across the window; instructions placed early lose salience as later tokens accumulate, and a window stuffed with near-duplicate or contradictory material produces the multi-needle collapse measured there. Context rot is the agentic, accumulating version of that effect: the renderer fix was still in the window when the agent re-broke it, but buried under an hour of superseded material, competing for attention with everything that came after.

The common belief is that a bigger context window solves this — that a million-token window means you never have to manage context. It does not. A bigger window raises $C$, which buys more iterations before the 80% line, but does nothing about the quality of attention over a polluted window. A window full of stale reads and failed attempts is a worse working context at one million tokens than a clean window at one hundred thousand. Capacity is not the same as signal.

---

## Persistent instruction files: the onboarding map

In chat, the system prompt carries your standing instructions. A CLI agent has a system prompt too, but you usually cannot edit it — it belongs to the tool. The lever you do control is a **persistent instruction file** that the agent reads at session start and re-reads during operation: `CLAUDE.md` for Claude Code, the emerging cross-tool `AGENTS.md` standard, `.cursorrules` for Cursor and Windsurf. These files survive across the loop's re-reads, compaction, and summarization in a way that a single opening message does not.

### Where the file actually lands

In Claude Code, `CLAUDE.md` is not injected as the system prompt. It is delivered as a `<system-reminder>` block inside a user message, after the real system prompt, carrying a caveat: this context may or may not be relevant; you should not respond to it unless it is highly relevant.

Read that caveat carefully. Your instructions are influential, not absolute. They sit in user-message position, which means the model weighs them against everything else in the window rather than treating them as inviolable. And the agent is explicitly instructed to ignore sections it judges irrelevant — a guardrail added after narrow, hotfix-laden instruction files were observed to degrade performance by injecting task-specific noise into every unrelated request. The consequence for design is sharp: only universally applicable instructions belong in the file. A rule that matters for one module, dropped into the global file, is either ignored when irrelevant or misapplied when the model wrongly judges it relevant. Vague or conflicting rules get applied inconsistently for the same reason.

### The instruction-capacity ceiling

There is a second, harder limit. A model can only follow so many distinct instructions before adherence degrades. Practitioner research circulating in 2025 (via HumanLayer) puts the ceiling for frontier thinking models at roughly 150 to 200 distinct instructions before adherence breaks down; smaller or non-thinking models decay faster. Crucially, the agent's own system prompt is reported to consume on the order of 50 of those slots before your file ever loads — leaving perhaps 100 to 150 for you. These are practitioner figures, not numbers from a controlled published study — treat them as order-of-magnitude guidance.

The dangerous consequence: past the threshold, models are reported to ignore instructions roughly uniformly, not selectively. It is not that your 201st instruction gets dropped while the first 200 hold. All of them start to slip. A bloated instruction file is worse than a short one even for the rules it shares with the short version: the bloat degrades adherence to everything, including the rules you cared about most. The mechanism rhymes with the long-context dilution of Chapter 10 — too many competing directives flatten the salience of each.

The practical ceiling: roughly 80 to 120 lines, and under 100 is better. The file is an onboarding map, not a manual.

![The instruction-capacity budget: a model can follow only so many distinct instructions before adherence degrades across the board — the agent's own system prompt consumes part of the budget before your CLAUDE.md ever loads, so a bloated file hurts even the rules you cared about.](../images/14-prompt-engineering-for-cli-coding-agents-fig-02.png)
*Figure 14.2 — The instruction-capacity budget*

### What belongs, what does not

The map answers three questions. **WHAT** — the tech stack, the project structure, a directory map: where things live. **WHY** — the purpose and design intent, so the agent can make judgment calls in situations the rules do not cover. **HOW** — the exact verification commands: not "run the tests" but the literal command string.

What does not belong: code-style rules (a linter enforces those far more reliably); task-specific instructions (ignored when off-task, costly when wrongly applied); commands the agent can discover for itself; and anything that could be loaded on demand from a sub-file.

### Progressive disclosure: the `agent_docs/` pattern

Keep `CLAUDE.md` minimal and make it an index that points to deeper documents the agent reads only when relevant:

```
agent_docs/
  building.md
  running_tests.md
  code_conventions.md
  service_architecture.md
  database_schema.md
```

The agent loads `database_schema.md` when it is touching the database and ignores it otherwise. This is progressive disclosure: the standing context stays small — protecting both the instruction budget and the token budget — while depth is available on demand.

Prefer pointers to copies. A path or a file-plus-line-number reference stays correct as the code evolves; a copied snippet goes stale the moment someone edits the original, and a stale snippet in your instruction file is a context-rot generator you built on purpose.

One cheap trick that exploits Chapter 10's recency effect directly: restate your most critical rules at the end of the file as well as the top. Claude Code's own system prompt repeats its hardest constraints at the bottom for exactly this reason — the most recent tokens carry disproportionate salience.

| Category | Belongs? | Why |
|---|---|---|
| Project structure & stack | Yes — **WHAT** | Where things live; the onboarding map |
| Test & build commands | Yes — **HOW** | The literal verification command strings |
| Design intent | Yes — **WHY** | Lets the agent make judgment calls the rules don't cover |
| Code-style rules | No | A linter enforces these far more reliably |
| Hotfix / task-specific notes | No | Ignored off-task, costly when wrongly applied — put in `agent_docs/` |
| Pasted schema / interfaces | No | Goes stale the moment the source changes — use a pointer to `agent_docs/` |

---

## Static context versus dynamic context

Two kinds of context fail in two different ways, and confusing them sends you fixing the wrong thing.

**Static context** is what you set before the session: the system prompt, `CLAUDE.md`, the rules files. Its characteristic failure is consistent convention violations — the agent always names things the wrong way, always misses a required step — because the standing instruction is missing, wrong, or being judged irrelevant.

**Dynamic context** is what arrives during the loop: file contents, tool results, conversation history. Its characteristic failure is acting on stale or incomplete information — the agent edits against a version of a file that has since changed, or never read the file that mattered. Context rot is a dynamic-context failure.

The fix differs by kind. A consistent convention violation is a static-context bug: fix the instruction file. An acting-on-stale-info failure is a dynamic-context bug: no amount of rewriting `CLAUDE.md` fixes it, because the rule was fine — the working context was wrong. Diagnose which kind you have before you touch anything.

For large or multi-package repositories, the instinct is to give the agent all the context — dump the repo into the window so it knows everything. This blows the token budget, dilutes attention, and seeds context rot. The disciplined moves: directory-scoped instruction files, one per package, loaded when the agent works in that package; on-demand architecture docs loaded when relevant; and when you need to understand several modules at once, sub-agents analyzing each in parallel, returning summaries to a coordinator that holds the summaries rather than the full file contents. The coordinator reasons over compressed findings while each worker burns its own disposable context on the raw material. The agent should read what it needs, when it needs it, and no more.

---

## Plan in one session, execute in another

This is the single highest-leverage workflow change in the chapter, and it follows directly from the monotonic-fill problem.

Exploration is dirty. To plan a non-trivial change, the agent reads widely, considers approaches, and rejects most of them. That rejected material — the tangents, the dead ends, the "actually no" reversals — is exactly the sediment that produces context rot. If you then execute in the same session, the agent executes from a window full of the approaches it already discarded. The renderer regression in the opening scenario is this failure: an hour of superseded exploration buried the clean answer.

The fix is to quarantine exploration from execution.

1. **Plan** in one session. Let the agent explore freely, read what it needs, and produce a written plan.
2. **Save the plan** to a markdown file — the durable artifact.
3. **`/clear` or open a fresh session** — discard the polluted exploratory context entirely.
4. **Execute** from only the plan document, in a clean window.

The execution session never sees the dead ends. It sees a clean plan and a clean window, which is the condition under which the agent performed well in minute one.

### What a good execution prompt specifies

A plan an agent can execute cleanly names four things. **Scope** — which files are in bounds and which are out: "only touch `src/render/`; ask before editing anything else." **Stopping condition** — what "done" looks like, concretely, and how to check it: not "fix the bug" but "the empty-dialogue case renders without truncation and `npm test` is green." **Acceptance criteria** — the exact verification commands. **Workflow** — the loop discipline: inspect before editing, make the smallest change that satisfies the criteria, run the verification command, then stop and summarize the diff.

For repeated workflows, capture them as reusable slash commands (`.claude/commands/*.md`) so the discipline is encoded once rather than retyped.

### The two-correction rule

When an execution attempt fails, the temptation is to correct it in place. After one or two corrections, stop. Each correction layers more superseded reasoning into the window — you are correcting your way into context rot. The practical rule: after two failed corrections, `/clear` and write a fresh prompt that folds in what you learned. Do not correct out of a polluted session; start a clean one with better instructions. This is the plan→execute split applied at small scale, mid-task.

![The plan-then-execute split: Session A explores freely, rejects approaches, and writes plan.md; then `/clear` discards the polluted context so Session B executes from a clean window and the plan alone — the dead ends never reach the execution session.](../images/14-prompt-engineering-for-cli-coding-agents-fig-03.png)
*Figure 14.3 — The plan-then-execute split*

---

## Multi-session continuity and injection security

Two problems remain, and they are the ones the loop's statefulness forces on you.

### Continuity: the model is stateless across sessions

`/clear` is a feature, but it cuts both ways. Whatever the agent "knew" at the end of session A is gone at the start of session B unless you persisted it explicitly. Three patterns do this. Compaction: `/compact` summarizes the history while preserving critical state, freeing budget without a full reset. Session notes: write a status file at the end of a session (files changed, tests run, open risks, follow-ups) and read it at the start of the next. Registry and sub-agent patterns: a coordinator keeps a date-stamped registry of what has been done; sub-agents return summaries. These tie back to Chapter 9: a handoff note is only useful if it accurately records what actually happened, which means verifying the agent's claims rather than trusting its summary.

### Security: the lethal trifecta

Now the hard part. A chat model reads what you send it. A CLI coding agent reads the filesystem — and a file is an instruction-delivery vector. Every README, every code comment, every dependency's docstring, every test fixture, every `.env` file is text the agent ingests, and any of it can carry instructions aimed at the agent rather than at you. The attack surface is vastly larger than chat's, and it expands further the moment the agent can read issues, email, or the web.

The framing that organizes this comes from Simon Willison: the **lethal trifecta**. An agent is exploitable, regardless of how well the model is hardened, when all three of the following are true: it has access to private data (your repo, your secrets, your environment); it is exposed to untrusted content (file contents, dependencies, web results it did not author); and it can communicate externally (network calls, writing files, opening PRs, running shell commands). A default CLI coding agent has all three.

![The lethal trifecta: an agent is exploitable when it has access to private data, is exposed to untrusted content, and can communicate externally — the danger is the structural combination, which prompting cannot remove.](../images/14-prompt-engineering-for-cli-coding-agents-fig-04.png)
*Figure 14.4 — The lethal trifecta*

The danger is not a clever individual exploit but the structural combination. If an attacker can get text in front of the agent while the agent can read your secrets and make a network call, the attacker can in principle read and exfiltrate. Documented patterns include file-content injection that reads and exfiltrates secrets, poisoned editor config achieving code execution, and malicious MCP server entries reaching arbitrary command execution. A single 2025 disclosure reported 30+ vulnerabilities across six tools (Cursor, Roo Code, JetBrains Junie, Copilot, Kiro.dev, Claude Code). OWASP's 2025 Top-10 for LLM Applications lists prompt injection at number one, and notes there is no foolproof model-layer prevention.

**What prompting can do.** You can reduce blast radius. Instruct the agent to treat repository text as untrusted data, never to follow instructions embedded in files, to make conservative writes, to require confirmation before destructive operations, never to print secrets, and to stay within an explicit scope. These are real mitigations.

**What prompting cannot do.** Against a determined attacker, prompting cannot prevent injection. An instruction in your file is influential, not absolute — and so is an instruction in the attacker's file. You cannot out-prompt a structural vulnerability. The trifecta is a property of the architecture, so the defense must be architectural too. Run the agent in a container or restricted filesystem. Disable auto-approve on untrusted repos — a human approves shell commands and writes when the code is not yours. Limit which tools the agent can call. Require human review of irreversible actions.

The operating rule: treat any repository you did not author as hostile, and never enable auto-approve on an unfamiliar codebase. The model is not your security boundary. The architecture is.

A misconception to retire: "a better, more aligned model will solve prompt injection." It will not, any more than a better-behaved employee solves a building with no locks. As long as the three legs of the trifecta are present, the exposure is present. Hardening the model raises the cost of an attack; it does not remove the surface.

---

## Three failures, one question

The three failure modes this chapter teaches look similar from the outside — the agent does something wrong, confidently — but they have different causes and different fixes.

![The three-failure diagnostic router: a single question — "does it work in a fresh session?" — separates context rot and injection (state-dependent) from instruction overflow (fix the file, not the session).](../images/14-prompt-engineering-for-cli-coding-agents-fig-05.png)
*Figure 14.5 — The three-failure diagnostic router*

| Symptom | Likely cause | Diagnostic test | Fix |
|---|---|---|---|
| Clean early, degrades as the session runs long; re-breaks fixed code; cites stale commands | Context rot (dynamic) | Does it work cleanly in a fresh session? | `/clear`, plan→execute split, compaction |
| Ignores rules from turn one, unpredictably, even on short tasks | Instruction overflow (static) | Is the instruction file over ~100 lines? | Cut to an onboarding map; move detail to `agent_docs/` |
| Takes an unasked action after reading an unfamiliar file; touches secrets or the network | Prompt injection | Did it just read untrusted content while holding the trifecta? | Sandbox, revoke auto-approve, scope the tools |

The single question that routes you: **does it work in a fresh session?** If yes, you have context rot or injection — state-dependent failure. If it fails even fresh and clean, you have an instruction problem or a task-beyond-scope: fix the file, not the session. Run that test first. It is the agentic analog of Chapter 6's single-turn isolation test and Chapter 9's insistence on controlled comparison.

---

## LLM Exercises

**Exercise 1 — Generate and examine.** Recreate the opening scenario at small scale: run a coding agent on a simple fix, then without clearing the session add two unrelated tasks, then ask the agent to revisit the first fix. Record whether it reintroduces the original bug or cites stale information. Then clear the session and run only the final step. Write two sentences comparing the results in terms of the $u(t)$ model.

**Exercise 2 — Apply to known context.** You are handed a 300-line `CLAUDE.md` (use any real or constructed example with code-style rules, hotfix notes, pasted schema, and buried test commands). Rewrite it to under 100 lines as a genuine onboarding map — WHAT, WHY, HOW. Create an `agent_docs/` index for at least three files showing what you moved out and why. Replace at least two pasted snippets with pointers. In one paragraph, identify which cuts protect the instruction budget and which protect the token budget.

**Exercise 3 — Stress-test a claim.** Run the same non-trivial refactor task two ways: entirely in one long session with corrections, and as a plan→execute split with the two-correction-then-clear rule. Track approximate context usage and count regressions or stale-reference errors in each. Report whether the split's overhead paid for itself and at what task size you would expect the answer to flip.

**Exercise 4 — Draft a professional deliverable.** Produce a `CLAUDE.md` plus task-prompt pair for a scoped change in a real repository you have access to. The `CLAUDE.md` must include a Security section naming the lethal trifecta and stating a permission posture. The task prompt must specify scope (paths in and out), a stopping condition with a concrete verification command, acceptance criteria, and the inspect→plan→minimal-change→verify→summarize workflow.

---

## References

- Willison, S. (2025). The lethal trifecta for AI agents: private data, untrusted content, and external communication. simonwillison.net, Jun 16, 2025. https://simonwillison.net/2025/Jun/16/the-lethal-trifecta/
- Anthropic (2025). Effective context engineering for AI agents. Anthropic Engineering blog. https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents
- Anthropic (2025). Equipping agents for the real world with Agent Skills. https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills
- HumanLayer (2025). Writing a good CLAUDE.md. *(Practitioner write-up; instruction-capacity figures are not from a controlled study.)*
- Yang, J., et al. (2024). SWE-agent: Agent-Computer Interfaces Enable Automated Software Engineering. NeurIPS 2024.
- Zhan, Q., et al. (2024). InjecAgent: Benchmarking Indirect Prompt Injections in Tool-Integrated Large Language Model Agents. ACL 2024.
- OWASP (2025). Top 10 for Large Language Model Applications — prompt injection ranked LLM01. https://genai.owasp.org/llmrisk/llm01-prompt-injection/
- VILA-Lab. Dive into Claude Code: The Design Space of Today's and Future AI Agent Systems. arXiv:2604.14228.

---

## Prompts

Use these prompts with Claude to generate interactive D3 v7 versions of the figures in this chapter. Each produces a standalone HTML file you can open in a browser and modify freely.

**Prerequisites:** Load `NEU/CLAUDE.md` and `NEU/DESIGN.md` into your Claude project context before using these prompts. They define the stack, naming conventions, color system, and typography the figures use.

---

### Figure 14.1 — The stateful loop and the monotonic token budget

A loop + budget diagram, single HTML file, inline CSS, D3 v7 from the CDN. Left: a read → reason → act → observe loop that appends to context each pass. Right: a rising line u(t) of token consumption approaching a dashed "~80% degradation" threshold in red, zero baseline. Caption: the loop only ever adds — manage the budget, not the model.

> Reference implementation: `d3/14-prompt-engineering-for-cli-coding-agents-fig-01.html`

---

### Figure 14.2 — The instruction-capacity budget

A stacked-budget bar, single HTML file, D3 v7 CDN. A horizontal capacity bar (~150–200 instruction "slots") with one segment consumed by the system prompt (~50) and the remainder available for your CLAUDE.md. Mark an overflow zone in red where adherence degrades across the board. Caption: past the ceiling, all instructions start to slip — not just the last one.

> Reference implementation: `d3/14-prompt-engineering-for-cli-coding-agents-fig-02.html`

---

### Figure 14.3 — The plan-then-execute split

A two-session cycle, single HTML file, D3 v7 CDN. Session A (explore widely, reject approaches, write plan.md) → an arrow labeled "/clear — discard polluted context" (in red) → Session B (fresh window, load only plan.md, execute with scope + stopping condition + verification). Caption: the dead ends stay in Session A; the execution session never sees them.

> Reference implementation: `d3/14-prompt-engineering-for-cli-coding-agents-fig-03.html`

---

### Figure 14.4 — The lethal trifecta

A three-circle Venn diagram, single HTML file, D3 v7 CDN. Circles: "private data," "untrusted content," "external communication." The central intersection, in red, is labeled "exploitable." Caption: the exposure is the structural combination, not any single leg — prompting cannot remove it.

> Reference implementation: `d3/14-prompt-engineering-for-cli-coding-agents-fig-04.html`

---

### Figure 14.5 — The three-failure diagnostic router

A decision flow, single HTML file, D3 v7 CDN. Root question "does it work in a fresh session?": yes → context rot or injection (state-dependent); no → instruction overflow / out-of-scope task (fix the file). Branch to the three fixes. Red marks the root routing test. Caption: run the fresh-session test first.

> Reference implementation: `d3/14-prompt-engineering-for-cli-coding-agents-fig-05.html`
