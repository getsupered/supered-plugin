---
name: build-content-and-guides
description: >
  Create and organize Supered knowledge content — Collections (Bases), content cards, folders,
  in-app Guides, and Page Triggers that surface cards/guides inside HubSpot or Salesforce. Use
  when the user wants to "create a card", "add content", "make a base/collection", "build a
  guide", or "show a card on a page / set up a trigger".
---

# Build content & guides

## Concepts

- **Collection** (aka Base) — the container for cards, folders, and guides. Create with
  `collection_create`; rename/archive/toggle visibility with `collection_update`.
- **Content (card)** — a markdown knowledge item inside a collection. `content_create` /
  `content_update`.
- **Content folder** — hierarchy within a collection. `content_folder_create` /
  `content_folder_update`.
- **Guide** — an interactive walkthrough. Read with `guide`; can be embedded in a card.
- **Page trigger** — surfaces a card/guide when a URL/title matches inside an external app.
  `page_trigger_create` / `page_trigger_update`.

## Workflow

1. Confirm the target collection. If none fits, create one with `collection_create` (name +
   description) and confirm before adding cards.
2. Create cards with `content_create`, passing `collection_id`, `title`, and a `markdown` body.
3. To change an existing card, **fetch its `markdownBody`, edit the string, and send the full
   new body via `content_update`** — do not create a duplicate. `markdown` replaces the whole
   body (no partial patch).

## Card markdown notes

The `markdown` field supports a rich but specific subset. Useful specifics:
- **Video embeds:** put a supported provider URL (YouTube, Loom, Vimeo, Wistia, Vidyard,
  Google Drive, direct `.mp4`, etc.) alone on its own line, blank lines above and below. A
  `[label](url)` link stays an inline link and does **not** embed.
- **Guide embed:** `[Title](https://app.supered.io/guides/<guideId>) <!-- embed -->`.
- **Images / audio / PDF:** image syntax `![caption](url)`; a `.pdf`/audio URL in image syntax
  becomes an inline PDF/audio block.
- **Table of contents:** an `<!-- toc -->` comment on its own line.
- Embedded raw HTML is captured as literal text (exceptions: `<u>` and `<font color/style>`).

## Page triggers

Use `page_trigger_create` to surface a card/guide in HubSpot/Salesforce based on URL/title
matching. Confirm the match paths (allow/exclude lists), the target card/guide, and any
user/group targeting before creating. Page triggers pair with the browser extension, so confirm
the user has it installed if they expect triggers to appear.

## Guardrails

- Confirm the collection and titles before creating; create one card at a time when onboarding.
- Prefer `*_update` over recreating. Card updates are last-write-wins — if a human may be
  editing the same card, re-fetch before writing.
