---
name: nomupay-webhook-handler
description: >
  Scaffold a compliant Nomupay webhook endpoint: signature verification, 30-second
  acknowledgement, idempotency, out-of-order handling, retry awareness, and tracking
  URL fallback. Language-agnostic. Prerequisites: nomupay-jws-setup, nomupay-verify-response.
  Trigger: 'nomupay webhook' or 'implement nomupay webhook'.
license: MIT
metadata:
  author: lfraile
  version: "1.0"
---

# Nomupay — Webhook Handler

> Scaffold a production-ready webhook endpoint that correctly receives,
> verifies, acknowledges, and processes Nomupay event notifications.

## Invocation

- `nomupay webhook` — scaffold a webhook handler for the current project
- `implement nomupay webhook` — alias

## Prerequisites

Complete these skills first:

- `nomupay-jws-setup` — environment variables and key setup
- `nomupay-verify-response` — signature verification logic

---

## Context

Nomupay sends event notifications as HTTP POST requests to your endpoint.
Events cover: `payment`, `increment`, `capture`, `cancel`, `refund`,
`credit`, `token`.

Nomupay's processing is **asynchronous** — the outcome of an API operation
(e.g. create payment) is delivered via webhook, not in the synchronous HTTP
response. Correct webhook handling is essential for reliable integration.

---

## Workflow

### Step 1 — Inspect the project

Before scaffolding:

- Identify the web framework (Express, FastAPI, Spring Boot, ASP.NET, Gin, etc.)
- Identify how raw request body buffering works in that framework
- Identify what persistence is available for idempotency records (DB, cache, etc.)

### Step 2 — Scaffold the endpoint

Create a **POST** endpoint (e.g. `/webhooks/nomupay`). It must:

- Accept `Content-Type: application/json`
- Be accessible over **HTTPS with TLS 1.2** from the internet
- Allow POST requests only (reject other methods with 405)

### Step 3 — Buffer the raw body

**Before any JSON parsing**, capture the raw request body bytes. This is
required for signature verification (see `nomupay-verify-response`).

Many frameworks auto-parse JSON bodies and discard the raw bytes — ensure
the framework is configured to preserve them (e.g. via middleware, raw body
parser, or request stream buffering).

### Step 4 — Verify the signature

Apply the verification logic from `nomupay-verify-response`:

1. Extract all `X-Signature` headers
2. For each: validate `alg` = `"HS256"`, `exp` not expired, `kid` is a valid KSUID
3. Re-attach raw body and verify against `NOMUPAY_HMAC_KEY`

**If no header passes verification → return HTTP 400 immediately. Do not process the payload.**

### Step 5 — Acknowledge within 30 seconds

Return **HTTP 200** (or any 2xx) as quickly as possible after verification passes.

- Do **not** wait for business logic to complete before responding
- Offload all processing to a background queue or worker after sending the response
- If Nomupay does not receive a 2xx within **30 seconds**, the notification is
  queued and retried according to the retry policy below

### Step 6 — Deduplicate

Each notification has a unique `id` field at the top level of the payload.
Before processing:

1. Check whether this `id` has already been processed (idempotency store)
2. If already processed → return HTTP 200 and skip (idempotent ACK)
3. If new → mark as in-progress, then process

### Step 7 — Route by event type

Parse the `type` field to determine the event kind:

| `type` | Meaning |
|---|---|
| `payment` | Initial payment operation result |
| `increment` | Amount increment result |
| `capture` | Capture result |
| `cancel` | Cancel result |
| `refund` | Refund result |
| `credit` | Standalone credit result |
| `token` | Token created or updated |

The `payload` field contains the operation-specific data.
The `live` boolean distinguishes production (`true`) from sandbox (`false`) events.

### Step 8 — Handle out-of-order delivery

Nomupay does **not** guarantee notification delivery order. Do not rely on
event arrival sequence to derive state. Instead:

- Read the `status` field from the `payload` to determine current state
- Use `createTime` / `updateTime` fields to resolve conflicts if needed
- Design handlers to be idempotent and state-independent

### Step 9 — Handle processing failures safely

If business logic fails after the HTTP 200 has been sent:

- Log the failure with the notification `id` for manual recovery
- Use a dead-letter queue or alerting mechanism
- Do not allow exceptions to propagate to the HTTP layer (response already sent)

---

## Retry Policy Reference

If Nomupay does not receive a 2xx within 30 seconds (or receives a non-2xx),
the notification is retried:

| Attempt | Delay | Cumulative |
|---|---|---|
| 1 | 2 min | 2 min |
| 2 | 5 min | 7 min |
| 3 | 8 min | 15 min |
| 4 | 15 min | 30 min |
| 5 | 30 min | 1 h |
| 6 | 1 h | 2 h |
| 7 | 2 h | 4 h |
| 8 | 4 h | 8 h |
| 9–21 | 8 h | up to ~96 h |

After 21 retries, the notification is **permanently dropped**.

---

## Polling Fallback (trackingUrl)

If webhooks are unavailable or a payment is stuck in `pending`, use the
`trackingUrl` field returned in the original operation response.

- `trackingUrl` is a **path only** (no host), e.g. `/v1/tracks/payments/MjdsSUM4N2x`
- Prepend the base URL: `https://api.sandbox.nomupay.com` (sandbox) or
  `https://api.nomupay.com` (live)
- Authenticate the GET request with an `X-Signature` header as normal
  (see `nomupay-sign-request`)
- The response schema matches the original operation response

---

## Payment Lifecycle Reference

Statuses that may appear in webhook payloads:

| Status | Stage |
|---|---|
| `pending` | Awaiting processing |
| `approved` | Authorized, not yet captured |
| `partially_captured` | Partially captured |
| `captured` | Fully captured |
| `partially_cancelled` | Partially cancelled |
| `cancelled` | Fully cancelled |
| `partially_refunded` | Partially refunded |
| `refunded` | Fully refunded |
| `failed` | Processing failure |
| `aborted` | Aborted |
| `declined` | Declined by issuer/processor |
| `rejected` | Rejected |
| `unknown` | Indeterminate outcome |

---

## Implementation Checklist

- [ ] Endpoint is POST-only and accessible over HTTPS/TLS 1.2
- [ ] Raw body is buffered before JSON parsing
- [ ] All `X-Signature` headers are extracted and verified (`nomupay-verify-response`)
- [ ] Returns HTTP 400 immediately if signature verification fails
- [ ] Returns HTTP 2xx within 30 seconds of receiving the request
- [ ] Business logic runs asynchronously after acknowledgement
- [ ] Notification `id` is checked for duplicates before processing
- [ ] Event `type` is used to route to the correct handler
- [ ] `live` field is checked to distinguish sandbox from production events
- [ ] Handlers are idempotent and read `status` from payload (not event order)
- [ ] Processing failures are logged with notification `id` and do not affect HTTP response
- [ ] `trackingUrl` polling is implemented as a fallback for pending operations
