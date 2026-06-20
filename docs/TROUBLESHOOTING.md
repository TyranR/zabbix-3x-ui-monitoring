# 3X-UI Zabbix Template Troubleshooting

Troubleshooting guide for **3X-UI Panel by Zabbix agent 2** v1.11.2.

---

## API item fails with `Failed to connect to 127.0.0.1:2053`

HTTP Agent items run from Zabbix Server/Proxy, not from local Agent 2.

Set `{$3XUI.API.URL}` to a URL reachable from Zabbix Server/Proxy:

```text
https://vpn.example.com:25551/secret-path/panel/api
```

Do not use `127.0.0.1` unless Zabbix Server/Proxy runs on the same host as 3X-UI.

---

## API endpoint returns `404` without token

This can be normal. 3X-UI may hide API endpoints from unauthenticated requests.

Test with Bearer token:

```bash
curl -i   -H "Authorization: Bearer YOUR_API_TOKEN"   https://vpn.example.com:25551/secret-path/panel/api/server/status
```

Expected:

```text
HTTP/1.1 200 OK
{"success":true,...}
```

---

## Active Agent 2 items do not arrive

Check Agent 2 configuration:

```bash
grep -E '^(Server|ServerActive|Hostname|HostnameItem)=' /etc/zabbix/zabbix_agent2.conf
```

Required conditions:

```text
ServerActive points to Zabbix Server/Proxy
Hostname exactly equals the Zabbix host name
3X-UI host can reach Zabbix Server/Proxy on TCP 10051
Host and template are enabled
```

Restart Agent 2:

```bash
systemctl restart zabbix-agent2
```

Check logs:

```bash
journalctl -u zabbix-agent2 -n 100 --no-pager
```

---

## Xray process trigger fires while Xray is running

Check the real command line:

```bash
ps aux | grep -i xray | grep -v grep
```

Example:

```text
bin/xray-linux-amd64 -c bin/config.json
```

Set:

```text
{$3XUI.PROCESS.XRAY}=xray-linux-amd64
```

The item uses command-line substring matching:

```text
proc.num[,,,{$3XUI.PROCESS.XRAY}]
```

Any value greater than `0` means Xray is present.

---

## Panel TCP port trigger fires while panel/API works

The local panel port may differ from the external API URL port.

Find local `x-ui` ports:

```bash
ss -lntp | grep x-ui
```

If you do not need local panel port alerts, keep them disabled:

```text
{$3XUI.PANEL.PORT.CHECK}=0
```

API availability is usually the better panel health signal.

---

## Inbound TCP/UDP port trigger fires on central node host

Central 3X-UI panels can expose inbounds belonging to remote nodes. Agent 2 checks only local sockets.

For central/admin panel hosts, use:

```text
{$3XUI.NODE.LLD.ENABLED}=1
{$3XUI.INBOUND.PORT.CHECK}=0
```

For standalone hosts where API inbounds are local, you may enable:

```text
{$3XUI.INBOUND.PORT.CHECK}=1
```

---

## Duplicate `net.tcp.listen[443]` or `net.udp.listen[443]`

Use the current template version with unique inbound port discovery rules and import with:

```text
Delete missing: checked
```

Older per-inbound port prototypes may remain if `Delete missing` is not enabled.

---

## Client or Node LLD creates no items

Check the relevant macro:

```text
{$3XUI.CLIENT.LLD.ENABLED}=1
{$3XUI.NODE.LLD.ENABLED}=1
```

Run discovery manually:

```text
Data collection → Hosts → <host> → Discovery → Execute now
```

---

## Logs return `404`

The log API endpoints must use POST.

Current template versions use:

```text
POST /panel/api/server/logs/{$3XUI.LOGS.COUNT}
POST /panel/api/server/xraylogs/{$3XUI.LOGS.COUNT}
Body: {}
```

Use v1.9.4 or newer.

---

