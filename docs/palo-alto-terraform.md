# Palo Alto Terraform

The Palo Alto PA-510 at the cabin/workstation site is managed separately from the Fortigate homelab firewall. It sits behind Starlink and is the firewall in front of the workstation network.

## Repository

```text
C:\Users\lars_\github\terraform-palo
```

Remote:

```text
git@github.com:larsj96/terraform-palo.git
```

## State Model

Use Cloudflare R2, not HCP Terraform, for the Palo Alto stacks.

```text
bucket: lanilsen-terraform-state

terraform-palo/core-bootstrap/terraform.tfstate
terraform-palo/network-base/terraform.tfstate
terraform-palo/vpn-vps/terraform.tfstate
```

Each stack has an ignored backend config copied from:

```text
backend.r2.tfbackend.example
```

Run pattern:

```bash
export AWS_ACCESS_KEY_ID="..."
export AWS_SECRET_ACCESS_KEY="..."

cd live/homelab/network-base
cp backend.r2.tfbackend.example backend.r2.tfbackend
terraform init -backend-config=backend.r2.tfbackend
terraform plan
```

Do not commit `terraform.tfvars`, `*.auto.tfvars`, `.tfbackend`, state files, plans, API keys, passwords, or pre-shared keys.

## Stack Split

| Stack | Purpose |
| --- | --- |
| `core-bootstrap` | Minimal PA-510 bootstrap: hostname, timezone, NTP, and provider connectivity. |
| `network-base` | Zones, WAN/LAN interfaces, VLAN subinterfaces, DHCP, virtual router intent, NAT, and broad security policy. |
| `vpn-vps` | Route-based IPsec tunnel from Palo Alto to the Frankfurt VPS. |

`network-base` owns the main virtual router. If `vpn-vps` creates `tunnel.10`, then `network-base` must include that interface in `virtual_router_extra_interfaces` before the next `network-base` apply.

## Policy-Based Routing Lesson

Do not route Discord, voice, or general interactive internet traffic over the VPS tunnel with app-specific PBF rules.

That design was unstable because it mixed normal client internet use with a fragile Starlink/VPN path. The workstation should use normal Starlink internet egress by default.

Allowed intent:

- route only known VPS hub/private subnets over the tunnel
- use temporary PBF only for short, deliberate tests
- keep `test_pbf_enabled = false` unless actively testing

Avoid:

- `discord.tf`
- Discord FQDN/application PBF
- forcing arbitrary UDP voice/video traffic through the VPS tunnel
- full internet hairpin through the VPS unless explicitly designing for that

## Current VPN Direction

The Palo Alto to VPS tunnel can stay as a site/control path, but it should not be the only long-term remote-access design.

Preferred direction:

1. Keep Palo Alto route-based IPsec to the Germany VPS for selected private subnets.
2. Test IPv6 IPsec between Palo Alto and Fortigate because Starlink supports IPv6 and avoids IPv4 CGNAT pain.
3. Add WireGuard outbound from a homelab Linux VM to the VPS as a boring automation fallback.

## Execution Location

Run Palo Alto Terraform from a host that can reach the PA-510 management IP.

Likely options:

- workstation/WSL behind the Palo Alto
- a local runner inside the cabin/workstation network
- a future VPN path if it can reliably reach the PA-510 API

The Frankfurt VPS should not be assumed to reach the Palo Alto management API unless the Palo Alto VPN path is proven up.
