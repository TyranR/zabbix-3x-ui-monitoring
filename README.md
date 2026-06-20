# Zabbix 3X-UI Monitoring

![Zabbix](https://img.shields.io/badge/Zabbix-7.4-red)
![3X-UI](https://img.shields.io/badge/VPN-3X--UI-orange)
![Agent](https://img.shields.io/badge/agent-Agent%202-blue)
![Platform](https://img.shields.io/badge/platform-Linux-lightgrey)
![License](https://img.shields.io/badge/license-MIT-green)

Reusable Zabbix 7.4 template for monitoring Linux servers running **3X-UI** and **Xray-core**.

The template combines native **Zabbix Agent 2** checks with **HTTP Agent** API polling, Dependent Items, JavaScript/JSONPath preprocessing, optional Low-Level Discovery, and optional Web UI/TLS/log/backup checks.

No external scripts are required.

---

## Current version

Current stable version: **v1.9.6**.

```text
Items: 119
Discovery rules: 6
Item prototypes: 55
Trigger prototypes: 20
Triggers total: 57
Web scenarios: 1
Macros: 60
```

Recent fixes included in v1.9.6:

```text
v1.9.4 — log API endpoints use POST instead of GET
v1.9.5 — fixed newline splitting in log preprocessing
v1.9.6 — removed regular expressions from log preprocessing for Zabbix import/runtime compatibility
```

---

## What this solves

3X-UI installations usually expose several moving parts at once:

- the `x-ui` panel service and process;
- Xray-core process and API-reported Xray state;
- local panel/API ports and public API URLs;
- client-facing inbounds;
- clients, traffic limits and expiry dates;
- remote nodes managed by a central 3X-UI panel;
- Xray outbounds and optional observatory data;
- panel/Xray logs;
- local SQLite database and optional backup files;
- optional Web UI and TLS certificate health.

The template provides a reusable monitoring layer for small and medium 3X-UI deployments, including single-node and central-panel-with-nodes setups.

---

## Repository structure

```text
zabbix-3x-ui-monitoring/
├── README.md
├── templates/
│   └── zabbix_3x_ui_panel_by_agent2.yaml
├── docs/
│   ├── MACROS.md
│   ├── TROUBLESHOOTING.md
│   └── screenshots/
└── LICENSE
```

---

## Requirements

- Zabbix Server 7.4 or compatible 7.x version;
- Zabbix Agent 2 installed on every monitored 3X-UI Linux host;
- 3X-UI Panel API token;
- network access from Zabbix Server/Proxy to `{$3XUI.API.URL}`;
- active Agent 2 checks configured, unless you convert local items to passive mode.

---

## Architecture

The template uses two data collection paths.

### HTTP Agent API polling

```text
Zabbix Server / Zabbix Proxy
    ↓ HTTP Agent item
3X-UI Panel API
```

Used for API, panel, Xray, inbounds, clients, nodes, outbounds, observatory and logs.

### Local Zabbix Agent 2 checks

```text
Zabbix Agent 2 on the 3X-UI host
    ↓ native agent keys
Linux / systemd / processes / local sockets / files
```

Used for systemd, processes, local ports, database, optional backups and optional TLS certificate checks.

---

## Installation

### 1. Import the template

Open **Data collection → Templates → Import** and upload:

```text
templates/zabbix_3x_ui_panel_by_agent2.yaml
```

Recommended import options when upgrading this template:

```text
Create new: checked
Update existing: checked
Delete missing: checked
```

`Delete missing` is recommended because older versions used different discovery prototypes for inbound port checks.

### 2. Link the template to a host

```text
Host name: your-3x-ui-host
Groups: VPN Nodes / Linux servers
Templates: 3X-UI Panel by Zabbix agent 2
Interfaces: Zabbix Agent interface for the 3X-UI Linux host
```

### 3. Configure required macros

At minimum, set these host-level macros:

| Macro | Example | Notes |
|---|---|---|
| `{$3XUI.API.URL}` | `https://vpn.example.com:25551/secret-path/panel/api` | Base API URL visible from Zabbix Server/Proxy. Do not append `/server/status`. |
| `{$3XUI.API.TOKEN}` | `<secret>` | Bearer API token. Store as Secret text. |
| `{$3XUI.PROCESS.XRAY}` | `xray-linux-amd64` | Command-line substring for Xray process detection. |
| `{$3XUI.DB.PATH}` | `/etc/x-ui/x-ui.db` | Local SQLite database file path. |

The full macro reference is in [docs/MACROS.md](docs/MACROS.md).

---

## Important: API URL vs local panel port

`{$3XUI.API.URL}` and `{$3XUI.PANEL.PORT}` are different values.

`{$3XUI.API.URL}` is used by Zabbix Server/Proxy HTTP Agent items:

```text
https://vpn.example.com:25551/secret-path/panel/api
```

`{$3XUI.PANEL.PORT}` is used by local Agent 2 socket checks:

```text
net.tcp.listen[{$3XUI.PANEL.PORT}]
```

Find local `x-ui` ports with:

```bash
ss -lntp | grep x-ui
```

Local panel port checks are disabled by default:

```text
{$3XUI.PANEL.PORT.CHECK}=0
```

Enable only when you know the actual local `x-ui` panel port and want to alert on it.

---

## Security recommendations

Treat the API token as a full administrator credential.

Recommended access model:

```text
Zabbix Server / Proxy IP → allowed to access 3X-UI API
All other public IPs       → denied or restricted
API token                  → stored only in Zabbix Secret macro
```

Example UFW rule:

```bash
ufw allow from <ZABBIX_SERVER_IP> to any port <3XUI_PANEL_EXTERNAL_PORT> proto tcp
ufw deny <3XUI_PANEL_EXTERNAL_PORT>/tcp
```

Do not store or expose database dumps, full config JSON, private keys, API tokens or subscription links in Zabbix history.

---

## Feature overview

### Core service and API health

- `x-ui.service` state;
- panel process count;
- Xray process count;
- API availability;
- API no-data detection;
- panel version and update availability;
- Xray version and API-reported Xray state.

### System metrics

From `/server/status`:

- CPU, memory, disk;
- disk read/write rates;
- network upload/download rates;
- load averages;
- TCP/UDP connection counts;
- public IPs when exposed;
- panel app memory, threads and uptime.

### Inbounds

- aggregate inbound counters;
- per-inbound discovery;
- protocol, tag, port, expiry, traffic, clients count;
- unique TCP/UDP port discovery to avoid duplicate `net.tcp.listen[443]` keys;
- local inbound port triggers disabled by default.

### Clients

- aggregate client counters always available;
- optional Client LLD disabled by default;
- per-client online state, traffic, expiry, group and last-online data.

### Nodes

- intended for central/admin 3X-UI panels with remote nodes;
- optional Node LLD disabled by default;
- node status, health, heartbeat, versions, counts and detail API data.

### Xray, outbounds and observatory

- Xray state, version and result text;
- Xray result error/warning pattern counts;
- outbound traffic aggregate;
- optional per-outbound LLD;
- optional observatory targets and delay checks.

### Logs

- panel logs raw;
- Xray logs raw;
- error/warning pattern counters;
- last error-looking log line;
- log triggers disabled by default.

### Database and backups

- local SQLite DB exists/size/mtime/owner/permissions;
- DB modification age and size delta;
- optional backup directory checks;
- optional latest backup age through `vfs.dir.get[]`.

### Web UI and TLS

- optional Web UI scenario;
- optional TLS certificate checks;
- both disabled by default until URL/host/port are configured.

---

## Default-disabled features

These features are intentionally off by default:

| Feature | Default | Why |
|---|---:|---|
| Client LLD | `{$3XUI.CLIENT.LLD.ENABLED}=0` | Can create many items on large installations. |
| Node LLD | `{$3XUI.NODE.LLD.ENABLED}=0` | Only useful on central/admin panels with nodes. |
| Outbound LLD | `{$3XUI.OUTBOUND.LLD.ENABLED}=0` | Outbound payloads vary by Xray config. |
| Inbound local port triggers | `{$3XUI.INBOUND.PORT.CHECK}=0` | Invalid on central panels when inbounds belong to remote nodes. |
| Panel local port trigger | `{$3XUI.PANEL.PORT.CHECK}=0` | Local port differs between installations. |
| Xray metrics missing trigger | `{$3XUI.XRAY.METRICS.CHECK}=0` | Xray metrics are optional. |
| Observatory triggers | `{$3XUI.OBSERVATORY.CHECK}=0` | Observatory is optional. |
| Log triggers | `{$3XUI.LOGS.CHECK}=0` | Logs can be noisy. |
| Web UI scenario trigger | `{$3XUI.WEB.CHECK}=0` | Web URL must be configured first. |
| TLS triggers/items | `{$3XUI.TLS.CHECK}=0` | TLS host/port must be configured first. |
| DB mtime trigger | `{$3XUI.DB.MTIME.CHECK}=0` | Quiet servers may not update DB often. |
| Backup triggers/items | `{$3XUI.BACKUP.CHECK}=0` | Backup path must be configured first. |

---

## Screenshots

Add screenshots under `docs/screenshots/`.

Suggested files:

```text
docs/screenshots/latest-data-overview.png
docs/screenshots/problems-empty.png
docs/screenshots/inbounds-lld.png
docs/screenshots/clients-lld.png
docs/screenshots/nodes-lld.png
docs/screenshots/web-tls.png
```

Example section:

```markdown
### Latest data overview

![Latest data overview](docs/screenshots/latest-data-overview.png)
```

---

## Troubleshooting

Detailed troubleshooting is in [docs/TROUBLESHOOTING.md](docs/TROUBLESHOOTING.md).

Quick checks after import:

```text
Monitoring → Problems
Data collection → Hosts → <host> → Items → State = Not supported
Data collection → Hosts → <host> → Discovery → Info
Monitoring → Latest data
```

Expected:

```text
Unexpected Problems: none
Not supported: 0 or only intentionally disabled optional checks
Discovery Info: empty
API availability: 1
Xray state: running / 1
Database file exists: 1
```

---

## Data intentionally not collected

For security reasons, the template intentionally does not collect or store:

- database dumps from `/server/getDb`;
- full Xray config JSON;
- private keys;
- generated UUID/private-key helper outputs;
- client subscription links;
- client connection URLs;
- API tokens;
- panel passwords.

The template focuses on health, counters, states and operational signals.

---

## Project stats

![GitHub stars](https://img.shields.io/github/stars/TyranR/zabbix-3x-ui-monitoring?style=social)
![GitHub forks](https://img.shields.io/github/forks/TyranR/zabbix-3x-ui-monitoring?style=social)
![GitHub issues](https://img.shields.io/github/issues/TyranR/zabbix-3x-ui-monitoring)
![GitHub pull requests](https://img.shields.io/github/issues-pr/TyranR/zabbix-3x-ui-monitoring)

---

## License

This project is licensed under the MIT License. See [LICENSE](LICENSE) for details.

TL;DR: Free to use for personal and commercial projects. Attribution appreciated but not required.
