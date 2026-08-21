# Chapter 1 — Why CLI Agent Prompting Is a Different Discipline

## A scene: hour three of a session that started fine

It is the third hour of a single agent session. The task was reasonable — refactor an authentication module, update its tests, and migrate the two call sites that use it. The first hour was a pleasure to watch. The agent read the module, found the call sites, proposed a clean plan, made the edits, ran the tests, fixed a failure, and moved on. You started trusting it. You stopped reading every diff.

Now, at hour three, something is wrong. The agent has begun re-reading files it already edited, as if seeing them for the first time. It reintroduces a bug it fixed ninety minutes ago. When you remind it that the project uses `pytest`, not `unittest` — a rule you wrote plainly in `CLAUDE.md` and that it followed perfectly at the start — it apologizes and then writes a `unittest` test anyway. The model has not gotten dumber. The same model, in a fresh session, would handle any one of these steps without a stumble. What has changed is not the intelligence in the loop. It is the *context* in the loop.

This is **context rot**, and it is the failure that has no equivalent in a chat window. Understanding why it happens — and why your chat-prompting instincts make it worse — is the whole job of this chapter.

## The loop is the unit of work, not the message

In a chat exchange, the unit of work is the message. You write one, the model writes one back, and the slate is, for practical purposes, clean again on the next turn. The conversation is short, and most of what the model needs is in the message you just sent.

A CLI agent does not work in messages. It works in a **loop**, the structure the research literature on tool-using agents calls *reasoning and acting interleaved* (Yao et al., the ReAct paper). Each turn of the loop has four moves:

1. **Read** — the agent pulls information into its context: it opens a file, runs `grep`, executes a test, lists a directory.
2. **Reason** — it interprets what it just saw and decides what to do next.
3. **Act** — it takes an action that changes the world: edits a file, runs a command, creates a branch.
4. **Observe** — it reads the result of that action, which becomes the input to the next *read* or *reason*.

Then it goes around again. A nontrivial task is not one loop but hundreds. And here is the structural fact that everything else in this book follows from: **every move appends to a single, growing context window, and almost nothing is ever removed.** The file the agent read in turn 4 is still sitting in context at turn 400. The failed command from turn 40, the verbose stack trace, the directory listing it glanced at and never used again — all still there, all still consuming the model's attention budget.

That is the difference. Chat prompting is the art of composing one message. CLI-agent prompting is the engineering of a *stateful process* — controlling what enters the context, in what order, and what is allowed to stay. The wording of any single message matters far less than the architecture of the whole session. `[High]`

![Two panels: on the left, four separate chat turns each starting from a clean slate; on the right, a single context window filling turn after turn, with a fill-bar crossing an ~80% danger line into a red rot zone.](images/01-why-cli-agent-prompting-is-a-different-discipline-fig-01.png)

*Figure 1.1 — Chat discards each turn; the agent accumulates everything into one window that degrades — context rot — as it fills past roughly 80% of capacity.*

## Context is the bottleneck, not intelligence

The thesis of this book is that frontier models are already capable enough, and that CLI-agent failures are overwhelmingly failures of *what the model knows at a given moment*, not of raw reasoning power. The hour-three scene is the proof. The same model that wrote the bug fix correctly at hour one cannot reliably hold the same rule at hour three — not because it forgot how to reason, but because the rule is now buried under tens of thousands of tokens of file reads, command output, and the agent's own prior chatter, and the model's ability to find and weight any single instruction degrades as the surrounding material grows.

This is not folklore. The peer-reviewed work on long contexts — Liu et al., *Lost in the Middle* — established that language models do not use all positions in a long context equally; information in the middle of a long context is retrieved and used less reliably than information at the ends. Long context windows are not neutral containers where more room is strictly better. They are crowded rooms where the thing you need can be drowned out. `[High]`

