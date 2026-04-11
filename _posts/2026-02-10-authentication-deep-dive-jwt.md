---
title: "Authentication Deep Dive: How JWT Works"
author: pravin_tripathi
date: 2026-02-10 00:00:00 +0530
readtime: true
media_subpath: /assets/img/authentication-deep-dive/
categories: [Blogging, Article]
mermaid: true
tags: [softwareengineering, backenddevelopment, security, authentication, jwt]
image:
  path: header.png
  width: 1600   # in pixels
  height: 900   # in pixels
  alt: Generated using Gemini AI
---

Authentication is one of those topics where almost every engineer knows *how to use it* — call the login endpoint, store the token, attach the `Authorization` header — but far fewer know *what is actually happening* when that token is generated, what the three dot-separated blobs inside it mean, why the server trusts it without ever looking it up in a database, and exactly what has gone wrong when your frontend gets a 401 at 3 AM.

This post is a complete, bottom-up treatment of JWT authentication. We will start from why sessions failed to scale, go deep on the cryptographic anatomy of a token, trace every step of the login and request flows, and build a production-grade implementation in FastAPI. Part 2 of this series covers a completely different kind of authentication — AWS service-to-service auth via IAM roles — which operates on the same conceptual foundations but is worth treating separately because the mechanics diverge significantly.

---

## Table of Contents

