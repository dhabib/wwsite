# Measure First, Tune Second: A Methodology for Developing and Maintaining Voice Fidelity in GPT-Based Content Generation

**David Habib**
Pilot WMS — February 2026

---

## Abstract

Large language models exhibit strong tendencies toward a default register — professional, balanced, moderately confident — that resists deviation through naive prompting. When deploying LLMs as the generation layer of a content management system, the ability to reliably reproduce a configurable editorial voice becomes a first-order product requirement, not a cosmetic preference. This paper describes a methodology for parameterizing, measuring, and iteratively improving voice fidelity in GPT-4o-mini, developed during the construction of Pilot WMS, a RAG-powered headless CMS with AI content generation. The central methodological claim is that building a quantitative test harness *before* attempting optimization — measuring the problem before proposing solutions — produces better outcomes than intuition-driven prompt engineering. Over two rounds of optimization, average fidelity improved from 72/100 (no voice control) to 78/100 (full pipeline), with gains concentrated in the voices that diverge most from the model's natural register. The results reveal a structural asymmetry: voices that align with GPT-4o-mini's default behavior score 87–92 with no effort, while extreme voices (casual, contrarian, humorous) plateau around 70–73, exposing the limits of prompt-only steering without fine-tuning.

---

## 1. Introduction

The promise of LLM-powered content generation is that an organization can maintain a consistent editorial voice across hundreds of generated artifacts — articles, research summaries, hook content — without manual rewriting at scale. In practice, this promise collides with a fundamental property of instruction-tuned language models: they have been optimized to produce helpful, professional, well-structured prose. This is precisely the voice most organizations *do not* need reproduced, because it is the voice of no one in particular.

A casual blogger does not write like GPT-4o-mini. A contrarian provocateur does not hedge the way GPT-4o-mini hedges. An academic researcher may want dense citations and heavy qualification, but the *pattern* of that qualification differs from the model's default caution. The challenge is not merely to instruct the model to "be casual" or "be contrarian," but to specify voice along multiple independent dimensions and then verify — quantitatively — that the generated output matches the specification.

Pilot WMS addresses this through an 11-dimensional voice parameterization system, a constitutional prompt architecture, and a measurement-driven optimization loop. This paper describes what we built, how we measured it, what we learned, and where the limits are.

---

## 2. The Case for Measurement-First Development

### 2.1 The Intuition Trap

The default approach to prompt engineering is iterative: write a prompt, read the output, adjust the prompt, repeat. This works for individual generations but fails systematically for voice fidelity, for three reasons:

1. **Human evaluation is noisy.** Reading a generated article and judging whether it "sounds casual enough" is subjective and inconsistent across evaluators, sessions, and moods.
2. **Confirmation bias is strong.** After spending an hour tuning a system prompt, there is enormous pressure to perceive improvement in the output, regardless of whether it exists.
3. **Marginal gains are invisible.** The difference between 67/100 and 73/100 fidelity is real and meaningful at scale, but difficult to perceive in a single sample.

### 2.2 Building the Harness Before Knowing the Outcome

The decision to build a quantitative voice fidelity test harness *before* attempting any optimization was deliberate and, in retrospect, the most important methodological choice in the project. The harness was designed to answer a simple question: does a given intervention improve voice reproduction, and by how much?

The critical discipline was this: we did not know what the baseline scores would be. We did not know whether the system prompt was helping at all. We did not know which voices would be easy and which would be hard. The harness was built to *discover* these facts, not to *confirm* hypotheses about them.

This matters because the results were surprising. Corporate Analyst scored 92/100 with *no system prompt at all* — the model's default register is already a corporate analyst. Had we only tested with extreme voices, we might have concluded the system prompt was essential. Had we only tested with corporate voices, we might have concluded voice fidelity was a solved problem. The harness revealed the asymmetry that shaped every subsequent decision.

### 2.3 Structure of the Harness

The test harness operates as follows:

