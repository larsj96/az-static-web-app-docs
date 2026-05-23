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
| Zyxel XGS1250-12 | 10 Gbit Ceph/storage and migration network |
| MikroTik CRS326-24G-2S+RM | VM internet traffic and Proxmox management switching |
| Access point | Guest/IoT and management SSIDs |

## Management Boundary

The Proxmox management network should remain reachable only from trusted local/admin paths. Since `192.168.13.0/24` is intentionally not routed across the VPN, Terraform execution against Proxmox needs one of these:

- Run Terraform locally while physically/logically on a network that can reach Proxmox management.
- Run a Terraform Cloud Agent VM inside a trusted management or automation subnet with explicit firewall policy to Proxmox APIs.
- Use the Frankfurt VPS agent only after VLAN 16 is routed over the IPsec/VPN path with narrow Fortigate policy.
- Add a temporary VPN route only during controlled maintenance windows, then remove it.

The preferred long-term option is a Terraform Cloud Agent VM with narrow firewall rules.

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
| `auth1` | `9080` | Authentik SSO/MFA identity provider | 4 CPU, 8 GiB RAM, 120 GiB disk | `10.0.0.36` |

Both were deployed with the workstation SSH key and the `ubuntu@bastion01` SSH key.

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

## Direct IPv6 Palo Alto To Fortigate VPN

The VPS path is useful as an operations hub, but it should not be the only private path between the cabin Palo Alto side and the Mo i Rana Fortigate side.

Live direct design:

```text
Palo Alto WAN global IPv6 <-> Fortigate WAN global IPv6
IKEv2/IPsec route-based tunnel
Protected IPv4 networks:
  Palo Alto side: 10.1.0.0/16
  Fortigate side: 10.0.0.0/16
```

IPv6 is the tunnel transport. The protected lab LANs stay IPv4 for now, which keeps routing, policies, and existing Terraform objects simpler.

Current live endpoints:

```text
Palo Alto: gp.lanilsen.com / 2a0d:3341:bb9c:af01::443
Fortigate: port9 / 2a0d:3341:bb00:6320:a5b:eff:feca:b2e9
Palo tunnel interface: tunnel.20
Palo VPN zone: vpn-fortigate
Fortigate phase1: palo-ipv6
Fortigate phase2: p2-v4-10-0-0-0-16-to-10-1-0-0-16
```

The Palo Alto route preference is:

```text
10.0.0.0/16 -> tunnel.20 direct Fortigate IPv6 IPsec, metric 5
10.0.0.0/16 -> tunnel.10 Frankfurt VPS fallback, metric 50
10.8.0.0/24 -> tunnel.10 Frankfurt VPS hub, metric 10
```

The Fortigate route preference is:

```text
10.1.0.0/16 -> palo-ipv6 direct Palo Alto IPv6 IPsec, distance 5
10.1.0.0/16 -> to-hostinger Frankfurt VPS fallback, distance 50
10.8.0.0/24 -> to-hostinger Frankfurt VPS hub, distance 10
```

NAT-T is required for the direct Starlink IPv6 data plane:

```text
Palo Alto ike-gw-fortigate-ipv6 -> NAT traversal enabled
Fortigate palo-ipv6 -> nattraversal forced
```

Without UDP encapsulation, IKE and IPsec SAs came up, but WSL/PC traffic was one-way: Palo allowed `10.1.1.5 -> 10.0.0.0/16` and sent it out `tunnel.20`, but no useful return traffic was decapsulated. After enabling NAT-T, Palo showed `<natt>True</natt>` and real decap counters.

Known-good validation from the Frankfurt VPS on `2026-05-23`:

```text
https://gp.lanilsen.com/ over IPv6 -> HTTP 302
Palo IKE gateway ike-gw-fortigate-ipv6 -> established
Palo IPsec tunnel ipsec-fortigate-ipv6:pid-v4-00-00 -> established
Fortigate phase1 palo-ipv6 -> up
Fortigate phase2 p2-v4-10-0-0-0-16-to-10-1-0-0-16 -> up
Palo route 10.0.0.0/16 -> tunnel.20 selected
Fortigate route 10.1.0.0/16 -> palo-ipv6 distance 5, VPS fallback distance 50
WSL TCP checks:
  10.0.0.37:22 -> open
  10.0.0.35:22 -> open
  10.0.0.33:444 -> open
  10.0.0.162:8006 -> open
WSL HTTP checks:
  https://10.0.0.33:444/ -> HTTP 302
  https://10.0.0.162:8006/ -> HTTP 200
```

Traffic counters stay at zero until real inside traffic crosses the tunnel. Test this from a Palo-side client to a Fortigate-side host, or from a Fortigate-side host to a Palo-side host. Testing from the VPS does not validate the direct tunnel because the VPS is a third path.

Before applying this, verify both firewalls have global IPv6 addresses, not only `fe80::` link-local addresses:

```text
Palo Alto:
  show interface ethernet1/1
  ping source <palo-global-ipv6> host <fortigate-global-ipv6>

Fortigate:
  get system interface physical
  execute ping6-options source <fortigate-global-ipv6>
  execute ping6 <palo-global-ipv6>
```