Practitioners put a number on the practical consequence. Field reports, most prominently HumanLayer's 2025 writing on `CLAUDE.md` and agent context, describe a noticeable falloff in agent quality as the window approaches capacity — often cited around **~80% of context capacity**, past which the agent starts dropping instructions, repeating work, and contradicting itself. `[verify — practitioner-sourced, HumanLayer 2025]` Treat the exact percentage as a rule of thumb rather than a measured constant; the *direction* is well attested even where the specific figure is not independently confirmed. The operational takeaway does not depend on the precise number: a long session degrades, and the fix is to manage the budget, not to find better words.

If you accept that context is the bottleneck, the entire strategy of CLI-agent work changes. You stop asking "how do I phrase this so the model understands?" and start asking "how do I keep the *right* information in context and the *wrong* information out?" Every later chapter — instruction files, capacity budgets, progressive disclosure, task scoping, compaction, multi-agent splits — is an answer to that second question.

## The token budget is a first-class concern

In chat, you almost never think about the token budget. Your messages are short; the model's replies are short; you would have to work to fill a modern context window in a normal conversation.

In an agent loop, the budget is the central resource you are managing, whether you manage it deliberately or not. A single `Read` of a large file can cost thousands of tokens. A verbose test run can cost thousands more. A blind exploration of a directory tree — the agent opening file after file to "understand the codebase" — can burn tens of thousands of tokens before any useful work begins. We will see in Chapter 6 a documented case of an agent accumulating 80,000-plus tokens of largely irrelevant context, with a measurable drop in task completeness and a near-doubling of time. `[verify — practitioner-sourced, Augment Code]`

So you learn to think like a budget owner. Every read has a cost. Reading the whole file when you needed ten lines is not free; it is a withdrawal from the same account that holds your instructions. This reframing — *reading is spending* — is one of the hardest habits to build coming from chat, where reading context felt like a pure good. In an agent, more context is not more knowledge; past a point it is more noise, and noise has a price.

## Persistent instruction files replace the system prompt

In chat, you shape the model's behavior with a system prompt that you set per conversation. With a CLI agent, the per-conversation system prompt is largely *not yours to write* — the vendor ships one, and it is substantial. Reverse-engineering work by practitioners such as Piebald-AI on Claude Code's system prompt shows that the agent arrives at your first message already carrying a large set of standing instructions about how to use its tools, how to format output, when to ask permission, and how to behave safely. `[verify — practitioner-sourced, Piebald-AI]`

Your lever, instead, is the **persistent instruction file**: `CLAUDE.md` for Claude Code, `AGENTS.md` for Codex CLI and the cross-tool standard, `CONVENTIONS.md` for Aider. You write it once, commit it to the repository, and the tool injects it into every session automatically. It is your standing system prompt — but it stacks on top of the vendor's, and it competes for the same attention budget as everything the agent reads. That competition is the subject of Chapters 3 and 4, and it is why "keep the file short" turns out to be a hard engineering constraint rather than a matter of taste.

For now, hold two facts. First, the instruction file is **static context** — written in advance, loaded the same way every session, identical regardless of what the agent later does. Second, because it is loaded every time and shares the budget with everything dynamic, only *universally applicable* content earns a place in it. A rule that applies to one file does not belong in a file the agent loads for every task. Chapter 2 makes the static/dynamic distinction precise; it is the lens you will use to debug everything that follows.

## The agent decides what to read — and that is the new failure surface

Here is a capability that simply does not exist in chat: **the agent chooses what to pull into its own context.** You ask it to fix a bug; it decides which files to open, which commands to run, how deep to explore. That autonomy is what makes agents powerful, and it is also a new and large failure surface.

The agent can read too little and act on an incomplete picture — editing a function without checking its call sites, "fixing" a test by reading only the test and never the code under test. It can read too much and pollute its own context with irrelevant material until the signal is buried — the overexploration trap. And it can read *untrusted* material — a file in a repository you did not write, a comment containing instructions, a tool result from an external service — and treat that text as if it came from you. That last possibility is the seed of the entire security discussion in Chapter 12: in an agent, *every file read is a potential instruction*, because the model does not have a hard boundary between "data I am examining" and "commands I should follow."