1. **Define voice profiles.** Four profiles spanning the register space: Academic Researcher (formal, hedged, citation-dense), Casual Blogger (informal, humorous, narrative), Corporate Analyst (structured, data-driven, confident), and Contrarian Provocateur (challenging, Socratic, witty).

2. **Fix the source material.** Three factual passages about remote work history, urban economics, and employee well-being. The same passages are used for every generation, eliminating source content as a variable.

3. **Generate under controlled conditions.** Each profile is tested under multiple modes: (A) no system prompt, (B) system prompt with voice instructions, (C) full pipeline with all enhancements.

4. **Analyze output via LLM.** A separate GPT-4o-mini call scores the generated text on each of the 11 voice dimensions, producing a 0–100 score per dimension. This is the same analysis used in production for drift detection.

5. **Compute fidelity.** Fidelity = 100 - MAE, where MAE is the mean absolute error across all 11 dimensions between intended and observed scores.

The use of LLM-as-judge for voice analysis introduces its own noise, but this noise is *consistent* across modes, making relative comparisons valid even if absolute scores carry uncertainty. The alternative — human evaluation panels — would be prohibitively expensive for the rapid iteration cycles required.

---

## 3. Voice Parameterization: The 11-Dimension Model

### 3.1 Dimension Selection

Voice is not a single axis. A piece of writing can be simultaneously formal and humorous, concise and citation-dense, hedged and contrarian. Representing voice as a single "tone" enum (Academic / Professional / Conversational / Casual) collapses this multidimensional space into four points, losing almost all expressive power.

Pilot WMS parameterizes voice along 11 continuous dimensions, each on a 0–100 scale:

| Dimension | Low (0) | High (100) |
|-----------|---------|------------|
| Brevity | Concise | Expansive |
| Research Depth | Surface | Deep multi-source |
| Formality | Casual | Scholarly |
| Specificity | General | Precise/technical |
| Confidence | Hedged/tentative | Assertive/direct |
| Citation Density | Sparse attribution | Rich attribution |
| Stance | Consensus-aligned | Contrarian/provocative |
| Evidence Style | Anecdotal/narrative | Data-driven/empirical |
| Rhetoric | Direct/declarative | Socratic/questioning |
| Humor | Serious/formal | Frequent humor/wit |
| Structure | Free-flowing/organic | Highly structured |

These dimensions were derived from a combination of writing craft literature, editorial style guides, and empirical observation of the axes along which LLM output can most meaningfully vary. The legacy `Tone` enum (Academic, Professional, Conversational, Casual) is now auto-derived from the Formality axis, preserving backward compatibility while exposing the full parameter space.

### 3.2 Graduated Instructions

Early experiments revealed that binary instructions ("be formal" / "be casual") produced minimal differentiation. A 5-tier graduated instruction system was developed, mapping each dimension's 0–100 range to five behavioral tiers:

- **Extreme Low (0–15):** Explicit DO/AVOID directives. Example for Confidence: *"Use heavily hedged language throughout. DO: 'This could potentially suggest...' AVOID: 'clearly', 'obviously', definitive claims."*
- **Low (16–35):** Directional guidance without specific phrasing.
- **Moderate (36–65):** Balanced default with room for contextual judgment.
- **High (66–85):** Strong directional guidance.
- **Extreme High (86–100):** Explicit DO/AVOID directives, mirroring the extreme low tier.

The extreme tiers carry the most behavioral weight. This is by design: values near 50 should produce minimal deviation from the model's natural register, while extreme values should produce unmistakable stylistic shifts. The graduated system ensures that every point on the 0–100 scale maps to concrete guidance, eliminating dead zones where changes to the slider produce no change in output.

---

## 4. Prompt Architecture and the Saliency Problem

### 4.1 The Saliency Hypothesis

Language models attend to different parts of the context window with different weight. In the system-user message architecture, the final instructions in the system message — those immediately preceding the user message — occupy the most salient position. Content earlier in the system message, while still attended to, has diminishing influence as the user message (with its source passages and constraints) grows longer.

