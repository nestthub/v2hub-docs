# IP Bans & Whitelist

## IP Ban Management

### `ban_ip(ip_address, duration_seconds=None)`

Ban an IP address.

```python
# Ban for 1 hour
ban = admin.ban_ip("192.168.1.100", duration_seconds=3600)
print(ban.banned_until)
print(ban.remaining_seconds)

# Ban with the server's default duration
ban = admin.ban_ip("192.168.1.100")
```

| Argument | Type | Description |
| --- | --- | --- |
| `ip_address` | `str` | IP address to ban |
| `duration_seconds` | `int \| None` | Ban duration in seconds; omit to use the server's default setting |

**Returns:** `IPBanStatusResponse` — see [Access List Models](#access-list-models).

**Raises:** `ValidationError` (invalid IP address), `AuthenticationError`.

### `unban_ip(ip_address)`

Unban an IP address.

```python
result = admin.unban_ip("192.168.1.100")
if result.was_banned:
    print(f"Unbanned {result.ip_address}")
else:
    print(f"{result.ip_address} was not banned")
```

**Returns:** `IPUnbanResponse`.

**Raises:** `AuthenticationError`.

### `get_ban_status(ip_address)`

Check whether an IP is currently banned.

```python
status = admin.get_ban_status("192.168.1.100")
if status.is_banned:
    print(f"Banned until: {status.banned_until}")
    print(f"Remaining: {status.remaining_seconds}s")
else:
    print("Not banned")
```

**Returns:** `IPBanStatusResponse`.

**Raises:** `AuthenticationError`.

### `get_ban_list()`

Get all currently banned IPs.

```python
bans = admin.get_ban_list()
print(f"Total bans: {bans.total}")
for ban in bans.entries:
    print(f"  {ban.ip_address} until {ban.banned_until}")
```

**Returns:** `IPBanListResponse`.

**Raises:** `AuthenticationError`.

## Whitelist Management

### `add_to_whitelist(ip_address, description=None)`

Add an IP address or CIDR range to the whitelist.

```python
result = admin.add_to_whitelist("10.0.0.0/24", description="Office network")
print(result.message)
```

| Argument | Type | Description |
| --- | --- | --- |
| `ip_address` | `str` | IP address or CIDR to whitelist |
| `description` | `str \| None` | Optional description/reason, up to 255 characters |

**Returns:** `WhitelistAddResponse`.

**Raises:** `ValidationError` (invalid IP address/CIDR), `AuthenticationError`.

### `remove_from_whitelist(ip_address)`

Remove an IP address from the whitelist.

```python
result = admin.remove_from_whitelist("10.0.0.0/24")
if result.was_whitelisted:
    print(f"Removed {result.ip_address} from whitelist")
else:
    print(f"{result.ip_address} was not whitelisted")
```

**Returns:** `WhitelistRemoveResponse`.

**Raises:** `AuthenticationError`.

### `list_whitelist()`

Get all whitelisted IPs.

```python
whitelist = admin.list_whitelist()
print(f"Total entries: {whitelist.total}")
for entry in whitelist.entries:
    print(f"  {entry.ip_address}: {entry.description}")
    print(f"    Added: {entry.added_at}")
```

**Returns:** `WhitelistListResponse`.

**Raises:** `AuthenticationError`.

## Access List Models

### `IPBanRequest`

| Field | Type | Description |
| --- | --- | --- |
| `ip_address` | `str` | IP address to ban |
| `duration_seconds` | `int \| None` | Ban duration in seconds; defaults to the system setting |

### `IPUnbanRequest`

| Field | Type | Description |
| --- | --- | --- |
| `ip_address` | `str` (min length 8) | IP address to unban |

### `IPUnbanResponse`

| Field | Type | Description |
| --- | --- | --- |
| `ip_address` | `str` | IP address |
| `was_banned` | `bool` | Whether the IP was previously banned |
| `message` | `str` | Result message |

### `IPBanStatusResponse`

Returned by both `ban_ip()` and `get_ban_status()`.

| Field | Type | Description |
| --- | --- | --- |
| `ip_address` | `str` | Banned IP address |
| `is_banned` | `bool` | Whether the IP is now banned |
| `banned_until` | `str \| None` | Ban expiration time |
| `remaining_seconds` | `int \| None` (`>= 0`) | Seconds until unban |

### `IPBanEntry`

| Field | Type | Description |
| --- | --- | --- |
| `ip_address` | `str` | Banned IP address |
| `banned_until` | `str \| None` | Ban expiration time |

### `IPBanListResponse`

| Field | Type | Description |
| --- | --- | --- |
| `entries` | `list[IPBanEntry]` | All current bans |
| `total` | `int` | Total number of bans |

### `WhitelistAddRequest`

| Field | Type | Description |
| --- | --- | --- |
| `ip_address` | `str` | IP address or CIDR to whitelist |
| `description` | `str \| None` (max 255 chars) | Description/reason for whitelisting |

### `WhitelistAddResponse`

| Field | Type | Description |
| --- | --- | --- |
| `ip_address` | `str` | Whitelisted IP/CIDR |
| `description` | `str \| None` | Description |
| `message` | `str` | Result message |

### `WhitelistRemoveRequest`

| Field | Type | Description |
| --- | --- | --- |
| `ip_address` | `str` | IP address to remove |

### `WhitelistRemoveResponse`

| Field | Type | Description |
| --- | --- | --- |
| `ip_address` | `str` | IP address |
| `was_whitelisted` | `bool` | Whether the IP was previously whitelisted |
| `message` | `str` | Result message |

### `WhitelistEntry`

| Field | Type | Description |
| --- | --- | --- |
| `ip_address` | `str` | IP address or CIDR |
| `description` | `str \| None` | Description |
| `added_at` | `str` | When the entry was added |

### `WhitelistListResponse`

| Field | Type | Description |
| --- | --- | --- |
| `entries` | `list[WhitelistEntry]` | All whitelist entries (defaults to empty list) |
| `total` | `int` (`>= 0`) | Total number of entries |

## Related

- [`v2hub-cli` IP Ban Management](../v2hub-cli/admin-commands.md#ip-ban-management) / [Whitelist Management](../v2hub-cli/admin-commands.md#whitelist-management) — the CLI wrapper around this API, including tab-completion of banned/whitelisted IPs.
