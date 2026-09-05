# Admin Commands

Admin commands (`v2hub admin ...`) require the optional [`v2hub-admin`](https://github.com/nestthub/v2hub-admin) package — see [Installation](installation.md#with-admin-support). They authenticate with `--secret-key`/`-k` (or `V2HUB_ADMIN_SECRET`) — HMAC auth — instead of `--api-token`. See [Configuration & Authentication](configuration.md).

Admin commands are deliberately independent of provider support: they're excluded from the provider context and always operate against the admin secret-key auth mechanism, never a provider/user API token.

```bash
v2hub admin --help
v2hub admin version
```

## User Management

```bash
v2hub admin create-user <user_id>
v2hub admin get-user <user_id>
v2hub admin delete-user <user_id>
v2hub admin set-user-status <user_id> --active|--inactive
v2hub admin refresh-token <user_id>
```

| Command | Description |
| --- | --- |
| `create-user <user_id>` | Create a user record for an external user ID |
| `get-user <user_id>` | Show user info |
| `delete-user <user_id>` | Delete a user |
| `set-user-status <user_id> --active/--inactive` | Activate or deactivate a user |
| `refresh-token <user_id>` | Issue a new API token for the user, invalidating the old one |

## Provider Management

```bash
v2hub admin create-provider <owner_hash> <provider_name> [--provider-url <url>]
v2hub admin get-providers
v2hub admin get-provider <provider_hash>
v2hub admin get-provider-by-name <provider_name>
v2hub admin get-provider-by-owner-id <owner_id>
v2hub admin delete-provider <provider_hash>
v2hub admin set-provider-status <provider_hash> --active|--inactive
v2hub admin update-provider-url <provider_hash> --provider-url <url>
v2hub admin update-provider-name <provider_hash> --provider-name <name>
v2hub admin refresh-provider-token <provider_hash>
```

| Command | Description |
| --- | --- |
| `create-provider <owner_hash> <name>` | Register a new provider account owned by `owner_hash` |
| `get-providers` | List all providers |
| `get-provider <provider_hash>` | Get provider info (tab-completable hash) |
| `get-provider-by-name <name>` | Look up a provider by its public name |
| `get-provider-by-owner-id <owner_id>` | Look up a provider by its owning user's ID |
| `delete-provider <provider_hash>` | Delete a provider |
| `set-provider-status <provider_hash> --active/--inactive` | Activate or deactivate a provider |
| `update-provider-url <provider_hash> --provider-url <url>` | Change the provider's URL |
| `update-provider-name <provider_hash> --provider-name <name>` | Rename the provider |
| `refresh-provider-token <provider_hash>` | Issue a new API token for the provider |

`<provider_hash>` supports tab completion, pulling live provider hashes from `get-providers`.

## User ↔ Provider Connections

```bash
v2hub admin get-user-providers <user_id>
v2hub admin get-user-provider <user_id> <provider_name>
```

| Command | Description |
| --- | --- |
| `get-user-providers <user_id>` | List a user's provider connections |
| `get-user-provider <user_id> <name>` | Get one specific user/provider connection |

## Provider Authorization Workflow

The commands below are the admin/operator view of the same connection lifecycle end-users manage themselves via [Connection Commands](connection-commands.md) and providers manage via [`v2hub provider <user_id> connection-*`](provider-commands.md#connection-lifecycle).

```bash
v2hub admin get-provider-authorization <provider_name> <user_id>
v2hub admin process-provider-authorization <user_id> <provider_name> [--hmac <hmac>]
v2hub admin approve-provider-authorization <user_id> <provider_name>
v2hub admin reject-provider-authorization <user_id> <provider_name> [--force]
```

| Command | Description |
| --- | --- |
| `get-provider-authorization <name> <user_id>` | Get the current authorization state |
| `process-provider-authorization <user_id> <name> [--hmac <hmac>]` | Process a connection invite; omit `--hmac` to query only |
| `approve-provider-authorization <user_id> <name>` | Approve a pending authorization |
| `reject-provider-authorization <user_id> <name> [--force]` | Reject/revoke an authorization; prompts for confirmation unless `--force` |

For `reject-provider-authorization`, the reported outcome is either `Deleted` (the server removed the record outright, when no subscriptions existed under it) or the resulting status, e.g. `revoked` (when subscriptions still exist and the record is kept).

## IP Ban Management

```bash
v2hub admin ban-ip <ip_address> [--duration <seconds>]
v2hub admin unban-ip <ip_address>
v2hub admin ban-status <ip_address>
v2hub admin ban-list
```

| Command | Description |
| --- | --- |
| `ban-ip <ip> [--duration/-d <seconds>]` | Ban an IP address; omit `--duration` for an indefinite ban |
| `unban-ip <ip>` | Remove a ban (tab-completable from the current ban list) |
| `ban-status <ip>` | Check whether an IP is currently banned, and for how long |
| `ban-list` | List all currently banned IPs, with ban ID, expiry, and remaining time |

```bash
# Ban for 1 hour
v2hub admin ban-ip 192.168.1.100 --duration 3600
```

## Whitelist Management

```bash
v2hub admin whitelist-add <ip_or_cidr> [--description <text>]
v2hub admin whitelist-remove <ip_or_cidr>
v2hub admin whitelist-list
```

| Command | Description |
| --- | --- |
| `whitelist-add <ip_or_cidr> [--description/-d <text>]` | Add an IP or CIDR range to the whitelist |
| `whitelist-remove <ip_or_cidr>` | Remove an entry (tab-completable from the current whitelist) |
| `whitelist-list` | List all whitelisted entries with their descriptions |

## Usage Statistics

```bash
v2hub admin stats
v2hub admin stats --period day
v2hub admin stats --period week
v2hub admin stats --period month
v2hub admin stats --start-date 2024-01-01T00:00:00 --end-date 2024-01-31T23:59:59
```

| Option | Short | Description |
| --- | --- | --- |
| `--period` | `-p` | Predefined window: `day`, `week`, or `month` |
| `--start-date` | `-s` | Explicit ISO 8601 start of the window |
| `--end-date` | `-e` | Explicit ISO 8601 end of the window |

`--period` and explicit `--start-date`/`--end-date` are alternatives — use whichever is more convenient. With none of the three, stats cover all time. Output includes total users, new users, and new subscriptions for the selected window.

## Graceful Admin Fallback

When `v2hub-admin` is not installed, the `admin` command group is hidden from `v2hub --help`, but `v2hub admin version` still works and reports the situation instead of the whole CLI failing to start:

```bash
$ v2hub admin version
Error: admin client is not available. Install 'v2hub_admin'.

$ v2hub --help
# Shows only regular and provider commands; the admin section is hidden
```

If `v2hub-admin` is installed but fails to import for some other reason (a runtime error during initialization), the CLI prints a warning and disables admin commands for that run — the rest of the CLI is unaffected:

```text
Warning: admin CLI failed to initialize and will be disabled. Reason: <error>
```
