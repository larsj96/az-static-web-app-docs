# Monitoring Platform

## Goal

Build a central monitoring platform that is managed like the rest of the homelab:

- Terraform creates the monitoring VM.
- cloud-init gives it the standard Ubuntu baseline and SSH keys.
- Ansible installs Docker and deploys the monitoring stack.
- Ansible installs Telegraf on monitored servers.
- Ansible installs Filebeat on the same host group for centralized system/audit log shipping.
- Metrics go to InfluxDB.
- Grafana is the primary dashboard front end.
- Syslog/audit/application logs go to OpenSearch through Logstash.

## Target VM

Terraform stack:

```text
terraform-proxmox/proxmox-core
```

Live VM:

| VM | VMID | VLAN | Purpose | Size |
| --- | ---: | ---: | --- | --- |
| `monitoring1` | `9040` | `12` | Central monitoring stack | 8 CPU, 32 GiB RAM, 500 GiB disk |

The VM should live on the services VLAN with the other app VMs:

```text
VLAN 12 / fortigate_onprem_k8s
CIDR 10.0.0.32/27
gateway 10.0.0.33
```

Current inventory:

```text
monitoring1 ansible_host=10.0.0.38 ansible_user=ubuntu
```

Fortigate DHCP assigned `10.0.0.38`, and the QEMU guest agent reports that address.

## Metrics Pipeline

```mermaid
flowchart LR
  agents["Telegraf agents\nservers and VMs"] --> influx["InfluxDB 2.x\nbucket homelab"]
  active["Telegraf active checks\nping + x509"] --> influx
  influx --> grafana["Grafana\nprimary dashboards"]
  influx --> chrono["Chronograf"]
  influx --> kap["Kapacitor"]
```

Telegraf should be installed on all normal Terraform-created Ubuntu VMs through Ansible:

```text
[telegraf_agents]
bastion01
mkdocs01
docker1
monitoring1
media1
```

When a new Terraform VM is added, add it to `telegraf_agents` after the first boot. That keeps the VM onboarding flow simple:

1. Terraform creates the VM with cloud-init and SSH keys.
2. Fortigate DHCP reservation gives the VM a stable IP.
3. Add the host to Ansible inventory.
4. Run `ansible-playbook playbooks/monitoring.yml`.

`log_shipper_agents` is a child group of `telegraf_agents`, so all hosts in that host list receive Filebeat by default.

## Active Remote Checks

The central Telegraf container on `monitoring1` runs active checks:

| Check | Purpose |
| --- | --- |
| `inputs.ping` | Gateway, key VMs, Proxmox management, internet reachability. |
| `inputs.x509_cert` | Certificate expiry and TLS health for public/internal HTTPS endpoints. |

Initial targets:

```text
10.0.0.1
10.0.0.33
10.0.0.35
10.0.0.102
10.0.0.161
10.0.0.162
10.0.0.163
10.0.0.164
10.0.0.165
10.0.124.165
10.0.124.166
10.0.124.167
1.1.1.1
https://docs.lanilsen.com:443
https://10.0.0.162:8006
https://10.0.124.165:443
https://10.0.124.166:443
https://10.0.124.167:443
```

## Network Device Monitoring

`monitoring1` is also the collector host for infrastructure devices:

| Device class | Current targets | Current method |
| --- | --- | --- |
| Proxmox management | `10.0.0.162`, `10.0.0.163`, `10.0.0.164`, `10.0.0.165` | Ping, Proxmox web certificate check on `10.0.0.162` |
| HP iLO | `10.0.124.165`, `10.0.124.166`, `10.0.124.167` | Ping and HTTPS certificate checks |
| Fortigate | `10.0.0.33`, `10.0.0.161` | Ping |
| Palo Alto | `10.1.1.3` | SNMPv2 polling through Telegraf |

SNMP is enabled for confirmed device targets:

