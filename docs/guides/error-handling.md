# Error Handling Patterns

How to structure error handling around `v2hub` (and `v2hub-admin`, which shares the same exception hierarchy) in real applications — beyond the bare exception list in [V2Hub → Error Handling](../v2hub/errors.md).

## Start from the base class, narrow where it matters

Every error the client can raise is a `VPNAPIError`. A reasonable default shape for most call sites is: catch the few specific exceptions you need to react to differently, then a catch-all for everything else:

```python
from v2hub import (
    VPNAPIError,
    SubscriptionNotFoundError,
    AuthenticationError,
    RateLimitError,
)

try:
    sub = await client.get_subscription(token)
except SubscriptionNotFoundError:
    return None  # caller can treat this as "doesn't exist", a normal outcome
except AuthenticationError:
    raise  # a config/credentials problem — don't swallow it, let it surface loudly
except RateLimitError as e:
    logger.warning("rate limited, retry after %s", e.retry_after)
    raise
except VPNAPIError as e:
    logger.error("v2hub call failed: %s (%s)", e, e.recovery_hint)
    raise
```

Resist the temptation to catch only `VPNAPIError` everywhere "to be safe" — you lose the ability to distinguish "this subscription doesn't exist" (often a normal, expected outcome) from "the API is down" (an incident) from "our code sent bad input" (a bug to fix), which usually need very different responses.

## Distinguish retryable from non-retryable at the call site

The client already retries transient failures automatically (see [Production Usage → Retries](production.md#retries) and [V2Hub → Retries & Circuit Breaker](../v2hub/retries.md)), so by the time an exception reaches your code, retries are exhausted or the error was never retryable. Use `.is_retryable` if your own code needs to decide whether _additional_, application-level retry logic makes sense (e.g. queuing a background job for later):

```python
except VPNAPIError as e:
    if e.is_retryable:
        enqueue_retry_later(job, delay=e.retry_after or 30)
    else:
        mark_job_failed(job, reason=str(e))
```

## Surfacing errors to end-users vs. logging for operators

Keep the message shown to an end-user separate from what you log for debugging — `recovery_hint` is written for a developer, not an end-user:

```python
except VPNAPIError as e:
    logger.error("subscription op failed: %s | hint: %s", e, e.recovery_hint)
    raise UserFacingError("We couldn't update your VPN configuration. Please try again shortly.")
```

Exceptions worth a more specific end-user message include `SubscriptionNotFoundError` ("this link is no longer valid"), `RateLimitError` (surface `retry_after` as "try again in N seconds"), and `TooManySubscriptionsError`/`TooManySourcesError`/`TooManyConfigsError` (a plan/quota limit — tell the user what to remove or upgrade, not a generic error).

## Validation errors are (usually) bugs, not user input problems

`ValidationError` and its subclasses (`InvalidURLError`, `InvalidConfigError`) mean the _request your code built_ didn't pass validation — since the client validates client-side before sending, this is often a sign of a bug in the calling code (e.g. constructing a name that's too long) rather than something to retry or show generically to a user:

```python
except ValidationError as e:
    logger.error("built an invalid request: %s", e)  # investigate the caller, don't just retry
    raise
```

If the input genuinely came from an end-user (e.g. a subscription name typed into a form), validate it in your own application layer _before_ calling the client, so you can give immediate, field-specific feedback rather than relying on the API round-trip to catch it.

## Wrapping vs. propagating

If you're building a library or service on top of `v2hub`, decide deliberately whether to let `v2hub`'s exceptions propagate to your own callers or wrap them in your own exception types. Propagating is simpler and keeps `recovery_hint`/`retry_after` intact; wrapping is worth it if your callers shouldn't need to depend on `v2hub` directly:

```python
class SubscriptionServiceError(Exception):
    def __init__(self, message: str, *, cause: VPNAPIError):
        super().__init__(message)
        self.cause = cause
        self.is_retryable = cause.is_retryable

try:
    sub = await client.get_subscription(token)
except VPNAPIError as e:
    raise SubscriptionServiceError("Failed to load subscription", cause=e) from e
```

## A note on `update_comment`

`update_comment()` is deprecated in favor of `update_source()` and emits a `DeprecationWarning` on use — not an exception, so it won't be caught by any of the patterns above. If you have `DeprecationWarning`s elevated to errors in tests (`-W error`) or CI, migrate call sites to `update_source()` proactively; see [V2Hub → Sync & Async Clients](../v2hub/clients.md#update_source-vs-the-deprecated-update_comment).

## Where to go next

- [V2Hub → Error Handling](../v2hub/errors.md) for the complete exception hierarchy and how server responses map to each type.
- [Production Usage](production.md) for how retries and the circuit breaker interact with everything above.
- [Administration Workflows → Error handling for admin operations](administration.md#error-handling-for-admin-operations) for the admin-specific angle.
