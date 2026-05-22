# Palo Alto SNMP Monitoring Plan

Use the existing `monitoring1` stack in `larsj96/Ansible` for Palo Alto metrics:

```text
Palo Alto SNMP -> monitoring1 Telegraf -> InfluxDB -> Grafana
```

Live polling target:

```text
10.1.1.3  pa510-homelab-ljn in-band/API interface
```

The dedicated management address in the running Palo config is `10.1.1.2/27`, but SNMP is currently answering on `10.1.1.3`, which is also reachable for HTTPS/API work.

On `2026-05-22`, SNMP was explicitly enabled on the Palo management-plane service config and committed through the XML API. A live test from `monitoring1` succeeded:

```text
snmpget -v2c 10.1.1.3 1.3.6.1.2.1.1.3.0
Timeticks returned successfully
```

The previous target `10.1.1.65` is not reachable and should not be used until it is intentionally restored.

## Terraform-Managed Palo Alto Settings

The Palo Alto in-band management profile is managed in:

```text
terraform-palo/live/homelab/network-base
```

Terraform now owns the dataplane/interface exposure needed for SNMP polling:

- `panos_interface_management_profile.inband`
- `snmp = true`
- optional `permitted_ips` allow-list
- attachment to inside interfaces where `allow_inband_mgmt = true`

Current intended allow-list:

```text
10.0.0.38/32   monitoring1 Telegraf SNMP polling
10.0.0.100/32  mgmt1 workbench
10.0.0.102/32  bastion01
10.1.1.0/27    local infra/admin VLAN
```

The Terraform variables are:

```hcl
management_profile_services = ["https", "ssh", "ping", "snmp"]
management_profile_permitted_ips = [
  "10.0.0.38/32",
  "10.0.0.100/32",
  "10.0.0.102/32",
  "10.1.1.0/27",
]
```

Run from a host that can reach the Palo Alto management API:

```bash
cd /mnt/c/github/terraform-palo/live/homelab/network-base
terraform plan
terraform apply
```

## Remaining Palo Alto Settings

- Configure SNMP statistics polling under `Device > Setup > Operations > SNMP Setup` if it is not already present.
- Use SNMPv3 where possible: `authPriv`, SHA authentication, AES privacy.
- Prefer polling the Terraform-managed in-band/dataplane interface instead of the dedicated MGT interface when possible.
- Confirm `monitoring1` (`10.0.0.38`) can reach `10.1.1.65` on UDP/161.
- Add firewall/security policy only if the monitoring path crosses zones. If polling the Palo Alto MGT interface directly, PAN-OS SNMP enablement is the key control; if polling through dataplane zones, policy may be required.

Provider note: the PAN-OS Terraform provider exposes interface management profile SNMP cleanly. A first-class provider resource for the global SNMP setup/user was not available in the local provider schema used here, so that part remains a separate follow-up unless we manage it through a lower-level API/tool.

## Metrics Approach

Start with numeric OIDs in Telegraf so the container does not need MIB files:

- Standard SNMP system identity and uptime.
- IF-MIB interface state and octet counters, including 64-bit `ifXTable` counters.
- PAN-COMMON-MIB session counters: utilization, active sessions, active TCP, active UDP, and active ICMP.

Live verified fields include `PanSystem` uptime, PAN-OS software version, HA state, active TCP/UDP/ICMP sessions, interface descriptions/status/octet counters, host CPU load indexes, and fan readings.

The route from `monitoring1` to the Palo site depends on the Fortigate-to-VPS and Palo-to-VPS hub path. `monitoring1` sends traffic to the Fortigate gateway, Fortigate routes `10.1.0.0/16` over `to-hostinger`, and the VPS forwards it into the Palo IPsec selector.

Important selector rule: the Palo-to-VPS stack must include both `10.8.0.0/24` and `10.0.0.0/16` as remote subnets. `10.8.0.0/24` is for VPS/OpenVPN-originated traffic; `10.0.0.0/16` is for Fortigate-side monitoring and log hosts crossing the VPS hub toward Palo. Avoid overlapping IPv4 and IPv6 Palo tunnels with the same inner IPv4 selectors unless xfrm policy preference is controlled.

Next dashboard candidates after the first walk:

- GlobalProtect active tunnels and utilization from the `panGlobalProtect` object group.
- HA local/peer state.
- Dataplane and management-plane CPU.
- Hardware health if the deployed Palo Alto platform exposes the relevant entity MIB objects.

## Repo Scaffold

The Ansible monitoring role contains:

- SNMPv3-capable Telegraf config generation.
- A disabled-by-default Palo Alto target example.
- Grafana dashboard provisioning.
- A placeholder `Palo Alto SNMP` dashboard backed by InfluxDB Flux queries.

Secrets remain outside git in the vaulted monitoring secrets file.
