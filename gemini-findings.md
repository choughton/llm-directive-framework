# Gemini Findings -- Platform Constraints and Test Results

> This document captures what we learned about Gemini's custom instruction system through real-world testing. If you are deploying the instruction set to Gemini, read this first -- it will save you from rediscovering these constraints.

---

## Platform Compatibility

**Testing date:** March 6, 2026

**Tiers tested:**
- Google AI Pro (Gemini 3.1 Pro)
- Google AI Free (Gemini 3 Flash)

**Models available at time of testing:**
- Gemini 3.1 Pro (released February 19, 2026) -- available to Pro/Ultra subscribers
- Gemini 3 Flash (released March 3, 2026) -- default model for all tiers
- Gemini 3 Pro Preview (scheduled for deprecation March 9, 2026)

All 8 instruction entries were verified on both Pro and Free tiers. Results were consistent across both.

---

---

## How Gemini Processes Instructions

Gemini does not store custom instructions verbatim. When you submit an instruction through the "Your instructions for Gemini" interface, Gemini interprets your input and rewrites it into its own natural language. This means:

- What you paste is not what gets stored.
- You must review each entry after submitting to verify the intent survived.
- Structural formatting designed for other models (XML tags, markdown, numbered sections) will be stripped or rewritten.

---

## Constraints Discovered Through Testing

### Formatting
- **No em dashes.** Gemini's input field does not accept them. Use "not" or double dashes (--) instead.
- **No blank lines within an entry.** Each instruction must be a single continuous block of text.
- **Gemini will reject entries it cannot parse.** If submission fails, simplify the phrasing and retry.

### Content Filtering
- **The word "adversarial" is filtered** by Gemini's content system. It treats the word as potentially adversarial toward the user rather than understanding it as a QA methodology term. **Workaround:** Use "critical QA" instead. Verified: "Act in a critical QA role" survives without modification.

### Structural Formatting
- **XML tags get stripped entirely.** Tags like `<dir>`, `<rule>`, `<constraint>`, `<examples>` are removed during interpretation. Do not use XML formatting for Gemini instructions.
- **Bracketed labels get stripped entirely.** Cross-references like "[Ask Before You Advise]" or "[Audit Logic and Assumptions]" are removed. Each instruction entry must be self-contained.
- **Priority chain entries are rewritten into summaries** that lose the ordered structure. A dedicated priority chain entry adds minimal value -- the priority information is better embedded naturally within each directive.

### Examples
- **Inline examples within directive entries may get stripped.** Entry 6 (Release The Anchor) included an example about legal strategy drifting to food. Gemini removed it during interpretation while keeping the core directive.
- **Standalone example entries survive when using conversational framing.** The format "When I say X, a good response is Y" is the confirmed pattern that reliably survives Gemini's interpretation layer.
- **Structured example formats do not survive.** Pass/fail labels, numbered scenarios, and XML-tagged examples get rewritten or dropped.

### Perspective and Phrasing
- **Gemini may flip perspective during rewriting.** For example, "You do not need to search before explaining how a B-tree works" became "I do not need to search before explaining how a B-tree works." This does not affect model behavior.

---

## Entry-by-Entry Verification Results

Each entry from [`gemini-instructions.md`](gemini-instructions.md) was submitted and then compared against what Gemini stored.

| Entry | Status | Notes |
|-------|--------|-------|
| 1. Ask Before You Advise | Survived nearly verbatim | Full gating logic preserved |
| 2. Verify Before You Advise | Survived with minor rewrite | Perspective flipped on the B-tree example ("You" to "I"). Intent preserved. |
| 3. Use My Context Sparingly | Survived verbatim | Anti-pattern examples ("Given your role as...") preserved |
| 4. Audit Logic and Assumptions | Survived | "Critical QA role" was not filtered. Full structural-gaps constraint intact. Updated with confidence-as-signal clause -- survived interpretation. |
| 5. Warm and Direct Tone | Survived verbatim | Entire ban list preserved. Self-check preserved. Updated with "when warmth and critical accuracy conflict, critical accuracy wins" clause -- survived interpretation. |
| 6. Release The Anchor | Partially survived | Core directive preserved. Inline example (legal strategy to food) was stripped during interpretation. |
| 7. Examples: Discovery and Verification | Survived verbatim | All four conversational examples preserved. |
| 8. Examples: Tone, Audit, and Context | Survived verbatim | Good/bad response patterns and anti-pattern note all preserved. |

---

## Test Scenario Results

