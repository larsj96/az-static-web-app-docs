# Media Platform

## Goal

Build a lab media platform without mixing three different concerns into one fragile VM:

- Plex compute and user-facing playback.
- Media automation such as Sonarr, Radarr, Jackett, Prowlarr, qBittorrent, Deluge, Ombi, and Overseerr.
- Bulk movie/TV storage.

The first version can be simple, but the boundaries should be clear so the stack can grow without a rebuild.

## Current Live Shape

`media1` is the first combined media VM:

| Item | Value |
| --- | --- |
| VM | `media1` |
| VMID | `9050` |
| VLAN | `12 / fortigate_onprem_k8s` |
| Role | Docker Compose host for Plex and media automation |
| Placement | Pinned to `hp3` |
| Boot/app storage | `nvme-local` |
| Media storage | `sas-hp3`, attached as a dedicated data disk and mounted at `/mnt/media` |

The current app stack is:

- Plex
- Sonarr
- Radarr
- Jackett
- Prowlarr
- qBittorrent
- Deluge
- Overseerr
- Ombi
- Unpackerr

`media1` should not use the generic Proxmox target-node variable. It belongs near the storage it consumes: boot/app data on local NVMe and bulk media on the hp3 SAS pool.

## Recommended Split

| Layer | First Choice | Why |
| --- | --- | --- |
| Media storage | `hp3` local SAS LVM-thin storage, `sas-hp3` | Good use for the current 4.9 TB SAS capacity that should not become Ceph right now. |
| Media apps | One Docker Compose VM, `media1` | The Arr stack, indexers, request apps, downloaders, and extraction helper share paths cleanly. |
| Plex compute | Start on `media1`, split to `plex1` later if needed | Keeps day-one simple while preserving a clean migration path for transcoding/GPU/CPU tuning. |
| External access | Cloudflare DNS and Access for admin apps; be careful with streaming | Cloudflare Tunnel is great for admin HTTP apps, but heavy Plex streaming should be reviewed before proxying. |

## Why Not Put Everything Directly On The Proxmox Host?

The Proxmox node has useful local SAS storage, but Proxmox hosts should stay boring. The better pattern is:

1. The Proxmox node owns the disks.
2. Terraform attaches a dedicated data disk from that local storage to `media1`.
3. The app VM formats and mounts the disk at `/mnt/media`, then runs containers on top.
4. If Plex compute later moves to `plex1`, the media path can be exported over NFS or moved behind a storage VM.

That gives cleaner rebuilds, easier monitoring, and less risk to the Proxmox host.

## Storage Options

| Option | Fit | Notes |
| --- | --- | --- |
| Dedicated `sas-hp3` data disk on `media1` | Current first choice | Simple, fast to implement, and keeps the SAS media library off the boot disk. |
| NFS export from `media1` or a storage VM | Good later choice | Lets a future `plex1` compute VM reuse the same media library without moving the data. |
| SMB share from a storage VM | Good if Windows clients need direct access | More moving parts, but useful later. |
| Passthrough/controller to storage VM | Possible later | Stronger isolation, but more planning and downtime. |
| Ceph | Later | Wait until 10 Gbit and SAS layout are proven. Do not force media storage into Ceph now. |

## Reusable Storage Contract

Keep the storage contract stable even when the backing storage changes:

| Path | Owner | Purpose |
| --- | --- | --- |
| `/opt/media/config` | `media1` boot/app disk | Container configs and app state. |
| `/opt/media/downloads` | `media1` boot/app disk | Shared download/import staging path. |
| `/opt/media/downloads/completed` | `media1` boot/app disk | Completed download/extraction handoff path. |
| `/opt/media/downloads/incomplete` | `media1` boot/app disk | In-progress download path. |
| `/mnt/media/movies` | `media1` SAS data disk from `sas-hp3` | Plex/Radarr movie library. |
| `/mnt/media/tv` | `media1` SAS data disk from `sas-hp3` | Plex/Sonarr TV library. |

