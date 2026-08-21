# Chapter 0 — Setup: Installing and Running a CLI Agent

*Optional front-load. If your machine already has a CLI agent installed and you have watched it read and edit a file, skip ahead to Chapter 1. This chapter exists for one purpose: to get a complete beginner to a single, observable* read → reason → act → observe *loop on a real repository, so the rest of the book has something concrete to point at.*

## A scene: the agent that edited before it looked

A developer new to agentic tooling opens a terminal in a small Python project, installs a CLI agent, and types the most natural thing in the world: *"Fix the failing test."* The agent thinks for a moment, then immediately opens a source file and rewrites a function. It announces, confidently, that the bug is fixed. The developer runs the test suite. Three tests fail now instead of one.

What went wrong is not intelligence. The agent never ran the test it was asked to fix. It never read the test file to see what behavior was expected. It pattern-matched "fix" to "edit" and skipped the *read* and the *observe* — two of the four moves that make an agent an agent. The repair is not a cleverer sentence. It is a prompt that *requires* the loop to run in order: inspect, then reason, then change, then verify.

That failure is the whole book in miniature, and you can reproduce it on your own machine in about fifteen minutes. Let us set that up.

## What you need before you start

This book assumes a working developer's baseline, and Chapter 0 assumes only that baseline:

- **Git and a terminal.** You can clone a repo, make a commit, read a diff. `[High]`
- **One language toolchain installed** — Python with `pytest`, or Node with a test runner, or whatever you already use. The examples below use Python because the failure is easy to stage, but nothing here is language-specific.
- **Experience with at least one chat LLM.** You know what a system prompt is and what a tool call is, even if you have never driven an agent.
- **An account and credentials for one CLI agent.** Most agents authenticate against a hosted model API and will refuse to run until they have a key or a logged-in session.

You do **not** need to understand any agent's internals. That is what Chapters 1 and 2 build. Here you are just getting a loop to run.

## Installing an agent (current-state, 2026)

CLI agents are installed and invoked in roughly the same shape across vendors, even though the exact commands differ and *will* drift — treat everything in this section as `[Medium]` / current-state and check the vendor's own docs before relying on a flag. `[verify — vendor docs, dated]`

The common pattern is: install a binary or package, authenticate once, then launch the agent from inside a project directory so it can see the repository.

```bash
# Claude Code (Anthropic) — illustrative current-state install/run
npm install -g @anthropic-ai/claude-code     # or the vendor's documented installer
cd ~/projects/my-small-repo
claude                                        # launches an interactive session in this repo

# Codex CLI (OpenAI) — illustrative
npm install -g @openai/codex
cd ~/projects/my-small-repo
codex                                         # interactive session, reads AGENTS.md

# Aider — illustrative
pip install aider-install && aider-install
cd ~/projects/my-small-repo
aider                                         # repo-map-based session
```

Three things are worth noticing even at this stage, because they are the book's spine showing through the install instructions:

1. **You launch from inside the repository.** The agent's view of the world starts at your current directory. Where you stand when you type the command determines what context it can reach.
2. **The agent looks for an instruction file on startup.** Claude Code reads `CLAUDE.md`; Codex CLI and several others read `AGENTS.md`; Aider reads `CONVENTIONS.md`. If the file exists, its contents are injected into the session before your first message. This is the **persistent instruction file** that replaces the per-conversation system prompt you are used to from chat. We build it properly in Chapter 3; right now you just need to know it is loaded automatically.
3. **The agent asks before it acts — sometimes.** Most agents have a permission posture: they prompt you before running a shell command or writing a file, unless you have granted standing approval. This is your first encounter with the injection surface that Chapter 12 is about. For now, leave approvals on. Watch what it wants to do before you let it.

## Staging a failure you can watch

Create a tiny repository with one obvious bug. The point is not the bug; the point is to make the four-move loop *visible*.

```python
# calc.py
def add(a, b):
    return a - b          # the bug: subtraction, not addition
```

```python
# test_calc.py
from calc import add

def test_add():
    assert add(2, 3) == 5
```

Run the suite once yourself so you know the ground truth: `pytest` reports `test_add` failing, `add(2, 3)` returned `-1`. Commit this state so you can reset between experiments (`git add -A && git commit -m "staged bug"`).

## Two prompts, two outcomes

Now run the agent twice on the same bug, changing only the prompt. This is the chapter's worked example, and the contrast is the lesson.

**Prompt A — the natural, lossy version:**

```
Fix the failing test.
```

Watch what the agent does. It *may* read first, but a naive agent often jumps straight to an edit, because "fix" reads as a command to change code. If it edits `calc.py` without running `pytest` first, you have reproduced the opening scene. Reset with `git checkout .` and try again.

**Prompt B — the version that requires the loop:**

```
There is a failing test in this repo. Before changing any code:
1. Run `pytest` and show me the output.
2. Read test_calc.py and tell me what behavior the test expects.
3. Only then propose the smallest change to make the test pass.
4. After editing, run `pytest` again and show the result.
Do not edit more than one file.
```

