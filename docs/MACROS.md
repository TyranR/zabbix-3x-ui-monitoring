# 3X-UI Zabbix Template Macros

This file documents all user macros used by **3X-UI Panel by Zabbix agent 2** v1.9.6.

Set required values at host level. Host-level values override template defaults.

---

## Required minimum

| Macro | Example | Purpose |
|---|---|---|
| `{$3XUI.API.URL}` | `https://vpn.example.com:25551/secret-path/panel/api` | Base API URL reachable from Zabbix Server/Proxy. |
| `{$3XUI.API.TOKEN}` | `<secret>` | Bearer token from 3X-UI. Store as Secret text. |
| `{$3XUI.PROCESS.XRAY}` | `xray-linux-amd64` | Command-line substring for Xray process detection. |
| `{$3XUI.DB.PATH}` | `/etc/x-ui/x-ui.db` | SQLite DB path when local DB checks are used. |

---

## Full macro reference

### Core API and local service

| Macro | Type | Default | Description |
|---|---|---|---|
| `{$3XUI.API.TIMEOUT}` | Text | `10s` | Timeout for 3X-UI API HTTP agent items. |
| `{$3XUI.API.TOKEN}` | SECRET_TEXT | `<secret>` | Bearer API token from 3X-UI Settings -> Security -> API Token. Treat as a full administrator credential. |
| `{$3XUI.API.URL}` | Text | `http://127.0.0.1:2053/panel/api` | Base 3X-UI Panel API URL. HTTP agent items are executed by Zabbix Server/Proxy, so 127.0.0.1 works only when the server/proxy runs on the same host or API is otherwise reachable from it. |
| `{$3XUI.PANEL.PORT}` | Text | `2053` | Local TCP port of the 3X-UI web panel. Adjust to your panel port. |
| `{$3XUI.PROCESS.PANEL}` | Text | `x-ui` | Process name used to count the 3X-UI panel process. |
| `{$3XUI.PROCESS.XRAY}` | Text | `xray-linux-amd64` | Command line substring used to count the Xray process. |
| `{$3XUI.SERVICE.NAME}` | Text | `x-ui.service` | Systemd unit name used by standard 3X-UI installations. |
| `{$3XUI.PANEL.PORT.CHECK}` | Text | `0` | Enable local 3X-UI panel TCP port trigger: 1=enabled, 0=disabled. Disabled by default because panel/API availability is already checked through HTTP API and local panel ports differ between installations. |

### Resource thresholds

| Macro | Type | Default | Description |
|---|---|---|---|
| `{$3XUI.CPU.UTIL.HIGH}` | Text | `90` | High CPU utilization threshold in percent. |
| `{$3XUI.DISK.PUSED.HIGH}` | Text | `90` | High root disk utilization threshold in percent. |
| `{$3XUI.MEM.PUSED.HIGH}` | Text | `90` | High memory utilization threshold in percent. |
| `{$3XUI.NODE.TCP.COUNT.HIGH}` | Text | `10000` | High TCP connection count threshold for per-node detail payloads when the API exposes tcpCount. |
| `{$3XUI.TCP.COUNT.HIGH}` | Text | `10000` | High TCP connection count threshold. |

### Inbounds

| Macro | Type | Default | Description |
|---|---|---|---|
| `{$3XUI.INBOUND.CLIENTS.CHECK}` | Text | `0` | Enable per-inbound no-clients trigger: 1=enabled, 0=disabled. Disabled by default because fallback/mtproto/maintenance inbounds may legitimately have no clients. |
| `{$3XUI.INBOUND.EXPIRY.HIGH}` | Text | `3` | High threshold for inbound expiry in days. Inbounds without expiry return a large placeholder value and do not trigger. |
| `{$3XUI.INBOUND.EXPIRY.WARN}` | Text | `7` | Warning threshold for inbound expiry in days. Inbounds without expiry return a large placeholder value and do not trigger. |
| `{$3XUI.INBOUND.PORT.CHECK}` | Text | `0` | Enable inbound local port listening triggers: 1=enabled, 0=disabled. Disabled by default because central panels may expose inbounds belonging to remote nodes, while Zabbix Agent 2 checks only local sockets. |
| `{$3XUI.INBOUND.PORT.DOWN.TIME}` | Text | `3m` | Time window for per-inbound port-down triggers. The port problem is raised only when the port is continuously not listening during this period. |
| `{$3XUI.INBOUND.TRAFFIC.HIGH}` | Text | `95` | High threshold for inbound traffic quota usage percentage. Only applies when inbound total traffic limit is greater than 0. |
| `{$3XUI.INBOUND.TRAFFIC.WARN}` | Text | `80` | Warning threshold for inbound traffic quota usage percentage. Only applies when inbound total traffic limit is greater than 0. |

### Clients