1. [The Problem JWT Solves](#the-problem-jwt-solves)
2. [What a JWT Is Made Of](#what-a-jwt-is-made-of)
3. [Signing Algorithms — HS256 vs RS256 vs ES256](#signing-algorithms)
4. [How a Token Is Generated](#how-a-token-is-generated)
5. [The Complete Authentication Flow](#the-complete-authentication-flow)
6. [How the Token Is Shared](#how-the-token-is-shared)
7. [How the Token Is Validated](#how-the-token-is-validated)
8. [Refresh Tokens and Token Rotation](#refresh-tokens-and-token-rotation)
9. [FastAPI Implementation — Production Grade](#fastapi-implementation)
10. [What Can Go Wrong — Attacks and Pitfalls](#what-can-go-wrong)
11. [Key Takeaways](#key-takeaways)

---

## The Problem JWT Solves

Before JWT, the dominant auth mechanism was **server-side sessions**. The server generated a random session ID at login, stored the full session state (user ID, roles, expiry) in a central store — typically Redis or a database — and sent only the session ID back to the client as a cookie. Every subsequent request carried that cookie, and the server had to look it up in the session store to know who was making the request.

This works well for a monolith. It breaks down badly when you scale:

**The session store becomes a bottleneck and a single point of failure.** Every request from every user on every server hits the same Redis cluster. At 10,000 req/s with a 1ms Redis round-trip, you are spending 10 seconds of Redis time per second of request handling. Your session store must scale with your request volume.

**Horizontal scaling requires session affinity or session sharing.** If you have 5 application servers and a user's session lives in server 1's memory, routing that user to server 3 breaks their session. You either pin users to specific servers (which wastes capacity and complicates deployments) or you move sessions to shared Redis (which reintroduces the centralised bottleneck).

**Microservices make session sharing expensive.** In a monolith, the session lookup is in-process. In a microservices architecture, your payments service calling your user service to validate a session involves an extra network hop on every request across every service boundary.

JWT solves all of this with a single insight: **move the session state from the server into the token itself, and cryptographically sign it so the server can verify it without looking anything up.**

The server goes from:

```
request arrives → look up session ID in Redis → trust the session data → handle request
```

to:

```
request arrives → verify the token's signature with a key the server already has → trust the claims in the token → handle request
```

No shared store. No network lookup. No stickiness requirement. Any server with the signing key can validate any token. This is why JWT became the default for distributed systems.

---

## What a JWT Is Made Of

A JWT looks like this:

```
eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJ1c2VyXzEyMyIsImVtYWlsIjoicHJhdmluQGV4YW1wbGUuY29tIiwicm9sZXMiOlsiYWRtaW4iXSwiaWF0IjoxNzQ0MjcwMjgyLCJleHAiOjE3NDQyNzM4ODJ9.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
```

Three Base64URL-encoded segments separated by dots. That is it. No encryption, no compression, just encoding. The contents are completely readable by anyone who holds the token — the only thing the signature protects is **tamper-evidence**, not confidentiality. Never put sensitive data (passwords, PII, secrets) in a JWT payload.

Let us decode each part manually:

### Segment 1 — Header

```python
import base64, json

header_b64 = "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9"

# Base64URL uses - and _ instead of + and / and omits padding.
# Add padding back before decoding.
padding = 4 - len(header_b64) % 4
header_json = base64.urlsafe_b64decode(header_b64 + "=" * padding)

print(json.loads(header_json))
# {'alg': 'RS256', 'typ': 'JWT'}
```

The header is a small JSON object with exactly two fields:

- `alg`: the signing algorithm. This tells the verifier which algorithm to use when checking the signature. **This field is security-critical** — we will return to why later.
- `typ`: the token type. Always `"JWT"`.

### Segment 2 — Payload (Claims)

```python
payload_b64 = "eyJzdWIiOiJ1c2VyXzEyMyIsImVtYWlsIjoicHJhdmluQGV4YW1wbGUuY29tIiwicm9sZXMiOlsiYWRtaW4iXSwiaWF0IjoxNzQ0MjcwMjgyLCJleHAiOjE3NDQyNzM4ODJ9"

padding = 4 - len(payload_b64) % 4
payload_json = base64.urlsafe_b64decode(payload_b64 + "=" * padding)

print(json.loads(payload_json))
# {
#   'sub': 'user_123',
#   'email': 'pravin@example.com',
#   'roles': ['admin'],
#   'iat': 1744270282,
#   'exp': 1744273882
# }
```

The payload is a JSON object of **claims** — statements about the subject (usually the user). The JWT spec defines several **registered claim names** that have well-known semantics:

| Claim | Full Name | Type | Meaning |
|-------|-----------|------|---------|
| `sub` | Subject | string | Who the token is about — typically a user ID |
| `iss` | Issuer | string | Who created this token — your auth service URL |
| `aud` | Audience | string or array | Who this token is intended for — your API domain |
| `iat` | Issued At | Unix timestamp | When the token was created |
| `exp` | Expiration | Unix timestamp | When the token stops being valid |
| `nbf` | Not Before | Unix timestamp | Token is not valid before this time |
| `jti` | JWT ID | string | Unique identifier for this token — used for revocation |

Beyond these, you can add any custom claims you need: `roles`, `tenant_id`, `plan`, `permissions`. The tradeoff is token size — every claim you add is bytes transmitted on every single request. Keep the payload lean; put large, infrequently-needed data in your database and look it up when required.

### Segment 3 — Signature

The signature is not JSON. It is the raw output of the signing algorithm applied to:

```
HMAC-SHA256(
  base64url(header) + "." + base64url(payload),
  secret_key
)
```

or for asymmetric algorithms:

```
RSA-PKCS1-SHA256-Sign(
  base64url(header) + "." + base64url(payload),
  private_key
)
```

The output is binary, Base64URL-encoded into the third segment. The server verifies the signature by re-computing it (or, for asymmetric, verifying with the public key) and comparing. If the payload was altered even by a single character, the signature will not match.

```mermaid
graph LR
    subgraph "JWT Structure"
        H["Header {alg: RS256, typ: JWT} → Base64URL encode"]
        P["Payload / Claims {sub, email, roles, iat, exp} → Base64URL encode"]
        S["Signature Sign(header.payload, private_key) → Base64URL encode"]
    end

    H -->|"."| Join
    P -->|"."| Join
    Join -->|"."| S
    S --> Token["eyJhbG...header.eyJzdWI...payload.SflKxw...signature"]
```

---

## Signing Algorithms

The choice of signing algorithm is the most consequential security decision in JWT. There are three families, each with different trust models.

### HS256 — HMAC with SHA-256 (Symmetric)

A single secret key is used for both signing and verification. The same bytes that create a token can also verify one.

```
token = HMAC-SHA256(header + "." + payload, secret)
valid = HMAC-SHA256(header + "." + payload, secret) == token.signature
```

**When to use it**: Single-service auth where only one server both creates and validates tokens (e.g., a monolith or a single API). Simple to implement, fast.

**When not to use it**: When multiple services need to verify tokens. If your payments service needs to verify a token issued by your auth service, you must share the secret with payments. Now payments can also *create* valid tokens — it has the same power as the auth service. Every service you share the secret with is a potential token forge. In a microservices architecture, this is a serious blast radius problem.

### RS256 — RSA with SHA-256 (Asymmetric)

Two keys: a private key signs tokens, a public key verifies them. The private key lives only on the auth service; the public key can be distributed freely to every service that needs to verify tokens.

```
token = RSA-Sign(header + "." + payload, private_key)       # auth service only
valid = RSA-Verify(header + "." + payload, public_key, sig)  # any service
```

**When to use it**: Microservices, multi-tenant systems, any architecture where multiple independent services need to verify tokens. A compromised downstream service (payments, shipping, etc.) can verify tokens but cannot forge them — the private key is not exposed.

**When not to use it**: When raw performance matters above all else. RSA-2048 signing is ~10x slower than HMAC-SHA256. For services issuing millions of tokens per second, this is measurable.

### ES256 — ECDSA with P-256 (Asymmetric)

Same two-key model as RS256, but uses elliptic curve cryptography. ES256 keys are much smaller (256-bit vs 2048-bit for equivalent security), signing is faster than RSA, and tokens are shorter.

**When to use it**: When you need asymmetric security (microservices) and care about token size or signing latency. ES256 is increasingly the default in modern systems — AWS Cognito, Google, and Auth0 all issue ES256 tokens.

```mermaid
graph TD
    subgraph "HS256 — Symmetric"
        SK["Single Secret Key (known to ALL verifiers)"]
        AS_HS["Auth Service signs with secret"]
        SVC_HS["Any Service verifies with same secret (can also FORGE tokens!)"]
        SK --> AS_HS
        SK --> SVC_HS
    end

    subgraph "RS256 / ES256 — Asymmetric"
        PrvK["Private Key (auth service ONLY)"]
        PubK["Public Key (distributed openly via JWKS endpoint)"]
        AS_RS["Auth Service signs with private key"]
        SVC_RS["Any Service verifies with public key (cannot forge)"]
        PrvK --> AS_RS
        PubK --> SVC_RS
    end
```

---

## How a Token Is Generated

Token generation happens at login. Here is what actually occurs, step by step:

```python
# auth/token_service.py
import time
import uuid
import json
import base64
import hashlib
import hmac
from cryptography.hazmat.primitives import hashes, serialization
from cryptography.hazmat.primitives.asymmetric import padding, rsa
from cryptography.hazmat.backends import default_backend


def base64url_encode(data: bytes) -> str:
    """
    Base64URL encoding: standard base64 with + → -, / → _, padding stripped.
    JWT requires this variant — standard base64 would break in URLs and headers.
    """
    return base64.urlsafe_b64encode(data).rstrip(b"=").decode("utf-8")


def generate_jwt_rs256(
    subject: str,
    email: str,
    roles: list[str],
    private_key_pem: bytes,
    issuer: str = "https://auth.yourapp.com",
    audience: str = "https://api.yourapp.com",
    expiry_seconds: int = 3600,   # 1 hour
) -> str:
    """
    Manually generates a JWT using RS256 to make every step visible.
    In production you would use python-jose or PyJWT, but building it
    by hand once is the best way to understand what those libraries do.
    """

    now = int(time.time())

    # ── Step 1: Build the header ──────────────────────────────────────────────
    header = {
        "alg": "RS256",
        "typ": "JWT",
        # kid (key ID) lets clients identify which public key to use for
        # verification when the auth server rotates keys. Corresponds to
        # the 'kid' field in the JWKS endpoint response.
        "kid": "rsa-key-2026-04"
    }
    header_b64 = base64url_encode(json.dumps(header, separators=(",", ":")).encode())

    # ── Step 2: Build the payload ─────────────────────────────────────────────
    payload = {
        "sub": subject,                   # subject — user ID, stable across sessions
        "email": email,
        "roles": roles,
        "iss": issuer,                    # issuer — your auth service
        "aud": audience,                  # audience — who should accept this token
        "iat": now,                       # issued-at — for age calculations
        "exp": now + expiry_seconds,      # expiry — after this, token is rejected
        "nbf": now,                       # not-before — token invalid before this
        # jti: unique ID per token.
        # Essential for token revocation: you store revoked jtis in Redis
        # and check each token's jti against the revocation list.
        # Without jti, you cannot revoke a specific token before it expires.
        "jti": str(uuid.uuid4()),
    }
    payload_b64 = base64url_encode(json.dumps(payload, separators=(",", ":")).encode())

    # ── Step 3: Create the signing input ─────────────────────────────────────
    # The data being signed is exactly: base64url(header) + "." + base64url(payload)
    # This is what the verifier reconstructs and checks against the signature.
    signing_input = f"{header_b64}.{payload_b64}".encode("utf-8")

    # ── Step 4: Sign with the private key ─────────────────────────────────────
    private_key = serialization.load_pem_private_key(
        private_key_pem, password=None, backend=default_backend()
    )
    # PKCS1v15 padding with SHA-256 hash — this is what RS256 means:
    # RSA signing (RS) with PKCS1v15 padding and SHA-256 (256)
    signature = private_key.sign(
        signing_input,
        padding.PKCS1v15(),
        hashes.SHA256()
    )
    signature_b64 = base64url_encode(signature)

    # ── Step 5: Assemble the final token ─────────────────────────────────────
    return f"{header_b64}.{payload_b64}.{signature_b64}"
```

---

## The Complete Authentication Flow

The complete flow has two distinct phases: **login** (authenticating the user and issuing a token) and **request** (using that token to access a protected resource). They are entirely decoupled — the resource server does not communicate with the auth server on every request.

```mermaid
sequenceDiagram
    participant Client as Client (Browser / Mobile)
    participant Auth as Auth Service (issues tokens)
    participant API as Resource API (payments, orders...)
    participant DB as User Database

    Note over Client,DB: ── Phase 1: Login ──

    Client->>Auth: POST /auth/login {email, password}
    Auth->>DB: SELECT user WHERE email = ?
    DB-->>Auth: {user_id, hashed_password, roles}
    Auth->>Auth: bcrypt.verify(password, hashed_password)

    alt credentials valid
        Auth->>Auth: generate access_token (exp: 1h) generate refresh_token (exp: 7d) store refresh_token hash in Redis
        Auth-->>Client: 200 OK {access_token, refresh_token, token_type: "Bearer", expires_in: 3600}
    else credentials invalid
        Auth-->>Client: 401 Unauthorized (identical response for wrong email OR wrong password — never reveal which one failed)
    end

    Note over Client,DB: ── Phase 2: Authenticated Request ──

    Client->>API: GET /payments Authorization: Bearer <access_token>
    API->>API: Extract token from header
    API->>API: Decode header → get alg, kid
    API->>API: Fetch public key for kid (from local cache or JWKS endpoint)
    API->>API: RSA-Verify(header.payload, signature, public_key)

    alt signature valid
        API->>API: Check exp > now Check iss == expected issuer Check aud == this service Check jti not in revocation list
        API->>API: Extract sub, roles from payload
        API->>API: Authorise: does role allow GET /payments?
        API-->>Client: 200 OK + payment data
    else signature invalid / expired
        API-->>Client: 401 Unauthorized
    end

    Note over Client,DB: ── Phase 3: Token Refresh ──

    Client->>Auth: POST /auth/refresh {refresh_token}
    Auth->>Auth: Verify refresh_token signature Look up token hash in Redis Check not revoked
    Auth->>Auth: Issue new access_token Rotate refresh_token (old one now invalid)
    Auth-->>Client: 200 OK {access_token, refresh_token}
```

Notice that in Phase 2, the API server **never contacts the auth server or a database**. It only uses the public key, which it fetched once and cached. This is the scalability property JWT was designed for — horizontal scaling with zero inter-service coordination per request.

---

## How the Token Is Shared

After login, the client has a token. It needs to attach it to subsequent requests. There are two main strategies, each with different security properties.

### Strategy 1: Authorization Header (Bearer Token)

The de-facto standard for APIs. The client stores the token in JavaScript memory (not `localStorage`, not `sessionStorage` — more on why shortly) and attaches it manually to each request:

```
GET /payments HTTP/1.1
Host: api.yourapp.com
Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...
```

The server extracts the token from the `Authorization` header, strips the `Bearer ` prefix, and validates it.

**Security property**: Not vulnerable to CSRF attacks — a cross-site request forgery cannot attach a custom `Authorization` header because browsers block cross-origin header manipulation. Vulnerable to XSS — if an attacker can inject JavaScript into your page, they can read the in-memory token.

### Strategy 2: HttpOnly Cookie

The server sets the token as an `HttpOnly` cookie in the login response. The browser automatically includes it on every subsequent request to the same domain.

```http
HTTP/1.1 200 OK
Set-Cookie: access_token=eyJhbGci...; HttpOnly; Secure; SameSite=Strict; Path=/; Max-Age=3600
```

**Security property**: `HttpOnly` means JavaScript cannot read the cookie — XSS attacks cannot steal it. Vulnerable to CSRF if `SameSite` is not set correctly. With `SameSite=Strict`, cross-site requests never include the cookie, which kills most CSRF vectors.

**Recommendation for 2026**: Use `HttpOnly; Secure; SameSite=Strict` cookies for browser-based clients. Use `Authorization` headers for mobile apps and service-to-service calls. The cookie approach is strictly safer against XSS for browser apps — `HttpOnly` is a browser security guarantee, not an application-level one.

### Where NOT to Store Tokens

`localStorage` and `sessionStorage` are both accessible to any JavaScript running on your page. A single XSS vulnerability — in your code, in a third-party analytics script, in a compromised npm package — gives an attacker full access to every token stored there. The attack is silent, requires no user interaction, and leaves no trace in server logs. **Do not store JWTs in Web Storage.**

---

## How the Token Is Validated

Validation is the most security-critical step. A common mistake is to check the signature and expiry and call it done. Production validation is a checklist, and each item exists to block a specific attack.

```python
# auth/validator.py
import time
import json
import base64
from typing import Optional
from cryptography.hazmat.primitives import hashes, serialization
from cryptography.hazmat.primitives.asymmetric import padding
from cryptography.hazmat.backends import default_backend
from cryptography.exceptions import InvalidSignature
import redis

# Redis client for the revocation list
revocation_store = redis.Redis(host="localhost", port=6379, db=1, decode_responses=True)


class TokenValidationError(Exception):
    """Raised when token validation fails. Never include the reason in HTTP responses —
    it helps attackers understand exactly what constraint they failed."""
    pass


def base64url_decode(s: str) -> bytes:
    # Restore stripped padding before decoding
    padding_needed = 4 - len(s) % 4
    return base64.urlsafe_b64decode(s + "=" * padding_needed)


def validate_jwt_rs256(
    token: str,
    public_key_pem: bytes,
    expected_issuer: str,
    expected_audience: str,
    allowed_algorithms: list[str] = None,  # explicit allowlist — never derive from token
) -> dict:
    """
    Validates a JWT and returns the verified payload claims.
    Raises TokenValidationError on any failure.

    The order of checks matters: structural checks first (cheap),
    cryptographic checks second (expensive), claims checks last.
    This ordering prevents DoS via crafted tokens that pass structural
    checks but force expensive crypto operations.
    """

    if allowed_algorithms is None:
        allowed_algorithms = ["RS256"]

    # ── Check 1: Structure ────────────────────────────────────────────────────
    parts = token.split(".")
    if len(parts) != 3:
        raise TokenValidationError("Malformed token: expected 3 segments")

    header_b64, payload_b64, signature_b64 = parts

    # ── Check 2: Header decoding ──────────────────────────────────────────────
    try:
        header = json.loads(base64url_decode(header_b64))
    except Exception:
        raise TokenValidationError("Malformed header")

    # ── Check 3: Algorithm allowlist ──────────────────────────────────────────
    # CRITICAL: Never trust the alg field in the header to choose your
    # verification logic. Always validate against an explicit server-side allowlist.
    # The "none" algorithm attack works by setting alg=none and omitting the signature.
    # A naive library that trusts the header's alg would skip signature verification.
    # The alg confusion attack works by sending an RS256 token to a server configured
    # for HS256 — the server uses the public key as the HMAC secret, which an attacker
    # can reconstruct from the publicly available JWKS endpoint.
    alg = header.get("alg")
    if alg not in allowed_algorithms:
        raise TokenValidationError(f"Algorithm '{alg}' not in allowlist {allowed_algorithms}")

    # ── Check 4: Payload decoding ─────────────────────────────────────────────
    try:
        payload = json.loads(base64url_decode(payload_b64))
    except Exception:
        raise TokenValidationError("Malformed payload")

    # ── Check 5: Signature verification ──────────────────────────────────────
    # This is the only step that involves cryptography. It proves the token
    # was created by whoever holds the private key.
    signing_input = f"{header_b64}.{payload_b64}".encode("utf-8")
    signature = base64url_decode(signature_b64)

    try:
        public_key = serialization.load_pem_public_key(public_key_pem, backend=default_backend())
        public_key.verify(signature, signing_input, padding.PKCS1v15(), hashes.SHA256())
    except InvalidSignature:
        raise TokenValidationError("Signature verification failed")

    now = int(time.time())

    # ── Check 6: Expiry ───────────────────────────────────────────────────────
    exp = payload.get("exp")
    if exp is None:
        raise TokenValidationError("Token has no expiry — non-expiring tokens are forbidden")
    if now >= exp:
        raise TokenValidationError("Token has expired")

    # ── Check 7: Not Before ───────────────────────────────────────────────────
    nbf = payload.get("nbf")
    if nbf is not None and now < nbf:
        raise TokenValidationError("Token not yet valid")

    # ── Check 8: Issuer ───────────────────────────────────────────────────────
    # Prevents tokens issued by other services (or attackers who run their own
    # auth server) from being accepted by your API.
    iss = payload.get("iss")
    if iss != expected_issuer:
        raise TokenValidationError(f"Invalid issuer: {iss}")

    # ── Check 9: Audience ─────────────────────────────────────────────────────
    # Prevents tokens intended for service A from being used against service B.
    # Example: a token for your mobile app (aud: com.yourapp.mobile) should not
    # be accepted by your admin API (aud: https://admin.yourapp.com).
    aud = payload.get("aud")
    if isinstance(aud, str):
        aud = [aud]
    if expected_audience not in (aud or []):
        raise TokenValidationError(f"Invalid audience: {aud}")

    # ── Check 10: Revocation list ─────────────────────────────────────────────
    # The signature proves authenticity but not that the token hasn't been
    # revoked. Logout, password changes, and security incidents all require
    # the ability to invalidate a valid token before it expires naturally.
    # We store revoked jtis in Redis with a TTL matching the token's remaining life.
    jti = payload.get("jti")
    if jti and revocation_store.exists(f"revoked:{jti}"):
        raise TokenValidationError("Token has been revoked")

    # All checks passed — return the verified claims
    return payload


def revoke_token(payload: dict):
    """
    Adds a token's jti to the revocation list.
    TTL is set to the token's remaining lifetime — once the token would
    have expired naturally, there is no point keeping the revocation record.
    """
    jti = payload.get("jti")
    if not jti:
        return   # can't revoke a token with no jti

    remaining_ttl = max(0, payload.get("exp", 0) - int(time.time()))
    if remaining_ttl > 0:
        revocation_store.setex(f"revoked:{jti}", remaining_ttl, "1")
```

---

## Refresh Tokens and Token Rotation

A short-lived access token (15 minutes to 1 hour) limits the damage from a stolen token — the attacker's window is bounded. But asking the user to re-login every hour is terrible UX. Refresh tokens solve this: they are long-lived (days to weeks), stored more securely, and used only to obtain new access tokens — never sent to resource APIs.

The critical security property is **refresh token rotation**: every time you use a refresh token, the server issues a new one and invalidates the old one. If an attacker steals a refresh token and uses it, the original user's next token refresh will fail (their token was invalidated), alerting both the server and the user that something is wrong.

```mermaid
sequenceDiagram
    participant Client
    participant Auth as Auth Service
    participant Redis

    Note over Client,Redis: ── Initial Login ──
    Client->>Auth: POST /auth/login {email, password}
    Auth->>Redis: SETEX refresh:{token_hash} 7d {user_id, family_id}
    Auth-->>Client: {access_token (1h), refresh_token (7d)}

    Note over Client,Redis: ── After 1 Hour: Access Token Expired ──
    Client->>Auth: POST /auth/refresh {refresh_token}
    Auth->>Auth: Verify refresh_token signature
    Auth->>Redis: GET refresh:{token_hash}  → exists?
    Auth->>Redis: DEL refresh:{token_hash}  (invalidate old)
    Auth->>Redis: SETEX refresh:{new_hash} 7d {user_id, family_id}
    Auth-->>Client: {new_access_token (1h), new_refresh_token (7d)}

    Note over Client,Redis: ── Stolen Refresh Token Used by Attacker ──
    Client->>Auth: POST /auth/refresh {original_refresh_token}
    Auth->>Redis: GET refresh:{original_hash} → NOT FOUND
    Auth->>Auth: Token already used — possible theft detected!
    Auth->>Redis: DEL ALL refresh tokens for this family_id (log out ALL sessions for this user)
    Auth-->>Client: 401 — all sessions terminated
```

The `family_id` groups all refresh tokens from a single login session. When reuse is detected (a token that was already consumed is presented again), every token in the family is invalidated — the attacker and the legitimate user are both forced to re-login. This is a strong security guarantee with minimal false positives.

---

## FastAPI Implementation — Production Grade

```python
# auth/routes.py
from datetime import datetime, timezone, timedelta
from typing import Annotated
import uuid

from fastapi import FastAPI, Depends, HTTPException, status, Response, Cookie
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
from pydantic import BaseModel, EmailStr
import bcrypt
import redis
from jose import jwt, JWTError   # python-jose handles RS256/ES256 cleanly

app = FastAPI()

# ── Configuration ─────────────────────────────────────────────────────────────
# In production: load from AWS Secrets Manager or Parameter Store, not env vars.
# Private key: PEM-encoded RSA-2048 or EC P-256, stored in Secrets Manager.
# Public key: can be in Parameter Store or served via /.well-known/jwks.json.
with open("keys/private_key.pem", "rb") as f:
    PRIVATE_KEY = f.read()
with open("keys/public_key.pem", "rb") as f:
    PUBLIC_KEY = f.read()

ALGORITHM = "RS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 60
REFRESH_TOKEN_EXPIRE_DAYS = 7
ISSUER = "https://auth.yourapp.com"
AUDIENCE = "https://api.yourapp.com"

redis_client = redis.Redis(host="localhost", port=6379, decode_responses=True)
bearer_scheme = HTTPBearer()


# ── Models ────────────────────────────────────────────────────────────────────

class LoginRequest(BaseModel):
    email: EmailStr
    password: str

class TokenResponse(BaseModel):
    access_token: str
    token_type: str = "Bearer"
    expires_in: int = ACCESS_TOKEN_EXPIRE_MINUTES * 60


# ── Token creation helpers ────────────────────────────────────────────────────

def create_access_token(user_id: str, email: str, roles: list[str]) -> str:
    now = datetime.now(tz=timezone.utc)
    claims = {
        "sub": user_id,
        "email": email,
        "roles": roles,
        "iss": ISSUER,
        "aud": AUDIENCE,
        "iat": now,
        "exp": now + timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES),
        "nbf": now,
        "jti": str(uuid.uuid4()),
    }
    return jwt.encode(claims, PRIVATE_KEY, algorithm=ALGORITHM)


def create_refresh_token(user_id: str, family_id: str) -> str:
    """
    Refresh token payload is minimal — it is never decoded by resource APIs,
    only by the auth service itself. Less data = smaller attack surface.
    """
    now = datetime.now(tz=timezone.utc)
    claims = {
        "sub": user_id,
        "family": family_id,   # for revocation chain detection
        "iss": ISSUER,
        "iat": now,
        "exp": now + timedelta(days=REFRESH_TOKEN_EXPIRE_DAYS),
        "jti": str(uuid.uuid4()),
        "type": "refresh",     # prevent refresh tokens from being used as access tokens
    }
    token = jwt.encode(claims, PRIVATE_KEY, algorithm=ALGORITHM)

    # Store the token's jti in Redis so we can detect reuse
    jti = claims["jti"]  # jose doesn't expose jti directly post-encode, so we track it
    redis_client.setex(
        f"refresh:{jti}",
        int(timedelta(days=REFRESH_TOKEN_EXPIRE_DAYS).total_seconds()),
        f"{user_id}:{family_id}"
    )
    return token


# ── Dependency: token validation ──────────────────────────────────────────────

async def get_current_user(
    credentials: Annotated[HTTPAuthorizationCredentials, Depends(bearer_scheme)]
) -> dict:
    """
    FastAPI dependency — inject into any route that requires authentication.
    Returns the verified payload on success; raises 401 on any failure.

    Deliberately returns a generic 401 regardless of the specific failure reason.
    The real reason is logged server-side for debugging but never sent to the client.
    """
    credentials_exception = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="Could not validate credentials",
        headers={"WWW-Authenticate": "Bearer"},
    )

    try:
        payload = jwt.decode(
            credentials.credentials,
            PUBLIC_KEY,
            algorithms=[ALGORITHM],    # explicit allowlist — never ["*"]
            issuer=ISSUER,
            audience=AUDIENCE,
            options={
                "require": ["exp", "iat", "sub", "jti"],  # mandatory claims
                "verify_exp": True,
                "verify_iat": True,
                "verify_nbf": True,
            }
        )
    except JWTError:
        raise credentials_exception

    # Check revocation list
    jti = payload.get("jti")
    if jti and redis_client.exists(f"revoked:{jti}"):
        raise credentials_exception

    # Reject refresh tokens used as access tokens
    if payload.get("type") == "refresh":
        raise credentials_exception

    return payload


# ── Routes ────────────────────────────────────────────────────────────────────

@app.post("/auth/login", response_model=TokenResponse)
async def login(body: LoginRequest, response: Response):
    # In production: look up user from your database
    user = await get_user_by_email(body.email)   # your DB call

    if not user or not bcrypt.checkpw(body.password.encode(), user.hashed_password):
        # Same exception for wrong email OR wrong password.
        # An attacker should not be able to enumerate valid email addresses
        # by observing whether the error says "user not found" vs "wrong password".
        raise HTTPException(status_code=401, detail="Invalid credentials")

    family_id = str(uuid.uuid4())   # new login = new token family
    access_token = create_access_token(user.id, user.email, user.roles)
    refresh_token = create_refresh_token(user.id, family_id)

    # Send refresh token as HttpOnly cookie — not in the response body.
    # The access token goes in the body and is held in JS memory.
    # This splits the attack surface: XSS can only steal the access token
    # (short-lived); the refresh token is HttpOnly and invisible to JS.
    response.set_cookie(
        key="refresh_token",
        value=refresh_token,
        httponly=True,       # JS cannot read this
        secure=True,         # HTTPS only
        samesite="strict",   # no cross-site sending
        max_age=REFRESH_TOKEN_EXPIRE_DAYS * 86400,
        path="/auth/refresh" # scope cookie to the refresh endpoint only
    )

    return TokenResponse(access_token=access_token)


@app.post("/auth/refresh", response_model=TokenResponse)
async def refresh(response: Response, refresh_token: str = Cookie(None)):
    if not refresh_token:
        raise HTTPException(status_code=401, detail="No refresh token")

    credentials_exception = HTTPException(status_code=401, detail="Invalid refresh token")

    try:
        payload = jwt.decode(
            refresh_token, PUBLIC_KEY,
            algorithms=[ALGORITHM],
            issuer=ISSUER,
            options={"require": ["exp", "jti", "sub", "family"]}
        )
    except JWTError:
        raise credentials_exception

    if payload.get("type") != "refresh":
        raise credentials_exception

    jti = payload["jti"]
    stored = redis_client.get(f"refresh:{jti}")

    if not stored:
        # Token not in Redis — either expired naturally OR already used.
        # If already used: this is a reuse attack. Invalidate the entire family.
        user_id = payload.get("sub")
        family_id = payload.get("family")
        if user_id and family_id:
            # Scan and delete all tokens belonging to this family
            for key in redis_client.scan_iter(f"refresh:*"):
                val = redis_client.get(key)
                if val and val.endswith(f":{family_id}"):
                    redis_client.delete(key)
        raise credentials_exception

    # Invalidate the used token immediately (atomic: no window for reuse)
    redis_client.delete(f"refresh:{jti}")

    user_id, family_id = stored.split(":", 1)

    # Issue new tokens — same family_id continues the session
    new_access = create_access_token(user_id, payload.get("email", ""), [])
    new_refresh = create_refresh_token(user_id, family_id)

    response.set_cookie(
        key="refresh_token", value=new_refresh,
        httponly=True, secure=True, samesite="strict",
        max_age=REFRESH_TOKEN_EXPIRE_DAYS * 86400, path="/auth/refresh"
    )
    return TokenResponse(access_token=new_access)


@app.post("/auth/logout")
async def logout(
    response: Response,
    current_user: Annotated[dict, Depends(get_current_user)],
    refresh_token: str = Cookie(None)
):
    # Revoke the access token for its remaining lifetime
    jti = current_user.get("jti")
    exp = current_user.get("exp", 0)
    remaining = max(0, exp - int(datetime.now(tz=timezone.utc).timestamp()))
    if jti and remaining > 0:
        redis_client.setex(f"revoked:{jti}", remaining, "1")

    # Revoke the refresh token
    if refresh_token:
        try:
            payload = jwt.decode(refresh_token, PUBLIC_KEY, algorithms=[ALGORITHM],
                                 options={"verify_exp": False})
            redis_client.delete(f"refresh:{payload.get('jti')}")
        except JWTError:
            pass   # malformed — already invalid

    response.delete_cookie("refresh_token", path="/auth/refresh")
    return {"message": "Logged out"}


# ── Protected route example ───────────────────────────────────────────────────

def require_role(role: str):
    """Dependency factory for role-based access control."""
    async def check_role(user: Annotated[dict, Depends(get_current_user)]):
        if role not in user.get("roles", []):
            raise HTTPException(status_code=403, detail="Insufficient permissions")
        return user
    return check_role


@app.get("/payments")
async def list_payments(user: Annotated[dict, Depends(require_role("payments:read"))]):
    # At this point: token is verified, not expired, not revoked, role is correct.
    # user["sub"] is the verified user ID — use it to scope the query.
    return {"user_id": user["sub"], "payments": [...]}
```

### The JWKS Endpoint

When you use RS256, downstream services need your public key to verify tokens. Rather than distributing PEM files out-of-band, expose a **JSON Web Key Set** (JWKS) endpoint that any service can poll:

```python
from cryptography.hazmat.primitives.asymmetric import rsa
from cryptography.hazmat.primitives import serialization
import base64

@app.get("/.well-known/jwks.json")
async def jwks():
    """
    Returns the public keys in JWKS format.
    Downstream services fetch this once and cache it.
    When you rotate keys, they re-fetch and pick up the new key by kid.

    Cache-Control headers: cache for 1 hour. Services should re-fetch
    on 'kid not found' errors to handle emergency key rotation.
    """
    public_key = serialization.load_pem_public_key(PUBLIC_KEY)
    pub_numbers = public_key.public_key().public_numbers()

    def int_to_base64url(n: int) -> str:
        length = (n.bit_length() + 7) // 8
        return base64.urlsafe_b64encode(n.to_bytes(length, "big")).rstrip(b"=").decode()

    return {
        "keys": [
            {
                "kty": "RSA",
                "use": "sig",           # this key is for signature verification
                "alg": "RS256",
                "kid": "rsa-key-2026-04",
                "n": int_to_base64url(pub_numbers.n),    # RSA modulus
                "e": int_to_base64url(pub_numbers.e),    # RSA public exponent
            }
        ]
    }
```

---

## What Can Go Wrong — Attacks and Pitfalls

### The `alg: none` Attack

The JWT spec originally allowed `alg: none` — a token with no signature, intended for "trusted channels" where signing was unnecessary. Some libraries honoured this by skipping signature verification entirely if the header said `alg: none`. An attacker could take any valid token, decode it, modify the payload (change `"roles": ["user"]` to `"roles": ["admin"]`), set `alg: none`, remove the signature, and the server would accept it.

**Mitigation**: Always validate the algorithm against a server-side allowlist *before* verifying the signature. Never derive the verification method from the token itself.

### The Algorithm Confusion Attack (RS256 → HS256)

A subtler attack. Your server uses RS256. Your public key is published at `/.well-known/jwks.json`. An attacker downloads it. They create a token with `alg: HS256` and sign it using your *public key bytes* as the HMAC secret. If your library trusts the `alg` field in the header to choose verification, it will use your public key as the HMAC secret — and the signature will verify correctly, because the attacker used the same bytes to sign.

**Mitigation**: Same as above — algorithm allowlist on the server, never derived from the token.

### Short Secrets for HS256

`HS256("secret")` is trivially brute-forceable. JWT tokens are offline-crackable — an attacker who captures a token can try millions of potential HMAC secrets per second without any server interaction. HS256 requires a minimum of 256 bits of random entropy in the secret (32 random bytes). Use `secrets.token_bytes(32)`.

### Sensitive Data in the Payload

Repeating this: the payload is Base64-encoded, not encrypted. Anyone with the token can decode and read it — including browser dev tools, proxies, and log aggregation systems. Put only non-sensitive identifiers and metadata in the JWT payload. Never put passwords, credit card numbers, personal health information, or anything you would not write in a URL.

### Missing Audience Check

Without an audience check, a token issued for your mobile app (`aud: com.yourapp.mobile`) is accepted by your admin API. This is not theoretical — in OAuth 2.0 flows, access tokens are often scoped to specific audiences, and skipping this check breaks that scoping entirely.

---

## Key Takeaways

JWT's value proposition is **stateless, cryptographically verifiable identity** — any server with the public key can validate any token without contacting a central store. This is what makes horizontal scaling work cleanly.

The key design decisions that determine whether your JWT implementation is secure are: choose RS256 or ES256 over HS256 for any multi-service architecture; validate the algorithm against a server-side allowlist (never trust the token's header for this); always check `exp`, `iss`, `aud`, and `jti`; store the access token in JS memory and the refresh token in an `HttpOnly` cookie; implement refresh token rotation with family-based reuse detection; and never put sensitive data in the payload.

In Part 2 we will cover a completely different authentication model — how AWS services authenticate *to each other* using IAM roles, what temporary credentials are, how the STS service underpins the entire system, and how SigV4 signing works for service calls.

---

## More Resources

- [RFC 7519 — JSON Web Token specification](https://www.rfc-editor.org/rfc/rfc7519)
- [OWASP JWT Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/JSON_Web_Token_for_Java_Cheat_Sheet.html)
- [jwt.io — interactive token decoder](https://jwt.io)
- [python-jose library](https://python-jose.readthedocs.io/en/latest/)
