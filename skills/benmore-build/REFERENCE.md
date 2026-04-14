# Benmore Build — Deep Reference

Loaded lazily from `SKILL.md`. Full list of component types, page directives, app.yaml keys, gotchas.

## Component types (for `save_page_design` / `write_design_pages`)

Layout: `box`, `scroll`, `row`, `column`, `grid`
Content: `text`, `heading`, `image`, `icon`, `badge`
Data: `table`, `list`, `chart`, `stat`, `board`, `detail`, `timeline`, `card`
Form: `input`, `select`, `textarea`, `option`, `pressable` (button/link), `form`
Feedback: `alert`, `empty`, `skeleton`, `progress`, `meter`
Navigation: `tabs`, `accordion`, `dropdown`, `breadcrumb`, `pagination`
Utility: `modal`, `sheet`, `tooltip`, `hover-card`, `switch`, `toggle`, `separator`, `kbd`, `copy`, `search`, `avatar`

Full reference: fetch `benmore://docs/components` as an MCP resource.

## Page directives (for HTML pages)

```html
<page title="X" auth="required" scope="owner" role="admin" layout="dashboard">
```

- `auth="required"` — gate behind login
- `scope="owner"` — auto-filter queries by `user_id`
- `role="admin"` — require specific role (comma-sep for OR)
- `require="plan:pro,enterprise verified:1"` — generic user-field gating
- `layout="name"` — named layout from `layouts/` dir, or "none" to skip sidebar
- `type="login|signup|forgot-password|settings"` — auth path discovery
- `title`, `description` — SEO

## Form directives

- `:create="table"` on `<form>` — POST to `/api/{table}`
- `:delete="table" :id="{{id}}"` — DELETE with confirmation
- `:patch="/api/{table}/{{id}}" :set="field=value"` — inline update
- `:validate="required,email,minlen:8"` — server-side validation

## Template syntax

- `{{var}}` — HTML-escaped
- `{{{var}}}` — sanitized (safe HTML kept)
- `{{{{var}}}}` — raw (dangerous)
- `{{var | pipe}}` — apply a pipe (currency, date, ago, upper, truncate:50, etc.)
- `{{#list}}...{{/list}}` — iteration
- `{{^list}}...{{/list}}` — inverted (empty state)
- `{{#if status = "active"}}...{{else}}...{{/if}}` — conditionals
- `{{#role "admin"}}...{{/role}}` — role sections
- `{{param_name}}` — **URL params are prefixed — this is the #1 footgun**

## Query tags

```html
<query from="tasks" pick="id,title,status" where="status='open'" as="open_tasks">
  {{#open_tasks}}<div>{{title}}</div>{{/open_tasks}}
</query>

<query cache="5m" sql="SELECT COUNT(*) as n FROM orders" as="stats">
  {{stats.n}} orders
</query>

<fetch url="https://api.example.com/data" as="data">
  {{#data}}{{value}}{{/data}}
</fetch>
```

## app.yaml (full config)

```yaml
theme: zinc | slate | blue | green | orange | rose
mode: dark | light
font: "Inter"
brand: "#e11d48"
css: tailwind | themed | none
radius: "0.5rem"

nav: { style: sidebar | none }

seo: { site_name, description, url, favicon, lang, security_contact }

auth:
  identifier: email | phone | username
  session_duration: 7d | 30d
  signup_fields: "first_name,last_name,company"
  otp: true
  domain: "app.com"            # restrict signups
  redirect: "/dashboard"
  require_verified: true
  verify_email: true

roles:
  viewer:  { scopes: "contacts:read" }
  editor:  { scopes: "contacts:read contacts:write" }
  admin:   { scopes: "*" }

features:
  admin: true       # /_admin panel
  sse: true         # /sse/events
  ws: true          # /ws WebSocket
  docs: true        # /api/_docs + /docs
  testing: true     # draft-only by default

groups:
  table: user_roles
  key: org_id
  user_field: email

pwa: { name, short_name, icon, theme_color, offline: true }

backup: { interval: "6h", keep: 20 }

aggregates:
  total_revenue:
    sql: "SELECT COALESCE(SUM(amount), 0) as value FROM orders"
    refresh: "5m"

database:
  mode: shared | per_user | per_org
```

## hooks.yaml

```yaml
on_insert:
  tablename:
    - sql: "INSERT INTO audit_log VALUES (?, ?)"
    - webhook: "{{env.SLACK_WEBHOOK}}"
      body: "{{field}} created"
    - email: { to: "{{email}}", subject: "Welcome", template: "welcome" }
    - notify: "{{user_id}}"
      when: "status = 'assigned'"

on_update:
  tablename:
    - sql: "UPDATE stats SET count = count + 1"
      when: "status = 'complete'"
    - webhook: "{{env.WEBHOOK_URL}}"
      body: "{{name}} changed from {{old_status}} to {{status}}"

before_insert:
  tablename:
    - sql: "SELECT CASE WHEN ... THEN RAISE(ABORT, 'error message') END"
```

Session context in all hooks: `{{user_email}}`, `{{user_id}}`, `{{user_role}}`, `{{user_group_id}}`.
Change delta in update hooks: `{{old_status}}`, `{{old_amount}}`, etc.

