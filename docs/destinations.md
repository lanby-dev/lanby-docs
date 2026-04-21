# Destinations

A destination is where Lanby delivers alerts and notifications — a webhook endpoint, a Telegram chat, or any other channel you configure. Destinations are shared across monitors: configure one, use it everywhere.

## How destinations work

Monitors and the [Publish API](notifications.md) both produce events. Events are routed to destinations via **topics**. Every destination subscribes to one or more topics; every monitor and notification publishes to a topic. Lanby fans each event out to all matching destinations.

Configure destinations once in the console under **Destinations**, then reference them by topic in each monitor's alert settings.

## Available destinations

### Webhook `live`

Posts a JSON payload to any HTTPS URL you control — Home Assistant, n8n, Make, Zapier, your own backend, a serverless function, or anything that can receive an HTTP POST.

### Telegram `live`

Delivers messages to a Telegram chat, group, or channel. Uses Lanby's bot via an OAuth-style linking flow — no bot token setup required on your end.

---

## Webhooks

Create a webhook destination by providing a URL. Lanby sends a `POST` request with a JSON body whenever an event is routed to that destination.

### Delivery and retries

Lanby considers a delivery successful if your endpoint returns any `2xx` status within 10 seconds. On failure, it retries with exponential backoff:

| Attempt | Delay |
|---|---|
| 1 (initial) | Immediate |
| 2 | ~30 seconds |
| 3 | ~2 minutes |
| 4 | ~10 minutes |
| 5 | ~1 hour |

After all retries are exhausted the event moves to the [dead-letter queue](#dead-letter-queue). Your endpoint should return `200` quickly and process asynchronously — don't do heavy work in the request handler.

### Payload — monitor alert

```json
{
  "id":       "evt_01j...",
  "type":     "monitor.down",
  "topic":    "homelab",
  "monitor": {
    "id":     "mon_01j...",
    "name":   "NAS HTTP check",
    "url":    "http://192.168.1.10:5000"
  },
  "message":  "dial tcp 192.168.1.10:5000: connect: connection refused",
  "priority": 3,
  "fired_at": "2025-11-14T03:21:05Z"
}
```

### Payload — monitor recovered

```json
{
  "id":       "evt_01j...",
  "type":     "monitor.recovered",
  "topic":    "homelab",
  "monitor": {
    "id":     "mon_01j...",
    "name":   "NAS HTTP check",
    "url":    "http://192.168.1.10:5000"
  },
  "message":  "Monitor recovered after 4m 32s",
  "priority": 3,
  "fired_at": "2025-11-14T03:25:37Z"
}
```

### Payload — notification (Publish API)

```json
{
  "id":       "evt_01j...",
  "type":     "notification",
  "topic":    "cron-jobs",
  "message":  "Backup completed — 142 GB in 4m 12s",
  "priority": 1,
  "fired_at": "2025-11-14T03:21:05Z"
}
```

### Event types

| Type | Fired when |
|---|---|
| `monitor.down` | Monitor transitions to down state |
| `monitor.degraded` | Monitor transitions to degraded (slow response, expiring cert) |
| `monitor.recovered` | Monitor returns to up from down or degraded |
| `notification` | Event published via the Publish API |

### Custom headers

Add arbitrary HTTP headers to every request from a webhook destination — useful for passing a shared secret to verify the request came from Lanby:

```
X-Webhook-Secret: mysecretvalue
```

Verify it in your handler:

```python
# Flask example
from flask import request, abort

@app.route('/lanby-webhook', methods=['POST'])
def webhook():
    if request.headers.get('X-Webhook-Secret') != 'mysecretvalue':
        abort(401)
    event = request.json
    # process event...
    return '', 200
```

```js
// Express example
app.post('/lanby-webhook', (req, res) => {
  if (req.headers['x-webhook-secret'] !== 'mysecretvalue') {
    return res.status(401).end();
  }
  const event = req.body;
  // process event...
  res.status(200).end();
});
```

### Home Assistant example

Use a webhook automation to trigger HA actions on monitor state changes:

```yaml
# configuration.yaml or automations.yaml
automation:
  - alias: "Lanby — NAS down"
    trigger:
      platform: webhook
      webhook_id: lanby-nas-alert
    condition:
      condition: template
      value_template: "{{ trigger.json.type == 'monitor.down' }}"
    action:
      - service: notify.mobile_app_myphone
        data:
          message: "NAS is down: {{ trigger.json.message }}"
```

Set the webhook destination URL to `https://your-ha-instance/api/webhook/lanby-nas-alert`.

---

## Telegram

Link your Telegram account (or a group/channel) from the Destinations page in the console. Click **Link Telegram**, open the bot link, and send the provided code. The destination is active immediately.

### Message templates

Each Telegram destination has a configurable message template. Available variables:

| Variable | Example value |
|---|---|
| `{{monitor_name}}` | `NAS HTTP check` |
| `{{status}}` | `down` |
| `{{message}}` | `connection refused` |
| `{{url}}` | `http://192.168.1.10:5000` |
| `{{fired_at}}` | `2025-11-14 03:21 UTC` |
| `{{topic}}` | `homelab` |

**Default template:**
```
🔴 {{monitor_name}} is {{status}}
{{message}}
```

**Custom template with MarkdownV2:**
```
*{{monitor_name}}* went `{{status}}`
Target: {{url}}
Time: {{fired_at}}
```

### Parse mode

Set **parse mode** to `MarkdownV2` to use Telegram's rich formatting — bold, code blocks, inline links. Use `None` if your template contains characters that conflict with Telegram's escaping (`.`, `!`, `-`, `(`, `)` must all be escaped in MarkdownV2).

---

## Topic routing

Topics decouple monitors from destinations. Assign monitors a topic (e.g. `homelab`, `critical`, `cron-jobs`) and configure destinations to receive events on those topics.

**Example topology:**

```
Monitors:
  NAS HTTP check      → topic: homelab
  Backup job          → topic: cron-jobs
  Public website      → topic: critical

Destinations:
  Telegram (personal) → subscribes to: homelab, cron-jobs
  Webhook (PagerDuty) → subscribes to: critical
```

With this setup, NAS and backup alerts go to Telegram. Public website outages page PagerDuty. Adding a new homelab monitor doesn't require touching any destination configuration.

!!! info
    A destination can subscribe to multiple topics. An event is delivered to every destination subscribed to the event's topic.

---

## Dead-letter queue

When all delivery retries for a destination are exhausted, the event moves to the **dead-letter queue**. Review and replay DLQ events from the console.

DLQ events are retained for 7 days, then dropped permanently. Replay re-queues the event for delivery using the same retry schedule — it doesn't guarantee delivery if the destination is still unreachable.

---

## Planned destinations

- **ntfy / Gotify** — self-hosted push notification servers; top of the roadmap for self-hosters
- **Slack, Discord** — direct integrations
- **Email** — alerts to an email address or distribution list
- **PagerDuty / Opsgenie** — for teams using on-call platforms
