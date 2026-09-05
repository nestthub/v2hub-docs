# API Reference

Flat, alphabetized index of everything importable from `v2hub`. See the linked page for full details on each.

## Clients

| Name | Description |
| --- | --- |
| [`AsyncVPNClient`](clients.md#asynchronous-client) | Async API client |
| [`VPNClient`](clients.md#synchronous-client) | Sync API client (wraps `AsyncVPNClient`) |

## Configuration

| Name | Description |
| --- | --- |
| [`CircuitBreakerConfig`](retries.md#circuitbreakerconfig-fields) | Circuit breaker behavior |
| [`CircuitState`](retries.md#circuitstate) | Enum: `CLOSED`, `OPEN`, `HALF_OPEN` |
| [`RetryConfig`](retries.md#retryconfig-fields) | Retry/backoff behavior |

## Models — subscriptions & sources

| Name | Description |
| --- | --- |
| [`RefreshSubscriptionResponse`](models.md#refreshsubscriptionresponse) | Result of `refresh_subscription()` |
| [`Source`](models.md#source) | A single resolved source within a subscription |
| [`SourceAddRequest`](models.md#sourceaddrequest) | *Internal* request model behind `add_sources()` |
| [`SourceCreate`](models.md#sourcecreate) | Input shape for adding/replacing sources |
| [`SourceRemoveRequest`](models.md#sourceremoverequest) | *Internal* request model behind `remove_sources()` |
| [`SourceReplaceRequest`](models.md#sourcereplacerequest) | *Internal* request model behind `replace_sources()` |
| [`SourceType`](models.md#sourcetype) | Enum: `config`, `external_url`, `internal_token` |
| [`SourceUpdateRequest`](models.md#sourceupdaterequest) | *Internal* request model behind `update_source()` |
| [`Subscription`](models.md#subscription) | Full subscription resource |
| [`SubscriptionCreateRequest`](models.md#subscriptioncreaterequest) | *Internal* request model behind `create_subscription()` |
| [`SubscriptionListItem`](models.md#subscriptionlistitem) | Subscription as returned by `list_subscriptions()` |
| [`SubscriptionUpdateRequest`](models.md#subscriptionupdaterequest) | *Internal* request model behind `update_subscription()` |
| [`CommentUpdateRequest`](models.md#commentupdaterequest-deprecated) | **Deprecated** — request model behind `update_comment()` |

## Models — public access

| Name | Description |
| --- | --- |
| [`PublicSubscriptionResponse`](models.md#publicsubscriptionresponse) | Resolved public subscription content, with `.decode()` / `.get_configs()` / `.config_count` |

## Models — providers & self-service

| Name | Description |
| --- | --- |
| [`ConnectionResponse`](models.md#connectionresponse) | A single provider connection (self-service view) |
| [`ConnectionsResponse`](models.md#connectionsresponse) | Result of `list_connections()` |
| [`MeResponse`](models.md#meresponse) | Result of `get_me()` |
| [`ProviderAuthorizationStatus`](models.md#providerauthorizationstatus) | Enum: `approved`, `pending`, `revoked`, `unknown` |
| [`ProviderConnectionCreateResponse`](models.md#providerconnectioncreateresponse) | Result of `create_provider_connection()` |
| [`ProviderConnectionDeleteResponse`](models.md#providerconnectiondeleteresponse) | Result of `delete_provider_connection()` |
| [`ProviderConnectionRequest`](models.md#providerconnectionrequest) | *Internal* request model for provider-connection endpoints |
| [`ProviderConnectionResponse`](models.md#providerconnectionresponse) | A provider→user connection (provider view) |

## Models — errors

| Name | Description |
| --- | --- |
| [`ErrorResponse`](models.md#errorresponse) | Shape of the server's JSON error body |

## Exceptions

| Name | Description |
| --- | --- |
| [`AuthenticationError`](errors.md#exception-hierarchy) | Invalid or missing API token (401) |
| [`AuthorizationError`](errors.md#exception-hierarchy) | Insufficient permissions (403) |
| [`CacheError`](errors.md#exception-hierarchy) | Server-side cache operation failed |
| [`CircularReferenceError`](errors.md#exception-hierarchy) | Circular reference between sources |
| [`ConflictError`](errors.md#exception-hierarchy) | Resource conflict (409) |
| [`DuplicateNameError`](errors.md#exception-hierarchy) | Name already exists |
| [`ExternalFetchError`](errors.md#exception-hierarchy) | Failed to fetch an external URL source |
| [`InvalidAuthorizationStatusError`](errors.md#exception-hierarchy) | Connection not in the expected status |
| [`InvalidConfigError`](errors.md#exception-hierarchy) | Invalid config format |
| [`InvalidURLError`](errors.md#exception-hierarchy) | URL rejected by SSRF protection |
| [`NestingTooDeepError`](errors.md#exception-hierarchy) | Max nesting depth exceeded |
| [`NetworkError`](errors.md#exception-hierarchy) | Network/connection failure |
| [`NotFoundError`](errors.md#exception-hierarchy) | Resource not found (404) |
| [`RateLimitError`](errors.md#exception-hierarchy) | Rate limit exceeded (429) |
| [`ServerError`](errors.md#exception-hierarchy) | Internal server error (500) |
| [`ServiceUnavailableError`](errors.md#exception-hierarchy) | Service unavailable (502/503) |
| [`SourceNotFoundError`](errors.md#exception-hierarchy) | Source ID not found |
| [`SubscriptionNotFoundError`](errors.md#exception-hierarchy) | Subscription not found |
| [`TimeoutError`](errors.md#exception-hierarchy) | Request or gateway timeout |
| [`TooManyConfigsError`](errors.md#exception-hierarchy) | Per-subscription config limit reached |
| [`TooManyProvidersError`](errors.md#exception-hierarchy) | Per-user provider limit reached |
| [`TooManySourcesError`](errors.md#exception-hierarchy) | Per-subscription source limit reached |
| [`TooManySubscriptionsError`](errors.md#exception-hierarchy) | Per-account subscription limit reached |
| [`ValidationError`](errors.md#exception-hierarchy) | Request validation failed (400/422) |
| [`VPNAPIError`](errors.md#vpnapierror) | Base exception for all errors |

## Internals (not exported from the top-level package)

| Name | Module | Description |
| --- | --- | --- |
| [`CircuitBreaker`](retries.md#circuitbreaker) | `v2hub.core.retry` | Implements the circuit breaker state machine |
| [`HTTPClient`](requests-responses.md#httpclient) | `v2hub.http.client` | Underlying HTTP client wrapping `httpx.AsyncClient` |
| [`LoggingMiddleware`](requests-responses.md#built-in-middleware-v2hubhttpmiddleware) | `v2hub.http.middleware` | Logs requests/responses |
| [`MetricsMiddleware`](requests-responses.md#built-in-middleware-v2hubhttpmiddleware) | `v2hub.http.middleware` | Tracks request count/duration/error rate |
| [`Middleware`](requests-responses.md#middleware) | `v2hub.http.client` | Base class for HTTP middleware |
| [`RequestContext`](requests-responses.md#requestcontext) | `v2hub.http.client` | Per-request metadata passed through middleware |
| [`RetryMiddleware`](requests-responses.md#built-in-middleware-v2hubhttpmiddleware) | `v2hub.http.middleware` | Tracks retry attempts on the request context |
| `get_exception_for_error()` | `v2hub.core.exceptions` | Maps an error-body `error` field to a typed exception |
| `get_exception_for_exception()` | `v2hub.core.exceptions` | Wraps a transport/runtime exception (e.g. `httpx`) into the client's hierarchy |
| `get_exception_for_status()` | `v2hub.core.exceptions` | Maps an HTTP status code to a typed exception |
| `with_async_retry()` | `v2hub.core.retry` | Decorator applying retry logic to async methods |
| `with_retry()` | `v2hub.core.retry` | Decorator applying retry logic to sync functions |

## Package metadata

| Name | Description |
| --- | --- |
| `v2hub.__version__` | Installed package version |
| `v2hub.__author__` | Package author metadata |
| `v2hub.__api_version__` | API version path segment used for all requests (currently `"v1"`) |
