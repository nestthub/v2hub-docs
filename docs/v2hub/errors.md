# Error Handling

Every error the client can raise is a subclass of `v2hub.VPNAPIError`, so catching that alone is always sufficient:

```python
from v2hub import (
    AsyncVPNClient,
    VPNAPIError,
    RateLimitError,
    NotFoundError,
)

async with AsyncVPNClient(base_url, token) as client:
    try:
        sub = await client.get_subscription("token123")
    except NotFoundError:
        print("Subscription not found")
    except RateLimitError as e:
        print(f"Rate limited. Retry after: {e.retry_after}")
    except VPNAPIError as e:
        if e.is_retryable:
            print(f"Retryable error: {e.recovery_hint}")
        else:
            print(f"Permanent error: {e}")
```

## `VPNAPIError`

Base exception for all errors raised by the client. Every other exception below subclasses it (directly or transitively).

### Constructor

| Argument | Type | Default | Description |
| --- | --- | --- | --- |
| `message` | `str` | — | The error message |
| `status_code` | `int \| None` | `None` | HTTP status code, if any |
| `response_data` | `dict[str, Any] \| None` | `None` | Raw error payload dict from the server, if any |
| `retry_after` | `int \| None` | `None` | Seconds to wait before retrying, if the server provided one |

### Attributes and properties

| Name | Type | Description |
| --- | --- | --- |
| `.message` | `str` | The error message |
| `.status_code` | `int \| None` | The HTTP status code |
| `.response_data` | `dict[str, Any]` | The raw error payload (empty dict if none) |
| `.retry_after` | `int \| None` | Seconds to wait, notably set on `RateLimitError` |
| `.is_retryable` | `bool` | `True` for `TimeoutError`, `ServerError`, `ServiceUnavailableError`, `RateLimitError`, and `NetworkError`; `False` otherwise |
| `.recovery_hint` | `str` | A short, human-readable suggestion for what to do next (overridden per subclass where relevant, otherwise a generic default) |

`str(exc)` renders as `"[status_code] message"` when a status code is present, or just `"message"` otherwise. `repr(exc)` includes the class name, message, status code, and retry-after value.

## Exception hierarchy

| Exception | Extends | Typical cause | `recovery_hint` |
| --- | --- | --- | --- |
| `ValidationError` | `VPNAPIError` | Request validation failed (400/422) | Check request parameters and fix validation errors |
| `InvalidURLError` | `ValidationError` | URL rejected by SSRF protection | Fix the source URL; internal/local/private URLs are blocked |
| `InvalidConfigError` | `ValidationError` | Invalid config format or unsupported structure | Validate config schema and unsupported fields |
| `AuthenticationError` | `VPNAPIError` | Invalid or missing API token (401) | Verify API token is valid and not expired |
| `AuthorizationError` | `VPNAPIError` | Insufficient permissions (403) | Verify your account has permission for this operation |
| `NotFoundError` | `VPNAPIError` | Resource not found (404) | Ensure the resource exists and token/name is correct |
| `SubscriptionNotFoundError` | `NotFoundError` | Subscription not found by token or name | Subscription not found - verify token/name or create new subscription |
| `SourceNotFoundError` | `NotFoundError` | Source ID not found | Source not found - verify source ID exists in subscription |
| `ConflictError` | `VPNAPIError` | Resource already exists / conflicts with current state (409) | Resource already exists - use update or choose different name |
| `DuplicateNameError` | `ConflictError` | Config with the same name already exists | Choose a unique name or update the existing item |
| `RateLimitError` | `VPNAPIError` | Rate limit exceeded (429) | Rate limited — wait `N` seconds (uses `.retry_after` if set) |
| `ServerError` | `VPNAPIError` | Internal server error (500) | Temporary server error - retry after delay |
| `ServiceUnavailableError` | `VPNAPIError` | Service unavailable (502/503) — also raised locally when the circuit breaker is `OPEN` | Service temporarily unavailable - retry later |
| `NetworkError` | `VPNAPIError` | Network/connection failure | Check network connectivity and retry |
| `TimeoutError` | `VPNAPIError` | Request or gateway timeout (504, or a client-side `httpx.TimeoutException`) | Request timed out - retry with longer timeout |
| `CircularReferenceError` | `VPNAPIError` | Circular reference detected between sources | Remove the dependency cycle and try again |
| `NestingTooDeepError` | `VPNAPIError` | Max nesting depth exceeded | Reduce nesting depth and retry |
| `TooManySubscriptionsError` | `VPNAPIError` | Per-account subscription limit reached | Remove some subscriptions or increase the limit |
| `TooManySourcesError` | `VPNAPIError` | Per-subscription source limit reached | Remove some sources or increase the limit |
| `TooManyConfigsError` | `VPNAPIError` | Per-subscription resolved-config limit reached | Remove some configs or increase the limit |
| `TooManyProvidersError` | `VPNAPIError` | Per-user approved-provider limit reached | Remove some providers or increase the limit |
| `InvalidAuthorizationStatusError` | `VPNAPIError` | Connection not in the expected status for this operation | The authorization is in an invalid state for this operation |
| `ExternalFetchError` | `VPNAPIError` | Failed to fetch an external URL source (network, DNS, or HTTP error) | Check external URL, DNS, TLS, and remote server availability |
| `CacheError` | `VPNAPIError` | Server-side cache operation failed | Inspect cache backend health and permissions |

