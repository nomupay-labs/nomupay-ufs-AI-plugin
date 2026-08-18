---
name: nomupay-api-reference
description: >
  Nomupay Ecommerce API quick-reference: key resources, base URLs, authentication
  token rules, response verification rules, full endpoints table, and payment method
  specifics. Language-agnostic. Use as context when implementing any Nomupay operation.
  Trigger: 'nomupay api reference' or 'nomupay endpoints'.
license: MIT
metadata:
  author: lfraile
  version: "1.0"
---

# Nomupay — API Reference

> Quick-reference for the Nomupay Ecommerce API. Covers resources, auth rules,
> endpoints, and payment method specifics. Language-agnostic — use alongside
> a language-specific agent for implementation.

## Invocation

- `nomupay api reference` — display the full reference
- `nomupay endpoints` — alias

---

## Key Resources

- **OpenAPI spec:** `https://docs.nomupay.com/openapi/online-payments.yaml`
- **General docs:** `https://docs.nomupay.com/payments/integration-up-api/`
- **API Reference:** `https://docs.nomupay.com/payments/integration-up-api/api-reference`
- **Security / Auth:** `https://docs.nomupay.com/payments/integration-up-api/security`
- **Webhooks:** `https://docs.nomupay.com/payments/integration-up-api/webhooks`

## Base URLs

| Environment | URL |
|---|---|
| **Sandbox (default)** | `https://api.sandbox.nomupay.com` |
| Live | `https://api.nomupay.com` |

> All API requests must be made over **HTTPS (TLS 1.2)**.

---

## Authentication — Token Rules (ES256)

Every outbound request **must** include an `X-Signature` header containing a
JWS detached token. Keys should be generated in format "pkcs8".

- Algorithm: **ES256** (EC NIST P-256 + SHA-256)
- Format: `{header}..{signature}` (payload is detached — not embedded in the token)
- The signed payload is the **raw JSON request body** encoded as UTF-8
- Header claims required:

| Claim | Value |
|---|---|
| `alg` | `"ES256"` |
| `aud` | `"{HTTP_METHOD} {path+querystring}"` — e.g., `"POST /v1/payments"` |
| `exp` | Unix timestamp, **max 5 minutes** from now |
| `kid` | Key ID (KSUID) provided by Nomupay support when the public key was registered |

> For setup (key generation, registration, env vars): use `nomupay-jws-setup`.
> For signing implementation: use `nomupay-sign-request`.

---

## Verifying Signed Responses — Rules (HS256)

All API responses and webhook messages include an `X-Signature` header signed
with **HS256** using a shared HMAC key provided by Nomupay support.

- Algorithm: **HS256**
- The payload to verify is the **raw response body / webhook body** as received (no transformation)
- Header claims to validate:

| Claim | Rule |
|---|---|
| `alg` | Must be exactly `"HS256"` — reject anything else including `"none"` |
| `exp` | Must not be expired (prevents timing attacks) |
| `kid` | Must be a valid KSUID (27 alphanumeric chars) — validate format to prevent injection |
| `aud` | Present only on webhook messages — contains the target URL |

> For verification implementation: use `nomupay-verify-response`.
> For webhook endpoint scaffolding: use `nomupay-webhook-handler`.

---

## Environment Variables

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
> 1. `NOMUPAY_PRIVATE_KEY` — full PEM as `\n`-escaped string
> 2. `NOMUPAY_PRIVATE_KEY_PATH` — filesystem path to the `.pem` file
> 3. `./private-key.pem` — local fallback

> For full setup walkthrough: see `nomupay-jws-setup`.

---

## API Endpoints

| Operation | Method | Path | Scope |
|---|---|---|---|
| Create payment | `POST` | `/v1/payments` | `payments:create` |
| Get all payments | `GET` | `/v1/payments` | `payments:read` |
| Get payment by ID | `GET` | `/v1/payments/{paymentId}` | `payments:read` |
| Get payment by idempotency key | `GET` | `/v1/requests/payments` | `payments:read` |
| Increment payment | `POST` | `/v1/payments/{paymentId}/increments` | `payments:create` |
| Get increment by idempotency key | `GET` | `/v1/requests/increments` | `payments:read` |
| Capture payment | `POST` | `/v1/payments/{paymentId}/captures` | `payments:create` |
| Get capture by idempotency key | `GET` | `/v1/requests/captures` | `payments:read` |
| Cancel payment | `POST` | `/v1/payments/{paymentId}/cancels` | `payments:create` |
| Get cancel by idempotency key | `GET` | `/v1/requests/cancels` | `payments:read` |
| Refund payment | `POST` | `/v1/payments/{paymentId}/refunds` | `payments:create` |
| Get refund by idempotency key | `GET` | `/v1/requests/refunds` | `payments:read` |
| Create credit | `POST` | `/v1/credits` | `payments:create` |
| Get credit by idempotency key | `GET` | `/v1/requests/credits` | `payments:read` |
| Get all transactions | `GET` | `/v1/transactions` | `payments:read` |
| Get transaction by ID | `GET` | `/v1/transactions/{transactionId}` | `payments:read` |
| Create token | `POST` | `/v1/tokens` | `payments:create` |
| Get token by ID | `GET` | `/v1/tokens/{tokenId}` | `payments:read` |
| Update token | `PATCH` | `/v1/tokens/{tokenId}` | `payments:create` |

---

## Payment Method Specifics

- **Card payments** (Visa, Mastercard, Maestro) have rich sub-schemas including
  `billingAddress`, `secureRemotePayment`, `threedsPassThrough`, `credentialOnFile`
- **APM** (Alternative Payment Methods) like FPX, iDEAL, PayPal, Klarna, etc.
  each have their own required fields — always consult the OpenAPI spec for the
  specific method
- **SEPA** requires `flowType` and bank account with optional mandate for recurring
- **UPI/AlipayPlus/WeChat** support `discountAmount`

> Always fetch the OpenAPI spec or API reference page before implementing a
> payment operation to confirm the exact request/response schema.

---

## Idempotency

Use the `idempotencyKey` query parameter for `POST` operations when retrying.
To recover an operation by its idempotency key, use the corresponding
`GET /v1/requests/{operation}` endpoint (see endpoints table).

---

## Error Handling

Map Nomupay `reason.code` values to domain errors/exceptions. Handle these
HTTP status codes:

| Status | Meaning |
|---|---|
| `400` | Bad request — invalid payload |
| `404` | Resource not found |
| `409` | Conflict — duplicate idempotency key with different payload |
| `429` | Rate limited — back off and retry |
| `500` | Server error — retry with idempotency key |
