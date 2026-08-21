# Chapter 7 — Structuring and Governing Output
*Four patterns that shape what the model emits, and one that decides what is allowed to leave — and why the difference between them is not cosmetic.*

---

In late 2024, a mid-sized logistics firm wired a language model into the middle of an automated intake pipeline. Inbound emails from carriers — "we can take the Tuesday load, Chicago to Memphis, $2,400, dry van" — were passed to the model, which was asked to "extract the key details so we can book the load." The model's output flowed straight into a parser that populated a database row and triggered a booking. No human in the loop. The prompt had been tested on a few dozen examples and worked.

For the first two weeks it held. Then the failures began arriving. One email got back a response that opened *"Sure! Here are the details from this carrier's offer:"* followed by a bulleted list. Another came back as a tidy paragraph. A third returned the fields correctly but reordered — origin and destination swapped from the order the parser expected, because the carrier had written the email destination-first. The parser, built around the assumption that line two was always the price, silently booked one load at the wrong rate and dropped three others into an error queue that no one was watching.

The root cause was not a model that "got worse." Run any one of those inputs in isolation and the model's extraction was substantively correct every time. The failure lived in the gap between the model's output distribution and the parser's input expectations. The model had been asked for "the key details" — a request with enormous output entropy. A hundred phrasings, several plausible orderings, optional preambles, variable punctuation — all reasonable completions, none of them what the rigid downstream consumer required. Free-form natural language is high-entropy output. A deterministic parser is a low-entropy consumer. Something has to absorb the difference. The engineers had assumed that something would be the model's good judgment. It was not. It was noise.

The fix for this pipeline was not a smarter parser or a bigger model. It was a template. One structural commitment, emitted first, that locked the shape of the output before the model generated a single field value. This chapter is about how that works — mechanically, not as a guess — and about four patterns that each exploit the same mechanism in a different scope. Then it is about the one additional pattern that governs what is allowed to leave once the output is shaped.

---

## The mechanism first

From Chapter 1: an autoregressive language model generates one token at a time, and each token is sampled from a probability distribution conditioned on every token already in the context — including the tokens the model itself just emitted. The model does not plan a whole response and then write it. It writes token $t$, and that token becomes part of the conditioning context for token $t+1$. The joint probability of an output sequence factors as:

$$P(x_1, x_2, \ldots, x_n) = \prod_{t=1}^{n} P(x_t \mid x_1, \ldots, x_{t-1})$$

This is the entire mechanism of output-structuring patterns. Stated as a falsifiable claim: **committing to structural tokens early narrows the conditional distribution over all later tokens.**

When the model has just emitted a heading like `## Price:` or a field label like `ORIGIN:`, the set of high-probability continuations is dramatically smaller than it was before that token existed. After `ORIGIN:` the next tokens are overwhelmingly likely to be a city and state, not a chatty preamble, not a paragraph about the carrier's reliability, not an equipment type. The structural token has done work: it has spent a small amount of the output budget to collapse the variance of everything downstream of it.

This is why supplying the scaffold beats asking for the content and hoping the formatting comes out right. If you ask for "the key details," the model first has to decide on a structure — high entropy, many structures are plausible — and then fill it. If you supply the structure, the deciding is already done and committed in context. The model only fills. You have moved the high-entropy decision out of the model's sampling loop and into your prompt, where it is fixed and deterministic.

The four patterns in this chapter differ only in *what* structure they commit early and *how much* of the task that structure governs. The same mechanism, aimed at four different scopes.

A note on names before we start. Template and Recipe come from the prompt-pattern catalog literature; Outline Expansion, Menu Actions, and the Semantic Filter later in this chapter are common practitioner patterns, named here for convenience. Treat all of them as rules of thumb — heuristics that exploit one mechanism — not as formal results. The value is the mechanism and the named failure mode, not the label.

![The same mechanism, aimed at four scopes: Template commits the format, Recipe commits the procedure, Outline Expansion commits the hierarchy, and Menu Actions commits the action space — one early-commitment mechanism at four different scopes.](../images/07-structuring-and-governing-output-fig-02.png)
*Figure 7.2 — The same mechanism, aimed at four scopes*

---

## Template Pattern: committing the format

The Template Pattern supplies a fixed output structure with labeled placeholders and instructs the model to fill them without altering the structure. The logistics fix was a template:

