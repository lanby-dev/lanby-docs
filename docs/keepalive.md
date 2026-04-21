# Keepalive heartbeats

Some things can't be probed from the outside — cron jobs, backup scripts, data pipelines, batch processors. Keepalive monitors flip the relationship: your job checks in with Lanby after each successful run, and Lanby alerts you if it stops hearing back.

## How it works

Each keepalive monitor gets a unique endpoint URL. After a successful run, your job sends a `POST` to that URL. Lanby records the check-in timestamp. If the next check-in doesn't arrive within the configured interval plus a grace period, the monitor goes down and an alert fires.

There's no agent to install and no port to open. Anything that can make an HTTP request — a shell script, a Python job, a Docker container — can send a heartbeat.

!!! info
    Keepalive monitors are for detecting *missed* runs. If your backup job completes but produces corrupt output, that's a separate concern — keepalives won't catch it. Use them alongside, not instead of, output validation in your scripts.

## Setup

1. In the console, go to **Monitors** and create a new monitor.
2. Choose **Keepalive heartbeat** as the monitor type.
3. Set the expected interval and grace period (see [Configuration](#configuration)).
4. Save. The console shows the unique heartbeat URL for this monitor.
5. Add the heartbeat call to the end of your job (see [examples below](#sending-a-heartbeat)).

The monitor starts in a **pending** state until the first heartbeat arrives. No alert fires during the initial pending window.

## Sending a heartbeat

Send a `POST` request to the monitor's endpoint URL with your API key as a Bearer token. The request body is ignored — only the arrival is recorded.

### curl

```sh
curl -s -X POST https://api.lanby.dev/beat/<monitor-id> \
  -H "Authorization: Bearer <your-api-key>"
```

### Shell / cron

The `&&` ensures the heartbeat only fires on success:

```sh
# crontab entry
0 3 * * * /opt/scripts/backup.sh && \
  curl -s -X POST https://api.lanby.dev/beat/<monitor-id> \
    -H "Authorization: Bearer <your-api-key>" > /dev/null
```

### Python

```python
import requests

def run_job():
    # ... your job logic ...
    pass

run_job()

# Notify Lanby on success
requests.post(
    "https://api.lanby.dev/beat/<monitor-id>",
    headers={"Authorization": "Bearer <your-api-key>"},
    timeout=5,
)
```

### systemd service

For systemd timers, add an `ExecStartPost` — it only runs if `ExecStart` exits zero:

```ini
[Service]
ExecStart=/opt/scripts/backup.sh
ExecStartPost=curl -s -X POST https://api.lanby.dev/beat/<monitor-id> \
  -H "Authorization: Bearer <your-api-key>"
```

!!! tip
    Wrap the heartbeat call in a timeout (`curl --max-time 5`) so a Lanby outage never blocks your job from completing.

## Configuration

| Field | Description |
|---|---|
| Interval | How often you expect a heartbeat. Set this to the period of your scheduled job — e.g. `1h` for an hourly job, `24h` for a daily one. |
| Grace period | Extra time allowed after the interval before the monitor goes down. A 5-minute grace on a daily job means it can arrive up to 24h 5m after the last heartbeat before alerting. |

The monitor goes down if no heartbeat arrives within **interval + grace period**. It recovers as soon as the next heartbeat is received.
