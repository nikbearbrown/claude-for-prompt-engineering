# Chapter 13 — Tool-Specific Practice

A colleague who has read the first twelve chapters of this book sits down at a new job. The previous shop ran Claude Code; the new one runs Cursor and Aider. She knows the discipline cold — short scoped instruction files, the capacity budget, progressive disclosure, plan-then-execute, the injection surface. And for an hour she is useless, because she keeps reaching for `CLAUDE.md` and `/compact` and they are not there. The *principles* transferred perfectly. The *filenames, byte caps, and commands* did not.

That hour is the entire reason this chapter exists, and the entire reason it is quarantined. Everything you have learned so far is durable. This chapter is the layer that is not. It is a **reference chapter**, and it ages faster than anything else in the book: filenames change, byte limits change, command names change, CVEs get patched, vendors rename their rule modes between releases. So read this chapter differently from the others. Read the prose for the *stable principle* — the thing that is true across every tool. Read the tables as a *dated snapshot* — accurate as of 2026, and to be rechecked against current vendor documentation before you rely on any specific number.

> **How to read this chapter.** Every byte cap, filename, command, and CVE below is marked [verify] and dated. They are practitioner- and vendor-sourced current-state facts that will drift. The bold sentences between the tables are the part that does not drift. If you are reading a printed copy more than a year old, trust the bold sentences and re-derive the tables from the live docs.

## The Stable Principle, Stated Once

Strip away the tool names and every CLI coding agent implements the same three abstractions you have spent twelve chapters learning to engineer:

1. **A persistent, auto-loaded instruction file** that plays the role a system prompt plays in chat — your project's durable WHAT/WHY/HOW, kept short because the capacity budget is real regardless of vendor.
2. **A mechanism for managing the context window over a long session** — clearing it, compacting it, or restarting — because token budgets degrade everywhere, not just in one product.
3. **A permission and tool model** that determines the injection surface and blast radius — which the previous chapter taught you to reason about independent of which tool is enforcing it.

Tools differ only in the *spelling*. One calls the instruction file `CLAUDE.md`, another `AGENTS.md`, another a directory of `.mdc` rule files. One caps the file at a byte count, another at a character count, another not at all. One clears context with `/clear`, another with a new session. **Your job when adopting a tool is translation, not relearning: find that tool's spelling of these three abstractions, learn its specific caps and gotchas, and apply the discipline you already have.** [High] The rest of this chapter is a translation table, organized tool by tool, followed by the cross-tool standard that is slowly making the translation unnecessary.

![A stratified diagram with a volatile dated 2026 top layer of per-tool spellings — CLAUDE.md, AGENTS.md, .cursor/rules, CONVENTIONS.md with their byte caps and a CVE — sitting above a stable foundation of the three durable abstractions: persistent instructions, context management, and permission model](images/13-tool-specific-practice-fig-01.png)

*Figure 13.1 — The fast-aging tool-specific layer is quarantined on top; the three abstractions underneath are the stable core that adopting any tool merely translates down to.*

## Claude Code

Claude Code is the book's running example, so you have already seen most of its spelling. Collected and dated here:

| Abstraction | Claude Code spelling (as of 2026) | Notes / [verify] |
|---|---|---|
| Persistent instructions | `CLAUDE.md` | Auto-loaded; supports **scoped/nested** files — a `CLAUDE.md` in a subdirectory layers on top of the project root file, so monorepo subprojects get local conventions. [Medium; current-state] |
| Instruction delivery | Injected as a `<system-reminder>` in the user-message stream | Carries an "ignore if irrelevant" caveat — so only *universally applicable* content belongs in it (Chapter 3). [Medium; current-state] |
| Context management | `/compact` (summarize-and-continue), `/clear` (fresh context) | Restart at ~80% capacity rather than degrade [verify — the 80% figure is practitioner-sourced]. |
| Reusable workflows | Slash commands in `.claude/commands/*.md`; lifecycle **hooks** (e.g., `PostCompact` to re-inject critical rules) | Hook names and lifecycle events are current-state. [verify] |
| Parallel contexts | Git **worktrees**, one branch per agent (Chapter 11) | Git feature, not Claude-specific; stable. [High] |
| Security controls | Managed settings, MCP scoping, `disableSkillShellExecution` | Exact setting names current-state. [verify] |

The translation: in Claude Code, the persistent-instruction abstraction is `CLAUDE.md` delivered via `<system-reminder>` with an "ignore if irrelevant" semantics, and context management is the `/compact`–`/clear` pair backed by hooks. Hold that shape; the next tools rhyme with it.

## Codex CLI

OpenAI's Codex CLI implements the same three abstractions with different spellings, and — importantly — it adopts the cross-tool `AGENTS.md` convention rather than a vendor-specific filename.

