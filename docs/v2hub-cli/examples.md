# Common Workflows & Examples

Set up authentication once per shell session (see [Configuration & Authentication](configuration.md)):

```bash
export V2HUB_API_URL="https://api.example.com"
export V2HUB_API_TOKEN="your-api-token"
```

## Create and Configure a Subscription

```bash
# Create subscription
v2hub create "work-vpn" --description "Office VPN servers"

# Add sources
v2hub add-sources <token> \
  -s vless://server1.example.com:443 \
  -s vmess://server2.example.com:443

# View subscription details, including its sources
v2hub get <token>
```

## List and Inspect

```bash
# List all your subscriptions
v2hub list

# Get one specific subscription
v2hub get <token>

# Fetch its public configs (what an end-user's VPN client sees)
v2hub public <token> --decode
```

## Update a Subscription

```bash
# Update name
v2hub update <token> --name "new-name"

# Update description
v2hub update <token> --description "Updated description"

# Update a specific config's comment/visibility/nesting depth
v2hub update-config <token> --config-id <id> --comment "Server 1" --hidden --max-depth 1

# Replace all sources
v2hub replace-sources <token> -s vless://new-server
```

## Managing Your Provider Connections

```bash
# See what's pending
v2hub connection list

# Approve or reject a request
v2hub connection approve trusted-provider
v2hub connection reject unwanted-provider

# Revoke access later
v2hub connection revoke trusted-provider
```

## Onboarding a New End-User as a Provider

```bash
export V2HUB_API_TOKEN="your-provider-api-token"

v2hub provider 98765 connection-create
v2hub provider 98765 create "welcome-vpn" -s vless://uuid@server1:443#Server1
v2hub provider 98765 list
```

See [Provider Commands](provider-commands.md) for the full command set available in provider context.

## Using JSON Sources for Per-Source Options

```bash
v2hub add-sources <token> \
  -s '{"data": "vless://uuid@server1:443#Server1", "hidden": true, "depth": 0}' \
  -s vmess://uuid@server2:443#Server2
```

The first source is created hidden with `max_depth=0`; the second uses default visibility and depth. See [Source Syntax](subscription-commands.md#source-syntax).

## Admin Operations

```bash
export V2HUB_ADMIN_SECRET="your-hmac-secret"

# Create a user
v2hub admin create-user 12345

# Get user info
v2hub admin get-user 12345

# Ban an IP for 1 hour
v2hub admin ban-ip 192.168.1.100 --duration 3600

# List whitelist entries
v2hub admin whitelist-list

# Check usage stats for the current week
v2hub admin stats --period week
```

See [Admin Commands](admin-commands.md) for the full reference, including the provider-authorization workflow.

## Scripting Without Rich Output

There is no `--format` flag on the CLI. For scripting or machine-readable output, use the [Python `v2hub` client](../v2hub/index.md) directly instead of parsing the CLI's Rich-formatted text:

```python
from v2hub import VPNClient

with VPNClient("https://api.example.com", "your-api-token") as client:
    for sub in client.list_subscriptions():
        print(sub.token, sub.name, sub.sources_count)
```
