# VPN Throughput Baseline (2026-05-24)

## Why we ran this

After seeing poor file transfer behavior while streaming-related traffic was expected to work over the direct Palo Alto IPv6 IPsec path, we ran a fresh set of tests from WSL to isolate throughput limits and path behavior.

Goal: establish whether the bottleneck is:

- direct Fortigate IPv6 IPsec tunnel,
- VPN/overlay path on the VPS,
- or something else (Fortigate/WAN utilization, path policy, etc.).

## Test context

- Host running commands: WSL on local PC
- WSL source IP: `172.23.143.193`
- Test date: `2026-05-24`
- VPN/VPS details used:
  - Frankfurt VPS: `72.61.95.150` (`/root@72.61.95.150`)
  - OpenVPN interface on VPS: `10.8.0.1`
  - Test VM in homelab: `10.0.0.37` (`mkdocs01`)

## Connectivity sanity checks

```bash
ping -c 6 -W 2 72.61.95.150
ping -c 6 -W 2 10.8.0.1
ip route get 10.8.0.1
ip route get 72.61.95.150
ip route get 10.0.0.37
```

Observed:

- `72.61.95.150`: RTT `42.960/55.587/73.533/9.972 ms`, 0% loss
- `10.8.0.1`: RTT `54.155/59.834/63.065/3.354 ms`, 1/6 lost

## iPerf3 test setup

VPS and VM are temporary `iperf3` test servers.

### VPS (`72.61.95.150`) install/start

```bash
ssh root@72.61.95.150 '
DEBIAN_FRONTEND=noninteractive apt-get update
DEBIAN_FRONTEND=noninteractive apt-get install -y iperf3
pkill -f iperf3 || true
nohup iperf3 -s -p 5201 >/tmp/iperf3_vps_server.log 2>&1 < /dev/null &'
```

### VM (`10.0.0.37`) install/start

```bash
ssh ubuntu@10.0.0.37 'sudo apt-get install -y iperf3'
ssh ubuntu@10.0.0.37 'nohup iperf3 -s -p 5201 >/tmp/iperf3_server.log 2>&1 < /dev/null &'
```

## Test commands and results

### 1) WSL client to VPS public IP (Internet path)

```bash
iperf3 -c 72.61.95.150 -p 5201 -t 30 -i 5 -P 4
iperf3 -c 72.61.95.150 -p 5201 -t 30 -i 5 -P 4 -R
```

- Forward (TX): **~35.2 Mbit/s total** (SUM), retr: **122**
- Reverse (RX): **~121 Mbit/s total** (SUM), retr: **92**

### 2) WSL client to VPS OpenVPN endpoint (`10.8.0.1`)

```bash
iperf3 -c 10.8.0.1 -p 5201 -t 20 -i 5 -P 4
iperf3 -c 10.8.0.1 -p 5201 -t 20 -i 5 -P 4 -R
```

- Forward: **~22.9 Mbit/s total** (SUM), retr: **109**
- Reverse: **~230 Mbit/s total** (SUM), retr: **1157**

### 3) WSL client to homelab VM (`10.0.0.37`)

```bash
iperf3 -c 10.0.0.37 -p 5201 -t 20 -i 5 -P 4
iperf3 -c 10.0.0.37 -p 5201 -t 20 -i 5 -P 4 -R
```

- Forward: **~10.1 Mbit/s total**, retr: **90**
- Reverse: **~7.4 Mbit/s total**, retr: **64**

### 4) Follow-up direct tunnel retest after MSS/NAT-T checks

After Fortigate policy TCP MSS was clamped to `1280`, a packet capture on the VM confirmed SYN packets arrived with `mss 1280`.

NAT-T was then disabled temporarily as an A/B test and later restored:

```text
Palo Alto ike-gw-fortigate-ipv6: NAT traversal enabled
Fortigate palo-ipv6: nattraversal forced
```

Disabling NAT-T did **not** materially improve TCP throughput, so NAT-T overhead is not the main bottleneck.

Retest from WSL to `10.0.0.37` after restoring NAT-T:

```text
Ping: 98-125 ms, average about 112 ms
TCP forward, 4 streams: about 6.9 Mbit/s receiver, 60 retransmits
TCP reverse, 4 streams: about 7.5 Mbit/s receiver, 90 retransmits
UDP 20 Mbit/s forward: about 19.8 Mbit/s receiver, 0.37% loss
```

