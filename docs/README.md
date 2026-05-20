# Homelab Blueprint

This workspace captures the target architecture for the Mo i Rana homelab: Proxmox on HP DL380 Gen9 nodes, Cloudflare R2-backed Terraform state, Fortigate-managed networking, self-hosted documentation, and future Cloudflare/cloudflared public access.

Future Codex sessions should read [AGENTS.md](AGENTS.md) first for the live VPS, IPsec, Terraform Agent, and Proxmox management context.

## Goals

- Keep Terraform state in Cloudflare R2 with one state key per repo/stack.
- Keep private management networks private.
- Manage Proxmox, Fortigate/Palo Alto, Cloudflare, and application infrastructure with Terraform where practical.
- Self-host the existing docs platform with high availability on Ubuntu VMs.
- Prepare for Ceph/shared storage over a dedicated 10 Gbit network.
- Expose selected public services later through Cloudflare Tunnel, starting with Plex when ready.

## Key Decision

For private infrastructure providers such as Proxmox and Fortigate, Terraform should run from a trusted workstation, the Frankfurt VPS, or a self-hosted runner/agent that can reach the homelab APIs.

Remote workers that do not sit on the VPN/IPsec path cannot reach `192.168.13.0/24`, VLAN 16 Proxmox management, or other private homelab management addresses behind Starlink CGNAT. Use one of these patterns:

1. R2 state with local execution from a trusted workstation or admin VM.
2. R2 state with execution from the Frankfurt VPS when IPsec is up.
3. R2 state with a self-hosted runner/agent inside the homelab with controlled access to the required management APIs.

See [Terraform State](docs/terraform-cloud-state.md) for the recommended state split.

## Documents

- [Architecture](docs/architecture.md)
- [Network Model](docs/network-model.md)
- [Terraform Cloud State](docs/terraform-cloud-state.md)
- [Infrastructure As Code Operating Model](docs/iac-operating-model.md)
- [Storage Plan](docs/storage-plan.md)
- [Proxmox And Ceph Plan](docs/proxmox-ceph-plan.md)
- [Self-Hosted Docs Platform](docs/self-hosted-docs-platform.md)
- [Cloudflare Tunnel And Plex](docs/cloudflare-tunnel-plex.md)
- [Frankfurt VPS Ops Hub](docs/frankfurt-vps-ops-hub.md)
- [Repository Split Plan](docs/repo-split-plan.md)
- [Implementation Roadmap](docs/implementation-roadmap.md)
