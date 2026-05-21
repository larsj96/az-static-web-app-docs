# Network Model

## Known Networks

| Network | Purpose | Notes |
| --- | --- | --- |
| `192.168.13.0/24` | Proxmox management | NIC0 on all Proxmox servers. Not reachable from VPN by design. |
| VLAN 16, `10.0.0.160/27` | Routed Proxmox management | Currently named `vmware`; should be renamed to `proxmox-mgmt`. DHCP/subnet gateway observed as `10.0.0.161/255.255.255.224`. |
| Future 10 Gbit storage network | Ceph, migration, storage replication | Dedicated RJ45 10 Gbit cards through Zyxel XGS1250-12. |
| VM/service networks | Guest workloads and application VMs | Routed/firewalled through Fortigate and switching through MikroTik. |
| Guest/IoT WLAN | Untrusted clients | SSID-to-VLAN mapping on AP/switching layer. |
| Management WLAN/VLAN | Admin access | Should be limited to trusted admin devices. |

## Physical Network Roles

| Device | Role |
| --- | --- |
| Fortigate 500D | Firewall, routing, NAT, policy, VPN/IPsec termination |
| Palo Alto PA-510 | Cabin/workstation firewall behind Starlink; managed in `terraform-palo`. |
| Zyxel XGS1250-12 | 10 Gbit Ceph/storage and migration network |
| MikroTik CRS326-24G-2S+RM | VM internet traffic and Proxmox management switching |
| Access point | Guest/IoT and management SSIDs |

## Palo Alto Cabin / Workstation Site

The Palo Alto is separate from the Mo i Rana Fortigate homelab firewall. It fronts the workstation network and currently has Starlink as the primary WAN.

Do not use app-specific policy-based routing for normal workstation internet. The Discord-over-VPS PBF experiment was unstable and caused internet issues. The sane default is:

```text
workstation internet -> Palo Alto -> Starlink WAN
selected private routes -> Palo Alto tunnel -> VPS or future IPv6 site-to-site
```

Future tunnel direction:

- keep the Palo Alto to VPS route-based tunnel for selected private/ops subnets
- test IPv6 IPsec directly between Palo Alto and Fortigate
- add WireGuard outbound as an automation fallback if needed

## Management Boundary

The old Proxmox management network should remain reachable only from trusted local/admin paths. Since `192.168.13.0/24` is intentionally not routed across the VPN, Terraform execution against Proxmox uses the routed VLAN 16 addresses instead.

Current working path:

```text
VPS/OpenVPN source 10.8.0.1
  -> IPsec to Fortigate
  -> VLAN 16 10.0.0.160/27
  -> hp1 10.0.0.162:8006
```

Terraform execution against Proxmox needs one of these:

- Run Terraform locally while physically/logically on a network that can reach Proxmox management.
- Run from the Frankfurt VPS over VLAN 16 with narrow Fortigate policy.
- Run a future self-hosted runner/agent inside a trusted automation subnet with explicit firewall policy to Proxmox APIs.
- Add a temporary VPN route only during controlled maintenance windows, then remove it.

The preferred long-term option is still a narrow automation path, not broad management routing.

## VLAN 16 Proxmox Management Plan

VLAN 16 is currently named `vmware` and should be renamed to `proxmox-mgmt`.

Subnet:

```text
10.0.0.160/27
netmask 255.255.255.224
gateway/firewall IP 10.0.0.161
usable host range 10.0.0.162-10.0.0.190
broadcast 10.0.0.191
```

Recommended host assignment:

| Host | Suggested IP |
| --- | --- |
| Proxmox node 1 / hp1 | `10.0.0.162` |
| Proxmox node 2 | `10.0.0.163` |
| Proxmox node 3 | `10.0.0.164` |
| Dell lab node, if used | `10.0.0.165` |
| Terraform/automation VM, if internal later | `10.0.0.166` |

DHCP is fine if the Fortigate uses static reservations for these hosts. Avoid changing Proxmox management addresses unexpectedly once Terraform starts depending on them.

Fortigate policy intent:

| Source | Destination | Services | Notes |
| --- | --- | --- | --- |
| Frankfurt VPS / Terraform agent path | VLAN 16 Proxmox hosts | TCP/8006 | Required for Terraform Proxmox provider. |
| Frankfurt VPS / admin source | VLAN 16 Proxmox hosts | TCP/22 | Optional. Allow only if SSH admin/provisioning is needed. |
| General VPN clients | VLAN 16 Proxmox hosts | Deny by default | Add explicit admin exceptions only. |

## Bastion VLAN

The Fortigate Terraform repo calculates this in `C:\github\terraform-fortigate\vlansinterfaces.tf`.

For `fortigate_onprem_bastion`:

```text
subnet index 3
VLAN ID 14
CIDR 10.0.0.96/27
gateway/firewall IP 10.0.0.97
usable host range 10.0.0.98-10.0.0.126
broadcast 10.0.0.127
```

