---
name: os-integration
description: Prepare an application for integration into the MaluDB Business OS — how its memories must be stored (record memory in PostgreSQL 17 with mcp_* visibility views; activity memory as an activity_log shipped to the tenant's MaluDB), how its MCP servers and action manifest must cover every question and every action, and how the platform's agents reach it (run tokens, tool grants, approvals, the ledger, skills, the expert agent, the maludb-os.json registration). Use when building or auditing an Apache/PHP/HTMX application that will be installed beside the Business OS on a tenant's server, when the user says "integrate with the OS", "make this app agent-ready", "prepare for the installation agent", or asks how an app should expose memory, MCP or agents to the platform.
---

# Preparing an application for the Business OS

An application from us is a separate memory-first, ask-me-anything application on the standard
stack (Ubuntu 24.04, Apache, PHP, HTMX, Bootstrap 5.3, PostgreSQL 17 + MaluDB) that the
**installation agent** installs on a tenant's own server beside the platform, at its own DNS name.
People reach it in the browser, signed in by the platform; agents reach it through MCP. This
skill is the contract that makes both true. It assumes the application is (or is being) built
with the `htmx-php-builder` plugin — that plugin governs how the app is built; this one governs
how it **fits**.

## The four things the platform expects

| Expectation | Reference | One-line test |
|---|---|---|
| **Memory is stored the platform's way** — own database, three roles, `security_barrier` `mcp_*` views over `app.member_id`; one `log_activity()` funnel writing `entity.verb` events; every row shipped to the tenant's one MaluDB as an `activity` episode | [references/memory.md](references/memory.md) | "Can the platform's agent recall what happened in this app last week without a tool call?" |
| **MCP and APIs cover the functionality** — a records server and an activity server the app runs, one named tool per recurring question plus one guarded search each, an action manifest for every button, JSON-mode handlers that report `emit_action_status()` and return a `record_id`-able location | [references/mcp-and-api.md](references/mcp-and-api.md) | "Is there anything a screen can do that an agent cannot ask for or do?" |
| **Agents can work here safely** — the tenant's run token is honoured everywhere with the shared `ACTION_TOKEN_KEY`/`ACTIONS_RELAY_KEY`; grants fail closed; approval-category actions pause for agents; no model key, no credential, no secret ever leaves | [references/agents.md](references/agents.md) | "If an agent held this app's tools, what could it do that a person did not decide?" |
| **It declares itself** — `maludb-os.json` at the repo root, shipped skills in `skills/`, the expert's job description in `os/expert.md`, a health endpoint | [references/registration.md](references/registration.md) | "Could the installation agent install, register and propose an expert from the repo alone?" |

## How to work

1. **Read the application first**: its schema, its `activity_log`, its MCP servers if any, its
   action manifest, its `.env.example`. Establish what exists before saying what is missing.
   Do not assume it matches the platform — the platform's own conventions moved
   (`source = 'web'` not `'ui'`; ids re-aliased `<entity>_id`; views gated in SQL, not Python).
2. **Produce a gap list against the four references**, in the order above — memory first,
   because activity memory cannot be backfilled and every later step depends on `app.member_id`
   meaning the platform's member. Each gap names the file to change and the platform rule it
   violates. Where a reference says a platform-side change is *owed*, do not build a substitute
   in the application — build to the contract and record the dependency.
3. **Make the changes in this order**: database and roles → `activity_log` + `log_activity()`
   + the ingest bridge → the `mcp_*` views → the two read servers → the action manifest,
   registry and JSON-mode handlers → identity (members mirror the platform's ids; token
   verification with the shared keys) → `maludb-os.json`, skills, `os/expert.md`, `/api/v1/health`
   → Apache vhost and systemd units under `deploy/`.
4. **Prove it** before calling it done: every migration applies clean on an empty database;
   `mcp_*` views grant only to the read roles; MCP Inspector lists the tools and a person's token
   answers a question from the question inventory; a signed run token (mint one with the shared
   key in a test) is accepted by the read servers and **refused every tool** (fail closed);
   `bin/build_action_registry.php --check` passes; the ingest bridge ships a row and advances the
   checkpoint; `/api/v1/health` answers.
5. **Report** what was changed, what is proven, and which platform-owed items the app now waits
   on — by name, from the tables in `agents.md` and `registration.md`.

## Non-negotiables (refuse to do otherwise, and say why)

- **Never** connect an application to the platform's database, or the platform to the
  application's. MaluDB episodes, MCP and the token API are the only crossings.
- **Never** invent a credential path: the run token is the credential; endpoints carry
  `secret_id = NULL`; nothing decrypts a tenant secret except the platform.
- **Never** fail open on grants, approvals or eval runs. An agent that cannot be confirmed is
  refused, with the platform's own sentence.
- **Never** log, return or display a secret, a token, or more than a 200-character excerpt of a
  sensitive text.
- **Never** create a second memory database — one MaluDB memory per tenant, every application's
  episodes in it, each tagged with its `application` key.
- **Never** register an endpoint before the server behind it answers.
- Ports, hostnames and keys come from `config/.env`; nothing that differs per tenant is a
  constant in code.

## Vocabulary the platform uses (use the same words)

*Tenant* — one business, one server, one database per application, one memory. *Application* —
a built-in module, an application from us, or anyone else's product; each a row in the platform's
registry with endpoints. *Endpoint* — one way in: `mcp`, `http_api`, `ui`, … with an `auth_kind`
and `agent_reachable`. *Expert* — the agent that knows an application. *Lead agent* — a
department's expert and orchestrator in one. *Run* — one piece of an agent's work, with a run
token and a request id. *Gate* — `insider`, `mod:<grant>`, `admin`, `super`. *Episode* — one
activity row as MaluDB stores it. *Skill* — a folder with `SKILL.md`, ingested into MaluDB and
assigned by scope. *Installation agent* — the agent we provide that installs and then watches an
installation for its owner; nothing it sees leaves the server.
