---
name: benmore-deploy
description: Use when the user explicitly asks to deploy, ship, publish, merge, rollback, or revert a Benmore app. User-invocable only — the model should not auto-invoke deploys.
disable-model-invocation: true
allowed-tools: mcp__benmore__describe mcp__benmore__get_journey_design_drift mcp__benmore__get_check_results mcp__benmore__run_security_scan mcp__benmore__merge_branch mcp__benmore__revert_branch mcp__benmore__list_versions mcp__benmore__get_version_diff mcp__benmore__help
---

# Benmore Deploy Skill

You are shipping a Benmore app from draft to prod, or rolling prod back to a prior version. This skill is **user-invocable only** (`disable-model-invocation: true`) because deploys are visible and hard to undo — the user must explicitly ask.

## Pre-flight — mandatory before every deploy

Run these three checks **in parallel** before any `merge_branch`:

```
get_journey_design_drift(app, branch: "draft")
get_check_results(app, branch: "draft")
run_security_scan(app, branch: "draft")
```

**Hard rules:**
- `drift.summary.blocking > 0` → STOP. Refuse to merge. Explain what's blocking.
- `check_results` has failures → STOP unless user explicitly overrides.
- `security_scan.critical > 0` → STOP. Show the finding.

If any are present, surface them clearly and ask the user:
> "I see 2 blocking drift issues and 1 critical security finding. Merging now would deploy broken + insecure code. Fix first, or do you want to force-deploy (destructive)?"

Don't auto-resolve. Deploy blockers are signals the build isn't ready.

## The merge

```
merge_branch(app: "<name>", from: "draft", to: "prod")
```

Server returns the merge SHA + deploy URL. Framework hot-reloads prod automatically — the app is live at `{subdomain}.benmore.ai` within seconds.

Confirm to user:
> "Shipped. sha `a1b2c3d`. Live at https://<sub>.benmore.ai. Drift panel clear, 0 blocking, 0 warnings."

## Rollback

```
describe(kind: "versions", app, branch: "prod")  # commit timeline
```

Show user the recent commits. They pick a SHA. Then:

```
revert_branch(app, branch: "prod", to: "<sha>")
```

**IMPORTANT: `revert_branch` WIPES `data.db` on that branch.** This is not undo-safe. Before calling, confirm with the user in plain language:

> "Reverting prod to `a1b2c3d` will ERASE the current production database — every row written since that commit is gone. Users active right now will lose their session data. Is that what you want?"

Only call after explicit yes.

## Named branches (advanced)

For teams iterating multiple features in parallel:

```
create_branch(app, from: "prod", name: "feat-onboarding")
# ... editing happens in apps/{sub}-feat-onboarding/ ...
merge_branch(app, from: "feat-onboarding", to: "prod")
delete_branch(app, name: "feat-onboarding")   # cleanup
```

Each named branch gets its own draft URL: `{sub}-feat-onboarding.benmore.ai`.

## Custom domain

```
set_custom_domain(app, domain: "myapp.com")
```

Server returns the CNAME target. Instruct user to:
1. Add CNAME at their registrar pointing to `{subdomain}.benmore.ai`
2. Come back in 5–15 minutes once DNS propagates
3. Call `set_custom_domain` again (or the server retries on a schedule)
4. Once `custom_domain_verified: true`, SSL auto-provisions via autocert

Cert is **single-level wildcard only**. `myapp.com` works; `a.b.myapp.com` does not.

## Gotchas

**IMPORTANT: Merging to prod is visible to end users instantly.** Unlike "deploy to staging", there's no buffer. Treat every merge as a production release.

**IMPORTANT: Deploy webhooks fire on merge.** If the user's app has webhook subscribers (Slack alerts, CI triggers), they'll fire on every merge — even tiny patches. Warn user if they're merging frequently.

**IMPORTANT: Don't merge during high-traffic windows without user explicit OK.** Hot-reload is fast but not instant; in-flight requests during reload can 500.

## What to tell the user

Before merge, recap:
> "Ready to ship `<app>` from draft to prod.
> - 4 pages changed: /dashboard, /settings, /billing, /admin
> - Drift: clean (0/0/0)
> - Security: clean
> - Checks: 47 pass
>
> Confirm?"

After merge:
> "Shipped. Live at https://<sub>.benmore.ai. sha `a1b2c3d`."

## When to STOP and ask

- Blocking drift → always stop
- Critical security findings → always stop
- User says "revert" without specifying which SHA → ask
- User says "deploy" without specifying which branch → default to draft → prod but confirm
- `delete_branch` on prod → impossible; refuse
- User asks to deploy someone else's app (not on their collab list) → refuse, the server will 403 anyway
