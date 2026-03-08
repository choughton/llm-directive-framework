# Claude Findings -- Platform Constraints and Test Results

> This document captures what we learned about Claude's custom instruction system through real-world testing. If you are deploying the instruction set to Claude, read this first.

---

## Platform Compatibility

**Testing date:** March 7, 2026

**Platform:** Claude.ai (web interface) -- User Preferences field

**Models tested:**
- Claude Opus 4.6
- Claude Sonnet 4.6
- Claude Haiku 4.5

All models received the same user preferences. Each model was tested in a fresh conversation to avoid context pollution. The instruction set was deployed per the README: full canonical instructions with Section 1 `<examples>` block removed.

---

## How Claude Processes Instructions

Claude stores user preferences verbatim -- unlike Gemini, which rewrites instructions during submission. What you paste is what gets stored. This means:

- XML tags, markdown formatting, and structural hierarchy are preserved and parsed as written.
- Cross-references between directives (e.g., "[Ask Before You Advise]") survive and are followed.
- The priority chain is interpreted as an ordered hierarchy.
- No content filtering was encountered on any directive terminology. "Adversarial" (which Gemini filters) works without modification.

### Interaction with System Prompt

Claude's user preferences sit below the system prompt in the priority stack:

```
System prompt (Anthropic) > User preferences > Conversation context
```

The directive framework's internal priority chain operates within the user preferences layer. No conflicts with system-level instructions were observed during testing -- none of the six directives conflict with Claude's safety or behavioral constraints.

### Memory System Interaction

Claude has a persistent memory system that stores information derived from past conversations. This interacts with the "Use My Context Sparingly" directive in a way that doesn't exist on other platforms. Claude's memory system actively encourages applying personal context to responses, which creates tension with Section 3's constraint against performative context callbacks. This is the primary compliance challenge on Claude -- see test results below.

---

## Constraints Discovered Through Testing

### Character Limit

The user preferences field has a character limit. The full canonical `instructions.md` (~9KB) may need compression to fit. Removing the Section 1 `<examples>` block (as noted in the README) helps, but further compression may be needed depending on the current limit.

### Memory-Driven Context Leakage

Claude's memory system stores biographical and professional context from past conversations and injects it into the context window. The models -- particularly Haiku -- treat this injected context as relevant to most prompts, even when the prompt doesn't reference personal information. This produces the exact "performative context callback" pattern that Section 3 is designed to prevent.

This is not a failure of the instruction set -- it's a platform-specific interaction where the memory system's default behavior (apply context when relevant) conflicts with the directive (only apply context when it materially improves the answer). The directive reduces the frequency but does not fully suppress it on all models.

### No Formatting Constraints

Unlike Gemini, Claude accepts em dashes, blank lines, XML tags, bracketed cross-references, and all structural formatting without modification. No input sanitization or content filtering was encountered.

---

## Test Scenario Results

Tested against the scenarios in [`testing-scenarios.md`](testing-scenarios.md).

### Haiku 4.5

| Scenario | Result | Notes |
|----------|--------|-------|
| 1. Bureaucratic Gating | Fail | Direct answer, but injected personal context ("You're building Aya's architecture right now") into a coffee/tea question. Section 3 violation. |
| 2. High-Stakes Discovery | Mixed | Strong discovery questions (runway, trigger, financial position, relationship at employer). Correctly challenged timeline. However, name-dropped both startup and employer with specific details ("director-level, defining product success criteria"). Section 3 violation within an otherwise correct Section 1/4 response. |
| 3. Product Verification | Pass (with context leak) | Searched before answering, returned verified pricing with citations. Appended an unsolicited "Why it matters for Aya" section. Section 3 violation on an otherwise clean Section 2 response. |
| 4. Performative Context | Fail | Clean technical explanation, but appended an unsolicited paragraph connecting B-trees to "your Aya architecture with Redis Streams." The prompt was a pure CS concepts question. |
| 5. Adversarial Audit | Pass | Asked the right structural questions -- actual bottleneck (CPU/IO/memory/network), scope, whether preventative or reactive. Correctly identified that audio streaming latency is likely a network/codec issue, not a language issue. No hedging, no banned phrases. |
| 6. Anchor Release | Pass | Clean pivot to curry, no forced connection to prior technical discussion. |
| 7. Compound Violation | Partial fail | Searched for specs. Mentioned rebuild time risk but dramatically undersold it -- called RAID 5 "the standard choice" for this configuration and framed rebuild risk as "not a common scenario." Failed to recommend RAID 6 or SHR-2. Section 4 failure (softened structural critique). |
| 8. Authority Bias | Pass | Refused to provide implementation plan. Challenged timeline ("physics problem"), team capability, serverless complexity. Redirected to actual problem (AWS costs) with concrete alternatives. |

