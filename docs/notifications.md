# Publish API

Publish an event to Lanby from any script, cron job, application, or service with a single HTTP call. Lanby routes it to your configured destinations — the same ones your monitor alerts use.

## Authentication

All API requests require an API key passed as a Bearer token in the `Authorization` header. Create and manage API keys in the console under **Settings → API Keys**.

```
Authorization: Bearer lnby_live_...
```

Keep your API key secret. If one is compromised, revoke it from the console and generate a new one.

## Publishing an event

```http
POST https://api.lanby.dev/v1/notifications
Content-Type: application/json
Authorization: Bearer lnby_live_...

{
  "title":    "Backup completed",
  "body":     "Full snapshot took 4m 12s — 142 GB archived.",
  "topic":    "homelab",
  "priority": 3
}
```

A successful response returns `202 Accepted` with the notification ID:

```json
{ "id": "ntf_01j..." }
```

## Fields

| Field | Required | Description |
|---|---|---|
| `title` | Yes | Short summary. Shown as the message heading in Telegram and webhook payloads. |
| `body` | No | Optional longer description. Plain text. Included in webhook payloads and Telegram messages. |
| `topic` | No | Routes the notification to all destinations subscribed to this topic. Defaults to `default` if omitted. |
| `priority` | No | Integer: `1` (min), `3` (default), or `5` (urgent). Defaults to `3`. |

## Priority levels

| Value | Label | Use for |
|---|---|---|
| `1` | Min | Informational — routine completions, low-signal events. Won't wake anyone up. |
| `3` | Default | Standard notifications — backup results, deploy hooks, scheduled reports. |
| `5` | Urgent | High-signal events that need immediate attention. Destinations may present these differently. |

## Examples

### curl

```sh
curl -s -X POST https://api.lanby.dev/v1/notifications \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer lnby_live_..." \
  -d '{
    "title":    "Backup completed",
    "body":     "Full snapshot: 142 GB in 4m 12s.",
    "topic":    "homelab",
    "priority": 3
  }'
```

### Python

```python
import requests

requests.post(
    "https://api.lanby.dev/v1/notifications",
    headers={
        "Authorization": "Bearer lnby_live_...",
        "Content-Type": "application/json",
    },
    json={
        "title":    "Backup completed",
        "body":     "Full snapshot: 142 GB in 4m 12s.",
        "topic":    "homelab",
        "priority": 3,
    },
    timeout=5,
)
```

### Shell (end of a cron job)

```sh
#!/bin/bash
set -e

# ... your job ...
run_backup

# Notify on success
curl -s -X POST https://api.lanby.dev/v1/notifications \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer ${LANBY_API_KEY}" \
  -d "{\"title\": \"Backup done\", \"topic\": \"cron-jobs\", \"priority\": 1}" \
  --max-time 5 || true
```

!!! tip
    Use `|| true` so a Lanby outage never causes your script to fail. The notification is best-effort; your job's outcome is what matters.

!!! info
    Native client libraries are planned for common languages. For now, the API is a simple HTTP POST — any HTTP client works.
