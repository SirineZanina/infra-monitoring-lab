# Infrastructure Monitoring Lab

A self-hosted monitoring stack for Linux hosts and network devices: **Prometheus** collects metrics,
**Grafana** shows them, and **Alertmanager** sends alerts to **Slack**. Linux hosts are monitored with
**node_exporter**; routers and switches are polled over **SNMPv3** with **snmp_exporter**.
The network side targets the routers of my [MPLS L3VPN lab](https://github.com/SirineZanina/mpls-l3vpn-lab) in GNS3.

## Architecture

```
  Linux host ──── node_exporter :9100 ───┐
                                          │  scrape every 15s      ┌──────────────┐     ┌─────────┐
  GNS3 routers ── SNMPv3 ── snmp_exporter │ ─────────────────────► │  Prometheus  │ ──► │ Grafana │
  (PE1, PE2)       (authPriv)   :9116    ─┘                        │  :9090       │     │  :3000  │
                                                                   │  alert rules │     └─────────┘
                                                                   └──────┬───────┘
                                                                          │ firing alerts
                                                                   ┌──────▼───────┐
                                                                   │ Alertmanager │ ──► Slack #infra-alerts
                                                                   │  :9093       │
                                                                   └──────────────┘
```

| Service | Image | Port | Role |
|---|---|---|---|
| Prometheus | `prom/prometheus:v2.54.1` | 9090 | Metric collection, storage (15 days), alert rule evaluation |
| Alertmanager | `prom/alertmanager:v0.27.0` | 9093 | Grouping, deduplication, inhibition and Slack routing of alerts |
| Grafana | `grafana/grafana-oss:11.2.0` | 3000 | Dashboards (provisioned automatically) |
| node_exporter | `prom/node-exporter:v1.8.2` | 9100 | Host CPU, memory, disk and network metrics |
| snmp_exporter | `prom/snmp-exporter:v0.26.0` | 9116 | SNMPv3 → Prometheus bridge for network devices |

## Alert rules

Defined in [`prometheus/rules/alerts.yml`](prometheus/rules/alerts.yml):

| Alert | Condition | Severity |
|---|---|---|
| `TargetDown` | A scrape target unreachable for 2 min | critical |
| `HostHighCpu` | CPU above 85% for 5 min | warning |
| `HostLowMemory` | Less than 10% memory available for 5 min | warning |
| `HostDiskAlmostFull` | Less than 10% disk space left | warning |
| `HostDiskWillFillIn24h` | `predict_linear` over the last 6h says the disk fills within 24h | warning |
| `InterfaceDown` | SNMP `ifOperStatus` = down for 1 min | critical |
| `InterfaceErrors` | More than 1 interface error per second for 5 min | warning |
| `InterfaceHighTraffic` | In + out traffic above 50 Mbit/s for 10 min | info (dashboard only) |

Alertmanager groups alerts per alert name and device, sends resolved notifications, and **inhibits**
`InterfaceDown` when the whole device is already `TargetDown`, so one outage does not produce a flood of messages.

## Dashboard

`Infrastructure Overview` (provisioned from [`grafana/dashboards/`](grafana/dashboards/)) shows:
targets up, firing alerts, CPU / memory / disk gauges, CPU over time, disk free space with its 24h forecast,
SNMP interface traffic in/out, interface errors, and an interface status table.

## Getting started

```bash
cp .env.example .env                                   # set GRAFANA_ADMIN_PASSWORD
echo 'https://hooks.slack.com/services/...' > secrets/slack_webhook_url
docker compose up -d
```

- Prometheus: http://localhost:9090 (Status → Targets, Alerts)
- Alertmanager: http://localhost:9093
- Grafana: http://localhost:3000 → *Infrastructure* → *Infrastructure Overview*

### Adding the GNS3 routers (SNMPv3)

1. In GNS3, connect each router to a **Cloud** node bound to `virbr0` (192.168.122.0/24), so the Docker host can reach it.
2. Apply [`gns3/router-snmpv3.cfg`](gns3/router-snmpv3.cfg) on each router: a read-only view, an `authPriv` SNMPv3 user
   (SHA + AES-128) and an ACL that only lets the monitoring host query SNMP.
3. Put the same passwords in [`snmp_exporter/snmp.yml`](snmp_exporter/snmp.yml) (`CHANGE_ME_…`), and the router IPs in
   the `snmp` job of [`prometheus/prometheus.yml`](prometheus/prometheus.yml).
4. Reload Prometheus: `curl -X POST http://localhost:9090/-/reload`.

### Testing the alerts

```bash
# CPU: load all cores for a few minutes
docker run --rm -it alpine sh -c 'for i in $(seq $(nproc)); do yes > /dev/null & done; sleep 600'

# Interface down: on a router
#   configure terminal / interface Serial2/1 / shutdown
```

## Security choices

- **SNMPv3 `authPriv` only** (authentication + encryption); no SNMPv1/v2c community strings.
- SNMP restricted to the monitoring host with an ACL, and a read-only view.
- Secrets (Grafana password, Slack webhook) stay out of Git: `.env` and `secrets/` are git-ignored.
- Grafana sign-up disabled.

## Repository layout

```
docker-compose.yml                    Whole stack
prometheus/prometheus.yml             Scrape jobs (node, SNMP)
prometheus/rules/alerts.yml           Alert rules
alertmanager/alertmanager.yml         Slack routing, grouping, inhibition
snmp_exporter/snmp.yml                SNMPv3 auth + IF-MIB module
grafana/provisioning/                 Datasource and dashboard provisioning
grafana/dashboards/                   Infrastructure Overview dashboard
gns3/router-snmpv3.cfg                Cisco IOS SNMPv3 configuration
```
