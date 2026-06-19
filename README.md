# Zabbix 3X-UI Monitoring

![Zabbix](https://img.shields.io/badge/Zabbix-7.4-red)
![3X-UI](https://img.shields.io/badge/VPN-3X--UI-orange)
![Agent](https://img.shields.io/badge/agent-Agent%202-blue)
![Platform](https://img.shields.io/badge/platform-Linux-lightgrey)
![License](https://img.shields.io/badge/license-MIT-green)

Reusable Zabbix 7.4 template for monitoring Linux servers running **3X-UI** and **Xray-core**.

The template combines:

- native **Zabbix Agent 2** checks for local Linux/systemd/process/file/socket data;
- native **HTTP Agent** items for the 3X-UI Panel API;
- Dependent Items with JSONPath and JavaScript preprocessing;
- optional Low-Level Discovery for inbounds, clients, nodes and outbounds;
- optional Web UI, TLS certificate, log and backup checks.

No external scripts are required.

---

## What this solves

3X-UI installations usually expose multiple moving parts at once: the 3X-UI panel process, Xray-core, local panel/API ports, public/reverse-proxied API URLs, client-facing inbounds, remote nodes, clients, traffic quotas, database files, backups, logs, and update state.

This template answers the operational questions:

- Is the 3X-UI service running?
- Is the `x-ui` process alive?
- Is Xray-core running locally and according to the API?
- Is the 3X-UI API reachable from Zabbix Server/Proxy?
- Does the API return valid JSON?
- What does 3X-UI report about CPU, memory, disk, load, network, connections and uptime?
- Are inbounds present, enabled and close to expiry/quota limits?
- Which clients are enabled, online, expired or close to traffic/expiry limits?
- Are remote nodes online and sending recent heartbeats?
- Are Xray logs or panel logs exposing error-like patterns?
- Is the local SQLite database present and sane?
- Are optional backup/TLS/Web UI checks configured and healthy?

---

## Project status

Current template version: **1.9.3**.

Current template contents:

```text
Items: 119
Discovery rules: 6
Item prototypes: 55
Trigger prototypes: 20
Web scenarios: 1
Triggers total: 57
Macros: 60
```

The template is suitable for a public GitHub release, but optional modules are intentionally disabled by default where they may be noisy or environment-specific.

---

## Repository structure

```text
zabbix-3x-ui-monitoring/
├── README.md
├── templates/
│   └── zabbix_3x_ui_panel_by_agent2.yaml
└── docs/
    └── screenshots/
```

---

## Requirements

- Zabbix Server 7.4 or compatible 7.x version;
- Zabbix Agent 2 installed on each monitored 3X-UI Linux host;
- 3X-UI Panel API token;
- network access from Zabbix Server/Proxy to the configured 3X-UI API URL;
- active Agent 2 checks configured, or local items changed to passive checks if you prefer passive mode.

---

## Architecture

The template uses two main data collection paths.

### 1. HTTP Agent API polling

```text
Zabbix Server / Zabbix Proxy
    ↓ HTTP Agent item
3X-UI Panel API
```

Used for:

- API availability;
- `/server/status` telemetry;
- Xray state and versions;
- panel version and update checks;
- inbounds, clients, nodes and outbounds data;
- Xray metrics/observatory data;
- panel/Xray logs.

Important: HTTP Agent items run from **Zabbix Server or Zabbix Proxy**, not from the local Zabbix Agent 2.

### 2. Local Zabbix Agent 2 checks

```text
Zabbix Agent 2 on the 3X-UI host
    ↓ native agent keys
Linux system / systemd / processes / local ports / files
```

Used for:

- `x-ui.service` state;
- panel and Xray process counts;
- local panel/inbound TCP/UDP listen checks when enabled;
- database file checks;
- optional backup directory checks;
- optional TLS certificate checks when enabled.

---

## Installation

### 1. Import the template

Open **Data collection → Templates → Import** and upload:

```text
templates/zabbix_3x_ui_panel_by_agent2.yaml
```

Recommended import options while upgrading this template:

```text
Create new: checked
Update existing: checked
Delete missing: checked
```

`Delete missing` is recommended because the template evolved from per-inbound port checks to unique per-port discovery rules. Without it, old prototypes may remain and cause duplicate keys such as `net.tcp.listen[443]`.

### 2. Link the template to a host

```text
Host name: your-3x-ui-host
Groups: VPN Nodes / Linux servers
Templates: 3X-UI Panel by Zabbix agent 2
Interfaces: Zabbix Agent interface for the 3X-UI Linux host
```

### 3. Configure required host macros

At minimum, set these at host level:

| Macro | Example | Notes |
|---|---|---|
| `{$3XUI.API.URL}` | `https://vpn.example.com:25551/secret-path/panel/api` | Base API URL visible from Zabbix Server/Proxy. Do not append `/server/status`. |
| `{$3XUI.API.TOKEN}` | `<secret>` | Bearer API token from 3X-UI. Store as Secret text. |
| `{$3XUI.PROCESS.XRAY}` | `xray-linux-amd64` | Command-line substring used to detect Xray. |
| `{$3XUI.PANEL.PORT}` | `26777` | Local `x-ui` port, only needed if local panel port checks are enabled. |
| `{$3XUI.DB.PATH}` | `/etc/x-ui/x-ui.db` | Local SQLite database path. |

---

## Important: API URL vs local panel port

`{$3XUI.API.URL}` and `{$3XUI.PANEL.PORT}` are not the same thing.

### API URL

Used by HTTP Agent items and must be reachable from **Zabbix Server/Proxy**:

```text
https://vpn.example.com:25551/secret-path/panel/api
```

Correct:

```text
https://vpn.example.com:25551/secret-path/panel/api
```

Incorrect:

```text
https://vpn.example.com:25551/panel/api
```

The secret path must be included when 3X-UI uses one.

### Local panel port

Used by Agent 2 when local panel port checks are enabled:

```text
net.tcp.listen[{$3XUI.PANEL.PORT}]
```

Find the local `x-ui` listen ports:

```bash
ss -lntp | grep x-ui
```

Example:

```text
LISTEN 0 4096 *:2096  *:* users:(("x-ui",pid=576386,fd=15))
LISTEN 0 4096 *:26777 *:* users:(("x-ui",pid=576386,fd=9))
```

Test which one is the Web/API port:

```bash
curl -k -i https://127.0.0.1:26777/secret-path/panel/api/server/status
```

Without a token, 3X-UI may return `404`. This can be normal. Confirm with Bearer token:

```bash
curl -k -i   -H "Authorization: Bearer YOUR_API_TOKEN"   https://127.0.0.1:26777/secret-path/panel/api/server/status
```

A valid response should be:

```text
HTTP/1.1 200 OK
{"success":true,...}
```

---

## Security recommendations

The 3X-UI API token should be treated as a full administrator credential.

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

If the panel is behind Nginx, Caddy or another reverse proxy, restrict access at the reverse-proxy level, especially for `/panel/api/*`.

Do not publish the API token, secret path, private keys, subscription links or database dumps.

---

## Active Agent 2 checks

For active checks, Agent 2 must connect to Zabbix Server/Proxy on port `10051` and use the exact Zabbix host name.

Check configuration:

```bash
grep -E '^(Server|ServerActive|Hostname|HostnameItem)=' /etc/zabbix/zabbix_agent2.conf
```

Example:

```ini
Server=<ZABBIX_SERVER_IP>
ServerActive=<ZABBIX_SERVER_IP>:10051
Hostname=your-3x-ui-host
```

Restart:

```bash
systemctl restart zabbix-agent2
```

Check logs:

```bash
journalctl -u zabbix-agent2 -n 100 --no-pager
```

---

## Default-disabled features

Several features are disabled by default to avoid noisy alerts and unsupported items on generic installations.

| Feature | Default | Enable when |
|---|---:|---|
| Client LLD | `{$3XUI.CLIENT.LLD.ENABLED}=0` | You want per-client items and have a manageable number of clients. |
| Node LLD | `{$3XUI.NODE.LLD.ENABLED}=0` | This host is a central/admin 3X-UI panel with remote nodes added. |
| Outbound LLD | `{$3XUI.OUTBOUND.LLD.ENABLED}=0` | You want per-outbound traffic items. |
| Inbound local port triggers | `{$3XUI.INBOUND.PORT.CHECK}=0` | API inbounds belong to the same local host and local port checks are valid. |
| Panel local port trigger | `{$3XUI.PANEL.PORT.CHECK}=0` | You know the actual local `x-ui` panel port and want to alert on it. |
| Xray metrics missing trigger | `{$3XUI.XRAY.METRICS.CHECK}=0` | You intentionally configured Xray metrics and want to alert if missing. |
| Observatory triggers | `{$3XUI.OBSERVATORY.CHECK}=0` | Xray observatory is configured and should be monitored. |
| Logs triggers | `{$3XUI.LOGS.CHECK}=0` | You reviewed log noise and want to alert on error-like patterns. |
| Web UI scenario trigger | `{$3XUI.WEB.CHECK}=0` | `{$3XUI.WEB.URL}` is set and the web scenario is enabled. |
| TLS triggers/items | `{$3XUI.TLS.CHECK}=0` | TLS host/port are set and TLS items are manually enabled. |
| Database mtime trigger | `{$3XUI.DB.MTIME.CHECK}=0` | The database is expected to change regularly. |
| Backup triggers/items | `{$3XUI.BACKUP.CHECK}=0` | Backup path is set and backup items are manually enabled. |

Disabled items by default:

- backup path/size/count items;
- backup directory listing/latest backup items;
- TLS certificate items;
- Web UI scenario.

---

## Macros

| Macro | Type | Default | Description |
|---|---|---|---|
| `{$3XUI.API.TIMEOUT}` | Text | `10s` | Timeout for 3X-UI API HTTP agent items. |
| `{$3XUI.API.TOKEN}` | SECRET_TEXT | `<secret>` | Bearer API token from 3X-UI Settings -> Security -> API Token. Treat as a full administrator credential. |
| `{$3XUI.API.URL}` | Text | `http://127.0.0.1:2053/panel/api` | Base 3X-UI Panel API URL. HTTP agent items are executed by Zabbix Server/Proxy, so 127.0.0.1 works only when the server/proxy runs on the same host or API is otherwise reachable from it. |
| `{$3XUI.BACKUP.CHECK}` | Text | `0` | Enable optional backup directory triggers: 1=enabled, 0=disabled. Backup items are disabled by default and should be enabled only after {$3XUI.BACKUP.PATH} is set correctly. |
| `{$3XUI.BACKUP.MAX.SIZE}` | Text | `1073741824` | Maximum expected backup directory size in bytes when backup checks are enabled. Default is 1 GiB. |
| `{$3XUI.BACKUP.MIN.COUNT}` | Text | `1` | Minimum expected number of files in the backup directory when backup checks are enabled. |
| `{$3XUI.BACKUP.PATH}` | Text | `/etc/x-ui/backup` | Path to the local 3X-UI backup directory, if backups are stored on disk. Adjust per host before enabling backup items. |
| `{$3XUI.CLIENT.EXPIRY.HIGH}` | Text | `3` | High threshold for client expiry in days. Clients without expiry return -1 and do not trigger expiry alerts. |
| `{$3XUI.CLIENT.EXPIRY.WARN}` | Text | `7` | Warning threshold for client expiry in days. Clients without expiry return -1 and do not trigger expiry alerts. |
| `{$3XUI.CLIENT.LASTONLINE.CHECK}` | Text | `0` | Enable per-client last-online trigger: 1=enabled, 0=disabled. Disabled by default because not every installation expects every client to be regularly online. |
| `{$3XUI.CLIENT.LASTONLINE.WARN}` | Text | `30` | Warning threshold for days since the client was last seen online. Used by optional client LLD triggers when enabled. |
| `{$3XUI.CLIENT.LLD.ENABLED}` | Text | `0` | Enable per-client Low-Level Discovery: 1=enabled, 0=disabled. Disabled by default because client discovery can create many items on large installations. |
| `{$3XUI.CLIENT.TRAFFIC.HIGH}` | Text | `95` | High threshold for per-client traffic quota usage percentage. Only applies when client traffic limit is greater than 0. |
| `{$3XUI.CLIENT.TRAFFIC.WARN}` | Text | `80` | Warning threshold for per-client traffic quota usage percentage. Only applies when client traffic limit is greater than 0. |
| `{$3XUI.CPU.UTIL.HIGH}` | Text | `90` | High CPU utilization threshold in percent. |
| `{$3XUI.DB.CHECK}` | Text | `1` | Enable local 3X-UI database file triggers: 1=enabled, 0=disabled. Disable when the panel does not use a local SQLite database file. |
| `{$3XUI.DB.MAX.SIZE}` | Text | `1073741824` | Maximum expected 3X-UI database file size in bytes. Default is 1 GiB; adjust for large installations. |
| `{$3XUI.DB.MIN.SIZE}` | Text | `1024` | Minimum expected 3X-UI database file size in bytes. Used to detect empty or suspiciously small database files. |
| `{$3XUI.DB.MTIME.CHECK}` | Text | `0` | Enable database modification age trigger: 1=enabled, 0=disabled. Disabled by default because quiet servers may not update the SQLite file frequently. |
| `{$3XUI.DB.MTIME.MAXAGE}` | Text | `86400` | Maximum allowed database file modification age in seconds when {$3XUI.DB.MTIME.CHECK}=1. Default is 86400 seconds. |
| `{$3XUI.DB.PATH}` | Text | `/etc/x-ui/x-ui.db` | Path to the SQLite database file. If PostgreSQL is used, adjust or disable database file items/triggers. |
| `{$3XUI.DISK.PUSED.HIGH}` | Text | `90` | High root disk utilization threshold in percent. |
| `{$3XUI.INBOUND.CLIENTS.CHECK}` | Text | `0` | Enable per-inbound no-clients trigger: 1=enabled, 0=disabled. Disabled by default because fallback/mtproto/maintenance inbounds may legitimately have no clients. |
| `{$3XUI.INBOUND.EXPIRY.HIGH}` | Text | `3` | High threshold for inbound expiry in days. Inbounds without expiry return a large placeholder value and do not trigger. |
| `{$3XUI.INBOUND.EXPIRY.WARN}` | Text | `7` | Warning threshold for inbound expiry in days. Inbounds without expiry return a large placeholder value and do not trigger. |
| `{$3XUI.INBOUND.PORT.CHECK}` | Text | `0` | Enable inbound local port listening triggers: 1=enabled, 0=disabled. Disabled by default because central panels may expose inbounds belonging to remote nodes, while Zabbix Agent 2 checks only local sockets. |
| `{$3XUI.INBOUND.PORT.DOWN.TIME}` | Text | `3m` | Time window for per-inbound port-down triggers. The port problem is raised only when the port is continuously not listening during this period. |
| `{$3XUI.INBOUND.TRAFFIC.HIGH}` | Text | `95` | High threshold for inbound traffic quota usage percentage. Only applies when inbound total traffic limit is greater than 0. |
| `{$3XUI.INBOUND.TRAFFIC.WARN}` | Text | `80` | Warning threshold for inbound traffic quota usage percentage. Only applies when inbound total traffic limit is greater than 0. |
| `{$3XUI.MEM.PUSED.HIGH}` | Text | `90` | High memory utilization threshold in percent. |
| `{$3XUI.NODE.DETAIL.CHECK}` | Text | `1` | Enable per-node detail API success trigger: 1=enabled, 0=disabled. Uses /panel/api/nodes/get/{id} for discovered nodes. |
| `{$3XUI.NODE.HEALTH.CHECK}` | Text | `1` | Enable per-node unhealthy trigger: 1=enabled, 0=disabled. Disable if the current 3X-UI version does not expose reliable node health/status fields. |
| `{$3XUI.NODE.HEARTBEAT.CHECK}` | Text | `1` | Enable per-node stale heartbeat trigger: 1=enabled, 0=disabled. Disable if the current 3X-UI version does not expose last heartbeat timestamps. |
| `{$3XUI.NODE.HEARTBEAT.MAXAGE}` | Text | `600` | Maximum allowed node heartbeat age in seconds for node stale-heartbeat triggers. Default is 600 seconds. |
| `{$3XUI.NODE.LLD.ENABLED}` | Text | `0` | Enable per-node Low-Level Discovery: 1=enabled, 0=disabled. Enable only on the 3X-UI host that acts as the central/admin panel for remote nodes. |
| `{$3XUI.NODE.TCP.COUNT.HIGH}` | Text | `10000` | High TCP connection count threshold for per-node detail payloads when the API exposes tcpCount. |
| `{$3XUI.NODE.UDP.COUNT.HIGH}` | Text | `10000` | High UDP connection count threshold for per-node detail payloads when the API exposes udpCount. |
| `{$3XUI.OBSERVATORY.CHECK}` | Text | `0` | Enable Xray observatory triggers: 1=enabled, 0=disabled. Disabled by default because observatory is optional and not configured on every 3X-UI installation. |
| `{$3XUI.OBSERVATORY.DELAY.HIGH}` | Text | `1000` | High observatory delay threshold in milliseconds. Used only when {$3XUI.OBSERVATORY.CHECK}=1. |
| `{$3XUI.OUTBOUND.LLD.ENABLED}` | Text | `0` | Enable per-outbound traffic Low-Level Discovery: 1=enabled, 0=disabled. Disabled by default because outbound tags and payload shape may vary by Xray configuration. |
| `{$3XUI.PANEL.PORT}` | Text | `2053` | Local TCP port of the 3X-UI web panel. Adjust to your panel port. |
| `{$3XUI.PROCESS.PANEL}` | Text | `x-ui` | Process name used to count the 3X-UI panel process. |
| `{$3XUI.PROCESS.XRAY}` | Text | `xray-linux-amd64` | Command line substring used to count the Xray process. |
| `{$3XUI.SERVICE.NAME}` | Text | `x-ui.service` | Systemd unit name used by standard 3X-UI installations. |
| `{$3XUI.TCP.COUNT.HIGH}` | Text | `10000` | High TCP connection count threshold. |
| `{$3XUI.XRAY.RESULT.ERROR.CHECK}` | Text | `1` | Enable trigger for error/fatal/panic/failed patterns in /xray/getXrayResult: 1=enabled, 0=disabled. |
| `{$3XUI.PANEL.PORT.CHECK}` | Text | `0` | Enable local 3X-UI panel TCP port trigger: 1=enabled, 0=disabled. Disabled by default because panel/API availability is already checked through HTTP API and local panel ports differ between installations. |
| `{$3XUI.XRAY.METRICS.CHECK}` | Text | `0` | Enable trigger for missing Xray metrics: 1=enabled, 0=disabled. Disabled by default because Xray metrics are optional and not required for 3X-UI to work. |
| `{$3XUI.WEB.CHECK}` | Text | `0` | Enable Web UI scenario trigger: 1=enabled, 0=disabled. The web scenario itself is disabled by default and should be enabled after {$3XUI.WEB.URL} is set. |
| `{$3XUI.WEB.URL}` | Text | `http://127.0.0.1:2053/` | Full 3X-UI Web UI URL visible from Zabbix Server/Proxy, including scheme, port and secret path when used. Example: https://vpn.example.com:2053/secret-path/ |
| `{$3XUI.WEB.STATUS_CODES}` | Text | `200,301,302,401,403` | Expected HTTP status codes for the optional Web UI scenario. |
| `{$3XUI.TLS.CHECK}` | Text | `0` | Enable TLS certificate triggers: 1=enabled, 0=disabled. TLS items are disabled by default and should be enabled only after {$3XUI.TLS.HOST} and {$3XUI.TLS.PORT} are set. |
| `{$3XUI.TLS.HOST}` | Text | `3x-ui.example.com` | Hostname used by optional TLS certificate checks. Set this to the public panel/API hostname before enabling TLS items. |
| `{$3XUI.TLS.PORT}` | Text | `443` | TCP port used by optional TLS certificate checks. |
| `{$3XUI.TLS.EXPIRY.WARN}` | Text | `14` | Warning threshold for TLS certificate expiry in days. |
| `{$3XUI.TLS.EXPIRY.HIGH}` | Text | `3` | High threshold for TLS certificate expiry in days. |
| `{$3XUI.LOGS.CHECK}` | Text | `0` | Enable panel/Xray log pattern triggers: 1=enabled, 0=disabled. Disabled by default because logs can be noisy. |
| `{$3XUI.LOGS.COUNT}` | Text | `100` | Number of recent log lines requested from 3X-UI log API endpoints. |
| `{$3XUI.LOGS.ERROR.THRESHOLD}` | Text | `1` | Number of error-like log patterns required to raise a log trigger when {$3XUI.LOGS.CHECK}=1. |
| `{$3XUI.BACKUP.MAXAGE}` | Text | `86400` | Maximum allowed latest backup age in seconds when optional backup listing checks are enabled. |

---

## Feature modules

### Core service and API health

Core monitoring includes:

- `x-ui.service` active state;
- panel process count;
- Xray process count;
- API availability;
- API no-data detection;
- panel version;
- Xray version and Xray state;
- update availability.

Xray process detection uses command-line substring matching:

```text
proc.num[,,,{$3XUI.PROCESS.XRAY}]
```

Recommended value:

```text
{$3XUI.PROCESS.XRAY}=xray-linux-amd64
```

Any value greater than `0` means Xray is present.

### System metrics from 3X-UI API

The `/server/status` master item provides dependent metrics for:

- CPU utilization;
- CPU cores/speed/logical processors;
- memory total/used/utilization;
- root disk total/used/utilization;
- disk read/write rate;
- network upload/download rate;
- load averages;
- TCP/UDP connection counters;
- system uptime;
- public IPs when exposed by API;
- 3X-UI application memory/threads/uptime.

### Inbounds

The template collects aggregate inbound metrics and optional LLD data from `/inbounds/list/slim`.

Inbound LLD creates per-inbound items such as:

- enabled state;
- protocol;
- tag;
- port;
- upload/download traffic;
- traffic limit and usage;
- expiry time and days before expiry;
- clients count;
- stream network/security.

Local port checks are handled by separate unique port discovery rules:

- `3X-UI: Inbound TCP ports discovery`;
- `3X-UI: Inbound UDP ports discovery`.

This avoids duplicate item keys when several inbounds share the same port, for example multiple objects on `443`.

Enable local inbound port triggers only when the API inbounds belong to the same host as the local Agent 2:

```text
{$3XUI.INBOUND.PORT.CHECK}=1
```

For central panels with remote nodes, keep it disabled:

```text
{$3XUI.INBOUND.PORT.CHECK}=0
```

### Clients

Client aggregate items work even when Client LLD is disabled:

- clients total/enabled/disabled;
- expired clients;
- clients expiring soon;
- clients over traffic quota;
- total client upload/download/limit/usage;
- online clients count.

Optional Client LLD is disabled by default:

```text
{$3XUI.CLIENT.LLD.ENABLED}=0
```

Enable per host when needed:

```text
{$3XUI.CLIENT.LLD.ENABLED}=1
```

Per-client items include:

- enabled;
- online;
- group;
- attached inbounds;
- upload/download traffic;
- traffic limit and usage;
- expiry time and days before expiry;
- last online and days since last online.

### Nodes

Nodes monitoring is intended for a central/admin 3X-UI panel that has remote nodes added.

Recommended setup:

```text
Central/admin host:
  {$3XUI.NODE.LLD.ENABLED}=1
  {$3XUI.INBOUND.PORT.CHECK}=0

Standalone remote hosts:
  {$3XUI.NODE.LLD.ENABLED}=0
```

Per-node items include:

- enabled;
- address;
- status text;
- health state;
- heartbeat age;
- last heartbeat;
- version;
- clients count;
- inbounds count;
- upload/download traffic;
- API detail raw;
- API detail success/error;
- panel version;
- Xray version;
- uptime;
- TCP/UDP connection counters when exposed by node detail API.

`health state` values:

```text
1  = healthy / online / running / ok / active / connected
0  = unhealthy
-1 = unknown or not reported by API
```

### Xray, outbounds and observatory

Xray-related items include:

- Xray state;
- Xray state text;
- Xray version;
- Xray error message;
- Xray result text;
- Xray result error/warning patterns;
- Xray metrics enabled;
- Xray metrics keys count;
- outbound traffic aggregate;
- optional per-outbound traffic LLD;
- observatory targets total/failed;
- observatory average/maximum delay.

Observatory triggers are disabled by default:

```text
{$3XUI.OBSERVATORY.CHECK}=0
```

### Logs

The template can collect recent panel and Xray logs through API endpoints:

- `3X-UI API: panel logs raw`;
- `3X-UI API: Xray logs raw`.

Dependent items count error-like and warning-like patterns and extract the last error-looking line.

Log triggers are disabled by default:

```text
{$3XUI.LOGS.CHECK}=0
```

Enable only after checking that the logs are not too noisy.

### Database health

Database checks use local Agent 2 items and do not download the database.

Collected items:

- database file exists;
- database file size;
- database file modification time;
- database file owner;
- database file permissions;
- database modification age;
- database file size change.

Important: the template intentionally does **not** use `/server/getDb`, because that would store sensitive database contents in Zabbix history.

If the installation does not use local SQLite, set:

```text
{$3XUI.DB.CHECK}=0
```

### Backup checks

Backup checks are optional and disabled by default.

Items:

- backup path exists;
- backup directory size;
- backup files count;
- backup directory listing raw;
- latest backup modification time;
- latest backup age.

To enable:

```text
{$3XUI.BACKUP.PATH}=/path/to/backups
{$3XUI.BACKUP.CHECK}=1
```

Then manually enable backup items in Zabbix.

### Web UI scenario

The optional Web UI scenario checks the configured Web UI URL.

Set:

```text
{$3XUI.WEB.URL}=https://vpn.example.com:25551/secret-path/
{$3XUI.WEB.CHECK}=1
```

Then enable the Web scenario:

```text
Data collection → Hosts/Templates → Web → 3X-UI Panel Web UI Access → Enable
```

### TLS certificate checks

TLS certificate items are disabled by default.

Set:

```text
{$3XUI.TLS.HOST}=vpn.example.com
{$3XUI.TLS.PORT}=443
{$3XUI.TLS.CHECK}=1
```

Then enable TLS certificate items:

- `3X-UI: TLS certificate raw`;
- `3X-UI: TLS certificate expires in days`;
- `3X-UI: TLS certificate subject`;
- `3X-UI: TLS certificate issuer`.

---

## Troubleshooting

### API item returns `Failed to connect to 127.0.0.1:2053`

HTTP Agent items run on Zabbix Server/Proxy, not on local Agent 2. Set `{$3XUI.API.URL}` to a URL reachable from Zabbix Server/Proxy.

### API endpoint returns `404` without token

This can be normal. 3X-UI may hide API endpoints from unauthenticated requests. Test with Bearer token.

### Active Agent 2 items do not arrive

Check:

- `ServerActive` points to Zabbix Server/Proxy;
- `Hostname` exactly matches the Zabbix host name;
- port `10051` is reachable from the 3X-UI host to Zabbix;
- the host and template are enabled.

### Xray process trigger fires while Xray is running

Check process command line:

```bash
ps aux | grep -i xray | grep -v grep
```

Use:

```text
{$3XUI.PROCESS.XRAY}=xray-linux-amd64
```

The trigger should alert only when process count is `0`.

### Panel port trigger fires while API works

Local panel port checks are optional. Keep disabled unless you know the local `x-ui` listen port:

```text
{$3XUI.PANEL.PORT.CHECK}=0
```

### Inbound port trigger fires on central node host

For central panels with remote nodes, API inbounds may not belong to the local Linux host. Keep disabled:

```text
{$3XUI.INBOUND.PORT.CHECK}=0
```

### Duplicate `net.tcp.listen[443]` or `net.udp.listen[443]`

Use the current template with unique inbound port discovery rules and import with:

```text
Delete missing: checked
```

Old per-inbound port prototypes should be removed.

### Client or node LLD creates no items

Check the enabling macros:

```text
{$3XUI.CLIENT.LLD.ENABLED}=1
{$3XUI.NODE.LLD.ENABLED}=1
```

Then run:

```text
Data collection → Hosts → <host> → Discovery → Execute now
```

### Logs are noisy

Keep disabled:

```text
{$3XUI.LOGS.CHECK}=0
```

The log items may still collect data, but triggers will not fire unless enabled.

### Backup items are disabled

This is expected. Configure path first, then enable items:

```text
{$3XUI.BACKUP.PATH}=/path/to/backups
{$3XUI.BACKUP.CHECK}=1
```

### TLS items are disabled

This is expected. Configure host/port first, then enable TLS items.

---

## Data intentionally not collected

For security reasons, the template intentionally does not collect or store:

- 3X-UI database dump from `/server/getDb`;
- full Xray config JSON;
- private keys;
- generated UUID/private key helper outputs;
- client subscription links;
- client connection URLs;
- API tokens;
- panel passwords.

The template focuses on health, counters, states and operational signals.

---

## Recommended smoke test after import

After importing and linking the template:

```text
Monitoring → Problems
Data collection → Hosts → <host> → Items → State = Not supported
Data collection → Hosts → <host> → Discovery → Info
Monitoring → Latest data
```

Expected:

```text
Unexpected Problems: none
Not supported: 0 or only intentionally disabled/optional checks
Discovery Info: empty
API availability: 1
Xray state: 1/running
Database file exists: 1
```

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
