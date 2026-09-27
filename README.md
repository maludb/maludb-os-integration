# maludb-os-integration

A Claude Code plugin with three skills — `os-integration`, `os-adopt` and `os-install` — for preparing an application to be
installed beside the **MaluDB Business OS** on a tenant's server — as one of the applications we
provide, at its own DNS name, reachable by people through the kernel's single sign-on and by the
kernel's AI agents through MCP.

The Business OS is a **kernel** *(decided 2026-09-22)*: it manages the agents, the super-admins
who administer them, the company's structure and estate, and sign-on — and runs no business
application. Every business application (HR first) is a separate application on this stack,
built with `htmx-php-builder` and fitted to the kernel with this plugin. The design the plugin
implements is *Business OS — Kernel and Integration Design* (2026-09-22), kept as
`docs/business-os-integration.md` in the platform repository.

It answers four questions for Claude Code while it works on such an application:

1. **How are memories expected to be stored?** Record memory in the application's own
   PostgreSQL 17 database behind `mcp_*` visibility views; activity memory as an append-only
   `activity_log` written through one `log_activity()` funnel and shipped to the tenant's one
   MaluDB memory as `activity` episodes. → `skills/os-integration/references/memory.md`
2. **How must MCP servers and APIs cover the application?** A records server and an activity
   server the application runs; one named tool per recurring question plus one guarded search;
   an action manifest for every button, executed by the platform's actions server against the
   application's JSON-mode handlers; a token API for outside systems.
   → `references/mcp-and-api.md`
3. **How does a person get in, and how does the app know who exists?** No password and no login
   form: a 60-second single-use hand-off token from the kernel's launcher opens the app's own
   session; the app mirrors the kernel's directory with the kernel's ids and keeps it fresh from
   a change feed; HR alone changes the directory, through the directory API, as the acting
   person; the command bar runs the app's expert in the kernel. → `references/sign-on-and-directory.md`
4. **How do the kernel's agents expect to interface?** The tenant's run token honoured with
   shared keys, tool grants that fail closed, approvals that pause for agents, the prompt ledger,
   skills assigned at application scope, the shipped agents (the expert first), and the
   `maludb-os.json` registration the installation agent reads. → `references/agents.md`, `references/registration.md`

The references are written from the platform's code as of 2026-09-21, revised to the kernel
design of 2026-09-22, and say plainly which kernel-side pieces are still owed (build-plan
phase 7) so an application is built to the contract and nothing is faked in the meantime.

## The three skills (0.4.0, 2026-09-27)

| Skill | Use it when | It produces |
|---|---|---|
| `os-integration` | Building a new application for the kernel, or auditing one. It holds the contract: memory, MCP, sign-on and the directory, **scoped applications** (one installation serving several sites or departments), **roles and rights** (the application publishes its roles and what each lets its holder do through `app_roles`; the super-admin grants any set of them; the application enforces), agents, registration | Code to the contract. `references/php-sign-on-kit.md` has proven PHP to copy, and `references/testing-without-a-kernel.md` shows how to prove sign-on and scopes on a server with no kernel |
| `os-adopt` | An **existing** application with its own login, users and perhaps tenants (a restaurant each) is to be run from the kernel | A branch in the application's repository. Its users are linked to the kernel's members and its tenants to kernel scopes, the kernel's hand-off token ends in the application's own login function, and everything sits behind one `OS_ENABLED` flag, so the application still runs standalone |
| `os-install` | A repository with a `maludb-os.json` is to be installed on a kernel's server | The application running at `<label>.<domain>`, registered in the kernel (catalog entry, application, roles, endpoints, token, scopes, grants), its expert proposed, and sign-on proven end to end |

The order for a new application on another server: build it with `htmx-php-builder` + `os-integration`,
and prove it with `testing-without-a-kernel.md`. Then, on the kernel's server, `os-install` it. For an
existing application: `os-adopt` in its repository, then `os-install`.

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
skills/os-integration/references/   memory.md · mcp-and-api.md · sign-on-and-directory.md · agents.md · registration.md
                                    scoped-applications.md · roles-and-rights.md · php-sign-on-kit.md · testing-without-a-kernel.md
```
