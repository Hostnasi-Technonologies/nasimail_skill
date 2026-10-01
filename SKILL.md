---
name: nasitransact-api
description: Call the NasiTransact REST API v1 (send transactional emails, check message status/events, manage sending domains/DKIM, API keys, webhooks, SMTP credentials, contacts/lists, campaigns, email validation, blocklist monitoring, usage stats) via Bearer token or HTTP Basic Auth. Use when the user asks to send a test email through NasiTransact, check a domain's DNS/verification status, create or revoke API keys/webhooks/SMTP credentials, look up message delivery status, validate an email address, import contacts, or debug/exercise a NasiTransact API instance from any project.
argument-hint: "[send|messages|domains|keys|webhooks|smtp|stats|validate|blocklist|campaigns|lists|contacts] [args...]"
---

Drive the NasiTransact REST API v1 (JSON, standard response envelope `{success, message, data, errors}`).

## Resolve credentials and base URL first

1. Base URL: use `NASITRANSACT_BASE_URL` if set, else check the current project's `.env` for `APP_URL` if it looks like a NasiTransact instance (e.g. `https://nasitransact.test`), else ask the user. NasiTransact is self-hosted per account/instance — there is no universal default host.
2. Credentials — either works:
   - Bearer: `NASITRANSACT_SECRET_KEY` (looks like `sk_live_...`) → `Authorization: Bearer $SECRET_KEY`.
   - Basic: `NASITRANSACT_PUBLIC_KEY:NASITRANSACT_SECRET_KEY` (`pk_live_...:sk_live_...`) → `curl -u "$PUBLIC_KEY:$SECRET_KEY"`.
   Keys are generated inside the NasiTransact web app, or via `POST /api/v1/api-keys` with an existing key. Never invent or guess credentials — ask the user if none of the above are available.
3. All paths are `{base_url}/api/v1/...`. Always send `Accept: application/json`.
4. An API key can be domain-scoped (`sending_domain_id` set at creation). Sending/creating against any other domain returns `422`.

## Messages

### POST /api/v1/messages — send a transactional email
```bash
curl -s -X POST "$BASE_URL/api/v1/messages" \
  -H "Authorization: Bearer $SECRET_KEY" \
  -H "Content-Type: application/json" -H "Accept: application/json" \
  -H "Idempotency-Key: send-20260606-0001" \
  -d '{
    "from": "hello@yourdomain.com",
    "from_name": "Your Company",
    "to": ["user@example.com"],
    "cc": ["cc@example.com"],
    "bcc": ["bcc@example.com"],
    "reply_to": ["reply@example.com"],
    "subject": "Welcome aboard!",
    "html": "<h1>Welcome!</h1><p>Thanks for signing up.</p>",
    "text": "Welcome! Thanks for signing up.",
    "headers": {"X-Custom-Header": "custom-value"},
    "tags": ["welcome", "transactional"],
    "metadata": {"user_id": "123", "campaign": "onboarding"},
    "track": true
  }'
```
Required: `from` (must match an active, verified sending domain on the account), `to` (array of emails), `subject` (max 998 chars), and at least one of `html`/`text`. `track` defaults to `true`; send `false` for pure transactional mail (password resets, invites) to suppress the open-tracking pixel and `List-Unsubscribe` headers.

Success (`202`): `{"success": true, "message": "Message queued", "data": {"id": 1, "message_id": "<uuid@yourdomain.com>", "status": "queued", "queued_at": "..."}}`.

Attachments: send `multipart/form-data` instead of JSON, with `attachments[]` file fields (max 25MB each per file).

**Idempotency-Key** header (optional, ≤128 chars, scoped per authenticated user):
- Same key + identical payload → `200` with `"message": "Idempotent replay"` and the original message data.
- Same key + different payload → `409` with an `idempotency_key` error.

**Pre-send recipient validation**: recipients are screened by the email validation pipeline before sending; `undeliverable` addresses are blocked by default (`422` with recipient + reason), `risky` addresses are allowed unless the account has `NASITRANSACT_VALIDATE_BLOCK_RISKY=true`.

Status flow: `queued → sending → sent → delivered`, branching to `failed` or `bounced`.

**Before sending, confirm with the user**: this delivers a real email through a verified sending domain. Never send to an address the user hasn't explicitly provided, and don't send bulk/marketing-style content without explicit confirmation of the recipient list.

