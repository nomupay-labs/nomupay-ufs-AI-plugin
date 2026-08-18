---
name: nomupay-jws-setup
description: >
  One-time setup for Nomupay JWS authentication: generate an EC P-256 key pair in
  PKCS8 format, share the public key with Nomupay support to obtain a kid, configure
  required environment variables, and understand key rotation. Language-agnostic.
  Trigger: 'nomupay-setup' or 'nomupay auth setup'.
license: MIT
metadata:
  author: lfraile
  version: "1.0"
---

# Nomupay JWS Setup

> Walk through the one-time key generation, registration, and environment
> configuration required before any Nomupay API call can succeed.

## Invocation

- `nomupay-setup` — run the full setup workflow
- `nomupay auth setup` — alias

---

## Context

Every request to the Nomupay Ecommerce API must be authenticated with a JWS
detached token (ES256). Before generating any code, the developer must have:

1. An EC P-256 private key in **PKCS8 format**
2. A `kid` (Key ID) issued by Nomupay support after registering the public key
3. A shared HMAC key (provided by Nomupay support) for verifying responses

Until the public key is registered and a valid `kid` is configured, every API
request returns **HTTP 401**.

---

## Workflow

### Step 1 — Check for an existing key pair

Before generating new keys, check whether `private-key.pem` and
`public-key.pem` already exist in the project root or a designated secrets
location. If they do, skip to **Step 3**.

### Step 2 — Generate the key pair

Run the following commands (OpenSSL required):

```bash
# Preferred: generate EC P-256 private key directly in PKCS8 format
openssl genpkey -algorithm EC -pkeyopt ec_paramgen_curve:P-256 -out private-key.pem

# Fallback (if genpkey is unavailable):
openssl ecparam -name prime256v1 -genkey -noout | \
  openssl pkcs8 -topk8 -nocrypt -out private-key.pem

# Extract the public key
openssl ec -in private-key.pem -pubout -out public-key.pem
```

**Important:** The private key must be in PKCS8 format — most JWT libraries
require this. Do not share or commit `private-key.pem`.

### Step 3 — Register the public key with Nomupay

Email `public-key.pem` to [support@nomupay.com](mailto:support@nomupay.com)
and request:

- A `kid` (Key ID, in KSUID format) — used in every request signature
- A shared HMAC key — used to verify all API responses and webhook messages

Nomupay support will provide both values. The API returns **HTTP 401** until
registration is complete.

### Step 4 — Configure environment variables

Never hardcode secrets. Set the following variables in your environment,
secrets manager, or `.env` file (never commit `.env`):

| Variable | Description |
|---|---|
| `NOMUPAY_KID` | Key ID (KSUID) provided by Nomupay support |
| `NOMUPAY_PRIVATE_KEY_PATH` | Path to `private-key.pem` (recommended for local dev) |
| `NOMUPAY_PRIVATE_KEY` | Full PEM content as a single `\n`-escaped string (for CI / secrets managers) |
| `NOMUPAY_HMAC_KEY` | Shared HMAC secret for verifying API responses and webhooks |
| `NOMUPAY_PROCESSING_ACCOUNT_ID` | 27-character account ID provided by Nomupay |
| `NOMUPAY_BASE_URL` | API base URL — use `https://api.sandbox.nomupay.com` for sandbox (default) |

> **Private key loading priority** (implement in this order in your code):
>
> 1. `NOMUPAY_PRIVATE_KEY` — full PEM as `\n`-escaped string (secrets managers store it this way; replace literal `\n` with real newlines before parsing)
> 2. `NOMUPAY_PRIVATE_KEY_PATH` — filesystem path to the `.pem` file
> 3. `./private-key.pem` — local fallback

### Step 5 — Add private-key.pem to .gitignore

Ensure `private-key.pem` is listed in `.gitignore`. Verify it is not already
tracked by git:

```bash
git status
# If tracked, remove from tracking:
git rm --cached private-key.pem
```

### Step 6 — Verify setup (smoke test)

With keys and environment variables in place, make a minimal authenticated
request (e.g. `GET /v1/payments` with a short time range) and confirm you
receive **HTTP 200** (not 401 or 403).

---

## Key Rotation

- Multiple active key pairs are supported simultaneously.
- When rotating, keep the **old keys active for several days** — Nomupay's
  async system may still issue responses signed with the old HMAC key during
  the transition.
- During a rotation window, API responses may include **multiple
  `X-Signature` headers** (one per active HMAC key). Your verification code
  must accept any valid one.
- To revoke keys, email [support@nomupay.com](mailto:support@nomupay.com)
  with the `kid` to revoke.

---

## Completion Checklist

- [ ] `private-key.pem` exists and is in PKCS8 format
- [ ] `public-key.pem` has been sent to `support@nomupay.com`
- [ ] `kid` received from Nomupay support and set in `NOMUPAY_KID`
- [ ] HMAC shared key received and set in `NOMUPAY_HMAC_KEY`
- [ ] `NOMUPAY_PROCESSING_ACCOUNT_ID` is set
- [ ] `NOMUPAY_BASE_URL` is set (or defaulting to sandbox)
- [ ] `private-key.pem` is in `.gitignore` and not tracked by git
- [ ] Smoke test request returns HTTP 200