Use this network for the Linux jump/bastion VM. Fortigate DHCP is enabled for this VLAN, so the VM should use DHCP rather than hardcoded cloud-init addressing.

Current DHCP observation:

```text
bastion01
VMID 9010
MAC BC:24:11:2F:B0:1F
DHCP IP 10.0.0.99
```

## K8s / Services VLAN

The Fortigate Terraform repo calculates this in `C:\github\terraform-fortigate\vlansinterfaces.tf`.

For `fortigate_onprem_k8s`:

```text
subnet index 1
VLAN ID 12
CIDR 10.0.0.32/27
gateway/firewall IP 10.0.0.33
usable host range 10.0.0.34-10.0.0.62
broadcast 10.0.0.63
```

Terraform-created service VMs on this VLAN:

| VM | VMID | Purpose | Size | DHCP observation |
| --- | --- | --- | --- | --- |
| `mkdocs` | `9020` | Self-hosted documentation VM | 2 CPU, 4 GiB RAM, 64 GiB disk | `10.0.0.37` |
| `docker1` | `9030` | Docker Compose services host | 8 CPU, 32 GiB RAM, 500 GiB disk | `10.0.0.35` |

Both were deployed with the workstation SSH key and the `ubuntu@bastion01` SSH key.

Public docs traffic now follows:

```text
Internet client -> Cloudflare Access -> Cloudflare Tunnel -> mkdocs 10.0.0.37:80
```

No inbound Starlink/Fortigate port forward is required for docs.

## hp1 VLAN 16 Configuration

hp1 has been configured with a routed Proxmox management IP:

```text
vmbr0.16@vmbr0 UP 10.0.0.162/27
```

The local VLAN gateway test succeeded:

```text
ping 10.0.0.161
3 packets transmitted, 3 received, 0% packet loss
```

The relevant Proxmox `/etc/network/interfaces` pattern is:

```text
auto vmbr0
iface vmbr0 inet static
        address 192.168.13.4/24
        gateway 192.168.13.1
        bridge-ports nic0
        bridge-stp off
        bridge-fd 0
        bridge-vlan-aware yes
        bridge-vids 2-4094

auto vmbr0.16
iface vmbr0.16 inet static
        address 10.0.0.162/27
```

There is intentionally no gateway on `vmbr0.16`; hp1 keeps one default gateway on the existing `vmbr0` management network.

## Proxmox Node Inventory

Observed node management interfaces:

| Node | Existing management | Bridge port | VLAN-aware before rollout | VLAN 16 target |
| --- | --- | --- | --- | --- |
| hp1 | `192.168.13.4/24` | `nic0` | yes, after manual change | `10.0.0.162/27` |
| hp2 | `192.168.13.3/24` | `nic0` | no | `10.0.0.163/27` |
| hp3 | `192.168.13.5/24` | `nic0` | no | `10.0.0.164/27` |
| dell1 | `192.168.13.6/24` | `nic2` | yes | optional `10.0.0.165/27` |

hp2 and hp3 match hp1's simple pre-change layout and can use the same `vmbr0.16` pattern with different addresses.

dell1 is not part of the reliable production cluster and already has custom VLAN stanzas (`jgn` VLAN 114 and `mgmt` VLAN 110), so avoid bulk-applying the HP node config to dell1. If adding VLAN 16 to dell1, preserve its existing style and add a named VLAN interface:

```text
auto proxmox_mgmt
iface proxmox_mgmt inet static
        address 10.0.0.165/27
        vlan-id 16
        vlan-raw-device vmbr0
```

There should be no gateway on this interface.

## Frankfurt VPS To VLAN 16

The VPS IPsec selector covers traffic from OpenVPN subnet `10.8.0.0/24` to homelab networks `10.0.0.0/16`.

The VPS must source homelab-bound traffic from `10.8.0.1`, not from its public IP `72.61.95.150`.

Persistent VPS route:

```text
10.0.0.0/16 via 72.61.95.254 dev eth0 src 10.8.0.1
```

Known-good tests from the VPS:

```text
ip route get 10.0.0.162
10.0.0.162 via 72.61.95.254 dev eth0 src 10.8.0.1

curl https://10.0.0.162:8006/
HTTP 200
```

## Suggested VLAN Pattern

| VLAN | Purpose | Reachability |
| --- | --- | --- |
| MGMT | Proxmox, switches, firewall management | Admin devices and automation agent only |
| INFRA | DNS, monitoring, docs, automation VMs | Internal clients, restricted admin |
| STORAGE | Ceph and migration | Proxmox nodes only |
| TRUSTED | Normal trusted clients | Internet and selected internal services |
| IOT | IoT/guest devices | Internet only, limited DNS/NTP |
| DMZ | Public-facing internal services | Cloudflare Tunnel origin access only |
