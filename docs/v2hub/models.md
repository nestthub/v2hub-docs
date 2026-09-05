# Typed Models

All models are Pydantic v2 (`BaseModel` subclasses via a shared `BaseModelConfig` base). They are importable from the top-level `v2hub` package (e.g. `from v2hub import Subscription`) or, equivalently, from `v2hub.models`.

Shared model configuration (`BaseModelConfig`, in `v2hub.models.base`): unknown fields from the server are ignored (`extra="ignore"`), enum fields serialize to their plain value (`use_enum_values=True`), string fields are stripped of surrounding whitespace, and fields can be set by either their Python name or alias (`populate_by_name=True`).

## Enums

### `SourceType`

Type of source entry in a subscription. String enum — compares/serializes as its plain value (e.g. `SourceType.CONFIG == "config"`).

| Value | Description |
| --- | --- |
| `CONFIG` (`"config"`) | Direct proxy configuration (`vless://`, `vmess://`, etc.) |
| `EXTERNAL_URL` (`"external_url"`) | HTTPS URL to a third-party subscription provider |
| `INTERNAL_TOKEN` (`"internal_token"`) | Token reference to another subscription (same user) |

### `ProviderAuthorizationStatus`

Status of a provider↔user authorization connection. String enum.

| Value | Description |
| --- | --- |
| `APPROVED` (`"approved"`) | Provider is authorized to manage this user's subscriptions |
| `PENDING` (`"pending"`) | Requested but not yet approved |
| `REVOKED` (`"revoked"`) | Previously approved, now revoked |
| `UNKNOWN` (`"unknown"`) | Fallback for any status value the client doesn't recognize (via Pydantic's `_missing_` hook), so older client versions degrade gracefully against newer servers instead of raising a validation error |

## Subscription models

### `Subscription`

Complete subscription with all details. Returned by `create_subscription`, `get_subscription`, `update_subscription`, `add_sources`, `replace_sources`, `remove_sources`.

| Field | Type | Description |
| --- | --- | --- |
| `token` | `str` | Unique subscription token |
| `name` | `str` | User-defined name (1–64 chars) |
| `provider_name` | `str \| None` | Name of the provider managing this subscription, if any |
| `description` | `str \| None` | Optional description (max 255 chars) |
| `sources` | `list[Source]` | Resolved list of sources |
| `sources_count` | `int` | Total resolved configs count (`>= 0`) |
| `created_at` | `datetime` | Creation timestamp |
| `updated_at` | `datetime` | Last update timestamp |

### `SubscriptionListItem`

Identical to `Subscription` (a plain subclass with no additional fields), returned by `list_subscriptions()`.

### `SubscriptionCreateRequest`

*Internal* request model built by `create_subscription()` — you don't normally construct this yourself, but it defines the exact client-side validation applied to your arguments.

