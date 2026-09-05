# Integration Examples

All examples below use the sync `AdminClient` for brevity; every method is available identically (as a coroutine) on `AsyncAdminClient` — see [Installation & Setup](installation.md).

```python
from v2hub_admin import AdminClient

admin = AdminClient(base_url="https://api.example.com", secret_key="your-hmac-secret")
```

## Onboard a New User

```python
with admin:
    user = admin.create_user(user_id=12345)
    print(f"API Token: {user.api_token}")
    print(f"User Hash: {user.user_hash}")
```

## Onboard a New Provider

```python
with admin:
    provider = admin.create_provider(
        owner_hash="a1b2c3d4e5f6...",
        provider_name="vpn123",
        provider_url="https://t.me/examplebot",
    )
    print(f"Provider hash: {provider.provider_hash}")
    print(f"API token: {provider.api_token}")
```

## Full Provider Authorization Handshake

Combines [Provider Management](provider-management.md) and [Provider Authorization](provider-authorization.md) end to end: creating a user, creating a provider, connecting them, and inspecting the result.

```python
with admin:
    user = admin.create_user(user_id=12345)
    provider = admin.create_provider(
        owner_hash=user.user_hash,
        provider_name="vpn123",
    )

    # The provider issues a connection invite out of band, producing an HMAC.
    # That HMAC is then submitted here to create a PENDING authorization:
    auth = admin.process_provider_authorization(
        user_id=user.user_id,
        provider_name=provider.provider_name,
        hmac="a1b2c3d4e5f6...",
    )
    assert auth.status == "pending"

    auth = admin.approve_provider_authorization(
        user_id=user.user_id,
        provider_name=provider.provider_name,
    )
    assert auth.status == "approved"

    # The provider can now manage this user's subscriptions via v2hub's
    # as_provider_for_user_id= — see the base client's docs.
```

## Deactivate and Rotate a Compromised Account

```python
with admin:
    # Immediately lock the account
    user = admin.set_user_status(12345, is_active=False)
    assert not user.is_active

    # Invalidate the old token
    result = admin.refresh_token(12345)
    print(f"New token: {result.new_api_token}")

    # Re-enable once the new token has been distributed
    admin.set_user_status(12345, is_active=True)
```

## Temporary IP Ban with Whitelist Exception

Combines [IP Bans & Whitelist](ip-bans-and-whitelist.md#ip-ban-management) with [Whitelist Management](ip-bans-and-whitelist.md#whitelist-management) to block a suspicious range while ensuring your office network is never affected.

```python
with admin:
    admin.add_to_whitelist("10.0.0.0/24", description="Office network")

    ban = admin.ban_ip("203.0.113.42", duration_seconds=3600)
    print(f"Banned until: {ban.banned_until}")

    status = admin.get_ban_status("203.0.113.42")
    if status.is_banned:
        print(f"Still banned, {status.remaining_seconds}s remaining")
```

## Weekly Usage Report

```python
with admin:
    stats = admin.get_stats(period="week")
    print(f"Total users:        {stats.general.total_users}")
    print(f"New users:          {stats.general.new_users}")
    print(f"New subscriptions:  {stats.general.new_subscriptions}")
```

## Audit a User's Provider Connections

```python
with admin:
    connections = admin.get_user_providers(12345)
    for conn in connections.connections:
        print(f"{conn.provider_name}: {conn.status}")

        if conn.status == "pending":
            admin.reject_provider_authorization(
                user_id=12345,
                provider_name=conn.provider_name,
            )
```

## Error Handling Across Operations

```python
from v2hub import AuthenticationError, AuthorizationError, ConflictError, NotFoundError, VPNAPIError

with admin:
    try:
        admin.approve_provider_authorization(user_id=12345, provider_name="vpn123")
    except ConflictError:
        print("Authorization is not pending, or the provider limit was reached")
    except NotFoundError:
        print("Provider, user, or authorization not found")
    except AuthenticationError:
        print("Invalid HMAC signature")
    except AuthorizationError:
        print("No admin privileges")
    except VPNAPIError as e:
        if e.is_retryable:
            print(f"Retryable error: {e.recovery_hint}")
        else:
            print(f"Permanent error: {e}")
```

See [Error Handling](../v2hub/errors.md) for the complete exception hierarchy, and [Retries & Circuit Breaker](../v2hub/retries.md) for how automatic retries are configured — both apply to the admin client exactly as they do to the base `v2hub` client.
