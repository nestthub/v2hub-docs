# Retries & Circuit Breaker

## Retry configuration

By default, every client method automatically retries on retryable errors (`TimeoutError`, `ServerError`, `ServiceUnavailableError`, `RateLimitError`) with exponential backoff and jitter. Configure it via `RetryConfig`, passed to the client constructor:

```python
from v2hub import AsyncVPNClient, RetryConfig

config = RetryConfig(
    max_retries=5,
    initial_delay=1.0,
    max_delay=60.0,
    exponential_base=2,
    jitter=True,
)

client = AsyncVPNClient(
    base_url="https://api.example.com",
    api_token="your-token",
    retry_config=config,
)
```

### `RetryConfig` fields

| Field | Type | Default | Description |
| --- | --- | --- | --- |
| `max_retries` | `int` | `3` | Maximum number of retry attempts after the initial try |
| `initial_delay` | `float` | `1.0` | Base delay in seconds before the first retry |
| `max_delay` | `float` | `60.0` | Upper bound on the computed delay |
| `exponential_base` | `float` | `2.0` | Multiplier applied per attempt: `initial_delay * exponential_base ** attempt`, capped at `max_delay` |
| `jitter` | `bool` | `True` | Randomize the computed delay by ±25% to avoid thundering-herd retries across clients |
| `retryable_exceptions` | `tuple[type[Exception], ...]` | `(TimeoutError, ServerError, ServiceUnavailableError, RateLimitError)` | Which exception types trigger a retry; other exceptions propagate immediately |

`RetryConfig.calculate_delay(attempt)` computes the backoff for a given (0-indexed) attempt number; you generally don't need to call this yourself, but it's public if you're building custom retry logic around the same policy.

If a `RateLimitError` carries a server-provided `retry_after`, that value is used as the delay directly instead of the computed exponential backoff — the server's guidance always takes precedence.

### How retries are applied

Every client method is wrapped with the `with_async_retry()` decorator (for `AsyncVPNClient`) internally — you don't apply this yourself. It's also exported for advanced use:

| Function | Description |
| --- | --- |
| `with_async_retry(config=None, circuit_breaker=None)` | Decorator factory for async functions; retries on `config.retryable_exceptions` up to `config.max_retries` times, sleeping between attempts per `calculate_delay()` (or the server's `retry_after` for rate limits) |
| `with_retry(config=None)` | Synchronous equivalent, for sync functions |

## Circuit breaker

A circuit breaker wraps calls to stop hammering a failing backend. Configure it via `CircuitBreakerConfig`, passed to the client constructor:

```python
from v2hub import AsyncVPNClient, CircuitBreakerConfig, VPNAPIError

breaker = CircuitBreakerConfig(
    failure_threshold=5,
    recovery_timeout=60.0,
    expected_exception=VPNAPIError,
)

client = AsyncVPNClient(
    base_url="https://api.example.com",
    api_token="your-token",
    circuit_breaker_config=breaker,
)
```

### `CircuitBreakerConfig` fields

| Field | Type | Default | Description |
| --- | --- | --- | --- |
| `failure_threshold` | `int` | `5` | Consecutive failures before the circuit opens |
| `success_threshold` | `int` | `2` | Successes required in `HALF_OPEN` before closing again |
| `timeout` | `float` | `60.0` | Seconds to wait in `OPEN` before trying `HALF_OPEN` |
| `enabled` | `bool` | `True` | Set `False` to disable the circuit breaker entirely (calls always pass straight through) |

### `CircuitState`

An enum representing the breaker's current state.

| State | Meaning |
| --- | --- |
| `CLOSED` | Normal operation; requests pass through |
| `OPEN` | The failure threshold was reached; requests are rejected immediately with `ServiceUnavailableError` without hitting the network, until `timeout` elapses |
| `HALF_OPEN` | After `timeout`, the next request(s) are allowed through as a test; enough consecutive successes (`success_threshold`) close the circuit again, while a single failure reopens it |

### `CircuitBreaker`

The class implementing the above, instantiated once per client from its `circuit_breaker_config`. Not normally constructed directly, but its public surface:

| Member | Description |
| --- | --- |
| `.state` | Current `CircuitState` |
| `.failure_count` | Consecutive failures recorded in the current window |
| `.success_count` | Consecutive successes recorded while `HALF_OPEN` |
| `.last_failure_time` | Timestamp (`time.time()`) of the most recent failure, or `None` |
| `async .call(func, *args, **kwargs)` | Execute `func` with circuit-breaker protection: raises `ServiceUnavailableError` immediately if `OPEN` and not yet due for a reset attempt; otherwise calls `func` and records success/failure |

## Interaction between retries and the circuit breaker

Both mechanisms are independent and composable: the retry decorator governs *within* a single logical call (re-attempting after transient failures), while the circuit breaker governs *across* calls over time (giving up early once a backend looks consistently unhealthy). When the circuit is `OPEN`, the `ServiceUnavailableError` it raises is itself one of the retryable exception types — so a call made while the circuit is open may still exhaust its own retry budget quickly rather than hanging, depending on your `RetryConfig`.
