---
name: benmore-build
description: Use when the user wants to build, scaffold, or modify a Benmore app from scratch — for asks like "build me a <X>", "create a CRM / todo / weather app", "scaffold an app", "add a feature to my app". Enforces the 5-phase opinionated build flow: Discover → Plan → Map → Design → Build, with explicit user approval between each phase.
allowed-tools: mcp__benmore__describe mcp__benmore__apply mcp__benmore__sql mcp__benmore__help mcp__benmore__create_app mcp__benmore__save_brief mcp__benmore__save_plan mcp__benmore__set_build_phase mcp__benmore__build_app mcp__benmore__create_journey mcp__benmore__add_journey_node mcp__benmore__add_journey_edge mcp__benmore__update_journey_node mcp__benmore__get_journey mcp__benmore__get_journey_for_design mcp__benmore__apply_design_kit mcp__benmore__write_design_pages mcp__benmore__save_page_design mcp__benmore__save_page_designs mcp__benmore__get_journey_design_drift mcp__benmore__get_app_map mcp__benmore__read_file mcp__benmore__write_file mcp__benmore__merge_branch
---

# Benmore Build — The 5-Phase Opinionated Flow

You are guiding a user through building an app on Benmore. **This is NOT a "scaffold and ship" job.** It is a **5-phase iterative process with hard stops between every phase.** The user reviews and approves each phase before you proceed. Skipping phases produces broken apps that look like generic templates. Do not skip phases.

```
1. DISCOVER  →  vision + business value + identity + design refs
   ⌐ STOP: brief.md approved
2. PLAN      →  compliance + tech red flags + inspiration teardown
   ⌐ STOP: 3 plan docs approved
3. MAP       →  journey graph in /_journeys
   ⌐ STOP: graph approved
4. DESIGN    →  kit (stop) + pages (stop) in /_design
   ⌐ STOP × 2
5. BUILD     →  promote → real routes + schema + hooks → live draft
   ⌐ STOP: signup, test, ship to prod
```

## The hard rule

**After producing any phase's artifact, your turn ends with a review prompt and you wait for the user.** You do NOT call the next phase's tools in the same turn. You do NOT bundle phases. The user's response is what advances the flow.

This is enforced at the tool level too — most tools refuse if the previous phase isn't approved (override: `skip_gate: true`, but that's for power users iterating, not first builds).

---

## Step 0: Always orient first

For any new session or any user mention of an existing app, call **in parallel**:

1. `mcp__benmore__describe(kind: "apps")` — see what's deployed
2. If user named an app: read `benmore://apps/{app}/build-state` (resource) — find out which phase to resume

If the build_state resource shows phase=`discover` and gates_passed=[], it's a fresh app — start at Phase 1. If phase=`map` and gates_passed includes `discover` and `plan`, the user already approved those — pick up at Map.

**Do not start over.** If a phase is already approved, skip past it.

---

## Phase 1: DISCOVER

**Goal:** Understand what the user wants. NOT what the app looks like — what the app IS.

### Run a focused 5-question Q&A

Don't be a form. Be conversational. Ask follow-ups. But cover all 5:

1. **What is it?** One sentence. ("A CRM for solo realtors who hate Salesforce.")
2. **Who specifically uses it?** Not "salespeople" — "a sales rep at a 50-person SaaS who logs 30 calls a week."
3. **What's the core flow?** The single most important thing they do. ("Drag deals through pipeline stages, log every touch as an activity.")
4. **Why does this exist?** Business value — what's the unfair edge or opportunity? ("Pipedrive is too configuration-heavy for solo founders. We're stripped-down + opinionated.")
5. **Visual references.** Ask the user to drag in screenshots or paste links of apps they want this to feel like. Save each via `mcp__benmore__add_image` or `mcp__benmore__upload_media` with notes about what they like about it.

Identity (name, brand personality) gets inferred from #1–4 — don't ask separately.

### Write the brief

Once Q&A is done, draft a markdown brief and call `mcp__benmore__save_brief(app, content, status: "draft")`. Sections to include:

