# Monitoring Platform

## Goal

Build a central monitoring platform that is managed like the rest of the homelab:

- Terraform creates the monitoring VM.
- cloud-init gives it the standard Ubuntu baseline and SSH keys.
- Ansible installs Docker and deploys the monitoring stack.
- Ansible installs Telegraf on monitored servers.
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
  pve["Proxmox API polling\nhp1 hp2 hp3"] --> influx
  snmp["SNMP polling\niLO Fortigate Palo Alto"] --> influx
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
| Proxmox management | `10.0.0.162`, `10.0.0.163`, `10.0.0.164`, `10.0.0.165` | Telegraf Proxmox API input, ping, Proxmox web certificate check on `10.0.0.162` |
| HP iLO | `10.0.124.165`, `10.0.124.163` | SNMPv2 polling through Telegraf |
| HP iLO pending | `10.0.124.164` | Currently times out on UDP/161; enable after hp1 iLO SNMP is fixed |
| Fortigate | `10.0.0.33` | SNMPv2 polling through Telegraf |
| Palo Alto | `10.1.1.65` | SNMPv2 polling through Telegraf over the Palo/Fortigate hub route |

SNMP is enabled in Ansible and uses the ignored monitoring secrets file for the community:

```yaml
monitoring_snmp_enabled: true
monitoring_snmp_community: "{{ snmp_community | default('') }}"
```

After changing targets or secrets, rerun the monitoring stack from `bastion01`:

```bash
ansible-playbook playbooks/monitoring.yml
```

Current SNMP collection includes generic system identity/uptime, interface counters, HP iLO health OIDs, Fortigate system/interface OIDs, and Palo Alto PAN-OS session/interface/CPU/fan OIDs.

Proxmox metrics are currently collected by Telegraf through the Proxmox API. The measurement is `proxmox`, and live samples include VM status, CPU load, uptime, disk totals, disk used, and disk used percentage. This is the working source of truth today and feeds the `Proxmox VE Homelab` Grafana dashboard.

Proxmox's native Datacenter metric server is not configured yet. The currently pinned `bpg/proxmox` provider in `terraform-proxmox/proxmox-core` does not support that resource, so the repo keeps `metrics.tf.example` as a future-provider-upgrade note rather than active Terraform.

## Logs Pipeline

ELK here means the log/audit pipeline, not metrics.

Use OpenSearch instead of Elastic licensing-sensitive packages for the first Docker-based version:

```mermaid
flowchart LR
  syslog["Syslog / audit logs"] --> logstash["Logstash\n5044 beats\n5514 syslog"]
  beats["Future Filebeat/Auditbeat"] --> logstash
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

Later, move this into Ansible Vault when the workflow settles.

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
2. Add Filebeat/Auditbeat or syslog forwarding after metrics are stable.
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
| Proxmox API metrics | Active through Telegraf for hp1, hp2, hp3, and dell1 |
| iLO/Fortigate/Palo Alto SNMP | Active for confirmed targets |
| hp1 iLO SNMP | Pending; UDP/161 currently times out |

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
