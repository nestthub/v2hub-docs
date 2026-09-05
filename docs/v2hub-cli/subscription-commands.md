# Subscription Commands

These commands manage the *currently authenticated user's own* subscriptions. Every command here has a provider-scoped counterpart under `v2hub provider <user_id> ...` — see [Provider Commands](provider-commands.md).

All commands accept `--base-url`/`-u` and `--api-token`/`-t`, which override `V2HUB_API_URL`/`V2HUB_API_TOKEN` — see [Configuration & Authentication](configuration.md).

## `v2hub version`

Show installed package versions (`v2hub`, `v2hub-cli`, `v2hub-admin`).

```bash
v2hub version
```

## `v2hub list`

List all your subscriptions as a Rich table (name, token, config count, description).

```bash
v2hub list
```

## `v2hub create`

Create a new subscription, optionally with initial sources.

```bash
v2hub create "work-vpn" --description "Office VPN servers"

v2hub create "my-vpn" \
  -s vless://uuid@server1.example.com:443#Server1 \
  -s vmess://uuid@server2.example.com:443#Server2
```

| Argument/Option | Short | Description |
| --- | --- | --- |
| `name` | — | Subscription name (required) |
| `--description` | `-d` | Optional description |
| `--source` | `-s` | Repeatable; see [Source Syntax](#source-syntax) below |

## `v2hub get <token>`

Show subscription details, including a table of its sources.

```bash
v2hub get <token>
```

## `v2hub update <token>`

Update the subscription's name and/or description. Only the options you pass are changed.

```bash
v2hub update <token> --name "new-name"
v2hub update <token> --description "Updated description"
```

Passing neither `--name` nor `--description` prints a warning and does nothing.

## `v2hub update-config <token>`

Update a single config's comment, visibility, or nesting depth within the subscription. Only the fields you pass are changed; everything else is left as-is.

```bash
v2hub update-config <token> --config-id <id> --comment "Server 1"
v2hub update-config <token> --config-id <id> --hidden
v2hub update-config <token> --config-id <id> --visible
v2hub update-config <token> --config-id <id> --max-depth 1
```

| Option | Short | Description |
| --- | --- | --- |
| `--config-id` | `-i` | Config ID to update (required; tab-completable) |
| `--comment` | `-c` | New comment |
| `--hidden` / `--visible` | — | Hide or unhide this source's configs from end users |
| `--max-depth` | — | Max nesting depth, `0`–`3` |

Passing none of `--comment`/`--hidden`/`--visible`/`--max-depth` prints a warning and does nothing.

!!! note "Deprecated: `update-comment`"
    `v2hub update-comment <token> --config-id <id> --comment <text>` still works but is deprecated and hidden from `--help`. It only updates the comment. Use `update-config` instead, which covers the same case plus visibility and depth.

## `v2hub add-sources <token>`

Add one or more sources to an existing subscription.

```bash
v2hub add-sources <token> -s vless://server1 -s vmess://server2
```

## `v2hub replace-sources <token>`

Replace *all* of a subscription's sources with a new set.

```bash
v2hub replace-sources <token> -s vless://new-server
```

## `v2hub remove-sources <token>`

Remove one or more sources by ID. Prompts for confirmation unless `--force`/`-f` is passed.

```bash
v2hub remove-sources <token> -s <source-id-1> -s <source-id-2>
v2hub remove-sources <token> -s <source-id> --force
```

## `v2hub delete <token>`

Delete a subscription. Prompts for confirmation unless `--force`/`-f` is passed.

```bash
v2hub delete <token>
v2hub delete <token> --force
```

## `v2hub refresh <token>`

Refresh the subscription's external URL sources (fetches the latest configs from any `external_url` sources). Reports how many were refreshed, skipped (e.g. cooldown active), or failed.

```bash
v2hub refresh <token>
```

## `v2hub me`

Show information about the currently authenticated user (user ID, active status).

```bash
v2hub me
```

## `v2hub public <token>`

Fetch a subscription's public configs — the same content an end-user's VPN client would fetch from the subscription URL.

```bash
v2hub public <token>              # prints raw base64 content
v2hub public <token> --decode     # prints decoded configs, one per line
```

The public-configs endpoint itself doesn't require the token you authenticate with to belong to this subscription, but the CLI still needs a valid `--api-token`/`V2HUB_API_TOKEN` to make the request at all.

## Source Syntax

Every `--source`/`-s` option (on `create`, `add-sources`, `replace-sources`, and their provider equivalents) accepts two forms:

**Plain string** — a source URI, unchanged from the original calling convention:

```bash
-s "vless://uuid@server1.example.com:443#Server1"
```

**JSON object** — for per-source `hidden`/`depth` options, detected automatically because it starts with `{`:

```bash
-s '{"data": "vless://uuid@server1:443#Server1", "hidden": true, "depth": 0}'
```

| JSON field | Required | Meaning |
| --- | --- | --- |
| `data` | Yes | The source URI/string |
| `hidden` | No | Defaults to `false`; sets `is_hidden` on the created source |
| `depth` | No | Sets `max_depth` on the created source |

A single `--source`/`-s` list can freely mix plain strings and JSON objects — sources with no modifiers are sent exactly as before, so existing scripts that don't use JSON sources are unaffected.

## Output

Listings (`list`) render as a Rich table; single-resource commands (`get`, `create`, `update`, etc.) print a key/value panel. There is currently no `--format` flag — for scripting, parse the plain text output or use the [Python `v2hub` client](../v2hub/index.md) directly.

## Exit Codes

- `0` — Success
- `1` — Error (authentication, not found, validation, or any other API/CLI error — the CLI does not currently distinguish error types by exit code)
- `2` — Invalid command or arguments (raised by Typer/Click itself)