```markdown
# Brief: <app name>

## Vision
<one-paragraph from Q1>

## Who uses it
<from Q2 — be specific>

## Core flow
<from Q3 — concrete steps>

## Why this exists
<from Q4 — the opportunity>

## Identity
<inferred name, personality (bold/clinical/playful/enterprise), tone>

## Design references
- <screenshot 1>: what to take from it
- <link 2>: what to take from it
- ...
```

### STOP

End your turn with:

> "Drafted the brief — see below. Anything you want to change?
> Reply with edits, or say **'approve'** to move to research (Phase 2: Plan)."

Then **STOP**. Do not call `run_research` or `save_plan`. Wait for the user.

When the user says approve, call `mcp__benmore__save_brief(app, content, status: "approved")` to lock the gate.

---

## Phase 2: PLAN

**Goal:** Auto-research the brief — compliance landscape, technical red flags, design inspiration teardown. Three docs, always.

### Do the actual research

You have web search. Use it. For each ref the user dropped in Phase 1, look up the company / product, summarize what's good. For the domain (CRM, healthcare, fintech, edu) look up the regulations.

### Three sections (always all three, regardless of domain)

**Compliance** — `docs/02-compliance.md`
- What regulations apply? (HIPAA if healthcare data; PCI if cardholder data; GDPR if any EU users; FERPA if student data; SOC 2 for enterprise selling)
- For each regulation: what Benmore covers OOTB (`audit_log`, `encrypted.yaml`, `retention.yaml`, `read_audit`, MFA, scopes, soft delete) and what stays on the user's plate (BAAs, certs, DPO, infrastructure attestations).
- If the domain is unregulated, say so explicitly: "No specific regulations apply to a personal todo list. Standard data hygiene applies."
- **Informational only — no signoffs required.**

**Tech red flags** — `docs/03-tech-plan.md`
- Top-level data model sketch (3–8 entities, relationships) — this directly informs Phase 3 journey entities.
- Integrations needed (Stripe, Gmail/IMAP, OAuth providers, Slack, webhooks).
- Real-time? Offline-first? Native mobile? — flag any "Benmore is a bad fit for this" early. Be honest. "Real-time multiplayer game — use Fly or Phoenix, not us." Premium positioning beats false promises.
- Feasibility verdict: **good fit / mostly good fit / bad fit**. State it.

**Inspiration teardown** — `docs/04-inspiration.md`
- For each design ref: what to steal, what to skip. ("Pipedrive — steal: stage-coded color dots, deal cards on kanban. Skip: dense top nav, modal-heavy.")
- Synthesized direction: **one sentence** describing the visual approach. "Lean Attio's data density + Folk's warmth, ditch Salesforce verbosity."
- Proposed visual tokens: theme (dark/light), accent color (pick one), typography, density.

### Save and STOP

Call `mcp__benmore__save_plan(app, compliance, tech_plan, inspiration, status: "draft")`. Then end your turn:

> "3 plan docs written. Highlights:
> - **Compliance:** <top 1–2 risks or 'no specific regulations apply'>
> - **Tech:** <feasibility verdict + any integrations needed>
> - **Inspiration:** <one-sentence direction>
>
> Anything to push back on? Reply with edits, or say **'approve'** to move to user flows (Phase 3: Map)."

**STOP.** Do not call `create_journey`. Wait.

When approved: `save_plan(... status: "approved")`.

---

## Phase 3: MAP (the journey)

**Goal:** Model every user flow as a journey graph. This is the spec — pages, entities, roles, transitions.

### Read the plan first

Read `docs/03-tech-plan.md` via `mcp__benmore__describe(kind: "documents", app)` — its data model sketch directly seeds your journey entity nodes. Don't invent new entities; use what the plan said.

### Build the journey

Single batched `mcp__benmore__create_journey(app, name, nodes, edges)` call. Include:
- **Entity nodes** (one per table from the tech plan, with `columns_json`)
- **Role nodes** (user, admin, viewer, etc.)
- **Page nodes** (route, role, optional layout)
- **Action nodes** (`create_X`, `update_X` — anything that mutates)
- **Decision nodes** (auth gates, branching)
- **Entry node** (the landing)
- **Edges**: user_action between pages, system between page+decision, creates/updates/reads between page+entity

### Show the user, STOP

End your turn with:

