# Client Integration

How to bring the `v2hub` client into different kinds of applications, and how to manage its connection lifecycle correctly in each.

## Choosing async vs sync

- **`AsyncVPNClient`** — use this if your application is already async: FastAPI, aiohttp, a Telegram bot framework, an asyncio worker, a Discord bot, etc. It's the primary implementation; everything else wraps it.
- **`VPNClient`** — use this in plain scripts, Django views, Flask routes, Celery tasks, or any codebase that isn't `async def` all the way down. It manages an internal event loop for you.

Both expose the identical set of methods (`create_subscription`, `add_sources`, `get_public_subscription`, ...) — see [V2Hub → Sync & Async Clients](../v2hub/clients.md) for the full list. Everything below applies equally to both unless noted.

## Connection lifecycle

Always construct the client as a context manager:

```python
async with AsyncVPNClient(base_url, api_token) as client:
    ...
```

```python
with VPNClient(base_url, api_token) as client:
    ...
```

This opens the underlying HTTP connection on entry and closes it cleanly on exit, even if an exception is raised inside the block. Avoid constructing a client and letting it go out of scope without closing it — connections and any pooled resources won't be released promptly.

### Long-lived processes (web servers, workers)

For a web server or worker that needs a client across many requests, don't open a new `async with` block per request — construct one client at startup and reuse it:

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI
from v2hub import AsyncVPNClient

@asynccontextmanager
async def lifespan(app: FastAPI):
    app.state.v2hub = AsyncVPNClient(base_url, api_token)
    await app.state.v2hub.connect()
    yield
    await app.state.v2hub.close()

app = FastAPI(lifespan=lifespan)

@app.get("/subscriptions/{token}")
async def get_subscription(token: str):
    return await app.state.v2hub.get_subscription(token)
```

This is the manual `connect()`/`close()` pattern described in [V2Hub → Installation & Setup](../v2hub/installation.md#manual-connection-lifecycle), used here because a web framework's lifespan hooks — not a single `async with` block — naturally bound the client's lifetime.

### Short-lived scripts

For a one-off script, the `async with` / `with` form is simplest and sufficient:

```python
import asyncio
from v2hub import AsyncVPNClient

async def main():
    async with AsyncVPNClient(base_url, api_token) as client:
        subs = await client.list_subscriptions()
        for sub in subs:
            print(sub.name, sub.token)

asyncio.run(main())
```

### Multiple sequential calls with the sync client

`VPNClient` used outside a `with` block creates and tears down a temporary event loop per call (`asyncio.run(...)` under the hood), which is correct but wasteful for more than one call in a row. Prefer wrapping multiple calls in a single `with` block:

```python
with VPNClient(base_url, api_token) as client:
    sub = client.create_subscription("my-vpn")
    client.add_sources(sub.token, ["vless://uuid@server1:443#Server1"])
    client.update_source(sub.token, sub.sources[0].id, comment="Primary")
```

## One client instance vs. one per credential

A single `AsyncVPNClient`/`VPNClient` instance is tied to exactly one `api_token` at construction time. If your application needs to act under multiple credentials — for example, a regular user token _and_ a provider token — construct one client per credential rather than trying to swap tokens on a shared instance:

```python
user_client = AsyncVPNClient(base_url, user_api_token)
provider_client = AsyncVPNClient(base_url, provider_api_token)
```

A single provider-token client can still act for many different end-users via `as_provider_for_user_id=` (see [Provider Workflows](providers.md)) — that argument is per-call, not per-client, so you don't need one client per end-user.

## Where to go next

- [Authentication Workflows](authentication.md) for how to obtain and manage the `api_token` itself.
- [V2Hub → Installation & Setup](../v2hub/installation.md) for the full constructor reference (`timeout`, `retry_config`, `circuit_breaker_config`).
- [Production Usage](production.md) for tuning retries, timeouts, and observability once you're past initial integration.
