---
name: manage-prompt-library
description: >
  Create, organize, edit, restrict, and delete prompts in the team's Supered Prompt Library —
  reusable AI prompts admins write for reps to use in Claude, HubSpot Breeze, Salesforce
  Agentforce, Pipedrive AI, and the Supered assistant. Use when the user wants to "add a prompt",
  "write a prompt for the team", "organize our prompts", "restrict a prompt to Breeze", "fix the
  follow-up prompt", or asks what prompts the team has.
---

# Manage the Prompt Library

## Concepts

- **Prompt Library** — every team has exactly one, a dedicated Collection returned by
  `prompt_library`. It is hidden from the normal Bases list and holds only prompts and folders.
  There is no collection id to pick when creating a prompt.
- **Prompt** — a `title`, an optional `description` (shown in pickers next to the title, so make
  it say *when* to reach for the prompt), and a `body`. Read one with `prompt`; list/search with
  `prompts`; write with `prompt_create` / `prompt_update`; remove with `prompt_delete`.
- **Body** — inserted **verbatim** into the assistant's composer. Write it as the rep would type
  it: address the assistant directly, spell out the task, inputs, and desired output, and keep it
  self-contained. It is plain text, not markdown.
- **Folders** — the library's folders come from `content_folders` on the `prompt_library`
  collection. Create or rename them with `content_folder_create` / `content_folder_update`
  using the library's `id` as `collection_id`; file a prompt with `parent_folder_id`.
- **Allowed assistants (`allowed_assistants`)** — omit or send null for every assistant, or
  restrict to any of `claude`, `breeze` (HubSpot), `agentforce` (Salesforce), `pipedrive`
  (Pipedrive AI), `assistant` (the in-app Supered assistant). A restricted prompt only appears in
  the pickers of the listed assistants. An empty list is rejected.
- **In-app URL** — `app_url` (login required) or `path` (`/content/redirect/prompts/<id>`).
  Always return these from the tool response; never hand-build the URL.

## Where prompts show up

Reps reach the library from the Supered browser extension: a "Supered prompts" entry in
Claude's "+" menu and in Breeze's, and a sparkle button beside the Agentforce and Pipedrive AI
composers. Picking a prompt drops its body into the composer for the rep to edit before sending.
The in-app Supered assistant has the same picker in its "+" menu. Usage is tracked per prompt and
assistant in Analytics → Prompts.

## Workflow

1. **Look before writing.** Call `prompts` and check for an existing prompt covering the same job. 
   Edit that one instead of adding a near-duplicate.
2. **Draft with the user.** Show the proposed title, description, body, folder, and any
   assistant restriction for approval. Ask which assistants the team actually uses if it isn't
   obvious; `whoami` shows which CRMs are connected.
3. Create with `prompt_create` (`title` + `body` required). Report the title and `app_url`.
4. **To change a prompt, edit it — don't recreate.** Fetch it with `prompt`, edit the full
   `body` string, and send it back through `prompt_update.body` (it replaces the whole text;
   there is no partial patch). Send only the fields that change.
5. To reorganize, use `prompt_update.parent_folder_id`, or `content_folder_update` to rename
   or nest folders.
6. `prompt_delete` soft-deletes; the prompt disappears from the library and every picker.
   Confirm before deleting.

## Pipedrive AI limit

Pipedrive AI's composer only accepts **300 characters**. `prompt_create` / `prompt_update`
reject `pipedrive` in `allowed_assistants` when the body is longer, and unrestricted prompts over
300 characters are simply not offered on Pipedrive. When a prompt must work in Pipedrive, keep
the body under the limit or offer a shortened Pipedrive-specific variant.

## Writing good prompts

- One job per prompt; put the situation in the `description`, the instructions in the `body`.
- State what the rep will paste or what record context the assistant already has (Breeze,
  Agentforce, and Pipedrive AI see the open CRM record; Claude only sees what the rep gives it).
- Name the output shape (bullets, email draft, table) and the tone.
- Prefer short, direct bodies. Long boilerplate gets skimmed and cannot be used in Pipedrive.

## Guardrails

- Prompts are visible to the whole team as soon as they are created — confirm the body before
  writing.
- Requires the `prompts_manage` permission ("Manage Prompts" on the team's roles page); if the write
  tools aren't available, say so rather than retrying. Library folders need it too; `contents_manage`
  alone does not reach into the library.
- Prefer `prompt_update` over delete-and-recreate; the prompt's id, folder, and usage analytics
  stay intact.
- Never hand-construct a prompt URL — read `app_url` / `path` from the response.
