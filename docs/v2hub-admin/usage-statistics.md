# Usage Statistics

## `get_stats(start_date=None, end_date=None, period=None)`

Get aggregated API usage statistics for the platform.

```python
from datetime import datetime, timedelta

# Predefined period
stats = admin.get_stats(period="week")
print(stats.general.total_users)
print(stats.general.new_users)
print(stats.general.new_subscriptions)

# Explicit date range
stats = admin.get_stats(
    start_date=datetime.now() - timedelta(days=30),
    end_date=datetime.now(),
)

# No arguments — uses the API's default range
stats = admin.get_stats()
```

| Argument | Type | Description |
| --- | --- | --- |
| `start_date` | `datetime \| None` | Optional start of the window; sent as ISO 8601 |
| `end_date` | `datetime \| None` | Optional end of the window; sent as ISO 8601 |
| `period` | `Literal["day", "week", "month"] \| None` | Optional predefined window, as an alternative to explicit dates |

**Returns:** `StatsResponse` — see [Statistics Models](#statistics-models).

Pass an explicit `start_date`/`end_date` range, a predefined `period`, or omit all arguments to use the API's default range. `start_date`/`end_date` and `period` are alternative ways of specifying the same thing — use whichever is more convenient for your use case; the client does not enforce mutual exclusivity itself, so which one takes precedence when both are supplied is determined by the server.

`None` values among `start_date`, `end_date`, and `period` are dropped from the request's query string entirely (rather than sent as empty parameters), keeping the request minimal.

## Statistics Models

### `GeneralStats`

General business metrics for the platform, for the selected window.

| Field | Type | Description |
| --- | --- | --- |
| `total_users` | `int` | Total number of registered users |
| `new_users` | `int` | New users within the selected period |
| `new_subscriptions` | `int` | New subscriptions within the selected period |

### `StatsResponse`

| Field | Type | Description |
| --- | --- | --- |
| `general` | `GeneralStats` | The general statistics payload |

## Related

- [`v2hub-cli` Usage Statistics](../v2hub-cli/admin-commands.md#usage-statistics) — the CLI wrapper around this API (`v2hub admin stats`), including `--period`/`--start-date`/`--end-date` flags.
