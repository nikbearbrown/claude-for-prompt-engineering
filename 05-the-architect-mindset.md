# Chapter 5 — The Architect Mindset: Structured Prompt Frameworks
*A prompt is not a sentence you polish until it works — it is a system with parts, and the parts that say "do not" are doing more work than you think.*

---

Wordsville is an educational-content platform that generates illustrated vocabulary lessons for primary-school children: a target word, a child-safe definition, an example sentence, and a one-line illustration brief an artist can render. The team's first prompt was the kind everyone writes first:

> *"Write a vocabulary lesson for the word 'gallop' for kids."*

On Tuesday it produced something lovely — a clean definition, a horse in a sentence, a sweet illustration brief. The team shipped it. On Wednesday, the same prompt on the word *"trial"* produced a definition that drifted into courtroom procedure, an example sentence referencing a criminal defendant, and an illustration brief featuring a judge's gavel. Technically a correct sense of the word. Entirely wrong for seven-year-olds. The next word was *"shoot."* The team did not screenshot what came back.

Nothing was wrong with the model. The prompt under-specified the task so completely that the model was free to sample from an enormous space of valid-but-unwanted completions. "For kids" is not a constraint; it is a vibe. The prompt named a task and then left every architectural decision — reading level, which word sense, safety boundaries, output shape — to the sampler. And a sampler conditioned on an under-specified prompt is the stochastic machine of Chapter 1, doing precisely what it should: drawing from the whole distribution that nobody narrowed.

By the end of this chapter the Wordsville prompt will be a system with named parts, and we will be able to point at the specific layer that fixes the *"trial"* failure and the specific layer that fixes the *"shoot"* failure. They are not the same layer. That pointing-ability is the whole return on the architect mindset.

![The four layers of a prompt: root, constraints, persona, and format are separable concerns, each conditioning the output distribution differently, so each can be debugged without disturbing the others.](../images/05-the-architect-mindset-fig-01.png)
*Figure 5.1 — The four layers of a prompt*

---

## What a prompt is made of

Strip any production prompt down and you find four separable concerns. Keeping them separable is the architect mindset — because when they are tangled into one paragraph, you cannot change one without disturbing the others, and you cannot tell which layer failed.

**Root.** The task itself and the frame around it: what is being produced, for what purpose, in what situation. The root answers "what is this prompt for?" The Tuesday prompt was all root — *write a vocabulary lesson for kids* — and almost nothing else. Root is necessary and almost never sufficient.

**Constraints.** The boundaries on the output — positive (*must include an example sentence*) and, crucially, negative (*must not use any word sense involving violence, law, or adult themes*). Constraints are where most reliability lives, and the last section of this chapter argues the negative ones are doing more work than they get credit for.

**Persona.** Who is speaking — the identity, register, and domain stance the model adopts. Wordsville might use *"You are a warm primary-school teacher."* Persona is a real architectural layer with documented effects and documented failure modes, and it is large enough to earn its own chapter. We name it here and hand it to Chapter 6.

**Format.** The shape of the output — fields, order, length, machine-readability. Wordsville needs a strict object: word, definition, example sentence, illustration brief. Format is what makes a prompt's output consumable by a system rather than just readable by a person.

The separation pays because each layer maps to a different kind of conditioning on the output distribution. The root selects the broad region of output space — lessons, not recipes. Constraints carve mass out of that region — no violent senses. Persona reweights toward a stylistic sub-region — warm, simple register. Format collapses the surviving mass onto a rigid structure. They compose, and because they compose you can debug them independently: change the format without touching the safety constraints, swap the persona without re-deriving the task. A single tangled paragraph offers none of this; every edit is a global edit with unpredictable reach.

There is a misconception worth naming: "Persona, constraints, format — that's just a longer prompt; the model reads it all as one blob anyway." It does read it as one token stream, true. But you do not have to author it as one blob, and the discipline of separating the layers is what lets you find the failing one. The model's flat input does not require flat authoring.

---

## PAST: a scaffold for the input

