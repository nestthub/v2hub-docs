# End-to-End Examples

Complete, runnable scenarios that combine client integration, authentication, error handling, and production settings from the guides above into a single coherent example each.

## Example 1: A self-service signup flow (FastAPI)

New users sign up in your own app; on signup, provision a V2Hub subscription and return the public link. Demonstrates [Client Integration → long-lived processes](client-integration.md#long-lived-processes-web-servers-workers), [Subscription Workflows](subscriptions.md), and [Error Handling Patterns](error-handling.md).

```python
import os
from contextlib import asynccontextmanager

from fastapi import FastAPI, HTTPException
from v2hub import AsyncVPNClient, ConflictError, VPNAPIError


@asynccontextmanager
async def lifespan(app: FastAPI):
    app.state.v2hub = AsyncVPNClient(
        base_url=os.environ["V2HUB_API_URL"],
        api_token=os.environ["V2HUB_API_TOKEN"],
    )
    await app.state.v2hub.connect()
    yield
    await app.state.v2hub.close()


app = FastAPI(lifespan=lifespan)


@app.post("/signup/{username}/provision-vpn")
async def provision_vpn(username: str):
    client = app.state.v2hub
    sub_name = f"user-{username}-vpn"

    try:
        sub = await client.create_subscription(sub_name)
    except ConflictError:
        # Already provisioned (e.g. a retried request) — look it up instead
        # of failing the signup.
        subs = await client.list_subscriptions()
        sub = next((s for s in subs if s.name == sub_name), None)
        if sub is None:
            raise HTTPException(500, "subscription conflict but not found")
    except VPNAPIError as e:
        raise HTTPException(502, f"Could not provision VPN: {e.recovery_hint}")

    return {"subscription_url": f"{os.environ['V2HUB_API_URL']}/sub/{sub.token}"}
```

## Example 2: A provider bot onboarding and managing end-users

A Telegram-bot-style service that authorizes itself for each new user and lets them add their own config sources. Demonstrates [Provider Workflows](providers.md) end to end.

```python
import asyncio
from v2hub import AsyncVPNClient, NotFoundError, VPNAPIError

PROVIDER_TOKEN = "provider-api-token"
BASE_URL = "https://api.example.com"


async def onboard_user(client: AsyncVPNClient, telegram_user_id: int, v2hub_user_id: int):
    """Runs when a Telegram user first interacts with the bot."""
    await client.create_provider_connection(user_id=v2hub_user_id)

    sub = await client.create_subscription(
        f"tg-{telegram_user_id}",
        as_provider_for_user_id=v2hub_user_id,
    )
    return sub.token


async def add_server_for_user(client: AsyncVPNClient, v2hub_user_id: int, sub_token: str, config_uri: str):
    """Runs when the user sends the bot a new config string to add."""
    try:
        await client.add_sources(sub_token, [config_uri], as_provider_for_user_id=v2hub_user_id)
    except NotFoundError:
        # Subscription token the bot had stored is stale — treat as a bug
        # in the bot's own storage, not a user-facing retry case.
        raise RuntimeError(f"stored token {sub_token} no longer exists")
    except VPNAPIError as e:
        return f"Couldn't add that server: {e.recovery_hint}"
    return "Added!"


async def main():
    async with AsyncVPNClient(BASE_URL, PROVIDER_TOKEN) as client:
        token = await onboard_user(client, telegram_user_id=555111, v2hub_user_id=98765)
        print(f"Onboarded. Subscription URL: {BASE_URL}/sub/{token}")

        result = await add_server_for_user(
            client, v2hub_user_id=98765, sub_token=token,
            config_uri="vless://uuid@server1:443#Server1",
        )
        print(result)


asyncio.run(main())
```

## Example 3: A nightly admin report combining stats and stale-ban cleanup

An operations script, run via cron, that pulls weekly usage stats and clears expired-looking bans. Demonstrates [Administration Workflows](administration.md) and basic [Production Usage](production.md) logging.

```python
import asyncio
import logging
import os

from v2hub import VPNAPIError
from v2hub_admin import AsyncAdminClient

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger("nightly-report")


async def main():
    async with AsyncAdminClient(
        base_url=os.environ["V2HUB_API_URL"],
        secret_key=os.environ["V2HUB_ADMIN_SECRET"],
    ) as admin:
        try:
            stats = await admin.get_stats(period="week")
        except VPNAPIError as e:
            logger.error("failed to fetch weekly stats: %s", e)
            return

        logger.info("weekly stats: %s", stats)

        bans = await admin.get_ban_list()
        logger.info("%d active ban(s)", bans.total)
        for ban in bans.entries:
            status = await admin.get_ban_status(ban.ip_address)
            if not status.is_banned:
                # Ban expired between listing and checking — nothing to do,
                # but useful to note for auditing.
                logger.info("%s ban already expired", ban.ip_address)


asyncio.run(main())
```

## Example 4: CLI-driven bulk provisioning script

A shell script wrapping the CLI for a one-off bulk import, demonstrating [CLI Workflows](cli-workflows.md) scripting patterns.

```bash
#!/usr/bin/env bash
set -euo pipefail

export V2HUB_API_URL="https://api.example.com"
export V2HUB_API_TOKEN="provider-api-token"

while IFS=, read -r user_id config_uri; do
  echo "Provisioning user $user_id..."
  v2hub provider "$user_id" create "user-$user_id-vpn" -s "$config_uri" --no-color --quiet \
    || echo "  failed for $user_id, continuing"
done < users_to_provision.csv
```

Where `users_to_provision.csv` has lines like:

```
98765,vless://uuid@server1:443#Server1
98766,vless://uuid@server2:443#Server2
```

## Where to go next

These examples deliberately combine only two or three concerns at a time for clarity. For the full depth on any one piece, revisit:

- [Client Integration](client-integration.md), [Authentication Workflows](authentication.md)
- [Subscription Workflows](subscriptions.md), [Provider Workflows](providers.md), [Administration Workflows](administration.md)
- [Error Handling Patterns](error-handling.md), [Production Usage](production.md)
- [V2Hub → API Reference](../v2hub/reference.md) and [V2Hub CLI → Command Reference](../v2hub-cli/reference.md) for exhaustive lookups.