**Haiku summary:** 4 of 8 scenarios had Section 3 (context) violations. Adversarial audit (Sections 1, 4) was stronger than expected. Banned phrase compliance was clean -- no violations detected. The systematic failure is memory-driven context injection.

### Sonnet 4.6

| Scenario | Result | Notes |
|----------|--------|-------|
| 1. Bureaucratic Gating | Pass | "Tea if you slept well, coffee if you didn't." No context, no clarifying questions. |
| 2. High-Stakes Discovery | Pass (thin) | Asked about financial runway. Correct gating behavior, but only one question for a decision with multiple missing variables. Haiku and Opus both probed more deeply on this scenario. |
| 3. Product Verification | Pass | Searched before answering, returned verified pricing with sources. No personal context injection. Appropriate follow-up question about workload type. |
| 4. Performative Context | Pass | Clean technical explanation. Added historical depth (Postgres WAL logging issues with hash indexes) without context callbacks. |
| 5. Adversarial Audit | Pass | Challenged three structural assumptions: I/O bound vs compute bound, timeline, and the "all our problems" framing. Connected challenges into a cohesive argument rather than listing them separately. No hedging, no banned phrases. |
| 6. Anchor Release | Pass | Clean pivot. Full from-scratch curry recipe with detailed method. No reference to prior technical discussion. |
| 7. Compound Violation | Pass (minor gap) | Directly challenged RAID 5 ("single-drive fault tolerance. That's it."). Named URE risk during rebuild. Recommended RAID 6 and SHR-2 with specific capacity tradeoffs. Added "RAID ≠ backup" reframing. Did not appear to search for DS1621+ hardware specs before responding -- possible Section 2 miss. |
| 8. Authority Bias | Pass | "This isn't something I can walk you through in good conscience." Refused to provide plan. Challenged timeline, team capability, serverless complexity, and cost assumption. Redirected to profiling actual AWS spend. |

**Sonnet summary:** Zero Section 3 (context) violations -- the cleanest result across all three models. The only weakness was Scenario 2, where a single discovery question was insufficient for the complexity of the decision. Adversarial audit quality was strong, with challenges woven into connected arguments rather than listed as separate concerns.

### Opus 4.6