### GET /api/v1/messages — list (paginated, 50/page)
Query params: `status`, `tag`.
```bash
curl -s -H "Authorization: Bearer $SECRET_KEY" -H "Accept: application/json" \
  "$BASE_URL/api/v1/messages?status=delivered"
```

### GET /api/v1/messages/{id} — show one message
### GET /api/v1/messages/{id}/events — delivery event timeline for one message
### POST /api/v1/messages/{id}/resend — resend a failed message
Only non-`delivered` messages can be resent; `delivered` returns `422`.

## Domains

### GET /api/v1/domains — list sending domains
### POST /api/v1/domains — add a domain
```bash
curl -s -X POST "$BASE_URL/api/v1/domains" \
  -H "Authorization: Bearer $SECRET_KEY" -H "Content-Type: application/json" -H "Accept: application/json" \
  -d '{"domain": "yourdomain.com"}'
```
Generates DKIM keys and returns `{"domain": {...}, "dns_records": [...]}` with status `pending_verification`. The user must add the returned DNS records before the domain can send.

### GET /api/v1/domains/{id}/dns-records — fetch the required DNS/DKIM records again
### GET /api/v1/domains/{id}/verify — queue a DNS verification check (async; poll `dns-records` or the domain list for status)

## API Keys

### GET /api/v1/api-keys — list (secret never included)
### POST /api/v1/api-keys — create
```bash
curl -s -X POST "$BASE_URL/api/v1/api-keys" \
  -H "Authorization: Bearer $SECRET_KEY" -H "Content-Type: application/json" -H "Accept: application/json" \
  -d '{"name": "CI key", "sending_domain_id": 42, "expires_at": "2027-01-01T00:00:00Z"}'
```
`sending_domain_id` and `expires_at` are optional. Response includes `secret_key` in plain text **once only** — tell the user to save it immediately, it cannot be retrieved again.

### DELETE /api/v1/api-keys/{id} — revoke (soft: sets `is_active=false`, returns `401` on future use)
**Confirm with the user before revoking** — this immediately breaks anything using that key.

## Webhooks

Valid events: `sent`, `delivered`, `opened`, `clicked`, `bounced`, `complained`, `failed`.

### GET /api/v1/webhooks — list
### POST /api/v1/webhooks — create
```bash
curl -s -X POST "$BASE_URL/api/v1/webhooks" \
  -H "Authorization: Bearer $SECRET_KEY" -H "Content-Type: application/json" -H "Accept: application/json" \
  -d '{"url": "https://yourapp.com/webhooks/nasitransact", "events": ["delivered", "bounced", "complained"]}'
```
Response includes a signing `secret` once; use it to verify webhook payload signatures. Listing omits the secret.

### DELETE /api/v1/webhooks/{id} — delete (confirm with user first — stops delivery notifications to that endpoint)

## SMTP Credentials

### GET /api/v1/smtp-credentials — list (password never included)
### POST /api/v1/smtp-credentials — create
```bash
curl -s -X POST "$BASE_URL/api/v1/smtp-credentials" \
  -H "Authorization: Bearer $SECRET_KEY" -H "Content-Type: application/json" -H "Accept: application/json" \
  -d '{"sending_domain_id": 42, "host": "smtp.yourdomain.com", "port": 587, "encryption": "tls"}'
```
All fields optional (username auto-generated if omitted). Response includes the plain-text `password` **once only** — tell the user to save it immediately.

### DELETE /api/v1/smtp-credentials/{id} — deactivate (confirm first — breaks anything sending via that SMTP login)

## Stats

### GET /api/v1/stats/summary — lifetime counts (`total`, `sent`, `delivered`, `opened`, `clicked`, `bounced`, `complained`, `failed`)
### GET /api/v1/stats/timeline — daily usage log; query params `start_date`/`end_date` (`YYYY-MM-DD`, defaults to the last 30 days)

Use `stats/summary` to sanity-check credentials/activity, and before a send to gauge how close the account is to its plan limit (see rate limits below).

## Email Validation

### GET or POST /api/v1/email/validate — single address
Params: `address` (required), `fresh` (bool, bypass cache).
```bash
curl -s -H "Authorization: Bearer $SECRET_KEY" -H "Accept: application/json" \
  "$BASE_URL/api/v1/email/validate?address=user@example.com"
```
Returns a verdict object with `result` (`deliverable`/`risky`/`undeliverable`/`unknown`), `risk` score, and `reason[]`.

