# Authentication Workflows

V2Hub has no self-service signup, login endpoint, or token-refresh call in the client — every credential is issued out of band by an administrator and handed to your application. This guide covers the three credential types in play across the ecosystem and the workflows around each.

## The three credential types

| Credential             | Used by                                          | Header/mechanism                                                      | Obtained via                                                                   |
| ---------------------- | ------------------------------------------------ | --------------------------------------------------------------------- | ------------------------------------------------------------------------------ |
| Regular user API token | `v2hub` (`AsyncVPNClient`/`VPNClient`)           | `API-Token` header                                                    | Issued by an admin; see [Administration Workflows](administration.md)          |
| Provider API token     | `v2hub` acting with `as_provider_for_user_id=`   | `API-Token` header                                                    | Same as above, created as a _provider_ account                                 |
| Admin secret key       | `v2hub-admin` (`AsyncAdminClient`/`AdminClient`) | HMAC-SHA256 request signature (`X-Signature` / `X-Timestamp` headers) | Configured directly on the API deployment; never issued through the API itself |

A **regular token** and a **provider token** are the same kind of credential (`API-Token` header) — the difference is which kind of account the token belongs to, not how it's transmitted. The **admin secret key** is structurally different: it's never sent as a bearer-style header. Instead, each request is signed with `HMAC-SHA256(secret_key, timestamp + method + path + body)`, and the signature plus timestamp go in `X-Signature`/`X-Timestamp`. This happens automatically inside `v2hub-admin`'s client — you never construct the signature yourself.

## Regular and provider tokens

```python
from v2hub import AsyncVPNClient

client = AsyncVPNClient(base_url="https://api.example.com", api_token="your-api-token")
```

Passing a provider token here doesn't change anything by default — a provider client behaves exactly like a regular client until you explicitly pass `as_provider_for_user_id=<user_id>` on a subscription or source call. See [Provider Workflows](providers.md) for that flow, and [V2Hub → Authentication](../v2hub/installation.md#authentication) for the underlying model.

### Where tokens come from

If you're integrating against a V2Hub instance someone else administers, ask them for a token — there is no signup flow to automate. If you administer the instance yourself, provision tokens via `v2hub-admin` or `v2hub admin` (the CLI):

```python
from v2hub_admin import AsyncAdminClient

async with AsyncAdminClient(base_url, secret_key="admin-secret") as admin:
    user = await admin.create_user(user_id=12345)
    print(user.api_token)  # hand this to the end-user's application
```

```bash
v2hub admin create-user 12345
```

See [Administration Workflows](administration.md#user-management) for the full user/provider lifecycle.

### Rotating a token

If a token is compromised or you rotate credentials on a schedule, refresh it through the admin client rather than trying to change it client-side (there is no self-service rotation endpoint):

```python
result = await admin.refresh_token(user_id=12345)
print(result.new_api_token)  # update your application's stored token to this value
```

The old token stops working the moment the new one is issued — plan for a brief coordinated update (store the new token, then redeploy or hot-reload the consuming application) rather than a gradual rollover.

## Admin secret key

```python
from v2hub_admin import AsyncAdminClient

async with AsyncAdminClient(base_url="https://api.example.com", secret_key="your-hmac-secret") as admin:
    user = await admin.get_user(12345)
```

Treat `secret_key` with more care than a regular API token: per the `v2hub-admin` security notes, it grants full administrative access to the system. Don't hardcode it in source; load it from an environment variable or secret manager, use HTTPS only in production, and rotate it periodically. The CLI equivalent reads it from `V2HUB_ADMIN_SECRET` or `--secret-key` — see [V2Hub CLI → Configuration & Authentication](../v2hub-cli/configuration.md).

## Provider authorization (a separate, per-user handshake)

Don't confuse **authentication** (the token/secret that lets _your process_ call the API at all) with **provider authorization** (whether a specific provider account is allowed to manage a specific end-user's subscriptions). The latter is a per-relationship approval flow, not a credential:

1. A provider requests access to a user's account (`create_provider_connection` / the admin-side `process_provider_authorization`).
2. The user (or an admin, depending on your flow) approves or rejects it.
3. Once approved, calls made with `as_provider_for_user_id=<user_id>` succeed; otherwise they fail with an authorization error.

This is covered in depth in [Provider Workflows](providers.md#connection-lifecycle) and, from the admin side, in [Administration Workflows](administration.md#provider-authorization-management).

## Handling authentication failures

An invalid or expired API token raises `AuthenticationError`; an invalid HMAC signature (usually a wrong or stale `secret_key`) raises the same exception from `v2hub-admin`, since it shares `v2hub`'s exception hierarchy:

```python
from v2hub import AuthenticationError

try:
    await client.get_subscription(token)
except AuthenticationError:
    # Token is invalid/expired — surface a re-auth prompt, alert an operator,
    # or fail the request with a clear message. Don't silently retry: this
    # is not a transient error and retrying won't help.
    ...
```

See [Error Handling Patterns](error-handling.md) for how this fits into a broader error-handling strategy, and [V2Hub → Error Handling](../v2hub/errors.md) for the full exception reference.