Chat prompting has none of this. The model in a chat reads only what you paste. The CLI agent reads what *it* decides to, from a filesystem you may not fully control, and what it reads becomes part of how it behaves. Engineering the context system means engineering *what the agent is allowed and inclined to read* — narrowing it, scoping it, and treating unfamiliar input as hostile until proven otherwise.

## Why your chat instincts backfire

It is worth naming the specific habits that transfer badly, because they are the ones that produce the hour-three scene:

- **"Give it everything; more context can't hurt."** In chat, dumping in the whole document often helped. In an agent, dumping in the whole codebase fills the budget with noise and accelerates context rot. *Less* is frequently more reliable.
- **"If it got it wrong, explain again."** In chat, restating clarifies. In a degrading agent session, restating *adds tokens to an already-polluted context* and often makes things worse. The repair is usually to `/clear` and start fresh from a saved plan, not to argue with a confused agent. (Chapter 7 formalizes the rule: after about two failed corrections, stop correcting and restart.)
- **"A good-looking answer is a good answer."** In chat you read the whole answer. With an agent it is tempting to trust a confident "done." But a successful-looking patch is not evidence of correctness — only the *observe* move, an actual test run, produces that evidence. The agent's fluency is not your verification.
- **"A bigger or newer model will fix this."** It usually will not, because the failure is contextual, not cognitive. A better model raises the ceiling on what is possible in a clean context; it does not exempt you from managing the context.

Each of these is a chat reflex misapplied to a stateful loop. Unlearning them is most of what it takes to become competent with agents.

## Worked example: turning the hour-three failure into a recovery

Return to the refactor that fell apart. Here is the difference between fighting context rot and managing it.

**What the developer did (and should not have):** kept correcting in place.

```
You: No, use pytest, not unittest.
Agent: You're right, sorry. [writes another unittest test]
You: pytest. Please. We use pytest everywhere.
Agent: Understood. [writes unittest again]
```

Two corrections, no improvement, growing context. The instruction is in `CLAUDE.md` and it is *still* being ignored — a clear signal that the static instruction has been drowned by dynamic noise.

**What works:** stop, capture state, restart clean.

```
You: Before we go further, write a short plan file: the remaining steps,
     the files involved, and the project conventions that matter here
     (test runner, where tests live). Then stop.
Agent: [writes plan.md, stops]

You: /clear
```

Then, in a *fresh* session — empty context, `CLAUDE.md` reloaded clean, none of the three hours of accumulated noise — you hand it only the plan:

```
You: Read plan.md and CLAUDE.md, then execute the remaining steps.
     Use pytest. Run the suite after each change.
```

The model is the same. The task is the same. The only thing that changed is the *context system*: you evicted the polluted history and reloaded a clean, scoped state. The agent that could not hold "use pytest" at hour three holds it effortlessly at minute one of the new session — because now it is near the top of a near-empty window instead of buried in the middle of a full one. This is the plan-then-execute split that Chapter 7 builds into a formal workflow, and it is, in Anthropic's own framing, among the highest-leverage habits in agentic coding. `[verify — Anthropic Engineering, current-state]`

## Exercises (Bloom: Understand)

The goal here is to *explain* the discipline, not yet to engineer it.

1. **Name the loop in a real session.** Run any nontrivial task in your CLI agent and capture the transcript. Mark each action as read, reason, act, or observe. Count how many *read* moves occurred before the first *act*. Write one paragraph: did the agent read enough, too little, or too much before acting?

2. **Diagnose context rot.** Deliberately run a long session (a multi-step refactor). Note the first moment the agent does something it would not have done fresh — repeats work, ignores a stated rule, re-reads an edited file. Estimate roughly how full the context was. Explain, in terms of *Lost in the Middle* and the ~80% rule of thumb, why this happened — and flag the figure you used as practitioner-sourced.

