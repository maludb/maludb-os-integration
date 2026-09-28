# Kernel services: texting a member (K6) and reading another application (K7)

*Built on the kernel 2026-09-28 (db/161; kernel spec `docs/build-specs/kernel-app-services.md`; proof
`bin/test_app_services.php`, 27 checks).* Both are on the kernel's internal port (`OS_INTERNAL_URL`) with the
application's token (`OS_APPLICATION_TOKEN`), like the directory API.

## K6 — text a member

An application never holds a Twilio key and never sees a phone number. It asks the kernel:

```
POST {OS_INTERNAL_URL}/api/v1/notify/sms.php   {"member_id": 26, "text": "…", "reference": "exchange:412"}
→ 202 {"notification": {"id": 91, "status": "queued"}}
GET  {OS_INTERNAL_URL}/api/v1/notify/sms.php?id=91   → {"notification": {"id", "status": "queued|sent|failed", "sent_at", "error"}}
```

| Refusal | Means | Do |
|---|---|---|
| 503 `no_sender` | the business has no notification number (`bin/notify_endpoint_set.php`) | send it by email |
| 422 `not_held` | the member holds no grant on this application | nothing to send |
| 422 `no_verified_phone` | the member never verified a phone in the OS (`/settings/channels`) | email; the screen may say where to add a phone |
| 422 `opted_out` | they turned this application's texts off, or the carrier's STOP | email |
| 422 `rate_limited` | 30 a member a day, 2,000 a day an application | email |
| 422 `invalid` | empty, or over 480 characters | shorten |

- The kernel puts the application's name in front (`txtSchedules: …`), sends from the business's number through its
  channels worker (five attempts), and logs `application.notify` **without the text**.
- Put no other person's details in a text — the facts of the member's own shift and a link.
- Keep the application's own preferences (which events a person wants texted); the kernel keeps the opt-out.

## K7 — read another application's shared tool

A **provider** declares what it shares; a **consumer** declares what it wants to read; a super-admin approves each
connection (`bin/app_connection.php list|approve|revoke`, logged `application_connection.approve|revoke`).

```json
"shares": [ { "tool": "covers_by_service", "description": "Booked covers per date and service at one restaurant", "scoped": true } ],
"reads":  [ { "app": "reservations", "tool": "covers_by_service", "why": "Expected covers for the staffing forecast" } ]
```

```
POST {OS_INTERNAL_URL}/api/v1/apps/read.php
     {"provider": "reservations", "tool": "covers_by_service", "arguments": {"from": "…", "to": "…"}, "location_id": 22}
→ 200 {"result": {…the provider's answer…}, "provider": "reservations", "tool": "covers_by_service"}
→ 403 not_shared | no_connection | not_at_location   422 invalid   502 provider_failed
```

**Provider.** Admit the kernel's own token (roles-and-rights.md §2) to `app_roles` **and the tools in your
`shares[]`** — nothing else. A `scoped` share receives the argument `scope_id`: **your own** scope id for the site the
consumer named, set by the kernel (any `scope_id` the consumer sent is replaced). Answer about the site, never about a
person: no member identity crosses. Keep answers small (the kernel refuses more than 256 kB).

**Consumer.** Name the kernel `location_id` of a site both applications serve (your mirror of the scope carries it).
Treat `no_connection` as "not approved yet" and degrade (e.g. let a manager type the forecast). Nothing writes across:
a consumer that must change another application asks a person.

The installer records `shares[]` in `application_shares` (a share dropped from the list is withdrawn) and proposes a
connection for each `reads[]` whose provider is installed; the rest wait until it is.

## Checklist
- [ ] Texts go through `/api/v1/notify/sms.php`; every refusal falls back to email; no Twilio key in `config/.env`.
- [ ] `shares[]` lists exactly the tools the kernel token may call besides `app_roles`; each answers about a site, not a person.
- [ ] `reads[]` names each tool read from another application, with `why`; `no_connection` degrades gracefully.
