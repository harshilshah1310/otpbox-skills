---
name: otpbox
description: Give a coding or testing agent a real disposable email inbox to drive signup, OTP, magic-link, and email-verification flows end-to-end - no human relaying a code. Use this whenever a task needs to receive a real verification email (a one-time code, a confirmation link, a password-reset link) as part of testing or automating a signup/login flow.
---

# OTPBox: real inboxes for agent-driven testing

OTPBox (https://otpbox.io) gives an agent a real, disposable email address, then extracts
the one-time code or verification link from whatever arrives in it - so a signup or
login flow under test can be driven all the way through, not just up to the OTP screen.

Use this skill when a task involves: signing up for an account as part of a test or
demo, verifying an email address, testing a password-reset flow, testing 2FA/magic-link
login, or provisioning realistic test personas (name, email, country) for QA data.

Don't use it for: an agent that needs to *send* email, or needs a *persistent* mailbox
across sessions - OTPBox inboxes are disposable by design and expire automatically.

## Getting a key

Every call needs an API key as a bearer token. Mint a free one with no account and no
payment info (200 requests/month, 5 keys/day per IP):

```bash
curl -X POST https://otpbox.io/api/v1/keys/free -H "content-type: application/json" -d '{}'
# → { "id": "lic_...", "key": "...", "plan": "free", "quotaLimit": 200 }
```

Save the returned `key` - it is shown once. For deterministic CI runs with no real mail
delivery, mint a sandbox key instead (`POST /api/v1/keys/sandbox`, same shape): every
inbox it creates already contains one synthesized message with a random 6-digit `code`.

Store the key as an environment variable (`OTPBOX_KEY`) - never hardcode it or print it
in logs.

## Two ways to call it: MCP or REST

**MCP (preferred when the agent's client supports it).** A remote, streamable-HTTP MCP
server at `https://otpbox.io/mcp`, authenticated the same way as REST:

```json
{
  "mcpServers": {
    "otpbox": {
      "url": "https://otpbox.io/mcp",
      "headers": { "Authorization": "Bearer <OTPBOX_KEY>" }
    }
  }
}
```

13 tools are exposed, covering the same operations as the REST API below:
`create_test_inbox`, `wait_for_email`, `get_otp`, `get_verification_link`,
`get_latest_email`, `search_emails`, `delete_inbox`, `create_batch`,
`create_test_identity`, `create_bulk_test_identities`, `delete_test_identity`,
`register_webhook`, `get_usage`. Prefer `wait_for_email` over polling in a loop - it
blocks server-side until a message arrives or a timeout elapses (max 25s).

**REST (when no MCP client is available, or from inside test code).** Same operations
under `https://otpbox.io/api/v1`, `Authorization: Bearer <key>` on every call. Full
reference: https://otpbox.io/docs.

## The core loop

1. Create an inbox (MCP `create_test_inbox`, or `POST /api/v1/inboxes`). You get back an
   `id`, `address`, and `expiresAt`.
2. Use `address` wherever the flow under test asks for an email - a signup form, an API
   request body, a test-identity field.
3. Block for the message: MCP `wait_for_email` (blocks up to 25s), or REST poll
   `GET /api/v1/inboxes/:id/messages` if not using MCP.
4. Read the result out. `get_otp` returns the extracted one-time code, `get_verification_link`
   returns a classified confirmation/reset/magic-link URL (`url`, `host`, `type`), and
   `get_latest_email`/`get_latest_email` returns the full message (subject, text, html,
   attachments) if you need more than the extracted fields.
5. Continue the flow under test with that code/link (fill the OTP field, navigate to the
   link, etc.).
6. Clean up: `delete_inbox` when done. Inboxes also expire automatically (1 hour by
   default), so a crashed run doesn't leak state forever, but explicit cleanup keeps a
   test run self-contained.

## Test identities (realistic personas, not just addresses)

When a flow needs a plausible person, not just an email address, use
`create_test_identity` (or `POST /api/v1/identities`) with a `template`:
`us_customer`, `indian_customer`, `european_customer`, `business_customer`, `student`,
or `employee`. It returns a synthetic name, country, and (for the business/student/
employee templates) a small profile, backed by a real inbox - fill the signup form with
the generated `name`/`email`/`country`, then use the returned `id` with the same
wait/read tools as any inbox. `create_bulk_test_identities` provisions up to 50 at once
for bulk QA data generation. `delete_test_identity` removes the identity and its backing
inbox together.

## Bulk / matrix testing

`create_batch` (or `POST /api/v1/batches`) creates up to 50 inboxes in one call - useful
for provisioning a whole test matrix (e.g. one inbox per locale, per plan tier) up
front rather than one-at-a-time.

## Event-driven instead of polling

If the agent's task is long-running or event-driven rather than a single blocking test
step, `register_webhook` (or `POST /api/v1/webhooks`) subscribes an HTTPS endpoint to
events like `message.received`, `otp.extracted`, `link.detected`, `inbox.expired`, or
`identity.created` - the agent (or the system it's orchestrating) gets a signed POST
instead of holding a connection open. Full event catalog and signature verification:
https://otpbox.io/docs#webhooks.

## Gotchas

- **Quota.** Every MCP tool call and REST request counts as one request against the
  key's monthly quota (free: 200/mo). Check `get_usage` before a large batch run, and
  don't loop `get_latest_email`/`search_emails` as a substitute for `wait_for_email` -
  each poll costs a request.
- **Rate limit.** MCP is separately capped at 60 tool calls/minute per key, independent
  of the monthly quota - back off if a tool call returns a rate-limit error.
- **Timeouts.** `wait_for_email`'s timeout maxes out at 25 seconds. For a flow where the
  real-world email might take longer, call it again rather than requesting a longer
  single wait.
- **Don't reuse an inbox across test runs.** Every inbox is meant to be created fresh
  and discarded - reusing one risks a stale message from a previous run being read
  instead of the new one.
- **Sandbox keys never receive real mail** - `create_test_inbox` with a sandbox key
  synthesizes one deterministic message immediately, so `message.received`/
  `otp.extracted`/`link.detected` webhooks never fire for sandbox inboxes. Use a normal
  key for webhook-driven testing.

## Reference

Full API/MCP reference: https://otpbox.io/docs. SDKs: `otpbox-sdk` (TypeScript/Node),
`otpbox-playwright` (Playwright fixture), and a Python SDK - see the guides at
https://otpbox.io/guides for framework-specific walkthroughs (Playwright, Cypress,
Selenium, Puppeteer, Postman).
