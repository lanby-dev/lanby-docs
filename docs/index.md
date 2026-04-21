# Lanby Docs

Guides and reference material for monitoring, relays, alerts, and the notification API.

---

## Concepts

[**Relay agents**](relays.md)
: How relays work, the security model, deployment, and claiming — for monitoring private network services without opening inbound firewall ports.

---

## Monitoring

[**Monitor types**](monitors.md)
: Every monitor type — active probes (HTTP, TCP, ping, DNS, gRPC) and keepalive heartbeats — including what's live and what's on the roadmap.

[**Keepalive heartbeats**](keepalive.md)
: Monitor cron jobs, backup scripts, and scheduled tasks by having them check in with Lanby after each successful run.

---

## Alerts & Delivery

[**Destinations**](destinations.md)
: Configure where Lanby sends alerts — webhooks, Telegram, topic routing, and the dead-letter queue.

[**Publish API**](notifications.md)
: Publish events to Lanby from any script, app, or cron job with a single HTTP call.
