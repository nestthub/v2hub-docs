# User Management

Every method below exists identically on both `AsyncAdminClient` (as a coroutine) and `AdminClient` (as a plain method) — examples use the sync client for brevity.

## `create_user(user_id)`

Create a new user account.

```python
user = admin.create_user(user_id=12345)
print(user.api_token)
```

| Argument | Type | Description |
| --- | --- | --- |
| `user_id` | `int` | External user ID; must be positive |

**Returns:** `UserCreateResponse` — see [User Models](#user-models).

**Raises:** `ValidationError` (invalid `user_id`), `ConflictError` (user already exists), `AuthenticationError`.

## `get_user(user_id)`

Get user info.

```python
user = admin.get_user(12345)
print(user.is_active)
```

**Returns:** `UserResponse`.

**Raises:** `NotFoundError`, `AuthenticationError`.

## `delete_user(user_id)`

Delete a user account.

```python
admin.delete_user(12345)
```

**Returns:** `None`.

**Raises:** `NotFoundError`, `AuthenticationError`.

## `set_user_status(user_id, is_active)`

Activate or deactivate a user.

```python
user = admin.set_user_status(12345, False)
print(user.is_active)  # False
```

| Argument | Type | Description |
| --- | --- | --- |
| `user_id` | `int` | External user ID |
| `is_active` | `bool` | `True` to activate, `False` to deactivate |

**Returns:** `UserResponse` (updated).

**Raises:** `NotFoundError`, `AuthenticationError`.

## `refresh_token(user_id)`

Issue a new API token for the user, invalidating the previous one.

```python
result = admin.refresh_token(user_id=12345)
print(result.new_api_token)
```

**Returns:** `TokenRefreshResponse`.

**Raises:** `NotFoundError`, `AuthenticationError`.

## `get_user_providers(user_id)`

List a user's provider connections and their authorization statuses.

```python
connections = admin.get_user_providers(12345)
for conn in connections.connections:
    print(conn.provider_name, conn.status)
```

**Returns:** `ConnectionsResponse` (from `v2hub.models` — see [Typed Models](../v2hub/models.md)).

**Raises:** `NotFoundError`, `AuthenticationError`.

## `get_user_provider(user_id, provider_name)`

Get one specific provider connection for a user.

```python
conn = admin.get_user_provider(12345, "vpn123")
print(conn.status)
```

**Returns:** `ConnectionResponse` (from `v2hub.models`).

**Raises:** `NotFoundError`, `AuthenticationError`.

## User Models

### `UserCreateRequest`

| Field | Type | Description |
| --- | --- | --- |
| `user_id` | `int` (`> 0`) | External user ID |

### `UserResponse` / `UserCreateResponse`

`UserCreateResponse` extends `UserResponse` with no additional fields — the same shape is returned by both `create_user()` and `get_user()`/`set_user_status()`.

| Field | Type | Description |
| --- | --- | --- |
| `user_hash` | `str` | Generated user hash |
| `user_id` | `int` | User ID |
| `api_token` | `str` | Current API token |
| `is_active` | `bool` | Account status |

### `UserStatusUpdateRequest`

| Field | Type | Description |
| --- | --- | --- |
| `is_active` | `bool` | New account status |

### `TokenRefreshRequest`

| Field | Type | Description |
| --- | --- | --- |
| `user_id` | `int` (`> 0`) | User ID |

### `TokenRefreshResponse`

| Field | Type | Description |
| --- | --- | --- |
| `user_id` | `int` | User ID |
| `new_api_token` | `str` | Newly issued API token |

## Related

- [Provider Management](provider-management.md) — the provider-side equivalent of these operations.
- [Provider Authorization](provider-authorization.md) — connecting a user to a provider once both exist.
- [`v2hub-cli` User Commands](../v2hub-cli/admin-commands.md#user-management) — the CLI wrapper around this API.
