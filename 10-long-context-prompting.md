# Chapter 10 — Long-Context Prompting: Position, Retrieval, and Injection
*Where in the window you put a thing decides whether the model uses it.*

---

A legal-discovery assistant for the "80 Days to Stay" matter is handed a 300-page exhibit bundle assembled from an SEC production. Somewhere in that bundle — page 150, dead center — is a single clause that exonerates the defendant. The assembled prompt is straightforward: 300 pages of documents, then the question, *"Does any document here contradict the SEC's allegation that the filing was knowingly false?"* The model reads it, reasons fluently, and answers: *"No document in the production contradicts the allegation."*

The clause was right there. The model had it in context. It read past it.

This is not a hallucination in the Chapter 2 sense — the model did not fabricate. It is not a capability gap — a model that finds the clause when you paste page 150 alone misses it when the clause is buried mid-bundle. The failure is positional. Where the clause sat in the window decided whether the model used it. Had the retrieval system placed that page at the start or end of the assembled context, the model would very likely have caught it.

This chapter makes that claim precise, gives you placement rules you can test, and then shows that the same machinery is an attack surface a careless prompt leaves wide open.

---

## The U-shaped curve

You already have a prior for this, even if you have never seen a language model. Recite a grocery list someone read you ten seconds ago and you will reliably get the first item and the last item. The middle is mush. Psychologists named this the serial-position effect — strong recall at the start, called primacy, and at the end, called recency, with weak recall in the middle. Murdock's 1962 free-recall curve is the canonical human plot.

Language models show a curve of the same shape. The mechanisms are not the same — there is no human short-term store inside a transformer — but the picture rhymes, and it is a useful hook so long as we puncture it before it misleads.

The empirical claim: Liu et al. (2024), published in TACL, ran two tasks — multi-document question answering and a synthetic key-value retrieval task. In each, they placed the relevant document or key at different positions in the input and measured accuracy. The result is the spine of this chapter.

> Accuracy is highest when the relevant information sits at the beginning or the end of the input, and drops sharply when the model must retrieve it from the middle — a U-shaped curve. The effect holds even for models explicitly built for long contexts.

Let $a(p)$ be accuracy when the target fact sits at relative position $p \in [0,1]$. The empirical finding is:

$$a(0) \approx a(1) > a(0.5)$$

with the gap between the edges and the middle reaching tens of percentage points on hard configurations. This is not a quirk to memorize. It is a consequence of how the model was trained and how it allocates attention. The two halves of the curve have different mechanisms and different fixability, and collapsing them into one story is the most common way to get the engineering wrong.

![Lost in the middle: accuracy by position of the relevant fact follows a U-shaped curve — high when the fact sits at the start or end of the context and sagging sharply in the middle, even for long-context models.](../images/10-long-context-prompting-fig-01.png)
*Figure 10.1 — Lost in the middle: accuracy by position of the relevant fact*

### Why recency: the autoregressive artifact

Here is the load-bearing mechanism. A model is trained by predicting each token from the tokens before it:

$$t_i \sim P(t_i \mid t_0, t_1, \ldots, t_{i-1})$$

During pretraining the objective rewards getting the next token right given the immediately preceding ones. In ordinary text, the strongest predictor of the next word is usually the words right before it — local context dominates. A model that learns to weight recent tokens heavily is a model that minimizes its training loss. So the autoregressive objective itself builds in a bias toward the most recent context.

![Each token is predicted from the ones immediately before it: because local context dominates next-token prediction during pretraining, the model learns to weight recent tokens heavily — recency bias is the training objective showing through.](../images/10-long-context-prompting-fig-02.png)
*Figure 10.2 — Each token is predicted from the ones immediately before it*

That is why recency is stable: it persists even near the window limit and even in long-context models, because it is baked into what the model was optimized to do. It is not a bug that a bigger window fixes. It is the training objective showing through. The thing practitioners call "recency bias" is the autoregressive next-token objective viewed from the prompt-design side.

### Why primacy: attention sinks, and why it is weaker

