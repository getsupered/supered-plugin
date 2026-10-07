---
name: build-packages
description: >
  Build Supered packages from bases, action plans, process rules, and HubSpot resources: add a base's
  cards and guides, an action plan template, or tagged rules, search a source HubSpot portal for workflows, properties,
  pipelines, lists, forms, reports, and more, add them with their dependencies, and publish the
  package for client accounts to install. Use when the user wants to "package up these workflows",
  "put this base in a package", "build a package from my portal", "copy this setup to clients", or
  asks what a package still depends on.
---

# Build packages

## Concepts

- **Package**: a set of HubSpot (and Supered) resources that teams with access install into their
  own portals. Resources added now land in the **draft**; installers only see them after
  `package_publish`. `package` shows the draft, each resource's `warnings`, and
  `unresolved_dependencies`.
- **Browser relay**: Supered can't read HubSpot on its own. HubSpot resources are read by the
  Supered Chrome extension inside the user's signed-in HubSpot tab, through a Supered tab that stays
  open. Every HubSpot tool (`package_builder_relay`, `hubspot_resource_search`,
  `package_hubspot_resources_add`) needs it.
- **Refs**: `hubspot_resource_search` returns resources as `ref` ids. Pass refs to
  `package_hubspot_resources_add`; never try to construct resource definitions yourself. Refs expire
  after 24 hours.
- **Links**: packages return `app_url` (full URL, login required) and `path` (relative, for in-app
  links). Link the package whenever you create, change, review, or publish it, using these values
  as-is; never build the URL yourself.
- **Dependencies**: a resource can use others (a workflow uses a list and a property). Each added item
  returns its `dependencies` as refs, flagged `standard_resource` (built into every portal) and
  `in_package`.

## Adding bases

Bases (Collections) don't need the browser relay.

1. Find the base with `collections`.
2. `package_collection_add` with the `package_id` and `collection_id`. The base's cards, folders,
   guides, and page triggers are included when the package is installed.
3. Leave the options at their defaults unless the user asks otherwise:
   - `install_into`: `installed_base` puts the content in the base the installer picks;
     `source_base` puts it in a base named after the source.
   - `mirror`: keeps installed copies in sync with later edits. It takes effect on reinstall, and
     turning it off later disables mirroring for every version, so confirm before enabling.
   - `include_action_plan_templates`: rarely wanted; copied templates can't be moved back.

Adding a base that is already in the draft updates its options, so pass the options you want kept.

## Adding action plans

Action plans don't need the browser relay either.

1. Find the template with `action_plans`. Only templates can be added; if the user means a plan that
   isn't one, offer `action_plan_convert_to_template` first.
2. `package_action_plan_add` with the `package_id` and `action_plan_id`.
3. Tell the user what installers get: they can assign the plan to their own users, progress is tracked
   from this (the partner) account, and no template is created in the installer's account.

## Adding process rules

Process rules are packaged by tag, and don't need the browser relay.

1. List the rules carrying the tag with `sync_engine_rules` (filter by `tags`) and confirm the list
   with the user. If the rules aren't tagged yet, tag them first.
2. `package_rule_tag_add` with the `package_id` and the `tag` name. Every rule carrying the tag is
   included, and the rules are looked up again at install, so rules tagged later come along too.
3. Tell the user what else happens:
   - cards, folders, and page triggers connected to those rules are included automatically;
   - CRM fields the rules use are **not** created by the install. They must already exist in the
     installer's CRM, or be added to the package as HubSpot resources.
4. `include_tags` keeps the rules' tags on the installed copies. Leave it off unless asked.

## Connecting the relay

Only HubSpot resources need this.

Call `package_builder_relay` first.

- If `connected` is false, give the user `relay_url` and ask them to open it in Chrome, keep that
  tab open, and have the source HubSpot portal open and signed in. Call `package_builder_relay`
  again once they have.
- When connected, `portals` lists the HubSpot portals open in their browser. Ask which one is the
  source when there's more than one. If the one they want is missing, ask them to open it.

## Adding HubSpot resources

1. **Pick the package.** List with `packages`, or create one with `package_create` (name plus a
   short description of what it sets up). Confirm with the user before creating.
2. **Find resources.** `hubspot_resource_types` lists searchable types (`available: false` means the
   plan doesn't include it). Types with `object_types` (Field, Field Group) need an
   `object_type_id`, e.g. `0-1` for Contact. Search with `hubspot_resource_search`, passing
   `package_id` so results show `in_package`. Show the matches and confirm before adding.
3. **Add them.** `package_hubspot_resources_add` with up to 25 refs from one portal. Report each
   item's `status`, `added` names, and any `warnings`.
4. **Resolve dependencies.** From every item's `dependencies`:
   - skip `standard_resource` and `in_package` ones;
   - add the rest with `package_hubspot_resources_add`, asking the user first when it isn't obvious
     the dependency belongs in the package (for example a shared list other teams maintain);
   - repeat with the dependencies those return until nothing new appears.
5. **Review.** Read `package`. Report warnings, and for each `unresolved_dependencies` entry say
   what it is and that it must already exist in the target portal or the install will fail or need
   manual setup.
6. **Publish only when asked.** `package_publish` makes the draft installable by every team with
   access, including auto-installs for client accounts. Summarize what's in the draft, link it with
   `app_url` so they can review it in Supered, and get a clear yes first.

## Long-running calls

Searches and adds wait up to a minute. If `status` is `pending` or `running`, call
`package_builder_request` with the returned `id` until it's `done` or `failed`. On `failed`, relay
`error`; most failures mean the relay tab closed or the HubSpot tab signed out.

## Editing

- `package_update` renames or rewrites descriptions immediately, without publishing.
- `package_resource_remove` takes a resource `id` from `package.draft_resources` and removes it from
  the next version.