This observation led to a specific architectural hypothesis: the ordering of components within the system prompt should place compliance-oriented instructions (provenance requirements) early and behavioral instructions (voice parameters) late. The compliance instructions are binary — the model either cites sources or it does not — and are robust to lower saliency. The voice instructions are continuous and subtle, requiring maximum saliency to influence generation.

### 4.2 Round 1: System Prompt Construction

The Round 1 system prompt was constructed in the following order:

1. Constitutional text (organization-specific purpose and constraints)
2. Tone instructions (from formality-derived Tone enum)
3. Voice parameters (11-dimension graduated instructions)
4. Voice identity (optional natural-language description from exemplar analysis)
5. Voice exemplars (text excerpts from exemplar artifacts)
6. Provenance requirement (source-only constraint)

This ordering placed the provenance block at the end — the most salient position — and voice instructions in the middle. The test harness measured the impact:

| Profile | [A] No Prompt | [B] Round 1 | Delta |
|---------|--------------|-------------|-------|
| Academic Researcher | 80 | 83 | +3 |
| Casual Blogger | 55 | 67 | +12 |
| Corporate Analyst | 92 | 91 | -1 |
| Contrarian Provocateur | 62 | 67 | +5 |
| **Average** | **72** | **77** | **+5** |

The system prompt helped substantially for extreme voices (+12 for Casual Blogger) but was neutral or slightly negative for the Corporate Analyst, whose voice already matches the model's default. Average fidelity reached 77/100, but Casual Blogger and Contrarian Provocateur plateaued in the high 60s.

### 4.3 Round 2: Reordering and Reinforcement

Five interventions were applied simultaneously in Round 2:

**C1. Prompt reordering.** Provenance moved to position 2 (after constitutional text); voice instructions moved to the end. Hypothesis: voice saliency increases when instructions immediately precede the user message.

**C2. User-prompt voice reinforcement.** A 2–3 line "VOICE REMINDER" injected at the end of the user prompt, after source material and constraints but before the generation instruction. The reminder extracts the 3–4 dimensions most distant from the neutral midpoint (50) and restates them in plain language: *"Write with heavily hedged confidence, expansive length, very dense source citations, and highly structured organization."* Hypothesis: voice instructions fade across long source context; a reminder at the decision boundary restores them.

**C3. Sampling parameters.** `top_p` and `frequency_penalty` derived from voice dimensions:
- `top_p = 0.95 - (specificity / 100) * 0.25` — higher specificity narrows vocabulary
- `frequency_penalty = (rhetoric / 100) * 0.4` — higher rhetoric encourages varied phrasing

**C4. Continuous temperature.** Temperature derived from `formality` and `humor` rather than the discrete Tone enum: `base = 0.9 - (formality / 100) * 0.6; humorBoost = (humor / 100) * 0.15`. This produces a continuous range of 0.1–1.0 rather than the four fixed values (0.3, 0.5, 0.7, 0.9) of the enum mapping.

**C5. Few-shot calibration sample.** A ~150-word paragraph generated using the current voice settings on a neutral topic, stored on the Constitution record and injected as an assistant message between the system and user messages. This provides the model with a concrete example of the desired voice before it encounters the actual generation task. The sample is regenerated only when voice settings change (~$0.001 per regeneration).

**C6. Two-pass voice rewrite.** An optional second LLM call that takes the initial generation and rewrites it to better match the voice parameters, preserving all factual claims and citations. Exposed as a user-facing "High Fidelity Mode" toggle with a cost warning (~2x generation cost).

---

## 5. Results

### 5.1 Three-Way Comparison

| Profile | [A] No Prompt | [B] Round 1 | [C] Full Pipeline | B→C |
|---------|--------------|-------------|-------------------|-----|
| Academic Researcher | 80 | 83 | 80 | -3 |
| Casual Blogger | 55 | 67 | 70 | +3 |
| Corporate Analyst | 92 | 91 | 87 | -4 |
| Contrarian Provocateur | 62 | 67 | 73 | +6 |
| **Average** | **72** | **77** | **78** | **+1** |