3. **Chat versus agent, side by side.** Take one task. Solve it in a chat LLM by pasting the relevant files; then solve it with a CLI agent that reads the files itself. Write half a page on what the *human* had to supply in each case that the model could not infer — the difference is the work this book teaches.

4. **Falsify "bigger model fixes it."** Find a failure in your agent. Without changing the prompt or the context strategy, switch to a larger or newer model if you have access. Did the failure go away? Whether it did or not, explain in terms of the thesis why this is *not* the lever the book recommends.

5. **Self-explain the system prompt.** Read a practitioner reverse-engineering of an agent's system prompt (e.g., Piebald-AI on Claude Code). In your own words, explain why your `CLAUDE.md` is *not* the agent's whole system prompt, and what that implies about how much room you actually have. Tag your own confidence in any number you cite.

## AI Wayback Machine: Lucy Suchman and the gap between the plan and the action

> In 1987, the anthropologist Lucy Suchman published *Plans and Situated Actions*, built on close study of people struggling with a "smart" photocopier whose designers had encoded a plan for how copying *should* go. Her central finding unsettled a generation of AI researchers: a plan is not a script that determines action. It is a *resource* for action, and what actually happens emerges moment to moment from the interaction between the actor and a real, messy environment. The copier failed not because its plan was wrong in the abstract, but because the situation kept diverging from the plan, and the machine had no way to notice or adapt.
>
> A CLI agent is Suchman's photocopier with a far better model inside it. Your `CLAUDE.md` and your task prompt are *plans* — resources, not guarantees. The agent's actual behavior emerges turn by turn from the collision between those plans and the live state of the repository and its own filling context window. The hour-three failure is a situated-action failure: the plan was fine, but the situation (a polluted context) drifted away from it, and the standing instruction lost its grip. Suchman's lesson, four decades early, is exactly the one this chapter ends on — you cannot engineer behavior purely by writing a better plan in advance. You have to engineer the *situation* in which the plan is read: what is in context, how full it is, when to reset. That is context engineering, and it is what we make precise next.

## Bridge to Chapter 2

You can now articulate why the agent loop is a different discipline from one-shot chat: it is stateful, its context grows and rots, the token budget is the resource you manage, your instruction file is a standing prompt competing for attention, and the agent's freedom to choose what it reads is both its power and its largest failure surface. But to debug an agent reliably, you need a sharper instrument than "the context got bad." You need to be able to look at any piece of what the agent knows and classify it — and to predict, from that classification alone, *how it will fail*. That instrument is the static-versus-dynamic distinction, and it is the subject of Chapter 2.

## Sources

- Yao et al., "ReAct: Synergizing Reasoning and Acting in Language Models," 2022/2023 — the read/reason/act/observe loop as interleaved reasoning and acting. [peer-reviewed/preprint]
- Shinn et al., "Reflexion: Language Agents with Verbal Reinforcement Learning," 2023 — agent performance depends on preserved memory and feedback. [peer-reviewed/preprint]
- Yang et al., "SWE-agent: Agent-Computer Interfaces Enable Automated Software Engineering," NeurIPS 2024 — interface and context, not just model intelligence, determine performance. [peer-reviewed]
- Liu et al., "Lost in the Middle: How Language Models Use Long Contexts," TACL 2024 — position and length affect retrieval; long context is not a neutral container. [peer-reviewed]
- HumanLayer, "Writing a good CLAUDE.md" / agent-context field notes, 2025 — the ~80%-capacity degradation rule of thumb. [practitioner, `[verify]`]
- Piebald-AI, Claude Code system-prompt reverse-engineering, 2025 — the vendor system prompt the agent already carries. [practitioner, `[verify]`]
- Anthropic Engineering, "Effective context engineering for AI agents," 2025; Claude Code documentation, current — plan-then-execute and `/clear` as endorsed practice. [vendor, current-state, `[verify]`]
- Suchman, L., *Plans and Situated Actions: The Problem of Human-Machine Communication*, Cambridge University Press, 1987. [historical/peer-reviewed]