```
Extract the load details from the email. Output using exactly this template
and nothing else. Do not add commentary. Use UNKNOWN for any field not
present in the email.

ORIGIN: <city, state>
DESTINATION: <city, state>
RATE_USD: <number only, no symbols>
EQUIPMENT: <dry van | reefer | flatbed | other>
PICKUP_DATE: <YYYY-MM-DD>
```

Three things in that prompt are doing mechanistic work. First, the labeled scaffold itself: each `LABEL:` token, once emitted, conditions the following tokens toward exactly the field that label names, in the order the template fixes. The destination-first carrier email no longer reorders the output because the output order is governed by the scaffold, not by the input's order. Second, the value-type hints inside the angle brackets narrow each slot's distribution further — the `RATE_USD` slot is conditioned toward a bare number, collapsing the `$2,400` / `2400 dollars` / `2.4k` variance. Third, the closed enumeration for `EQUIPMENT` is a miniature version of the Menu Actions pattern embedded in a slot: it reduces that field to a four-way choice.

The Template Pattern produces the most reliably machine-parseable output of any structuring pattern in this chapter, because it maps most directly onto the consumer's expectations. The strongest production use is to make the template a literal specification of the downstream schema — a JSON shape, a CSV header row, a fixed Markdown layout. The model's job collapses from "communicate the details" to "fill these slots," and filling slots is a low-entropy task.

**The failure mode lives here in its sharpest form.** A template is a set of slots that must be filled. If the input does not contain the information a slot demands, the model faces a conflict: honor the committed structure, or admit the field is empty. Under that conflict, the model will frequently confabulate a value — invent a pickup date, guess an equipment type, manufacture an origin city — because emitting some plausible token is higher-probability than violating the committed structure. This is why the example includes `UNKNOWN`: an explicit escape value gives the model a legal, low-entropy completion for an empty slot, so it no longer has to choose between fabrication and structure violation. **A template without an escape hatch is a confabulation generator.** The escape hatch is not a politeness. It is the architectural component that converts the structure from a trap into a scaffold.

There is a quieter cost too. A rigid template emits only its slots, so a genuinely important detail that has no slot — "the carrier mentioned the bridge on I-40 is closed" — is silently dropped, because the committed structure has no place to put it and the model's variance has been collapsed precisely to prevent off-template emission. The engineering response is to add a deliberate free-text channel: `NOTES: <anything else relevant>`. Structure exactly the part of the output whose variance hurts your consumer. Leave a channel for everything else.

![Committing a structural token narrows what comes next: once a field label like ORIGIN: is emitted, the high-probability continuations collapse from a chatty paragraph to exactly the value that label names, in the order the template fixes.](../images/07-structuring-and-governing-output-fig-01.png)
*Figure 7.1 — Committing a structural token narrows what comes next*

---

## Recipe Pattern: committing the procedure

The Recipe Pattern asks for a complete, ordered sequence of steps to reach a stated goal, given some steps the user already knows. White et al.'s canonical framing: *"I am trying to achieve X. I know I need to do steps A and B. Provide a complete sequence of steps. Fill in any missing steps and identify any unnecessary steps."*

Where the Template governs format, the Recipe governs *procedure*: the unit of structure is the ordered step, and the entropy it collapses is the entropy of which steps belong and in what order. The mechanism is the same conditioning argument applied to a sequence. Once the model has emitted `1. Provision the VM and create a non-root user`, the distribution over step 2 is conditioned on step 1 having happened. Each committed step narrows the plausible successors. This is why a Recipe surfaces missing steps that a free-form "how do I do this?" answer routinely skips: the explicit enumeration forces the model to make the decomposition visible, and a gap in a numbered chain is more conspicuous — to the model's own next-token prediction, and to you — than a gap in flowing prose.

The Recipe is strongest where the failure of unstructured procedural output is silent step-skipping: the model gives you four of the seven things you needed, the four it gave you were correct, and nothing looks wrong until step five fails in the real world. Forcing the complete ordered sequence converts a silent omission into a visible one.

The Recipe's version of slot-confabulation is the **fabricated step**: asked for a complete sequence, the model may invent steps that do not belong, or assert ordering constraints that are not real, because "complete and ordered" is the structure it committed to. The Recipe is somewhat less exposed than the Template, because a step list does not have a fixed number of slots that all demand filling — the model can legitimately produce a short recipe. But the pressure is there whenever the goal is vague or the domain is thin. Mitigation: state the goal precisely enough that "complete" is well-defined, and explicitly license the model to mark steps as conditional ("only if you are using a custom domain") rather than forcing every step to read as mandatory.

