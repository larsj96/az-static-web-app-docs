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

## Interpretation

The results indicate:

- The VPS public destination path is comparatively healthy.
- The homelab-private path is the current bottleneck (much lower and unstable throughput with TCP retransmits).
- This pattern is consistent with tunnel/private-path degradation rather than a simple Fortigate WAN download issue.

For 4K media streaming, this is insufficient. Even modest 4K streaming profiles can exceed this and are sensitive to jitter/retransmits.

## 4K readiness delta to close

Target baseline for comfortable 4K streaming over this path:

- Sustained throughput: **>100 Mbit/s**
- Low jitter (or at least stable latency under load)
- Minimal retransmits/congestion behavior on TCP tests, or robust UDP path checks with controlled loss

## Next-step troubleshooting checklist

1. Verify direct-path state when Palo is active (`tunnel.20`, `palo-ipv6`) is actually carrying this traffic (route lookup on both ends).
2. Compare with a **native Linux host** (not WSL) to remove WSL path variance.
3. Measure UDP:
   - `iperf3 -u -b 100M ...`
4. Check MTU on each leg and tunnel policy:
   - `ip route`, `ip link`, `iptables/nft`/security policy, PMTUD behavior
5. Confirm offload/path asymmetry and CPU bottleneck on tunnel endpoints:
   - tunnel interface stats and `iperf` window growth behavior
6. If using OpenVPN-transit fallback, confirm policy/routing on VPS path is intentionally selected and not adding extra serial hops.

## Cleanup executed

- All temporary `iperf3` server processes were stopped on test systems after runs.

## Follow-up

- Re-run the same suite from a native host and then during a concurrent stream to compare quality metrics (startup delay, rebuffer events, dropped frames).
