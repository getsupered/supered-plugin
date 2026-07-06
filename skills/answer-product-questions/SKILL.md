---
name: answer-product-questions
description: >
  Answer questions about how Supered itself works — what a concept means, how to set something
  up, how a feature behaves (rules, rulesets, process boards, action plans, bases, packages,
  guides, page triggers, etc.). Use when the user asks "how do I…", "how does X work", or "what
  does Y mean" about Supered. Grounds answers in the official help center.
---

# Answer product questions

`help_center_search` is the canonical source for how Supered works — it's written and maintained
by the Supered product team, so it's more accurate and current than answering from memory. Reach
for it before answering "how does X work / how do I set up Y / what does Z mean" questions.

## How to use it

1. Call `help_center_search` with the question **rephrased into search terms**, not the full
   conversational sentence.
2. Skim the `snippet` / `short_description` fields. If you need the full body, call
   `help_center_card` with the most relevant `public_url_slug`.
3. Write a **tight** summary — usually 2-4 sentences or 3-5 bullets. Lead with the answer, name
   the 2-3 key steps or concepts, and stop; the article has the full detail. Skip framing like
   "here's how" / "before you start" unless the user asked for a step-by-step.
4. End with a link formatted `[Card title](public_url)`. If `public_url` is null, answer from the
   body without a link — **never construct help-center URLs yourself.**

Human-readable docs also live at https://help.supered.io.

## Offer to do the work when the ask implies action

Most help questions carry an action verb ("how do I build/set up/add…"). When the action is
something you can actually do with your tools (create rules, rulesets, process boards, composite
boards, content, action-plan instances), close the summary with a one-line offer:

- "How do I build a process board?" → summary + "Want me to build one? Tell me the CRM object
  and which rules to include."
- "How do I create a hygiene ruleset?" → summary + "Happy to draft one — what pipeline should it
  target?"

Don't offer for purely informational asks ("what does X mean?", "difference between Y and Z?") or
for actions outside your tools (managing users, billing, integration setup). When in doubt,
default to offering — the user can decline.

If `help_center_search` returns nothing, say so honestly rather than inventing an answer.