---

## Outline Expansion Pattern: committing the hierarchy

The Outline Expansion Pattern starts from a high-level skeleton and iteratively expands each node into sub-nodes and detail. It is the pattern for long outputs — documents, plans, curricula, reports — where the thing most likely to go wrong is not the format of any one part but the coherence of the whole.

The mechanism is the conditioning argument plus an explicit answer to the long-range-coherence problem introduced by autoregressive generation. A model generating a 4,000-word document in one pass must hold the global structure coherent while every token is sampled from a distribution dominated by local recent context. Over a long generation, early organizational commitments lose salience in the attention budget — the same salience-decay effect Chapter 6 names for persona drift — and the document wanders: sections overlap, the promised structure quietly dissolves, the conclusion answers a question the introduction did not ask.

Outline Expansion attacks this by separating the level that carries global structure from the level that carries local detail. The skeleton is short. It fits comfortably in context and stays salient. Each expansion step then operates in a small local context — expand *this* node — where the model's strength at local fluency is exactly what you want and where long-range drift has no room to accumulate. You are not asking the model to hold a whole document coherent in one pass. You are asking it to hold a short skeleton coherent and then fill leaves one at a time, with the skeleton always in view.

The operational bonus: because expansion is iterative, you can correct the skeleton before paying for the detail. That is the cheapest possible point to catch a structural mistake — before any words have been written under the bad heading.

The failure mode is the **over-expanded branch**: a node that genuinely warrants two sentences gets expanded into five sub-points because the pattern's rhythm is "expand each node," and the model manufactures filler to populate a branch the content does not support. It is the slot-filling failure one level up. The mitigation is to make expansion content-driven, not uniform: instruct the model that nodes may legitimately stay shallow, and review the skeleton for nodes that exist only because the outline looked lopsided without them.

---

## Menu Actions Pattern: committing the action space

The Menu Actions Pattern defines a fixed set of commands the model can invoke and instructs it to map each input to exactly one of them. It is the pattern for bounded interactive systems — a support bot, a command interface, a triage front-end — where the danger is not malformed output but out-of-scope output: the model helpfully doing something it was never supposed to do.

```
You operate a document system with exactly these actions:
  CREATE <title>       — start a new document
  APPEND <id> <text>   — add text to a document
  SUMMARIZE <id>       — return a summary
  LIST                 — list all document IDs

For each user message, respond with exactly one action in the above syntax,
or with UNSUPPORTED: <reason> if the request maps to no action. Take no
other action and add no commentary.
```

Where the three structuring patterns so far constrain how the model expresses an answer, Menu Actions constrains what answers are even legal. The mechanism is still conditioning, but the committed structure is a closed set of permissible outputs, and the entropy it collapses is the entropy of the action space itself. Out-of-scope behavior — the model freelancing a fifth action because the user asked nicely — drops sharply, because a response outside the menu is now a low-probability completion relative to the committed in-menu options.

When the model's output triggers real operations, an unbounded action space is an unbounded liability. A closed menu makes the set of things the system can do auditable and finite.

The failure mode is **forced misclassification**: an input that maps to no menu item still has to become some output, and without an escape the model will jam it into the nearest action — routing a billing question to `SUMMARIZE` because the menu offered nothing better. The mitigation is the explicit escape: `UNSUPPORTED: <reason>` is to the Menu pattern what `UNKNOWN` is to the Template — a legal, low-entropy output for the genuinely out-of-scope input.