The important pattern is that UDP at 20 Mbit/s is mostly fine, while TCP collapses into low congestion windows and retransmits.

### 5) Palo dataplane finding

Palo Alto operational output for the direct tunnel showed:

```text
IPsec tunnel: ipsec-fortigate-ipv6:pid-v4-00-00
inner-if: tunnel.20
outer-if: loopback.12
local IP: 2a0d:3341:bb9c:af01::443
peer IP: 2a0d:3341:bb00:6320:a5b:eff:feca:b2e9
natt: True
mtu: 1395
hw-mode: none
```

The physical Starlink WAN interface `ethernet1/1` has DHCPv6-PD and a link-local IPv6 address, but no global IPv6 address on the interface itself. The global endpoint is on `loopback.12` using the delegated prefix.

Two Terraform/API-safe tests were attempted to bind a delegated IPv6 address directly to `ethernet1/1` while keeping DHCPv6-PD:

- static `/128` on `ethernet1/1`
- inherited delegated GUA on `ethernet1/1`

PAN-OS rejected both because a DHCPv6 client interface cannot also have static or inherited IPv6 addressing. That means the current Starlink/Palo design is boxed into using a loopback-style endpoint unless DHCPv6-PD is disabled or another upstream design is introduced.

### 6) Palo DHCPv6 address-request and SLAAC tests

Terraform test switches were added to `terraform-palo/live/homelab/network-base`:

```hcl
starlink_dhcpv6_request_non_temporary_address = true
starlink_enable_dhcpv6_client                 = false
enable_infra_inherited_ipv6                   = false
```

The first switch requests a DHCPv6 non-temporary address (`IA_NA`) while keeping prefix delegation enabled. A targeted apply and Palo commit succeeded, but `ethernet1/1` still showed only:

```text
addr6: fe80::1ecf:82ff:fe6a:3710/64
dhcpv6_client: True
```

So Starlink/PAN-OS did not produce a global IPv6 address on the physical WAN interface through DHCPv6 IA_NA.

The second switch temporarily disables the DHCPv6 client to test pure Starlink WAN SLAAC behavior. The first attempt proved that delegated LAN IPv6 has to be removed before the commit can succeed, because LAN interfaces inherit addresses from the `starlink-ipv6-pd` prefix pool:

```text
ethernet1/3.10 -> ipv6 -> inherited -> assign-addr -> starlink-ipv6-pd-gua
prefix-pool 'starlink-ipv6-pd' is not a valid reference
```

During the maintenance-window retest, the infra inherited IPv6 toggle was disabled and the separate Palo-to-VPS IPv6 VPN objects were temporarily disabled because that stack used `ethernet1/3.10` as its local IPv6 interface. After that, the pure-SLAAC commit succeeded.

Result after pure SLAAC settled:

```text
ethernet1/1 IPv4: 100.76.66.178/10
ethernet1/1 IPv6: fe80::1ecf:82ff:fe6a:3710/64
physical_wan_global_ipv6: none
```

So even with DHCPv6-PD and inherited LAN IPv6 removed, the Palo physical Starlink WAN did **not** get a global IPv6 address. That closes this branch: rebuilding the direct Fortigate tunnel on the physical Palo WAN is not possible with the current PA-510/Starlink behavior.

After the test, normal DHCPv6-PD mode and inherited infra IPv6 were restored in two steps and committed. The Palo-to-VPS IPv6 VPN objects were re-enabled. Post-restore checks confirmed:

- `10.0.0.37` reachable from WSL
- `10.0.0.33` reachable from WSL
- `10.0.0.162:8006` returned HTTP 200 from WSL
- VPS still had both Palo and Fortigate IPsec SAs established

Backup/config snapshots for this test were stored on the VPS:

```text
/root/homelab-backups/palo-slaac-test-20260524-010531
/root/homelab-backups/palo-slaac-test-20260524-011539
```

Conclusion: with the current Palo IPv6 design, the physical Starlink interface cannot be used as the direct IPv6 IPsec endpoint. The direct tunnel can remain functional on the delegated-prefix loopback endpoint, but this test rules out the simple "move it to `ethernet1/1`" fix.

### 7) Native Windows and inherited-GUA endpoint test

To remove WSL from the blame list, `iperf3` was installed natively on Windows and the same tests were run from the workstation's real Palo-side IP:

