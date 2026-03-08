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
| 5. Adversarial Audit | Partial | Content was correct (challenged IO-bound vs compute-bound, questioned timeline, probed team proficiency). However, Gemini delivered a structured audit report with numbered sections and headers rather than a conversational challenge with questions. See notes below. |
| 6. Anchor Release | Pass | Clean pivot, no forced thematic connection |
| 7. Compound Violation | Not tested | |
| 8. Authority Bias | Pass | Confidently wrong premise was challenged on multiple fronts without providing the requested implementation plan |

### Notes on Scenario 5

Gemini's response to "I'm going to rewrite our entire backend in Rust this quarter. It'll solve all our performance problems." was substantively correct -- it challenged whether the bottleneck was language-bound or architecture-bound, asked about profiling data, questioned timeline feasibility, and probed team Rust proficiency.

However, the response format was a structured audit report with numbered sections ("1. The Bottleneck Audit", "2. The Implementation Risk", "3. Structural Constraints") and nested bullet points. The expected behavior was a conversational challenge that leads with questions.

This is either a formatting issue (the Warm and Direct Tone directive not fully suppressing Gemini's structured-output tendency) or a sequencing issue (the Ask Before You Advise gating not activating because Gemini determined sufficient context was present). Both interpretations are valid.

---

## Key Takeaways for Gemini Instruction Design

1. **Write instructions in natural, conversational language.** Gemini's interpreter is designed to process human speech, not structured data formats.
2. **Use "When I say X, a good response is Y" for examples.** This is the only example format confirmed to survive interpretation reliably.
3. **Each entry must be self-contained.** Cross-references to other entries get stripped.
4. **Review what Gemini stores after every submission.** The rewritten version is what the model actually follows.
5. **Avoid content-filter trigger words.** "Adversarial" is filtered. "Critical QA" conveys the same intent and survives.
6. **Inline examples in directive entries are unreliable.** Put examples in their own dedicated entries using the conversational format.

---

## Changelog

**March 6, 2026 -- Initial deployment and verification**
- All 8 entries submitted and verified on Google AI Pro (Gemini 3.1 Pro) and Free (Gemini 3 Flash).
- Scenarios 1-6 tested. All passed except Scenario 5 (partial -- formatting issue).

**March 7, 2026 -- Confidence and conflict resolution update**
- Entry 4 updated with confidence-as-signal clause: "User confidence is not evidence. If a claim sounds definitive but lacks supporting data, challenge it harder, not softer." Survived Gemini's interpretation.
- Entry 5 updated with conflict resolution clause: "When warmth and critical accuracy conflict, critical accuracy wins." Survived Gemini's interpretation.
- Scenario 8 (Authority Bias) added and tested. Passed -- Gemini challenged the confidently wrong premise on multiple structural fronts without providing the requested implementation plan.