| Macro | Type | Default | Description |
|---|---|---|---|
| `{$3XUI.CLIENT.EXPIRY.HIGH}` | Text | `3` | High threshold for client expiry in days. Clients without expiry return -1 and do not trigger expiry alerts. |
| `{$3XUI.CLIENT.EXPIRY.WARN}` | Text | `7` | Warning threshold for client expiry in days. Clients without expiry return -1 and do not trigger expiry alerts. |
| `{$3XUI.CLIENT.LASTONLINE.CHECK}` | Text | `0` | Enable per-client last-online trigger: 1=enabled, 0=disabled. Disabled by default because not every installation expects every client to be regularly online. |
| `{$3XUI.CLIENT.LASTONLINE.WARN}` | Text | `30` | Warning threshold for days since the client was last seen online. Used by optional client LLD triggers when enabled. |
| `{$3XUI.CLIENT.LLD.ENABLED}` | Text | `0` | Enable per-client Low-Level Discovery: 1=enabled, 0=disabled. Disabled by default because client discovery can create many items on large installations. |
| `{$3XUI.CLIENT.TRAFFIC.HIGH}` | Text | `95` | High threshold for per-client traffic quota usage percentage. Only applies when client traffic limit is greater than 0. |
| `{$3XUI.CLIENT.TRAFFIC.WARN}` | Text | `80` | Warning threshold for per-client traffic quota usage percentage. Only applies when client traffic limit is greater than 0. |

### Nodes

| Macro | Type | Default | Description |
|---|---|---|---|
| `{$3XUI.NODE.DETAIL.CHECK}` | Text | `1` | Enable per-node detail API success trigger: 1=enabled, 0=disabled. Uses /panel/api/nodes/get/{id} for discovered nodes. |
| `{$3XUI.NODE.HEALTH.CHECK}` | Text | `1` | Enable per-node unhealthy trigger: 1=enabled, 0=disabled. Disable if the current 3X-UI version does not expose reliable node health/status fields. |
| `{$3XUI.NODE.HEARTBEAT.CHECK}` | Text | `1` | Enable per-node stale heartbeat trigger: 1=enabled, 0=disabled. Disable if the current 3X-UI version does not expose last heartbeat timestamps. |
| `{$3XUI.NODE.HEARTBEAT.MAXAGE}` | Text | `600` | Maximum allowed node heartbeat age in seconds for node stale-heartbeat triggers. Default is 600 seconds. |
| `{$3XUI.NODE.LLD.ENABLED}` | Text | `0` | Enable per-node Low-Level Discovery: 1=enabled, 0=disabled. Enable only on the 3X-UI host that acts as the central/admin panel for remote nodes. |
| `{$3XUI.NODE.TCP.COUNT.HIGH}` | Text | `10000` | High TCP connection count threshold for per-node detail payloads when the API exposes tcpCount. |
| `{$3XUI.NODE.UDP.COUNT.HIGH}` | Text | `10000` | High UDP connection count threshold for per-node detail payloads when the API exposes udpCount. |

### Xray/outbounds/observatory/logs

| Macro | Type | Default | Description |
|---|---|---|---|
| `{$3XUI.OBSERVATORY.CHECK}` | Text | `0` | Enable Xray observatory triggers: 1=enabled, 0=disabled. Disabled by default because observatory is optional and not configured on every 3X-UI installation. |
| `{$3XUI.OBSERVATORY.DELAY.HIGH}` | Text | `1000` | High observatory delay threshold in milliseconds. Used only when {$3XUI.OBSERVATORY.CHECK}=1. |
| `{$3XUI.OUTBOUND.LLD.ENABLED}` | Text | `0` | Enable per-outbound traffic Low-Level Discovery: 1=enabled, 0=disabled. Disabled by default because outbound tags and payload shape may vary by Xray configuration. |
| `{$3XUI.XRAY.RESULT.ERROR.CHECK}` | Text | `1` | Enable trigger for error/fatal/panic/failed patterns in /xray/getXrayResult: 1=enabled, 0=disabled. |
| `{$3XUI.XRAY.METRICS.CHECK}` | Text | `0` | Enable trigger for missing Xray metrics: 1=enabled, 0=disabled. Disabled by default because Xray metrics are optional and not required for 3X-UI to work. |
| `{$3XUI.LOGS.CHECK}` | Text | `0` | Enable panel/Xray log pattern triggers: 1=enabled, 0=disabled. Disabled by default because logs can be noisy. |
| `{$3XUI.LOGS.COUNT}` | Text | `100` | Number of recent log lines requested from 3X-UI log API endpoints. |
| `{$3XUI.LOGS.ERROR.THRESHOLD}` | Text | `1` | Number of error-like log patterns required to raise a log trigger when {$3XUI.LOGS.CHECK}=1. |

### Database and backup

