---
name: nomupay-sign-request
description: >
  Build the X-Signature JWS detached header required for every outbound Nomupay
  API request. Language-agnostic conceptual steps; adapt to whichever JWT/crypto
  library is available in the target language. Prerequisite: nomupay-jws-setup.
  Trigger: 'nomupay sign' or 'add X-Signature to request'.
license: MIT
metadata:
  author: lfraile
  version: "1.0"
---

# Nomupay — Sign Request (X-Signature Header)

> Produce a JWS detached token and attach it as the `X-Signature` header on
> every outbound request to the Nomupay API.

## Invocation

- `nomupay sign` — generate signing code for the current project
- `add X-Signature to request` — alias

## Prerequisite

Complete `nomupay-jws-setup` first. You must have:

- `NOMUPAY_KID` set (KSUID key identifier)
- A private EC P-256 key in PKCS8 format, loadable via one of the three
  env-var mechanisms described in `nomupay-jws-setup`

---

## Concept

A JWS detached token has three Base64URL segments separated by dots:

```
{base64url_header}.{base64url_payload}.{base64url_signature}
```

For Nomupay, the **payload is detached** — it is removed from the token but
is still part of what is signed. The final `X-Signature` value looks like:

```
{base64url_header}..{base64url_signature}
```

The thing being signed is the **raw JSON request body** (UTF-8 encoded, no
transformation). For requests with no body (e.g. GET), the payload is an
empty string `""`.

---

## Workflow

### Step 1 — Inspect the project

Before writing code:

- Check the project's dependency manifest (`package.json`, `requirements.txt`,
  `pom.xml`, `*.csproj`, `go.mod`, etc.)
- Identify any existing JWT or crypto library (e.g. `jose`, `jsonwebtoken`,
  `PyJWT`, `System.IdentityModel.Tokens.Jwt`, `github.com/golang-jwt/jwt`)
- Use that library if present; otherwise recommend the most idiomatic one for
  the language

### Step 2 — Load the private key

Implement private key loading with this priority order (see `nomupay-jws-setup`):

1. `NOMUPAY_PRIVATE_KEY` env var — full PEM as `\n`-escaped string; must
   replace literal `\n` with real newlines before parsing
2. `NOMUPAY_PRIVATE_KEY_PATH` env var — path to `.pem` file on disk
3. `./private-key.pem` — local fallback

The key must be parsed as an **EC P-256 private key in PKCS8 format**.

### Step 3 — Build the protected header

The JWS protected header must contain exactly these claims:

| Claim | Value |
|---|---|
| `alg` | `"ES256"` |
| `kid` | Value of `NOMUPAY_KID` env var |
| `aud` | `"{HTTP_METHOD} {urlPath}"` — method uppercased, path includes query string but NOT the host. Example: `"POST /v1/payments"` or `"GET /v1/payments?fromTime=2024-01-01T00:00:00Z&toTime=2024-01-02T00:00:00Z"` |
| `exp` | Unix timestamp — current time + at most **5 minutes** (3 minutes recommended) |

No other claims should be added to the header.

### Step 4 — Sign the payload

Sign the **raw JSON request body** bytes (UTF-8) using **ES256**
(ECDSA with P-256 curve and SHA-256 hash).

- The signed payload is the exact bytes sent in the HTTP body
- For requests with no body (GET, DELETE), sign an empty string `""`
- Do not re-serialize or pretty-print the body — use the exact bytes

### Step 5 — Detach the payload

After signing, a standard JWT/JWS token has the structure:

```
{header}.{payload}.{signature}
```

Remove the middle segment to produce the Nomupay detached format:

```
{header}..{signature}
```

### Step 6 — Attach to the request

Set the HTTP request header:

```
X-Signature: {base64url_header}..{base64url_signature}
```

Every outbound request — regardless of HTTP method — must include this header.

---

## Security Rules

- **Never log or expose** the private key or the full (non-detached) JWS token
- **Clock sync:** ensure the server clock is accurate — a skewed clock causes
  unexpected 401 responses
- **Max 5 minutes** expiry: shorter is better for security (3 minutes recommended)
- The `aud` claim must **exactly** match `{METHOD} {path+querystring}`:
    - Method is uppercase (`POST`, `GET`, `PATCH`)
    - Path includes the query string but not the scheme or host
    - Case-sensitive

---

## Validation Checklist

- [ ] Library identified from dependency manifest before writing code
- [ ] Private key loaded via 3-priority env-var mechanism
- [ ] Protected header contains exactly `alg`, `kid`, `aud`, `exp` — no extra fields
- [ ] `aud` format is `"METHOD /path?query"` with the real query string
- [ ] `exp` is no more than 5 minutes in the future
- [ ] Payload signed is the raw JSON body bytes (not re-serialized or pretty-printed)
- [ ] Token format is `header..signature` (two dots, no middle segment)
- [ ] `X-Signature` header is present on every outbound request
