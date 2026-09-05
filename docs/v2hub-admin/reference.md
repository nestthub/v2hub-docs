# API Reference

Flat, alphabetized-within-section index of every public class, method, and model in `v2hub_admin`. See the linked pages for full descriptions and examples.

## Clients

| Class | Description |
| --- | --- |
| `AsyncAdminClient` | Async admin client — see [Installation & Setup](installation.md) |
| `AdminClient` | Sync wrapper around `AsyncAdminClient` — see [Installation & Setup](installation.md) |

Both are exported from the top-level `v2hub_admin` package: `from v2hub_admin import AdminClient, AsyncAdminClient`.

## Methods

Every method exists identically on both clients (as a coroutine on `AsyncAdminClient`, as a plain method on `AdminClient`).

### User Management — see [User Management](user-management.md)

| Method | Returns |
| --- | --- |
| `create_user(user_id)` | `UserCreateResponse` |
| `get_user(user_id)` | `UserResponse` |
| `delete_user(user_id)` | `None` |
| `set_user_status(user_id, is_active)` | `UserResponse` |
| `refresh_token(user_id)` | `TokenRefreshResponse` |
| `get_user_providers(user_id)` | `ConnectionsResponse` |
| `get_user_provider(user_id, provider_name)` | `ConnectionResponse` |

### Provider Management — see [Provider Management](provider-management.md)

| Method | Returns |
| --- | --- |
| `create_provider(owner_hash, provider_name, provider_url=None)` | `ProviderCreateResponse` |
| `get_providers()` | `AllProvidersResponse` |
| `get_provider(provider_hash)` | `ProviderResponse` |
| `get_provider_by_name(provider_name)` | `ProviderResponse` |
| `get_provider_by_owner_id(owner_id)` | `ProviderResponse` |
| `delete_provider(provider_hash)` | `None` |
| `set_provider_status(provider_hash, is_active)` | `ProviderResponse` |
| `update_provider_url(provider_hash, provider_url)` | `ProviderResponse` |
| `update_provider_name(provider_hash, provider_name)` | `ProviderResponse` |
| `refresh_provider_token(provider_hash)` | `ProviderTokenRefreshResponse` |

### Provider Authorization — see [Provider Authorization](provider-authorization.md)

| Method | Returns |
| --- | --- |
| `get_provider_authorization(provider_name, user_id)` | `ProviderAuthorizationInfoResponse` |
| `process_provider_authorization(user_id, provider_name, hmac=None)` | `ProviderAuthorizationInfoResponse` |
| `approve_provider_authorization(user_id, provider_name)` | `ProviderAuthorizationInfoResponse` |
| `reject_provider_authorization(user_id, provider_name)` | `ProviderAuthorizationInfoResponse` |

### IP Ban Management — see [IP Bans & Whitelist](ip-bans-and-whitelist.md)

| Method | Returns |
| --- | --- |
| `ban_ip(ip_address, duration_seconds=None)` | `IPBanStatusResponse` |
| `unban_ip(ip_address)` | `IPUnbanResponse` |
| `get_ban_status(ip_address)` | `IPBanStatusResponse` |
| `get_ban_list()` | `IPBanListResponse` |

### Whitelist Management — see [IP Bans & Whitelist](ip-bans-and-whitelist.md)

| Method | Returns |
| --- | --- |
| `add_to_whitelist(ip_address, description=None)` | `WhitelistAddResponse` |
| `remove_from_whitelist(ip_address)` | `WhitelistRemoveResponse` |
| `list_whitelist()` | `WhitelistListResponse` |

### Usage Statistics — see [Usage Statistics](usage-statistics.md)

| Method | Returns |
| --- | --- |
| `get_stats(start_date=None, end_date=None, period=None)` | `StatsResponse` |

## Models

Imported from `v2hub_admin.models` (also re-exported via `v2hub_admin` for the two client classes).

### Users — see [User Management](user-management.md#user-models)

`UserCreateRequest`, `UserResponse`, `UserCreateResponse`, `UserStatusUpdateRequest`, `TokenRefreshRequest`, `TokenRefreshResponse`

### Providers — see [Provider Management](provider-management.md#provider-models)

`ProviderCreateRequest`, `ProviderResponse`, `ProviderCreateResponse`, `AllProvidersResponse`, `ProviderStatusUpdateRequest`, `ProviderURLUpdateRequest`, `ProviderNameUpdateRequest`, `ProviderTokenRefreshRequest`, `ProviderTokenRefreshResponse`

### Provider Authorization — see [Provider Authorization](provider-authorization.md#provider-authorization-models)

`ProviderAuthorizationBaseRequest`, `ProviderAuthorizationInfoResponse`, `ProviderAuthorizationRequest`, `ProviderAuthorizationDecisionRequest`

### Access Lists (IP Bans & Whitelist) — see [IP Bans & Whitelist](ip-bans-and-whitelist.md#access-list-models)

`IPBanRequest`, `IPUnbanRequest`, `IPUnbanResponse`, `IPBanStatusResponse`, `IPBanEntry`, `IPBanListResponse`, `WhitelistAddRequest`, `WhitelistAddResponse`, `WhitelistRemoveRequest`, `WhitelistRemoveResponse`, `WhitelistEntry`, `WhitelistListResponse`

### Statistics — see [Usage Statistics](usage-statistics.md#statistics-models)

`GeneralStats`, `StatsResponse`

### Base

| Model | Description |
| --- | --- |
| `AdminBaseModel` | Shared Pydantic base for every admin model: `frozen=False`, `validate_assignment=True`, `use_enum_values=True`, `str_strip_whitespace=True`, `populate_by_name=True` |

## Auth

| Class | Description |
| --- | --- |
| `AdminAuthenticator` | HMAC-SHA256 request signer — see [Authentication & Authorization](authentication.md) |

`AdminAuthenticator(secret_key)` exposes `sign_request(method, path, body="")` (returns `X-Signature`/`X-Timestamp`/`Content-Type` headers) and `verify_timestamp(timestamp, max_age_seconds=300)`.

## Reused from `v2hub`

The admin client reuses these directly from the base [`v2hub`](../v2hub/index.md) package rather than redefining them:

| Name | Source | See |
| --- | --- | --- |
| `RetryConfig` | `v2hub.core.retry` | [Retries & Circuit Breaker](../v2hub/retries.md) |
| `HTTPClient` | `v2hub.http.client` | [Requests & Responses](../v2hub/requests-responses.md) |
| `ConnectionResponse`, `ConnectionsResponse` | `v2hub.models` | [Typed Models](../v2hub/models.md) |
| `ProviderAuthorizationStatus` | `v2hub.models` | [Typed Models](../v2hub/models.md) |
| `VPNAPIError` and all its subclasses (`AuthenticationError`, `AuthorizationError`, `NotFoundError`, `ConflictError`, `ValidationError`, ...) | `v2hub` | [Error Handling](../v2hub/errors.md) |

## Requirements

- `v2hub>=1.1.2`
- `httpx>=0.25.0`
- `pydantic>=2.0.0`
- Python `>=3.10`
