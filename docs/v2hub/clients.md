# Sync & Async Clients

`AsyncVPNClient` and `VPNClient` expose the same methods, grouped below by resource. See [Installation & Setup](installation.md) for how to construct and connect either client.

Every subscription and source method additionally accepts an optional, keyword-only `as_provider_for_user_id: int` argument, covered in [Providers](#providers) below. It is omitted from the per-method signatures in this page for brevity except where it changes behavior meaningfully.

## Asynchronous Client

`AsyncVPNClient` is the primary implementation; `VPNClient` delegates to it. Use it directly in any async application (e.g. behind FastAPI, an async worker, or a bot framework):

```python
from v2hub import AsyncVPNClient

async with AsyncVPNClient("https://api.example.com", "your-api-token") as client:
    sub = await client.create_subscription(
        "my-vpn",
        sources=["vless://uuid@server1:443#Server1"],
    )
    await client.add_sources(sub.token, ["vmess://uuid@server2:443#Server2"])
    public = await client.get_public_subscription(sub.token)
    print(public.get_configs())
```

## Synchronous Client

`VPNClient` wraps `AsyncVPNClient` and exposes the same methods without `await`, for use in non-async codebases. Within a `with` block it manages one internal event loop for the whole block:

```python
from v2hub import VPNClient

with VPNClient("https://api.example.com", "your-api-token") as client:
    sub = client.create_subscription("my-vpn")
    client.add_sources(sub.token, ["vless://uuid@server1:443#Server1"])

    subs = client.list_subscriptions()
    for s in subs:
        print(f"{s.name}: {s.token}")
```

Outside a `with` block, each call creates and tears down a temporary event loop via `asyncio.run(...)`, which works but is less efficient for multiple sequential calls — prefer the context-manager form when making more than one call.

## Subscriptions

Subscriptions are the core resource: a named collection of sources that resolve into a set of proxy configs served from a public token URL.

| Method | Description |
| --- | --- |
| `create_subscription(name, description=None, sources=None, *, as_provider_for_user_id=None)` | Create a new subscription, optionally seeded with initial sources. Returns [`Subscription`](models.md#subscription) |
| `get_subscription(token, *, as_provider_for_user_id=None)` | Get subscription details by token. Returns [`Subscription`](models.md#subscription) |
| `list_subscriptions(*, as_provider_for_user_id=None)` | List all subscriptions for the current account. Returns `list[SubscriptionListItem]` |
| `update_subscription(token, name=None, description=None, *, as_provider_for_user_id=None)` | Update subscription metadata (name and/or description; at least one must be given). Returns [`Subscription`](models.md#subscription) |
| `delete_subscription(token, *, as_provider_for_user_id=None)` | Delete a subscription. Returns `None` |
| `refresh_subscription(token, *, as_provider_for_user_id=None)` | Manually trigger a refresh of external URL sources. Returns [`RefreshSubscriptionResponse`](models.md#refreshsubscriptionresponse) |

```python
sub = await client.create_subscription(
    "my-vpn",
    description="Production VPN configs",
    sources=["vless://uuid@server:443#Server1"],
)

sub = await client.get_subscription(sub.token)

subs = await client.list_subscriptions()

sub = await client.update_subscription(sub.token, name="renamed-vpn")

result = await client.refresh_subscription(sub.token)

await client.delete_subscription(sub.token)
```

`create_subscription` and `update_subscription` validate their arguments client-side using the same Pydantic models the server expects (`SubscriptionCreateRequest`, `SubscriptionUpdateRequest` — see [Requests & Responses](requests-responses.md)), raising `v2hub.ValidationError` before a request is even sent if, for example, `name` is empty or exceeds 64 characters.

### Raises

All subscription methods can raise `AuthenticationError` and, more generally, `VPNAPIError`. Additionally:

- `get_subscription`, `update_subscription`, `delete_subscription`, `refresh_subscription` — `SubscriptionNotFoundError` if `token` doesn't exist.
- `create_subscription`, `update_subscription` — `ValidationError` for invalid parameters; `ConflictError` if the (new) name already exists.

See [Error Handling](errors.md) for the full hierarchy.

### Sources within a subscription

A subscription's `sources` field (see the [`Subscription`](models.md#subscription) model) is a list of resolved [`Source`](models.md#source) objects, each with a `source_type` of `config`, `external_url`, or `internal_token`. Managing that list is done through the dedicated source methods below rather than by mutating `sources` on the model directly.

## Sources

Sources are added, replaced, removed, or individually updated by subscription token. Every method accepts sources as plain strings (shorthand for `{"data": <string>}`), dicts, or [`SourceCreate`](models.md#sourcecreate) instances — mixed within the same list if needed.

| Method | Description |
| --- | --- |
| `add_sources(token, sources, *, as_provider_for_user_id=None)` | Add one or more sources to a subscription. Returns [`Subscription`](models.md#subscription) |
| `replace_sources(token, sources, *, as_provider_for_user_id=None)` | Replace *all* sources in a subscription. Returns [`Subscription`](models.md#subscription) |
| `remove_sources(token, source_ids, *, as_provider_for_user_id=None)` | Remove specific sources by their `id`. Returns [`Subscription`](models.md#subscription) |
| `update_source(token, config_id, comment=None, is_hidden=None, max_depth=None, *, as_provider_for_user_id=None)` | Partially update a single source's comment, visibility, or nesting depth. Returns `None` |
| `update_comment(token, config_id, comment, *, as_provider_for_user_id=None)` | **Deprecated** — use `update_source()` instead. Returns `None` |

```python
sub = await client.add_sources(
    sub.token,
    [
        "vless://uuid@server1:443#Server1",
        {"data": "https://provider.com/subscription", "is_hidden": True},
    ],
)

sub = await client.replace_sources(sub.token, ["vmess://uuid@server2:443#Server2"])

sub = await client.remove_sources(sub.token, [sub.sources[0].id])

# Only change is_hidden; comment and max_depth are left untouched
await client.update_source(sub.token, "cfg123", is_hidden=True)
```

Source input is deduplicated client-side by the (stripped) `data` value, preserving first-occurrence order, before the request is sent — a source string added twice in the same call results in a single source.

### Raises

- `add_sources`, `replace_sources` — `SubscriptionNotFoundError`, `ValidationError` (invalid sources), `InvalidURLError` (SSRF protection triggered).
- `remove_sources` — `SubscriptionNotFoundError`, `SourceNotFoundError` (source ID not found), `ValidationError`.
- `update_source`, `update_comment` — `SubscriptionNotFoundError`, `ValidationError`.
- All of the above can also raise `AuthenticationError` or a generic `VPNAPIError`.

### `update_source` vs. the deprecated `update_comment`

`update_source()` replaces `update_comment()` and additionally supports `is_hidden` and `max_depth`. Only the fields you explicitly pass (non-`None`) are changed server-side; any field left as `None` is left untouched — you don't need to know or re-supply a source's current `is_hidden`/`max_depth` just to change its comment, or vice versa.

`update_comment()` still works and is fully supported, but emits a `DeprecationWarning` on use and will not receive further updates.

### Source types and per-source options

Each resolved [`Source`](models.md#source) has a `source_type` (`config`, `external_url`, or `internal_token`) determined server-side from the value of `data`:

- **`config`** — a direct proxy URI (`vless://`, `vmess://`, `trojan://`, etc.)
- **`external_url`** — an `https://` URL to a third-party subscription provider, fetched and cached server-side
- **`internal_token`** — a reference to another one of your own subscriptions, resolved recursively (with circular-reference detection)

Two options apply per source, settable at creation (via `SourceCreate`) or later (via `update_source`):

- **`is_hidden`** (`bool`) — hide this source from the resolved public output without deleting it.
- **`max_depth`** (`int`, `0`–`3`) — how many additional levels of nested `internal_token` references to follow when resolving this source.

## Providers

Providers manage subscriptions on behalf of end-users. A single client instance, authenticated with a **provider** API token, can act for any number of end-users by passing `as_provider_for_user_id` on subscription and source calls. An approved connection to the target `user_id` must exist first.

```python
from v2hub import AsyncVPNClient

async with AsyncVPNClient("https://api.example.com", "provider-api-token") as client:
    # Establish (or re-approve) authorization for an end-user
    await client.create_provider_connection(user_id=12345)

    # Manage that end-user's subscriptions on their behalf
    sub = await client.create_subscription(
        "managed-vpn",
        sources=["vless://uuid@server1:443#Server1"],
        as_provider_for_user_id=12345,
    )
    await client.add_sources(
        sub.token, ["vmess://uuid@server2:443#Server2"], as_provider_for_user_id=12345
    )

    # Revoke access when no longer needed
    await client.revoke_provider_connection(user_id=12345)
```

`as_provider_for_user_id` is deliberately a verbose, keyword-only argument (never positional) on every subscription and source method, so it can't be confused with another ID or passed accidentally. Omitting it (the default, `None`) always means "act on my own account"; passing it always means "act as a provider for this specific end-user."

### Provider connection management

Requires a **provider** API token. These establish and manage the authorization link between a provider and an end-user — a prerequisite for any `as_provider_for_user_id=` call.

| Method | Description |
| --- | --- |
| `create_provider_connection(user_id)` | Create or re-approve authorization for an end-user. If no account exists yet for `user_id`, one is created. Returns [`ProviderConnectionCreateResponse`](models.md#providerconnectioncreateresponse) (includes a `connection_link`) |
| `get_provider_connection(user_id)` | Get the current authorization status. Returns [`ProviderConnectionResponse`](models.md#providerconnectionresponse) |
| `revoke_provider_connection(user_id)` | Revoke authorization without deleting the record (can be re-approved later). Returns [`ProviderConnectionResponse`](models.md#providerconnectionresponse) |
| `delete_provider_connection(user_id)` | Permanently delete the authorization record. Returns [`ProviderConnectionDeleteResponse`](models.md#providerconnectiondeleteresponse) |

`create_provider_connection` currently auto-approves rather than waiting on end-user confirmation — this is documented server-side as a temporary/evolving flow; check the server's own docs/changelog for the current behavior.

#### Raises

- `create_provider_connection` — `TooManyApprovedUsersError` (provider's approved-user quota reached), `AuthenticationError`.
- `revoke_provider_connection` — `AuthenticationError` if there's no existing authorization to revoke.
- All four can raise a generic `VPNAPIError`.

### User self-service ("Me")

Separate from provider-mode subscription methods and from provider connection management above — these operate on the account tied to your *own* `api_token`, whether that token is a regular user token or a provider token acting on its own behalf.

| Method | Description |
| --- | --- |
| `get_me()` | Get information about the currently authenticated user. Returns [`MeResponse`](models.md#meresponse) |
| `list_connections()` | List the current user's provider connections (pending and approved; revoked ones excluded). Returns [`ConnectionsResponse`](models.md#connectionsresponse) |
| `get_connection(provider_name)` | Get the current user's connection status for a specific provider. Returns [`ConnectionResponse`](models.md#connectionresponse) |
| `approve_connection(provider_name)` | Approve a pending provider connection request (subject to the server-side `MAX_PROVIDERS_PER_USER` limit). Returns [`ConnectionResponse`](models.md#connectionresponse) |
| `reject_connection(provider_name)` | Reject a pending provider connection request. If subscriptions already exist for the provider, the server preserves the authorization as `REVOKED` instead of deleting it. Returns [`ConnectionResponse`](models.md#connectionresponse) |
| `revoke_connection(provider_name)` | Revoke the current user's provider authorization (existing subscriptions from that provider remain available). Returns `None` |

```python
from v2hub import AsyncVPNClient

async with AsyncVPNClient("https://api.example.com", "your-api-token") as client:
    me = await client.get_me()
    print(me.user_id, me.is_active)

    connections = await client.list_connections()
    for conn in connections.connections:
        print(conn.provider_name, conn.status)

    # Approve a pending request from a provider
    await client.approve_connection("my-provider")
```

#### Raises

- `get_connection`, `approve_connection`, `reject_connection`, `revoke_connection` — `NotFoundError` if the provider or authorization doesn't exist.
- `approve_connection` — `InvalidAuthorizationStatusError` if the connection isn't pending; `TooManyProvidersError` if the approved-provider quota is reached.
- `reject_connection` — `InvalidAuthorizationStatusError` if the connection isn't pending.
- All six can raise `AuthenticationError` or a generic `VPNAPIError`.

## Public Access

The single public, unauthenticated method returns the resolved subscription content served at `/sub/{token}` — this is the endpoint you'd put into a VPN client's subscription-URL field, and it requires no `api_token`:

| Method | Description |
| --- | --- |
| `get_public_subscription(token)` | Get the resolved subscription (base64-encoded configs + title), no auth required. Returns [`PublicSubscriptionResponse`](models.md#publicsubscriptionresponse) |

```python
public = await client.get_public_subscription(sub.token)
print(public.title)
print(public.config_count)
for config_line in public.get_configs():
    print(config_line)
```

See [`PublicSubscriptionResponse`](models.md#publicsubscriptionresponse) for the full set of decoding helpers.
