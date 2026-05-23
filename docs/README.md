# Homelab Blueprint

This site captures the current architecture for the Mo i Rana homelab: Proxmox on HP DL380 Gen9 nodes, Cloudflare R2-backed Terraform state, Fortigate-managed networking, self-hosted documentation, and Cloudflare Tunnel/Access for selected public services.

Future Codex sessions should read `AGENTS.md` in the local Homelab workspace first for live VPS, IPsec, Terraform, and Proxmox management context.

## Goals

- Keep Terraform state in Cloudflare R2 with one state key per repo/stack.
- Keep private management networks private.
- Manage Proxmox, Fortigate/Palo Alto, Cloudflare, and application infrastructure with Terraform where practical.
- Self-host the docs platform on Ubuntu VMs.
- Prepare for Ceph/shared storage over a dedicated 10 Gbit network.
- Expose selected public services through Cloudflare Tunnel and protect admin/docs surfaces with Cloudflare Access.

## Live Status

| Area | Current state |
| --- | --- |
| Proxmox | HP cluster is healthy after removing `dell1`; Terraform-managed VMs run on `hp1` with `nvme-local` storage. |
| Bastion | `bastion01` is live on VLAN 14 at `10.0.0.102` and is the Ansible control/jump host. |
| Docs VM | `mkdocs` is live on VLAN 12 at `10.0.0.35`, built by Ansible from this repo. |
| Identity | `auth1` is the Authentik SSO/MFA host on VLAN 12 at `10.0.0.36`; public endpoint is `https://auth.lanilsen.com`. |
| Public docs | `https://docs.lanilsen.com/` is published by Cloudflare Tunnel and protected by Cloudflare Access. |
| Terraform state | Proxmox, Cloudflare docs, and Palo Alto stacks use Cloudflare R2 bucket `lanilsen-terraform-state`. |
| Automation model | Terraform creates infrastructure; cloud-init bootstraps guests; Ansible configures services. |

## Key Decision

For private infrastructure providers such as Proxmox and Fortigate, Terraform should run from a trusted workstation, the Frankfurt VPS, or a self-hosted runner/agent that can reach the homelab APIs.

Remote workers that do not sit on the VPN/IPsec path cannot reach `192.168.13.0/24`, VLAN 16 Proxmox management, or other private homelab management addresses behind Starlink CGNAT. Use one of these patterns:

1. R2 state with local execution from a trusted workstation or admin VM.
2. R2 state with execution from the Frankfurt VPS when IPsec is up.
3. R2 state with a self-hosted runner/agent inside the homelab with controlled access to the required management APIs.

See [Terraform State](terraform-cloud-state.md) for the recommended state split.

## Documents

- [Architecture](architecture.md)
- [Network Model](network-model.md)
- [Terraform State](terraform-cloud-state.md)
- [Infrastructure As Code Operating Model](iac-operating-model.md)
- [Secrets Management](secrets-management.md)
- [Identity And SSO](identity-and-sso.md)
- [Storage Plan](storage-plan.md)
- [Proxmox And Ceph Plan](proxmox-ceph-plan.md)
- [Self-Hosted Docs Platform](self-hosted-docs-platform.md)
- [Cloudflare Tunnel And Plex](cloudflare-tunnel-plex.md)
- [Frankfurt VPS Ops Hub](frankfurt-vps-ops-hub.md)
- [Repository Split Plan](repo-split-plan.md)
- [Palo Alto Terraform](palo-alto-terraform.md)
- [Implementation Roadmap](implementation-roadmap.md)