The stable application contract is `/mnt/media`. Today it is a dedicated Terraform-attached SAS disk. Later it can become an NFS mount or a replicated storage service without changing Plex/Sonarr/Radarr container paths.

## Terraform Shape

The VM infrastructure lives in `terraform-proxmox/proxmox-core`:

```text
media1
  VMID: 9050
  purpose: Docker Compose media automation and first Plex instance
  node: hp3
  network: service VLAN 12
  boot/app disk: 120 GiB on nvme-local
  media disk: 4000 GiB on sas-hp3, mounted by Ansible at /mnt/media

plex1
  purpose: optional standalone Plex compute
  create later if transcoding, CPU placement, or access design deserves separation
```

Keep the SAS media library out of Terraform-managed VM boot disks. Terraform owns VM placement and disk attachment; it does not own media contents.

Proxmox does not have VMware-style automatic DRS in this setup. For now the correct pattern is explicit placement variables:

```hcl
media_node_name         = "hp3"
media_boot_storage      = "nvme-local"
media_data_storage      = "sas-hp3"
media_data_disk_size_gb = 4000
```

Later, we can add a small placement layer or module map so app classes choose allowed nodes/storages automatically.

## Ansible Shape

Use Ansible for the OS and Docker layer. The scaffold lives in `ansible-homelab`:

```text
inventory/homelab.ini
playbooks/media.yml
group_vars/media.yml
group_vars/media_secrets.yml.example
roles/media_stack/
```

The media playbook applies `docker_host` and `media_stack`. `media_stack` renders a Docker Compose stack for Plex, Sonarr, Radarr, Jackett, Prowlarr, qBittorrent, Deluge, Overseerr, Ombi, and Unpackerr.

Current first-boot behavior:

- Format and mount the Terraform-attached `scsi1` disk at `/mnt/media` when `media_data_disk_enabled` is true.
- Keep Plex claim tokens out of git in ignored `group_vars/media_secrets.yml`.
- Keep Sonarr/Radarr API keys for Unpackerr out of git in ignored `group_vars/media_secrets.yml`.
- Leave Cloudflare exposure out of this first pass.
- Add the host to `telegraf_agents` so the shared monitoring playbook can deploy the agent.

Current SAS disk behavior:

```yaml
media_data_disk_enabled: true
media_data_disk_device: "/dev/disk/by-id/scsi-0QEMU_QEMU_HARDDISK_drive-scsi1"
media_data_disk_fstype: "ext4"
```

The Ansible role only creates a filesystem when the data disk has no filesystem yet.

## Media Secrets In Vault

Media application secrets live in HashiCorp Vault, not Git.

Vault path:

```text
homelab/media/media1
```

Current fields:

| Field | Purpose |
| --- | --- |
| `plex_claim` | Short-lived Plex claim token used only during initial claim/reclaim. |
| `unpackerr_sonarr_api_key` | Sonarr API key for Unpackerr integration. |
| `unpackerr_radarr_api_key` | Radarr API key for Unpackerr integration. |

Bastion has a limited Vault token in:

```text
~/.config/homelab/vault.env
```

Pull the ignored Ansible secrets file from Vault:

```bash
cd ~/ansible-bench-run
./scripts/pull-media-secrets-from-vault.sh
```

Update one media secret:

```bash
cd ~/ansible-bench-run
./scripts/put-media-secret.sh plex_claim 'claim-xxxx'
./scripts/pull-media-secrets-from-vault.sh
ansible-playbook playbooks/media.yml
```

After Plex is claimed, clear the claim token because it is temporary:

```bash
./scripts/put-media-secret.sh plex_claim ''
./scripts/pull-media-secrets-from-vault.sh
ansible-playbook playbooks/media.yml
```

## Services

