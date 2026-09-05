# V2Hub Admin

Admin extension for [V2Hub](../v2hub/index.md), providing privileged operations for user, provider, and IP management through HMAC-SHA256 authentication. Async and sync clients, Pydantic v2 models, automatic retries — the same production-grade foundations as the base `v2hub` client.

- **Package**: [`v2hub-admin`](https://pypi.org/project/v2hub-admin/) on PyPI
- **Source**: [`nestthub/v2hub-admin`](https://github.com/nestthub/v2hub-admin)
- **Requires**: Python 3.10+
- **License**: MIT

```bash
pip install v2hub-admin
```

```python
from v2hub_admin import AdminClient

with AdminClient(
    base_url="https://api.example.com",
    secret_key="your-hmac-secret",
) as admin:
    user = admin.create_user(user_id=12345)
    print(user.api_token)
```

## Features

- 🔐 **HMAC Authentication** (SHA-256 signing) — a separate credential from the regular API token
- 👤 **User Management API** — user CRUD, activation, token refresh
- 🤝 **Provider Management API** — provider CRUD, status, URL/name updates, token refresh, lookups
- 🔑 **Provider Authorization API** — approve/reject a provider's access to a user's subscriptions
- 📊 **Usage Statistics API** — aggregated stats by date range or predefined period
- 🚫 **IP Ban System** — ban/unban, status checks, listing
- ✅ **Whitelist Management** — add/remove/list IPs and CIDR ranges
- 🔄 **Async & Sync clients** — `AsyncAdminClient` and `AdminClient`
- 📦 Built on top of [`v2hub`](../v2hub/index.md) — reuses its HTTP client, retry logic, and exception hierarchy
- 🛡️ Fully typed (type hints + Pydantic v2)

## Documentation

| Page | Covers |
| --- | --- |
| [Installation & Setup](installation.md) | Installing the package, initializing `AdminClient` / `AsyncAdminClient` |
| [Authentication & Authorization](authentication.md) | HMAC request signing, the secret key, how it differs from the regular API token, error types |
| [User Management](user-management.md) | Creating, fetching, deleting, activating/deactivating users; refreshing tokens |
| [Provider Management](provider-management.md) | Provider CRUD, status, URL/name updates, token refresh, lookups by hash/name/owner |
| [Provider Authorization](provider-authorization.md) | The connection handshake between a provider and a user: process, approve, reject |
| [IP Bans & Whitelist](ip-bans-and-whitelist.md) | Banning/unbanning IPs, checking status, listing bans; whitelist management |
| [Usage Statistics](usage-statistics.md) | `get_stats()` — predefined periods vs. explicit date ranges |
| [Integration Examples](examples.md) | End-to-end examples combining multiple admin operations |
| [API Reference](reference.md) | Flat, alphabetized index of every method, model, and field |

## Requirements

- **v2hub** `>=1.1.2` (installed automatically)
- Python `>=3.10`

## Extensions

- **[v2hub](https://github.com/nestthub/v2hub-core)** — the underlying Python client library.
- **[v2hub-cli](https://github.com/nestthub/v2hub-cli)** — command-line interface; `v2hub admin ...` wraps this library when `v2hub-cli[admin]` is installed.

## Development

```bash
git clone https://github.com/nestthub/v2hub-admin.git
cd v2hub-admin
pip install -e ".[dev]"

pytest              # run tests
mypy src/           # type checking
ruff check src/     # linting
```

## Security Notes

- Do not hardcode `secret_key` in source code — use environment variables or a secret manager.
- Use HTTPS only in production.
- Rotate keys regularly.
- The Admin API has full access to the system — treat the secret key with the same care as a root credential.

## Changelog

See [CHANGELOG.md](https://github.com/nestthub/v2hub-admin/blob/main/CHANGELOG.md) for release notes.

## License

MIT License — see the [LICENSE](https://github.com/nestthub/v2hub-admin/blob/main/LICENSE) file for details.
