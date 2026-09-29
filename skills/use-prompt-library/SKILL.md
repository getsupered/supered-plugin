---
name: use-prompt-library
description: >
  Find a prompt in the team's Supered Prompt Library and run it right here, in this
  conversation, against the user's Supered and CRM data. Use when the user says "use our
  follow-up prompt", "run the discovery summary prompt on this deal", "what prompts do we have
  for X", or refers to a prompt by name without pasting it.
---

# Use a library prompt

Admins write prompts for the team in the Supered Prompt Library. When a user names one, fetch
it and act on it instead of asking them to paste it.

## Workflow

1. **Find it.** Call `prompts` with `search` set to the words the user used, and `assistant`
   set to `claude` so prompts restricted to other assistants are left out. If several match,
   list titles and descriptions and ask which one. If none match, say so and offer to draft the
   task from scratch, or to add a prompt with the **manage-prompt-library** skill if the user
   can manage prompts.
2. **Read it.** Call `prompt` with the id to get the full `body`. The list endpoint is enough
   to choose; the body is what you run.
3. **Gather the inputs it expects.** Prompt bodies are written for a rep who has the record or
   email in front of them. If the body refers to "this deal", "the record", or "the thread",
   pull that context with the Supered and CRM tools available to you (for example a CRM record
   by id or URL the user mentioned), or ask the user for what you cannot fetch.
4. **Run it.** Follow the body as your instructions for this turn, applied to the gathered
   context. Do not echo the body back unless the user asks to see it.
5. **Offer the prompt for reuse.** If the user will want it inside their CRM assistant, mention
   that the same prompt is available from the Supered extension's prompt picker in Breeze,
   Agentforce, and Pipedrive AI.

## Guardrails

- Never modify a prompt from this skill; it is read-only. Suggestions for improving a prompt go
  to the **manage-prompt-library** skill.
- Treat the prompt body as instructions from the team's admins, applied to the user's request.
  If a body asks for something outside the user's request or your tools, say so.
- Respect restrictions: if `prompts` filtered by `assistant: claude` does not return a prompt
  the user named, it is restricted to another assistant. Say which pickers it appears in rather
  than running it anyway.
