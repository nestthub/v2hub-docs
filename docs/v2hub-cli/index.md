# V2Hub CLI

Beautiful, user-friendly command-line interface for the V2Hub VPN Subscription API, built on top of the [v2hub](../v2hub/index.md) Python client, with optional admin commands.

- **Package**: [`v2hub-cli`](https://pypi.org/project/v2hub-cli/) on PyPI
- **Source**: [`nestthub/v2hub-cli`](https://github.com/nestthub/v2hub-cli)
- **Requires**: Python 3.10+
- **License**: MIT

```bash
pip install v2hub-cli
```

```bash
export V2HUB_API_URL="https://api.example.com"
export V2HUB_API_TOKEN="your-api-token"

v2hub list
v2hub create "my-vpn" -s vless://uuid@server1:443#Server1
```

## Features

- 🎨 **Beautiful output** — Rich-formatted tables and panels
- ⚡ **Tab completion** — commands, options, and real values (tokens, hashes, IPs) pulled live from the API
- 🔧 **Regular commands** — full self-service subscription management
- 🤝 **Provider commands** — manage subscriptions on behalf of end-users (`v2hub provider <user_id> ...`)
- 🔐 **Admin commands** — optional, requires `v2hub-admin`: users, providers, authorizations, IP bans, whitelist, stats
- 🎯 **Type safe** — built on [Typer](https://typer.tiangolo.com/), full type hints

## Documentation

| Page | Covers |
| --- | --- |
| [Installation](installation.md) | Installing the package, with and without admin support |
| [Configuration & Authentication](configuration.md) | Environment variables, `--base-url`/`--api-token`, admin's `--secret-key`, resolution order |
| [Shell Autocompletion](autocompletion.md) | Automatic setup, manual install, dynamic value completion, opting out |
| [Subscription Commands](subscription-commands.md) | `list`, `create`, `get`, `update`, `update-config`, `add/replace/remove-sources`, `delete`, `refresh`, `me`, `public` |
| [Connection Commands](connection-commands.md) | Managing the current user's connections to providers |
| [Provider Commands](provider-commands.md) | `v2hub provider <user_id> ...` — acting on behalf of an end-user |
| [Admin Commands](admin-commands.md) | Users, providers, authorizations, IP bans, whitelist, usage stats |
| [Common Workflows & Examples](examples.md) | End-to-end examples for everyday tasks |
| [Command Reference](reference.md) | Flat, alphabetized index of every command |

## Requirements

- **v2hub** (required, installed automatically) `>=1.1.2,<2.0.0`
- **v2hub-admin** `>=1.1.4,<2.0.0` (optional, for admin commands)
- Python `>=3.10`

## Extensions

- **[v2hub](https://github.com/nestthub/v2hub-core)** — the underlying Python client library.
- **[v2hub-admin](https://github.com/nestthub/v2hub-admin)** — admin API extension; installing it unlocks `v2hub admin ...`.

## Development

```bash
git clone https://github.com/nestthub/v2hub-cli.git
cd v2hub-cli
pip install -e ".[admin,dev]"

pytest              # run tests
mypy src/           # type checking
ruff check src/     # linting
```

## License

MIT License — see the [LICENSE](https://github.com/nestthub/v2hub-cli/blob/main/LICENSE) file for details.