PAST is a scaffold for the thing you hand the model. It forces the root and part of the constraints into four named slots: **P**roblem, **A**ction, **S**teps, **T**ask. Read it as: here is the situation, here is what I want done about it, here is how to go about it, here is the precise deliverable.

**Problem** — the situation and its stakes. Why does this task exist? For Wordsville: *"Children aged 6–8 are learning new vocabulary and need definitions they can actually understand; existing dictionary definitions are written for adults and confuse them."* Stating the problem does real work: it gives the model the *purpose* against which it can resolve ambiguity. The *"trial"* failure happened partly because the model had no stated purpose telling it a courtroom sense was wrong for this reader.

**Action** — the high-level verb. What is the one thing to do? *"Write one complete vocabulary lesson for a single target word."* This bounds scope: not a lesson plan, not five words, not a quiz. One lesson. The Action slot refuses to let the model expand the task definition by sampling from its own assumptions about what "a vocabulary lesson" entails.

**Steps** — the decomposition. In what order, by what method? *"(1) Choose the word sense most familiar to a 6–8-year-old. (2) Write a definition under 15 words using only common words. (3) Write one example sentence set in everyday child experience. (4) Write a one-line illustration brief."* Steps are where you encode method, and they double as a lightweight chain-of-thought scaffold. Critically, Step 1 is where the *"trial"* failure gets fixed: an explicit instruction to select the child-familiar sense, before any words are written.

**Task** — the precise deliverable and its acceptance condition. What exactly must come back, and how will you know it is right? *"Return the four fields. The lesson is acceptable only if a 6-year-old could read the definition aloud and a parent would find the illustration brief unobjectionable."* Task is the slot most practitioners skip and the one that most reduces the gap between "the model did something" and "the model did the thing."

PAST is not magic words. It is a checklist that converts an under-specified request into a specified one by refusing to let you leave a slot empty. The Tuesday prompt had an Action and a thin Task and nothing else. Filling Problem and Steps is what closes the ambiguity the sampler was exploiting.

| Slot | What it specifies | Which failure it prevents |
|---|---|---|
| **P**roblem | The situation and its stakes — why the task exists | Wrong word sense (no stated purpose to resolve ambiguity against) |
| **A**ction | The one high-level verb; bounds the scope | Task expansion (the model inventing what "a lesson" entails) |
| **S**teps | The ordered method/decomposition | Skipped method — e.g., not choosing the child-familiar sense first |
| **T**ask | The precise deliverable + acceptance condition | The "did something" vs. "did the thing" gap |

---

## PLFR: a scaffold for the pipeline

PAST structures the input. PLFR structures the pipeline — the path from a prompt to a result you can trust: **P**rompt, **L**ogic, **F**ormat, **R**esult.

**Prompt** — the authored input, often a PAST-structured one, plus persona and constraints. This is the layer everything else is built on.

**Logic** — the reasoning or processing the model performs, and the place you decide how much reasoning to elicit and whether to expose it. For Wordsville the logic is light — sense selection, then simplification. For the investment assistant of Chapter 4, it is heavy and worth making explicit. Logic is also where you decide whether a single call suffices or whether the task needs decomposition into multiple calls — the seam where prompt engineering becomes prompt-system engineering.

**Format** — the contract on the output's shape. PLFR promotes format from "ask nicely" to "validated contract." Wordsville's format is a JSON object with four required string fields; the format layer says not just "return JSON" but "these keys, these types, these length bounds." The distinction matters when downstream code breaks on a missing field.

**Result** — the validated output, and the explicit recognition that generation and validation are different steps. This is the PLFR move that PAST alone does not make: PLFR insists that something checks the Result against the format contract and the Task's acceptance condition before it ships. For Wordsville: a schema validator confirms the four fields exist and the definition is under 15 words; a safety classifier — a *separate* call, following Chapter 4's context-isolation logic — screens the illustration brief. The *"shoot"* failure is fixed here, at the Result layer, by an external check the generating model cannot talk its way past. Not by a better-worded prompt.

The relationship between the scaffolds: PAST builds the Prompt slot of PLFR. PAST is the zoom-in on input authoring; PLFR is the zoom-out to the whole input→output→validation pipeline. A small task may need only PAST. Any task whose wrong outputs are costly needs the Result discipline that PLFR makes explicit.

