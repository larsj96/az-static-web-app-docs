# Frankfurt VPS Ops Hub

VPS:

- Host: `root@72.61.95.150`
- Hostname observed: `srv1263160`
- OS observed: Ubuntu 24.04.3 LTS
- Role: public Frankfurt VPN/IPsec and ops hub

## Installed On 2026-05-19

- Docker Engine from Ubuntu packages
- Docker Compose v2 plugin
- Terraform CLI through Docker via `/usr/local/bin/tf`
- Terraform Cloud Agent Compose template at `/opt/homelab-ops/tfc-agent`
- Local VPS notes at `/opt/homelab-ops/README.md`
- Persistent homelab IPsec source route service: `homelab-ipsec-routes.service`
- R2/Cloudflare env file at `/root/.config/homelab/cloudflare-r2.env`

Terraform wrapper verification:

```bash
tf version
```

Observed Terraform version from `hashicorp/terraform:latest`:

```text
Terraform v1.15.3
```

## Terraform Cloud Agent

The Terraform Cloud Agent is running as a Docker Compose service.

Observed registration:

```text
Agent registered successfully with HCP Terraform
agent_id=agent-Kn1UTKqJqhBkv6kY
agent_pool_id=apool-VMAdwyVNzc5DyUL2
```

Check status on the VPS:

```bash
cd /opt/homelab-ops/tfc-agent
docker compose ps
docker logs --tail 80 tfc-agent
```

Restart after config changes:

```bash
cd /opt/homelab-ops/tfc-agent
docker compose up -d
```

The agent uses Docker host networking so Terraform jobs inherit the VPS route to homelab networks:

```yaml
network_mode: host
```

Keep that setting unless a replacement source-routing/SNAT design is added for Docker bridge traffic.

## Homelab Route Fix

The IPsec policy on the VPS matches:

```text
10.8.0.0/24 -> 10.0.0.0/16
```

The VPS itself was originally sourcing local traffic from public IP `72.61.95.150`, which missed that selector. The persistent route service fixes that by forcing `10.0.0.0/16` traffic to use source `10.8.0.1`:

```bash
systemctl status homelab-ipsec-routes.service
ip route get 10.0.0.162
```

Expected route:

```text
10.0.0.162 via 72.61.95.254 dev eth0 src 10.8.0.1
```

Known-good test:

```bash
curl -k --connect-timeout 5 -m 8 -o /dev/null -w 'http=%{http_code} remote=%{remote_ip}\n' https://10.0.0.162:8006/
docker run --rm --network host curlimages/curl:latest -k --connect-timeout 5 -m 8 -o /dev/null -w 'container_http=%{http_code} remote=%{remote_ip}\n' https://10.0.0.162:8006/
```

Expected:

```text
http=200 remote=10.0.0.162
container_http=200 remote=10.0.0.162
```

## Palo GlobalProtect Transit

GlobalProtect clients terminate on the Palo Alto cabin firewall, but the VPS is still part of the path when those clients reach Mo i Rana networks:

```text
GlobalProtect client 172.31.250.0/24
  -> Palo Alto NAT to 10.1.255.250
  -> Palo-to-VPS IPsec
  -> Frankfurt VPS
  -> VPS SNAT to 10.8.0.1
  -> Fortigate IPsec
  -> 10.0.0.0/16
```

The VPS SNAT is deliberate. The Fortigate-side selector expects `10.8.0.0/24 -> 10.0.0.0/16`, so Palo-originated transit traffic must arrive at the Fortigate tunnel as `10.8.0.1`.

Current nftables/iptables intent:

```text
10.1.0.0/16 -> 10.0.0.0/16 SNAT to 10.8.0.1
10.8.0.0/24 -> 10.0.0.0/16 accepted into Fortigate tunnel
10.0.0.0/16 -> 10.8.0.0/24 accepted back
```

Do not remove this while the Fortigate selectors stay scoped to the VPS transit subnet. If the Fortigate tunnel is later widened to include the Palo networks directly, document that as a deliberate migration.

## Direct Palo To Fortigate Fallback Role

As of `2026-05-23`, the direct Palo Alto to Fortigate IPv6 site-to-site tunnel is up and should be preferred for normal Palo-side to Mo i Rana-side traffic:

```text
Palo 10.1.0.0/16 -> Fortigate 10.0.0.0/16
transport: IPv6 IPsec
Palo tunnel: tunnel.20
Fortigate phase1: palo-ipv6
```

Palo route preference:

```text
10.0.0.0/16 -> tunnel.20 direct Fortigate IPv6 IPsec, metric 5
10.0.0.0/16 -> tunnel.10 Frankfurt VPS fallback, metric 50
```

The VPS remains important for:

```text
OpenVPN admin access
Terraform runs that need a stable public execution point
Fortigate-side Starlink CGNAT recovery path
fallback route if direct Palo-to-Fortigate IPv6 has an issue
external IPv6 validation for gp.lanilsen.com
```

Initial setup command, if rebuilding:

```bash
cd /opt/homelab-ops/tfc-agent
cp .env.example .env
nano .env
docker compose up -d
```

Keep the token out of Git and paste it directly into `.env` on the VPS.

## Terraform CLI Runs With R2 State

The active CLI-driven stacks run from:

```text
/opt/homelab-ops/repos/terraform-proxmox/proxmox-core
/opt/homelab-ops/repos/terraform-cloudflare/docs-tunnel
```

Credential source:

```bash
source /root/.config/homelab/cloudflare-r2.env
```

That file is sourced from `/root/.bashrc`, so new root shells should already have:

```text
CLOUDFLARE_API_TOKEN
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
R2_STATE_BUCKET
```

Do not copy the token into random shell history or repo files.

## Cloudflare IPv4 Pinning

The VPS has IPv6, and Cloudflare API calls may prefer it:

```text
2a02:4780:41:59ef::1
```

If a Cloudflare token is IP-restricted only to `72.61.95.150/32`, pin Docker Terraform runs to an IPv4 Cloudflare API address:

```bash
CF_API_IPV4=$(getent ahostsv4 api.cloudflare.com | awk 'NR==1{print $1}')
docker run --rm --network host --add-host "api.cloudflare.com:$CF_API_IPV4" ...
```

Better long term: include both source IPs in the token condition:

```text
72.61.95.150/32
2a02:4780:41:59ef::1/128
```

## Published Docs Operations

Cloudflare resources are managed from:

```bash
cd /opt/homelab-ops/repos/terraform-cloudflare/docs-tunnel
```

The docs origin is configured from bastion:

```bash
ssh ubuntu@10.0.0.99
cd ~/ansible-homelab
git pull
ansible-playbook playbooks/mkdocs.yml
```

Current public verification:

```bash
curl -I https://docs.lanilsen.com/
```

Expected unauthenticated result:

```text
HTTP/2 302
location: https://lanilsen.cloudflareaccess.com/...
```

## Important Boundary

Use the VPS Terraform Cloud Agent only for workspaces whose target APIs are deliberately reachable from Frankfurt over OpenVPN/IPsec.

Do not use this VPS agent for old Proxmox management on `192.168.13.0/24` unless that network is intentionally routed to the VPS. The intended routed Proxmox management path is VLAN 16, `10.0.0.160/27`.

## Maintenance Note

The VPS reported a pending kernel upgrade during setup. Reboot during a maintenance window:

```bash
reboot
```
