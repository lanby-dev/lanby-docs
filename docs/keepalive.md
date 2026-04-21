# Keepalive heartbeats

Some things can't be probed from the outside — cron jobs, backup scripts, data pipelines, batch processors. Keepalive monitors flip the relationship: your job checks in with Lanby after each successful run, and Lanby alerts you if it stops hearing back.

## How it works

Each keepalive monitor gets a unique endpoint URL. After a successful run, your job sends a `POST` to that URL. Lanby records the timestamp. If the next check-in doesn't arrive within the configured interval plus a grace period, the monitor goes down and an alert fires.

There's no agent to install and no port to open. Anything that can make an HTTP request can send a heartbeat.

!!! info
    Keepalive monitors detect *missed* runs, not bad output. If your backup job completes but produces corrupt data, that's a separate concern. Use them alongside output validation in your scripts, not instead of it.

## Setup

1. In the console, go to **Monitors** and create a new monitor.
2. Choose **Keepalive heartbeat** as the monitor type.
3. Set the expected interval and grace period.
4. Save. The console shows the unique heartbeat URL and your API key.
5. Add the heartbeat call to the end of your job.

The monitor starts in a **pending** state until the first heartbeat arrives. No alerts fire during this initial window.

## The endpoint

```
POST https://in.lanby.dev/beat/<monitor-id>
Authorization: Bearer <your-api-key>
```

The request body is ignored. Only the arrival time matters. A `200 OK` response confirms receipt.

!!! tip
    Store your API key in an environment variable (`LANBY_API_KEY`) rather than hardcoding it in scripts. The examples below use this pattern.

## Sending a heartbeat

### curl

```sh
curl -s -X POST https://in.lanby.dev/beat/<monitor-id> \
  -H "Authorization: Bearer ${LANBY_API_KEY}" \
  --max-time 5
```

### wget

Useful on minimal systems where curl isn't available:

```sh
wget -q -O /dev/null --post-data="" \
  --header="Authorization: Bearer ${LANBY_API_KEY}" \
  --timeout=5 \
  https://in.lanby.dev/beat/<monitor-id>
```

### Shell / cron — success only

The `&&` ensures the heartbeat only fires if the job exits successfully:

```sh
# /etc/cron.d/backup
0 3 * * * root /opt/scripts/backup.sh && \
  curl -s -X POST https://in.lanby.dev/beat/<monitor-id> \
    -H "Authorization: Bearer ${LANBY_API_KEY}" \
    --max-time 5 || true
```

The `|| true` at the end prevents a Lanby outage from making cron report the job as failed.

### systemd timer

For systemd timers, use `ExecStartPost` — it only runs if `ExecStart` exits zero:

```ini
# /etc/systemd/system/backup.service
[Unit]
Description=Nightly backup

[Service]
Type=oneshot
EnvironmentFile=/etc/lanby.env
ExecStart=/opt/scripts/backup.sh
ExecStartPost=curl -s -X POST https://in.lanby.dev/beat/<monitor-id> \
  -H "Authorization: Bearer ${LANBY_API_KEY}" \
  --max-time 5
```

```sh
# /etc/lanby.env
LANBY_API_KEY=lnby_live_...
```

### Python

```python
import os
import requests

API_KEY = os.environ["LANBY_API_KEY"]
MONITOR_ID = "<monitor-id>"

def run_backup():
    # ... your job logic ...
    pass

run_backup()

# Notify Lanby — don't let a Lanby outage fail the job
try:
    requests.post(
        f"https://in.lanby.dev/beat/{MONITOR_ID}",
        headers={"Authorization": f"Bearer {API_KEY}"},
        timeout=5,
    )
except Exception:
    pass
```

### Go

```go
package main

import (
    "fmt"
    "net/http"
    "os"
    "time"
)

func ping() {
    monitorID := "<monitor-id>"
    apiKey := os.Getenv("LANBY_API_KEY")

    client := &http.Client{Timeout: 5 * time.Second}
    req, _ := http.NewRequest(http.MethodPost,
        fmt.Sprintf("https://in.lanby.dev/beat/%s", monitorID), nil)
    req.Header.Set("Authorization", "Bearer "+apiKey)
    resp, err := client.Do(req)
    if err == nil {
        resp.Body.Close()
    }
}

func main() {
    runBackup()
    ping() // best-effort, don't check error
}
```

### Node.js

```js
const MONITOR_ID = '<monitor-id>';
const API_KEY = process.env.LANBY_API_KEY;

async function ping() {
  try {
    await fetch(`https://in.lanby.dev/beat/${MONITOR_ID}`, {
      method: 'POST',
      headers: { Authorization: `Bearer ${API_KEY}` },
      signal: AbortSignal.timeout(5000),
    });
  } catch (_) {
    // best-effort
  }
}

await runBackup();
await ping();
```

### Docker healthcheck

Use keepalives to verify a long-running container is functioning, not just running. Add the ping to your container's internal health logic:

```dockerfile
HEALTHCHECK --interval=5m --timeout=10s --start-period=30s \
  CMD curl -sf -X POST https://in.lanby.dev/beat/<monitor-id> \
    -H "Authorization: Bearer ${LANBY_API_KEY}" || exit 1
```

Or ping from a sidecar script in the container at the end of each work cycle.

### Ansible

At the end of a playbook that should complete on schedule:

```yaml
- name: Notify Lanby on success
  uri:
    url: "https://in.lanby.dev/beat/{{ monitor_id }}"
    method: POST
    headers:
      Authorization: "Bearer {{ lanby_api_key }}"
    timeout: 5
    status_code: 200
  ignore_errors: true  # don't fail the playbook if Lanby is unreachable
```

### Sonarr / Radarr / *arr apps

The *arr apps support custom scripts on events. Create a script that pings Lanby after a successful import or health check:

```sh
#!/bin/bash
# Called by Sonarr on "On Import" event
curl -s -X POST https://in.lanby.dev/beat/<monitor-id> \
  -H "Authorization: Bearer ${LANBY_API_KEY}" \
  --max-time 5 || true
```

Set the interval in Lanby to match how often you expect new content to import (e.g. `24h` with a 2-hour grace period).

---

## Configuration

| Field | Description |
|---|---|
| **Interval** | How often you expect a heartbeat. Match this to the period of your scheduled job — `1h` for hourly, `24h` for daily. |
| **Grace period** | Extra time allowed after the interval. A 10-minute grace on a daily job allows up to 24h 10m between heartbeats before alerting. Set this to cover expected runtime variance. |

The monitor goes down if no heartbeat arrives within **interval + grace period**. It recovers immediately on the next heartbeat.

### Choosing grace period

| Job frequency | Suggested grace |
|---|---|
| Every minute | 2–5 minutes |
| Hourly | 10–15 minutes |
| Daily | 30–60 minutes |
| Weekly | 4–12 hours |

Set it high enough to survive occasional slow runs, but low enough that you'd still want to know if the job is stuck.
