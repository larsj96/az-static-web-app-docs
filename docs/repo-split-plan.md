# Repository Split Plan

This workspace remains the planning and runbook area. Automation is being split into smaller repos with separate state, credentials, and blast radius.

## Target Repos

```text
az-static-web-app-docs
  Published homelab documentation source. Despite the old repo name, this now builds the self-hosted MkDocs site.

terraform-proxmox
  Proxmox VMs, cloud images, cloud-init, VM modules, and storage references.

terraform_fortigate
  Fortigate VLANs, DHCP reservations, firewall policy, and IPsec to the Frankfurt VPS.

terraform-palo
  Palo Alto zones, objects, policies, and related provider state.

terraform_cloudfare / local terraform-cloudflare
  Cloudflare DNS, tunnels, Access applications, identity providers, lanilsen.com, and public service exposure.

Ansible / local ansible-homelab
  Ubuntu baseline, Docker install, app roles, monitoring agents, and post-cloud-init config.

docker / local docker-homelab
  Docker Compose stacks and service runtime config.
```

## Local Working Folders

The split working folders were created locally under:

```text
C:\github
```

Current local folders:

```text
C:\github\homelab-docs
C:\github\terraform-proxmox
C:\github\terraform-fortigate
C:\github\terraform-palo
C:\github\terraform-cloudflare
C:\github\ansible-homelab
C:\github\docker-homelab
```

As of `2026-05-20`, these are real local clones managed through WSL:

```text
C:\github\terraform-proxmox
C:\github\terraform-fortigate
C:\github\terraform-palo
C:\github\terraform-cloudflare
C:\github\ansible-homelab
C:\github\docker-homelab
C:\github\az-static-web-app-docs
```

The preserved scaffold folders may still exist:

```text
C:\github\terraform-cloudflare-scaffold
C:\github\ansible-homelab-scaffold
```

Do not delete scaffold folders until useful starter files have been intentionally merged or discarded.

## GitHub State

`larsj96/terraform-proxmox` now contains:

```text
proxmox-core/main.tf
proxmox-core/providers.tf
proxmox-core/variables.tf
proxmox-core/versions.tf
proxmox-core/outputs.tf
proxmox-core/terraform.tfvars.example
proxmox-core/README.md
proxmox-core/bastion.tf
proxmox-core/mkdocs.tf
proxmox-core/docker1.tf
```

The published `proxmox-core` scaffold reflects the successful `2026-05-20` live VM run:

```text
VM ID: 9001
name: ubuntu-noble-test-01
node: hp1
disk: nvme-local, 60 GiB
network: vmbr0, VLAN 110
```

`larsj96/docker` was empty after clone, so a starter `README.md` was added via the GitHub connector.

`larsj96/terraform_cloudfare` now contains the live Cloudflare docs tunnel stack:

```text
docs-tunnel/access.tf
docs-tunnel/identity.tf
docs-tunnel/main.tf
docs-tunnel/outputs.tf
docs-tunnel/providers.tf
docs-tunnel/variables.tf
docs-tunnel/versions.tf
docs-tunnel/backend.r2.tfbackend.example
```

Live resources in that state:

```text
docs.lanilsen.com DNS record
homelab-docs Cloudflare Tunnel
Cloudflare Tunnel config -> http://10.0.0.37
Homelab Docs Cloudflare Access application
One-time PIN Access identity provider
```

`larsj96/Ansible` contains the live service deployment roles:

```text
roles/mkdocs/
roles/cloudflared/
playbooks/mkdocs.yml
playbooks/cloudflared-docs.yml
```

## State Boundary Rule

Keep Terraform state split by platform/failure domain:

```text
Proxmox changes should not lock Fortigate state.
Fortigate changes should not risk Proxmox VM state.
Cloudflare changes should not need private homelab/VPN reachability.
Palo Alto and Fortigate credentials should stay separate.
```

Use explicit variables and documentation first. Avoid remote-state coupling until there is a strong reason.

## Naming Cleanup Still Pending

Some remote repo names are historical:

| Current remote | Local folder | Long-term preferred name |
| --- | --- | --- |
| `terraform_cloudfare` | `terraform-cloudflare` | `terraform-cloudflare` |
| `Ansible` | `ansible-homelab` | `ansible-homelab` |
| `docker` | `docker-homelab` | `docker-homelab` |
| `az-static-web-app-docs` | `az-static-web-app-docs` | `homelab-docs` or keep as-is with updated description |

Renaming repos is optional. State keys and docs should use logical stack names even if GitHub repo names lag behind.
