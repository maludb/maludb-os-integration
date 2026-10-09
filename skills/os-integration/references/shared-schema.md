# The estate's data model — reuse a table before inventing one *(0.7.0, 2026-10-05)*

Every application keeps its own database; that rule stands. But the applications are built on one server, by the
same people, with every sibling's schema in view — and by October 2026 the same concepts had been given a different
table in each: `attachments` in four shapes across four applications, `notification_outbox` in four, `time_entries`
meaning hours on an engagement in one application and minutes on a ticket in another. From 0.7.0 the owner's rule is:
**the estate has one data model, expressed in many databases.** A new application looks at the tables the other
applications already have and uses the same table — same name, same columns — whenever an existing table is the
concept, or close to it. A new table is the last resort, and it becomes the canonical one for the next application.

## The rule in one paragraph

Before any `CREATE TABLE`, find the closest table any sibling application already has. If another application
**owns that data**, do not copy its table — read it through the kernel (K7 `reads[]`, the directory mirror). If your
application needs **its own rows of the same concept**, copy the canonical definition verbatim into your own migration
and extend it only by appending. Only when nothing is close do you design a new table — and then yours is the
canonical definition. **Reuse is structural, never a dependency**: the definition lives in your own `db/*.sql`, the
kernel's installer creates the table in your own database, and the application installs and runs on a server where
the application you copied from was never installed.

## 1. Survey the estate first — the whole schema at once

Where the schemas are: on the build server every application's clone is at `/srv/apps/<key>/db/*.sql`; anywhere
else, the repositories `github.com/maludb/maludb-os-<key>` (the "Repositories" table in `maludb-os-core`'s README is
the list). Two commands answer most of it:

```bash
# every table in the estate: name, application, file — sorted so recurring names sit together
for d in /srv/apps/*/; do k=$(basename "$d"); grep -HoiE '^CREATE TABLE (IF NOT EXISTS )?[a-z_]+' "$d"db/*.sql \
  | sed -E "s|^([^:]+):CREATE TABLE (IF NOT EXISTS )?([a-z_]+)|\3  $k  \1|I"; done | sort

# one table's definition in every application that has it
t=attachments; for f in /srv/apps/*/db/*.sql; do grep -qiE "^CREATE TABLE (IF NOT EXISTS )?$t\b" "$f" && { echo "-- $f"; \
  awk -v t="$t" 'BEGIN{IGNORECASE=1} $0 ~ "^CREATE TABLE (IF NOT EXISTS )?"t"[ (]"{p=1} p{print} p&&/\);/{p=0}' "$f"; }; done
```

Do this in Phase 1 for the entire planned schema, not table by table while coding: a model designed against the
estate is a different thing from one patched to match it afterwards. Read the sibling's design document too
(`docs/<key>-design.md`) — it says *why* the table has the shape it has.

## 2. Three outcomes for every table — decide, then record

| The closest existing table is… | Do | Record in the design as |
|---|---|---|
| **Another application's data** — the rows are theirs: HR's employment, the ledger's chart, parties and books, Projects' issues, Help Desk's tickets, Consultant Tracking's engagements and time, the kernel's members, departments and scopes | Do not copy the table. Hold at most the provider's id and the display fields you need, keyed by the provider's id, refreshed from a K7 read ([sms-and-reads.md](sms-and-reads.md), `reads[]`) or from the directory feed. When the provider is not installed or the connection is not approved, the application degrades (a person types the value) — it never fails to install or to run | `read: <app>.<tool>` |
| **The same concept, and the rows are yours** — attachments, notes, notifications and the outbox, document sequences, tax rates, billing documents … | Copy the canonical definition **verbatim**: table name, column names, types, defaults, CHECK constraints, index names. Extend only by **appending** a column, widening an enumeration (`record_type`, `kind`) or adding an index. One substitution is allowed: the external-party column takes the application's own word for its outsider (`customer_id`, `client_id`, `requester_id`) | `reuse: <app>/db/NNN_x.sql` and, if any, `appended: <columns>` |
| **Nothing close** | Design it. It is now the canonical table for that concept; the next application copies yours | `new`, with one line saying what was compared and why nothing fit |