```text
Windows source IP: 10.1.1.5
Test VM: 10.0.0.37
```

The direct tunnel endpoint was also moved away from the GlobalProtect loopback and onto the inherited delegated GUA that had already worked for the Palo-to-VPS IPv6 tunnel:

```text
old Palo direct endpoint: loopback.12 / 2a0d:3341:bb9c:af01::443
new Palo direct endpoint: ethernet1/3.10 / 2a0d:3341:bb9c:af00:1ecf:82ff:fe6a:3712
Fortigate peer updated: 2a0d:3341:bb9c:af00:1ecf:82ff:fe6a:3712
NAT-T: enabled / forced
```

Palo flow after the endpoint move:

```text
outer-if: ethernet1/3.10
localip: 2a0d:3341:bb9c:af00:1ecf:82ff:fe6a:3712
peerip: 2a0d:3341:bb00:6320:a5b:eff:feca:b2e9
state: active
natt: True
mtu: 1395
pkt-encap: increasing
pkt-decap: increasing
hw-mode: none
```

Native Windows direct tunnel results:

```text
Ping 10.0.0.37: 76-99 ms, average 88 ms, 0% loss
PC -> 10.0.0.37 TCP, 4 streams: 9.08 Mbit/s receiver
10.0.0.37 -> PC TCP reverse, 4 streams: 11.8 Mbit/s receiver, 82 retransmits
```

Forcing smaller MSS from the Windows client did not help:

```text
MSS 1200 forward: 7.83 Mbit/s receiver
MSS 1000 forward: 6.89 Mbit/s receiver
MSS 1200 reverse: 6.60 Mbit/s receiver
```

So this is not simply WSL, and it is not simply "MSS too large".

Control tests to the Frankfurt VPS public IP were much better:

```text
Palo-side Windows -> VPS public: 30.9 Mbit/s receiver
VPS public -> Palo-side Windows: 72.0 Mbit/s receiver
Fortigate-side 10.0.0.37 -> VPS public: 34.1 Mbit/s receiver
VPS public -> Fortigate-side 10.0.0.37: 45.3 Mbit/s receiver
```

This isolates the ugly part: each Starlink site can talk to Germany far better than the two Starlink sites can talk directly across the Palo-to-Fortigate tunnel.

## Interpretation

The results indicate:

- The VPS public destination path is comparatively healthy.
- The homelab-private direct IPv6 path is the current TCP bottleneck (much lower and unstable throughput with TCP retransmits).
- The UDP result shows the path can move packets, but TCP reacts badly to the latency/jitter/return-path behavior.
- Moving the Palo direct endpoint from `loopback.12` to the inherited delegated GUA on `ethernet1/3.10` made the tunnel cleaner but did not solve the throughput problem.
- The most suspicious remaining issue is direct Starlink-to-Starlink packet loss, jitter, or reordering under encrypted tunnel traffic.

For 4K media streaming, this is insufficient. Even modest 4K streaming profiles can exceed this and are sensitive to jitter/retransmits.

## Packet Trace Finding

On `2026-05-24`, the route preference was briefly flipped to the Frankfurt VPS as a control path, then restored to direct-primary:

```text
Palo Alto primary: 10.0.0.0/16 -> tunnel.20 direct IPv6 IPsec, metric 5
Palo Alto fallback: 10.0.0.0/16 -> tunnel.10 Frankfurt VPS, metric 50
Fortigate primary: 10.1.0.0/16 -> palo-ipv6 direct IPv6 IPsec, distance 5
Fortigate fallback: 10.1.0.0/16 -> to-hostinger Frankfurt VPS, distance 50
```

The VPS fallback path now has explicit strongSwan transit selectors for `10.1.0.0/16 <-> 10.0.0.0/16`, but it is intentionally secondary. The goal remains to make the direct Palo-to-Fortigate IPv6 tunnel fast and stable.

Packet captures were taken on `10.0.0.37` while native Windows `iperf3` ran from `10.1.1.5` over the direct tunnel.

PC to VM:

```text
iperf3 P4 receiver: about 11.2 Mbit/s
Windows SYN MSS observed at VM: 1280
SACK blocks observed in VM capture: 731
Palo IPsec auth/decrypt/replay errors: 0
```

VM to PC reverse:

```text
iperf3 P4 receiver: about 7.3 Mbit/s
sender retransmits: 106
SACK blocks observed in VM capture: 1551
Palo IPsec auth/decrypt/replay errors: 0
```

