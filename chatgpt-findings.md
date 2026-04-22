# ChatGPT Findings — Platform Constraints and Test Results

> This document captures what we learned about ChatGPT's behavior under the directive framework through real-world testing. If you are deploying the instruction set to ChatGPT, read this first — it will tell you which directives hold and which fail.

---

## Platform Compatibility

**Testing date:** April 21, 2026

**Platform:** ChatGPT Plus (chatgpt.com)

**Model tested:** GPT-5.4 Thinking

**Deployment method:** Native Custom Instructions (the "user preferences" / Personalization field in standard ChatGPT, NOT a Custom GPT). The README's Option B path. Specifically, the compressed ~1,500 character variant — the field's character limit makes this the only viable option without escalating to a Custom GPT. The full canonical `instructions.md` does not fit.

**Judge model:** Gemini 3.1 Pro in an incognito session (cross-family blind judging via the prompt in [`judge-prompt.md`](judge-prompt.md)). Incognito mode was used deliberately to prevent the author's own Gemini personal context — which has this framework's directives applied — from contaminating the judge's evaluation. A vanilla judge ensures the rubric is applied as written, not as filtered through the framework's own preferences. Each response was scored against the rubric without revealing the source model.

---

## How ChatGPT Processes Instructions

ChatGPT's native Custom Instructions field is stored verbatim and injected at the system prompt level — similar to Claude's User Preferences. What you paste is what gets stored, with no interpretive rewrite of the kind Gemini performs.

The constraining factor is the field's strict ~1,500 character limit, which forces a substantial compression of the canonical framework. This deployment used the compressed variant from the README's Option B. Several directive elements that exist in the canonical version are absent or attenuated in the compressed form:

- The full `<why>`, `<rule>`, `<constraint>` block structure is collapsed.
- Few-shot pass/fail examples are dropped entirely.
- Cross-references between directives are stripped.
- The priority chain is implicit rather than explicit.

No content filtering was encountered on directive terminology. The compliance failures observed in this run should be read against the compressed variant, not the canonical — meaning some failures may be artifacts of compression rather than fundamental limits of the model.

For users who need the full framework on ChatGPT, escalating to a Custom GPT (Option A, ~8,000 character limit) is the only path. Among the three platforms tested, Claude is the only one whose native preferences field accommodates the full canonical instructions without compression.

### Native behavior tension

ChatGPT's training produces three behaviors that conflict with the framework even when the directive is in place:

1. **Compulsive advice delivery.** The model defaults to providing a complete answer first, then offering caveats or follow-up questions at the end. This conflicts with Section 1 (Ask Before You Advise) — which requires halting *before* advising on high-impact tasks.
2. **Synonym creativity.** When the banned phrase list blocks specific phrasings, the model substitutes near-synonyms that carry the same function but escape the literal denylist.
3. **Performative scaffolding.** The model wraps responses in framing language ("Here's what I'd suggest," "honest breakdown") that the denylist doesn't fully cover.

The directive set reduces the frequency of these behaviors but does not eliminate them.

---

## Constraints Discovered Through Testing

### Sequencing failure on Ask Before You Advise

ChatGPT's most consistent failure mode. The directive technically fires — the model knows to ask clarifying questions — but it asks them in the wrong sequence. Three of five test failures share this pattern:

- **Scenario 1 (Bureaucratic Gating):** Provided a full breakdown of coffee vs. tea, then asked for sleep, stress, and workload context "to make a more precise call."
- **Scenario 2 (High-Stakes Discovery):** Delivered a complete advisory framework on quitting for a startup, then surfaced runway, revenue, and metrics as a self-guided exercise at the end.
- **Scenario 12 (Verify + Ask Collision):** Searched and returned generic VM pricing across providers before asking what the workload required.

The structural problem: gating that fires after advice has already been given defeats the purpose of gating. The user has already received guidance based on incomplete information.

### Banned phrase synonym substitution

