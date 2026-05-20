# Proxmox And Ceph Plan

## Current Proxmox Access

Proxmox management is on NIC0:

- `https://192.168.13.xxx:8006/`
- 1 Gbit management network
- Not reachable from VPN by design

That is a good security boundary. Keep it that way unless a specific automation host is granted access.

## Cluster Plan

Use the three reliable HP DL380 Gen9 servers as the real Proxmox cluster. Keep the Dell R820 out of quorum and critical workloads because it is not reliable.

Recommended role split:

| Node type | Role |
| --- | --- |
| HP DL380 Gen9 x3 | Proxmox cluster, Ceph monitors/managers/OSDs, production VMs |
| Dell R820 | Temporary lab workloads, backup restore tests, disposable compute |

## Storage Plan

Future storage target:

- 8x 1.2 TB SAS disks per HP server for Ceph capacity.
- NVMe for fast local storage, metadata/WAL/DB where appropriate, or selected high-performance pools.
- Dedicated 10 Gbit RJ45 network through Zyxel XGS1250-12 for Ceph and migration/storage traffic.

Do not build Ceph over the 1 Gbit management network except for a short lab test.

## Network Separation

| Interface/network | Purpose |
| --- | --- |
| NIC0, 1 Gbit | Proxmox management |
| 10 Gbit RJ45 | Ceph, migration, storage replication |
| VM uplinks | VM/service traffic via MikroTik/Fortigate |

## Terraform Scope

Terraform is a good fit for:

- VM definitions
- cloud-init snippets
- templates
- resource pools
- tags
- networks/bridges if the pattern is stable

Terraform is not the first tool I would use for:

- Initial Proxmox installation
- Ceph bootstrap
- Emergency cluster repair
- One-off storage recovery

For Ceph bootstrap, document the manual/proxmox-native steps first. Add Terraform only after the target shape is stable.

## Current State Update

As of 2026-05-19, the HP node NVMes were wiped and rebuilt as local LVM-thin storage for immediate VM use:

```text
storage ID: nvme-local
VG: nvme
thinpool: data
nodes: hp1,hp2,hp3
```

This is intentionally a near-term bootstrap choice, not the final Ceph design.

Ceph should wait until the 10 Gbit storage network and SAS disk plan are ready. When Ceph is built, prefer separate device classes/pools for SAS HDD and NVMe rather than mixing them blindly.
