# Benmore Plugin for Claude Code

Build, iterate, and ship full apps on the Benmore platform from inside Claude — using a structured **Discover → Plan → Map → Design → Build** flow with explicit user review at every step.

## Install

```
/plugin marketplace add ric897/benmore-plugin
/plugin install benmore@benmore
```

Then restart Claude Code (or `/reload`).

## What's in the box

- **4 skills** that activate when you say "build me an app", "create a CRM", etc.
  - `benmore-build` — top-level Discover → Plan → Map → Design → Build orchestrator
  - `benmore-journey` — user-flow modeling
  - `benmore-design` — design kit + per-page composition
  - `benmore-deploy` — merge + rollback (user-invocable only)
- **MCP server** at `https://mcp.benmore.ai/mcp` — 95+ tools for app management
- OAuth-based — first MCP call prompts you to authorize at benmore.ai

## Usage

After install, just say: **"build me an app for X"** and the `benmore-build` skill takes over with the 5-phase flow.

```
/plugin                       # See plugin status
/skills                        # See loaded skills
/mcp                          # See MCP connection status
/benmore:benmore-build        # Force-invoke the build skill
```

## Updating

```
/plugin update benmore@benmore
```

## Uninstalling

```
/plugin uninstall benmore@benmore
/plugin marketplace remove benmore
```

## Support

- Docs: https://benmore.ai/docs-mcp
- Email: support@benmore.tech
