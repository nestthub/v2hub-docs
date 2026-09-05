# Requests & Responses

## Request/response flow

All request bodies and response payloads are represented as Pydantic v2 models (see [Typed Models](models.md)). Client methods:

1. Build the outgoing request model client-side (e.g. `SubscriptionCreateRequest`), applying the same validation constraints as the server.
2. Serialize it to JSON with `model_dump(mode="json", exclude_none=True)` and send the HTTP request.
3. Parse the JSON response into the corresponding response model (e.g. [`Subscription`](models.md#subscription)) and return it.

If client-side model construction fails validation, a Pydantic `ValidationError` is caught internally and re-raised as `v2hub.ValidationError` — callers only ever need to catch `v2hub`'s own exception hierarchy, never `pydantic.ValidationError` directly.

If the server returns a non-2xx response, the JSON error body is parsed and mapped to a specific exception subclass (see [Error Handling](errors.md)) rather than being returned as a raw HTTP error.

## Internal HTTP client (`v2hub.http`)

`AsyncVPNClient` is built on an internal `HTTPClient` (in `v2hub.http.client`), which wraps [`httpx.AsyncClient`](https://www.python-httpx.org/) and is not part of the package's public `__all__` — you don't normally construct or import it directly, but it's useful to understand when debugging or extending the client (e.g. writing custom middleware).

### `HTTPClient`

| Constructor argument | Type | Default | Description |
| --- | --- | --- | --- |
| `base_url` | `str` | — | Base URL for all requests (trailing slash stripped) |
| `headers` | `dict[str, str] \| None` | `None` | Default headers sent with every request (this is where `API-Token` and `Content-Type` are set) |
| `timeout` | `float` | `30.0` | Request timeout in seconds |
| `middleware` | `list[Middleware] \| None` | `None` | Ordered middleware chain — see below |

| Method | Description |
| --- | --- |
| `connect()` | Open the underlying `httpx.AsyncClient` (idempotent) |
| `close()` | Close the underlying client and release resources |
| `request(method, path, **kwargs)` | Run the full middleware chain, then execute the request; raises on 4xx/5xx (see below) |
| `get(path, **kwargs)` / `post(path, **kwargs)` / `put(path, **kwargs)` / `patch(path, **kwargs)` / `delete(path, **kwargs)` | Convenience wrappers around `request()` for each HTTP method |

`HTTPClient` is itself an async context manager (`async with HTTPClient(...) as http: ...`), mirroring `AsyncVPNClient`.

On a response with `status_code >= 400`, `HTTPClient` parses the JSON error body (falling back to raw response text if it isn't JSON) and raises the exception returned by `get_exception_for_status()` — see [Error Handling](errors.md). On `httpx.TimeoutException` it raises `v2hub.TimeoutError`; on `httpx.NetworkError` or any other `httpx.HTTPError` it raises `v2hub.NetworkError`.

### `RequestContext`

A small metadata container passed through the middleware chain for each request.

| Field | Type | Description |
| --- | --- | --- |
| `method` | `str` | HTTP method |
| `url` | `str` | Full request URL |
| `retries` | `int` | Number of retries attempted so far (default `0`) |
| `metadata` | `dict[str, Any]` | Free-form metadata dict for middleware to read/write |

### `Middleware`

Base class for request/response middleware. Subclass it and override `__call__` to observe or modify requests:

```python
from v2hub.http.client import Middleware, RequestContext

class MyMiddleware(Middleware):
    async def __call__(self, context: RequestContext, call_next):
        # runs before the request
        response = await call_next()
        # runs after the response
        return response
```

Middleware is applied in the order given, wrapping the actual HTTP call from the outside in (the first middleware in the list is the outermost).

### Built-in middleware (`v2hub.http.middleware`)

| Class | Description |
| --- | --- |
| `LoggingMiddleware(log_level=logging.DEBUG)` | Logs each request (`→ METHOD url`) and response (`← status METHOD url (duration)`) at the given log level; logs and re-raises on failure |
| `MetricsMiddleware()` | Tracks `request_count`, `error_count`, and `total_duration`; exposes `.average_duration`, `.error_rate` properties and a `.get_metrics()` dict |
| `RetryMiddleware()` | Tracks retry attempts on the request context; the actual retry/backoff logic lives in the `with_async_retry`/`with_retry` decorators (see [Retries & Circuit Breaker](retries.md)), not in this middleware |

None of these are wired in by default — `AsyncVPNClient` constructs its internal `HTTPClient` with no middleware. They're available for advanced use if you construct your own `HTTPClient` or need request-level observability beyond what the client already provides.
