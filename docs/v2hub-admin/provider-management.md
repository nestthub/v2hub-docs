# Provider Management

Providers are external services (bots, resellers, integrations) that manage subscriptions on behalf of end-users, via `v2hub`'s `as_provider_for_user_id=` argument (see [Sync & Async Clients](../v2hub/clients.md#providers)) or `v2hub-cli`'s [`v2hub provider <user_id> ...`](../v2hub-cli/provider-commands.md). This page covers the *admin*-side lifecycle of provider accounts themselves — creating them, rotating their tokens, enabling/disabling them, and looking them up. For connecting a provider to a specific user, see [Provider Authorization](provider-authorization.md).

## `create_provider(owner_hash, provider_name, provider_url=None)`

Create a new provider account.

```python
provider = admin.create_provider(
    owner_hash="a1b2c3d4e5f6...",
    provider_name="vpn123",
    provider_url="https://t.me/examplebot",
)
print(provider.provider_hash)
print(provider.api_token)
```

| Argument | Type | Description |
| --- | --- | --- |
| `owner_hash` | `str` | Hash of the user who owns the provider |
| `provider_name` | `str` | Unique provider name |
| `provider_url` | `str \| None` | Optional provider website or bot URL |

**Returns:** `ProviderCreateResponse` — see [Provider Models](#provider-models).

**Raises:** `ValidationError`, `ConflictError` (provider name or owner already exists), `AuthenticationError`.

## `get_providers()`

Get all providers, as a mapping of provider names to provider hashes.

```python
providers = admin.get_providers()
for name, provider_hash in providers.provider_hashes.items():
    print(name, provider_hash)
```

**Returns:** `AllProvidersResponse`.

**Raises:** `AuthenticationError`.

## `get_provider(provider_hash)`

Get provider information by its hash.

```python
provider = admin.get_provider("a1b2c3d4e5f6...")
print(provider.provider_name)
```

**Returns:** `ProviderResponse`.

**Raises:** `NotFoundError`, `AuthenticationError`.

## `get_provider_by_name(provider_name)`

Get provider information by provider name.

```python
provider = admin.get_provider_by_name("vpn123")
```

**Returns:** `ProviderResponse`.

**Raises:** `NotFoundError`, `AuthenticationError`, `VPNAPIError`.

## `get_provider_by_owner_id(owner_id)`

Get provider information by the owning user's ID.

```python
provider = admin.get_provider_by_owner_id(12345)
```

**Returns:** `ProviderResponse`.

**Raises:** `NotFoundError`, `AuthenticationError`, `VPNAPIError`.

## `delete_provider(provider_hash)`

Delete a provider account.

```python
admin.delete_provider("a1b2c3d4e5f6...")
```

**Returns:** `None`.

**Raises:** `NotFoundError`, `AuthenticationError`.

## `set_provider_status(provider_hash, is_active)`

Activate or deactivate a provider.

```python
provider = admin.set_provider_status(provider.provider_hash, False)
print(provider.is_active)  # False
```

**Returns:** `ProviderResponse` (updated).

**Raises:** `NotFoundError`, `AuthenticationError`.

## `update_provider_url(provider_hash, provider_url)`

Update a provider's URL. Pass `None` to clear it.

```python
admin.update_provider_url(provider.provider_hash, "https://t.me/newbot")
```

**Returns:** `ProviderResponse` (updated).

**Raises:** `NotFoundError`, `AuthenticationError`.

## `update_provider_name(provider_hash, provider_name)`

Rename a provider.

```python
admin.update_provider_name(provider.provider_hash, "new-name")
```

**Returns:** `ProviderResponse` (updated).

**Raises:** `NotFoundError`, `ConflictError` (name already taken), `AuthenticationError`.

## `refresh_provider_token(provider_hash)`

Issue a new API token for the provider, invalidating the previous one.

```python
result = admin.refresh_provider_token(provider.provider_hash)
print(result.new_api_token)
```

**Returns:** `ProviderTokenRefreshResponse`.

**Raises:** `NotFoundError`, `AuthenticationError`.

## Provider Models

### `ProviderCreateRequest`

| Field | Type | Description |
| --- | --- | --- |
| `owner_hash` | `str` | Provider's owner hash |
| `provider_name` | `str` | Provider name |
| `provider_url` | `str \| None` | Provider address URL |

### `ProviderResponse` / `ProviderCreateResponse`

`ProviderCreateResponse` extends `ProviderResponse` with no additional fields.

| Field | Type | Description |
| --- | --- | --- |
| `provider_hash` | `str` | Provider hash |
| `owner_hash` | `str` | Owner hash |
| `provider_name` | `str` | Provider name |
| `api_token` | `str` | Current API token |
| `provider_url` | `str \| None` | Provider URL |
| `is_active` | `bool` | Account status |

### `AllProvidersResponse`

| Field | Type | Description |
| --- | --- | --- |
| `provider_hashes` | `dict[str, str]` | Mapping of provider names to provider hashes |

### `ProviderStatusUpdateRequest`

| Field | Type | Description |
| --- | --- | --- |
| `is_active` | `bool` | New account status |

### `ProviderURLUpdateRequest`

| Field | Type | Description |
| --- | --- | --- |
| `provider_url` | `str \| None` | New provider URL |

### `ProviderNameUpdateRequest`

| Field | Type | Description |
| --- | --- | --- |
| `provider_name` | `str` | New provider name |

### `ProviderTokenRefreshRequest`

| Field | Type | Description |
| --- | --- | --- |
| `provider_hash` | `str` | Provider hash |

### `ProviderTokenRefreshResponse`

| Field | Type | Description |
| --- | --- | --- |
| `provider_hash` | `str` | Provider hash |
| `new_api_token` | `str` | Newly issued API token |

## Related

- [Provider Authorization](provider-authorization.md) — connecting a created provider to a specific user.
- [User Management](user-management.md) — `get_user_providers()`/`get_user_provider()` list connections from the user's side.
- [`v2hub-cli` Provider Commands](../v2hub-cli/admin-commands.md#provider-management) — the CLI wrapper around this API.