![PAST nests inside the Prompt stage of PLFR: PAST structures the authored input, which is the first stage of the larger Prompt → Logic → Format → Result pipeline that ends in an external validation step.](../images/05-the-architect-mindset-fig-02.png)
*Figure 5.2 — PAST nests inside the Prompt stage of PLFR*

There is a misconception worth defusing: "If the prompt is good enough, I don't need a validation step." This is the Chapter 4 error in new packaging. The model that wrote the output is the worst judge of the output — same distribution, same blind spots. PLFR's Result layer exists precisely because no prompt makes the generator a reliable self-validator.

---

## Negative constraints are first-class — the entropy argument

Here is the chapter's sharpest claim, and it is mechanistic. Practitioners treat "don't do X" lines as defensive padding — nice-to-haves you add after the real prompt. That is backwards. Negative constraints are among the most powerful tools you have, and there is a clean reason why, grounded in Chapter 1's picture of output as a sample from a distribution.

The model produces output by sampling tokens from a conditional distribution $p(y \mid \text{prompt})$. The uncertainty in that distribution — how spread out it is across possible outputs — is measured by its entropy:

$$H(Y \mid \text{prompt}) = -\sum_{y} p(y \mid \text{prompt}) \log p(y \mid \text{prompt})$$

High entropy means the model is choosing among many plausible continuations; that is exactly the freedom that let *"trial"* wander into a courtroom. Every constraint you add changes the conditioning, and a well-aimed constraint removes probability mass from a region of output space. Removing mass from a region, when the surviving mass renormalizes onto fewer effective outcomes, lowers the entropy. Lower entropy means a narrower admissible set: fewer ways for the output to be wrong, more agreement across samples.

Now the asymmetry that makes negative constraints first-class rather than redundant with positive ones. A positive constraint — *"use simple words"* — reweights toward a region but leaves the complement diffusely populated. "Simple" is itself a wide target. A negative constraint — *"do not use any word sense involving violence, law, courts, or adult themes; do not use any word longer than three syllables"* — does something sharper: it zeros out specific, nameable regions of the distribution. You are not nudging the model toward good outputs; you are deleting the bad ones you can anticipate.

Here is an intuition that makes this precise. Imagine the output space as a field, and the model's probability mass as soil heaped over it. A positive constraint piles more soil on the good patch; rain still falls everywhere else. A negative constraint fences off and removes the soil from the bad patches entirely; the probability mass that would have sat there now falls on what remains. The second operation shapes the distribution more decisively because it acts on what you know you do not want, which is often far easier to specify precisely than what you do want. "Don't mention violence" is crisp. "Be appropriate" is not.

For Wordsville, the negative constraints carry the safety guarantee:

> *Do not use any word sense involving violence, weapons, law, courts, death, or adult themes. Do not use vocabulary above a 2nd-grade reading level. Do not exceed 15 words in the definition. Do not write an illustration brief depicting any person in distress.*

Each line removes a nameable failure region. Together they collapse the admissible output set to something a 6-year-old's parent would approve — and they make the output more consistent across the thousands of words Wordsville will process, because a lower-entropy distribution produces less sample-to-sample variation. Reliability and safety, from the same mechanism.

One honest limit. Constraints are conditioning, not hard guarantees. The model can violate a stated negative constraint, exactly as it can ignore any other instruction. The entropy argument says negative constraints strongly reshape the distribution. It does not say they bound it to zero. That residual is why the PLFR Result layer exists — the negative constraints make violations rare, the external validator catches the rest. The two layers are complementary: constraints lower the rate; validation catches the remainder.

![Deletion concentrates mass more than reweighting: a positive constraint piles more probability on the good region while the rest stays populated, whereas a negative constraint removes the failure regions entirely, forcing the surviving mass onto what remains.](../images/05-the-architect-mindset-fig-03.png)
*Figure 5.3 — Deletion concentrates mass more than reweighting*

---

## Assembling the Wordsville prompt system