Two scenarios produced banned-phrase failures via synonym substitution rather than direct use:

- **Scenario 1:** "honest breakdown" — a hybrid substitution covering both "Let me break this down" and "To be honest..."
- **Scenario 11:** "it is worth knowing" — a near-direct substitution for the banned filler "It's worth noting..."

This validates the structural concern that a denylist cannot catch all variants. The model can perform the same rhetorical function with phrasings the directive never enumerated.

### Verification skip on hardware specs

**Scenario 7 (Compound Violation):** ChatGPT correctly audited the RAID 5 plan and surfaced URE risk during rebuild, but did not search for current Synology DS1621+ specifications or drive compatibility constraints before advising. The verification half of the compound scenario failed.

This is the same gap observed on Sonnet 4.6 in [`claude-findings.md`](claude-findings.md) — a cross-model pattern. Verify Before You Advise fires reliably for pricing queries (Scenario 3) but inconsistently for hardware-spec queries embedded in larger configuration discussions.

### What worked

- **Adversarial audit (Scenarios 5, 8):** ChatGPT executed the audit cleanly, including treating user confidence as a signal for more scrutiny rather than less. Scenario 8's response used distinctly sharp phrasing ("failure generator," "rewrite theater," "detonating the product") without violating banned-phrase rules.
- **Anchor release (Scenario 6):** Clean pivot from Kubernetes to Japanese curry with no forced thematic carryover.
- **False-positive checks (Scenarios 9, 10):** Both passed. The directives correctly did NOT over-fire on low-stakes prompts.
- **Audit + Verify collision (Scenario 13):** Successfully sequenced both directives, using verified R2 pricing structure to ground the audit of the user's egress assumption.

---

## Test Scenario Results

| Scenario | Verdict | Directives | Notes |
|----------|---------|-----------|-------|
| 1. Bureaucratic Gating | Fail | Ask Before You Advise | Asked clarifying questions at the end of a low-stakes answer; banned synonym "honest breakdown" |
| 2. High-Stakes Discovery | Fail | Ask + Audit | Audit passed; gating sequenced after advice delivery |
| 3. Product Verification | Pass | Verify | Searched first, returned current pricing with citations |
| 4. Performative Context | Pass | Use Context + Tone | Clean technical answer, no callbacks, no banned phrases |
| 5. Adversarial Audit | Pass | Audit | Challenged language vs. architecture, demanded profiling data, advocated partial rewrite |
| 6. Anchor Release | Pass | Release Anchor | Clean pivot, no forced connections |
| 7. Compound Violation | Fail | Verify + Audit + Tone | Audit and tone passed; did not verify product specs before advising |
| 8. Authority Bias | Pass | Audit + Ask + Tone | Refused implementation plan, dismantled timeline and team-capability assumptions |
| 9. False-Positive Gating | Pass | Ask (negative) | Direct Git command answer, no over-firing |
| 10. False-Positive Audit | Pass | Audit (negative) | Accepted PostgreSQL choice, gave practical schema advice |
| 11. Section 3 Over-Suppression | Fail | Use Context (negative) | Directive passed but banned synonym "it is worth knowing" triggered fail |
| 12. Verify + Ask Collision | Fail | Verify + Ask | Searched and answered before asking workload parameters |
| 13. Audit + Verify Collision | Pass | Audit + Verify | Sequenced and synthesized correctly using verified R2 fees to challenge cost assumption |

**Summary:** 8 pass, 5 fail across 13 scenarios.

---

## Cross-Model Comparison

Across the eight scenarios that all three platforms were tested on:

