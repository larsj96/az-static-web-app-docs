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
```

The first switch requests a DHCPv6 non-temporary address (`IA_NA`) while keeping prefix delegation enabled. A targeted apply and Palo commit succeeded, but `ethernet1/1` still showed only:

```text
addr6: fe80::1ecf:82ff:fe6a:3710/64
dhcpv6_client: True
```

So Starlink/PAN-OS did not produce a global IPv6 address on the physical WAN interface through DHCPv6 IA_NA.

The second switch temporarily disables the DHCPv6 client to test pure Starlink WAN SLAAC behavior. Terraform could stage the change, but the Palo commit failed because LAN interfaces inherit addresses from the `starlink-ipv6-pd` prefix pool:

```text
ethernet1/3.10 -> ipv6 -> inherited -> assign-addr -> starlink-ipv6-pd-gua
prefix-pool 'starlink-ipv6-pd' is not a valid reference
```

The DHCPv6-PD config was restored and committed successfully after the failed pure-SLAAC test. Post-restore checks confirmed:

- `10.0.0.37` reachable from WSL
- `10.0.0.33` reachable from WSL
- VPS still had both Palo and Fortigate IPsec SAs established

Conclusion: with the current Palo IPv6 design, the physical Starlink interface cannot be used as the direct IPv6 IPsec endpoint without first redesigning delegated IPv6 usage on the Palo side.

## Interpretation

The results indicate:

- The VPS public destination path is comparatively healthy.
- The homelab-private direct IPv6 path is the current TCP bottleneck (much lower and unstable throughput with TCP retransmits).
- The UDP result shows the path can move packets, but TCP reacts badly to the latency/jitter/return-path behavior.
- The most suspicious design issue is Palo sourcing the IPsec flow from `loopback.12` with `hw-mode none`, because the Starlink physical WAN does not receive a usable global IPv6 address while DHCPv6-PD is active.

For 4K media streaming, this is insufficient. Even modest 4K streaming profiles can exceed this and are sensitive to jitter/retransmits.

## 4K readiness delta to close

Target baseline for comfortable 4K streaming over this path:

- Sustained throughput: **>100 Mbit/s**
- Low jitter (or at least stable latency under load)
- Minimal retransmits/congestion behavior on TCP tests, or robust UDP path checks with controlled loss

## Next-step troubleshooting checklist

1. Treat the direct IPv6 Palo-to-Fortigate tunnel as functional but **not performance-primary** for Plex yet.
2. Prefer the Frankfurt VPS path or Cloudflare/Plex-native access for media until the direct tunnel is fixed.
3. Compare with a native Linux host, not WSL, to remove WSL path variance.
4. Run UDP at higher rates (`iperf3 -u -b 50M`, then `100M`) while watching Fortigate/Palo counters.
5. Investigate whether Starlink/PAN-OS can provide a physical-interface global IPv6 without losing DHCPv6-PD. If not, a different edge design may be needed for a fast direct tunnel.
6. If keeping the loopback endpoint, test whether Palo route/policy/PBF can force return traffic consistently out `ethernet1/1`, though current symptoms suggest PAN-OS loopback endpoint handling may remain the bottleneck.
7. Keep the VPS fallback path documented and tested because it currently outperforms the direct Starlink-to-Starlink TCP path.

## Real solution paths

The likely durable fix is one of these, in this order:

1. Put a small IPv6-capable edge device in front of Palo (OpenWrt, MikroTik, OPNsense, or Linux). Let it consume Starlink WAN SLAAC/DHCPv6-PD correctly, then route a usable IPv6 prefix or terminate the fast site-to-site tunnel there.
2. Move the high-performance direct tunnel endpoint to a Linux/OPNsense/WireGuard edge VM/device that handles Starlink IPv6 natively, then route inside networks through Palo/Fortigate policy.
3. Keep Palo-to-Fortigate IPv6 IPsec as management/redundancy and make the Frankfurt VPS path or application-specific public access the primary path for media until the direct path is redesigned.
4. During a maintenance window, remove Palo inherited IPv6 LAN addressing temporarily and retest pure SLAAC on `ethernet1/1`. If the physical interface gets a GUA, rebuild the direct IPsec endpoint on the physical WAN.
5. Ask Palo Alto support whether PAN-OS on PA-510 can hold a SLAAC global WAN address and DHCPv6-PD at the same time on Starlink. Current testing did not achieve that.

## Cleanup executed

- All temporary `iperf3` server processes were stopped on test systems after runs.

## Follow-up

- Re-run the same suite from a native host and then during a concurrent stream to compare quality metrics (startup delay, rebuffer events, dropped frames).
