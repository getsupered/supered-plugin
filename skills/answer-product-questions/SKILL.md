---
name: answer-product-questions
description: >
  Answer questions by grounding them in the right source — the official help center for how
  Supered itself works (concepts, setup, feature behaviour: rules, rulesets, process boards,
  action plans, bases, packages, guides, page triggers, etc.), and the customer's own knowledge
  base (their cards, guides, and action plans) for their processes, policies, and company-specific
  answers. Use when the user asks "how do I…", "how does X work", "what does Y mean", or anything
  that might be answered by their stored content. Search before deciding a topic is out of scope.
---

# Answer questions

Two different sources answer two different kinds of question — pick based on whose knowledge the
question is about:

- **`help_center_search`** — how the **Supered product** works: features, setup, configuration,
  "how does X work / how do I set up Y / what does Z mean". This is the canonical source for
  product how-to, written and maintained by the Supered product team, so it's more accurate and
  current than answering from memory.
- **`search_collectionable`** — the **customer's own knowledge** stored in their bases: their
  cards (content), guides, and action plans — i.e. their processes, policies, playbooks, and
  company-specific answers. Reach for this whenever the question is about *their* content rather
  than about Supered itself. You must opt into the types you want: set `include_content=true` for
  cards (most knowledge lookups), and add `include_guides=true` / `include_action_plans=true`
  when relevant — they all default to `false`, so an un-flagged call searches nothing.

**When it's ambiguous which one applies, search both** — the customer may have stored their own
answer to what looks like a product question, and vice versa. Don't stop after one source comes
back empty or thin; try the other before concluding there's no answer. Lead your reply with
whichever source actually answered, and prefer the customer's own content when both do.

**Search before you decline.** Customers store all kinds of content in their bases — including
things that look nothing like a "Supered" question. So a question sounding off-topic or "outside
your wheelhouse" is not a reason to decline up front: it's a reason to run `search_collectionable`
(`include_content=true`) first. Only if that genuinely returns nothing relevant should you say the
topic isn't covered. Never turn a question away as out of scope without having searched the
customer's knowledge base for it.

## How to use it

1. Call the tool with the question **rephrased into search terms**, not the full conversational
   sentence.
2. For `help_center_search`, skim the `snippet` / `short_description` fields. If you need the full
   body, call `help_center_card` with the most relevant `public_url_slug`.
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

If a search returns nothing, try the other source before giving up (see "search both" above). If
both come back empty, say so honestly rather than inventing an answer.
