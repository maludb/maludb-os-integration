---
name: os-integration
description: Prepare an application for integration into the MaluDB Business OS kernel — how its memories must be stored (record memory in PostgreSQL 17 with mcp_* visibility views; activity memory as an activity_log shipped to the tenant's MaluDB), how its MCP servers and action manifest must cover every question and every action, how the kernel signs people in (the hand-off token, the directory mirror, the directory API), and how the kernel's agents reach it (run tokens, tool grants, approvals, the ledger, skills, the shipped expert, the command bar through the kernel's chat endpoint, the maludb-os.json registration). Use when building or auditing an Apache/PHP/HTMX application that will be installed beside the Business OS on a tenant's server, when the user says "integrate with the OS", "make this app agent-ready", "prepare for the installation agent", or asks how an app should expose memory, MCP, sign-on or agents to the platform.
---

# Preparing an application for the Business OS

The Business OS is a **kernel** *(decided 2026-09-22)*: it manages the AI agents, the
super-admins who administer them, the company's structure and estate, and single sign-on. It
runs no business application. An application from us is a separate memory-first, ask-me-anything
application on the standard stack (Ubuntu 24.04, Apache, PHP, HTMX, Bootstrap 5.3, PostgreSQL 17
+ MaluDB) that the **installation agent** installs on a tenant's own server beside the kernel, at
its own DNS name (`hr.subello.com`, `reservations.subello.com`). People reach it in the browser,
signed in once at the kernel's launcher (`app.subello.com`); agents reach it through MCP. This
skill is the contract that makes both true. It assumes the application is (or is being) built
with the `htmx-php-builder` plugin — that plugin governs how the app is built; this one governs
how it **fits**. The design it implements: *Business OS — Kernel and Integration Design*
(2026-09-22), `docs/business-os-integration.md` on the platform.

## The five things the kernel expects

| Expectation | Reference | One-line test |
|---|---|---|
| **Memory is stored the kernel's way** — own database, three roles, `security_barrier` `mcp_*` views over `app.member_id`; one `log_activity()` funnel writing `entity.verb` events; every row shipped to the tenant's one MaluDB as an `activity` episode | [references/memory.md](references/memory.md) | "Can the kernel's agent recall what happened in this app last week without a tool call?" |
| **MCP and APIs cover the functionality** — a records server and an activity server the app runs, one named tool per recurring question plus one guarded search each, an action manifest for every button, JSON-mode handlers that report `emit_action_status()` and return a `record_id`-able location | [references/mcp-and-api.md](references/mcp-and-api.md) | "Is there anything a screen can do that an agent cannot ask for or do?" |
| **The kernel signs people in** — no password, no login form; a hand-off token (60 s, single use, audience-bound) opens the app's own session; a mirror of the directory with the kernel's ids, refreshed by the change feed; HR alone changes the directory, through the directory API, as the acting person | [references/sign-on-and-directory.md](references/sign-on-and-directory.md) | "If the kernel deactivated this person a minute ago, is every door here already shut?" |
| **Agents can work here safely** — the tenant's run token is honoured everywhere with the shared `ACTION_TOKEN_KEY`/`ACTIONS_RELAY_KEY`; grants fail closed; approval-category actions pause for agents; the command bar runs the app's expert *in the kernel*; no model key, no credential, no secret ever leaves | [references/agents.md](references/agents.md) | "If an agent held this app's tools, what could it do that a person did not decide?" |
| **It declares itself** — `maludb-os.json` at the repo root with `sso`, `directory`, `assistant` and `agents[]`; shipped skills in `skills/`; each agent's job description under `os/`; a health endpoint | [references/registration.md](references/registration.md) | "Could the installation agent install, register, sign people in and propose the agents from the repo alone?" |

## How to work

1. **Read the application first**: its schema, its `activity_log`, its MCP servers if any, its
   action manifest, its `.env.example`, its login. Establish what exists before saying what is
   missing. Do not assume it matches the kernel — the kernel's own conventions moved
   (`source = 'web'` not `'ui'`; ids re-aliased `<entity>_id`; views gated in SQL, not Python;
   no built-in business modules since 2026-09-22).
2. **Produce a gap list against the five references**, in the order above — memory first,
   because activity memory cannot be backfilled and every later step depends on `app.member_id`
   meaning the kernel's member. Each gap names the file to change and the kernel rule it
   violates. Where a reference says a kernel-side change is *owed*, do not build a substitute
   in the application — build to the contract and record the dependency.
3. **Make the changes in this order**: database and roles → `activity_log` + `log_activity()`
   + the ingest bridge → the `mcp_*` views → the two read servers → the action manifest,
   registry and JSON-mode handlers → identity (the mirror tables with the kernel's ids; the
   `/sso` and `/sso/logout` receivers; the login form removed; the directory timer; HR's writes)
   → the command bar wired to the kernel's chat endpoint → `maludb-os.json`, skills, `os/*.md`,
   `/api/v1/health` → Apache vhost and systemd units under `deploy/`.
4. **Prove it** before calling it done: every migration applies clean on an empty database;
   `mcp_*` views grant only to the read roles; MCP Inspector lists the tools and a person's token
   answers a question from the question inventory; a signed run token (mint one with the shared
   key in a test) is accepted by the read servers and **refused every tool** (fail closed); a
   signed hand-off token (same key) opens a session once and is refused the second time, after
   61 s, and for another `app_key`; a member id with no mirror row is refused; the directory timer
   applies a change and advances its cursor; `bin/build_action_registry.php --check` passes; the
   ingest bridge ships a row and advances the checkpoint; `/api/v1/health` answers.
5. **Report** what was changed, what is proven, and which kernel-owed items the app now waits
   on — by name, from the tables in `agents.md` and `registration.md`.

## Non-negotiables (refuse to do otherwise, and say why)

- **Never** connect an application to the kernel's database, or the kernel to the
  application's. MaluDB episodes, MCP, the hand-off token and the directory, ledger and chat
  endpoints are the only crossings. "Sharing tables" with the kernel is refused by name.
- **Never** keep a password, a login form or an account of the application's own. The hand-off
  token is the only way in for a person; a bearer token the only way in for a tool.
- **Never** hold a model key or call a model from the application. The command bar runs the
  app's expert in the kernel; every model call is the kernel's to ledger.
- **Never** invent a credential path: the run token is the credential; endpoints carry
  `secret_id = NULL`; nothing decrypts a tenant secret except the kernel.
- **Never** fail open on grants, approvals, eval runs, or an unknown member id. An agent or a
  person that cannot be confirmed is refused, with the kernel's own sentence.
- **Never** log, return or display a secret, a token, or more than a 200-character excerpt of a
  sensitive text.
- **Never** create a second memory database — one MaluDB memory per tenant, every application's
  episodes in it, each tagged with its `application` key.
- **Never** register an endpoint before the server behind it answers.
- Ports, hostnames and keys come from `config/.env`; nothing that differs per tenant is a
  constant in code.

## Vocabulary the kernel uses (use the same words)

*Kernel* — the Business OS: agents, super-admins, structure, sign-on; no business applications.
*Tenant* — one business, one server, one database per application, one memory. *Application* —
an application from us, or anyone else's product reached through MCP; each a row in the kernel's
registry with endpoints. *Launcher* — `app.<domain>`, where every human signs in and picks an
application. *Hand-off token* — the 60-second signed token that carries a person from the
launcher into an application. *Application token* — the application's own bearer for the
kernel's directory, ledger and chat endpoints. *Directory* — who exists, their role and status,
their departments; the kernel's, mirrored into every application. *Endpoint* — one way in:
`mcp`, `http_api`, `ui`, … with an `auth_kind` and `agent_reachable`. *Expert* — the agent that
knows an application, shipped with it, hired in the kernel. *Lead agent* — a department's expert
and orchestrator in one. *Run* — one piece of an agent's work, with a run token and a request
id. *Gate* — `insider`, `mod:<grant>`, `admin`, `super`. *Episode* — one activity row as MaluDB
stores it. *Skill* — a folder with `SKILL.md`, ingested into MaluDB and assigned by scope.
*Installation agent* — the agent we provide that installs and then watches an installation for
its owner; nothing it sees leaves the server.