| Abstraction | Codex CLI spelling (as of 2026) | Notes / [verify] |
|---|---|---|
| Persistent instructions | `AGENTS.md` | The cross-tool standard file (see below); read by Codex and an increasing number of other agents. [Medium; current-state] |
| Instruction-file size cap | Documentation/instruction content capped around **32 KiB** | [verify — the 32 KiB figure is practitioner-reported; recheck against current Codex docs. Past the cap, excess content may be truncated or ignored, which is exactly the capacity-budget failure of Chapter 4 with a hard byte boundary.] |
| Context management | Session-level clear/restart | Command spellings current-state. [verify] |
| Known vulnerability | **CVE-2025-61260** associated with Codex CLI in the disclosure record | [verify — CVE identifier and its scope are practitioner/press-reported; confirm number, affected versions, and patch status against the official advisory before citing.] |

The translation: Codex's persistent-instruction abstraction is `AGENTS.md`, and its distinctive gotcha is a **hard byte cap** — which makes Chapter 4's capacity budget not just good practice but a literal limit the tool enforces. If your `AGENTS.md` exceeds the cap, the overflow does not gracefully degrade; it can be silently dropped. The discipline ("keep it short") and the mechanism (a byte ceiling) point the same direction.

## Cursor and Windsurf

Cursor and the related Windsurf editor are IDE-embedded agents, but they share enough of the discipline — and the AGENTS.md trajectory — to belong here. Their distinctive contribution is a **structured, multi-mode rules system** rather than a single flat file.

| Abstraction | Cursor spelling (as of 2026) | Notes / [verify] |
|---|---|---|
| Persistent instructions | Rule files under `.cursor/rules/*.mdc` (Markdown-with-frontmatter) | A *directory* of rules, not one file — naturally encourages the progressive-disclosure pattern of Chapter 5. [Medium; current-state] |
| Rule size guidance | Per-rule character caps in the **~6k / 12k** range depending on rule type | [verify — these character figures are practitioner-reported; recheck. The point survives the exact number: rules are bounded, so word-count discipline matters.] |
| Rule application modes | Three modes: **always-applied**, **auto-attached** (by file/glob match), and **agent-requested/manual** | Mode names are current-state and have changed across releases. [verify] The conceptual split — always-on vs. context-triggered vs. on-demand — *is* progressive disclosure formalized in the tool. [Medium] |
| Cross-tool file | Increasingly reads `AGENTS.md` alongside `.cursor/rules` | Convergence on the standard. [verify] |

The translation: Cursor's three rule modes are the book's static/dynamic and progressive-disclosure distinctions wearing tool clothing. **Always-applied** rules are your minimal universal `CLAUDE.md` core; **auto-attached** rules are domain docs that load only when you touch matching files; **agent-requested** rules are the load-on-demand `agent_docs` index. If you understood Chapter 5, you already understand the three modes — you only need the current names and caps.

## Aider

Aider's distinctive contribution is the answer to Chapter 6's large-codebase problem, built into the tool: the **repository map**.

| Abstraction | Aider spelling (as of 2026) | Notes / [verify] |
|---|---|---|
| Persistent conventions | `CONVENTIONS.md` | Project coding conventions loaded into context; the persistent-instruction abstraction. [Medium; current-state] |
| Large-codebase context | The **repo map** — an auto-generated, ranked summary of the repository's structure (files, key symbols) | The canonical large-codebase answer (Chapter 6): the agent gets a *map* instead of the *territory*, avoiding the overexploration trap. [Medium; current-state] |
| Context management | Add/drop files from the chat context explicitly | Manual scoping rather than autonomous exploration. [Medium; current-state] |

The translation: Aider operationalizes "give the agent a map, not the whole repository." The repo map is precisely the antidote to the overexploration trap from Chapter 6 — the agent reasons over a compact structural summary and pulls full files into context only on demand. Where Claude Code leaves repository scoping largely to the agent's judgment, Aider bakes a ranked map into the default loop. Same principle (scope context in large codebases), different default.

## Continue

Continue is worth one short note precisely because of a *gotcha* that catches transferring practitioners: its rules do not govern every feature uniformly.

| Abstraction | Continue spelling (as of 2026) | Notes / [verify] |
|---|---|---|
| Persistent instructions | Rules/config files | Spelling current-state. [verify] |
| Coverage gotcha | Rules apply to chat/agent flows but **do not necessarily cover autocomplete and "apply" features** | [verify; current-state] The lesson is general: a tool's instruction file may not reach every surface of the tool. |

The translation, and the transferable lesson: **do not assume your instruction file governs every feature of a tool.** Continue's rules shape its conversational agent but may leave autocomplete and inline-apply on their own defaults. When adopting any tool, ask explicitly which surfaces your persistent instructions actually reach — the answer is rarely "all of them."

