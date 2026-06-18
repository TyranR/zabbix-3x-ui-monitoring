# Zabbix 3X-UI Monitoring

![Zabbix](https://img.shields.io/badge/Zabbix-7.4-red)
![3X-UI](https://img.shields.io/badge/VPN-3X--UI-orange)
![Agent](https://img.shields.io/badge/agent-Agent%202-blue)
![Platform](https://img.shields.io/badge/platform-Linux-lightgrey)
![License](https://img.shields.io/badge/license-MIT-green)

Reusable Zabbix 7.4 template for monitoring a Linux server running **3X-UI** and **Xray-core**. The template combines native Zabbix Agent 2 checks for local operating-system data with HTTP Agent API polling of the 3X-UI Panel API.

Repository name:

```text
zabbix-3x-ui-monitoring
```

Template name:

```text
3X-UI Panel by Zabbix agent 2
```

Vendor/version:

```text
TyranR / 1.0b
```

---

## What this solves

3X-UI installations usually expose multiple moving parts at once: the 3X-UI panel process, Xray-core, a local panel/API port, one or more client-facing inbound ports, database files, API telemetry, resource usage, traffic counters, and update state.

This template provides a first monitoring layer that answers the operational questions:

- Is the 3X-UI service running?
- Is the 3X-UI panel process alive?
- Is Xray-core running?
- Is the 3X-UI API reachable from Zabbix Server/Proxy?
- Does the API return valid JSON?
- What does 3X-UI report about CPU, memory, disk, network, connections, uptime and Xray state?
- Are inbounds present and enabled?
- Is the panel update endpoint reporting a newer version?
- Is the local SQLite database file present?

---

## Project status

This template is currently in early public version **1.0b**.

The first version intentionally contains:

- no external scripts;
- no discovery rules;
- HTTP Agent master items for the 3X-UI API;
- Dependent Items for JSON processing;
- Zabbix Agent 2 checks for local Linux service/process/port/file data;
- triggers for core availability and basic resource thresholds.

Planned future improvements:

- inbound Low-Level Discovery;
- per-inbound port checks;
- optional client Low-Level Discovery;
- optional node discovery;
- dashboard export;
- more refined trigger dependencies.

---

## Current limitations

- The API master items are executed by **Zabbix Server or Zabbix Proxy**, not by the local Zabbix Agent 2.
- `{$3XUI.API.URL}` must be reachable from the Zabbix Server/Proxy that runs the HTTP Agent items.
- The local panel port check requires `{$3XUI.PANEL.PORT}` to be set to the actual local port listened by the `x-ui` process.
- The external panel/API port may differ from the local port shown by `ss -lntp`.
- Client-level and inbound-level discovery are not included in v1.0b.
- Some 3X-UI endpoints may return empty objects depending on local configuration, for example Xray metrics or observatory.
- Xray process counting uses a command-line substring; values greater than `0` mean Xray is present.

---

## Tested with

- Zabbix Server 7.4
- Zabbix Agent 2
- Linux host running 3X-UI
- 3X-UI Panel API with Bearer token authentication
- Active Agent 2 checks
- HTTP Agent master items
- Dependent Items using JSONPath and JavaScript preprocessing

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

No custom external scripts are required by the template.

Required components:

- Zabbix Server 7.4 or compatible 7.x version;
- Zabbix Agent 2 installed on the 3X-UI Linux host;
- 3X-UI Panel API token;
- network access from Zabbix Server/Proxy to the 3X-UI API URL;
- active Agent 2 checks configured, or local items changed to passive checks.

---

## Architecture

The template uses two data collection paths.

### 1. HTTP Agent API polling

```text
Zabbix Server / Zabbix Proxy
    ↓ HTTP Agent item
3X-UI Panel API
```

Used for:

- 3X-UI API availability;
- `/server/status` telemetry;
- Xray state from API;
- Xray version;
- panel version;
- update availability;
- inbounds aggregate data;
- online clients aggregate data;
- nodes aggregate data.

### 2. Local Zabbix Agent 2 checks

```text
Zabbix Agent 2 on the 3X-UI host
    ↓ native agent keys
Linux system / systemd / processes / local ports / files
```

Used for:

- `x-ui.service` state;
- panel process count;
- Xray process count;
- local panel TCP port listening;
- 3X-UI database file presence, size and modification time.

---

## Zabbix setup

### 1. Import template

1. Open **Data collection → Templates**.
2. Click **Import**.
3. Upload:

```text
templates/zabbix_3x_ui_panel_by_agent2.yaml
```

4. Use safe import options during testing:

```text
Create new: checked
Update existing: checked
Delete missing: unchecked
```

### 2. Link template to host

Create or open the Zabbix host representing the 3X-UI server:

```text
Host name: your-3x-ui-host
Groups: VPN Nodes / Linux servers
Templates: 3X-UI Panel by Zabbix agent 2
Interfaces: Zabbix Agent interface for the 3X-UI Linux host
```

### 3. Configure host macros

Set the macros on the host level. Host-level values should override template defaults.

| Macro | Type | Example | Description |
|---|---|---|---|
| `{$3XUI.API.URL}` | Text | `https://vpn.example.com:25551/secret-path/panel/api` | Base 3X-UI Panel API URL reachable from Zabbix Server/Proxy. Do not include endpoint suffix such as `/server/status`. |
| `{$3XUI.API.TOKEN}` | Secret text | `xxxxxxxxxxxxxxxx` | Bearer API token from 3X-UI. Treat it as a full administrator credential. |
| `{$3XUI.API.TIMEOUT}` | Text | `10s` | HTTP Agent timeout for API requests. |
| `{$3XUI.SERVICE.NAME}` | Text | `x-ui.service` | systemd unit name for 3X-UI. |
| `{$3XUI.PROCESS.PANEL}` | Text | `x-ui` | Process name used to count the 3X-UI panel process. |
| `{$3XUI.PROCESS.XRAY}` | Text | `xray-linux-amd64` | Command-line substring used to count the Xray process. |
| `{$3XUI.PANEL.PORT}` | Text | `26777` | Local TCP port actually listened by the `x-ui` process on the 3X-UI host. This may differ from the external API URL port. |
| `{$3XUI.DB.PATH}` | Text | `/etc/x-ui/x-ui.db` | Path to the SQLite database file. Adjust or disable DB file checks when PostgreSQL is used. |
| `{$3XUI.CPU.UTIL.HIGH}` | Text | `90` | High CPU utilization threshold in percent. |
| `{$3XUI.MEM.PUSED.HIGH}` | Text | `90` | High memory utilization threshold in percent. |
| `{$3XUI.DISK.PUSED.HIGH}` | Text | `90` | High root disk utilization threshold in percent. |
| `{$3XUI.TCP.COUNT.HIGH}` | Text | `10000` | High TCP connection count threshold. |

---

## Important: API URL vs local panel port

`{$3XUI.API.URL}` and `{$3XUI.PANEL.PORT}` are not the same thing.

### API URL

`{$3XUI.API.URL}` is used by HTTP Agent items and must be reachable from **Zabbix Server or Zabbix Proxy**:

```text
https://vpn.example.com:25551/secret-path/panel/api
```

The API URL must include the 3X-UI secret web path when the panel uses one.

Correct:

```text
https://vpn.example.com:25551/secret-path/panel/api
```

Incorrect:

```text
https://vpn.example.com:25551/panel/api
```

### Local panel port

`{$3XUI.PANEL.PORT}` is used by Zabbix Agent 2 with:

```text
net.tcp.listen[{$3XUI.PANEL.PORT}]
```

It must be the local port listened by `x-ui` on the monitored host, not necessarily the external port from the public URL.

Find it with:

```bash
ss -lntp | grep -E 'x-ui|2053|2096|26777'
```

Example output:

```text
LISTEN 0 4096 *:2096  *:* users:(("x-ui",pid=576386,fd=15))
LISTEN 0 4096 *:26777 *:* users:(("x-ui",pid=576386,fd=9))
```

In that example, test which port is the web/API panel port:

```bash
curl -k -i https://127.0.0.1:26777/secret-path/panel/api/server/status
```

Without a token, 3X-UI may return `404`. This can be normal.

Confirm with a Bearer token:

```bash
curl -k -i \
  -H "Authorization: Bearer YOUR_API_TOKEN" \
  https://127.0.0.1:26777/secret-path/panel/api/server/status
```

A valid response should be:

```text
HTTP/1.1 200 OK
{"success":true,...}
```

Then set:

```text
{$3XUI.PANEL.PORT}=26777
```

---

## Security recommendations

The 3X-UI API token should be treated as a full administrator credential.

Recommended setup:

```text
Zabbix Server / Proxy IP → allowed to access 3X-UI API
All other public IPs → denied or restricted
API token → stored only in Zabbix Secret macro
```

### Firewall example with UFW

Allow only the Zabbix Server IP to reach the 3X-UI panel/API port:

```bash
ufw allow from <ZABBIX_SERVER_IP> to any port <3XUI_PANEL_EXTERNAL_PORT> proto tcp
ufw deny <3XUI_PANEL_EXTERNAL_PORT>/tcp
```

Example:

```bash
ufw allow from 203.0.113.10 to any port 25551 proto tcp
ufw deny 25551/tcp
```

If the panel is behind Nginx, Caddy or another reverse proxy, restrict access at the reverse-proxy level or allow-list the Zabbix Server IP for the API path.

When the panel and API share the same public port, firewall rules cannot distinguish `/panel/api/*` from the web UI path. In that case, use one of these approaches:

- restrict the whole panel to trusted IPs or VPN;
- restrict `/panel/api/*` at reverse proxy level;
- run a Zabbix Proxy close to the 3X-UI server;
- use a local script/UserParameter mode in a future template variant.

---

## Active Agent 2 checks

The template uses active Agent 2 checks for local system data.

The monitored host must have Agent 2 configured with:

```ini
ServerActive=<ZABBIX_SERVER_OR_PROXY_IP>:10051
Hostname=<EXACT_ZABBIX_HOST_NAME>
```

Check the current configuration:

```bash
grep -E '^(Server|ServerActive|Hostname|HostnameItem)=' /etc/zabbix/zabbix_agent2.conf
```

Restart the agent after changes:

```bash
systemctl restart zabbix-agent2
```

Check active-check logs:

```bash
journalctl -u zabbix-agent2 -n 100 --no-pager
```

Expected signs of success:

```text
active check configuration update from [ZABBIX_SERVER:10051] is working
```

If no local agent items arrive, check:

- `Hostname` exactly matches the Zabbix host name;
- port `10051` is reachable from the 3X-UI host to Zabbix Server/Proxy;
- the host is enabled in Zabbix;
- the template is linked to the host.

---

## Xray process check

3X-UI may run Xray with a binary name such as:

```text
bin/xray-linux-amd64 -c bin/config.json
```

For this reason, the template should check Xray by command-line substring rather than only by process name:

```text
proc.num[,,,{$3XUI.PROCESS.XRAY}]
```

Recommended macro value:

```text
{$3XUI.PROCESS.XRAY}=xray-linux-amd64
```

Test locally:

```bash
zabbix_agent2 -t 'proc.num[,,,xray-linux-amd64]'
```

Example result:

```text
proc.num[,,,xray-linux-amd64] [s|2]
```

Any value greater than `0` means an Xray process is present. During a manual `zabbix_agent2 -t` test the value may be `2`, because the test command itself contains the same substring in its command line. The trigger should alert only when the value is `0`.

---

## Items

The template includes **67 items** in v1.0b.

Main item groups:

### API master items

| Item | Type | Notes |
|---|---|---|
| `3X-UI API: server status raw` | HTTP Agent | Main `/server/status` JSON source. |
| `3X-UI API: Xray versions raw` | HTTP Agent | Raw `/server/getXrayVersion` response. |
| `3X-UI API: panel update info raw` | HTTP Agent | Raw `/server/getPanelUpdateInfo` response. |
| `3X-UI API: Xray metrics state raw` | HTTP Agent | Raw `/server/xrayMetricsState` response. Endpoint is case-sensitive. |
| `3X-UI API: Xray observatory raw` | HTTP Agent | Raw `/server/xrayObservatory` response. |
| `3X-UI API: Xray result raw` | HTTP Agent | Raw `/xray/getXrayResult` response. |
| `3X-UI API: outbounds traffic raw` | HTTP Agent | Raw `/xray/getOutboundsTraffic` response. |
| `3X-UI API: inbounds slim raw` | HTTP Agent | Raw `/inbounds/list/slim` response. |
| `3X-UI API: clients online raw` | HTTP Agent | Raw `/clients/onlines` response. |
| `3X-UI API: clients active inbounds raw` | HTTP Agent | Raw `/clients/activeInbounds` response. |
| `3X-UI API: clients last online raw` | HTTP Agent | Raw `/clients/lastOnline` response. |
| `3X-UI API: nodes list raw` | HTTP Agent | Raw `/nodes/list` response. |

### API dependent items

Examples:

| Item | Notes |
|---|---|
| `3X-UI: API availability` | Returns `1` when `/server/status` returns valid successful JSON. |
| `3X-UI: panel version` | Panel version from `/server/status`. |
| `3X-UI: Xray state` | Numeric Xray state: `1=running`, `0=stopped`, `2=error`, `3=unknown`. |
| `3X-UI: Xray version` | Xray-core version reported by 3X-UI. |
| `3X-UI: CPU utilization` | CPU utilization reported by 3X-UI API. |
| `3X-UI: memory utilization` | Calculated from API memory counters. |
| `3X-UI: root disk utilization` | Calculated from API disk counters. |
| `3X-UI: network upload/download rate` | Network rate reported by 3X-UI API. |
| `3X-UI: TCP/UDP connections` | Connection counters reported by 3X-UI API. |
| `3X-UI: inbounds total/enabled/disabled` | Aggregate inbound counters from `/inbounds/list/slim`. |
| `3X-UI: online clients count` | Count from `/clients/onlines`. |

### Local Agent 2 items

| Item | Key | Notes |
|---|---|---|
| `3X-UI: service active state` | `systemd.unit.info[{$3XUI.SERVICE.NAME},ActiveState]` | Expected value: `active`. |
| `3X-UI: panel process count` | `proc.num[{$3XUI.PROCESS.PANEL}]` | Counts `x-ui` process. |
| `3X-UI: Xray process count` | `proc.num[,,,{$3XUI.PROCESS.XRAY}]` | Counts Xray by command-line substring. Value greater than `0` is OK. |
| `3X-UI: panel TCP port listening` | `net.tcp.listen[{$3XUI.PANEL.PORT}]` | Checks the local `x-ui` listen port. |
| `3X-UI: database file exists` | `vfs.file.exists[{$3XUI.DB.PATH}]` | Checks SQLite DB path. |
| `3X-UI: database file size` | `vfs.file.size[{$3XUI.DB.PATH}]` | Tracks database file size. |
| `3X-UI: database file modification time` | `vfs.file.time[{$3XUI.DB.PATH},modify]` | Tracks DB modification time. |

---

## Triggers

The template includes core triggers for availability and resource thresholds.

| Trigger | Severity | Condition |
|---|---|---|
| `3X-UI service is not active` | High | `x-ui.service` is not `active`. |
| `3X-UI panel process is not running` | High | `x-ui` process count is `0`. |
| `Xray process is not running` | High | Xray command-line match count is `0`. |
| `3X-UI API is unavailable` | High | API availability is `0` for 5 minutes. |
| `3X-UI API has no recent data` | Warning | Main API master item has no data for 5 minutes. |
| `Xray state is not running according to 3X-UI API` | High | API-reported Xray state is not `running`. |
| `CPU utilization is high` | Warning | CPU usage is above `{$3XUI.CPU.UTIL.HIGH}` for 5 minutes. |
| `Memory utilization is high` | Warning | Memory usage is above `{$3XUI.MEM.PUSED.HIGH}` for 5 minutes. |
| `Root disk utilization is high` | Warning | Root disk usage is above `{$3XUI.DISK.PUSED.HIGH}` for 5 minutes. |
| `TCP connection count is high` | Warning | TCP connection count is above `{$3XUI.TCP.COUNT.HIGH}` for 5 minutes. |
| `No enabled inbounds` | High | Inbounds exist, but none are enabled. |
| `Some inbounds are disabled` | Info | One or more inbounds are disabled. |
| `3X-UI panel update is available` | Info | Update endpoint reports a newer panel version. |
| `Xray metrics are not enabled or not configured` | Info | Xray metrics endpoint is empty or disabled. |
| `One or more 3X-UI nodes look unhealthy` | Warning | Node payload contains an unhealthy-looking status. |

### Optional local panel port trigger

The local panel TCP port trigger may be noisy on installations where:

- the public port differs from the local port;
- 3X-UI is behind a reverse proxy;
- 3X-UI listens on multiple local ports;
- API availability is already monitored externally.

For public reusable deployments, `3X-UI API is unavailable` should be considered the primary panel availability trigger. The local panel TCP port trigger is useful only when `{$3XUI.PANEL.PORT}` is set correctly for each host.

---

## Troubleshooting

### API item returns `Failed to connect to 127.0.0.1:2053`

HTTP Agent items are executed by Zabbix Server/Proxy, not by local Agent 2. Therefore `127.0.0.1` means the Zabbix Server/Proxy itself.

Set `{$3XUI.API.URL}` to the API URL reachable from Zabbix Server/Proxy:

```text
https://vpn.example.com:25551/secret-path/panel/api
```

### API endpoint returns `404` without token

This can be normal. 3X-UI may hide API endpoints from unauthenticated requests.

Test with Bearer token:

```bash
curl -i \
  -H "Authorization: Bearer YOUR_API_TOKEN" \
  https://vpn.example.com:25551/secret-path/panel/api/server/status
```

Expected result:

```text
HTTP/1.1 200 OK
{"success":true,...}
```

### Xray process trigger fires while Xray is running

Check actual command line:

```bash
ps aux | grep -i xray | grep -v grep
```

Example:

```text
root 576431 0.3 7.1 ... bin/xray-linux-amd64 -c bin/config.json
```

Set:

```text
{$3XUI.PROCESS.XRAY}=xray-linux-amd64
```

and make sure the item key uses command-line matching:

```text
proc.num[,,,{$3XUI.PROCESS.XRAY}]
```

### Panel TCP port trigger fires while the panel works

Find the real local port listened by `x-ui`:

```bash
ss -lntp | grep x-ui
```

Example:

```text
LISTEN 0 4096 *:2096  *:* users:(("x-ui",pid=576386,fd=15))
LISTEN 0 4096 *:26777 *:* users:(("x-ui",pid=576386,fd=9))
```

Test which one is the web/API panel port:

```bash
curl -k -i https://127.0.0.1:26777/secret-path/panel/api/server/status
```

Then set:

```text
{$3XUI.PANEL.PORT}=26777
```

### Only part of the items are visible in Latest data

Clear the `Latest data` filters. The template contains 67 items, but `Latest data` displays only records matching the current host/name/tag/data-state filter.

Recommended view:

```text
Monitoring → Latest data
Host = your 3X-UI host
Name = empty
Tags = empty
Data = With data
```

---

## License

This project is licensed under the MIT License. See [LICENSE](LICENSE) for details.

TL;DR: Free to use for personal and commercial projects. Attribution appreciated but not required.