"Close" means a person who knows the existing table would recognise yours as the same thing: the same concept, the
same key shape (a polymorphic `record_type`/`record_id` pair, a `member_id`, a `status` enumeration) and most columns
in common. Judge on the concept, not the name: a different concept never takes a shared name (minutes logged on a
ticket is not `time_entries` when `time_entries` already means hours on an engagement — it is `ticket_time_entries`,
or it reuses the engagement shape). A copy with a renamed or re-typed column is a defect even when the code works;
the worker's escalation rule applies — stop, write the question down, escalate. When a copy genuinely needs a
different shape, change the canonical definition first (append there, in that application's repository, as an owed
item if it is not yours to build) and copy the result.

The same table gets the same `mcp_<table>` view and, where the question is the same, the same tool name on the
records server — an agent that learned `attachments` in one application knows it in the next.

## 3. What the installer does — nothing new, and that is the point

The installer runs *your* `db/*.sql` in order into *your* database. A table you reused is created by your own
migration, whether or not the application it came from is installed on that server, because installations differ
in which applications they hold. So, in a migration: never `\i` or `\include` another application's file, never
`REFERENCES` a table in another database, never read `/srv/apps/<other>` at install or at run time, never make a
`database.provision` script look for a sibling. The only cross-application dependency allowed is a K7 read, and its
absence is the `no_connection` answer the application already handles.

## 4. The catalogue (as of 2026-10-09)

*Canonical* is the definition to copy. Where copies had already diverged before this rule, the canonical is the most
recent planning-class design (the General Ledger and Consultant Tracking, both 2026-10-05, built with the others in
view) unless the owner names another; the divergent copies are listed so they converge when next touched — by a
migration that appends the canonical columns and keeps the old ones until the code has moved.

**The kernel contract** — identical in every application; copy from the reference, not from a sibling:

| Tables | Canonical definition |
|---|---|
| `members`, `department_members` | [sign-on-and-directory.md](sign-on-and-directory.md) (`members.roles text[]` from roles-and-rights.md) |
| `sso_nonces`, `member_sessions`, `directory_sync_state` | [php-sign-on-kit.md](php-sign-on-kit.md) |
| `activity_log`, `activity_ingest_state` | [memory.md](memory.md) |
| `os_scopes`, `member_scope_roles` (a scoped application) | [scoped-applications.md](scoped-applications.md) |
| `mcp_access_tokens` | identical in six applications; the origin is HR `db/003_access_tokens.sql` |

**Supporting tables** — the same concept kept by several applications:

| Concept | Canonical | Also in | Note |
|---|---|---|---|
| A file attached to any record | `attachments` — GL `db/009_parties.sql`: `record_type`/`record_id`, `filename`, `mime_type`, `byte_size`, `sha256`, `storage_path`, `uploaded_by` | Consultant Tracking `db/006` (canonical + `client_visible`); Help Desk `db/010` (direct FKs, two uploader columns); Projects `db/009` (`file_name`, `size_bytes`, `stored_path`); **Spaces `db/012`** (canonical + `record_uuid`, `width`, `height`, `thumbnail_path`) | widen the `record_type` CHECK per application; **an application whose records are UUID-keyed appends `record_uuid uuid` and makes `record_id` nullable with `CHECK ((record_id IS NULL) <> (record_uuid IS NULL))` — the one agreed loosening (Spaces, 2026-10-05)**; Help Desk and Projects converge when next touched |
| A note on any record | `notes` — GL `db/009` = Consultant Tracking `db/006` | — | identical already |
| In-app notifications, a person's preferences, the outbox the worker sends | `notifications`, `notification_prefs`, `notification_outbox` — GL `db/014_recurring_notifications.sql` | Consultant Tracking `db/013` (`client_id` for `customer_id`); Help Desk `db/013` (`ticket_id` for `record_type`/`record_id`, prefs `+digest`); txtSchedules `db/012` (own shape: `by_email`/`by_sms`, `not_before`, `error`); **Spaces `db/012`** (canonical; no external-party column since every recipient is a member; `notifications` appends `record_uuid`, `channel_id`, `message_id`, `actor_member_id`; prefs append `digest`, `away_minutes`) | the `kind` list is the application's own; an application with no outside recipient drops the external-party column and its pair CHECK |
| An agent asked to act | `agent_dispatches` — GL `db/014` = Consultant Tracking `db/013` | Help Desk `db/013` (`ticket_id`, no `attempts`); **Spaces `db/012`** (canonical + `record_uuid`, `channel_id`, `conversation_id`, `reply_message_id`, `pending_message_id`, `next_attempt_at`; `kind` mention/dm/duty_proposal/ask) | |
| Numbered documents, tax rates | `document_sequences`, `tax_rates` — GL `db/006_settings_sequences.sql` | Consultant Tracking `db/005` | identical but for the `kind` list |
| Billing documents | `invoices`, `invoice_lines`, `credit_notes`, `credit_note_lines`, `credit_applications`, `invoice_links_secure`, `invoice_payments` — GL `db/010_receivables.sql` | Consultant Tracking `db/012` (it bills and never keeps books — the owner's D16); Reservations (adopted, predates the rule) | |
| The kernel's AI statement, imported | `ai_statements`, `ai_statement_lines` — GL `db/012_ai_statements.sql` | Consultant Tracking `db/011` | |
| The mirror of the kernel's site scopes | `sites` — txtSchedules `db/001` (`scope_id` PK, `location_id`, `timezone`) | GL `db/001` (own `id`, `kernel_location_id`) | a scoped application mirrors scopes; txtSchedules' is the scoped shape |
| The application's one settings row | `app_settings` — the skeleton `id smallint PRIMARY KEY DEFAULT 1 CHECK (id = 1)` … `updated_at` | Help Desk `db/005`, txtSchedules `db/005` | the skeleton is shared, the columns are yours |
| Time worked | `time_entries` — Consultant Tracking `db/009_time.sql` (hours on an engagement, under a timesheet) | Help Desk `db/011` is a different concept under the same name (minutes on a ticket) | a name collision, recorded; the next application that logs time on its own records does not reuse the name for the other meaning |
| A comment on one of my records | `comments` — two shapes under one family: Projects `db/009` (bigint, on an issue, Markdown body) and Spaces `db/011` (UUID, on a page or a block, the one rich-text body, threaded, resolved) | — | the family (record key + author + body + edited_at + deleted_at) is shared; the key type follows the application's records |
| A message | `messages` — two shapes under one family: Help Desk `db/010` (a ticket's correspondence) and Spaces `db/010` (a channel message with a one-level thread) | — | recorded as siblings, not a reuse |
| A place data is read from under a policy, its pulls and its credential | `sources`, `source_pulls`, `source_credentials` — **Inventory `db/009`** (connector, settings, schedule, rate, user-agent, robots state, back-off, pause; a pull with counts and the policy followed; a credential as libsodium ciphertext with last4) | **Shareholder Intelligence `db/009`** (verbatim; `connector` = its own list, `role` widened with public_data/market_data/licensed/manual, `supplier_id` kept nullable with no FK — the `income_account_id` precedent) | canonical from 2026-10-05; Knowledge's `sources` is a DOCUMENT INGESTED — a name collision of a different concept, recorded; the next application that reads a public host takes Inventory's |
| A cross-entity search index | `search_index` — **Spaces `db/013`** (entity_kind, entity_uuid/entity_id, tsvector, has_link/has_file, occurred_at) | — | canonical from 2026-10-05 |
| Pages, blocks and the permission tree; channels and conversations | `pages`, `page_permissions`, `page_links`, `page_versions`, `page_publications`, `blocks`; `channels`, `channel_members`, `dm_pairs`, `message_mentions`, `message_reactions`, `saved_messages`, `reminders` — **Spaces `db/007`, `db/008`, `db/010`** | — | canonical from 2026-10-05 (the first application designed under this rule) |
| A company at any point of a relationship; a person who may belong to no company; every address a person uses | `accounts`, `contacts`, `contact_emails` — **Sales CRM `db/007`** (accounts: name, legal_name, domain, kind prospect/customer/partner/former, owner_member_id, restricted_to_*, merged_into_id, custom_values, deleted_at; contacts: first/last/name generated, email, email_alt, role, is_primary, lifecycle derived, opt-outs, owner, merged_into_id) | — | canonical from 2026-10-09; the GL's `customers` stays the billing party (read through K7 once a deal is won — the shared column names name, legal_name, email, phone, billing_address, shipping_address, currency are kept); Help Desk's `organizations`/`requesters` stay support's view (read) |
| A dated touch with an owner, a state, an outcome and a reminder, on any record | `activities`, `activity_participants` — **Sales CRM `db/010`** (kind call/email/meeting/task/deadline/lunch, subject, body, the four record keys, owner_member_id, due_at or starts_at/ends_at, done_at, outcome, remind_minutes_before, remind_at, reminded_at, snoozed_until, email_direction, is_private, priority, recurrence, source) | — | canonical from 2026-10-09; Spaces' `reminders` is a note to self, Help Desk's `time_entries` and Projects' `worklogs` are time — none is a touch |
| A sales pipeline: stages as columns, deals with history, products on a deal | `pipelines`, `stages`, `deals`, `deal_stage_history`, `deal_contacts`, `deal_products`, `lost_reasons`, `quotas` — **Sales CRM `db/008`**, `db/006` | — | canonical from 2026-10-09; a stage IS a column (no mapping table, unlike Projects' `board_columns`); `deal_products` lines by name with an optional provider id (`inventory`/`gl`) — no catalog of the CRM's own |
| A lead and its routing | `leads`, `lead_sources`, `assignment_rules` — **Sales CRM `db/009`**, `db/006` | — | canonical from 2026-10-09 |
| A listed company, its share classes, a dated share count, a daily price | `issuers`, `issuer_members`, `securities`, `share_counts`, `prices` — **Shareholder Intelligence `db/006`**, `db/011` (an issuer with CIK, ticker, exchange, transfer agent; a security per CUSIP, one primary; a share count by kind — outstanding, float, cede, drs, certificated, plan — as of a date from a source; OHLCV per security per trade date) | — | canonical from 2026-10-09; nothing in the estate was a listed company with CUSIPs or a price series |
| A DTC participant and its daily position | `participants`, `participant_classes`, `participant_positions` — **Shareholder Intelligence `db/006`**, `db/007` (the four-character number, the directory's name and aliases, a class that is a setting; one row per participant per position date, absent ≠ zero) | — | canonical from 2026-10-09 |
| A holder on a list, their accounts and positions, the matcher's proposals | `holders`, `holder_accounts`, `holder_positions`, `holder_match_proposals` — **Shareholder Intelligence `db/008`** (a normalised person or entity with a name key and an address key; an account per registration with its custodian participant and kind; shares at a record date in a channel; a scored proposal accepted automatically at a threshold or decided by a person) | — | canonical from 2026-10-09; compared with the CRM's `contacts` (a relationship) and Help Desk's `requesters` — a holder is a registration on a list; `merged_into_id` from `requesters` |
| An EDGAR filing and its text; a 13F/13D/13G filer; the ownership forms | `filings`, `filing_text`, `institutions`, `institutional_holdings`, `beneficial_ownership`, `insider_transactions`, `form144_notices` — **Shareholder Intelligence `db/009`**, `db/010` (the submissions JSON's columns; the SEC data set's and XML's field names kept: shares, shares_type, put_call, discretion, voting_sole/shared/none, transaction_code, shares_owned_after, ownership) | — | canonical from 2026-10-09; `filing_text` takes Knowledge's `source_text` shape (`source_id` → `filing_id`) |
| The short side of a stock | `short_interest`, `daily_short_volume`, `fails_to_deliver`, `threshold_listings`, `borrow_observations`, `form_sho_aggregates`, `securities_lending_events` — **Shareholder Intelligence `db/011`** (FINRA's and the SEC's field names; a revision a new row by observed_at) | — | canonical from 2026-10-09; the last two are the 2028 shapes, created empty |
| A dated reconciliation of a count against its components | `share_reconciliations` — **Shareholder Intelligence `db/012`** | the GL's `bank_reconciliations` is the same idea over money | recorded 2026-10-09 as two shapes under one family |
| An event on a day; an instrument that dilutes; a rule with an evidence class and the signal it raised | `events`, `dilution_instruments`, `watch_hits`, `signal_rules`, `signal_rule_overrides`, `signals`, `signal_outcomes` — **Shareholder Intelligence `db/009`**, `db/013` | Inventory's `watches` is a threshold on one variant without a class or a window — a different concept | canonical from 2026-10-09 |
| A pack approved by a manager; a saved column mapping for a tabular upload | `reports`, `import_templates` — **Shareholder Intelligence `db/014`** | Spaces' `exports` is a file, not an approved document | canonical from 2026-10-09 |
| Campaigns, lists and segments, import and export jobs | `campaigns`, `campaign_members`, `lists`, `list_members`, `import_templates`, `import_rows`, `email_templates` — **Sales CRM `db/011`, `db/012`** | — | canonical from 2026-10-09; `imports`/`exports` stay Spaces' (reused with the CRM's targets appended) |
| A tag on any record | `record_tags` (record_type + record_id + tag_id) — DocCloud `db/009` (as `document_tags`) = **Sales CRM `db/006`** | Help Desk `ticket_tags` (one record kind) | the polymorphic shape is the one to copy from 2026-10-09 |
| Email in and out through a mailbox | `mailboxes`, `inbound_email`, `outbound_email` — Help Desk `db/012` | **Sales CRM `db/011`** (verbatim; `queue_id`, `ticket_id`, `message_ref_id` kept nullable and unused; appended `sender_member_id`, `deal_id`, `match_status`, `matched_contact_ids`, `is_private`; outbound appends the body, `message_id` for threading, `template_id`) | the second application to keep a mailbox (2026-10-09) |

**Owned data — read, never copied:** employment (HR), the chart of accounts, customers, vendors and the books
(GL), the relationship — accounts, contacts, leads, deals, activities (Sales CRM, through its five shares),
(GL), issues (Projects), tickets and requesters (Help Desk), engagements and time (Consultant Tracking), covers
(Reservations), shifts (txtSchedules), the directory and scopes (the kernel).

A new canonical table, or a new divergence found, is added to this catalogue in the same change; the catalogue is
part of the contract, and the next application's Phase 1 reads it before it reads the server.

## Checklist

- [ ] Phase 1 began with the survey of every sibling's `db/*.sql`, and the design document records every table as `read`, `reuse` (with the source file and any appended columns) or `new` (with why).
- [ ] Every reused table is the canonical definition verbatim; the only differences are appended columns, a widened enumeration, an added index, or the external-party column's name.
- [ ] No migration or provision script refers to another application's files, database or install path; a K7 read degrades on `no_connection`.
- [ ] The same table has the same `mcp_<table>` view shape and the same tool name as in its canonical application.
- [ ] A new canonical table or a newly found divergence was added to the catalogue above.
