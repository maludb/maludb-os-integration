# Agents — how the kernel's workforce expects to reach an application

An agent on the kernel is an employee: a member row (`member_kind = 'agent'`, always role
`user`), an employment profile (job description = system prompt, model, budget, manager, tool
grants, duties), and a run record for every piece of work it does. **An application never runs
an agent, never holds a model key, and never sees a credential.** It is a place agents work.

## 1. What a run looks like from the application's side

The kernel's agent runner (one process: a runner API on 8815 and a **ledger proxy** on 8816)
starts a run, mints two per-run credentials, prepares a sandboxed profile, and executes it in a
harness (`hermes` or `claude_agent_sdk`). What reaches the application:

| Arrives as | On | Meaning |
|---|---|---|
| `Authorization: Bearer {member}.{exp}.{run}.{hmac}` | The application's records / activity MCP | **The run token** — signed with the tenant's `ACTION_TOKEN_KEY` over `"run:member.exp.run"`. The member id is the agent; the run id is the `agent_runs` row. TTL = the run's timeout + 60 s. |
| `X-Action-Token` + `X-Action-Relay: hmac(ACTIONS_RELAY_KEY, token)` | The application's PHP handlers, on the internal port | The same token, relayed by the kernel's actions server. Only that server holds the relay key, so an agent cannot reach a handler directly. |
| `X-Action-Token` + `X-Approval-Replay: {id}.{hmac}` | A PHP handler | A person approved a paused action; the kernel re-POSTs the original body as the requester (120 s token). |

The application therefore verifies, with the tenant's shared keys, and then **acts as the member
the token names**: sets `app.member_id`, refuses an id it has no member row for (§4 of
`mcp-and-api.md`), sends no cookie, and logs with `source = 'agent'` and the run's `request_id`.

### What the agent already knows before it calls

- Its **job description** and department handbooks; its **core memory** and a recall from
  MaluDB (`agent:<id>`, its departments, `org`) — screened so nothing that reads like an
  instruction gets in.
- Its **skills**, synced read-only into its profile from MaluDB — the application's shipped
  skills among them when it may use the application (§3).
- Its **tools**: for every endpoint it holds grants on, an MCP client entry `{url, headers:
  {Authorization: "Bearer ${BOS_RUN_TOKEN}"}}` and the exact list of tool names granted. The
  tool name is what the server reports; the harness prefixes `mcp__<slug>__`. A tool it was not
  granted is not offered, and if called anyway is refused at the server.
- Where to look first: `my_applications` on the kernel (what it can reach — name, url,
  capability, endpoint count) and `get_application` (the endpoints: name, kind, url,
  `auth_kind`, `has_credential` as a boolean, `mcp_surface_version`). **Never a credential.**

### What the agent does not do

- **Call a model itself.** Every model call goes through the ledger proxy, which holds the
  provider keys, forces the registered model, refuses once the month's budget is spent (402),
  and writes the **prompt ledger** row (full context, full response, tokens, latency, cost) —
  linked to `activity_log` by `request_id`. A run whose call could not be ledgered is void.
- **Approve anything.** An agent may never answer an approval, give a run verdict, or assign a
  skill.
- **Write memory directly.** Remembering is a kernel action that is gated, approved and logged.

## 2. Writes: grants, approvals, evaluations — three controls, all kernel-side

1. **Tool grants** (`agent_tool_grants`: agent × endpoint × tool, optional
   `constraints.max_amount`). Enforced at the MCP boundary; a grant can only name an endpoint that
   is `kind='mcp'`, `agent_reachable` and `active`. For an application, the grantable things are
   its read tools (on its own endpoints) and its action tools (on the kernel's Actions MCP,
   once its registry is loaded).
2. **Approvals.** A handler asks `check_approval(action_key, log_event, summary, parameters,
   entity, amount, currency)` before it writes. A policy matches on the **log event**
   (`booking.cancel`, `refund.*`, `*.delete`), on who (`agents`, one agent, a department,
   everyone) and on a money threshold; if one matches, the handler answers **202
   `{status:'pending_approval', did:'… waits for approval', approval_request_id}`**, the run ends
   `awaiting_approval`, the approver (the policy's, else the nearest *human* up the management
   chain) is notified, and the agent's persona says: stop and report. Approval **replays** the
   request to the same handler. The manifest's "Agent approval" column declares the category
   (`money_out`, `deletion`, `external_send`, `other`).