```yaml
monitoring_snmp_enabled: true
monitoring_snmp_community: "{{ snmp_community | default('') }}"
```

The community belongs in the ignored `group_vars/monitoring_secrets.yml`. After changing targets or secrets, rerun:

```bash
ansible-playbook playbooks/monitoring.yml
```

Palo Alto SNMP was fixed on `2026-05-22` by enabling the management-plane SNMP service and moving Telegraf from the stale `10.1.1.65` target to `10.1.1.3`. Live samples now include `PanSystem` fields such as `UpTime`, `PanSysSwVersion`, `PanSessionActiveTcp`, interface descriptions/status, fan values, and CPU load fields.

## Cloudflare Metrics

Cloudflare can be added as an extra data source in the same stack by running a Prometheus-style exporter next to Telegraf and scraping it into InfluxDB.

The implementation in this repo uses the [lablabs/cloudflare-exporter](https://github.com/lablabs/cloudflare-exporter):

- In `docker-compose.yml`, Ansible creates `cloudflare-exporter` when `monitoring_cloudflare_enabled: true`.
- Telegraf adds a `[[inputs.prometheus]]` scrape on `http://cloudflare-exporter:{{ monitoring_cloudflare_container_port }}/metrics`.
- Scraped metrics are written by Telegraf into the existing InfluxDB bucket (`homelab` by default).
- Grafana gets a provisioning template at `roles/monitoring_stack/templates/grafana-dashboard-cloudflare.json.j2`.

This design keeps your current stack intact (InfluxDB + Grafana) and avoids introducing a separate Prometheus installation.

### Enabling it

In `group_vars/monitoring.yml`:

```yaml
monitoring_cloudflare_enabled: true
monitoring_cloudflare_accounts:
  - "ACCOUNT_ID-1"
monitoring_cloudflare_zones:
  - "ZONE_ID-1"
monitoring_cloudflare_excluded_zones: []
```

In `group_vars/monitoring_secrets.yml` (vault-managed), set:

```text
cloudflare_api_token: "<token-with-analytics-permissions>"
```

Then deploy:

```bash
ansible-playbook playbooks/monitoring.yml --ask-vault-pass
```

### Token scopes

The exporter documentation requires these minimum permissions for broad zone data:

- `Zone / Analytics:Read`
- `Account / Account Analytics:Read` (if you enable account-level metrics)

Optional scopes are required for optional metric families:

- `Zone / Firewall Services:Read` (to enrich firewall metrics names)
- `Account / Account Settings:Read` (required for listing accessible accounts when worker/account metrics are enabled)
- Any additional Cloudflare product permissions your selected dashboard requires (Workers, Load Balancing, API Gateway, etc.).

For Free zones, GraphQL analytics visibility is reduced:

- Only a subset of metrics are available; the exporter marks skipped zones with `cloudflare_zones_skipped_free_tier`.

### Collection approach and alternatives

This stack uses `lablabs/cloudflare-exporter`, which is a maintained open-source Prometheus exporter backed by Cloudflare GraphQL + REST APIs.

An alternative is Cloudflare's official Worker-based exporter (`cloudflare-prometheus-exporter`), which is also supported by Cloudflare docs and pushes data in Prometheus format.

Both fit the same scrape pattern (`/metrics` to Telegraf -> InfluxDB -> Grafana), so the dashboard/provisioning layer can stay the same if you later swap collectors.

The first SNMP collection uses generic `SNMPv2-MIB` and `IF-MIB` data: system name, uptime, interface names, interface status, and in/out octets. Later we can add richer device-specific collection with HP iLO Redfish/IPMI exporter, Fortigate SNMP OIDs, or a Prometheus exporter if we want deeper dashboards.

## Logs Pipeline

ELK here means the log/audit pipeline, not metrics.

Use OpenSearch instead of Elastic licensing-sensitive packages for the first Docker-based version:

```mermaid
flowchart LR
  syslog["Syslog / audit logs"] --> logstash["Logstash\n5044 beats\n5514 syslog"]
  beats["Filebeat (system + audit)"] --> logstash
  logstash --> os["OpenSearch"]
  os --> grafana["Grafana\nlog views"]
  os --> dash["OpenSearch Dashboards"]
```

Live exposed ports on `monitoring1`:

| Port | Service |
| ---: | --- |
| `8086` | InfluxDB |
| `8888` | Chronograf |
| `3000` | Grafana |
| `9092` | Kapacitor |
| `9200` | OpenSearch API |
| `5601` | OpenSearch Dashboards |
| `5044` | Beats input |
| `5514/tcp+udp` | Syslog input |

Restrict these ports with Fortigate policy. Do not expose dashboards publicly without Cloudflare Access or VPN.

## Ansible Layout

Repo:

```text
larsj96/Ansible
```

New roles/playbook:

```text
roles/docker_host/
roles/monitoring_stack/
roles/telegraf_agent/
roles/log_shipper/
playbooks/monitoring.yml
```

Run from `bastion01`:

```bash
cd ~/ansible-homelab
git pull
ansible-playbook playbooks/monitoring.yml
```

Secrets are stored on `bastion01` in the ignored file:

```text
~/ansible-homelab/group_vars/monitoring_secrets.yml
```

Do not commit this file. It contains:

```text
influxdb_admin_password
influxdb_admin_token
opensearch_initial_admin_password
grafana_admin_password
```

Keep this file encrypted with Ansible Vault; share generated access secrets only through Vaultwarden as needed.

## Terraform And Cloud-Init Contract

Terraform should continue using the shared Noble cloud-init snippet:

```text
qemu-guest-agent
curl
ca-certificates
timezone Europe/Oslo
workstation SSH key
bastion SSH key
```

Do not pack full monitoring configuration into cloud-init. Keep cloud-init as the bootstrap layer and let Ansible own installed services.

## Next Steps

1. Add Grafana data sources and first dashboards.
2. Expand Filebeat parsing coverage as more host services are added (Docker, SSH, nginx, etc.).
3. Point Fortigate, Proxmox, and key Linux syslog toward `10.0.0.38:5514`.
4. Decide whether dashboards should stay VPN-only or be protected by Cloudflare Access.
5. Add the Dell iDRAC management IP when confirmed.
6. Add richer SNMP/Redfish/IPMI dashboards after device SNMP is enabled.

## Deployment Status

Deployed on `2026-05-21`:

| Component | Status |
| --- | --- |
| Terraform VM `monitoring1` | Created and running |
| Cloud-init snippet | Uploaded to `local:snippets/terraform-noble-base-cloud-config.yaml` |
| Docker monitoring stack | Running |
| InfluxDB health | HTTP `200` locally |
| Grafana login | HTTP `200` locally |
| OpenSearch API | HTTP `200` locally |
| OpenSearch Dashboards | HTTP `302` locally |
| Telegraf on `bastion01` | Active |
| Telegraf on `mkdocs01` | Active |
| Telegraf on `docker1` | Active |
| Telegraf on `monitoring1` | Active |
| Telegraf on `media1` | Active |
| Grafana InfluxDB datasource | Provisioned |
| Grafana OpenSearch datasource | Provisioned |
| iLO/Fortigate/Proxmox active checks | Configured on central Telegraf |
| SNMP collection | Active for confirmed iLO, Fortigate, and Palo Alto targets |

## Grafana Public Access Option

Grafana should be the friendly front door for dashboards. Keep it internal first:

```text
http://10.0.0.38:3000
```

When ready, expose it the same way as docs:

```text
grafana.lanilsen.com
  -> Cloudflare Access
  -> Cloudflare Tunnel
  -> http://10.0.0.38:3000
```

Use Cloudflare Access for edge authentication, then keep Grafana's own admin password strong for local or VPN access.