## Log preprocessing fails with `unterminated regexp`

Use v1.11.2 or newer. Log preprocessing no longer uses regular expressions.

---

## Logs are noisy

Keep log triggers disabled:

```text
{$3XUI.LOGS.CHECK}=0
```

---

## Web UI scenario is disabled

This is expected.

To enable:

```text
{$3XUI.WEB.URL}=https://vpn.example.com:25551/secret-path/
{$3XUI.WEB.CHECK}=1
```

Then enable:

```text
Data collection → Hosts/Templates → Web → 3X-UI Panel Web UI Access
```

---

## TLS items are disabled

This is expected.

Set:

```text
{$3XUI.TLS.HOST}=vpn.example.com
{$3XUI.TLS.PORT}=443
{$3XUI.TLS.CHECK}=1
```

Then enable TLS items.

---

## Backup items are disabled

This is expected.

Set:

```text
{$3XUI.BACKUP.PATH}=/path/to/backups
{$3XUI.BACKUP.CHECK}=1
```

Then enable the backup items manually.

---

## General smoke test after import

Check:

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

## Dashboard widgets show `No data`

Some dashboard widgets are intentionally connected to optional checks.

Check whether the related module is configured and enabled:

```text
Web UI  → Web scenario enabled, {$3XUI.WEB.URL} configured
TLS     → TLS items enabled, {$3XUI.TLS.HOST} and {$3XUI.TLS.PORT} configured
Backup  → backup items enabled, {$3XUI.BACKUP.PATH} configured
Nodes   → {$3XUI.NODE.LLD.ENABLED}=1 on central/admin panels only
Clients → {$3XUI.CLIENT.LLD.ENABLED}=1 if per-client discovery is needed
```

Dashboard `No data` is expected for optional modules that are not enabled.

---

## Web UI response graph has no data

The Web UI graph uses Zabbix web scenario items:

```text
web.test.in[3X-UI Panel Web UI Access,,bps]
web.test.time[3X-UI Panel Web UI Access,Check 3X-UI Web UI,resp]
```

Check:

```text
Data collection → Hosts/Templates → Web
3X-UI Panel Web UI Access = Enabled
```

Then wait for at least one web scenario interval.

Also check Latest data for generated web items:

```text
Download speed for scenario "3X-UI Panel Web UI Access"
Response time for step "Check 3X-UI Web UI"
```

---

## Database health graph shows milliseconds in modification age

Use v1.11.2 or newer.

The trigger item remains seconds-based:

```text
3xui.db.file.age
```

The graph/dashboard item uses whole minutes:

```text
3xui.db.file.age.minutes
```

This avoids millisecond noise in graph legends.

---

## Traffic total is zero while upload/download have values

Some 3X-UI API responses expose `up` and `down`, but not a ready-made `total` field.

Current template versions calculate:

```text
total = up + down
```

for inbound and client total traffic items.

If total stays zero, check the raw API item and dependent item preprocessing.

---

## Hosts have different item/graph counts

This is expected when LLD is enabled.

Different hosts may discover different numbers of:

```text
inbounds
clients
nodes
outbounds
TCP/UDP ports
```

A central panel with remote nodes normally has more objects than standalone hosts.

Compare these values in Latest data:

```text
3X-UI: Inbounds total
3X-UI: Clients total
3X-UI: Nodes total
3X-UI: Xray outbounds count
```

Also compare host macros:

```text
{$3XUI.CLIENT.LLD.ENABLED}
{$3XUI.NODE.LLD.ENABLED}
{$3XUI.OUTBOUND.LLD.ENABLED}
```

---

## One graph with all discovered inbounds is missing

Zabbix classic template graphs cannot dynamically put all LLD-discovered objects as separate lines on one reusable graph.

Graph prototypes generate one graph per discovered object.

Use aggregate template graphs for sums, or create a manual host graph/Grafana dashboard when you need all individual objects on one canvas.