The start of the sequence also gets outsized attention, through a different route. Transformers exhibit an attention-sink phenomenon — early tokens, often the very first, receive large attention weight somewhat regardless of their relevance, acting as a place for attention to "park." Hsieh et al.'s "Found in the Middle" (2024) showed that the U-shaped accuracy curve maps onto a U-shaped attention curve: start and end tokens get more attention irrespective of whether they are relevant.

But primacy is weaker than recency and decays as the window fills. Early tokens compete with ever more material as the context grows; the start-of-document advantage that holds at 10K tokens erodes at 100K. Greg Kamradt's needle-in-a-haystack pressure tests show exactly this texture: top-of-document recall holds better than mid-document recall, but degrades as total length grows.

### How much of the curve is fixable

Here is where the easy story breaks, and you should let it. "Found in the Middle" proposed a training-free, inference-time calibration: estimate the position-only component of the attention bias and subtract it, recovering up to 15 percentage points of RAG accuracy. That recovery is evidence that part of the bias is a correctable attention artifact — not the immutable consequence of the training objective.

The honest position is two-part:

The **autoregressive component** of recency bias comes from the training objective. It is structural and hard to fix at inference time. The **attention-sink component** is an architectural artifact and is partially correctable — the 15-point calibration recovery is the evidence.

Anyone who tells you the whole curve is an unfixable law of nature is overclaiming. Anyone who tells you it is a dial you can tune away is also overclaiming. The curve has a hard floor and a soft ceiling.

One misconception to defuse explicitly: "a bigger context window means the model reads everything in it." No. Advertised context length is the nominal window. RULER (Hsieh et al. 2024) made this sharp: the vanilla needle-in-a-haystack test measures only superficial retrieval, and once you extend it with multi-key, variable-tracking, and aggregation tasks, most models hold consistent accuracy only well below their advertised token limit. The effective context is much smaller than the nominal context. A 200K window does not mean 200K usable tokens. It means 200K tokens you are allowed to send, of which a model- and task-dependent fraction is actually attended to with full reliability.

---

## Multi-needle collapse: the benchmark that hides the failure

Single-needle retrieval — find the one out-of-place sentence in this haystack — is largely solved on frontier models. Numbers near 95 to 96 percent are routine for one fact in one long document. If your evaluation stops there, you will conclude your long-context system works.

It does not, and here is the gotcha. Ask for more than one fact at once and the floor drops out. Work on multilingual needle-in-a-haystack reports that the best models hit roughly 96 percent in English with a single target sentence, but three target sentences in English drops to roughly 40 percent. That is the empirical basis for "multi-needle collapse," and the figure matters: it is approximately 40 percent for three needles, not the rounder, more optimistic 60 percent that circulates in practitioner summaries.

![Single-needle retrieval is solved; multi-needle synthesis is not: finding one fact in a long context runs near 96% in English, but holding three facts simultaneously salient while composing an answer drops to roughly 40%.](../images/10-long-context-prompting-fig-03.png)
*Figure 10.3 — Single-needle retrieval is solved; multi-needle synthesis is not*

Why does adding a second and third needle break things so much harder than the first? Two mechanisms compound. First, synthesis is not retrieval: pulling one fact to the front of generation is easy; holding three facts simultaneously salient while composing an answer competes for the same finite attention budget. Second, the U-curve applies to each needle separately — with three needles scattered through a long context, at least one of them is probably sitting in the middle, where the curve says it will be missed.

The broader lesson generalizes past long context, and it is the Chapter 9 lesson wearing new clothes: a benchmark that looks solved can hide the real failure. Single-needle at 95 percent is a press release. Three-needle synthesis at 40 percent is your production system.

For the "80 Days to Stay" SEC task, the relevant question is rarely "find the one clause." It is usually "reconcile the revenue-recognition note on page 40, the restatement disclosure on page 150, and the auditor's letter on page 280." That is a three-needle synthesis across a long bundle. The single-needle benchmark that made the pipeline look ready is exactly the wrong evidence for the job it has to do.

---

## Placement rules you can test