3. **Evaluations.** An eval run may change nothing. The kernel's actions server records what a
   write would have called and never sends it, and refuses outright when it cannot tell whether
   the run is an evaluation; the eval runner checks after every trial that nothing changed.

For an application from us, (1) and (3) come free by routing writes through the kernel's
actions server. (2) needs the application's handlers to pause the same way — see the contract:

**Contract, until the kernel's approval hook exists** (owed, below): a handler whose manifest
row carries an approval category, when called by an **agent** (a four-part token), answers
`emit_action_status(false, …)` with HTTP **202** and the `pending_approval` body above *without
creating a request* — `approval_request_id: null` — and logs `booking.cancel.paused` with the
parameters. The agent stops and reports, a person does the action by hand, and nothing an agent
should not do alone is done. Fail closed; never let an approval-category action through because
the policy engine is not reachable.

## 3. Skills, expert and memory — what the application contributes

- **Skills it ships** (`skills/<name>/` in the repository — bundle rules in `memory.md` §3).
  The installation agent ingests them and assigns them at **application scope**, so they reach
  the application's expert and every agent that may use it. Specificity when skills conflict:
  agent → role → department → **application** → org. Write them for an agent that has the
  application's tools in front of it: what each tool is for, in what order a job is done, which
  actions pause for approval, what the record vocabulary means. A skill that says "call
  `booking_create` with a contact id and a start time; if the slot is taken, `find_slots` for
  the same day and offer three" is worth more than a feature list.
- **Its expert** — the agent people and other agents ask about the application. The application
  proposes it (`registration.md`, the `expert` block): a job description, the tools it should be
  granted on each endpoint, the access capability. The kernel proposes the hire; a person
  confirms (model, budget, manager). Naming an expert grants nothing — the proposal is what
  makes the access request complete.
- **Memory** — episodes only (`memory.md` §2). The application does not write agent memory.
  Its episodes are what let an agent recall "the last time this customer rebooked" without a
  tool call.

## 4. Telemetry the application must not break

The kernel keeps, for every run: `agent_run_events` (tool call / result / denied, with
duration and status — no arguments, no results), `prompt_ledger` + `prompt_payloads`, and a
person's `run_verdicts`. Evidence an evaluation will later need, none of it backfillable. The
application's part is small and exact:

- Honour `X-Request-Id`; under a run token use the run's request id for every `activity_log`
  row (`memory.md` §2) — that is the join to the ledger.
- Answer tool calls with **actionable errors**, not stack traces: the event row records
  `status: error` and the agent reads the message.
- Return **`record_id`**-able locations from every create.
- Never put a secret, a token or a full document body in a tool result, a log row or an error.

## 5. The command bar — the application's assistant, run by the kernel *(2026-09-22)*

The voice-first command bar every application ships stays the application's feature, and until
the desktop companion exists it is the one place a person meets an agent. The agent behind it is
one the application ships (`registration.md`, `assistant.agent`), hired in the kernel; the bar
posts each utterance to the kernel's chat endpoint with the application token and the acting
member, and the kernel runs one turn of that agent — the application's tools attached, every
model call through the ledger proxy, grants and approvals enforced as for any run — and answers
with the reply, the actions taken or paused, and where to navigate. **The application never
calls a model and never holds a model key**; the alternative (its own router, ledger rows shipped
afterwards) was rejected for that reason. Wire format and the interim behaviour:
`sign-on-and-directory.md` §5.

## 6. Duties and delegation (context, nothing to build)

Agents pick up **duties** on a cron schedule (a missed day is one run, not 24), an
**orchestrator** may delegate one level down to subagents on its roster, and a **voice agent**
answers calls and delegates nothing. Work addressed to a location is meant to go to that
location's office-manager agent, and to a department its lead agent — the routing is designed,
not built. An application does not address agents; it exposes tools, and duties or people call
them.

## What the kernel still owes before an application's tools are live for agents

Recorded so the application is built to the contract and nothing is faked meanwhile. All are
build-plan phase 7 items on the kernel side.

