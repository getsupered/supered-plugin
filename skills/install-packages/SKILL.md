---
name: install-packages
description: >
  Install a Supered package into the user's Supered account and a HubSpot portal: preview what it
  adds, install bases, rules, announcements, and action plans, then have the user confirm the
  HubSpot install and report each resource's result. Use when the user wants to "install this
  package", "set up the onboarding package in my portal", "test my package in the sandbox", or
  asks why part of an install failed.
---

# Install packages

## Concepts

- **Two stages.** The Supered side (bases, rules, announcements, action plans) installs on the
  server. HubSpot resources are created in the user's browser by the Supered extension, in a portal
  the user is signed in to.
- **You never write to HubSpot.** `package_hubspot_install` only creates a request. The user has to
  open its `confirm_url`, review the list, and click Install. Until then nothing reaches HubSpot.
- **Draft installs.** `draft: true` installs a package's unpublished changes. Only the team that
  owns the package can, which is how a partner tests a package before publishing.

## Workflow

1. **Find the package.** `packages_installable` lists public packages, the team's own, and ones
   assigned to it.
2. **Preview.** `package_install_plan` changes nothing. Show the user:
   - `supered_resources` and `hubspot_resources`, grouped by type;
   - every `warnings` entry;
   - `unresolved_dependencies`: these must already exist in the target portal, or those resources
     will fail.
3. **Install the Supered side.** If the plan says `needs_base`, ask which base (Collection) to
   install into; find it with `collections`. A base in `supered_resources` whose `warnings` say it
   has a set install location ignores that choice. When the user picks a base, warn them which
   bases will go elsewhere and where, before installing. Then `package_install`. Report
   `resource_counts`.
4. **Request the HubSpot install.** For each entry in `hubspot_installs`:
   - call `package_hubspot_install`. Pass `portal_name` with whatever the user called the target
     portal ("Test 3"). Only pass `portal_id` if they gave you the numeric id. Never ask them to look
     an id up: without one, they choose the portal when they confirm;
   - always show the user the full `confirm_url` as a link. In Claude on the web with the Supered
     extension, an install card appears under the link; otherwise they open the link;
   - tell them to have the target portal open in another Chrome tab, choose or check the portal,
     tick the box, and click Install, and to keep both tabs open while it runs.
5. **Report.** Poll `package_install_request` until `status` is `done` or `failed`. For each
   resource give the `result`:
   - `success`: created;
   - `existed`: the portal already had it, so nothing changed;
   - `error`: quote the message and explain the likely cause, usually a missing dependency or a
     feature the portal doesn't have;
   - `pending`: not attempted, because the install stopped first.
6. **Retry what's left.** Call `package_hubspot_install` again with the same `install_id`. Only
   resources not yet installed are attempted.

## Limits

- The confirmation link expires after 30 minutes and is tied to one user, one install, and one
  portal.
- There is no uninstall. Tell the user before installing into a portal they care about, and suggest
  a sandbox portal for testing.