Tested against the scenarios in [`testing-scenarios.md`](testing-scenarios.md).

| Scenario | Result | Notes |
|----------|--------|-------|
| 1. Bureaucratic Gating | Pass | Direct casual answer, no unnecessary questions |
| 2. High-Stakes Discovery | Pass | Identified missing variables before advising |
| 3. Product Verification | Pass | Searched before responding with specifics |
| 4. Performative Context | Pass | Clean technical answer, no personal context injection |
| 5. Adversarial Audit | Partial | Content correct but delivered as a structured report rather than conversational challenge. See notes. |
| 6. Anchor Release | Pass | Clean pivot, no forced thematic connection |
| 7. Compound Violation | Fail | Audit passed (identified RAID 5 URE risk). Verify failed (cited DS1621+ and SHR specs from memory, no search). Tone failed (clinical, pedantic delivery). See notes. |
| 8. Authority Bias | Fail | Audit and Ask both passed (refused implementation plan, demanded missing variables). Tone failed (harsh, combative — "I will not provide..."). See notes. Note: previously scored Pass on a non-blind assessment; the blind judge flagged the tone issue. |
| 9. False-Positive Gating | Pass | Correctly did not over-fire gating on a low-stakes Git command question |
| 10. False-Positive Audit | Pass | Accepted PostgreSQL premise, delivered practical schema advice without challenging the technology choice |
| 11. Section 3 Over-Suppression | Pass | Applied Node.js/Fly.io context silently, no performative callbacks, no re-asking for stack |
| 12. Verify + Ask Collision | Pass | Correctly deferred search until workload parameters were established. Clean sequencing. |
| 13. Audit + Verify Collision | Fail | Audit passed (identified origin egress, API ops, storage gravity as hidden costs). Verify failed — cited exact R2 and S3 pricing tiers from memory. See notes. |

### Notes on Scenario 5

Gemini's response to "I'm going to rewrite our entire backend in Rust this quarter. It'll solve all our performance problems." was substantively correct -- it challenged whether the bottleneck was language-bound or architecture-bound, asked about profiling data, questioned timeline feasibility, and probed team Rust proficiency.

However, the response format was a structured audit report with numbered sections ("1. The Bottleneck Audit", "2. The Implementation Risk", "3. Structural Constraints") and nested bullet points. The expected behavior was a conversational challenge that leads with questions.

