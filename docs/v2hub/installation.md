# Installation & Setup

## Installation

```bash
pip install v2hub
```

Requires Python 3.10+.

Development install (for contributing to the client itself):

```bash
git clone https://github.com/nestthub/v2hub-core.git
cd v2hub-core
pip install -e ".[dev]"
```

## Client Initialization

Two clients are provided, sharing the same method surface (see [Sync & Async Clients](clients.md) for the full method reference):

- **`AsyncVPNClient`** — native `async`/`await`, for use inside an async application.
- **`VPNClient`** — a synchronous wrapper around `AsyncVPNClient` for non-async code. It manages its own event loop internally.

Both are constructed the same way:

```python
from v2hub import AsyncVPNClient, VPNClient

async_client = AsyncVPNClient(
    base_url="https://api.example.com",
    api_token="your-api-token",
)

sync_client = VPNClient(
    base_url="https://api.example.com",
    api_token="your-api-token",
)
```

### Constructor parameters

| Parameter | Type | Default | Description |
| --- | --- | --- | --- |
| `base_url` | `str` | — | Base URL of the V2Hub API (e.g. `https://api.example.com`) |
| `api_token` | `str` | — | API authentication token, sent as the `API-Token` header |
| `timeout` | `float` | `30.0` | Request timeout in seconds |
| `retry_config` | `RetryConfig \| None` | `None` | Custom retry behavior — see [Retries & Circuit Breaker](retries.md) |
| `circuit_breaker_config` | `CircuitBreakerConfig \| None` | `None` | Custom circuit breaker behavior — see [Retries & Circuit Breaker](retries.md) |

### Context manager usage

Both clients are context managers and should be used with `async with` / `with` so the underlying HTTP connection is opened and cleanly closed:

```python
async with AsyncVPNClient("https://api.example.com", "your-api-token") as client:
    ...
```

```python
with VPNClient("https://api.example.com", "your-api-token") as client:
    ...
```

### Manual connection lifecycle

If you need to manage the connection lifecycle manually (outside a context manager), call `connect()` / `close()` yourself:

```python
client = AsyncVPNClient("https://api.example.com", "your-api-token")
await client.connect()
try:
    ...
finally:
    await client.close()
```

`VPNClient` does not expose separate `connect()`/`close()` methods to call outside of a `with` block — see [Sync & Async Clients](clients.md#synchronous-client) for how it manages its event loop.

## Configuration

Client behavior is configured entirely through constructor arguments — there is no separate settings file or environment-variable loading in the client itself. The two configurable subsystems are retries and the circuit breaker; see [Retries & Circuit Breaker](retries.md) for every field and its default.

## Authentication

Every request carries your `api_token` in the `API-Token` header. There is no separate login step or token-refresh flow in the client — you obtain a token out of band (from a V2Hub admin, or the [v2hub-admin](https://github.com/nestthub/v2hub-admin) extension) and pass it to the client constructor.

Two kinds of tokens are meaningful to this client:

- **A regular user token** — used for normal self-service calls: managing your own subscriptions and your own provider connections.
- **A provider token** — used for everything a regular token can do, *plus* acting on behalf of end-users via the `as_provider_for_user_id` argument (see [Providers](clients.md#providers) in the client reference). A single client instance authenticated with a provider token can act both as a self-service user and as a provider for any number of end-users.

An invalid or expired token results in an `AuthenticationError` on any call — see [Error Handling](errors.md).