![Every pattern's empty slot, made safe by an escape value: when committed structure outruns the available content the model confabulates to fill it, and an explicit escape value gives it a legal low-entropy completion instead.](../images/07-structuring-and-governing-output-fig-03.png)
*Figure 7.3 — Every pattern's empty slot, made safe by an escape value*

| Pattern | Structure committed | Empty-slot confabulation | Escape hatch |
|---|---|---|---|
| Template | Labeled fields | Invented field values | `UNKNOWN` |
| Recipe | Ordered steps | Fabricated steps | Conditional-step labels |
| Outline Expansion | Hierarchy | Over-expanded filler | Permission for shallow nodes |
| Menu Actions | Closed action set | Forced misclassification | `UNSUPPORTED: <reason>` |

---

## The shared cost, named once

Four patterns, four expressions of one failure mode:

> **Over-structuring forces the model to confabulate to fill a committed structure whose slots outrun the available content.**

The structure is a promise the model made in its own context, and the autoregressive mechanism that makes structure *work* — committed tokens narrow later distributions — is the same mechanism that makes the failure bite. Once `PICKUP_DATE:` is committed, emitting a date is higher-probability than emitting nothing, even when no date exists. The pattern that buys reliability by spending tokens on structure pays for it by manufacturing content to honor the structure it cannot fill.

The mitigation generalizes. Every pattern is made safe by the same move: **give the model a legal low-entropy output for the case the structure does not fit.** The escape hatch converts the structure from a confabulation trap into a reliable scaffold, because it gives the model a high-probability honest completion to reach for instead of a fabricated one.

And the second quiet cost generalizes too: structure suppresses content that does not fit a slot. The engineer who understands both costs structures exactly the part of the output whose variance hurts the consumer, installs the escape for the parts that do not fit, and adds a free-text channel — `NOTES: <anything else relevant>` — for the details that should not be suppressed.

More structure is not always safer. Structure is conditioning, and conditioning toward a shape the content cannot honestly fill is conditioning toward fabrication. The right amount of structure is the amount that covers the variance that actually hurts your consumer, plus an escape for everything else — never one slot more.

---

## Governing the exit: the Semantic Filter

Structuring shapes the answer. It does not decide whether the shaped answer is *allowed out*. That is the job of the Semantic Filter Pattern.

*"Filter this output to remove or flag content that meets criterion X."*

Where the four structuring patterns lower entropy during generation, the Semantic Filter narrows the admissible output set after generation. You define criteria — relevance, sensitivity, privacy, safety, tone — and the system evaluates produced text against them, blocking or modifying what fails. The relationship to the structuring patterns is clean. Structuring reduces the entropy of the output's form. The Semantic Filter reduces the set of outputs permitted to leave on content grounds. One makes the answer parseable; the other makes it admissible.

### In-prompt filter versus separate classifier

There are two ways to implement a Semantic Filter, and the difference is the load-bearing engineering decision.

**In-prompt filtering:** you instruct the same model, in the same call, to screen its own output: *"After answering, review your response and remove any content revealing personal data."* Cheap and often helpful — and weak in a predictable way. The screening reasoning shares a context window with the content being screened and with whatever pressure produced it. The judge and the defendant are the same process, with the same incentives, in the same room. A user who can talk the generator into emitting disallowed content can usually, in the same breath, talk the self-filter into passing it.

**Separate-model filtering:** you route the output through a distinct call — a separate classifier, or the same base model invoked with a clean context containing only the output and the criteria, not the original prompt or the conversation that produced the content. This is more robust for the same reason Chapter 4 gives for the dissenting sub-agent: **context isolation.** The filter does not see the pressure signal. It cannot be jailbroken by the conversation that jailbroke the generator, because it was never in that conversation.

This is the identical failure mode Chapter 4 named as calibration drift. A dissenter who shares the defendant's context looks like governance and provides little. The Semantic Filter has the same failure: an in-prompt filter is a dissenter sharing the generator's context. A separately invoked filter with a clean context is the real check — and it degrades the same way if you "helpfully" pass it the original prompt for context.

![In-prompt self-filter versus separate classifier: an in-prompt filter shares the generator's context and pressure and can be talked past, while a separately invoked filter with a clean context — criteria only — is the real check.](../images/07-structuring-and-governing-output-fig-04.png)
*Figure 7.4 — In-prompt self-filter versus separate classifier*

### The two-sided error budget

A filter is a classifier, and every classifier has two error types you trade against each other. Over-filtering — false positives — blocks or mangles benign output. A privacy filter that redacts every number destroys a financial report. Over-filtering is the failure that looks safe and quietly destroys utility; it is invisible in safety metrics because nothing harmful got out. You only see it in the complaints about a useless product.

Leakage — false negatives — passes content that violates the criterion. Tightening the threshold trades leakage for over-filtering; they move in opposite directions, always. The engineering question is never "is the filter accurate" but "where is the threshold, given the asymmetric cost of the two errors in this deployment?" A filter on a children's product accepts heavy over-filtering to drive leakage near zero. A filter on an internal research tool for experts accepts more leakage to preserve utility. State the asymmetry explicitly; a filter without a stated error-cost asymmetry is untuned by definition.

One honest limit, stated before you trust the filter: it governs admissibility, not truth. It can keep PII or off-topic prose from leaving. It cannot tell you the load-bearing claim inside an admissible answer is false. That is Chapter 2's orthogonality again, and the Fact-Check List's job — not the filter's.

---

## A decision procedure

The patterns are not interchangeable and the choice is not aesthetic. Match the pattern to where your uncertainty lives.

![Where does the variance that hurts live? Format variance points to Template, procedural variance to Recipe, long-range structural variance to Outline Expansion, action-space variance to Menu Actions, and admissibility to the Semantic Filter — and if the risk is correctness, none of them helps.](../images/07-structuring-and-governing-output-fig-05.png)
*Figure 7.5 — Match the pattern to where the variance that hurts lives*

Does a downstream consumer require a specific format? If a parser, schema, API, or rigid workflow consumes the output, the variance that hurts you is format variance. Use the **Template Pattern** and make the template the consumer's grammar. Always include an escape value for missing fields and a free-text notes slot.

Is the deliverable a procedure where the danger is missing or out-of-order steps? The variance that hurts you is procedural. Use the **Recipe Pattern**. State the goal precisely so "complete" is well-defined, and license conditional steps.

Is the output long, where the danger is global incoherence? The variance that hurts you is long-range structural. Use the **Outline Expansion Pattern**. Lock the skeleton first, review it before paying for detail, allow shallow nodes.

Is the model driving a bounded interactive system that acts? The variance that hurts you is in the action space. Use the **Menu Actions Pattern**. Enumerate the actions, demand exactly one per input, provide an `UNSUPPORTED` escape.

Must some content be kept from leaving on grounds other than shape? If the content is adversarially pressured — safety, jailbreak-resistance — run the filter as a separate call with isolated context. State the error-cost asymmetry so the threshold is tuned, not guessed.

These compose. A common production stack is Outline Expansion → Template → Semantic Filter: expand a skeleton to get a coherent long document, render each leaf through a fixed template so the output is machine-parseable, then screen the rendered output through a separate-context filter before it leaves. When you compose, let each pattern govern its own scope. Do not have them fight over the same tokens.

And there is a sixth answer the procedure must allow: **none of the above.** If your uncertainty lives in content correctness rather than in format, procedure, organization, action space, or admissibility — if the risk is that the model is wrong, not shaped wrong — then no pattern in this chapter helps. A beautifully formatted, filtered template delivers a confident, parseable, admissible, wrong answer. Structure reduces output entropy. The filter narrows admissibility. Neither adds knowledge. Over-structuring a task whose real problem is correctness is worse than not structuring it, because the polish raises the reader's trust without raising the truth.

---

## Back to the pipeline

The logistics firm's failure was never a model failure. The model extracted the right details nearly every time. The failure was an architecture failure: a high-entropy output fed to a low-entropy consumer, with nothing in between to absorb the mismatch. The Template Pattern absorbed it — not by making the model smarter, but by committing the output's shape before the model generated a single value. The destination-first email, the chatty preamble, and the paragraph-instead-of-list all became impossible completions rather than rare ones. And had any extracted field carried something the firm could not let leave — a driver's personal phone number pasted into the email body — a Semantic Filter on the rendered row, run as a separate call, would have caught it without the generator ever getting a vote.

Output structure is not decoration applied after the fact. It is conditioning applied up front. Because the model samples each token from a distribution shaped by the tokens already committed, the structure you put into context early is the structure that governs everything sampled later. That is the whole reason these patterns work, and through the empty slot the structure cannot fill, the whole reason they fail. Governance is the complement: structuring shapes what the model emits; the Semantic Filter decides what is allowed out, and it is only real when the check runs somewhere the generator cannot talk its way past.

---

## LLM Exercises

**Exercise 1 — Generate and examine.** Send the same free-form extraction request ("extract the key details from this email") to a model five times on the same input. Record how many distinct output formats you receive. Then add a template scaffold and repeat. Report the format variance before and after, and write two sentences explaining the difference in terms of the conditioning mechanism.

**Exercise 2 — Apply to known context.** For each of four scenarios — (a) extracting invoice fields for a database, (b) writing a twelve-section runbook, (c) a triage bot that opens tickets, (d) a how-to for configuring TLS — name which structuring pattern fits, which kind of variance it collapses, and write the one escape-hatch clause that makes it safe.

**Exercise 3 — Stress-test the escape hatch.** Take a template you have written. Run it on an input where one required field is genuinely absent — no value exists in the source. Run it with the escape hatch (`UNKNOWN` or equivalent) and without it. Record what the model emits for the missing slot in each case. Write one sentence explaining the result in terms of the competing pressures on the model when the slot is committed but the value is absent.

**Exercise 4 — Draft a professional deliverable.** Build an Outline Expansion → Template → Semantic Filter stack for a task in your domain. Implement the filter twice — once in-prompt (self-review) and once as a separate isolated call. Feed both an adversarial input that pressures the system to leak a piece of sensitive content. Report the leakage difference, state your error-cost asymmetry, and specify where you would set the threshold for this deployment.

---

## References

- White, J., Fu, Q., Hays, S., Sandborn, M., Olea, C., Gilbert, H., Elnashar, A., Spencer-Smith, J., & Schmidt, D. C. (2023). A Prompt Pattern Catalog to Enhance Prompt Engineering with ChatGPT. arXiv:2302.11382. *(Source for the Template and Recipe patterns. Outline Expansion, Menu Actions, and the Semantic Filter are common practitioner patterns — rules of thumb, not formal results from any single paper.)*
- Shannon, C. E. (1948). A Mathematical Theory of Communication. *Bell System Technical Journal*, 27(3), 379–423. *(Source of the entropy definition underlying the "structure reduces output entropy" claim.)*

---

## Prompts

Use these prompts with Claude to generate interactive D3 v7 versions of the figures in this chapter. Each produces a standalone HTML file you can open in a browser and modify freely.

**Prerequisites:** Load `NEU/CLAUDE.md` and `NEU/DESIGN.md` into your Claude project context before using these prompts. They define the stack, naming conventions, color system, and typography the figures use.

---

### Figure 7.1 — Committing a structural token narrows what comes next

A before/after distribution diagram, single HTML file, inline CSS, D3 v7 from the CDN. Left: a wide distribution over many plausible continuations after "extract the key details." Right: after emitting a field label (`ORIGIN:`), a sharply narrowed distribution concentrated on the value that label names. Red marks the collapsed nucleus; ink for the wide prior. Caption: a committed structural token spends a little budget to collapse downstream variance.

> Reference implementation: `d3/07-structuring-and-governing-output-fig-01.html`

---

### Figure 7.2 — The same mechanism, aimed at four scopes

A four-panel scope diagram, single HTML file, D3 v7 CDN. One row, four panels — Template (format), Recipe (procedure), Outline Expansion (hierarchy), Menu Actions (action space) — each showing the same "commit structure early" mechanism applied at a different scope. Red marks the committed structure in each. Caption: one mechanism, four scopes.

> Reference implementation: `d3/07-structuring-and-governing-output-fig-02.html`

---

### Figure 7.3 — Every pattern's empty slot, made safe by an escape value

A 4-row reference diagram, single HTML file, D3 v7 CDN. Rows: Template/Recipe/Outline Expansion/Menu Actions; columns: structure committed, empty-slot confabulation, escape value (UNKNOWN / conditional labels / shallow-node permission / UNSUPPORTED). Red marks the escape value that converts a trap into a scaffold. Caption: an escape value gives the model a legal low-entropy completion.

> Reference implementation: `d3/07-structuring-and-governing-output-fig-03.html`

---

### Figure 7.4 — In-prompt self-filter versus separate classifier

A two-architecture comparison, single HTML file, D3 v7 CDN. Left: generator and filter share one context window (including user pressure) — drawn as one box. Right: generator output passed to a second call with a clean context (criteria only), returning pass/fail — drawn as two isolated boxes, the isolated check in red. Caption: context isolation makes the separate-classifier version the real check.

> Reference implementation: `d3/07-structuring-and-governing-output-fig-04.html`

---

### Figure 7.5 — Match the pattern to where the variance that hurts lives

A decision tree, single HTML file, D3 v7 CDN. Root: "where does your uncertainty live?" Branches to format → Template, procedure → Recipe, long-range structure → Outline Expansion, action space → Menu Actions, admissibility → Semantic Filter, and correctness → "none of these helps" (in red). Caption: structure reduces entropy; it does not add knowledge.

> Reference implementation: `d3/07-structuring-and-governing-output-fig-05.html`