> "Journey modeled — N nodes, M edges. **Open the canvas to review:**
>
> https://{app}-draft.benmore.ai/_journeys
>
> Look at the graph — does the flow match how a real user would move through your app? Any missing pages? Wrong roles? Misnamed entities?
>
> Reply with edits, or say **'approve'** to move to design (Phase 4)."

**STOP.** Do not call `apply_design_kit`. Wait.

User edits via `mcp__benmore__add_journey_node` / `update_journey_node` / `delete_journey_node` / etc. Iterate until they approve.

When approved: `mcp__benmore__set_build_phase(app, phase: "design")`.

---

## Phase 4: DESIGN

**Two sub-phases, two stops.**

### 4a: Kit (visual foundation)

Read `docs/04-inspiration.md` for direction. Propose:
- `layouts/app.html` — the shell
- `partials/app-nav.html` — the sidebar
- `partials/kit.html` — CSS variables + utility classes
- `partials/<entity>-card.html`, `partials/<entity>-row.html` for each entity from the journey

Call `mcp__benmore__apply_design_kit(app, kit_name, files, app_yaml_patch)`.

**STOP:**

> "Kit applied — `kit_name`. **Review the partial library in the design canvas:**
>
> https://{app}-draft.benmore.ai/_design
>
> Click around the partials — are the colors right? Typography readable? Cards the right density? Any partial you'd want different (more compact, more visual)?
>
> Reply with changes, or say **'approve'** to move to page compositions."

Iterate via additional `apply_design_kit` calls (it accepts partial file lists — only the changed ones).

### 4b: Page compositions

Once kit approved, call `mcp__benmore__write_design_pages(app, pages)` with full `<page>...</page>` HTML for every journey page. Use `<query mock="N" from="entity">` so the prototypes render with realistic mock data.

**MANDATORY: After EVERY `write_design_pages` call (or any batch of `save_page_design` edits), IMMEDIATELY call `mcp__benmore__get_journey_design_drift(app)` BEFORE replying to the user.** Surface any blocking issues in your reply. Don't wait for the user to ask. If drift shows ANY blocking issues, fix them before the STOP — don't ship a known-broken state to the review.

**MANDATORY syntax rules for page HTML — get these wrong and the page renders blank / raw / 404s on login:**

- Include partials with `<include src="partials/foo.html" />` — NOT `{{> foo}}` (Mustache hash-syntax). Benmore is NOT Handlebars/Mustache for partials.
- Always wrap the content in `<page title="..." layout="app">` (or `layout="none"` for landing/auth pages).
- **Any page that should require login MUST have `auth="required"` on its `<page>` tag.** This is load-bearing: if NO page in the app has `auth="required"`, the framework skips creating `_benmore_users` / `_benmore_sessions` tables AND skips registering the `/login` + `/signup` POST handlers. Signup forms will silently 404 on submit. Rule: if the journey node for this page has a `role`, the page MUST have `auth="required"`. `build_app` will auto-inject this during promotion as a safety net, but include it in your source HTML so what you ship matches what's live.
- Auth-page-type pages (login, signup, forgot-password, reset-password) need `type="login"` / `type="signup"` / etc. on the `<page>` tag — this tells Benmore to wire them to the auth handlers. WITHOUT this attribute, the form is just a decorative form, the POST has nowhere to go.
- Signup + login forms need: `<form method="POST" action="/signup">` (or `/login`), a hidden `csrf_token` field: `<input type="hidden" name="csrf_token" value="{{csrf_token}}">`, a display slot for auth errors: `{{#if param_error}}<div class="error">{{param_error}}</div>{{/if}}`, and `name=` attributes on every input.
- Inside the page body, the layout's `{{content}}` slot receives whatever you put inside `<page>...</page>`.
- Templates use `{{var}}` (escaped) or `{{{var}}}` (sanitized) for interpolation. Loops are `{{#list}}...{{/list}}`. Conditionals are `{{#if cond}}...{{/if}}`.
- URL params are `{{param_name}}` — NOT `{{name}}`. (e.g., for route `/student?id=X`, use `{{param_id}}`, not `{{id}}`.)
- For dynamic detail-page routes, name the file `<entity>-param_<key>.html` so the path resolver picks it up.
- `auth.redirect` in `app.yaml` MUST point to an existing authed page (e.g., `/dashboard`). Don't point it at `/` if `/` is your marketing landing — successful logins will bounce the user back to the landing and look broken.