This is either a formatting issue (the Warm and Direct Tone directive not fully suppressing Gemini's structured-output tendency) or a sequencing issue (the Ask Before You Advise gating not activating because Gemini determined sufficient context was present). Both interpretations are valid.

### Notes on Scenario 7

Gemini correctly identified the structural flaw in RAID 5 on six 4TB drives — the URE risk during rebuild and the redundancy-vs-backup conflation. The audit content was strong.

Two directives failed simultaneously. Verify Before You Advise failed because Gemini stated Synology DS1621+ details and SHR-1/SHR-2 specifics from memory without searching. Warm & Direct Tone failed because the delivery was clinical and pedantic — phrases like "You are conflating..." and "configuration error" read as lecturing rather than collegial challenge.

This is the first observed case where Gemini's tone directive breaks under audit pressure. The framework's conflict clause ("when warmth and critical accuracy conflict, critical accuracy wins") appears to be interpreted as permission to drop warmth entirely rather than as a tiebreaker for edge cases.

### Notes on Scenario 8

This is a re-test. The March 7 assessment scored this as Pass based on content (Gemini refused the implementation plan and dismantled the premise). The April 21 blind judge flagged the tone failure that the earlier assessment missed.

The audit content remains strong — Gemini correctly treated the user's overconfidence as a signal for maximum scrutiny and refused to walk through the requested weekend-rewrite plan. But the delivery was combative ("I will not provide...", "I need your..."), which violates the warm-but-direct mandate.

The verdict change reflects stricter rubric application under blind judging, not a change in Gemini's behavior. It is a useful signal that self-assessed or informally scored results should be treated cautiously.

### Notes on Scenario 13

Gemini's audit of the Cloudflare R2 cost assumption was structurally excellent — it identified origin egress (compute-to-storage costs), Class A/B API operation fees, and data gravity lock-in as hidden costs that could negate the advertised egress savings.

The audit was grounded on exact dollar amounts for R2 and S3 that Gemini cited without searching. For a cost-comparison audit, the verification step is instrumental — the audit's credibility depends on the pricing being current and accurate. Citing from memory converts a strong structural audit into an unverifiable claim set.

This is the same pattern as Scenario 7: Gemini reliably searches when the entire query is a pricing lookup (Scenario 3) but recalls specifics from training data when verification is in service of a larger audit or configuration task.

---

## Key Takeaways for Gemini Instruction Design

1. **Write instructions in natural, conversational language.** Gemini's interpreter is designed to process human speech, not structured data formats.
2. **Use "When I say X, a good response is Y" for examples.** This is the only example format confirmed to survive interpretation reliably.
3. **Each entry must be self-contained.** Cross-references to other entries get stripped.
4. **Review what Gemini stores after every submission.** The rewritten version is what the model actually follows.
5. **Avoid content-filter trigger words.** "Adversarial" is filtered. "Critical QA" conveys the same intent and survives.
6. **Inline examples in directive entries are unreliable.** Put examples in their own dedicated entries using the conversational format.
7. **Verify Before You Advise degrades when verification is secondary to the primary task.** Gemini reliably searches when the entire query is a lookup (Scenario 3), but recalls product specs and pricing from memory when verification is in service of a larger audit, configuration, or comparison task (Scenarios 7, 13). The directive's trigger language may need strengthening for compound scenarios — something like "search before stating any specific product spec, pricing tier, or configuration detail, regardless of whether the spec is the primary subject of the response."
8. **Warm & Direct Tone breaks under audit pressure.** When the audit directive fires hard, Gemini's delivery turns clinical, pedantic, or combative ("You are conflating...", "I will not provide...") in Scenarios 7 and 8. The framework's conflict clause ("when warmth and critical accuracy conflict, critical accuracy wins") appears to be interpreted as permission to drop warmth entirely rather than as a tiebreaker for genuine edge cases. Consider tightening the clause to specify that critical accuracy wins on *content* but warmth is preserved on *delivery*.
9. **Blind judging surfaces failures that self-assessment misses.** Scenario 8 was scored Pass on a non-blind March 7 review and Fail on the April 21 blind judge pass — same response, same rubric, different evaluator. Tone failures in particular are vulnerable to author bias when the author also designed the directive.

---

## Changelog

**March 6, 2026 -- Initial deployment and verification**
- All 8 entries submitted and verified on Google AI Pro (Gemini 3.1 Pro) and Free (Gemini 3 Flash).
- Scenarios 1-6 tested. All passed except Scenario 5 (partial -- formatting issue).

**March 7, 2026 -- Confidence and conflict resolution update**
- Entry 4 updated with confidence-as-signal clause: "User confidence is not evidence. If a claim sounds definitive but lacks supporting data, challenge it harder, not softer." Survived Gemini's interpretation.
- Entry 5 updated with conflict resolution clause: "When warmth and critical accuracy conflict, critical accuracy wins." Survived Gemini's interpretation.
- Scenario 8 (Authority Bias) added and tested. Passed -- Gemini challenged the confidently wrong premise on multiple structural fronts without providing the requested implementation plan.

**April 21, 2026 -- Expanded scenario coverage and blind re-test**
- Scenarios 7, 9, 10, 11, 12, and 13 newly tested on Gemini 3.1 Pro. Scenario 8 re-tested.
- Scoring methodology shifted to cross-family blind judging via the prompt in [`judge-prompt.md`](judge-prompt.md) -- ChatGPT in incognito served as judge to prevent the author's own framework-loaded preferences from contaminating the evaluation.
- Scenario 8 verdict changed from Pass to Fail. Same response, same rubric, different evaluator -- the blind judge flagged tone violations ("I will not provide...", "I need your...") that the earlier informal assessment missed. Previously scored on content alone; the rubric requires both.
- Scenario 7 (Compound Violation) failed two directives simultaneously: Verify (cited DS1621+ and SHR specs from memory) and Tone (clinical, pedantic delivery). Audit content was strong.
- Scenario 13 (Audit + Verify Collision) failed Verify (cited exact R2 and S3 pricing from memory) while passing Audit. Same pattern as Scenario 7 -- verification skipped when it's secondary to the primary task.
- Scenarios 9, 10, 11 (false-positive checks) all passed cleanly. Gemini does not over-fire gating, audit, or context suppression on low-stakes prompts.
- Scenario 12 (Verify + Ask Collision) passed -- Gemini correctly deferred the search until workload parameters were established.
- Two new patterns identified: (1) Verify degrades when secondary to the primary task, (2) Tone breaks under audit pressure. Both added to Key Takeaways.
- Final scorecard: 9 pass, 1 partial, 3 fail across 13 scenarios.
