# Installation & Setup

## Installation

```bash
pip install v2hub-admin
```

This pulls in [`v2hub`](../v2hub/index.md) `>=1.1.2` automatically — the admin package is a thin extension built on top of it and reuses its HTTP client, retry logic, and exception hierarchy. Requires Python 3.10+.

## Initializing the Client

Both clients take the same two required arguments: the API base URL, and an HMAC **secret key** (not a regular API token — see [Authentication & Authorization](authentication.md)).

### Async

```python
from v2hub_admin import AsyncAdminClient

async with AsyncAdminClient(
    base_url="https://api.example.com",
    secret_key="your-hmac-secret",
) as admin:
    user = await admin.get_user(12345)
    print(user)
```

### Sync

```python
from v2hub_admin import AdminClient

with AdminClient(
    base_url="https://api.example.com",
    secret_key="your-hmac-secret",
) as admin:
    user = admin.get_user(12345)
    print(user)
```

`AdminClient` is a synchronous wrapper around `AsyncAdminClient`: it manages its own event loop inside the `with` block (creating one on `__enter__`, running every call through `run_until_complete`, and tearing it down on `__exit__`), so every method has the identical signature and behavior as its async counterpart, just without `await`. Used outside a `with` block, each call falls back to `asyncio.run(...)` for that single call.

## Constructor Parameters

Both `AdminClient` and `AsyncAdminClient` accept:

| Parameter | Type | Default | Description |
| --- | --- | --- | --- |
| `base_url` | `str` | — | API base URL, e.g. `"https://api.example.com"` |
| `secret_key` | `str` | — | Admin secret key used for HMAC-SHA256 request signing |
| `timeout` | `float` | `30.0` | Request timeout in seconds |
| `retry_config` | `RetryConfig \| None` | `None` | Custom retry configuration; falls back to `v2hub`'s default `RetryConfig()` — see [Retries & Circuit Breaker](../v2hub/retries.md) |

## Context Manager Lifecycle

Both clients are async-context-manager (`async with`) / context-manager (`with`) friendly, and this is the recommended way to use them — it ensures the underlying HTTP connection pool is properly opened and closed:

```python
async with AsyncAdminClient(base_url, secret_key) as admin:
    ...  # connection is open here
# connection is closed here
```

If you need to manage the lifecycle manually instead (e.g. a long-lived client held elsewhere in your application), call `connect()`/`close()` directly:

```python
admin = AsyncAdminClient(base_url, secret_key)
await admin.connect()
try:
    ...
finally:
    await admin.close()
```

## Next Steps

See [Authentication & Authorization](authentication.md) for details on how the secret key is used to sign requests, and how it's distinct from the API tokens used by the regular [`v2hub`](../v2hub/index.md) client.