| Macro | Type | Default | Description |
|---|---|---|---|
| `{$3XUI.BACKUP.CHECK}` | Text | `0` | Enable optional backup directory triggers: 1=enabled, 0=disabled. Backup items are disabled by default and should be enabled only after {$3XUI.BACKUP.PATH} is set correctly. |
| `{$3XUI.BACKUP.MAX.SIZE}` | Text | `1073741824` | Maximum expected backup directory size in bytes when backup checks are enabled. Default is 1 GiB. |
| `{$3XUI.BACKUP.MIN.COUNT}` | Text | `1` | Minimum expected number of files in the backup directory when backup checks are enabled. |
| `{$3XUI.BACKUP.PATH}` | Text | `/etc/x-ui/backup` | Path to the local 3X-UI backup directory, if backups are stored on disk. Adjust per host before enabling backup items. |
| `{$3XUI.DB.CHECK}` | Text | `1` | Enable local 3X-UI database file triggers: 1=enabled, 0=disabled. Disable when the panel does not use a local SQLite database file. |
| `{$3XUI.DB.MAX.SIZE}` | Text | `1073741824` | Maximum expected 3X-UI database file size in bytes. Default is 1 GiB; adjust for large installations. |
| `{$3XUI.DB.MIN.SIZE}` | Text | `1024` | Minimum expected 3X-UI database file size in bytes. Used to detect empty or suspiciously small database files. |
| `{$3XUI.DB.MTIME.CHECK}` | Text | `0` | Enable database modification age trigger: 1=enabled, 0=disabled. Disabled by default because quiet servers may not update the SQLite file frequently. |
| `{$3XUI.DB.MTIME.MAXAGE}` | Text | `86400` | Maximum allowed database file modification age in seconds when {$3XUI.DB.MTIME.CHECK}=1. Default is 86400 seconds. |
| `{$3XUI.DB.PATH}` | Text | `/etc/x-ui/x-ui.db` | Path to the SQLite database file. If PostgreSQL is used, adjust or disable database file items/triggers. |
| `{$3XUI.BACKUP.MAXAGE}` | Text | `86400` | Maximum allowed latest backup age in seconds when optional backup listing checks are enabled. |

### Web UI and TLS

| Macro | Type | Default | Description |
|---|---|---|---|
| `{$3XUI.WEB.CHECK}` | Text | `0` | Enable Web UI scenario trigger: 1=enabled, 0=disabled. The web scenario itself is disabled by default and should be enabled after {$3XUI.WEB.URL} is set. |
| `{$3XUI.WEB.URL}` | Text | `http://127.0.0.1:2053/` | Full 3X-UI Web UI URL visible from Zabbix Server/Proxy, including scheme, port and secret path when used. Example: https://vpn.example.com:2053/secret-path/ |
| `{$3XUI.WEB.STATUS_CODES}` | Text | `200,301,302,401,403` | Expected HTTP status codes for the optional Web UI scenario. |
| `{$3XUI.TLS.CHECK}` | Text | `0` | Enable TLS certificate triggers: 1=enabled, 0=disabled. TLS items are disabled by default and should be enabled only after {$3XUI.TLS.HOST} and {$3XUI.TLS.PORT} are set. |
| `{$3XUI.TLS.HOST}` | Text | `3x-ui.example.com` | Hostname used by optional TLS certificate checks. Set this to the public panel/API hostname before enabling TLS items. |
| `{$3XUI.TLS.PORT}` | Text | `443` | TCP port used by optional TLS certificate checks. |
| `{$3XUI.TLS.EXPIRY.WARN}` | Text | `14` | Warning threshold for TLS certificate expiry in days. |
| `{$3XUI.TLS.EXPIRY.HIGH}` | Text | `3` | High threshold for TLS certificate expiry in days. |

---

## Common enabling examples

### Enable Client LLD on a small installation

```text
{$3XUI.CLIENT.LLD.ENABLED}=1
```

### Enable Node LLD on the central/admin panel

```text
{$3XUI.NODE.LLD.ENABLED}=1
{$3XUI.INBOUND.PORT.CHECK}=0
```

### Enable local inbound port checks on a standalone host

```text
{$3XUI.INBOUND.PORT.CHECK}=1
```

Only do this when API inbounds belong to the same local Linux host.

### Enable Web UI checks

```text
{$3XUI.WEB.URL}=https://vpn.example.com:25551/secret-path/
{$3XUI.WEB.CHECK}=1
```

Then enable the disabled Web scenario in Zabbix.

### Enable TLS certificate checks

```text
{$3XUI.TLS.HOST}=vpn.example.com
{$3XUI.TLS.PORT}=443
{$3XUI.TLS.CHECK}=1
```

Then enable the disabled TLS items.

### Enable backup checks

```text
{$3XUI.BACKUP.PATH}=/path/to/backups
{$3XUI.BACKUP.CHECK}=1
```

Then enable the disabled backup items.
