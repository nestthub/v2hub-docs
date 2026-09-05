# Provider Commands

If your API token belongs to a **provider** account, `v2hub provider <user_id> ...` manages subscriptions and the connection lifecycle on behalf of a specific end-user. Every top-level subscription command from [Subscription Commands](subscription-commands.md) has a provider-scoped counterpart here, plus connection lifecycle commands.

`user_id` is captured once, right after `provider`, and applies to every subcommand that follows:

```bash
v2hub provider <user_id> COMMAND [args...]
```

All commands accept `--base-url`/`-u` and `--api-token`/`-t` (a **provider** API token) — see [Configuration & Authentication](configuration.md).

## Connection Lifecycle

An approved connection (`connection-create`) must exist before any subscription command below will succeed for that `user_id`.

### `v2hub provider <user_id> connection-get`

Get the current authorization status between this provider and the user.

```bash
v2hub provider 98765 connection-get
```

### `v2hub provider <user_id> connection-create`

Create (or re-request/re-activate) a connection to the user. Prints the connection link to send them.

```bash
v2hub provider 98765 connection-create
```

### `v2hub provider <user_id> connection-revoke`

Revoke this provider's authorization for the user; the connection record is kept. Prompts for confirmation unless `--force`/`-f`.

```bash
v2hub provider 98765 connection-revoke
```

### `v2hub provider <user_id> connection-delete`

Permanently delete the connection record to the user. Prompts for confirmation unless `--force`/`-f`.

```bash
v2hub provider 98765 connection-delete --force
```

## Subscription Commands (Provider Context)

Identical semantics to their [top-level counterparts](subscription-commands.md), scoped to `user_id`:

```bash
v2hub provider <user_id> list
v2hub provider <user_id> create "vpn-name" -s vless://server1
v2hub provider <user_id> get <token>
v2hub provider <user_id> add-sources <token> -s vless://server1
v2hub provider <user_id> replace-sources <token> -s vless://server1
v2hub provider <user_id> remove-sources <token> -s <source-id> [--force]
v2hub provider <user_id> delete <token> [--force]
v2hub provider <user_id> update <token> --name "new-name"
v2hub provider <user_id> update-config <token> --config-id <id> [--hidden/--visible] [--max-depth 0-3]
v2hub provider <user_id> refresh <token>
```

The `--source`/`-s` syntax (plain string or JSON object with `hidden`/`depth`) is identical to the top-level commands — see [Source Syntax](subscription-commands.md#source-syntax).

## How Provider Scoping Works

Every subscription operation exists once, as a shared internal function taking an already-constructed client plus an optional user ID. Both the top-level commands and their `v2hub provider <user_id> ...` counterparts call that same function — they only differ in whether a user ID is forwarded. This is what keeps provider support from duplicating any request or formatting logic: nothing in the CLI talks HTTP directly, and the [`v2hub` client](../v2hub/clients.md#providers) is what actually routes the request to the provider-scoped endpoint and applies provider authorization. The provider token is whatever `--api-token`/`V2HUB_API_TOKEN` resolves to — the CLI does not treat it differently from a regular user token; that distinction belongs to the server and the underlying client.

## Example: Onboarding a New End-User

```bash
export V2HUB_API_URL="https://api.example.com"
export V2HUB_API_TOKEN="your-provider-api-token"

# 1. Create (or re-activate) the connection
v2hub provider 98765 connection-create

# 2. Once the user approves it, provision a subscription for them
v2hub provider 98765 create "welcome-vpn" -s vless://uuid@server1:443#Server1

# 3. Manage it going forward, exactly like a self-service subscription
v2hub provider 98765 list
v2hub provider 98765 get <token>
```
