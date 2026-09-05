# Connection Commands

`v2hub connection ...` manages the *current authenticated user's own* connections to providers — the mirror image of `v2hub provider <user_id> connection-*`, which operates from a provider's point of view about one of their users (see [Provider Commands](provider-commands.md)).

All commands accept `--base-url`/`-u` and `--api-token`/`-t` — see [Configuration & Authentication](configuration.md).

## `v2hub connection list`

List your provider connections (pending and approved) as a table: provider name, URL, authorized status, and connection status.

```bash
v2hub connection list
```

## `v2hub connection get <provider_name>`

Get your connection status for a specific provider.

```bash
v2hub connection get my-provider
```

## `v2hub connection approve <provider_name>`

Approve a pending provider connection request.

```bash
v2hub connection approve my-provider
```

## `v2hub connection reject <provider_name>`

Reject a pending provider connection request. Prompts for confirmation unless `--force`/`-f` is passed.

```bash
v2hub connection reject my-provider
v2hub connection reject my-provider --force
```

## `v2hub connection revoke <provider_name>`

Revoke your authorization for a provider. Prompts for confirmation unless `--force`/`-f` is passed.

```bash
v2hub connection revoke my-provider
```

Existing subscriptions from that provider remain available; the authorization record is preserved as `REVOKED` rather than deleted.

## Connection States

| Status | Meaning |
| --- | --- |
| `pending` | A provider has requested a connection; awaiting your approval |
| `approved` | The provider is authorized to act on your behalf |
| `revoked` | Authorization was revoked; the record is kept but no longer active |

## Typical Flow

```bash
# See who's requesting access
v2hub connection list

# Look at one in more detail
v2hub connection get my-provider

# Approve it
v2hub connection approve my-provider

# ...later, if you no longer want that provider to have access
v2hub connection revoke my-provider
```