## The AGENTS.md Cross-Tool Standard

The most important thing in this chapter is the trend that is slowly making the rest of it obsolete in the best way. `AGENTS.md` is an emerging **cross-tool convention**: a single, vendor-neutral persistent-instruction file that multiple agents agree to read. Codex CLI uses it; Cursor and others increasingly read it alongside their native formats; the direction of travel is convergence. [Medium; current-state-as-of-2026 — verify adoption breadth before print.]

Why this matters to the discipline: a cross-tool standard means your colleague from the opening scene loses less of that wasted hour every year. If a repository ships a well-formed `AGENTS.md`, an agent that honors the standard inherits the project's conventions regardless of vendor — the *spelling* converges, so the *translation* shrinks toward identity. The book's bet is that the three abstractions are permanent and the standard around the first one is consolidating. Author your persistent instructions in `AGENTS.md` where the tool supports it, keep a thin tool-specific file (`CLAUDE.md`, `.cursor/rules`) only for what the standard does not yet cover, and you minimize the part of your work that ages.

A practical caution that connects to Chapter 12: a cross-tool instruction file is also a cross-tool *injection vector*. A malicious `AGENTS.md` shipped in a cloned repository can poison *any* standard-honoring agent you open in that directory. Convenience and attack surface scale together; inspect a third-party repository's `AGENTS.md` with your own eyes before any agent reads it.

## A Note on What Did Not Transfer

