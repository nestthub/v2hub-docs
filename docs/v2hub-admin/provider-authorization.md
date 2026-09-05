# Provider Authorization

Providers request access to manage a specific user's subscriptions via an HMAC-signed connection invite. This page covers the admin-side authorization handshake: inspecting the current state, processing an incoming request, and approving or rejecting it.

## The Flow

1. A provider issues a connection invite to a user, producing an `hmac`.
2. The invite is submitted via `process_provider_authorization(user_id, provider_name, hmac)`, creating a `PENDING` authorization.
3. An admin approves or rejects it with `approve_provider_authorization()` / `reject_provider_authorization()`.
4. `get_provider_authorization()` can be used at any point to check the current status.

```python
# Create/process an authorization from an issued HMAC
auth = admin.process_provider_authorization(
    user_id=12345,
    provider_name="vpn123",
    hmac="a1b2c3d4e5f6...",
)
print(auth.status)  # "pending"

# Approve it (only PENDING authorizations can be approved)
auth = admin.approve_provider_authorization(user_id=12345, provider_name="vpn123")
print(auth.status)  # "approved"

# Check status later
auth = admin.get_provider_authorization("vpn123", 12345)
print(auth.status)

# Reject/revoke access
admin.reject_provider_authorization(user_id=12345, provider_name="vpn123")
```

!!! note
    `approve_provider_authorization()` and `reject_provider_authorization()` only operate on `PENDING` authorizations and raise `ConflictError` otherwise — including when the server's provider-per-user limit has been reached.

## `get_provider_authorization(provider_name, user_id)`

Get the current authorization status between a provider and a user.

| Argument | Type | Description |
| --- | --- | --- |
| `provider_name` | `str` | Provider name |
| `user_id` | `int` | Target user ID |

**Returns:** `ProviderAuthorizationInfoResponse` — see [Provider Authorization Models](#provider-authorization-models).

**Raises:** `NotFoundError` (provider, user, or authorization not found), `AuthenticationError`, `VPNAPIError`.

## `process_provider_authorization(user_id, provider_name, hmac=None)`

Process a provider authorization request. The HMAC is passed to the server as-is.

| Argument | Type | Description |
| --- | --- | --- |
| `user_id` | `int` | Target user ID |
| `provider_name` | `str` | Provider name |
| `hmac` | `str \| None` | Optional authorization HMAC. **Required** to create a new authorization; omit to query an existing one |

**Returns:** `ProviderAuthorizationInfoResponse`.

**Raises:** `AuthenticationError` (invalid admin secret key or HMAC), `NotFoundError` (provider not found), `VPNAPIError`.

## `approve_provider_authorization(user_id, provider_name)`

Approve a pending provider authorization. Only `PENDING` authorizations can be approved; the server enforces its provider-per-user limit when granting the authorization.

**Returns:** `ProviderAuthorizationInfoResponse` (approved).

**Raises:** `ConflictError` (authorization is not pending, or the user has reached the provider limit), `NotFoundError`, `AuthenticationError`, `VPNAPIError`.

## `reject_provider_authorization(user_id, provider_name)`

Reject a pending provider authorization.

**Returns:** `ProviderAuthorizationInfoResponse` — the resulting state. The `status` is `None` when the authorization record was deleted outright (no subscriptions existed under it), or `REVOKED` when the authorization had existing subscriptions and the record was kept instead.

**Raises:** `ConflictError` (authorization is not pending), `NotFoundError`, `AuthenticationError`, `VPNAPIError`.

## Provider Authorization Models

### `ProviderAuthorizationBaseRequest`

Shared fields identifying a provider/user authorization pair; the base for the other request models below.

| Field | Type | Description |
| --- | --- | --- |
| `user_id` | `int` | User ID |
| `provider_name` | `str` | Provider name |

### `ProviderAuthorizationInfoResponse`

Extends `ProviderAuthorizationBaseRequest`.

| Field | Type | Description |
| --- | --- | --- |
| `provider_url` | `str \| None` | Provider URL |
| `status` | `ProviderAuthorizationStatus \| None` | Current authorization status. `None` when no authorization record exists (e.g. after it has been deleted) |

`ProviderAuthorizationStatus` is defined in `v2hub.models` — see [Typed Models](../v2hub/models.md) for its full set of values (including `PENDING`, `APPROVED`, and `REVOKED`, referenced throughout this page).

### `ProviderAuthorizationRequest`

Extends `ProviderAuthorizationBaseRequest`. Used by `process_provider_authorization()`.

| Field | Type | Description |
| --- | --- | --- |
| `hmac` | `str \| None` | Authorization HMAC issued with a connection invite. Required to create a new authorization; omit to query an existing one |

### `ProviderAuthorizationDecisionRequest`

Extends `ProviderAuthorizationBaseRequest` with no additional fields. Used by `approve_provider_authorization()` and `reject_provider_authorization()` — the pair alone is enough to identify which pending authorization to decide on.

## Related

- [Provider Management](provider-management.md) — creating and managing the provider account itself.
- [User Management](user-management.md) — `get_user_providers()`/`get_user_provider()` for a read-only, user-centric view of connections.
- [`v2hub-cli` Provider Authorization Workflow](../v2hub-cli/admin-commands.md#provider-authorization-workflow) — the CLI wrapper around this API.
- [Connection Commands](../v2hub-cli/connection-commands.md) / [Provider Commands](../v2hub-cli/provider-commands.md#connection-lifecycle) — how end-users and providers manage their own side of this handshake, without admin privileges.