Terraform scaffolds:

```text
C:\Users\lars_\github\terraform-palo\live\homelab\vpn-fortigate-ipv6
C:\github\terraform-fortigate\live\homelab\vpn-palo-ipv6
```

The Palo Alto stack creates `tunnel.20`, zone `vpn-fortigate`, IKE/IPsec profiles, IPv4 proxy IDs, and bidirectional security policy. `network-base` owns the virtual router interface membership and route preference.

The Fortigate stack creates a route-based IPsec interface, phase2 selectors, IPv4 address objects, static route to `10.1.0.0/16`, and bidirectional policies.

Keep the existing VPS VPN as a backup and remote operations path. The direct IPv6 VPN is the intended low-latency site-to-site path, with the VPS path retained as fallback.

Provider limitation:

The Fortigate can model separate IPv4 and IPv6 phase2 selectors, which is the pattern documented by Weberblog for Palo Alto to Fortigate IPv6 IPsec. The current PAN-OS Terraform provider rejected IPv6 proxy IDs in the same resource, so the live Terraform configuration only manages IPv4 protected networks over IPv6 transport. If IPv6 inside the tunnel is needed later, verify provider support or use a small PAN-OS API helper for the IPv6 proxy ID rather than hand-editing it permanently in the GUI.

## Palo Alto GlobalProtect Over Starlink IPv6

GlobalProtect remote access is published directly on the Palo Alto over IPv6:

```text
gp.lanilsen.com AAAA 2a0d:3341:bb9c:af01::443
listener interface loopback.12
listener zone gp-public
WAN source zone wan-starlink
client tunnel tunnel.12
client pool 172.31.250.0/24
client route 0.0.0.0/0
```

Cloudflare must stay DNS-only for this record. Do not proxy it through Cloudflare, because GlobalProtect needs a direct connection to the firewall.

Traffic model:

```text
Internet client
  -> gp.lanilsen.com over IPv6
  -> Palo Alto loopback.12
  -> GlobalProtect client pool 172.31.250.0/24
```

Internet egress:

```text
172.31.250.0/24
  -> Palo rule gp-clients-to-internet
  -> Palo NAT nat-gp-clients-to-starlink
  -> ethernet1/1 / wan-starlink
```

Mo i Rana access through the VPS hub:

```text
172.31.250.0/24
  -> Palo rule gp-clients-to-vps-hub
  -> Palo NAT nat-gp-clients-to-vps-hub, translated source 10.1.255.250
  -> Palo tunnel.10 / vpn-vps
  -> Frankfurt VPS
  -> VPS SNAT to 10.8.0.1
  -> Fortigate IPsec
  -> 10.0.0.0/16
```

This NAT chain is there because the established IPsec selectors are not `172.31.250.0/24` aware. If a client can connect to GlobalProtect and Palo logs show allowed `172.31.250.1 -> 10.0.0.x` sessions that age out as `app=incomplete`, check `nat-gp-clients-to-vps-hub` and the VPS SNAT rules first.

Validation from the Frankfurt VPS:

```text
https://gp.lanilsen.com/ -> 302 /global-protect/login.esp
```

Local workstation tests to the public IPv6 can fail because they hairpin back to the same firewall. Use the VPS or another external IPv6 network for public reachability checks.

## Fortigate Remote Access VPN Over IPv6

This is the planned Mo i Rana-side equivalent to Palo GlobalProtect. It should be a separate remote-access stack, not part of the Palo-to-Fortigate site-to-site tunnel.

Recommended intent:

```text
fortigate-vpn.lanilsen.com AAAA <Fortigate port9 IPv6>
Cloudflare DNS mode: DNS-only
Remote clients terminate directly on Fortigate over IPv6
Client access: selected Mo i Rana networks first, full tunnel only if explicitly needed
Authentication: MFA-backed identity provider or strong Fortigate local users as temporary bootstrap
```

Use Terraform for the Fortigate portal, address pools, policies, and Cloudflare DNS record. Keep the public VPN DNS record unproxied; Cloudflare Tunnel/Access is for HTTP applications, not generic VPN control/data planes.

Design choice to make before implementation:

```text
IKEv2/IPsec remote access:
  cleaner VPN design, good for always-on clients, but client setup can be more fiddly

SSL-VPN:
  easier with FortiClient and browser-style portal, but should be locked down hard because SSL-VPN portals have a larger exposed attack surface
```

Either way, do not expose broad management networks by default. Start with an admin group and explicit routes/policies, then add access deliberately.

## Suggested VLAN Pattern

| VLAN | Purpose | Reachability |
| --- | --- | --- |
| MGMT | Proxmox, switches, firewall management | Admin devices and automation agent only |
| INFRA | DNS, monitoring, docs, automation VMs | Internal clients, restricted admin |
| STORAGE | Ceph and migration | Proxmox nodes only |
| TRUSTED | Normal trusted clients | Internet and selected internal services |
| IOT | IoT/guest devices | Internet only, limited DNS/NTP |
| DMZ | Public-facing internal services | Cloudflare Tunnel origin access only |
