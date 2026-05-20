# Implementation Roadmap

## Phase 1: State And Repo Hygiene

- [x] Use Cloudflare R2 as the default Terraform backend.
- [x] Migrate Proxmox core state to `terraform-proxmox/proxmox-core/terraform.tfstate`.
- [x] Create Cloudflare docs tunnel state at `terraform-cloudflare/docs-tunnel/terraform.tfstate`.
- [x] Keep credentials out of Git in VPS env files and ignored backend config.
- [ ] Migrate Fortigate/Palo/Plex/etc. only when those stacks are ready.

## Phase 2: Proxmox Baseline

- [x] Restore healthy HP-only Proxmox cluster after removing `dell1`.
- [x] Configure routed Proxmox management on VLAN 16.
- [x] Manage first Ubuntu Noble test VM with Terraform.
- [x] Deploy `bastion01`, `mkdocs`, and `docker1` with Terraform.
- [x] Use Ubuntu cloud image plus cloud-init, not custom ISOs.
- [ ] Decide later whether a self-hosted runner inside the homelab is worth it.

## Phase 3: Execution Model

- [x] Install Terraform Cloud Agent container on the VPS for future experiments.
- [x] Prefer R2 state plus CLI runs from VPS/workstation for current stacks.
- [x] Run VPS Terraform containers with `--network host`.
- [x] Pin Cloudflare API to IPv4 when using IPv4-only token restrictions.
- [ ] Add GitHub Actions/self-hosted runner later if PR-driven plans become useful.

## Phase 4: Docs Platform

- [x] Create `mkdocs` VM in Proxmox.
- [x] Build `az-static-web-app-docs/docs` into static HTML.
- [x] Serve from Nginx on `10.0.0.37`.
- [x] Publish `docs.lanilsen.com` through Cloudflare Tunnel.
- [x] Protect public docs with Cloudflare Access One-time PIN.
- [ ] Add a second docs VM or failover pattern later if needed.
- [ ] Add monitoring and backup checks.

## Phase 5: 10 Gbit And Ceph

- [ ] Cable the 10 Gbit RJ45 cards through the Zyxel switch.
- [ ] Create a dedicated storage network.
- [ ] Validate MTU and throughput.
- [ ] Add SAS disks to all HP nodes.
- [ ] Bootstrap Ceph through Proxmox tooling.
- [ ] Only after validation, encode the stable parts in docs or Terraform.

## Phase 6: Monitoring

- [ ] Create `monitoring1` with Terraform.
- [ ] Deploy Docker-based InfluxDB, Telegraf, Chronograf, Kapacitor, OpenSearch, Dashboards, and Logstash.
- [ ] Install Telegraf agents on bastion/docs/docker/monitoring VMs.
- [ ] Add ping and x509 checks from the central monitoring host.
- [ ] Add syslog/audit forwarding into Logstash/OpenSearch.
- [ ] Decide dashboard access path: VPN-only or Cloudflare Access.

## Phase 7: Cloudflare

- [x] Create dedicated Cloudflare Terraform stack under `docs-tunnel`.
- [x] Manage `docs.lanilsen.com` DNS record.
- [x] Deploy `cloudflared` service on the `mkdocs` VM.
- [x] Publish docs through tunnel.
- [x] Add Cloudflare Access OTP allowlist.
- [ ] Reassess Plex exposure carefully before enabling it.
