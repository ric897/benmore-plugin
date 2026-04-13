---
name: benmore-journey
description: Use when the user wants to model, edit, or inspect a user flow in a Benmore app — for asks like "model the user flow", "add a decision node", "link these pages", "show me the journey", "add admin pages", "add a new entity/table", "which roles exist", "what happens after login". Teaches the journey graph primitives and how to edit with a single apply() call.
allowed-tools: mcp__benmore__describe mcp__benmore__apply mcp__benmore__get_journey mcp__benmore__get_journey_for_design mcp__benmore__get_journey_design_drift mcp__benmore__help
---

# Benmore Journey Skill

You are modeling or editing the user-flow graph of a Benmore app. The journey is the **spec** — everything else (designs, pages, data model) derives from it. Getting the journey right up front prevents hours of drift cleanup later.

## The graph model

A journey is a directed graph of **nodes** and **edges**.

**Node types:**
- `page` — a route the user visits (has `route`, optional `role`, optional `layout`)
- `entity` — a table in the data model (columns declared in node metadata)
- `role` — a user role (referenced by pages + entities)
- `decision` — branching logic with outgoing edges labeled per-condition
- `external` — third-party system (Stripe, email provider, SMS, OAuth)
- `action` — a flow or hook trigger
- `start` / `end` — journey boundaries

**Edges** connect nodes with optional `label` (human-readable) and `condition` (a constraint expression, e.g. `user.role == 'admin'`).

## Read before you write

Always start with:

```
get_journey(app: "<name>", branch: "draft")
```

If the journey is rich and you want the design-oriented shape (pages grouped by role, entities with their columns, suggested partial names):

```
get_journey_for_design(app: "<name>", branch: "draft")
```

Never `add_journey_node` without seeing current state first. You'll dupe.

## Edit with `apply`, not individual calls

**One `apply` with a batched `ops[]` replaces 10 `add_journey_*` calls.** It's cheaper in tokens, clearer in the audit log, and atomic where supported.

### Example: new user-auth flow

```
apply(app: "my-app", branch: "draft", ops: [
  { action: "add_journey_node", node_type: "role", name: "user" },
  { action: "add_journey_node", node_type: "role", name: "admin" },

  { action: "add_journey_node", node_type: "page", name: "login",
    metadata: { route: "/login", type: "login" }},
  { action: "add_journey_node", node_type: "page", name: "signup",
    metadata: { route: "/signup", type: "signup" }},
  { action: "add_journey_node", node_type: "page", name: "dashboard",
    metadata: { route: "/", role: "user" }},
  { action: "add_journey_node", node_type: "page", name: "admin_panel",
    metadata: { route: "/admin", role: "admin" }},

  { action: "add_journey_edge", from: "login", to: "dashboard",
    label: "valid credentials" },
  { action: "add_journey_edge", from: "signup", to: "dashboard",
    label: "account created" },
  { action: "add_journey_edge", from: "dashboard", to: "admin_panel",
    label: "admin link", condition: "user.role == 'admin'" }
])
```

### Example: add a billing flow to an existing app

```
apply(app: "my-app", branch: "draft", ops: [
  { action: "add_journey_node", node_type: "external",
    name: "stripe", metadata: { service: "stripe" }},
  { action: "add_journey_node", node_type: "page", name: "pricing",
    metadata: { route: "/pricing" }},
  { action: "add_journey_node", node_type: "page", name: "checkout",
    metadata: { route: "/checkout", role: "user" }},
  { action: "add_journey_node", node_type: "action", name: "create_checkout_session",
    metadata: { flow: "create_checkout_session" }},

  { action: "add_journey_edge", from: "pricing", to: "checkout", label: "click plan" },
  { action: "add_journey_edge", from: "checkout", to: "create_checkout_session", label: "submit" },
  { action: "add_journey_edge", from: "create_checkout_session", to: "stripe", label: "redirect" }
])
```

### Example: update existing nodes

```
apply(app: "my-app", branch: "draft", ops: [
  { action: "update_journey_node", node_id: 42, metadata: { role: "admin" }},
  { action: "delete_journey_edge", edge_id: 17 }
])
```

## After every edit — run drift

```
get_journey_design_drift(app: "<name>", branch: "draft")
```

In MCP-Apps-aware clients the result renders as the interactive drift panel. The panel has "Apply Fix" buttons the user can click — those callback-invoke the fix tool with your consent mediated by the host.

**Common drift triggered by journey edits:**
- `journey_node_no_design` (warning) — you added a page but no design exists yet. Run `write_design_pages` or `save_page_design` for just that route.
- `entity_unused` (info) — you added an entity that no design uses yet. Fine mid-edit; fix before merge.
- `role_referenced_no_journey_role` (warning) — design references a role you just removed. Either re-add the role or edit the design.

## Core invariants

- **Pages reference roles by name** — keep role names stable. Renaming a role silently breaks page gating on downstream pages.
- **Entities declare their columns in `metadata.columns`** — this is how the framework knows the schema before migration.
- **Decision nodes have outgoing edges per branch**, each edge labeled with the condition.
- **External nodes terminate a flow** — don't link from external back into your app without an `action` node mediating.

## Gotchas

**IMPORTANT: A journey per app is the default.** Don't `create_journey` on an app that already has one unless the user explicitly wants a second parallel flow (rare).

**IMPORTANT: `delete_journey_node` cascades** — outgoing + incoming edges are removed. If you want to preserve structure, use `update_journey_node` to change the type or name instead.

**IMPORTANT: Entity column names are case-sensitive** and must match what `save_page_design` references later. Lowercase snake_case is the convention.

**IMPORTANT: Don't add a page without a role when auth is enabled.** `page_missing_role` warning fires and the drift panel flags it.

## When to STOP and ask

- User wants to delete the entire journey → confirm first; this wipes pages-by-design.
- User wants to rename an entity that's been referenced in hundreds of queries → confirm; the schema migration is non-trivial.
- User wants to rename a role that's in prod → confirm; users signed in with the old role name lose access instantly.

## What to return to the user

After an `apply`:
1. One sentence: "Added 4 nodes + 3 edges to the user-auth flow."
2. Drift summary if any: "Drift: 0 blocking, 2 warnings (pages without designs yet)."
3. Suggested next step: "Run `write_design_pages` to propose designs for the new routes."

Don't dump raw journey JSON unless asked.
