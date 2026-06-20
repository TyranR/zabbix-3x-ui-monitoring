# 3X-UI Zabbix Template Dashboard

Dashboard documentation for **3X-UI Panel by Zabbix agent 2** v1.11.2.

The template includes one host-level dashboard:

```text
3X-UI Host Overview
```

Dashboard stats:

```text
Pages: 5
Widgets: 62
```

---

## Dashboard pages

### Overview

Purpose: quick operational status.

Typical widgets:

```text
API availability
Xray state
Panel process
Xray process
Online clients
Inbounds enabled
Nodes online
TLS expires
System utilization
Network traffic rates
Connections
Web UI response time and speed
Upload/download traffic totals
```

Use this page to answer: is the host basically healthy?

---

### Traffic

Purpose: traffic analysis without mixing rates and counters.

Main graphs:

```text
3X-UI: Upload traffic totals
3X-UI: Download traffic totals
3X-UI: Total traffic totals
3X-UI: Upload and download traffic totals
3X-UI: Inbounds traffic aggregate
3X-UI: Clients traffic aggregate
3X-UI: Xray outbounds traffic
3X-UI: Network traffic rates
```

`Network traffic rates` is a rate graph (`Bps`). The other traffic graphs are cumulative counters (`B`).

---

### Inbounds & Clients

Purpose: operational view of 3X-UI users and inbound configuration.

Typical widgets:

```text
Inbounds total
Inbounds enabled
Inbounds disabled
Inbound clients total
Clients total
Clients enabled
Clients disabled
Online clients
Inbounds aggregate
Clients aggregate
Clients quota and expiry summary
Clients traffic aggregate
Inbounds traffic aggregate
```

Per-client and per-inbound details are better viewed through generated graph prototypes and Latest data.

---

### Nodes & Xray

Purpose: central-panel and Xray diagnostics.

Typical widgets:

```text
Xray state
Xray version / available versions
Xray metrics state
Panel update status
Nodes total
Nodes online
Nodes unhealthy
Stale heartbeat
Nodes aggregate
Xray outbounds traffic
Observatory delay
Connections
Load averages
Disk I/O rates
```

This page is most useful on central/admin panels with:

```text
{$3XUI.NODE.LLD.ENABLED}=1
```

On standalone hosts, node widgets may be zero or empty.

---

### Web, TLS, Logs & Backup

Purpose: external access, TLS, local database and optional operational hygiene checks.

Typical widgets:

```text
Web failed step
TLS expires, days
DB size
DB mtime age, min
Backup count
Backup age
Panel log errors
Xray log errors
Web UI Response Time and Speed
TLS and backup freshness
Database health
Log patterns
Application runtime
System utilization
```

Some widgets show `No data` until optional checks are enabled.

---

## Optional checks and dashboard `No data`

The dashboard intentionally includes optional widgets.

These may show `No data` until configured:

| Area | Required action |
|---|---|
| Web UI graph | Set `{$3XUI.WEB.URL}`, enable the Web scenario, optionally set `{$3XUI.WEB.CHECK}=1`. |
| TLS widgets | Set `{$3XUI.TLS.HOST}`, `{$3XUI.TLS.PORT}`, enable TLS items, optionally set `{$3XUI.TLS.CHECK}=1`. |
| Backup widgets | Set `{$3XUI.BACKUP.PATH}`, enable backup items, optionally set `{$3XUI.BACKUP.CHECK}=1`. |
| Logs | Logs are collected by API items, but log triggers are disabled by default with `{$3XUI.LOGS.CHECK}=0`. |
| Nodes | Enable Node LLD only on central/admin panels with `{$3XUI.NODE.LLD.ENABLED}=1`. |
| Clients | Enable Client LLD only when you want per-client data with `{$3XUI.CLIENT.LLD.ENABLED}=1`. |

---

## Why host item/graph counts differ

Hosts linked to the same template can have different item, trigger and graph counts.

The reason is LLD:

```text
static template objects
+ discovered inbounds
+ discovered clients
+ discovered nodes
+ discovered outbounds
+ discovered ports
= host-specific object count
```

A central panel with nodes usually has more discovered objects than a standalone server.

---

## Recommended use

Use the dashboard for aggregate status and operational overview.

Use generated graphs and Latest data for detailed investigation:

```text
Monitoring → Hosts → <host> → Graphs
Monitoring → Latest data
Data collection → Hosts → <host> → Discovery
```
