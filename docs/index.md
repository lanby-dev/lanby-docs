# Lanby Docs

A **LANBY** — *Large Automatic Navigation BuoY* — is a floating navigational aid designed to replace crewed lightships. It sits offshore, watches over shipping lanes, and runs without intervention.

That's what Lanby the product is: infrastructure that watches your services quietly and reliably, from the outside, so you don't have to.

---

Lanby watches your services from the outside and pings you when something's wrong. It runs on infrastructure that's completely separate from what you're monitoring, so an outage can't take the watchman down with it.

These docs cover the two jobs Lanby does:

1. **Uptime monitoring** — probes that actively check services (HTTP, TCP, DNS, ping, gRPC) on a schedule, and keepalive heartbeats where your jobs check in with Lanby after each run.
2. **Notification delivery** — a simple publish API and a set of destinations (webhooks, Telegram, more on the way) that route events to the people and systems that should know.

---

## How the pieces fit

Three patterns cover most setups. Color key used across all diagrams:
**Blue** = Lanby API &nbsp;·&nbsp; **Teal** = Relay &nbsp;·&nbsp; **Orange** = your service or job &nbsp;·&nbsp; **Purple** = alert destination

---

### Managed probe monitor

Lanby probes a publicly reachable service on a schedule. No agent required.

```mermaid
%%{init:{'theme':'base','themeVariables':{'primaryColor':'#3a52a8','primaryBorderColor':'#7090ee','primaryTextColor':'#fff','edgeLabelBackground':'#1a1a2e','lineColor':'#7090ee'}}}%%
flowchart LR
    classDef lanby  fill:#3a52a8,stroke:#7090ee,stroke-width:2px,color:#fff
    classDef svc    fill:#a05a20,stroke:#e08840,stroke-width:2px,color:#fff
    classDef dest   fill:#5c3480,stroke:#9d5cce,stroke-width:2px,color:#fff

    L["Lanby API\n(probe engine)"]:::lanby
    S["Your service\nhttps://myapp.com"]:::svc
    D["Destination\nTelegram · Webhook"]:::dest

    L == "HTTP · TCP · DNS\nevery N seconds" ==> S
    S -. "pass / fail + latency" .-> L
    L -- "alert on\nfailure or recovery" --> D

    linkStyle 0 stroke:#7090ee,stroke-width:3px,color:#fff
    linkStyle 1 stroke:#7090ee,stroke-width:2px,stroke-dasharray:6
    linkStyle 2 stroke:#9d5cce,stroke-width:2px
```

---

### Keepalive monitor

Your job calls Lanby after each successful run. Lanby alerts if check-ins stop arriving.

```mermaid
%%{init:{'theme':'base','themeVariables':{'primaryColor':'#3a52a8','primaryBorderColor':'#7090ee','primaryTextColor':'#fff','edgeLabelBackground':'#1a1a2e','lineColor':'#7090ee'}}}%%
flowchart LR
    classDef lanby  fill:#3a52a8,stroke:#7090ee,stroke-width:2px,color:#fff
    classDef job    fill:#a05a20,stroke:#e08840,stroke-width:2px,color:#fff
    classDef dest   fill:#5c3480,stroke:#9d5cce,stroke-width:2px,color:#fff

    J["Your cron job\nor script"]:::job
    L["Lanby API\nexpects a beat every N\nsilence = alert fires"]:::lanby
    D["Destination\nTelegram · Webhook"]:::dest

    J == "POST /beat/id\nafter each successful run" ==> L
    L -- "alert" --> D

    linkStyle 0 stroke:#e08840,stroke-width:3px
    linkStyle 1 stroke:#9d5cce,stroke-width:2px
```

---

### Relay probe monitor

A Relay runs inside your private network. It polls Lanby for probe assignments, runs the checks locally, and ships results back — no inbound firewall ports required.

```mermaid
%%{init:{'theme':'base','themeVariables':{'primaryColor':'#3a52a8','primaryBorderColor':'#7090ee','primaryTextColor':'#fff','edgeLabelBackground':'#1a1a2e','lineColor':'#7090ee'}}}%%
flowchart LR
    classDef lanby  fill:#3a52a8,stroke:#7090ee,stroke-width:2px,color:#fff
    classDef relay  fill:#1e6e5e,stroke:#40c0a8,stroke-width:2px,color:#fff
    classDef svc    fill:#a05a20,stroke:#e08840,stroke-width:2px,color:#fff
    classDef dest   fill:#5c3480,stroke:#9d5cce,stroke-width:2px,color:#fff

    subgraph cloud["Lanby cloud"]
        L["Lanby API"]:::lanby
        D["Destination\nTelegram · Webhook"]:::dest
    end
    subgraph net["Your private network"]
        R["Relay"]:::relay
        S["Internal service\n192.168.x.x · myhost.local"]:::svc
    end

    L -- "probe config\n(relay polls)" --> R
    R == "HTTP · TCP · ICMP\nprobe locally" ==> S
    R -- "results via\nPOST /sync" --> L
    L -- "alert on\nfailure or recovery" --> D

    linkStyle 0 stroke:#40c0a8,stroke-width:2px,stroke-dasharray:4
    linkStyle 1 stroke:#e08840,stroke-width:3px
    linkStyle 2 stroke:#40c0a8,stroke-width:2px
    linkStyle 3 stroke:#9d5cce,stroke-width:2px
```

---

## Start here

New to Lanby? These pages in order cover 80% of what you'll actually configure:

1. [**Monitor types**](monitors.md) — pick the right probe or keepalive for what you're watching.
2. [**Destinations**](destinations.md) — set up where alerts go.
3. [**Relay**](relays.md) — deploy one if you want to monitor private network services.
4. [**Keepalive heartbeats**](keepalive.md) — wire up cron jobs, backups, and scheduled tasks.
5. [**Publish API**](notifications.md) — send arbitrary events from any script to your existing destinations.

---

## All documentation

### Concepts

[**Relay**](relays.md)
: How Relay works, the security model, deployment, and claiming — for monitoring private network services without opening inbound firewall ports.

### Monitoring

[**Monitor types**](monitors.md)
: Every monitor type — active probes (HTTP, TCP, ping, DNS, gRPC) and keepalive heartbeats — including what's live and what's on the roadmap.

[**Keepalive heartbeats**](keepalive.md)
: Monitor cron jobs, backup scripts, and scheduled tasks by having them check in with Lanby after each successful run.

### Alerts & Delivery

[**Destinations**](destinations.md)
: Configure where Lanby sends alerts — webhooks, Telegram, topic routing, and the dead-letter queue.

[**Publish API**](notifications.md)
: Publish events to Lanby from any script, app, or cron job with a single HTTP call.

