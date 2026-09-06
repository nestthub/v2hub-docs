# Provider Workflows

How to build a service — a bot, a reseller panel, an automation tool — that manages subscriptions on behalf of end-users, rather than just for your own account. This is the "provider" role in V2Hub.

## Mental model

A **provider** is an account, authenticated the same way as any other (`API-Token` header), that has been granted permission to act for one or more **end-users**. Once authorized for a given `user_id`, the provider's client passes `as_provider_for_user_id=<user_id>` on subscription and source calls to operate on that user's subscriptions instead of its own:

```python
sub = await provider_client.create_subscription(
    "managed-vpn",
    sources=["vless://uuid@server1:443#Server1"],
    as_provider_for_user_id=98765,
)
```

One provider-token client can serve any number of end-users this way — you don't need a separate client instance per user, only a valid authorization per `user_id`.

## Connection lifecycle

Authorization is a separate handshake from authentication, and it goes through explicit states:

1. **Establish** — the provider creates (or re-approves) a connection to a `user_id`:

   ```python
   connection = await provider_client.create_provider_connection(user_id=98765)
   print(connection.connection_link)  # share this with the end-user if your flow requires their confirmation
   ```

2. **Check status** — at any point, either side can inspect the current state:

   ```python
   status = await provider_client.get_provider_connection(user_id=98765)
   print(status.status)  # ProviderAuthorizationStatus: approved / pending / revoked / unknown
   ```

3. **Act** — once approved, subscription and source calls with `as_provider_for_user_id=98765` succeed.

4. **Revoke** — when access is no longer needed:

   ```python
   await provider_client.revoke_provider_connection(user_id=98765)
   ```

   This ends the authorization without deleting historical records; use `delete_provider_connection` instead if you need to permanently remove the authorization record.

Note that `create_provider_connection` in the current `v2hub` client auto-approves rather than waiting on a separate end-user confirmation step — check [V2Hub → Sync & Async Clients](../v2hub/clients.md#provider-connection-management) and the server's own changelog for the current behavior, since this is documented as an evolving flow.

## The end-user's side of the relationship

End-users manage *their own* incoming provider connections through the regular (non-provider) client methods — this is the mirror image of the provider-side calls above:

```python
connections = await user_client.list_connections()
for conn in connections.connections:
    print(conn.provider_name, conn.status)

await user_client.approve_connection("my-provider")
# or
await user_client.reject_connection("my-provider")
# or, later, to revoke a previously-approved provider:
await user_client.revoke_connection("my-provider")
```

If you're building a provider service, you'll typically want to surface `approve_connection`/`reject_connection` in your *own* onboarding UI on behalf of the user (with their consent), or document that the user needs to run these calls (or the equivalent CLI commands — see [V2Hub CLI → Connection Commands](../v2hub-cli/connection-commands.md)) themselves.

## Onboarding a new end-user end-to-end

A typical provider onboarding flow, combining the above:

```python
from v2hub import AsyncVPNClient, ConflictError

async with AsyncVPNClient(base_url, provider_api_token) as provider:
    # 1. Establish authorization for the new user
    await provider.create_provider_connection(user_id=new_user_id)

    # 2. Provision their subscription
    try:
        sub = await provider.create_subscription(
            "welcome-vpn",
            sources=["vless://uuid@server1:443#Server1"],
            as_provider_for_user_id=new_user_id,
        )
    except ConflictError:
        # subscription with that name already exists for this user —
        # look it up instead of creating a duplicate
        subs = await provider.list_subscriptions(as_provider_for_user_id=new_user_id)
        sub = next(s for s in subs if s.name == "welcome-vpn")

    # 3. Hand the resolved subscription URL back to the user
    print(f"Your VPN config: {base_url}/sub/{sub.token}")
```

## Provider account lifecycle (admin-side)

Creating the *provider account itself* — as opposed to authorizing it for a specific user — is an administrative operation, not something the provider does to itself. See [Administration Workflows → Provider Management](administration.md#provider-management) for `create_provider`, token rotation, and enable/disable operations.

## Common pitfalls

- **Forgetting `as_provider_for_user_id`.** Omitting it always means "act on the provider's own account" — a call without it will silently create or modify a subscription owned by the provider itself, not the intended end-user. There's no implicit "current user" context to fall back on.
- **Acting before authorization is approved.** Calls with `as_provider_for_user_id` for a user who hasn't approved (or been auto-approved for) the provider fail with an authorization error — check status with `get_provider_connection` if you're unsure, especially after a revoke.
- **One-token-per-user thinking.** Resist the urge to mint a separate API token per end-user for a provider integration — the provider token plus `as_provider_for_user_id` *is* the mechanism for serving many users from one credential.

## Where to go next

- [Subscription Workflows](subscriptions.md) for everything you can do once you're operating in a given user's context.
- [Administration Workflows](administration.md) for provisioning provider accounts and managing the provider↔user authorization from the admin side.
- [V2Hub CLI → Provider Commands](../v2hub-cli/provider-commands.md) for the equivalent terminal workflow.
