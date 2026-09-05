# Command Reference

Flat, alphabetized-within-section index of every `v2hub` command. See the linked pages for options, flags, and examples.

## Top-Level

| Command | Description |
| --- | --- |
| `v2hub version` | Show installed package versions |

## Subscription Commands

See [Subscription Commands](subscription-commands.md) for full details.

| Command | Description |
| --- | --- |
| `v2hub list` | List your subscriptions |
| `v2hub create <name>` | Create a new subscription |
| `v2hub get <token>` | Get subscription details |
| `v2hub update <token>` | Update subscription name/description |
| `v2hub update-config <token>` | Update a config's comment/visibility/depth |
| `v2hub update-comment <token>` | *(deprecated, use `update-config`)* Update a config's comment |
| `v2hub delete <token>` | Delete subscription |
| `v2hub add-sources <token> -s <uri>...` | Add sources |
| `v2hub remove-sources <token> -s <id>...` | Remove sources |
| `v2hub replace-sources <token> -s <uri>...` | Replace all sources |
| `v2hub refresh <token>` | Refresh external URL sources |
| `v2hub me` | Show your own account info |
| `v2hub public <token>` | Fetch a subscription's public configs |

## Connection Commands

See [Connection Commands](connection-commands.md) for full details.

| Command | Description |
| --- | --- |
| `v2hub connection list` | List your connections to providers |
| `v2hub connection get <provider_name>` | Get your connection status for a provider |
| `v2hub connection approve <provider_name>` | Approve a pending connection request |
| `v2hub connection reject <provider_name>` | Reject a pending connection request |
| `v2hub connection revoke <provider_name>` | Revoke your authorization for a provider |

## Provider Commands

See [Provider Commands](provider-commands.md) for full details.

| Command | Description |
| --- | --- |
| `v2hub provider <user_id> connection-get` | Get authorization status for a user |
| `v2hub provider <user_id> connection-create` | Create/re-approve authorization |
| `v2hub provider <user_id> connection-revoke` | Revoke authorization |
| `v2hub provider <user_id> connection-delete` | Permanently delete authorization record |
| `v2hub provider <user_id> list` | List that user's subscriptions |
| `v2hub provider <user_id> create <name>` | Create subscription for that user |
| `v2hub provider <user_id> get <token>` | Get their subscription details |
| `v2hub provider <user_id> update <token>` | Update their subscription |
| `v2hub provider <user_id> update-config <token>` | Update a config on their subscription |
| `v2hub provider <user_id> delete <token>` | Delete their subscription |
| `v2hub provider <user_id> add-sources <token>` | Add sources |
| `v2hub provider <user_id> replace-sources <token>` | Replace sources |
| `v2hub provider <user_id> remove-sources <token>` | Remove sources |
| `v2hub provider <user_id> refresh <token>` | Refresh external URL sources |

## Admin Commands (Optional)

Requires `v2hub-admin` — see [Admin Commands](admin-commands.md) for full details.

| Command | Description |
| --- | --- |
| `v2hub admin --help` | Show admin commands help |
| `v2hub admin version` | Show admin module version |
| `v2hub admin create-user <user_id>` | Create user |
| `v2hub admin get-user <user_id>` | Get user info |
| `v2hub admin delete-user <user_id>` | Delete user |
| `v2hub admin set-user-status <user_id>` | Activate/deactivate user |
| `v2hub admin refresh-token <user_id>` | Refresh user's API token |
| `v2hub admin create-provider <owner_hash> <name>` | Create provider account |
| `v2hub admin get-providers` | List all providers |
| `v2hub admin get-provider <provider_hash>` | Get provider info |
| `v2hub admin get-provider-by-name <name>` | Look up provider by name |
| `v2hub admin get-provider-by-owner-id <owner_id>` | Look up provider by owner ID |
| `v2hub admin delete-provider <provider_hash>` | Delete provider |
| `v2hub admin set-provider-status <provider_hash>` | Activate/deactivate provider |
| `v2hub admin update-provider-url <provider_hash>` | Update provider URL |
| `v2hub admin update-provider-name <provider_hash>` | Update provider name |
| `v2hub admin refresh-provider-token <provider_hash>` | Refresh provider's API token |
| `v2hub admin get-user-providers <user_id>` | List a user's provider connections |
| `v2hub admin get-user-provider <user_id> <name>` | Get one user/provider connection |
| `v2hub admin get-provider-authorization <name> <user_id>` | Get authorization state |
| `v2hub admin process-provider-authorization <user_id> <name>` | Process a connection invite |
| `v2hub admin approve-provider-authorization <user_id> <name>` | Approve a pending authorization |
| `v2hub admin reject-provider-authorization <user_id> <name>` | Reject/revoke an authorization |
| `v2hub admin ban-ip <ip_address>` | Ban an IP address |
| `v2hub admin unban-ip <ip_address>` | Unban an IP address |
| `v2hub admin ban-status <ip_address>` | Check an IP's ban status |
| `v2hub admin ban-list` | List banned IPs |
| `v2hub admin whitelist-add <ip_or_cidr>` | Add IP/CIDR to whitelist |
| `v2hub admin whitelist-remove <ip_or_cidr>` | Remove IP/CIDR from whitelist |
| `v2hub admin whitelist-list` | List whitelisted IPs |
| `v2hub admin stats` | Show usage statistics |

## Global Flags

| Flag | Short | Applies to | Overrides |
| --- | --- | --- | --- |
| `--base-url` | `-u` | All commands | `V2HUB_API_URL` |
| `--api-token` | `-t` | Regular & provider commands | `V2HUB_API_TOKEN` |
| `--secret-key` | `-k` | Admin commands | `V2HUB_ADMIN_SECRET` |
| `--install-completion` | — | Root command | Installs shell completion for the current shell |
| `--show-completion` | — | Root command | Prints the completion script |

## Exit Codes

| Code | Meaning |
| --- | --- |
| `0` | Success |
| `1` | Error (authentication, not found, validation, or any other API/CLI error) |
| `2` | Invalid command or arguments (raised by Typer/Click itself) |