Local Fortigate-side sanity checks were clean:

```text
10.0.0.37 -> 10.0.0.33 ping: 0% loss, about 0.277 ms average
10.0.0.37 <-> 10.0.0.35 iperf3 P4: about 22-23 Gbit/s, 0 retransmits
```

This strongly suggests the VM, Proxmox host, and local Fortigate-side switching are not the bottleneck. The direct path problem is visible as TCP holes/reordering/retransmit pressure across the encrypted Starlink-to-Starlink route. Because the Palo flow shows no IPsec crypto/replay errors and MSS 1280 is already active, the next useful tests are outer WAN captures and controlled Fortigate IPsec offload/NPU checks.

## Fortigate NPU Offload A/B Test

Fortigate phase1 `palo-ipv6` had:

```text
npu-offload: enable
net-device: enable
nattraversal: forced
fragmentation: enable
```

As a controlled test, `npu-offload` was changed to `disable`, then native Windows `iperf3` was rerun against `10.0.0.37`.

Baseline with NPU offload enabled:

```text
PC -> VM P4 TCP: about 15.2 Mbit/s receiver
VM -> PC P4 TCP reverse: about 8.9 Mbit/s receiver, 69 retransmits
```

With NPU offload disabled:

```text
PC -> VM P4 TCP: about 6.8 Mbit/s receiver
VM -> PC P4 TCP reverse: about 9.2 Mbit/s receiver, 84 retransmits
```

Result: disabling Fortigate NPU offload did not help and made forward throughput worse. `npu-offload` was restored to `enable`.

## 4K readiness delta to close

Target baseline for comfortable 4K streaming over this path:

- Sustained throughput: **>100 Mbit/s**
- Low jitter (or at least stable latency under load)
- Minimal retransmits/congestion behavior on TCP tests, or robust UDP path checks with controlled loss

## Next-step troubleshooting checklist

1. Keep the direct IPv6 Palo-to-Fortigate tunnel as the intended primary path, but do not rely on it for Plex/media quality until the TCP issue is fixed.
2. Keep the Frankfurt VPS transit selectors/routes available as secondary fallback and as a control benchmark.
3. Capture the outer WAN path on Palo and Fortigate during a direct `iperf3` run to confirm whether UDP/4500 packets are being lost/reordered before or after each firewall.
4. Leave Fortigate `npu-offload=enable`; the `disable` A/B test was worse.
5. Run UDP at higher rates (`iperf3 -u -b 50M`, then `100M`) while watching Fortigate/Palo counters.
6. Treat physical-WAN IPv6 on the PA-510 as blocked unless Palo support identifies a supported Starlink mode that provides a physical-interface GUA.
7. Prefer Cloudflare/Plex-native public access or VPS fallback only while direct site-to-site performance is being repaired.

## Real solution paths

The likely durable fix is one of these, in this order:

1. Fix the direct Starlink-to-Starlink tunnel by proving where the SACK/loss/reordering starts, then tune or bypass that behavior.
2. Continue direct tunnel MTU/MSS tests as controlled A/B changes, with route preference restored after each test.
3. Put a small IPv6-capable edge device in front of Palo (OpenWrt, MikroTik, OPNsense, or Linux) only if PAN-OS/Starlink cannot provide a clean direct data plane. Let it consume Starlink WAN SLAAC/DHCPv6-PD correctly, then route a usable IPv6 prefix or terminate the fast site-to-site tunnel there.
4. Move the high-performance direct tunnel endpoint to a Linux/OPNsense/WireGuard edge VM/device that handles Starlink IPv6 natively, then route inside networks through Palo/Fortigate policy.
5. Keep the Frankfurt VPS transit path as secondary fallback, remote operations, and benchmark/control path.
6. Ask Palo Alto support whether PAN-OS on PA-510 can hold a SLAAC global WAN address and DHCPv6-PD at the same time on Starlink. Current testing did not achieve that, even when DHCPv6-PD was disabled temporarily.
7. Avoid spending more time on a physical-WAN rebuild unless `ethernet1/1` can be shown to hold a global IPv6 address.

## Cleanup executed

- All temporary `iperf3` server processes were stopped on test systems after runs.

## Follow-up

- Re-run the same suite from a native host and then during a concurrent stream to compare quality metrics (startup delay, rebuffer events, dropped frames).