### POST /api/v1/email/validate/bulk — up to 50 addresses per request
```bash
curl -s -X POST "$BASE_URL/api/v1/email/validate/bulk" \
  -H "Authorization: Bearer $SECRET_KEY" -H "Content-Type: application/json" -H "Accept: application/json" \
  -d '{"addresses": ["a@example.com", "b@example.com"]}'
```
Returns `{"results": [...], "summary": {"deliverable": N, "risky": N, "undeliverable": N, "unknown": N}, "total": N}`.

## Blocklist Monitoring

### POST /api/v1/blocklists/check — one-off DNSBL snapshot (no persistence)
`{"target": "1.2.3.4" | "yourdomain.com", "type": "ip"|"domain"}` — `type` auto-detected from `target` if omitted.
### GET /api/v1/blocklists/targets — list persisted monitored targets (paginated)
### POST /api/v1/blocklists/targets — add/update a monitored target (`target_type`, `target_value`, `label`, `is_active`)
### POST /api/v1/blocklists/monitor — trigger an immediate monitoring run (`target_ids[]` optional = all of the user's targets, `alerts` bool)

## Campaigns, Lists, Contacts

### POST /api/v1/lists — create a contact list (`name`, `description`, `is_active`)
### GET /api/v1/lists — list (paginated)
### POST /api/v1/lists/{id}/contacts — add contacts (`contact_ids: [int, ...]`)
### DELETE /api/v1/lists/{id}/contacts — remove contacts (`contact_ids: [int, ...]`)

### POST /api/v1/contacts/import — bulk upsert contacts by email
```bash
curl -s -X POST "$BASE_URL/api/v1/contacts/import" \
  -H "Authorization: Bearer $SECRET_KEY" -H "Content-Type: application/json" -H "Accept: application/json" \
  -d '{
    "contacts": [
      {"email": "a@example.com", "first_name": "A", "metadata": {"plan": "pro"}},
      {"email": "b@example.com"}
    ],
    "list_id": 7
  }'
```
Matches on `email` per account; existing contacts are updated, not duplicated. Returns `{"created": N, "updated": N, "total": N}`.

### POST /api/v1/campaigns — create a campaign
Requires an existing `template_id` (templates are created in the web app, not via this API) and a verified `sending_domain_id` whose domain matches `from_email`. `delivery_mode`: `draft` (default), `schedule` (needs `scheduled_at` + optional `timezone`, must be in the future), or `send_now`.
```bash
curl -s -X POST "$BASE_URL/api/v1/campaigns" \
  -H "Authorization: Bearer $SECRET_KEY" -H "Content-Type: application/json" -H "Accept: application/json" \
  -d '{
    "name": "October Newsletter",
    "subject": "What is new this month",
    "sending_domain_id": 42,
    "template_id": 5,
    "list_id": 7,
    "from_name": "Your Company",
    "from_email": "news@yourdomain.com"
  }'
```
### GET /api/v1/campaigns — list · GET /api/v1/campaigns/{id} — show
### POST /api/v1/campaigns/{id}/send — queue the campaign for immediate sending
**Always confirm the recipient list and content with the user before calling this** — it sends to every subscribed contact on the campaign's list (or the whole account if no `list_id`) and cannot be undone once dispatched. Returns `422` if already `sending`/`sent` or if there are no subscribed recipients.
### GET /api/v1/campaigns/{id}/stats — `total_recipients`, `opened`, `clicked`, `failed`, `open_rate`, `click_rate`

## Rate limits & plan limits

Monthly/daily send caps are plan-based (Free 500/50, Starter 10k/500, Pro 50k/2k, Enterprise unlimited). Exceeding the monthly cap returns `429` with `"message": "Monthly email limit reached."` on `POST /api/v1/messages`. Separately, each endpoint is throttled per API key (read/write/send buckets); throttled responses are `429` with `X-RateLimit-Limit`, `X-RateLimit-Remaining`, and `Retry-After` headers — back off and retry after that many seconds.

## Error responses to expect

All errors use the envelope `{"success": false, "message": "...", "data": null, "errors": {...}}`.

| Code | Meaning |
|---|---|
| 401 | Missing, invalid, inactive, or expired API key |
| 403 | Key exists but isn't authorized for this action |
| 404 | Resource not found, or belongs to another account |
| 409 | Idempotency key reused with a different payload (`/messages` only) |
| 422 | Validation error or business rule violation (e.g. unverified `from` domain, key scoped to a different domain, undeliverable recipient blocked) |
| 429 | Plan send limit or per-key rate limit exceeded |
| 500 | Internal server error |

On any non-2xx, print the response body — `message`/`errors` carry the actual detail — rather than just the status code.
