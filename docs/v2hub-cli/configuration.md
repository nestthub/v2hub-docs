# Configuration & Authentication

There is no config file. The CLI is configured entirely through environment variables and per-command flags, which always take precedence over the environment.

## Environment Variables

```bash
# Required for regular/provider commands
export V2HUB_API_URL="https://api.example.com"
export V2HUB_API_TOKEN="your-api-token"

# Required for admin commands instead of V2HUB_API_TOKEN
export V2HUB_ADMIN_SECRET="your-hmac-secret"
```

| Variable | Used by | Purpose |
| --- | --- | --- |
| `V2HUB_API_URL` | all commands | Base URL of the V2Hub API |
| `V2HUB_API_TOKEN` | regular & provider commands | Bearer-style API token |
| `V2HUB_ADMIN_SECRET` | admin commands | HMAC secret key for admin authentication |
| `V2HUB_NO_AUTOCOMPLETE` | shell setup | Opt out of automatic tab-completion install — see [Shell Autocompletion](autocompletion.md) |

## Per-Command Flags

Every regular/provider command also accepts explicit flags, which override the environment variables for that invocation:

| Flag | Short | Overrides |
| --- | --- | --- |
| `--base-url` | `-u` | `V2HUB_API_URL` |
| `--api-token` | `-t` | `V2HUB_API_TOKEN` |

```bash
v2hub list --base-url https://api.example.com --api-token your-api-token
```

Admin commands use a different pair of flags, since they authenticate with HMAC instead of a bearer token:

| Flag | Short | Overrides |
| --- | --- | --- |
| `--base-url` | `-u` | `V2HUB_API_URL` |
| `--secret-key` | `-k` | `V2HUB_ADMIN_SECRET` |

```bash
v2hub admin get-user 12345 --base-url https://api.example.com --secret-key your-hmac-secret
```

## Resolution Order

For any given setting, the CLI resolves it in this order, using the first non-empty value:

1. The explicit flag (`--base-url`, `--api-token`, `--secret-key`)
2. The matching environment variable

If neither is set, the command fails immediately with a clear error, before making any network call:

```text
Error: API token not provided. Use --api-token or V2HUB_API_TOKEN env var
```

## Authentication Model

The CLI does not perform a login step — you obtain credentials out of band and pass them in:

- **API token** (`--api-token` / `V2HUB_API_TOKEN`) — a regular user token or a provider token, obtained from a V2Hub admin or the `v2hub-admin` extension. A provider token can also drive `v2hub provider <user_id> ...` commands on behalf of end-users.
- **Admin secret key** (`--secret-key` / `V2HUB_ADMIN_SECRET`) — an HMAC secret used only by `v2hub admin ...` commands. This is a completely separate credential from the API token; admin commands never use `--api-token`.

An invalid or expired token results in an API error surfaced by the CLI — see [Command Reference](reference.md) and the underlying client's [Error Handling](../v2hub/errors.md) for details on what each error means.
