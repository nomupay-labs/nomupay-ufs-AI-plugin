---
name: nomupay-verify-response
description: >
  Verify the X-Signature JWS header on Nomupay API responses and webhook messages
  using HS256 and a shared HMAC key. Includes multi-header rotation handling and
  security validation checklist. Language-agnostic. Prerequisite: nomupay-jws-setup.
  Trigger: 'nomupay verify' or 'verify webhook signature'.
license: MIT
metadata:
  author: lfraile
  version: "1.0"
---

# Nomupay — Verify Response / Webhook Signature

> Verify that an API response or webhook notification genuinely came from
> Nomupay by validating the `X-Signature` header.

## Invocation

- `nomupay verify` — generate verification code for the current project
- `verify webhook signature` — alias

## Prerequisite

Complete `nomupay-jws-setup` first. You must have:

- `NOMUPAY_HMAC_KEY` set — the shared HMAC secret provided by Nomupay support

---

## Concept

All Nomupay API responses and webhook messages include an `X-Signature` header
containing a JWS detached token signed with **HS256** using a shared HMAC key.

The token format uses the same detached structure as request signatures:

```
{base64url_header}..{base64url_signature}
```

To verify, you must **re-attach** the raw response/webhook body as the
payload, then verify the resulting compact JWS against the shared HMAC key.

---

## Workflow

### Step 1 — Inspect the project

Before writing code:

- Check the dependency manifest for an existing JWT/HMAC/crypto library
- Use that library if present; otherwise recommend the most idiomatic option
  for the language

### Step 2 — Capture the raw body

**Before any parsing or transformation**, capture the raw response/webhook
body bytes exactly as received. Any modification (re-serialisation,
whitespace normalisation, encoding changes) will break the signature.

- For **webhook endpoints**: buffer the raw request body before JSON-parsing it
- For **API responses**: read the raw response bytes before decoding JSON

### Step 3 — Extract all X-Signature headers

The response may contain **multiple `X-Signature` headers** during a key
rotation period. Collect all of them as a list. The message is authentic if
**any one** of them verifies successfully.

### Step 4 — Validate header claims before verifying

For each `X-Signature` token, decode the header segment (Base64URL decode,
no signature verification yet) and check:

| Claim | Required |
|---|---|
| `alg` | Must be exactly `"HS256"` — reject anything else including `"none"` |
| `exp` | Must be in the future — reject expired tokens (prevents timing attacks) |
| `kid` | Must be a valid KSUID: 27 alphanumeric characters — reject malformed values (prevents injection) |
| `aud` | Present only on **webhook** messages — contains the target URL; validate it matches your endpoint |

Reject the message immediately if `alg` ≠ `"HS256"` or `exp` is expired.

### Step 5 — Re-attach the payload

Re-attach the raw body bytes as the JWS payload:

1. Encode the raw body bytes as **Base64URL** (no padding)
2. Insert as the middle segment of the detached token:

   ```
   {header}.{base64url_encoded_raw_body}.{signature}
   ```

### Step 6 — Verify the signature

Verify the reassembled compact JWS using:

- Key: `NOMUPAY_HMAC_KEY` (UTF-8 encoded bytes)
- Algorithm: **HS256**

Outcomes:

- Verification passes → the message is authentic; proceed to process it
- Verification fails → reject the message; do not process; return HTTP 400

Treat all verification exceptions as failures. Do not leak error details in
responses.

---

## Security Rules

- **Never skip verification** — always verify before processing the payload
- **Reject unknown `alg`** — never allow `"none"` or any non-HS256 value; this
  prevents algorithm confusion attacks
- **Check `exp` strictly** — expired tokens must be rejected even if the
  signature is mathematically valid
- **Validate `kid` format** — a KSUID is exactly 27 alphanumeric characters;
  reject values that do not match this pattern to prevent injection attacks
- **Do not log** the HMAC key or the raw signature value
- **During rotation:** accept the first header that passes verification;
  monitor which `kid` values are in use to track rotation progress

---

## Validation Checklist

- [ ] Library identified from dependency manifest before writing code
- [ ] Raw body is captured before any parsing or transformation
- [ ] All `X-Signature` headers are collected (multiple possible during rotation)
- [ ] `alg` claim is checked to be exactly `"HS256"` before verifying
- [ ] `exp` claim is checked — token must not be expired
- [ ] `kid` is validated as exactly 27 alphanumeric characters (KSUID)
- [ ] For webhooks: `aud` is present and matches the endpoint URL
- [ ] Payload is re-attached as Base64URL-encoded raw body bytes (no padding)
- [ ] Message is rejected and not processed if no header verifies successfully
- [ ] HTTP 400 (or appropriate error) is returned on verification failure
