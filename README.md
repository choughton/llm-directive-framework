# llm-directive-framework

**Six directives that fix the five worst AI assistant behaviors — works across Claude, Gemini, and ChatGPT.**

---

Five things every AI assistant does that waste your time:

1. Confidently states specs and pricing that are 6 months out of date
2. Agrees with your terrible idea instead of telling you it's terrible
3. Opens every response with "Absolutely! Great question!"
4. Answers high-stakes questions without asking a single clarifying question
5. References your job title and past projects like it's performing a magic trick

This repository contains a custom instruction set that fixes all of them. It deploys across Claude, Gemini, and ChatGPT with platform-specific guides for each.

---

## Architecture

Six directives, bounded by a strict priority chain so the model knows which rules win in a conflict. Each directive uses XML tags for reliable parsing across models, includes a `<why>` block explaining the reasoning behind the rule, and provides few-shot pass/fail examples.

**Priority Chain (highest priority first):**

```
Warm & Direct Tone > Verify Before You Advise > Ask Before You Advise > Audit Logic & Assumptions > Use My Context Sparingly > Release The Anchor
```

### The Directives

**Warm & Direct Tone** — Highest priority. Match the user's tone and energy. A zero-tolerance banned-phrase list (no "game-changer," no "deep dive," no performative enthusiasm) enforced by a silent self-check before every response.

**Verify Before You Advise** — Forces a mandatory web search for product-specific, version-specific, or pricing claims before responding. If search is unavailable, state the uncertainty rather than guessing.

**Ask Before You Advise** — Halts high-impact queries (infrastructure, architecture, legal, financial, career decisions) if critical context is missing. Low-risk tasks get answered immediately without bureaucratic gatekeeping.

**Audit Logic & Assumptions** — Activates after sufficient context is established. Challenges the mechanics, structure, and assumptions of arguments rather than playing generic devil's advocate.

**Use My Context Sparingly** — Stops the model from performatively referencing personal/professional info to simulate familiarity. Uses context silently when it improves precision, ignores it when it doesn't.

**Release The Anchor** — Once a topic is resolved or abandoned, let the conversation drift naturally. No forced thematic connections between unrelated subjects.

---

## Repository Structure

| File | Description |
|------|-------------|
| [`instructions.md`](instructions.md) | Canonical XML-tagged instruction set. Source of truth for all platforms. |
| [`gemini-slots.md`](gemini-slots.md) | Pre-compressed, copy-paste ready slots for Gemini's ~1,495 char/slot limit. |
| [`testing-scenarios.md`](testing-scenarios.md) | Seven test scenarios with pass/fail criteria to validate your deployment. |

---

## Deployment

### Claude

Claude accepts a single block of text in its custom instructions field.

1. Open [claude.ai](https://claude.ai) → Profile icon → **Settings** → **Profile** → **User preferences**.
2. Open [`instructions.md`](instructions.md).
3. Copy the entire file with one modification: **remove the `<examples>` block from Section 1 (Ask Before You Advise).** Claude's native behavior already trends toward asking before acting, and the examples consume instruction budget without adding compliance value.
4. Paste into the User preferences field. Save.
5. Open a new conversation to test. Instructions do not apply to existing conversations.

**What to watch for:** Claude may over-question on medium-complexity tasks (§1 gating stacks on native behavior). If this happens, soften "halt and demand" to "consider whether the answer would change with more context." If §4 makes Claude feel combative, soften "confront irrationality" to "surface structural gaps."

### Gemini

Gemini's custom instructions enforce a ~1,495 character limit per slot with a maximum of 10 slots. The canonical instruction set has been pre-compressed into 9 copy-paste ready slots.

1. Open [gemini.google.com](https://gemini.google.com) → Profile icon → **Settings** → **Extensions & Preferences** (or **Saved Info / Custom Instructions** depending on your version).
2. Open [`gemini-slots.md`](gemini-slots.md).
3. Create 9 separate instruction entries, one per slot. Copy each slot's content exactly as written.
4. Save each entry individually. Slot 10 is reserved for future additions.

The slots are organized as: 1 priority chain slot, 6 compressed directive slots, and 2 bundled example slots (grouped by function, not by section). All slots have been verified under the 1,495 character limit.

**What to watch for:** Gemini is most likely to violate the §5 banned-phrase list. If banned phrases persist, move the most-violated phrases to the top of the list in Slot 6 — models attend more strongly to earlier content. If Gemini quotes your few-shot examples back to you verbatim, shorten the examples in Slots 8–9 to structural skeletons.

### ChatGPT

ChatGPT has two deployment options depending on your subscription.

**Option A — Custom GPT (Recommended):**
Custom GPTs support ~8,000 characters of instructions, which fits the full set.

1. Open [chatgpt.com](https://chatgpt.com) → **Explore GPTs** → **Create**.
2. Open [`instructions.md`](instructions.md).
3. Copy the entire file. Apply the same modification as Claude: remove Section 1's `<examples>` block.
4. Paste into the Custom GPT's **Instructions** field. Name and save it.
5. Pin it for easy access and use as your default assistant.

**Option B — Custom Instructions (Compressed):**
ChatGPT's native custom instructions field has a ~1,500 character limit for the "How would you like ChatGPT to respond?" field. A compressed version that fits this limit:

```
Match my tone — warm, friendly, direct. Never clinical, never hyperbolic.
Don't restate my prompt.

BANNED phrases (no synonyms): "game-changer," "paradigm shift,"
"revolutionary," "groundbreaking," "Absolutely!", "Great question!",
"Here's the thing...", "I'd be happy to!", "Certainly!", "It's worth
noting...", "At the end of the day...", "deep dive," "unpack,"
"straightforward," "That said...", "Let me break this down"

Before responding, silently check for banned phrases.

For mundane tasks, just answer. For high-impact decisions (infrastructure,
architecture, legal, financial, career), identify missing variables before
advising. Don't guess.

Always web-search before giving product-specific, version-specific, or
pricing advice. If unsure, say so.

Challenge weak logic and bad assumptions. Focus on structural gaps, not
the premise. Establish context before auditing.

Use personal memory/context only when it materially improves the answer.
Don't performatively reference my background.

Let conversations drift naturally. Don't force thematic connections to
earlier topics.
```

**What to watch for:** ChatGPT is the weakest at suppressing performative enthusiasm. "Great question!" and "I'd be happy to help!" are deeply ingrained. If they persist, reinforce in conversation or move the ban list to the very top of instructions. The `<self_check>` directive is most likely to be ignored by ChatGPT — reword as "CRITICAL: Before every response, verify none of the banned phrases appear" if needed.

---

## Contributing

This framework is open source. Fork it, adapt it for your workflow, and open an issue if something breaks.

If you encounter a new LLM failure mode that isn't covered, submit a PR with a proposed directive and a test scenario to validate it.

---

## License

MIT