**STOP:**

> "14 prototype pages written. **Click through the top 3:**
>
> - https://{app}-draft.benmore.ai/_design/p/dashboard
> - https://{app}-draft.benmore.ai/_design/p/pipeline (or whatever the core flow page is)
> - https://{app}-draft.benmore.ai/_design/p/<some detail page>
>
> Do they match the brief? Layout right? Information hierarchy clear? Anything missing or wrong on any specific page?
>
> Reply with changes (you can name specific pages or routes), or say **'ship it'** to move to build (Phase 5)."

Iterate via `mcp__benmore__save_page_design` for individual route changes.

When approved: `mcp__benmore__set_build_phase(app, phase: "build")`.

---

## Phase 5: BUILD

**Goal:** Promote prototypes to a real running app the user can sign up for and test.

### One call

`mcp__benmore__build_app(app, generate_hooks: false)`. This:
- Generates `schema.sql` from journey entity nodes (their `columns_json`)
- Promotes `_design/*.html` → app root (real routes — `/dashboard`, `/pipeline`, etc., live with auth + scoping)
- Hot-reloads the draft

### Drift check

Call `mcp__benmore__get_journey_design_drift(app)` after build. Should be 0 blocking. If not, surface the issues + fix them via `apply` ops.

### STOP — give the live URL

> "**Live at https://{app}-draft.benmore.ai/**
>
> 1. Sign up at /signup (creates the first user)
> 2. Create a {core entity from journey} — does the flow match what we modeled?
> 3. Click through the journey edges — do all pages load? Forms work?
>
> Tell me what breaks or what's off. When everything works, say **'ship to prod'** and I'll merge."

When user says ship: `mcp__benmore__merge_branch(app, from: "draft", to: "prod")`.

---

## Anti-patterns (STOP if you find yourself doing these)

- **Skipping Phase 1's Q&A** because "I can infer from the request." — No. Ask the 5 questions. The user's exact phrasing matters.
- **Running `create_journey` in the same turn as `save_brief(approved)`.** Each phase ends your turn.
- **Calling `apply_design_kit` before Map is approved.** The tool will refuse with a phase-gate error. Don't pass `skip_gate: true` to bypass on a first build.
- **Using a template starter.** Benmore has none. Always build from the brief.
- **Picking a design kit before reading the inspiration teardown.** The kit must reflect what the user told us, not what you'd default to.
- **Bundling phases ("here's the brief AND the journey AND the design").** Each phase = one turn = one stop.

## Gotchas

**IMPORTANT: URL params are `{{param_name}}` — NOT `{{name}}`.** URL params always take a `param_` prefix in templates. Mixing these up costs hours of debugging.

**IMPORTANT: `revert_branch` wipes `data.db` on that branch.** Treat as destructive. Confirm with user.

**IMPORTANT: Cert is single-level wildcard.** `foo-bar.benmore.ai` works; `sub.foo.benmore.ai` does not.

**IMPORTANT: `_platform` is a reserved subdomain.** Never try to create an app with that name.

**IMPORTANT: `write_design_pages` writes to `_design/` (sandbox).** It does NOT make routes live. You MUST call `build_app` afterward to promote them to real routes.

## When to call which tool

| Phase | Tools |
|---|---|
| Orient | `describe(kind: "apps")`, read `benmore://apps/{app}/build-state` |
| 1 Discover | `add_image` / `upload_media`, `save_brief` |
| 2 Plan | `save_plan` |
| 3 Map | `create_journey`, `add_journey_node`, `add_journey_edge`, `update_journey_node`, `set_build_phase(phase: "design")` |
| 4 Design | `apply_design_kit`, `write_design_pages`, `save_page_design`, `get_journey_design_drift`, `set_build_phase(phase: "build")` |
| 5 Build | `build_app`, drift check, `merge_branch` |

For deeper reference (component types, page directives, app.yaml schema, all directives) read `REFERENCE.md` in this directory.
