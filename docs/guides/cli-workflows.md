# CLI Workflows

Everyday tasks from the terminal using `v2hub-cli`, and how to combine it with the Python client when a task needs both. Full command-by-command reference lives under [V2Hub CLI](../v2hub-cli/index.md); this guide focuses on the workflows.

## Setup, once per shell/environment

```bash
pip install v2hub-cli          # add [admin] for admin commands: pip install v2hub-cli[admin]

export V2HUB_API_URL="https://api.example.com"
export V2HUB_API_TOKEN="your-api-token"
```

See [V2Hub CLI → Configuration & Authentication](../v2hub-cli/configuration.md) for the full variable/flag list, including the separate `V2HUB_ADMIN_SECRET` used by admin commands.

## Everyday subscription management

```bash
v2hub list
v2hub create "my-vpn" -s vless://uuid@server1:443#Server1
v2hub get <token>
v2hub add-sources <token> -s vmess://uuid@server2:443#Server2
v2hub update <token> --name "renamed-vpn"
v2hub public <token>       # resolved, decoded content — what end-users actually consume
v2hub refresh <token>      # force-refresh external URL sources
v2hub delete <token>
```

Full flag reference: [V2Hub CLI → Subscription Commands](../v2hub-cli/subscription-commands.md).

## Managing your own provider connections

```bash
v2hub connection list
v2hub connection approve <provider_name>
v2hub connection reject <provider_name>
v2hub connection revoke <provider_name>
```

See [V2Hub CLI → Connection Commands](../v2hub-cli/connection-commands.md).

## Acting as a provider

Every subscription command has a provider-scoped counterpart under `v2hub provider <user_id> ...`, using a provider API token:

```bash
export V2HUB_API_TOKEN="your-provider-token"

v2hub provider 98765 create "welcome-vpn" -s vless://uuid@server1:443#Server1
v2hub provider 98765 list
```

See [V2Hub CLI → Provider Commands](../v2hub-cli/provider-commands.md) and [Provider Workflows](providers.md) for the underlying model.

## Administration from the terminal

Requires `v2hub-cli[admin]` and `V2HUB_ADMIN_SECRET` (or `--secret-key`) instead of an API token:

```bash
v2hub admin create-user 12345
v2hub admin ban-ip 192.168.1.100 --duration 3600
v2hub admin get-stats --period week
```

If `v2hub-admin` isn't installed, the `admin` group is hidden from `--help`, but `v2hub admin version` still works and reports the situation — see [V2Hub CLI → Graceful Admin Fallback](../v2hub-cli/admin-commands.md#graceful-admin-fallback). Full command list: [V2Hub CLI → Admin Commands](../v2hub-cli/admin-commands.md).

## Scripting with the CLI

For shell scripts and CI jobs, disable Rich's decorative output and parse plain values instead:

```bash
v2hub list --no-color --quiet
```

See [V2Hub CLI → Common Workflows & Examples → Scripting Without Rich Output](../v2hub-cli/examples.md#scripting-without-rich-output) for the exact flags and an example of piping CLI output into other tools.

## When to reach for the CLI vs. the Python client

- **CLI**: one-off inspection ("what sources does this subscription have"), quick fixes, admin tasks you do by hand, shell scripts and cron jobs where spinning up Python isn't worth it.
- **Python client**: anything embedded in application logic — a web backend, a bot, a background worker — or workflows that need branching logic, error recovery, or structured data beyond what's convenient to parse from CLI output.

The two aren't mutually exclusive within one system: it's common to provision and debug via the CLI while the production integration itself uses `v2hub`/`v2hub-admin` directly. Both ultimately call the same API and share the same [error hierarchy](../v2hub/errors.md), so troubleshooting knowledge transfers directly between them.

## Where to go next

- [V2Hub CLI → Command Reference](../v2hub-cli/reference.md) for a flat index of every command.
- [Authentication Workflows](authentication.md) for how `V2HUB_API_TOKEN`/`V2HUB_ADMIN_SECRET` relate to the credentials used by the Python clients.
