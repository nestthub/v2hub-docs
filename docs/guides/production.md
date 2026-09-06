# Production Usage

Settings and practices worth revisiting once you're past initial integration and preparing to run a V2Hub-backed service in production.

## Timeouts

The default `timeout=30.0` (seconds) is a reasonable general-purpose default, but tune it to your context:

```python
client = AsyncVPNClient(base_url, api_token, timeout=10.0)
```

- Lower it for latency-sensitive request paths (e.g. a user-facing API endpoint that calls `v2hub` synchronously as part of its own response) so a slow upstream doesn't stall your own service's response time budget.
- Raise it for background jobs (bulk refresh, reporting) where a slower-but-successful call is preferable to a spurious timeout.

## Retries

Retries are on by default (`max_retries=3`, exponential backoff with jitter) and cover `TimeoutError`, `ServerError`, `ServiceUnavailableError`, and `RateLimitError`. For most production services, the defaults in `RetryConfig` are a good starting point — adjust `max_retries`/`max_delay` mainly if you have a hard end-to-end latency budget:

```python
from v2hub import RetryConfig

client = AsyncVPNClient(
    base_url, api_token,
    retry_config=RetryConfig(max_retries=5, initial_delay=0.5, max_delay=10.0),
)
```

See [V2Hub → Retries & Circuit Breaker](../v2hub/retries.md) for every field. A couple of production-specific notes:

- If a caller-facing request has a strict latency SLA, cap `max_delay` well below it — three retries with a 60-second cap can, in the worst case, add nearly two minutes before an exception surfaces.
- `RateLimitError`'s `retry_after` (when the server provides one) is honored directly instead of the exponential backoff — don't fight this by setting a much shorter `max_delay`, since it will be overridden for that specific error type anyway.

## Circuit breaker

The circuit breaker is enabled by default and protects your service from hammering a backend that's already failing — useful in production specifically because it turns a slow-failing dependency into a fast-failing one, which is usually easier to handle gracefully upstream (fallback content, cached data, a clear "service unavailable" response) than a pile of slow timeouts:

```python
from v2hub import CircuitBreakerConfig

client = AsyncVPNClient(
    base_url, api_token,
    circuit_breaker_config=CircuitBreakerConfig(failure_threshold=10, timeout=30.0),
)
```

Raise `failure_threshold` for a noisy environment where occasional failures are expected and shouldn't trip the breaker; lower `timeout` if you want to attempt recovery more eagerly. See [V2Hub → Retries & Circuit Breaker → Circuit breaker](../v2hub/retries.md#circuit-breaker) for state semantics (`CLOSED`/`OPEN`/`HALF_OPEN`).

If your service already has its own circuit breaker or bulkhead pattern at a higher layer (e.g. in a service mesh), consider whether you need both — two independent breakers wrapping the same calls can interact in confusing ways (e.g. one opens while the other is still half-open). It's usually simpler to pick one layer to own this responsibility.

## Secrets management

Never hardcode `api_token` or (especially) `secret_key` in source. Load them from environment variables, a secrets manager, or your platform's equivalent:

```python
import os

client = AsyncVPNClient(
    base_url=os.environ["V2HUB_API_URL"],
    api_token=os.environ["V2HUB_API_TOKEN"],
)
```

For the admin `secret_key` specifically, treat it as more sensitive than a regular API token (see [Authentication Workflows → Admin secret key](authentication.md#admin-secret-key)) — restrict which processes/environments can read it, and rotate it if you suspect exposure.

## Logging and observability

The client doesn't log anything on its own by default. For visibility into request/response behavior, wrap the internal HTTP client with the built-in middleware described in [V2Hub → Requests & Responses → Built-in middleware](../v2hub/requests-responses.md#built-in-middleware-v2hubhttpmiddleware) if you're constructing your own `HTTPClient`, or — more commonly — log at your own call sites:

```python
import logging
import time

logger = logging.getLogger("v2hub_client")

async def get_subscription_logged(client, token):
    start = time.monotonic()
    try:
        sub = await client.get_subscription(token)
        logger.info("get_subscription ok token=%s duration=%.3fs", token, time.monotonic() - start)
        return sub
    except VPNAPIError as e:
        logger.error(
            "get_subscription failed token=%s duration=%.3fs error=%s",
            token, time.monotonic() - start, e,
        )
        raise
```

At minimum, log: which operation ran, how long it took, and — on failure — the exception type and `recovery_hint`. Avoid logging the raw `api_token`/`secret_key` even at debug level.

## Rate limits

`RateLimitError` is retried automatically using the server's `retry_after` when provided. Beyond that automatic handling, in production:

- Watch for a sustained pattern of `RateLimitError`s (e.g. via the logging above) as a signal to reduce request volume or negotiate a higher limit, rather than just letting retries mask it indefinitely.
- For bulk operations, prefer bounded concurrency (see [Common Integration Patterns → Fan-out subscription creation](integration-patterns.md#fan-out-subscription-creation-for-bulk-provisioning)) over firing everything at once, so you approach limits gradually instead of in a burst.

## Connection reuse

Construct one client per credential and reuse it for the lifetime of your process (see [Client Integration → Long-lived processes](client-integration.md#long-lived-processes-web-servers-workers)) rather than creating a new client per request. Beyond the connection-pooling benefit, this also means your `RetryConfig`/`CircuitBreakerConfig` state (in particular the circuit breaker's failure count) is shared and meaningful across requests, instead of resetting constantly.

## Where to go next

- [Error Handling Patterns](error-handling.md) for how to react to what production actually throws at you.
- [V2Hub → Retries & Circuit Breaker](../v2hub/retries.md) for the complete configuration reference.
- [End-to-End Examples](end-to-end-examples.md) for a fuller example combining production-oriented settings with real workflow code.