### 5.2 Per-Dimension Analysis

The per-dimension deltas reveal where the full pipeline helps and where it does not.

**Contrarian Provocateur** showed the broadest improvement, with 9 of 11 dimensions improving from B to C:

| Dimension | Intended | [B] Delta | [C] Delta | Improved? |
|-----------|----------|-----------|-----------|-----------|
| Stance | 95 | 35 | 30 | Yes |
| Rhetoric | 90 | 40 | 30 | Yes |
| Humor | 70 | 60 | 50 | Yes |
| Citation Density | 30 | 40 | 30 | Yes |
| Evidence Style | 40 | 40 | 30 | Yes |
| Structure | 40 | 45 | 35 | Yes |
| Formality | 40 | 35 | 30 | Yes |
| Specificity | 50 | 30 | 25 | Yes |
| Research Depth | 55 | 30 | 25 | Yes |

This profile represents the ideal use case for the full pipeline: a voice that diverges substantially from the model's default on multiple dimensions simultaneously.

**Academic Researcher** showed mixed results. Formality improved (delta 15→15, holding at near-perfect), but Confidence worsened (delta 70→60) — the model struggled to maintain heavy hedging even with explicit instructions. The brevity dimension regressed significantly (delta 15→55), suggesting the rewrite pass produced more concise output than intended for an expansive voice.

**Corporate Analyst** regressed on Citation Density (delta 10→30) and Rhetoric (delta 5→15). Since this profile is already near the model's default register, the additional pipeline complexity introduced noise rather than correction.

### 5.3 The Asymmetry Finding

The most important finding is structural: **GPT-4o-mini has an inherent voice, and that voice is a corporate analyst.** Profiles that align with this default (formal, structured, data-driven, moderately confident) achieve fidelity scores above 87 with minimal intervention. Profiles that deviate from it require progressively more intervention and still plateau 15–20 points lower.

This asymmetry is not a bug in the prompt engineering — it is a property of the model. Instruction-tuned models are optimized to be helpful, clear, and professional. "Be unhelpful" or "be unclear" are adversarial to the training objective. "Be casual," "be contrarian," and "be funny" are not adversarial in the same way, but they compete with deeply embedded behavioral priors.

The practical implication is that **voice fidelity is not a uniform challenge.** A system that works well for 70% of use cases (professional, moderate voices) may fail meaningfully for the remaining 30% (extreme voices). The test harness, by measuring across the full register space, prevents this failure mode from going undetected.

---

## 6. Discussion

### 6.1 The Value of Negative Results

Several interventions produced neutral or negative results for specific profiles:

- **Prompt reordering (C1)** did not produce a measurable improvement in isolation. The saliency hypothesis may be correct in principle but insufficient in magnitude to overcome the model's behavioral priors.
- **Two-pass rewrite (C6)** helped extreme voices (+6 for Contrarian Provocateur) but hurt moderate voices (-4 for Corporate Analyst). The rewrite pass introduces an additional opportunity for the model to "correct" toward its default register.
- **Sampling parameters (C3)** had subtle effects that were difficult to isolate from the other interventions.

These negative results are as valuable as the positive ones. Without the test harness, each of these interventions would have been evaluated by reading a few samples and forming an impression. The impression would likely have been positive — the system prompt is longer, more sophisticated, and clearly more "engineered" — regardless of whether fidelity actually improved. The harness prevents this self-deception.

### 6.2 Cost-Fidelity Tradeoff

The full pipeline approximately doubles generation cost due to the calibration sample generation (one-time, ~$0.001), the assistant prefill (adds ~200 input tokens per generation, ~$0.00003), and the optional two-pass rewrite (~$0.002 per generation). For a system generating hundreds of projections per day, the rewrite pass represents a meaningful cost increase.

The data suggests a targeted strategy: enable High Fidelity Mode only for voices that score below 75 with the standard pipeline. For voices above 85 (Corporate Analyst, Academic Researcher with most settings), the standard pipeline is sufficient and the rewrite pass may actually reduce fidelity.

