# Chapter 8 — Reasoning and Range Patterns
*When to tighten the path and when to widen the sample — and why the answer depends on the model, not the task.*

---

Here is a documented sequence, not a hypothetical. Take a single grade-school math problem from the GSM8K benchmark: so many crates, so many apples per crate, so many spoiled, how many sellable. A capable but not reasoning-tuned model, asked for just the answer, decodes greedily to a number and gets it wrong. Add five words to the prompt — *"Let's think step by step"* — and the same model, same weights, same question, now writes out the intermediate arithmetic and lands on the right number. Kojima et al. (2022) measured exactly this jump on text-davinci-002: MultiArith went from 17.7% to 78.7%, GSM8K from 10.4% to 40.7%, with no examples at all, just the trigger phrase.

![Adding "let's think step by step" lifts accuracy: a five-word zero-shot trigger raised MultiArith from 17.7% to 78.7% and GSM8K from 10.4% to 40.7% on the same model — the phrase changes the region of the distribution decoded, not the model's knowledge.](../images/08-reasoning-and-range-patterns-fig-03.png)
*Figure 8.3 — Adding "let's think step by step" lifts accuracy*

The obvious reading is that the phrase made the model reason. That reading is wrong, and the experiment that breaks it is the spine of this chapter.

Wang et al. (2023), in an ACL paper whose whole job was to ask what actually matters in chain-of-thought, fed models exemplars whose reasoning steps were invalid — the logic was broken, the intermediate steps did not follow — and still recovered roughly 80 to 90 percent of the performance of valid reasoning exemplars. What mattered was that the steps were relevant to the query and in the right order, not that they were true. The model was not following the logic. It was being conditioned onto a trajectory that had the shape of step-by-step work, and step-shaped trajectories land on correct answers more often than a single confident leap to a number.

That is the mechanism. Chain-of-thought does not install reasoning. It changes the region of the output distribution the model decodes through. And once you see it as an operation on the distribution rather than a cognitive upgrade, three things follow that the rest of the chapter unpacks: the reasoning can be recovered by decoding alone with no prompt change at all; the benefit is emergent with scale and was negligible or harmful on small models; and on models trained to produce that trajectory internally, telling them to do it again from the prompt is at best redundant and at worst destabilizing.

---

## Two opposite moves on one machine

From Chapter 1: an output is a sample from a conditional distribution. Given your prompt and the tokens so far, the model assigns a probability to every possible next token, and a decoding procedure picks one. The full answer is a path through this distribution, token by token. Greedy decoding walks the single most-probable path — the mode of the distribution.

Every pattern in this chapter is a move on that picture, and there are exactly two directions.

**Tighten the path.** A reasoning pattern reshapes the prompt so that the high-probability path itself becomes a better path. Chain-of-thought and few-shot exemplars do not add randomness; they add conditioning. They make step-shaped, correctly-formatted trajectories more probable, so that when the model walks the mode, the mode now passes through intermediate work instead of leaping to a guess. You are still taking essentially one path — you have just bent it.

**Widen the sample.** A range pattern stops accepting the single path and pulls more of the distribution into view. Alternative Approaches asks for several distinct paths in one pass. Self-consistency draws many independent paths and votes. Game Play and Tail Generation extend a single path into sustained state across turns. The common move: do not take the mode and stop; look at breadth.

These are opposite operations, and the confusion that wrecks systems is treating them as interchangeable "make it better" knobs. Tightening the path is the right move when there is a single correct answer and the model's most-probable trajectory is missing it for want of intermediate work — arithmetic, multi-step deduction, structured extraction. Widening the sample is the right move when the mode is not the best answer — contestable plans, design decisions, anything where "most probable" and "best" come apart. Reach for a range pattern on a problem with one right answer and you manufacture confident wrong alternatives.

One misconception to dislodge before going further: "raising temperature is how you get range." Temperature widens the distribution the sampler reaches into per token, which makes each single output internally less coherent — a low-probability early token destabilizes everything conditioned on it. A range pattern like Alternative Approaches widens the set of answers while leaving decoding parameters alone; each alternative is generated as its own coherent span. You get the diversity of a high-temperature ensemble with the per-answer coherence of a low-temperature decode. That is the whole reason range patterns are engineering primitives and not just "turn the knob up."

---

## Reasoning patterns: tightening the path

### Few-shot: it teaches format more than reasoning

Few-shot prompting supplies a handful of worked examples before the real query. The folk reading is that examples teach the task. The sharper empirical reading is that examples mostly teach the shape of the answer.

The cleanest evidence is Min et al. (2022, "Rethinking the Role of Demonstrations"), who found that in many few-shot settings the correctness of the labels in the exemplars matters surprisingly little — randomizing the labels barely dented performance — while the label space, the input distribution, and the format did the work. The exemplars tell the model "answers look like this: this length, this vocabulary, this structure, drawn from this set of options." They condition format and framing far more than they impart new reasoning capacity.

This is why few-shot belongs on the tighten-the-path side even though it is not chain-of-thought. It narrows the output distribution toward a region of correctly-shaped answers. When your problem is "the model can do this but keeps formatting the output wrong, or labeling out of the allowed set, or rambling," few-shot is the precise tool. When your problem is "the model genuinely cannot do the multi-step inference," few-shot exemplars with reasoning traces in them may help — but the help is coming from the reasoning trajectory, not the example-ness.

One cost worth naming: every exemplar is tokens in context that compete for attention and cost money and latency, and the order of those exemplars is itself a brittleness surface that can swing accuracy dramatically (Lu et al. 2022 — Chapter 9 develops this). Use the fewest exemplars that fix the format problem you actually have.

| | Few-shot | Zero-shot CoT | Few-shot CoT |
|---|---|---|---|
| What it changes | Format / answer shape | Trajectory shape | Both |
| Where it helps | Format problems | Multi-step reasoning | Both |
| What it costs | Context tokens (and order brittleness) | Token count | Both |
| Reach for it when | Output is mis-shaped, not wrong | The model can't surface the steps | Both problems at once |

### Chain-of-thought: emergent, conditional, and not about valid logic

Chain-of-thought elicits an intermediate reasoning trace before the answer — either by few-shot exemplars that include the steps, or by a zero-shot trigger like "Let's think step by step." Three findings fix its scope precisely, and the discipline of this book is to carry all three rather than the slogan.

**It is emergent with scale.** Wei et al. (2022) reported that chain-of-thought's benefit appears in large models and is negligible or negative in small ones. Prompting a 540B PaLM with eight CoT exemplars reached state-of-the-art on GSM8K; small models gained little or got worse. Chain-of-thought is not a universal good. It is a behavior that switches on past a capability threshold.

**Its benefit is trajectory shape, not valid inference.** Wang et al. (2023): invalid reasoning steps recover roughly 80 to 90 percent of valid-CoT performance; relevance and ordering dominate correctness. The model is conditioned onto a step-shaped manifold, not executing logic. The practical consequence is that a CoT trace is not a faithful audit log of the model's reasoning — do not treat the visible steps as a guarantee the answer follows from them.

**The reasoning is in the distribution, not the prompt string.** Wang and Zhou (2024, "Chain-of-Thought Reasoning Without Prompting") showed that inspecting the top-k alternative first tokens rather than greedy-decoding surfaces latent reasoning paths already present in the pretrained model — yielding roughly ten to thirty absolute GSM8K points across the PaLM-2 family with no prompt change at all. The step-by-step capability was latent in the output distribution; the prompt phrase was just one way to reach it, and a decoding choice is another.

![The reasoning was in the distribution, not the words: the same model reaches a step-by-step trajectory three ways — direct greedy answer (wrong), a "step by step" prompt that conditions the path, and top-k decoding that surfaces the latent path with no prompt change.](../images/08-reasoning-and-range-patterns-fig-02.png)
*Figure 8.2 — The reasoning was in the distribution, not the words*

Three routes to the same trajectory on the same model: direct answer, greedy — wrong; zero-shot "let's think step by step" — right, because the prompt conditions a step-shaped path; CoT-decoding, same direct prompt but inspecting top-k first tokens — right, because the step-shaped path was reachable in the distribution without any prompt cue. The reasoning was never in the words "step by step."

### When reasoning-tuned models make explicit CoT redundant — or harmful

This is the contested frontier, so the book separates the stable mechanism from current model behavior.

The stable mechanism: a reasoning-tuned model has been trained to produce the step-shaped trajectory internally before emitting an answer. If the trajectory is already being generated, instructing the model from the prompt to also generate it is, at best, telling it to do what it was going to do anyway.

The current behavior: Meincke et al. (2025, "The Decreasing Value of Chain of Thought in Prompting," Wharton Generative AI Labs) report that for non-reasoning models explicit CoT still gives small average gains, but for reasoning-tuned models the gains are marginal and CoT adds answer-to-answer variance, sometimes flipping otherwise-correct answers. Adding the instruction does not just fail to help — it injects instability. Layered on top of Wang et al.'s finding that the trace is shape not logic, this is coherent: you are perturbing an already-good internal trajectory with a redundant external cue, and the perturbation occasionally knocks the model off a correct path.

The decision is not "use CoT." It is "does this model, on this task, still need me to elicit a trace?" The honest summary by tier:

| Model tier | Reasoning elicitation |
|---|---|
| Older / base, large enough | Few-shot CoT or zero-shot "step by step" — real gains |
| Mid-tier instruction-tuned, non-reasoning | Zero-shot CoT helps modestly |
| Reasoning-tuned (internal CoT) | Add nothing — explicit CoT is redundant and adds variance |
| Any tier, you control decoding | CoT-decoding (top-k inspection) is an alternative to prompt-side CoT |

The reasoning-tuned-model finding is recent and the model landscape moves fast. Re-verify against the current model tier rather than carrying the 2025 result as a law.

![Whether to elicit reasoning depends on the tier: older large models gain real accuracy from explicit CoT, mid-tier instruction-tuned models gain modestly, reasoning-tuned models gain nothing and lose stability, and decoding-control opens a separate lever.](../images/08-reasoning-and-range-patterns-fig-05.png)
*Figure 8.5 — Whether to elicit reasoning depends on the model tier*

---

## Range patterns: widening the sample

The structuring patterns of the surrounding chapters narrow the output. The patterns here deliberately widen or extend it. The unifying engineering judgment is range calibration: too little range and you anchor on a weak first answer; too much and you burn tokens and tempt the model to confabulate options that do not exist. Asking for more range is asking the model to generate into regions where it has less to say, and where what it says is more likely to be invented.

### Self-consistency: range as a sampling procedure

Self-consistency (Wang et al. 2022, "Self-Consistency Improves Chain of Thought Reasoning") is the clearest place to see that a range pattern is the Chapter 1 sampling machine. The procedure: sample many CoT trajectories at non-zero temperature, then take the majority-vote answer across them. It is literally draw N samples from the distribution and aggregate — a Monte-Carlo estimate of the answer the model most consistently arrives at, rather than the answer of any single decode.

![Draw many trajectories, take the majority vote: self-consistency samples many chain-of-thought paths at non-zero temperature and returns the plurality answer, reducing variance by averaging rather than by asking the model to judge itself.](../images/08-reasoning-and-range-patterns-fig-04.png)
*Figure 8.4 — Draw many trajectories, take the majority vote*

This is why self-consistency belongs on the widen side and pairs naturally with the tighten side: you tighten each path with CoT, then widen across paths by sampling many and voting. It works because a single sampled trajectory can land on a wrong answer by chance, and the plurality of trajectories is a better estimator than any one of them — variance reduction by averaging, exactly the Chapter 1 math. It is expensive because you pay N forward passes. And on a reasoning-tuned model whose internal trajectory is already strong, the marginal value of external voting shrinks, just as the marginal value of external CoT does.

Self-consistency is not a separate clever trick. It is the sampling-and-aggregation you already understand from Chapter 1, applied to reasoning outputs. If you understand why averaging many noisy estimates beats one, you understand self-consistency.

### Alternative Approaches: breadth without losing coherence

*"Propose three structurally distinct approaches to [task], differing in [axis]. For each: the core idea in one sentence, the dimension it wins on, the point at which it fails first, and a one-line cost. Then recommend one and state the condition under which your recommendation flips."*

Alternative Approaches instructs the model to enumerate multiple distinct methods and compare them rather than committing to one. It samples the distribution's breadth instead of its mode. As §8.2 argued, this is mechanistically different from raising temperature: the breadth comes from task framing, decoding stays put, each option is its own coherent span, and — a subtler benefit — enumerating in one pass lets each later option condition on the earlier ones, so "Approach 3" can be steered to differ from 1 and 2. Independent high-temperature samples have no such mechanism and frequently cluster.

Three construction rules, each preventing a failure. First, demand distinctness on a named axis — otherwise you get three near-duplicates, which is pseudo-diversity and wastes all the tokens. Second, demand comparable trade-off axes — otherwise three options arrive in three incommensurable vocabularies with no basis for choosing. Third, ask where each one breaks first — which drags implicit risk into the open and makes a non-viable padded option expensive to fake. An option that has to name its own failure mode cannot be confabulated cheaply. That requirement is the instruction that keeps widening from becoming fabrication.

Ten alternatives is not ten times as useful as three. It is a longer document, and the tenth option is usually invented to fill space.

### Game Play: widening along the time axis

Game Play installs a bounded interactive state machine — a premise, a rule-set, objectives, a turn structure — that the model must maintain across turns. The model has no literal state register. The game state lives in the context window, and on each turn the model re-derives it by attending over the rules and the running transcript. This is the same attention dynamic that drives persona drift in Chapter 6: the rule-set defined early competes for attention against an ever-growing transcript.

The signature failure is rule drift and self-cheating — the model breaks its own rules while appearing to honor them, often in the player's favor because the RLHF approval disposition of Chapter 4 pushes it to let the player win. The mitigations follow directly from the mechanism. Externalize the mutable state by restating it at the top of every turn. Keep the rule-set small. Separate immutable rules (system prompt) from mutable state (the turn tail). For high-stakes adjudication, enforce the rules in code and let the model narrate, not referee. The model is a good storyteller and an unreliable judge.

### Tail Generation: seeding the next path

Tail Generation ends each output with a recap of state and an explicit next-step menu. The appended tail is high-salience recent context that the next turn conditions on — solving the cold-restart problem, where the model picks up the loudest dangling thread rather than the most important one. Construct it by summarizing decision-relevant state, offering a small menu of steps the model can actually execute, and making the tail conditional on session length.

The dangerous failure is fabricated next steps: inventing options that assert facts about work never done. "Next, deploy the validated config" asserts a config was validated. If it was not, the next turn builds on the falsehood. Because the tail sits in the highest-salience position in context, fabrications here propagate with unusual force. The mitigation is to make each step executable — a step that has to be genuinely runnable cannot be easily invented.

One boundary to keep sharp: Tail Generation is continuation seeding, not turn-taking control. Patterns where the model drives by asking you questions, or solicits a specific input before proceeding, are interaction patterns about who holds the conversational initiative. They live in Chapter 11. A next-step menu can contain such a prompt, but the patterns answer different questions: "what should be most salient when the next turn begins?" versus "who should ask the questions?"

![Two opposite moves on one sampling machine: a single axis from tightening the path (greedy → few-shot → CoT) toward the mode, to widening the sample (Alternative Approaches, self-consistency, Game Play / Tail Generation) away from it — all operations on the same output distribution.](../images/08-reasoning-and-range-patterns-fig-01.png)
*Figure 8.1 — Two opposite moves on one sampling machine*

---

## The decision: tighten, widen, both, or neither

The two moves are not a menu you pick from by taste. There is a procedure, and it starts with the model tier because the reasoning-tuned-model finding made tier the first variable.

What model tier? Reasoning-tuned model — do not add explicit CoT. It is redundant and adds variance. Let the internal trajectory run. Older or non-reasoning but capable model — explicit CoT is on the table.

Is the answer unique or contestable? Unique correct answer and the model's mode is missing it for want of intermediate work — tighten the path with CoT, subject to the tier check, and widen across paths with self-consistency if reliability matters and you can afford N passes. Contestable answer where "probable" and "best" diverge — widen the sample with Alternative Approaches. Do not tighten onto one trajectory you will then over-trust.

Is the failure actually a format problem? If the model can do the task but mis-shapes the output, the tool is few-shot, not CoT and not range.

Does the task need state across turns or continuity forward? Sustained rule-bounded state — Game Play, small rule-set, externalized state, code-side adjudication for stakes. Long session where cold restart is costly — Tail Generation, decision-relevant recap, small real menu.

In every widening case: ask what makes the range expensive to fake. "Where does it break first" for options. Restated state for games. Executable steps for tails. Majority vote for self-consistency. The instruction that forces the model to ground its range is the one that keeps widening from becoming confabulation.

Two places where this is not the problem and you need to fix a different layer. If your output swings wildly when you change a colon to a dash or reorder identical exemplars, that is brittleness — a Chapter 9 problem, not a reasoning or range problem, and no amount of CoT fixes it. If your output degrades because the model loses the thread over many turns or cannot be controlled about which tools it reaches for, that is agentic control — Chapter 11.

One honest caveat: the behavioral claims here are reliable and reproducible. The mechanistic story — mode versus breadth of the output distribution, attention salience, trajectory-shape conditioning — is a useful intuition built on Chapter 1's foundations, not a claim grounded in interpretability research. Treat the behavior as something you can rely on and the mechanism as something you keep testing.

---

## LLM Exercises

**Exercise 1 — Generate and examine.** Take one multi-step arithmetic problem. Run three prompts on a non-reasoning-tuned model: direct answer, zero-shot "let's think step by step," and a few-shot prompt with reasoning traces in the exemplars. Record accuracy and trace quality across five runs each. Then run the same direct-answer prompt on a reasoning-tuned model. Write two sentences explaining any difference between the reasoning-tuned model's direct answer and the non-reasoning model's CoT-prompted answer, in terms of where the step-shaped trajectory came from.

**Exercise 2 — Apply to known context.** Build an Alternative Approaches prompt for a contestable engineering decision in your domain. Then run a deliberately bad version — "give me five approaches" with no axis, no trade-off dimensions, no "where does it break first." Diagnose which range failure modes the bad version exhibits (pseudo-diversity, false breadth, over-enumeration) and rewrite it so each piece of range is expensive to fake.

**Exercise 3 — Stress-test a claim.** The chapter claims that on reasoning-tuned models, explicit CoT adds answer-to-answer variance and sometimes flips correct answers. Design an experiment: which model, which task, how many runs, and what measurement would confirm or falsify the claim? Run it if you have model access and report whether the result matches the prediction.

**Exercise 4 — Draft a professional deliverable.** Build a one-page reasoning and range decision card: a flowchart from the §8.5 procedure that takes model tier, answer-uniqueness, and failure symptom as inputs and outputs one of {few-shot, zero-shot CoT, nothing, self-consistency, Alternative Approaches, Game Play, Tail Generation, "this is a Chapter 9 brittleness problem," "this is a Chapter 11 control problem"}. Each terminal node must name the mechanism that justifies it.

---

## References

- Wei, J., et al. (2022). Chain-of-Thought Prompting Elicits Reasoning in Large Language Models. *NeurIPS 2022*. arXiv:2201.11903.
- Kojima, T., et al. (2022). Large Language Models are Zero-Shot Reasoners. *NeurIPS 2022*. arXiv:2205.11916.
- Wang, B., et al. (2023). Towards Understanding Chain-of-Thought Prompting: An Empirical Study of What Matters. *ACL 2023*. arXiv:2212.10001.
- Wang, X., & Zhou, D. (2024). Chain-of-Thought Reasoning Without Prompting. *NeurIPS 2024*. arXiv:2402.10200.
- Wang, X., et al. (2022). Self-Consistency Improves Chain of Thought Reasoning in Language Models. *ICLR 2023*. arXiv:2203.11171.
- Meincke, L., Mollick, E. R., Mollick, L., & Shapiro, D. (2025). Prompting Science Report 2: The Decreasing Value of Chain of Thought in Prompting. Wharton Generative AI Labs. arXiv:2506.07142.
- Brown, T. B., et al. (2020). Language Models are Few-Shot Learners. *NeurIPS 2020*. arXiv:2005.14165.
- Min, S., et al. (2022). Rethinking the Role of Demonstrations: What Makes In-Context Learning Work? *EMNLP 2022*. arXiv:2202.12837.
- Lu, Y., et al. (2022). Fantastically Ordered Prompts and Where to Find Them: Overcoming Few-Shot Prompt Order Sensitivity. *ACL 2022*.
- White, J., et al. (2023). A Prompt Pattern Catalog to Enhance Prompt Engineering with ChatGPT. arXiv:2302.11382.

---

## Prompts

Use these prompts with Claude to generate interactive D3 v7 versions of the figures in this chapter. Each produces a standalone HTML file you can open in a browser and modify freely.

**Prerequisites:** Load `NEU/CLAUDE.md` and `NEU/DESIGN.md` into your Claude project context before using these prompts. They define the stack, naming conventions, color system, and typography the figures use.

---

### Figure 8.1 — Two opposite moves on one sampling machine

A single horizontal axis diagram, single HTML file, inline CSS, D3 v7 from the CDN. Left pole "tighten the path" (greedy decode → few-shot → zero-shot CoT → few-shot CoT); right pole "widen the sample" (Alternative Approaches → self-consistency → Game Play / Tail Generation). Mark the mode at the left, breadth at the right. Red marks the axis; ink for the labels. Caption: both directions are operations on the same output distribution.

> Reference implementation: `d3/08-reasoning-and-range-patterns-fig-01.html`

---

### Figure 8.2 — The reasoning was in the distribution, not the words

A three-route diagram, single HTML file, D3 v7 CDN. The same model reaching a step-shaped trajectory three ways: direct greedy answer (wrong, marked); "let's think step by step" prompt (conditions the path); top-k decoding inspection (surfaces the latent path, no prompt change). Red marks the shared correct trajectory. Caption: the capability was latent; the phrase is one way to reach it.

> Reference implementation: `d3/08-reasoning-and-range-patterns-fig-02.html`

---

### Figure 8.3 — Adding "let's think step by step" lifts accuracy

A grouped bar chart, single HTML file, D3 v7 CDN, zero baseline. Two benchmarks (MultiArith, GSM8K) on x; accuracy 0–100 on y; paired bars for "direct" vs "+ step by step" (17.7→78.7 and 10.4→40.7). Red for the CoT-prompted bars, ink for the direct bars. Caption: a five-word trigger changes which region is decoded, not the model's knowledge.

> Reference implementation: `d3/08-reasoning-and-range-patterns-fig-03.html`

---

### Figure 8.4 — Draw many trajectories, take the majority vote

A self-consistency schematic, single HTML file, D3 v7 CDN. One prompt fans out to N sampled chain-of-thought paths (non-zero temperature); each ends in an answer; a tally box takes the plurality. Red marks the winning answer; ink for the sampled paths. Caption: variance reduction by averaging, with no self-judgment step.

> Reference implementation: `d3/08-reasoning-and-range-patterns-fig-04.html`

---

### Figure 8.5 — Whether to elicit reasoning depends on the model tier

A four-row tier table rendered as a diagram, single HTML file, D3 v7 CDN. Rows: older/base large model (CoT helps), mid-tier instruction-tuned (helps modestly), reasoning-tuned (add nothing — redundant + variance), any tier with decoding control (CoT-decoding alternative). Red marks the "add nothing" row. Caption: route by tier first.

> Reference implementation: `d3/08-reasoning-and-range-patterns-fig-05.html`