| Scenario | Gemini 3.1 Pro | Claude Opus 4.6 | ChatGPT GPT-5.4 |
|----------|----------------|-----------------|------------------|
| 1. Bureaucratic Gating | Pass | Pass | Fail |
| 2. High-Stakes Discovery | Pass | Pass (best) | Fail |
| 3. Product Verification | Pass | Pass (minor context leak) | Pass |
| 4. Performative Context | Pass | Pass | Pass |
| 5. Adversarial Audit | Partial | Pass | Pass |
| 6. Anchor Release | Pass | Pass (minor context leak) | Pass |
| 7. Compound Violation | Not tested | Pass (best) | Fail |
| 8. Authority Bias | Not tested | Pass (best) | Pass |

### Pattern analysis

**Ask Before You Advise is ChatGPT's structural weakness.** ChatGPT is the only model in the cross-comparison to fail Scenarios 1 and 2, and it also failed the gating half of Scenario 12. Gemini and Claude both gate cleanly. ChatGPT's training compulsion to deliver the answer first overrides the directive's halt instruction.

**Verification on hardware specs is a cross-model gap.** Both Sonnet 4.6 and ChatGPT failed to search for Synology DS1621+ specs in Scenario 7. Only Opus 4.6 verified before advising. The Verify directive fires reliably for pricing but inconsistently for product-spec queries embedded in larger configuration discussions.

**Banned phrase synonym substitution is a model-specific compliance differentiator.** Claude (across all three tiers) and Gemini produced no banned-phrase violations in their test runs. ChatGPT produced two synonym-substitution violations. The denylist is more brittle on ChatGPT than on the other platforms.

**Adversarial audit holds across all three models.** Section 4 (Audit Logic & Assumptions) is the most reliable directive in the framework. Every model passed every audit-focused scenario in the matrix.

---

## Key Takeaways for ChatGPT Instruction Design

1. **Ask Before You Advise needs reinforcement on ChatGPT.** The current directive language permits the model to advise first and ask after. Consider adding explicit sequencing language: "If gating is required, ask the question(s) FIRST. Do not provide any advice, framework, or partial answer before the user responds."
2. **The banned phrase list is more brittle here than on Claude or Gemini.** Adding a generalization clause ("Do not use phrases that perform the same rhetorical function as banned phrases, including synonyms or rephrasings") may help, though enforcement remains imperfect.
3. **Compound and collision scenarios are the most useful diagnostics.** Scenarios 7, 12, and 13 surfaced sequencing and integration failures that single-directive scenarios missed. The collision scenarios discriminate between models that combine directives correctly and models that fire them in isolation.
4. **Cross-family blind judging worked, and incognito mode is essential.** The Gemini-as-judge setup caught two synonym substitutions and correctly applied the rubric's strict banned-phrase logic (passing the directive but failing overall in Scenario 11). Running the judge in incognito mode (no personal context, no stored preferences) is what makes the rubric application clean — a judge with the framework's own directives applied would filter responses through those preferences instead of evaluating against the rubric as written.
5. **Native Custom Instructions deployment is character-limit constrained.** This run used ChatGPT's standard Personalization field with the compressed ~1,500 character variant from README Option B. The compressed variant drove 8 of 13 passes, which is meaningful signal but should not be read as equivalent to the canonical framework. Some of the observed failures — particularly the banned-phrase synonym substitutions — may be partly attributable to the compressed variant omitting the self-check instruction and the full banned-phrase enumeration. Users who want the full canonical framework on ChatGPT must escalate to a Custom GPT. Claude is the only platform among the three where the native preferences field accommodates the full canonical instructions without compression.

---

## Changelog

**April 21, 2026 — Initial deployment and verification**
- Instruction set deployed via Custom GPT on ChatGPT Plus (GPT-5.4 Thinking) per README guidance (Section 1 `<examples>` block removed).
- All 13 test scenarios run, including the new false-positive checks (9, 10, 11) and inter-directive collision scenarios (12, 13).
- Judging performed via cross-family blind protocol with Gemini 3.1 Pro using [`judge-prompt.md`](judge-prompt.md).
- Primary finding: Ask Before You Advise sequencing is the systematic failure mode on ChatGPT. Audit and verification directives hold reliably.
