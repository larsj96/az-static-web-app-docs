# Media Platform

## Goal

Build a lab media platform without mixing three different concerns into one fragile VM:

- Plex compute and user-facing playback.
- Media automation such as Sonarr, Radarr, Jackett/Prowlarr, qBittorrent/Transmission, and Overseerr.
- Bulk movie/TV storage.

The first version can be simple, but the boundaries should be clear so the stack can grow without a rebuild.

## Recommended First Split

| Layer | First Choice | Why |
| --- | --- | --- |
| Media storage | HP2 local SAS set, 10x600 GB | Good use for the existing 5 TB-ish SAS capacity that should not become Ceph right now. |
| Media apps | One Docker Compose VM, for example `media1` | Sonarr/Radarr/Jackett/qBittorrent/Overseerr work well together and share download/import paths. |
| Plex compute | Start on `media1`, split to `plex1` later if needed | Keeps day-one simple while preserving a clean migration path for transcoding/GPU/CPU tuning. |
| External access | Cloudflare DNS and Access for admin apps; be careful with streaming | Cloudflare Tunnel is great for admin HTTP apps, but heavy Plex streaming should be reviewed before proxying. |

## Why Not Put Everything Directly On HP2?

HP2 has useful local SAS storage, but Proxmox hosts should stay boring. The better pattern is:

1. HP2 owns the disks.
2. A storage service exposes a media path to the media VM.
3. The app VM runs containers and mounts that path.

That gives cleaner backups, easier VM rebuilds, and less risk to the Proxmox host.

## Storage Options

| Option | Fit | Notes |
| --- | --- | --- |
| NFS export from HP2 | Good first lab choice | Simple for Linux VMs, predictable permissions, easy to mount on `media1` or `plex1`. |
| SMB share from a storage VM | Good if Windows clients need direct access | More moving parts, but useful later. |
| Passthrough/controller to storage VM | Possible later | Stronger isolation, but more planning and downtime. |
| Ceph | Later | Wait until 10 Gbit and SAS layout are proven. Do not force media storage into Ceph now. |

## Terraform Shape

Create the VM infrastructure in `terraform-proxmox/proxmox-core`. The first scaffold is now:

```text
media1
  VMID: 9050
  purpose: Docker Compose media automation and first Plex instance
  node: current proxmox-core target node, usually hp1 until hp2 placement is made explicit
  network: service VLAN 12
  disk: 250 GiB VM boot/app disk on nvme-local
  media: future mount from HP2 SAS path, not baked into the boot disk

plex1
  purpose: optional standalone Plex compute
  create later if transcoding, CPU placement, or access design deserves separation
```

Keep the SAS media library out of Terraform-managed VM boot disks. Terraform should create machines and attach networks/disks where appropriate; it should not become the source of truth for media files.

## Ansible Shape

Use Ansible for the OS and Docker layer. The first scaffold lives in `ansible-homelab`:

```text
inventory/homelab.ini
playbooks/media.yml
group_vars/media.yml
group_vars/media_secrets.yml.example
roles/media_stack/
```

The media playbook applies `docker_host` and `media_stack`. `media_stack` renders a Docker Compose stack for Plex, Sonarr, Radarr, Prowlarr, qBittorrent, and Overseerr.

Current first-boot behavior:

- Use local `/mnt/media`, `/mnt/media/movies`, and `/mnt/media/tv` directories.
- Keep Plex claim tokens out of git in ignored `group_vars/media_secrets.yml`.
- Leave Cloudflare exposure out of this first pass.
- Add the host to `telegraf_agents` so the shared monitoring playbook can deploy the agent.

Future HP2 SAS mount behavior:

```yaml
media_create_local_library_dirs: false
media_nfs_enabled: true
media_nfs_src: "hp2.example:/export/media"
```

The exact HP2 export path and permissions still need to be confirmed before enabling this.

Ansible owns:

- Install Docker and Compose.
- Create service users and directory layout.
- Mount media storage.
- Render Docker Compose for Plex and automation services.
- Manage environment files outside Git.
- Configure Telegraf so the media VM reports metrics automatically.

Suggested app set:

| Service | Role |
| --- | --- |
| Plex | Playback and library server |
| Sonarr | TV automation |
| Radarr | Movie automation |
| Prowlarr | Indexer bridge |
| qBittorrent | Downloads |
| Overseerr | Request workflow |

Initial internal ports:

| Port | Service |
| ---: | --- |
| `32400` | Plex |
| `8989` | Sonarr |
| `7878` | Radarr |
| `9696` | Prowlarr |
| `8080` | qBittorrent Web UI |
| `5055` | Overseerr |

## Cloudflare And Access

Use Cloudflare Access for admin-facing HTTP apps:

- `overseerr.lanilsen.com`
- `sonarr.lanilsen.com`
- `radarr.lanilsen.com`
- `jackett.lanilsen.com` or `prowlarr.lanilsen.com`
- `qbittorrent.lanilsen.com`

Plex needs a separate decision. Cloudflare Tunnel may be fine for management/light access, but avoid blindly proxying heavy video traffic through Cloudflare. Prefer VPN or Plex-native remote access for streaming unless the policy and traffic profile are confirmed acceptable.

## Deployment Status

Deployed on `2026-05-21`:

| Component | Status |
| --- | --- |
| Terraform VM `media1` | Created and running |
| VMID | `9050` |
| IP | `10.0.0.39` |
| Plex | Running |
| Sonarr | Running |
| Radarr | Running |
| Prowlarr | Running |
| qBittorrent | Running |
| Overseerr | Running |
| Telegraf | Active |

The current media library path is local to the VM for bootstrapping. Move it to HP2 SAS-backed storage once the export path and permissions are decided.

## Next Implementation Steps

1. Confirm the HP2 SAS volume layout, filesystem, export path, and permissions.
2. Decide whether HP2 exports NFS directly or a storage VM owns the export.
3. Enable the media NFS variables once the HP2 SAS export is real.
4. Pin qBittorrent credentials through ignored Ansible secrets instead of relying on the generated temporary password.
5. Add Cloudflare DNS/Access only after the internal app works.
