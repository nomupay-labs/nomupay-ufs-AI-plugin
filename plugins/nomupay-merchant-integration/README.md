# nomupay-merchant-integration

> **Version:** 0.1.0 · **Author:** Nomupay · **Category:** Development
>
> Part of the [`nomupay-labs/nomupay-ufs-AI-plugin`](../../README.md) plugin
> repository.

A plugin that bundles language-specific integration agents (C#,
JavaScript/TypeScript, PHP) and cross-language skills for implementing the
**Nomupay Ecommerce API** in merchant applications. Covers authentication (JWS),
request signing, response verification, webhooks, and payment operations.

---

## Agents

Each agent is a language-specialist that understands the Nomupay API and all
five skills listed below. Invoke the agent that matches your project's
technology stack.

| Agent file                                                                     | Description                                                          |
| ------------------------------------------------------------------------------ | -------------------------------------------------------------------- |
| [`nomupay-integration-csharp.md`](agents-claude/nomupay-integration-csharp.md) | Expert C#/.NET developer. Uses `jose-jwt` for JWS signing.           |
| [`nomupay-integration-js.md`](agents-claude/nomupay-integration-js.md)         | Expert TypeScript/JavaScript developer. Uses `jose` for JWS signing. |
| [`nomupay-integration-php.md`](agents-claude/nomupay-integration-php.md)       | Expert PHP developer. Uses `firebase/php-jwt` for JWS signing.       |

**Capabilities covered by all agents:**

- Create, capture, cancel, refund, and credit payment operations
- JWS authentication (ES256) — key generation, signing, and `X-Signature` header
  construction
- Response and webhook signature verification (HS256 HMAC)
- Webhook endpoint scaffolding (idempotency, retry awareness, 30-second
  acknowledgement)
- Payment token workflows

---

## Skills

Skills are language-agnostic knowledge units consumed by all three agents. They
can also be invoked independently.

### `nomupay-api-reference`

Quick-reference for the Nomupay Ecommerce API: base URLs, authentication token
rules, the full endpoints table, and payment method specifics.

**Trigger:** `nomupay api reference` · `nomupay endpoints`

### `nomupay-jws-setup`

One-time setup walkthrough: generate an EC P-256 key pair in PKCS8 format,
register the public key with Nomupay support to obtain a `kid`, and configure
the required environment variables.

**Trigger:** `nomupay-setup` · `nomupay auth setup`

**Required environment variables after setup:**

| Variable                   | Description                                                            |
| -------------------------- | ---------------------------------------------------------------------- |
| `NOMUPAY_KID`              | KSUID key identifier issued by Nomupay support                         |
| `NOMUPAY_PRIVATE_KEY`      | Full PEM private key as a single `\n`-escaped string (secrets manager) |
| `NOMUPAY_PRIVATE_KEY_PATH` | Path to the `.pem` file on disk (local dev alternative)                |
| `NOMUPAY_HMAC_KEY`         | Shared HMAC secret for verifying responses and webhooks                |

### `nomupay-sign-request`

Build the `X-Signature` JWS detached header required on every outbound API
request. Provides language-agnostic signing steps; agents adapt these to the
target language's JWT library.

**Trigger:** `nomupay sign` · `add X-Signature to request`

**Prerequisite:** `nomupay-jws-setup`

### `nomupay-verify-response`

Verify the `X-Signature` header on API responses and webhook notifications using
HS256 and the shared HMAC key. Includes multi-header key rotation handling and a
security validation checklist.

**Trigger:** `nomupay verify` · `verify webhook signature`

**Prerequisite:** `nomupay-jws-setup`

### `nomupay-webhook-handler`

Scaffold a production-ready webhook endpoint: signature verification, 30-second
acknowledgement, idempotency, out-of-order event handling, retry awareness, and
tracking URL fallback.

**Trigger:** `nomupay webhook` · `implement nomupay webhook`

**Prerequisites:** `nomupay-jws-setup`, `nomupay-verify-response`
