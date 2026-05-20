# Storage Plan

This document captures the live storage state and the intended direction.

## Current Decision

Use local NVMe storage now so Terraform can create VMs quickly.

Do not build Ceph yet. Ceph should wait until:

- 10 Gbit storage network is cabled and tested.
- SAS disks are installed consistently across the HP nodes.
- hp3 cluster membership is stable.
- The pool/device-class design is decided intentionally.

## Current Local NVMe Storage

The HP nodes had their NVMe devices wiped and rebuilt as LVM-thin:

```text
VG: nvme
thinpool: data
Proxmox storage ID: nvme-local
content: images,rootdir
nodes: hp1,hp2,hp3
```

Observed sizes:

| Node | Approx thinpool size | Notes |
| --- | ---: | --- |
| hp1 | 2.31 TiB | Includes MP510, Samsung 970 EVO, Intel NVMe. Old Ceph and old LVM signatures removed. |
| hp2 | 1.66 TiB | Two MP510 NVMes. Old LVM removed. |
| hp3 | 2.49 TiB | Three MP510 NVMes. Old VMFS signatures removed. Cluster membership issue remains. |

The Proxmox storage config was added from the quorate side:

```text
lvmthin: nvme-local
        thinpool data
        vgname nvme
        content images,rootdir
        nodes hp1,hp3,hp2
```

## dell1

dell1 must not be wiped.

It has a working 4 TB NVMe-backed directory storage:

```text
storage ID: nvme-dell
path: /mnt/pve/nvme-dell
```

It also has a running VM:

```text
VM 100: mgmt-win1
storage: nvme-dell
```

dell1 is not reliable enough for production cluster storage decisions.

## Ceph Later

Future Ceph should likely use separate pools/device classes:

- SAS HDD capacity pool for bulk/shared storage.
- NVMe pool for fast workloads, or NVMe as DB/WAL assist for HDD OSDs if chosen deliberately.

Do not mix HDD and NVMe in the same Ceph pool without explicit device-class rules.

Potential future shape:

| Pool | Devices | Notes |
| --- | --- | --- |
| `ceph-hdd` | 8x 1.2 TB SAS per HP node | Capacity/shared VM storage. |
| `ceph-nvme` | 2x MP510 per HP node | Fast replicated pool, optional. |
| `nvme-local` | leftover/local NVMe | Fast non-HA scratch/templates if kept. |

## Immediate Terraform Use

Use `nvme-local` for first Terraform-created VMs:

- Ubuntu cloud image based template/test VM.
- Docs platform VMs.
- Any non-critical bootstrap services.

Practical direction chosen at the end of this session:

- Use `nvme-local` now for immediate VM bring-up.
- Treat hp1 Intel and Samsung local NVMe capacity as valuable fast local storage for templates and early VMs.
- Do not pause VM work waiting for Ceph.
- Revisit the final Ceph split later, after network and SAS work is ready.

Terraform/cloud-init still needs snippets-capable storage. Retrying this is a next step:

```bash
pvesm set local --content backup,vztmpl,iso,import,snippets
```

It previously failed due a Proxmox cluster filesystem lock timeout:

```text
cfs-lock 'file-storage_cfg' error: got lock request timeout
```

Fix cluster/cfs health before retrying.
