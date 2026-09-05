# Usage Examples

## Full self-service lifecycle

```python
from v2hub import AsyncVPNClient, NotFoundError

async with AsyncVPNClient("https://api.example.com", "your-api-token") as client:
    sub = await client.create_subscription(
        "my-vpn",
        description="Personal VPN configs",
        sources=["vless://uuid@server1:443#Server1"],
    )

    await client.add_sources(sub.token, ["vmess://uuid@server2:443#Server2"])
    await client.update_source(sub.token, sub.sources[0].id, comment="Home server")

    sub = await client.get_subscription(sub.token)
    print(f"{sub.name}: {sub.sources_count} configs")

    public = await client.get_public_subscription(sub.token)
    print(f"Subscription URL content ({public.config_count} configs):")
    print(public.decode())

    try:
        await client.get_subscription("nonexistent-token")
    except NotFoundError:
        print("That subscription doesn't exist")

    await client.delete_subscription(sub.token)
```

## Provider onboarding a new end-user

```python
from v2hub import AsyncVPNClient

async with AsyncVPNClient("https://api.example.com", "provider-api-token") as client:
    connection = await client.create_provider_connection(user_id=98765)
    print("Send this link to the user:", connection.connection_link)

    sub = await client.create_subscription(
        "welcome-vpn",
        sources=["vless://uuid@server1:443#Server1"],
        as_provider_for_user_id=98765,
    )
    print(f"Provisioned {sub.token} for user 98765")
```

## Handling rate limits and retryable errors explicitly

```python
from v2hub import AsyncVPNClient, RetryConfig, VPNAPIError, RateLimitError

config = RetryConfig(max_retries=5, initial_delay=0.5)

async with AsyncVPNClient(
    "https://api.example.com", "your-api-token", retry_config=config
) as client:
    try:
        subs = await client.list_subscriptions()
    except RateLimitError as e:
        # Retries are already exhausted at this point (max_retries reached)
        print(f"Still rate limited after retries; wait {e.retry_after}s")
    except VPNAPIError as e:
        print(f"{e} — {e.recovery_hint}")
    else:
        print(f"Fetched {len(subs)} subscriptions")
```

## Sync usage in a script

```python
from v2hub import VPNClient

with VPNClient("https://api.example.com", "your-api-token") as client:
    sub = client.create_subscription("my-vpn")
    client.add_sources(sub.token, ["vless://uuid@server1:443#Server1"])

    subs = client.list_subscriptions()
    for s in subs:
        print(f"{s.name}: {s.token}")
```

## Managing a user's own provider connections

```python
from v2hub import AsyncVPNClient

async with AsyncVPNClient("https://api.example.com", "your-api-token") as client:
    connections = await client.list_connections()
    for conn in connections.connections:
        print(conn.provider_name, conn.status)

    await client.approve_connection("trusted-provider")
    await client.reject_connection("unwanted-provider")
    # ...later, if access is no longer needed:
    await client.revoke_connection("trusted-provider")
```

## Hiding a source without deleting it

```python
# Keep the source resolvable internally, but exclude it from the public output
await client.update_source(sub.token, config_id="cfg123", is_hidden=True)

# Later, make it visible again
await client.update_source(sub.token, config_id="cfg123", is_hidden=False)
```
