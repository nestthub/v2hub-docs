# Installation

## Basic Installation

```bash
pip install v2hub-cli
```

This installs the `v2hub` command and pulls in the [`v2hub`](../v2hub/index.md) client library automatically. Requires Python 3.10+.

## With Admin Support

Admin commands (`v2hub admin ...`) require the optional [`v2hub-admin`](https://github.com/nestthub/v2hub-admin) package:

```bash
pip install v2hub-cli[admin]
```

If `v2hub-admin` is not installed, the `admin` command group is hidden from `v2hub --help`, but `v2hub admin version` still works and reports the situation — see [Graceful Admin Fallback](admin-commands.md#graceful-admin-fallback).

## Development Install

```bash
git clone https://github.com/nestthub/v2hub-cli.git
cd v2hub-cli
pip install -e ".[admin,dev]"
```

`dev` pulls in `pytest`, `mypy`, `ruff`, and `v2hub-admin` so the full test suite (including admin commands) runs locally.

## Verifying the Install

```bash
v2hub version
```

```text
ⓘ Versions
v2hub: 1.1.2
v2hub-cli: 1.1.4
v2hub-admin: not installed
```

`v2hub-admin` shows `not installed` unless you installed the `[admin]` extra.

## Requirements

| Package | Version | Required |
| --- | --- | --- |
| `v2hub` | `>=1.1.2,<2.0.0` | Yes (installed automatically) |
| `v2hub-admin` | `>=1.1.4,<2.0.0` | No (enables `v2hub admin ...`) |
| Python | `>=3.10` | Yes |

## Next Steps

Once installed, set up [configuration and authentication](configuration.md) so `v2hub` knows which API to talk to.
