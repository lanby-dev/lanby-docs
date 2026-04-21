# Destinations

A destination is where Lanby delivers alerts and notifications — a webhook endpoint, a Telegram chat, or any other channel you configure. Destinations are shared across monitors: configure one, use it everywhere.

## How destinations work

Monitors and the [Publish API](notifications.md) both produce events. Those events are routed to destinations via **topics**. Every destination subscribes to one or more topics; every monitor and notification is published to a topic. Lanby fans each event out to all destinations subscribed to that topic.

Configure destinations once in the console under **Destinations**, then reference them by topic in each monitor's alert settings.

## Available destinations

### Webhook `live`

Posts a JSON payload to any HTTPS URL you control. Works with anything that can receive an HTTP POST — Home Assistant automations, n8n, Make, Zapier, your own backend, or a serverless function.

### Telegram `live`

Delivers messages to a Telegram chat — personal, group, or channel. Lanby links to your chat via the console's OAuth flow; no bot token setup required. Supports configurable message templates with Markdown formatting.

---

## Webhooks

Create a webhook destination by providing a URL. Lanby sends a `POST` request with a JSON body whenever an event is routed to that destination.

### Payload

```json
{
  "id":        "evt_01j...",
  "type":      "monitor.down",
  "topic":     "homelab",
  "monitor": {
    "id":      "mon_01j...",
    "name":    "NAS HTTP check",
    "url":     "http://192.168.1.10:5000"
  },
  "message":   "Connection refused",
  "priority":  3,
  "fired_at":  "2025-11-14T03:21:05Z"
}
```

Event types: `monitor.down`, `monitor.degraded`, `monitor.recovered`, `notification`.

### Custom headers

You can add arbitrary HTTP headers to webhook requests — useful for passing a shared secret to your endpoint for verification.

!!! tip
    Lanby retries failed webhook deliveries with exponential backoff. After all retries are exhausted, the event is moved to the [dead-letter queue](#dead-letter-queue).

---

## Telegram

Link your Telegram account (or a group/channel) from the Destinations page in the console. Click **Link Telegram**, open the bot link, and send the provided code. The destination is active immediately.

### Message templates

Each Telegram destination has a configurable message template with access to event variables: `{{monitor_name}}`, `{{status}}`, `{{message}}`, `{{url}}`, `{{fired_at}}`. Lanby supports both plain text and MarkdownV2 formatting.

### Parse mode

Set **parse mode** to `MarkdownV2` to use Telegram's rich formatting. Use `None` if your template contains characters that conflict with Telegram's Markdown escaping.

---

## Topic routing

Topics decouple monitors from destinations. Instead of wiring each monitor directly to each destination, you assign monitors a topic (e.g. `homelab`, `critical`, `cron-jobs`) and configure destinations to receive events on those topics.

This means you can add or remove destinations without touching monitor configuration. One Telegram destination subscribed to `critical` gets all critical alerts; swap it for a webhook without reconfiguring any monitors.

!!! info
    A destination can subscribe to multiple topics. An event published to a topic is delivered to every destination subscribed to that topic.

---

## Dead-letter queue

When a destination is unreachable and all retries are exhausted, Lanby moves the event to the **dead-letter queue**. You can review and replay DLQ events from the console — useful for recovering alerts that fired while a webhook endpoint was briefly down.

DLQ events are retained for 7 days, then dropped.

---

## Planned destinations

More delivery channels are on the roadmap:

- **Slack, Discord, ntfy, Gotify** — direct integrations with the most common self-hosted and SaaS notification channels
- **Email** — deliver alerts to an email address or distribution list
- **PagerDuty / Opsgenie** — for teams already using an on-call platform
