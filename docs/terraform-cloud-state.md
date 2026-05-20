# Terraform State

## Current Decision

Do not use HCP Terraform/Terraform Cloud as the primary state backend for large homelab stacks. The free tier managed-resource limit can become a problem once Proxmox, Fortigate, Palo Alto, Cloudflare, VMs, DNS, firewall objects, and future services are all managed.

Use Cloudflare R2 as the default remote state backend instead:

- S3-compatible backend.
- Native S3 lock file support with `use_lockfile = true`.
- No HCP Terraform managed-resource billing.
- One bucket, separate keys per repo/stack.
- Keep credentials outside git.

Suggested bucket:

```text
lanilsen-terraform-state
```

Suggested state keys:

```text
terraform-proxmox/proxmox-core/terraform.tfstate
terraform-fortigate/core/terraform.tfstate
terraform-palo/core/terraform.tfstate
terraform-cloudflare/dns/terraform.tfstate
```

## Migration Status

Completed on `2026-05-20`:

```text
bucket: lanilsen-terraform-state
state: terraform-proxmox/proxmox-core/terraform.tfstate
source: /opt/homelab-ops/repos/proxmox-firstvm/terraform.tfstate
```

Verified from the Frankfurt VPS with:

```bash
terraform state list
```

Expected resources were present:

```text
proxmox_virtual_environment_download_file.ubuntu_noble_cloud_image
proxmox_virtual_environment_vm.bastion01
proxmox_virtual_environment_vm.docker1
proxmox_virtual_environment_vm.mkdocs
proxmox_virtual_environment_vm.ubuntu_test
```

## Execution Constraint

Terraform Cloud remote workers cannot reach private homelab APIs such as:

- Proxmox at `https://192.168.13.x:8006`
- Fortigate internal management interfaces
- Any private service behind Starlink CGNAT

That means execution should stay local, on the workstation, on the Frankfurt VPS, or on a self-hosted CI runner/agent that can reach the private APIs.

## Recommended Execution Phases

### Phase 1: R2 State, Local/VPS Runs

Use R2 as the backend/state store, but run `terraform plan` and `terraform apply` locally from the workstation or VPS/admin VM that can reach the target APIs.

This is the simplest and safest first step.

### Phase 2: Self-Hosted Runner Or Agent

Deploy a small Ubuntu VM or use the Frankfurt VPS as a self-hosted runner/automation host.

The agent should have:

- Outbound HTTPS to GitHub/Cloudflare/provider APIs.
- API access to Proxmox management only if it needs to manage Proxmox.
- API access to Fortigate only if it needs to manage firewall policy.
- No broad east-west network access.

## Workspace Split

| Workspace | Repo | Execution | Purpose |
| --- | --- | --- | --- |
| `terraform-proxmox/proxmox-core` | `terraform-proxmox` | Local/VPS/runner | Proxmox cluster, VMs, templates |
| `terraform-fortigate/core` | firewall repo | Local/VPS/runner | Fortigate base config, VLANs, policy, NAT |
| `homelab-palo-lab` | `terraform-palo` | Local only unless needed | Palo Alto lab/firewall migration work |
| `terraform-cloudflare/dns` | `terraform-cloudflare` | Local/VPS/runner | DNS, tunnels, access policies |
| `homelab-docs-platform` | future infra/app repo | Local/VPS/runner | Ubuntu VMs, docs deployment, load balancing |

Cloudflare can run from almost anywhere because the provider talks to public APIs, but still use R2 state for consistency and cost control.

## State Naming

Keep state boundaries close to ownership boundaries:

- One state for Proxmox cluster primitives and reusable VM templates.
- One state for firewall/network base.
- One state for Cloudflare public DNS/tunnel resources.
- One state for the docs application stack.

Avoid one giant state for everything. It makes credentials, blast radius, and dependency handling too tangled.

## Secrets

Use environment variables, ignored tfvars files, or CI secrets for:

- Proxmox API tokens
- Fortigate API tokens
- Cloudflare API tokens and R2 S3 keys
- SSH private keys, if Terraform must provision VMs directly

Do not commit `terraform.tfvars` files containing secrets.