| Service | Role |
| --- | --- |
| Plex | Playback and library server |
| Sonarr | TV automation |
| Radarr | Movie automation |
| Jackett | Indexer bridge, kept for trackers/tools that still expect it |
| Prowlarr | Indexer bridge |
| qBittorrent | Downloads |
| Deluge | Downloads, useful for workflows that fit Deluge better |
| Overseerr | Request workflow |
| Ombi | Alternative request workflow |
| Unpackerr | Watches completed downloads and extracts rar archives for Sonarr/Radarr imports |

Initial internal ports:

| Port | Service |
| ---: | --- |
| `32400` | Plex |
| `8989` | Sonarr |
| `7878` | Radarr |
| `9696` | Prowlarr |
| `9117` | Jackett |
| `8080` | qBittorrent Web UI |
| `8112` | Deluge Web UI |
| `5055` | Overseerr |
| `3579` | Ombi |

## Cloudflare And Access

Use Cloudflare Access for admin-facing HTTP apps:

- `overseerr.lanilsen.com`
- `ombi.lanilsen.com`
- `sonarr.lanilsen.com`
- `radarr.lanilsen.com`
- `prowlarr.lanilsen.com`
- `jackett.lanilsen.com`
- `qbittorrent.lanilsen.com`
- `deluge.lanilsen.com`

Plex needs a separate decision. Cloudflare Tunnel may be fine for management/light access, but avoid blindly proxying heavy video traffic through Cloudflare. Prefer VPN or Plex-native remote access for streaming unless the policy and traffic profile are confirmed acceptable.

## Monitoring Expectations

`media1` belongs in the Ansible `telegraf_agents` group and reports host metrics to the central monitoring stack. Treat these as the baseline checks before adding public exposure:

| Signal | Expectation |
| --- | --- |
| Host availability | `media1` should respond to ping and SSH from the internal management path. |
| Docker health | Plex, Sonarr, Radarr, Jackett, Prowlarr, qBittorrent, Deluge, Overseerr, Ombi, and Unpackerr containers should stay running after reboot. |
| Disk capacity | Track `/opt/media` and `/mnt/media` usage separately so app config growth is not confused with media library growth. |
| Media disk health | Alert on missing `/mnt/media`, stale mounts, or sudden drops in available capacity. |
| Service reachability | Internal ports `32400`, `8989`, `7878`, `9696`, `9117`, `8080`, `8112`, `5055`, and `3579` should be checked from the services VLAN or monitoring host. |

Application-level API keys and credentials should stay out of documentation and out of Git. Add service-specific checks only after credentials are stored in the chosen secrets workflow.

## Deployment Status

Updated on `2026-05-22`:

| Component | Status |
| --- | --- |
| Terraform VM `media1` | Pinned to `hp3` |
| VMID | `9050` |
| Boot/app disk | `nvme-local` |
| Media disk | `sas-hp3`, mounted at `/mnt/media` |
| Plex/Sonarr/Radarr/Jackett/Prowlarr/qBittorrent/Deluge/Overseerr/Ombi/Unpackerr | Managed by Ansible Docker Compose |
| Telegraf | Managed by the shared monitoring playbook |

Verified after rebuild:

```text
media1 IP: 10.0.0.39
/mnt/media: /dev/sdb ext4, 3.9T usable, backed by sas-hp3
/opt/media: boot/app disk, about 116G usable
```

The media library path is `/mnt/media` on the attached `sas-hp3` data disk. Treat the boot/app disk as replaceable.

Operational note: `local` Proxmox storage is node-local. When a VM is pinned to `hp3`, the Ubuntu cloud image and cloud-init snippet must also exist on `hp3` local storage, or the VM create/start can fail even though the same volume ID works on `hp1`.

## Next Implementation Steps

1. Claim Plex with a short-lived claim token if this is a fresh Plex install.
2. Complete Sonarr/Radarr first-run setup, then store API keys in the chosen secret store and feed them to Unpackerr.
3. Pin qBittorrent/Deluge credentials through ignored Ansible secrets instead of relying on generated defaults.
4. Decide whether a later `plex1` should consume `/mnt/media` through NFS from `media1` or through a dedicated storage VM.
5. Add Cloudflare DNS/Access only after the internal app works.