The clearest way to feel the difference between the durable principle and the perishable spelling is to port a real workflow between two tools and write down exactly what broke. A practitioner moving a logging-migration workflow from Claude Code to Aider, for instance, finds that the *shape* survives intact — short conventions file, scoped task, plan before edits, verify with tests — while a specific cluster of mechanics does not: the `/compact` habit has no equivalent (Aider's context is managed by explicitly adding and dropping files, so "compaction" becomes "drop the files I no longer need"); the `PostCompact` hook has nothing to attach to; and the worktree-per-worker orchestration from Chapter 11, while still possible because worktrees are a Git feature, is not wired into the tool's defaults the way it can be around Claude Code. None of this is a failure of the discipline. It is the discipline meeting a different *default loop* — Aider leans on its repo map and manual file scoping where Claude Code leans on agent-driven exploration and compaction. The transferable lesson is to expect the *principle* to port and the *mechanism* to require re-spelling, and to write down which is which so the next port is faster. [Medium; illustrative]

A related boundary worth naming: several tools here (Cursor, Windsurf, Continue) are IDE-embedded rather than pure command-line agents. This book's scope is the CLI discipline, and these tools earn their place only because they share the three abstractions and are converging on the `AGENTS.md` standard. Their IDE-specific surfaces — inline completion, in-editor apply, UI affordances — are outside the book's scope and outside the reach of the discipline you have built; treat them as a separate concern governed by separate (and often weaker) controls, which is exactly the coverage-gap caution the Continue row raised.

## Putting the Translation to Work

Here is the workflow that turns this chapter into a five-minute exercise instead of a wasted hour, for any tool you adopt:

1. **Find the instruction file.** What is this tool's spelling of the persistent, auto-loaded instruction file? Where does it live, and is it scoped/nested or flat?
2. **Find the cap.** What size limit does it enforce — bytes, characters, or none — and what happens at the boundary (truncation, ignore, error)? This sets your capacity budget concretely.
3. **Find the context-management commands.** How do you clear, compact, or restart? At what fullness should you restart?
4. **Find the permission model.** What can the agent do unattended, what requires approval, and what tools/MCP servers are connected? This is your injection surface from Chapter 12.
5. **Find the coverage gaps.** Which features does your instruction file *not* govern (autocomplete, apply, sub-features)?
6. **Check whether it honors `AGENTS.md`.** If so, author there and keep the tool-specific file thin.

Six questions, answered from current vendor docs, and you have translated the entire discipline onto a new tool. The bridge to the capstone is exactly this: in the final chapter you will assemble the durable layers — instruction file, capacity budget, task prompt, security posture — into shippable templates. Those templates are written in the *stable* vocabulary of this book. This chapter is the lookup table that renders them into whatever tool is in front of you, this year.

## Exercises

These exercises emphasize **Apply** — translating the discipline — and **Evaluate**.

1. **(Apply) Translate the same rule.** Take one concrete project convention (for example, "all new HTTP handlers must call `validateRequest()` first"). Express it as it would appear in `CLAUDE.md`, in `AGENTS.md`, and in a `.cursor/rules/*.mdc` file with an appropriate application mode. Note any size or format constraint each imposes.

2. **(Apply) Build the lookup table for a tool you use.** Answer the six "Putting the Translation to Work" questions for your primary tool, citing the current vendor doc and the access date for each answer. Mark every number [verify] with its date — practice the discipline the chapter preaches.

3. **(Evaluate) Audit a number in this chapter.** Pick one dated fact from the tables — the 32 KiB Codex cap, a Cursor character cap, the `<system-reminder>` mechanism, CVE-2025-61260. Go to the current vendor documentation or advisory and report whether it still holds, what changed, and what you would rewrite. This is the maintenance work a reference chapter requires.

4. **(Apply→Evaluate) Port a workflow and document the gap.** Take a workflow you run in one tool (a slash command, a compaction habit, a worktree setup) and port it to a second tool. Write down precisely what did *not* transfer and why — the principle that held versus the spelling that broke. Submit the before/after plus the gap list.

5. **(Evaluate) The AGENTS.md bet.** Argue, in a paragraph, whether you would author a real project's persistent instructions in `AGENTS.md` or in a vendor-specific file today, given current adoption and the injection-vector caution. There is no single right answer; the grade is on how you reason about durability versus coverage.

> ### The AI Wayback Machine — Douglas Engelbart and the Command Line as Coordination Surface
>
> In December 1968, Douglas Engelbart gave the demonstration later called "The Mother of All Demos": a live, networked, interactive computing session with a mouse, hyperlinks, and collaborative editing, decades before any of it was ordinary. Engelbart's lifelong thesis was not that computers would replace human thought but that they would *augment human intellect* — that the value lay in the interactive coupling between a person and a responsive system, and that the interface to that system was not a cosmetic detail but the very thing that determined how much augmentation you got.
>
> Every tool in this chapter is an answer to Engelbart's question — *what is the interface to the augmenting system?* — and the answers differ in exactly the ways he would have predicted mattered. `CLAUDE.md` versus `AGENTS.md` versus `.cursor/rules`; a byte cap versus a character cap; three rule modes versus one flat file: these are not trivia. They are competing designs for the coordination surface between a developer and a coding agent, and a practitioner who treats the command-line agent as a "magic box" rather than a coordination surface to be configured will get a fraction of the augmentation available. Engelbart saw, sixty years ago, that the interface *is* the leverage. The reason this chapter ages fast is that the field is still, very visibly, arguing about how to build the surface he pointed at.
>
> *Wayback prompt:* Connect tool-specific practice to Engelbart's vision of augmenting human intellect through interactive systems. What did he see about the interface as a coordination surface that CLI-agent users rediscover each time they configure a tool's instruction file rather than treating the agent as a magic box?
>
> *(Engelbart's augmentation thesis and the 1968 demo are well documented [High]; the framing of instruction files as his "coordination surface" is the author's pedagogical connection.)*

## Sources

- Anthropic, "Claude Code documentation," current online docs (accessed 2026). — Source for `CLAUDE.md`, scoped/nested files, `<system-reminder>` delivery, `/compact`, `/clear`, hooks, worktree guidance, and managed-settings controls. **All command names, mechanisms, and setting names are current-state-as-of-2026. [verify]**
- OpenAI, "Codex CLI documentation and AGENTS.md guidance," current online docs and repo materials (accessed 2026). — Source for `AGENTS.md` adoption and the ~32 KiB instruction-content cap. **Byte cap is practitioner-reported and dates fast. [verify]**
- CVE-2025-61260 (associated with Codex CLI in the disclosure record). — **Identifier, scope, affected versions, and patch status are practitioner/press-reported; confirm against the official advisory before citing. [verify]**
- Cursor and Windsurf documentation, current online docs (accessed 2026). — Source for `.cursor/rules/*.mdc`, the ~6k/12k character caps, and the three rule-application modes. **Caps and mode names are current-state and have changed across releases. [verify]**
- Aider documentation, current online docs (accessed 2026). — Source for `CONVENTIONS.md` and the repository-map feature. [Medium; current-state-as-of-2026]
- Continue documentation, current online docs (accessed 2026). — Source for the rules-coverage gotcha (autocomplete/apply not necessarily governed). [verify; current-state]
- AGENTS.md cross-tool convention. — Emerging vendor-neutral standard; adoption breadth is current-state. [verify; current-state-as-of-2026]
- Engelbart, D. "Augmenting Human Intellect" (1962) and the 1968 demonstration. — Anchor for the AI Wayback Machine box. [High for Engelbart; the coordination-surface framing is editorial.]

> **Note on evidence.** This is the book's fastest-aging chapter by design. Every byte cap, character cap, filename, command, rule-mode name, and CVE identifier is current-state-as-of-2026, practitioner- or vendor-sourced, and tagged [verify]. The stable, citable claim is structural: every CLI agent implements the same three abstractions (persistent instructions, context management, permission model), and adopting a tool is translation, not relearning. Trust the bold sentences; re-derive the tables.
