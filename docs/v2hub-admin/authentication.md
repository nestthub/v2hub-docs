# Authentication & Authorization

The admin client authenticates every request with an **HMAC-SHA256 signature**, computed from a shared **secret key** — a fundamentally different mechanism from the bearer-style API token used by the regular [`v2hub`](../v2hub/index.md) client and by [`v2hub-cli`](../v2hub-cli/configuration.md)'s regular/provider commands.

## The Secret Key

```python
AdminClient(base_url="https://api.example.com", secret_key="your-hmac-secret")
```

The `secret_key` is never sent over the wire directly. Instead, it's used locally to compute a signature for each request, which the server recomputes and compares. This means:

- The secret key must be identical on both the client and the server (it's a shared secret, not a token issued per-session).
- An intercepted request cannot be replayed to forge a new one, because the signature is bound to the request's method, path, body, and timestamp (see [How Signing Works](#how-signing-works)).
- The Admin API grants full administrative access to the system — treat this key like a root credential. See [Security Notes](index.md#security-notes).

## How Signing Works

Every request is signed by `AdminAuthenticator.sign_request(method, path, body)`, internal to the client:

1. A **timestamp** is generated in milliseconds since the epoch: `str(int(time.time() * 1000))`.
2. A **payload** string is built by concatenating, in order: `timestamp + method + path + body` (the JSON-encoded request body, or an empty string for bodyless requests).
3. The **signature** is `HMAC-SHA256(secret_key, payload)`, hex-encoded.
4. Three headers are attached to the request:

| Header | Value |
| --- | --- |
| `X-Signature` | The hex-encoded HMAC-SHA256 signature |
| `X-Timestamp` | The timestamp used in the payload, in milliseconds |
| `Content-Type` | `application/json` |

This all happens automatically inside `_request()` — every public method on `AsyncAdminClient`/`AdminClient` calls it, so you never construct these headers yourself.

```python
from v2hub_admin.auth import AdminAuthenticator

auth = AdminAuthenticator("secret-key")
headers = auth.sign_request("POST", "/api/v1/admin/users", '{"user_id":123}')
print(headers["X-Signature"])
```

## Timestamp Validation

Signed requests are time-bound: the server is expected to reject requests whose timestamp is too old, which limits the window during which a captured request (and its signature) could be replayed. The client exposes the same check used for this, `AdminAuthenticator.verify_timestamp(timestamp, max_age_seconds=300)` (5 minutes by default), primarily useful if you're implementing or testing a compatible server-side verifier rather than as part of normal client usage.

## Query Parameters and Signing

For `GET` requests with query parameters (e.g. [`get_stats()`](usage-statistics.md)), the parameters are URL-encoded onto the path *before* signing — so the signature covers the full path including the query string, not just the bare endpoint path. `None`-valued parameters are dropped rather than encoded as empty strings.

## Errors

Authentication and authorization errors surface through the same typed exception hierarchy as the rest of `v2hub` — see [Error Handling](../v2hub/errors.md) for the full table. The two most relevant here:

| Exception | Cause |
| --- | --- |
| `AuthenticationError` | The signature is invalid or missing — usually an incorrect `secret_key`, or a request signed against the wrong path/body |
| `AuthorizationError` | The signature is valid, but the account doesn't have admin privileges |

```python
from v2hub import AuthenticationError, AuthorizationError, VPNAPIError

try:
    admin.delete_user(12345)
except AuthenticationError:
    print("Invalid HMAC signature")
except AuthorizationError:
    print("No admin privileges")
except VPNAPIError as e:
    print(e)
```

## Secret Key vs. API Token

| | Regular / provider (`v2hub`) | Admin (`v2hub-admin`) |
| --- | --- | --- |
| Credential | API token (bearer-style) | HMAC secret key |
| Sent as | `Authorization`-style header, as-is | Never sent directly — used to derive `X-Signature`/`X-Timestamp` per request |
| Scope | A specific user or provider account | Full administrative access |
| Rotated via | `refresh_token()` (user) / `refresh_provider_token()` (provider) — see [User Management](user-management.md) / [Provider Management](provider-management.md) | Managed out-of-band by whoever operates the deployment; not rotated via the API |

These two credentials are never interchangeable — passing an API token as `secret_key` (or vice versa) will fail signature verification and raise `AuthenticationError`.