Now put the layers and scaffolds together. The Wordsville prompt as an architected system, with each part labeled:

```
[PERSONA — handed to Chapter 6]
You are a warm, patient primary-school teacher who explains words
to young children.

[ROOT / PAST-Problem]
Children aged 6–8 are learning new vocabulary. Adult dictionary definitions
confuse them and sometimes expose them to inappropriate word senses.

[PAST-Action]
Write one complete vocabulary lesson for a single target word: {WORD}.

[PAST-Steps / LOGIC]
1. Choose the word sense most familiar to a 6–8-year-old.
2. Write a definition under 15 words using only common, 2nd-grade words.
3. Write one example sentence set in everyday child experience.
4. Write a one-line illustration brief suitable for a children's picture book.

[CONSTRAINTS — negative, first-class]
Do NOT use any word sense involving violence, weapons, law, courts, death,
  or adult themes.
Do NOT use vocabulary above a 2nd-grade reading level.
Do NOT exceed 15 words in the definition.
Do NOT depict any person in distress in the illustration brief.

[FORMAT]
Return ONLY a JSON object:
{ "word": str, "definition": str (<15 words),
  "example_sentence": str, "illustration_brief": str }

[PAST-Task / acceptance condition]
The lesson is acceptable only if a 6-year-old could read the definition aloud
and a parent would find the illustration brief unobjectionable.
```

And the PLFR pipeline around it: Prompt is the block above. Logic is a single call; the four Steps are the model's reasoning scaffold. Format is schema-validated JSON. Result is a schema validator — fields present, definition length — plus a separate-context safety classifier on the illustration brief before anything ships.

Trace the two original failures through the finished system. The *"trial"* failure — wrong word sense — is fixed at **PAST-Step 1 plus the negative constraint** on law and courts: the model is told to pick the child-familiar sense and forbidden the courtroom region of output space. The *"shoot"* failure — a genuinely unsafe completion — is reduced by the negative constraint on violence and weapons, then caught at the **PLFR Result layer** by the external classifier if the constraint is ever violated. Two failures, two different layers, each pointable.

![Two failures pinned to two different layers: the "trial" wrong-sense failure is fixed at the Steps slot plus the negative constraint, while the "shoot" unsafe-completion failure is caught at the PLFR Result layer by an external classifier.](../images/05-the-architect-mindset-fig-04.png)
*Figure 5.4 — Two failures pinned to two different layers*

That pointing-ability is the whole return on the architect mindset.

---

## From prompt to system

The shift this chapter asks for is from prompt-as-phrasing to prompt-as-system. The ad-hoc author tweaks words and hopes the distribution narrows; the architect names the layers, fills the scaffold slots, deletes the failure regions on purpose, and puts a validator at the exit. Same model, same task — categorically different reliability. Architecture is the leverage point, not the model. The Wordsville system is not smarter than the Tuesday prompt. It is *structured*, and structure is what converts a stochastic machine's freedom into engineered behavior.

PAST stops you from under-specifying the input — its slots are a refusal to leave the task ambiguous. PLFR stops you from conflating generation with validation — its Result layer is the external check Chapter 4 argued no self-governing prompt can replace. And the negative-constraint argument reframes "do not" lines from padding to your most decisive control over the output distribution, because deleting a nameable failure region is a stronger operation than nudging toward a vague good.

One layer we named and deferred: persona. We slotted "You are a warm primary-school teacher" into the Wordsville system as one architectural layer among four, and that is exactly the right way to hold it — not as the prompt's personality, but as a component with a job and, as Chapter 6 will show, its own documented failure modes. Persona is large enough and treacherous enough to earn a chapter of its own. We hand it forward not as decoration but as the next architectural layer to engineer with the same discipline we just applied to root, constraints, and format.

---

## LLM Exercises

**Exercise 1 — Generate and examine.** Take an ad-hoc prompt you have actually used — "summarize this article," "write an email to my colleague," anything single-sentence. Submit it to a model five times. Record how much the outputs vary. Then identify which of the four layers (root, constraints, persona, format) are present and which are absent. Write one sentence predicting which absent layer is responsible for most of the variance you observed.

