# V2Hub

Professional Python client library for the V2Hub VPN Subscription API, with async/sync support, Pydantic v2 models, automatic retries with a circuit breaker, and a typed exception hierarchy.

- **Package**: [`v2hub`](https://pypi.org/project/v2hub/) on PyPI
- **Source**: [`nestthub/v2hub-core`](https://github.com/nestthub/v2hub-core)
- **Requires**: Python 3.10+
- **License**: MIT

```bash
pip install v2hub
```

```python
from v2hub import AsyncVPNClient

async with AsyncVPNClient("https://api.example.com", "your-api-token") as client:
    sub = await client.create_subscription(
        "my-vpn",
        sources=["vless://uuid@server1:443#Server1"],
    )
    public = await client.get_public_subscription(sub.token)
    print(public.get_configs())
```

## Documentation

| Page | Covers |
| --- | --- |
| [Installation & Setup](installation.md) | Installing the package, initializing `AsyncVPNClient` / `VPNClient`, configuration, authentication |
| [Sync & Async Clients](clients.md) | Using the synchronous and asynchronous clients, subscriptions, sources, providers, public access — full method reference |
| [Requests & Responses](requests-responses.md) | How requests are built and validated, how responses are parsed, and the internal HTTP client/middleware |
| [Typed Models](models.md) | Every request/response model and enum, field by field |
| [Error Handling](errors.md) | The exception hierarchy, error attributes, and how server errors map to client exceptions |
| [Retries & Circuit Breaker](retries.md) | `RetryConfig`, `CircuitBreakerConfig`, and how automatic retry/backoff works |
| [Usage Examples](examples.md) | End-to-end examples: self-service lifecycle, provider onboarding, explicit rate-limit handling |
| [API Reference](reference.md) | Flat, alphabetized index of every public class, method, and exception |

## Extensions

- **[v2hub-admin](https://github.com/nestthub/v2hub-admin)** — Admin API extension with HMAC authentication.
- **[v2hub-cli](https://github.com/nestthub/v2hub-cli)** — Command-line interface built on this client.

## Development

```bash
git clone https://github.com/nestthub/v2hub-core.git
cd v2hub-core
pip install -e ".[dev]"

pytest              # run tests
mypy src/           # type checking
ruff check src/     # linting
```

## License

MIT License — see the [LICENSE](https://github.com/nestthub/v2hub-core/blob/main/LICENSE) file for details.
