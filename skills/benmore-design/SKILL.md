---
name: benmore-design
description: Use when the user wants to design, restyle, or redesign pages in a Benmore app — for asks like "redesign this page", "make it look like <reference>", "apply a dark theme", "propose designs from my journey", "add a form to this page", "match our brand colors". Teaches the design-kit model, per-page component composition, and drift-aware iteration.
allowed-tools: mcp__benmore__describe mcp__benmore__apply mcp__benmore__get_journey_for_design mcp__benmore__get_page_designs mcp__benmore__save_page_design mcp__benmore__save_page_designs mcp__benmore__apply_design_kit mcp__benmore__write_design_pages mcp__benmore__get_journey_design_drift mcp__benmore__help
---

# Benmore Design Skill

You are proposing or editing designs for a Benmore app. Designs are **component-JSON compositions** (not HTML), materialized by the framework at render time. The visual editor lives at `https://{app}-draft.benmore.ai/_design`.

## The design model — two layers

**1. Kit (global)** — `layouts/app.html`, `partials/app-nav.html`, `partials/kit.html`, CSS variables in `theme.css`. One call establishes it:

```
apply_design_kit(app: "<name>", theme: "linear-dark")
```

Themes available: `linear-dark`, `slate-light`, `zinc`, `rose`. Custom themes are fine — modify `theme.css` after applying the starter.

**2. Pages (per-route)** — each route's components JSON in `platform_page_designs` DB.

Two ways to write pages:

**Bulk from journey (easiest):**
```
write_design_pages(app: "<name>")
```
One call → the server reads the journey, generates reasonable component compositions for every page node, writes them all.

**Per-route surgical:**
```
save_page_design(app: "<name>", branch: "draft", route: "/dashboard",
                 components: [...])
```

**Batched multi-route:**
```
save_page_designs(app, branch, designs: [
  { route: "/dashboard", components: [...] },
  { route: "/tasks", components: [...] }
])
```

## Read before you write

```
get_journey_for_design(app: "<name>", branch: "draft")  # design-shaped journey
get_page_designs(app: "<name>", branch: "draft")        # current designs
```

Never compose blindly — the design-shaped journey output tells you exactly which entities + roles each page uses, which partials to create, etc.

## Component composition shape

Every component is `{type, variant?, props?, children?}`. Types fall into these buckets:

- **Layout**: `box`, `row`, `column`, `grid`, `scroll`
- **Content**: `text`, `heading`, `image`, `icon`, `badge`
- **Data**: `table`, `list`, `card`, `stat`, `board`, `chart`, `detail`, `timeline`
- **Form**: `form`, `input`, `select`, `textarea`, `pressable` (button/link)
- **Feedback**: `alert`, `empty`, `skeleton`, `progress`
- **Navigation**: `tabs`, `accordion`, `dropdown`, `breadcrumb`, `pagination`
- **Utility**: `modal`, `sheet`, `tooltip`, `switch`, `separator`

Full reference: fetch `benmore://docs/components` as an MCP resource.

### Example: a dashboard page composition

```json
{
  "route": "/dashboard",
  "components": [
    { "type": "heading", "props": { "text": "Your tasks", "level": 1 }},
    { "type": "row", "props": { "gap": "sm" }, "children": [
      { "type": "stat", "props": { "label": "Open", "bind": "counts.open" }},
      { "type": "stat", "props": { "label": "Done", "bind": "counts.done" }},
      { "type": "stat", "props": { "label": "Today", "bind": "counts.today" }}
    ]},
    { "type": "table", "props": {
      "source": "tasks",
      "columns": ["title", "status", "due_date"],
      "actions": ["edit", "delete"]
    }},
    { "type": "form", "props": {
      "action": "/api/tasks",
      "method": "POST",
      "fields": [
        { "name": "title",  "required": true },
        { "name": "status", "enum": ["todo", "doing", "done"] }
      ]
    }}
  ]
}
```

## Partial library — the reusable layer

When a visual pattern repeats across pages (task card, user avatar, status badge), extract it to a partial:

```
apply(app, ops: [
  { action: "write_file", path: "partials/task-card.html",
    content: "<div class='card'>{{title}} — {{status}}</div>" }
])
```

Reference in designs via partial props. Convention: `{entity}-card.html`, `{entity}-row.html`, `{entity}-form.html`.

## Drift is your safety net

After every design edit:

```
get_journey_design_drift(app, branch: "draft")
```

**Drift kinds you trigger from design edits:**
- `entity_referenced_no_definition` (blocking) — design uses an entity not in the journey. Fix: add the entity to the journey OR remove the reference.
- `design_uses_undeclared_partial` (blocking) — design references a partial that isn't on disk. Fix: `write_file` the partial, or fix the reference.
- `role_referenced_no_journey_role` (warning) — you used admin UI patterns without declaring the role. Fix: add role to journey OR simplify the design.

The drift panel (MCP App) renders interactive — user can click "Apply Fix" directly.

## Iterate with screenshots

Benmore captures rendered pages for visual feedback:

```
capture_rendered_page(app: "<name>", branch: "draft", route: "/dashboard")
```

Use this after a design edit to verify the result matches intent. For DOM-level debugging (element positions, computed styles):

```
capture_dom(app, branch, route)
```

## Patterns by user ask

| User says | You do |
|---|---|
| "Propose designs for my app" | `get_journey_for_design` → show user the plan → `apply_design_kit` → `write_design_pages` → drift check |
| "Make this page look like <reference>" | `get_page_designs` for current state → propose new composition → `save_page_design` → drift |
| "Apply a dark theme" | `apply_design_kit(theme: 'linear-dark')` — that's it. Theme owns all surface colors. |
| "Change our brand color" | `write_file` to update CSS var `--brand` in `theme.css` |
| "Add <entity> list to <page>" | `save_page_design(route, components: [...existing..., new_table_component])` |
| "Extract this into a partial" | `write_file` partial + `save_page_design` referencing it |
| "Something looks off on <page>" | `capture_rendered_page` → inspect → iterate |

## Gotchas

**IMPORTANT: Apply design kit BEFORE writing pages.** If you `save_page_design` before `apply_design_kit`, the pages render against default styling and look wrong. The kit owns the shell.

**IMPORTANT: Components are JSON, NOT HTML.** Don't pass HTML strings to `components`. If you need a one-off HTML block, either add a new component type (out of scope for a design session) or write a partial.

**IMPORTANT: Don't edit `_platform` app designs.** That's the framework dashboard — owned by the Benmore team, not a customer app.

**IMPORTANT: Page designs are scoped to branch.** Edits in draft don't appear in prod until merge. Always confirm which branch before editing.

## What to tell the user

Short updates: "Applied linear-dark kit. 4 page designs proposed. Drift: 0 blocking, 1 warning (admin role referenced without journey role — recommend adding it)."

Then offer the next step: "Want me to add the admin role to the journey to clear the warning, or leave it for now?"

Don't dump component JSON unless they ask. Paraphrase.