### 6.3 LLM-as-Judge Limitations

Using GPT-4o-mini to evaluate its own output introduces circular dependency. The model's perception of what constitutes "humor: 85" may not align with human perception. The analysis model may systematically undercount or overcount certain dimensions.

However, the methodology does not require absolute accuracy of the judge — only consistency. If the judge consistently scores humor at 60 when the intended value is 85, and this scoring is stable across modes, then the relative comparison (Mode A vs. B vs. C) remains valid. The fidelity score should be interpreted as a *relative* measure of voice adherence, not an absolute one.

A future refinement would use a different model family (e.g., Claude) as the judge to break the self-evaluation loop, or calibrate the LLM judge against human evaluation panels on a small sample.

### 6.4 Toward Fine-Tuning

The 70–73 plateau for extreme voices suggests that prompt-only steering has fundamental limits. The model's behavioral priors, encoded in billions of parameters during RLHF, cannot be fully overridden by a few hundred tokens of system prompt. For organizations that require extreme voice fidelity — a humor-heavy brand voice, a consistently contrarian editorial stance — fine-tuning on voice-matched examples may be the only path to scores above 85.

The 11-dimension parameterization and test harness developed here would transfer directly to a fine-tuning workflow: the dimensions define the training objective, and the harness measures convergence. The few-shot calibration sample (C5) could serve as seed data for a fine-tuning dataset.

---

## 7. Methodology Recommendations

For teams building voice-controlled LLM generation systems, we recommend the following:

1. **Build the test harness first.** Define your voice profiles, fix your source material, and measure the baseline *before* writing a single line of prompt engineering. You cannot improve what you cannot measure, and you cannot measure what you have not defined.

2. **Test across the register space.** Include at least one profile that matches the model's default (to establish an upper bound) and at least one profile that diverges maximally (to establish a lower bound). The gap between these two reveals the difficulty of your problem.

3. **Parameterize voice continuously, not categorically.** Enums are lossy. An 11-dimension continuous parameterization captures meaningful variation that a 4-value tone selector cannot.

4. **Use graduated instructions with explicit behavioral directives.** The extreme tiers (0–15, 86–100) should include DO/AVOID patterns with concrete phrasing examples. Moderate tiers can be more general.

5. **Make expensive interventions opt-in.** Two-pass rewriting helps extreme voices but hurts moderate ones. Expose it as a user-facing toggle rather than applying it universally.

6. **Treat negative results as data, not failure.** An intervention that reduces fidelity for one profile while improving it for another is not a failure — it is information about when the intervention should be applied.

7. **Expect diminishing returns from prompt engineering.** The gap between 72 (no intervention) and 78 (full pipeline) represents the practical ceiling of prompt-only steering for GPT-4o-mini. Further gains require fine-tuning, model selection, or acceptance of the asymmetry.

---

## 8. Conclusion

Voice fidelity in LLM-generated content is a measurable, optimizable property — but only if you measure it. The central contribution of this work is not any individual prompt engineering technique, but the methodology of building a quantitative test harness before attempting optimization. This harness revealed the structural asymmetry between default-aligned and divergent voices, guided the selection and evaluation of interventions, and prevented confirmation bias from masquerading as progress.

The techniques developed — graduated 11-dimension voice parameterization, prompt reordering for saliency, user-prompt voice reinforcement, continuous sampling parameter derivation, few-shot calibration, and two-pass rewriting — collectively improved average fidelity from 72 to 78, with the largest gains (+6 to +15 points) in the voices that needed them most. The remaining gap to 85+ for extreme voices appears to be a structural property of instruction-tuned models that prompt engineering alone cannot bridge.

The test harness, the parameterization model, and the optimization methodology are open for adaptation. The most important thing is to measure first.

---

*Pilot WMS is a RAG-powered headless CMS with AI content generation, editorial workflow, and provenance tracking. The voice fidelity system described here is in production, serving configurable editorial voices across web, email, and social channels.*
