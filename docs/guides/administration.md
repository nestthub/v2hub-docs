# Administration Workflows

Privileged operations for running a V2Hub instance: provisioning users and providers, managing IP bans and whitelists, and pulling usage statistics. These require the `v2hub-admin` package (or `v2hub-cli` with its optional admin extra) and an admin `secret_key` — see [Authentication Workflows → Admin secret key](authentication.md#admin-secret-key).

```python
from v2hub_admin import AsyncAdminClient

async with AsyncAdminClient(base_url="https://api.example.com", secret_key="admin-secret") as admin:
    ...
```

The CLI equivalent is `v2hub admin ...` — see [V2Hub CLI → Admin Commands](../v2hub-cli/admin-commands.md) for every command this guide's Python examples map to.

## User Management

Provision, inspect, and manage end-user accounts:

```python
user = await admin.create_user(user_id=12345)
print(user.api_token)  # hand this to the user/their application — see Authentication Workflows

user = await admin.get_user(12345)
print(user.is_active)

user = await admin.set_user_status(12345, is_active=False)  # suspend

result = await admin.refresh_token(user_id=12345)
print(result.new_api_token)  # rotate a compromised or expiring token

await admin.delete_user(12345)  # irreversible
```

A common pattern: create the user, immediately store `user.api_token` in your own system (it's shown once, at creation, like most API tokens), and hand it to whatever application or bot acts on the user's behalf.

## Provider Management

Providers are external services (bots, resellers, integrations) that manage subscriptions on behalf of end-users via `as_provider_for_user_id=` — see [Provider Workflows](providers.md) for that side of the relationship. This section is about the provider _account_ lifecycle itself:

```python
provider = await admin.create_provider(
    owner_hash="a1b2c3d4e5f6...",   # the user_hash of the account that owns this provider
    provider_name="my-bot",
    provider_url="https://t.me/examplebot",
)
print(provider.provider_hash, provider.api_token)

providers = await admin.get_providers()
for name, provider_hash in providers.provider_hashes.items():
    print(name, provider_hash)

provider = await admin.get_provider_by_name("my-bot")
provider = await admin.get_provider_by_owner_id(owner_id=12345)

provider = await admin.set_provider_status(provider.provider_hash, is_active=False)

result = await admin.refresh_provider_token(provider.provider_hash)
print(result.new_api_token)

await admin.delete_provider(provider.provider_hash)  # cascades to provider-owned data
```

`owner_hash` ties a provider account back to the user account that "owns" it (typically the operator running the bot/service) — obtain it from `get_user`/`create_user`'s response for that owning account before calling `create_provider`.

## Provider Authorization Management

The admin-side view of the provider↔user authorization handshake described from the provider's perspective in [Provider Workflows → Connection lifecycle](providers.md#connection-lifecycle):

```python
# Process an incoming request (the provider issued an HMAC-signed invite to the user)
auth = await admin.process_provider_authorization(
    user_id=12345,
    provider_name="my-bot",
    hmac="a1b2c3d4e5f6...",
)
print(auth.status)  # "pending"

# Approve or reject it
auth = await admin.approve_provider_authorization(user_id=12345, provider_name="my-bot")
# or
await admin.reject_provider_authorization(user_id=12345, provider_name="my-bot")

# Check status at any time
auth = await admin.get_provider_authorization("my-bot", user_id=12345)
```

`approve_provider_authorization` and `reject_provider_authorization` only operate on `PENDING` authorizations and raise `ConflictError` otherwise — including when the target user has already reached the server's provider-per-user limit. `reject_provider_authorization`'s result depends on history: the authorization is deleted outright if the user never had subscriptions from that provider, or kept as `REVOKED` if they did (so existing subscriptions remain traceable).

## Getting Users' Provider Connections (read side)

To inspect a user's connections from the admin side without going through the user's own token:

```python
connections = await admin.get_user_providers(user_id=12345)
for conn in connections.connections:
    print(conn.provider_name, conn.status)

connection = await admin.get_user_provider(user_id=12345, provider_name="my-bot")
```

## IP Ban Management

```python
ban = await admin.ban_ip("192.168.1.100", duration_seconds=3600)  # 1 hour
print(ban.banned_until, ban.remaining_seconds)

ban = await admin.ban_ip("203.0.113.5")  # server default duration

status = await admin.get_ban_status("192.168.1.100")
if status.is_banned:
    print(f"Banned until {status.banned_until}")

result = await admin.unban_ip("192.168.1.100")
print(result.was_banned)  # False if it wasn't actually banned — safe to call unconditionally

bans = await admin.get_ban_list()
print(f"{bans.total} active bans")
for ban in bans.entries:
    print(ban.ip_address, ban.banned_until)
```

## Whitelist Management

Whitelisting takes precedence over bans for the listed range — use it for known-good infrastructure (your own servers, office networks) that should never be affected by rate limiting or the ban system:

```python
result = await admin.add_to_whitelist("10.0.0.0/24", description="Office network")
print(result.message)

whitelist = await admin.list_whitelist()
for entry in whitelist.entries:
    print(entry.ip_address, entry.description, entry.added_at)

result = await admin.remove_from_whitelist("10.0.0.0/24")
print(result.was_whitelisted)
```

## Usage Statistics

```python
from datetime import datetime, timedelta

# Predefined period
stats = await admin.get_stats(period="week")  # "day" | "week" | "month"

# Explicit range
stats = await admin.get_stats(
    start_date=datetime.now() - timedelta(days=30),
    end_date=datetime.now(),
)

# Omit everything for the API's default range
stats = await admin.get_stats()
```

Useful for a periodic reporting job, an internal dashboard, or alerting on unusual traffic — see [Common Integration Patterns](integration-patterns.md) for a scheduled-job shape you can adapt.

## Error handling for admin operations

Admin errors share `v2hub`'s exception hierarchy, so the same patterns from [Error Handling Patterns](error-handling.md) apply directly:

```python
from v2hub import VPNAPIError, AuthenticationError, AuthorizationError, NotFoundError

try:
    await admin.delete_user(12345)
except AuthenticationError:
    # invalid HMAC signature — almost always a wrong or misconfigured secret_key
    ...
except NotFoundError:
    # user_id doesn't exist — already deleted, or never existed
    ...
except AuthorizationError:
    # signature is valid but this operation isn't permitted
    ...
except VPNAPIError as e:
    print(e, e.recovery_hint)
```

## Where to go next

- [Authentication Workflows → Admin secret key](authentication.md#admin-secret-key) for how the `secret_key` itself is handled and secured.
- [Provider Workflows](providers.md) for the non-admin side of provider authorization.
- [V2Hub CLI → Admin Commands](../v2hub-cli/admin-commands.md) for every one of these operations from the terminal, including the graceful fallback behavior when `v2hub-admin` isn't installed.
