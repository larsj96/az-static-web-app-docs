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
terraform-cloudflare/docs-tunnel/terraform.tfstate
terraform-cloudflare/dns/terraform.tfstate
```

## Migration Status

Completed Proxmox migration on `2026-05-20`:

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

Completed Cloudflare docs stack on `2026-05-20` and Access lock-down on `2026-05-21`:

```text
bucket: lanilsen-terraform-state
state: terraform-cloudflare/docs-tunnel/terraform.tfstate
execution host: Frankfurt VPS
```

Expected resources:

```text
random_id.tunnel_secret
cloudflare_zero_trust_tunnel_cloudflared.docs
cloudflare_zero_trust_tunnel_cloudflared_config.docs
cloudflare_dns_record.docs
cloudflare_zero_trust_access_application.docs
cloudflare_zero_trust_access_identity_provider.onetimepin
data.cloudflare_zero_trust_tunnel_cloudflared_token.docs
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
| `terraform-cloudflare/docs-tunnel` | `terraform_cloudfare` / local `terraform-cloudflare` | VPS | Docs DNS, tunnel, Access app, OTP IdP |
| `terraform-cloudflare/dns` | `terraform_cloudfare` / local `terraform-cloudflare` | Local/VPS/runner | Future shared DNS records |
| `homelab-docs-platform` | `terraform-proxmox` + `Ansible` + `terraform_cloudfare` | VPS/bastion | VM, docs deployment, public tunnel |

Cloudflare can run from almost anywhere because the provider talks to public APIs, but still use R2 state for consistency and cost control.

## Current Backend Pattern

Each stack keeps an ignored backend config copied from an example:

```bash
cp backend.r2.tfbackend.example backend.r2.tfbackend
terraform init -backend-config=backend.r2.tfbackend
```

The VPS stores backend credentials in:

```text
/root/.config/homelab/cloudflare-r2.env
```

`/root/.bashrc` sources that file so fresh root shells have the R2 and Cloudflare variables available. Do not commit that env file.

## Cloudflare API IPv6 Gotcha

The VPS has both IPv4 and IPv6:

```text
72.61.95.150
2a02:4780:41:59ef::1
```

The Cloudflare bootstrap token was originally restricted only to `72.61.95.150/32`. Cloudflare API calls from the VPS preferred IPv6 and failed with:

```text
Cannot use the access token from location: 2a02:4780:41:59ef::1
```

Use one of these approaches:

- Add `2a02:4780:41:59ef::1/128` to the token IP condition.
- Or pin Terraform API calls to IPv4 with Docker `--add-host`.

Successful pattern:

```bash
CF_API_IPV4=$(getent ahostsv4 api.cloudflare.com | awk 'NR==1{print $1}')
docker run --rm --network host --add-host "api.cloudflare.com:$CF_API_IPV4" ...
```

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
