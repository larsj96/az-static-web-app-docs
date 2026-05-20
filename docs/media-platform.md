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

Create the VM infrastructure in `terraform-proxmox/proxmox-core`:

```text
media1
  purpose: Docker Compose media automation and maybe first Plex instance
  node: hp2 preferred if storage locality matters
  network: service VLAN
  disk: VM boot disk on nvme-local
  media: mounted from HP2 SAS path, not baked into the boot disk

plex1
  purpose: optional standalone Plex compute
  create later if transcoding, CPU placement, or access design deserves separation
```

Keep the SAS media library out of Terraform-managed VM boot disks. Terraform should create machines and attach networks/disks where appropriate; it should not become the source of truth for media files.

## Ansible Shape

Use Ansible for the OS and Docker layer:

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
| Jackett or Prowlarr | Indexer bridge |
| qBittorrent or Transmission | Downloads |
| Overseerr | Request workflow |

## Cloudflare And Access

Use Cloudflare Access for admin-facing HTTP apps:

- `overseerr.lanilsen.com`
- `sonarr.lanilsen.com`
- `radarr.lanilsen.com`
- `jackett.lanilsen.com` or `prowlarr.lanilsen.com`
- `qbittorrent.lanilsen.com`

Plex needs a separate decision. Cloudflare Tunnel may be fine for management/light access, but avoid blindly proxying heavy video traffic through Cloudflare. Prefer VPN or Plex-native remote access for streaming unless the policy and traffic profile are confirmed acceptable.

## First Implementation Steps

1. Confirm the HP2 SAS volume layout and filesystem.
2. Decide whether HP2 exports NFS directly or a storage VM owns the export.
3. Add `media1` to Terraform.
4. Add a media Docker Compose role to Ansible.
5. Add Telegraf to `media1` through the shared `telegraf_agents` group.
6. Add Cloudflare DNS/Access only after the internal app works.