| Field | Type | Default | Description |
| --- | --- | --- | --- |
| `name` | `str` | — | Subscription name (1–64 chars, non-empty after stripping) |
| `description` | `str \| None` | `None` | Optional description (max 255 chars) |
| `sources` | `list[SourceCreate]` | `[]` | Initial sources — normalized/deduplicated the same way as `add_sources` (see [`SourceCreate`](#sourcecreate)) |

### `SubscriptionUpdateRequest`

*Internal* request model built by `update_subscription()`.

| Field | Type | Default | Description |
| --- | --- | --- | --- |
| `name` | `str \| None` | `None` | New name (1–64 chars if given) |
| `description` | `str \| None` | `None` | New description |

At least one of `name` / `description` must be provided — constructing this model with both `None` raises a validation error (surfaced as `v2hub.ValidationError`).

### `RefreshSubscriptionResponse`

Returned by `refresh_subscription(token)`.

| Field | Type | Default | Description |
| --- | --- | --- | --- |
| `refreshed` | `int` | `0` | Number of successfully refreshed sources |
| `failed` | `int` | `0` | Number of sources that failed to refresh |
| `skipped` | `int` | `0` | Number of sources skipped during refresh |
| `total` | `int` | `0` | Total URLs processed |
| `message` | `str \| None` | `None` | Optional status message |
| `errors` | `list[str] \| None` | `None` | Per-URL error details, e.g. `["https://example.com: timeout"]` |

## Source models

### `Source`

An individual, resolved source within a subscription's `sources` list.

| Field | Type | Description |
| --- | --- | --- |
| `id` | `str` | Unique source identifier (hash) |
| `source_type` | `SourceType` | `config`, `external_url`, or `internal_token` |
| `data` | `str` | The source data (config URI, URL, or token); validated non-empty |
| `order_index` | `int` | Display order (`>= 0`) |
| `is_hidden` | `bool` | Whether hidden from resolved public output (default `False`) |
| `max_depth` | `int` | Max nesting depth to follow, `0`–`3` (default `3`) |
| `created_at` | `datetime` | Creation timestamp |
| `updated_at` | `datetime` | Last update timestamp |

### `SourceCreate`

Input shape for adding/replacing sources — also accepted as a plain `str` or `dict` wherever a list of sources is expected (`add_sources`, `replace_sources`, `create_subscription(sources=...)`); see [Sources](clients.md#sources).

| Field | Type | Default | Description |
| --- | --- | --- | --- |
| `data` | `str` | — | Source data (config, URL, or token); non-empty |
| `is_hidden` | `bool \| None` | `None` | Hide from resolved output |
| `max_depth` | `int \| None` | `None` | Max nesting depth (0–3) |

### `SourceAddRequest`

*Internal* request model built by `add_sources()`.

| Field | Type | Description |
| --- | --- | --- |
| `sources` | `list[SourceCreate]` | Sources to add; at least one required. Input is normalized and deduplicated by (stripped) `data` value before validation — see below |

### `SourceReplaceRequest`

*Internal* request model built by `replace_sources()`.

| Field | Type | Description |
| --- | --- | --- |
| `sources` | `list[SourceCreate]` | New sources, replacing all existing ones. Same normalization/dedup as `SourceAddRequest`, but an empty list is allowed (clears all sources) |

### `SourceRemoveRequest`

*Internal* request model built by `remove_sources()`.

| Field | Type | Description |
| --- | --- | --- |
| `source_ids` | `list[str]` | Source IDs to remove; at least one required. Deduplicated, preserving first-occurrence order |

### `SourceUpdateRequest`

*Internal* request model built by `update_source()`. Supersedes `CommentUpdateRequest` below: same comment update, plus `is_hidden` and `max_depth`. Only fields explicitly provided are changed server-side; omitted (`None`) fields are left untouched.

| Field | Type | Default | Description |
| --- | --- | --- | --- |
| `config_id` | `str` | — | Config id; non-empty |
| `comment` | `str \| None` | `None` | Comment text (max 255 chars) |
| `is_hidden` | `bool \| None` | `None` | Whether the source is hidden from end users |
| `max_depth` | `int \| None` | `None` | Max nesting depth (0–3) |

### `CommentUpdateRequest` (deprecated)

*Internal* request model built by the deprecated `update_comment()`. Still fully functional, but emits a `DeprecationWarning` on construction and receives no further updates — use [`SourceUpdateRequest`](#sourceupdaterequest) / `update_source()` instead.

| Field | Type | Description |
| --- | --- | --- |
| `config_id` | `str` | Config id; non-empty |
| `comment` | `str \| None` | Comment text (max 255 chars) |

### Source normalization

`SourceCreate` lists passed to `SourceAddRequest`, `SourceReplaceRequest`, and `SubscriptionCreateRequest` all go through the same internal normalization step: each item — a plain `str`, a `dict`, or a `SourceCreate` instance — is converted to a `{"data": ..., ...}` dict, and duplicates (by stripped `data` value) are dropped, keeping the first occurrence. This is why mixed-type lists (`["vless://...", {"data": "https://...", "is_hidden": True}]`) work transparently.

## Public model

### `PublicSubscriptionResponse`

Returned by `get_public_subscription(token)`. Represents the resolved, base64-encoded subscription content served at the public `/sub/{token}` endpoint (no auth required).

| Field | Type | Default | Description |
| --- | --- | --- | --- |
| `title` | `str \| None` | `"v2hub"` | Subscription title, decoded from the response's `profile-title` header |
| `content` | `str` | — | Raw base64-encoded content, exactly as returned by the server |

| Method / Property | Returns | Description |
| --- | --- | --- |
| `decode()` | `str` | Decode `content` to a plain UTF-8 string; raises `ValueError` if it isn't valid base64 |
| `get_configs()` | `list[str]` | Decode and split into a list of individual, non-empty config lines |
| `config_count` | `int` | `len(get_configs())` |

```python
public = await client.get_public_subscription(sub.token)
print(public.title)
print(public.config_count)
for line in public.get_configs():
    print(line)
```

## Provider and connection models

### `MeResponse`

Returned by `get_me()`.

| Field | Type | Description |
| --- | --- | --- |
| `user_id` | `int` | The authenticated user's numeric ID |
| `is_active` | `bool` | Whether the account is active |

### `ConnectionResponse`

A single provider connection from the current user's point of view. Returned by `get_connection()`, `approve_connection()`, `reject_connection()`, and as items of `ConnectionsResponse.connections`.

| Field | Type | Description |
| --- | --- | --- |
| `provider_name` | `str` | Public provider name |
| `provider_url` | `str \| None` | Provider's URL, if published |
| `is_authorized` | `bool` | Whether the provider is currently authorized |
| `status` | `ProviderAuthorizationStatus \| None` | Current authorization status |

### `ConnectionsResponse`

Returned by `list_connections()`.

| Field | Type | Description |
| --- | --- | --- |
| `connections` | `list[ConnectionResponse]` | Pending and approved connections (revoked ones excluded) |

### `ProviderConnectionRequest`

*Internal* request model for provider-connection endpoints.

| Field | Type | Description |
| --- | --- | --- |
| `user_id` | `int` | Target user ID (`> 0`) |

### `ProviderConnectionResponse`

A provider↔user connection from the *provider's* point of view. Returned by `get_provider_connection()` and `revoke_provider_connection()`.

| Field | Type | Description |
| --- | --- | --- |
| `user_id` | `int` | The end-user's numeric ID |
| `status` | `ProviderAuthorizationStatus` | Current authorization status |

### `ProviderConnectionCreateResponse`

Returned by `create_provider_connection()`. Extends [`ProviderConnectionResponse`](#providerconnectionresponse) with one additional field.

| Field | Type | Description |
| --- | --- | --- |
| `user_id` | `int` | *(inherited)* The end-user's numeric ID |
| `status` | `ProviderAuthorizationStatus` | *(inherited)* Current authorization status |
| `connection_link` | `str \| None` | Link the end-user can use to view/approve the connection |

### `ProviderConnectionDeleteResponse`

Returned by `delete_provider_connection()`.

| Field | Type | Description |
| --- | --- | --- |
| `detail` | `str` | Human-readable confirmation of the deletion |

## Error model

### `ErrorResponse`

Shape of the JSON error body the server returns (used internally to build typed exceptions — see [Error Handling](errors.md)); not something you construct yourself.

| Field | Type | Description |
| --- | --- | --- |
| `error` | `str` | Error code/type; non-empty |
| `message` | `str` | Human-readable error message; non-empty |
| `details` | `dict[str, Any] \| None` | Additional error details |
