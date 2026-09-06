# Subscription Workflows

Day-to-day tasks around creating, updating, and resolving subscriptions and their sources — the core of what most applications built on V2Hub actually do. For the full method and model reference, see [V2Hub → Sync & Async Clients](../v2hub/clients.md) and [V2Hub → Typed Models](../v2hub/models.md).

## Creating a subscription with initial sources

The common case: create a subscription and seed it with sources in one call rather than creating empty and adding sources afterward.

```python
sub = await client.create_subscription(
    "customer-42-vpn",
    description="Auto-provisioned on signup",
    sources=[
        "vless://uuid@server1:443#Server1",
        "vless://uuid@server2:443#Server2",
    ],
)
```

`sub.token` is what you store and use for every subsequent call on this subscription, and it's also the value embedded in the public subscription URL (`/sub/{token}`) that end-users paste into their VPN client.

## Growing a subscription over time

Add sources incrementally as they become available, rather than re-creating the subscription:

```python
await client.add_sources(sub.token, ["vmess://uuid@server3:443#Server3"])
```

`add_sources` is additive and deduplicates by source content — calling it again with a source that's already present is a safe no-op for that entry, not a duplicate. If you instead need to fully replace the set (e.g. syncing from an external source of truth), use `replace_sources` — be aware it discards anything not in the new list:

```python
await client.replace_sources(sub.token, current_source_list)
```

## Hiding vs. removing a source

If a source needs to be temporarily excluded from the resolved output — a server under maintenance, for example — prefer hiding it over deleting it, so it can be restored without re-adding:

```python
await client.update_source(sub.token, config_id=source_id, is_hidden=True)
# ... later
await client.update_source(sub.token, config_id=source_id, is_hidden=False)
```

Only pass the fields you want to change; `update_source` leaves any field you omit untouched. Use `remove_sources` only when the source is truly gone for good:

```python
await client.remove_sources(sub.token, [source_id])
```

## Serving the resolved subscription

The endpoint end-users actually consume is the **public** one — unauthenticated, and returning base64-encoded config content ready for a VPN client:

```python
public = await client.get_public_subscription(sub.token)
print(public.title)          # from the profile-title header
print(public.config_count)   # number of resolved config lines
print(public.decode())       # raw decoded content, ready to serve as-is
```

If you're building a web app, the simplest integration is to redirect or link users directly to your API's `/sub/{token}` URL rather than proxying `get_public_subscription` through your own backend — it's designed to be hit directly by VPN clients. Use `get_public_subscription` in your own code when you need to _inspect_ or _display_ the resolved content (e.g. showing a config count in a dashboard), not to re-serve it.

## Refreshing external sources

Sources of type `external_url` are cached and refreshed automatically in the background (every 15 minutes, per the API's default), but you can force an immediate refresh — useful right after adding a new external URL source, or when troubleshooting a stale subscription:

```python
result = await client.refresh_subscription(sub.token)
print(f"{result.refreshed} refreshed, {result.failed} failed, {result.skipped} skipped")
if result.errors:
    for err in result.errors:
        print("  -", err)
```

## Nested subscriptions (internal tokens)

A source can reference another one of your own subscriptions by token (`source_type == "internal_token"`), which is resolved recursively. Two knobs control this per source:

- **`max_depth`** (0–3) — how many further levels of nesting to follow from that source.
- Circular references are detected automatically and raise `CircularReferenceError` rather than looping forever.

```python
await client.add_sources(
    parent_sub.token,
    [{"data": child_sub.token, "max_depth": 1}],
)
```

Keep nesting shallow in practice — it's a tool for composing a handful of shared "building block" subscriptions (e.g. a base server list reused across several customer-facing subscriptions), not a general hierarchy mechanism.

## Handling not-found and validation errors

The two errors you'll hit most often in subscription code are `SubscriptionNotFoundError` (bad or stale token) and `ValidationError` (bad input, e.g. a name that's empty or too long):

```python
from v2hub import SubscriptionNotFoundError, ValidationError

try:
    sub = await client.get_subscription(token)
except SubscriptionNotFoundError:
    # token doesn't exist (or was deleted) — don't retry, surface a 404-style
    # response or trigger re-provisioning
    ...
except ValidationError as e:
    # caller passed bad input — this is a bug in the calling code, not a
    # transient condition
    logger.error("invalid subscription request: %s", e)
    raise
```

See [Error Handling Patterns](error-handling.md) for a more general strategy, and [V2Hub → Error Handling](../v2hub/errors.md) for the complete exception hierarchy.

## Where to go next

- [Provider Workflows](providers.md) if these subscriptions are being managed on behalf of end-users rather than for your own account.
- [Common Integration Patterns](integration-patterns.md#periodic-refresh) for a worked example of a scheduled refresh job.
- [V2Hub → Typed Models](../v2hub/models.md#subscription) for every field on `Subscription` and `Source`.