The mechanism converts directly into engineering. Four rules, each stated as a falsifiable claim with the experiment that would break it.

**Rule 1 — Put the highest-priority instruction last.** Because recency persists even near the window limit, the instruction you most need obeyed belongs at the end of the assembled prompt, after the documents, not in a system preamble buried under 200 pages.

*Falsifiable test:* take a working prompt, move the key instruction to the middle, hold everything else fixed, and measure the compliance drop. If compliance does not fall, the rule is wrong for your model and task.

**Rule 2 — Put critical retrieved facts at the extremes; filler in the middle.** Rerank retrieved chunks so the highest-relevance passages land at the start and end of the assembled context, deliberately exploiting the U-curve.

*Falsifiable test:* A/B the rerank order — best-first-and-last versus best-in-the-middle — and measure answer accuracy. If middle-placement does not hurt, the U-curve is not biting your pipeline.

**Rule 3 — Put safety constraints at the end, and never let user content be the last thing.** Recency that helps compliance also helps an attacker whose content is recent. Safety rules go last; untrusted user or document content must not occupy the final, highest-attention position.

*Falsifiable test:* run a position sweep on a known-adversarial prompt and confirm attack success is lower with rules-last than with user-content-last.

**Rule 4 — Do not trust single-needle benchmarks as evidence of synthesis ability.** A pipeline that scores 95 percent on one-needle retrieval can score 40 percent on three-needle synthesis.

*Falsifiable test:* run a three-needle synthesis task and watch accuracy fall. If it holds near 95 percent, your synthesis is unusually robust — measure it, do not assume it.

One counterintuitive corollary: more documents can hurt. Teams routinely raise their retrieval count to "improve recall," reasoning that more chunks means more chances to include the right one. Beyond a point this backfires: the correct fact gets buried in the middle of a now-longer context, and the lost-in-the-middle penalty swamps the recall gain. The fix is not more documents. It is better-placed documents.

A schematic of the recommended layout:

```
[ top ]      system role / low-priority context
[ middle ]   filler / mid-relevance retrieved docs
[ end ]      highest-priority instruction
             + safety / constraint rules
             + the most-relevant retrieved facts
             + the user's actual question
```

Recency makes the bottom of that block the most-attended region. That is where the load-bearing tokens go.

