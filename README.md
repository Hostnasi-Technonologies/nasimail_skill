# nasitransact-api

A Claude Code [skill](https://docs.claude.com/en/docs/claude-code/skills) that teaches Claude how to drive the **NasiTransact** REST API v1 — a self-hosted transactional email platform — directly from the terminal via `curl`.

With this skill installed, Claude can send transactional emails, check delivery status, manage sending domains/DKIM, API keys, webhooks, SMTP credentials, contacts/lists, campaigns, run email validation, and monitor blocklists, all without you having to hand-write the requests yourself.

## What's in here

- **[SKILL.md](SKILL.md)** — the skill definition: metadata (`name`, `description`, `argument-hint`) plus the full operating instructions Claude follows, including request/response shapes, required fields, and behavioral rules (e.g. always confirming before a real send).

## Installation

Drop this directory into your Claude Code skills path (project-level `.claude/skills/` or your global `~/.claude/skills/`) under a `nasitransact-api` folder, or symlink it there. Claude Code picks up skills automatically — no build step required.

## Usage

Just ask, in plain language, from any project. Claude loads this skill automatically when the request matches, e.g.:

- "Send a test email through NasiTransact to me@example.com"
- "Check the DNS/verification status of yourdomain.com on NasiTransact"
- "Create a new NasiTransact API key scoped to domain 42"
- "Did message abc-123 get delivered?"
- "Validate this list of email addresses"
- "Check if 1.2.3.4 is on any blocklists"

You can also invoke it explicitly as a slash command:

```
/nasitransact-api send ...
/nasitransact-api messages ...
/nasitransact-api domains ...
/nasitransact-api keys ...
/nasitransact-api webhooks ...
/nasitransact-api smtp ...
/nasitransact-api stats ...
/nasitransact-api validate ...
/nasitransact-api blocklist ...
/nasitransact-api campaigns ...
/nasitransact-api lists ...
/nasitransact-api contacts ...
```

## Setup / credentials

NasiTransact is **self-hosted per account** — there's no universal default host, so Claude needs to resolve a base URL and credentials before making any call:

| What | How it's resolved |
|---|---|
| Base URL | `NASITRANSACT_BASE_URL` env var, or `APP_URL` in the current project's `.env` if it looks like a NasiTransact instance, or asked from you |
| Auth (Bearer) | `NASITRANSACT_SECRET_KEY` (`sk_live_...`) → `Authorization: Bearer $SECRET_KEY` |
| Auth (Basic) | `NASITRANSACT_PUBLIC_KEY:NASITRANSACT_SECRET_KEY` (`pk_live_...:sk_live_...`) |

Keys are generated in the NasiTransact web app, or via `POST /api/v1/api-keys` using an existing key. Claude will never invent credentials — it asks if none are available.

## Capabilities covered

- **Messages** — send transactional email (JSON or multipart with attachments), list/show messages, view delivery event timelines, resend failed messages, idempotency-key support
- **Domains** — add sending domains, fetch DKIM/DNS records, trigger verification
- **API Keys** — create/list/revoke, with optional domain scoping and expiry
- **Webhooks** — subscribe to delivery events (`sent`, `delivered`, `opened`, `clicked`, `bounced`, `complained`, `failed`)
- **SMTP Credentials** — create/list/deactivate SMTP logins per sending domain
- **Stats** — lifetime summary and daily usage timeline
- **Email Validation** — single and bulk (up to 50) address verdicts
- **Blocklist Monitoring** — one-off DNSBL checks and persisted monitored targets
- **Campaigns, Lists, Contacts** — manage contact lists, bulk-import contacts, create/send/track email campaigns

## Safety behaviors baked into the skill

The skill instructs Claude to pause and confirm with you before any action with real-world or irreversible consequences:
- Sending an actual email (especially to addresses you haven't explicitly provided, or bulk/marketing content)
- Revoking an API key or SMTP credential (breaks anything currently using it)
- Deleting a webhook (stops delivery notifications)
- Sending a campaign (dispatches to every subscribed contact and can't be undone)

It also knows the full error envelope (`{success, message, data, errors}`), the status/HTTP code table for common failures (401/403/404/409/422/429/500), and plan-based rate limits, so it can explain failures instead of just reporting a status code.
