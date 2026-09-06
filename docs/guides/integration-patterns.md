# Common Integration Patterns

Recurring shapes that come up across V2Hub integrations, distilled into patterns you can adapt directly.

## Sync-on-signup

Provision a subscription the moment a user completes signup in your own application, and store the token immediately so it's never lost even if a later step fails:

```python
async def on_user_signup(user):
    async with AsyncVPNClient(base_url, provider_api_token) as client:
        await client.create_provider_connection(user_id=user.v2hub_user_id)

        sub = await client.create_subscription(
            f"user-{user.id}-vpn",
            as_provider_for_user_id=user.v2hub_user_id,
        )

        # Persist the token before doing anything else — this is the one
        # value you cannot easily recover later without re-listing and
        # matching on name.
        await user.save(v2hub_subscription_token=sub.token)
```

If `create_subscription` fails after `create_provider_connection` already succeeded, it's safe to retry the whole function — `create_provider_connection` re-approves rather than erroring on an existing connection, and a repeat `create_subscription` call with the same name will raise `ConflictError` (or `DuplicateNameError`) rather than creating a duplicate, which you can catch and resolve by looking the existing subscription up via `list_subscriptions`.

## Periodic refresh

A scheduled job (cron, Celery beat, an APScheduler job) that force-refreshes subscriptions containing external URL sources, so changes on the upstream provider's side propagate promptly instead of waiting for the API's own background refresh interval:

```python
async def refresh_all_subscriptions():
    async with AsyncVPNClient(base_url, api_token) as client:
        subs = await client.list_subscriptions()
        for sub in subs:
            has_external = any(s.source_type == "external_url" for s in sub.sources)
            if not has_external:
                continue
            try:
                result = await client.refresh_subscription(sub.token)
                if result.failed:
                    logger.warning("%s: %d source(s) failed to refresh", sub.token, result.failed)
            except VPNAPIError as e:
                logger.error("refresh failed for %s: %s", sub.token, e)
                # continue with the next subscription rather than aborting the whole job
```

Run this on an interval longer than the API's own background refresh cadence (a good starting point is hourly) — the point is catching sources that need an out-of-band nudge, not replacing the built-in refresh entirely.

## Provider webhook / callback on authorization change

If your provider service needs to react when a user approves or revokes access (rather than polling), poll `get_provider_connection` from a lightweight periodic check and diff against last-known state, since V2Hub doesn't push webhooks itself:

```python
async def check_authorization_changes(client, tracked_user_ids: dict[int, str]):
    for user_id, last_status in list(tracked_user_ids.items()):
        conn = await client.get_provider_connection(user_id)
        if conn.status != last_status:
            await on_authorization_changed(user_id, conn.status)
            tracked_user_ids[user_id] = conn.status
```

Keep the poll interval modest (every few minutes is usually plenty) and batch it across all tracked users in one job run rather than one job per user.

## Config caching for high-traffic public endpoints

If `/sub/{token}` (or your own proxy in front of it) sees heavy traffic, cache the _decoded_ result of `get_public_subscription` for a short TTL rather than calling it on every request — the underlying content only changes when sources are added/removed/refreshed, not continuously:

```python
async def get_cached_public_subscription(client, token: str, cache, ttl: int = 60):
    cached = await cache.get(f"pubsub:{token}")
    if cached is not None:
        return cached

    public = await client.get_public_subscription(token)
    content = public.decode()
    await cache.set(f"pubsub:{token}", content, ttl=ttl)
    return content
```

Invalidate the cache key explicitly after any call that changes the subscription's sources (`add_sources`, `replace_sources`, `remove_sources`, `refresh_subscription`) rather than relying purely on TTL expiry, if your application controls both the write and read paths.

## Fan-out subscription creation for bulk provisioning

When provisioning many users at once (e.g. an import job), bound concurrency rather than firing every request at once — the client's retry and circuit-breaker machinery handles transient failures, but a large unbounded burst can still trip rate limits unnecessarily:

```python
import asyncio

async def provision_users(client, user_ids: list[int], concurrency: int = 10):
    semaphore = asyncio.Semaphore(concurrency)

    async def provision_one(user_id: int):
        async with semaphore:
            await client.create_provider_connection(user_id=user_id)
            return await client.create_subscription(
                f"user-{user_id}-vpn", as_provider_for_user_id=user_id
            )

    results = await asyncio.gather(
        *(provision_one(uid) for uid in user_ids), return_exceptions=True
    )
    for user_id, result in zip(user_ids, results):
        if isinstance(result, Exception):
            logger.error("failed to provision user %s: %s", user_id, result)
```

## Where to go next

- [Production Usage](production.md) for tuning the retry/circuit-breaker/timeout settings referenced above under real load.
- [End-to-End Examples](end-to-end-examples.md) for fuller, runnable scenarios built from these same building blocks.
