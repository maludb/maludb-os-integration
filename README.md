# maludb-os-integration

A Claude Code plugin with one skill, `os-integration`, for preparing an application to be
installed beside the **MaluDB Business OS** on a tenant's server — as one of the applications we
provide, at its own DNS name, reachable by people through the platform's sign-on and by the
platform's AI agents through MCP.

It answers three questions for Claude Code while it works on such an application:

1. **How are memories expected to be stored?** Record memory in the application's own
   PostgreSQL 17 database behind `mcp_*` visibility views; activity memory as an append-only
   `activity_log` written through one `log_activity()` funnel and shipped to the tenant's one
   MaluDB memory as `activity` episodes. → `skills/os-integration/references/memory.md`
2. **How must MCP servers and APIs cover the application?** A records server and an activity
   server the application runs; one named tool per recurring question plus one guarded search;
   an action manifest for every button, executed by the platform's actions server against the
   application's JSON-mode handlers; a token API for outside systems.
   → `references/mcp-and-api.md`
3. **How do the platform's agents expect to interface?** The tenant's run token honoured with
   shared keys, tool grants that fail closed, approvals that pause for agents, the prompt ledger,
   skills assigned at application scope, a proposed expert agent, and the `maludb-os.json`
   registration the installation agent reads. → `references/agents.md`, `references/registration.md`

The references are written from the platform's code as of 2026-09-21 and say plainly which
platform-side pieces are still owed (build-plan phase 7) so an application is built to the
contract and nothing is faked in the meantime.

## Installation

In Claude Code:

```
/plugin marketplace add maludb/maludb-os-integration
/plugin install maludb-os-integration@maludb-os
```

The repository is its own marketplace (`.claude-plugin/marketplace.json`, marketplace name
`maludb-os`) with the plugin at the repository root. Locally, before it is published:

```
/plugin marketplace add /home/maludb/maludb-os-integration
```

## Relationship to `htmx-php-builder`

`htmx-php-builder` governs how an application on this stack is **built** (build order, PHP
patterns, design system, the two memory MCP servers, chat actions). `maludb-os-integration`
governs how it **fits the platform**. Use both: build with the first, integrate with the second.
Where the two say different things about a detail (for example the activity log's `source`
values, or who runs the actions server), this plugin is the platform's current word.

## Layout

```
.claude-plugin/plugin.json          plugin manifest
.claude-plugin/marketplace.json     this repo as a marketplace
skills/os-integration/SKILL.md      the skill: what the platform expects, how to work, non-negotiables
skills/os-integration/references/   memory.md · mcp-and-api.md · agents.md · registration.md
```