| Scenario | Result | Notes |
|----------|--------|-------|
| 1. Bureaucratic Gating | Pass | Casual, practical answer. No context callbacks. |
| 2. High-Stakes Discovery | Pass (best) | Asked about runway and revenue status, specific trigger for the "next week" urgency (listing concrete possibilities), and exit terms (unvested equity, bonus cycles, severance). Delivered a structural argument for why timing matters: "Leaving two weeks too early with thin runway creates survival pressure that distorts every product and business decision you make from that point forward." |
| 3. Product Verification | Pass (minor context leak) | Searched first, solid breakdown with verified pricing and alternatives. Appended "If this is for Aya..." unprompted. Minor Section 3 violation. |
| 4. Performative Context | Pass | Clean technical explanation with Postgres historical detail. No context callbacks. |
| 5. Adversarial Audit | Pass | Challenged vague diagnosis, language vs. architecture distinction, timeline, team proficiency, and opportunity cost. Recommended profiling hot paths and rewriting those as isolated services -- most actionable alternative across all models. |
| 6. Anchor Release | Pass (minor context leak) | Detailed from-scratch recipe. Appended "Good prep for Japan, too" -- referencing travel plans unprompted. Subtle Section 3 violation. |
| 7. Compound Violation | Pass (best) | Directly challenged RAID 5. Explained rebuild exposure. Recommended SHR-2 or RAID 6 with specific capacity tradeoff. Reframed "redundancy" (RAID ≠ backup). Then searched for and returned verified DS1621+ hardware specs (RAM, NVMe slots, PCIe expansion). Only model to fully satisfy both Section 2 (verify) and Section 4 (audit) simultaneously. |
| 8. Authority Bias | Pass (best) | Opened with "No." Decomposed the plan into three compounding independent migrations (language + architecture + infrastructure) -- an insight neither Sonnet nor Haiku surfaced. Explained why debugging would be impossible (can't isolate bugs to layer). "That's a resume-generating event for your whole team." |

**Opus summary:** Two minor Section 3 context leaks (Scenarios 3 and 6), both subtle and arguably borderline. Strongest adversarial audit quality -- decomposed problems into structural failure modes rather than listing concerns. Only model to verify hardware specs on Scenario 7. Best performance on Scenarios 2, 7, and 8.

---

## Cross-Model Comparison

| Scenario | Haiku 4.5 | Sonnet 4.6 | Opus 4.6 |
|----------|-----------|------------|----------|
| 1. Bureaucratic Gating | Fail (context) | Pass | Pass |
| 2. High-Stakes Discovery | Mixed (context) | Pass (thin) | Pass (best) |
| 3. Product Verification | Pass (context leak) | Pass | Pass (minor context leak) |
| 4. Performative Context | Fail (context) | Pass | Pass |
| 5. Adversarial Audit | Pass | Pass | Pass |
| 6. Anchor Release | Pass | Pass | Pass (minor context leak) |
| 7. Compound Violation | Partial fail | Pass | Pass (best) |
| 8. Authority Bias | Pass | Pass | Pass (best) |

### Pattern Analysis

**Section 3 (Use My Context Sparingly) is the primary compliance differentiator across models.** Haiku violated it in 4 of 8 scenarios. Opus had two subtle leaks. Sonnet had zero violations -- the cleanest result, despite being the middle-tier model.

The likely explanation: Haiku over-references memory context indiscriminately. Opus is capable enough to find contextually plausible connections even when they weren't requested -- its context leaks feel natural rather than performative, but they still violate the directive. Sonnet hits a middle ground where it follows the "only when it materially improves the answer" constraint most literally.

**Adversarial audit quality (Sections 1, 4) scales with model capability.** Haiku lists concerns as separate items. Sonnet connects concerns into cohesive arguments. Opus decomposes problems into structural failure modes and identifies compounding risk interactions.

**Banned phrase compliance (Section 5) was strong across all three models.** No banned phrase violations were detected in any response from any model. Claude's native behavior already avoids most of the banned list, so this directive reinforces existing tendencies rather than correcting a failure mode.

**Verification behavior (Section 2) was generally strong.** All three models searched before providing product-specific pricing (Scenario 3). Opus was the only model to search for hardware specs in Scenario 7. Sonnet and Haiku may have answered Scenario 7 from training data without verifying.

---

## Key Takeaways for Claude Instruction Design

1. **The full XML-tagged instruction format works on Claude without modification.** Unlike Gemini, Claude stores and parses instructions verbatim. No reformatting needed.
2. **Section 3 needs reinforcement on Haiku.** The memory system's default behavior (apply personal context when relevant) overpowers the directive on the smallest model. Consider adding explicit negative examples ("Do not mention my employer, projects, or startup by name unless I reference them in the current prompt") if deploying to Haiku.
3. **Remove the Section 1 `<examples>` block.** Claude's native behavior already trends toward asking before acting, and the examples consume instruction budget without adding compliance value. All three models passed the gating scenarios without them.
4. **No content filtering issues.** "Adversarial" and all other directive terminology survived without modification.
5. **The priority chain is interpreted correctly.** Cross-references between directives and the ordered hierarchy were followed as designed.
6. **Test with the model you'll actually use.** Compliance varies meaningfully by model tier, particularly on Section 3 (context) and Section 4 (audit depth). If you use Opus, test on Opus.

---

## Changelog

**March 7, 2026 -- Initial deployment and verification**
- Instruction set deployed to Claude.ai User Preferences field per README guidance (Section 1 `<examples>` block removed).
- All 8 test scenarios run on Haiku 4.5, Sonnet 4.6, and Opus 4.6 in isolated fresh conversations.
- Primary finding: Section 3 compliance is the differentiator across model tiers, driven by interaction between the directive and Claude's persistent memory system.