| Owed | Why the application cannot do it alone | Until then |
|---|---|---|
| ~~Sign-on~~ — **built 2026-09-22 (A2–A3)**: the launcher at `app.<domain>`, `/launch/<id>` mints the token and sends the browser to the application's `sso_path`, sign-out posts the notice to `sso_logout_path` | — | Build the receivers (`sign-on-and-directory.md` §1–2) and register the two paths |
| ~~The directory API and the application token~~ — **built 2026-09-22 (A4)** (`sign-on-and-directory.md` §4) | — | Build the mirror, the timer and (HR) the write calls |
| ~~The chat endpoint~~ — **built 2026-09-22 (A6)** (§5) | — | Wire the bar to it |
| ~~The renderers attach bearer endpoints of applications from us~~ — **built 2026-09-22 (A7 a)**: an endpoint of an application whose catalog kind is `ours` and whose `auth_kind` is `bearer` is attached with `${BOS_RUN_TOKEN}` on both harnesses | — | Register the application against an `ours` catalog entry (below) and its MCP endpoints as `bearer`, `agent_reachable` |
| ~~A run-facts call~~ — **built 2026-09-22 (A7 b)**: `POST {OS_INTERNAL_URL}/api/v1/runs/facts.php` with the application token, body `{"token": <the bearer the caller presented>}` → `{valid, is_agent, member_id, member_name, run_id, request_id, trigger, is_eval, run_status, endpoints:[{id, name, url, tools:{name: constraints}}], capability, role, scopes:[{scope_id, kind, id, name, role, capability}]}` (the last three since 2026-09-25 — the caller's holding on THIS application, the claims' shape; a scoped application lets an agent act only in the scopes listed) — the tools are this application's endpoints only; a person's token answers `is_agent:false` (never filtered); an invalid token `valid:false` | — | Call it once per run id (cache by run id), fail closed on `valid:false`, filter tools by the answer, refuse writes when `is_eval` |
| ~~The actions server loads an application's registry~~ — **built 2026-09-22 (A7 c)**: `mcp/registries/<app_key>.json` on the kernel = `{schema:"maludb-os.registry/1", app_key, name, base_url, records_url, resolve:{entity:{tool, query_param, id_field, label_field, params[]}}, registry:<the app's mcp/action_registry.json>}`; every built action becomes a tool on the kernel's Actions MCP, names resolved through the application's own `find_*` tool with the caller's token | — | Ship the manifest and registry; the installation agent writes the file |
| ~~An approval hook~~ — **built 2026-09-22 (A7 d)**: the actions server asks the kernel (`html/approvals/hook.php`) before posting an action with an approval category; a matching policy pauses it (202 `pending_approval`) and a later approval REPLAYS the request to the application's handler with `X-Action-Token` + `X-Approval-Replay: {id}.{hmac}` (hmac over `replay:{id}.{path}.{sha256(canonical body)}` with `ACTION_TOKEN_KEY`, the path = the handler's own `REQUEST_URI` path) | — | Verify the replay as `approval_replay_verify()` does on the kernel, then run the action; §2's "answer 202 with no request" contract is no longer needed |
| ~~`application_catalog.kind = 'ours'`~~ — **built 2026-09-22 (A7 f)**: db/139; `application_catalog_save` (super-admin) adds the row: `catalog_key, name, business_area, kind=ours, description, icon, category, vendor` | — | The installation agent adds the row, then registers the application against it |
| ~~A skills path for `claude_agent_sdk` agents~~ — **provided 2026-09-22 (A7 g)**: the runner hands the Claude harness the agent's synced skills as a plugin (`<agent_dir>/plugin`, `--plugin-dir`) and admits the `Skill` tool; see the kernel's `docs/build-specs/kernel-owed-items.md` for what the pinned CLI showed | — | Ship skills in `skills/` as before |

*Withdrawn 2026-09-22:* the ledger ingestion call for an application's own assistant — an
application no longer makes model calls of its own (§5).

Build order on the kernel, from the design: the cut → hosts → sign-on → the directory API → the
ledger's period export → the chat endpoint → the rest of this table → HR. All but HR are built (2026-09-22).