Now watch the loop run in order. The agent *reads* (`pytest`, then `test_calc.py`), *reasons* (the test wants `add(2,3) == 5`, so `add` must sum, not subtract), *acts* (changes `-` to `+`), and *observes* (re-runs `pytest`, sees green). Four moves, each visible in the transcript.

![A four-node ring — read, reason, act, observe — with arrows running clockwise and the observe node feeding back into reason; observe is highlighted as the move the naive agent skips.](images/00-setup-installing-and-running-a-cli-agent-fig-01.png)

*Figure 0.1 — One full turn of the read → reason → act → observe loop; the opening-scene failure was skipping read and observe.*

You did not give Prompt B a bigger model or more elegant phrasing. You gave it *structure that forces the loop*. That is the difference between chat prompting and CLI-agent prompting in a single side-by-side, and it is why the rest of this book is about engineering the context and the control flow rather than polishing wording.

## What you should be able to see now

After running both prompts, you can point at each move in the transcript:

- **Read** — the `pytest` output and the file contents that entered the agent's context as *dynamic* information it gathered itself.
- **Reason** — the agent's stated interpretation of what the test wants.
- **Act** — the diff it applied.
- **Observe** — the second `pytest` run, the feedback that closes the loop.

Hold onto that transcript. Chapter 1 names this loop formally and explains why each turn quietly fills the context window; Chapter 2 will have you classify what entered the agent's context as *static* (anything from an instruction file) versus *dynamic* (everything the agent read or ran). You will reuse this exact session.

## Exercises (Bloom: Understand)

These are observation exercises, not engineering exercises. The goal is to *see* the loop, not yet to control it.

1. **Annotate the loop.** Take your Prompt B transcript and label every line as read, reason, act, or observe. Where did the agent skip a move or do them out of order?
2. **Break it on purpose.** Re-run with Prompt A several times. How often does the agent edit before running the test? Write one sentence explaining, in terms of the loop, why "Fix the failing test" is a worse prompt than Prompt B — without using the words "better" or "smarter."
3. **Find the instruction file.** Check whether your agent created or looked for `CLAUDE.md` / `AGENTS.md` / `CONVENTIONS.md` on startup. If none exists, add a one-line file: `Always run the test suite before editing code.` Re-run Prompt A. Did the standing instruction change the behavior? This is your first taste of static context.
4. **Watch the permission prompts.** With approvals on, list every action the agent asked permission for. Which of those touched a file or ran a command that could, in principle, have done something you did not intend? (You are previewing Chapter 12.)
5. **Self-explain (no tool needed).** In three sentences, explain to a teammate why a successful-looking patch is *not* evidence the bug is fixed. What is the only move in the loop that produces such evidence?

## AI Wayback Machine: Douglas Engelbart and the command line as a coordination surface

> In December 1968, Douglas Engelbart gave the demonstration later nicknamed "The Mother of All Demos," showing a working system of interactive editing, hyperlinks, and a pointing device — and, underneath the spectacle, a thesis: computers should *augment human intellect*, not replace it. Engelbart did not see the terminal as a magic box that answers questions. He saw it as a *coordination surface* between a human's intent and a machine's capability, where the human supplies goals and judgment and the machine supplies tireless execution.
>
> That is exactly the posture this chapter asks you to take toward a CLI agent. The agent is not an oracle you query; it is a partner in a loop, and the quality of the work depends on how well you structure the coordination — what you ask it to inspect, what evidence you demand, where you keep your hands on the approvals. Engelbart's bet was that the leverage lives in the interface between human and machine, not in the raw power of either alone. Fifty-odd years later, every developer who writes a careful task prompt instead of "fix it" is rediscovering precisely that.

## Bridge to Chapter 1

You now have a loop you can watch and a transcript you can annotate. What you do not yet have is a *model* of why that loop is so different from a chat exchange — why each turn matters, why the context window fills, and why a long session can rot in a way no chat conversation ever does. That is the subject of Chapter 1: why CLI agent prompting is a different discipline, and why "context is the bottleneck, not intelligence" is the sentence that organizes everything else.

## Sources

- Yao et al., "ReAct: Synergizing Reasoning and Acting in Language Models," 2022/2023 — the read/reason/act/observe vocabulary. [peer-reviewed/preprint]
- Yang et al., "SWE-agent: Agent-Computer Interfaces Enable Automated Software Engineering," NeurIPS 2024 — interface design changes agent performance. [peer-reviewed]
- Anthropic, "Claude Code documentation," current online docs — install/run, `CLAUDE.md` auto-loading, permissions. [vendor, current-state, `[verify]` against docs at time of use]
- OpenAI, "Codex CLI / AGENTS.md documentation," current online docs — install/run, `AGENTS.md` loading. [vendor, current-state, `[verify]`]
- Aider documentation — install/run, `CONVENTIONS.md`, repo map. [vendor/OSS, current-state, `[verify]`]
- Engelbart, D., "Augmenting Human Intellect: A Conceptual Framework," 1962; and the 1968 Fall Joint Computer Conference demonstration. [historical]
