# Publish API

Publish an event to Lanby from any script, cron job, application, or service with a single HTTP call. Lanby routes it to your configured destinations — the same webhook and Telegram destinations your monitor alerts use.

This is useful for surfacing non-monitoring events: backup completions, deploy results, scheduled report outputs, or anything you want delivered to your existing alert channels without setting up a separate notification pipeline.

## Authentication

All requests require an API key as a Bearer token. Create and manage API keys in the console under **Settings → API Keys**.

```http
Authorization: Bearer lnby_live_...
```

Keep your API key in an environment variable, not hardcoded. If one is compromised, revoke it from the console and generate a new one.

## Endpoint

```
POST https://in.lanby.dev/v1/notifications
Content-Type: application/json
Authorization: Bearer lnby_live_...
```

## Request fields

| Field | Type | Required | Description |
|---|---|---|---|
| `title` | string | Yes | Short summary. Shown as the message heading. Max 255 characters. |
| `body` | string | No | Longer description. Plain text. Included in webhook payloads and Telegram messages. Max 4096 characters. |
| `topic` | string | No | Routes to all destinations subscribed to this topic. Defaults to `default`. |
| `priority` | integer | No | `1` (min), `3` (default), or `5` (urgent). Defaults to `3`. |

## Response

**Success — `202 Accepted`:**
```json
{ "id": "ntf_01j..." }
```

**Error — `400 Bad Request`** (missing or invalid fields):
```json
{ "error": "title is required" }
```

**Error — `401 Unauthorized`** (missing or invalid API key):
```json
{ "error": "unauthorized" }
```

**Error — `429 Too Many Requests`** (rate limited):
```json
{ "error": "rate limit exceeded" }
```

## Priority levels

| Value | Label | Typical use |
|---|---|---|
| `1` | Min | Informational — routine completions, low-signal events |
| `3` | Default | Standard alerts — backup results, deploy hooks, cron reports |
| `5` | Urgent | High-signal events needing immediate attention |

Destinations may present priority differently — e.g. Telegram can be configured to use loud notifications for priority 5.

## Examples

### curl

```sh
curl -s -X POST https://in.lanby.dev/v1/notifications \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer ${LANBY_API_KEY}" \
  -d '{
    "title":    "Backup completed",
    "body":     "Full snapshot: 142 GB in 4m 12s.",
    "topic":    "homelab",
    "priority": 1
  }'
```

### Shell script (end of a cron job)

```sh
#!/bin/bash
set -e

run_backup

# Report result — || true prevents Lanby outage from failing the job
curl -s -X POST https://in.lanby.dev/v1/notifications \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer ${LANBY_API_KEY}" \
  -d "{\"title\": \"Backup done\", \"topic\": \"cron-jobs\", \"priority\": 1}" \
  --max-time 5 || true
```

### Python

```python
import os
import requests

def notify(title: str, body: str = "", topic: str = "default", priority: int = 3) -> None:
    """Send a notification to Lanby. Best-effort — never raises."""
    try:
        requests.post(
            "https://in.lanby.dev/v1/notifications",
            headers={
                "Authorization": f"Bearer {os.environ['LANBY_API_KEY']}",
                "Content-Type": "application/json",
            },
            json={"title": title, "body": body, "topic": topic, "priority": priority},
            timeout=5,
        )
    except Exception:
        pass

# Usage
notify("Deploy complete", "v1.4.2 deployed to production", topic="ops")
notify("Disk usage high", f"/ is at 91%", topic="critical", priority=5)
```

### Go

```go
package lanby

import (
    "bytes"
    "encoding/json"
    "net/http"
    "os"
    "time"
)

type Event struct {
    Title    string `json:"title"`
    Body     string `json:"body,omitempty"`
    Topic    string `json:"topic,omitempty"`
    Priority int    `json:"priority,omitempty"`
}

var client = &http.Client{Timeout: 5 * time.Second}

// Notify sends an event to Lanby. It is best-effort and never returns an error.
func Notify(title, body, topic string, priority int) {
    b, _ := json.Marshal(Event{Title: title, Body: body, Topic: topic, Priority: priority})
    req, _ := http.NewRequest(http.MethodPost, "https://in.lanby.dev/v1/notifications", bytes.NewReader(b))
    req.Header.Set("Content-Type", "application/json")
    req.Header.Set("Authorization", "Bearer "+os.Getenv("LANBY_API_KEY"))
    resp, err := client.Do(req)
    if err == nil {
        resp.Body.Close()
    }
}

// Usage:
// lanby.Notify("Backup complete", "142 GB in 4m 12s", "homelab", 1)
// lanby.Notify("Disk usage critical", "/var is at 95%", "critical", 5)
```

### Node.js

```js
const API_KEY = process.env.LANBY_API_KEY;
const BASE = 'https://in.lanby.dev';

async function notify(title, { body = '', topic = 'default', priority = 3 } = {}) {
  try {
    await fetch(`${BASE}/v1/notifications`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        Authorization: `Bearer ${API_KEY}`,
      },
      body: JSON.stringify({ title, body, topic, priority }),
      signal: AbortSignal.timeout(5000),
    });
  } catch (_) {
    // best-effort
  }
}

// Usage
await notify('Deploy complete', { body: 'v1.4.2 live', topic: 'ops' });
await notify('Disk critical', { body: '/ at 91%', topic: 'critical', priority: 5 });
```

### Ansible playbook

```yaml
- name: Notify Lanby — deploy complete
  uri:
    url: "https://in.lanby.dev/v1/notifications"
    method: POST
    headers:
      Authorization: "Bearer {{ lookup('env', 'LANBY_API_KEY') }}"
      Content-Type: "application/json"
    body_format: json
    body:
      title: "Deploy complete"
      body: "{{ inventory_hostname }}: {{ app_version }} deployed"
      topic: "ops"
      priority: 3
    timeout: 5
    status_code: 202
  ignore_errors: true
```

### Sonarr / Radarr custom script

Add to your *arr app's custom script settings (On Import / On Download):

```sh
#!/bin/bash
TITLE="${sonarr_series_title:-${radarr_movie_title}} downloaded"
BODY="${sonarr_episodefile_quality:-${radarr_moviefile_quality}}"

curl -s -X POST https://in.lanby.dev/v1/notifications \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer ${LANBY_API_KEY}" \
  -d "{\"title\": \"${TITLE}\", \"body\": \"${BODY}\", \"topic\": \"media\", \"priority\": 1}" \
  --max-time 5 || true
```

## ntfy-compatible ingest

Lanby's ingest endpoint is compatible with the [ntfy](https://ntfy.sh) publish API. If you already have scripts or apps publishing to ntfy, you can point them at Lanby's ingest with minimal changes:

```sh
# ntfy-style publish — title in header, body as request body
curl -X POST https://in.lanby.dev/<topic> \
  -H "Authorization: Bearer ${LANBY_API_KEY}" \
  -H "Title: Backup complete" \
  -d "Full snapshot: 142 GB"
```

This is useful for self-hosted apps that have built-in ntfy support (Vaultwarden, Gotify bridges, etc.) — point them at `in.lanby.dev` with your API key and they route through your existing Lanby destinations.

!!! info
    Native client libraries are on the roadmap. For now the API is a simple HTTP POST — any HTTP client works.