## flows.yaml (multi-step endpoints)

```yaml
flow_name:
  trigger: POST /api/path/:param
  auth: required
  role: admin
  transaction: true
  verify: X-Hub-Signature         # optional webhook sig header
  secret: "{{env.SECRET}}"
  steps:
    - name: data
      sql: "SELECT * FROM table WHERE id = :param"
    - if: "data.status = 'active'"
      steps:
        - api:
            method: POST
            url: "https://external.api/endpoint"
            auth: "Bearer {{env.API_KEY}}"
            json: { key: "{{data.value}}" }
            timeout: "10s"
            retry: 3
      else:
        - respond: { status: 400, json: { error: "Not active" } }
    - email: { to: "{{data.email}}", subject: "Done", template: "notify" }
    - redirect: "/success"
```

Step types: `sql`, `api`, `webhook`, `email`, `if`/`else`, `for_each`/`as`, `set`, `redirect`, `respond`, `parse`, `on_error`.

## workflows.yaml

```yaml
orders:
  table: orders
  field: status
  initial: pending
  states: [pending, processing, shipped, delivered, cancelled]
  transitions:
    pending:      { processing: { role: admin }, cancelled: {} }
    processing:   { shipped: { role: admin, when: "tracking_number IS NOT NULL" } }
    shipped:      { delivered: {} }
  on_transition:
    shipped:
      - webhook: "{{env.SHIPPING_WEBHOOK}}"
      - notify: "{{user_id}}"
  timeouts:
    pending: { after: "24h", to: cancelled }
```

## computed.yaml / encrypted.yaml / retention.yaml

Covered in `benmore://docs/capability-map`. Fetch as a resource when needed.

## File layout

```
myapp/
├── schema.sql / schema.yaml
├── app.yaml
├── hooks.yaml / flows.yaml / workflows.yaml / computed.yaml / encrypted.yaml / retention.yaml
├── *.html              # pages — filename = route
├── partials/           # shared snippets
├── layouts/            # named layouts
├── emails/             # email templates
├── static/             # images, favicon, docs
├── head.html           # tracking scripts
├── theme.css           # overrides
├── seeds.sql
└── tests/              # *.yaml test cases
```

## Pipes (in templates)

`upper`, `lower`, `title`, `trim`, `truncate:N`, `currency`, `cents`, `number`, `percent`, `date`, `ago`, `digits`, `count`, `md5`, `sha256`, `base64`, `urlencode`, `first`, `last`, `initials`, `default:X`, `pluralize`, `json`, `strip_html`, `nl2br`, `slug`.

## MCP tools cheat sheet

### Composers (prefer these)
- `describe(kind, app?, branch?, filter?)` — replaces all list_* tools
- `apply(app, ops[], stop_on_error?)` — batched mutations
- `sql(app, statement, branch?)` — read-only SELECT
- `help(topic?)` — self-discovery

### Frequently used narrow tools
- App: `create_app`, `get_app_map`, `search_app`, `get_route`
- Files: `read_file`, `write_file`, `get_schema`
- Data: `list_tables`, `get_counts`, `query_db`, `reset_and_seed` (draft only — wipes user tables + re-runs seeds.sql)
- Branch: `create_branch`, `list_branches`, `merge_branch`, `delete_branch`, `revert_branch`, `list_versions`, `get_version_diff`
- Journey: `create_journey`, `get_journey`, `get_journey_for_design`, `add_journey_node`, `add_journey_edge`, `update_journey_node`, `update_journey_edge`, `delete_journey_node`, `delete_journey_edge`
- Design: `apply_design_kit`, `get_page_designs`, `save_page_design`, `save_page_designs`, `write_design_pages`, `get_journey_design_drift`
- Env: `list_env`, `set_env`, `unset_env`
- Platform: `get_draft_url`, `get_usage`, `set_demo_password`, `set_custom_domain`, `upload_static`, `generate_upload_url`
- QA: `capture_rendered_page`, `capture_dom`, `get_check_results`, `run_security_scan`, `list_comments`, `create_comment`, `resolve_comment`, `delete_comment`
- Feedback: `get_testing_status`, `list_visitor_feedback`, `add_visitor_feedback_comment`, `update_visitor_feedback_status`
- Planning: `list_documents`, `create_document`, `update_document`, `list_diagrams`, `create_diagram`, `list_dynamic_assets`
- Logs/health: `get_health`, `get_recent_activity`, `get_request_logs`, `get_analytics`
- Audit: `get_audit_log`, `get_read_audit`
- Users: `list_app_users`, `revoke_session`
- Context: `get_context_graph`

### MCP resources
- `benmore://docs/llm-guide` — full framework doc
- `benmore://docs/components` — every component type
- `benmore://docs/capability-map` — every feature + YAML config
- `benmore://docs/workflows` — common recipes
- `benmore://docs/tool-guide` — how to use the server
- `benmore://docs/architecture` — the MCP/API/CLI/Skills stack
- `ui://benmore/drift-panel.html` — drift UI (MCP App)
