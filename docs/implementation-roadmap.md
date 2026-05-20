# Implementation Roadmap

## Phase 1: State And Repo Hygiene

- Add Terraform Cloud backend/state configuration to the active Terraform repos.
- Create workspace variable sets for shared secrets and provider credentials.
- Keep Proxmox and firewall execution local until an internal Terraform Cloud Agent exists.
- Document every state boundary in each repo README.

## Phase 2: Proxmox Baseline

- Confirm Proxmox node names, management IPs, and cluster status.
- Create a golden Ubuntu VM template.
- Manage a first test VM with Terraform.
- Decide where the Terraform Cloud Agent VM should live.

## Phase 3: Terraform Cloud Agent

- Deploy a small Ubuntu VM for Terraform Cloud Agent.
- Allow outbound HTTPS to Terraform Cloud.
- Allow narrow API access to Proxmox and firewall targets.
- Move selected private workspaces from local execution to agent execution.

## Phase 4: Docs Platform

- Create two Ubuntu docs VMs.
- Build the `az-static-web-app-docs/docs` content into static HTML.
- Serve from Nginx or Caddy.
- Add internal DNS or VIP-based HA.
- Add monitoring and backups.

## Phase 5: 10 Gbit And Ceph

- Cable the 10 Gbit RJ45 cards through the Zyxel switch.
- Create a dedicated storage network.
- Validate MTU and throughput.
- Add SAS disks to all HP nodes.
- Bootstrap Ceph through Proxmox tooling.
- Only after validation, encode the stable parts in docs or Terraform.

## Phase 6: Cloudflare

- Create a dedicated Cloudflare Terraform workspace.
- Manage `lanilsen.com` DNS records.
- Deploy a `cloudflared` VM in the homelab.
- Publish docs first if external access is desired.
- Reassess Plex exposure carefully before enabling it.

