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

By default, only the arrival time matters. A `200 OK` response confirms receipt.

The request body is optional. When provided, it can carry a self-reported status — see [Reporting job status](#reporting-job-status).

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

## Reporting job status

The heartbeat body is optional. Send a small JSON payload to tell Lanby the job ran but didn't fully succeed — useful for jobs that should page you even when they *completed*, just not cleanly.

```sh
curl -s -X POST https://in.lanby.dev/beat/<monitor-id> \
  -H "Authorization: Bearer ${LANBY_API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{"status": "degraded", "message": "2 of 47 files skipped"}' \
  --max-time 5
```

### Accepted values

| `status` | Resulting state |
|---|---|
| *(absent)* or `"ok"` or `0` | `up` |
| `"degraded"` or `1` | `degraded` |
| `"failed"`, `"fail"`, `"error"`, `"down"`, or `2` | `down` |

The optional `message` field is surfaced in the console and passed through to destination templates as `{{message}}`.

### When to use it

- **Partial success.** A backup that completed but skipped files. A sync that ran but missed some records.
- **Soft failure.** A build that succeeded but a downstream step timed out. Your job still considers itself "done" but wants a warning.
- **Self-reported errors.** The job caught its own exception, ran the cleanup, and wants to page you without failing the process.

If your script exits non-zero, you generally don't want to send a heartbeat at all — Lanby will detect the missed window and alert. Status payloads are for the case where the process completed but the outcome wasn't clean.

---

## Maintenance windows

A maintenance window suppresses alerts and stops the monitor from timing out during planned outages — nightly reboots, scheduled downtime, OS patching. Configure them per-monitor under **Maintenance** on the monitor's settings page.

Two types are supported:

| Type | Example |
|---|---|
| **Recurring** | `0 2 * * 0` with duration `2h` — every Sunday at 02:00 local for two hours |
| **One-off** | From `2026-05-12 22:00` to `2026-05-13 02:00` |

Both types accept an optional reason (e.g. "Weekly reboot") that's shown in the UI and included in timeline entries.

During a maintenance window:

- Missed beats do not trigger alerts.
- The monitor does not transition to `down`.
- The next expected beat time is pushed forward so the window ends cleanly.
- Beats that *do* arrive are still recorded, so you retain visibility.

!!! info
    Maintenance windows only apply to keepalive monitors. To suspend a probe monitor, use pause.

---

## Pausing a monitor

For ad-hoc suspension — debugging, decommissioning, a one-off change — use **Pause** instead of a maintenance window. Two forms:

- **Indefinite pause** — monitor stays paused until manually resumed.
- **Pause until** — monitor auto-resumes at a specific time.

While paused, the monitor does not time out, fire alerts, or change state. You can record a reason that's shown in the UI.

Resuming a monitor re-opens the next expected window *from the current time* — previous missed windows are not backfilled.

---

## Configuration

### Schedule

A keepalive monitor has to know *when to expect the next beat*. Three scheduling modes are available:

| Mode | When to use |
|---|---|
| **Interval** | Jobs that run every N seconds/minutes/hours with no fixed wall-clock time. Picks up from the last beat. |
| **Simple** | Jobs that run on a daily, weekly, or monthly cadence at a specific time-of-day (e.g. "3:30 AM daily"). |
| **Cron** | Jobs driven by a standard 5-field cron expression (e.g. `0 */6 * * *`) — matches your crontab exactly. |

**Interval mode:**
```
Schedule: Interval
Every: 1 hour
```

**Simple mode:**
```
Schedule: Simple
Frequency: Daily
Time of day: 03:30
Timezone: America/Los_Angeles
```

**Cron mode:**
```
Schedule: Cron
Expression: 0 3 * * *
Timezone: America/Los_Angeles
```

!!! tip
    Set the timezone on wall-clock schedules so daylight-saving transitions don't cause a false alert. Interval-mode schedules are timezone-independent.

### Grace window

Grace absorbs normal runtime variance so a slightly-late or slightly-early beat doesn't flap the monitor. The window is two-sided:

| Field | Description |
|---|---|
| **Grace before** | How early a beat may arrive before the expected time and still count as on-schedule. |
| **Grace after** | How late a beat may arrive after the expected time before the monitor transitions to `down`. |

The monitor stays `up` as long as each beat arrives within the window. If **grace after** elapses with no beat, the monitor transitions to `down` and an alert fires. Recovery is immediate on the next beat.

#### Choosing grace-after

| Job frequency | Suggested grace-after |
|---|---|
| Every minute | 2–5 minutes |
| Hourly | 10–15 minutes |
| Daily | 30–60 minutes |
| Weekly | 4–12 hours |

Set it high enough to survive occasional slow runs, but low enough that you'd still want to know if the job is stuck.

### Early beats

What happens when a beat arrives *before* the grace-before window opens?

| Behavior | Effect |
|---|---|
| `record` (default) | Beat is saved to history; timer is not reset; state is unchanged. |
| `ignore` | Beat is discarded entirely; no history entry; no state change. |
| `degraded` | Beat is saved; monitor is marked `degraded` (a job that's running too often may indicate a stuck loop). |

The `degraded` option is useful when you explicitly want to catch unintended double-runs — e.g. a cron job that's been manually triggered on top of its normal schedule.

### Anchor time

By default, a new keepalive monitor stays in `pending` until the first beat arrives. No alerts fire during this initial window. If you'd rather have the first window open at a specific wall-clock time — e.g. you know the next cron run is at 03:00 tomorrow — set an **anchor time**. The monitor will time out at `anchor + interval + grace-after` if no beat arrives before then.

### Re-alert on sustained outage

When a keepalive goes `down`, a single alert fires on the transition. If the job stays broken for days, that one alert is easy to miss. Set **Re-alert after missed** to fire a repeat alert every N consecutive missed windows:

```
Re-alert after: 3 missed windows
```

With a daily schedule and re-alert-after = 3, you get an alert on day 1 (when it first went down), then again on day 4, day 7, and so on — until the job recovers.