![Engineer the layout to the attention curve: low-priority context and filler go in the middle trough, while the high-priority instruction, safety rules, most-relevant facts, and the user's question go at the end — the highest-attention zone.](../images/10-long-context-prompting-fig-05.png)
*Figure 10.5 — Engineer the layout to the attention curve*

---

## Position is an attack surface

Everything above is about quality. The same machinery is a security problem, and this is the part a careless prompt designer never sees coming.

If the model attends most to the front and end of a long context, then an attacker who controls where their content sits controls how much the model attends to it.

**Many-shot jailbreaking.** Anil et al. (2024), Anthropic, showed that stuffing hundreds of faux dialogue turns demonstrating an unwanted behavior into the context overrides alignment. Attack strength scales with shot count — near-zero at 5 shots and reliable around 256, following a power law in the log of the number of shots — and it works across GPT-3.5 and 4, Claude 2.0, Llama 2 70B, and Mistral 7B. Long context is not just a recall problem. It is an attack surface, and in-context demonstrations exploit the same positional and recency machinery that helps legitimate instructions.

**NINJA — position-dependent safety.** The sharper result for this chapter is NINJA, "Jailbreaking in the Haystack" (2025). NINJA appends benign, thematically relevant filler to a harmful goal and varies the position of that goal in the context. On HarmBench, attack success rose from a 23.7% baseline to 58.8% for Llama-3.1-8B-Instruct — also from 23.7% to 42.5% for Qwen2.5-7B-Instruct, and roughly 23% to 29% for Gemini Flash. NINJA explicitly attributes the effect to the same U-shaped primacy and recency bias: putting the harmful goal at the beginning maximizes attack success, and putting it at the end mitigates.

![Where the harmful goal sits changes how often the attack succeeds: on HarmBench, moving a goal to the high-attention front of a long context raised attack success from a 23.7% baseline to 58.8% on Llama-3.1-8B — the same positional machinery that helps legitimate instructions.](../images/10-long-context-prompting-fig-04.png)
*Figure 10.4 — Where the harmful goal sits changes how often the attack succeeds*

State that result precisely, because it is easy to mangle. 58.8% is the post-attack success rate for one specific model, up from a 23.7% baseline. "Up to 58.8% on Llama-3.1-8B, from a 23.7% baseline" is the honest phrasing. "Up to 58.8% attack success" stated bare implies a universal law the data does not support.

The defense falls straight out of the mechanism. Because harmful-goal-at-front maximizes attack success, two structural controls follow. Safety rules go last — recency makes the final instruction the most-attended, so put the constraints there, where they win the attention competition against earlier content. Never let untrusted content be the last thing — if a user or a retrieved document can place their content at the end of the assembled prompt, recency works for them. Wrap user content, and re-append your constraints after it.

This is the same instinct as Chapter 4's context isolation, expressed as a layout rule. "Put instructions last" is not only a quality tip. It is a safety control. The mechanism that makes your good instruction obeyed is the mechanism an attacker borrows; you keep it on your side by owning the final position.

One limit to state honestly: "instructions-last" mitigates injection, it does not eliminate it. A determined attacker controls the position of their content too, and against an adaptive attacker who can append after your rules, recency stops being purely your friend. Instructions-last raises the cost of injection. It does not close the door.

---

## Back to page 150

Return to the exonerating clause. Nothing was wrong with the model and nothing was wrong with the clause. The retrieval system assembled 300 pages and let the one page that mattered fall in the dead center of the window — the trough of the U-curve, the region the autoregressive objective and the attention machinery jointly under-weight. The model read fluently past it because the architecture placed it where the model reads least.

The fix is not a better model and not a cleverer question. The fix is placement: rerank so the high-relevance pages land at the start and end, keep filler in the middle, append the question and the constraints last. Same model, same documents, different layout — and the clause moves from the trough to the edge, where the curve says it gets used.

Architecture is the leverage point, not the model. In long-context prompting, "architecture" is partly just where in the window each token sits. You do not get to change how the model attends. You do get to decide what occupies the high-attention positions. The engineer who treats the context window as an undifferentiated bag of tokens ships systems that miss page 150. The engineer who treats position as a designed variable ships systems that hold up under length and under attack.

One honest caveat to carry forward: the U-shaped curve is robust and replicated, but the exact multi-needle degradation varies by model, language, needle similarity, and distractor design. The approximately-40%-at-three-needles figure is a calibration point, not a law. When a new model or a new domain arrives, the shape is a strong prior and the magnitudes are an experiment you still have to run.

---

## LLM Exercises

**Exercise 1 — Generate and examine.** Take a 500-word filler document, one target fact, and a question the fact answers. Run three prompts that place the fact at the start, the middle, and the end of the filler. Record whether the model uses the fact correctly in each case. Write two sentences explaining any difference in terms of the autoregressive mechanism rather than model "carelessness."

**Exercise 2 — Apply to known context.** You are assembling a RAG pipeline for a three-document reconciliation task. Sketch the before and after of your prompt layout — before applying the placement rules and after. Annotate where the high-priority instruction, the most-relevant facts, the filler, and the safety rules sit in each version. Identify which single change you expect to move accuracy most and why.

**Exercise 3 — Stress-test a claim.** A teammate reports 96 percent on a needle-in-a-haystack benchmark and concludes the pipeline is production-ready for a task that requires synthesizing three facts from a long document. Write the two-sentence objection that names the specific failure the benchmark hides, and the one experiment you would run before agreeing.

**Exercise 4 — Draft a professional deliverable.** Write a one-page placement-rules reference card for your team. It must state all four rules with their falsifiable tests, include the recommended prompt layout schematic, identify which rule doubles as a safety control and explain why, and state the one honest limit of "instructions-last" as a defense against adaptive attackers.

---

## References

- Liu, N. F., Lin, K., Hewitt, J., Paranjape, A., Bevilacqua, M., Petroni, F., & Liang, P. (2024). Lost in the Middle: How Language Models Use Long Contexts. *Transactions of the Association for Computational Linguistics*, 12, 157–173. arXiv:2307.03172.
- Hsieh, C.-P., et al. (2024). RULER: What's the Real Context Size of Your Long-Context Language Models? arXiv:2404.06654.
- Hsieh, C.-Y., et al. (2024). Found in the Middle: Calibrating Positional Attention Bias Improves Long Context Utilization. *ACL 2024 Findings*. arXiv:2406.16008.
- Anil, C., et al. (2024). Many-shot Jailbreaking. Anthropic; NeurIPS 2024. https://www.anthropic.com/research/many-shot-jailbreaking
- NINJA — Jailbreaking in the Haystack (2025). arXiv:2511.04707.
- Multilingual Needle in a Haystack / mLongRR (2024). arXiv:2409.18006.
- Kamradt, G. (2023). Needle In A Haystack — Pressure Testing LLMs. https://github.com/gkamradt/LLMTest_NeedleInAHaystack
- LangChain (2024). Multi Needle in a Haystack. https://blog.langchain.com/multi-needle-in-a-haystack/
- Databricks (2024). Long Context RAG Performance of Large Language Models. arXiv:2411.03538.

---

## Prompts

Use these prompts with Claude to generate interactive D3 v7 versions of the figures in this chapter. Each produces a standalone HTML file you can open in a browser and modify freely.

**Prerequisites:** Load `NEU/CLAUDE.md` and `NEU/DESIGN.md` into your Claude project context before using these prompts. They define the stack, naming conventions, color system, and typography the figures use.

---

### Figure 10.1 — Lost in the middle: accuracy by position of the relevant fact

A line chart, single HTML file, inline CSS, D3 v7 from the CDN. X-axis: relative position of the target fact (0 = start → 1 = end); y-axis: retrieval accuracy, zero baseline. Plot a U-shaped curve, high at both edges and sagging in the middle; annotate the "lost in the middle" trough in red. Caption: accuracy is not uniform across positions.

> Reference implementation: `d3/10-long-context-prompting-fig-01.html`

---

### Figure 10.2 — Each token is predicted from the ones immediately before it

A short autoregressive-chain sketch, single HTML file, D3 v7 CDN. Show a token predicted from a window of immediately preceding tokens, with attention weight concentrated on the most recent ones (heavier in red). Caption: recency bias is the next-token objective viewed from the prompt-design side.

> Reference implementation: `d3/10-long-context-prompting-fig-02.html`

---

### Figure 10.3 — Single-needle retrieval is solved; multi-needle synthesis is not

A grouped bar chart, single HTML file, D3 v7 CDN, zero baseline. X: number of needles (1 vs 3, English); y: accuracy 0–100. Show ≈96% for one needle and ≈40% for three, the collapse bar in red. Caption: synthesis is not retrieval — holding three facts salient competes for one attention budget.

> Reference implementation: `d3/10-long-context-prompting-fig-03.html`

---

### Figure 10.4 — Where the harmful goal sits changes how often the attack succeeds

A bar or line chart of attack success by goal position, single HTML file, D3 v7 CDN, zero baseline. Show attack success rising from a 23.7% baseline toward 58.8% as the goal moves to the front of a long context (Llama-3.1-8B). Red marks the high-success front position. Caption: the same positional machinery that helps instructions helps an attacker.

> Reference implementation: `d3/10-long-context-prompting-fig-04.html`

---

### Figure 10.5 — Engineer the layout to the attention curve

A vertical prompt-layout stack, single HTML file, D3 v7 CDN. Three zones — top (low-priority context/filler), middle (mid-relevance docs), end (high-priority instruction, safety rules, most-relevant facts, user question). A right-side attention gradient is high at top and bottom, low in the middle (red marks the high-attention end zone). Caption: put load-bearing content where the model attends.

> Reference implementation: `d3/10-long-context-prompting-fig-05.html`
