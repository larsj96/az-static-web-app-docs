# Repository Split Plan

This workspace remains the planning and runbook area. Long term, automation should live in smaller repos with separate Terraform Cloud workspaces, credentials, and blast radius.

## Target Repos

```text
homelab-docs
  Architecture, runbooks, network/storage notes, and future NetBox/IPAM decisions.

terraform-proxmox
  Proxmox VMs, cloud images, cloud-init, VM modules, and storage references.

terraform_fortigate
  Fortigate VLANs, DHCP reservations, firewall policy, and IPsec to the Frankfurt VPS.

terraform-palo
  Palo Alto zones, objects, policies, and related provider state.

terraform_cloudfare / future terraform-cloudflare
  Cloudflare DNS, tunnels, lanilsen.com, and public service exposure.

Ansible / future ansible-homelab
  Ubuntu baseline, Docker install, app roles, monitoring agents, and post-cloud-init config.

docker / future docker-homelab
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

Git was not installed on PATH in the Codex Windows session, and WSL was not visible from this runtime. The user ran the clone commands manually from their WSL Ubuntu shell.

As of `2026-05-20`, these are now real local clones:

```text
C:\github\terraform-proxmox
C:\github\terraform-fortigate
C:\github\terraform-palo
C:\github\terraform-cloudflare
C:\github\ansible-homelab
C:\github\docker-homelab
```

The preserved scaffold folders are:

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

## State Boundary Rule

Keep Terraform state split by platform/failure domain:

```text
Proxmox changes should not lock Fortigate state.
Fortigate changes should not risk Proxmox VM state.
Cloudflare changes should not need private homelab/VPN reachability.
Palo Alto and Fortigate credentials should stay separate.
```

Use explicit variables and documentation first. Avoid remote-state coupling until there is a strong reason.
