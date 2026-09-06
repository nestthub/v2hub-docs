# Guides

Practical, task-oriented guides for common V2Hub development workflows. These walk through *how to accomplish a task end-to-end*; for exhaustive field-by-field detail, see the [V2Hub](../v2hub/index.md), [V2Hub Admin](../v2hub-admin/index.md), and [V2Hub CLI](../v2hub-cli/index.md) reference docs, which these guides link to throughout.

New to V2Hub? Start with [Getting Started](../getting-started/index.md) first — it covers installation and your very first request. The guides below assume that groundwork is already done.

## Guides

| Guide | Covers |
| --- | --- |
| [Client Integration](client-integration.md) | Choosing async vs sync, wiring the client into a web app, worker, or script, connection lifecycle |
| [Authentication Workflows](authentication.md) | Token types, where tokens come from, rotating tokens, admin HMAC auth |
| [Subscription Workflows](subscriptions.md) | Creating, updating, and resolving subscriptions; managing sources day to day |
| [Provider Workflows](providers.md) | Building a service that manages subscriptions on behalf of end-users |
| [Administration Workflows](administration.md) | User and provider lifecycle, IP bans, whitelisting, usage stats, via `v2hub-admin` or `v2hub admin` |
| [CLI Workflows](cli-workflows.md) | Everyday tasks from the terminal, scripting, and mixing the CLI with the Python client |
| [Error Handling Patterns](error-handling.md) | Structuring `try`/`except` around `VPNAPIError`, retry-aware code, user-facing error messages |
| [Common Integration Patterns](integration-patterns.md) | Recurring shapes: sync-on-signup, periodic refresh, provider webhooks, config caching |
| [Production Usage](production.md) | Timeouts, retries, circuit breakers, logging, secrets, observability, rate limits |
| [End-to-End Examples](end-to-end-examples.md) | Complete, runnable scenarios that combine several of the above |

## How to use these guides

Each guide is self-contained but assumes you have a `base_url` and an `api_token` (or admin `secret_key`) already, as described in [Getting Started](../getting-started/index.md). Code samples default to the async client (`AsyncVPNClient`); where the sync client (`VPNClient`) differs, the guide says so explicitly — the two share an identical method surface, so translating between them is mechanical (drop `await`, use `with` instead of `async with`).