Any exception not listed with a specific `recovery_hint` above falls back to: *"Contact API support if problem persists."*

## How errors are mapped

The client determines which exception type to raise from two, complementary sources, exposed as internal helper functions in `v2hub.core.exceptions` (not part of the public `__all__`, but useful to know about if you're debugging unexpected exception types):

- **`get_exception_for_status(status_code, message, response_data=None)`** — maps an HTTP status code to a typed exception, using the table below as a fallback, but preferring a more specific type if the response body contains a recognized `error` field.

  | Status | Default exception |
  | --- | --- |
  | 400 | `ValidationError` |
  | 401 | `AuthenticationError` |
  | 403 | `AuthorizationError` |
  | 404 | `NotFoundError` |
  | 409 | `ConflictError` |
  | 422 | `ValidationError` |
  | 429 | `RateLimitError` |
  | 500 | `ServerError` |
  | 502 / 503 | `ServiceUnavailableError` |
  | 504 | `TimeoutError` |
  | *(any other)* | `VPNAPIError` |

- **`get_exception_for_error(message, response_data=None, status_code=None)`** — maps a response body's `error` field directly to a typed exception, independent of (or in addition to) the HTTP status code. Useful when a backend returns a structured error object without reliable status-code semantics.

  | `error` / `error_code` / `code` / `type` value | Exception |
  | --- | --- |
  | `validation_error` | `ValidationError` |
  | `authentication_error` | `AuthenticationError` |
  | `authorization_error`, `permission_denied` | `AuthorizationError` |
  | `not_found` | `NotFoundError` |
  | `subscription_not_found` | `SubscriptionNotFoundError` |
  | `source_not_found` | `SourceNotFoundError` |
  | `conflict` | `ConflictError` |
  | `duplicate_name` | `DuplicateNameError` |
  | `rate_limit_exceeded` | `RateLimitError` |
  | `timeout` | `TimeoutError` |
  | `server_error` | `ServerError` |
  | `service_unavailable` | `ServiceUnavailableError` |
  | `invalid_url` | `InvalidURLError` |
  | `invalid_config` | `InvalidConfigError` |
  | `circular_reference` | `CircularReferenceError` |
  | `nesting_too_deep` | `NestingTooDeepError` |
  | `too_many_subscriptions` | `TooManySubscriptionsError` |
  | `too_many_providers` | `TooManyProvidersError` |
  | `too_many_configs` | `TooManyConfigsError` |
  | `too_many_sources` | `TooManySourcesError` |
  | `invalid_authorization_status` | `InvalidAuthorizationStatusError` |
  | `external_fetch_error` | `ExternalFetchError` |
  | `cache_error` | `CacheError` |
  | `network_error` | `NetworkError` |

- **`get_exception_for_exception(exc, message=None)`** — wraps a transport/runtime exception (e.g. from `httpx`) into the client's own hierarchy: exceptions whose class name contains `"Timeout"` become `TimeoutError`; those containing `"Connection"`, `"Network"`, `"DNS"`, `"SSLError"`, or `"ProxyError"` become `NetworkError`; anything else becomes a generic `VPNAPIError`.

In practice, a well-formed error body (with a recognized `error`/`error_code`/`code`/`type` field) always produces the most specific exception available, even if the raw HTTP status code alone would only map to a generic one — the error-type lookup takes priority over the status-code table.

The message shown on the raised exception is extracted from the response body's `message`, `detail`, `error_message`, or `description` field (in that order), falling back to the first entry of an `errors` list if present, and finally to a generic fallback string. A `retry_after` value is read from the response body the same way, when present, and attached to the exception regardless of which exception type was chosen.