**Exercise 2 — Apply to known context.** Take the ad-hoc prompt *"Summarize this customer review for our dashboard."* Rewrite it as a PAST-structured prompt, filling every slot explicitly. For each slot, state which class of failure an empty slot would have allowed. Then add three negative constraints, each justified by the failure region it removes.

**Exercise 3 — Stress-test a claim.** The chapter claims negative constraints are more powerful than matched positive ones because they remove mass from specific regions rather than reweighting toward a vague good. Design an experiment to test this: same task, one prompt with a positive framing ("use child-appropriate language"), one with a matched negative framing ("do not use any word sense involving violence, law, or adult themes"). Run each ten times on a word like "trial" or "sentence." Does the negative constraint produce fewer unsafe outputs? Write two sentences explaining the result in terms of the entropy argument.

**Exercise 4 — Draft a professional deliverable.** Design a complete prompt system for a task of your own choosing — not Wordsville. Deliver: the four labeled layers (root, constraints, persona, format), at least three negative constraints each justified by the failure region it deletes, a PAST-structured input, and a one-paragraph PLFR pipeline naming the Result-layer validation. State one acceptance condition a reviewer could test against.

---

## References

- White, J., Fu, Q., Hays, S., Sandborn, M., Olea, C., Gilbert, H., Elnashar, A., Spencer-Smith, J., & Schmidt, D. C. (2023). A Prompt Pattern Catalog to Enhance Prompt Engineering with ChatGPT. arXiv:2302.11382.
- Shannon, C. E. (1948). A Mathematical Theory of Communication. *Bell System Technical Journal*, 27(3), 379–423. *(Source of the entropy definition used in the negative-constraints section.)*

---

## Prompts

Use these prompts with Claude to generate interactive D3 v7 versions of the figures in this chapter. Each produces a standalone HTML file you can open in a browser and modify freely.

**Prerequisites:** Load `NEU/CLAUDE.md` and `NEU/DESIGN.md` into your Claude project context before using these prompts. They define the stack, naming conventions, color system, and typography the figures use.

---

### Figure 5.1 — The four layers of a prompt

A stacked-layer diagram, single HTML file, inline CSS, D3 v7 from the CDN. Four labeled bands — Root, Constraints, Persona, Format — each annotated with how it conditions the output distribution (selects the region / carves mass out / reweights toward a sub-region / collapses onto a structure). Red highlights the Constraints layer; ink for the rest. Caption: separable layers can be debugged independently.

> Reference implementation: `d3/05-the-architect-mindset-fig-01.html`

---

### Figure 5.2 — PAST nests inside the Prompt stage of PLFR

A nested-pipeline diagram, single HTML file, D3 v7 CDN. Outer pipeline: Prompt → Logic → Format → Result (with Result drawn as an external validation step, in red). Inside the Prompt stage, draw the PAST sub-structure (Problem, Action, Steps, Task). Ink for the outer pipeline. Caption: PAST authors the input; PLFR governs the whole input→output→validation path.

> Reference implementation: `d3/05-the-architect-mindset-fig-02.html`

---

### Figure 5.3 — Deletion concentrates mass more than reweighting

Two probability-landscape panels, single HTML file, D3 v7 CDN, shared zero baseline. Left "positive constraint": mass added to a good region, the rest still populated. Right "negative constraint": failure regions fenced off and removed, surviving mass concentrated on what remains (in red). Ink for the landscape. Caption: deleting a nameable failure region is a stronger operation than nudging toward a vague good.

> Reference implementation: `d3/05-the-architect-mindset-fig-03.html`

---

### Figure 5.4 — Two failures pinned to two different layers

A two-column mapping diagram, single HTML file, D3 v7 CDN. Left column: the "trial" wrong-sense failure → fixed at PAST-Step 1 + the negative constraint on law/courts. Right column: the "shoot" unsafe-completion failure → caught at the PLFR Result layer by an external classifier. Red marks the layer that fixes each. Caption: two failures, two different layers, each pointable.

> Reference implementation: `d3/05-the-architect-mindset-fig-04.html`
