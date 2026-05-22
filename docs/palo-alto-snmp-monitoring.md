# Palo Alto SNMP Monitoring Plan

Use the existing `monitoring1` stack in `larsj96/Ansible` for Palo Alto metrics:

```text
Palo Alto SNMP -> monitoring1 Telegraf -> InfluxDB -> Grafana
```

## Required Palo Alto Settings

- Configure SNMP statistics polling under `Device > Setup > Operations > SNMP Setup`.
- Use SNMPv3 where possible: `authPriv`, SHA authentication, AES privacy.
- Enable SNMP on the interface that `monitoring1` can reach:
  - MGT interface: enable SNMP on the management interface settings.
  - Dataplane interface: create and attach an interface management profile with SNMP enabled.
- Confirm `monitoring1` (`10.0.0.38`) can reach the chosen Palo Alto address on UDP/161.
- Add firewall/security policy only if the monitoring path crosses zones. If polling the Palo Alto MGT interface directly, PAN-OS SNMP enablement is the key control; if polling through dataplane zones, policy may be required.

## Metrics Approach

Start with numeric OIDs in Telegraf so the container does not need MIB files:

- Standard SNMP system identity and uptime.
- IF-MIB interface state and octet counters, including 64-bit `ifXTable` counters.
- PAN-COMMON-MIB session counters: utilization, active sessions, active TCP, active UDP, and active ICMP.

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
