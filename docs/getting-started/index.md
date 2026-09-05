# Getting Started

This guide gets a new developer from zero to a first successful V2Hub API call: what the ecosystem looks like, what you need before starting, how to install and configure the client, and how to make and handle your first request.

## The V2Hub Ecosystem

V2Hub is a small ecosystem of tools built around one VPN Subscription API:

| Component | What it's for | Docs |
| --- | --- | --- |
| **v2hub** | Python client library (async & sync) for the API — subscriptions, sources, providers | [V2Hub](../v2hub/index.md) |
| **v2hub-admin** | Admin extension for privileged operations (users, providers, bans, stats), HMAC-authenticated | [V2Hub Admin](../v2hub-admin/index.md) |
| **v2hub-cli** | Command-line interface built on `v2hub`, with optional admin commands | [V2Hub CLI](../v2hub-cli/index.md) |

Most application developers only need **v2hub**, the Python client — that's what the rest of this guide walks through. `v2hub-admin` is for operators managing the platform itself, and `v2hub-cli` is a terminal-friendly alternative to writing Python for quick tasks and scripting. Both are built on top of `v2hub` and reuse the same configuration and error-handling model described below.

For the API server itself, self-hosting, and operator-facing resources, see [v2hub.link](https://v2hub.link). For the full list of repositories (web panel, Telegram bot, etc.), see the [V2Hub Ecosystem overview](https://github.com/nestthub/nestthub/tree/main/ecosystems/v2hub).

## Prerequisites

Before you start, you'll need:

- **Python 3.10 or later** — check with `python3 --version`.
- **Access to a V2Hub API instance** — a `base_url` (e.g. `https://api.example.com`) that you or your organization runs, or a hosted instance you've been given access to. This guide doesn't cover deploying the API server itself; see [v2hub.link](https://v2hub.link) for that.
- **An API token** — V2Hub has no self-service signup or login flow. Tokens are issued out of band by whoever administers your V2Hub instance (using `v2hub-admin`, directly against the database, or via the web panel). Ask them for a token before continuing.

## Installation

Install the Python client with pip:

```bash
pip install v2hub
```

That's the only package you need for application code. See [V2Hub → Installation & Setup](../v2hub/installation.md) for the development install (editable install with test/lint tooling) if you plan to contribute to the client itself.

!!! tip "Prefer the command line?"
    If you'd rather manage subscriptions from a terminal than write Python, install [`v2hub-cli`](../v2hub-cli/index.md) instead (`pip install v2hub-cli`) and skip to its own [Installation](../v2hub-cli/installation.md) and [Configuration](../v2hub-cli/configuration.md) pages — the concepts below (base URL, token, errors) are the same either way.

## Basic Configuration

Every client instance needs two things: the API's `base_url` and your `api_token`. There's no config file or environment-variable auto-loading built into the `v2hub` package itself — you pass these explicitly when constructing a client (a common pattern is to read them from your own app's environment variables or secrets manager first):

```python
from v2hub import AsyncVPNClient

client = AsyncVPNClient(
    base_url="https://api.example.com",
    api_token="your-api-token",
)
```

A synchronous client with the identical interface is also available for non-async code:

```python
from v2hub import VPNClient

client = VPNClient(
    base_url="https://api.example.com",
    api_token="your-api-token",
)
```

Both accept the same optional `timeout`, `retry_config`, and `circuit_breaker_config` arguments — defaults are sensible for getting started, and you can tune them later. See [V2Hub → Installation & Setup](../v2hub/installation.md) for the full list of constructor parameters.

## Authentication Setup

Your `api_token` is sent as the `API-Token` header on every request — there's no login call or token refresh in the client. There are two kinds of tokens:

- **A regular user token** — for managing your own subscriptions.
- **A provider token** — for everything a regular token does, plus acting on behalf of end-users (via `as_provider_for_user_id=`) once they've approved your access. Most getting-started scenarios only need a regular token.

Keep your token out of source control. A common pattern is to load it from an environment variable:

```python
import os
from v2hub import AsyncVPNClient

client = AsyncVPNClient(
    base_url=os.environ["V2HUB_API_URL"],
    api_token=os.environ["V2HUB_API_TOKEN"],
)
```

See [V2Hub → Authentication](../v2hub/installation.md#authentication) for the full token model, including how provider tokens work.

## Making Your First Request

Always use the client as a context manager (`async with` / `with`) so the underlying HTTP connection opens and closes cleanly. Here's a complete, runnable first request — create a subscription and read back its public, resolved content:

```python
import asyncio
from v2hub import AsyncVPNClient


async def main():
    async with AsyncVPNClient(
        base_url="https://api.example.com",
        api_token="your-api-token",
    ) as client:
        # Create a subscription with one config source
        sub = await client.create_subscription(
            "my-first-vpn",
            sources=["vless://uuid@server1:443#Server1"],
        )
        print(f"Created subscription: {sub.token}")

        # Fetch the resolved, public subscription content
        public = await client.get_public_subscription(sub.token)
        print(f"Resolved {public.config_count} config(s):")
        print(public.decode())


asyncio.run(main())
```

The synchronous equivalent, if you're not writing async code:

```python
from v2hub import VPNClient

with VPNClient(
    base_url="https://api.example.com",
    api_token="your-api-token",
) as client:
    sub = client.create_subscription(
        "my-first-vpn",
        sources=["vless://uuid@server1:443#Server1"],
    )
    print(f"Created subscription: {sub.token}")

    public = client.get_public_subscription(sub.token)
    print(f"Resolved {public.config_count} config(s):")
    print(public.decode())
```

`sub.token` is the identifier you'll use for every later call on this subscription (`get_subscription`, `add_sources`, `delete_subscription`, ...), and it's also the value that goes into the public subscription URL served at `/sub/{token}` — the link you'd paste into a VPN client. See [V2Hub → Sync & Async Clients](../v2hub/clients.md) for the complete method reference (subscriptions, sources, providers, public access).

## Basic Error Handling

Every error the client can raise is a subclass of `VPNAPIError`, so catching that alone is always a safe baseline. Catch more specific exceptions first when you want to handle particular cases differently:

```python
from v2hub import AsyncVPNClient, VPNAPIError, NotFoundError, AuthenticationError

async with AsyncVPNClient(base_url, api_token) as client:
    try:
        sub = await client.get_subscription("some-token")
    except NotFoundError:
        print("That subscription doesn't exist")
    except AuthenticationError:
        print("Invalid or expired API token")
    except VPNAPIError as e:
        print(f"Request failed: {e} — {e.recovery_hint}")
```

Transient failures (timeouts, 5xx responses, rate limits) are retried automatically with exponential backoff before an exception ever reaches your code, so the `except` blocks above only run once retries are exhausted or the error isn't retryable. See [V2Hub → Error Handling](../v2hub/errors.md) for the full exception hierarchy, and [V2Hub → Retries & Circuit Breaker](../v2hub/retries.md) for how the automatic retry behavior works and how to tune it.

## Next Steps

You've installed the client, configured a connection, made a request, and handled an error — from here:

| I want to... | Go to |
| --- | --- |
| See every method the client offers (subscriptions, sources, providers, public access) | [V2Hub → Sync & Async Clients](../v2hub/clients.md) |
| Look up a specific model or field | [V2Hub → Typed Models](../v2hub/models.md) |
| See more end-to-end examples (provider onboarding, rate-limit handling, etc.) | [V2Hub → Usage Examples](../v2hub/examples.md) |
| Manage subscriptions from the terminal instead of Python | [V2Hub CLI](../v2hub-cli/index.md) |
| Perform admin operations (users, providers, IP bans, stats) | [V2Hub Admin](../v2hub-admin/index.md) |
| Browse every class, method, and exception in one place | [V2Hub → API Reference](../v2hub/reference.md) |
